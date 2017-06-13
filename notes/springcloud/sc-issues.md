# SpringCloud问题整理

### 使用Feign, Hystrix进行服务间调用时，第一次调用总是超时

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


---

### 使用Archaius动态调整运行时配置

* pom依赖，集成`archaius`和`actuator`(提供管理接口)

    ```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-archaius</artifactId>
</dependency>
    ```

* 代码集成，可以参考[这里](http://cloud.spring.io/spring-cloud-static/spring-cloud.html#_external_configuration_archaius)

	```java
class ArchaiusTest {
	DynamicStringProperty myprop = DynamicPropertyFactory.getInstance()
		.getStringProperty("my.prop");

	void doSomething() {
		OtherClass.someMethod(myprop.get());
	}
}
	```

* 启动服务时，添加以下JVM参数：

    ```ini
-Darchaius.dynamicPropertyFactory.registerConfigWithJMX=true
    ```

* 使用，请求`actuator`提供的接口`jolokia`，调用`archaius`注册的JMX操作。
  
    参数格式如下:
    ```json
{
	"operation": <method signature>,
	"type": "exec",
	"arguments": [<method arguments list>],
	"mbean": "Config-com.netflix.config.jmx:class=BaseConfigMBean"
}
    ```

    例如，获取所有配置项: 

	```sh
curl -s -XPOST 'http://127.0.0.1:8080/jolokia' \
    -H 'Content-type: application/json' \
    -d '{"operation": "clearProperty()", "type": "exec", "arguments": [], "mbean": "Config-com.netflix.config.jmx:class=BaseConfigMBean"}'
	```


* 支持的操作如下：

operation | arguments | 描述
--------- | --------- | ----
clearProperty(java.lang.String) | ["prop"] | 清除某项
getProperty(java.lang.String) | ["prop"] | 获取某项配置的值
updateProperty(java.lang.String,java.lang.String) | ["prop","value"] | 更新某项配置的值
addProperty(java.lang.String,java.lang.String) | ["prop","value"] | 新增配置项
obtainProperties() | [] | 获取所有配置项



---

### Zuul中如何在pre-filter中拒绝一个请求

比如，一个请求进入zuul，如果没有认证，则我们想直接返回401，附带一句`not authenticated`，那该怎么做呢？

##### GitHub Issues
* https://github.com/spring-cloud/spring-cloud-netflix/issues/2009
* https://github.com/spring-cloud/spring-cloud-netflix/issues/1874

##### Solutions

> **方案一**: 我的解决方法是，直接抛出一个`ZuulRuntimeException`。

例如：

```java
if (!auth) {
	throw new ZuulRuntimeException(new ZuulException(
				"not authenticated", HttpStatus.UNAUTHORIZED.value(), "not authenticated"));
}
```
则，response为：
```
HTTP/1.1 401 Unauthorized
Content-Type: application/json;charset=utf-8
Transfer-Encoding: chunked

{"timestamp":1496314171649,"status":401,"error":"Unauthorized","exception":"com.netflix.zuul.exception.ZuulException","message":"not auth"}
```

为什么可以这样做，可以参考，我的这篇笔记：[SpringCloud-Zuul 异常处理](sc-zuul-excpetion.md)

> **方案二**: 根据[spencergibb](https://github.com/spencergibb)的[提示](https://github.com/spring-cloud/spring-cloud-netflix/issues/2009#issuecomment-305580226)，将`serviceId`从`RequestContext`中删掉。

例如:

```java
RequestContext ctx = RequestContext.getCurrentContext();
if (!auth) {
	ctx.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
	ctx.setResponseBody("not authenticated");
	ctx.remove(FilterConstants.SERVICE_ID_KEY);
}
```

其中删除`serviceId`的原因是，后续的`route` filter: `RibbonRoutingFilter`是根据是否存在`serviceId`来决定是否执行；删除了就不会执行。但是这对于`SimpleHostRoutingFilter`, `SendForwardFilter`等一些filter不起作用。

---

### Zuul中如何限制API的超时时间

如果启用`@EnableZuulProxy`，并且路由配置为服务名，则使用ribbon进行路由，超时由ribbon参数控制。如下：

```ini
ribbon.ConnectTimeout=3000
ribbon.ReadTimeout=3000
```

支持服务级别的配置：`<serviceId>.ribbon.<property>=<value>`

其他参数配置参考：[zuul 参数调优](http://www.jianshu.com/p/d401452fe76e)



---

未完待续...
