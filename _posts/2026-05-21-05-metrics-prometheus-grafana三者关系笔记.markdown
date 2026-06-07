---
layout:      post
title:       "Metrics、Prometheus、Grafana 三者关系笔记"
subtitle:    "这三者如何区分、职责分别是什么、为什么总一起出现但又绝不是一回事"
author:      "Ekko"
header-img:  "img/bg/bg-scattered.png"
catalog:     true
tags:
  - 学习笔记
  - Metrics
  - Prometheus
  - Grafana
  - 可观测性
---

> 这篇笔记是对前面几篇的关系总览。目标不是重复定义，而是把 `Metrics`、`Prometheus`、`Grafana` 这三层概念拆清楚，让后面再学 `Micrometer`、`PromQL`、`Dashboard`、`Alerting` 时不容易混。

> Metrics 深度学习笔记：`/2026/05/21/02/metrics监控数据/`

> Prometheus 深度学习笔记：`/2026/05/21/03/prometheus监控系统/`

> Grafana 深度学习笔记：`/2026/05/21/04/grafana监控可视化/`

[TOC]

---

## 1. 最短答案

如果只用三句话回答：

- `Metrics` 是一种观测数据抽象
- `Prometheus` 是一个以 Metrics 为核心的数据采集、存储、查询与告警系统
- `Grafana` 是一个把数据源查询结果组织成可视化界面的平台

也就是说：

- `Metrics` 是“数据类型 / 观测方式”
- `Prometheus` 是“处理这类数据的系统”
- `Grafana` 是“把这些数据展示给人的界面层”

这三者经常一起出现，是因为它们刚好形成一条很自然的链路：

```text
应用产生 Metrics
    -> Prometheus 抓取并存储
    -> Grafana 查询并展示
```

---

## 2. 为什么很多人会把三者混在一起

因为在日常工作里，大家经常同时接触到：

- 应用暴露 `/metrics`
- Prometheus 抓这些 metrics
- Grafana 把它们画成图

久而久之就会形成一种错觉：

- “这不就是一套东西吗”

但这其实是：

- 三层不同抽象恰好串成了一条常见工具链

就像：

- SQL 是查询语言
- MySQL 是数据库
- DataGrip 是客户端

三者有关，但显然不是同一种东西。

---

## 3. 先拆开看：Metrics 是抽象，不是产品

### 3.1 Metrics 本质是什么

Metrics 是一种把系统状态、事件和分布压缩成数值时间序列的方式。

它关注的是：

- 当前状态
- 变化趋势
- 聚合统计
- 告警阈值

典型例子：

- 请求总数
- 错误总数
- 当前连接数
- 队列长度
- 请求耗时分布

### 3.2 Metrics 解决什么问题

它擅长回答：

- 系统整体现在怎么样
- 趋势有没有异常
- 某类问题有没有扩大
- 哪个维度最有问题

它不擅长回答：

- 某条具体请求的详细过程
- 某个用户做了什么操作
- 一段错误日志的完整上下文

### 3.3 Metrics 不是 Prometheus 发明的

这是一个常见误解。

Metrics 这个观测思路早于 Prometheus，也不依赖 Prometheus。

你完全可以把 Metrics 发到：

- Prometheus
- InfluxDB
- CloudWatch
- Datadog
- OpenTelemetry 后端

所以：

- `Metrics` 是更上层的概念
- `Prometheus` 只是其中一种非常主流的落地系统

---

## 4. 再看 Prometheus：它是 Metrics 系统

### 4.1 Prometheus 处理的是 Metrics

Prometheus 围绕 Metrics 做了完整系统化设计，包括：

- 抓取
- 存储
- 查询
- 聚合
- 录制规则
- 报警规则

所以它不是抽象定义，而是：

- 具体产品 / 具体系统

### 4.2 它和 Metrics 的关系

可以这样理解：

- Metrics 是“粮食”
- Prometheus 是“仓库 + 加工厂 + 计算系统”

也就是说：

- Metrics 是被处理对象
- Prometheus 是处理它们的基础设施

### 4.3 为什么不是所有 Metrics 都一定进 Prometheus

因为 Prometheus 有自己的适用边界：

- 适合数值型时序
- 适合 pull 模型
- 适合 label-based data model

如果你的需求是：

- 全文检索日志
- 分布式 trace 分析
- 海量长期归档

那 Prometheus 就不是唯一答案。

---

## 5. 再看 Grafana：它是界面层，不是 Metrics 系统本体

### 5.1 Grafana 不定义 Metrics

Grafana 不负责定义：

- Counter 是什么
- Histogram 是什么
- 标签基数应该怎么控制

这些语义不是 Grafana 创造的。

### 5.2 Grafana 也不天然负责存储 Metrics

Grafana 官方文档明确说明：

- data source 才是存储后端
- Grafana 只负责查询这些后端并展示结果

所以如果没有后端数据源：

- Grafana 自己并不会凭空产生监控数据

### 5.3 它的职责更像“人机交互层”

Grafana 最核心的价值在于：

- 把 Prometheus 等后端的查询结果变成 dashboard、panel、variable、Explore、alerting 等人可消费的形式

所以更准确地说：

- Grafana 是 observability UI / UX 层

---

## 6. 三者的层级关系

我比较推荐这样记：

### 6.1 第一层：数据抽象层

- `Metrics`

回答：

- 我们要观测的是什么数据形态

### 6.2 第二层：监控系统层

- `Prometheus`

回答：

- 这些 Metrics 怎么采、怎么存、怎么查、怎么算、怎么报警

### 6.3 第三层：展示与协作层

- `Grafana`

回答：

- 人怎么查看、筛选、组合、分享和分析这些结果

所以三者并不是平级替代关系，而是：

- 上下游协作关系

---

## 7. 一个最常见的真实链路

在 Java / Spring Boot 场景里，一条典型链路通常是：

```text
应用代码
    -> Micrometer 记录 Metrics
    -> Spring Boot Actuator 暴露 /actuator/prometheus
    -> Prometheus 周期抓取
    -> Prometheus TSDB 存储
    -> Grafana 配置 Prometheus data source
    -> Dashboard / Explore 查询展示
```

如果把它拆开看：

- `Micrometer`：应用埋点门面
- `Metrics`：暴露出来的数据模型
- `Prometheus`：采存算
- `Grafana`：展示和探索

这条链路里每一层都有自己的职责，没有谁应该替代谁。

---

## 8. 一个高频误区：把 Metrics 当成 Prometheus 语法

很多人学了 `Counter`、`Gauge`、`Histogram` 之后，会下意识觉得：

- 这就是 Prometheus 的东西

其实不完全对。

更准确的说法应该是：

- 这些是 Metrics 世界里的核心类型
- Prometheus 采用并强化了这套思维

同样地：

- Micrometer 有 `Timer`
- OpenTelemetry Metrics 也有自己的 instrument 语义

所以：

- `Metrics` 是更广义抽象
- `Prometheus` 是其中一个非常重要的实现生态

---

## 9. 一个高频误区：把 Grafana 当成“监控系统”

很多团队平时打开 Grafana 最多，所以会下意识说：

- “我们 Grafana 监控挂了”

这句话日常沟通没问题，但技术上其实要拆开。

因为一个监控链路里可能出现几种不同故障：

### 9.1 应用没暴露 Metrics

这时问题在：

- 应用埋点或 exporter

### 9.2 Prometheus 没抓到

这时问题在：

- target
- 网络
- 配置
- service discovery
- relabeling

### 9.3 Prometheus 有数据但 Grafana 查错了

这时问题在：

- panel query
- variable
- time range
- panel transform

### 9.4 Grafana 页面正常，但图为空

这并不说明：

- Grafana 没数据存储能力不行

更可能是：

- 后端根本没返回数据

这就是为什么三者一定要拆开理解。

---

## 10. 用“职责表”来硬区分

### 10.1 Metrics 的职责

- 定义观测数据是什么
- 提供聚合和告警友好的数值化表达
- 让系统状态可持续采样

### 10.2 Prometheus 的职责

- 抓取 Metrics
- 存储 time series
- 提供 PromQL 查询
- 执行 recording rules 和 alerting rules

### 10.3 Grafana 的职责

- 连接 data source
- 组织 dashboard 和 panel
- 提供 variables / Explore / visualization
- 提供统一查询与展示入口

所以如果把一句话说透，就是：

- `Metrics` 决定“看什么”
- `Prometheus` 决定“怎么采存算”
- `Grafana` 决定“怎么给人看”

---

## 11. 如果少了其中一个，会发生什么

### 11.1 没有 Metrics

那你甚至没有统一的可观测数值模型。

结果通常是：

- 只能看日志
- 很难做趋势监控
- 很难做聚合告警

### 11.2 没有 Prometheus

你仍然可以有 Metrics，但需要别的系统来承担：

- 采集
- 存储
- 查询

比如别的 TSDB 或云监控平台。

### 11.3 没有 Grafana

Prometheus 仍然可以工作。

你仍然能：

- 在表达式浏览器里查
- 做 recording rules
- 做 alerting rules

但体验通常会差很多，尤其在：

- 大盘展示
- 多图联动
- 参数切换
- 多数据源统一查看

这些方面。

所以三者并不是：

- 缺一个绝对不能运行

而是：

- 缺一个就会丢掉对应层面的能力

---

## 12. 三者为什么常常要一起学

因为真正工程落地时，问题不会只出在某一层。

举几个常见例子：

### 12.1 图为空

你要同时判断：

- Metrics 有没有打出来
- Prometheus 有没有抓到
- Grafana query 有没有写对

### 12.2 告警不准

你要同时判断：

- Metrics 语义是不是就不对
- Prometheus rule 是否合适
- Grafana 图是不是掩盖了真实问题

### 12.3 查询很慢

你要同时判断：

- Metrics 标签基数是不是爆了
- Prometheus 查询是不是太重
- Grafana dashboard 是否一次刷太多 panel

这说明在真实工作里：

- 这三层必须串起来理解

---

## 13. 用一个具体例子彻底打通

假设你要监控一个订单服务的下单接口。

### 13.1 Metrics 层做什么

你定义并上报：

- 请求总数
- 错误总数
- 下单耗时

例如：

- `http_server_requests_seconds`
- `order_created_total`
- `order_create_failed_total`

### 13.2 Prometheus 层做什么

Prometheus 周期抓取这些指标，并支持你写：

```promql
sum(rate(order_created_total[5m]))
```

或：

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(http_server_requests_seconds_bucket{uri="/orders"}[5m]))
)
```

同时它还能根据规则判断：

- 错误率是否过高
- p95 是否超过阈值

### 13.3 Grafana 层做什么

Grafana 把这些表达式做成：

- 订单服务总览 dashboard
- 环境变量切换
- 接口分维度图表
- 值班大盘

这样值班同学就不需要每次都手写 PromQL。

这个例子里：

- Metrics 提供原料
- Prometheus 提供计算和存储
- Grafana 提供认知界面

---

## 14. 一个常见学习顺序

我建议的顺序通常是：

### 14.1 先学 Metrics

先搞清楚：

- Counter / Gauge / Histogram / Timer
- 标签基数
- 命名和单位

因为如果这层错了，后面都建立在错误数据上。

### 14.2 再学 Prometheus

再学：

- data model
- scrape
- TSDB
- PromQL
- rules

因为这是把 Metrics 系统化落地的核心。

### 14.3 再学 Grafana

最后学：

- dashboard
- panel
- variables
- Explore
- alerting
- provisioning

因为这一步是把系统变成团队可消费的界面。

这个顺序的好处是：

- 不容易把展示层误当成本体

---

## 15. 如果再加一层 Micrometer，该怎么放

你前面已经在写 Spring Boot / Metrics 笔记，所以这一点值得顺手说明。

在 Java 体系里，常见链路是：

```text
业务代码
    -> Micrometer
    -> Metrics 暴露
    -> Prometheus 抓取
    -> Grafana 展示
```

这里：

- `Micrometer` 是埋点门面
- `Metrics` 是数据抽象
- `Prometheus` 是后端系统
- `Grafana` 是界面层

所以 `Micrometer` 和 `Prometheus` 也不是一回事。

### 15.1 如果把这套东西用到 Caffeine 缓存上

结合你前面的 Caffeine 笔记，这条链路其实可以非常具体地落下来：

```text
业务代码使用 Caffeine / Spring Cache
    -> Caffeine 统计 hit / miss / load / eviction / size
    -> Micrometer 把这些统计绑定到 MeterRegistry
    -> Spring Boot Actuator 暴露 /actuator/prometheus
    -> Prometheus 周期抓取
    -> Grafana 画出缓存命中率、淘汰、容量和加载耗时
```

这时候三者的分工就非常清楚：

- `Metrics`：缓存命中、miss、淘汰、加载耗时、当前大小这些观测数据
- `Prometheus`：负责把这些缓存指标采集下来并支持查询、聚合和告警
- `Grafana`：负责把缓存健康度做成 dashboard 给人看

也就是说：

- 你不是“把 Caffeine 接入 Grafana”
- 而是“把 Caffeine 的运行统计先变成 Metrics，再交给 Prometheus 和 Grafana”

### 15.2 监控 Caffeine 时，最该关心哪些指标

如果只抓最有价值的一组，通常是：

- `hit` / `miss`
- `hit rate`
- `load success` / `load failure`
- `load duration`
- `eviction count`
- `cache size`

为什么是这几个？

- `hit rate` 直接反映缓存到底有没有帮你减少回源
- `load duration` 反映 miss 之后回源到底贵不贵
- `eviction count` 反映容量或策略是否太激进
- `cache size` 反映当前缓存规模是否接近上限

如果结合你那篇 Caffeine 笔记，可以把它理解成：

- `recordStats()` 让 Caffeine 开始积累命中、miss、加载、淘汰这类统计
- `maximumSize` / `maximumWeight`、过期策略、刷新策略会直接影响这些指标长什么样
- `removalListener(...)` 更适合看具体移除事件，Metrics 更适合看整体趋势

### 15.3 `.recordStats()` 到底做了什么

先把最关键的一句说透：

- `.recordStats()` 只会让 `Caffeine` 开始在本地内存里累计统计
- 它不会自己把指标暴露给 `Prometheus`

也就是说，它做的是：

```text
开启 Caffeine 内部统计
    -> 产生 CacheStats
```

它没有做的是：

```text
把 CacheStats 自动暴露成 /actuator/prometheus
```

所以如果你脑子里把这一步理解成：

- `.recordStats()` = 已经接入监控

那就一定会混乱。

更准确的理解应该是：

- `.recordStats()` 只是让缓存开始“记账”
- `Micrometer` 才负责把这本账翻译成 `Metrics`
- `Actuator + micrometer-registry-prometheus` 才负责把这些 `Metrics` 暴露出来

把完整链路写成一行就是：

```text
recordStats()
    -> Caffeine 内部有了 CacheStats
    -> Micrometer 绑定成 meter
    -> Actuator 暴露 /actuator/prometheus
    -> Prometheus 抓取
```

所以一个最短判断口诀是：

- `.recordStats()` = 开始记账
- `Micrometer` = 把账本变成 Metrics
- `Actuator` = 把 Metrics 暴露出来
- `Prometheus` = 来抓这些 Metrics

少任何一步，都不算真正接入完成。

### 15.4 Spring Boot 自动接入怎么做

如果你走的是 Spring Boot 的缓存抽象，也就是：

- `@EnableCaching`
- `@Cacheable`
- `CaffeineCacheManager`

那么最常见、最省事的做法是走自动接入。

#### 第一步：引入依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>

    <dependency>
        <groupId>com.github.ben-manes.caffeine</groupId>
        <artifactId>caffeine</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

这四组依赖分别负责：

- `starter-cache`：开启 Spring Cache 体系
- `caffeine`：真正的本地缓存实现
- `actuator`：提供 metrics 端点
- `micrometer-registry-prometheus`：把 metrics 导出成 Prometheus 格式

#### 第二步：让底层 Caffeine 开启统计

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("user", "dict");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return cacheManager;
    }
}
```

这里最关键的一行就是：

- `.recordStats()`

没有它，Caffeine 自己都没有把 hit、miss、eviction 这些统计积累起来，后面 Micrometer 也就没有足够的原料可绑。

#### 第三步：暴露 Prometheus 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

这样你通常就能看到：

- `/actuator/metrics`
- `/actuator/prometheus`

#### 第四步：确认自动绑定是否真的成立

自动接入能成立，通常要满足这些前提：

- cache 是通过 Spring `CacheManager` 管理的
- 应用里已经有 `Actuator`
- 应用里已经有 `micrometer-registry-prometheus`
- 底层 Caffeine 开了 `recordStats()`
- 这些 cache 在启动时就已经能被 Spring 看到

如果这些条件满足，Spring Boot / Micrometer 通常会帮你把 cache metrics 绑定到 `MeterRegistry`。

#### 第五步：直接验证，不要靠猜

启动应用后，先不要急着写 Grafana 大盘，先做两个检查：

1. 打开 `/actuator/metrics`，看看有没有类似 `cache.gets`、`cache.evictions`、`cache.size`
2. 打开 `/actuator/prometheus`，看看有没有对应的 cache 相关导出项

如果这里都没有，再去看 Prometheus 抓取配置也没用，因为应用本身还没真正暴露出来。

#### 一个最小使用示意

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Cacheable(cacheNames = "user", key = "#id")
    public User findById(Long id) {
        return loadUserFromDb(id);
    }
}
```

这一套跑起来以后，流程才是：

```text
@Cacheable 命中 user cache
    -> 底层 Caffeine 记录 hit / miss
    -> Micrometer 自动绑定这些统计
    -> /actuator/prometheus 暴露出来
```

### 15.5 非 Spring Cache 场景怎么手动接入

如果你不是用 Spring `CacheManager` 管理缓存，而是自己直接写：

- `Cache<K, V> cache = Caffeine.newBuilder()...build()`

那就不要指望 Spring Boot 一定帮你自动绑上。

这时更稳妥的方式是：

- 手动创建 cache
- 手动调用 `CaffeineCacheMetrics.monitor(...)`

示例：

```java
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.cache.CaffeineCacheMetrics;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class LocalCacheConfig {

    @Bean
    public Cache<Long, String> userCache(MeterRegistry registry) {
        Cache<Long, String> cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats()
            .build();

        CaffeineCacheMetrics.monitor(registry, cache, "userCache");
        return cache;
    }
}
```

这里每一步的作用要分得很清楚：

- `recordStats()`：让 Caffeine 自己开始统计
- `monitor(registry, cache, "userCache")`：把这些统计注册到 `MeterRegistry`
- `/actuator/prometheus`：把 registry 里的内容导出来

如果少了中间这一步：

```java
CaffeineCacheMetrics.monitor(registry, cache, "userCache");
```

那你得到的只是：

- 缓存内部有统计

而不是：

- Prometheus 能抓到统计

#### 什么时候通常需要手动接入

下面这些场景，优先按“手动绑定”来理解更稳：

- 你直接 new 了独立 Caffeine cache
- cache 不是 Spring `CacheManager` 管的
- cache 是运行过程中动态创建的
- 你发现自动接入没有生效，但又确定缓存本身在工作

#### 最小排查顺序

如果你说“我明明开了 `.recordStats()`，为什么还是没有指标”，可以按这个顺序排：

1. 有没有 `spring-boot-starter-actuator`
2. 有没有 `micrometer-registry-prometheus`
3. `/actuator/prometheus` 是否真的暴露了
4. cache 是否真的开了 `recordStats()`
5. 当前是自动接入，还是应该手动 `monitor(...)`
6. 最后再看 Prometheus 的 `scrape_configs`

### 15.6 Prometheus 在这条链路里具体干什么

Prometheus 并不关心你底层是不是 Caffeine，它只关心：

- 你的应用有没有把缓存相关 Metrics 暴露出来

抓取配置示意：

```yaml
scrape_configs:
  - job_name: "spring-boot"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["app:8080"]
```

抓到之后，你就能写一些非常典型的缓存查询。

例如，5 分钟命中率：

```promql
sum(rate(cache_gets_total{cache="user",result="hit"}[5m]))
/
sum(rate(cache_gets_total{cache="user"}[5m]))
```

例如，5 分钟淘汰速率：

```promql
sum(rate(cache_evictions_total{cache="user"}[5m]))
```

例如，当前缓存大小：

```promql
max(cache_size{cache="user"})
```

例如，平均加载耗时：

```promql
sum(rate(cache_load_duration_seconds_sum{cache="user"}[5m]))
/
sum(rate(cache_load_duration_seconds_count{cache="user"}[5m]))
```

实际写查询时还要注意一点：

- 具体 metric 名和 label 名可能会随着 Micrometer / Spring Boot 版本、导出方式略有差异
- 最稳妥的做法始终是先打开 `/actuator/prometheus` 看实际暴露结果，再写 PromQL

这些查询的意义分别是：

- 命中率低：缓存收益可能不足
- 淘汰速率高：容量或策略可能不合理
- 加载耗时高：miss 的代价可能很贵
- 大小持续顶格：缓存空间可能不够

### 15.7 Grafana 上通常看什么

到了 Grafana 这一层，重点就不再是“有没有数据”，而是“怎么把缓存状态展示得便于排障和决策”。

一个比较实用的 Caffeine dashboard，通常会放：

- 命中率趋势
- hit / miss QPS
- load success / failure
- 平均加载耗时
- eviction 趋势
- cache size

如果再做得工程化一点，还可以把这些图和下面指标一起放着看：

- 下游数据库 QPS
- 下游 RPC RT
- JVM 堆内存
- GC 停顿

因为真实线上排查时，你通常想一起判断：

- 命中率下降，是 key 设计问题、TTL 问题、容量问题，还是下游本来就慢

### 15.8 用 Caffeine 这个例子再反过来看三者关系

如果你监控的是 Caffeine，本质上不是三者关系变了，而是例子变具体了。

这时可以把三层重新翻译成：

- `Metrics`：缓存系统的可观测事实，例如命中、miss、淘汰、大小、加载耗时
- `Prometheus`：把这些事实按时间序列保存下来，并支持你算 hit rate、eviction rate、平均 load latency
- `Grafana`：把这些结果变成缓存健康度大盘、值班图表和告警入口

所以真正的接入动作不是：

- Caffeine 直接把图画给你看

而是：

- Caffeine 先通过 Micrometer 暴露缓存指标
- Prometheus 再负责采集和查询
- Grafana 最后负责展示和协作

### 15.9 监控 Caffeine 时几个很容易踩的坑

#### 15.9.1 只配了缓存，没配统计

这时你可能有缓存功能，但没有足够好的缓存观测数据。

也就是说：

- 缓存在跑
- 但你不知道命中率到底好不好

#### 15.9.2 只看命中率，不看加载耗时

命中率不低，不代表缓存策略就一定合理。

因为还可能出现：

- 命中率还行
- 但一旦 miss，回源特别慢

这会让尾延迟非常难看。

#### 15.9.3 忘了 Caffeine 是本地缓存

如果你有多个应用实例，那么每个实例都有自己的本地缓存统计。

这意味着：

- 某一台机器 hit rate 很高
- 不代表整个集群所有实例都一样

所以看 Prometheus / Grafana 时，通常要注意实例维度和聚合方式。

#### 15.9.4 把“图空了”误判成 Grafana 问题

缓存图为空时，仍然要按这条链路拆开排查：

- 是不是应用没暴露指标
- 是不是 Prometheus 没抓到
- 还是 Grafana 查询条件写错了

---

## 16. 一个最值得记住的心智模型

我比较推荐记成下面这句话：

- `Metrics` 是语言，`Prometheus` 是引擎，`Grafana` 是仪表盘

更展开一点：

- `Metrics` 告诉你“系统可以如何被数值化描述”
- `Prometheus` 告诉你“这些数值如何被采集、存储、计算和告警”
- `Grafana` 告诉你“这些结果如何被人高效查看和使用”

如果脑子里始终有这个分层，很多概念混乱都会自动消失。

---

## 17. 最常见的错误说法与更准确的说法

### 17.1 错误说法

- “Grafana 存了我们的监控数据”

更准确的说法：

- Grafana 查询并展示后端监控数据，真正存储通常在 Prometheus 或其他数据源

### 17.2 错误说法

- “Prometheus 就是 Metrics”

更准确的说法：

- Prometheus 是处理 Metrics 的系统，不等于 Metrics 这个抽象本身

### 17.3 错误说法

- “有 Grafana 就有监控”

更准确的说法：

- Grafana 是观察入口，真正监控是否成立还取决于 Metrics 设计、Prometheus 采集和告警规则

---

## 18. 这篇笔记最该带走的结论

1. `Metrics`、`Prometheus`、`Grafana` 是三个不同层次的概念。
2. `Metrics` 是观测数据抽象，不是某个具体产品。
3. `Prometheus` 是围绕 Metrics 构建的采集、存储、查询和告警系统。
4. `Grafana` 是连接数据源并进行可视化、探索与协作的平台。
5. `Metrics` 决定数据语义，`Prometheus` 决定系统处理方式，`Grafana` 决定人怎么消费这些数据。
6. 把三者混成一个东西，会直接影响排障和系统设计判断。
7. 真实链路通常是“应用产生 Metrics -> Prometheus 抓取 -> Grafana 展示”。
8. 学习顺序通常是先 `Metrics`，再 `Prometheus`，最后 `Grafana`。
9. `Grafana` 不是数据库，`Prometheus` 也不是 Metrics 这个概念本身。
10. 真正成熟的可观测性认知，一定要建立在这种分层理解上。

---

## 19. 适合继续延伸的方向

如果后面你想把这组专题继续补全，比较自然的下一步是：

- `PromQL 深度学习笔记`
- `Grafana Dashboard 设计实战笔记`
- `Alertmanager 深度学习笔记`
- `Micrometer 深度学习笔记`

这样整套从埋点到展示的链路就会更完整。
