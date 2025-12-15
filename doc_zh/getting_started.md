# 入门

## 设置 Rust

确保安装了最新稳定版的 `Rust`。
我们建议为此使用 [rustup][ru]。

完成此操作后，`Rust` 应该已完全安装。

## 设置 Sōzu

### 安装

`sozu` 发布在 [crates.io][cr] 上。

要安装它们，您只需执行 `cargo install sozu`。

它们将被构建并放在 `~/.cargo/bin` 文件夹中。

### 从源代码构建

构建 sozu 可执行文件和命令行：

`cd bin && cargo build --release --locked`

> `--release` 参数通知 cargo 在编译 sozu 时打开优化。
> 仅使用 `--release` 制作生产版本。
>
> `--locked` 标志告诉 cargo 坚持使用 `Cargo.lock` 中指定的依赖项版本
> 从而防止依赖项中断。

[ru]: https://rustup.rs
[cr]: https://crates.io/
