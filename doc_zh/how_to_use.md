# 如何使用 Sōzu

> 如果您还没有看过[配置文档](./configure.md)，我们建议您先看一下，因为您需要知道在配置文件中放些什么。

## 运行它

如果您使用 `cargo install` 的方式，`sozu` 已经 在您的 `$PATH` 中。

    sozu start -c <path/to/your/config.toml>

但是，如果您从源代码构建项目，`sozu` 会被放置在 `target` 目录中。

    ./target/release/sozu start -c <path/to/your/config.toml>

> `cargo build --release --locked` 将生成的可执行文件放在 `target/release` 而不是 `target/debug` 中。

您可以在[此处][cfg]找到一个可用的 `config.toml` 示例。

要启动反向代理：

```bash
sozu start -c config.toml
```

您可以使用 `config.toml` 文件编辑反向代理的配置。您可以通过该文件声明新的集群、它们的前端和后端。

**但是**为了获得更大的灵活性，您应该使用命令套接字（您可以在配置文件中由 `command_socket` 设计的路径处找到该 unix 套接字的一端）。

您可以使用 `sozu` 二进制文件作为 CLI 与反向代理进行交互。

有关更多信息，请查看命令行[文档](./configure_cli.md)。

## 使用 Docker 运行它

该存储库提供了一个基于 `alpine:edge` 的多阶段 [Dockerfile][df] 镜像。

您可以通过执行以下命令来构建镜像：

    docker build -t sozu .

还有一个 [clevercloud/sozu](https://hub.docker.com/r/clevercloud/sozu/) 镜像
它跟随着 master 分支（已过时）。

使用以下命令运行它：

```bash
docker run \
  --ulimit nofile=262144:262144 \
  --name sozu-proxy \
  -v /run/sozu:/run/sozu \
  -v /path/to/config/file:/etc/sozu \
  -v /my/state/:/var/lib/sozu \
  -p 8080:80 \
  -p 8443:443 \
  sozu
```

要构建具有特定版本 Alpine 的镜像：

    docker build --build-arg ALPINE_VERSION=3.14 -t sozu:main-alpine-3.14 .

### 使用自定义 `config.toml` 配置文件

sozu 的默认配置可以在 `../os-build/docker/config.toml` 中找到。
如果 `/my/custom/config.toml` 是您的自定义配置文件的路径和名称，您可以使用卷启动您的 sozu 容器以覆盖默认配置（请注意，此命令中仅使用自定义配置文件的目录路径）：

    docker run -v /my/custom:/etc/sozu sozu

### 将 sozu 命令行与 docker 容器一起使用

要从主机使用 `sozu` CLI 和 docker 容器，您必须使用 docker 卷将 `/run/sozu` 与主机绑定：

    docker run -v /run/sozu:/run/sozu sozu

要更改配置套接字的路径，请修改配置文件中的 `command_socket` 选项（默认值为 `/var/lib/sozu/sock`）。

### 提供初始配置状态

Sōzu 可以使用 JSON 文件为其路由加载初始配置状态。您可以使用卷挂载它，您可以使用卷启动您的 sozu 容器（请注意，此命令中仅使用自定义配置文件的目录路径）：

    docker run -v /my/state:/var/lib/sozu sozu

要更改已保存状态文件的路径，请修改配置文件中的 `saved_state` 选项（默认值为 `/var/lib/sozu/state.json`）。

[cfg]: ../bin/config.toml
[df]: ../Dockerfile

## Systemd 集成

存储库在[此处][unit-file]提供了一个单元文件。您可以将其复制到 `/etc/systemd/system/` 并调用 `systemctl daemon-reload`。

这将使 systemd 注意到它，现在您可以使用 `systemctl start sozu.service` 启动服务。此外，您可以启用它，以便在将来的启动中默认激活它，使用 `systemctl enable sozu.service`。

[unit-file]: ../os-build/systemd/sozu.service
