# Debian rootfs

本文档描述 OpenTina 的 Debian 根文件系统：构建路径、profile、配置项、产物、
部署与验收。

文档基线：`opentina-org/debian` 仓库 `main` 分支；`opentina-org/build` 仓库
`tina-dev` 分支 commit `ed590a8`（2026-09-18）。

## 1. 定位

Debian 只作为 rootfs 供应商。启动链（boot0、TF-A、OP-TEE、U-Boot、内核、设备树）
全部由 `build` 仓库的对应组件产出，Debian 侧只生成 `rootfs.ext2`。内核模块由
`build` 仓库在 rootfs 生成后叠加，不由 Debian 的包管理提供。

## 2. 构建路径

仓库提供三条路径，`build_debian` 按可用性依次选择：

| 优先级 | 路径 | 入口 | 前提 |
|---|---|---|---|
| 1 | BuildKit + QEMU | `docker/build-rootfs-buildx.sh` | Docker 且 `docker buildx` 可用 |
| 2 | Docker + debootstrap | `docker/build-rootfs.sh` | Docker 可用 |
| 3 | 宿主机 debootstrap | `mk-lite-rootfs.sh` | 宿主机本身是 arm64 |

可用 `OPENTINA_DEBIAN_USE_BUILDX=0` 或 `OPENTINA_DEBIAN_USE_DOCKER=0` 关闭前两条。
在容器内构建时（`OPENTINA_IN_DOCKER` 已设置）前两条自动跳过。

desktop profile 与 OEM 注入只在路径 1 实现；在路径 2 或 3 上请求它们会直接报错
退出，而不是静默产出缺少内容的 rootfs。

需求 D2 要求使用 `live-build`。当前三条路径均非 `live-build`，仓库中没有
`lb config`、package list、hook 或 include 配置，该需求尚未满足。

## 3. 构建命令

### 3.1 通过 build 仓库

```shell
cd build
./build.sh <BOARD_NAME> debian build          # 完整镜像
./build.sh <BOARD_NAME> debian build debian   # 仅 rootfs
```

### 3.2 直接调用

```shell
cd build/sources/debian
MAKE_EXT4=1 ARCH=arm64 ./docker/build-rootfs-buildx.sh trixie
```

## 4. 配置项

### 4.1 build 仓库侧

| 变量 | 默认 | 说明 |
|---|---|---|
| `DEBIAN_RELEASE` | `trixie` | 亦可用 `bookworm` |
| `DEBIAN_ARCH` | `arm64` | |
| `OPENTINA_DEBIAN_PROFILE` | `lite` | `lite` 或 `desktop` |
| `OPENTINA_DEBIAN_SERIAL_FIX` | `0` | 串口相关的临时修正开关 |
| `OPENTINA_DEBIAN_USE_BUILDX` | `1` | 置 0 跳过 buildx 路径 |
| `OPENTINA_DEBIAN_USE_DOCKER` | `1` | 置 0 跳过全部 Docker 路径 |
| `OPENTINA_OEM_DIR` | 无 | OEM 注入目录，见第 6 节 |

`OPENTINA_DEBIAN_PROFILE` 只接受 `lite` 和 `desktop`，其它取值报错退出。
profile 会写入 `output/<BOARD>/.debian-profile`。

### 4.2 Debian 仓库侧

| 变量 | 默认 | 说明 |
|---|---|---|
| `ARCH` | `arm64` | 亦支持 `armhf` |
| `DEBIAN_MIRROR` / `DEBIAN_SECURITY_MIRROR` | Debian 官方 | 镜像源 |
| `ROOT_PASSWORD` | `root` | |
| `EXTRA_DEBS` | 空 | 追加的 apt 包，空格分隔 |
| `OPENTINA_SERIAL_TTY` | `ttyS0` | 启用 getty 的串口 |
| `MAKE_EXT4` | 未设 | 设为 1 时额外产出 `.ext4` |
| `ROOTFS_EXT4_MB` | rootfs 实际占用 + 400 | ext4 映像大小 |
| `DESKTOP` | `0` | 置 1 加入桌面/HMI 软件栈 |
| `BASE_FLAVOR` | `-slim` | 基础镜像后缀，desktop 使用标准镜像 |
| `HOSTNAME` | `debian-lite` | |
| `QEMU_BINFMT_SETUP` | 未设 | 置 0 跳过在宿主机注册 qemu binfmt |

## 5. 镜像内容

### 5.1 lite

```
openssh-server sudo locales ca-certificates curl wget
iproute2 iputils-ping ifupdown isc-dhcp-client
vim-tiny less file pciutils usbutils kmod dbus
systemd-sysv systemd-timesyncd udev
```

init 为 systemd。若基础镜像未提供 `/sbin/init`，构建脚本会把它链接到
`/lib/systemd/systemd`。

### 5.2 desktop

在 lite 基础上追加：

```
weston seatd xwayland
libgl1-mesa-dri mesa-utils
gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good
gstreamer1.0-plugins-bad gstreamer1.0-libav
gstreamer1.0-alsa gstreamer1.0-pulseaudio gstreamer1.0-gl
pulseaudio alsa-utils
chromium fonts-dejavu-core dbus-user-session
```

当前 desktop profile 在 `/etc/environment` 中写入
`LIBGL_ALWAYS_SOFTWARE=1` 与 `GALLIUM_DRIVER=llvmpipe`，即固定使用 llvmpipe
软件渲染。原因是 trixie 自带的 Mesa 为 25.0，而 A733 的 Imagination BXM-4-64
需要 Mesa 25.3 及以上；镜像也未安装 `mesa-vulkan-drivers`，因此不存在可用的
Vulkan ICD。硬件加速路径的状态见第 9 节。

### 5.3 X11

只安装 XWayland，不安装 Xorg server。需要原生 X11 服务的场景当前不被满足。

## 6. OEM 注入

用于在不 fork 仓库的前提下加入客户私有 `.deb`、文件 overlay 和收尾脚本。
仅在 buildx 路径可用。

目录约定：

```
<OEM_DIR>/
  packages/          *.deb，由 apt-get install ./packages/*.deb 安装
  rootfs-overlay/    cp -a 到 /
  post.sh            在 chroot 内执行，可选
```

路径优先级为环境变量 `OPENTINA_OEM_DIR`，其次板级 `configs/<板>/oem/`，都没有
则不启用。

依赖策略上不使用 `apt-get -f install`：`packages/*.deb` 的依赖必须由 base apt
源或同目录下的其它 `.deb` 满足，否则构建直接失败。这样缺失依赖会立即暴露，而
不是被 apt 隐式修复后留下难以追溯的镜像。

回归用例位于 `build` 仓库的 `tests/oem/build-test-debs.sh`，生成依赖可解的 `good/` 和故意写入
不存在依赖的 `broken/` 两套样例。

## 7. 产物与部署

Debian 仓库的产物在 `sources/debian/out/`，命名为
`debian-<release>-<profile>-<arch>-<时间戳>.{tar.gz,ext4}`。

`build_debian` 取其中最新的 `.ext4` 拷贝为 `output/<BOARD>/rootfs.ext2`，随后把
`linux` 组件 staged 的 `*.ko` 叠加进去。因此如果需要内核模块，必须先构建
`linux` 组件。

烧写方式与分区布局见构建系统文档。

## 8. 启动与使用

### 8.1 串口与登录

默认在 `OPENTINA_SERIAL_TTY`（`ttyS0`）上启用 getty，波特率 115200。
root 口令由 `ROOT_PASSWORD` 决定，默认 `root`。

### 8.2 启动参数

`build_bootfs` 会在 extlinux 的 `append` 中追加 `systemd.gpt_auto=0`。这是必需的：
GPT 上的 FAT 启动分区由 U-Boot 使用，不在 rootfs 内挂载，若不关闭 systemd 的
GPT 自动挂载，systemd 会尝试挂载不存在的 `/boot` 并报错。

### 8.3 内核模块

模块由 `build` 仓库叠加到 rootfs，不在 Debian 的 dpkg 数据库中。`apt` 不会管理
它们，升级内核后需要重新构建并重新烧写 root 分区。

## 9. 已知限制

- D2 未满足：当前实现不是 `live-build`。
- desktop profile 固定软件渲染，硬件加速未启用。改用 `trixie-backports` 的
  Mesa 26.1 并安装 `mesa-vulkan-drivers` 的改动目前只存在于本地分支，未合并，
  也未经真机验证。
- 只有 XWayland，没有 Xorg server。
- PulseAudio 与 ALSA 工具已在 desktop profile 中，但真机上 `/dev/snd` 只有
  timer，`aplay -l` 与 `arecord -l` 均无声卡，ALSA machine 未注册。播放与录音
  未通过验收。
- 镜像不含 `dpkg-buildpackage`、`debhelper`、`fakeroot`、`lintian`、`sbuild`、
  `gcc`、`make`，无法在板上构建标准 Debian 源码包。
- 不含 glmark2、perf、stress-ng、sysbench、iostat 等 benchmark 工具，没有性能
  基线与回归阈值。
- 真机测试中曾出现 SD 卡写响应超时与驱动复位，以及串口输出的 NUL 字符污染；
  测试过程尚未脚本化。
- 除 `buildroot` 外 manifest 使用分支而非 commit，Debian 仓库同样如此，构建
  结果不可严格复现。

## 10. 故障排查

**`Debian desktop profile requires the buildx path.`**
desktop profile 与 OEM 注入只在 buildx 路径实现。检查 `docker buildx version`
是否可用，以及是否设置了 `OPENTINA_DEBIAN_USE_BUILDX=0`。

**在 x86_64 宿主机上报需要 Docker**
非 arm64 宿主机必须有 Docker。若确实要用原生 debootstrap，需要宿主机本身是
arm64，并安装 `debootstrap`、`qemu-user-static`、`binfmt-support`，同时设置
`OPENTINA_DEBIAN_USE_DOCKER=0`。

**rootfs 中缺少内核模块**
`build_debian` 只叠加 `linux` 组件 staged 的模块。先构建 `linux` 组件，再重新
构建 debian 组件。

**OEM 的 `.deb` 安装失败并中止构建**
这是预期行为。`packages/*.deb` 的依赖必须由 base apt 源或同目录下其它 `.deb`
满足，仓库不使用 `apt-get -f install` 做隐式修复。补齐依赖后重试。

**systemd 报无法挂载 `/boot`**
检查 extlinux 的 `append` 中是否包含 `systemd.gpt_auto=0`。

**挂载 root 分区后发现文件属主异常**
在非 root 且无 fakeroot 的环境中生成 rootfs 会导致 setuid 二进制属主错误，
挂载时表现为权限相关失败。构建路径应保证以 root 或 fakeroot 生成映像。
