### 通过 Sozu 命令行配置集群

本说明演示如何使用 `sozu` 命令行工具来配置集群、前端和后端，以替代直接编辑 `config.toml` 文件。

**重要提示:** 所有 `sozu` 命令都需要通过 `--config <your-config-file>` 参数来指定配置文件。此文件包含了 sozu 实例正在监听的命令套接字（command socket）的路径，从而让命令行工具能够与 sozu 通信。在以下示例中，我们使用 `--config config.toml`。

#### 原始 `config.toml` 配置示例：

```toml
# A unique identifier for our routing rule.
[clusters.komo]

# The protocol this cluster will handle.
protocol = "http"
load_balancing = "ROUND_ROBIN"

# 'frontends' define which requests this cluster will handle.
# It matches requests coming to the listener at 'address' with the specified 'hostname'.
frontends = [
    { address = "0.0.0.0:80", hostname = "dash.bdev.cn" },
    { address = "0.0.0.0:443", hostname = "dash.bdev.cn" }
]

# 'backends' define where to forward the matched requests.
backends = [
    { address = "komodo-core-1:9120" }
]
```

#### 将 `config.toml` 配置转换为命令行指令

以下是将上述 `config.toml` 配置转换为等效 `sozu` 命令行指令的步骤：

##### 1. 添加集群 (Cluster)

首先，创建一个名为 `komo` 的集群，并设置其负载均衡策略。请注意，根据之前的错误提示，我们使用 `--id` 和 `--load-balancing-policy` 参数。

```bash
sozu --config config.toml cluster add --id komo --load-balancing-policy ROUND_ROBIN
```
**说明:**
*   `--config config.toml`: 指定 sozu 的配置文件。
*   `cluster add --id komo`: 创建一个唯一标识符为 `komo` 的新集群。
*   `--load-balancing-policy ROUND_ROBIN`: 设置负载均衡算法为轮询。

**关于协议 (`protocol`):**
`cluster add` 命令可能不直接支持设置协议。如果 `http` 不是默认协议，您可能需要使用 `sozu cluster modify` 命令或检查 `sozu cluster add --help` 以获取完整选项。

##### 2. 添加前端 (Frontends)

接下来，为 `komo` 集群添加两个前端，用于匹配传入的请求。注意命令的结构：协议 (`http` 或 `https`) 在 `frontend` 之后，并且必须在末尾使用 `id <cluster_id>` 子命令来将其关联到集群。

```bash
# 添加处理 HTTP (80端口) 的前端
sozu --config config.toml frontend http add --address 0.0.0.0:80 --hostname dash.bdev.cn id komo

# 添加处理 HTTPS (443端口) 的前端
sozu --config config.toml frontend https add --address 0.0.0.0:443 --hostname dash.bdev.cn id komo
```
**说明:**
*   `frontend http add` / `frontend https add`: 分别定义一个 HTTP 或 HTTPS 协议的前端添加操作。
*   `--address ... --hostname ...`: 定义匹配流量的规则。
*   `id komo`: 将此前端规则关联到 ID 为 `komo` 的集群。

##### 3. 添加后端 (Backend)

最后，为 `komo` 集群添加后端的服务地址，sozu 会将匹配到的请求转发到这里。注意，需要使用 `--id` 指定集群，并使用 `--backend-id` 为此后端提供一个唯一的标识符。

```bash
sozu --config config.toml backend add --id komo --backend-id komodo-core-1 --address komodo-core-1:9120
```
**说明:**
*   `backend add`: 定义一个后端添加操作。
*   `--id komo`: 指定要添加后端的目标集群 ID。
*   `--backend-id komodo-core-1`: 为此后端在集群中设置一个唯一的标识符。
*   `--address komodo-core-1:9120`: 设置后端服务器的地址和端口。

---

### 完整命令行配置流程

`sozu` 的设计思路是**将监听器、证书和路由规则（前端）分开管理**。配置一个完整的、由 `config.toml` 文件描述的 HTTPS 服务，需要通过命令行分步执行。

假设您的证书和私钥文件位于：
*   证书: `/etc/sozu/certs/certificate.pem`
*   私钥: `/etc/sozu/certs/key.pem`

以下是完整的步骤：

#### 第 1 步: 创建集群 (Cluster)
首先，创建一个名为 `komo` 的集群，并设置其负载均衡策略。

```bash
# 1. 创建集群
sozu --config config.toml cluster add --id komo --load-balancing-policy ROUND_ROBIN
```

#### 第 2 步: 为 HTTPS 添加证书
对于 HTTPS 流量，您必须先将 TLS 证书和私钥关联到将要监听的地址 (`0.0.0.0:443`)。

```bash
# 2. 添加证书
sozu --config config.toml certificate add \
  --address 0.0.0.0:443 \
  --certificate /etc/sozu/certs/certificate.pem \
  --certificate-chain /etc/sozu/certs/certificate.pem \
  --key /etc/sozu/certs/key.pem
```
**说明:**
*   `certificate add`: 定义一个证书添加操作。
*   `--address 0.0.0.0:443`: 指定此证书用于哪个监听地址。
*   `--certificate`, `--certificate-chain`, `--key`: 分别指定证书、证书链和私钥的文件路径。如果证书和链在同一文件，可使用相同路径。

#### 第 3 步: 关联前端 (Frontends)
现在，创建前端规则来告诉 sozu 如何根据 `hostname` 路由来自端口 80 (HTTP) 和 443 (HTTPS) 的流量。

```bash
# 3. 关联前端
sozu --config config.toml frontend http add --address 0.0.0.0:80 --hostname dash.bdev.cn id komo
sozu --config config.toml frontend https add --address 0.0.0.0:443 --hostname dash.bdev.cn id komo
```
**说明:**
*   `frontend http add` / `frontend https add`: 分别为 HTTP 和 HTTPS 定义路由规则。
*   `id komo`: 将匹配此规则的流量路由到 ID 为 `komo` 的集群。

#### 第 4 步: 关联后端 (Backend)
最后，为 `komo` 集群添加后端的服务地址。

```bash
# 4. 关联后端
sozu --config config.toml backend add --id komo --backend-id komodo-core-1 --address komodo-core-1:9120
```
**说明:**
*   `backend add`: 定义一个后端添加操作。
*   `--id komo`: 指定要添加后端的目标集群。
*   `--backend-id ... --address ...`: 定义后端的唯一 ID 和地址。

---
通过以上四个步骤，您就通过命令行完整地复现了 `config.toml` 文件中的配置。
