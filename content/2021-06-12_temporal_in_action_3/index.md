+++
title = "Temporal实战 (3) 访问安全性"
description = ""
draft = false

[taxonomies]
tags = ["Temporal", "Workflow"]
categories = ["Temporal in Action"]
+++

Temporal作为一个工作流平台, 会直接承载线上业务请求和数据, 其访问安全性应受到严格保证. 对分布式网络应用来说, 访问安全性应至少涵盖3个方面: 流量加密, 身份认证, 权限控制. Temporal在这方面提供了比较完善的可扩展的支持.

<!-- more -->

## 流量加密

与Temporal有关的流量包括Temporal组件之间以及Temporal与客户端之间. 在Temporal配置中提供了mTLS集成. `internode`用于配置集群中各节点的加密通信, `frontend`用于配置客户端与Frontend Server的加密通信.

## 身份认证

可通过指定mTLS配置中的`serverName`和`client`字段, 避免 [中间人攻击][2].

## 权限控制

常用的权限控制都有着类似的模型: 某个角色对某个资源有着怎样的访问权限. 用表达式来表示就是:

```text
AuthzResult = Authz(Role, Resource)
```

Temporal的权限控制模型也比较类似, 它提供了针对API调用的插件式授权接口: `Authorizer` 和 `ClaimMapper`. 这两个接口可以灵活地实现多种应用场景的权限控制策略. 当客户端请求向Frontend发起调用请求时, Frontend服务会访问这两个接口来实现请求鉴权. 整体的工作流程如下图:

<img src="https://docs.temporal.io/assets/images/frontend-authorization-order-of-operations-0f5cad8e61c9aafd5124d511c2d49772.png" alt="authorization-order" />

### ClaimMapper

要想对调用方用户进行权限控制, 首先获取调用方拥有的权限. Temporal用`ClaimMapper`接口来获取调用方拥有的权限.

```go
type ClaimMapper interface {
    GetClaims(authInfo *AuthInfo) (*Claims, error)
}
```

`GetClaims`方法的参数为AuthInfo, 这里的Auth指的是认证 (Authn), 即通过身份认证结果获取调用方的权限信息. `AuthInfo`结构定义如下:

```go
// Authentication information from subject's JWT token or/and mTLS certificate
type AuthInfo struct {
    AuthToken     string
    TLSSubject    *pkix.Name
    TLSConnection *credentials.TLSInfo
    ExtraData     string
}
```

返回值`Claims`结构表示针对调用方的身份, 授予调用方的权限. 这里需要注意的是, `Authorizer`已经假定, 调用方的身份是经过认证的真实身份. 对非信任的系统来说, 如果没有严格的认证机制保证用户身份的真实性, 权限控制也就无从谈起. `Claims`结构定义如下:

```go
type Claims struct {
    Subject string // 身份主体标识
    System Role // 全局Role
    Namespaces map[string]Role // Namespace级别Role
    Extensions interface{} // 扩展数据
}
```

其中包含了`Role`类型, Role是用bitmask表示的, 在Temporal中存在4种合法的原子Role, 以及1种未定义Role. 原子Role可以叠加形成组合Role.

```go
const (
    RoleWorker = Role(1 << iota)
    RoleReader
    RoleWriter
    RoleAdmin
    RoleUndefined = Role(0)
)
```

#### JWT ClaimMapper

Temporal提供了一个默认的JSON Web Token (JWT) ClaimMapper, 可以从JWT token中解析出身份信息, 将其转换成Temporal Claims身份信息. 默认JWT ClaimMapper要求token采用如下格式表示, 并且token是Base64 URL编码的值.

```text
bearer <token>
```

默认使用`permissions`字段表示Temporal权限, 每个权限用`<namespace>:<permission>`格式表示. 举例:

```json
{
    "permissions": [
        "accounting:read",
        "accounting:write"
    ]
}
```

对同一个namespace的多个权限之间是OR的关系. 例如以上示例中, token中存在`accounting:read`和`accounting:write`两个权限声明, 则会被转换成`authorization.RoleReader | authorization.RoleWriter`.

### Authorizer

Authorizer接口只有一个Authorize方法, 在每次Frontend gRPC请求调用的实际业务逻辑处理之前, 都会先调用该方法.

```go
// Authorizer is an interface for implementing authorization logic
type Authorizer interface {
    Authorize(ctx context.Context, caller *Claims, target *CallTarget) (Result, error)
}
```

该方法中的请求参数包含3个参数: ctx即为go标准context, caller参数声明了调用者的角色 (即模型中的Role). target参数声明了调用目标 (Resource).

Claims在ClaimMapper中已经介绍过, 我们再看看CallTarget的结构定义:

```go
// CallTarget is contains information for Authorizer to make a decision.
// It can be extended to include resources like WorkflowType and TaskQueue
type CallTarget struct {
    // APIName must be the full API function name.
    // Example: "/temporal.api.workflowservice.v1.WorkflowService/StartWorkflowExecution".
    APIName string
    // If a Namespace is not being targeted this be set to an empty string.
    Namespace string
    // If a Namespace is not being targeted this be set to an empty string.
    Request interface{}
}
```

`Authorize`方法调用的返回值Result包含Desicion字段, 存在两种鉴权结果:

- DecisionDeny: 禁止访问 (不会执行实际API调用, 直接返回一个鉴权错误)
- DecisionAllow: 允许访问 (执行实际API调用)

## SSO集成

使用`ClaimMapper`和`Authorizer`可以实现Temporal的SSO单点登录. Temporal Web已经提供了相关支持, 具体可参考 [配置示例][3].

## 总结一下

对于内网用户来说, 网络安全性中的`流量加密`, `身份认证`往往已经通过网络接入层解决了, 而对Temporal本身资源的细粒度权限控制, 才是我们应该关注的重点. [Temporal文档][1]也确实是花了比较大的篇幅介绍它的权限控制机制. 通过抽象的`ClaimMapper`和`Authorizer`插件接口, 以及4种Role角色定义, Temporal实现了对调用方访问其资源的细粒度权限控制, 并默认提供了基于JWT的权限控制实现.

然而, 作为Temporal的使用者, 要想实现自定义的权限控制, 需要修改源码并重新编译Temporal, 这似乎并不是一个优雅的扩展方案.

## 参考资料

- <https://docs.temporal.io/docs/server/security>
- <https://github.com/temporalio/customization-samples>

[1]: https://docs.temporal.io/docs/server/security
[2]: https://en.wikipedia.org/wiki/Man-in-the-middle_attack
[3]: https://github.com/temporalio/web#configuring-authentication-optional