# ARM SDK Repo 使用手册

本文档说明本仓库的目录结构、环境准备、代码同步、构建、镜像制作和 QEMU 启动流程。命令默认在仓库根目录执行：

```bash
cd /home/hl/code/arm_sdk
```

## 1. 仓库概览

本 SDK 使用 Google `repo` 管理多个子仓库，manifest 位于 `.repo/manifests/default.xml`。当前 manifest 默认从 `git@github.com:huangliang367/` 拉取 `main` 分支，包含以下项目：

| 路径 | 说明 |
| --- | --- |
| `linux/` | Linux kernel 源码 |
| `u-boot/` | U-Boot 源码 |
| `scripts/` | 构建、rootfs、镜像和 QEMU 运行脚本 |
| `dockerfile/` | 构建环境 Dockerfile 和容器启动脚本 |
| `prebuild/` | 预构建资源，例如 Debian rootfs 缓存包 |
| `build/` | 本地构建输出目录 |
| `docs/` | 项目文档 |

顶层 `Makefile` 提供完整流水线：

```bash
make all
```

它会依次执行 `repo` 检查、manifest 同步、Docker 镜像准备、Docker 容器启动和项目构建。

注意：`docker-run` 是交互式容器入口。执行 `make all` 时，Makefile 会先进入容器；退出容器后才会继续执行后续 `build` 目标。日常开发更推荐手动进入容器后在容器内执行 `scripts/` 下的构建脚本。

## 2. 环境准备

### 2.1 宿主机依赖

宿主机至少需要：

- Git 和 SSH key，能够访问 `git@github.com:huangliang367/manifest.git`
- `make`
- Docker
- `sudo` 权限，供安装 Docker、安装 `repo` 或操作镜像时使用

如果宿主机没有 `repo` 或 Docker，可以使用顶层 Makefile 自动检查和安装：

```bash
make repo
make docker-setup
```

如网络环境需要代理，可传入 `PROXY`：

```bash
make PROXY=http://proxy.example.com:7890 all
```

### 2.2 Docker 构建环境

Docker 镜像由 `dockerfile/Dockerfile` 定义，默认镜像名为：

```text
qemu-arm64-build:latest
```

镜像基于 Ubuntu 22.04，内置 ARM64 交叉编译工具链、QEMU、U-Boot tools、debootstrap、mtools、parted 等构建和镜像制作工具。

单独构建镜像：

```bash
bash dockerfile/build.sh
```

强制重建镜像：

```bash
docker build -t qemu-arm64-build:latest dockerfile
```

启动容器：

```bash
bash dockerfile/run.sh
```

容器启动参数会将当前仓库路径挂载到容器内相同路径，并以 `hl` 用户进入工作目录。

## 3. 同步代码

首次初始化和同步：

```bash
make manifest
```

如果 `.repo/` 已存在，`make manifest` 会跳过 `repo init`，直接执行：

```bash
repo sync
```

手动初始化 manifest：

```bash
repo init -u git@github.com:huangliang367/manifest.git
repo sync
```

## 4. 构建

推荐在 Docker 容器内执行构建命令，避免宿主机工具链差异。

### 4.1 一键构建 U-Boot 和 Kernel

```bash
bash scripts/build.sh
```

等价于：

```bash
bash scripts/build.sh all
```

产物包括：

| 产物 | 路径 |
| --- | --- |
| U-Boot binary | `build/uboot/u-boot.bin` |
| Linux Image | `build/kernel/arch/arm64/boot/Image` |
| initramfs | `build/kernel/initrd.cpio.gz` |

### 4.2 只构建 U-Boot

```bash
bash scripts/build.sh uboot
```

或直接执行：

```bash
bash scripts/build_uboot.sh
```

U-Boot 默认使用 `qemu_arm64_defconfig`，并额外打开 UDP fastboot、MMC、GPT、bootflow 等配置。

默认交叉编译前缀为：

```bash
aarch64-none-elf-
```

如工具链不在 `PATH` 中，可指定：

```bash
export CROSS_COMPILE_ELF=/path/to/toolchain/bin/aarch64-none-elf-
bash scripts/build_uboot.sh
```

Docker 镜像中会将 `aarch64-linux-gnu-*` 软链接为 `aarch64-none-elf-*`，通常不需要额外设置。

### 4.3 只构建 Linux Kernel

```bash
bash scripts/build.sh kernel
```

或分步执行：

```bash
bash scripts/build_kernel.sh defconfig
bash scripts/build_kernel.sh build
bash scripts/build_kernel.sh initrd
```

Kernel 默认架构为 `arm64`，交叉编译前缀为：

```bash
aarch64-linux-gnu-
```

如工具链不在 `PATH` 中，可指定：

```bash
export CROSS_COMPILE_GNU=/path/to/toolchain/bin/aarch64-linux-gnu-
bash scripts/build_kernel.sh build
```

`defconfig` 阶段会额外启用 ext4、MMC、SDHCI PCI 等用于 QEMU MMC rootfs 启动的配置。

## 5. 构建 Debian Rootfs

`scripts/build_rootfs.sh` 用于生成 ARM64 Debian Bookworm rootfs。默认输出目录：

```text
build/debian-rootfs
```

执行：

```bash
bash scripts/build_rootfs.sh
```

也可以指定输出目录：

```bash
bash scripts/build_rootfs.sh build/my-rootfs
```

脚本会优先使用缓存：

```text
prebuild/debian-bookworm-arm64-rootfs.tar.gz
```

如果缓存不存在，脚本会使用 `debootstrap --foreign` 从阿里云 Debian 镜像构建 rootfs，并配置：

- Debian suite: `bookworm`
- Hostname: `debian-qemu`
- Root password: 空密码，开发环境使用
- rootfs 分区: `/dev/mmcblk0p2`
- boot 分区: `/dev/mmcblk0p1`
- 网络: `eth0` DHCP
- 串口登录: `ttyAMA0 @ 115200`

注意：rootfs 构建和解包需要 `sudo`，因为脚本会执行 chroot、tar 解包、文件属主保留等操作。

## 6. 制作 MMC 镜像

构建完 U-Boot、Kernel、initramfs 和 rootfs 后，制作 QEMU 使用的 MMC 镜像：

```bash
bash scripts/make_mmc.sh
```

默认输出：

```text
build/mmc.img
```

默认输入：

| 输入 | 路径 |
| --- | --- |
| Kernel Image | `build/kernel/arch/arm64/boot/Image` |
| initramfs | `build/kernel/initrd.cpio.gz` |
| DTB | `build/uboot/arch/arm/dts/qemu-arm64.dtb` |
| Debian rootfs | `build/debian-rootfs` |

也可以指定镜像路径和 rootfs 路径：

```bash
bash scripts/make_mmc.sh build/mmc.img build/debian-rootfs
```

镜像布局：

| 分区 | 偏移 | 大小 | 文件系统 | 内容 |
| --- | --- | --- | --- | --- |
| `mmcblk0p1` | 2 MiB | 64 MiB | FAT32 | `/image.fit`, `/boot.scr` |
| `mmcblk0p2` | 66 MiB | 2048 MiB | ext4 | Debian rootfs |

脚本会生成 FIT Image，并写入 U-Boot boot script。rootfs 分区写入过程需要 `sudo losetup`、`mount` 和 `cp -a`。

## 7. 运行 QEMU

完整启动 U-Boot + MMC 镜像：

```bash
bash scripts/run_qemu.sh
```

默认使用 user networking，不需要 root：

```bash
bash scripts/run_qemu.sh user
```

启动后按以下组合键退出 QEMU：

```text
Ctrl-A X
```

`run_qemu.sh` 会检查以下文件：

| 文件 | 说明 |
| --- | --- |
| `build/uboot/u-boot.bin` | QEMU BIOS/U-Boot |
| `build/mmc.img` | MMC 磁盘镜像 |
| `build/envstore.img` | U-Boot 持久化环境 pflash，首次运行自动创建 |

网络模式：

| 模式 | 命令 | 说明 |
| --- | --- | --- |
| user | `bash scripts/run_qemu.sh user` | 默认模式，SLIRP，fastboot 使用 `127.0.0.1` |
| tap | `bash scripts/run_qemu.sh tap` | 需要宿主机预先配置 `tap0` bridge |

user 模式下 fastboot 地址：

```bash
fastboot -s udp:127.0.0.1 <cmd>
```

tap 模式下 fastboot 地址：

```bash
fastboot -s udp:<QEMU_IP> <cmd>
```

## 8. 常用工作流

### 8.1 从零同步并进入 Docker 环境

```bash
make manifest
bash dockerfile/build.sh
bash dockerfile/run.sh
```

进入容器后继续：

```bash
bash scripts/build.sh all
bash scripts/build_rootfs.sh
bash scripts/make_mmc.sh
bash scripts/run_qemu.sh
```

### 8.2 修改 Kernel 后重编并启动

```bash
bash scripts/build_kernel.sh build
bash scripts/build_kernel.sh initrd
bash scripts/make_mmc.sh
bash scripts/run_qemu.sh
```

如果改动涉及 kernel config，先执行：

```bash
bash scripts/build_kernel.sh defconfig
```

### 8.3 修改 U-Boot 后重编并启动

```bash
bash scripts/build_uboot.sh
bash scripts/make_mmc.sh
bash scripts/run_qemu.sh
```

如果只改 U-Boot 且 MMC 镜像内容未变，通常可以跳过 `make_mmc.sh`：

```bash
bash scripts/build_uboot.sh
bash scripts/run_qemu.sh
```

### 8.4 仅运行 Kernel 直接启动测试

`build_kernel.sh` 提供了直接以 `-kernel` 方式启动 QEMU 的入口：

```bash
bash scripts/build_kernel.sh run
```

或一键 defconfig、build、initrd、run：

```bash
bash scripts/build_kernel.sh all
```

该方式不经过 U-Boot，也不加载 `build/mmc.img`。

## 9. 常见问题

### 9.1 `Toolchain not found`

U-Boot 需要：

```text
aarch64-none-elf-gcc
```

Kernel 需要：

```text
aarch64-linux-gnu-gcc
```

优先在 Docker 容器中构建。如果使用宿主机工具链，分别设置：

```bash
export CROSS_COMPILE_ELF=/path/to/toolchain/bin/aarch64-none-elf-
export CROSS_COMPILE_GNU=/path/to/toolchain/bin/aarch64-linux-gnu-
```

### 9.2 `repo sync` 失败

检查 SSH key 是否可以访问 GitHub：

```bash
ssh -T git@github.com
```

如需代理：

```bash
make PROXY=http://proxy.example.com:7890 manifest
```

### 9.3 Docker 需要 sudo

脚本会自动检测 `docker info`，如果当前用户不能直接访问 Docker daemon，会改用 `sudo docker`。

也可以把当前用户加入 docker 组后重新登录：

```bash
sudo usermod -aG docker "$USER"
```

### 9.4 rootfs 构建慢或失败

优先确认 `prebuild/debian-bookworm-arm64-rootfs.tar.gz` 是否存在。存在时脚本会直接解包缓存，避免重新 debootstrap。

如果需要重新构建，确认宿主机或容器中有：

```text
debootstrap
qemu-user-static
sudo
```

### 9.5 QEMU 找不到 MMC 镜像

先按顺序生成依赖产物：

```bash
bash scripts/build.sh all
bash scripts/build_rootfs.sh
bash scripts/make_mmc.sh
```

然后再运行：

```bash
bash scripts/run_qemu.sh
```

### 9.6 QEMU 退出方式

QEMU 使用 `-nographic`，退出不是 `Ctrl-C`，而是：

```text
Ctrl-A X
```

## 10. 产物清单

常见构建产物如下：

| 路径 | 说明 |
| --- | --- |
| `build/uboot/u-boot.bin` | U-Boot binary |
| `build/uboot/arch/arm/dts/qemu-arm64.dtb` | U-Boot 生成的 QEMU ARM64 DTB |
| `build/kernel/arch/arm64/boot/Image` | Linux kernel Image |
| `build/kernel/initrd.cpio.gz` | initramfs |
| `build/debian-rootfs/` | Debian ARM64 rootfs |
| `build/mmc.img` | QEMU MMC 磁盘镜像 |
| `build/envstore.img` | QEMU pflash，用于保存 U-Boot 环境 |
