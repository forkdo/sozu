# 如何对 Sōzu 进行基准测试/观察

用于调试和优化目的。

## 在你的机器上运行 Sōzu，并使用虚拟后端

### 创建一个包含 HTTP 和 HTTPS 监听器的配置

`bin/config.toml` 的大多数默认值都是合理的，将它们复制到 `bin/my-config.toml` 中。
一个需要调整的重要值是 `worker_count`。只测试一个
工作进程的性能似乎是理想的：

```toml
worker_count = 1
```

HTTP 和 HTTPS 监听器应该是相同的。
为 HTTPS 监听器选择一个 TLS 版本和一个密码列表，并坚持使用它。

```toml
[[listeners]]
protocol = "http"
address = "0.0.0.0:8080"

[[listeners]]
protocol = "https"
address = "0.0.0.0:8443"
tls_versions = ["TLS_V13"]

cipher_list = [
  # "TLS13_AES_128_GCM_SHA256",
  "TLS13_CHACHA20_POLY1305_SHA256",
  # ...
]
```

创建一个集群（相当于一个应用程序），它有两个前端：
一个用于 HTTP 请求，一个用于 HTTPS 请求。

确保你为 `localhost` 域创建了自己的证书和密钥。

```toml
frontends = [
    { address = "0.0.0.0:8080", hostname = "localhost", path = "/api" },
    { address = "0.0.0.0:8443", hostname = "localhost", certificate = "/path/to/your/certificate.pem", key = "/path/to/your/key.pem", certificate_chain = "/path/to/your/certificate.pem" },
]
```

或者使用 `config.toml` 的证书、密钥和证书链，
它们位于 `/lib/assets/` 中。
但这样你就必须使用 `lolcatho.st` 作为本地域，
并确保你的 `/etc/hosts` 中有以下几行：

```
127.0.0.1       lolcatho.st
::1             lolcatho.st
```

### 生成 4 个后端

创建四个简单的 HTTP 服务器，当通过 HTTP GET 请求 `/api` 路径（任意）时，每个服务器都返回 200 OK。
使用你喜欢的编程语言，它应该很简单。
这是 [8 行 javascript](https://www.digitalocean.com/community/tutorials/how-to-create-a-web-server-in-node-js-with-the-http-module)。
确保每个后端都在 localhost 上监听，并且端口不同，
例如：1051、1052、1053、1054。

将这些后端添加到 `my-config.toml` 的前端下方：

```toml
backends = [
    { address = "127.0.0.1:1051", backend_id = "backend_1" },
    { address = "127.0.0.1:1052", backend_id = "backend_2" },
    { address = "127.0.0.1:1053", backend_id = "backend_3" },
    { address = "127.0.0.1:1054", backend_id = "backend_4" },
]
```

### 构建并运行 Sōzu

在发布模式下构建 Sōzu。

```bash
car go build --release
```

提示：重命名发布二进制文件并将其放入 cargo 路径中：

```bash
mv target/release/sozu $HOME/.cargo/bin/sozu-0.15.14
```

现在你可以在发布模式下运行 Sōzu：

```bash
cd bin
sozu-0.15.14 --config my-config.toml start
```

通过 Sōzu 使用 curl 请求后端来检查 Sōzu 是否工作：

```bash
curl https://localhost:8443/api -v
curl http://localhost:8080/api  -v
```


## Bombardier

[Bombardier](https://github.com/codesenberg/bombardier)
是一个用 go 编写的 HTTP 和 HTTPS 基准测试工具。
它在大多数 linux 发行版中都以软件包的形式提供，
并且在命令行中易于使用。

### 在一个连接上测试大量请求

```bash
bombardier --connections=1 --requests=10000 https://localhost:8443/api --latencies
bombardier --connections=1 --requests=10000 http://localhost:8080/api --latencies
```

### 测试并发连接

```bash
bombardier --connections=100 --requests=10000 https://localhost:8443/api --latencies
bombardier --connections=100 --requests=10000 http://localhost:8080/api --latencies
```

增加连接数，直到 Sõzu 无法承受。

## TLS-perf

[`tls-perf`](https://github.com/tempesta-tech/tls-perf) 是一个用 C 语言编写的 TLS 压力测试工具。
它没有依赖项，并且易于构建。

```bash
git clone git@github.com:tempesta-tech/tls-perf.git
cd tls-perf
make
mv tls-perf $HOME/.local/bin
```

假设 `$HOME/.local/bin` 在你的路径中。

此命令执行：
- 一万次 TLS 握手
- 仅使用一个连接
- 使用 TLS 1.3 版本

```bash
tls-perf \
  -n 10000 \
  -l 1 \
  --sni localhost \
  --tls 1.3 \
  127.0.0.1 8443
```

## 使用 `strace` 查找系统调用

[`strace`](https://github.com/strace/strace) 是一个诊断工具，可以监视
进程（Sōzu）和 linux 内核之间的交互。它在大多数 linux 发行版中都以软件包的形式提供。

Sōzu 运行后，找到工作进程的 pid：

```bash
ps -aux | grep sozu
user   13368  0.7  0.1 188132 40344 pts/2    Sl+  11:35   0:00 /path/to/sozu --config my-config.toml start
user   14157  0.0  0.0  79908 29628 pts/2    S+   11:35   0:00 /path/to/sozu worker --id 0 --fd 5 --scm 7 --configuration-state-fd 3 --command-buffer-size 16384 --max-command-buffer-size 16384
user   14205  0.0  0.0   6556  2412 pts/3    S+   11:35   0:00 grep --color=auto sozu
```

第二行是工作进程，其 pid 是 `14157`。

使用 strace 跟踪此 pid。你可能需要 root 权限。

```bash
strace --attach=14157
```

你应该会看到常规的 `epoll_wait` 系统调用，这是 Sōzu 的主循环。

```
epoll_wait(3, [], 1024, 1000)           = 0
epoll_wait(3, [], 1024, 1000)           = 0
epoll_wait(3, [], 1024, 1000)           = 0
epoll_wait(3, [], 1024, 318)            = 0
```

如果你执行其中一个 curl，你将看到 Sōzu 在
HTTP 或 HTTPS 请求期间执行的所有系统调用。
以下是在一个简单的后端上进行 HTTP GET 期间发生的系统调用。
这只是在监听器套接字上接受流量。

```julia
# 主循环等待事件
epoll_wait(3, [], 1024, 1000)           = 0

# 一个可读事件 (EPOLLIN)。一个客户端连接到监听器套接字。
epoll_wait(3, [{events=EPOLLIN, data={u32=3, u64=3}}], 1024, 1000) = 1

# 代理接受连接，通过告诉监听器接受
# 套接字上的传入连接
accept4(9, {sa_family=AF_INET, sin_port=htons(46144), sin_addr=inet_addr("127.0.0.1")}, [128 => 16], SOCK_CLOEXEC|SOCK_NONBLOCK) = 11

# 完成！没有更多连接要接受
accept4(9, 0x7ffe89e39b98, [128], SOCK_CLOEXEC|SOCK_NONBLOCK) = -1 EAGAIN (Resource temporarily unavailable)

# HTTP 代理在套接字上设置 NODELAY
setsockopt(11, SOL_TCP, TCP_NODELAY, [1], 4) = 0

# Sōzu 在接受队列中注册套接字
# 接受队列在另一个循环中弹出，以创建会话

# 创建会话时，套接字在事件循环中注册
epoll_ctl(6, EPOLL_CTL_ADD, 11, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, data={u32=267, u64=267}}) = 0

# 获取套接字的地址（这里，主机相同，但端口不同）
getpeername(11, {sa_family=AF_INET, sin_port=htons(46144), sin_addr=inet_addr("127.0.0.1")}, [128 => 16]) = 0

# 返回主事件循环
epoll_wait(3, [{events=EPOLLIN|EPOLLOUT, data={u32=267, u64=267}}], 1024, 1000) = 1

# 我们已准备好让流量通过此套接字

# 来了！
recvfrom(11, "GET /api HTTP/1.1\r\nHost: localho"..., 16400, 0, NULL, NULL) = 80
```

### 找出代码的哪些部分导致了系统调用

对于这种挖掘，Sōzu 必须在调试模式下运行，即：
*没有使用 `--release` 标志编译*。执行：

```bash
cargo run -- --config my-config.toml start
```

如上所述找到 pid，然后执行：

```bash
strace --attach=14157 --stack-traces
# or
strace -p 14157 -k
```

curl Sōzu 一次，例如

```bash
curl http://localhost:8080/api  -v
```

现在你可以看到每个系统调用是由哪段代码负责的。
例如，以下是负责在文件描述符为 12 的套接字上进行 `connect` 系统调用的代码。
我们可以看到它是一个 HTTP 会话连接到其后端：

```julia
connect(12, {sa_family=AF_INET, sin_port=htons(1052), sin_addr=inet_addr("127.0.0.1")}, 16) = -1 EINPROGRESS (Operation now in progress)
 > /usr/lib/libc.so.6(__connect+0x14) [0x112454]
 > /path/to/sozu/target/debug/sozu(mio::sys::unix::tcp::connect+0x84) [0x17b2ec4]
 > /path/to/sozu/target/debug/sozu(mio::net::tcp::stream::TcpStream::connect+0x127) [0x17ad9e7]
 > /path/to/sozu/target/debug/sozu(sozu_lib::backends::Backend::try_connect+0x88) [0xb03898]
 > /path/to/sozu/target/debug/sozu(sozu_lib::backends::BackendMap::backend_from_cluster_id+0x465) [0xb04145]
 > /path/to/sozu/target/debug/sozu(sozu_lib::protocol::kawa_h1::Http<Front,L>::get_backend_for_sticky_session+0x5ab) [0xca8beb]
 > /path/to/sozu/target/debug/sozu(sozu_lib::protocol::kawa_h1::Http<Front,L>::backend_from_request+0x159) [0xca69b9]
 > /path/to/sozu/target/debug/sozu(sozu_lib::protocol::kawa_h1::Http<Front,L>::connect_to_backend+0xb7a) [0xcaafba]
 > /path/to/sozu/target/debug/sozu(sozu_lib::protocol::kawa_h1::Http<Front,L>::ready_inner+0x423) [0xcaf3a3]
 > /path/to/sozu/target/debug/sozu(<sozu_lib::protocol::kawa_h1::Http<Front,L> as sozu_lib::protocol::SessionState>::ready+0x2d) [0xcb040d]
 > /path/to/sozu/target/debug/sozu(<sozu_lib::http::HttpStateMachine as sozu_lib::protocol::SessionState>::ready+0xf2) [0xdbb9b2]
 > /path/to/sozu/target/debug/sozu(<sozu_lib::http::HttpSession as sozu_lib::ProxySession>::ready+0x114) [0xdb4b54]
 > /path/to/sozu/target/debug/sozu(sozu_lib::server::Server::ready+0x746) [0xc95846]
 > /path/to/sozu/target/debug/sozu(sozu_lib::server::Server::run+0x66d) [0xc8869d]
 > /path/to/sozu/target/debug/sozu(sozu::worker::begin_worker_process+0x3060) [0x32e480]
 > /path/to/sozu/target/debug/sozu(sozu::main+0x42e) [0x556afe]
 > /path/to/sozu/target/debug/sozu(core::ops::function::FnOnce::call_once+0xb) [0x1e453b]
 > /path/to/sozu/target/debug/sozu(std::sys_common::backtrace::__rust_begin_short_backtrace+0xe) [0x287e6e]
 > /path/to/sozu/target/debug/sozu(std::rt::lang_start::{{closure}}+0x11) [0x2ff2e1]
 > /path/to/sozu/target/debug/sozu(std::rt::lang_start_internal+0x42e) [0x18217fe]
 > /path/to/sozu/target/debug/sozu(std::rt::lang_start+0x3a) [0x2ff2ba]
 > /path/to/sozu/target/debug/sozu(main+0x1e) [0x556f6e]
 > /usr/lib/libc.so.6(__libc_init_first+0x90) [0x27cd0]
 > /usr/lib/libc.so.6(__libc_start_main+0x8a) [0x27d8a]
 > /path/to/sozu/target/debug/sozu(_start+0x25) [0x17bca5]
```