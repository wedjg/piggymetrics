[![Build Status](https://travis-ci.org/sqshq/PiggyMetrics.svg?branch=master)](https://travis-ci.org/sqshq/PiggyMetrics)
[![codecov.io](https://codecov.io/github/sqshq/PiggyMetrics/coverage.svg?branch=master)](https://codecov.io/github/sqshq/PiggyMetrics?branch=master)
[![GitHub license](https://img.shields.io/github/license/mashape/apistatus.svg)](https://github.com/sqshq/PiggyMetrics/blob/master/LICENCE)
[![Join the chat at https://gitter.im/sqshq/PiggyMetrics](https://badges.gitter.im/sqshq/PiggyMetrics.svg)](https://gitter.im/sqshq/PiggyMetrics?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

# Piggy Metrics

**处理个人财务的简单方法**

T这是一个用来演示[微服务架构模式](http://martinfowler.com/microservices/)的[概念应用证明](https://piggymetrics.tk)，包括了Spring Boot, Spring Cloud和 Docker.
顺便说一下，它有一个非常整洁的用户界面。

![](https://cloud.githubusercontent.com/assets/6069066/13864234/442d6faa-ecb9-11e5-9929-34a9539acde0.png)
![Piggy Metrics](https://cloud.githubusercontent.com/assets/6069066/13830155/572e7552-ebe4-11e5-918f-637a49dff9a2.gif)

## 功能性服务

PiggyMetrics 被分解为3个微服务. 它们都是独立部署的应用程序，围绕特定的业务域组织。

<img width="880" alt="Functional services" src="https://cloud.githubusercontent.com/assets/6069066/13900465/730f2922-ee20-11e5-8df0-e7b51c668847.png">

#### 账户服务
包含常规用户输入逻辑和验证: 收入/支出项目、储蓄和账户设置。

方法	| 路径	| 描述	| 用户权限	| 提供用户界面
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /accounts/{account}	| 获取指定账户数据	|  | 	
GET	| /accounts/current	| 获取当前账户数据	| × | ×
GET	| /accounts/demo	| 获取演示账户数据（预填收入/支出项目等）	|   | 	×
PUT	| /accounts/current	| 保存当前账户数据	| × | ×
POST	| /accounts/	| 注册新账户	|   | ×


#### 统计服务
对主要统计参数执行计算并捕获每个账户的时间. 数据点包含值，标准化的基础货币和时间段. 此数据用于跟踪帐户生命周期中的现金流动态。

方法	| 路径	| 描述	| 用户权限	| 提供用户界面
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /statistics/{account}	| 获取指定帐户统计信息	          |  | 	
GET	| /statistics/current	| 获取当前帐户统计信息	| × | × 
GET	| /statistics/demo	| 获取演示帐户统计信息	|   | × 
PUT	| /statistics/{account}	| 为指定帐户创建或更新时序数据点	|   | 


#### 通知服务
存储用户联系信息和通知设置 (比如提醒和备份频率). 计划工作人员从其他服务收集所需信息，并向订阅的客户发送电子邮件。

方法	| 路径	| 描述	| 用户权限	| 提供用户界面
------------- | ------------------------- | ------------- |:-------------:|:----------------:|
GET	| /notifications/settings/current	| 获取当前帐户通知设置	| × | ×	
PUT	| /notifications/settings/current	| 保存当前帐户通知设置	| × | ×

#### 注意
- 每个微服务都有自己的数据库，因此无法绕过API直接访问持久性数据。
- 在这个项目中，我使用MongoDB作为每个服务的主数据库。拥有一个多语言持久性体系结构也是有意义的。 (选择最适合服务要求的DB类型).
- 服务到服务的通信比较简单: 微服务之间的通信仅使用同步的REST API. 现实系统中的常见做法是使用交互样式的组合. 例如，执行同步GET 请求以检索数据，并通过消息代理使用异步方法来创建/更新操作，以分离服务和缓冲消息。不过，这带来了[最终一致性](http://martinfowler.com/articles/microservice-trade-offs.html#consistency)。

## 基础设施服务
分布式系统中有许多常见的模式，可以帮助我们使所描述的核心服务工作。 [Spring cloud](http://projects.spring.io/spring-cloud/) 提供了增强SpringBoot应用程序行为以实现这些模式的强大工具。我会简单介绍一下.
<img width="880" alt="Infrastructure services" src="https://cloud.githubusercontent.com/assets/6069066/13906840/365c0d94-eefa-11e5-90ad-9d74804ca412.png">
### 配置服务
[Spring Cloud Config](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html)为分布式系统提供水平可扩展的集中式配置服务。 它使用了一个可插入的存储库层，该层目前支持 local storage、Git和Subversion。 

在这个项目中，我使用`native profile`，它只从本地类路径加载配置文件。你可以在[Config service resources](https://github.com/sqshq/PiggyMetrics/tree/master/config/src/main/resources)看到`share`目录。现在，当通知服务请求其配置时， Spring Cloud Config 会返回在所有客户端应用程序之间共享的`shared/notification-service.yml` 和`shared/application.yml` .

##### 客户端使用
只需要使用`spring-cloud-starter-config` 依赖来构建 Spring Boot 应用，自动化配置会完成其他步骤。

现在，你无须添加其他嵌入的属性在你的应用中。只需要提供带有应用名称的`bootstrap.yml`和配置服务路径：
```yml
spring:
  application:
    name: notification-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
```

##### 使用Spring Cloud Config, 你可以动态地改变应用的配置。 
例如, [EmailService bean](https://github.com/sqshq/PiggyMetrics/blob/master/notification-service/src/main/java/com/piggymetrics/notification/service/EmailServiceImpl.java) 被`@RefreshScope`注解.这意味着，你可以修改邮件的文本和主题而无须重新构建并重启通知服务。

首先，在配置服务中修改必需的属性。然后，对通知服务执行刷新请求：
`curl -H "Authorization: Bearer #token#" -XPOST http://127.0.0.1:8000/notifications/refresh`

当然，你也可以使用此仓库 [webhooks to automate this process](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_push_notifications_and_spring_cloud_bus)

##### 注意
- 但是，动态刷新有一些限制. `@RefreshScope` 不会与`@Configuration` 的类一起工作并且不会作用于`@Scheduled` 方法；
- `fail-fast` 属性意味着如果连接不上配置服务， Spring Boot 应用将无法立即启动；
- 以下是重要的 [安全说明](https://github.com/sqshq/PiggyMetrics#security)；

### 授权服务
授权责任被完全提取到单独的服务器，该服务器为后端资源服务授予[OAuth2令牌](https://tools.ietf.org/html/rfc6749)。授权服务用于用户授权，也用于在外围环境中进行安全的机器到机器通信。

在项目中，我使用 [`密码凭据`](https://tools.ietf.org/html/rfc6749#section-4.3) 的授权类型为用户授权 (因为它只被本地Piggmetrics用户界面使用) 和 [`客户端凭据`](https://tools.ietf.org/html/rfc6749#section-4.4) 为微服务授权；

SpringCloudSecurity提供了方便的注释和自动配置，使得从服务器和客户端实现这一点非常容易。 你可以在[文档](http://cloud.spring.io/spring-cloud-security/spring-cloud-security.html)中学习更多以及在[身份验证服务器代码](https://github.com/sqshq/PiggyMetrics/tree/master/auth-service/src/main/java/com/piggymetrics/auth)中查找配置详情；

从客户机的角度来看，一切工作方式都与基于会话的传统授权完全相同。您可以从请求中检索`principal`对象，使用基于表达式的访问控制和`@preauthorize`来检查用户角色和其他内容。

PiggyMetrics中的每个客户端（帐户服务、统计服务、通知服务和浏览器）都有其作用域: `server` 用于后端，`ui`用于前端. 所以我们也可以保护控制器不受外部访问的影响，例如：

``` java
@PreAuthorize("#oauth2.hasScope('server')")
@RequestMapping(value = "accounts/{name}", method = RequestMethod.GET)
public List<DataPoint> getStatisticsByAccountName(@PathVariable String name) {
	return statisticsService.findByAccountName(name);
}
```

### API Gateway
如您所见，有三个核心服务，它们向客户机公开外部API。在现实世界中，这个数字可以非常迅速地增长，也可以增加整个系统的复杂性。实际上，数百个服务可能涉及到一个复杂网页的呈现。

理论上，客户机可以直接向每个微服务发出请求。但显然，这个选项存在挑战和局限性，比如需要知道所有终端地址，分别对每一条信息执行HTTP请求，在客户端合并结果。另一个问题是后端可能使用的非Web友好协议。

通常更好的方法是使用API网关。它是进入系统的单一入口点，用于通过将请求路由到适当的后端服务或通过调用多个后端服务并[聚合结果](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html)来处理请求。此外，它还可以用于身份验证、洞察、stress and canary测试、服务迁移、静态响应处理、主动流量管理。

像Netflix opensourced [这样的边缘服务](http://techblog.netflix.com/2013/06/announcing-zuul-edge-service-in-cloud.html)， 现在有了SpringCloud，我们可以通过一个`@enablezuulproxy`注释来启用它。在这个项目中，我使用zuul存储静态内容（UI应用程序），并将请求路由到适当的微服务。以下是通知服务基于前缀的简单路由配置：

```yml
zuul:
  routes:
    notification-service:
        path: /notifications/**
        serviceId: notification-service
        stripPrefix: false

```

这意味着以`/notifications`开头的所有请求都将路由到通知服务。如您所见，没有硬编码地址。zuul使用[服务发现](https://github.com/sqshq/PiggyMetrics/blob/master/README.md#service-discovery)机制来定位通知服务实例，以及下面描述的[断路器和负载均衡](https://github.com/sqshq/PiggyMetrics/blob/master/README.md#http-client-load-balancer-and-circuit-breaker)。

### 服务发现

另一种常见的体系结构模式是服务发现。它允许自动检测服务实例的网络位置，由于自动伸缩、故障和升级，服务实例可以动态分配地址。

服务发现的关键部分是注册表。我在这个项目中使用Netflix Eureka。当客户机负责确定可用服务实例的位置（使用注册表服务器）和跨实例的负载平衡请求时，Eureka是客户端发现模式的一个很好的例子。

使用Spring引导，您可以使用`Spring Cloud Starter Eureka Server`依赖项、`@EnableEurekaServer`注解和简单的配置属性轻松构建Eureka注册表。
客户端支持使用注解`@EnableDiscoveryClient`和带有应用名的配置文件`bootstrap.yml`来启用：
``` yml
spring:
  application:
    name: notification-service
```

现在，在应用程序启动时，它将注册到Eureka服务器并提供元数据，如主机和端口、健康指示器URL、主页等。Eureka从属于服务的每个实例接收心跳消息。如果心跳在可配置的时间表上失败，则实例将从注册表中删除。

此外，Eureka提供了一个简单的界面，您可以在其中跟踪正在运行的服务和许多可用实例： `http://localhost:8761`

### 负载均衡，断路器和 Http客户端

Netflix OSS提供了另一套很棒的工具。

#### Ribbon
Ribbon是一个客户端负载均衡器，它为您提供了对HTTP和TCP客户端行为的大量控制。与传统的负载均衡器相比，不需要为每一次在线调用增加跃点——您可以直接联系所需的服务。

开箱即用，它与SpringCloud和服务发现本地集成。 [Eureka Client](https://github.com/sqshq/PiggyMetrics#service-discovery) 提供可用服务器的动态列表，以便功能区可以在它们之间进行平衡。

#### Hystrix
Hystrix是[断路器模式](http://martinfowler.com/bliki/CircuitBreaker.html)的实现，它通过网络访问依赖项来控制延迟和故障。 其主要思想是在具有大量微服务的分布式环境中停止级联故障。这有助于快速失败并尽快恢复-自我修复的容错系统的重要方面。

除了断路器控制之外，使用hystrix，还可以添加一个回退方法，在主命令失败时调用该方法以获取默认值。

此外，hystrix为每个命令生成有关执行结果和延迟的度量，我们可以使用这些度量来[监视系统行为](https://github.com/sqshq/PiggyMetrics#monitor-dashboard)。

#### Feign
Feign是一个声明性HTTP客户机，它与Ribbon和Hystrix无缝集成。实际上，有了一个`spring-cloud-starter-feign`依赖项和`@EnableFeignClients`注解，您就拥有了一套完整的负载均衡器、断路器和HTTP客户机，以及明智的随时可用的默认配置。

以下是账户中心的例子:

``` java
@FeignClient(name = "statistics-service")
public interface StatisticsServiceClient {

	@RequestMapping(method = RequestMethod.PUT, value = "/statistics/{accountName}", consumes = MediaType.APPLICATION_JSON_UTF8_VALUE)
	void updateStatistics(@PathVariable("accountName") String accountName, Account account);

}
```

- 你所需要的只是一个接口
- 你可以在Spring MVC controller和Feign 方法之间共享`@RequestMapping` 部分;
- 上面的示例仅指定所需的服务ID-`statistics-service`，这归功于通过Eureka进行的自动发现（但显然，您可以使用特定的URL访问任何资源）。

### 监控面板

在这个项目配置中，每个带有Hystrix的微服务都通过SpringCloud总线（使用AMQP代理）将metrics 推送到turbine。监控项目只是一个带有[Turbine](https://github.com/Netflix/Turbine)和[Hystrix仪表板](https://github.com/Netflix/Hystrix/tree/master/hystrix-dashboard)的Spring boot应用程序。  

见下文[如何启动和运行](https://github.com/sqshq/PiggyMetrics#how-to-run-all-the-things).

让我们看看我们的系统在负载下的行为：帐户服务调用统计服务，它以不同的模拟延迟响应。响应超时阈值设置为1秒。

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14194375/d9a2dd80-f7be-11e5-8bcc-9a2fce753cfe.png">

<img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127349/21e90026-f628-11e5-83f1-60108cb33490.gif">	| <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127348/21e6ed40-f628-11e5-9fa4-ed527bf35129.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127346/21b9aaa6-f628-11e5-9bba-aaccab60fd69.gif"> | <img width="212" src="https://cloud.githubusercontent.com/assets/6069066/14127350/21eafe1c-f628-11e5-8ccd-a6b6873c046a.gif">
--- |--- |--- |--- |
| `0 ms delay` | `500 ms delay` | `800 ms delay` | `1100 ms delay`
| 行为良好的系统。吞吐量约为每秒22个请求。统计服务中的活动线程数很少。平均服务时间约为50 ms。 | 活动线程的数量正在增加。我们可以看到紫色数量的线程池拒绝，因此大约30-40%的错误，但电路仍然是关闭的。 | 半开状态：指令失效率大于50%，断路器投入。在睡眠窗口时间段之后，下一个请求被释放。 | 100%的请求失败。电路现在永久断开。睡眠后重试不会再次关闭电路，因为单个请求太慢。

### 日志分析

当试图在分布式环境中识别问题时，集中式日志记录非常有用。 ElasticSearch、Logstash和Kibana Stack允许您轻松搜索和分析日志、利用率和网络活动数据。
准备就绪的Docker配置在[另一个项目](http://github.com/sqshq/ELK-docker)有描述.

### 分布式跟踪

分析分布式系统中的问题可能很困难，例如，跟踪从一个微服务传播到另一个微服务的请求。试图找出请求如何在系统中传播是一个很大的挑战，特别是如果您对微服务的实现没有任何了解。即使有日志记录，也很难判断哪个操作与单个请求相关。

[Spring Cloud Sleuth](https://cloud.spring.io/spring-cloud-sleuth/) 通过为分布式跟踪提供支持来解决这个问题。它将两种类型的ID添加到日志中：traceId和spanId。spanId表示基本的工作单元，例如发送HTTP请求。 traceId包含一组构成树状结构的跨度。例如，对于分布式大数据存储，跟踪可能由一个Put请求形成。对每个操作使用traceid和spanid，我们知道应用程序在何时何地处理请求，这使得读取日志更加容易。

日志如下，注意slf4j mdc中的`[appname，traceid，spanid，exportable]`条目：

```text
2018-07-26 23:13:49.381  WARN [gateway,3216d0de1384bb4f,3216d0de1384bb4f,false] 2999 --- [nio-4000-exec-1] o.s.c.n.z.f.r.s.AbstractRibbonCommand    : The Hystrix timeout of 20000ms for the command account-service is set lower than the combination of the Ribbon read and connect timeout, 80000ms.
2018-07-26 23:13:49.562  INFO [account-service,3216d0de1384bb4f,404ff09c5cf91d2e,false] 3079 --- [nio-6000-exec-1] c.p.account.service.AccountServiceImpl   : new account has been created: test
```

- *`appname`*: 记录属性`spring.application.name`范围的应用程序的名称 
- *`traceId`*: 这是分配给单个请求、作业或操作的ID
- *`spanId`*: 发生的特定操作的ID
- *`exportable`*: 是否应将日志导出到 [Zipkin](https://zipkin.io/)

## 安全

高级安全配置超出了这个概念验证项目的范围。 为了更真实地模拟真实的系统，考虑使用HTTPS、JCE密钥库来加密微服务密码和配置服务器属性内容。 (详情请看 [文档](http://cloud.spring.io/spring-cloud-config/spring-cloud-config.html#_security)).

## 基础设施自动化

与部署单片应用程序相比，部署具有相互依赖性的微服务要复杂得多。拥有完全自动化的基础设施是很重要的。我们可以通过持续交付方法实现以下好处：

- 随时发布软件的能力
- 任何构建最终都可能是一个版本
- 一次性构建artifacts-按需部署

下面是一个简单的连续交付工作流，在本项目中实现：

<img width="880" src="https://cloud.githubusercontent.com/assets/6069066/14159789/0dd7a7ce-f6e9-11e5-9fbb-a7fe0f4431e3.png">

在这个[配置](https://github.com/sqshq/PiggyMetrics/blob/master/.travis.yml), Travis CI为每次成功的Git推送构建标记图像。因此，在[Docker Hub](https://hub.docker.com/r/sqshq/)上的每个微服务和旧的映像都有“最新”映像，并用git commit hash标记。如果需要，很容易部署它们中的任何一个并快速回滚。

## 如何运行所有的东西?

记住，您将启动8个Spring引导应用程序、4个MongoDB实例和RabbitMQ。确保您的计算机上有可用的`4 GB`RAM。 但是，您始终可以运行重要的服务：网关、注册表、配置、认证服务和帐户服务。

#### Before you start
- 安装Docker和Docker Compose.
- 导出环境变量: `CONFIG_SERVICE_PASSWORD`, `NOTIFICATION_SERVICE_PASSWORD`, `STATISTICS_SERVICE_PASSWORD`, `ACCOUNT_SERVICE_PASSWORD`, `MONGODB_PASSWORD` (make sure they were exported: `printenv`)
- 确保构建项目: `mvn package [-DskipTests]`

#### 生产模式
在此模式下，所有最新的图像将从Docker Hub中提取。
只需要复制`docker-compose.yml`并点击`docker-compose up`

#### 开发模式
如果您想自己构建图像（例如代码中有一些更改），您必须克隆所有存储库并使用Maven构建工件。然后执行`docker-compose -f docker-compose.yml -f docker-compose.dev.yml up`

`docker-compose.dev.yml` 继承 `docker-compose.yml` 还可以在本地构建图像并公开所有容器端口以方便开发。

#### 重要端口
- http://localhost:80 - Gateway
- http://localhost:8761 - Eureka Dashboard
- http://localhost:9000/hystrix - Hystrix Dashboard (Turbine stream link: `http://turbine-stream-service:8080/turbine/turbine.stream`)
- http://localhost:15672 - RabbitMq management (default login/password: guest/guest)

#### Notes
所有Spring引导应用程序都需要运行[配置中心](https://github.com/sqshq/PiggyMetrics#config-service)才能启动。但是我们可以同时启动所有容器，因为`depends_on`docker-compose选项。

此外，服务发现机制在所有应用程序启动后需要一段时间。在实例、Eureka服务器和客户端的本地缓存中都具有相同的元数据之前，客户端无法发现任何服务，因此可能需要3次心跳。默认心跳周期为30秒。

## 欢迎贡献!

PiggyMetrics是开源的, 并且非常感谢你的帮助. 随时提出建议并实施改进。
