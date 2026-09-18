# Ubuntu rootfs

本文档描述 OpenTina 的 Ubuntu 根文件系统。整体结构与 Debian 相近，但配置项、
默认包集和仓库内容都有实质差异，不能照搬 Debian 文档。

文档基线：`opentina-org/ubuntu` 仓库 `main` 分支；`opentina-org/build` 仓库
`tina-dev` 分支 commit `ed590a8`（2026-09-18）。

## 1. 定位

与 Debian 相同，Ubuntu 只作为 rootfs 供应商，启动链由 `build` 仓库的组件产出，
内核模块在 rootfs 生成后由 `build` 仓库叠加。

## 2. 构建路径

`build_ubuntu` 按可用性依次选择：

| 优先级 | 路径 | 入口 | 前提 |
|---|---|---|---|
| 1 | BuildKit + QEMU | `docker/build-rootfs-buildx.sh` | Docker 且 `docker buildx` 可用 |
| 2 | Docker + debootstrap | `docker/build-rootfs.sh` | Docker 可用 |
| 3 | 宿主机 debootstrap | `mk-base-ubuntu.sh` 然后 `mk-ubuntu-rootfs.sh` | 宿主机本身是 arm64 |

与 Debian 的差异：原生路径是**两个脚本分两步**（先做 base，再做 rootfs），
Debian 只有单个 `mk-lite-rootfs.sh`。

desktop profile 只在路径 1 实现，在路径 2 或 3 上请求会报错退出。

可用 `OPENTINA_UBUNTU_USE_BUILDX=0` 或 `OPENTINA_UBUNTU_USE_DOCKER=0` 关闭前两条。

## 3. 构建命令

```shell
cd build
./build.sh <BOARD_NAME> ubuntu build          # 完整镜像
./build.sh <BOARD_NAME> ubuntu build ubuntu   # 仅 rootfs
```

## 4. 配置项

### 4.1 build 仓库侧

| 变量 | 默认 | 说明 |
|---|---|---|
| `UBUNTU_RELEASE` | `24.04` | |
| `UBUNTU_ARCH` | `arm64` | |
| `OPENTINA_UBUNTU_PROFILE` | `lite` | `lite` 或 `desktop`，其它值报错退出 |
| `OPENTINA_UBUNTU_USE_BUILDX` | `1` | |
| `OPENTINA_UBUNTU_USE_DOCKER` | `1` | |
| `OPENTINA_MOTD_BANNER_FILE` | 无 | 自定义登录横幅，仅原生路径的 `mk-ubuntu-rootfs.sh` 消费 |

### 4.2 Ubuntu 仓库侧

| 变量 | 默认 | 说明 |
|---|---|---|
| `SUITE` | `noble` | |
| `UBUNTU_MIRROR` | `http://archive.ubuntu.com/ubuntu` | amd64 使用 |
| `UBUNTU_PORTS_MIRROR` | `http://ports.ubuntu.com/ubuntu-ports` | arm 架构使用 |
| `ROOT_PASSWORD` | `root` | |
| `EXTRA_USER` | `cat` | 除 root 外额外创建的账号 |
| `EXTRA_USER_PASSWORD` | `temppwd` | 该账号的默认口令 |
| `EXTRA_DEBS` | 空 | 追加的 apt 包 |
| `SERIAL_TTY` | `ttyS0` | |
| `HOSTNAME` | `ubuntu-lite` | |
| `TIMEZONE` | `Asia/Shanghai` | Debian 侧没有此项 |
| `DESKTOP` | `0` | |

两个 mirror 变量是必要的：`ubuntu:<codename>` 基础镜像对非 amd64 默认使用 ports
源，但 Docker Hub 的多架构 manifest 有时仍给出 `archive.*`，因此构建脚本按架构
改写为 `UBUNTU_MIRROR` 或 `UBUNTU_PORTS_MIRROR`。

与 Debian 的其它差异：Ubuntu 没有 `BASE_FLAVOR`（Debian 用它在 `-slim` 与标准
镜像间切换），也没有 `SERIAL_FIX`。

## 5. 镜像内容

### 5.1 lite

Ubuntu 的 lite 包集明显比 Debian 丰富：

```
rsyslog sudo dialog apt-utils evtest acpid
net-tools openssh-server ifupdown
inetutils-ping libssl-dev tcpdump i2c-tools strace vim iperf3
ethtool toilet htop pciutils usbutils curl whiptail
gnupg bc gdisk parted gpiod libgpiod-dev u-boot-tools bsdmainutils
file fdisk locales ca-certificates
systemd-sysv systemd-timesyncd
```

其中 `i2c-tools`、`gpiod`、`u-boot-tools`、`evtest`、`iperf3`、`strace` 等在
Debian lite 中没有，需要板级调试工具时 Ubuntu 开箱即用的程度更高。

### 5.2 desktop

```
weston seatd xwayland
libgl1-mesa-dri mesa-utils
gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good
gstreamer1.0-plugins-bad gstreamer1.0-libav
gstreamer1.0-alsa gstreamer1.0-pulseaudio gstreamer1.0-gl
pulseaudio alsa-utils
chromium fonts-dejavu-core dbus-user-session
```

Chromium 取自 xtradeb PPA，因为 Ubuntu 自带的 `chromium-browser` 只是指向 snap
的 shim，在无 snapd 的 rootfs 中不可用。

与 Debian 相同，desktop profile 当前在 `/etc/environment` 写入
`LIBGL_ALWAYS_SOFTWARE=1` 与 `GALLIUM_DRIVER=llvmpipe`，固定软件渲染，且未安装
`mesa-vulkan-drivers`。noble 自带的 Mesa 为 25.2.8，低于 A733 所需的 25.3，
硬件加速路径见第 8 节。

### 5.3 X11

只安装 XWayland，不安装 Xorg server。

## 6. 产物与部署

产物在 `sources/ubuntu/out/`，命名为
`ubuntu-<release>-<profile>-<arch>-<时间戳>.ext4`。`build_ubuntu` 取最新的一个
拷贝为 `output/<BOARD>/rootfs.ext2`，lite profile 在找不到时回退到仓库根目录的
`ubuntu-rootfs.ext4`。随后叠加 `linux` 组件 staged 的 `*.ko`。

烧写方式与分区布局见构建系统文档。

## 7. 启动与登录

默认在 `SERIAL_TTY`（`ttyS0`）上启用 getty。账号有两个：root（口令由
`ROOT_PASSWORD` 决定，默认 `root`），以及 `EXTRA_USER` 指定的账号（默认用户名
`cat`，口令 `temppwd`）。

`build_bootfs` 会为 Ubuntu 追加 `systemd.gpt_auto=0`，原因与 Debian 相同：
GPT 上的 FAT 启动分区由 U-Boot 使用，不在 rootfs 内挂载。

## 8. 已知限制

- desktop profile 固定软件渲染。noble 自带 Mesa 25.2.8 低于 A733 所需的 25.3，
  且未安装 `mesa-vulkan-drivers`。硬件加速需要更高的 Mesa 基线，Ubuntu 侧可用
  的来源是 kisak-mesa PPA（26.2.2）；该改造尚未实施。
- 默认创建的额外账号 `cat` 带固定弱口令 `temppwd`。对外交付的镜像应通过
  `EXTRA_USER_PASSWORD` 覆盖，或去掉该账号。
- 只有 XWayland，没有 Xorg server。
- 仓库中存在与本项目无关的遗留文件，均未被当前构建路径引用：
  - `post-build.sh` 含 Rockchip 的 `RK_LEGACY_PARTITIONS` 等变量，仅被
    `mk-image.sh` 引用，而 `build_ubuntu` 不走该脚本。
  - `overlay-firmware/` 下是 Intel iwlwifi 固件，仓库内无任何引用。
  - `sources.list`、`sources.list.jammy`、`sources.list.noble` 无任何引用。
- manifest 中 Ubuntu 使用分支而非 commit，构建结果不可严格复现。

## 9. 故障排查

**`Ubuntu desktop profile requires the buildx path.`**
desktop profile 只在 buildx 路径实现。检查 `docker buildx version` 是否可用，
以及是否设置了 `OPENTINA_UBUNTU_USE_BUILDX=0`。

**apt 报找不到 arm64 的包**
基础镜像的源被写成了 `archive.ubuntu.com`，该站点不提供 arm 架构的包。确认
构建脚本已按架构改写为 `UBUNTU_PORTS_MIRROR`。

**chromium 安装后无法运行**
确认装的是 xtradeb PPA 的 `chromium` 而不是 Ubuntu 自带的 `chromium-browser`，
后者是 snap shim，在无 snapd 的 rootfs 中不可用。

**rootfs 中缺少内核模块**
先构建 `linux` 组件，再重新构建 ubuntu 组件。

**在 x86_64 宿主机上报需要 Docker**
非 arm64 宿主机必须有 Docker。原生路径要求宿主机本身是 arm64，并安装
`binfmt-support`、`qemu-user-static`，同时设置 `OPENTINA_UBUNTU_USE_DOCKER=0`。
