---
layout:      post
title:       "MyBatis 插件机制与 SQL 改写"
subtitle:    "从拦截器、分页、数据权限到 BoundSql/MappedStatement/SqlSource 源码关系"
author:      "Ekko"
header-img:  "img/bg/bg-mysql.jpg"
catalog:     true
tags:
  - 学习笔记
  - Java
  - MyBatis
  - MyBatis-Plus
  - 数据库
---

> 这篇笔记不再只把视角停留在 `Interceptor` 或 `InnerInterceptor` 的“怎么用”，而是把问题拉到更底层：**MyBatis 如何从 Mapper 方法一路生成 SQL、组织参数、进入 JDBC，再允许插件在执行链上改写 SQL**。

> 如果只是想回答“业务方怎么拿到将要执行的 SQL”，一句话就够了：**运行期最核心的对象是 `BoundSql`**。但如果想回答“为什么分页插件、数据权限插件、动态表名插件都能工作”，那就必须把 `SqlSource`、`MappedStatement`、`BoundSql`、`Executor`、`StatementHandler` 这一整套对象关系看明白。

> 参考资料：
>
> MyBatis 官方配置文档：https://mybatis.org/mybatis-3/configuration.html#plugins
>
> MyBatis `Configuration` 源码文档：https://mybatis.org/mybatis-3/jacoco/org.apache.ibatis.session/Configuration.java.html
>
> MyBatis `MappedStatement` 源码文档：https://mybatis.org/mybatis-3/zh_CN/jacoco/org.apache.ibatis.mapping/MappedStatement.java.html
>
> MyBatis `BoundSql` 源码文档：https://mybatis.org/mybatis-3/zh_CN/jacoco/org.apache.ibatis.mapping/BoundSql.java.html
>
> MyBatis-Plus `InnerInterceptor` Javadoc：https://javadoc.io/doc/com.baomidou/mybatis-plus-extension/3.5.5/com/baomidou/mybatisplus/extension/plugins/inner/InnerInterceptor.html

[TOC]

---

## 一、先把问题说透：为什么研究 MyBatis 插件，不只是为了写一个拦截器

大多数人第一次接触 MyBatis 插件，出发点都很朴素：

- 我想打印最终 SQL
- 我想在 SQL 上自动追加租户条件
- 我想统一做数据权限
- 我想做分页，不想每个 Mapper 手写 `limit`
- 我想按月份动态切表

这些问题表面上看像五件事，底层其实是一件事：

> **在 MyBatis 的执行链中，找到一层既看得见“本次真实要执行的 SQL”，又还能影响后续执行结果。**

所以真正值得研究的不是某个单独插件，而是下面三个问题：

1. SQL 是在哪一层“定型”的
2. 参数是怎么和 SQL 绑定到一起的
3. MyBatis 允许在哪些点把这份 SQL 拿出来、改掉、再继续执行

如果这三件事理解透了，那么：

- 原生 `Interceptor`
- MyBatis-Plus 的 `InnerInterceptor`
- 分页插件
- 数据权限插件
- 动态表名插件
- 多租户插件

本质上都只是同一套执行模型上的不同扩展策略。

### 1.1 不同层次的问题，应该用不同层次的方案

实际开发里常见需求大致可以分成三类。

第一类是“观测型”需求：

- 打印待执行 SQL
- 统计慢 SQL
- 做链路埋点
- 记录审计日志

这类需求的重点是“看见”，不是“重写”。

第二类是“轻改写”需求：

- 动态表名
- 自动加 hint
- 自动加少量 where 片段
- 某些 SQL 注释注入

这类需求重点是“改一点点”。

第三类是“重组型”需求：

- 分页插件生成分页 SQL 和 count SQL
- 数据权限按规则注入复杂 where 条件
- 多租户在多表 join、子查询下重写 SQL
- 动态分表不只是换表名，还要同步调整 count / update / delete 语义

这类需求重点已经不是“改字符串”，而是“重新组织一套执行计划输入”。

MyBatis 插件机制的价值，就在于它能同时覆盖这三类需求，但三类需求对应的切入点和实现姿势并不一样。

---

## 二、MyBatis 一条 SQL 的运行主链：插件为什么只能拦这几层

如果只记 API，不记执行链，后面很容易混乱。先把运行路径拉平。

从一次普通查询来看，大致可以理解为：

```text
Mapper 方法
  -> MapperProxy / MapperMethod
  -> SqlSession
  -> Executor
  -> StatementHandler
  -> ParameterHandler
  -> JDBC PreparedStatement
  -> 数据库
  -> ResultSetHandler
  -> 返回对象
```

如果只看这张链路图，很容易产生一个误解：

- 这些名字到底是“概念”
- 还是“接口”
- 还是“具体类”
- 还是“某个阶段的方法调用”

这件事如果不先分清，后面看 `Interceptor` 会越来越糊。

### 2.1 先把这些名字分层：哪些是入口代理，哪些是核心接口，哪些是默认实现

最简单的理解方式，是把它们分成 4 层：

| 层次 | 代表对象 | 它更像什么 |
| --- | --- | --- |
| Mapper 调用入口层 | `MapperProxy`、`MapperMethod` | Java 代理入口与方法分发器 |
| 会话门面层 | `SqlSession` | 对外执行门面接口 |
| 执行与 JDBC 协作层 | `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler` | MyBatis 核心 SPI 接口 |
| JDBC / 数据库层 | `PreparedStatement`、`ResultSet`、数据库 | 真正执行 SQL 的基础设施 |

如果再说得更直白一点：

- `MapperProxy`：是 Mapper 接口的动态代理实现，不是“概念名词”，而是真正的代理类
- `MapperMethod`：是对某个 Mapper 方法的解析结果与执行封装，它更像“方法调用计划”
- `SqlSession`：是对外暴露的执行门面，业务最少会直接感知到这一层
- `Executor` / `StatementHandler` / `ParameterHandler` / `ResultSetHandler`：是 MyBatis 内部最核心、最稳定的几个扩展接口
- `PreparedStatement` / `ResultSet`：已经是 JDBC 世界，不再是 MyBatis 自己抽象出来的层

所以这条链并不是“全是类名”，而是：

- 一部分是接口
- 一部分是默认实现或代理实现
- 一部分是职责名
- 一部分是 JDBC 对象

### 2.2 用“接口 / 默认实现 / 职责”三个维度分别看

下面这张表更适合建立整体感。

| 名字 | 它主要是什么 | 常见实现或形态 | 在链路里的职责 |
| --- | --- | --- | --- |
| `MapperProxy` | 具体代理类 | JDK 动态代理关联的 `InvocationHandler` | 把 Mapper 接口方法调用转成 MyBatis 执行 |
| `MapperMethod` | 方法调用封装对象 | 每个 Mapper 方法对应一个 | 判断命令类型并调用 `SqlSession` |
| `SqlSession` | 接口 / 门面 | `DefaultSqlSession` | 提供 `selectOne`、`selectList`、`update` 等统一入口 |
| `Executor` | 核心接口 | `SimpleExecutor`、`ReuseExecutor`、`BatchExecutor`、`CachingExecutor` | 调度 query/update、缓存、批处理 |
| `StatementHandler` | 核心接口 | `RoutingStatementHandler`、`PreparedStatementHandler` 等 | 创建 `Statement` 并准备 SQL |
| `ParameterHandler` | 核心接口 | `DefaultParameterHandler` | 把 Java 参数绑定到 JDBC 占位符 |
| `ResultSetHandler` | 核心接口 | `DefaultResultSetHandler` | 把结果集映射回 Java 对象 |
| `Interceptor` | 插件接口 | 你自己实现的插件类 | 代理并增强上面几个核心接口的调用 |

这里最关键的一句是：

> **`Interceptor` 不在业务 SQL 执行主链“中间凭空多一层”，它是附着在 `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler` 这些核心接口外面的代理增强机制。**

这句话必须单独记住。

### 2.3 再落到具体类：接口 -> 默认实现类 映射表

如果你还是觉得“都是接口名”，那就继续往下落一层，直接看常见默认实现。

| 抽象名 | 类型 | 常见默认实现类 | 说明 |
| --- | --- | --- | --- |
| `SqlSession` | 门面接口 | `DefaultSqlSession` | 业务最常间接使用到的默认会话实现 |
| `Executor` | 核心接口 | `SimpleExecutor` | 最基础的执行器，每次创建新的 `Statement` |
| `Executor` | 核心接口 | `ReuseExecutor` | 会复用 `Statement` |
| `Executor` | 核心接口 | `BatchExecutor` | 批处理执行器 |
| `Executor` | 核心接口 | `CachingExecutor` | 对基础 `Executor` 再包一层二级缓存能力 |
| `StatementHandler` | 核心接口 | `RoutingStatementHandler` | 外层路由实现，按语句类型分发到具体 Handler |
| `StatementHandler` | 核心接口 | `PreparedStatementHandler` | 最常见，处理预编译 SQL |
| `StatementHandler` | 核心接口 | `SimpleStatementHandler` | 对应普通 `Statement` |
| `StatementHandler` | 核心接口 | `CallableStatementHandler` | 对应存储过程调用 |
| `ParameterHandler` | 核心接口 | `DefaultParameterHandler` | 默认参数绑定实现 |
| `ResultSetHandler` | 核心接口 | `DefaultResultSetHandler` | 默认结果集映射实现 |
| `Interceptor` | 插件接口 | 业务自定义实现类 | 例如日志插件、租户插件、数据权限插件 |

这张表里最容易在调试和插件开发里遇到的几个类通常是：

- `DefaultSqlSession`
- `CachingExecutor`
- `SimpleExecutor`
- `RoutingStatementHandler`
- `PreparedStatementHandler`
- `DefaultParameterHandler`
- `DefaultResultSetHandler`

尤其是：

- 大多数查询最终都会走到 `PreparedStatementHandler`
- 很多项目里 `Executor` 外层经常会先看到 `CachingExecutor`
- 你在 `StatementHandler` 插件里看到的往往不是最里层实现，而是 `RoutingStatementHandler`

所以实际调试时，最常见的体感通常不是：

- 我拿到了一个干净的 `StatementHandler`

而是：

- 我拿到了一个代理对象，往里看是 `RoutingStatementHandler`，再往里看是 `PreparedStatementHandler`

这也是为什么很多插件代码里会出现：

```java
metaObject.getValue("delegate.boundSql.sql")
```

因为这里的 `delegate`，就是 `RoutingStatementHandler` 内部真正持有的具体实现。

### 2.4 这条链到底是谁调谁，不要只记成一串名词

更贴近实际的理解方式如下：

```text
业务调用 UserMapper.selectById(...)
  -> MapperProxy 拦截接口调用
  -> MapperMethod 判断这是 SELECT 还是 UPDATE
  -> MapperMethod 调用 SqlSession 对应方法
  -> DefaultSqlSession 把调用转给 Executor
  -> Executor 在执行过程中创建/使用 StatementHandler
  -> StatementHandler 继续使用 ParameterHandler 绑定参数
  -> JDBC PreparedStatement 执行 SQL
  -> 查询结果返回后，ResultSetHandler 负责结果映射
  -> Executor / SqlSession / MapperProxy 逐层返回结果
```

这时就容易看明白：

- `MapperProxy` 是“入口代理”
- `MapperMethod` 是“方法分发与命令封装”
- `SqlSession` 是“门面”
- `Executor` 是“执行总调度”
- `StatementHandler` / `ParameterHandler` / `ResultSetHandler` 是“更贴近 JDBC 的三个执行协作对象”

### 2.5 为什么说这 4 个 Handler/Executor 更像稳定扩展边界

因为 `MapperProxy`、`MapperMethod`、`DefaultSqlSession` 这些对象虽然很重要，但它们更像：

- 框架入口
- 调用编排
- 对外 API 门面

而真正和“SQL 如何执行”强相关、同时又在运行期边界稳定的，是：

- `Executor`
- `StatementHandler`
- `ParameterHandler`
- `ResultSetHandler`

这就是为什么 MyBatis 插件机制最终只对这 4 类对象开放。

这里最关键的不是“顺序”本身，而是每一层职责不同。

- `Executor`：执行层总调度，负责 query/update、一级缓存、二级缓存协作、批处理等
- `StatementHandler`：负责把 SQL 组织成 JDBC `Statement`，更接近“即将执行的 SQL”
- `ParameterHandler`：负责把 Java 参数设置进 `PreparedStatement`
- `ResultSetHandler`：负责把 JDBC 结果集映射成 Java 对象

MyBatis 官方插件机制之所以只允许拦截这 4 类对象，不是随便定的，而是因为这 4 类对象正好构成运行期最稳定、最关键的边界。

### 2.6 为什么不是“任何地方都能拦截”

因为插件机制不是字节码增强，也不是任意切面，它本质上是：

- 创建核心对象时
- 把对象交给 `InterceptorChain.pluginAll(...)`
- 通过 JDK 动态代理包装成代理对象
- 代理对象只对约定接口的方法生效

也就是说，插件能拦截谁，不取决于你想拦谁，而取决于：

- MyBatis 有没有把这个对象暴露在插件链里
- 这个对象是不是以接口方式被代理
- 你声明的 `@Signature` 和真实方法是否匹配

所以插件的边界从一开始就是“有限的、受控的”，这也是它相对稳定的原因。

### 2.7 `Interceptor` 和这条执行主链到底是什么关系

很多人第一次看插件，最容易误会成：

- 执行链本来就有 `Interceptor`
- SQL 是先经过 `Interceptor`，再进入 `Executor`

这个理解并不准确。

更准确的描述应该是：

```text
MyBatis 创建 Executor / StatementHandler / ParameterHandler / ResultSetHandler
  -> InterceptorChain.pluginAll(target)
  -> 如果某个插件声明要拦这个接口的方法
  -> 就把 target 包成代理对象
  -> 后续调用的其实是“代理后的 target”
```

所以：

- `Interceptor` 不是主链里的业务对象
- 它是套在主链对象外层的代理增强
- 它拦的是这些对象的方法调用，而不是“拦截 SQL 字符串”本身

换句话说，`Interceptor` 和执行链的关系更像：

```text
                +----------------------+
                |   Interceptor Proxy  |
                +----------+-----------+
                           |
业务调用 -> SqlSession -> Executor -> StatementHandler -> ParameterHandler -> JDBC
                           |
                +----------v-----------+
                |   ResultSetHandler   |
                +----------------------+
```

如果再结合实际看，可以把它理解成：

- `Executor` 上可以套插件
- `StatementHandler` 上可以套插件
- `ParameterHandler` 上可以套插件
- `ResultSetHandler` 上可以套插件

但 `MapperProxy`、`MapperMethod`、`SqlSession` 默认不是官方插件机制直接开放的拦截点。

### 2.8 为什么 `StatementHandler` 和 `Executor` 最常被拿来改 SQL

如果目标是“拿到即将执行的 SQL”，最常见切点是两个：

- `StatementHandler`
- `Executor`

原因分别不同。

`StatementHandler` 更接近 JDBC，意味着：

- 你拿到的 SQL 更像“马上要执行的样子”
- 适合打印、观测、做轻量改写
- 拿 `BoundSql` 很方便

`Executor` 更靠上层，意味着：

- 你能更早参与执行逻辑
- 更容易成体系地重建 `BoundSql`
- 能同时覆盖 query/update 等行为
- 更容易处理缓存键、分页 count、查询包装等问题

这也是为什么：

- 很多“打印 SQL”的插件拦 `StatementHandler.prepare(...)`
- 很多“分页 / 数据权限 / 重写查询语义”的插件喜欢拦 `Executor.query(...)`

---

## 三、源码对象模型：`SqlSource`、`MappedStatement`、`BoundSql` 三者到底是什么关系

这是整篇笔记最核心的一章。

因为很多人学 MyBatis 插件时，脑子里只有“SQL 字符串”。但 MyBatis 真正运行时不是只传一个字符串，它至少在维护三层抽象：

- `SqlSource`
- `MappedStatement`
- `BoundSql`

如果把它们分别说成人话：

- `SqlSource`：**产出 SQL 的能力**
- `MappedStatement`：**某个 Mapper 语句的完整定义**
- `BoundSql`：**某次实际执行最终得到的 SQL 包**

### 3.1 `SqlSource`：负责“生产 SQL”

`SqlSource` 是一个很小但很关键的接口，核心方法只有一个：

```java
BoundSql getBoundSql(Object parameterObject);
```

这个定义已经把它的职责说得很清楚了：

- 输入：本次执行的参数对象
- 输出：本次执行对应的 `BoundSql`

所以 `SqlSource` 不是 SQL 本身，而是“根据参数生成本次 SQL 的生产器”。

常见实现可以粗略理解为：

- `DynamicSqlSource`：处理 `<if>`、`<foreach>`、`<trim>` 这类动态 SQL
- `RawSqlSource`：相对静态的 SQL，启动阶段就能完成大部分解析
- `StaticSqlSource`：更接近一个已经固定好的 SQL 模板
- `ProviderSqlSource`：来自 `@SelectProvider` / `@UpdateProvider` 这类 provider 方法

这里最重要的认知是：

> **在 MyBatis 里，“SQL 语句定义”不是直接以字符串存放，而是以 `SqlSource` 的形式被持有。**

也就是说，Mapper XML 或注解在解析完成以后，最终并不是简单地存一个 `String sql`，而是存成一个“将来可以根据参数生产 `BoundSql` 的对象”。

### 3.2 `MappedStatement`：负责“定义一个可执行语句”

`MappedStatement` 可以理解为 MyBatis 对“一个 Mapper 语句”的编译结果。

例如 `UserMapper.selectById` 在运行期不会只是一个方法名，它会对应一个 `MappedStatement`。这个对象里会放很多东西：

- `id`：语句唯一标识，一般是 `namespace + statementId`
- `sqlSource`：真正负责生产 `BoundSql` 的对象
- `sqlCommandType`：`SELECT` / `UPDATE` / `INSERT` / `DELETE`
- `parameterMap`
- `resultMaps`
- `statementType`
- `resultSetType`
- `cache`
- `useCache`
- `flushCacheRequired`
- `keyGenerator`
- `lang`

所以 `MappedStatement` 不是 SQL，也不是参数，它更像：

> **一条 Mapper 语句在 MyBatis 内部的“元数据总装对象”。**

插件里经常会用到它的原因也在这里。

你一旦拿到 `MappedStatement`，就能知道：

- 当前执行的是哪条语句
- 它属于哪个 Mapper
- 它是什么命令类型
- 它应该用什么 `SqlSource` 产出本次 SQL
- 它的结果映射和缓存策略是什么

### 3.3 `BoundSql`：负责“承载本次实际执行的 SQL 与参数映射”

`BoundSql` 是运行期对象，不是语句定义期对象。

这一点特别重要。

`MappedStatement` 更像“模板定义”，而 `BoundSql` 更像“这次真正执行时的实例化结果”。

`BoundSql` 里最关键的几个字段是：

- `sql`：最终 SQL 字符串，通常还是带 `?`
- `parameterMappings`：每个 `?` 对应的参数映射信息
- `parameterObject`：本次执行入参
- `additionalParameters`：动态 SQL 额外产生的参数，比如 `foreach`、`bind` 等

所以如果说：

- `SqlSource` 决定“怎么产出 SQL”
- `MappedStatement` 决定“这条语句整体怎么执行”

那么 `BoundSql` 决定的就是：

- **这一轮执行真正拿什么 SQL、按什么参数顺序去跑**

### 3.4 三者关系可以压缩成一句链路

最推荐记的一句链路是：

```text
MappedStatement 持有 SqlSource
SqlSource 根据 parameterObject 产出 BoundSql
BoundSql 被 StatementHandler / ParameterHandler / Executor 消费
```

如果把这句话展开，就是：

1. 启动期解析 Mapper，生成 `MappedStatement`
2. `MappedStatement` 内部持有 `SqlSource`
3. 运行期执行某个 Mapper 方法时，根据参数调用 `MappedStatement.getBoundSql(parameterObject)`
4. 其内部再委托给 `sqlSource.getBoundSql(parameterObject)`
5. 得到 `BoundSql`
6. 后续执行链围绕这份 `BoundSql` 展开

也就是说，运行期真正被消费的对象不是 XML，不是注解文本，而是 `BoundSql`。

### 3.5 为什么 `MappedStatement.getBoundSql(...)` 很关键

`MappedStatement` 有一个经常在插件里看到的方法：

```java
BoundSql boundSql = ms.getBoundSql(parameterObject);
```

这个方法重要的点不只是“拿 SQL”，而是它代表着：

- 从语句元数据层进入运行期 SQL 实例层
- 正式把动态 SQL、参数对象、额外参数、参数映射绑到一起

而且它还不只是简单转发给 `SqlSource`。

从源码逻辑上看，`MappedStatement.getBoundSql(...)` 还会在必要时兜底处理 `parameterMappings`，保证后续执行链看到的是一份尽量完整的 `BoundSql`。

所以很多插件如果拿到了 `MappedStatement` 和参数对象，会优先调用它，而不是自己凭空拼一份 SQL 字符串。

### 3.6 这三者的边界一旦搞清楚，很多插件原理就顺了

比如：

- 为什么分页插件经常不直接改 Mapper XML，而是在运行期改 `BoundSql`
- 为什么复杂改写常常需要复制 `MappedStatement`
- 为什么动态 SQL 的 `foreach` 参数必须小心 `additionalParameters`
- 为什么 `StatementHandler` 能拿到 `BoundSql`，而 `Executor` 往往会从 `MappedStatement` 出发再拿一遍

因为它们本来就在同一条对象链上。

---

## 四、先把 `Interceptor` 机制拆开：声明方式、拦截点、代理链、执行顺序

如果只停留在“实现一个 `Interceptor` 接口”，那离真正会用还有很远。

因为业务里真正要回答的是：

- 该拦 `Executor` 还是 `StatementHandler`
- 我拿到的 `invocation.getTarget()` 到底是谁
- 多个插件叠在一起时顺序怎么算
- 我的改写为什么有时生效、有时又像没生效

先看原生接口：

```java
public interface Interceptor {
    Object intercept(Invocation invocation) throws Throwable;

    default Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    default void setProperties(Properties properties) {
    }
}
```

这个接口本身不复杂，但它背后至少有四个必须搞懂的点。

### 4.1 `@Intercepts` 和 `@Signature` 到底在声明什么

一个最常见的声明方式如下：

```java
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
    )
})
public class SqlLogInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        return invocation.proceed();
    }
}
```

这里三部分含义非常明确：

- `type`：要代理哪一个核心接口
- `method`：要拦哪个方法
- `args`：方法签名参数必须精确匹配

也就是说，`@Signature` 不是“写个大概就行”，而是“告诉 MyBatis，只有这一类目标对象上的这一组方法调用才需要进入 `intercept(...)`”。

如果三者有一个不匹配，结果通常就是：

- 插件注册成功
- 但运行时根本进不了 `intercept(...)`

### 4.2 `invocation.getTarget()` 为何经常不是你以为的那个类

这是很多人第一次调试插件时最困惑的地方。

你明明写的是拦 `StatementHandler`，但 `target` 看起来像代理对象；你以为拿到的是 `BaseExecutor`，结果外面可能还包了 `CachingExecutor`。

原因就是插件机制本质上是代理链，而不是直接回调：

```text
plugin3(plugin2(plugin1(target)))
```

所以 `invocation.getTarget()` 拿到的经常是：

- 被上一层插件包过的对象
- 或者某个路由对象，而不是最底层具体实现

典型例子就是 `StatementHandler`。

业务里经常看到的写法：

```java
MetaObject metaObject = SystemMetaObject.forObject(statementHandler);
String sql = (String) metaObject.getValue("delegate.boundSql.sql");
```

这里的 `delegate` 之所以存在，就是因为真正常见的外层对象是 `RoutingStatementHandler`，内部再持有具体的 `PreparedStatementHandler`、`SimpleStatementHandler` 等实现。

### 4.3 再补一张关系图：四类可拦截对象不是并列概念，而是执行协作对象

如果还是容易把这四类对象看成“四个散点名词”，可以直接看下面这张图：

```text
MapperProxy
  -> MapperMethod
    -> SqlSession(DefaultSqlSession)
      -> Executor
           |-- query/update 总调度
           |-- 缓存、批处理、事务内执行协作
           |
           +--> StatementHandler
                 |-- prepare(Connection, timeout)
                 |-- parameterize(Statement)
                 |-- update/query(Statement)
                 |
                 +--> ParameterHandler
                 |     |-- setParameters(PreparedStatement)
                 |
                 +--> ResultSetHandler
                       |-- handleResultSets(Statement)
```

再叠加插件机制以后，真实运行时更接近：

```text
Proxy(Executor)
  -> Proxy(StatementHandler)
      -> Proxy(ParameterHandler)
      -> Proxy(ResultSetHandler)
```

这里要特别区分两件事：

- 第一张图是“职责协作关系”
- 第二张图是“插件代理关系”

很多人之所以会混乱，就是把这两张图混成一张看了。

### 4.4 四个拦截点分别更适合什么场景

MyBatis 允许插件拦的核心对象只有四类，但四类的用途差异很大。

`Executor` 更适合：

- 分页
- count SQL
- 数据权限重写
- 复杂查询包装
- 需要连缓存键一起考虑的扩展

`StatementHandler` 更适合：

- 打印待执行 SQL
- 动态表名
- hint 注入
- 轻量 where 条件追加
- 离 JDBC 最近的观测与改写

`ParameterHandler` 更适合：

- 参数审计
- 参数脱敏
- 某些参数补写
- 特定场景下定制绑定行为

`ResultSetHandler` 更适合：

- 结果脱敏
- 统一字典翻译
- 结果后处理

所以真正的第一步不是“我要写插件”，而是“我要解决的问题落在哪个边界上”。

### 4.5 这四类对象和 `Interceptor` 的关系，最后再压缩成一句话

如果只留一句最有用的话，我会写成：

> `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler` 是 MyBatis 运行期的核心协作接口；`Interceptor` 是对它们的方法调用做代理增强的插件机制，而不是与它们平级的另一条执行链。

把这句话吃透，再去看 `@Intercepts`、`@Signature`、`invocation.getTarget()`，就不会再把这些对象混成一团了。

### 4.6 插件顺序为什么会直接改变结果

只要你的插件不止做观测，而是会改 SQL，就必须认真看顺序。

例如系统里同时存在：

- 动态表名插件
- 数据权限插件
- 分页插件

更合理的顺序通常是：

1. 先换表名
2. 再补权限条件
3. 最后做分页包装和 count

如果反过来，可能出现：

- count SQL 条件不一致
- where 注入错层
- 已分页的 SQL 再被重写后不可解析

所以“插件顺序”不是文档里的注意事项，而是业务结果的一部分。

### 4.7 MyBatis-Plus 的 `InnerInterceptor` 其实只是上层编排

如果项目用了 MyBatis-Plus，经常会看到：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    interceptor.addInnerInterceptor(new TenantLineInnerInterceptor(...));
    return interceptor;
}
```

这套机制更像：

- `MybatisPlusInterceptor` 自己仍然是 MyBatis 插件
- 内部再维护一组 `InnerInterceptor`
- 再把查询前、更新前、prepare 前等时机转发下去

所以如果你先把原生 `Interceptor` 学明白，再看 `InnerInterceptor`，会发现它只是把常见扩展点模板化了，而不是另起炉灶。

---

## 五、先把几类典型插件放回业务场景：分页、数据权限、动态表名到底在改什么

如果只从“插件名字”理解这些能力，会很容易泛化。最好直接按业务问题去理解。

### 5.1 分页插件解决的不是 `limit`，而是“一次查询协议扩展”

业务上的真实需求不是“把 SQL 后面补个分页尾巴”，而是：

- 这次查询是分页查询
- 既要拿当前页数据
- 又要知道总条数
- 还要兼容不同数据库方言

所以分页插件通常至少要做三件事：

1. 把原始 SQL 改写为分页 SQL
2. 生成 count SQL
3. 把结果重新装回分页对象

这也是为什么分页插件更像站在 `Executor` 层思考，而不是只做一次简单文本替换。

### 5.2 数据权限插件解决的是“过滤语义统一注入”

业务语义通常是：

- 当前用户只能看自己部门
- 超管不受限制
- 某些 Mapper 不需要权限控制

这类需求如果散落在每个 XML 里，会非常容易漏；如果统一放插件里，本质就是：

- 判断当前语句是否需要权限
- 拿到当前用户的数据域
- 把过滤条件注入 SQL

所以它的本质不是“拼 where 字符串”，而是“把业务过滤语义统一下沉到 SQL 执行层”。

### 5.3 多租户插件是数据权限的强约束版本

普通数据权限可能按部门、角色、区域等规则变化；多租户更固定，往往围绕：

- `tenant_id`
- 哪些表需要租户列
- 哪些 SQL 需要忽略
- insert 时是否要自动补租户字段

所以多租户插件通常会同时覆盖：

- `SELECT`
- `UPDATE`
- `DELETE`
- `INSERT`

复杂度其实高于很多普通数据权限场景。

### 5.4 动态表名插件解决的是“路由到哪张真实表”

业务上的真实问题往往是：

- 订单表按月分表
- 日志表按租户分表
- 某些历史表按年份归档

所以它不是单纯换个字符串，而是在回答：

- 这次请求应该命中哪张物理表
- 同一条查询里的 count / update / delete 要不要跟着改
- 子查询和 join 中的表要不要一起改

只要 SQL 一复杂，这种插件也会迅速逼近 AST 级改写。

### 5.5 为什么成熟插件几乎都会碰到 SQL 解析器

因为下面这些结构，只靠正则和字符串拼接很容易翻车：

- join
- 子查询
- union
- distinct
- group by
- cte

所以成熟的分页、数据权限、多租户插件最终都会不同程度依赖 SQL AST 解析能力，例如 JSQLParser 这类工具。

---

## 六、Interceptor 实战一：先做最容易落地的 SQL 打印与审计插件

你最关心的问题之一，其实可以从最朴素的场景入手：

- 我怎么拿到“将要执行的 SQL”
- 我怎么把它打印出来
- 我怎么知道它对应的是哪个 Mapper 方法

这个场景最适合从 `StatementHandler.prepare(...)` 切入。

### 6.1 场景定义：统一打印待执行 SQL 和耗时

假设业务目标是：

- 所有业务 Mapper 的 SQL 都打印日志
- 打印 `MappedStatement id`
- 打印展示用 SQL
- 打印执行耗时

一个比较实用的写法如下：

```java
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
    )
})
public class SqlAuditInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = SystemMetaObject.forObject(statementHandler);

        MappedStatement ms =
            (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        BoundSql boundSql = (BoundSql) metaObject.getValue("delegate.boundSql");

        String msId = ms.getId();
        String rawSql = boundSql.getSql();
        String fullSql = renderSql(ms.getConfiguration(), boundSql);

        long start = System.currentTimeMillis();
        try {
            return invocation.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;
            log.info("msId={}, cost={}ms, rawSql={}, fullSql={}",
                msId, cost, rawSql, fullSql);
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    private String renderSql(Configuration configuration, BoundSql boundSql) {
        String sql = boundSql.getSql().replaceAll("\\s+", " ");
        List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
        Object parameterObject = boundSql.getParameterObject();

        if (parameterMappings == null || parameterMappings.isEmpty() || parameterObject == null) {
            return sql;
        }

        TypeHandlerRegistry registry = configuration.getTypeHandlerRegistry();
        if (registry.hasTypeHandler(parameterObject.getClass())) {
            return sql.replaceFirst("\\?", formatValue(parameterObject));
        }

        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        for (ParameterMapping pm : parameterMappings) {
            String property = pm.getProperty();
            Object value;
            if (boundSql.hasAdditionalParameter(property)) {
                value = boundSql.getAdditionalParameter(property);
            } else if (metaObject.hasGetter(property)) {
                value = metaObject.getValue(property);
            } else {
                value = null;
            }
            sql = sql.replaceFirst("\\?", formatValue(value));
        }
        return sql;
    }

    private String formatValue(Object value) {
        if (value == null) {
            return "null";
        }
        if (value instanceof Number || value instanceof Boolean) {
            return String.valueOf(value);
        }
        return "'" + String.valueOf(value).replace("'", "''") + "'";
    }
}
```

这段代码里最值得注意的不是日志，而是拿值路径：

- `delegate.mappedStatement`：拿到当前是哪个 Mapper 语句
- `delegate.boundSql`：拿到本次执行的 SQL 包
- `boundSql.getSql()`：拿预编译 SQL
- `parameterMappings + parameterObject + additionalParameters`：渲染展示 SQL

### 6.2 这个场景为什么适合 `StatementHandler.prepare(...)`

因为这里的诉求是：

- 看见最终要执行的 SQL
- 不需要重写复杂语义
- 也不需要引入额外查询

所以拦最接近 JDBC 的位置最自然。

### 6.3 这个场景在真实业务里最容易踩的坑

最常见的不是拿不到 SQL，而是：

- 日志里泄露敏感参数
- `foreach` 参数展示不完整
- SQL 太长把日志刷爆
- 某些系统 SQL 也被一起打印

所以真正落地时，建议至少补：

- `msId` 白名单 / 黑名单
- 慢 SQL 阈值
- 敏感字段脱敏
- 是否只在开发环境开启

---

## 七、Interceptor 实战二：业务代码如何真正“侵入 SQL”并实现租户 / 数据权限扩展

这部分才是最贴近你关心的问题的核心。

业务里真正常见的诉求不是“打日志”，而是：

- 从上下文拿到租户 ID
- 自动给 SQL 补 `tenant_id`
- 某些查询再补数据权限条件
- 改完 SQL 之后还能继续让 MyBatis 正确执行

这个问题最好拆成两种难度来做。

### 7.1 轻度侵入：`StatementHandler.prepare(...)` 阶段做租户条件注入

先看一个简化但很贴近业务的场景：

- 项目有 `TenantContext`
- 某些订单查询要自动补租户过滤
- 规则还比较固定，不需要很复杂的 AST 改写

```java
public final class TenantContext {
    private static final ThreadLocal<Long> HOLDER = new ThreadLocal<>();

    public static void set(Long tenantId) {
        HOLDER.set(tenantId);
    }

    public static Long get() {
        return HOLDER.get();
    }

    public static void clear() {
        HOLDER.remove();
    }
}
```

插件可以写成：

```java
@Intercepts({
    @Signature(
        type = StatementHandler.class,
        method = "prepare",
        args = {Connection.class, Integer.class}
    )
})
public class TenantSqlInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = SystemMetaObject.forObject(statementHandler);

        MappedStatement ms =
            (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        String msId = ms.getId();
        if (!needTenant(msId)) {
            return invocation.proceed();
        }

        Long tenantId = TenantContext.get();
        if (tenantId == null) {
            return invocation.proceed();
        }

        String originalSql = (String) metaObject.getValue("delegate.boundSql.sql");
        String newSql = appendTenantCondition(originalSql, tenantId);

        metaObject.setValue("delegate.boundSql.sql", newSql);
        return invocation.proceed();
    }

    private boolean needTenant(String msId) {
        return msId.startsWith("com.example.order")
            && !msId.endsWith("adminQuery");
    }

    private String appendTenantCondition(String sql, Long tenantId) {
        return "select * from (" + sql + ") t where t.tenant_id = " + tenantId;
    }
}
```

这段代码表达的是一种非常真实的业务姿势：

1. 先判断当前 `MappedStatement` 是否属于需要增强的范围
2. 再从业务上下文拿租户信息
3. 然后改 SQL
4. 最后继续执行

这已经是很多公司第一版多租户插件的雏形。

但它只是第一版，不是最终版。原因也很明显：

- 直接把值拼进 SQL，不够安全
- 包一层子查询虽然简单，但并不总是最佳语义
- 对 update/delete/insert 没有覆盖
- 遇到复杂 SQL 很容易越来越脆

所以轻度侵入适合快速落地，但不适合承载复杂规则。

### 7.2 重度侵入：在 `Executor` 层重建 `BoundSql` 和 `MappedStatement`

如果业务需求升级成：

- 数据权限不是固定 `tenant_id`
- 还要根据角色拼部门条件
- 需要增加新的 `?`
- 需要兼容 `foreach`
- 不能破坏后续参数绑定

那就不能只改 `boundSql.sql` 了。

更稳的姿势是：

1. 拿原始 `BoundSql`
2. 生成新的 SQL
3. 生成新的 `ParameterMapping`
4. 把额外参数复制过去
5. 必要时复制 `MappedStatement`

一个偏业务化的示意如下：

```java
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args = {
            MappedStatement.class,
            Object.class,
            RowBounds.class,
            ResultHandler.class
        }
    )
})
public class DataPermissionInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameterObject = args[1];

        if (!needPermission(ms.getId())) {
            return invocation.proceed();
        }

        BoundSql oldBoundSql = ms.getBoundSql(parameterObject);
        PermissionRule rule = PermissionContext.getRule();
        if (rule == null || rule.isAllData()) {
            return invocation.proceed();
        }

        RewriteResult result = rewriteWithPermission(ms, oldBoundSql, parameterObject, rule);
        if (!result.changed()) {
            return invocation.proceed();
        }

        BoundSql newBoundSql = new BoundSql(
            ms.getConfiguration(),
            result.getSql(),
            result.getParameterMappings(),
            result.getParameterObject()
        );
        copyAdditionalParameters(oldBoundSql, newBoundSql);

        MappedStatement newMs = copyMappedStatement(ms, new BoundSqlSqlSource(newBoundSql));
        args[0] = newMs;
        args[1] = result.getParameterObject();

        return invocation.proceed();
    }

    private RewriteResult rewriteWithPermission(
        MappedStatement ms,
        BoundSql oldBoundSql,
        Object parameterObject,
        PermissionRule rule
    ) {
        String oldSql = oldBoundSql.getSql();
        String newSql = "select * from (" + oldSql + ") src where src.dept_id in (?, ?)";

        List<ParameterMapping> newMappings = new ArrayList<>(oldBoundSql.getParameterMappings());
        Configuration configuration = ms.getConfiguration();
        newMappings.add(new ParameterMapping.Builder(configuration, "__permDeptId1", Long.class).build());
        newMappings.add(new ParameterMapping.Builder(configuration, "__permDeptId2", Long.class).build());

        Map<String, Object> newParameterObject = new HashMap<>();
        newParameterObject.put("_origin", parameterObject);
        newParameterObject.put("__permDeptId1", rule.getDeptIds().get(0));
        newParameterObject.put("__permDeptId2", rule.getDeptIds().get(1));

        return new RewriteResult(true, newSql, newMappings, newParameterObject);
    }

    private void copyAdditionalParameters(BoundSql oldBoundSql, BoundSql newBoundSql) {
        oldBoundSql.getAdditionalParameters().forEach(newBoundSql::setAdditionalParameter);
    }
}
```

这段代码真正体现了“业务侵入 SQL”的几个关键动作：

- 先用 `ms.getId()` 判断作用范围
- 用业务上下文拿权限规则
- 不只改 SQL，还同步改 `ParameterMapping`
- 通过新 `SqlSource` 包装新 `BoundSql`
- 让后续 `ParameterHandler` 看到的是一套新的执行输入

这已经不是“改字符串”，而是“重新组装一次执行协议”。

### 7.3 为什么这里经常还要复制 `MappedStatement`

因为你改的不只是 SQL 文本，而是“本次执行该用哪一份 SQL 输入”。

而 `MappedStatement` 自身还带着：

- `resultMaps`
- `statementType`
- `cache`
- `sqlCommandType`
- `parameterMap`

所以更合理的做法通常不是粗暴替换，而是：

- 复用原来的绝大部分元数据
- 只换掉用于生产 SQL 的那一层 `SqlSource`

这就是很多复杂插件里 `copyMappedStatement(...)` 存在的原因。

### 7.4 更硬核一点：`copyMappedStatement(...)` 到底该复制哪些字段

这一点非常值得单独展开。

因为很多文章里都会给一个 `copyMappedStatement(...)`，但经常只复制了几项最显眼的字段，然后告诉你“这样就能用了”。  
问题在于，**“能跑一次”不代表“在所有 Mapper 上都没坑”**。

先给一份更完整、实战里更稳妥的模板：

```java
private MappedStatement copyMappedStatement(MappedStatement ms, SqlSource newSqlSource) {
    MappedStatement.Builder builder = new MappedStatement.Builder(
        ms.getConfiguration(),
        ms.getId(),
        newSqlSource,
        ms.getSqlCommandType()
    );

    builder.resource(ms.getResource());
    builder.fetchSize(ms.getFetchSize());
    builder.timeout(ms.getTimeout());
    builder.statementType(ms.getStatementType());
    builder.resultSetType(ms.getResultSetType());
    builder.parameterMap(ms.getParameterMap());
    builder.resultMaps(ms.getResultMaps());
    builder.cache(ms.getCache());
    builder.flushCacheRequired(ms.isFlushCacheRequired());
    builder.useCache(ms.isUseCache());
    builder.resultOrdered(ms.isResultOrdered());
    builder.keyGenerator(ms.getKeyGenerator());

    if (ms.getKeyProperties() != null && ms.getKeyProperties().length > 0) {
        builder.keyProperty(String.join(",", ms.getKeyProperties()));
    }
    if (ms.getKeyColumns() != null && ms.getKeyColumns().length > 0) {
        builder.keyColumn(String.join(",", ms.getKeyColumns()));
    }
    if (ms.getDatabaseId() != null) {
        builder.databaseId(ms.getDatabaseId());
    }
    if (ms.getLang() != null) {
        builder.lang(ms.getLang());
    }
    if (ms.getResultSets() != null && ms.getResultSets().length > 0) {
        builder.resultSets(String.join(",", ms.getResultSets()));
    }

    return builder.build();
}
```

这份模板的核心思想只有一句话：

> **除了要换掉的 `SqlSource`，其余“描述这条语句怎么执行”的元数据，原则上都应该沿用原始 `MappedStatement`。**

也就是说，你不是在“创建一条全新的 Mapper 语句”，而是在“复制原语句的执行元数据，只替换本次执行要产出的 SQL”。

### 7.5 字段清单一：最基本、几乎一定要复制的字段

下面这些字段，如果你在重建 `MappedStatement` 时漏掉，最容易直接出问题。

1. `configuration`
2. `id`
3. `sqlCommandType`
4. `statementType`
5. `parameterMap`
6. `resultMaps`
7. `resultSetType`
8. `timeout`
9. `fetchSize`

为什么它们重要：

- `configuration`：整个 MyBatis 运行环境都挂在这里，不对就根本不是同一个世界
- `id`：语句身份标识，不保留原值会影响日志、拦截匹配、缓存语义和定位
- `sqlCommandType`：决定当前语句是 `SELECT`、`UPDATE` 还是 `INSERT`
- `statementType`：决定是不是 `PREPARED`、`STATEMENT`、`CALLABLE`
- `parameterMap`：参数映射定义仍然可能被下游逻辑依赖
- `resultMaps`：结果映射一旦丢失，查询结果很容易直接映射错乱
- `resultSetType`：某些数据库驱动和游标行为会依赖这个配置
- `timeout` / `fetchSize`：虽然不是语义正确性的核心，但属于语句执行特征，不复制会出现行为偏差

如果这些字段漏了，典型表现一般是：

- 查询结果变成 `Map` / `Object` 风格异常映射
- 存储过程、游标、Callable 场景直接失效
- 插件日志里 `msId` 不对，排查困难
- 参数绑定行为和原语句不一致

### 7.6 字段清单二：最容易被忽略，但漏掉后会出现“诡异问题”的字段

真正难的是下面这些字段。它们不像 `resultMaps` 那样一漏就爆，而是容易在某些场景下悄悄出错。

1. `cache`
2. `useCache`
3. `flushCacheRequired`
4. `resultOrdered`
5. `keyGenerator`
6. `keyProperties`
7. `keyColumns`
8. `databaseId`
9. `lang`
10. `resultSets`

这些字段的意义分别是：

- `cache`：绑定二级缓存实例
- `useCache`：当前查询是否参与缓存
- `flushCacheRequired`：当前语句执行后是否需要清缓存
- `resultOrdered`：嵌套结果映射顺序相关配置
- `keyGenerator`：插入语句主键回填策略
- `keyProperties` / `keyColumns`：主键回填字段映射
- `databaseId`：数据库方言/厂商区分
- `lang`：脚本语言驱动，例如 XMLLanguageDriver
- `resultSets`：多结果集场景配置

这类字段漏掉后的问题往往很“业务化”，排查起来也最痛苦。

比如：

- `cache` / `useCache` / `flushCacheRequired` 漏掉：查询缓存命中率异常，或者 update 执行后缓存没有按预期失效
- `keyGenerator` / `keyProperties` 漏掉：`insert` 成功了，但主键没回填到实体对象上
- `databaseId` 漏掉：多数据库兼容场景下命中了错误 SQL 分支
- `lang` 漏掉：某些自定义语言驱动或脚本解析行为和原语句不一致
- `resultSets` 漏掉：存储过程、多结果集查询出现解析异常

所以从工程经验上说，**漏掉这些字段的可怕之处在于：代码不一定立刻报错，但行为会悄悄偏离原始语句**。

### 7.7 哪些字段通常不用你手动复制，但要知道它们为什么存在

还有一些字段你在 `Builder` 上不一定都能直接设置，或者一般业务插件不太会碰到，但它们在 `MappedStatement` 里依然存在。

比如：

- `hasNestedResultMaps`
- `dirtySelect`
- `statementLog`

对普通 SQL 改写插件来说，一般不需要为了这些字段自己额外构造复杂逻辑，因为：

- 它们很多会由 `resultMaps`、`id`、`configuration` 等间接推导
- 或属于构建期衍生状态，不是常规插件直接操控的点

但你要知道：  
如果你的插件已经复杂到需要完全“伪造一条新语句”，那就不能再满足于普通的 copy 模板了，而要回到 `MappedStatement.Builder` 的完整构造语义去逐项校对。

### 7.8 一个实用判断：查询语句和插入语句，漏字段后的表现完全不同

这个经验很重要。

如果是 `SELECT` 场景，漏字段后最常见的是：

- 结果映射错
- 缓存行为怪
- 多结果集异常

如果是 `INSERT` 场景，漏字段后最常见的是：

- 主键回填失败
- `useGeneratedKeys` 相关行为丢失
- 返回对象里 `id` 还是空

如果是 `UPDATE` / `DELETE` 场景，漏字段后更容易表现为：

- 缓存没有及时刷新
- 某些执行配置和原语句不一致

也就是说，同一段不完整的 `copyMappedStatement(...)`，在不同命令类型上暴露的问题可能完全不一样。

### 7.9 为什么我更推荐“保守复制”，而不是“只复制我现在觉得会用到的字段”

因为插件代码往往一开始只服务一个场景，但后面会逐渐扩散：

- 一开始只拦 select
- 后来要支持 update
- 再后来业务说 insert 也想复用
- 再后来某条 Mapper 恰好用了主键回填、二级缓存、多结果集

如果最开始你写的是“极简 copy”，后面很容易进入补锅模式。

所以更稳妥的思路是：

- 一开始就按“保留原语句执行语义”的原则尽量完整复制
- 只把 `SqlSource` 当成真正要替换的变量

这样后面扩展场景时，你踩的坑会少很多。

### 7.10 最后给一个排查口诀

如果你在做 SQL 改写后出现下面这些现象：

- SQL 明明执行了，但结果映射怪异
- insert 成功了，但主键没回填
- update 之后缓存没刷新
- 某些特定 Mapper 才会莫名失效

第一时间就该怀疑两件事：

1. 新 `BoundSql` 的参数体系有没有构造完整
2. 新 `MappedStatement` 的元数据有没有复制完整

很多“看起来像 MyBatis 黑魔法”的问题，最后根源都只是：  
`copyMappedStatement(...)` 漏复制了本该沿用的字段。

### 7.11 再回到本质：`copyMappedStatement(...)` 不是辅助函数，而是“保留原执行语义”的关键动作

现在再回看这个函数，它就不是一个样板工具方法了，而是复杂插件里很关键的一步：

- `BoundSql` 负责替换“这次执行的 SQL 输入”
- `copyMappedStatement(...)` 负责保留“这条语句原本应该具备的执行语义”

两者缺一不可。

所以真正成熟的复杂拦截器，通常都不是只改一个 `sql` 字符串，而是同时维护：

- SQL 本身
- 参数映射
- 动态附加参数
- `MappedStatement` 元数据

这样重组后的执行链，才更接近原生 MyBatis 的行为预期。

### 7.12 `BoundSqlSqlSource` 这种包装写法到底安不安全，边界在哪

很多示例代码在重写 SQL 时都会顺手写一个类：

```java
static class BoundSqlSqlSource implements SqlSource {
    private final BoundSql boundSql;

    BoundSqlSqlSource(BoundSql boundSql) {
        this.boundSql = boundSql;
    }

    @Override
    public BoundSql getBoundSql(Object parameterObject) {
        return boundSql;
    }
}
```

这段代码看起来很“取巧”，所以很多人会下意识问一句：  
它到底安不安全？

我的结论是：

> **它不是绝对不安全，但它只适合“单次调用、当前 invocation 内部的临时包装”，不适合被当成可复用、可缓存、可跨调用共享的通用 `SqlSource`。**

也就是说，它的安全前提不是这个类本身有多高级，而是你怎么使用它。

### 7.13 为什么它在很多插件里“看起来能用”

因为在最典型的使用姿势里，流程是这样的：

1. 当前一次 `query` / `update` 进入插件
2. 你基于本次参数生成一个新的 `BoundSql`
3. 你创建一个新的 `BoundSqlSqlSource`
4. 你基于它创建一个新的 `MappedStatement`
5. 这个新 `MappedStatement` 只服务当前这一次调用

这个流程下，`BoundSqlSqlSource` 的语义其实非常清楚：

- 我不是一个“通用 SQL 生产器”
- 我只是当前 invocation 内部的“固定 `BoundSql` 搬运器”

在这个语境里，`getBoundSql(parameterObject)` 虽然忽略了传入参数，但仍然是合理的，因为：

- 这份 `BoundSql` 本来就是针对此次 `parameterObject` 预先构造好的
- 这个 `MappedStatement` 也只会在这一次调用中被消费

所以如果你把它看成“当前调用栈里的临时适配器”，它是成立的。

### 7.14 它真正不安全的地方，不在类本身，而在“复用”

只要你开始做下面这些事，风险就会迅速升高：

- 把这个 `BoundSqlSqlSource` 缓存起来
- 把 new 出来的 `MappedStatement` 放到全局 map 里复用
- 试图让同一个 `BoundSqlSqlSource` 服务多次调用
- 让不同参数对象共用同一份 `BoundSql`

为什么？

因为 `BoundSql` 本质上就是一次调用的运行期对象，它里面绑定了：

- 当前 SQL
- 当前 `parameterMappings`
- 当前 `parameterObject`
- 当前 `additionalParameters`

这些东西天然就是 invocation 级别的。

一旦你把它拿去跨调用复用，就会出现：

- A 请求的参数对象被 B 请求复用了
- 上一次动态 SQL 产生的 `additionalParameters` 污染下一次调用
- `foreach` 相关绑定变量串调用
- 同一条缓存结构里混入不同参数下的运行期状态

所以 `BoundSqlSqlSource` 的真正边界可以压缩成一句话：

> **可以“临时包装”，不要“长期持有”。**

### 7.15 哪些场景下它是相对安全的

下面这些场景，通常是它比较安全、也比较实用的使用方式：

1. 在一次 `Executor.query(...)` 拦截里临时生成新 `MappedStatement`
2. 当前插件只想让后续链路消费“这一次已经重写好的 `BoundSql`”
3. 新 `MappedStatement` 不会注册回全局 `Configuration`
4. 当前逻辑不打算复用这份 `MappedStatement`

这类场景下，它的最大优点就是简单：

- 不用再写一个真正依赖 `parameterObject` 动态生成 SQL 的新 `SqlSource`
- 也不用把“本次已经构造好的 `BoundSql`”再拆回模板再生产一次

换句话说，它是一个**工程上很实用的 invocation 级适配器**。

### 7.16 哪些场景下它不够安全，甚至不应该用

如果你面对的是下面这些需求，我不建议继续用 `BoundSqlSqlSource`：

1. 你打算缓存新生成的 `MappedStatement`
2. 你希望一个 `MappedStatement` 能服务不同参数对象的多次调用
3. 你希望 `SqlSource.getBoundSql(parameterObject)` 在每次调用时重新基于参数计算 SQL
4. 你做的是更长期、可复用的语句改写框架，而不是当前插件中的一次临时替换

这时更合理的做法通常是：

- 自己实现一个真正依赖参数对象的 `SqlSource`
- 或者保留原始 `SqlSource`，只在必要时局部增强生成逻辑

因为一旦你的需求从“当前调用临时改写”升级为“可复用 SQL 生产逻辑”，`BoundSqlSqlSource` 就显得过于静态了。

### 7.17 它最容易造成误解的一点：`getBoundSql(parameterObject)` 明明有参，却完全不看参数

这恰恰是它最大的边界提示。

标准 `SqlSource` 的直觉语义应该是：

- 给我参数对象
- 我给你这个参数下对应的 `BoundSql`

而 `BoundSqlSqlSource` 实际上变成了：

- 不管你给我什么参数
- 我都返回这份预先固定好的 `BoundSql`

所以它本质上已经不是“生产器”，而是“返回常量结果的壳”。

如果你理解了这一点，就能知道为什么它只能适合当前调用局部使用。

### 7.18 `BoundSqlSqlSource + copyMappedStatement + additionalParameters` 一起用时，最核心的注意事项

这是最值得单独记的一段。

这三者组合起来时，真正要维护的是三套一致性：

1. `new BoundSql` 的 SQL 与 `parameterMappings` 一致
2. `new BoundSql` 的 `parameterObject` 与 `invocation.getArgs()` 一致
3. `new BoundSql` 的 `additionalParameters` 与动态 SQL 运行期上下文一致

只要这三套关系里有一套错了，后面就会出现非常诡异的问题。

最常见的组合错误有这些。

第一类：SQL 改了，参数映射没改。  
表现：

- 新增 `?` 后参数绑定报错
- 条件顺序变了，但值绑定错位

第二类：`parameterObject` 改了，但 `args[1]` 没同步。  
表现：

- 你以为新参数生效了，实际后续链路仍在用旧参数对象
- 某些插件或缓存键仍按旧参数计算

第三类：`additionalParameters` 没复制。  
表现：

- `foreach` 场景突然报找不到参数
- `<bind>` 生成的变量失效
- 动态 SQL 在简单查询能跑，在复杂集合参数场景炸掉

第四类：new 出来的 `MappedStatement` 被错误复用。  
表现：

- 上一次请求的 `BoundSql` 污染下一次请求
- 并发下出现偶发脏数据式 SQL 行为

### 7.19 一个更稳妥的组合模板

如果你决定继续用这套组合，我更推荐按照下面的顺序做：

```java
BoundSql oldBoundSql = ms.getBoundSql(parameterObject);

RewriteResult result = rewrite(oldBoundSql, parameterObject);

BoundSql newBoundSql = new BoundSql(
    ms.getConfiguration(),
    result.getSql(),
    result.getParameterMappings(),
    result.getParameterObject()
);

oldBoundSql.getAdditionalParameters().forEach(newBoundSql::setAdditionalParameter);

MappedStatement newMs = copyMappedStatement(ms, new BoundSqlSqlSource(newBoundSql));

args[0] = newMs;
args[1] = result.getParameterObject();
```

这里最关键的不是顺序形式，而是这三个动作必须成对出现：

- 改 SQL，就同步改 `parameterMappings`
- 改参数对象，就同步改 `args[1]`
- 新建 `BoundSql`，就同步复制 `additionalParameters`

如果这三对关系没同时维护，代码往往“看起来很像对的”，但运行起来就会有边角问题。

### 7.20 再更硬核一点：`MappedStatement.Builder` 到底有哪些可设置字段，源码默认值是什么

这部分可以直接结合官方源码来看。

`MappedStatement.Builder` 构造器里，源码已经先给一部分字段设置了默认值：

```java
mappedStatement.configuration = configuration;
mappedStatement.id = id;
mappedStatement.sqlSource = sqlSource;
mappedStatement.statementType = StatementType.PREPARED;
mappedStatement.resultSetType = ResultSetType.DEFAULT;
mappedStatement.parameterMap = new ParameterMap.Builder(configuration, "defaultParameterMap", null,
    new ArrayList<>()).build();
mappedStatement.resultMaps = new ArrayList<>();
mappedStatement.sqlCommandType = sqlCommandType;
mappedStatement.keyGenerator =
    configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType)
        ? Jdbc3KeyGenerator.INSTANCE
        : NoKeyGenerator.INSTANCE;
mappedStatement.statementLog = LogFactory.getLog(logId);
mappedStatement.lang = configuration.getDefaultScriptingLanguageInstance();
```

把它翻译成人话，大概就是下面这张表。

| 字段 | 默认值 / 默认行为 | 来源 |
| --- | --- | --- |
| `configuration` | 必填，无默认兜底空值 | 构造器参数 |
| `id` | 必填 | 构造器参数 |
| `sqlSource` | 必填 | 构造器参数 |
| `sqlCommandType` | 必填 | 构造器参数 |
| `statementType` | `PREPARED` | 构造器默认 |
| `resultSetType` | `DEFAULT` | 构造器默认 |
| `parameterMap` | 一个空的默认 `ParameterMap` | 构造器默认 |
| `resultMaps` | 空 `ArrayList` | 构造器默认 |
| `keyGenerator` | `INSERT + useGeneratedKeys` 时为 `Jdbc3KeyGenerator`，否则 `NoKeyGenerator` | 构造器推导 |
| `statementLog` | 基于 `logPrefix + id` 生成 | 构造器推导 |
| `lang` | `configuration.getDefaultScriptingLanguageInstance()` | 构造器默认 |
| 其余布尔 / 可选字段 | 基本保持 Java 默认值，如 `false`、`null` | 未显式设置时 |

这里可以把字段再分成三类理解。

### 7.21 第一类：构造器强制给定的字段

这类字段没有“合理缺省”，本质上是 `MappedStatement` 的身份骨架：

- `configuration`
- `id`
- `sqlSource`
- `sqlCommandType`

如果这些不对，这条语句从根上就不是原来的语句了。

### 7.22 第二类：构造器自动填的默认值

这类字段哪怕你不手动设置，Builder 也会给一个默认值：

- `statementType = PREPARED`
- `resultSetType = DEFAULT`
- `parameterMap = 默认空 ParameterMap`
- `resultMaps = 空列表`
- `lang = 默认 LanguageDriver`
- `keyGenerator = 根据 INSERT + useGeneratedKeys 自动推导`
- `statementLog = 根据 logPrefix/id 自动推导`

这也是为什么有些极简 copy 模板“勉强也能跑”。

因为即使你什么都不补，Builder 也不会真的留空，而是给你一套默认配置。

但这里的关键问题是：

> **Builder 给的默认值，不一定等于原语句的真实值。**

这也是很多 bug 的来源。

比如：

- 原语句的 `statementType` 可能不是 `PREPARED`
- 原语句有明确的 `resultMaps`
- 原语句用了自定义 `lang`
- 原语句的 `keyGenerator` 不是默认推导出来的那套

这时如果你只依赖默认值，而不是复制原值，行为就会漂移。

### 7.23 第三类：需要你在 copy 时主动保留的字段

从插件作者视角，这类字段最值得重视，因为它们往往不会被 Builder 自动推导回“原语句的真实配置”。

通常包括：

- `resource`
- `fetchSize`
- `timeout`
- `statementType`
- `resultSetType`
- `parameterMap`
- `resultMaps`
- `cache`
- `flushCacheRequired`
- `useCache`
- `resultOrdered`
- `keyGenerator`
- `keyProperties`
- `keyColumns`
- `databaseId`
- `lang`
- `resultSets`

这类字段的原则很简单：

- 如果原语句显式配置过，copy 时就尽量保留
- 不要指望 Builder 默认值“刚好等于原语句”

### 7.24 哪些字段属于“构建期自动推导”，你要知道但不一定手动操控

结合源码，有几项是非常典型的“衍生字段”：

- `hasNestedResultMaps`
- `statementLog`

其中：

- `hasNestedResultMaps` 会在 `builder.resultMaps(...)` 时根据 `ResultMap` 自动累积推导
- `statementLog` 会在构造器里基于 `id` 和 `logPrefix` 自动生成

这意味着：

- 你通常不需要手工给这两个字段赋值
- 但你必须保证它们赖以推导的上游字段是正确的

例如：

- `resultMaps` 不对，`hasNestedResultMaps` 也会跟着不对
- `id` 不对，`statementLog` 也会跟着偏掉

### 7.25 还有一个很容易漏掉的字段：`dirtySelect`

从源码看，Builder 还提供了：

```java
builder.dirtySelect(boolean dirtySelect)
```

这说明 `dirtySelect` 也是一个可配置字段。

它不像 `resultMaps`、`cache` 那样高频，但既然它存在，就说明在某些执行语义里可能有特殊意义。

问题在于，很多 copy 模板根本不会管它。

这件事给我们的启发不是“现在立刻所有插件都要去复制 `dirtySelect`”，而是：

- 如果你写的是一般业务 SQL 改写插件，通常先把高频核心字段复制完整
- 如果你写的是框架级、通用级语句重写能力，就要回到具体 MyBatis 版本源码，把 Builder 可设置项逐个校对

因为越通用的框架，越不能靠“这几个字段大概够了”的心态。

### 7.26 把这一整段收束成一句工程结论

如果只留一句最有用的话，我会写成：

> `BoundSqlSqlSource` 可以作为当前 invocation 内的临时适配器；`copyMappedStatement(...)` 的本质是保留原语句执行语义；`additionalParameters` 则是动态 SQL 场景下绝不能漏的运行期状态。三者组合时，必须同时维护 SQL、参数、元数据三套一致性。

这句话基本概括了复杂 SQL 改写里最容易翻车的核心问题。

### 7.27 真实业务里这类插件最容易出问题的地方

如果把经验压缩成几条，最容易翻车的就是：

- 新 SQL 新增了 `?`，但没补 `ParameterMapping`
- `foreach` 生成的 `additionalParameters` 没复制
- count SQL 和主查询 SQL 改写规则不一致
- 某些 Mapper 不该拦却被拦了
- 日志里打印的 full SQL 和真实执行协议混淆了

所以真正成熟的实现，一定是：

- 有匹配范围
- 有忽略策略
- 有参数同步
- 有单测覆盖复杂 SQL

---

## 八、纯源码专题：`DynamicSqlSource`、`RawSqlSource`、`StaticSqlSource` 是怎么生成出来并参与调用的

前面讲了 `SqlSource` 的角色，但如果不继续往下看，很容易停留在“知道有这些类”。这一章专门回答两个问题：

1. 它们分别是在什么时候生成的
2. 运行期调用链到底怎么走

### 8.1 先说结论：这三个类分别对应三种不同的 SQL 形态

可以先粗暴记成下面这样：

- `DynamicSqlSource`：运行期还要根据参数继续拼装 SQL
- `RawSqlSource`：启动期就能把大部分 SQL 结构定下来
- `StaticSqlSource`：已经是最终静态 SQL 模板，拿来直接产出 `BoundSql`

其中最关键的关系其实是：

- `RawSqlSource` 内部最终也会落到 `StaticSqlSource`
- `DynamicSqlSource` 在运行期求值后，也会临时把结果交给 `SqlSourceBuilder` 生成一个 `StaticSqlSource`

也就是说，`StaticSqlSource` 很像“最终静态执行模板”。

### 8.2 XML Mapper 是怎么决定生成 `DynamicSqlSource` 还是 `RawSqlSource` 的

MyBatis 解析 XML SQL 时，核心会经过 `XMLLanguageDriver` 和 `XMLScriptBuilder`。

可以把流程粗略理解为：

```text
Mapper XML / <script>
  -> XMLLanguageDriver
  -> XMLScriptBuilder.parseScriptNode()
  -> 先把 XML 节点解析成 SqlNode 树
  -> 判断这棵树是不是动态 SQL
  -> 动态：DynamicSqlSource
  -> 非动态：RawSqlSource
```

这里所谓“是不是动态 SQL”，典型判断依据包括：

- 有没有 `<if>`
- 有没有 `<foreach>`
- 有没有 `<choose>`
- 有没有 `${}`
- 有没有 `<trim>` / `<where>` / `<set>` 等动态节点

如果有这些，通常就是 `DynamicSqlSource`。

如果没有，通常就是 `RawSqlSource`。

### 8.3 `DynamicSqlSource` 的生成与调用链

先看它的创建时机：

- XML 中存在动态标签
- `XMLScriptBuilder` 把节点树解析成 `MixedSqlNode`
- 再把根节点包装成 `DynamicSqlSource`

它更像“持有一棵动态 SQL 语法树的对象”。

运行期调用时，大致链路可以理解为：

```text
MappedStatement.getBoundSql(parameterObject)
  -> DynamicSqlSource.getBoundSql(parameterObject)
  -> 创建 DynamicContext
  -> rootSqlNode.apply(context)
  -> 得到拼好的 SQL 字符串
  -> SqlSourceBuilder.parse(...)
  -> 生成临时 StaticSqlSource
  -> StaticSqlSource.getBoundSql(parameterObject)
  -> 把动态绑定值放进 additionalParameters
```

这里最关键的点有两个：

第一，`DynamicSqlSource` 自己并不直接维护最终 `parameterMappings`。  
它先把动态节点求值成一段完整 SQL，再交给 `SqlSourceBuilder` 去解析 `#{}` 占位符。

第二，动态 SQL 运行期产生的绑定值会进入 `additionalParameters`。  
这也是为什么你在插件里重建 `BoundSql` 时，经常要把这些参数复制过去。

### 8.4 `RawSqlSource` 的生成与调用链

`RawSqlSource` 产生的前提是：

- SQL 结构本身不是动态的
- 启动阶段就可以完成 `#{}` 解析

它的创建可以粗略理解为：

```text
静态 XML SQL
  -> XMLScriptBuilder.parseScriptNode()
  -> 判定为非动态
  -> new RawSqlSource(...)
  -> 内部通过 SqlSourceBuilder.parse(...)
  -> 生成 StaticSqlSource
```

所以 `RawSqlSource` 更像一个“启动期中间态”。

它的价值是：

- 把原本还带 `#{}` 的文本
- 在启动阶段就解析成静态 SQL 模板和参数映射

运行期再调用时，实际上已经不会再做复杂动态求值，而是直接走内部持有的 `StaticSqlSource`。

所以可以把它理解成：

> `RawSqlSource` = 启动期预编译后的 `StaticSqlSource` 包装器

### 8.5 `StaticSqlSource` 的角色：最接近“最终执行模板”

`StaticSqlSource` 的职责最纯粹：

- 持有一份静态 SQL
- 持有对应的参数映射
- 在运行期根据参数对象直接产出 `BoundSql`

它不关心：

- `<if>` 命不命中
- `<foreach>` 展不展开
- OGNL 表达式怎么求值

这些事在它之前都已经解决了。

所以从职责边界上说：

- `DynamicSqlSource` 负责“运行期求值”
- `RawSqlSource` 负责“启动期预解析”
- `StaticSqlSource` 负责“最终静态产出 `BoundSql`”

### 8.6 一个完整视角：三者并不是并列替代，而是有上下游关系

很多文章会把这三个类横着列出来，但更准确的理解应该是有链路关系：

```text
动态 SQL:
  DynamicSqlSource
    -> 运行期求值
    -> 临时转成 StaticSqlSource
    -> 产出 BoundSql

静态 SQL:
  RawSqlSource
    -> 启动期预解析
    -> 内部持有 StaticSqlSource
    -> 运行期产出 BoundSql
```

所以最终真正最接近执行模板的，始终是 `StaticSqlSource`。

### 8.7 为什么这个源码专题对写插件很重要

因为它直接解释了三个高频问题：

第一，为什么动态 SQL 下插件更容易踩 `additionalParameters` 的坑。  
因为这些值是运行期动态求值塞进去的，不是原始参数对象自带的。

第二，为什么有些 SQL 在启动期就已经“基本定型”，而有些要到运行期才定型。  
因为前者走 `RawSqlSource`，后者走 `DynamicSqlSource`。

第三，为什么很多插件最终只关心 `BoundSql`。  
因为不管上游是 `DynamicSqlSource` 还是 `RawSqlSource`，真正落到执行链里被消费的，都是 `BoundSql`。

---

## 九、工程判断与常见陷阱：什么时候该用插件，什么时候不该用插件

源码理解完以后，最后还是要回到工程判断。

### 9.1 插件适合做“横切规则”，不适合做“零散业务 if-else”

插件最适合的场景通常有这些共性：

- 对多个 Mapper 生效
- 规则相对统一
- 希望对业务代码无侵入
- 离数据库执行很近才方便处理

例如：

- 分页
- 多租户
- 数据权限
- 动态表名
- SQL 审计
- 慢 SQL 统计

而插件不太适合的场景通常是：

- 只服务某一个业务接口
- 高度依赖业务上下文且经常变化
- 需要调用很多外部服务才能决定 SQL 怎么改
- 逻辑本身已经不像基础设施，而像业务编排

这类需求硬塞进插件里，结果通常是：

- 作用域不透明
- 排障困难
- 可测试性很差
- 后续没人敢动

### 9.2 不要把“改 SQL 字符串”当成“已经做完插件”

很多初学插件的代码只做了这一步：

```java
metaObject.setValue("delegate.boundSql.sql", newSql);
```

这当然有用，但只能说明你“改到了 SQL 字符串”，不能说明你“正确完成了一次 SQL 执行改写”。

真正还要继续问的是：

- 新 SQL 有没有新增占位符
- 参数映射是否同步修改
- `additionalParameters` 有没有丢
- 查询缓存键是否一致
- count SQL 是否一致
- join / 子查询 / union 是否还能保持语义

只改字符串，很多时候只是完成了第一步，不是完成了整件事。

### 9.3 动态 SQL 场景要特别警惕 `additionalParameters`

`<foreach>`、`<bind>` 之类的动态 SQL 场景，是最容易让人误以为“SQL 已经改好了，怎么运行还报错”的地方。

原因通常不是 SQL 本身，而是：

- 你新建了 `BoundSql`
- 但没把动态附加参数带过去

所以凡是重建 `BoundSql`，都应该默认检查：

- `parameterMappings`
- `additionalParameters`

是不是都被完整复制了。

### 9.4 能用 AST 的地方，尽量别用正则硬拼

一旦碰到以下结构：

- 子查询
- 多表 join
- union / union all
- with / cte
- group by / distinct
- 数据库方言函数

字符串替换的风险会迅速上升。

所以：

- 简单场景：字符串替换可以接受
- 复杂场景：优先 SQL AST 解析

这也是为什么成熟分页插件、租户插件、数据权限插件都会越来越依赖 SQL 解析器。

### 9.5 任何 SQL 改写插件都要认真设计“忽略策略”

这是很多实现文章不爱写，但工程上非常关键的一点。

因为没有哪个插件应该“默认改所有 SQL”。

至少要有这些能力中的一部分：

- 根据 `MappedStatement.getId()` 精准匹配
- 根据表名白名单 / 黑名单控制
- 根据注解显式忽略某个插件
- 根据线程上下文临时关闭插件
- 对系统语句、运维语句、初始化语句进行排除

否则你迟早会遇到：

- 系统后台查询也被权限过滤了
- count SQL 被重复改写
- 某个初始化脚本被动态表名替换了

### 9.6 一个实用的方案选择建议

如果是下面这些目标，可以这样选。

只想观测 SQL：

- 优先 `StatementHandler.prepare(...)`
- 关注 `BoundSql.getSql()`、`parameterMappings`
- 展示 SQL 只用于日志

想做统一轻改写：

- 可从 `StatementHandler` 入手
- 动态表名、hint 注入、小范围条件追加通常够用

想做复杂语义扩展：

- 优先从 `Executor` 视角设计
- 必要时重建 `BoundSql`
- 再重一点就复制 `MappedStatement`

项目本身是 MyBatis-Plus：

- 优先考虑 `InnerInterceptor`
- 但脑子里仍然要按 MyBatis 原生对象模型思考

### 9.7 我现在对这套机制的最终理解

如果现在再回到最初那个问题：

> 业务方如何拿到将要执行的 SQL，并对 SQL 做侵入修改、重新组装、实现业务扩展？

我现在的回答会比一开始更完整：

- 运行期真正承载“本次执行 SQL”的核心对象是 `BoundSql`
- `BoundSql` 又来自 `MappedStatement` 持有的 `SqlSource`
- 所有分页、数据权限、多租户、动态表名插件，本质上都是在这条对象链上选择一个切入点进行扩展
- 轻量需求可以在 `StatementHandler` 阶段改 `BoundSql`
- 重组型需求应该在 `Executor` 视角下把 `BoundSql`、参数映射、必要时连 `MappedStatement` 一起重建
- MyBatis-Plus 不是另一套完全不同的世界，只是把 MyBatis 这套执行扩展模型进一步工程化了

---

## 十、收束总结

这篇笔记如果只保留最关键的几句话，我会留下下面这些：

- MyBatis 插件研究的核心，不是记住 `@Intercepts`，而是看清运行期对象模型
- `SqlSource` 负责生产 SQL，`MappedStatement` 负责定义语句，`BoundSql` 负责承载本次执行实例
- 真正的 SQL 改写，不止是改字符串，更是重建“SQL + 参数映射 + 执行语义”的一致性
- 分页插件的本质是查询协议扩展，数据权限插件的本质是过滤语义注入，动态表名插件的本质是表级路由改写
- MyBatis-Plus 的 `InnerInterceptor` 是上层封装，底层仍然建立在 MyBatis 原生插件链之上
- 复杂 SQL 改写越往后走，越要依赖 AST 解析、忽略策略、参数同步与缓存一致性设计

如果后面继续深挖，我觉得最值得继续补的两个方向是：

- 结合具体源码，专门拆一篇 `DynamicSqlSource`、`RawSqlSource`、`StaticSqlSource` 的生成与调用链
- 以“数据权限插件”为例，完整走一遍 AST 改写、`BoundSql` 重建、插件忽略策略、测试设计
