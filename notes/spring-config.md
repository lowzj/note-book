# Spring Config

* `spring.config.name` and `spring.config.location` are used very early to determine which files have to be loaded so they have to be defined as an environment property (typically **OS env**, **system property** or **command line argument**).

    > 这两个配置项放在配置文件里面是无效的

* Spring Boot uses a very particular PropertySource order that is designed to allow sensible overriding of values, properties are considered in the following order:

    * Command line arguments.
    * Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property)
    * JNDI attributes from `java:comp/env`.
    * Java System properties (`System.getProperties()`).
    * OS environment variables.
    * A `RandomValuePropertySource` that only has properties in `random.*`.
    * [Profile-specific application properties](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) outside of your packaged jar (`application-{profile}.properties` and YAML variants)
    * [Profile-specific application properties](http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config-profile-specific-properties) packaged inside your jar (`application-{profile}.properties` and YAML variants)
    * Application properties outside of your packaged jar (`application.properties` and YAML variants).
    * Application properties packaged inside your jar (`application.properties` and YAML variants).
    * `@PropertySource` annotations on your `@Configuration` classes.
    * Default properties (specified using `SpringApplication.setDefaultProperties`).
