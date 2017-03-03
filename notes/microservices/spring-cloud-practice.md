🔨 Spring Cloud 实践
===================

<pre align="center">
config, consul, ribbon, feign, hystrix, zuul, metrics, tracing
</pre>

* [Spring Cloud Consul](#-spring-cloud-consul)

Spring Cloud Config
-------------------
> 统一的配置管理服务，基于spring-cloud-config，适用于Spring Boot Application。
> 目的是将程序与配置完全分离，方便查看更改，便于定位由配置文件导致的问题，同时有利于服务的自动化部署。
> 服务代码中去除默认配置文件，将所需配置交由 ConfigSerer 管理，ConfigServer 配置文件的持久化系统是github。服务集成 ConfigClient，启动时，向 ConfigServer 请求配置文件，ConfigServer 从github拉取下最新的配置文件，然后寻找到指定的配置文件，根据spring configuration的规则拼装好所有配置文件，然后返回给ConfigCient，结果是JSON格式。

远程仓库模式

```ini
spring.application.name=ConfigServer
management.context-path=/management
health.config.enabled=false

spring.cloud.config.server.health.enabled=false
spring.cloud.config.server.git.timeout=120
spring.cloud.config.server.git.cloneOnStart=true
spring.cloud.config.server.git.searchPaths=**
spring.cloud.config.server.git.uri=https://github.com/lowzj/config-server-test.git
spring.cloud.config.server.git.username=xxx
spring.cloud.config.server.git.password=yyy
```

本地仓库模式

```ini
spring.application.name=ConfigServer
management.context-path=/management
health.config.enabled=false

spring.cloud.config.server.health.enabled=false
spring.cloud.config.server.git.searchPaths=**
spring.cloud.config.server.git.uri=file:///tmp/config-server-test/
```

Spring Cloud Consul
-------------------

* consul cluster 搭建
* 服务健康检测

Feign & Ribbon
-------------------

Hystrix
-------

Zuul
----

Metrics
-------

Tracing: Pinpoint
-----------------
