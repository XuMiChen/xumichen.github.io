---
title: H3C 路由器登录机制与 Telnet 开启接口分析
date: 2026-03-29 10:40:00
tags:
  - H3C
  - telnet
  - 网络
---

最近在分析某 H3C 路由器（NX54）时，对其 Web 登录流程和隐藏接口（Telnet 开启）做了一些测试，踩了不少坑，这里做一个简要总结。

---

## 一、登录流程分析

### 1. 登录接口

登录接口为：

POST /router_password_mobile.asp

请求参数：

login_passwd=加密后的密码  
psd=加密后的密码

⚠️ 注意：

- 两个字段是一样的
- 都是 encryptPwd() 处理后的密码
- 不是“用户名 + 密码”的组合

---

### 2. 正确登录请求示例

curl -d 'login_passwd=加密后的密码' \
     -d 'psd=加密后的密码' \
     http://路由管理地址/router_password_mobile.asp

---

### 3. 登录态机制（重点）

该设备不依赖 cookie / session，而是：

✅ 按客户端 IP 记录登录状态

表现为：

- 登录成功后：同一 IP 下无痕 / 新浏览器仍然已登录
- 换设备（新 IP）：必须重新登录

---

### 4. 登录状态刷新机制

再次发送登录请求会覆盖当前 IP 的登录状态：

- 正确密码 → 登录成功
- 错误密码 → 当前 IP 登录状态失效

---

## 二、Telnet 开启接口

### 1. 接口路径

/goform/aspForm

示例：

curl "http://路由管理地址/goform/aspForm?param=1&SetTo=8&CMD=Asp_SetTelnetDebug&RMTELNETEN=1"

---

### 2. 生效条件

必须来自已登录 IP，否则不会生效。

---

### 3. 调用流程

# 1. 登录
curl -d 'login_passwd=xxx' -d 'psd=xxx' http://路由管理地址/router_password_mobile.asp

# 2. 调用接口
curl "http://路由管理地址/goform/aspForm?...&RMTELNETEN=1"

---

## 三、常见坑

### 坑1：误以为 login_passwd 是用户名
实际上两个字段都是密码（加密后）

### 坑2：忽略前端加密
真实流程会调用 encryptPwd()

### 坑3：误判接口未授权
同 IP 登录后会一直有效

### 坑4：错误登录会清除状态
错误密码会让当前 IP 失效

### 坑5：无痕模式无效
认证与浏览器无关

---

## 四、安全分析

认证模型：

登录 → 标记 IP → 信任请求

缺点：

- 粒度粗（按 IP）
- 易受 CSRF
- 无隔离

---

## 五、总结

这是一个典型的按 IP 绑定登录态的嵌入式设备认证模型。
