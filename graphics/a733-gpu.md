# A733 GPU 与显示栈

本文档是跨平台专题，描述 Allwinner A733 的 GPU 硬件、内核驱动、Mesa 版本要求，
以及各 rootfs 上启用硬件加速的方法与验证步骤。

文档基线：`opentina-org/build` 仓库 `tina-dev` 分支 commit `ed590a8`
（2026-09-18）。

## 1. 硬件与驱动

A733（sun60i）集成的是 **Imagination BXM-4-64**，BVNC `36.56.104.183`，属于
PowerVR Rogue 系列，不是 Mali。

内核侧走 **mainline** 的 `drivers/gpu/drm/imagination`，板级 defconfig 中
`CONFIG_DRM_POWERVR=y`。该驱动向用户态注册的 DRM 驱动名为 `powervr`。

固件为 `rogue_36.56.104.183_v1.fw`，取自 freedesktop 的
`imagination/linux-firmware` 仓库，安装到 `/lib/firmware/powervr/`，供内核的
`request_firmware()` 加载。`build` 仓库的 `_fetch_powervr_firmware` 与
meta-opentina 的 `powervr-firmware-a733.bb` 使用同一来源，后者带 sha256 校验。

## 2. 两条路线的区别

历史上存在两条互不兼容的路线，混用会导致难以定位的现象。

| | mainline 路线（当前） | vendor 路线（早期） |
|---|---|---|
| 内核驱动 | `drivers/gpu/drm/imagination`，DRM 名 `powervr` | `pvrsrvkm` |
| 用户态 | 上游 Mesa 的 imagination Vulkan 驱动 + zink | vendor DDK（`libpvr*`、`pvr_dri.so`） |
| 判定方式 | Mesa 按 DRM 驱动名匹配 | 按 vendor 自有接口 |

上游 Mesa 的 `pvr_drm_device_is_compatible()` 只校验 DRM 驱动名是否为
`powervr`（`PVR_SUPPORT_SERVICES_DRIVER` 编译开关另外允许 `pvrsrvkm`，但发行版
构建通常不开）。因此**上游 Mesa 不会在 `pvrsrvkm` 上工作**，反之 vendor DDK 也
不适配 mainline 驱动。

这一点直接影响对历史测试结论的解读：早期记录的「Weston GL 实测为 llvmpipe、
用户态硬件加速未通过」来自 vendor DDK 路线的实验，对当前的 mainline 路线不适用，
不应据此判断 mainline 路线也不可行。

## 3. Mesa 版本要求

Mesa 的 `pvr_device_info_init()` 从 **25.3.0** 起才收录 BVNC `36.56.104.183`。
25.0、25.1、25.2 上的表现是：所有 DRM ioctl 都成功，但 Vulkan 初始化返回
`VK_ERROR_INITIALIZATION_FAILED`。这个现象容易被误判为内核或固件问题，实际
原因在用户态版本。

从 **26.1** 起，上游删除了硬编码的 `pvr_drm_configs[]` 兼容字符串表，改为只校验
DRM 驱动名。此前 Mesa 需要在该表中逐个登记 SoC 的 compatible 字符串，A733 未被
收录，因此 26.0 及更早版本需要本地补丁
`0001-pvr-add-Allwinner-A733-to-the-DRM-device-table.patch`（向表中加入
`DEF_CONFIG("allwinner,a733-gpu")`）。

**该补丁在 26.1 及以后不再需要**。layer 升级过 26.0.x 之后，应连同
`mesa.bbappend` 中引用它的 `SRC_URI` 一起删除。

此外，imagination 驱动自 25.3 起依赖 `mesa_clc` / `pco_clc` 预编译器，Yocto 侧
需要在 `PACKAGECONFIG` 中加入 `libclc`，它会引入 `mesa-tools-native`。

## 4. 各发行版的 Mesa 基线

| 发行版 / 来源 | Mesa 版本 | 是否满足 >= 25.3 |
|---|---|---|
| Debian trixie | 25.0.7 | 否 |
| **Debian trixie-backports** | **26.1.2** | 是 |
| Debian forky / sid | 26.1.6 / 26.2.2 | 是 |
| Ubuntu noble | 25.2.8 | 否 |
| **kisak-mesa PPA（noble）** | **26.2.2** | 是 |
| Yocto wrynose | 26.0.x | 是（需第 3 节的补丁） |
| Yocto scarthgap | 25.0.2 | 否 |

Debian 的 `mesa-vulkan-drivers` 包含 `libvulkan_powervr_mesa.so` 与
`/usr/share/vulkan/icd.d/powervr_mesa_icd.json`，即上游已启用 imagination
Vulkan 驱动，不需要自行改 `debian/rules` 重新打包。

trixie-backports 的优先级为 100（`NotAutomatic`），低于 stable 的 500，因此加入
该源不会自动升级任何包，必须显式 `-t trixie-backports` 才生效。升级 Mesa 时需
同时指定 `libgl1-mesa-dri`、`libegl-mesa0`、`libglx-mesa0`、`libgbm1`、
`mesa-libgallium`、`mesa-vulkan-drivers`；其中 `mesa-libgallium` 与
`libglx-mesa0` 容易遗漏，虽然 apt 通常会因强版本依赖自动带上，但不应依赖这种
隐式行为。

## 5. 运行时要求

imagination 驱动尚未通过 Vulkan 一致性认证，不设置下列变量时会拒绝加载：

```shell
export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
```

OpenGL 走 **zink**（在 Vulkan 之上实现 GL），不是独立的 GL 驱动。

要对照软件渲染时，临时设置即可，不应写进镜像：

```shell
export LIBGL_ALWAYS_SOFTWARE=1
export GALLIUM_DRIVER=llvmpipe
```

## 6. 验证步骤

按顺序执行，每一步失败都会让后续步骤失去意义。

### 6.1 内核与固件

```shell
dmesg | grep -iE 'powervr|pvr'
ls -l /dev/dri/
ls -l /lib/firmware/powervr/
```

期望出现 render node（`renderD128`）与对应的 card 节点，固件文件存在。

### 6.2 Vulkan

```shell
export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
vulkaninfo --summary
```

期望 `deviceName` 含 `BXM-4-64`，`apiVersion` 为 1.2.x。若返回
`VK_ERROR_INITIALIZATION_FAILED` 而 DRM ioctl 均成功，优先核对 Mesa 版本是否
达到 25.3。

### 6.3 合成器

```shell
export XDG_RUNTIME_DIR=/run/opentina-hmi
weston --backend=drm-backend.so --drm-device=card0
```

Debian/Ubuntu 的 desktop profile 已提供 `opentina-weston.service`，正常情况下
由 systemd 启动，此处的手动命令用于排查。

### 6.4 应用

```shell
vkcube --c 300
glmark2-es2-wayland -b build:duration=5
LIBGL_ALWAYS_SOFTWARE=1 glmark2-es2-wayland -b build:duration=5
```

`vkcube` 期望在屏幕上显示旋转立方体。两次 `glmark2` 的 `GL_RENDERER` 与
`glmark2 Score` 对比是判断是否真正走硬件加速的依据：仅看单次跑分无法区分
zink 与 llvmpipe。

Debian/Ubuntu 的 desktop profile 提供了 `opentina-hmi-check`，按上述顺序执行
并输出可直接归档的报告。

## 7. 已知限制

- 驱动未通过 Vulkan 一致性认证，必须显式 opt-in。
- Yocto 的 HMI profile 尚未合并到 `meta-opentina` 的 `main`。
- Debian 的 backports Mesa 改动与 Ubuntu 的 PPA 改动均未合并。
- 上述改动都尚未在真机验证；同事此前报告 Mesa 26.1 在硬件上无需补丁即可工作，
  但本项目自身尚无可复现的真机记录。
- 未定义性能基线与回归阈值，`glmark2` 的对比目前只能给出相对结论。
