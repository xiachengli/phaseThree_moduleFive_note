####  

#### Spring Cloud概述

###### Spring Cloud是什么

Spring Cloud是一系列框架的有序集合，如服务注册发现、配置中心、消息总线、负载均衡、断路器等。它利用springboot的特性简化了分布式系统基础设施的开发。

###### 架构及其核心组件

核心组件：

| 组件           | 第一代Spring Cloud         | 第二代Spring Cloud                  |
| -------------- | -------------------------- | ----------------------------------- |
| 注册中心       | Netflix Eureka             | Nacos                               |
| 客户端负载均衡 | Netflix Ribbon             | Dubbo LB、Spring Cloud LoadBalancer |
| 熔断器         | Netflix Hystrix            | Sentinel                            |
| 网关           | Netflix Zuul               | Spring Cloud Gateway                |
| 配置中心       | Spring Cloud Config        | Nacos                               |
| 服务调用       | Netflix Feign              | Dubbo RPC                           |
| 消息驱动       | Spring Cloud Stream        |                                     |
| 链路追踪       | Spring Cloud Sleuth/Zipkin |                                     |
| 分布式事务     |                            | seata分布式事务方案                 |

spring Cloud体系结构：

![image-20200723132021389](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723132021389.png)

###### Spring Cloud VS Dubbo

Dubbo是一款高性能的远程调用框架，基于RPC调用。而Spring Cloud是微服务下的一系列解决方案，基于HTTP。

###### Spring Cloud VS Spring Boot

Spring Cloud利用了Spring Boot的特性使得开发人员能够快速上手实现微服务组件的开发

#### Eureka服务注册中心

分布式微服务中，服务注册中心用于存储服务提供者的信息。服务消费者通过主动查询或被动通知的方式获取服务提供者的信息。

注册中心一般性原理：

![image-20200723132709447](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723132709447.png)

1. 服务提供者启动

2. 服务提供者将相关信息主动注册到注册中心

3. 服务消费者获取服务注册信息

   poll模式：主动拉取

   push模式：被动推送

4. 服务消费者调用服务

Eureka架构：

![image-20200723133832339](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723133832339.png)

Eureka包含两个组件：Eureka Server和Eureka Client,client是一个java客户端，用于简化与Eureka Server的交互。Eureka Server提供服务发现的能力，各个微服务启动时，会通过Eureka Client向Eureka Server进行服务注册。

有关架构图的补充说明：

- 图中us-east-1c、us-east-1d、us-east-1e代表不同的区也就是不同的机房
- 图中每一个Eureka Server都是一个集群
- 图中Application Service作为服务提供者向Eureka Server中注册服务，Eureka Server接受到注册事件会在集群和分区中进行数据同步。Application Client作为消费端可以从注册中心获取服务提供者的信息
- 微服务启动后，会周期性地向Eureka Server发送心跳（默认周期为30秒）以续约自己的信息
- Eureka Server在一定时间内没有接收到某个微服务的心跳，Eureka Server将会注销该节点（默认90s）
- 每个Eureka Server同时也是Eureka Client，多个Eureka Server之间通过复制方式完成服务注册列表的同步
- Eureka Client会缓存Eureka Server中的信息

Eureka元数据详解：Eureka的元数据有两种，标准元数据和自定义元数据

标准元数据：主机名、IP地址、端口号等信息，这些信息都会被发布在服务注册表中，用于服务之间的调用

自定义元数据：可以使用eureka.instance.metadata-map配置

```yml
instance:    
	prefer-ip-address: true   
	metadata-map: 
	#自定义元数据<k,v>
		key: value
```

服务续约：服务每隔30秒会自动向注册中心发送心跳（也称续约或保活），如果没有续约。租约在90秒后到期，然后服务会被失败

```yml
#向Eureka服务注册中心注册服务
eureka:  
	instance:   
	#发送心跳时间间隔，默认30秒
	lease-renewal-interval-in-seconds: 30  
	#租约到期，服务失效时间，默认值90秒（服务超过90秒没有发生心跳，EurekaServer会将服务从列表移除）
    lease-expiration-duration-in-seconds: 90 

```

获取服务列表详情：每隔30秒服务会从注册中心拉去服务提供者列表，这个时间可以通过配置进行修改

```yml
eureka:
	client:
		registry-fetch-interval-seconds: 30
```

服务下线：当服务正常关闭时，会发送服务下线的REST请求给EurekaServer。服务中心接收到请求后，将服务置为下线状态

失效剔除：Eureka Server会定期进行健康检查，如果发现实例在配置的时间内没有发送心跳，则注销此实例

自我保护：在15分钟内超过85%的客户端节点都没有正常的心跳，那么Eureka server就会认为客户端与注册中心出现了网络故障，那Eureka Server会进入自我保护机制状态

当处于自我保护模式时：

1. 不会剔除任何服务实例（保证大多数服务依然可用）
2. Eureka Server依然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上。待网络稳定时，当前Eureka Server新的注册信息会被同步到其它节点上
3. 在Eureka Server工程中通过eureka.server.enable-self-preservation配置可关闭/开启自我保护模式（默认开启）

#### Ribbon客户端负载均衡

负载均衡分为服务器端负载均衡和客户端负载均衡。服务器端负载均衡指的是请求到达服务器之后由负载均衡器根据一定的算法将请求路由到某个目标服务器进行处理；客户端负载均衡指的是服务消费者拥有一个服务提供者地址列表，调用方在请求前通过一定的负载均衡算法选择一个目标服务器进行访问

![image-20200723143659735](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723143659735.png)

Ribbon负载均衡策略：

| 负载均衡策略                          | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| RoundRobinRule轮询策略                | 默认超过10次获取到的server都不可用的话，会返回一个空的server |
| RandomRule随机策略                    | 如果随机到的server为null或者不可用的话，会while不停的循环选取 |
| RetryRule重试策略                     | 一定时间内循环重试                                           |
| BestAvailableRule最小连接数策略       | 遍历服务列表，选取可用且连接数最小的一个server               |
| AvailabilityFilteringRule可用过滤策略 | 扩展了轮询策略。先通过轮询选取一个server，然后判断是否超时，当前连接数是否超限，判断成功再返回 |
| ZoneAvoidanceRule区域权衡策略（默认） | 扩展了轮询策略。除了过滤超时和链接数过多的server外，还会过滤掉不符合要求的zone区域里面的节点 |

修改负载均衡策略

```yml
#针对被调用方的微服务名称，不加就是全局生效
service-resume:   
	ribbon:    
		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule 
```

Ribbon负载均衡原理：

![image-20200723144948484](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723144948484.png)

####  Hystrix熔断器

什么是雪崩效应：微服务中，一个请求可能需要调用多个微服务接口才能完成，会形成了复杂的调用链路。如果对下游的某个微服务调用响应时间过长或者不可用，导致对上游的某个微服务的调用会占用越来越多的系统资源，进而引起系统崩溃的现象

雪崩效应的解决方案：

- 服务熔断

  当扇出链路的某个微服务不可用或者响应时间太长时，熔断该节点微服务的调用，进行服务的降级，快速返回错误的响应信息。当检测到该节点服务调用正常后，恢复调用链路

- 服务降级

  当某个服务熔断之后，此时客户端可以执行本地的fallback回调返回一个缺省值

- 服务限流

  限制服务的并发量，措施有：

  - 限制总并发数（比如数据库连接池、线程池）
  - 限制瞬时并发数（如nginx限制瞬时并发连接数）
  - 限制时间窗口内的平均速率（如Guava的RateLimiter、nginx的limit_req模块，限制每秒的平均速率）
  - 限制远程接口调用速率，限制MQ的消费速率

Hystrix时netflix开源的一个延迟和容错库，用于隔离访问远程系统，防止级联失败，从而提高系统的可用性和容错性。Hystrix主要通过以下几点实现延迟和容错

- 包裹请求：使用@HystrixCommand包裹对依赖的调用逻辑
- 跳闸机制：当某服务的错误率超过一定的阈值时，Hystrix可以跳闸，停止请求该服务一段时间
- 资源隔离：Hystrix为每个依赖都维护了一个小型的线程池（舱壁模式），如果该线程池已满，发往该依赖的请求将立即被拒绝而不是排队等待，从而加速失败判定
- 监控：Hystrix可以近乎实时地监控运行指标和配置地变化
- 回退机制：当请求失败、超时、被拒绝或断路器打开时，执行回退逻辑
- 自我修复：断路器打开一段时间后，会自动进入“半开”状态

####  Feign远程调用

feign时Netflix开发地一个轻量级RESTful的HTTP服务客户端

- feign可以帮助我们更加便捷，优雅的调用HTTP API
- SpringCloud对Feign进行了增强，使Feign支持SpringMVC注解

feign对负载均衡的支持：feign本身集成了Ribbon的依赖和自动配置，因此不需要额外引入依赖，可以通过ribbon.xx进行全局配置，也可以通过服务名.ribbon.xx来对指定服务进行细节配置

feign的超时时长

```yml
#针对的被调用方微服务名称,不加就是全局生效
ribbon:
  #请求连接超时时间
  ConnectTimeout: 2000
  #请求处理超时时间
  ##########################################Feign超时时长设置
  ReadTimeout: 3000
  #对所有操作都进行重试
  OkToRetryOnAllOperations: true
  ####根据如上配置，当访问到故障请求的时候，它会再尝试访问一次当前实例（次数由MaxAutoRetries配置），
  ####如果不行，就换一个实例进行访问，如果还不行，再换一次实例访问（更换次数由MaxAutoRetriesNextServer配置），
  ####如果依然不行，返回失败信息。
  MaxAutoRetries: 0 #对当前选中实例重试次数，不包括第一次调用
  MaxAutoRetriesNextServer: 0 #切换实例的重试次数
  NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #负载策略调整
# 开启Feign的熔断功能
feign:
  hystrix:
    enabled: false
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            ##########################################Hystrix的超时时长设置
            timeoutInMilliseconds: 15000
```

####  GateWay网关

Spring Cloud GateWay异步非阻塞，基于Reactor模型

一个请求到来时，网关根据一定的条件进行匹配，匹配成功后将请求转发到指定的服务地址，而在这个过程中，我们可以进行一些比较具体的控制（限流、日志、黑白名单）

网关的核心概念：

- 路由：由一个Id、一个目标URL、一系列的断言和Filter过滤器组成。如果断言为true则匹配该路由
- 断言：可以匹配Http请求中的所有内容（请求头、请求参数）
- 过滤器：一个标准的spring webFilter，使用过滤器，可以在请求之前或者之后执行业务逻辑

GateWay工作流程：

![image-20200723154650312](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723154650312.png)

GateWay核心逻辑：路由转发+执行过滤器链

application.yml部分配置内容：

```yml
server:
  port: 9002
eureka:
  client:
    serviceUrl: # eureka server的路径
      defaultZone: http://localhost:8761/eureka/ #把 eureka 集群中的所有 url 都填写了进来，也可以只写一台，因为各个 eureka server 可以同步注册表
  instance:
    #使用ip注册，否则会使用主机名注册了（此处考虑到对老版本的兼容，新版本经过实验都是ip）
    prefer-ip-address: true
    #自定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@
spring:
  application:
    name: server-gateway
  cloud:
    gateway:
      routes: # 路由可以有多个
        - id: server-user # 我们自定义的路由 ID，保持唯一
          #uri: http://127.0.0.1:8096  # 目标服务地址  自动投递微服务（部署多实例）  动态路由：uri配置的应该是一个服务名称，而不应该是一个具体的服务实例的地址
          uri: lb://server-user                                                                    # gateway网关从服务注册中心获取实例信息然后负载后路由
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/user/**
          filters:
            - StripPrefix=1
        - id: server-email      # 我们自定义的路由 ID，保持唯一
          uri: lb://server-email
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/email/**
          filters:
            - StripPrefix=1
        - id: server-code      # 我们自定义的路由 ID，保持唯一
          uri: lb://server-code
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/api/code/**
          filters:
            - StripPrefix=1

```

GateWay路由规则详解：

![image-20200723155147176](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723155147176.png)

时间点后匹配

```yml
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- After=2017-01-20T17:42:47.789-07:00[America/Denver]

```

时间点前匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Before=2017-01-20T17:42:47.789-07:00[America/Denver]
```

指定Cookie正则匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Cookie=chocolate,ch
```

请求路径正则匹配

```
spring:  
	cloud:    
		gateway:     
        	routes:       
            	- id: after_route        
                  uri: https://example.org          
                  predicates:            
                  	- Path=/api/user
```

GateWay过滤器

从生命周期时间点进行划分，过滤器可分为以下两种

| 生命周期时间点 | 作用                                             |
| -------------- | ------------------------------------------------ |
| pre            | 请求被路由之前调用。可实现统一认证、黑白名单过滤 |
| post           | 请求被路由之后执行。可实现日志记录、统计信息     |

从过滤器类型的角度分，过滤器分为GateWayFilter和GlobalFilter两种

#### Spring Cloud Config分布式配置中心

Spring Cloud Config是一个分布式配置管理中心，包含了Server端和Client端两个部分

server端：提供配置文件的存储，以接口的形式将配置文件的内容提供出去，通过@EnableConfigServer开启服务

Client端：通过接口获取配置数据并初始化自己的应用

![image-20200723161318764](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723161318764.png)

```yml
spring:
  application:
    name: server-config
  cloud:
    config:
      server:
        git:
          uri: https://github.com/xxx/server-config-repo.git #配置git服务地址
          username: ******@qq.com #配置git用户名
          password: ******** #配置git密码
          search-paths:
            - server-config-repo
      # 读取分支
      label: master
```



####  Spring Cloud Stream消息驱动组件

spring Cloud Stream是一个构建消息驱动微服务的框架。应用程序通过inputs（相当于消费者）或者outputs（相当于生产者）来与Stream中的binder对象交互。Binder对象是用来屏蔽底层MQ细节的，它负责与具体的消息中间件进行交互

![image-20200723171242176](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200723171242176.png)

优点：屏蔽底层的消息中间件的细节差异，统一了MQ的编程模型，降低了学习、开发、维护MQ的成本

stream消息通信方式：Stream中的消息通信方式遵循了发布-订阅模式。当一条消息被投递到消息中间件后，它会通过共享的topic主题进行广播，消息消费者在订阅的主题中接收并触发业务逻辑处理

topic：spring cloud stream中的一个抽象概念，代表发布共享消息给消费者的地方。在不同的消息中间件中，topic对应着不同的概念，比如：rabbitMQ中它对应了exchange、kafka中对应了topic

spring cloud stream的相关注解

| 注解            | 描述                          |
| --------------- | ----------------------------- |
| @Input          | 标识输入通道                  |
| @Output         | 标识输出通道                  |
| @StreamListener | 监听队列                      |
| @EnableBinding  | 将Channel和Exchange绑定在一起 |

#### 分布式链路追踪技术

###### 分布式链路追踪技术问题场景

在微服务中，一次请求少则经过几次服务调用，多则几十或几百次服务调用，由此引发了一系列问题，如下：

- 如何动态展示服务的调用链路
- 如何进行故障排查
- 如何分析调用链路中的瓶颈节点

基于以上问题，分布式链路技术应运而生

###### 分布式链路追踪技术核心思想

本质：记录日志

微服务架构中，针对请求处理的调用链可以展现为一棵树，如下图所示：

![image-20200804084212531](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804084212531.png)

相关概念

| 名词     | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| Trace    | 服务追踪的追踪单元。从请求抵达被追踪系统的边界开始，到被追踪系统向客户返回响应的这整个过程 |
| Trace ID | 唯一标识一条链路。当请求到达系统的入口端点时，链路追踪框架为该请求创建一个唯一的跟踪标识Trace ID |
| Span     | 日志数据结构。记录一些日志信息，比如时间戳，spanId           |
| Span ID  | 唯一标识span                                                 |

一个Trace由一个或多个Span组成，每个Span都有一个SpanID，Span中会记录Trace ID，同时还有ParentId指向另一个span的spanID表明调用关系

每一个Span都有一个唯一标识Span ID，若干个有序的span组成了一个trace

Span中也抽象出了另外一个概念，叫做事件，核心事件如下：

- CS client send/start：客户端发出一个请求。描述的是一个span的开始
- SR server received/start：服务端接收请求。SR-CS为请求发送的网络延迟
- SS server send/finish：服务端发送应答。SS-SR为服务端消耗时间
- CR cient received/finished：客户端接收应答。CR-SS表示响应的网络延迟

###### Sleuth + Zipkin实践

引入依赖坐标

```xml
<!--分布式链路追踪-->
<dependency>  
	<groupId>org.springframework.cloud</groupId>  
	<artifactId>spring-cloud-starter-sleuth</artifactId> 
</dependency>

<!--zipkin-->
<dependency>  
	<groupId>org.springframework.cloud</groupId>  
	<artifactId>spring-cloud-starter-zipkin</artifactId> 
</dependency>


```

配置

```yml
spring:
  application:
    name: xx
  cloud:
    gateway:
      routes: # 路由可以有多个
        - id: service-oauth-router # 我们自定义的路由 ID，保持唯一
          uri: lb://lagou-cloud-oauth-server                                                                    # gateway网关从服务注册中心获取实例信息然后负载后路由
          predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
            - Path=/oauth/**
  zipkin:
    base-url: http://127.0.0.1:9411 # zipkin server的请求地址
    sender:
      # web 客户端将踪迹日志数据通过网络请求的方式传送到服务端，另外还有配置
      # kafka/rabbit 客户端将踪迹日志数据传递到mq进行中转
      type: web
  sleuth:
    sampler:
      # 采样率 1 代表100%全部采集 ，默认0.1 代表10% 的请求踪迹数据会被采集
      # 生产环境下，请求量非常大，没有必要所有请求的踪迹数据都采集分析，对于网络包括server端压力都是比较大的，可以配置采样率采集一定比例的请求的踪迹数据进行分析即可
      probability: 1
#分布式链路追踪
logging:
  level:
    org.springframework.cloud.sleuth: debug
```

Zipkin包括Zipkin server和Zipkin Client两部分，Zipkin Server是一个单独的服务，Zipkin Client就是具体的微服务，以下构建zipkin server服务

1.引入zipkin server坐标

```xml
<!--zipkin-server的依赖坐标-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-server</artifactId>
            <version>2.12.3</version>
            <exclusions>
                <!--排除掉log4j2的传递依赖，避免和springboot依赖的日志组件冲突-->
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-log4j2</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--zipkin-server ui界面依赖坐标-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-ui</artifactId>
            <version>2.12.3</version>
        </dependency>


        <!--zipkin针对mysql持久化的依赖-->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-storage-mysql</artifactId>
            <version>2.12.3</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
```

2.在入口类上添加@EnableZipkinServer注解

3.配置

```yaml
server:
  port: 9411
management:
  metrics:
    web:
      server:
        auto-time-requests: false # 关闭自动检测
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/zipkin?useUnicode=true&characterEncoding=utf-8&useSSL=false&allowMultiQueries=true
    username: root
    password: 123456
    druid:
      initialSize: 10
      minIdle: 10
      maxActive: 30
      maxWait: 50000
# 指定zipkin持久化介质为mysql
zipkin:
  storage:
    type: mysql
```

4.追踪数据持久化到mysql

- mysql中创建名为zipkin的数据库，并执行如下sql语句

```sql
--- Copyright 2015-2019 The OpenZipkin Authors --- Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except -- in compliance with the License. You may obtain a copy of the License at --- http://www.apache.org/licenses/LICENSE-2.0 --- Unless required by applicable law or agreed to in writing, software distributed under the License -- is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express -- or implied. See the License for the specific language governing permissions and limitations under -- the License. -
CREATE TABLE IF NOT EXISTS zipkin_spans (  `trace_id_high` BIGINT NOT NULL DEFAULT 0 COMMENT 'If non zero, this means the trace uses 128 bit traceIds instead of 64 bit',  `trace_id` BIGINT NOT NULL,  `id` BIGINT NOT NULL,  `name` VARCHAR(255) NOT NULL,  `remote_service_name` VARCHAR(255),  `parent_id` BIGINT,  `debug` BIT(1),  `start_ts` BIGINT COMMENT 'Span.timestamp(): epoch micros used for endTs query and to implement TTL',  `duration` BIGINT COMMENT 'Span.duration(): micros used for minDuration and maxDuration query',  PRIMARY KEY (`trace_id_high`, `trace_id`, `id`)
) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;
ALTER TABLE zipkin_spans ADD INDEX(`trace_id_high`, `trace_id`) COMMENT 'for getTracesByIds'; ALTER TABLE zipkin_spans ADD INDEX(`name`) COMMENT 'for getTraces and getSpanNames'; ALTER TABLE zipkin_spans ADD INDEX(`remote_service_name`) COMMENT 'for getTraces and getRemoteServiceNames'; ALTER TABLE zipkin_spans ADD INDEX(`start_ts`) COMMENT 'for getTraces ordering and range';
CREATE TABLE IF NOT EXISTS zipkin_annotations (  `trace_id_high` BIGINT NOT NULL DEFAULT 0 COMMENT 'If non zero, this means the trace uses 128 bit traceIds instead of 64 bit',  `trace_id` BIGINT NOT NULL COMMENT 'coincides with zipkin_spans.trace_id',  `span_id` BIGINT NOT NULL COMMENT 'coincides with zipkin_spans.id',  `a_key` VARCHAR(255) NOT NULL COMMENT 'BinaryAnnotation.key or Annotation.value if type == -1',  `a_value` BLOB COMMENT 'BinaryAnnotation.value(), which must be smaller than 64KB',  `a_type` INT NOT NULL COMMENT 'BinaryAnnotation.type() or -1 if Annotation',  `a_timestamp` BIGINT COMMENT 'Used to implement TTL; Annotation.timestamp or zipkin_spans.timestamp',  `endpoint_ipv4` INT COMMENT 'Null when Binary/Annotation.endpoint is null',  `endpoint_ipv6` BINARY(16) COMMENT 'Null when Binary/Annotation.endpoint is null, or no IPv6 address',  `endpoint_port` SMALLINT COMMENT 'Null when Binary/Annotation.endpoint is null',  `endpoint_service_name` VARCHAR(255) COMMENT 'Null when Binary/Annotation.endpoint is null' ) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;
ALTER TABLE zipkin_annotations ADD UNIQUE KEY(`trace_id_high`, `trace_id`, `span_id`, `a_key`, `a_timestamp`) COMMENT 'Ignore insert on duplicate'; ALTER TABLE zipkin_annotations ADD INDEX(`trace_id_high`, `trace_id`, `span_id`) COMMENT 'for joining with zipkin_spans'; ALTER TABLE zipkin_annotations ADD INDEX(`trace_id_high`, `trace_id`) COMMENT 'for getTraces/ByIds'; ALTER TABLE zipkin_annotations ADD INDEX(`endpoint_service_name`) COMMENT 'for getTraces and getServiceNames'; ALTER TABLE zipkin_annotations ADD INDEX(`a_type`) COMMENT 'for getTraces and autocomplete values'ץ
ALTER TABLE zipkin_annotations ADD INDEX(`a_key`) COMMENT 'for getTraces and autocomplete values'; ALTER TABLE zipkin_annotations ADD INDEX(`trace_id`, `span_id`, `a_key`) COMMENT 'for dependencies job';
CREATE TABLE IF NOT EXISTS zipkin_dependencies (  `day` DATE NOT NULL,  `parent` VARCHAR(255) NOT NULL,  `child` VARCHAR(255) NOT NULL,  `call_count` BIGINT,  `error_count` BIGINT,  PRIMARY KEY (`day`, `parent`, `child`) ) ENGINE=InnoDB ROW_FORMAT=COMPRESSED CHARACTER SET=utf8 COLLATE utf8_general_ci;
 
```

- 注入事务管理器

  ```Java
  @Bean 
  public PlatformTransactionManager txManager(DataSource dataSource) {  return new DataSourceTransactionManager(dataSource); }
  ```



#### 微服务统一认证方案Spring Cloud OAuth2 + JWT

###### OAuth2开放授权协议/标准

OAuth是一个开放协议，允许用户授权第三方应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方应用或分享他们数据的所有内容

###### 什么情况下需要使用OAuth2

1. 第三方授权登录
2. 单点登录

###### OAuth2 Token授权方式

1. 授权码

2. 密码式（提供用户名+密码）

3. 隐藏式

4. 客户端凭证

   授权码模式使用了回调模式，是最复杂的授权方式，微博等第三方登录就是这种模式。我们常用的是密码式

###### Spring Cloud OAuth2介绍

Spring Cloud OAuth2是Spring Cloud体系对OAuth2协议的实现，可以用来做多个微服务的统一认证和授权。通过向统一认证授权服务发送某个类型的grant_type进行集中认证和授权，从而获得access_token，而这个token是受其他微服务信任的

###### Spring Cloud OAuth2构建统一认证服务

- 新建maven项目

- pom.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
      <parent>
          <artifactId>lagou-parent</artifactId>
          <groupId>com.lagou.edu</groupId>
          <version>1.0-SNAPSHOT</version>
      </parent>
      <modelVersion>4.0.0</modelVersion>
  
      <artifactId>lagou-cloud-oauth-server-9999</artifactId>
  
  
  
      <dependencies>
          <!--导入Eureka Client依赖-->
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
          </dependency>
  
  
          <!--导入spring cloud oauth2依赖-->
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-oauth2</artifactId>
              <exclusions>
                  <exclusion>
                      <groupId>org.springframework.security.oauth.boot</groupId>
                      <artifactId>spring-security-oauth2-autoconfigure</artifactId>
                  </exclusion>
              </exclusions>
          </dependency>
          <dependency>
              <groupId>org.springframework.security.oauth.boot</groupId>
              <artifactId>spring-security-oauth2-autoconfigure</artifactId>
              <version>2.1.11.RELEASE</version>
          </dependency>
          <!--引入security对oauth2的支持-->
          <dependency>
              <groupId>org.springframework.security.oauth</groupId>
              <artifactId>spring-security-oauth2</artifactId>
              <version>2.3.4.RELEASE</version>
          </dependency>
  
  
  
  
          <dependency>
              <groupId>mysql</groupId>
              <artifactId>mysql-connector-java</artifactId>
          </dependency>
          <dependency>
              <groupId>com.alibaba</groupId>
              <artifactId>druid-spring-boot-starter</artifactId>
              <version>1.1.10</version>
          </dependency>
          <!--操作数据库需要事务控制-->
          <dependency>
              <groupId>org.springframework</groupId>
              <artifactId>spring-tx</artifactId>
          </dependency>
          <dependency>
              <groupId>org.springframework</groupId>
              <artifactId>spring-jdbc</artifactId>
          </dependency>
  
  
          <dependency>
              <groupId>com.lagou.edu</groupId>
              <artifactId>lagou-service-common</artifactId>
              <version>1.0-SNAPSHOT</version>
          </dependency>
      </dependencies>
  
  </project>
  ```

- 认证服务器配置类

  ```java
  package com.lagou.edu.config;
  
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.HttpMethod;
  import org.springframework.security.authentication.AuthenticationManager;
  import org.springframework.security.jwt.crypto.sign.MacSigner;
  import org.springframework.security.jwt.crypto.sign.SignatureVerifier;
  import org.springframework.security.jwt.crypto.sign.Signer;
  import org.springframework.security.oauth2.config.annotation.configurers.ClientDetailsServiceConfigurer;
  import org.springframework.security.oauth2.config.annotation.web.configuration.AuthorizationServerConfigurerAdapter;
  import org.springframework.security.oauth2.config.annotation.web.configuration.EnableAuthorizationServer;
  import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerEndpointsConfigurer;
  import org.springframework.security.oauth2.config.annotation.web.configurers.AuthorizationServerSecurityConfigurer;
  import org.springframework.security.oauth2.provider.ClientDetailsService;
  import org.springframework.security.oauth2.provider.client.JdbcClientDetailsService;
  import org.springframework.security.oauth2.provider.token.*;
  import org.springframework.security.oauth2.provider.token.store.InMemoryTokenStore;
  import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
  import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;
  
  import javax.sql.DataSource;
  import java.util.ArrayList;
  import java.util.List;
  
  
  /**
   * 当前类为Oauth2 server的配置类（需要继承特定的父类 AuthorizationServerConfigurerAdapter）
   */
  @Configuration
  @EnableAuthorizationServer  // 开启认证服务器功能
  public class OauthServerConfiger extends AuthorizationServerConfigurerAdapter {
  
  
      @Autowired
      private AuthenticationManager authenticationManager;
  
      @Autowired
      private LagouAccessTokenConvertor lagouAccessTokenConvertor;
  
  
      private String sign_key = "lagou123"; // jwt签名密钥
  
  
      /**
       * 认证服务器最终是以api接口的方式对外提供服务（校验合法性并生成令牌、校验令牌等）
       * 那么，以api接口方式对外的话，就涉及到接口的访问权限，我们需要在这里进行必要的配置
       * @param security
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
          super.configure(security);
          // 相当于打开endpoints 访问接口的开关，这样的话后期我们能够访问该接口
          security
                  // 允许客户端表单认证
                  .allowFormAuthenticationForClients()
                  // 开启端口/oauth/token_key的访问权限（允许）
                  .tokenKeyAccess("permitAll()")
                  // 开启端口/oauth/check_token的访问权限（允许）
                  .checkTokenAccess("permitAll()");
      }
  
      /**
       * 客户端详情配置，
       *  比如client_id，secret
       *  当前这个服务就如同QQ平台，拉勾网作为客户端需要qq平台进行登录授权认证等，提前需要到QQ平台注册，QQ平台会给拉勾网
       *  颁发client_id等必要参数，表明客户端是谁
       * @param clients
       * @throws Exception
       */
      @Override
      public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
          super.configure(clients);
  
  
          // 从内存中加载客户端详情
  
          /*clients.inMemory()// 客户端信息存储在什么地方，可以在内存中，可以在数据库里
                  .withClient("client_lagou")  // 添加一个client配置,指定其client_id
                  .secret("abcxyz")                   // 指定客户端的密码/安全码
                  .resourceIds("autodeliver")         // 指定客户端所能访问资源id清单，此处的资源id是需要在具体的资源服务器上也配置一样
                  // 认证类型/令牌颁发模式，可以配置多个在这里，但是不一定都用，具体使用哪种方式颁发token，需要客户端调用的时候传递参数指定
                  .authorizedGrantTypes("password","refresh_token")
                  // 客户端的权限范围，此处配置为all全部即可
                  .scopes("all");*/
  
          // 从数据库中加载客户端详情
          clients.withClientDetails(createJdbcClientDetailsService());
  
      }
  
      @Autowired
      private DataSource dataSource;
  
      @Bean
      public JdbcClientDetailsService createJdbcClientDetailsService() {
          JdbcClientDetailsService jdbcClientDetailsService = new JdbcClientDetailsService(dataSource);
          return jdbcClientDetailsService;
      }
  
  
      /**
       * 认证服务器是玩转token的，那么这里配置token令牌管理相关（token此时就是一个字符串，当下的token需要在服务器端存储，
       * 那么存储在哪里呢？都是在这里配置）
       * @param endpoints
       * @throws Exception
       */
      @Override
      public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
          super.configure(endpoints);
          endpoints
                  .tokenStore(tokenStore())  // 指定token的存储方法
                  .tokenServices(authorizationServerTokenServices())   // token服务的一个描述，可以认为是token生成细节的描述，比如有效时间多少等
                  .authenticationManager(authenticationManager) // 指定认证管理器，随后注入一个到当前类使用即可
                  .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST);
      }
  
  
      /*
          该方法用于创建tokenStore对象（令牌存储对象）
          token以什么形式存储
       */
      public TokenStore tokenStore(){
          //return new InMemoryTokenStore();
          // 使用jwt令牌
          return new JwtTokenStore(jwtAccessTokenConverter());
      }
  
      /**
       * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
       * 在这里，我们可以把签名密钥传递进去给转换器对象
       * @return
       */
      public JwtAccessTokenConverter jwtAccessTokenConverter() {
          JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
          jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
          jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
          jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);
  
          return jwtAccessTokenConverter;
      }
  
  
  
  
      /**
       * 该方法用户获取一个token服务对象（该对象描述了token有效期等信息）
       */
      public AuthorizationServerTokenServices authorizationServerTokenServices() {
          // 使用默认实现
          DefaultTokenServices defaultTokenServices = new DefaultTokenServices();
          defaultTokenServices.setSupportRefreshToken(true); // 是否开启令牌刷新
          defaultTokenServices.setTokenStore(tokenStore());
  
          // 针对jwt令牌的添加
          defaultTokenServices.setTokenEnhancer(jwtAccessTokenConverter());
  
          // 设置令牌有效时间（一般设置为2个小时）
          defaultTokenServices.setAccessTokenValiditySeconds(20); // access_token就是我们请求资源需要携带的令牌
          // 设置刷新令牌的有效时间
          defaultTokenServices.setRefreshTokenValiditySeconds(259200); // 3天
  
          return defaultTokenServices;
      }
  }
  
  ```

- 认证服务器安全配置类

  ```Java
  package com.lagou.edu.config;
  
  import com.lagou.edu.service.JdbcUserDetailsService;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.autoconfigure.quartz.QuartzProperties;
  import org.springframework.cglib.proxy.NoOp;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.security.authentication.AuthenticationManager;
  import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
  import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
  import org.springframework.security.core.userdetails.User;
  import org.springframework.security.core.userdetails.UserDetails;
  import org.springframework.security.crypto.password.NoOpPasswordEncoder;
  import org.springframework.security.crypto.password.PasswordEncoder;
  
  import java.util.ArrayList;
  
  
  /**
   * 该配置类，主要处理用户名和密码的校验等事宜
   */
  @Configuration
  public class SecurityConfiger extends WebSecurityConfigurerAdapter {
  
      @Autowired
      private PasswordEncoder passwordEncoder;
  
      @Autowired
      private JdbcUserDetailsService jdbcUserDetailsService;
  
      /**
       * 注册一个认证管理器对象到容器
       */
      @Bean
      @Override
      public AuthenticationManager authenticationManagerBean() throws Exception {
          return super.authenticationManagerBean();
      }
  
  
      /**
       * 密码编码对象（密码不进行加密处理）
       * @return
       */
      @Bean
      public PasswordEncoder passwordEncoder() {
          return NoOpPasswordEncoder.getInstance();
      }
  
      /**
       * 处理用户名和密码验证事宜
       * 1）客户端传递username和password参数到认证服务器
       * 2）一般来说，username和password会存储在数据库中的用户表中
       * 3）根据用户表中数据，验证当前传递过来的用户信息的合法性
       */
      @Override
      protected void configure(AuthenticationManagerBuilder auth) throws Exception {
          // 在这个方法中就可以去关联数据库了，当前我们先把用户信息配置在内存中
          // 实例化一个用户对象(相当于数据表中的一条用户记录)
          /*UserDetails user = new User("admin","123456",new ArrayList<>());
          auth.inMemoryAuthentication()
                  .withUser(user).passwordEncoder(passwordEncoder);*/
  
          auth.userDetailsService(jdbcUserDetailsService).passwordEncoder(passwordEncoder);
      }
  }
  
  ```

  ###### 资源服务器

  - 资源服务配置类

    ```Java
    package com.lagou.edu.config;
    
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.security.config.annotation.web.builders.HttpSecurity;
    import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
    import org.springframework.security.config.http.SessionCreationPolicy;
    import org.springframework.security.jwt.crypto.sign.MacSigner;
    import org.springframework.security.jwt.crypto.sign.RsaVerifier;
    import org.springframework.security.jwt.crypto.sign.SignatureVerifier;
    import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
    import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
    import org.springframework.security.oauth2.config.annotation.web.configurers.ResourceServerSecurityConfigurer;
    import org.springframework.security.oauth2.provider.token.RemoteTokenServices;
    import org.springframework.security.oauth2.provider.token.TokenStore;
    import org.springframework.security.oauth2.provider.token.store.JwtAccessTokenConverter;
    import org.springframework.security.oauth2.provider.token.store.JwtTokenStore;
    
    @Configuration
    @EnableResourceServer  // 开启资源服务器功能
    @EnableWebSecurity  // 开启web访问安全
    public class ResourceServerConfiger extends ResourceServerConfigurerAdapter {
    
        private String sign_key = "lagou123"; // jwt签名密钥
    
        @Autowired
        private LagouAccessTokenConvertor lagouAccessTokenConvertor;
    
        /**
         * 该方法用于定义资源服务器向远程认证服务器发起请求，进行token校验等事宜
         * @param resources
         * @throws Exception
         */
        @Override
        public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
    
            /*// 设置当前资源服务的资源id
            resources.resourceId("autodeliver");
            // 定义token服务对象（token校验就应该靠token服务对象）
            RemoteTokenServices remoteTokenServices = new RemoteTokenServices();
            // 校验端点/接口设置
            remoteTokenServices.setCheckTokenEndpointUrl("http://localhost:9999/oauth/check_token");
            // 携带客户端id和客户端安全码
            remoteTokenServices.setClientId("client_lagou");
            remoteTokenServices.setClientSecret("abcxyz");
    
            // 别忘了这一步
            resources.tokenServices(remoteTokenServices);*/
    
    
            // jwt令牌改造
            resources.resourceId("autodeliver").tokenStore(tokenStore()).stateless(true);// 无状态设置
        }
    
    
        /**
         * 场景：一个服务中可能有很多资源（API接口）
         *    某一些API接口，需要先认证，才能访问
         *    某一些API接口，压根就不需要认证，本来就是对外开放的接口
         *    我们就需要对不同特点的接口区分对待（在当前configure方法中完成），设置是否需要经过认证
         *
         * @param http
         * @throws Exception
         */
        @Override
        public void configure(HttpSecurity http) throws Exception {
            http    // 设置session的创建策略（根据需要创建即可）
                    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                    .and()
                    .authorizeRequests()
                    .antMatchers("/autodeliver/**").authenticated() // autodeliver为前缀的请求需要认证
                    .antMatchers("/demo/**").authenticated()  // demo为前缀的请求需要认证
                    .anyRequest().permitAll();  //  其他请求不认证
        }
    
    
    
    
        /*
           该方法用于创建tokenStore对象（令牌存储对象）
           token以什么形式存储
        */
        public TokenStore tokenStore(){
            //return new InMemoryTokenStore();
    
            // 使用jwt令牌
            return new JwtTokenStore(jwtAccessTokenConverter());
        }
    
        /**
         * 返回jwt令牌转换器（帮助我们生成jwt令牌的）
         * 在这里，我们可以把签名密钥传递进去给转换器对象
         * @return
         */
        public JwtAccessTokenConverter jwtAccessTokenConverter() {
            JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
            jwtAccessTokenConverter.setSigningKey(sign_key);  // 签名密钥
            jwtAccessTokenConverter.setVerifier(new MacSigner(sign_key));  // 验证时使用的密钥，和签名密钥保持一致
            jwtAccessTokenConverter.setAccessTokenConverter(lagouAccessTokenConvertor);
            return jwtAccessTokenConverter;
        }
    
    }
    
    ```

  ###### JWT改造统一认证授权中心的令牌存储机制

  当资源服务器和授权服务不在一起时资源服务使用RemoteTokenServices远程请求授权服务验证token，如果访问量较大将会影响系统的性能。为了解决这个问题，令牌采用JWT格式解决。用户认证通过会得到一个JWT令牌，JWT令牌中已经包括了用户相关的信息，客户端只需要携带JWT访问资源服务，资源服务根据事先约定的算法完成令牌校验，无需每次都请求认证服务完成授权

  1. 什么是JWT

     JSON Web Token是一个开放的行业标准，用于在通信双方传递json对象，传递的信息经过数字签名可以被验证和信任

  2. JWT令牌结构

     JWT令牌由三部分组成，每部分中间使用.分隔

     - Header

       头部包括令牌的类型及使用的哈希算法，例如

       ```json
       //将该内容使用Base64Url编码，得到JWT令牌的第一部分
       {  
       	"alg": "HS256",  
           "typ": "JWT"
       }
       ```

     - Payload

       负载，内容也是一个json对象，它是存放有效信息的地方，可以存放JWT提供的现成字段，比如：iss签发者、exp过期时间戳、sub面向的用户等，也可自定义字段。此部分不建议存放敏感信息，因为此部分可以解码还原原始内容

       ```json
       //此部分使用Base64Url编码，得到JWT令牌的第二部分
       {  
           "sub": "1234567890",  
           "name": "John Doe",  
           "iat": 1516239022 
       }
       
       ```

     - Signature

       签名，此部分用于防止JWT内容被篡改

```
//使用Base64Url将前两部分进行编码，编码后使用.连接成字符串，最后使用header中声明的签名算法进行签名
HMACSHA256(  base64UrlEncode(header) + "." +   base64UrlEncode(payload),   secret)
 
```



#### 第二代Spring Cloud 核心组件

###### Nacos

Nacos是阿里巴巴开源的一个针对微服务架构中的服务发现、配置管理和服务管理平台

1. Nacos功能特性

   - 服务发现与健康检查
   - 动态配置管理
   - 动态DNS服务
   - 服务和元数据管理、动态的服务权重调整、动态服务优雅下线

2. Nacos数据模型

   | 名词      | 说明                                                       |
   | --------- | ---------------------------------------------------------- |
   | namespace | 命名空间。对不同环境进行隔离                               |
   | group     | 分组。将若干个服务或配置集归为一组，通常一个系统归为一个组 |
   | service   | 某一具体微服务                                             |
   | dataId    | 配置文件                                                   |

   namespace+group+service如同maven中的GAV坐标，GAV坐标是为了锁定jar，而这里是为了锁定服务

   namespace+group+dataId如同maven中的GAV坐标，GAV坐标是为了锁定jar，而这里是为了锁定配置文件

3. Nacos服务的分级模型

   ![image-20200804113249796](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804113249796.png)

注册中心

- 在父pom中引入SCA依赖

  ```xml
   <dependencyManagement>
          <dependencies>
              <!--SCA -->
              <dependency>
                  <groupId>com.alibaba.cloud</groupId>
                  <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                  <version>2.1.0.RELEASE</version>
                  <type>pom</type>
                  <scope>import</scope>
              </dependency>
          </dependencies>
          <!--SCA -->
      </dependencyManagement>
  ```

- 在服务提供者工程中引入nacos客户端依赖

  ```xml
    <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
          </dependency>
  ```

- application.yml添加nacos配置信息

  ```yml
  server:
    port: 8081
  
  spring:
    application:
      name: server-user
    cloud:
      nacos:
        discovery:
          server-addr: 127.0.0.1:8848
  ```

配置中心

- 添加依赖

  ```xml
   <!--nacos config client 依赖-->
          <dependency>
              <groupId>com.alibaba.cloud</groupId>
              <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
          </dependency>
  ```

- application.yml添加配置

  ```yaml
  server:
    port: 8081
  
  spring:
    application:
      name: server-user
    cloud:
      nacos:
      	# 注册中心
        discovery:
          server-addr: 127.0.0.1:8848
            # 配置中心
        config:
            server-addr: 127.0.0.1:8848
            # 锁定server端的配置文件（读取它的配置项）
            namespace: 74507098-3f1d-4b48-823e-217078b3122e
            group: DEFAULT_GROUP
            file-extension: yaml   #默认properties
  ```

- 通过Spring Cloud原生注解@RefreshScope实现配置自动更新

###### Sentinel

sentinel是一个面向云原生服务的流量控制、服务熔断降级组件

1. Sentinel特性

   - 丰富的应用场景：秒杀、流量控制、消息削峰填谷、实时熔断不可用服务

   - 完备的实时监控

   - 广泛的开源生态：sentinel提供开箱即用的与其他开源框架库的整合模块，例如与spring cloud、dubbo的整合，只需要引入相应的依赖并进行简单的配置即可快速接入sentinel

   - 完善的SPI扩展点：sentinel提供简单易用、完善的SPI扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源

     ![image-20200804133730963](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804133730963.png)

2. sentinel部署

   下载地址：https://github.com/alibaba/Sentinel/releases

   启动： java -jar sentinel-dashboard-1.7.1.jar &

   用户名/密码：sentinel/sentinel

3. 接入sentinel

   - pom.xml引入依赖

     ```xml
      <!--sentinel 核心环境 依赖-->
             <dependency>
                 <groupId>com.alibaba.cloud</groupId>
                 <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
             </dependency>
     ```

   - application.yml修改

     ```yaml
     server:
       port: 9002
     spring:
       application:
         name: server-gateway
       cloud:
         nacos:
           discovery:
             server-addr: 127.0.0.1:8848
         sentinel:
           transport:
             dashboard: 127.0.0.1:8080
             port: 8719
         gateway:
           routes: # 路由可以有多个
             - id: server-user # 我们自定义的路由 ID，保持唯一
               #uri: http://127.0.0.1:8096  # 目标服务地址  自动投递微服务（部署多实例）  动态路由：uri配置的应该是一个服务名称，而不应该是一个具体的服务实例的地址
               uri: lb://server-user                                                                    # gateway网关从服务注册中心获取实例信息然后负载后路由
               predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
                 - Path=/api/user/**
               filters:
                 - StripPrefix=1
             - id: server-email      # 我们自定义的路由 ID，保持唯一
               uri: lb://server-email
               predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
                 - Path=/api/email/**
               filters:
                 - StripPrefix=1
             - id: server-code      # 我们自定义的路由 ID，保持唯一
               uri: lb://server-code
               predicates:                                         # 断言：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默 认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。
                 - Path=/api/code/**
               filters:
                 - StripPrefix=1
     
     ```

4. Sentinel关键概念

   | 概念名称 | 描述                                                         |
   | :------- | ------------------------------------------------------------ |
   | 资源     | 可以是java应用程序中的任何内容，例如，由应用程序提供的服务或一段代码。请求的API接口也是资源 |
   | 规则     | 流量控制规则、熔断降级规则以及系统保护规则，所有规则都可以动态实时调整 |

5. 流量控制

   资源名：默认请求路径

   针对来源：sentinel可以针对调用者进行限流、填写微服务名称、默认default(不区分来源)

   阈值类型：QPS每秒钟请求数量，当调用该资源的QPS达到阈值时进行限流、线程数：当调用该资源的线程数达到阈值的时候进行限流

   是否集群：是否集群限流

   流控模式：直接/关联/链路

   直接：资源调用达到限流条件时，直接限流

   关联：关联的资源调用达到阈值的时候限流自己

   链路：只记录指定链路上的流量

   流控效果：快速失败/warm up/排队等待

   快速失败：直接失败，抛出异常

   warm up：根据冷加载因子（默认3）的值，从阈值/冷加载因子，经过预热时长，才达到设置的QPS阈值

   排队等待：匀速排队，让请求匀速通过，阈值类型必须设置为QPS，否则无效

6. 降级策略

   熔断触发后，时间窗口内拒绝请求，时间窗口后就恢复

   - RT平均响应时间

     当1s内持续进入大于等于5个请求，平均响应时间超过阈值（以ms为单位），那么在接下的时间窗口之内，对这个方法的调用都会自动熔断。sentinel默认的RT上限是4900ms，超过此阈值的都会算作4900ms，可通过启动配置项-Dcsp.sentinel.statistic.max.rt=xxx来配置

     ![image-20200804141311822](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804141311822.png)

   - 异常比例

     当资源的每秒请求量大于等于5，并且每秒异常总数占通过量的比值超过阈值之后，资源进入降级状态，即在接下来的时间窗口内，对这个方法的调用都会自动地返回。异常比例地阈值范围是[0.0,1.0]，代表0%-100%

     ![image-20200804141617101](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804141617101.png)

   - 异常数

     当资源近1分钟地异常数目超过阈值之后会进行熔断。注意由于统计时间窗口是分钟级别地，若timeWindow小于60s，则结束熔断状态后仍可能再进入熔断状态

   ![image-20200804141909207](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200804141909207.png)

   

7. Sentinel自定义兜底逻辑

   - API接口处配置@SentinelResource

     ```java
      @GetMapping("/checkState/{userId}")
         // @SentinelResource注解类似于Hystrix中的@HystrixCommand注解
         @SentinelResource(value = "findResumeOpenState",blockHandlerClass = SentinelHandlersClass.class,
                 blockHandler = "handleException",fallbackClass = SentinelHandlersClass.class,fallback = "handleError")
         public Integer findResumeOpenState(@PathVariable Long userId) {
             // 模拟降级：
             /*try {
                 Thread.sleep(1000);
             } catch (InterruptedException e) {
                 e.printStackTrace();
             }*/
             // 模拟降级：异常比例
             //int i = 1/0;
             Integer defaultResumeState = resumeServiceFeignClient.findDefaultResumeState(userId);
             return defaultResumeState;
         }
     ```

     

   - 自定义逻辑类

   ```java
   package com.lagou.edu.config;
   
   import com.alibaba.csp.sentinel.slots.block.BlockException;
   
   public class SentinelHandlersClass {
   
       // 整体要求和当时Hystrix一样，这里还需要在形参最后添加BlockException参数，用于接收异常
       // 注意：方法是静态的
       public static Integer handleException(Long userId, BlockException blockException) {
           return -100;
       }
   
       public static Integer handleError(Long userId) {
           return -500;
       }
   
   }
   
   ```

8. 基于nacos实现sentinel规则持久化

   - pom.xml添加依赖

     ```xml
     <!--sentinel支持采用nacos作为规则配置数据源，引入该适配依赖-->
     <dependency>  <groupId>com.alibaba.csp</groupId>  <artifactId>sentinel-datasource-nacos</artifactId> </dependency>
     
     ```

     

   - application.yml配置数据源

     ```yml
     server:
       port: 8098
     spring:
       application:
         name: lagou-service-autodeliver
       cloud:
         nacos:
           discovery:
             server-addr: 127.0.0.1:8848,127.0.0.1:8849,127.0.0.1:8850
     
         sentinel:
           transport:
             dashboard: 127.0.0.1:8080 # sentinel dashboard/console 地址
             port: 8719   #  sentinel会在该端口启动http server，那么这样的话，控制台定义的一些限流等规则才能发送传递过来，
                           #如果8719端口被占用，那么会依次+1
           # Sentinel Nacos数据源配置，Nacos中的规则会自动同步到sentinel流控规则中
           datasource:
             # 自定义的流控规则数据源名称
             flow:
               nacos:
                 server-addr: ${spring.cloud.nacos.discovery.server-addr}
                 data-id: ${spring.application.name}-flow-rules
                 groupId: DEFAULT_GROUP
                 data-type: json
                 rule-type: flow  # 类型来自RuleType类
             # 自定义的降级规则数据源名称
             degrade:
               nacos:
                 server-addr: ${spring.cloud.nacos.discovery.server-addr}
                 data-id: ${spring.application.name}-degrade-rules
                 groupId: DEFAULT_GROUP
                 data-type: json
                 rule-type: degrade  # 类型来自RuleType类
     ```

   - Nacos server中添加对应规则配置集

     流量控制规则配置集

     ```json
     [   
         {          
             "resource":"findResumeOpenState",       
             "limitApp":"default",       
             "grade":1,      
             "count":1,       
             "strategy":0,      
             "controlBehavior":0,       
             "clusterMode":false    
         } 
     ]  
     
     ```

     resource：资源名称

     limitApp：来源应用

     grade：阈值类型，0线程数、1QPS

     count：单机阈值

     strategy：流控模式，0直接、1关联、2链路

     controlBehavior：流控效果，0快速失败、1warm up、2排队等待

     clusterMode：true/false是否集群

     降级规则配置集

     ```json
     [   
         {          
             "resource":"findResumeOpenState",     
             "grade":2,       
             "count":1,       
             "timeWindow":5   
         } 
     ] 
     ```

     resource：资源名称

     grade：降级策略，0RT、1异常比例、2异常数

     count：阈值

     timeWindow：时间窗口

###### 