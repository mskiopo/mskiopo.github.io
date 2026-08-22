---
title: 第三方 app 管理接口实现
date: 2026-08-17 17:00:00 +0800
categories: [复盘]
tags: [Spring Boot, MyBatis-Plus, PostgreSQL, 排错, 复盘]
---

> 记录 2026-08-17 在 idev 项目里给「第三方 app」管理加 CRUD 的完整过程：建表、9 处代码改动、菜单权限数据，一路走到接口自测，最后在第一个接口上调出一个字段类型坑。功能本身不难，麻烦的是把类型对齐这件事想当然。

## 项目背景

AI 眼镜管理平台（idev 项目），Spring Boot + yudao 风格模块化工程，后端拆成 `idev-module-system` / `idev-server` 等模块，数据库是 PostgreSQL。

这次新增「第三方 app」管理：创建、更新、删除（逻辑删除）、详情、分页列表 5 个接口，后台补菜单和 4 个按钮权限。新表 `system_app_manage`，字段包含应用名、`client_key`（唯一）、`client_secret`、状态、图标、回调 url、`internal`（是否内部应用）。

## 一、开发流程

先定表，再按 DO / Mapper / Service / Controller / VO 的顺序建代码，最后插权限数据。一共 9 处改动。

### 1. 建表 SQL + 权限数据

`idev-server/src/main/resources/sql/update.sql`：

```sql
drop table if exists system_app_manage;
create table system_app_manage(
    id int8 NOT NULL PRIMARY KEY,              -- 主键(雪花ID,应用生成)
    name varchar(64),                          -- 应用名
    client_key varchar(64),                    -- 客户端编号(唯一)
    client_secret varchar(255),                -- 客户端密钥
    status int2 DEFAULT 0 NOT NULL,            -- 状态 0开启 1关闭
    icon varchar(512),                         -- 图标
    callback_url varchar(512),                 -- 回调url
    internal int2 DEFAULT 0 NOT NULL,          -- 是否内部应用 0否 1是
    creator varchar(64), create_time timestamp,  -- 审计字段
    updater varchar(64), update_time timestamp,
    deleted int2 DEFAULT 0 NOT NULL,             -- 逻辑删除
    tenant_id int8 DEFAULT 0 NOT NULL            -- 租户
);
create unique index uk_system_app_manage_tenant_client_key
    on system_app_manage(tenant_id, client_key);  -- 按租户维度 client_key 唯一
-- + comment on table/column 字段注释
-- + insert into system_menu(菜单 3000 + 4 个按钮权限 3001~3004)
```

几个关键点：

- 数据库是 **PostgreSQL**，字段用 `int2`/`int8`，不是 MySQL 的 `tinyint`/`int`。
- **`update.sql` 不会在应用启动时自动执行**，要手动连库跑。表头带 `drop table if exists`，重复执行会先删后建。
- 唯一索引按租户维度 `(tenant_id, client_key)` 建，多租户下 `client_key` 才唯一。
- 主键用雪花 ID，应用生成，不依赖数据库序列。

### 2. DO

`AppManageDO.java`：

```java
@TableName(value = "system_app_manage", autoResultMap = true)
public class AppManageDO extends TenantBaseDO {
    @TableId(type = IdType.ASSIGN_ID)   // 雪花ID,无需 DB 序列
    private Long id;
    private String name;
    private String clientKey;
    private String clientSecret;
    private Integer status;              // 对应 CommonStatusEnum 0/1
    private String icon;
    private String callbackUrl;
    private Integer internal;            // 与库表 int2 对齐,0/1
}
```

继承 `TenantBaseDO` 自动带出 `tenantId` + 审计字段（creator/createTime/updater/updateTime/deleted），不用自己写。字段类型要和库表逐列对齐，这里的 `internal` 就是后面埋坑的地方。

### 3. Mapper

复用框架的 `BaseMapperX` + `LambdaQueryWrapperX`，条件查询不用手写 XML：

```java
@Mapper
public interface AppManageMapper extends BaseMapperX<AppManageDO> {
    default AppManageDO selectByClientKey(String clientKey) { ... }   // 唯一性校验
    default PageResult<AppManageDO> selectPage(AppManagePageReqVO reqVO) { ... }
    // like name/clientKey, eq status/internal, order by id desc
}
```

### 4-5. Service 接口与实现

接口 5 个方法：`createAppManage` / `updateAppManage` / `deleteAppManage` / `getAppManage` / `getAppManagePage`。

实现的核心是校验顺序：

```java
// 创建:client_key 先查重,重复直接抛错
if (mapper.selectByClientKey(reqVO.getClientKey()) != null) {
    throw exception(APP_MANAGE_CLIENT_KEY_EXISTS);
}
// 更新:先校验存在,再查重(排除自身)
// 删除:先校验存在,再逻辑删除
```

异常统一走错误码，`APP_MANAGE_NOT_EXISTS` / `APP_MANAGE_CLIENT_KEY_EXISTS`。

### 6. Controller

`@RequestMapping("/system/app-manage")`，管理后台统一带 `/admin-api` 前缀。5 个接口每个都挂权限注解：

```java
@PostMapping("/create")
@PreAuthorize("@ss.hasPermission('system:app-manage:create')")
public Long createAppManage(@Valid @RequestBody AppManageSaveReqVO reqVO) { ... }
```

### 7. VO

三个：创建/更新入参、分页入参、返回详情。

入参里 `status` 用 `@InEnum(CommonStatusEnum)` 校验，`internal` 用 0/1 的 `Integer`。

### 8. 错误码

`ErrorCodeConstants.java` 在社交客户端区间末尾追加：

```java
// ========== 第三方 app 1-002-018-220 ==========
ErrorCode APP_MANAGE_NOT_EXISTS        = new ErrorCode(1_002_018_220, "第三方 app 不存在");
ErrorCode APP_MANAGE_CLIENT_KEY_EXISTS = new ErrorCode(1_002_018_221, "客户端编号已存在");
```

### 9. 菜单权限数据

`system_menu` 插入 1 个菜单 + 4 个按钮权限，后台可直接授权：

| id | 名称 | 类型 | 权限标识 |
|---|---|---|---|
| 3000 | 第三方app | 菜单(2) | — |
| 3001 | 第三方app查询 | 按钮(3) | `system:app-manage:query` |
| 3002 | 第三方app创建 | 按钮(3) | `system:app-manage:create` |
| 3003 | 第三方app更新 | 按钮(3) | `system:app-manage:update` |
| 3004 | 第三方app删除 | 按钮(3) | `system:app-manage:delete` |

## 二、接口清单

| 方法 | 路径 | 权限 | 说明 |
|---|---|---|---|
| POST | `/system/app-manage/create` | `system:app-manage:create` | 创建，返回 id |
| POST | `/system/app-manage/update` | `system:app-manage:update` | 更新 |
| POST | `/system/app-manage/delete` | `system:app-manage:delete` | 删除，参数 id |
| GET | `/system/app-manage/get` | `system:app-manage:query` | 详情，参数 id |
| GET | `/system/app-manage/page` | `system:app-manage:query` | 分页列表 |

## 三、部署与验证

1. 连接 PG `intelligent_device`，整段执行 `update.sql`。验证表存在、菜单权限数据在：
   ```sql
   select * from information_schema.tables where table_name = 'system_app_manage';
   select id, name, permission from system_menu where id between 3000 and 3004;
   ```
2. 启动 `idev-server`（local profile），确认启动无报错，MyBatis 扫到新 Mapper。本地先 `mvn compile -pl idev-module-system -am` 保证编译过。
3. 带登录 token 自测 5 个接口：create → get → page → update → delete，再用相同 `client_key` 重复 create，应报 `APP_MANAGE_CLIENT_KEY_EXISTS`（1002018221）。
4. 刷新管理后台，能看到「第三方app」菜单和按钮权限（前端组件路径 `system/appManage/index`）。

这套流程走下来不算难，真正的坑在自测阶段，见下一节。

## 四、问题与反思

调 `/system/app-manage/create` 创建一条 app，返回 500，日志里是一条数据库报错：

```
ERROR: column "internal" is of type smallint but expression is of type boolean
```

`internal` 这一列，库表里是 `int2`（smallint），而 DO 里最初写的是 `Boolean`。MyBatis-Plus 把 `Boolean` 绑定成 PG 的 `boolean` 类型，PG 强类型直接拒绝：smallint 列收不下 boolean 值。

排查没绕路。先确认周边都正常：权限注解生效、client_key 唯一校验生效、tenant_id 自动注入生效。报错只落在 `internal` 这一列，问题被圈定在字段类型。

再去对库表 DDL，`internal int2 DEFAULT 0`，DO 里是 `private Boolean internal`，答案就出来了：**Java 类型和列类型没对齐。**

修复本身很小，`internal` 从 `Boolean` 改成 `Integer`（0/1），一共 4 个文件：

- `AppManageDO.java`
- `AppManageSaveReqVO.java`
- `AppManagePageReqVO.java`
- `AppManageRespVO.java`

库表不用动，`internal int2` 保持不变，也不用重跑 update.sql。改完重新编译，自测通过。

这个坑是自己埋的。写 DO 时看着"是否内部应用"这个字段名，顺手就用了 `Boolean`，没有先翻库表 DDL 确认这一列到底是什么类型。

## 五、经验提炼

1. **写 DO 前先对齐库表列类型**。PG 下 smallint ↔ Integer、boolean ↔ Boolean，ORM 不做转换，错位直接报错。写实体类之前先看 DDL。
2. **报错信息会说话**。`column "internal" is of type smallint but expression is of type boolean`，类型不匹配已经写在报错里了。读报错，别急着猜。
3. **先确认库是什么**。PG 的 int2/int8 和 MySQL 的 tinyint/int，语法体系不同，建表 SQL 写错一行就是另一个坑。
4. **软删除和唯一索引要一起设计**。逻辑删除的数据还在表里，唯一约束会挡住"删了再建"，双保险是有代价的。
5. **环境细节写进文档**。表不会自动创建、SQL 要手动执行，一句话的事，不写就是给别人埋坑。

下次写实体类，第一件事是对齐库表列类型，再写字段。