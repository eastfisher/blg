+++
title = "Temporal实战 (1) 简介"
description = ""

[taxonomies]
tags = ["Temporal", "Workflow"]
categories = ["Blog"]
+++

在微服务架构体系中, 要完成一个业务流程需要多个微服务共同协作, 在分布式环境中对协作关系的治理和维护是一大难题. Temporal作为新兴的工作流编排平台, 其设计初衷就是解决上述问题. 这一系列文章, 我们会从 **微服务工作流编排** 问题出发, 到 **Temporal** 解决方案中去, 由浅入深地学习和理解Temporal.

<!-- more -->

## 背景

### 微服务工作流编排

从微服务架构的概念产生至今已过去7个年头. 在这期间, 面向微服务架构的基础设施发展迅速, 从最初的Docker容器化, 到Kubernetes容器云平台, 再到后来的Service Mesh等, 技术角度上, 微服务的部署使用的成本和门槛已经越来越低.

在解决了这些技术问题之后, 我们的服务就真的**微服务化**了吗? 要回答这一问题, 我们首先要明确, 微服务架构主要是解决单体架构的核心问题--短板效应, 即: 服务无法根据不同应用的需要进行独立扩展. 为解决这一问题, 我们将单体服务按照不同领域拆分成多个微服务, 微服务之间通过相互通信协作, 共同完成原来单体应用可独立完成的业务功能. 然而, 拆分微服务后却引入了新的问题: 我们的系统由单机系统变成了分布式系统. 支持微服务架构的那些复杂而精巧的基础设施, 恰恰是为了解决分布式系统的各种问题而设计的.

那么, 从业务角度看, 微服务又面临哪些问题呢? 在单体时代, 业务流程不论简单与复杂, 均能收敛在同一个单体系统之内, 从单体系统即可看到整个业务流程的全貌. 而拆分为微服务之后, 一个完整的业务流程也被切分到各个独立的微服务之内. 随着时间不断推移, 业务不断变化, 散落在各个微服务代码中的业务流程也会越来越模糊, 并且难以形成一个**全局统一视图**. 代码无法清晰地表达业务流程, 对开发人员来说是件非常遗憾的事. 其次, 跨微服务的流程状态管理, 具有通用性和复杂性. 如果由各个微服务自行维护管理, 相当于一直在重复解决同一个复杂问题, 这将是非常低效的. **微服务工作流编排引擎**就是解决以上两个问题的工具.

微服务工作流编排 (Microservice Workflow Orchestration) 与微服务编排 (Microservice Orchestration) 不同, 前者重业务流程, 后者重基础设施. Kubernetes是典型的微服务编排平台, 但并不是微服务工作流编排平台. 微服务工作流编排, 旨在解决微服务业务流程的统一编排管理的问题, 并尽可能地将业务管理与技术细节隔离 (如分布式系统中的容错, 事务处理等), 让开发者专注于业务. 以下列举了几款相对流行的开源工作流引擎实现.

### 工作流引擎实现

#### Zeebe

[Zeebe][1] 是开源的面向微服务编排的工作流引擎. 为横跨多个微服务的业务流程提供可控性和可观测性. 采用Java语言编写, 提供Java, Go等语言的SDK.

#### Conductor

[Conductor][2] 是Netflix公司开源的面向云环境的编排引擎. 采用Java语言编写, 提供Java, Python语言的SDK.

#### Cadence

[Cadence][3] 是Uber公司开源的分布式高可用编排引擎, 主要用于执行异步的长时间运行的业务逻辑, 并提供扩展性和容错保证. 采用Go语言编写, 提供Java, Go语言的SDK.

#### Temporal

[Temporal][4] 是开源的微服务编排引擎, 用于执行关键业务 (Mission Critical) 代码并提供良好的伸缩性. Temporal与Cadence有着很深的渊源, 其开发团队中的多人曾经是Cadence项目的核心开发成员, 功能特性也与Cadence相仿. 采用Go语言编写, 提供Java, Go语言的SDK.

通过对项目质量, 社区生态, 个人技术栈友好性等多方面进行综合考量, 我决定选择Temporal作为重点调研学习的对象, 所以本文将仅介绍Temporal项目. 后面我会另写一篇文章对以上各项目进行横向对比.

## Temporal ABC

结合Temporal官方文档, 我们首先看看Temporal的典型应用场景, 了解下Temporal能做些什么. 然后熟悉下Temporal的核心概念, 看看Temporal是如何抽象它要解决的问题的. 最后结合官方示例代码, 对Temporal的使用有一个基本的了解和掌握.

### 应用场景

#### 微服务流程编排

微服务架构下, 一个完整的业务流程需要调用多个微服务来共同完成, 当某个环节出现故障时, 需要通过技术手段维护业务规则的完整性与数据最终一致性. 此外, 服务之间的依赖关系也会变得错综复杂. 

Temporal非常适合用于这一场景, 其保证了工作流代码最终能够完成. 通过Temporal提供的多语言客户端, 可以方便地定义重试, 回滚, 甚至人工干预操作, 相比于传统的基于DSL的引擎具有更高的灵活性.

Temporal还提供了对工作流状态的可见性查询. 而基于消息队列的协同式编排, 想要查看当前工作流的状态是非常困难的.

Temporal的伸缩性很好, 可以同时运行非常多的工作流.

#### 分布式事务

分布式环境下, 传统单机事务已经无法满足跨服务的事务一致性. 我们更倾向于使用更为灵活和可靠的Saga事务, 来保证最终一致性.

与`微服务流程编排`很相似, Temporal提供的基础能力可以很好地支持Saga事务. Temporal的Java客户端提供了Saga事务API, 基于该API, 为每个业务动作定义对应的补偿动作, 剩下的事情交给Temporal管理即可. 当中间的业务环节出现异常时, Temporal会根据用户声明的策略, 以补偿的方式对事务执行回滚, 保证业务规则的完整性约束.

#### 长耗时任务

一些业务流程的持续时间可能达到数小时, 数天甚至数年. 面对这一场景, 传统的做法是使用异步化的事件驱动架构来实现. 然而, 这同样会导致业务流程分散在各个服务中, 无法窥其全貌.

Temporal提供了对信号 (Signal) 的支持. 通过信号机制, 我们可以轻松地对工作流的启停进行控制, 当工作流因某些原因中断时, 等待信号的发生. 当信号发生时, 继续执行工作流. 其中的状态维护和容错全部交给Temporal即可.

除以上3个核心应用场景外, 基于Temporal还可实现分布式定时任务调度, 数据pipeline, 基础设施配置等. 我认为, Temporal非常适合做`跨系统的状态管理和调度`, 而在分布式环境中, 跨系统的状态几乎是必然存在的, 因此Temporal的应用场景是非常广泛的.

### 核心概念

#### Workflow

工作流, 即业务流程, 一般会跨多个服务, 需要体现出业务完整性约束. 例如: 下单流程.

#### Activity

我把它称为任务, 即工作流中的一个原子操作, 一般限制在同一个服务内. 例如: 下单流程中的扣减库存操作.

#### Worker

执行Workflow中的Activity的进程实例.

#### Signal

主要用于表示Workflow中发生的事件. Workflow可等待Signal的发生, 并在接收Signal时做出某种响应.

#### 总结

除以上概念外, Temporal还包括Temporal Server, Task Queue和Query等概念, 这些概念更多涉及到实现, 在此先省略.

上面几个概念的关系可以用一句话进行总结: Workflow由一组Activity组成, 由Temporal通过分配Task Queue将其路由到某个Worker实例上执行, 并且Workflow可接收外部Signal改变执行行为.

### 使用示例

在本地启动Temporal, 执行[samples-go][5]中的helloworld示例代码.

首先, 通过docker-compose方式在本地启动Temporal Server和它依赖的服务. 这里我们使用MySQL作为存储, 并启动ElasticSearch用于任务检索.

```
> git clone https://github.com/temporalio/docker-compose
> cd docker-compose
> docker compose -f docker-compose-mysql-es.yml up
```

服务启动完成后, 将[samples-go][5]代码克隆到本地, 执行helloworld示例.

```
> git clone https://github.com/temporalio/samples-go
> cd samples-go/helloworld
> go run helloworld/worker/main.go &
> go run helloworld/starter/main.go
```

helloworld示例相对比较简单, 其中包含`worker`和`starter`. worker即为Temporal概念中的Worker, 该进程会注册helloworld Workflow和Activity, 启动后会连接到Temporal Server的gRPC server上 (默认7233端口), 等待分配任务. starter会通过SDK提交一个workflow到Temporal Server, 由其路由到我们刚刚启动的worker上面, 执行该workflow.

整个流程如下图所示:

{% mermaid() %}
sequenceDiagram
    participant st as Starter
    participant w as Worker
    participant s as Server
    s->>s: Start and Listen
    w->>s: Register
    st->>s: Submit Workflow
    s->>w: Dispatch
    w->>w: Running
    w->>s: Report Result
    s->>st: Result
{% end %}

Temporal的SDK还是比较容易上手的, 作为开发者, 使用Temporal编写跨服务的工作流是一件轻松愉快的事: 将流程声明好, 将实现定义好, 剩下的交给Temporal.

## 参考资料

- https://microservices.io/patterns/microservices.html
- https://temporal.io/usecases
- https://docs.temporal.io/docs/concepts/introduction
- https://docs.temporal.io/docs/glossary


[1]: https://github.com/camunda-cloud/zeebe
[2]: https://github.com/Netflix/conductor
[3]: https://github.com/uber/cadence
[4]: https://github.com/temporalio/temporal
[5]: https://github.com/temporalio/samples-go