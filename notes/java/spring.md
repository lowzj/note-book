# Spring

### Spring Annotation

* @RequestMapping @RequestBody
  * [Post JSON to spring REST webservice](http://www.leveluplunch.com/java/tutorials/014-post-json-to-spring-rest-webservice/)

### Spring Config
* `spring.config.name` and `spring.config.location` are used very early to determine which files have to be loaded so they have to be defined as an environment property (typically **OS env**, **system property** or **command line argument**).

    > 这两个配置项放在配置文件里面是无效的

* Spring Boot uses a very particular PropertySource order that is designed to allow sensible overriding of values, properties are considered in the following order[^1]:

    * Command line arguments.
    * Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property)
    * JNDI attributes from `java:comp/env`.
    * Java System properties (`System.getProperties()`).
    * OS environment variables.
    * A `RandomValuePropertySource` that only has properties in `random.*`.
    * [Profile-specific application properties](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) outside of your packaged jar (`application-{profile}.properties` and YAML variants)
        * 多个profile，越靠后引入优先级越高
        * 一个profile文件里使用`spring.profiles.include`引入的配置，优先级比本文件内的配置高
    * [Profile-specific application properties](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) packaged inside your jar (`application-{profile}.properties` and YAML variants)
    * Application properties outside of your packaged jar (`application.properties` and YAML variants).
    * Application properties packaged inside your jar (`application.properties` and YAML variants).
    * `@PropertySource` annotations on your `@Configuration` classes.
    * Default properties (specified using `SpringApplication.setDefaultProperties`).

* [Creating your own auto-configuration](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-auto-configuration.html)
    * 使用`@Configuration`注解来实现**auto-configuration**类.
    * `@AutoConfigureAfter`, `@AutoConfigureBefore`, `@AutoConfigureOrder`指定顺序.
    * 定位需要自动配置的类, spring boot 会检查已发布的jar包里是否存在文件`META-INF/spring.factories`. 文件内容如下:
        ```
        org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
        com.mycorp.libx.autoconfigure.LibXAutoConfiguration,\
        com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
        ```

### Disabling specific auto-configuration
[Disabling specific auto-configuration](http://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-auto-configuration.html#using-boot-disabling-specific-auto-configuration)

使用`@EnableAutoConfiguration`剔除不想引用的自动配置类

```java
@Configuration
@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
public class MyConfiguration {
}
```
或者在`properties`文件里使用配置项`spring.autoconfigure.exclude`，例如
```
spring.autoconfigure.exclude=com.lowzj.example.MyAutoConfiguration1,\
  com.lowzj.example.MyAutoConfiguration2
```

---

[^1]: [Spring Externalized Configuration](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config)
