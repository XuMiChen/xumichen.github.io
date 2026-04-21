---
title: Windows 双网卡场景下的自动路由切换实践（WiFi 内网 + 有线外网）
date: 2026-03-18 14:25:00
tags:
  - 网络
  - Windows
  - PowerShell
categories:
  - 技术
---

## 一、背景

在日常开发或企业办公环境中，经常会遇到这样的网络结构：

* 网线 连接外网
* WiFi 连接内网
* 内网环境（IP、网关）可能会动态变化
* 网线 WiFi 可互换 内外网

为了避免手动频繁修改路由，希望实现：

> 根据当前连接的 WiFi 自动识别内网环境，并动态更新 Windows 路由表

---

## 二、整体思路

核心流程如下：

```
监听网络变化 → 获取 WiFi 信息 → 判断是否为内网 → 动态修改路由
```

实现方式：

* PowerShell 脚本：负责逻辑判断与路由操作
* 任务计划程序：负责自动触发执行

---

## 三、核心脚本实现

### 1. 获取当前无线网卡

```powershell
$wifi = Get-NetAdapter | Where-Object {
    $_.Status -eq "Up" -and $_.InterfaceDescription -match 'Wireless|Wi-Fi|802.11|WLAN'
} | Select-Object -First 1
```

---

### 2. 获取 WiFi IPv4 地址（过滤无效 IP）

```powershell
$ip = Get-NetIPAddress -AddressFamily IPv4 -InterfaceIndex $wifi.ifIndex |
    Where-Object { $_.IPAddress -notlike "169.*" } |
    Select-Object -ExpandProperty IPAddress -First 1
```

说明：

* `169.254.*` 是 DHCP 失败产生的无效地址
* 必须过滤，否则会误判网络状态

---

### 3. 获取当前 WiFi 名（SSID）

```powershell
$ssid = netsh wlan show interfaces |
    Select-String '^\s*SSID\s*:\s*(.+)$' |
    Where-Object { $_.Line -notmatch 'BSSID' } |
    Select-Object -First 1 |
    ForEach-Object { $_.Matches[0].Groups[1].Value.Trim() }
```

---

### 4. 判断是否为目标内网

假设内网 WiFi 名以某个前缀开头，并且 IP 为 `10.*` 网段：

```powershell
if ((-not $ssid.StartsWith("目标前缀", [System.StringComparison]::OrdinalIgnoreCase)) -or ($ip -notlike "10.*")) {
    exit 0
}
```

说明：

* SSID：确认连接的是目标网络
* IP：确认地址合法

---

### 5. 动态获取网关

```powershell
$gateway = Get-NetRoute -InterfaceIndex $wifi.ifIndex -DestinationPrefix "0.0.0.0/0" |
    Where-Object { $_.NextHop -and $_.NextHop -ne "0.0.0.0" } |
    Sort-Object RouteMetric |
    Select-Object -ExpandProperty NextHop -First 1
```

---

### 6. 更新路由

```powershell
# 删除默认路由
route delete 0.0.0.0 mask 0.0.0.0 $gateway

# 删除旧内网路由
route delete 10.0.0.0 mask 255.0.0.0 $gateway

# 添加新路由
route add 10.0.0.0 mask 255.0.0.0 $gateway metric 3 if $($wifi.ifIndex)
```

---

## 四、自动触发执行

通过 Windows 任务计划程序实现自动运行。

### 1. 触发器配置

使用系统网络事件：

* 日志：`Microsoft-Windows-NetworkProfile/Operational`
* 事件：

  * `10000`：网络连接变化
  * `10001`：网络断开

同时增加：

* 登录时触发（兜底）

---

### 2. 操作配置

使用 VBS 脚本隐藏运行 PowerShell。

#### VBS 内容：

```vbscript
Set WshShell = CreateObject("WScript.Shell")
WshShell.Run "powershell.exe -NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File ""D:\...\update-inner-route.ps1""", 0, False
```

#### 任务计划设置：

* 程序：`wscript.exe`
* 参数：VBS 脚本绝对路径
* 起始目录：脚本所在目录

---

## 五、常见问题

### 1. 获取不到 WiFi IP

原因：

* 网卡名称不是固定的 `"Wi-Fi"`
* WiFi 未连接

解决：

```powershell
Get-NetAdapter
Get-NetIPAddress -AddressFamily IPv4
```

---

### 2. 中文日志显示为 `????`

原因：

* 编码不一致（PowerShell 5.1 常见问题）

解决：

* 脚本保存为 **UTF-8 with BOM**

---

### 3. 修改 Windows 密码后任务失效

原因：

* 任务计划保存了旧凭据

解决：

* 重新输入密码
* 或改为“仅用户登录时运行”

---

### 4. 空密码无法运行任务

原因：

* Windows 不允许空密码账户用于后台任务

解决：

* 改任务运行方式（推荐）

---

### 5. PowerShell 窗口弹出

原因：

* 任务在用户会话中运行

解决：

* 使用 VBS 隐藏执行

---

## 六、总结

本方案的关键在于：

1. 正确识别当前网络（SSID + IP）
2. 动态获取网关
3. 自动触发执行（任务计划）

通过 PowerShell + 任务计划程序，可以实现：

* 自动感知网络变化
* 自动更新路由
* 无需人工干预

适用于典型的：

> 内网 WiFi + 外网有线 的双网卡办公环境
