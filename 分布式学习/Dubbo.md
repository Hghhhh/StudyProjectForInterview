参考博客：https://blog.csdn.net/noaman_wgs/article/details/70214612

## 电商系统的演变

###1.单一应用框架 
当网站流量很小时，只需一个应用，将所有功能如下单支付等都部署在一起，以减少部署节点和成本。 
**缺点**：单一的系统架构，使得在开发过程中，占用的资源越来越多，而且随着流量的增加越来越难以维护 

- 当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。
- 此时，用于简化增删改查工作量的 数据访问框架是关键。



###2.垂直应用框架(MVC) 
垂直应用架构解决了单一应用架构所面临的扩容问题，流量能够分散到各个子系统当中，且系统的体积可控，一定程度上降低了开发人员之间协同以及维护的成本，提升了开发效率。 
**缺点**：但是在垂直架构中相同逻辑代码需要不断的复制，不能复用。 

###3.分布式应用架构(RPC) 

远程过程调用

当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，服务之间相互调用。 

###4.流动计算架构(SOA) 

面向服务的架构（SOA）是一个组件模型，它将应用程序的不同功能单元（称为服务）进行拆分，并通过这些服务之间定义良好的接口和契约联系起来。

随着服务化的进一步发展，服务越来越多，服务之间的调用和依赖关系也越来越复杂，诞生了面向服务的架构体系(SOA)，也因此衍生出了一系列相应的技术，如对服务提供、服务调用、连接处理、通信协议、序列化方式、服务发现、服务路由、日志输出等行为进行封装的服务框架

- 当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。
- 此时，用于提高机器利用率的 资源调度和治理中心(SOA) 是关键。

## Dubbo是什么？

- 一款分布式服务框架
- 高性能和透明化的RPC（远程服务调用）方案
- SOA服务治理方案

###Dubbo架构

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/Dubbo/Dubbo%E6%9E%B6%E6%9E%84.png)

**Provider**: 暴露服务的服务提供方。 
**Consumer**: 调用远程服务的服务消费方。 
**Registry**: 服务注册与发现的注册中心。 
**Monitor**: 统计服务的调用次数和调用时间的监控中心。

###调用流程 
0. 服务容器负责启动，加载，运行服务提供者。 
1. 服务提供者在启动时，向注册中心注册自己提供的服务。 
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。 
3. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。 
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。 
5. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心

### Dubbo注册中心

通过将服务统一管理起来，可以有效地优化内部应用对服务发布/使用的流程和管理。服务注册中心可以通过特定协议来完成服务对外的统一。

**Dubbo提供的注册中心有如下几种类型可供选择**：

- Multicast注册中心
- Zookeeper注册中心
- Redis注册中心
- Simple注册中心

###Dubbo优缺点

#### 缺点：只支持JAVA语言

####优点

- 透明化的远程方法调用：像调用本地方法一样调用远程方法；只需简单配置，没有任何API侵入。
- 软负载均衡及容错机制 ：
  -可在内网替代nginx lvs等硬件负载均衡器。
- 服务注册中心自动注册 & 配置管理 ：
  -不需要写死服务提供者地址，注册中心基于接口名自动查询提供者ip。 
  -使用类似zookeeper等分布式协调服务作为服务注册中心，可以将绝大部分项目配置移入zookeeper集群。
- 服务接口监控与治理 
  -Dubbo-admin与Dubbo-monitor提供了完善的服务接口管理与监控功能，针对不同应用的不同接口，可以进行 多版本，多协议，多注册中心管理。

## Zookeeper是什么？

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

Dubbo建议使用Zookeeper作为服务的注册中心。



有关Zookeeper的配置：单机、集群、伪集群，还有Zookeeper的其他配置文件参数介绍、Zookeeper的一致性保证和Leeder选举，请参考[文章][/https://blog.csdn.net/u010752082/article/details/78225785]

