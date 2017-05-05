
Spring Cloud Config
==============

> 统一的配置管理服务，基于[spring-cloud-config](https://github.com/spring-cloud/spring-cloud-config)。

> 目的是将程序与配置完全分离，方便查看更改，便于定位由配置文件导致的问题，同时有利于服务的自动化部署。

> spring-cloud-config项目中包含两个核心的部分：`spring-cloud-config-server`及`spring-cloud-config-client`。应用服务集成ConfigClient，在启动时，向ConfigServer请求配置文件，ConfigServer从配置仓库中获取最新的配置文件，然后以JSON格式返回给ConfigCient。

本文主要介绍如何结合spring-cloud-config, git以及spring configuration profile来搭建一个配置中心。

* [ConfigServer](#configserver)
* [ConfigClient](#configclient)
* [GitHub Repo 目录结构](#github-repo-目录结构)
* [Spring Configuration Profile](#spring-configuration-profile)

ConfigServer
==============

* 配置ConfigServer -- 远程仓库模式

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

  > **NOTE**
  > * `spring.cloud.config.server.git.searchPaths=**`。 ConfigServer递归搜索该repo下的所有子目录来寻找配置文件。
  > * `health.config.enabled=false`。 禁止健康检测，开启的话会请求GitHub，非常耗时。

* 配置ConfigServer -- 本地仓库模式

    ```ini
    spring.application.name=ConfigServer
    management.context-path=/management
    health.config.enabled=false
    
    spring.cloud.config.server.health.enabled=false
    spring.cloud.config.server.git.searchPaths=**
    spring.cloud.config.server.git.uri=file:///tmp/config-server-test/
    ```

  > **NOTE**
  > * `spring.cloud.config.server.git.uri=/tmp/config-server-test/` 为本地的git repository目录, 并且将remote信息删除，这样ConfigServer不会请求远程仓库。
  > ```
  > git remote remove [remote_name]
  > ```

* 查看 ConfigServer 中的配置文件
    - 例如配置文件在 git 仓库中的名字为 application-dev.properties
    - 可使用 http://${server.ip}:${server.port}/application/dev 来查看属性
    - URL拼接规则

        ```
        /{name}/{profile}[/{label}]
        /{name}-{profile}.yml
        /{label}/{name}-{profile}.yml
        /{name}-{profile}.properties
        /{label}/{name}-{profile}.properties
        ```

> **NOTE**

国内访问GitHub很慢，用于生产环境很不合适，可以在内网自建GitLab来代替。如果想要基于GitHub工作，可以利用GitHub提供的Webhook功能将GitHub上的更改同步到自建的GitLab上。

这里有个例子：https://github.com/lowzj/gitlab-mirror-webhook


ConfigClient
==============

> ConfigClient会连接到ConfigServer获取指定的配置文件，将其作为运行配置加载

* 应用服务集成ConfigClient，添加ConfigClient依赖

    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-client</artifactId>
    </dependency>
    ```

* 配置 ConfigClient

    * 配置文件方式

        ```ini
        spring.cloud.config.uri=http://localhost:8888
        spring.cloud.config.name=Kefu
        spring.cloud.config.profile=AdminServer-1.3
        spring.cloud.config.label=sandbox
        ```

    * 环境变量方式

        ```sh
        export SPRING_CLOUD_CONFIG_URI=http://localhost:8888
        export SPRING_CLOUD_CONFIG_NAME=Kefu
        export SPRING_CLOUD_CONFIG_PROFILE=AdminServer-1.3
        export SPRING_CLOUD_CONFIG_LABEL=sandbox
        ```

  > **NOTE**
  > * `spring.cloud.config.profile=AdminServer-1.3` 可以配置多个，多个`profile`用逗号隔开，比如：`spring.cloud.config.profile=AdminServer-common,AdminServer-1.3`。
  > * 指定多个`profile`，位置越靠后优先级越高，即后面`profile`里的配置项会覆盖掉前面`profile`中相同配置项。

* 服务中的 ConfigClient 拼接URL访问 ConfigServer 来加载配置文件:

    ```sh
    curl -XGET "http://localhost:8888/Kefu/AdminServer-1.3/sandbox"
    ```

  ConfigServer 最终找到的配置文件在 github 上的地址:

    ```sh
    https://github.com/lowzj/config-server-test/blob/sandbox/AdminServer/Kefu-AdminServer-1.3.properties
    ```

  > **NOTE**: 推荐使用`jq`，非常强大的JSON处理工具。
  > ```sh 
  > curl -XGET "http://localhost:8888/Kefu/AdminServer-1.3/sandbox" | jq .
  > ```

* 服务启动时输出日志，用于检查服务是否拿到正确配置。如下：

    ```
    1: o.s.c.c.c.ConfigServicePropertySourceLocator - Fetching config from server at: http://localhost:8888
    2: o.s.c.c.c.ConfigServicePropertySourceLocator - Located environment: name=Kefu, profiles=[AdminServer-1.3], label=sandbox, version=2971e66b6b68eaefd9d9e574936cef0ba0894bf5
    3: o.s.c.b.c.PropertySourceBootstrapConfiguration - Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource [name='file:///tmp/config-server-test/AdminServer/Kefu-Adminerver-1.3.properties'], MapPropertySource [name='file:///tmp/config-server-test/common/Kefu-bootstrap.properties'], MapPropertySource [name='file:///tmp/config-server-test/common/Kefu-management.properties'], MapPropertySource [name='file:///tmp/config-server-test/common/Kefu.properties']]]
    ```

  1. 第一行，显示ConfigServer地址。
  2. 第二行，ConfigClient向ConfigServer请求的参数。用于检查ConfigClient相关配置是否正确。
  3. 第三行，ConfigServer返回的配置文件，优先级由高到低排列。由其中`proertySource`的`name=file:///...`可以看出，ConfigServer是以本地模式启动的。

GitHub Repo 目录结构
===============

所有配置文件都放在了github上，为了方便管理，统一的目录结构如下：

```sh
.
├── AdminServer
│   └── Kefu-AdminServer-1.3.properties
├── Webapp
│   ├── Kefu-Webapp-43.1.properties
│   └── Kefu-Webapp-43.2.properties
└── common
    ├── Kefu-bootstrap.properties
    ├── Kefu-management.properties
    └── Kefu.properties
```

子目录为服务名称，比如`AdminServer`。其中`common`目录下面是通用配置。
配置文件的名称必须是全局唯一的，即使是在不同的子目录中。

使ConfigServer支持子目录，须添加如下配置中的一个：
```ini
# 支持一级子目录
spring.cloud.config.server.git.searchPaths=*
# 支持多级子目录
spring.cloud.config.server.git.searchPaths=**
```


Spring Configuration Profile
================

以AdminServer服务的配置为例子。

#### <a>配置文件命名规则(properties格式)</a>

* 全称: `{spring.config.name}[-{spring.config.profile}].properties`
* `spring.config.name`. 配置文件的名字，默认为`application`。
    * 对应到ConfigClient中的配置为: `spring.cloud.config.name` 或者 `SPRING_CLOUD_CONFIG_NAME`。
    * 以下全部简称**name**
* `spring.config.profile`. 可选的，没有则为空，那么配置文件就是: `{spring.config.name}.properties`。
    * [profile](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html)
    * 对应到ConfigClient中的配置为: `spring.cloud.config.profile` 或者 `SPRING_CLOUD_CONFIG_PROFILE`。
    * 以下全部简称**profile**

举几个例子

**Full Name**                       | **name**      | **profile**
------------------------------------|---------------|---------------
application.properties              | application   | -
application-dev.properties          | application   | dev
Kefu.properties                     | Kefu          | -
Kefu-AdminServer-1.3.properties     | Kefu          | AdminServer-1.3
AdminServer-1.3.properties          | AdminServer   | 1.3


> **NOTE**
>
> 我们可以将**name**定义成一个组名，这个组下的服务用**profile**定义。并且**profile**中包含服务名以及配置版本号，格式为**{server name}-{version}**。
> 可以使用`spring.profiles.include`配置，用来激活该组服务所需的通用配置。
>
> 上面例子中组名**name**为`Kefu`，**profile**指定了服务配置`AdminServer-1.3`。


#### <a>配置的来源</a>

我们服务中的配置来源主要有4个方面

1. 命令行参数
2. 系统环境变量
3. ConfigServer
4. JAR包中的默认配置

以上按配置生效的优先级排序(由高到低)。详细的优先级请参考: [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html#boot-features-external-config)

下面详细介绍下各个来源的配置作用

##### 命令行参数

* JVM 参数
* server.port
* logging.config
* spring.config.location. 此配置只能在配置文件外指定，用于指定ConfigClient的配置文件。
* 自定义参数

##### 系统环境变量

* ConfigCient 相关配置。用于从ConfigServer中获取该服务所需配置。

    ```sh
    export SPRING_CLOUD_CONFIG_URI=http://localhost:8888
    export SPRING_CLOUD_CONFIG_NAME=Kefu
    export SPRING_CLOUD_CONFIG_PROFILE=AdminServer-1.3
    export SPRING_CLOUD_CONFIG_LABEL=sandbox
    ```

* JAR包相关信息。groupId, artifactId, version等，用于下载JAR包。

    ```sh
    export PKG_GROUP=io.github.lowzj.example
    export PKG_NAME=spring-boot-admin
    export PKG_VERSION=1.3.4
    export PKG_REPOSITORY=release
    ```

以上就可将配置文件和服务启动包分离开，我们可以写一个通用的启停脚本或制作一个通用的docker镜像，输入参数(环境变量)即以上8条。再结合marathon或者kubernetes等实现自动化部署。步骤大致如下：

* 从nexus等包管理仓库中下载启动包。
* 设置好ConfigClient相关配置，指定`spring.config.location`。
* 服务启动时从ConfigServer中获取到所需配置，完成启动。

另外可以增加一个额外配置项，用于自定义JVM参数和程序参数。

`IMPORTANT` 默认的，来自ConfigServer的配置，优先级会高于系统参数，可以在存储于ConfigServer端(git)的配置文件中加入下面这个配置项：
```ini
# 默认为true
spring.cloud.config.overrideSystemProperties=false
```

##### ConfigServer

所有ConfigServer中的配置都是以**Kefu**为**name**，**profile**指定具体的服务及配置版本。
所有服务向ConfigServer请求配置，返回的配置文件列表如下:

* Kefu.properties。由`SPRING_CLOUD_CONFIG_NAME=Kefu`指定的配置文件。用于引入所有服务的通用配置，内容如下：

    ```
    spring.profiles.include=management,bootstrap
    ```

* 由环境变量`SPRING_CLOUD_CONFIG_PROFILE=AdminServer-1.3`指定的配置文件
    * Kefu-AdminServer-1.3.properties. 服务本身所需的所有配置项。

* Kefu.properties文件中由 `spring.profiles.include` 指定的 **profile** 配置文件
    * Kefu-management.properties. spring management 相关配置。
    * Kefu-bootstrap.properties. 启用服务注册机制相关配置。

ConfigServer以JSON格式返回给服务，最终包含以下配置文件的配置:

* Kefu-AdminServer-1.3.properties
* Kefu-bootstrap.properties
* Kefu-management.properties
* Kefu.properties

优先级从高到低排序，其中以 `Kefu-AdminServer-1.3.properties` 中的配置优先级最高，可以覆盖掉其他配置文件的配置项。
开发人员需要关心的服务本身的配置即: `Kefu-AdminServer-1.3.properties` 里的内容。

* 一个profile文件里使用`spring.profiles.include`引入的配置，优先级比本文件内的配置高
* 多个profile，越靠后引入优先级越高

> **NOTE**: 通过HTTP API从ConfigServer中获取配置
>
> ```sh
> curl -s -XGET 'http://{config-server-address}/{name}/{profile}/{label}' | jq .
> ```
>
> 其中`label`，为git repo分支名，用于区分部署环境。

##### JAR包中的默认配置

最好是移除掉，全部从ConfigServer中获取。


READINGS
==============
* [Spring Cloud Config](http://cloud.spring.io/spring-cloud-static/spring-cloud.html#_spring_cloud_config)
* [Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)
* [Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html)
* [GitHub Issue: How to use local properties in config client to override remote source](https://github.com/spring-cloud/spring-cloud-config/issues/651)

