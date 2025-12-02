# 入门

## 设置 Rust

确保已安装最新稳定版的 `Rust`。
我们建议为此使用 [rustup][ru]。

完成此操作后，`Rust` 应已完全安装。

## 设置 Sōzu

### 安装

`sozu` 已发布在 [crates.io][cr] 上。

要安装它们，您只需执行 `cargo install sozu`。

它们将被构建并放在 `~/.cargo/bin` 文件夹中。

### 从源代码构建

构建 sozu 可执行文件和命令行：

`cd bin && cargo build --release --locked`

> `--release` 参数通知 cargo 在编译 sozu 时启用优化。
> 仅使用 `--release` 来制作生产版本。
>
> `--locked` 标志告诉 cargo 遵循 `Cargo.lock` 中指定的依赖项版本
> 并因此防止依赖项中断。

[ru]: https://rustup.rs
[cr]: https://crates.io/
