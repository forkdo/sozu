# Sōzu

## Sōzu 是什么？

Sōzu 是一个用 Rust 编写的用于负载均衡的反向代理。它的主要工作是在两个或多个集群后端之间平衡入站请求以分散负载。

* 它充当 TLS 会话的终止点。因此，处理加密的工作负载从后端卸载。

* 它可以通过防止从网络直接访问来保护后端。

* 它返回一些与客户端和其后面的后端集群之间的流量相关的指标。

## 介绍

* [入门][gs]

* [配置 Sōzu][cg]

* [如何使用它][hw]

* [为什么你应该使用 Sōzu][ws]

* [设计动机][dm]

* [食谱][r]

## 概述

* [架构概述][ar]

* [工具和库][tl]

## 深入

* [会话的生命周期][li]

## 发行说明

待办事项

## 演示和幻灯片

* [Sōzu, a hot reconfigurable reverse HTTP proxy by Geoffroy Couprie](https://youtu.be/y4NdVW9sHtU)

* [(FR) Refondre le reverse proxy en 2017 pour faire de l’immutable infrastructure. by Quentin Adam](https://youtu.be/uv3BG1J8YKc)

[gs]: ./getting_started.md
[cg]: ./configure.md
[hw]: ./how_to_use.md
[dm]: ./design_motivation.md
[ar]: ./architecture.md
[tl]: ./tools_libraries.md
[ws]: ./why_you_should_use.md
[r]: ./recipes.md
[li]: ./lifetime_of_a_session.md
