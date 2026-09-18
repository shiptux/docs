# OpenWrt rootfs

本文档描述 OpenTina 的 OpenWrt 根文件系统：定位、补丁与配置、构建、产物与限制。

文档基线：`opentina-org/build` 仓库 `tina-dev` 分支 commit `ed590a8`
（2026-09-18）；OpenWrt 上游 pin 在 `openwrt-25.12` 分支。

## 1. 定位

OpenWrt 在本项目中**只作为 rootfs 供应商**。启动链（boot0、TF-A、OP-TEE、
U-Boot、内核、设备树）全部复用 `build` 仓库的组件；OpenWrt 自产的内核、kmod 与
per-device 镜像全部丢弃，只消费 target 级的
`bin/targets/<target>/<subtarget>/openwrt-*-rootfs.tar.gz`。

丢弃 kmod 是必须的：OpenWrt 的 kmod 按它自己的内核 ABI（如 6.12.x）编译，与
`sources/linux` 的内核不匹配，无法加载。因此 OpenWrt 用户态需要的内核特性只能
以 builtin 形式补进内核配置，见第 4 节。

需求 O1（在 OpenWrt 仓库内集成 A733 的 kernel、U-Boot、TF-A 构建）**尚未满足**：
OpenWrt 侧只有 rootfs-only 的设备定义，BSP 仍由外层 `build` 仓库构建。

## 2. 源码

manifest 中 OpenWrt 直接指向上游：

```xml
<project name="openwrt" url="https://github.com/openwrt/openwrt.git"
         path="openwrt" revision="openwrt-25.12" clone-depth="1"/>
```

pin 在 `openwrt-25.12` 稳定分支而非滚动的 `main`，浅克隆。

也可设 `OPENTINA_OPENWRT_DIR` 指向本地已有的 OpenWrt 树（例如已预热
`build_dir/` 的 fork），跳过 manifest 克隆。

首次构建时 `build_openwrt` 会自动执行 `./scripts/feeds update -a` 与
`install -a`。

## 3. 补丁与 overlay

### 3.1 补丁

应用顺序为先 `configs/common/openwrt-patches/*.patch`，再
`configs/<板>/openwrt-patches/*.patch`，每组内按字典序 `git apply`。

当前 common 目录下只有 `100-add-sunxi-armv8-subtarget.patch`，它为 sunxi target
增加 `armv8` subtarget 与两个 A733 设备定义。设备仍为 rootfs-only 占位
（`IMAGES :=`），不产出 per-device 镜像。

为保证可重复执行，应用前会把补丁涉及的文件 `git checkout` 回 HEAD。另外
`scripts/config/` 会被 OpenWrt 的每次 `make` 改写，因此也会被重置。

若某个补丁的新增行已全部存在于树中（例如该补丁已合入所使用的 fork），会跳过并
打印提示，而不是报错。

### 3.2 文件 overlay

`configs/common/openwrt-files/` 下的文件由 `install-openwrt-overlay.sh` 拷贝进
解包后的 rootfs：

| 文件 | 用途 |
|---|---|
| `usr/sbin/fw_printenv` | U-Boot 环境变量读写，同时建立 `fw_setenv` 符号链接 |
| `lib/preinit/79_move_config` | preinit 阶段的配置迁移 |
| `etc/board.d/02_network_opentina` | 板级网络接口配置 |

## 4. 内核配置

选择 `OPENTINA_ROOTFS=openwrt` 构建 `linux` 组件时，会在板级 `LINUX_CONFIG`
之上合并 `configs/common/linux-openwrt.fragment`。可用 `LINUX_OPENWRT_FRAGMENT`
覆盖路径。

该 fragment 内的所有选项都必须是 `=y`，原因见第 1 节。当前覆盖：

- procd / ubus / netifd 需要的基础项：`CONFIG_TMPFS`、`CONFIG_UNIX`、
  `CONFIG_PACKET`、`CONFIG_INET`、`CONFIG_IPV6` 等
- A733 GMAC：`CONFIG_STMMAC_ETH`、`CONFIG_DWMAC_SUN55I`、`CONFIG_REALTEK_PHY`
- USB 网卡作为无 PHY 或无网线时的兜底
- firewall4 需要的 nftables 相关项

OpenWrt 用户态缺少某项能力时，只能往这个 fragment 里加 builtin，加 kmod 无效。

## 5. 构建

```shell
cd build
./build.sh <BOARD_NAME> openwrt build          # 完整镜像
./build.sh <BOARD_NAME> openwrt build openwrt  # 仅 rootfs
```

板级配置变量：

| 变量 | 默认 | 说明 |
|---|---|---|
| `OPENWRT_CONFIG` | 板级 `openwrt.config` | `.config` 种子，拷入后经 `make defconfig` 归一化 |
| `OPENWRT_TARGET` | `sunxi` | |
| `OPENWRT_SUBTARGET` | `armv8` | |
| `OPENWRT_ROOTFS_MB` | `2048` | `mkfs.ext4` 映像大小，需与 `partitions.cfg` 的 root 分区一致 |
| `OPENWRT_JOBS` | 继承 `JOBS` | OpenWrt 顶层 make 的并行数 |

首次构建会 bootstrap OpenWrt 自带的工具链，耗时较长。

## 6. 产物

`bin/targets/<target>/<subtarget>/openwrt-*-rootfs.tar.gz` 解包后经
`mkfs.ext4 -d` 生成 `output/<BOARD>/rootfs.ext2`。

解包与 `mkfs.ext4 -d` 必须以 root 身份或 fakeroot 执行：tarball 中记录的是
root 属主，以普通用户解包会把构建用户的 uid 写进每个 inode。构建脚本按
root、fakeroot、Docker 的顺序选择可用方式，两个步骤共享同一个 fakeroot 会话，
使伪造的属主信息能延续到 mkfs 阶段。

解包时会删除 OpenWrt 自带的 `lib/modules/`，再拷入 `linux` 组件 staged 的
`*.ko`；随后叠加 PowerVR 固件、OP-TEE TA（OP-TEE 开启时）与第 3.2 节的
OpenTina overlay。

## 7. 验证状态

OpenWrt rootfs 已在 Radxa A7A 真机完成启动验证。

## 8. 已知限制

- O1 未满足：OpenWrt 仓库内没有 A733 的 kernel、U-Boot、TF-A 集成，BSP 由外层
  `build` 仓库构建。
- 设备定义为 rootfs-only 占位（`IMAGES :=`），不产出 per-device 镜像。
- OpenWrt 自产的 kmod 全部丢弃，缺失的内核特性只能以 builtin 形式补进
  `linux-openwrt.fragment`。
- sun55i 的 VLAN filter 寄存器在 VID 0 上可能超时；netifd 仍使用 eth0。
- 需求 O2 的功能测试尚未脚本化，没有统一的测试报告格式。

## 9. 故障排查

**`OpenWrt feeds layout missing`**
OpenWrt 树不完整，缺少 `feeds.conf` 或 `feeds.conf.default`。检查
`OPENTINA_OPENWRT_DIR` 是否指向了正确的树，或重新执行 `./build.sh init`。

**`OpenWrt: failed to apply <补丁>`**
补丁与当前 OpenWrt 版本不匹配。注意 manifest pin 的是 `openwrt-25.12`；若指向了
其它分支或 fork，需同步调整 `configs/common/openwrt-patches/` 下的补丁。

**`make -j` 失败在 `.ver_check` 或 `stamp` 目录**

```
mv .../tmp/.ver_check .../staging_dir/toolchain-<...>/stamp/.ver_check
mv: ... No such file or directory
make[2]: *** No rule to make target '.../stamp/.ver_check', needed by '.../.prepared'
```

OpenWrt 顶层 `make -j` 的竞态：`mv` 跑在创建 `stamp/` 目录之前。这是偶发的，
重跑通常即可通过。若频繁出现，可通过 `OPENWRT_JOBS` 降低并行数规避，代价是
构建时间变长。

**rootfs 中的文件属主是构建用户而非 root**
解包或 `mkfs.ext4 -d` 没有以 root 或 fakeroot 执行。检查主机是否安装了
`fakeroot`，或是否具备可用的 Docker 回退路径。

**板上缺少某个内核特性，装 kmod 无效**
OpenWrt 的 kmod 在构建时已被丢弃。把对应选项以 `=y` 加入
`configs/common/linux-openwrt.fragment`，然后重新构建 `linux` 与 `openwrt`
两个组件。
