# OpenTina 构建系统

本文档描述 `opentina-org/build` 仓库的构建流程：环境准备、源码获取、配置选择、
构建命令、产物、烧写与验收。

文档基线：`build` 仓库 `tina-dev` 分支，commit `ed590a8`（2026-09-18）。
覆盖板型为 Allwinner A733（sun60i）的 `radxa_a7a` 与 `demo_aiot_a733_v3`。
其它芯片未纳入当前构建脚本。

## 1. 构建环境

### 1.1 宿主机要求

x86_64 的 Debian 或 Ubuntu。构建过程中的 Debian/Ubuntu rootfs 与部分镜像组装
步骤依赖 Docker；Yocto 与 Buildroot 在宿主机上直接执行。

### 1.2 主机软件包

```shell
sudo apt install -y \
    bc bison build-essential ca-certificates chrpath cpio device-tree-compiler \
    diffstat dosfstools flex g++-aarch64-linux-gnu gcc-aarch64-linux-gnu \
    genimage git libgnutls28-dev libssl-dev make mtools patch perl \
    python3 python3-cryptography python3-pyelftools python3-setuptools \
    rpcsvc-proto rsync swig texinfo u-boot-tools wget xz-utils
```

其中 `chrpath`、`diffstat`、`texinfo`（提供 `makeinfo`）、`rpcsvc-proto`
（提供 `rpcgen`）属于 BitBake 的 `HOSTTOOLS` 要求，仅在构建 Yocto 组件时需要。
`python3-pyelftools` 与 `python3-cryptography` 由 OP-TEE 的镜像生成与 TA 签名
脚本使用，缺失时 `optee` 组件会在链接阶段失败。

若不使用 Docker 而在宿主机直接构建 Debian 或 Ubuntu rootfs，另需：

```shell
sudo apt install -y debootstrap qemu-user-static binfmt-support
```

### 1.3 Ubuntu 24.04 及以上的 user namespace 限制

AppArmor 默认限制无特权 user namespace，BitBake 会报
`User namespaces are not usable`。一次性放开：

```shell
echo 'kernel.apparmor_restrict_unprivileged_userns=0' | sudo tee /etc/sysctl.d/99-bitbake-userns.conf
sudo sysctl -p /etc/sysctl.d/99-bitbake-userns.conf
```

### 1.4 交叉工具链

默认使用宿主机的 `aarch64-linux-gnu-` 前缀工具链，用于 TF-A、U-Boot、Linux 和
OP-TEE。板级变量 `OPENTINA_CROSS_COMPILE` 可覆盖。Buildroot 不使用该工具链，
它自行下载 Bootlin 外部工具链。

## 2. 获取源码

```shell
cd build
./build.sh init
```

`init` 解析 `scripts/opentina-manifest.xml` 并克隆其中的项目到 `sources/`。
已存在的目录只跳过，不校验 URL 或分支。

| 仓库 | 路径 | 用途 |
|---|---|---|
| `trusted-firmware-a` | `sources/trusted-firmware-a` | BL31 |
| `optee_os` | `sources/optee_os` | BL32（OP-TEE OS）与 TA devkit |
| `u-boot` | `sources/u-boot` | SPL 与 U-Boot，打包 BL31/BL32 进 FIT |
| `linux` | `sources/linux` | 内核 Image 与设备树 |
| `awbin` | `sources/awbin` | boot0、U-Boot header 与打包工具 |
| `buildroot` | `sources/buildroot` | Buildroot rootfs |
| `debian` | `sources/debian` | Debian rootfs |
| `ubuntu` | `sources/ubuntu` | Ubuntu rootfs |
| `meta-opentina` | `sources/meta-opentina` | Yocto layer |
| `openwrt` | `sources/openwrt` | OpenWrt rootfs |
| `docs` | `sources/docs` | 文档，不参与固件构建 |

manifest 的 revision 可以是分支、tag 或 commit sha。写 sha 时 `repo_clone.sh`
会先完整克隆再 detach 到该 commit，且 `--sync` 不会执行 `pull`。

同步已有仓库：

```shell
./scripts/repo_clone.sh --sync ./scripts/opentina-manifest.xml
```

`--sync` 的 `pull --ff-only` 失败会被忽略，因此它不构成严格可复现的同步机制。

## 3. 构建命令

### 3.1 语法

```
./build.sh <BOARD_NAME> [ROOTFS] <build|clean> [COMPONENT...]
```

`BOARD_NAME` 必须与某个 `configs/*/config` 中的 `BOARD_NAME` 一致。省略
`ROOTFS` 等价于 `buildroot`。省略 `COMPONENT` 时按顺序构建整条组件链。

`./build.sh targets` 列出板型与 rootfs 类型。

### 3.2 组件链

开启 OP-TEE 时的顺序为：

```
optee -> atf -> uboot -> linux -> <rootfs> -> bootfs -> image
```

关闭 OP-TEE 时 `optee` 从链中移除，其余不变。`<rootfs>` 由 `ROOTFS` 参数决定，
取值为 `br2`、`debian`、`ubuntu`、`yocto`、`openwrt` 之一。

各组件的依赖以 `output/<BOARD>/.done.<组件>` 的存在为准，依赖不满足时报错退出。
`.done` 只记录某次组件命令成功结束，不保存输入摘要，因此源码、配置或工具链变化
不会使它失效，也不会触发自动重建。

### 3.3 常用调用

```shell
# 完整构建，默认 Buildroot rootfs
./build.sh radxa_a7a build

# 指定 rootfs
./build.sh radxa_a7a debian build
./build.sh radxa_a7a yocto build

# 只构建单个组件
./build.sh radxa_a7a build linux
./build.sh radxa_a7a yocto build yocto

# 清理
./build.sh radxa_a7a clean
./build.sh radxa_a7a clean linux
```

### 3.4 OP-TEE 开关

OP-TEE 为整项目可选，默认开启。

```shell
./build.sh --no-optee radxa_a7a build
OPENTINA_OPTEE=0 ./build.sh radxa_a7a build
```

关闭时 TF-A 以 `SPD=none` 构建，U-Boot 不打包 BL32，内核设备树中的 TZDRAM/SHM
预留节点与 `firmware/optee` 节点被删除，Buildroot 侧的 OP-TEE 用户态包被禁用并
清理已安装文件。开关切换会强制重建 TF-A，因为 TF-A 的增量构建不会因 `SPD=`
变化而重新编译平台目标文件。

### 3.5 并行度

环境变量 `JOBS` 控制并行数，未设置时取 `nproc`，并写入 `MAKEFLAGS=-j$JOBS`。
Yocto 的并行度另由 `sources/meta-opentina` 的 `local.conf.sample` 控制。

在内存受限的主机上建议同时限制并行度与内存上限，例如：

```shell
systemd-run --user --scope -p MemoryMax=20G -p MemorySwapMax=0 -p CPUQuota=800% \
    -- env JOBS=4 ./build.sh radxa_a7a build
```

### 3.6 Docker 构建环境

```shell
./build.sh --docker radxa_a7a build    # 单次命令在容器内执行
export OPENTINA_DOCKER=1               # 整个 shell 会话在容器内构建
./build.sh --docker-shell              # 只进入容器交互 shell
```

`--docker` 与 `--docker-shell` 必须出现在子命令之前。容器镜像默认
`opentina-buildenv:24.04`，不存在时由 `scripts/docker-exec.sh` 依据
`docker/Dockerfile` 构建。该镜像未预装完整的 Yocto 宿主依赖，`yocto` 组件
建议在宿主机构建。

## 4. 配置项

### 4.1 板级配置

每块板一个目录 `configs/<目录名>/`，其中 `config` 定义下列变量：

| 变量 | 含义 |
|---|---|
| `BOARD_NAME` | 命令行第一个参数使用的名字，同时决定 `output/` 子目录 |
| `PRETTY_NAME` | 仅用于 `./build.sh targets` 显示 |
| `OPENTINA_CROSS_COMPILE` | 交叉编译前缀，默认 `aarch64-linux-gnu-` |
| `FDT_NAME` | 拷贝到 `output/` 的设备树文件名 |
| `UBOOT_CONFIG` | U-Boot defconfig 名 |
| `LINUX_CONFIG` | 内核 defconfig 名 |
| `PARTITION_CONFIG` | genimage 配置文件路径 |
| `BUILDROOT_DEFCONFIG` | Buildroot 片段 defconfig 文件名 |
| `OPENTINA_OPTEE` | OP-TEE 默认开关 |
| `OPENWRT_CONFIG` / `OPENWRT_TARGET` / `OPENWRT_SUBTARGET` / `OPENWRT_ROOTFS_MB` | OpenWrt 构建参数 |
| `EXTLINUX_ROOT` / `EXTLINUX_CONSOLE` | 写入 `extlinux.conf` 的 `append` 内容 |

### 4.2 各 rootfs 的配置入口

| rootfs | 配置入口 |
|---|---|
| Buildroot | `configs/<板>/br2_opentina.defconfig`，以及 `BR2_GLOBAL_PATCH_DIR` 指向的补丁目录 |
| Debian | `sources/debian/docker/Dockerfile.rootfs` 的 `ARG`，经 `DEBIAN_RELEASE`、`EXTRA_DEBS`、`DESKTOP` 等环境变量传入 |
| Ubuntu | 同 Debian，位于 `sources/ubuntu/` |
| Yocto | `OPENTINA_YOCTO_PROFILE`（`minimal` 或 `qt`）、`OPENTINA_YOCTO_MACHINE`、`OPENTINA_YOCTO_DIR` |
| OpenWrt | 板级 `OPENWRT_CONFIG` 种子配置，以及 `configs/common/openwrt-patches/` 与板级同名目录下的补丁 |

Buildroot 的 defconfig 是片段式的，写入其中的符号若在当前 Buildroot 版本中不
存在或依赖不满足，会被 kconfig 静默丢弃而不报错。修改后应比对解析结果：

```shell
make -C sources/buildroot O=/tmp/chk -j1 defconfig \
    BR2_DEFCONFIG=$PWD/configs/radxa_a7a/br2_opentina.defconfig
grep '^BR2_' configs/radxa_a7a/br2_opentina.defconfig \
    | while read -r l; do grep -qxF "$l" /tmp/chk/.config || echo "dropped: $l"; done
```

## 5. 产物

构建产物位于 `output/<BOARD_NAME>/`：

| 文件 | 说明 |
|---|---|
| `sdcard.img` | genimage 生成的完整 GPT 镜像 |
| `u-boot.fex` | 打包后的 SPL/U-Boot，含 BL31 与 BL32 |
| `boot0.fex` | 一级引导，由 `sources/awbin/bin/a733/<板>/boot0_sdcard.fex` 拷贝而来 |
| `boot.img` | FAT 启动分区映像，含内核、设备树与 extlinux 配置 |
| `rootfs.ext2` | 所选 rootfs 的 ext4 文件系统映像 |
| `Image.gz`、`*.dtb` | 内核与设备树 |
| `tee.bin` | OP-TEE BL32，仅在开启 OP-TEE 时生成 |
| `.done.<组件>` | 组件完成标记 |

`sdcard.img` 的分区布局由 `configs/<板>/partitions.cfg` 定义：

| 分区 | 偏移 | 大小 | 内容 |
|---|---|---|---|
| boot0 | 128K | — | `boot0.fex` |
| uboot | 16400K | — | `u-boot.fex` |
| boot | 20M | 128M | `boot.img`，FAT32 |
| root | 148M | 2048M | `rootfs.ext2` |

当前只提供 SD 卡布局，没有独立的 eMMC 分区配置。

## 6. 烧写

构建脚本不包含烧写命令，需手工写入。

```shell
lsblk                      # 确认目标设备，避免写错盘
sudo umount /dev/sdX*      # 卸载所有已挂载分区
sudo dd if=output/radxa_a7a/sdcard.img of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

`/dev/sdX` 需替换为实际的 SD 卡设备节点，且必须是整盘而非分区。脚本没有设备
识别、防误写和写后校验，这些步骤目前由操作者负责。

写入后建议回读校验：

```shell
sudo dd if=/dev/sdX bs=4M count=$(( $(stat -c%s output/radxa_a7a/sdcard.img) / 4194304 + 1 )) \
    | head -c $(stat -c%s output/radxa_a7a/sdcard.img) | sha256sum
sha256sum output/radxa_a7a/sdcard.img
```

eMMC 烧写路径尚未纳入脚本，也未经验证。

## 7. 启动与登录

### 7.1 启动链

开启 OP-TEE：

```
BROM -> boot0 -> SPL FIT（BL31 + BL32/OP-TEE + U-Boot）-> TF-A -> OP-TEE -> U-Boot -> Linux
```

关闭 OP-TEE：

```
BROM -> boot0 -> SPL FIT（BL31 + U-Boot）-> TF-A -> U-Boot -> Linux
```

### 7.2 串口

默认 `console=ttyS0,115200`，由板级 `EXTLINUX_CONSOLE` 控制。使用 U-Boot 的
distroboot/extlinux 启动时，内核 cmdline 来自 `extlinux.conf` 的 `append`，
应与 U-Boot defconfig 中可能存在的 `CONFIG_BOOTARGS` 保持一致，避免 root 或
console 冲突。

### 7.3 默认账号

Yocto 镜像的默认口令由 `sources/meta-opentina/conf/include/opentina-default-users.inc`
定义，`OPENTINA_ROOT_PASSWORD` 与 `OPENTINA_USER_PASSWORD` 默认分别为 `root` 和
`opentina`，可在 `local.conf` 中覆盖。其它 rootfs 的账号策略由各自
的构建配置决定，参见对应文档。

## 8. 测试与验收

### 8.1 构建侧

每个组件成功后在 `output/<BOARD>/` 写入 `.done.<组件>`，完整构建以
`build component image done` 结束。判定构建成功的最小条件是 `sdcard.img` 存在
且分区表可解析：

```shell
test -f output/radxa_a7a/sdcard.img
fdisk -l output/radxa_a7a/sdcard.img
```

检查 rootfs 内容不必挂载，也不需要 root，可用 `debugfs` 直接读取。该工具来自
`e2fsprogs`；构建过 Buildroot 后也可直接使用
`sources/buildroot/output/host/sbin/debugfs`。

```shell
debugfs -R "ls /usr/bin" output/radxa_a7a/rootfs.ext2
debugfs -R "stat /usr/sbin/sshd" output/radxa_a7a/rootfs.ext2
```

### 8.2 板级基础用例

各 rootfs 共用同一套基础用例：串口输出、根文件系统挂载、init 启动完成、USB
枚举、网络接口与包管理。具体命令与判定标准见各 rootfs 的文档。

### 8.3 图形与 GPU

A733 的 GPU 为 Imagination BXM-4-64（BVNC 36.56.104.183），由内核中
mainline 的 `drivers/gpu/drm/imagination` 驱动（`CONFIG_DRM_POWERVR=y`）。
用户态需要 Mesa 25.3 及以上版本，早于该版本的 Mesa 不认识此 BVNC，表现为所有
DRM ioctl 成功但 Vulkan 初始化返回 `VK_ERROR_INITIALIZATION_FAILED`。

该驱动尚未通过 Vulkan 一致性认证，运行前需设置：

```shell
export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
```

验证顺序：

```shell
vulkaninfo --summary          # 期望 deviceName 含 BXM-4-64，apiVersion 为 1.2.x
weston --backend=drm-backend.so --drm-device=card0 &
vkcube --c 300                # 期望屏幕显示旋转立方体
glmark2-es2-wayland -b build:duration=5
LIBGL_ALWAYS_SOFTWARE=1 glmark2-es2-wayland -b build:duration=5   # llvmpipe 基线
```

最后两条的分数对比用于判定是否真正走了硬件加速。

## 9. 已知限制

- manifest 只覆盖 A733 板级配置，没有按芯片选择 manifest 或 revision 集合的机制。
- 除 `buildroot` 外，manifest 中的项目使用分支而非 commit，构建不可严格复现。
- `.done` 标记不关联输入摘要，源码或配置变化不会触发重建，切换 rootfs 或工具链后需手工 clean。
- 没有统一的依赖预检命令，缺失的主机包只在执行到具体步骤时才暴露。
- 没有统一的烧写命令，也没有设备识别、防误写与写后校验。
- 只有 SD 卡分区布局，eMMC 未配置也未验证。
- Buildroot 的片段 defconfig 中无效符号会被静默丢弃，需按 4.2 节的方法自行比对。

## 10. 故障排查

**`User namespaces are not usable`**
Ubuntu 24.04 及以上的 AppArmor 限制，按 1.3 节放开 sysctl。

**`arm-linux-gnueabihf-gcc: command not found`（构建 optee 组件时）**
OP-TEE OS 在 64 位核心下默认同时构建 32 位和 64 位 TA devkit。当前构建脚本已
通过 `CFG_USER_TA_TARGETS=ta_arm64` 限定为 64 位。若在旧版本上遇到，可安装
`gcc-arm-linux-gnueabihf` 或补上该参数。

**`ModuleNotFoundError: No module named 'elftools'` 或 `'cryptography'`**
缺少 `python3-pyelftools` 或 `python3-cryptography`。注意 OP-TEE 的签名脚本使用
`/usr/bin/python3`，conda 等环境中的同名包不生效。

**Buildroot 中某个软件包未出现在 rootfs 里，且构建没有报错**
该符号很可能被 kconfig 丢弃。常见原因是依赖未满足，例如 `wget`、`lsof`、
`i2c-tools` 都依赖 `BR2_PACKAGE_BUSYBOX_SHOW_OTHERS`。按 4.2 节比对解析结果。

**`No hash found for <包>.tar.gz`**
`BR2_DOWNLOAD_FORCE_CHECK_HASHES=y` 会清空 Buildroot 的 `BR_NO_CHECK_HASH_FOR`
豁免表，因此自定义版本的 tarball 必须自带 hash。把 hash 文件放入
`BR2_GLOBAL_PATCH_DIR` 指向目录下的 `<包名>/<包名>.hash`。

**切换 `--no-optee` 与 `--optee` 后行为不一致**
TF-A 的增量构建不会因 `SPD=` 变化重新编译平台目标文件，Buildroot 也不会卸载已
禁用的软件包。当前脚本已对这两处做强制清理；若使用旧版本，需手工 clean 对应组件。

**Yocto 构建被 OOM 终止**
BitBake 的 `BB_NUMBER_THREADS` 与 `PARALLEL_MAKE` 默认都取 CPU 核数，mesa 引入
的 llvm/clang 编译会因此耗尽内存。在 `local.conf` 中同时下调两者，并设置
`BB_PRESSURE_MAX_MEMORY` 让 BitBake 在内存压力下退让。
