# 配方：常见问题的解决方案、提示和技巧

- [配方：常见问题的解决方案、提示和技巧](#配方：常见问题的解决方案、提示和技巧)
- [以非 root 用户身份使用端口 80 或 443](#以非-root-用户身份使用端口-80-或-443)
  - [权能](#权能)
  - [iptables](#iptables)
- [高可用性架构](#高可用性架构)

# 以非 root 用户身份使用端口 80 或 443

1024 以下的端口号通常非 root 用户无法访问。使用 sōzu，
我们经常需要监听端口 80 (HTTP) 和 443 (HTTPS)。为避免以
root 身份运行 sōzu，以下是一些访问这些端口的解决方案。

## 权能

最近的 linux 版本 (> 2.2) 具有一项称为权能的功能，可以
根据上下文激活。要在保留端口上创建侦听套接字，
我们需要 `CAP_NET_BIND_SERVICE` 权能。

我们可以通过创建一个非特权 `sozu` 用户并编写以下
systemd 单元文件来设置它：

```
[Unit]
Description=Sozu - 一个 HTTP 反向代理，可在运行时配置，快速且安全，用 Rust 构建。
Documentation=https://docs.rs/sozu/
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/sozu start --config /etc/sozu/config.toml
ExecReload=/usr/bin/sozu --config /etc/sozu/config.toml reload
Restart=on-failure
User=sozu
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

也可以直接将权能赋予 sozu 二进制文件，使用
`setcap 'cap_net_bind_service=+eip' /usr/bin/sozu`，
但这样任何可以执行 sōzu 的用户都可以访问保留端口（因此他们
可以为 SSH、SMTP 等设置 TCP 代理到自己的软件）。
建议使用单元文件的方式。

## 使用非特权端口

可以使用不同的防火墙将来自保留端口的连接路由到其他非特权端口。
最常见的重定向遵循 80 -> 8080 和 443 -> 8443。

### iptables

可以使用 iptables，使用简单的 nat。

```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 8443
```

### firewalld

firewalld 的语法与 iptables 非常相似。可以使用 `--permanent` 使其永久化。

```
firewall-cmd --direct --add-rule ipv4 nat PREROUTING 0 -p tcp --dport 80 -j REDIRECT --to-port 8080
firewall-cmd --direct --add-rule ipv4 nat PREROUTING 0 -p tcp --dport 443 -j REDIRECT --to-port 8443
```

请注意，与 sōzu 具有相同 uid 的任何软件都将能够监听
8080 和 8443 端口，因为这些端口是非特权的，并且 sōzu 使用
`SO_REUSEPORT` 选项设置侦听套接字。

# 高可用性架构

待办事项
