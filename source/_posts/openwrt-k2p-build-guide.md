---
title: 从零编译 OpenWrt 固件：斐讯 K2P 实战全记录
date: 2026-11-12 18:30:00
tags:
  - OpenWrt
  - K2P
  - 路由器
  - WSL2
categories:
  - 技术
---

家里的斐讯 K2P（A2 版，MT7621 平台）一直跑着 Breed，趁周末从零编译了一份官方 OpenWrt 23.05 固件，全程记录下来。相比"照着教程敲"，这篇文章更关注每一步**背后发生了什么**，以及真实的时间开销。

## 为什么 K2P 适合练手

- MT7621 + 16MB flash + 128MB RAM，官方主线支持（target: `ramips/mt7621`，设备名 `phicomm_k2p`）
- 已刷 Breed，刷错随时能救，试错成本为零

## 环境准备（WSL2 Ubuntu）

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib \
  g++-multilib gettext git libncurses-dev libssl-dev python3-distutils \
  python3-setuptools rsync swig unzip zlib1g-dev file wget
```

拉源码并锁定稳定分支：

```bash
git clone --depth 1 -b openwrt-23.05 https://github.com/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
```

## 配置：一个语言选项的坑

最小配置三行就能编出固件，但没有 LuCI 没有中文。我想要的配置是 LuCI + 中文 + UPnP，直觉写法：

```ini
CONFIG_TARGET_ramips=y
CONFIG_TARGET_ramips_mt7621=y
CONFIG_TARGET_ramips_mt7621_DEVICE_phicomm_k2p=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-i18n-base-zh-cn=y   # ❌ 会被 defconfig 丢弃
```

问题出在最后一行：单独选翻译包而不开语言总开关，依赖不满足，`make defconfig` 会静默清掉它。正确写法是：

```ini
CONFIG_LUCI_LANG_zh_Hans=y               # ✅ 语言总开关
CONFIG_PACKAGE_luci-app-upnp=y
```

然后 `make defconfig` 补全依赖，会看到 `luci-i18n-base-zh-cn`、`luci-i18n-firewall-zh-cn` 等自动全选上。这个符号名是我在 `feeds/luci/luci.mk` 里 grep 出来的，符号名每代版本都可能变，**猜不如查源码树**。

## 下载前置：断网也能编译

所有网络操作可以前置（克隆、feeds、`make download`），之后编译基本不再需要网络：

```bash
make download -j8
make download -j1 V=s   # 单线程再跑一遍，能暴露具体哪个包下载失败
```

`make download` 必须在 `.config` 定稿之后执行，改了配置勾新包就要重新 download。

## 编译：真实时间线

`make -j$(nproc) V=s`，i7-1165G7（WSL2 分到 8 线程 + 7.6GB 内存），实测：

| 阶段 | 耗时 |
|---|---|
| tools（宿主机工具） | ~7 分钟 |
| binutils + gcc-initial + musl + gcc-final（工具链） | ~19 分钟 |
| 内核 5.15.211 + 数百软件包（**-j8 下并行**） | ~10 分钟 |
| 打包生成镜像 | ~5 分钟（含一次失败重跑） |
| **合计** | **约 41 分钟** |

比网上流传的"1~3 小时"快不少，原因有三：`-j8` 下内核和包是并行的（不是先内核后包的串行模型）；我只选了精简包集；下载全部前置了。

一个有意思的观察：`-j8` 时会在进程里同时看到内核编译和 `csstidy`（LuCI 依赖的包）在编——target 和 package 是两条并行流水线。

## WSL2 的坑：PATH 里的空格断掉了编译

编译到最后一步 `package/install` 报错：

```
find: The relative path 'Files' is included in the PATH environment variable,
which is insecure in combination with the -execdir action of find.
make[2]: *** [package/Makefile:74: package/install] Error 1
```

原因：WSL2 默认把 Windows PATH 拼进 Linux PATH，Windows 侧带空格且未加引号的路径（`C:\Program Files\...`）在某种 shell 环境下被截断成相对路径 `Files`，OpenWrt 打包用的 `find -execdir` 出于安全要求 PATH 里不能有相对路径，直接拒绝执行。

修复很简单，重跑时清洗 PATH：

```bash
env PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  make -j$(nproc)
```

因为 make 是增量编译，95% 已完成的工作全部有效，重跑只花了 5 分钟就出了固件。根治方案是在 `/etc/wsl.conf` 里加 `[interop] appendWindowsPath=false`。

## 产物与验证

```
bin/targets/ramips/mt7621/
├── openwrt-ramips-mt7621-phicomm_k2p-squashfs-sysupgrade.bin   # 正式刷这个（6.9MB）
├── openwrt-ramips-mt7621-phicomm_k2p-initramfs-kernel.bin      # 内存试运行版
└── sha256sums
```

6.9MB 对 16MB flash 余量充足。刷机走 Breed：按住 Reset 上电进 Breed → 固件更新 → 只勾"固件"上传 sysupgrade.bin。

## 几条经验

1. `make defconfig` 是配置的"补全器"：写片段 → defconfig → grep 验证，比 menuconfig 截图可靠
2. 编译中断不用怕，make 的时间戳机制保证重跑从断点续编，**报错时千万别 `make clean`**
3. 首次编译慢的大头是工具链（占 60%+），之后增量编译只要几分钟——单包编译甚至只要 17 秒
4. WSL2 编译注意内存默认只有物理机一半（.wslconfig 的 `memory=12GB` 可调），8 并发下 7.6GB 没触发 OOM 属于幸运

整个流程走完，最大的收获不是固件本身，而是把"配置→下载→工具链→内核→包→镜像"这条流水线跑通了一遍——之后再想加包、换设备、移植第三方应用，都只是在这条流水线上做增量。
