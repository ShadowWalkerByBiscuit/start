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

## 几个比较有意思的配置：  
1.结果缓存 cache  
2.回声测试 直接强转EchoService，然后调用echo方法。  
3.隐式参数 RpcContext.getContext().setAttachment("index", "1"); // 隐式传参，后面的远程调用都会隐式将这些参数发送到服务器端，类似cookie，用于框架集成，不建议常规业务使用  RpcContext.getContext().getAttachment("index");   

## 异步调用
首先要求服务提供方，定义接口返回值为CompletableFuture<>。  
服务使用方，需要增加回调，
```java
// 调用直接返回CompletableFuture
CompletableFuture<String> future = asyncService.sayHello("async call request");
// 增加回调
future.whenComplete((v, t) -> {
    if (t != null) {
        t.printStackTrace();
    } else {
        System.out.println("Response: " + v);
    }
});
// 早于结果输出
System.out.println("Executed before response return.");

//使用RpcContext的方式 此调用会立即返回null
asyncService.sayHello("world");
// 拿到调用的Future引用，当结果返回后，会被通知和设置到此Future
CompletableFuture<String> helloFuture = RpcContext.getContext().getCompletableFuture();
//或者换一种方式调用
CompletableFuture<String> future = RpcContext.getContext().asyncCall(
    () -> {
        asyncService.sayHello("oneway call request1");
    }
);

future.get(); 
```

sent="true" 等待消息发出，消息发送失败将抛出异常。
sent="false" 不等待消息发出，将消息放入 IO 队列，即刻返回。

## 参数回调
在注解里面配置callbacks methods，添加服务完成后的回调操作。

## 本地存根
stub 在服务提供方进行一次包装，做容错

## 本地伪装
mock 用于服务降级

##  延迟暴露  
delay=-1 表示等待spring初始化结束之后才向外暴露，设置其他数据为启动延迟的毫秒数，最新版本是默认在spring启动完成后才暴露

# 优雅停机
前提是使用kill pid，如果有-9 就不行。
使用的是JDK 的 ShutdownHook （通过Runtime类的addShutdownHook方法）;  

## 服务治理  
通过telnet访问dubbo服务，可以通过命令行进行操作。

## 可扩展点
dubbo不是基于java原生的spi技术，只是核心思想是一致的，做到接口的动态扩展，典型的例子就是lb的配置实现。
相较与java的ServerLoader，dubbo对应的扫描加载工具叫 ExtentionLoader。
默认的扫描路径为：  
META-INF/dubbo/internal  
META-INF/dubbo  
META-INF/services  
java写文件的规则是在MEAT-INF/Service下创建一个文件，文件名为接口的完整名（包括全部的路径名，如org.apache.spi.Robot）
内容为实现了该接口的实体类的全路径（可以存在多个实现类）
dubbo的不同点在于文件内容是k-v形式，以提供根据参数化来实现动态配置。如 bumblebee = org.apache.spi.Bumblebee

其中主要的注解为@SPI(RandomLoadBalance.NAME)，@Adaptive("loadbalance")
@SPI的值为默认实现，@Adaptive的值为url中传递的k-v参数中的key，对应的value就是代替@SPI中的默认值，也就是上面提到文件中的key，决定了具体加载的实现类。

## 扩展点的实现
通过ExtensionLoader.getExtensionLoader(xxx)获取实例，然后根据xxx去找寻要加载的实体类，首先查找缓存中是否已经加载过，如果没有在缓存中，则创建。
创建的过程是通过扫描文件中对应xxx的实现类的全路径，运用反射机制创建实例，创建完成之后还要注入依赖。接下来要包装成wrapper类。

## 查询实体类过程
首先解析了@SPI注解，把默认实现的value值缓存起来，然后依次去扫描默认的文件夹，然后解析里面的k-v对，然后将v指定的实现类加载并加入缓存。

## @Adaptive 
可以出现在类或者方法上。主要还是在方法上，如果某个某个实现类被 Adaptive 注解修饰了，那么该类就会被赋值给 cachedAdaptiveClass 变量。这样就会在加载的时候，访问缓存不为空，直接返回。这种情况是@SPI，跟@Adaptive都没有值的时候的时候，由实现类提供默认实现。Dubbo 要求该接口至少有一个方法被 Adaptive 注解修饰。这样才存在扩展加载的意义。Dubbo 不会为没有标注 Adaptive 注解的方法生成代理逻辑，对于该种类型的方法，仅会生成一句抛出异常的代码。
@Adaptive({client, transporter})值可以是一个数组，第一个没有找到，则第二个起作用。

## 服务导出
scope = none，不导出服务
scope != remote，导出到本地
scope != local，导出到远程
最重要的是Invoker 的创建  
[书签](http://dubbo.apache.org/zh-cn/docs/source_code_guide/export-service.html)


