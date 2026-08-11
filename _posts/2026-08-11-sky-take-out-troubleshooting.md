---
title: 苍穹外卖项目排错总结：从图片回显到启售/停售的 12 个坑
date: 2026-08-11 20:30:00 +0800
categories: [技术]
tags: [Spring Boot, MyBatis, 苍穹外卖, 排错, 经验总结]
---

> 记录两次排错：
> - **2026-08-10**：图片回显 + 新增菜品 500（问题 1~3）
> - **2026-08-11**：套餐/菜品启售停售功能开发（问题 4~12）

---

## 一、背景

在本地开发中完成了 OSS 文件上传、公共字段自动填充（`@AutoFill`）等功能后，联调时依次遇到：

1. 图片回显链路不通（上传接口返回空、配置错误）
2. 网页新增菜品时 `Request failed with status code 500`

最终全部修复，新增菜品功能正常。

---

## 二、排错时间线

| 阶段 | 问题 | 根因 | 结果 |
|---|---|---|---|
| 1 | 图片回显失败 | OSS 配置键笔误、endpoint 为内网地址、上传接口未实现 | 已修复 |
| 2 | 新增菜品 500（第一次） | JWT 拦截器未把员工 id 存入 `BaseContext` | 已修复 |
| 3 | 新增菜品 500（第二次） | 口味表插入 SQL 引用了不存在的 `type` 列 | 已修复 |

---

## 三、问题详解

### 问题 1：图片回显链路不通

**现象**：前端拿不到图片地址，无法回显。

**根因（三处）**：

- `application-dev.yml` 中配置键写成了 `aloss`，而 `application.yml` 引用的是 `sky.alioss.*`，占位符无法解析，`AliOssUtil` Bean 无法创建；
- endpoint 使用的是阿里云**内网地址** `oss-cn-beijing-internal.aliyuncs.com`，本地开发环境无法访问；
- `CommonController.upload()` 只是占位实现，没有调用 `AliOssUtil`，也没返回文件 URL。

**修复**：

```yaml
# application-dev.yml
sky:
  alioss:                                    # aloss → alioss
    endpoint: oss-cn-beijing.aliyuncs.com    # 去掉 -internal
    access-key-id: xxx
    access-key-secret: xxx
    bucket-name: web-go
```

`CommonController` 补全上传逻辑：生成 UUID 文件名 → 调用 `aliOssUtil.upload()` → 返回 URL。

**后续前提**：OSS 桶 `web-go` 需在阿里云控制台开启**公共读**，否则浏览器访问返回的 URL 仍是 403。

---

### 问题 2：新增菜品 500（第一阶段）—— `BaseContext` 未设置

**现象**：登录正常，但新增菜品返回 500。

**报错定位**：SQL 插入 dish 表违反 `NOT NULL` 约束（`create_user` / `update_user` 为 null）。

**调用链**：

```
POST /admin/dish → JwtTokenAdminInterceptor
  → DishServiceImpl.saveWithFlavor()
  → dishMapper.insert()  (@AutoFill(INSERT))
  → AutoFillAspect.autoFill()  →  BaseContext.getCurrentId()  ← 返回 null
  → dish.createUser/updateUser = null → INSERT 违反 NOT NULL → 500
```

**根因**：`JwtTokenAdminInterceptor.preHandle()` 解析出登录员工 id（`empId`）后，**没有调用 `BaseContext.setCurrentId(empId)`** 写入 ThreadLocal，导致自动填充切面取不到当前操作人。

**连带 bug**：`DishMapper.xml` 的 insert 未配置 `useGeneratedKeys`，`dish.getId()` 永远为 null，后续口味表的 `dish_id` 也无法写入。

**修复**（`JwtTokenAdminInterceptor.java`）：

```java
// preHandle 中，解析出 empId 后：
BaseContext.setCurrentId(empId);   // 写入当前登录员工 id

// 新增 afterCompletion，请求结束清理 ThreadLocal，防止内存泄漏：
@Override
public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                            Object handler, Exception ex) {
    BaseContext.removeCurrentId();
}
```

修复（`DishMapper.xml`）：

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="id">
```

顺带修复：`log.info("当前员工id：", empId)` 缺 `{}` 占位符、删除调试用的 `System.out.println`。

---

### 问题 3：新增菜品 500（第二阶段）—— 口味表引用不存在的 `type` 列

**现象**：第一阶段修复后 dish 主表插入成功，但仍在口味表插入时报 500。

**报错**：

```
ReflectionException: There is no getter for property named 'type' in 'class com.sky.entity.DishFlavor'
```

**根因**：`DishFlavor` 实体只有 `id / dishId / name / value` 四个属性，但 `DishFlavorMapper.xml` 的 `insertBatch` SQL 多写了 `type` 列和 `#{item.type}`（手误），MyBatis 找不到对应 getter。

**修复**（`DishFlavorMapper.xml`）：

```xml
<insert id="insertBatch">
    insert into dish_flavor (dish_id, name, value)   <!-- 去掉 type -->
    values
    <foreach collection="flavors" item="item" separator=",">
        (#{item.dishId}, #{item.name}, #{item.value})
    </foreach>
</insert>
```

---

## 四、涉及文件清单

| 文件 | 改动 |
|---|---|
| `sky-server/src/main/resources/application-dev.yml` | `aloss`→`alioss`；endpoint 改公网地址 |
| `sky-server/src/main/java/com/sky/controller/admin/CommonController.java` | 补全上传：注入 `AliOssUtil`、UUID 文件名、返回 URL |
| `sky-server/src/main/java/com/sky/interceptor/JwtTokenAdminInterceptor.java` | 增加 `BaseContext.setCurrentId()` / `afterCompletion()`；修日志占位符；删调试输出 |
| `sky-server/src/main/resources/mapper/DishMapper.xml` | insert 加 `useGeneratedKeys="true" keyProperty="id"` |
| `sky-server/src/main/resources/mapper/DishFlavorMapper.xml` | insertBatch 删除不存在的 `type` 列 |

---

## 五、经验教训

1. **配置键一定要对齐**：YAML 的键名、占位符引用、`@ConfigurationProperties` 的 prefix 三者必须完全一致，笔误（如 `aloss`/`alioss`）会导致启动期占位符解析失败，且不好排查。

2. **拦截器与 ThreadLocal 是配套的**：解析出登录用户后必须写入 `BaseContext`（ThreadLocal），并在请求结束清理，否则自动填充等功能取不到用户，或造成线程复用时的数据串扰。

3. **MyBatis 插入主键必须回填**：涉及外键（`dish_id`）写入时，主表 insert 必须配 `useGeneratedKeys="true" keyProperty="id"`，否则主键拿不到。

4. **实体 / XML / 表结构三者对齐**：SQL 里引用的每个字段都要能在实体中找到对应属性、在表中找到对应列，`type` 这类手误会以 `ReflectionException` 暴露。

5. **看 500 的正确姿势**：遇到 500 先看异常堆栈定位到具体 SQL/方法，再用「拆链路」的思路逐层验证（主表 → 从表），能快速缩小范围。

6. **密钥安全**：AccessKey 不应硬编码在配置文件中提交到 git，应改用环境变量注入，并定期在 RAM 控制台轮换。

---

## 六、2026-08-11：套餐 / 菜品「启售停售」功能开发排错

> 本次在实现「套餐、菜品启售/停售」时，暴露了 **9 个问题**，其中 2 个致命（编译不过、启动崩溃），2 个会导致数据丢失/脏数据，其余为代码质量与 SQL 语义问题。全部已修复并通过 `mvn compile`。

### 6.1 排错时间线

| 阶段 | 问题 | 根因 | 结果 |
|---|---|---|---|
| 1 | 编译失败 | `SetmealServiceImpl` 引用了未注入的 `dishMapper` | 已修复 |
| 2 | 启动崩溃 | `DishController` 两个方法映射同一 URL `/status/{status}` | 已修复 |
| 3 | 修改套餐描述不生效、修改人/时间不落库 | `SetmealMapper.xml` update 漏 `description` / `update_time` / `update_user` | 已修复 |
| 4 | 多表更新缺事务（脏数据隐患） | `update()` / `updateWithFlavor()` 未加 `@Transactional` | 已修复 |
| 5 | 批量删除在 for 循环里重复执行 | 逻辑未分层，删除语句误放循环体 | 已修复 |
| 6 | 死代码、参数注解、SQL 语义、魔法数字 | 见问题 9~12 | 已清理 |

### 6.2 问题详解

### 问题 4：`SetmealServiceImpl` 使用了未定义的 `dishMapper`（编译不过）

**现象**：`mvn compile` 编译失败，报 `Cannot resolve symbol 'dishMapper'`；`startOrStop()` 调用了 `dishMapper.getBySetmealId(id)`，但类里没有声明这个字段。

**根因**：从参考代码**只抄了"用法"**（`dishMapper.getBySetmealId(...)`），**没抄"依赖声明"**（`@Autowired private DishMapper dishMapper;`）。写完也没编译、没处理 IDE 红线。

**修复**：在类顶部补注入（`import com.sky.mapper.DishMapper` 原本已有）：

```java
@Autowired
private DishMapper dishMapper;
```

顺带删掉了从未使用过的 `@Autowired private SetmealService setmealService;`（无用自注入）。

**教训**：每个被引用的对象都要能回答"它从哪来"。写完代码先 `mvn compile` 再交付。

---

### 问题 5：`DishController` 重复映射 `/status/{status}`（启动崩溃）

**现象**：应用启动时报 `IllegalStateException: Ambiguous mapping`。

**根因**：新增 `status()` 方法前**没检查已有代码**。`updateStatus()`（`DishController.java:73-77`）早已完整实现了"启售/停售菜品"（Controller → Service → Mapper 一条链齐全），新方法又占用了同一个 URL `/admin/dish/status/{status}`。**编译不报，只有启动才报**。

**修复**：删除新增的 `status()` 方法，并同步删除冗余链路 `DishService.startOrStop()`、`DishServiceImpl.startOrStop()`，保留原有 `updateStatus` 一条链。

**教训**：加功能前先 `grep` 关键词（`status`、`startOrStop`）确认项目里已有什么，再决定要不要加；每次改动后要**启动应用**验证，不能只编译。

---

### 问题 6：`SetmealMapper.xml` 的 update 漏字段

**现象**：修改套餐时，**描述文字保存后不生效**；`update_time` / `update_user` 一直不更新。

**根因**（两处叠加）：
- `description`：同表 insert 明明写了，update 的 `<set>` 却漏了——**没和 insert 逐字段对照**；
- `update_time` / `update_user`：`@AutoFill(UPDATE)` 切面会用反射把这两个值**设进实体**，但 SQL 没写这两列 → **设置了等于白设**。这是最隐蔽的坑，以为切面处理了，其实数据没落地。

**修复**：对照 insert（`SetmealMapper.xml:7`）并仿照 `DishMapper.xml:30-31` 的写法补全：

```xml
<if test="description != null">
    description = #{description},
</if>
<if test="image != null">
    image = #{image},
</if>
<if test="status != null">
    status = #{status},
</if>
update_time = #{updateTime},
update_user = #{updateUser}
```

**教训**：update 语句要跟同表 insert **逐字段核对**；凡是标了 `@AutoFill` 的方法，必须确认对应 SQL 里**真的持久化了**这些字段。

---

### 问题 7：`update()` / `updateWithFlavor()` 缺 `@Transactional`

**现象**：不直接报错，但若"重插关联/口味"这一步失败，会出现"主表已改、从表已删、新数据没写入"的**脏数据**。

**根因**：两个方法都是「改主表 + 删从表 + 重插从表」的多步写操作，却没加事务注解——事务意识还没内化成条件反射（`saveWithDish`、`delete` 都加了，唯独 `update` 漏了）。

**修复**：

```java
@Override
@Transactional
public void update(SetmealDTO setmealDTO) { ... }

@Transactional
public void updateWithFlavor(DishDTO dishDTO) { ... }
```

**教训**：一个方法里出现 **≥2 条写库 SQL**，先加 `@Transactional`。

---

### 问题 8：`delete()` 把批量删除塞进 for 循环

**现象**：`setmealMapper.deleteByIds(ids)` 和 `setmealDishMapper.deleteBySetmealIds(ids)` 写在 for 循环内，**每次迭代都重复删同一批 id**。

**根因**：循环本意只是"逐个检查套餐是否在售"，但写的时候把真正的批量删除也顺手写进了循环体，**逻辑没分层**。

**修复**：循环内只做只读校验，删除语句移到循环外；状态判断改用常量 `StatusConstant.ENABLE`（原来硬编码 `== 1`）：

```java
@Transactional
public void delete(List<Long> ids) {
    // 先校验：套餐是否在售
    for (Long id : ids) {
        Setmeal setmeal = setmealMapper.getById(id);
        if (setmeal.getStatus() == StatusConstant.ENABLE) {
            throw new DeletionNotAllowedException(MessageConstant.SETMEAL_ON_SALE);
        }
    }
    // 再删除：只执行一次
    setmealMapper.deleteByIds(ids);
    setmealDishMapper.deleteBySetmealIds(ids);
}
```

**教训**：写之前先用注释把流程列出来：**循环（只读校验）→ 循环外（写操作执行一次）**。

---

### 问题 9：`SetmealDishMapper` 里加了无 SQL 支撑的 `update(Setmeal)`

**现象**：方法一旦被调用会抛 `BindingException: Invalid bound statement (not found)`；当前无人调用，属于**定时炸弹**。

**根因**：职责没分清——改套餐应该去 `SetmealMapper`（那里本来就有 `update`），却加到了管 `setmeal_dish` 表的 `SetmealDishMapper`；而且方法加了却**没配任何 SQL**（既无 `@Update` 注解，XML 里也没有对应语句）。

**修复**：删除该方法及多余的 import（`AutoFill` / `OperationType` / `Setmeal`）。

**教训**：**每个 Mapper 对应一张表**，按表分职责；加方法必配 SQL。用「**方法名 ↔ XML id ↔ namespace**」三对照检查。

---

### 问题 10：Controller 参数 `Long id` 不写注解

**现象**：功能"碰巧能跑"——Spring 对无注解的简单类型默认按 request param 绑定，但可读性差，且与项目其他接口不一致。

**根因**：对 Spring MVC 参数绑定规则理解停留在"能跑就行"；`CategoryController`、`EmployeeController` 里也有同样写法，属于**抄来的坏习惯**。

**修复**：

```java
public Result startOrStop(@PathVariable Integer status, @RequestParam Long id) { ... }
```

**教训**：三个注解明确写——路径 `@PathVariable`、URL 参数 `@RequestParam`、JSON 体 `@RequestBody`。

---

### 问题 11：`getBySetmealId` 的 `LEFT JOIN` 语义错误

**现象**：功能能查出结果，但 SQL 语义是错的。

**根因**：

```sql
select d.* from dish d left join setmeal_dish s on d.id = s.dish_id where s.setmeal_id = #{id}
```

- 对**右表**加 `WHERE` 会抵消 `LEFT JOIN` 的作用，退化为 `INNER JOIN`；
- 同一菜品在套餐里出现多行时，结果会**重复**。

**修复**：

```sql
select distinct d.* from dish d
inner join setmeal_dish s on d.id = s.dish_id
where s.setmeal_id = #{id}
```

**教训**：过滤右表直接写 `INNER JOIN`；需要保留左表全行时用 `LEFT JOIN` 且过滤条件放 `ON`；涉及一对多时用 `DISTINCT` 防重。

---

### 问题 12：魔法数字与无用代码

**现象**：`setmeal.getStatus() == 1` 硬编码；`@Autowired private SetmealService setmealService;` 从未被使用。

**根因**：不习惯用常量；抄模板时把不需要的字段也带了进来。

**修复**：状态判断一律用 `StatusConstant.ENABLE / DISABLE`；删除无用字段与 import。

**教训**：状态判断用常量，别写魔法数字；提交前清理无用代码。

---

### 6.3 涉及文件清单（本次）

| 文件 | 改动 |
|---|---|
| `SetmealServiceImpl.java` | 注入 `DishMapper`；`delete()` 移出循环 + 用常量；`update()` 加 `@Transactional`；删无用自注入 |
| `DishController.java` / `DishService.java` / `DishServiceImpl.java` | 删除重复的 `status` 链路；`updateWithFlavor()` 加 `@Transactional` |
| `SetmealMapper.xml` | update 补 `description` / `update_time` / `update_user` |
| `SetmealController.java` | `Long id` 补 `@RequestParam` |
| `DishMapper.java` | `getBySetmealId` 改 `INNER JOIN + DISTINCT` |
| `SetmealDishMapper.java` | 删除无 SQL 支撑的死方法 `update(Setmeal)` |
| `AutoFillAspect.java` | `e.printStackTrace()` → `log.error()`（两处） |

---

### 6.4 经验教训（本次）

1. **抄代码要抄"全套"**：用法 + 依赖注入 + 注解 + SQL，只抄一行用法，必出问题（#4、#9）。

2. **加功能前先摸清已有实现**：`grep` 关键词，确认项目里已经有什么，避免重复造轮子（#5）。

3. **三层验证缺一不可**：`mvn compile` 抓符号错误、**启动应用**抓映射错误、**调接口**抓逻辑错误（#4、#5）。

4. **字段对称性检查**：同表 insert ↔ update 逐字段对照；`@AutoFill` 标注的字段，确认 SQL 真的持久化（#6）。

5. **事务条件反射**：一个方法 ≥2 条写库 SQL → `@Transactional`（#7）。

6. **逻辑分层**：先列流程（校验循环 → 写操作一次），再写代码（#8）。

7. **SQL 语义要过关**：JOIN 类型的选择、`DISTINCT` 去重（#11）。

8. **规范优先**：显式 `@RequestParam`、用 `StatusConstant` 常量、清理死代码（#10、#12）。
