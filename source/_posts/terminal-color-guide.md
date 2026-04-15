---
title: 从零搞懂终端配色：Fish + Windows Terminal + VS Code
date: 2026-04-15 17:28:00
tags:
  - Fish
  - VS Code
  - color
categories:
  - 技术
---

在使用 Linux / WSL 的过程中，常常会遇到一些困惑：

- 为什么我改了 Fish 的颜色，有些生效，有些却没有？
- 为什么 Windows Terminal 切换配色后，背景也变了？
- VS Code 的主题到底会不会影响终端？

这些问题本质上都和「终端配色机制」有关。  
本文从整体结构出发，逐步梳理清楚它们之间的关系。

---

# 一、整体结构：三层模型

一个典型环境（WSL + Fish + Windows Terminal）可以抽象为三层：

```
Fish（Shell）
   ↓ 输出 ANSI 颜色
Windows Terminal（终端）
   ↓ 颜色映射 & 渲染
VS Code（如果使用内置终端）
```

---

## 每一层负责什么？

| 层级 | 作用 |
|------|------|
| Fish | 决定“谁使用什么颜色” |
| Windows Terminal | 决定“颜色具体是什么样” |
| VS Code Theme |（仅内置终端）替代终端配色 |

可以理解为：

> Shell 决定逻辑，Terminal 决定外观

---

# 二、Fish 的配色机制

Fish 的颜色通过变量控制，例如：

```fish
set -U fish_color_cwd green
set -U fish_color_command c397d8
```

---

## 两种颜色类型（关键区别）

### 1. 颜色名字（会随终端变化）

```
green / red / blue / brgreen
```

这类颜色是 ANSI 颜色编号，最终显示效果由终端决定。

---

### 2. HEX 颜色（固定）

```
c397d8 / 7aa6da
```

等价于：

```
#c397d8
```

这种写法不会受终端主题影响。

---

## 小结

| 写法 | 是否受 Terminal 影响 |
|------|----------------------|
| green | 会变化 |
| #xxxxxx | 不会变化 |

---

# 三、fish_variables 文件

Fish 的持久化配置保存在：

```
~/.config/fish/fish_variables
```

内容类似：

```
SETUVAR fish_color_cwd:green
SETUVAR fish_color_command:c397d8
```

---

## `set -U` 的作用

```fish
set -U fish_color_cwd blue
```

作用是：

- 写入 `fish_variables`
- 所有终端会话生效
- 重启后仍然存在

---

## 删除变量

```fish
set -eU fish_color_cwd
```

---

# 四、为什么有些颜色改不了？

一个常见现象是：

> 修改了颜色配置，但目录或用户名颜色没有变化

原因通常是：

👉 prompt 自己定义了颜色

---

## 示例

在 `fish_prompt` 中：

```fish
(set_color $fish_color_cwd) (prompt_pwd)
```

路径颜色来自：

```
fish_color_cwd
```

---

但用户名、主机名通常来自：

```fish
(prompt_login)
```

这部分往往在 prompt 中单独控制。

---

## 控制来源总结

| 元素 | 来源 |
|------|------|
| 命令颜色 | fish_color_* |
| 路径 | fish_color_cwd |
| 用户名 / 主机名 | fish_prompt |
| `>` 符号 | prompt |

---

# 五、Windows Terminal 的配色机制

Windows Terminal 的核心是 `schemes`，本质是：

> ANSI 颜色映射表

---

## 示例

```json
{
  "green": "#98c379",
  "red": "#e06c75"
}
```

---

## 工作方式

当 Fish 输出：

```fish
set_color green
```

终端会执行：

```
green → #98c379
```

---

## 关键点

- Terminal 不决定“谁是绿色”
- 只决定“绿色长什么样”

---

# 六、为什么切换配色后背景也会变化？

因为配色方案本身就包含背景定义：

```json
"background": "#1e1e1e"
```

同时：

- 前景颜色变化
- ANSI 颜色变化

整体对比度改变，因此视觉上会明显不同。

---

# 七、VS Code Theme 的作用

VS Code 的主题主要影响：

- 编辑器语法高亮
- UI
- 内置终端颜色

---

## 区别

| 场景 | 是否影响 |
|------|----------|
| VS Code 内置终端 | 会 |
| Windows Terminal | 不会 |

---

# 八、统一三端配色的思路

推荐统一方案（如 Tokyo Night）：

- VS Code：安装并启用主题
- Windows Terminal：添加 scheme 并应用
- Fish：设置统一颜色变量

---

## 核心原则

```
Terminal：定义颜色值
Fish：定义颜色用途
VS Code：统一视觉风格
```

---

# 九、实践建议

## 推荐

- Fish 使用颜色名字（green / red）
- Terminal 控整体配色
- VS Code 使用相同风格主题

---

## 不推荐

- 混用 HEX 和 ANSI（难统一）
- 在 prompt 中写死颜色

---

# 总结

终端配色可以归纳为：

```
Fish：谁是绿色
Terminal：绿色是什么颜色
VS Code：整体视觉风格
```

理解这一点后，大多数颜色问题都可以快速定位和解决。
