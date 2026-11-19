---
title: 为什么 OpenWrt 能把几百 MB 的 Linux 内核编成 6.9MB 的固件
date: 2026-11-12 19:30:00
tags:
  - OpenWrt
  - Linux内核
  - 嵌入式
categories:
  - 技术
---

编译 K2P 固件时有个容易忽略的事实：`dl/` 里的内核源码压缩包有上百 MB，而最终刷进路由器的固件只有 6.9MB。三十倍的差距是怎么来的？答案分三层：**大部分源码根本没参与编译，参与编译的被裁剪，裁剪完的又被压缩**。

## 第一层：源码大 ≠ 编译对象多

Linux 内核 tarball 包含的是它支持的**所有架构、所有驱动、所有文件系统**的全集：

- 30+ 个 CPU 架构（x86、ARM64、RISC-V、s390 大型机……K2P 只用 MIPS）
- 几千个驱动（InfiniBand 高速网卡、GPU、声卡、USB 摄像头、游戏手柄……路由器几乎都不用）
- 几十种文件系统（Btrfs、XFS、NFS……固件只用 squashfs/jffs2 两三种）

内核编译由 `.config` 决定，每个选项三态：`y` 编入 / `m` 模块 / **不设（完全不编译，编译器都不碰）**。K2P 的内核配置（`target/linux/ramips/config-5.15`）只开了：

```
CONFIG_MIPS=y            # 只编 MIPS 架构
CONFIG_MT7621=y          # 只编这颗 SoC 的平台代码
CONFIG_SQUASHFS=y        # 只编要用的文件系统
CONFIG_MT76_xx=y         # 只编 mt76 无线
# CONFIG_SND is not set  # 声音？不编
# CONFIG_DRM is not set  # 显卡？不编
... 几万个 "not set"
```

几千个驱动目录里，真正参与编译的可能只有几百个目标文件。**未选中的源文件连编译器都不会碰**——这是缩减的主体。

## 第二层：链接后三轮瘦身

编译链接出的原始内核远比最终固件里的大，瘦身分三步：

```
build_dir/.../vmlinux        ~30-50MB   ← 原始内核（带符号表、调试信息、未压缩）
  ↓ strip 去掉符号和调试段
内核镜像                     ~3-6MB
  ↓ LZMA/XZ 压缩（内核自带自解压头，启动时 CPU 自解压进内存）
压缩后内核                   ~2-3MB     ← 固件里的那个
```

- **strip**：调试信息另存（`vmlinux` 符号文件留着排错用），固件里不带
- **压缩**：LZMA/XZ 能压到 1/3 左右，MIPS 启动早期解压开销很小
- **架构红利**：MIPS 代码密度不错，且配置剔除了非必要特性

再加上同样被裁剪压缩的 squashfs rootfs（busybox/musl/LuCI 等用户态），最终 sysupgrade.bin 是 6.9MB。

## 第三层：用户态也在做同样的事

固件里的用户态同样贯彻"全集与子集"：

- busybox：一个二进制通过配置裁剪内置几百个命令的子集
- musl libc：比 glibc 小一个数量级，专为嵌入式设计
- 每个包的 Makefile 都可以 `--disable-xxx` 掉不需要的特性

## 动手验证

在自己的编译目录里可以直接观察这个过程：

```bash
ls -lh build_dir/target-*/linux-*/vmlinux        # 原始内核有多大
ls -lh build_dir/target-*/linux-*/loader.bin     # 压缩后的内核部分
du -sh build_dir/target-*/root-*/               # 未压缩的 rootfs
ls -lh bin/targets/*/*/​*sysupgrade.bin          # 最终固件
```

## 换个角度理解

- tarball 的大小代表"Linux 能做什么的全集"
- 固件的大小代表"这台设备需要什么"
- 构建系统（Kbuild）的角色就是**按配置从全集中抽取子集**，链出最小可用系统

这也解释了为什么同一份 OpenWrt 源码能编出 3MB 的 K2P 固件和几百 MB 的 x86 全家桶——**源码是全集，`.config` 才是灵魂**。编译耗时同理：我那次 41 分钟里，工具链占了 60% 以上，内核和包的编译量本身就很小，因为"没选的都不编"。
