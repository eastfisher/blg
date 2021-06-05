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

### 应用场景

### 核心概念

### 使用示例

## 参考资料

- https://microservices.io/patterns/microservices.html
- https://temporal.io/usecases
- https://docs.temporal.io/docs/concepts/introduction
- https://docs.temporal.io/docs/glossary
- https://github.com/temporalio/samples-go/tree/master/expense

[1]: https://github.com/camunda-cloud/zeebe
[2]: https://github.com/Netflix/conductor
[3]: https://github.com/uber/cadence
[4]: https://github.com/temporalio/temporal