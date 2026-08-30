---
title: 苍穹外卖C端三模块复盘
date: 2026-08-26 21:30:00 +0800
pin: true
categories: [技术]
tags: [Spring Boot, Redis, Spring Cache, MyBatis, 苍穹外卖, 排错, 复盘]
---

> 复盘在苍穹外卖里的一段：C 端购物车存不进库、Redis 缓存连踩两个序列化坑，下单链路从接口到库里转一圈。排错收尾后，把套餐缓存、购物车、用户下单三个模块串起来梳理了一遍。

## 项目背景

苍穹外卖项目（`sky-server`），Spring Boot + MyBatis + MySQL，Redis 通过 Spring Cache 注解管理。C 端三块：套餐浏览是高频读，靠缓存减压；购物车负责加购减购；用户下单把购物车转成订单，多表写入。

排错集中在购物车和缓存，报错一个接一个，从"数据存不进去"一路到"下单跑通"。

## 一、购物车存不进去

### 现象：接口正常，表是空的

POST `/user/shoppingCart/add` 返回正常，日志也打了商品信息，但 `shopping_cart` 表一直是空表，`auto_increment` 不涨。

这个现象很误导人。接口不报错，谁会第一反应去查数据库有没有写进去。

### 排查：开通用日志抓真实 SQL

开 MySQL 通用日志（`SET GLOBAL general_log='ON'`，`log_output='TABLE'`），抓到真实执行的语句：

```sql
insert into shopping_cart(..., user_id, ...) value ('清炒西兰花', ..., null, ...)
```

`user_id` 传的是 null，表结构 `user_id bigint NOT NULL`，MySQL 拒收。**报错被数据库吞掉，接口层面看起来一切正常。**

### 根因：C 端没有认证拦截器

顺着 user_id 追，`BaseContext.getCurrentId()` 拿到 null。Admin 端有 `JwtTokenAdminInterceptor` 拦 `/admin/**`，**C 端一个拦截器都没注册**。JWT 解析了没人往 ThreadLocal 写，userId 自然恒为 null。

中间走过弯路：临时 `BaseContext.setCurrentId(1L)` 写死 userId 让数据先落库，还把 C 端拦截器整个移除过。最后建了 `JwtTokenUserInterceptor`，参照 admin 版，`preHandle` 解析 token 后 `BaseContext.setCurrentId(userId)`，`afterCompletion` 里 `BaseContext.removeCurrentId()` 防泄漏。

注册时把浏览类接口排除掉：

```java
registry.addInterceptor(jwtTokenUserInterceptor)
        .addPathPatterns("/user/**")
        .excludePathPatterns("/user/user/login")
        .excludePathPatterns("/user/shop/status")
        .excludePathPatterns("/user/category/**")
        .excludePathPatterns("/user/dish/list")
        .excludePathPatterns("/user/setmeal/list")
        .excludePathPatterns("/user/setmeal/dish/**");
```

还有一个配置对齐的坑：`application.yml` 里 `user-token-name` 写的是 `weixtoken`，前端实际请求头是 `authentication`。名字对不上，拦截器取不到 token，一律 401。**前端传什么头，配置上就得写什么。**

### 教训

涉及"当前用户"的功能，动手前先确认两件事：认证拦截器有没有覆盖这条路径，ThreadLocal 有没有人写、有没有人清。缺一环，"能跑"和"数据对"就是两回事。

## 二、Redis 缓存连踩两个序列化坑

### ClassCastException：String 序列化器存了 List

`DishController.list` 报 `ArrayList cannot be cast to String`，发生在 `StringRedisSerializer.serialize`。

`RedisConfiguration` 把 value 序列化器设成 `StringRedisSerializer`，它只能处理 String。接口存的却是 `List<DishVO>`，传进去必然强转失败。修复是换成 `GenericJackson2JsonRedisSerializer`，或者不设序列化器走默认 JDK 序列化，实体类实现 `Serializable`。

**序列化器是"存什么类型"的声明，存 List 却配 String 专用，等于自己给自己挖坑。**

### EOFException：换了序列化器，旧 key 没清

`ShopController.getStatus` 报 `EOFException`，发生在 `JdkSerializationRedisSerializer.deserialize`。

根因是 `Shop_Status` 这个 key 是旧配置时期用裸字符串写进去的。换成 JDK 序列化后，用 `ObjectInputStream` 去读裸字符串，流头魔数对不上，直接 EOF。

真正有价值的教训在这里：**换序列化器之后，历史遗留同 key 的脏数据必须清掉**，否则读出来必然反序列化异常。

## 三、双 Redis 并存，排查绕了大圈

现象很怪：某个 Redis 里查不到 key，应用却有数据。

查 `netstat -ano | findstr :6379`，真相是这台机器两个 Redis 同时监听：**Memurai**（Windows 服务，应用实际连的就是它）和 **WSL 里的 redis-server**（通过 wslrelay 转发）。两个实例数据完全不是一回事，对着错误的实例查 key，当然查不到。

定位方法很朴素：先看监听进程是哪一个，`wslrelay.exe` 是 WSL 转发痕迹，`memurai` 是 Windows 原生。项目后来把 Redis 切到 db1，多个项目用不同 db 隔离。

**多环境机器上，排查缓存问题第一步不是看代码，是先确认连的是哪个实例。**

## 四、几个"小"错，全是契约问题

这轮还连着踩了几个注解/命名的小坑，每个都不难，但容易连环触发：

- `@Cacheable` 导错包：IDEA 补全出来的是 `springfox.documentation.annotations.Cacheable`，写上去不报错但完全不生效。**同名注解好几个，写缓存注解先看 import 行是不是 `org.springframework.cache`。**
- MyBatis `#{}` 匹配的是 Java 属性名不是 SQL 列名：`#{user_id}` 直接报 `There is no getter for property named 'user_id'`，改成 `#{userId}`。
- POST 传 JSON 少了 `@RequestBody`：DTO 里 dishId/setmealId 全是 null，接口看着正常，数据是空的。
- Service 实现类漏写 `implements`：容器里没有接口类型的 Bean，注入直接报 `required a bean of type ... that could not be found`。顺带把接口方法名 `addShoppingCard`（Card）和实现 `addShoppingCart`（Cart）不一致的笔误一起修了。

这四个错放在一起看，本质都是**接口契约的两端没对上**：注解用错包、占位符用错命名、参数没声明解析方式、实现没声明接口。

## 五、三个模块的实现流程与代码

排错只是让功能"能跑"，把机制吃透还得看实现。把套餐缓存、购物车、用户下单三块的流程和代码串一遍。

### 套餐缓存：注解声明缓存

C 端套餐浏览是高频读，查询接口直接叠 `@Cacheable`；管理端对套餐做增删改时用 `@CacheEvict` 清缓存，保证改完立即可见。

```java
@RestController("userSetmealController")   // 与 admin 包同名 Controller 区分 Bean 名
@RequestMapping("/user/setmeal")
public class SetmealController {
    @GetMapping("/list")
    @Cacheable(cacheNames = "setmealCache", key = "#categoryId")
    public Result<List<Setmeal>> list(Long categoryId) {
        Setmeal setmeal = new Setmeal();
        setmeal.setCategoryId(categoryId);
        setmeal.setStatus(StatusConstant.ENABLE);
        return Result.success(setmealService.list(setmeal));
    }
}
```

执行流程：

1. AOP 拦截到 `@Cacheable`，先去 Redis 找 `setmealCache::10001` 这样的 key
2. 命中：不执行方法体，缓存值反序列化直接返回
3. 未命中：执行方法查库，返回值自动序列化写回 Redis

管理端写操作分两种粒度：

```java
@PostMapping        // 新增：套餐只归属一个分类，精确删对应 key，误伤最小
@CacheEvict(cacheNames = "setmealCache", key = "#setmealDTO.categoryId")
public Result save(@RequestBody SetmealDTO setmealDTO) { ... }

@DeleteMapping      // 批量删：可能跨多个分类，无法穷举 key，全清
@CacheEvict(cacheNames = "setmealCache", allEntries = true)
public Result delete(@RequestParam List<Long> ids) { ... }

@PutMapping         // 修改、启停售同理，状态变化影响所有分类的可见性，全清最安全
@CacheEvict(cacheNames = "setmealCache", allEntries = true)
public Result update(@RequestBody SetmealDTO setmealDTO) { ... }
```

配套还有两件事：启动类加 `@EnableCaching`；被缓存的 `Setmeal` 实现 `Serializable`，默认 JDK 序列化才能落 Redis。

### 购物车：同一商品合并数量

How（判断同一商品）：userId + dishId + dishFlavor 共同决定"是不是同一件"，口味不同算两条记录（"微辣土豆丝"和"中辣土豆丝"）；套餐按 userId + setmealId。购物车还冗余了 name/image/amount 快照，下单后才不怕商家改价。

加购核心逻辑：

```java
@Override
public void addShoppingCart(ShoppingCartDTO dto) {
    ShoppingCart cart = new ShoppingCart();
    BeanUtils.copyProperties(dto, cart);          // dishId/setmealId/dishFlavor
    cart.setUserId(BaseContext.getCurrentId());   // 当前用户从 ThreadLocal 取

    List<ShoppingCart> list = shoppingCartMapper.list(cart);  // 动态 SQL 精确查是否已存在

    if (list != null && list.size() > 0) {        // 已存在：数量 +1
        ShoppingCart c = list.get(0);
        c.setNumber(c.getNumber() + 1);
        shoppingCartMapper.updateNumberById(c);
    } else {                                      // 不存在：查商品信息冗余快照后插入
        Long dishId = dto.getDishId();
        if (dishId != null) {
            Dish dish = dishMapper.getById(dishId);
            if (dish == null) throw new ShoppingCartBusinessException(MessageConstant.DISH_NOT_FOUND);
            cart.setName(dish.getName()); cart.setImage(dish.getImage()); cart.setAmount(dish.getPrice());
        } else {
            Setmeal setmeal = setmealMapper.getById(dto.getSetmealId());
            if (setmeal == null) throw new ShoppingCartBusinessException(MessageConstant.SETMEAL_NOT_FOUND);
            cart.setName(setmeal.getName()); cart.setImage(setmeal.getImage()); cart.setAmount(setmeal.getPrice());
        }
        cart.setNumber(1);
        cart.setCreateTime(LocalDateTime.now());
        shoppingCartMapper.insert(cart);
    }
}
```

减购是 add 的对称操作：

```java
if (cart.getNumber() > 1) {
    cart.setNumber(cart.getNumber() - 1);          // 数量 > 1：减一
    shoppingCartMapper.updateNumberById(cart);
} else {
    shoppingCartMapper.deleteById(cart.getId());   // 数量 = 1：整条删除
}
```

`list` 查询是 XML 动态 SQL，一个方法两用：查"我的购物车全部"只传 userId，查"某件商品是否存在"传齐各字段。

```xml
<select id="list" resultType="com.sky.entity.ShoppingCart">
    select * from shopping_cart
    <where>
        <if test="userId != null"> and user_id=#{userId} </if>
        <if test="setmealId != null"> and setmeal_id=#{setmealId} </if>
        <if test="dishId != null"> and dish_id=#{dishId} </if>
        <if test="dishFlavor != null"> and dish_flavor=#{dishFlavor} </if>
    </where>
</select>
```

### 用户下单：一对多原子写

Why（为什么整段包事务）：下单 = 插 1 条主单 + 批量插 N 条明细 + 清空购物车，任一步失败都要整体回滚，否则会留下"有主单没明细"的脏数据。

```java
@Transactional
public OrderSubmitVO submitOrder(OrdersSubmitDTO dto) {
    // 1. 校验收货地址
    AddressBook addressBook = addressBookMapper.getById(dto.getAddressBookId());
    if (addressBook == null)
        throw new AddressBookBusinessException(MessageConstant.ADDRESS_BOOK_IS_NULL);

    // 2. 校验购物车非空
    Long userId = BaseContext.getCurrentId();
    ShoppingCart query = new ShoppingCart();
    query.setUserId(userId);
    List<ShoppingCart> cartList = shoppingCartMapper.list(query);
    if (cartList == null || cartList.isEmpty())
        throw new ShoppingCartBusinessException(MessageConstant.SHOPPING_CART_IS_NULL);

    // 3. 插主单，useGeneratedKeys 回填 id
    Orders orders = new Orders();
    BeanUtils.copyProperties(dto, orders);
    orders.setOrderTime(LocalDateTime.now());
    orders.setNumber(String.valueOf(System.currentTimeMillis()));
    orders.setUserId(userId);
    orders.setPhone(addressBook.getPhone());      // 地址簿快照冗余
    orders.setConsignee(addressBook.getConsignee());
    orders.setAddress(addressBook.getDetail());
    orderMapper.insert(orders);                   // 执行完 orders.getId() 已有值

    // 4. 购物车转订单明细，批量插入
    List<OrderDetail> detailList = new ArrayList<>();
    for (ShoppingCart c : cartList) {
        OrderDetail od = new OrderDetail();
        BeanUtils.copyProperties(c, od);          // 名称/图片/单价/数量/口味一并拷走
        od.setOrderId(orders.getId());
        detailList.add(od);
    }
    orderDetailMapper.insertBatch(detailList);

    // 5. 清空购物车
    shoppingCartMapper.deleteById(userId);

    // 6. 返回给前端去支付
    return OrderSubmitVO.builder().id(orders.getId()).orderTime(orders.getOrderTime())
            .orderNumber(orders.getNumber()).orderAmount(orders.getAmount()).build();
}
```

一对多批量插入有两个关键点。

主键回填（OrderMapper.xml）：

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into orders(number, status, user_id, address_book_id, order_time, ...)
    values (#{number}, #{status}, #{userId}, #{addressBookId}, #{orderTime}, ...)
</insert>
```

明细批量 foreach（OrderDetailMapper.xml）：

```xml
<insert id="insertBatch">
    insert into order_detail(name, image, order_id, dish_id, setmeal_id, dish_flavor, number, amount)
    values
    <foreach collection="orderDetailList" item="od" separator=",">
        (#{od.name},#{od.image},#{od.orderId},#{od.dishId},#{od.setmealId},#{od.dishFlavor},#{od.number},#{od.amount})
    </foreach>
</insert>
```

How（一对多插入套路）：`useGeneratedKeys="true"` 让 JDBC 取回自增主键并赋回参数对象，明细才能拿到 orderId。foreach 的 `separator` 用英文逗号，取属性要带前缀 `#{od.xxx}`，写成顶层 `#{dishFlavor}` 会报 `BindingException`。

留一个疑点：`submitOrder` 里 `orders.setStatus(Orders.PAID)`，但 `payStatus` 还是未支付。项目模拟支付流程跳过了微信支付，状态被提前置为已付款。逻辑上不严谨，做支付模块时得改回待付款。

## 六、沉淀成排查顺序

这轮攒下的，排成一张检查单：

1. 换序列化器 / 换存储，先清历史脏数据
2. `#{}` 是 Java 属性名，不是 SQL 列名
3. POST 传 JSON 必须有 `@RequestBody`
4. 接口 404，先确认 Controller 里映射真的存在
5. NPE 前先判空，抛业务异常而不是裸崩
6. "改了还报错"先对堆栈行号与源码，排除旧进程旧编译产物
7. 多实例环境先 `netstat` 定位监听进程，再谈查数据
8. 用户身份相关的功能，先确认拦截器覆盖和 ThreadLocal 写入链路

## 结尾

这轮最大的体会：排查效率的差别不在"会不会看日志"，在于**对环境的判断顺序**。先确认连的是哪个 Redis、哪个拦截器在生效、配置和前端对不对得上，再往代码深处走，能省掉大半弯路。

下次再做新模块，先把"身份从哪来、数据落在哪个实例、契约两端是否对齐"这三件事问清楚，再动手写业务。