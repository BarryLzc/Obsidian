## 1. 基础开发环境 (Linux 端)

|**依赖项**|**作用与原因**|
|---|---|
|**Rust Nightly**|**核心编译器**。Aya 依赖 Nightly 版本的特性（如 `build-std`）来为 eBPF 这种 `no_std` 环境编译核心库。|
|**rust-src**|**源码包**。编译 eBPF 时需要重新编译 Rust `core` 库，必须有 Rust 源码才能按需构建。|
|**bpf-linker**|**链接器**。普通的链接器无法处理 eBPF 指令集，`bpf-linker` 专门负责将 Rust 编译后的产物转换为内核能理解的 eBPF 字节码。|

## 2. 系统编译工具 (Linux 端)

eBPF 本质上是内核代码，需要与 C 语言环境打交道：

- **LLVM & Clang**: eBPF 的后端技术栈基于 LLVM。虽然你写的是 Rust，但底层的编译和优化逻辑依赖 LLVM。

- **libelf-dev**: 读写 ELF 文件的开发库。eBPF 字节码存储在 ELF 格式的文件中，用户态程序加载它时需要解析这些文件。

- **pkg-config**: 帮助 Cargo 找到系统库（如 `libelf`）的安装位置。


Bash

```
# 安装命令参考
sudo apt install llvm clang libelf-dev pkg-config build-essential
```

## 3. 权限与运行环境 (Linux 端)

- **Root 权限 (sudo)**: eBPF 程序涉及内核操作。出于安全考虑，只有超级用户才能通过 `bpf()` 系统调用将程序加载到内核挂钩点（如 XDP 或 Kprobe）。

- **现代 Linux 内核 (5.4+)**: 虽然旧内核也支持 eBPF，但 Aya 使用的很多高级特性（如 BTF 调试信息）需要较新版本的内核支持。

## 4. 开发交互工具 (Mac & CLion)

- **Deployment (SFTP)**: 解决物理隔阂。因为 Mac 无法直接运行 eBPF，必须通过 SFTP 将代码实时同步到 Linux 编译。
- **SSH 配置**: CLion 通过 SSH 协议远程调用 Linux 上的工具链和终端。

## 总结：整个工作流是如何流转的？

1. **Mac**: 你在 CLion 写下 Rust 代码。

2. **Deployment**: 代码通过 SFTP 传送到 Linux 目录。

3. **Cargo (User space)**: 在 Linux 上触发编译。
	- **build.rs** 调用 **Nightly rustc** 和 **rust-src**。
	- **bpf-linker** 将内核态代码压制成 `.elf` 字节码。
4. **Sudo**: 用户态程序启动，利用 **libelf** 解析字节码，通过 `sudo` 权限将其塞进 Linux 内核。（sudo -E RUST_LOG=info /home/it/.cargo/bin/cargo run -p my_ebpf --config 'target."cfg(all())".runner="sudo -E"'）