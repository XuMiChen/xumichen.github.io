---
title: 在官方 OpenWrt 里移植第三方包：mentohust 17 秒编译实战
date: 2026-11-12 20:00:00
tags:
  - OpenWrt
  - 校园网
  - 编译
categories:
  - 技术
---

校园网认证客户端 mentohust 只存在于 LEDE/ImmortalWrt 等分支的源里，官方 OpenWrt 没有。但它本质上只是一个带 Makefile 的"打包定义"，完全可以搬到官方源码树自己编译。这篇记录完整流程，全程实际耗时约 17 秒。

## 外部包接入的三种方式

| 方式 | 适用场景 |
|---|---|
| 直接拷贝包目录进 `package/` | 从别的源码树/GitHub 拿现成的包定义（本文用的）|
| `src-link` 挂本地目录当 feed | 自己开发包，源码在独立目录 |
| Makefile 引用旁边 src/ | 纯源码项目，不想依赖外部下载 |

注意：**本地放进 `package/` 的包不需要 `feeds install`**——install 的作用只是把外部 feed 仓库软链接进 package/ 让构建系统看见，本地包天然可见。

## 实战：从 LEDE 树搬 mentohust

**1. 取包定义**

```bash
curl -sL -o /tmp/lede.tar.gz https://github.com/coolsnowwolf/lede/archive/master.tar.gz
tar xzf /tmp/lede.tar.gz lede-master/package/lean/mentohust
cp -r lede-master/package/lean/mentohust ~/openwrt/package/
```

拷过来的只有一个 Makefile——这是常态，包的真正源码在 `PKG_SOURCE_URL` 指向的上游仓库，OpenWrt 树只维护"从哪取、锁哪个提交、怎么编"。

**2. 选包并补全依赖**

```bash
cd ~/openwrt
echo 'CONFIG_PACKAGE_mentohust=m' >> .config
make defconfig
```

`make defconfig` 会自动补依赖。验证一下：

```bash
grep -E "mentohust|libpcap" .config
# CONFIG_PACKAGE_libpcap=m      ← 依赖被自动带出
# CONFIG_PACKAGE_mentohust=m
```

**3. 单包编译**

```bash
make package/mentohust/compile V=s
```

首次会自动下载上游源码、用已有的 toolchain 交叉编译、打 ipk。实测输出：

```
Packaged contents of ... into .../mentohust_0.3.1-1_mipsel_24kc.ipk
time: package/mentohust/compile#1.26#0.27#16.82   ← 17 秒
```

**4. 产物**

```
bin/packages/mipsel_24kc/base/libpcap1_1.10.4-1_mipsel_24kc.ipk   ← 依赖
bin/packages/mipsel_24kc/base/mentohust_0.3.1-1_mipsel_24kc.ipk   ← 本体
```

## 部署与安装顺序

```bash
scp bin/packages/mipsel_24kc/base/{libpcap1,mentohust}_*_mipsel_24kc.ipk root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "cd /tmp && opkg install libpcap1*.ipk mentohust*.ipk"
```

依赖和本体可以一条命令给全，opkg 按 Depends 自动排序；分开装则必须先依赖后本体，反了会报 `unresolved dependency` 直接拒绝。

## 几个值得注意的点

1. **为什么 17 秒这么快**：toolchain（首次编译花了 26 分钟建的那个）和内核全部复用，单包编译只执行该包自己的依赖链
2. **`make package/foo/clean` 的陷阱**：clean 会删掉 build_dir 里的源码并重新解压上游包，手改过的源码会丢——开发自己的包时要么用 src-link 指向独立目录，要么把改动做成 patch 放进包的 patches/ 目录
3. **想内置进固件**：把 `=m` 改成 `=y` 再 `make -j$(nproc)`，几分钟出带 mentohust 的新固件
4. **`make defconfig` 是配置的补全器**：手写片段 → defconfig → grep 验证，比截图 menuconfig 可靠得多
5. **版本代际**：不同分支的包定义可能耦合各自主树的私有改动，本文的 mentohust 在官方 23.05 树原样可用；遇到编不过的深耦合包，读报错小改 Makefile，实在不行就换用它的源码树编译

## 延伸：怎么找"某分支专属"的包

一个实用技巧：GitHub 上看到包在某分支消失，用 API 查它的提交历史，commit message 通常写明了原因：

```
GET /repos/immortalwrt/packages/commits?path=net/mentohust
→ 2025-08-26 "mentohust: drop legacy package. Use minieap instead."
```

mentohust 是被 minieap 取代（锐捷新协议），luci-app-minieap 则被重写成了 luci-proto-minieap——查证比猜测可靠得多。

## 总结

外部包移植到官方树，本质就是"把打包定义放进 package/，让 defconfig 拉起依赖，单包编译出 ipk"。加上之前搭好的局域网 opkg 源（python http.server 起 bin/ 目录 + customfeeds 指过去），一套"官方源码树 + 自选第三方包"的自给自足体系就完整了。
