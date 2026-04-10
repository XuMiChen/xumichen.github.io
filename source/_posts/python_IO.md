---
title: Python IO 编程入门：文件读写、JSON、StringIO 与 pathlib 一次讲清
date: 2026-04-10 16:13:00
tags:
  - Python
  - IO
categories:
  - 技术
---

Python 初学者在学语法时，最容易忽略的一块其实是 IO 编程。

所谓 IO，简单说就是“程序和外部世界交换数据”的过程，比如：

- 读取配置文件
- 写入日志
- 处理 JSON
- 操作 CSV
- 创建目录
- 读写文本文件

这类能力在真实开发里几乎天天都会用到。相比只会写变量、循环和函数，真正能把程序写成一个“小工具”，往往靠的就是 IO 基础。

这篇文章就围绕 Python 中最常见的几个 IO 主题展开：

- `open()` 和文件对象
- `json.load / loads / dump / dumps`
- `StringIO` 的作用
- `pathlib.Path` 为什么值得学
- `Path` 里最常用的方法和属性
- 文件读写时几个很实用的工程习惯

## 1. 什么是 IO 编程

IO 是 `Input / Output` 的缩写，也就是“输入 / 输出”。

对 Python 来说，常见的 IO 包括：

- 从磁盘读取文件内容
- 把内容写回文件
- 从网络读取数据
- 把内容输出到终端

如果把程序想象成一个房间，那么 IO 就是这个房间的门和窗。程序只有通过 IO，才能接触外部的数据。

在初学阶段，最常见、也最值得优先掌握的是文件 IO。

---

## 2. `open()` 到底做了什么

Python 里最常见的文件操作入口就是 `open()`。

例如：

```python
with open("settings.json", "r", encoding="utf-8") as f:
    text = f.read()
```

这里的意思是：

- 打开 `settings.json`
- 以只读模式 `r` 读取
- 用 `utf-8` 编码解释文件内容
- 返回一个文件对象 `f`

### 2.1 文件对象是什么

`open()` 返回的不是字符串，而是一个“文件对象”。

这个对象提供了很多方法，例如：

- `read()`：读取内容
- `write()`：写入内容
- `readline()`：读取一行
- `readlines()`：读取多行

所以这段代码里：

```python
with open("a.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

真正的过程是：

1. `open()` 打开文件
2. 得到文件对象 `f`
3. `f.read()` 从文件对象中读取内容

### 2.2 为什么推荐配合 `with`

因为 `with` 会在代码块结束后自动关闭文件。

这比手动写：

```python
f = open("a.txt", "r", encoding="utf-8")
content = f.read()
f.close()
```

更安全，也更不容易忘记关闭文件。

---

## 3. `json.load()`、`json.loads()`、`json.dump()`、`json.dumps()` 的区别

这是初学者最容易混淆的一组方法，但规律其实非常整齐。

先记两条：

- `load / loads`：读取 JSON
- `dump / dumps`：写出 JSON

再记一条：

- 没有 `s`：处理文件
- 有 `s`：处理字符串 `string`

### 3.1 一张表记住

| 方法 | 作用 | 输入 | 输出 |
| --- | --- | --- | --- |
| `json.load()` | 从 JSON 读取为 Python 对象 | 文件对象 | Python 对象 |
| `json.loads()` | 从 JSON 字符串读取为 Python 对象 | 字符串 | Python 对象 |
| `json.dump()` | 把 Python 对象写成 JSON | Python 对象 + 文件对象 | 写入文件 |
| `json.dumps()` | 把 Python 对象变成 JSON 字符串 | Python 对象 | 字符串 |

### 3.2 最常见的例子

```python
import json

with open("settings.json", "r", encoding="utf-8") as f:
    data = json.load(f)
```

这里传给 `json.load()` 的是文件对象 `f`，不是字符串。

而下面这个才是处理字符串：

```python
import json

text = '{"name": "Alice", "age": 20}'
data = json.loads(text)
```

### 3.3 写回文件和转成字符串

```python
import json

data = {"name": "Alice", "age": 20}

with open("out.json", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)

text = json.dumps(data, ensure_ascii=False, indent=2)
print(text)
```

### 3.4 一句话记忆

- `load`：从文件读
- `loads`：从字符串读
- `dump`：写到文件
- `dumps`：变成字符串

---

## 4. `StringIO` 是什么

`StringIO` 可以理解成“内存里的文本文件”。

它来自标准库 `io`：

```python
from io import StringIO
```

很多标准库或第三方库并不要求你必须传“真实文件”，它们只要求你传一个“像文件一样的对象”。这时候 `StringIO` 就很有用。

### 4.1 一个最小例子

```python
from io import StringIO

s = StringIO()
s.write("hello\n")
s.write("world\n")

print(s.getvalue())
```

输出：

```text
hello
world
```

这里没有创建任何磁盘文件，但你依然可以像操作文件一样去 `write()`。

### 4.2 它和普通字符串的区别

普通字符串只是数据：

```python
text = "hello"
```

而 `StringIO` 是“像文件一样的对象”：

```python
s = StringIO()
s.write("hello")
```

这意味着：

- 字符串适合存结果
- `StringIO` 适合模拟文件接口

### 4.3 什么时候适合用 `StringIO`

常见场景有：

- 某个库要求传文件对象
- 想先在内存中拼接一整段文本
- 暂时不想写磁盘文件
- 想把生成过程和最终写文件过程分开

---

## 5. `pathlib` 是什么，为什么现在更推荐它

`pathlib` 是 Python 标准库里专门处理路径的模块，`Path` 是其中最常用的类。

例如：

```python
from pathlib import Path

p = Path("data") / "settings.json"
```

这行代码的意思是：构造一个路径对象，表示 `data/settings.json`。

### 5.1 为什么不用纯字符串拼路径

传统写法通常像这样：

```python
import os

path = os.path.join("data", "settings.json")
```

而 `pathlib` 写法更直观：

```python
from pathlib import Path

path = Path("data") / "settings.json"
```

好处是：

- 更像对象操作
- 可读性更好
- 路径相关的方法都集中在 `Path` 对象上

### 5.2 `Path` 负责什么，`open()` 负责什么

可以这样理解：

- `Path`：表示路径，负责路径计算和文件路径相关操作
- `open()`：负责打开文件

所以它们经常一起出现，但不是互相替代关系。

---

## 6. `Path.open()` 和直接 `open()` 有什么区别

下面两种写法都可以：

```python
from pathlib import Path

p = Path("settings.json")
with p.open("r", encoding="utf-8") as f:
    text = f.read()
```

```python
from pathlib import Path

p = Path("settings.json")
with open(p, "r", encoding="utf-8") as f:
    text = f.read()
```

### 6.1 本质区别不大

这两种写法最终都是打开同一个文件。

### 6.2 主要区别在代码风格

- `open(...)` 是内置函数
- `Path.open(...)` 是路径对象自己的方法

如果整个程序都在用 `Path`，通常继续用 `p.open()` 会更统一。

### 6.3 初学者怎么选

都可以。

但如果你已经开始用 `pathlib`，建议逐渐习惯下面这种风格：

```python
path = Path("a.txt")
with path.open("r", encoding="utf-8") as f:
    ...
```

---

## 7. `Path.mkdir()` 和 `os.mkdir()` 的区别

### 7.1 `Path.mkdir()`

```python
from pathlib import Path

Path("a/b/c").mkdir(parents=True, exist_ok=True)
```

这句的意思是：

- 创建目录 `a/b/c`
- 如果父目录不存在，就一起创建
- 如果目录已存在，也不要报错

### 7.2 `os.mkdir()`

```python
import os

os.mkdir("test")
```

它更传统，通常只创建单层目录。

如果父目录不存在，会失败；如果目录已存在，也会报错。

### 7.3 更准确的对比对象其实是 `os.makedirs()`

下面这两句更接近：

```python
os.makedirs("a/b/c", exist_ok=True)
```

```python
Path("a/b/c").mkdir(parents=True, exist_ok=True)
```

### 7.4 一句话记忆

- `os.mkdir()`：创建单层目录
- `os.makedirs()`：创建多层目录
- `Path.mkdir()`：`pathlib` 风格，可通过参数实现单层或多层创建

---

## 8. `path.mkdir(parents=True, exist_ok=True)` 详细拆解

这是实际开发里非常常见的一句：

```python
path.mkdir(parents=True, exist_ok=True)
```

它可以拆成四部分：

- `path`：一个 `Path` 对象
- `.mkdir()`：创建目录
- `parents=True`：父目录不存在时自动创建
- `exist_ok=True`：目录已存在时不报错

### 8.1 为什么它实用

因为很多时候你并不确定目录是否已经存在，也不想手写很多判断。

例如：

```python
from pathlib import Path

log_dir = Path("logs")
log_dir.mkdir(parents=True, exist_ok=True)
```

这句通常可以放心执行很多次，不需要担心重复创建报错。

---

## 9. `suffix`、`stem`、`with_suffix()` 的区别

这几个属性和方法经常一起出现，因为它们都和文件名后缀有关。

先看一个例子：

```python
from pathlib import Path

path = Path("rooms.csv")
```

### 9.1 `path.suffix`

返回最后一个后缀，包含点号：

```python
path.suffix
# '.csv'
```

### 9.2 `path.stem`

返回去掉最后一个后缀后的文件名：

```python
path.stem
# 'rooms'
```

### 9.3 `path.with_suffix()`

返回一个替换了后缀的新路径对象：

```python
path.with_suffix(".txt")
# Path('rooms.txt')
```

注意，它不会修改原来的 `path`，而是返回新的路径。

### 9.4 一个很实用的例子

```python
from pathlib import Path

path = Path("rooms.csv")
tmp = path.with_suffix(path.suffix + ".tmp")

print(tmp)
# rooms.csv.tmp
```

这个技巧很适合生成临时文件名。

### 9.5 一句话记忆

- `suffix`：看后缀
- `stem`：看主文件名
- `with_suffix()`：换后缀

---

## 10. `name / stem / suffix / suffixes / parent` 一起对比

还是用一个例子：

```python
from pathlib import Path

p = Path("/home/user/archive.tar.gz")
```

| 属性 | 结果 | 含义 |
| --- | --- | --- |
| `p.name` | `'archive.tar.gz'` | 完整文件名 |
| `p.stem` | `'archive.tar'` | 去掉最后一个后缀后的文件名 |
| `p.suffix` | `'.gz'` | 最后一个后缀 |
| `p.suffixes` | `['.tar', '.gz']` | 所有后缀 |
| `p.parent` | `Path('/home/user')` | 父目录 |

### 10.1 最容易混淆的点

- `name` 是完整文件名
- `stem` 只去掉最后一个后缀
- `suffix` 只看最后一个后缀
- `suffixes` 会返回所有后缀

例如：

```python
Path("archive.tar.gz").stem
# 'archive.tar'
```

不是 `'archive'`，因为它只去掉最后一个 `.gz`。

---

## 11. `Path` 初学者高频方法清单

下面这些是最推荐优先掌握的。

| 方法/属性 | 作用 | 返回值 |
| --- | --- | --- |
| `exists()` | 路径是否存在 | `bool` |
| `is_file()` | 是否是文件 | `bool` |
| `is_dir()` | 是否是目录 | `bool` |
| `open()` | 打开文件 | 文件对象 |
| `read_text()` | 读取文本文件 | `str` |
| `write_text()` | 写入文本文件 | 写入字符数 |
| `mkdir()` | 创建目录 | `None` |
| `unlink()` | 删除文件 | `None` |
| `rename()` | 改名或移动 | 路径变化结果 |
| `with_name()` | 替换文件名 | 新 `Path` |
| `with_suffix()` | 替换后缀名 | 新 `Path` |
| `resolve()` | 转为绝对路径 | 新 `Path` |
| `parent` | 父目录 | `Path` |
| `name` | 文件名 | `str` |
| `stem` | 主文件名 | `str` |
| `suffix` | 最后一个后缀 | `str` |

### 11.1 `exists()`

```python
from pathlib import Path

if Path("settings.json").exists():
    print("文件存在")
```

### 11.2 `read_text()`

```python
from pathlib import Path

text = Path("a.txt").read_text(encoding="utf-8")
```

适合快速读取小文本文件。

### 11.3 `write_text()`

```python
from pathlib import Path

Path("a.txt").write_text("hello", encoding="utf-8")
```

### 11.4 `with_name()`

```python
from pathlib import Path

p = Path("/tmp/rooms.csv")
print(p.with_name("settings.json"))
# /tmp/settings.json
```

### 11.5 `resolve()`

```python
from pathlib import Path

print(Path("a.txt").resolve())
```

它会返回规范化后的绝对路径。

---

## 12. 文件读写里几个很实用的习惯

学会 API 很重要，但很多时候真正拉开差距的是习惯。

### 12.1 尽量显式指定编码

例如：

```python
with open("a.txt", "r", encoding="utf-8") as f:
    text = f.read()
```

这样能减少不同系统上的乱码问题。

### 12.2 优先使用 `with`

这样可以自动关闭文件，减少资源泄露。

### 12.3 小文本可以直接用 `read_text()` / `write_text()`

如果你只是读写一个不大的文本文件，这种写法通常更简洁：

```python
from pathlib import Path

config_text = Path("settings.json").read_text(encoding="utf-8")
```

### 12.4 需要兼容库接口时考虑 `StringIO`

有些库要求“文件对象”，但你又不想真的写磁盘文件，这时 `StringIO` 非常好用。

### 12.5 修改重要文件时，优先考虑“先写临时文件，再替换”

这是一种很常见的稳妥写法，能减少文件被写坏的风险。

---

## 13. 一份适合初学者的记忆清单

如果你现在刚学到这里，先记住下面这些就很够用了：

- `open()` 返回的是文件对象，不是字符串
- `json.load()` 读文件，`json.loads()` 读字符串
- `json.dump()` 写文件，`json.dumps()` 变字符串
- `StringIO` 是“内存里的文本文件”
- `Path` 是路径对象，适合替代很多字符串路径操作
- `Path.open()` 和 `open(path)` 本质接近，主要区别是风格
- `mkdir(parents=True, exist_ok=True)` 是非常常用的安全建目录方式
- `suffix` 看后缀，`stem` 看主名，`with_suffix()` 改后缀

---

## 14. 总结

对 Python 初学者来说，IO 编程是很值得尽早打牢的一部分。

因为一旦你掌握了这些基础能力，你写的程序就不再只是“控制台里跑几行逻辑”，而是开始具备真正处理文件和数据的能力。

这篇文章如果只留下一条主线，那就是：

先把“文件对象、JSON、StringIO、Path”这几个概念搞清楚，再去学更复杂的网络编程、日志系统、配置管理、多线程，理解会顺很多。

很多看起来“像工程”的代码，本质上只是把这些基础能力组合在一起而已。
