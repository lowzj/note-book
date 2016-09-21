# 服务注册与发现


### Zookeeper

### ETCD

### Consul

* `Agent`
    * server
    * client. forward rpc requests to consul server. every node has a consul client
* `Key/Value`
* `Services`
* `Health Checking`

### SmartStack

* `Zookeeper`. 保存服务的信息。
* `Synapse`. 从 zk 中获取服务信息，并监测服务变更，将服务信息转成 HAProxy 的配置，更新 HAProxy。
* `Nerve`. 管控服务，与服务一起部署，nerve 将服务信息以临时节点的形式注册到zk，同时对服务做check health，支持http等。
* `HAProxy`. local部署，服务间请求走 local haproxy。

