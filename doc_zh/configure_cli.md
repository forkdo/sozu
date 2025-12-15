# 通过命令行配置 sozu

sozu 可执行文件可用于启动代理并对其进行配置：添加新的后端服务器、读取指标等。
它通过 unix 套接字与当前正在运行的代理进行通信。

您可以通过将以下内容添加到您的 `config.toml` 来指定其路径：

```toml
command_socket = "path/to/your/command_folder/sock"
```

## 添加具有 http 和 https 前端的集群

首先，您需要创建一个具有 id 和负载平衡策略（roundrobin 或 random）的新集群：

```bash
sozu --config /etc/sozu/config.toml cluster add --id <my_cluster_id> --load-balancing-policy roundrobin
```

它不会显示任何内容，但您可以通过查询 sozu 来验证集群是否已成功添加：

```bash
sozu --config /etc/sozu/config.toml query clusters
```

然后你需要添加一个后端：

```bash
sozu --config /etc/sozu/config.toml backend add --address 127.0.0.1:3000 --backend-id <my_backend_id> --id <my_cluster_id>
```

### 添加 http 前端

和一个 http 监听器：

```bash
sozu --config /etc/sozu/config.toml listener http add --address 0.0.0.0:80 --tls-versions TLSv1.2 --tls-cipher-list ECDHE-ECDSA-AES256-GCM-SHA384 --tls-cipher-suites TLS_AES_256_GCM_SHA384 --tls-signature-algorithms ECDSA+SHA512 --tls-groups-list x25519 --expect-proxy
```

最后，您必须创建一个前端以允许 sozu 将流量从侦听器发送到您的后端：

```bash
sozu --config /etc/sozu/config.toml frontend http add --address 0.0.0.0:80 --hostname <my_cluster_hostname> id <my_cluster_id>
```

### 添加 https 前端

和一个 https 监听器：

```bash
sozu --config /etc/sozu/config.toml listener https add --address 0.0.0.0:443
```

最后，您必须创建一个前端以允许 sozu 将流量从侦听器发送到您的后端：

```bash
sozu --config /etc/sozu/config.toml frontend https add --address 0.0.0.0:443 --hostname <my_cluster_hostname> id <my_cluster_id>
```

## 检查 sozu 的状态

它显示了一个工作进程列表并显示有关其状态的信息。

```bash
sozu --config /etc/sozu/config.toml status
```

## 获取指标和统计信息

它将显示有关 sozu、工作进程和集群指标的全局统计信息。

```bash
sozu --config /etc/sozu/config.toml query metrics
```

## 转储和恢复状态

如果 sozu 配置（集群、前端和后端）未写入配置文件，您可以保存 sozu 状态以便稍后恢复。

```bash
sozu --config /etc/sozu/config.toml state save --file state.json
```

然后正常关闭 sozu：

```bash
sozu --config /etc/sozu/config.toml shutdown
```

重新启动 sozu 并恢复其状态：

```bash
sozu --config /etc/sozu/config.toml state load --file state.json
```

您应该能够像关闭前一样请求您的集群。

### 使用事件监控后端状态

此 CLI 命令：

```bash
sozu --config /path/to/config.toml events
```

侦听 Sōzu 工作进程在后端关闭、再次启动或没有可用后端时发送的事件。
