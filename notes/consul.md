# Consul

## spring-cloud-consul

## nginx-upsync

## tengine-dyups

## 两者结合

* spring boot application 启动后注册到 service
* 写一个Daemon程序，监控consul上的所有注册services
* 当有变化的时候，将变化值写入consul key-value中，给nginx-upsync使用
* 是有一定延迟，但是代码修改量很少
