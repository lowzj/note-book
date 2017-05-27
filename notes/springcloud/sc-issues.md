# SpringCloud问题整理

#### 使用Feign, Hystrix进行服务间调用时，第一次调用总是超时

##### GitHub Issues
* https://github.com/spring-cloud/spring-cloud-netflix/issues/1864
* https://github.com/spring-cloud/spring-cloud-netflix/issues/1921

##### Solution
添加如下配置

```ini
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000
ribbon.ConnectTimeout=1000
ribbon.ReadTimeout=1000
```

默认的hystrix的超时是1秒，但是我测试的时候发现，因为feign使用的是懒加载，第一次调用时，会初始化各种bean，速度很慢，大概2秒左右，所以默认1秒总是超时。以上是全局配置，如果要单独配置每个feign，可以如下配置：

```ini
hystrix.command.<hystrixCommandName>.execution.isolation.thread.timeoutInMilliseconds=3000
<feignClientName>.ribbon.ConnectTimeout=1000
<feignClientName>.ribbon.ReadTimeout=1000
```

其中`hystrixCommandName`为`FeignClassName#methodSignature`，由此可以做到方法级别的隔离；`feignClientName`为注解`FeignClient`中属性`name`的值。


#### 使用Archaius动态调整运行时配置

添加以下JVM参数：

```ini
-Darchaius.dynamicPropertyFactory.registerConfigWithJMX=true
```
