# 1. Basic

## 1.1 Policy

In addition to mandatory properties (e.g. durable or exclusive), queues and exchanges in RabbitMQ have optional parameters (arguments), sometimes referred to as x-arguments. Those are provided by clients when they declare queues (exchanges) and control various optional features, such as [queue length limit](https://www.rabbitmq.com/maxlength.html) or [TTL](https://www.rabbitmq.com/ttl.html). （除了可选参数，exchange和queue还可以有一些可选参数，有时候也叫做X参数，他们通常由客户端在创建exchange和queue的时候提供用来控制一些特性，比如说queue_length_limit、TTL等）

Client-controlled properties in some of the protocols RabbitMQ supports generally work well but they can be inflexible: updating TTL values or mirroring parameters that way required application changes, redeployment and queue re-declaration (which involves deletion). In addition, there is no way to control the extra arguments for groups of queues and exchanges. Policies were introduced to address the above pain points. （客户端控制的参数在RabbitMQ提供的部分协议里可以很好工作，但是不太灵活：更新TTL或者镜像参数需要应用改动、重新部署并且重新声明queue（包括删除）。此外，没有办法去控制成组的queue和exchange的额外参数。Policy就是被用来解决这些痛点的。）

A policy matches one or more queues by name (using a regular expression pattern) and appends its definition (a map of optional arguments) to the x-arguments of the matching queues. In other words, it is possible to configure x-arguments for multiple queues at once with a policy, and update them all at once by updating policy definition. （一个Policy可以通过名字（使用正则表达式）匹配一个或多个queue，然后把Policy的定义（可选参数的Map）应用到匹配queue的可选参数上。换句话说，现在就有可能用一个policy去为多个queue配置x-arguments，然后通过更新Policy的定义来批量更新queue的定义）

How Policies Work（Policy是如何工作的呢？）

Key policy attributes are

- name: it can be anything but ASCII-based names without spaces are recommended （可以是基于ASCII的非空的任意字符串）
- pattern: a regular expression that matches one or more queue (exchange) names. Any regular expression can be used. （任何正则表达式）
- definition: a set of key/value pairs (think a JSON document) that will be injected into the map of optional arguments of the matching queues and exchanges（会追加到所匹配的queue和exchange的定义上的K-V 集合）
- priority: TODO

Policies automatically match against exchanges and queues, and help determine how they behave. Each exchange or queue will have at most one policy matching (see [Combining Policy Definitions](https://www.rabbitmq.com/parameters.html#combining-policy-definitions) below), and each policy then injects a set of key-value pairs (policy definition) on to the matching queues (exchanges). （Policy会自动地匹配到exchange和queue上，然后帮助决定它们要怎么做。每个exchange或者queue至多匹配到一个policy，然后每个policy会把它的属性集应用到匹配的queue或exchange上）

Policies can match only queues, only exchanges, or both. This is controlled using the apply-to flag when a policy is created. （Policy可以只匹配queue或exchange，也可以都匹配。这些东西由policy创建时的`apply-to`标志来指明）

Policies can change at any time. When a policy definition is updated, its effect on matching exchanges and queues will be reapplied. Usually it happens instantaneously but for very busy queues can take a bit of time (say, a few seconds). （Policy可以随时改动。当Policy定义改变时，它会重新应用到所匹配的queue和exchange上。通常这会立即生效，但是对于那些繁忙的队列，可能会等上一会）

Policies are matched and applied every time an exchange or queue is created, not just when the policy is created. （Policy会在exchange或queue创建的时候匹配并应用上，并不仅仅只有Policy创建的时候）

Policies can be used to configure the [federation plugin](https://www.rabbitmq.com/federation.html), [mirroring of classic queues](https://www.rabbitmq.com/ha.html), [alternate exchanges](https://www.rabbitmq.com/ae.html), [dead lettering](https://www.rabbitmq.com/dlx.html),[per-queue TTLs](https://www.rabbitmq.com/ttl.html), and [queue length limit](https://www.rabbitmq.com/maxlength.html). （Policy可以用来配置 [federation plugin](https://www.rabbitmq.com/federation.html), [mirroring of classic queues](https://www.rabbitmq.com/ha.html), [alternate exchanges](https://www.rabbitmq.com/ae.html), [dead lettering](https://www.rabbitmq.com/dlx.html),[per-queue TTLs](https://www.rabbitmq.com/ttl.html), and [queue length limit](https://www.rabbitmq.com/maxlength.html)）

## 1.2 User

RabbitMQ 是一个多租户系统，使用vhost来逻辑上隔离资源。

## 1.3 Heartbeat

客户端与服务端的超时时间不应该太小，否则可能会造成瞬时的网络拥塞，更容易触发流控等。实践证明，大多数环境5-20s为宜。

## 1.4 X-Arguments

