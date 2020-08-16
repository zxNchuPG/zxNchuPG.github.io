---
layout:      post
title:       "Zookeeper总结"
subtitle:    "Zookeeper"
author:      "Ekko"
header-img:  "img/bg/bg-zookeeper.jpg"
catalog:     true
tags:
  - 学习笔记
  - 分布式
  - 协调服务
  - 基础
  - 分布式锁
---

> 参考资料[腾讯技术](https://zhuanlan.zhihu.com/p/134549250)、[3y](https://www.zhihu.com/question/65852003/answer/656091418)、[CSDN老虎](https://blog.csdn.net/liuyuehu/article/details/52136231)、[Zookeeper官网](https://zookeeper.apache.org/)、[阿里云](https://zhuanlan.zhihu.com/p/44731983)、[CSDN统一配置管理](https://blog.csdn.net/u011320740/article/details/78742625)
、[JavaGuide](https://github.com/Snailclimb/JavaGuide)

[TOC]

---

## 分布式和集群

首先粗略介绍下分布式和集群的区别

集群（ Cluster ）：比如现在有一个秒杀服务，并发量太大，单机系统承受不住，那么加几台机器也一样提供秒杀服务，可以说是 集群

分布式（ Distributed ）：同样的秒杀服务，但是将秒杀服务**拆分**成多个子服务，然后将子服务部署在不同的服务器上，那么这个时候是 分布式

集群确实只需要加机器即可，但是分布式需要对业务进行拆分，之后再加机器，同时需要解决分布式带来的问题：各个分布式组件之间如何协调、解耦、事务处理、配置等等

---

## Zookeeper 简介

在大数据和云计算盛行的今天，应用服务由很多个独立的程序组成，这些独立的程序则运行在形形色色，千变万化的一组计算机上，而如何让一个应用中的多个独立的程序协同工作是一件非常困难的事情

而 ZooKeeper 就是一个分布式的，开放源码的分布式应用程序协调服务。它使得应用开发人员可以更多的关注应用本身的逻辑，而不是协同工作上。从系统设计看，ZooKeeper 从文件系统 API 得到启发，提供一组简单的 API，使得开发人员可以实现通用的协作任务，例如选举主节点，管理组内成员的关系，管理元数据等，同时 ZooKeeper 的服务组件运行在一组专用的服务器之上，也保证了高容错性和可扩展性

可以用 ZooKeeper 来做：统一配置管理、统一命名服务、分布式锁、集群管理

ZooKeeper 是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能

使用分布式系统就无法避免对节点管理的问题(需要实时感知节点的状态、对节点进行统一管理等等)，而由于这些问题处理起来可能相对麻烦和提高了系统的复杂性，ZooKeeper 作为一个能够通用解决这些问题的中间件就应运而生了

- ZooKeeper 本身就是一个分布式程序（只要半数以上节点存活，ZooKeeper 就能正常服务）

- 为了保证高可用，最好是以集群形态来部署 ZooKeeper，这样只要集群中大部分机器是可用的（能够容忍一定的机器故障），那么 ZooKeeper 本身仍然是可用的。

- ZooKeeper 将数据保存在内存中，这也就保证了 高吞吐量和低延迟（但是内存限制了能够存储的容量不太大，此限制也是保持znode中存储的数据量较小的进一步原因）。

- ZooKeeper 是高性能的。 在“读”多于“写”的应用程序中尤其地高性能，因为“写”会导致所有的服务器间同步状态。（“读”多于“写”是协调服务的典型场景。）

- ZooKeeper有临时节点的概念。 当创建临时节点的客户端会话一直保持活动，瞬时节点就一直存在。而当会话终结时，瞬时节点被删除。持久节点是指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在Zookeeper上。

- ZooKeeper 底层其实只提供了两个功能：①管理（存储、读取）用户程序提交的数据；②为用户程序提交数据节点监听服务

---

## ZooKeeper 的使命

ZooKeeper 主要的系统功能是在分布式系统中协作多个任务

例如，典型的主-从工作模式中，我们需要主节点和从节点进行协作，在从节点处于空闲状态时会通知主节点可以接受工作，于是主节点就会分配任务给从节点，同时我们只想有一个主节点，而很多进程可能都想成为主节点，这些操作都是要在多个任务中进行协作。另外，协同并不总是采取像主节点选举或者加锁等同步原语的形式，配置元数据也是一个进程通知其他进程需要做什么的一种常用数据，例如，在一个主-从系统中，从节点需要知道任务已经分配给他们，即便在主节点发生崩溃的情况下，这些信息也需要有效

在 ZooKeeper 之前，一些系统也可以采用分布式锁管理器或者分布式数据库来实现协作，例如，用数据库，redis 实现分布式锁。

那么 ZooKeeper 改变了什么呢？ZooKeeper 的设计更专注于任务协作，它不提供任何锁的接口或者通用的存储数据的接口，也没有强加任何特殊的同步原语，而是提供一个更加敏捷健壮的分布协作方案，例如在主-从模型中，ZooKeeper 没有为应用实现主节点选举，或者进程存活与否的跟踪功能，但是，ZooKeeper 提供了实现这些任务的工具，对于实现什么样的协同任务，有开发人员自己决定

分布式系统中关键在于进程通信，其有两种选择：直接通过网络进行信息交换，或者读写某些共享存储

对于 ZooKeeper 实现协作和同步原语本质上是使用共享存储模型，即开发的应用是连接到 ZooKeeper 服务器端的客户端，他们连接到 ZooKeeper 服务器端进行相关的操作，以来影响服务器端存储的共享数据，最终应用间实现协作

> 原语：操作系统或计算机网络用语范畴。是由若干条指令组成的，用于完成一定功能的一个过程。具有不可分割性·即原语的执行必须是连续的，在执行过程中不允许被中断

**ZooKeeper 不适合的场景**

整个 ZooKeeper 的服务器集群管理着应用协作的关键数据，ZooKeeper 不适合用作海量的数据存储，对于需要海量的应用数据的情况，可以使用数据库和分布式文件系统，所以在设计应用时，最佳实践是把应用数据和协同数据独立分开

---

## Zookeeper 数据结构

ZooKeeper的数据结构，跟Unix文件系统非常类似，可以看做是一颗树，每个节点叫做ZNode，并且暴露操作 API 接口。每一个节点可以通过路径来标识，结构图如下：

![Zookeeper数据结构](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.jpg?raw=true)

**Znode 的节点类型：** 在新建 znode 节点，需要指定该节点的类型，不同的类型决定了 znode 节点的行为方式，Znode 的类型分为三种：

- 持久节点（ PERSISTENT ）
- 临时节点（ EPHEMERAL ）
- 有序节点（ SEQUENTIAL ）

具体在节点创建过程中，一般是组合使用，可以生成以下 4 种节点类型：

- 持久节点（ PERSISTENT ）
- 持久有序节点（ PERSISTENT_SEQUENTIAL ）
- 临时节点（ EPHEMERAL ）
- 临时有序节点（ EPHEMERAL_SEQUENTIAL ）

**对于持久节点** 只能主动调用 delete 来删除，一般持久类型的 znode 为应用保存数据，即使 znode 的创建者不再属于应用系统时，数据会保存下来而不丢失

**临时的 znode** 在当创建该节点的客户端崩溃或者关闭了与 ZooKeeper 的连接时，这个节点就会被删除，临时 Znode 仅当创建者的会话有效时这些信息必须有效保存，会话超时或者主动关闭时，临时 znode 会自动消失

**有序 Znode 节点** 是被分配唯一一个单调递增的整数

**节点类型 枚举类 org.apache.zookeeper.CreateMode**

```java
public enum CreateMode {
// 第二个参数 持久 ；第三个参数 有序
/**
* The znode will not be automatically deleted upon client's disconnect.
* 持久节点
*/
PERSISTENT (0, false, false),
/**
* The znode will not be automatically deleted upon client's disconnect,
* and its name will be appended with a monotonically increasing number.
* 持久有序节点
*/
PERSISTENT_SEQUENTIAL (2, false, true),
/**
* The znode will be deleted upon the client's disconnect.
* 临时节点
*/
EPHEMERAL (1, true, false),
/**
* The znode will be deleted upon the client's disconnect, and its name
* will be appended with a monotonically increasing number.
* 临时有序节点
*/
EPHEMERAL_SEQUENTIAL (3, true, true);
    ...
    ...
}
```

**API 接口**

```java
create /path data 创建一个名为/path 的 znode 节点，并包含数据 data。
delete /path 删除名为/path 的 znode。
exists /path 检查是否存在名为/path 的节点。
setData /path data
getData /path
getChildren /path
```

需要注意的是，ZooKeeper 并不允许局部写入或读取 znode 节点的数据。当设置一个 znode 节点的数据或读取时，znode 节点的内容会被整个替换或者全部读取进来，特别是 getChildren，如果是数据量比较大，会获取大量的数据

---

## ZooKeeper 监视与通知

ZooKeeper 客户端获得服务器的数据或者变化，不是通过轮询的模式，而是基于通知的机制，客户端向 ZooKeeper 服务器端注册需要接收通知的 znode，通过对 znode 设置监视点来接收通知，需要强调的是监视点是一个单次触发的操作

常见的监听场景有两项：

- 监听 Znode 节点的**数据变化**
- 监听子节点的**增减变化**

---

## Zookeeper 统一配置管理

做项目时用到的配置比如数据库配置等...大多数都是写死在项目里面，如果需要更改，那么也是修改配置文件然后再投产上去，那么问题来了，如果做集群的呢，有100台机器，这时候做修改那就太不切实际了；那么就需要用到统一配置管理

比如我们现在有三个系统A、B、C，他们有三份配置，分别是ASystem.yml、BSystem.yml、CSystem.yml，然后，这三份配置又非常类似，很多的配置项几乎都一样。

此时，如果我们要改变其中一份配置项的信息，很可能其他两份都要改。并且，改变了配置项的信息很可能就要重启系统

于是，我们希望把ASystem.yml、BSystem.yml、CSystem.yml相同的配置项抽取出来成一份公用的配置common.yml，并且即便common.yml改了，也不需要系统A、B、C重启

**解决思路**

1. 把公共配置抽取出来

2. 对公共配置进行维护

3. 修改公共配置后应用不需要重新部署

![Zookeeper统一配置管理](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E7%BB%9F%E4%B8%80%E9%85%8D%E7%BD%AE%E7%AE%A1%E7%90%86.jpg?raw=true)

做法：我们可以将 common.yml 这份配置放在 ZooKeeper 的 Znode 节点中，系统 A、B、C 监听着这个 Znode 节点有无变更，如果变更了，及时响应

![Zookeeper监听统一配置](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E7%9B%91%E5%90%AC%E7%BB%9F%E4%B8%80%E9%85%8D%E7%BD%AE.jpg?raw=true)

**代码大致实现演示，需要用到的 jar 是 zkclient**

**配置文件实体**
```java
package com.nchu.zk.util;
import java.io.Serializable;
 
public class Config implements Serializable{

	private static final long serialVersionUID = 1L;
	private String userName;
	private String userPassword;
	
	public Config() {
	}

	public Config(String userName, String userPassword) {
		this.userName = userName;
		this.userPassword = userPassword;
	}

	public String getUserName() {
		return userName;
	}

	public void setUserName(String userName) {
		this.userName = userName;
	}

	public String getUserPassword() {
		return userPassword;
	}

	public void setUserPassword(String userPassword) {
		this.userPassword = userPassword;
	}

	@Override
	public String toString() {
		return "Config [userName=" + userName + ", userPassword=" + userPassword + "]";
	}
}
```

**配置管理中心ZkConfigManager**
```java
package com.nchu.zk.util;
import org.I0Itec.zkclient.ZkClient;
 
public class ZkConfigManager {
	private Config config;
	/**
	 * 从数据库加载配置
	 */
	public Config downLoadConfigFromDB(){
		//getDB
		config = new Config("nm", "pw");
		return config;
	}
	
	/**
	 * 配置文件上传到数据库
	 */
	public void upLoadConfigToDB(String name, String password){
		if(config == null) config = new Config();
		config.setUserName(name);
		config.setUserPassword(password);
		//updateDB
	}
	
	/**
	 * 配置文件同步到zookeeper
	 */
	public void syncConfigToZk(){
		ZkClient zk = new ZkClient("localhost:2181");
		if (!zk.exists("/zkConfig")) {
			zk.createPersistent("/zkConfig",true);
		}
		zk.writeData("/zkConfig", config);
		zk.close();
	}
}
```

**应用监听实现ZkGetConfigClient**
```java
package com.nchu.zk.util;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;
 
public class ZkGetConfigClient {
	private Config config;
	public Config getConfig() {
		ZkClient zk = new ZkClient("localhost:2181");
		config = (Config)zk.readData("/zkConfig");
		System.out.println("加载到配置："+ config.toString());
		
		//监听配置文件修改
		zk.subscribeDataChanges("/zkConfig", new IZkDataListener(){
			@Override
			public void handleDataChange(String arg0, Object arg1) throws Exception {
				config = (Config) arg1;
				System.out.println("监听到配置文件被修改：" + config.toString());
			}
 
			@Override
			public void handleDataDeleted(String arg0) throws Exception {
				config = null;
				System.out.println("监听到配置文件被删除");
			}
			
		});
		return config;
	}

	public static void main(String[] args) {
		ZkGetConfigClient client = new ZkGetConfigClient();
		client.getConfig();
		System.out.println(client.config.toString());
		for( int i = 0;i < 10; i++){
			System.out.println(client.config.toString());
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```

**测试，启动配置管理中心**
```java
package com.nchu.zkConfig.test;
 
import com.nchu.zk.util.Config;
import com.nchu.zk.util.ZkConfigManager;
 
public class ZkConfigTest {
	public static void main(String[] args) {
		ZkConfigManager manager = new ZkConfigManager();
		Config config = manager.downLoadConfigFromDB();
		System.out.println("....加载数据库配置...." + config.toString());
		manager.syncConfigToZk();
		System.out.println("....同步配置文件到zookeeper....");
		
		try {
			Thread.sleep(10000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		mag.upLoadConfigToDB("cwhcc", "passwordcc");
		System.out.println("....修改配置文件...."+config.toString());
		mag.syncConfigToZk();
		System.out.println("....同步配置文件到zookeeper....");
	}
}
```

**测试结果**

![Zookeeper配置中心测试结果](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png?raw=true)

**应用监听**

![Zookeeper应用监听测试结果](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E5%BA%94%E7%94%A8%E7%9B%91%E5%90%AC%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C.png?raw=true)

---

## Zookeeper 统一命名服务

统一命名服务的理解其实跟域名一样，为某一部分的资源给它取一个名字，别人通过这个名字就可以拿到对应的资源

如说，现在有一个域名 www.nchu.com ，但这个域名下有多台机器

- 192.168.1.1
- 192.168.1.2
- 192.168.1.3
- 192.168.1.4

别人访问 www.nchu.com 即可访问到机器，而不是通过IP去访问

![Zookeeper统一命名服务](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E7%BB%9F%E4%B8%80%E5%91%BD%E5%90%8D%E6%9C%8D%E5%8A%A1.jpg?raw=true)

---

## Zookeeper 分布式锁

分布式锁的实现方式有很多种，比如 Redis 、数据库 、zookeeper 等

**当使用 zookeeper 实现分布式锁时：**

上面已经提到过了 zk 在高并发的情况下保证节点创建的全局唯一性，其实就已经实现了互斥锁，又因为能在分布式的情况下，所以能实现分布式锁

可以利用临时节点的创建来实现分布式锁

**方案一：有序节点实现 共享锁 和 独占锁**

这个时候规定所有创建节点必须有序，当你是**读请求（要获取共享锁）**的话，如果没有比自己更小的节点，或比自己小的节点都是**读请求** ，则可以获取到读锁，然后就可以开始读了。若比自己小的节点中有写请求 ，则当前客户端无法获取到读锁，只能等待前面的写请求完成

如果你是**写请求（获取独占锁）**，若没有比自己更小的节点 ，则表示当前客户端可以直接获取到写锁，对数据进行修改。若发现有比自己更小的节点，无论是读操作还是写操作，当前客户端都无法获取到写锁 ，等待所有前面的操作完成

这就很好地同时实现了共享锁和独占锁，当然还有优化的地方，比如当一个锁得到释放它会通知所有等待的客户端从而造成羊群效应 。此时你可以通过让等待的节点只监听他们前面的节点

具体怎么做呢？其实也很简单，你可以让 读请求监听比自己小的最后一个写请求节点，写请求只监听比自己小的最后一个节点 ，感兴趣的小伙伴可以自己去研究一下

系统 A、B、C 都去访问 `/locks` 节点

![Zookeeper分布式锁](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81.jpg?raw=true)

访问的时候会创建带顺序号的临时/短暂( EPHEMERAL_SEQUENTIAL )节点，比如，系统 A 创建了 id_000000 节点，系统 B 创建了 id_000002 节点，系统 C 创建了 id_000001 节点

![Zookeeper分布式锁2](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zookeeper%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%812.jpg?raw=true)

接着，拿到 /locks 节点下的所有子节点( id_000000 , id_000001 , id_000002 )，判断自己创建的是不是最小的那个节点

- 如果是，则拿到锁。 释放锁：执行完操作后，把创建的节点给删掉
- 如果不是，则监听比自己要小1的节点变化

案例：

- 系统 A 拿到 /locks 节点下的所有子节点，经过比较，发现自己( id_000000 )，是所有子节点最小的，所以得到锁
- 系统 B 拿到 /locks 节点下的所有子节点，经过比较，发现自己( id_000002 )，不是所有子节点最小的。所以监听比自己小1的节点 id_000001 的状态
- 系统C拿到 /locks 节点下的所有子节点，经过比较，发现自己( id_000001 )，不是所有子节点最小的。所以监听比自己小1的节点 id_000000 的状态
- ……
- 等到系统 A 执行完操作以后，将自己创建的节点删除( id_000000 )。通过监听，系统 C 发现 id_000000 节点已经删除了，发现自己已经是最小的节点了，于是顺利拿到锁
- 系统 B 依次循环

**方案二：简单锁**

因为创建节点的唯一性，可以让多个客户端同时创建一个临时节点，创建成功的就说明获取到了锁 。然后没有获取到锁的客户端也像上面选主的非主节点创建一个 watcher 进行节点状态的监听，如果这个互斥锁被释放了（可能获取锁的客户端宕机了，或者那个客户端主动释放了锁）可以调用回调函数重新获得锁

> zk 中不需要向 redis 那样考虑锁得不到释放的问题了，因为当客户端挂了，节点也挂了，锁也释放了

---

##  ZooKeeper 架构

![ZooKeeper架构1](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/ZooKeeper%E6%9E%B6%E6%9E%841.jpg?raw=true)

![ZooKeeper架构2](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/ZooKeeper%E6%9E%B6%E6%9E%842.jpg?raw=true)

ZooKeeper 服务器端运行于两种模式下：
- 独立模式
- 仲裁模式

独立模式只有一个单独的服务器，ZooKeeper 状态无法复制

仲裁模式下，具有一组 ZooKeeper 服务器，称为 ZooKeeper 集合，它们之间可以进行状态的复制，并同时服务客户端的请求。不过服务器集合并不会让客户端等待每个服务器完成数据保存后再继续，而是在满足仲裁数目的服务器保存或者同步了状态就会返回给客户端

在解决这一分布式数据一致性，ZooKeeper 采用 ZAB(ZooKeeper Atomic Broadcast 原子广播协议)的一致性协议，关于 ZAB 协议后面会详细的介绍。

ZooKeeper 客户端在服务器集群中执行任何请求前必须先与服务器建立会话（session）

客户端提交给 ZooKeeper 的所有操作均关联在一个会话上。客户端初始化连接到集合中某个服务器或一个独立的服务器，客户端提供 TCP 协议与服务器进行连接并通信，但当会话无法与当前连接的服务器继续通信时，会话就可能转移到另外一个服务器，ZooKeeper 客户端透明地转移一个会话到不同的服务器。需要指明的，会话提供了顺序保障，同一个会话中的请求会以 FIFO（先进先出）顺序执行

---

## ZooKeeper 应用案例

**Apache HBase**

HBase 是一个通常与 Hadoop 一起使用的数据存储仓库。在 HBase 中，ZooKeeper 用于选举一个集群内的主节点，以便跟踪可用的服务器，并保持集群的元数据。

**Apache Kafka**

Kafka 是一个基于发布-订阅模型的消息系统。其中 ZooKeeper 用于检测崩溃，实现主题的发现，并保持主题的生产和消费状态。

**Apache Solr**

Solr 是一个企业级的搜索平台，它使用 ZooKeeper 来存储集群的元数据，并协作更新这些元数据。

ZooKeeper 应该是 “The King Of Coordination for Big Data”

总体来说 ZooKeeper 运行于一个集群环境中，选举出某个服务器作为群首（ Leader ），其他服务器追随群首（ Follower ）

群首作为中心处理所有对 ZooKeeper 系统变更的请求，它就像一个定序器，建立了所有对 ZooKeeper 状态的更新的顺序，追随者接收群首所发出更新操作请求，并对这些请求进行处理，以此来保障状态更新操作不会发生碰撞

---

## 请求、事务、标识符

ZooKeeper 服务器会在本地处理只读请求（exists、getData、getChildren），例如一个服务器接收客户端的 getData 请求，服务器读取该状态信息，并把这些信息返回给客户端

那些会改变 ZooKeeper 状态的客户端请求（create，delete 和 setData）将会转发到群首，群首执行对应的请求，并形成状态的更新，称为**事务（transaction）**，其中事务要以原子方式执行。同时，一个事务还要具有幂等性，事务的幂等性在我们进行恢复处理时更加简单，后面我们可以看到如何利用幂等性进行数据恢复或者灾备

在群首产生了一个事务，就会为该事务分配一个标识符，称为**会话 id（zxid）**，通过 zxid 对事务进行标识，就可以按照群首所指定的顺序在各个服务器中按序执行。服务器之间在进行新的群首选举时也会交换 zxid 信息，这样就可以知道哪个无故障服务器接收了更多的事务，并可以同步他们之间的状态信息

zxid 为一个 long 型（8 字节 = 64 位）整数，分为两部分：时间戳（epoch）部分和计数器（counter）部分。每一部分为 32 位，在我们讨论 zab 协议时，我们就会发现时间戳（epoch）和计数器（counter）的具体作用，我们通过 zab 协议来广播各个服务器的状态变更信息

---

## 群首选举

**群首** 为集群中的服务器选择出来的一个服务器，并会一直被集群所认可

设置群首的目的是为了对客户端所发起的 ZooKeeper 状态更新请求进行排序，包括 create，setData 和 delete 操作。群首将每一个请求转换为一个事务，将这些事务发送给追随者，确保集群按照群首确定的顺序接受并处理这些事务

每个服务器启动后进入 LOOKING 状态，开始选举一个新的群首或者查找已经存在的群首。如果群首已经存在，其他服务器就会通知这个新启动的服务器，告知哪个服务器是群首，于此同时，新服务器会与群首建立连接，以确保自己的状态与群首一致。如果群首中的所有的服务器均处于 LOOKING 状态，这些服务器之间就会进行通信来选举一个群首，通过信息交换对群首选举达成共识的选择。在本次选举过程中胜出的服务器将进入 LEADING 状态，而集群中其他服务器将会进入 FOLLOWING 状态

具体看，一个服务器进入 LOOKING 状态，就会发送向集群中每个服务器发送一个通知信息，该消息中包括该服务器的投票（vote）信息，投票中包含服务器标识符（sid）和最近执行事务的 zxid 信息

当一个服务器收到一个投票信息，该服务器将会根据以下规则修改自己的投票信息：

- 将接收的 voteId 和 voteZxid 作为一个标识符，并获取接收方当前的投票中的 zxid，用 myZxid 和 mySid 表示接收方服务器自己的值
- 如果（voteZxid > myZxid）或者（voteZxid == myZxid 且 voteId >mySid），保留当前的投票信息
- 否则，修改自己的投票信息，将 voteZxid 赋值给 myZxid，将 voteId 赋值给 mySid

从上面的投票过程可以看出，只有最新的服务器将赢得选举，因为其拥有最近一次的 zxid。如果多个服务器拥有的最新的 zxid 值，其中的 sid （服务器标识符）值最大的将会赢得选举

当一个服务器连接到仲裁数量的服务器发来的投票都一样时，就表示群首选举成功，如果被选举的群首为某个服务器自己，该服务器将会开始行使群首角色，否则就会成为一个追随者并尝试连接被选举的群首服务器。一旦连接成功，追随者和群首之间将会进行状态同步，在同步完成后，追随者才可以进行新的请求

---

## Zab：状态更新的广播协议

Zab 协议是为分布式协调服务Zookeeper专门设计的一种 支持崩溃恢复 的 原子广播协议 ，是 Zookeeper 保证数据一致性的核心算法。Zab 借鉴了 Paxos 算法，但又不像 Paxos 那样，是一种通用的分布式一致性算法。它是特别为 Zookeeper 设计的支持崩溃恢复的原子广播协议

Zab 核心思想是当多数 Server 写成功，则任务数据写成功。仲裁数量相当于法人，例如 5个 Server 中，有 3个 法人，只要 3个Server 成功，那么继续执行后面的操作

这也是为什么 Zookeeper 中建议最好使用奇数台服务器构成 Zookeeper 集群

在接收到一个写请求操作后，追随者会将请求转发给群首，群首将会探索性的执行该请求，并将执行结果以事务的方式对状态更新进行广播。如何确认一个事务是否已经提交，ZooKeeper 由此引入了 zab 协议，即原子广播协议（ZooKeeper Atomic Broadeast protocol）。该协议提交一个事务非常简单，类似于一个**两阶段提交**

- 群首向所有追随者发送一个 PROPOSAL 消息 p。
- 当一个追随者接收到消息 p 后，会响应群首一个 ACK 消息，通知群首其已接受该提案（proposal）。
- 当收到仲裁数量的服务器发送的确认消息后（该仲裁数包括群首自己），群首就会发送消息通知追随者进行提交（COMMIT）操作

**Zab中的三个角色：Leader 领导者、Follower跟随者、Observer观察者**

- **Leader：** 集群中 唯一的写请求处理者 ，能够发起投票（投票也是为了进行写请求）
- **Follower：** 能够接收客户端的请求，如果是读请求则可以自己处理，如果是写请求则要转发给 Leader 。在选举过程中会参与投票，有选举权和被选举权
- **Observer：** 就是没有选举权和被选举权的 Follower

在 ZAB 协议中对 zkServer(即上面说的三个角色的总称) 还有两种模式的定义，分别是 **消息广播** 和 **崩溃恢复**

---

**消息广播模式：**

ZAB 协议其实就是如何处理写请求的，上面提到只有 Leader 群首能处理写请求，但是 Follower 和 Observer 也是需要同步更新数据的，这也就是**在整个集群中保持数据的一致性**

第一步需要 Leader 将写请求 广播 出去，让 Leader 问问 Followers 是否同意更新，如果超过半数以上（仲裁数量）的同意那么就进行 Follower 和 Observer 的更新（和 Paxos 一样）

Zab 需要让 Follower 和 Observer 保证顺序性 。何为顺序性，比如我现在有一个写请求A，此时 Leader 将请求 A 广播出去，因为只需要半数同意就行，所以可能这个时候有一个 Follower F1 因为网络原因没有收到，而 Leader 又广播了一个请求B，因为网络原因，F1竟然先收到了请求 B 然后才收到了请求 A ，这个时候请求处理的顺序不同就会导致数据的不同，从而 产生数据不一致问题 

所以在 Leader 这端，它为每个其他的 zkServer 准备了一个 队列 ，采用先进先出的方式发送消息。由于协议是 **通过 TCP**来进行网络通信的，保证了消息的发送顺序性，接受顺序性也得到了保证。

除此之外，在 ZAB 中还定义了一个 全局单调递增的事务 ID ZXID ，它是一个 64 位 long 型，其中高 32 位表示 epoch 年代，低 32 位表示事务id。epoch 是会根据 Leader 的变化而变化的，当一个 Leader 挂了，新的 Leader 上位的时候，年代（epoch）就变了。而低 32 位可以简单理解为递增的事务id

定义这个的原因也是为了顺序性，每个 proposal 在 Leader 中生成后需要 通过其 ZXID 来进行排序 ，才能得到处理

---

**崩溃恢复模式**

说到**崩溃恢复**首先要提到 Zab 中的 Leader 选举算法，当系统出现崩溃影响最大的应该是 Leader 的崩溃，因为我们只有一个 Leader ，所以当 Leader 出现问题的时候我们势必需要重新选举 Leader

Leader 选举可以分为两个不同的阶段，第一个是我们提到的 Leader 宕机需要重新选举，第二则是当 Zookeeper 启动时需要进行系统的 Leader 初始化选举。下面先介绍一下 Zab 是如何进行初始化选举的

假设集群中有 3 台机器，那也就意味着需要两台以上同意（超过半数，也就是仲裁数量）。比如这个时候我们启动了 server1 ，它会首先 投票给自己 ，投票内容为服务器的 myid 和 ZXID ，因为初始化所以 ZXID 都为 0，此时 server1 发出的投票为 (1,0)。但此时 server1 的投票仅为1，所以不能作为 Leader ，此时还在选举阶段所以整个集群处于 Looking 状态

接着 server2 启动了，它首先也会将投票选给自己(2,0)，并将投票信息广播出去（server1也会，只是它那时没有其他的服务器了），server1 在收到 server2 的投票信息后会将投票信息与自己的作比较。首先它会比较 ZXID ，ZXID 大的优先为 Leader，如果相同则比较 myid，myid 大的优先作为 Leader。所以此时 server1 发现 server2 更适合做 Leader，它就会将自己的投票信息更改为 (2,0) 然后再广播出去，之后 server2  收到之后发现和自己的一样无需做更改，并且自己的投票已经超过半数 ，则 确定 server2 为 Leader，server1 也会将自己服务器设置为 Looking 变为 Follower。整个服务器就从 Looking 变为了正常状态

当 server3 启动发现集群没有处于 Looking 状态时，它会直接以 Follower 的身份加入集群

还是前面三个 server 的例子，如果在整个集群运行的过程中 server2 挂了，那么整个集群会如何重新选举 Leader 呢？其实和初始化选举差不多。

首先毫无疑问的是剩下的两个 Follower 会将自己的状态 从 Following 变为 Looking 状态 ，然后每个 server 会向初始化投票一样首先给自己投票（不过这里的 zxid 可能不是 0了，这里为了方便随便取个数字）

假设 server1 给自己投票为 (1,99)，然后广播给其他 server，server3 首先也会给自己投票(3,95)，然后也广播给其他 server。server1 和 server3 此时会收到彼此的投票信息，和一开始选举一样，他们也会比较自己的投票和收到的投票（zxid 大的优先，如果相同那么就 myid 大的优先）。这个时候 server1 收到了 server3 的投票发现没自己的合适故不变，server3 收到 server1 的投票结果后发现比自己的合适于是更改投票为 (1,99) 然后广播出去，最后 server1 收到了发现自己的投票已经超过半数就把自己设为 Leader，server3 也随之变为 Follower

**上面为选举，下面说什么是奔溃恢复**

其实主要就是当集群中有机器挂了，整个集群如何保证数据一致性？

如果只是 Follower 挂了，而且挂的没超过半数的时候，因为我们一开始讲了在 Leader 中会维护队列，所以不用担心后面的数据没接收到导致数据不一致性。

如果 Leader 挂了那就麻烦了，我们肯定需要先暂停服务变为 Looking 状态然后进行 Leader 的重新选举（上面我讲过了），但这个就要分为两种情况了，分别是确保已经被Leader 提交的提案最终能够被所有的 Follower 提交 和 跳过那些已经被丢弃的提案

确保已经被 Leader 提交的提案最终能够被所有的 Follower 提交是什么意思呢？

假设 Leader (server2) 发送 commit 请求，他发送给了 server3，然后要发给 server1 的时候突然挂了。这个时候重新选举的时候我们如果把 server1 作为 Leader 的话，那么肯定会产生数据不一致性，因为 server3 肯定会提交刚刚 server2 发送的 commit 请求的提案，而 server1 根本没收到所以会丢弃

![Zab崩溃恢复](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zab%E5%B4%A9%E6%BA%83%E6%81%A2%E5%A4%8D.jpg?raw=true)

所以这个时候 server1 已经不可能成为 Leader 了，因为 server1 和 server3 进行投票选举的时候会比较 ZXID ，而此时 server3 的 ZXID 肯定比 server1 的大了

那么跳过那些已经被丢弃的提案又是什么意思呢？

假设 Leader (server2) 此时同意了提案 N1，自身提交了这个事务并且要发送给所有 Follower 要 commit 的请求，却在这个时候挂了，此时肯定要重新进行 Leader 的选举，比如说此时选 server1 为 Leader （这无所谓）。但是过了一会，这个 挂掉的 Leader 又重新恢复了 ，此时它肯定会作为 Follower 的身份进入集群中，需要注意的是刚刚 server2 已经同意提交了提案 N1，但其他 server 并没有收到它的 commit 信息，所以其他 server 不可能再提交这个提案 N1 了，这样就会出现数据不一致性问题了，所以 该提案 N1 最终需要被抛弃掉

![Zab崩溃恢复2](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/Zab%E5%B4%A9%E6%BA%83%E6%81%A2%E5%A4%8D2.jpg?raw=true)

---

**Zab 保障了以下几个重要的属性**

如果群首按顺序广播了事务 T1 和事务 T2，那么每个服务器在提交 T2 事务前保证事务 T1 已经完成提交。

如果某个服务器按照事务 T1 和事务 T2 的顺序提交了事务，所有其他服务器也必然会在提交事务 T2 前提交事务 T1。

第一个属性保证事务在服务器之间传送顺序的一致，而第二个属性保证服务器不会跳过任何事务

---

## 观察者

观察者与追随者有一些共同的特点，他们提交来自群首的提议，不同于追随者的是，观察者不参与选举过程，他们仅仅学习经由 INFORM 消息提交的提议。

引入观察者的一个主要原因是提高读请求的可扩展性。通过加入多个观察者，我们可以在不牺牲写操作的吞吐率的前提下服务更多的读操作。但是引入观察者也不是完全没有开销，每一个新加入的观察者将对应于每一个已提交事务点引入的一条额外消息。

采用观察者的另外一个原因是进行跨多个数据中心部署。由于数据中心之间的网络链接延时，将服务器分散于多个数据中心将明显地降低系统的速度。引入观察者后，更新请求能够先以高吞吐量和低延迟的方式在一个数据中心内执行，接下来再传播到异地的其他数据中心得到执行

---

## 服务器的构成

群首，追随者，观察者根本上都是服务器。在实现服务器主要抽象概念是请求处理器。请求处理器是对处理流水线上不同阶段的抽象，每个服务器实现一个请求处理器的序列

**独立服务器**

PrepRequestProcessor 接受客户端的请求并执行这个请求，处理结果则是生成一个事务。不过只有改变 ZooKeeper 状态的操作才会产生事务，对于读操作并不会产生任何事务

SyncRequestProcessor 负责将事务持久化到磁盘上。实际上就是将事务数据按照顺序追加到事务日志中，并形成快照数据

最后一个处理器为 FinalRequestProcessor，如果 Request 对象包含事务数据，该处理器就会接受对 ZooKeeper 数据树的修改，否则，该处理器会从数据树中读取数据并返回客户端

**群首服务器**

在切换到仲裁模式时，服务器的流水线则有一些变化

![zookeeper仲裁模式](https://github.com/zxNchuPG/zxNchuPG.github.io/blob/master/img/notes/zookeeper%E4%BB%B2%E8%A3%81%E6%A8%A1%E5%BC%8F.jpg?raw=true)

第一个处理器同样是 PrepRequestProcessor，而之后的处理器则为 ProposalRequestProcessor，该处理器会准备一个提议，并将该提议发送给跟随者，并且会把所有请求转发给 CommitRequestProcessor，对于写操作请求，还会把请求转发给 SyncRequestProcessor 处理器

SyncRequestProcessor 和独立服务器的功能一样，是持久化事务到磁盘上，执行完后会触发 AckRequestProcessor 处理器，它仅仅生成确认消息并返回给自己

CommitRequestProcessor 会将收到足够多的确认消息的提议进行提交

**追随者和观察者服务器**

Follower 服务器是先从 FollowerRequestProcessors 处理器开始，该处理器接收并处理客户端请求，FollowerRequestProcessors 处理器之后转发请求给 CommitRequestProcessor，同时也会转发写请求到群首服务器。CommitRequestProcessor 会直接转发读取请求到 FinalRequestProcessor 处理器，而且对于写请求，在转发前会等待提交事务。而群首接收到一个新的写请求时会生成一个提议，之后转发到追随者服务器，在收到一个提议，追随服务器会发送这个提议到 SyncRequestProcessor，SendRequestProcessor 会向群首发送确认消息

当群首服务器接收到足够多确认消息来提交这个提议是，群首就会发送提交事务消息给追随者，当收到提交的事务消息时，追随者就通过 CommitRequestProcessor 处理器进行处理。为了保证执行的顺序，CommitRequestProcessor 处理器会在收到一个写请求处理器时暂停后续的请求处理

对于观察者服务器不需要确认提议消息，因此观察者服务器并不需要发送确认消息给群首服务器，一般情况下，也不用持久化事务到磁盘。对于观察者服务器是否持久化事务到磁盘，以便加速观察者服务器的恢复速度，可以根据具体情况决定

---

## 本地存储

SyncRequestProcessor 处理器是用于处理提议写入的日志和快照

**日志和磁盘的使用**

服务器通过事务日志来持久化事务。在接受一个提议时，一个服务器就会将提议的事务持久化到事务日志中，该事务日志保存在服务器本地磁盘中，而事务将会按照顺序追加其后

写事务日志是写请求操作的关键路径，因此 ZooKeeper 必须有效处理写日志问题。在持久化事务到磁盘时，还有一个重要说明：现代操作系统通常会缓存脏页（Dirty Page），并将他们异步写入磁盘介质。然而，我们需要在继续之前，要确保事务已经被持久化。因此我们需要冲刷（Flush）事务到磁盘介质

冲刷在这里就是指我们告诉操作系已经把脏页写入到磁盘，并在操作完成后返回。同时为了提高 ZooKeeper 系统的运行速度，也会使用组提交和补白的。其中组提交是指一次磁盘写入时追加多个事务，可以减少磁盘寻址的开销。补白是指在文件中预分配磁盘存储块

**快照**

快照是 ZooKeeper 数据树的拷贝副本，每一个服务器会经常以序列化整个数据树的方式来提取快照，并将这个提取的快照保存到文件。服务器在进行快照时不需要进行协作，也不需要暂停处理请求。因此服务器在进行快照时还会继续处理请求，所以当快照完成时，数据树可能又发生了变化，称为快照是模糊的，因为它们不能反映出在任意给定的时间点数据树的准确的状态

---

## 服务器与会话

会话（session）是 ZooKeeper 的一个重要的抽象。保证请求有序，临时 znode 节点，监控点都与会话密切相关。因此会话的跟踪机制对 ZooKeeper 来说也是非常重要的。

在独立模式下，单个服务器会跟踪所有的会话，而在仲裁模式下则由群首服务器来跟踪和维护。而追随者服务器仅仅是简单地把客户端连接的会话信息转发到群首服务器。

为了保证会话的存活，服务器需要接收会话的心跳信息。心跳的形式可以是一个新的请求或者显式的 ping 信息。两种情况下，服务器通过更新会话的过期时间来触发会话活跃，在仲裁模式下，群首服务器发送一个 PING 信息给它的追随者们，追随者们返回自从最新一次 PING 消息之后的一个 session 列表。群首服务器每半个 tick 就会发送一个 ping 信息给追随者们

---

## 服务器与监视点

监视点是由读取操作所设置的一次性触发器，每个监视点有一个特定操作来触发，即通过监视点，客户端可以对指定的 znode 节点注册一个通知请求，在发生时就会收到一个单次的通知。监视点只会存在内存，而不会持久化到硬盘，当客户端与服务端的连接断开时，它的所有的监视点会从内存中清除。因为客户端也会维护一份监视点的数据，在重连之后，监视点数据会再次同步到服务端

---

## 客户端

在客户端库中有 2 个主要的类：ZooKeeper 和 ClientCnxn，写客户端应用程序时通过实例化 ZooKeeper 类来建立一个会话。一旦建立起一个会话，ZooKeeper 就会使用一个会话标识符来关联这个会话。这个会话标识符实际上是由服务端所生产的。

ClientCnxn 类管理连接到 server 的 socket 连接。该类维护一个可连接的 ZooKeeper 的服务列表，并当连接断掉的时候无缝地切换到其他服务器，当重连到一个其他的服务器时会使用同一个会话，客户端也会重置所有的监视点到刚连接的服务器上

---

## 序列化

对于网络传输和磁盘保存的序列化消息和事务，ZooKeeper 使用了 Hadoop 中的 Jute 来做序列化
