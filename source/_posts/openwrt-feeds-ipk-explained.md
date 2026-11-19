---
title: OpenWrt 的 feeds 机制：源、包、ipk 到底是怎么回事
date: 2026-11-13 18:40:00
tags:
  - OpenWrt
  - Linux
categories:
  - 技术
---

刚接触 OpenWrt 编译时最困惑的一组问题：feeds 是什么？update 和 install 有什么区别？为什么 `package/` 里大多只有一个 Makefile？`bin/packages` 和 `bin/targets` 为什么都有 ipk？这篇把这条链路一次讲透。

## feeds 三步各自做什么

```
① ./scripts/feeds update -a
   └─ git clone 各 feed 仓库到 feeds/ 目录
      下载的是各包的"打包定义"（Makefile），不是源代码

② ./scripts/feeds install -a
   └─ 在 package/feeds/<feed名>/ 下创建软链接指向 feeds/ 里的包
      作用：让 menuconfig 和 make 能"看见"这些包

③ make download / make package/foo/compile
   └─ 这时才按包 Makefile 里的 PKG_SOURCE_URL 下载真正的上游源码
```

| 步骤 | 产生的东西 | 类比 |
|---|---|---|
| feeds update | 配方仓库 | 菜谱库 |
| feeds install | 软链接进 package/ | 把菜登记上菜单 |
| make download/compile | 上游源码 + 产物 | 真材实料 |

关键点：**menuconfig 和 make 扫描包的唯一来源是 `package/` 目录树**。feeds 仓库 clone 在 `feeds/` 里时构建系统根本不看那里，install 这一步存在的意义就是把外部仓库"链接"进可见范围。源码树自带的包（uci、procd 等）天然在 package/ 里，所以不需要 install。

## 包目录里为什么只有一个 Makefile

看 Lean 树里的 mentohust，整个包就一个 Makefile：

```makefile
PKG_NAME:=mentohust
PKG_SOURCE_URL:=https://github.com/KyleRicardo/MentoHUST-OpenWrt-ipk.git
PKG_SOURCE_VERSION:=557cffca8032b6d8ac948be8a79255dc64a1915d
```

它不包含源码，只是配方：从哪取、锁哪个提交、怎么交叉编译。真正写源码的场景只有两种——OpenWrt 自研组件（uci/procd 这类没有上游），或包自带 patches/ 子目录给上游源码打补丁。**仓库里是差异，不在仓库里的是原厂**。

## ipk 是什么

ipk 是统一的打包格式（ar 归档）：

```
foo.ipk
├── control（元数据：包名、版本、Depends）
└── data.tar.gz（解压到路由器文件系统的实际文件）
```

**里面装什么完全取决于打包者**。同一个格式下，可能是：

| 类型 | 例子 | 内容 |
|---|---|---|
| 后端守护进程 | mentohust、miniupnpd | 编译出的二进制 + init 脚本 |
| 纯前端 | luci-app-firewall | 只有 JS/JSON（PKGARCH:=all，架构无关）|
| 库 | libpcap1 | 只有 .so |
| 内核模块 | kmod-usb3 | .ko 文件，强绑内核版本 |
| 元包 | luci | **空壳**，只有 Depends，装它=拉一串别的包 |
| 数据固件 | kmod-mt7615-firmware | 厂商 blob，零代码 |

## "装一个包就有全部功能"的真相

装 `luci-app-upnp` 一个命令，页面和服务都有了，怎么回事？

```
opkg install luci-app-upnp
  ① 读元数据：Depends: miniupnpd → 未安装
  ② 从同一源自动装上 miniupnpd.ipk（后端）
  ③ 装 luci-app-upnp 本体（前端文件到 /www）
```

是**依赖链在安装瞬间拉进来一串 ipk**，不是单个 ipk 里塞了一切。这也解释了手动装本地 ipk 的规则：`opkg install a.ipk b.ipk` 一条命令给全，opkg 自己排顺序；分开装则必须先依赖后本体，反了会直接报 `unresolved dependency`。

很多 luci 包"看起来只有前端"，是因为 LuCI 的哲学就是**给已有系统组件做界面**——firewall4、netifd、opkg 这些后端本来就是固件标配，界面包自然只有 JS 和 ACL 声明。

## 两个 bin 目录的区别

编译完的产物分两处，正好对应 opkg 的双源结构：

```
bin/
├── packages/mipsel_24kc/        ← 架构级"应用商店"（按 feed 分目录）
│   ├── base/  luci/  packages/ ...
│   （0 个 kmod）
└── targets/ramips/mt7621/       ← 设备级"内核专属货架"
    ├── packages/                ← kernel.ipk + 全部 kmod-*（锁内核版本和 hash）
    ├── openwrt-...-sysupgrade.bin
    └── sha256sums
```

kmod 必须按"内核实例"分组（内核无稳定 ABI，跨内核版本装载可能崩溃），所以跟着 target 走；应用包只关心 CPU 架构，跨设备共享。路由器上 `/etc/opkg/distfeeds.conf` 里的两条源（`targets/.../packages` 和 `packages/mipsel_24kc/base`）就对应这两个目录——官方服务器是把它们原样挂到网上的。

## 顺手就能搭的本地源

自编固件 + 自编 kmod 内核 hash 天然匹配，把 bin/ 目录用 `python3 -m http.server 8000` 起个服务，路由器写上 customfeeds 就是一个完美的局域网源，解决了"官方源 kmod 装不上自编固件"的经典问题。

## 总结

- feeds 是"配方仓库"，install 只建软链接，不碰源码
- 包目录只有 Makefile 是常态，源码在上游
- ipk 是内容无关的容器，依赖声明才是包之间的连接方式
- bin/ 的两个目录 = opkg 双源 = "架构通用"与"内核绑定"的分界

理解了这条链路，加包、移植、搭源、分发都是顺着它在做增量。
