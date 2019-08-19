打算看看dubbo的源码，做个简单的读书笔记。

#Dubbo官方手册阅读

架构：
组件：1.container 2.provider 3.register 4.consumer 5.monitor  
值得注意的是，consumer与register之间采用了异步订阅模式，当服务上线或者下线，获取对应的地址。  
Dubbo目前支持4种注册中心,（multicast,zookeeper,redis,simple） 推荐使用Zookeeper注册中心  
[register简介](https://www.cnblogs.com/duanxz/p/3772765.html)  

从手册中摘要的主要流程：
调用关系说明
1.服务容器负责启动，加载，运行服务提供者。  
2.服务提供者在启动时，向注册中心注册自己提供的服务。  
3.服务消费者在启动时，向注册中心订阅自己所需的服务。  
4.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。  
5.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。  
6.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。  

值得注意的点：  
1.服务之间最后的调用，register在提供相应地址后不参与调用过程。  
2.monitor上报的数据先缓存在内存，隔一段时间在汇总上报。  
3.注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外。  
4.注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表。同时缓存也会出现数据不一致问题。  

dubbo提供多种配置方法，个人觉得最优雅的还是注解的形式。
其中关键主键如下：  
@Service（这个包名不是spring的，是阿里的）
@Reference

除了注册中心之外，还有一个配置中心，在服务治理（如lb），基础服务配置（就是property写的配置，如dubbo.properties中内容）提供配置。  
zk也可以承担这个角色（config分支存储）。  
值得注意的是，多个配置同时存在（如本地，外部），外部默认有更高的优先级，即会覆盖本地的配置。(后面提到，最高优先级是jvm -D参数的配置)  
（-Ddubbo.config-center.highest-priority=false）取消优先  

配置的编写也分级别：  
1.应用级别（对整个应用其作用）
2.服务级别（对应的服务起作用）
3.多配置级别（有多个对接的应用，如多个注册中心，多个配置中心）
摘抄的具体公式如下
## 应用级别
dubbo.{config-type}[.{config-id}].{config-item}={config-item-value}
## 服务级别
dubbo.service.{interface-name}[.{method-name}].{config-item}={config-item-value}
dubbo.reference.{interface-name}[.{method-name}].{config-item}={config-item-value}
## 多配置项
dubbo.{config-type}s.{config-id}.{config-item}={config-item-value}

## check="true"
用于启动时，检查服务是否可用（服务有可能已经是不可用的，但是本地缓存着之前的地址）

dubbo.reference.check=false，强制改变所有 reference 的 check 值，就算配置中有声明，也会被覆盖。
dubbo.consumer.check=false，是设置 check 的缺省值，如果配置中有显式的声明，如：<dubbo:reference check="true"/>，不会受影响。
dubbo.registry.check=false，前面两个都是指订阅成功，但提供者列表是否为空是否报错，如果注册订阅失败时，也允许启动，需使用此选项，将在后台定时重试。

## 集群容错
cluster进行设置，value如下  
1.Failover 失败重试，可以配置重试次数（retries）  
2.Failfast 快速失败，失败直接报错  
3.Failsafe 失败忽略  
4.Failback 后台记录失败请求，定时重发  
5.Forking 并行调用，同时请求多个，（通常是读操作，fork可以设置并行数）  
6.Broadcast 广播通知，逐一调用，任一一台报错就报错。  

## 负载均衡
loadbalance进行设置，value如下：
1.Random 随机。
2.RoundRobin  轮询。
3.LeastActive  最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。  
4.ConsistentHash 一致性 Hash，相同参数的请求总是发到同一提供者。  

[书签记录](http://dubbo.apache.org/zh-cn/docs/user/demos/subscribe-only.html)
