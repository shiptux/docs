# OpenTina 文档

本仓库存放 OpenTina 各平台的正式文档。文档的覆盖范围与验收口径见
`opentina-org/build` 仓库 `requirements/README.md` 中的全局规则。

| 文档 | 范围 |
|---|---|
| [build/README.md](build/README.md) | 构建系统：环境、源码获取、配置、构建、产物、烧写、验收与故障排查 |
| [rootfs/debian.md](rootfs/debian.md) | Debian rootfs：构建路径、profile、OEM 注入、产物、部署与限制 |
| [rootfs/yocto.md](rootfs/yocto.md) | Yocto rootfs：layer 结构、profile、并行度与内存、产物与限制 |
| [rootfs/openwrt.md](rootfs/openwrt.md) | OpenWrt rootfs：rootfs-only 定位、补丁与 overlay、内核 builtin 要求、产物与限制 |
| [rootfs/ubuntu.md](rootfs/ubuntu.md) | Ubuntu rootfs：与 Debian 的实质差异、双 mirror、默认账号、产物与限制 |
| [graphics/a733-gpu.md](graphics/a733-gpu.md) | A733 GPU 与显示栈：两条驱动路线的区别、Mesa 版本门槛、各发行版基线、验证步骤 |

待补充：Buildroot rootfs 的构建与验证文档。
