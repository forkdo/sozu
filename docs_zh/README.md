# Sōzu

## Sōzu 是什么？

Sōzu 是一个用 Rust 编写的用于负载均衡的反向代理。它的主要工作是在两个或多个集群后端之间平衡入站请求，以分散负载。

* 它作为 TLS 会话的终止点。因此，处理加密的工作负载从后端卸载。

* 它可以通过阻止来自网络的直接访问来保护后端。

* 它返回一些与其后面的客户端和后端集群之间流量相关的指标。

## 介绍

* [入门][gs]

* [配置 Sōzu][cg]

* [如何使用它][hw]

* [为什么你应该使用 Sōzu][ws]

* [设计动机][dm]

* [技巧][r]

## 概述

* [架构概述][ar]

* [工具和库][tl]

## 深入

* [会话的生命周期][li]

## 发行说明

待办

## 演示和幻灯片

* [Sōzu，一个可热重构的反向 HTTP 代理，作者：Geoffroy Couprie](https://youtu.be/y4NdVW9sHtU)

* [(法语) 2017 年重构反向代理以实现不可变基础设施，作者：Quentin Adam](https://youtu.be/uv3BG1J8YKc)

[gs]: ./getting_started.md
[cg]: ./configure.md
[hw]: ./how_to_use.md
[dm]: ./design_motivation.md
[ar]: ./architecture.md
[tl]: ./tools_libraries.md
[ws]: ./why_you_should_use.md
[r]: ./recipes.md
[li]: ./lifetime_of_a_session.md
