---
layout:      post
title:       "SpringCloud入门"
subtitle:    "SpringCloud"
author:      "Ekko"
header-img:  "img/bg/bg-springcloud.jpg"
catalog:     true
tags:
  - 学习笔记
  - 微服务
  - 分布式
---

> 参考资料 [3y](https://zhuanlan.zhihu.com/p/43023436)、[CAP理论中的P到底是个什么意思](https://www.zhihu.com/question/54105974)、[分布式系统的CAP理论](https://www.hollischuang.com/archives/666)、[SpringCloud学习之路（一）](https://juejin.im/post/6844903806531190791)、[JavaGuide](https://github.com/Snailclimb/JavaGuide/blob/master/docs/system-design/micro-service/spring-cloud.md)

> [Spring Cloud Ribbon的原理-负载均衡策略](https://www.cnblogs.com/kongxianghai/p/8477781.html)、[Eureka参数配置项详解](https://www.cnblogs.com/fangfuhai/p/7070325.html)、[Spring Cloud Eureka详解](https://blog.csdn.net/sunhuiliang85/article/details/76222517)、[Hystrix几篇文章《青芒》](https://segmentfault.com/u/yedge/articles)、[什么是负载均衡，什么是轮询策略、随机策略、哈希策略](https://www.jianshu.com/p/eb9df00b6d1d)

[TOC]

---

## CAP 理论

- **C（consistency）：** 数据一致性，所有节点拥有数据的最新版本
- **A（availability）：** 可用性，数据具备高可用性
- **P（partition-tolerance）：** 分区容错性，容忍网络出现分区，分区之间网络不可达

下面有三个节点(它们是集群的)，此时三个节点都能够相互通信：

![三节点通信.png](/asserts/images/2020-08-16-SpringCloud入门/三节点通信.png)

由于系统是分布式的，节点之间的通信是通过网络来进行的。只要是分布式系统，那很有可能会出现一种情况：因为一些故障，使得有些节点之间不连通了，整个网络就分成了几块区域

**数据就散布在了这些不连通的区域中，这就叫分区**

![三节点通信失败.png](/asserts/images/2020-08-16-SpringCloud入门/三节点通信失败.png)

现在出现了网络分区后，此时有一个请求过来了，想要注册一个账户

![网络分区后接受请求.png](/asserts/images/2020-08-16-SpringCloud入门/网络分区后接受请求.png)

此时节点一和节点三是不可通信的，这就有了抉择：

如果允许当前用户注册一个账户，此时注册的记录数据只会在节点一和节点二或者节点二和节点三同步，因为节点一和节点三的记录不能同步的

这种情况其实就是**选择了可用性(availability)，抛弃了数据一致性(consistency)**

如果不允许当前用户注册一个账户(就是要等到节点一和节点三恢复通信)。节点一和节点三一旦恢复通信，我们就可以保证节点拥有的数据是最新版本

这种情况其实就是**抛弃了可用性(availability)，选择了数据一致性(consistency)**

一般说的分布式系统，P：分区容错性(partition-tolerance)这个是必需的，这是客观存在的

CAP 是无法完全兼顾的，从上面的例子也可以看出，可以选 AP，也可以选 CP。但是，要注意的是：不是说选了 AP，C 就完全抛弃了。不是说选了 CP，A 就完全抛弃了

在CAP理论中，C所表示的一致性是强一致性(每个节点的数据都是最新版本)，其实一致性还有其他级别的：

- **弱一致性：** 弱一致性是相对于强一致性而言，它不保证总能得到最新的值
- **最终一致性(eventual consistency)：** 放宽对时间的要求，在被调完成操作响应后的某个时间点，被调多个节点的数据最终达成一致

用性指“Reads and writes always succeed”，即服务一直可用，而且是正常响应时间

对于一个可用性的分布式系统，每一个非故障的节点必须对每一个请求作出响应。所以，一般在衡量一个系统的可用性的时候，都是通过停机时间来计算的

![可用性停机时间.png](/asserts/images/2020-08-16-SpringCloud入门/可用性停机时间.png)

所以，CAP 理论定义的其实是在容忍网络分区的条件下，“强一致性”和“极致可用性”无法同时达到

---

## SpringCloud 是什么

![SpringCloud架构.png](/asserts/images/2020-08-16-SpringCloud入门/SpringCloud架构.png)

Spring Cloud 是一系列框架的有序集合。它利用Spring Boot的开发便利性巧妙地简化了分布式系统基础设施的开发，如**服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等**，都可以用 Spring Boot 的开发风格做到一键启动和部署

Spring 并没有重复制造轮子，它只是将目前各家公司开发的比较成熟、经得起实际考验的服务框架组合起来，通过 Spring Boot 风格进行再封装屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、易部署和易维护的分布式系统开发工具包

Spring Cloud 正是对 Netflix 的多个开源组件进一步的封装而成，同时又实现了和云端平台，和 Spring Boot 开发框架很好的集成

Spring Cloud 是一个相对比较新的微服务框架，2016年才推出 1.0 的 release 版本. 虽然 Spring Cloud 时间最短, 但是相比 Dubbo 等 RPC 框架, Spring Cloud 提供的全套的分布式系统解决方案

Spring Cloud 为开发者提供了在分布式系统（配置管理，服务发现，熔断，路由，微代理，控制总线，一次性 token，全局琐，leader 选举，分布式 session，集群状态）中快速构建的工具，使用 Spring Cloud 的开发者可以快速的启动服务或构建应用、同时能够快速和云平台资源进行对接

**基础功能：**

* **服务治理：** Spring  Cloud Eureka
* **客户端负载均衡：** Spring Cloud Ribbon
* **服务容错保护：** Spring  Cloud Hystrix  容错管理工具，实现断路器模式，通过控制服务的节点，从而对延迟和故障提供更强大的容错能力
* 声明式服务调用：Spring  Cloud Feign 基于Ribbon 和Hystrix 的声明式服务调用组件
* API网关服务：Spring Cloud Zuul 提供动态路由，访问过滤等服务
* 分布式配置中心：Spring Cloud Config Config：配置管理工具，支持使用Git 存储配置内容，支持应用配置的外部化存储，支持客户端配置信息刷新、加解密配置内容等

**高级功能：**

* 消息总线：Spring  Cloud Bus
* 消息驱动的微服务：Spring Cloud Stream
* 分布式服务跟踪：Spring  Cloud Sleuth

**主要项目：**

|项目名称|项目职能|
|:----|:----|
|Spring Cloud Config|Spring Cloud 提供的分布式配置中心，为外部配置提供了客户端和服务端的支持|
|**Spring Cloud Netflix**|**与各种Netflix OSS组件集成（Eureka，Hystrix，Zuul，Archaius等）**|
|Spring Cloud Bus|用于将服务和服务实例与分布式消息传递连接在一起的事件总线。用于跨群集传播状态更改（例如，配置更改事件）|
|Spring Cloud Cloudfoundry|提供应用程序与 Pivotal Cloud Foundry 集成。提供服务发现实现，还可以轻松实现受SSO和OAuth2保护的资源|
|Spring Cloud Open Service Broker|为构建实现 Open service broker API 的服务代理提供了一个起点|
|Spring Cloud Cluster|提供Leadership选举，如：Zookeeper, Redis, Hazelcast, Consul等常见状态模式的抽象和实现|
|Spring Cloud Consul|封装了Consul操作，consul 是一个服务发现与配置工具，与Docker容器可以无缝集成|
|Spring Cloud Security|基于spring security的安全工具包，为你的应用程序添加安全控制。在Zuul代理中为负载平衡的OAuth2 rest客户端和身份验证头中继提供支持|
|Spring Cloud Sleuth|Spring Cloud 提供的分布式链路跟踪组件，兼容zipkin、HTracer和基于日志的跟踪（ELK）|
|Spring Cloud Data Flow|大数据操作工具，作为Spring XD的替代产品，它是一个混合计算模型，结合了流数据与批量数据的处理方式|
|**Spring Cloud Stream**|**数据流操作开发包，封装了与Redis,Rabbit、Kafka等发送接收消息**|
|Spring Cloud CLI|基于 Spring Boot CLI，可以让你以命令行方式快速建立云组件|
|Spring Cloud OpenFeign|一个http client客户端，致力于减少http client客户端构建的复杂性|
|Spring Cloud Gateway|Spring Cloud 提供的网关服务组件|
|Spring Cloud Stream App Starters|Spring Cloud Stream App Starters是基于Spring Boot的Spring 集成应用程序，可提供与外部系统的集成|
|Spring Cloud Task|提供云端计划任务管理、任务调度|
|Spring Cloud Task App Starters|Spring Cloud任务应用程序启动器是SpringBoot应用程序，它可以是任何进程，包括不会永远运行的Spring批处理作业，并且在有限的数据处理周期后结束/停止|
|Spring Cloud Zookeeper|操作Zookeeper的工具包，用于使用zookeeper方式的服务发现和配置管理|
|Spring Cloud AWS|提供与托管的AWS集成|
|Spring Cloud Connectors|便于云端应用程序在各种PaaS平台连接到后端，如：数据库和消息代理服务|
|Spring Cloud Starters|Spring Boot式的启动项目，为Spring Cloud提供开箱即用的依赖管理|
|Spring Cloud Contract|Spring Cloud Contract是一个总体项目，其中包含帮助用户成功实施消费者驱动合同方法的解决方案|
|Spring Cloud Pipelines|Spring Cloud Pipelines提供了一个固定意见的部署管道，其中包含确保您的应用程序可以零停机方式部署并轻松回滚出错的步骤|
|Spring Cloud Function|Spring Cloud Function通过函数促进业务逻辑的实现。 它支持Serverless 提供商之间的统一编程模型，以及独立运行（本地或PaaS）的能力|

---

## Eureka 服务注册中心

分布式拆分子服务后，子系统之间的通讯必然成为需要考虑的问题

子系统与子系统之间不是在同一个环境下，那就需要远程调用。远程调用可能就会想到httpClient ，WebService 等等这些技术来实现

既然是远程调用，就必须知道 ip 地址，可能出现以下的场景

**功能实现一：A 服务需要调用 B 服务**

在 A 服务的代码里面调用 B 服务，显式通过 IP 地址调用：`http://123.123.123.123:8888/java3y/3`

**功能实现二：A服务调用B服务，B服务调用C服务，C服务调用D服务**

在 A 服务的代码里面调用 B 服务，显式通过 IP 地址调用：`http://123.123.123.123:8888/java3y/3`，(同样地)B->C，C->D

...
...

万一， B 服务的 IP 地址变了，那么其他的 A 服务, D 服务等都需要手动更新 B 服务的地址，在服务多的情况下手动维护这些静态配置是比较难的一件事

为了解决微服务架构中的服务实例维护问题( ip 地址)， 产生了大量的服务治理框架和产品。这些框架和产品的实现都围绕着服务注册与服务发现机制来完成对微服务应用实例的自动化管理

SpringCloud 中的服务治理框架一般使用的就是 Eureka

创建一个E服务，将 A、B、C、D 四个服务的信息都注册到 E 服务上，E 服务维护这些已经注册进来的信息

![Eureka服务注册中心.png](/asserts/images/2020-08-16-SpringCloud入门/Eureka服务注册中心.png)

A、B、C、D 四个服务都可以拿到 Eureka ( 服务E ) 那份注册清单。A、B、C、D 四个服务互相调用不再通过具体的 IP 地址，而是通过服务名来调用

- 拿到注册清单--->注册清单上有服务名--->自然就能够拿到服务具体的位置了(IP)
- 其实简单来说就是：代码中通过服务名找到对应的IP地址(IP地址会变，但服务名一般不会变)

Eureka 专门用于给其他服务注册的称为 Eureka Server(服务注册中心)，其余注册到Eureka Server 的服务称为 Eureka Client

**Eureka 注册中心的三种角色：**

1. **Eureka Server：** 通过Register、Get、Renew 等接口提供服务的注册和发现

2. **Application Service (Service Provider)** 服务提供方把自身的服务实例注册到Eureka Server 中

3. **Application Client (Service Consumer)** 服务调用方通过 Eureka Server 获取服务列表，消费服务

**在 Eureka Server 一般会这样配置：**

```java
register-with-eureka: false     #false表示不向注册中心注册自己。
    fetch-registry: false     #false表示自己端就是注册中心，我的职责就是维护服务实例，并不需要去检索服务
```

Eureka Client 分为服务提供者和服务消费者

> 某服务可以既是服务提供者又是服务消费者

即使 SpringCloud 的某个服务配置没有"注册"到 Eureka-Server ，但是它也是可以获取Eureka服务清单的，很可能只是把该服务认作为单纯的服务消费者，单纯的服务消费者无需对外提供服务，也就不必注册到 Eureka 中

```java
eureka:
  client:
    register-with-eureka: false  # 当前微服务不注册到eureka中(消费端)
    service-url: 
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/ 
```

---

**Eureka的治理机制：**

**服务提供者：**

- **服务注册：** 启动的时候会通过发送 REST 请求的方式将自己注册到 Eureka Server上，同时带上了自身服务的一些元数据信息
- **服务续约：** 在注册完服务之后，服务提供者会维护一个心跳用来持续告诉 Eureka Server “我还活着”
- **服务下线：** 当服务实例进行正常的关闭操作时，它会触发一个服务下线的 REST 请求给Eureka Server, 告诉服务注册中心：“我要下线了”

**服务消费者：**

- **获取服务：** 当启动服务消费者的时候，它会发送一个REST请求给服务注册中心，来获取上面注册的服务清单
- **服务调用：** 服务消费者在获取服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。在进行服务调用的时候，优先访问同处一个Zone中的服务提供方

**Eureka Server(服务注册中心)：**

- **失效剔除：** 默认每隔一段时间（默认为60秒） 将当前清单中超时（默认为90秒）没有续约的服务剔除出去
- **自我保护：** EurekaServer 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%(通常由于网络不稳定导致)。 Eureka Server 会将当前的实例注册信息保护起来， 让这些实例不会过期，尽可能保护这些注册信息

![Eureka服务治理.png](/asserts/images/2020-08-16-SpringCloud入门/Eureka服务治理.png)

---

## Eureka 和 zookeeper 对比

**Eureka：**

Spring Cloud Netflix 在设计 **Eureka 时遵守的就是 AP （高可用、分区容错性）原则**

Eureka Server 也可以运行多个实例来构建集群，解决单点问题，但不同于 ZooKeeper 的选举 leader 的过程，Eureka Server采用的是 Peer to Peer 对等通信。这是一种去中心化的架构，无 master/slav e区分，每一个 Peer 都是对等的。在这种架构中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的 serviceUrl 指向其他节点。每个节点都可被视为其他节点的副本

如果某台 Eureka Server 宕机，Eureka Client 的请求会自动切换到新的 Eureka Server 节点，当宕机的服务器重新恢复后，Eureka 会再次将其纳入到服务器集群管理之中。当节点开始接受客户端请求时，所有的操作都会进行 replicateToPeer（节点间复制）操作，将请求复制到其他 Eureka Server 当前所知的所有节点中

一个新的 Eureka Server 节点启动后，会首先尝试从邻近节点获取所有实例注册表信息，完成初始化。Eureka Server 通过 getEurekaServiceUrls() 方法获取所有的节点，并且会通过心跳续约的方式定期更新

默认配置下，如果 Eureka Server 在一定时间内没有接收到某个服务实例的心跳，Eureka Server 将会注销该实例（默认为90秒，通过 eureka.instance.lease-expiration-duration-in-seconds 配置）。当 Eureka Server 节点在短时间内丢失过多的心跳时（比如发生了网络分区故障），那么这个节点就会进入自我保护模式

**Zookeeper：**

**Zookeeper 是基于 CP （强一致性、分区容错性）来设计的**，即任何时刻对 Zookeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性，但是它不能保证每次服务请求的可用性。从实际情况来分析，在使用 Zookeeper 获取服务列表时，如果 zookeeper 正在选主，或者 Zookeeper 集群中半数以上机器不可用，那么将无法获得数据。所以说，Zookeeper 不能保证服务可用性

在大多数分布式环境中，尤其是涉及到数据存储的场景，数据一致性应该是首先被保证的，这也是 zookeeper 设计成 CP 的原因

但是对于服务发现场景来说，情况就不太一样了：针对同一个服务，即使注册中心的不同节点保存的服务提供者信息不尽相同，也并不会造成灾难性的后果。因为对于服务消费者来说，能消费才是最重要的——拿到可能不正确的服务实例信息后尝试消费一下，也好过因为无法获取实例信息而不去消费。 （尝试一下可以快速失败，之后可以更新配置并重试）所以，对于服务发现而言，可用性比数据一致性更加重要—— AP 胜过 CP

**总结：** 

ZooKeeper 基于 CP，不保证高可用，如果 zookeeper 正在选主，或者 Zookeeper集群中半数以上机器不可用，那么将无法获得数据

Eureka 基于 AP，能保证高可用，即使所有机器都挂了，也能拿到本地缓存的数据

作为注册中心，其实配置是不经常变动的，只有发版和机器出故障时会变。对于不经常变动的配置来说，CP 是不合适的，而 AP 在遇到问题时可以用牺牲一致性来保证可用性，既返回旧数据，缓存数据

理论上 Eureka 是更适合做注册中心。而现实环境中大部分项目可能会使用 ZooKeeper ，那是因为集群不够大，并且基本不会遇到用做注册中心的机器一半以上都挂了的情况。所以实际上也没什么大问题

---

## Ribbon 客户端负载均衡

通过 Eureka 服务治理框架，我们可以通过服务名来获取具体的服务实例的位置了(IP)。一般在使用 SpringCloud 的时候不需要自己手动创建 HttpClient 来进行远程调用

可以使用 Spring 封装好的 RestTemplate 工具类，使用起来很简单：

```java
// 传统的方式，直接显示写死IP是不好的
//private static final String REST_URL_PREFIX = "http://localhost:8001";
	
// 服务实例名
private static final String REST_URL_PREFIX = "http://MICROSERVICECLOUD-DEPT";

/**
* 使用 使用restTemplate访问restful接口非常的简单粗暴无脑。 (url, requestMap,
* ResponseBean.class)这三个参数分别代表 REST请求地址、请求参数、HTTP响应转换被转换成的对象类型。
*/
@Autowired
private RestTemplate restTemplate;

@RequestMapping(value = "/consumer/dept/add")
public boolean add(Dept dept) {
    return restTemplate.postForObject(REST_URL_PREFIX + "/dept/add", dept, Boolean.class);
}
```

为了实现服务的高可用，可以将服务提供者集群。比如说，现在一个秒杀系统设计出来了，准备上线了。在11月11号时为了能够支持高并发，开多台机器来支持并发量

![服务提供者集群.png](/asserts/images/2020-08-16-SpringCloud入门/服务提供者集群.png)

现在想要这三个秒杀系统合理摊分用户的请求(专业来说就是负载均衡)，可能你会想到nginx

其实 SpringCloud 也支持的负载均衡功能，只不过它是客户端的负载均衡，这个功能实现就是 Ribbon

**负载均衡又区分了两种类型：**

**客户端负载均衡(Ribbon)**

- 服务实例的清单在客户端，客户端进行负载均衡算法分配
- 客户端可以从 Eureka Server 中得到一份服务清单，在发送请求时通过负载均衡算法，在多个服务器之间选择一个进行访问

**服务端负载均衡(Nginx)**

- 服务实例的清单在服务端，服务器进行负载均衡算法分配

![客户端服务端负载均衡.png](/asserts/images/2020-08-16-SpringCloud入门/客户端服务端负载均衡.png)

---

**Ribbon 细节：**

Ribbon是支持负载均衡，默认的负载均衡策略是轮询，我们也是可以根据自己实际的需求自定义负载均衡策略的

```java
@Configuration
public class MySelfRule
{
	@Bean
	public IRule myRule()
	{
		//return new RandomRule();// Ribbon默认是轮询，自定义为随机
		//return new RoundRobinRule();// Ribbon默认是轮询，自定义为随机
		return new RandomRule_ZY();// 我自定义为每台机器5次
	}
}
```

实现起来也很简单：继承 AbstractLoadBalancerRule 类，重写 public Server choose(ILoadBalancer lb, Object key) 即可

SpringCloud 在 CAP 理论是选择了 AP 的，在 Ribbon 中还可以配置重试机制的

---

## Nginx 和 Ribbon 的对比

**Nignx** 

和Ribbon不同的是，它是一种集中式的负载均衡器。集中式简单理解就是将所有请求都集中起来，然后再进行负载均衡。如下图

![Nignx负载均衡.png](/asserts/images/2020-08-16-SpringCloud入门/Nignx负载均衡.png)

**Ribbon** 

来说它是在消费者端进行的负载均衡。如下图

![Ribbon负载均衡.png](/asserts/images/2020-08-16-SpringCloud入门/Ribbon负载均衡.png)

注意 Request 的位置，在 Nginx 中请求是先进入负载均衡器，而在Ribbon中是先在客户端进行负载均衡才进行请求的

---

## Hystrix 服务容错保护

调用多个远程服务时，当某个服务出现延迟：

![SpringCloud调用延迟.png](/asserts/images/2020-08-16-SpringCloud入门/SpringCloud调用延迟.png)

在高并发的情况下，由于单个服务的延迟，可能导致所有的请求都处于延迟状态，甚至在几秒钟就使服务处于负载饱和的状态，资源耗尽，直到不可用，最终导致这个分布式系统都不可用，这就是“雪崩”

![SpringCloud延迟雪崩.png](/asserts/images/2020-08-16-SpringCloud入门/SpringCloud延迟雪崩.png)

针对上述问题， Spring Cloud Hystrix 实现了断路器、线程隔离等一系列服务保护功能

- **Fallback(失败快速返回)：** 当某个服务单元发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延
- **资源/依赖隔离(线程池隔离)：** 它会为每一个依赖服务创建一个独立的线程池，这样就算某个依赖服务出现延迟过高的情况，也只是对该依赖服务的调用产生影响，而不会拖慢其他的依赖服务

**Hystrix提供几个熔断关键参数：滑动窗口大小（20）、 熔断器开关间隔（5s）、错误率（50%）**

- 每当 **20** 个请求中，有 **50%** 失败时，熔断器就会打开，此时再调用此服务，将会直接返回失败，不再调远程服务
- 直到 **5s** 钟之后，重新检测该触发条件，判断是否把熔断器关闭，或者继续打开

**熔断：** 

是服务雪崩的一种有效解决方案。当指定时间窗内的请求失败率达到设定阈值时，系统将通过断路器直接将此请求链路断开。可以使用简单的 @HystrixCommand 注解来标注某个方法，这样 Hystrix 就会使用断路器来“包装”这个方法，每当调用时间超过指定时间时(默认为1000ms)，断路器将会中断对这个方法的调用

```java
@HystrixCommand(
    commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1200")}
)
public List<Xxx> getXxxx() {
    // ...省略代码逻辑
}
```

**降级** 

是为了更好的用户体验，当一个方法调用异常时，通过执行另一种代码逻辑来给用户友好的回复。这也就对应着 Hystrix 的后备处理模式

比如这个时候有一个热点新闻出现了，系统会推荐给用户查看详情，然后用户会通过 id 去查询新闻的详情，但是因为这条新闻太火了，大量用户同时访问可能会导致系统崩溃，那么系统就进行服务降级，一些请求会做一些降级处理比如当前人数太多请稍后查看等等

```java
// 指定了后备方法调用
@HystrixCommand(fallbackMethod = "getHystrixNews")
@GetMapping("/get/news")
public News getNews(@PathVariable("id") int id) {
    // 调用新闻系统的获取新闻api 代码逻辑省略
}
//
public News getHystrixNews(@PathVariable("id") int id) {
    // 做服务降级
    // 返回当前人数太多，请稍后查看
}
```

---

## Feign 声明式服务调用

上面已经介绍了 Ribbon （客户端负载均衡）和 Hystrix （服务容错保护）了，可以发现的是：他俩作为基础工具类框架广泛地应用在各个微服务的实现中。我们会发现对这两个框架的使用几乎是同时出现的

为了简化我们的开发，Spring Cloud Feign 出现了！它基于 Netflix Feign 实现，整合了 Spring Cloud Ribbon 与 Spring Cloud Hystrix, 除了整合这两者的强大功能之外，它还提供了**声明式的服务调用(不再通过RestTemplate)**

Feign 是一种声明式、模板化的 HTTP 客户端。在 Spring Cloud 中使用 Feign, 我们可以做到使用 HTTP 请求远程服务时能与调用本地方法一样的编码体验，开发者完全感知不到这是远程方法，更感知不到这是个 HTTP 请求

**Feign 服务绑定**

```java
// value --->指定调用哪个服务
// fallbackFactory--->熔断器的降级提示
@FeignClient(value = "MICROSERVICECLOUD-DEPT", fallbackFactory = DeptClientServiceFallbackFactory.class)
public interface DeptClientService {
    // 采用Feign我们可以使用SpringMVC的注解来对服务进行绑定！
    @RequestMapping(value = "/dept/get/{id}", method = RequestMethod.GET)
    public Dept get(@PathVariable("id") long id);

    @RequestMapping(value = "/dept/list", method = RequestMethod.GET)
    public List<Dept> list();

    @RequestMapping(value = "/dept/add", method = RequestMethod.POST)
    public boolean add(Dept dept);
}
```

**Feign中使用熔断器：**

```java
/**
 * Feign中使用断路器
 * 这里主要是处理异常出错的情况(降级/熔断时服务不可用，fallback就会找到这里来)
 */
@Component // 不要忘记添加，不要忘记添加
public class DeptClientServiceFallbackFactory implements FallbackFactory<DeptClientService> {
    @Override
    public DeptClientService create(Throwable throwable) {
        return new DeptClientService() {
            @Override
            public Dept get(long id) {
                return new Dept().setDeptno(id).setDname("该ID：" + id + "没有没有对应的信息,Consumer客户端提供的降级信息
                ,此刻服务Provider已经关闭").setDb_source("no this database in MySQL");
            }

            @Override
            public List<Dept> list() {
                return null;
            }

            @Override
            public boolean add(Dept dept) {
                return false;
            }
        };
    }
}
```

**调用：**

```java
@RestController
public class DeptController_Consumer{
    @Autowired
    private DeptClientService deptClientService;

    @RequestMapping(value = "/consumer/dept/get/{id}")
    public Dept get(@PathVarible("id") Lopg id){
        return deptClientService.get(id);
    }

    @RequestMapping(value = "/consumer/dept/list")
    public List<Dept> list(){
        return deptClientService.list();
    }

    @RequestMapping(value = "/consumer/dept/add")
    public Object add(Dept dept){
        return deptClientService.add(dept);
    }
}
```

---

## Zuul 网关服务

基于上面的学习，现在的架构很可能会设计成这样：

![没有网关的微服务.png](/asserts/images/2020-08-16-SpringCloud入门/没有网关的微服务.png)

这样的架构会有两个比较麻烦的问题：

- **路由规则与服务实例的维护间题：** 外层的负载均衡(nginx)需要维护所有的服务实例清单(图上的OpenService)
- **签名校验、 登录校验冗余问题：** 为了保证对外服务的安全性， 我们在服务端实现的微服务接口，往往都会有一定的权限校验机制，但我们的服务是独立的，我们不得不在这些应用中都实现这样一套校验逻辑，这就会造成校验逻辑的冗余

![nginx维护每个服务实例地址.png](/asserts/images/2020-08-16-SpringCloud入门/nginx维护每个服务实例地址.png)

每个服务都有自己的 IP 地址，Nginx 想要正确请求转发到服务上，就必须维护着每个服务实例的地址

更是灾难的是：这些服务实例的IP地址还有可能会变，服务之间的划分也很可能会变

购物车和订单模块都需要用户登录了才可以正常访问，基于现在的架构，只能在购物车和订单模块都编写校验逻辑，这无疑是冗余的代码

为了解决上面这些常见的架构问题，**API 网关**的概念应运而生。在 SpringCloud 中了提供了基于 Netflix Zuul 实现的API网关组件 Spring Cloud Zuul

Spring Cloud Zuul 是这样解决上述两个问题的：

- SpringCloud Zuul 通过与 SpringCloud Eureka 进行整合，将自身注册为 Eureka 服务治理下的应用，同时从 Eureka 中获得了所有其他微服务的实例信息。外层调用都必须通过API 网关，使得将维护服务实例的工作交给了服务治理框架自动完成
- 在 API 网关服务上进行统一调用来对微服务接口做前置过滤，以实现对微服务接口的拦截和校验

Zuul 天生就拥有线程隔离和断路器的自我保护功能，以及对服务调用的客户端负载均衡功能。也就是说：Zuul 也是支持 Hystrix 和 Ribbon

---

**Zuul 过滤功能**

如果说，路由功能是 Zuul 的基操的话，那么过滤器就是 Zuul 的利器。毕竟所有请求都经过网关(Zuul)，那么我们可以进行各种过滤，这样我们就能实现限流，灰度发布，权限控制等等

![Zuul过滤器.png](/asserts/images/2020-08-16-SpringCloud入门/Zuul过滤器.png)

**过滤器类型：Pre、Routing、Post**

- 前置 Pre 就是在请求之前进行过滤
- Routing 路由过滤器就是我们上面所讲的路由策略
- Post 后置过滤器就是在 Response 之前进行过滤的过滤器

简单实现一个请求时间日志打印，要实现自己定义的 Filter 我们只需要继承 ZuulFilter 然后将这个过滤器类以 @Component 注解加入 Spring 容器中就行了

```java
// 加入Spring容器
@Component
public class PreRequestFilter extends ZuulFilter {
    // 返回过滤器类型 这里是前置过滤器
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }
    // 指定过滤顺序 越小越先执行，这里第一个执行
    // 当然不是只真正第一个 在 Zuul 内置中有其他过滤器会先执行
    // 那是写死的 比如 SERVLET_DETECTION_FILTER_ORDER = -3
    @Override
    public int filterOrder() {
        return 0;
    }

    // 什么时候该进行过滤
    // 这里我们可以进行一些判断，这样我们就可以过滤掉一些不符合规定的请求等等
    @Override
    public boolean shouldFilter() {
        return true;
    }
    
    // 如果过滤器允许通过则怎么进行处理
    @Override
    public Object run() throws ZuulException {
        // 这里我设置了全局的RequestContext并记录了请求开始时间
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set("startTime", System.currentTimeMillis());
        return null;
    }
}

// lombok的日志
@Slf4j
// 加入 Spring 容器
@Component
public class AccessLogFilter extends ZuulFilter {
    // 指定该过滤器的过滤类型
    // 此时是后置过滤器
    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }
    // SEND_RESPONSE_FILTER_ORDER 是最后一个过滤器
    // 我们此过滤器在它之前执行
    @Override
    public int filterOrder() {
        return FilterConstants.SEND_RESPONSE_FILTER_ORDER - 1;
    }
    @Override
    public boolean shouldFilter() {
        return true;
    }
    // 过滤时执行的策略
    @Override
    public Object run() throws ZuulException {
        RequestContext context = RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
        // 从RequestContext获取原先的开始时间 并通过它计算整个时间间隔
        Long startTime = (Long) context.get("startTime");
        // 这里我可以获取HttpServletRequest来获取URI并且打印出来
        String uri = request.getRequestURI();
        long duration = System.currentTimeMillis() - startTime;
        log.info("uri: " + uri + ", duration: " + duration / 100 + "ms");
        return null;
    }
}
```

---

**Zuul 令牌桶限流**

不仅仅只有令牌桶限流方式，Zuul 只要是限流的活都能干，这里只简单举个例子

![Zuul令牌桶.png](/asserts/images/2020-08-16-SpringCloud入门/Zuul令牌桶.png)

**令牌桶限流：** 

会有个桶，如果里面没有满那么就会以一定固定的速率会往里面放令牌，一个请求过来首先要从桶中获取令牌，如果没有获取到，那么这个请求就拒绝，如果获取到那么就放行

通过 Zuul 的前置过滤器来实现一下令牌桶限流

```java
@Component
@Slf4j
public class RouteFilter extends ZuulFilter {
    // 定义一个令牌桶，每秒产生2个令牌，即每秒最多处理2个请求
    private static final RateLimiter RATE_LIMITER = RateLimiter.create(2);
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return -5;
    }

    @Override
    public Object run() throws ZuulException {
        log.info("放行");
        return null;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        if(!RATE_LIMITER.tryAcquire()) {
            log.warn("访问量超载");
            // 指定当前请求未通过过滤
            context.setSendZuulResponse(false);
            // 向客户端返回响应码429，请求数量过多
            context.setResponseStatusCode(429);
            return false;
        }
        return true;
    }
}
```

还可以实现权限校验，包括上面提到的灰度发布等等。

Zuul 作为网关肯定也存在单点问题，如果我们要保证 Zuul 的高可用，我们就需要进行 Zuul 的集群配置，这个时候可以借助额外的一些负载均衡器比如 Nginx

---

**Zuul 的路由功能**

比如这个时候已经向 Eureka Server 注册了两个 Consumer 、三个 Provicer ，这个时候再加个 Zuul 网关应该变成这样子

![加zuul网关的架构.png](/asserts/images/2020-08-16-SpringCloud入门/加zuul网关的架构.png)

首先，Zuul 需要向 Eureka 进行注册，可以拿到所有 consumer 的信息，包括 consumer 的元数据（名称、ip、端口），拿到这些元数据后便可以做**路由映射**，比如原来用户调用 Consumer1 的接口 localhost:8001/studentInfo/update 这个请求，现在可以变成 localhost:9000/consumer1/studentInfo/update

**zuul 最基本的配置：**

```java
server:
  port: 9000
eureka:
  client:
    service-url:
      # 这里只要注册 Eureka 就行了
      defaultZone: http://localhost:9997/eureka
```

然后在启动类上加入 @EnableZuulProxy 注解就行了

**zuul 统一前缀：**

```java
zuul:
  prefix: /zuul
```

这样我们就需要通过 localhost:9000/zuul/**consumer1**/studentInfo/update 来进行访问了

**路由策略配置：**

你会发现前面的访问方式(直接使用服务名)，需要将微服务名称暴露给用户，会存在安全性问题。所以，可以自定义路径来替代微服务名称，即自定义路由策略

```java
zuul:
  routes:
    consumer1: /FrancisQ1/**
    consumer2: /FrancisQ2/**
```

这个时候就可以使用 localhost:9000/zuul/**FrancisQ1**/studentInfo/update` 进行访问了

**服务名屏蔽：**

这个时候你别以为你好了，你可以试试，在你配置完路由策略之后使用微服务名称还是可以访问的，这个时候你需要将服务名屏蔽

```java
zuul:
  ignore-services: "*"
```

**路径屏蔽：**

Zuul 还可以指定屏蔽掉的路径 URI，即只要用户请求中包含指定的 URI 路径，那么该请求将无法访问到指定的服务。通过该方式可以限制用户的权限

```java
zuul:
  ignore-patterns: **/auto/**
```

这样关于 auto 的请求我们就可以过滤掉了

> ** 代表匹配多级任意路径
> *代表匹配一级任意路径

**敏感请求头屏蔽：**

默认情况下，像 Cookie、Set-Cookie 等敏感请求头信息会被 zuul 屏蔽掉，我们可以将这些默认屏蔽去掉，当然，也可以添加要屏蔽的请求头



---

**Zuul 小结**

Zuul（网关）支持Ribbon（负载均衡）和Hystrix（容错保护），也能够实现客户端的负载均衡。我们的Feign不也是实现客户端的负载均衡和Hystrix的吗？既然Zuul已经能够实现了，那Feign还有必要吗？

![Zuul网关.png](/asserts/images/2020-08-16-SpringCloud入门/Zuul网关.png)

可以这样理解：

* zuul 是对外暴露的唯一接口相当于路由的是 controller 的请求，而 Ribbonhe 和 Fegin 路由了 service 的请求
* zuul 做最外层请求的负载均衡 ，而 Ribbon 和 Fegin 做的是系统内部各个微服务的service 的调用的负载均衡

---

## SpringCloud Config 分布式配置中心

随着业务的扩展，服务会越来越多。每个服务都有自己的配置文件，既然是配置文件，配置的东西难免会有些改动。比如每个服务中写的数据库配置，配置文件中的密码需要更换，那就得三个都要重新更改

![Config配置中心.png](/asserts/images/2020-08-16-SpringCloud入门/Config配置中心.png)

> 在分布式系统中，某一个基础服务信息变更，都很可能会引起一系列的更新和重启

Spring Cloud Config 项目是一个解决分布式系统的配置管理方案。它包含了 Client 和 Server 两个部分，server 提供配置文件的存储、以接口的形式将配置文件的内容提供出去，client 通过接口获取数据、并依据此数据初始化自己的应用

- 简单来说，使用 Spring Cloud Config 就是将配置文件放到统一的位置管理(比如 GitHub )，客户端通过接口去获取这些配置文件。
- 在 GitHub 上修改了某个配置文件，应用加载的就是修改后的配置文件

![Config配置中心结构.png](/asserts/images/2020-08-16-SpringCloud入门/Config配置中心结构.png)

在 SpringCloud Config 的服务端，对于配置仓库的默认实现采用了 Git，我们也可以配置 SVN

* 配置文件内的信息加密和解密
* 修改了配置文件，希望不用重启来动态刷新配置，配合Spring Cloud Bus 使用
* bootstrap.yml（bootstrap.properties）用来在程序引导时执行，应用于更加早期配置信息读取，如可以使用来配置application.yml中使用到参数等
* application.yml（application.properties) 应用程序特有配置信息，可以用来配置后续各个模块中需使用的公共参数等
* bootstrap.yml 先于 application.yml 加载，可以通过设置`spring.cloud.bootstrap.enabled=false`来禁用`bootstrap`

---

## Spring Cloud Bus 事件总线

如果在应用运行时去更改远程配置仓库( Git )中的对应配置文件，那么依赖于这个配置文件的已启动的应用并不会进行其相应配置的更改

动态修改配置文件可以使用 Webhooks，这是 github 提供的功能，它能确保远程库的配置文件更新后客户端中的配置信息也得到更新

但是 Webhooks 不适合用于生产环境，所以基本不会使用

一般选择使用 Bus 消息总线 + SpringCloud Config 进行配置的动态刷新

**SpringCloud Bus 用于将服务和服务实例与分布式消息系统链接在一起的事件总线。在集群中传播状态更改很有用（例如配置更改事件）**

简单理解为 Spring Cloud Bus 的作用就是管理和广播分布式系统中的消息，也就是消息引擎系统中的广播模式。当然作为 消息总线 的 Spring Cloud Bus 可以做很多事而不仅仅是客户端的配置刷新功能

而拥有了 Spring Cloud Bus 之后，我们只需要创建一个简单的请求，并且加上 @ResfreshScope 注解就能进行配置的动态修改了

![bus事件总线动态修改配置.png](/asserts/images/2020-08-16-SpringCloud入门/bus事件总线动态修改配置.png)