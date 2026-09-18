# Buildroot rootfs

本文档描述 OpenTina 的 Buildroot 根文件系统。Buildroot 是默认 rootfs：
`./build.sh <BOARD> build` 不指定 rootfs 时走的就是它，组件名为 `br2`。

文档基线：`opentina-org/build` 仓库 `tina-dev` 分支 commit `ed590a8`
（2026-09-18）。

## 1. 定位

与其它 rootfs 相同，Buildroot 只产出根文件系统。`BR2_LINUX_KERNEL=n`，内核由
`build` 仓库的 `linux` 组件构建，模块经 post-build hook 装入。

组件名用 `br2` 而不是 `buildroot`，以区别于命令行上的发行版槽位
`OPENTINA_ROOTFS=buildroot`。

## 2. 工具链

使用 Bootlin 的外部工具链，不自行构建：

```
BR2_TOOLCHAIN_EXTERNAL=y
BR2_TOOLCHAIN_EXTERNAL_BOOTLIN=y
BR2_TOOLCHAIN_EXTERNAL_BOOTLIN_AARCH64_GLIBC_STABLE=y
```

该工具链带的内核头文件版本为 **5.4**。这一点在选择软件包时是实际约束：需要更
新内核接口的包会编译失败，且失败点往往在某个源文件的 `#include` 上，而不是在
配置阶段。

## 3. 片段式 defconfig

板级配置位于 `configs/<板>/br2_opentina.defconfig`，由板级 `BUILDROOT_DEFCONFIG`
指定文件名。它是**片段式**的：只列出与 Buildroot 默认值不同的项，其余由
`make defconfig` 补全。

### 3.1 必须掌握的陷阱

片段中的符号如果在当前 Buildroot 版本中不存在，或其依赖未被满足，**会被 kconfig
静默丢弃，构建不会报错**。表现为软件包没有出现在 rootfs 里，而日志一切正常。

已经踩到过的两类：

- 符号不存在。`BR2_PACKAGE_OPTEE_CLIENT_4_6_0` 在 Buildroot 2026.05 中没有定义
  （版本 choice 只有 `_AS_OS` / `_LATEST` / `_CUSTOM_TARBALL`），写了等于没写，
  版本回落到默认值。
- 依赖未满足。`wget`、`lsof`、`i2c-tools` 都 `depends on
  BR2_PACKAGE_BUSYBOX_SHOW_OTHERS`，该开关默认关闭，因此这三个包会被丢弃。

### 3.2 比对方法

任何修改 defconfig 的改动都应比对解析结果：

```shell
cd build
make -C sources/buildroot O=/tmp/chk -j1 defconfig \
    BR2_DEFCONFIG=$PWD/configs/radxa_a7a/br2_opentina.defconfig
grep '^BR2_' configs/radxa_a7a/br2_opentina.defconfig \
    | while read -r l; do grep -qxF "$l" /tmp/chk/.config || echo "dropped: $l"; done
```

输出为空表示片段中的每一行都真正生效。注意写成 `BR2_FOO=n` 的行会被 kconfig
写成 `# BR2_FOO is not set`，用上面的精确匹配会误报，比对时需排除。

## 4. 下载校验与自定义版本

defconfig 中设有：

```
BR2_DOWNLOAD_FORCE_CHECK_HASHES=y
```

该选项会**清空** Buildroot 的 `BR_NO_CHECK_HASH_FOR` 豁免表
（见 `package/pkg-download.mk`），因此任何使用自定义版本或自定义 tarball 的包
都必须自带 hash，否则报：

```
ERROR: No hash found for <文件名>
```

Buildroot 为此提供的机制是把 hash 文件放进 `BR2_GLOBAL_PATCH_DIR` 指向的目录：

```
<BR2_GLOBAL_PATCH_DIR>/<包名>/<包名>.hash
<BR2_GLOBAL_PATCH_DIR>/<包名>/<版本>/<包名>.hash   # 仅对该版本生效
```

同一目录也用于存放补丁，命名规则相同。版本子目录存在时优先于包目录。

## 5. post-build hook

`BR2_ROOTFS_POST_BUILD_SCRIPT` 指向 `configs/common/br2-post-build.sh`，它按顺序
调用：

| 脚本 | 作用 |
|---|---|
| `install-linux-modules.sh` | 装入 `linux` 组件 staged 的 `*.ko` |
| `install-powervr-firmware.sh` | 装入 PowerVR 固件到 `/lib/firmware/powervr/` |
| `install-optee-ta.sh` | 装入 OP-TEE TA 到 `/lib/optee_armtz/`，仅 `OPENTINA_OPTEE` 非 0 时执行 |

路径写成相对 Buildroot 顶层目录的形式（`../../configs/common/...`）。

## 6. OP-TEE 用户态

defconfig 启用 `BR2_PACKAGE_OPTEE_CLIENT` / `OPTEE_TEST` / `OPTEE_EXAMPLES`，
并通过 `BR2_ROOTFS_USERS_TABLES` 引入 `br2-users-optee.txt` 创建 `tee` 组，
配合 optee-client 的 udev 规则给 `/dev/tee*` 归组。安全存储目录为 `/data/tee`。

**当前实现存在两个已知缺陷**，PR #12 已提出修复但尚未合并：

- `BR2_PACKAGE_OPTEE_TEST` 与 `_EXAMPLES` 都 `depends on BR2_TARGET_OPTEE_OS`
  （它们的 TA 需要 TA devkit），而 defconfig 刻意没有开启该项，因此两个符号被
  丢弃，`xtest` 与 `optee_example_*` **从未被构建**。
- `BR2_PACKAGE_OPTEE_{CLIENT,TEST,EXAMPLES}_4_6_0` 这三个符号不存在，版本选择
  静默回落到 `_LATEST`（4.9.0），与 `optee` 组件的 4.6.0 安全核不一致。

可用第 3.2 节的方法复现：修复前的解析结果中没有任何 optee-test / optee-examples
条目，且 `BR2_PACKAGE_OPTEE_CLIENT_VERSION="4.9.0"`。

## 7. 构建与产物

```shell
cd build
./build.sh <BOARD_NAME> build          # 完整镜像，默认即 Buildroot
./build.sh <BOARD_NAME> build br2      # 仅 rootfs
```

产物为 `sources/buildroot/output/images/rootfs.ext2`，拷贝为
`output/<BOARD>/rootfs.ext2`。映像大小由 `BR2_TARGET_ROOTFS_EXT2_SIZE` 决定，
当前为 1024M，需与 `partitions.cfg` 中 root 分区的大小相容。

`demo_aiot_a733_v3` 与 `radxa_a7a` 的 defconfig 当前只差一行注释，两块板的
Buildroot rootfs 内容一致。

## 8. 验收

构建侧检查 rootfs 内容不需要挂载，也不需要 root：

```shell
debugfs -R "ls /usr/bin" output/<BOARD>/rootfs.ext2
debugfs -R "stat /usr/sbin/tee-supplicant" output/<BOARD>/rootfs.ext2
```

Buildroot 构建过程中也会产出一份 `debugfs`，位于
`sources/buildroot/output/host/sbin/debugfs`。

## 9. 已知限制

- 第 3.1 节的静默丢弃行为是 kconfig 的固有特性，只能靠比对发现。
- 第 6 节的两个 OP-TEE 缺陷尚未合并修复。
- Bootlin stable 工具链的内核头文件停在 5.4，限制了可选软件包的范围。
- manifest 中 Buildroot 曾使用分支而非 commit；固定到具体 commit 的改动尚未
  合并，在此之前 `./build.sh init` 每次取到的树可能不同，本地与 CI 的构建结果
  因此可能不一致。
- rootfs 中没有任何图形或音频组件，它是纯控制台系统。需要 HMI 时应选择其它
  rootfs。

## 10. 故障排查

**某个软件包没有出现在 rootfs 中，且构建没有报错**
该符号很可能被 kconfig 丢弃。用第 3.2 节的方法比对。

**`ERROR: No hash found for <文件名>`**
见第 4 节，自定义版本需要在 `BR2_GLOBAL_PATCH_DIR` 中提供 hash 文件。

**某个包在 `#include <linux/...>` 处编译失败**
Bootlin stable 工具链的内核头文件为 5.4，该接口可能更新。确认所需的最低内核
头文件版本，必要时为该包提供补丁，或改用其它实现。

**切换 rootfs 类型后产物不对**
`.done.<组件>` 只记录某次组件命令成功结束，不保存输入摘要。切换 rootfs 或修改
配置后需要手工 `clean` 对应组件。
