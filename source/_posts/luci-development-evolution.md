---
title: LuCI 开发模式演进：用同一个功能的两代实现讲透 CBI、JS View 与 luci-proto
date: 2026-11-18 19:00:00
tags:
  - OpenWrt
  - LuCI
  - 前端
  - 嵌入式
categories:
  - 技术
---

校园网认证在 OpenWrt 上有两代实现：老锐捷客户端 mentohust 的界面 `luci-app-mentohust`（Lua CBI 写法），和新版 `luci-proto-minieap`（JS + 协议注册写法）。同一个功能的两代代码放在一起，正好把 LuCI 十几年的架构演进讲透。

## 三种开发模式总览

| | CBI | JS View | luci-proto |
|---|---|---|---|
| 语言 | Lua | JavaScript | JavaScript（+ netifd shell）|
| 渲染 | 服务端 | 客户端（浏览器）| 客户端 |
| 本质 | 表单生成器 | 任意页面 | 接口协议类型 |
| 入口 | controller 注册菜单 | controller/URL | 接口编辑页协议下拉框 |
| 时代 | 2008 经典 | 2019+ 主流 | 配合 JS View |

## CBI：Lua 声明式表单（经典模式）

`luci-app-mentohust` 的核心就一个 CBI 文件，全文没有一行 HTML：

```lua
m = Map("mentohust", "MentoHUST")          -- 绑定 /etc/config/mentohust

s = m:section(TypedSection, "mentohust", "Settings")
s:tab("base", "基本设置")

enable = o:taboption("base", Flag,   "enable",  "启用")      -- 复选框
name   = o:taboption("base", Value,  "username", "用户名")    -- 文本框
pass   = o:taboption("base", Value,  "password", "密码")
pass.password = true                                        -- 显示为圆点
ifname = o:taboption("base", ListValue, "ifname", "接口")     -- 下拉框
for k, v in ipairs(luci.sys.net.devices()) do                -- 选项动态枚举网卡
    ifname:value(v)
end

local apply = luci.http.formvalue("cbi.apply")
if apply then
    io.popen("/etc/init.d/mentohust restart")                -- 保存时重启服务
end
```

几十行声明，框架自动生成完整的表单页面、UCI 读写、保存应用。状态显示也是土办法：`pidof` 查进程、`ping` 探测在线。

CBI 的优点是极简，缺点也明显：每次交互刷新整页、跑在路由器 CPU 上（Lua 解释器对 16MB flash 设备是负担）、动态交互要靠 hack。官方已将其转入维护模式。

## JS View：客户端渲染（现行主流）

JS View 把逻辑放进浏览器，路由器只提供数据接口（ubus/rpcd）。API 形状刻意保持和 CBI 相似，迁移成本低：

```javascript
'use strict';
'require view';
'require form';

return view.extend({
    render: function() {
        var m = new form.Map('myapp', _('我的应用'));
        var s = m.section(form.NamedSection, 'main', 'myapp');

        var o = s.option(form.Flag, 'enable', _('启用'));
        o = s.option(form.Value, 'server', _('服务器地址'));
        o.datatype = 'host';

        return m.render();
    }
});
```

独有的现代能力：局部刷新、轮询（`poll.add`）、异步数据（`rpc.declare` 声明 ubus 调用）、前端即时校验（`datatype`）。

## luci-proto：把功能变成"系统的一等公民"

`luci-proto-minieap` 是另一个思路：不做独立页面，而是**注册一种网络协议**，让 miniEAP 认证出现在 接口编辑页的协议下拉框里（和 PPPoE、WireGuard 并列）：

```javascript
'require form';
'require network';

network.registerErrorCode('MISSING_USER_OR_PASS', _('账号或密码缺失'));

return network.registerProtocol('minieap', {
    getI18n: function() { return _('MiniEAP client'); },
    renderFormOptions: function(s) {
        var o = s.taboption('general', form.Value, 'username', _('用户名'));
        o = s.taboption('general', form.Value, 'password', _('密码'));
        o.password = true;
    }
});
```

包里同时带了 `/lib/netifd/proto/minieap.sh`（netifd 协议脚本）。注册成协议后**白得一整套系统能力**：

- 接口生命周期管理：接口启用自动拉起认证进程，掉线自动重连
- LuCI 接口页的状态灯、重连按钮直接反映认证状态
- 防火墙 zone 归属、netifd 事件全部原生联动

对比之下，CBI 时代的应用是"外挂"——自己 pidof 查状态、自己 init.d 重启，和系统若即若离。这是两代实现最本质的差别：**旧版是"给程序做了个网页表单"，新版是"把程序变成系统的一种接口"**。

## ucode：长得像 JS 的另一个东西

现代 luci-app 的完整三明治是"JS 前端 + ucode 桥接层 + C 核心"：

```
luci-app-xxx/
├── htdocs/.../view/xxx.js                      ← 前端
├── root/usr/share/rpcd/ucode/xxx.uc            ← ucode 桥接层（自定义 ubus 方法）
├── root/usr/share/rpcd/acl.d/luci-app-xxx.json ← ACL 权限声明（漏了它前端调不到接口，最常见坑）
└── po/zh_Hans/                                 ← 翻译
```

注意 ucode **不是 JavaScript**：它是 OpenWrt 自研的脚本语言，C 写的解释器、几百 KB 大小，**语法刻意采用 ECMAScript 风格**，但跑在路由器上、标准库是 fs/uci/ubus 这些系统模块。辨别方法：`'require form'` 是前端 JS，`require('uci')` 是 ucode。

ucode 出场的场景是需要自定义数据接口时（firewall4 就整个用 ucode 重写了）；纯配置型应用（如 minieap 这种表单字段与 UCI 一一对应的）不需要它，netifd 协议体系已经把桥修好了。

## 选型口诀

- 给某个 UCI 配置做管理页 → CBI 快速搞定（旧项目）/ JS View（新项目）
- 需要动态交互、实时状态 → JS View
- 功能本质是"一种上网方式" → luci-proto，没有例外
- 现实常见组合：一个 luci-app 管配置 + 一个 luci-proto 让流量进系统（WireGuard/OpenVPN 官方实现都是这个组合）

## 尾声：一个时代的注脚

这两代实现的命运也值得记录：mentohust 的 Lua 界面随 21.02 分支封存（后端认证协议太老被移除），luci-app-minieap 在 2025 年被重写成 proto 形态——维护者的 commit message 干脆利落："Remove deprecated package. Use luci-proto-minieap instead."

代码会过时，但这条演进线（服务端渲染 → 客户端渲染 → 系统集成）几乎适用于所有嵌入式 Web 界面的发展史。
