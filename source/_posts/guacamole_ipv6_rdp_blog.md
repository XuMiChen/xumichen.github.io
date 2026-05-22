---
title: 群晖公网 IPv6 环境下使用 Guacamole 安全访问 Windows 远程桌面
date: 2026-05-22 14:06:00
tags:
  - Guacamole
  - RDP
  - IPv6
  - 群晖
  - Docker
  - 远程桌面
  - 远程访问
  - 网络安全
categories:
  - 技术
  - 远程访问
description: 记录如何在公网 IPv6、群晖反向代理和 Guacamole 的组合下，通过 HTTPS Web 网关安全访问内网 Windows 远程桌面，避免直接暴露 RDP 端口
---

## 背景

家里内网中有多台设备，其中一台群晖已经具备公网 IPv6 访问能力，并且已经配置了域名解析和 HTTPS 证书。

现在希望在外网访问内网中另一台 Windows 电脑的远程桌面。

最开始想到的方案是：既然群晖已经可以被公网访问，那么是否可以通过群晖做端口转发，把内网 Windows 的 RDP 服务转发出来？

例如：

```text
外网客户端
  -> 公网 IPv6 / 域名:13389
  -> 群晖
  -> 内网 Windows:3389
```

技术上可以实现，但安全性并不理想。

---

## 开始前需要准备什么

本文默认已经具备以下条件：

- 群晖已经可以通过公网 IPv6 访问
- 域名已经配置 AAAA 解析到群晖公网 IPv6
- 群晖已经配置 HTTPS 证书
- 群晖已安装 Container Manager
- 内网 Windows 已开启远程桌面
- Windows 防火墙允许内网设备访问 RDP
- 路由器或光猫侧可以控制对外开放的端口

如果还没有公网 IPv6 或域名证书，也可以先在局域网中验证 Guacamole 到 Windows RDP 的连接，确认容器和账号配置没有问题后，再处理公网 HTTPS 入口。

---

## 一、为什么不建议直接转发 RDP

Windows 远程桌面默认使用 TCP 3389 端口。

如果直接把内网 Windows 的 3389 转发到公网，例如：

```text
公网IPv6:13389 -> Windows:3389
```

即使外部访问时使用的是域名：

```text
rdp.example.com:13389
```

本质上仍然是：

```text
公网暴露 RDP 服务
```

域名只是更方便记忆，并不会提高 RDP 本身的安全性。

同样，如果群晖上已经配置了 HTTPS 证书，也不能直接理解为 RDP 被 HTTPS 保护了。

原因是：**RDP 不是 HTTP/HTTPS 协议**。

浏览器访问：

```text
https://example.com
```

走的是 HTTPS。

但 Windows 远程桌面客户端访问：

```text
example.com:13389
```

走的是 RDP 协议。

如果只是 TCP 端口转发，那么 SSL 证书并没有参与这条 RDP 连接。

因此：

```text
域名 + SSL证书 + RDP端口转发
```

并不等于：

```text
RDP被HTTPS保护
```

更准确地说，它只是：

```text
用域名访问一个暴露在公网的 RDP 转发端口
```

---

## 二、方案选型

### 方案 1：直接端口转发

结构：

```text
外网
  -> 群晖公网IPv6:13389
  -> Windows:3389
```

优点：

```text
配置简单
可以直接用 mstsc 连接
```

缺点：

```text
RDP 暴露在公网
容易被扫描
存在爆破和漏洞利用风险
安全性较差
```

不推荐长期使用。

---

### 方案 2：SSH 隧道

结构：

```text
外网电脑
  -> SSH连接群晖或跳板机
  -> 本地端口转发
  -> Windows:3389
```

例如：

```bash
ssh -L 13389:<Windows内网IP>:3389 <SSH用户名>@<你的域名>
```

然后本地远程桌面连接：

```text
127.0.0.1:13389
```

优点：

```text
不直接暴露 RDP
安全性比端口转发高
```

缺点：

```text
每次连接前需要建立 SSH 隧道
使用体验不如浏览器访问直观
```

---

### 方案 3：VPN / Tailscale / WireGuard

结构：

```text
外网设备加入虚拟内网
  -> 直接访问 Windows 内网 IP 或虚拟 IP
```

优点：

```text
安全性高
不暴露 RDP
适合长期使用
```

缺点：

```text
每台客户端都要安装或配置 VPN
如果只想通过浏览器访问，不够方便
```

---

### 方案 4：Guacamole Web RDP 网关

结构：

```text
外网浏览器
  -> HTTPS
  -> 群晖反向代理
  -> Guacamole
  -> guacd
  -> Windows:3389
```

优点：

```text
公网只暴露 HTTPS
不直接暴露 Windows 3389
浏览器即可访问远程桌面
适合已有域名和证书的群晖环境
```

缺点：

```text
画质和流畅度通常不如原生 mstsc
需要维护 Guacamole 容器
公网仍然暴露 Guacamole 登录入口
```

综合来看，如果目标是：

```text
通过域名 + HTTPS + 浏览器访问内网 Windows 桌面
```

那么 Guacamole 是比较合适的方案。

---

## 三、最终架构

最终采用 Guacamole 方案。

整体访问路径如下：

```text
外网浏览器
  -> https://rdp.example.com/guacamole
  -> 群晖反向代理 HTTPS
  -> http://127.0.0.1:8080/guacamole
  -> Guacamole Web 容器
  -> guacd 容器
  -> 内网 Windows:3389
```

公网只需要开放：

```text
443
```

不开放：

```text
3389
13389
8080
5000
5001
```

其中：

```text
443：外部 HTTPS 访问入口
8080：Guacamole 容器内部 Web 端口，只给群晖本机反代访问
3389：Windows RDP 端口，只在内网被 Guacamole 访问
```

---

## 四、群晖 Container Manager 部署 Guacamole

在群晖中创建目录：

```text
/volume1/docker/guacamole
/volume1/docker/guacamole/init
/volume1/docker/guacamole/postgres
```

然后在 Container Manager 中创建项目，使用 Docker Compose 部署。

推荐 compose：

```yaml
services:
  guacamole-init:
    image: guacamole/guacamole:latest
    container_name: guacamole-init
    restart: "no"
    volumes:
      - /volume1/docker/guacamole/init:/init
    entrypoint: /bin/sh
    command:
      - -c
      - |
        if [ ! -f /init/initdb.sql ]; then
          /opt/guacamole/bin/initdb.sh --postgresql > /init/initdb.sql
        fi

  postgres:
    image: postgres:15
    container_name: guacamole-postgres
    restart: unless-stopped
    depends_on:
      guacamole-init:
        condition: service_completed_successfully
    environment:
      POSTGRES_DB: guacamole_db
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: <强密码>
    volumes:
      - /volume1/docker/guacamole/postgres:/var/lib/postgresql/data
      - /volume1/docker/guacamole/init:/docker-entrypoint-initdb.d:ro

  guacd:
    image: guacamole/guacd:latest
    container_name: guacd
    restart: unless-stopped

  guacamole:
    image: guacamole/guacamole:latest
    container_name: guacamole
    restart: unless-stopped
    depends_on:
      - guacd
      - postgres
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_HOSTNAME: postgres
      POSTGRESQL_DATABASE: guacamole_db
      POSTGRESQL_USER: guacamole_user
      POSTGRESQL_PASSWORD: <同一个强密码>
    ports:
      - "127.0.0.1:8080:8080"
```

上面示例为了方便演示使用了 `latest` 镜像。长期使用时更建议固定 Guacamole、guacd 和 PostgreSQL 的版本号，避免镜像更新后行为变化导致排障困难。

这里有一个关键点：

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

这样 Guacamole 的 8080 端口只监听群晖本机，不会直接暴露给局域网或公网。

外部访问只能走群晖反向代理的 HTTPS 入口。

不过这仍然建议部署后实际验证一次：如果局域网或公网能直接访问 `http://<群晖地址>:8080/guacamole`，就说明 8080 没有被限制在本机回环地址，需要检查端口映射或防火墙规则。

---

## 五、数据库初始化说明

Guacamole 需要数据库保存用户、连接配置和权限信息。

因此第一次部署时，需要初始化数据库结构。

上面的 compose 中使用了一个一次性容器：

```text
guacamole-init
```

它的作用是生成初始化 SQL：

```text
/volume1/docker/guacamole/init/initdb.sql
```

然后 PostgreSQL 第一次启动时，会自动执行这个 SQL 文件。

因此正常情况下，不需要再手动执行：

```bash
docker run ...
cat initdb.sql | docker exec ...
docker compose restart
```

只需要通过 Container Manager 创建项目即可。

---

## 六、群晖目录权限问题

部署过程中可能遇到错误：

```text
Bind mount failed: '/volume1/docker/guacamole/init' does not exist
```

原因是群晖不会自动创建 bind mount 目录。

解决方法是手动创建：

```text
/volume1/docker/guacamole/init
/volume1/docker/guacamole/postgres
```

还可能遇到：

```text
cannot create /init/initdb.sql: Permission denied
```

原因是容器内部进程没有权限写入群晖宿主机目录。

本质是：

```text
容器里的用户 ≠ DSM 当前登录用户
```

虽然目录是用 DSM 管理员创建的，但容器里的进程可能是另一个 UID/GID。DSM 的 ACL 权限不一定允许它写入。

为了快速跑通，可以临时给目录递归读写权限：

```text
/volume1/docker/guacamole
```

在 File Station 中：

```text
右键目录
  -> 属性
  -> 权限
  -> 给 administrators 或 Everyone 读写
  -> 应用到子文件夹和文件
```

`Everyone` 并不是必须，只是最简单粗暴的方式。

如果只是为了排查问题，可以临时这样处理；跑通后建议收紧权限，不要长期给无关用户开放写权限。

更精确的方式是通过 SSH 查看容器运行用户：

```bash
sudo docker inspect guacamole-init --format '{{.Config.User}}'
sudo docker run --rm --entrypoint id guacamole/guacamole:latest
sudo docker run --rm --entrypoint id postgres:15
```

查看宿主机目录 UID/GID：

```bash
ls -ldn /volume1/docker/guacamole/init
ls -ldn /volume1/docker/guacamole/postgres
```

然后使用 `chown` 或 `chmod` 精确放行对应 UID/GID。

如果不想处理这类权限问题，也可以改用 Docker 命名卷，让 Docker 自己管理数据目录。

---

## 七、群晖反向代理配置

在 DSM 中进入：

```text
控制面板
  -> 登录门户
  -> 高级
  -> 反向代理服务器
```

新增规则。

来源：

```text
协议：HTTPS
主机名：rdp.example.com
端口：443
```

目的地：

```text
协议：HTTP
主机名：localhost
端口：8080
```

注意不要把来源端口填成 8080。

8080 已经被 Guacamole 容器占用：

```text
群晖本机 8080 -> Guacamole 容器 8080
```

反向代理的来源端口应该是外部访问入口，也就是 443。

最终访问地址：

```text
https://rdp.example.com/guacamole
```

如果启用了 HSTS，建议在确认 HTTPS 访问完全正常后再开启。调试阶段先关闭 HSTS 会更方便。

---

## 八、Guacamole 初始登录和修改密码

Guacamole 默认账号密码通常是：

```text
guacadmin / guacadmin
```

登录后应立即修改密码。

路径：

```text
右上角用户名
  -> Settings
  -> Preferences
  -> Change Password
```

建议进一步操作：

```text
1. 新建自己的管理员账号
2. 用新账号登录确认正常
3. 删除或禁用默认 guacadmin
```

这样可以避免默认账号长期存在。

---

## 九、创建 Windows RDP 连接

进入 Guacamole 后：

```text
Settings
  -> Connections
  -> New Connection
```

基本配置：

```text
名称：自定义，例如 Windows-PC
协议：RDP
```

注意一个容易填错的地方：  
**“GUACAMOLE 代理参数（GUACD）”不要填写 Windows 的地址。**

这部分是 Guacamole Web 连接 guacd 代理服务用的。因为 docker compose 中已经配置了：

```yaml
GUACD_HOSTNAME: guacd
```

所以页面中的 GUACD 代理参数可以留空。

Windows 地址应填写在：

```text
参数
  -> 网络
```

例如：

```text
主机名：<Windows内网IP>
端口：3389
```

认证部分：

```text
用户名：<Windows用户名>
密码：<Windows密码>
域：家用本地账号一般留空
安全模式：NLA
忽略服务器证书：勾选
```

如果是 Windows 本地账户，可以使用：

```text
<Windows用户名>
```

或者：

```text
.\<Windows用户名>
```

如果是 Microsoft 账户登录 Windows，用户名可能需要写成：

```text
MicrosoftAccount\<邮箱地址>
```

其中“忽略服务器证书”主要是为了兼容家用 Windows 默认的自签名 RDP 证书。如果 Windows 侧已经配置了可信证书，优先不要忽略证书校验。

保存后回到首页，点击刚创建的连接即可进入远程桌面。

---

## 十、画面发糊的原因和优化

Guacamole 是浏览器远程桌面网关，不是原生 Windows RDP 客户端。

它的画面路径大致是：

```text
Windows RDP 画面
  -> guacd 处理
  -> Guacamole Web
  -> 浏览器显示
```

因此画质、字体渲染和流畅度通常不如 `mstsc.exe`。

常见画面发糊原因：

```text
浏览器缩放不是 100%
Guacamole 远程桌面分辨率与浏览器窗口不一致
颜色深度较低
字体平滑未启用
浏览器对画面进行了缩放
```

建议调整：

```text
浏览器按 Ctrl + 0 恢复 100% 缩放
尽量全屏访问
Display 宽度设置 1920
Display 高度设置 1080
DPI 设置 96
Color depth 设置 32-bit
开启字体平滑
开启桌面组合
```

如果外网带宽有限，可以适当降低分辨率或关闭壁纸、动画等效果。

需要接受一个现实：

```text
清晰度 / 流畅度：
原生 mstsc > Microsoft Remote Desktop 客户端 > Guacamole 浏览器 RDP
```

Guacamole 的优势是安全入口和浏览器访问，而不是极限画质。

---

## 十一、安全性分析

现在的方案比直接端口转发 RDP 安全很多。

直接转发 RDP 时：

```text
外网
  -> 公网IPv6:13389
  -> Windows:3389
```

攻击者可以直接接触 Windows RDP 服务。

攻击面是：

```text
Windows RDP 服务
Windows 登录认证
RDP 漏洞
爆破攻击
```

使用 Guacamole 后：

```text
外网
  -> HTTPS 443
  -> Guacamole 登录
  -> guacd
  -> Windows:3389
```

攻击者无法直接访问 Windows 3389。  
Windows RDP 只在内网中被 Guacamole 访问。

安全提升来自三点：

第一，公网不再暴露 RDP 端口。

```text
公网看不到 Windows:3389
```

第二，增加了一层 Guacamole Web 登录认证。

```text
必须先登录 Guacamole
才能看到和使用远程桌面连接
```

第三，所有外部访问通过 HTTPS 进行。

```text
浏览器 -> 群晖反代 -> Guacamole
```

这比裸 RDP 转发更合理。

但这不代表绝对安全。  
现在公网暴露的是 Guacamole 登录入口，因此风险变成：

```text
Guacamole 弱密码
Guacamole 漏洞
默认账号未删除
容器长期不更新
群晖反代配置错误
8080 被直接暴露
```

所以安全加固仍然很重要。

---

## 十二、安全加固建议

建议至少做到以下几点。

### 1. 不开放 3389、13389、8080

公网只开放：

```text
443
```

不要开放：

```text
3389
13389
8080
5000
5001
```

尤其是 8080。  
如果外部能访问：

```text
http://<你的域名>:8080/guacamole
```

说明绕过了 HTTPS 反向代理。

compose 中建议使用：

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

---

### 2. 删除默认账号

不要长期保留：

```text
guacadmin
```

建议新建管理员账号后删除默认账号。

---

### 3. 使用强密码

Guacamole 密码不要和 Windows 密码一样。

建议：

```text
长度 12 位以上
大小写字母 + 数字 + 符号
```

---

### 4. Windows 开启 NLA

Windows 远程桌面建议启用：

```text
仅允许运行使用网络级别身份验证的远程桌面的计算机连接
```

Guacamole 连接中安全模式选择：

```text
NLA
```

---

### 5. 定期更新容器镜像

Guacamole、guacd、PostgreSQL 都应定期更新。

如果使用 Container Manager，可以定期拉取新镜像并重建项目。

---

### 6. 可选：启用二次认证或上游访问控制

如果愿意继续加固，可以考虑 Guacamole 的 TOTP、LDAP、OIDC 等认证扩展，或者在群晖反向代理前增加额外的访问控制。

这样即使 Guacamole 密码泄露，攻击者也不一定能直接进入远程桌面入口。

---

### 7. 可选：限制来源 IP

如果访问来源比较固定，可以在群晖防火墙中限制：

```text
只允许指定公网 IP / IPv6 段访问 443
```

如果来源不固定，这项可以暂时不做。

---

## 十三、部署后检查清单

部署完成后，建议按下面几项确认：

```text
1. 外网可以通过 https://rdp.example.com/guacamole 打开 Guacamole
2. 外网无法访问 http://rdp.example.com:8080/guacamole
3. 外网无法访问 rdp.example.com:3389 或 rdp.example.com:13389
4. Guacamole 中可以成功连接内网 Windows
5. 默认 guacadmin 已删除或禁用
6. Windows 远程桌面已启用 NLA
7. 群晖防火墙和路由器只放行必要端口
```

其中第 2、3 点很重要。Guacamole 的价值就在于不让公网直接碰到 RDP，也不让外部绕过 HTTPS 反向代理访问容器端口。

---

## 十四、常见问题

### 1. 打开域名后 502 或无法访问

优先检查群晖反向代理的目的地是否填成：

```text
协议：HTTP
主机名：localhost
端口：8080
```

同时确认 Guacamole 容器已经正常启动，并且本机可以访问：

```text
http://127.0.0.1:8080/guacamole
```

### 2. Guacamole 可以打开，但连接 Windows 失败

常见原因有：

```text
Windows 没有开启远程桌面
Windows 防火墙不允许内网访问 3389
Guacamole 中 Windows 主机名或 IP 填错
用户名格式不正确
Windows 账户没有远程登录权限
NLA 或证书设置不兼容
```

可以先在同一局域网内用另一台电脑的 `mstsc.exe` 连接 Windows，确认 Windows RDP 本身是通的，再排查 Guacamole。

### 3. PostgreSQL 初始化失败

如果反复重建容器后仍然无法登录，可能是数据库目录已经生成过但初始化 SQL 没有正确执行。

这种情况下要先备份现有目录，再确认：

```text
/volume1/docker/guacamole/init/initdb.sql
/volume1/docker/guacamole/postgres
```

是否存在权限问题或残留的半初始化数据。不要在有重要数据时直接删除 PostgreSQL 目录。

---

## 十五、总结

这次方案的核心变化是：

```text
从“公网暴露 RDP”
变成“公网暴露 HTTPS Web 网关”
```

原始方案：

```text
外网 -> 域名:13389 -> Windows:3389
```

虽然简单，但本质是裸 RDP 端口转发。

最终方案：

```text
外网浏览器
  -> https://rdp.example.com/guacamole
  -> 群晖反向代理
  -> Guacamole
  -> guacd
  -> 内网 Windows:3389
```

安全性明显更好，因为公网无法直接访问 Windows 远程桌面端口。

不过 Guacamole 不是 VPN，也不是绝对安全的魔法盾。它只是把攻击面从 Windows RDP 转移到了一个 HTTPS Web 网关上。

所以最终推荐配置是：

```text
1. 群晖只开放 443
2. Guacamole 8080 只监听 127.0.0.1
3. 不开放 Windows 3389
4. Guacamole 使用强密码
5. 删除默认 guacadmin
6. Windows 开启 NLA
7. 定期更新容器
8. 有条件时启用二次认证或访问控制
```

这样，在群晖 + 公网 IPv6 + 域名证书的环境下，就可以比较安全、方便地通过浏览器访问内网 Windows 远程桌面。
