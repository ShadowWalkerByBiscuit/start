软件的三种基本模式：单机，cs，bs(browser)。

### tomcat源码阅读  
基础架构关系图：  
Tomcat->Server(最多一个)->service(多个)->Connection(有三种，默认使用效率最高的APR)->Container   
Connector->ProtocolHandler（实际实现交给了adapter，Nio为例，实际上是有多种协议的）->Endpoint->Accepter(处理socket网络连接处理)->poller(注册Selector)->worker(具体处理)  
Container->Engine+Host(多个，不同的host代表不同的服务)+Context(在同一Host下，不同context表示不同的子服务)+Wrapper(servlet的包装)  

Tomcat的main函数在Boostrap类中。

因为涉及到XML文件的解析，所以使用量Apache的开源主键Digester。

在container的流程中，进入到Engine的阶段，通过pipeline的形式，可以向其中注册handler用于过程中的处理。

Tomcat对于各个组件的管理，是通过lifeCircle接口来实现了生命周期的管理。LifecycleEvent是生命周期中发布事件，通过事件驱动（监听者模式）来调用逻辑，
lifeCircle->lifeCircleBase->lifeCircleSupport.

这里提到了JMX，可以简单的理解为java对于一些资源，组件的容器管理，相对于spring的容器管理要更底层，例如硬件，网络，协议等等这方面的资源进行调度跟统一
转换适配，也就是常见的MBean。  

所有容器的转态转换（如新建、初始化、启动、停止等）都是由外到内，由上到下进行，即先执行父容器的状态转换及相关操作，
然后再执行子容器的转态转换，这个过程是层层迭代执行的。  

标准启动过程：  
1-调用方调用容器父类LifecycleBase的start方法，LifecycleBase的start方法主要完成一些所有容器公共抽象出来的动作；  
2-LifecycleBase的start方法先将容器状态改为LifecycleState.STARTING_PREP，然后调用具体容器的startInternal方法实现，
此startInternal方法用于对容器本身真正的初始化；  
3-具体容器的startInternal方法会将容器状态改为LifecycleState.STARTING，容器如果有子容器，会调用子容器的start方法启动子容器；  
4-容器启动完毕，LifecycleBase会将容器的状态更改为启动完毕，即LifecycleState.STARTED。  
