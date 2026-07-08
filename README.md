# Linux Rust 内核模块实验

在 Linux 7.0 内核上编译并加载 Rust 树外模块（out-of-tree module）的完整实验记录。

> 如果只需要快速概览，可直接跳到 [命令速查](#命令速查)。

## 项目结构

```text
.
├── linux-rust-kernel-module.md   # 原始完整实验笔记
├── README.md                       # 本文件（整合后的完整指南）
└── rust-mod/                       # Rust 内核模块源码
    ├── Kbuild
    ├── Makefile
    ├── README.md
    ├── rust_out_of_tree.rs
    └── LICENSE
```

## 实验环境

| 项 | 值 |
|------|------|
| 机器 | `test102` (`192.168.x.x:19200`)，用户 `lhcloud` |
| OS | Debian 13 (bookworm) |
| 原内核 | `7.0.10+deb13-amd64`（发行版） |
| 自编译内核 | `7.0.0+deb13-amd64`（vanilla 7.0 from kernel.org） |
| Rust 工具链 | rustc 1.85.0（stable，via rustup） |
| bindgen | 0.72.1 |
| 内核源码 | `~/linux-7.0/` |
| 模块源码 | `~/rust-mod/`（即本项目的 `rust-mod/` 目录） |

## 背景

Linux 7.0 已支持使用 Rust 编写内核模块（`CONFIG_RUST=y`）。但 Debian 13 发行版内核**未启用**该选项，且发行版的 `linux-headers` 包不包含 Rust 编译所需的元数据（`rust/` 目录）。因此必须下载完整内核源码自行编译。

## 1. 安装依赖

### 内核编译依赖

```bash
sudo apt install -y build-essential flex bison bc rsync \
  libssl-dev libelf-dev cpio dwarves
```

### Rust 工具链

```bash
# 安装 rustup（不要用 apt）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# 安装 bindgen 和 rust-src
cargo install bindgen-cli --version 0.72.1
rustup component add rust-src

# 锁定 1.85.0 版本
# 原因：nightly 1.87 的 "extern blocks must be unsafe" 变更会导致内核 Rust 代码编译失败
# 发行版（Debian 13）也是用 1.85.0 构建内核，锁定此版本保证 ABI 兼容
rustup install 1.85.0
rustup default 1.85.0
```

验证：

```bash
rustc --version                         # rustc 1.85.0 (4d91de4e4 2025-02-17)
bindgen --version                       # bindgen 0.72.1
rustup component list --installed | grep rust-src
```

## 2. 下载内核源码

发行版 headers 缺少 `rust/` 目录，必须用完整内核源码：

```bash
mkdir -p ~/linux-7.0
cd ~/linux-7.0

# 从阿里云镜像下载（国内更快）
wget https://mirrors.aliyun.com/linux-kernel/v7.x/linux-7.0.tar.gz
tar xzf linux-7.0.tar.gz --strip-components=1
```

## 3. 配置内核

```bash
cd ~/linux-7.0

# 以当前运行内核配置为基础
cp /boot/config-$(uname -r) .config

# 设置本地版本号（必须匹配发行版命名约定，否则模块 vermagic 不匹配）
sed -i 's/CONFIG_LOCALVERSION=.*/CONFIG_LOCALVERSION="+deb13-amd64"/' .config

# 刷新默认配置
make olddefconfig

# 检查 Rust 是否可用
source $HOME/.cargo/env
make rustavailable
# 输出应为：Rust is available!

# 启用 Rust 支持（关键步骤！rustavailable 通过不代表 CONFIG_RUST 已设置）
./scripts/config --enable CONFIG_RUST
make olddefconfig

# 验证
grep 'CONFIG_RUST=' .config
# 必须输出：CONFIG_RUST=y
```

## 4. 编译内核

```bash
cd ~/linux-7.0
source $HOME/.cargo/env

# 首次编译约 30-60 分钟，增量约 2-5 分钟
make -j$(nproc)

# 验证 Rust 符号
nm vmlinux | grep -c 'rust_\|_RNv'
# 应输出：15906（大约）
```

## 5. 安装内核

```bash
sudo make modules_install
sudo make install

# update-grub 会自动执行，检查生成的启动项
sudo grep -E 'menuentry|submenu' /boot/grub/grub.cfg
```

## 6. 配置 GRUB 启动自定义内核

在云 VM 上，自定义内核通常挂在 "Advanced options" 子菜单下，需要用 `>` 语法：

```bash
# 查看确切的 menuentry 名称
sudo grep 'menuentry.*7\.0\.0' /boot/grub/grub.cfg

# 编辑 grub 默认项
sudo vi /etc/default/grub
```

关键配置：

```ini
# 注意子菜单用 ">" 分隔
GRUB_DEFAULT="Advanced options for Debian GNU/Linux>Debian GNU/Linux, with Linux 7.0.0+deb13-amd64"
GRUB_SAVEDEFAULT=true
```

然后更新并重启：

```bash
sudo update-grub
sudo reboot
```

重启后验证：

```bash
uname -r
# 应输出：7.0.0+deb13-amd64

grep CONFIG_RUST= /boot/config-$(uname -r)
# 应输出：CONFIG_RUST=y
```

## 7. 编译 Rust 树外模块

进入本项目的 `rust-mod/` 目录：

```bash
cd rust-mod
source $HOME/.cargo/env
make
```

预期输出：

```text
  RUSTC [M] rust_out_of_tree.o
  MODPOST Module.symvers
  CC [M]  .module-common.o
  LD [M]  rust_out_of_tree.ko
  BTF [M] rust_out_of_tree.ko
```

`rust-mod/Makefile` 内容：

```makefile
# SPDX-License-Identifier: GPL-2.0

KDIR ?= /lib/modules/$(shell uname -r)/build

default:
	$(MAKE) -C $(KDIR) M=$$PWD modules

modules_install: default
	$(MAKE) -C $(KDIR) M=$$PWD modules_install

clean:
	$(MAKE) -C $(KDIR) M=$$PWD clean
```

`rust-mod/rust_out_of_tree.rs` 是一个示例模块，使用 `kernel::prelude::*`、`module!` 宏和 `KVec` 分配内核内存。

## 8. 加载和验证

```bash
cd rust-mod
sudo insmod rust_out_of_tree.ko

lsmod | grep rust
# rust_out_of_tree       12288  0

sudo dmesg | tail -3
# rust_out_of_tree: Rust out-of-tree sample (init)
# rust_out_of_tree: My numbers are [72, 108, 200]

# 卸载
sudo rmmod rust_out_of_tree
sudo dmesg | tail -1
# rust_out_of_tree: Rust out-of-tree sample (exit)
```

## 9. 跨机器加载 Rust 模块

Rust `.ko` 不能像 C 模块那样随意跨机器加载。除 vermagic / modversions 等通用限制外，Rust 额外多了 ABI 不稳定的约束。

### 加载条件

| 条件 | 说明 |
|------|------|
| `CONFIG_RUST=y` | 目标内核必须启用 Rust 运行时，否则 Rust 符号全部未定义 |
| 内核版本一致 | vermagic 必须精确匹配，如 `7.0.0+deb13-amd64` |
| **rustc 版本一致** | 内核编译用的 rustc 和模块编译用的必须**完全相同** |
| 架构一致 | x86_64 ↔ x86_64，ARM64 ↔ ARM64 |
| `CONFIG_MODVERSIONS` | 如果开启，符号 CRC 必须匹配 |

### 为什么 rustc 版本必须一致

Rust 没有稳定 ABI。模块中引用的内核符号都是 mangled 名称（如 `_RNvNtCsaRPFapPOzLs_6kernel5print11call_printk`），里面编码了编译器版本和 crate 哈希。内核如果换了 rustc 版本重新编译，导出的符号名会变，模块加载时直接找不到符号。

这与 C 模块不同——C 有稳定 ABI，换 gcc 版本通常不影响。Rust 模块的二进制兼容性严格绑定到构建内核时使用的那一个 rustc。

### 查看内核/模块的 rustc 版本

```bash
# 内核编译用的版本
grep CONFIG_RUSTC_VERSION_TEXT /boot/config-$(uname -r)
# CONFIG_RUSTC_VERSION_TEXT="rustc 1.85.0 (4d91de4e4 2025-02-17)"

# 模块编译用的版本
modinfo rust_out_of_tree.ko | grep vermagic
```

### 当前 test102 环境

```text
内核 rustc: 1.85.0
模块 rustc: 1.85.0
vermagic:   7.0.0+deb13-amd64 SMP preempt mod_unload
```

## 踩坑记录

| # | 问题 | 根因 | 解决 |
|------|------|------|------|
| 1 | 发行版 `linux-headers` 无 `rust/` 目录 | 发行版头文件不包含 Rust 编译元数据 | 下载完整内核源码 |
| 2 | `CONFIG_RUST` 默认关闭 | `rust_is_available.sh` 通过 ≠ `CONFIG_RUST=y` | `./scripts/config --enable CONFIG_RUST` + `make olddefconfig` |
| 3 | nightly rustc 1.87 编译失败 | `extern blocks must be unsafe` 语法变化 | 换 stable 1.85.0 |
| 4 | `LLVM=1` 编译失败 | 需要 clang | 去掉该标志，用 GCC 编 C + rustc 编 Rust |
| 5 | `/tmp` 空间不足（2GB） | tmpfs 太小 | 源码和编译目录移到 `/home/lhcloud/`（50GB） |
| 6 | 模块 vermagic `7.0.0` 不匹配 `7.0.0+deb13-amd64` | 内核未设置 LOCALVERSION | `CONFIG_LOCALVERSION="+deb13-amd64"` |
| 7 | GRUB 不启动自定义内核 | `GRUB_DEFAULT` 指向了子菜单条目但未用 `>` 语法 | 改为 `"Advanced options...>Debian GNU/Linux, with Linux 7.0.0+deb13-amd64"` |
| 8 | 模块编译时 `undefined` Rust 符号 | `CONFIG_RUST` 未启用，内核不导出 Rust 运行时符号 | 启用 `CONFIG_RUST=y` 并重新编译内核 |

## 命令速查

```bash
# SSH 连接（示例，请替换为实际 IP 和端口）
ssh -p 19200 lhcloud@192.168.x.x

# 检查内核
uname -r
grep CONFIG_RUST= /boot/config-$(uname -r)

# 编译模块
cd rust-mod && source ~/.cargo/env && make

# 加载/卸载
sudo insmod rust_out_of_tree.ko
sudo rmmod rust_out_of_tree
sudo dmesg | tail -5
```

## 参考链接

- [Rust for Linux](https://rust-for-linux.com)
- [Kernel Rust Documentation](https://docs.kernel.org/rust/)
- [Out-of-tree Modules](https://docs.kernel.org/kbuild/modules.html)

## License

本项目源码采用 **GPL-2.0** 许可证，与 Linux 内核 Rust 示例模块保持一致。

```text
SPDX-License-Identifier: GPL-2.0
```
