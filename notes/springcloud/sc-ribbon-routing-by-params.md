# Ribbon: 按参数路由请求

_2017-05-22_

考虑这样几个应用场景
* 需要上线一个服务的新版本，为了降低上线可能造成的影响范围，对所有请求做路由控制，逐步引流到新版本服务上；在此过程中，观察以及监控系统及用户状态，发现问题时可随时回滚。对于一个基于用户体系的系统，可以按照用户ID逐步放量，例如按测试用户，内部用户，体验用户，全量用户等等进行逐步放量；或者直接对用户ID取模，指定模数路由到新版。
* 对于一个社交系统或者广告系统，经常会向用户推荐内容(新鲜事/广告)。推荐算法的优劣会直接影响用户PV/CV的量，算法团队需要不断的调整参数进行比较以找到合适的值。如果每次调参都涉及到所有用户，显然对全站影响很大，而且周期也很长。所以通常做法是让参数或者是算法(比如服务中嵌套动态脚本)做到在线动态可调，另外对用户分组：比如90%的用户使用之前的算法，另外10%的用户平均分成5组来使用不同的5种算法，这样就形成了1个对照组和5个实验组，可以同时进行比较，而且是同时段比较(避免节假日等特殊日子的影响)，影响范围至多10%。这里就需要一个路由策略，将不同用户路由到不同的服务实例上。
* 服务所要维护的数据量巨大，一个服务实例无法处理全量数据。通常一个简单的做法就是对数据进行分片(sharding)，每个服务实例只处理部分数据。对于每个请求，根据数据ID，将其路由到特定的服务实例上。

以上，最终都归结于一个点，就是要根据请求中的某些信息，将该请求路由到特定的服务实例上。

`Ribbon`从设计上是支持按参数路由请求的，这点从`IRule#choose(Object)`这个接口设计就能看得出来，这个方法里的参数称为`LoadBalancerKey`。但是在实际运行时却发现，传给`IRule#choose(Object)`的这个`loadBalancerKey`一直是`null`，这就坑爹了，有必要弄清楚到底是怎么回事。

## IRule

## BaseLoadBalancer

## LoadBalancerCommand

## AbstractLoadBalancerAwareClient

#### spring-cloud-netflix中AbstractLoadBalancerAwareClient的派生类

```
AbstractLoadBalancerAwareClient
  |-- AbstractLoadBalancingClient 
  |     |-- OkHttpLoadBalancingClient
  |     |     \__ RetryableOkHttpLoadBalancingClient
  |     \__ RibbonLoadBalancingHttpClient
  |           \__ RetryableRibbonLoadBalancingHttpClient
  |-- FeignLoadBalancer
  |     \__ RetryableFeignLoadBalancer
  \__ RestClient
        \__ OverrideRestClient
              \__ TestRestClient
```

## Related
以下是我提的issue及PR，希望能此功能能尽早合进官方版本。

* Issue: https://github.com/spring-cloud/spring-cloud-netflix/issues/1272
* PR: https://github.com/spring-cloud/spring-cloud-netflix/pull/1556
* PR: https://github.com/Netflix/ribbon/pull/309
