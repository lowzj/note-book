# StackStorm

<p align='right'>
If This Then That for DevOps Automation
</p>

StackStorm是第三波操作自动化的新兴领导者。从DevOps的基础上, StackStorm的愿景是一个自驱动的数据中心，可学习如何更好地操作自己。StackStorm解决方案利用了现有的配置管理和监控解决方案, 实现了今天松散耦合、异构、不断变化的云计算基础设施的自动化。通过Stackstorm的操作会被平台记录，为审计提供数据。

## Overview

#### 架构

![StackStorm architecture diagram](https://cloud.githubusercontent.com/assets/20028/5688946/fabef9ec-9822-11e4-859e-29bbb67df85b.jpg)

StackStorm采用模块化的架构，由多个松耦合的能水平扩展的服务组成；这些服务之间通过消息总线(message bug)进行通信。提供Web UI，CLI以及完整的REST API。另外还发布了python客户端，以方便开发者。

#### 术语

* **感应器(Sensor)** 用于监视外部系统事件的python插件，并在事件发生时触发StackStorm的触发器。
* **触发器(Trigger)** 是StackStorm内部表示外部系统事件。分为通用的触发器(定时器, Webhook等)和集成触发器(例如Sensu alert, JIRA updated等)。新类型的触发器，可以通过编写感应器插件来定义一个新的触发器类型。
* **动作(Action)** 是StackStorm的外部集成。有通用action(如ssh, REST call)，集成action(如OpenStack，Docker，Puppet)，或者自定义action。Action可以是python插件，或者任何脚本，通过添加几行元数据即可被StackStorm使用。用户可以直接通过CLI/API来调用action，或者被当作rule/workflow中的一部分来执行。
* **规则(Rule)** 将trigger映射到action(或者workflow)，遵循匹配条件并且将trigger载体映射为action输入。
* **工作流(Workflow)** 将多个action组合起来。定义顺序，转换条件以及传递数据。大多数自动化操作都不止一个步骤，因此需要多个action。跟'原子'的action一样，workflow也可以手动调用或者被rule触发。
* **包(Pack)** 是部署单元。pack通过分组集成(trigger、action)和自动化(rule、workflow)，来简化StackStorm插件化内容的管理和分享。StackStorm社区提供越来越多的pack。用户也可以创建自己的pack，并在GitHub上分享，或者提交到StackStorm的repo中。
* **审计追踪(Audit trail)** 用于记录action的执行操作(手动、自动)，包括详细的触发上下文和执行结果。审计日志可以与外部日志记录和分析工具集成：LogStash，Splunk，statsd，syslog。




