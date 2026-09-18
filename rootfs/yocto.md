# Yocto rootfs

本文档描述 OpenTina 的 Yocto 根文件系统：layer 结构、profile、构建、产物与限制。

文档基线：`opentina-org/meta-opentina` 仓库 `main` 分支；`opentina-org/build`
仓库 `tina-dev` 分支 commit `ed590a8`（2026-09-18）。

## 1. 定位

`meta-opentina` 当前是 **rootfs-only** layer。`local.conf.sample` 中设置

```
PREFERRED_PROVIDER_virtual/kernel = "linux-dummy"
```

即 Yocto 不构建内核。启动链（boot0、TF-A、OP-TEE、U-Boot、内核、设备树）全部由
`build` 仓库的对应组件产出，Yocto 只产出根文件系统。内核模块由 `build` 仓库在
rootfs 生成后叠加。

`conf/machine/a733-aiot.conf` 中虽然预留了

```
UBOOT_MACHINE ?= "a733-aiot_defconfig"
KERNEL_DEVICETREE ?= "allwinner/sun60i-a733-aiot.dtb"
```

但当前没有对应的 U-Boot、kernel、TF-A recipe，这两个变量不被实际消费。需求 Y1
（编写可被镜像与其它 layer 引用的 BSP recipe）因此**尚未满足**。

## 2. Layer 与源码

`yocto-init.sh` 在 `OPENTINA_YOCTO_DIR`（默认 `sources/yocto`）下拉取：

| Layer | 默认来源 | 说明 |
|---|---|---|
| poky | `https://git.yoctoproject.org/poky` | 合并版，含 bitbake 与 openembedded-core |
| meta-openembedded | `https://github.com/openembedded/meta-openembedded.git` | meta-oe 等 |
| meta-qt5 | 见 `scripts/fetch-qt-layers.sh` | 仅 `--qt` 时拉取 |

默认分支由 `YOCTO_BRANCH` 控制，当前为 `scarthgap`。各 layer 可用
`POKY_REPO` / `POKY_BRANCH` / `OE_REPO` / `OE_BRANCH` / `QT_BRANCH` 单独覆盖，
也可通过 `yocto-sources.conf`（模板见 `yocto-sources.conf.example`）配置。

`--force` 会重新克隆已存在的 layer 目录。

这些 layer 体积较大，首次拉取耗时长。

## 3. Profile

| profile | DISTRO | image | 构建目录 |
|---|---|---|---|
| `minimal` | `opentina-minimal` | `opentina-image-minimal` | `build-opentina` |
| `qt` | `opentina-qt` | `opentina-image-qt` | `build-opentina-qt` |

`minimal` 面向串口控制台，含 `packagegroup-core-boot`、OpenSSH 与基础网络工具。
`qt` 在此基础上加入 qtbase、qtdeclarative、qtsvg、qtwayland、qtgraphicaleffects、
qtmultimedia 与 mesa，并在 `wayland` DISTRO_FEATURE 开启时加入 weston。
`qt` 需要 meta-qt5，因此必须以 `yocto-init.sh --qt` 初始化。

`opentina-build.sh` 另有一个 `env` 模式，只建立构建环境而不执行 bitbake，便于
手动运行 bitbake 命令。

## 4. 构建

### 4.1 通过 build 仓库

```shell
cd build
./build.sh <BOARD_NAME> yocto build yocto   # 仅 rootfs，耗时长
./build.sh <BOARD_NAME> yocto build         # 完整镜像
```

`build_yocto` 会拒绝以 root 运行。若 `meta-openembedded` 尚未就绪，它会先自动
执行 `yocto-init.sh`（`qt` profile 时带 `--qt`）。

相关变量：

| 变量 | 默认 | 说明 |
|---|---|---|
| `OPENTINA_YOCTO_PROFILE` | `minimal` | 只接受 `minimal` 或 `qt`，其它值报错退出 |
| `YOCTO_MACHINE` / `OPENTINA_YOCTO_MACHINE` | `a733-aiot` | layer 的 machine 名，与 `BOARD_NAME` 无关 |
| `OPENTINA_YOCTO_DIR` | `sources/yocto` | Yocto 工作区 |

### 4.2 直接调用

```shell
cd build/sources/meta-opentina
./yocto-init.sh                 # 或 ./yocto-init.sh --qt
./opentina-build.sh minimal     # 或 qt / env
```

### 4.3 并行度与内存

`conf/templates/local.conf.sample` 当前为：

```
BB_NUMBER_THREADS ?= "${@oe.utils.cpu_count()}"
PARALLEL_MAKE ?= "-j ${@oe.utils.cpu_count()}"
```

两者都取 CPU 核数，没有上限。在核多内存少的主机上，mesa 带入的 llvm/clang 编译
会同时开出 `cpu_count` 个任务、每个再 `-j cpu_count`，很容易被 OOM killer 终止。

构建前应在 `local.conf` 中下调，例如：

```
BB_NUMBER_THREADS ?= "${@min(oe.utils.cpu_count(), 6)}"
PARALLEL_MAKE ?= "-j ${@min(oe.utils.cpu_count(), 8)}"
BB_PRESSURE_MAX_MEMORY ?= "50000"
```

`BB_PRESSURE_MAX_MEMORY` 让 bitbake 在内存压力超阈值时暂缓派发新任务。

在无 swap 的主机上，另建议为整个构建加一层 cgroup 内存上限：

```shell
systemd-run --user --scope -p MemoryMax=20G -p MemorySwapMax=0 -p CPUQuota=800% \
    -- ./build.sh <BOARD> yocto build yocto
```

这样触发上限时只会终止构建，而不会拖垮整机。

## 5. 产物

bitbake 的产物位于

```
sources/yocto/<构建目录>/tmp/deploy/images/<MACHINE>/opentina-image-*-<MACHINE>.ext4
```

`build_yocto` 将其拷贝为 `output/<BOARD>/rootfs.ext2`，随后叠加 `linux` 组件
staged 的 `*.ko`。因此需要内核模块时必须先构建 `linux` 组件。

## 6. 启动与登录

默认账号由 `conf/include/opentina-default-users.inc` 定义：

```
OPENTINA_ROOT_PASSWORD ??= "root"
OPENTINA_USER_PASSWORD ??= "opentina"
```

两者都是可覆盖的默认值，可在 `local.conf` 中改写。账号在 rootfs 后处理阶段写入
shadow。镜像的 `IMAGE_FEATURES` 含 `allow-root-login`，SSH 允许 root 登录。

内核 cmdline 与串口参数由 `build` 仓库的 `bootfs` 组件生成，见构建系统文档。
`build_bootfs` 会为 Yocto 追加 `systemd.gpt_auto=0`。

## 7. 测试与验收

构建侧：`opentina-build.sh` 正常结束且 deploy 目录下存在对应的 `.ext4` 即为成功。
检查 rootfs 内容可用 `debugfs` 直接读取，无需挂载：

```shell
debugfs -R "ls /usr/bin" output/<BOARD>/rootfs.ext2
```

板级基础用例与其它 rootfs 共用：串口输出、根文件系统挂载、init 启动完成、USB
枚举、网络接口与包管理。

## 8. 已知限制

- Y1 未满足：没有 U-Boot、kernel、TF-A 的 Yocto recipe，layer 为 rootfs-only。
- Y2 部分完成：`qt` profile 引用的是 OE 上游的 Mesa 与 Qt recipe，只能软件渲染；
  layer 内没有 A733 GPU 相关的 recipe 或 bbappend。
- **HMI profile 尚未合并**。`opentina-image-hmi.bb`、`conf/distro/opentina-hmi.conf`、
  mesa bbappend 与 PowerVR 固件 recipe 都不在 `main` 上，只存在于未合并的分支与
  草稿 PR。`main` 上可用的只有 `minimal` 与 `qt` 两个 profile。
- **迁移到 wrynose 之后的 layer 拆分尚未合并**。`main` 仍使用合并版 poky 与
  `scarthgap` 分支。基于 openembedded-core / bitbake / meta-yocto 分别拉取的改动
  同样在未合并的分支上。
- `local.conf.sample` 的并行度没有上限，见 4.3 节。
- MACHINE 固定为 `a733-aiot`，没有按板型区分的 machine 配置。

## 9. 故障排查

**`Yocto/bitbake must not run as root`**
`build_yocto` 主动拒绝 root。改用普通用户执行。

**`meta-openembedded/meta-oe` 不存在**
只拉取了 poky 而没有 meta-openembedded。在 `sources/meta-opentina` 执行
`./yocto-init.sh` 后重试，或直接用 `./build.sh <BOARD> yocto build yocto`，
它会自动补跑初始化。

**`User namespaces are not usable`**
Ubuntu 24.04 及以上的 AppArmor 限制。见构建系统文档的对应章节。

**BitBake 报缺少 `HOSTTOOLS`**
缺少 `chrpath`、`diffstat`、`texinfo`（`makeinfo`）或 `rpcsvc-proto`（`rpcgen`）。
按构建系统文档的主机软件包清单安装。

**构建被 OOM killer 终止**
见 4.3 节，同时下调 `BB_NUMBER_THREADS` 与 `PARALLEL_MAKE`，并设置
`BB_PRESSURE_MAX_MEMORY`。只调其中一个不够：两者相乘才是实际并发编译数。

**在 Docker 容器内构建 yocto 组件失败**
OpenTina 的构建镜像未预装完整的 Yocto 宿主依赖。`yocto` 组件应在宿主机构建。

**qt profile 报找不到 meta-qt5**
初始化时未带 `--qt`。执行 `./yocto-init.sh --qt` 后重试。
