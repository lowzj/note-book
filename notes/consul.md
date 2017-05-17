# Consul

## spring-cloud-consul

## nginx-upsync

## tengine-dyups

## 两者结合

* spring boot application 启动后注册到 service
* 写一个Daemon程序，监控consul上的所有注册services
* 当有变化的时候，将变化值写入consul key-value中，给nginx-upsync使用
* 是有一定延迟，但是代码修改量很少

<pre align="center">
自己实现服务注册发现
</pre>

> 服务启动完成后，调用注册接口；停止之前，调用注销接口；并且提供一个health接口，用于健康检测。

## 注册

调用本地consul agent接口: POST /v1/agent/service/register，如下: 

```sh
curl -s -i -XPOST 'localhost:8500/v1/agent/service/register' -H 'Content-type:application/json' -d @register-body.json
```

**register-body.json**

```
{
    "ID": "${SERVICE_ID}",
        "Name": "${SERVIEC_NAME}",
        "Tags": [
        ],
        "Address": "${SERVICE_IP}",
        "Port": ${SERVICE_PORT},
        "EnableTagOverride": false,
        "Check": {
            "DeregisterCriticalServiceAfter": "5m",
            "HTTP": "http://${SERVICE_IP}:${SRVICE_PORT}/management/health",
            "Interval": "10s",
            "Timeout": "1s"
        }
}
```

`SERVICE_NAME`: 服务名

`SERVICE_ID`: 用于区分同种服务的不同实例, 可以设置为 `{SERVICE_ID}-{SERVICE_PORT}-{RANDOM_VALUE}`

> `${xxx}`的地方需要替换掉

## 注销

调用本地consul agent接口: POST /v1/agent/service/deregister/{serviceId}

```
curl -s -i -XPOST "localhost:8500/v1/agent/service/deregister/${serviceId}"
```

## 服务提供一个health接口

* 服务本身提供一个check health的http接口
```
URL: `/management/health`
Response: 正常返回status code 200
```

# Readings

* [gh-issue: Unable to deregister a service](https://github.com/hashicorp/consul/issues/1188)
* [gh: consul-api for java](https://github.com/Ecwid/consul-api)
* [gh: python-consul](https://github.com/cablehead/python-consul)
* [Consul Official Document](https://www.consul.io/docs/index.html)
  * [Consul Architechture](https://www.consul.io/docs/internals/architecture.html)
  * [Consul: Consensus Protocol](https://www.consul.io/docs/internals/consensus.html)
  * [Gossip Protocol in Consul](https://www.consul.io/docs/internals/gossip.html)
  * [Consul Confiugration](https://www.consul.io/docs/agent/options.html)
  * [Consul: service definition](https://www.consul.io/docs/agent/services.html)
  * [Consul: check definition](https://www.consul.io/docs/agent/checks.html)
  * [Consul REST API](https://www.consul.io/api/catalog.html)
* [Configure Consul in a Prod Env](https://www.digitalocean.com/community/tutorials/how-to-configure-consul-in-a-production-environment-on-ubuntu-14-04)
