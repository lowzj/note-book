# Ribbon: 按参数路由请求

_2017-05-22_

## AbstractLoadBalancerAwareClient

## AbstractLoadBalancerAwareClient的派生类

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
