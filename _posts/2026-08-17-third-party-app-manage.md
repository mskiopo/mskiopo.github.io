---
title: 给第三方 app 管理加 CRUD，栽在字段类型上
date: 2026-08-17 17:30:00 +0800
categories: [技术]
tags: [Spring Boot, MyBatis-Plus, PostgreSQL, 排错, 复盘]
---

> 记录 2026-08-17 在 idev 项目里新增「第三方 app」管理功能时的一次排障：库表的 `internal` 是 smallint，实体类里却写成了 `Boolean`，PG 强类型直接报了错。功能本身不复杂，真正花时间的是把第一个接口调通。

## 项目背景

AI 眼镜管理平台（idev 项目），Spring Boot + yudao 风格模块化工程，后端拆成 `idev-module-system` / `idev-server` 等模块，数据库是 PostgreSQL。

这次的功能：新增「第三方 app」管理，提供创建、更新、删除（逻辑删除）、详情、分页列表 5 个接口，后台补菜单和 4 个按钮权限。表是新建的 `system_app_manage`，字段有应用名、`client_key`（唯一）、`client_secret`、状态、图标、回调 url、`internal`（是否内部应用）。

按框架惯例，DO / Mapper / Service / Controller / VO 各建一遍，再插菜单权限数据。这套流程走下来不算难，真正卡住的是调试阶段的一个报错。

## 一、发生了什么

调 `/system/app-manage/create` 创建一条 app，返回 500，日志里是一条数据库报错：

```
ERROR: column "internal" is of type smallint but expression is of type boolean
```

`internal` 这一列，库表里是 `int2`（smallint），而 DO 里最初写的是 `Boolean`。MyBatis-Plus 把 `Boolean` 绑定成 PG 的 `boolean` 类型，PG 强类型直接拒绝：smallint 列收不下 boolean 值。

## 二、排查：报错其实已经指了路

这个报错把方向指得很明确，排查没绕路。

先确认周边都是正常的：权限注解生效、client_key 唯一校验生效、tenant_id 自动注入生效。报错只落在 `internal` 这一列，问题被圈定在字段类型。

再去对库表 DDL，`internal int2 DEFAULT 0`，DO 里是 `private Boolean internal`，答案就出来了：**Java 类型和列类型没对齐。**

修复本身很小，`internal` 从 `Boolean` 改成 `Integer`（0/1），一共 4 个文件：

- `AppManageDO.java`
- `AppManageSaveReqVO.java`
- `AppManagePageReqVO.java`
- `AppManageRespVO.java`

库表不用动，`internal int2` 保持不变，也不用重跑 update.sql。改完重新编译，自测通过。

## 三、错在哪

这个坑是自己埋的。

写 DO 时看着"是否内部应用"这个字段名，顺手就用了 `Boolean`。没有先翻库表 DDL，确认这一列到底是什么类型。

教训很朴素：**ORM 不会帮你做类型转换，PG 更不会。** 写实体类之前先对齐库表列类型，是强类型数据库下绕不开的一步。

## 四、顺带记下的几个点

排障之外，这次还有几件值得记的事。

### 1. 先确认库是什么

表字段是 PG 的 `int2`/`int8`，不是 MySQL 的 `tinyint`/`bigint`。这次建表 SQL 本来就按 PG 写，没踩到语法坑，但"动手前先确认库是什么"这条要刻进流程。

### 2. 表不会自动创建

`update.sql` 不会在应用启动时自动执行，要手动连数据库跑。这类环境细节不写进文档，下次部署的人就得猜。

### 3. 软删除和唯一索引，双保险有代价

`client_key` 唯一性做了两层保险：服务层校验 + DB 唯一索引 `(tenant_id, client_key)`。

问题出在软删除场景。记录删掉后数据还在（`deleted = 1`），再用相同 `client_key` 新建，会撞上唯一索引，因为旧记录还占着这个键。真遇到这个场景，处理方案是去掉 DB 唯一索引，只留服务层校验。

双保险不是没成本的，删除策略和唯一约束要放在一起设计。

### 4. 主键用雪花 ID

`@TableId(type = IdType.ASSIGN_ID)`，主键由应用生成，不依赖数据库序列。这是框架惯例，沿用即可。

## 五、经验提炼

1. **Java 类型和 DB 列类型必须对齐**。PG 下 smallint ↔ Integer、boolean ↔ Boolean，ORM 不做转换，错位直接报错。写 DO 前先看 DDL。
2. **报错信息会说话**。`column "internal" is of type smallint but expression is of type boolean`，类型不匹配已经写在报错里了。读报错，别急着猜。
3. **先确认库是什么**。PG 的 int2/int8 和 MySQL 的 tinyint/int，语法体系不同。
4. **软删除和唯一索引要一起设计**。逻辑删除的数据还在表里，唯一约束会挡住"删了再建"。
5. **环境细节写进文档**。表不会自动创建、SQL 要手动执行，一句话的事，不写就是给别人埋坑。

下次写实体类，第一件事是对齐库表列类型，再写字段。
