---
title: 在 WSL + GPU 环境下使用 faster-whisper 本地生成中文字幕
date: 2026-05-02 10:30:00
tags:
  - WSL
  - Whisper
  - faster-whisper
  - 字幕
  - GPU
categories:
  - 技术
description: 记录在 WSL2 中使用 ffmpeg 和 faster-whisper 搭建本地语音识别流程，将视频批量转成中文字幕 SRT 文件
---

最近折腾了一套完全本地的视频字幕生成方案，核心是：

- Whisper（语音识别）
- faster-whisper（高性能推理）
- ffmpeg（音频处理）
- WSL + NVIDIA GPU 加速

这篇文章整理一下完整流程，以及一些踩坑经验。

---

## 1. 整体方案

目标很简单：

```text
MP4 → WAV → SRT 字幕
```

核心流程：

1. 使用 ffmpeg 提取音频
2. 使用 faster-whisper 识别语音
3. 输出 SRT 字幕文件

这套方案的优先级是：

- 不上传音频，完全本地处理
- 能在 WSL2 里跑，方便接入现有 Linux 工具链
- 有 NVIDIA GPU 时尽量使用 GPU 加速
- 输出标准 SRT 字幕，后续可以直接导入播放器或剪辑软件

---

## 2. 环境准备

本文环境以 WSL2 + Ubuntu 为例，Windows 侧需要先安装 NVIDIA 驱动，并确保 WSL 能识别 GPU。

### 安装基础工具

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv ffmpeg aria2
```

如果当前已经是 root 用户，也可以去掉 `sudo`：

```bash
apt update
apt install -y python3-pip python3-venv ffmpeg aria2
```

### 创建虚拟环境

```bash
python3 -m venv whisper-env
source whisper-env/bin/activate
```

### 安装依赖

```bash
pip install faster-whisper
```

GPU 需要额外安装 CUDA 相关运行库：

```bash
pip install nvidia-cublas-cu12 nvidia-cudnn-cu12
```

### 验证 GPU

```bash
nvidia-smi
```

如果能看到显卡信息，说明 WSL 已经可以调用 GPU。

---

## 3. 模型下载

faster-whisper 可以在第一次运行时自动下载模型，但国内网络经常比较慢。更稳的方式是提前把模型放到本地目录。

以 `medium` 模型为例：

```bash
mkdir -p ~/models/medium
cd ~/models/medium

aria2c -x 16 -s 16 -c -o model.bin \
  https://hf-mirror.com/Systran/faster-whisper-medium/resolve/main/model.bin

wget https://hf-mirror.com/Systran/faster-whisper-medium/resolve/main/config.json
wget https://hf-mirror.com/Systran/faster-whisper-medium/resolve/main/tokenizer.json
wget https://hf-mirror.com/Systran/faster-whisper-medium/resolve/main/vocabulary.txt
```

目录结构：

```text
~/models/medium/
├── model.bin
├── config.json
├── tokenizer.json
└── vocabulary.txt
```

也可以设置 Hugging Face 镜像地址，让程序自动下载时走镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

---

## 4. 模型选择建议

常用模型可以按下面的方式选：

| 模型 | 大小 | 特点 |
|------|------|------|
| small | ~500MB | 快，但不够准 |
| medium | ~1.5GB | 推荐，速度和准确率比较平衡 |
| large | ~3GB | 最准，但更慢 |

日常建议：

```text
默认：medium
批量：small
高质量：large
```

如果是中文视频，并且显存足够，可以优先从 `medium` 开始。速度不够时再降到 `small`。

---

## 5. 单文件处理流程

先使用 `ffmpeg` 从视频中提取单声道 16k WAV 音频：

```bash
ffmpeg -i input.mp4 -vn -map 0:a:0 -ar 16000 -ac 1 -c:a pcm_s16le -y input.wav
```

参数说明：

| 参数 | 作用 |
|------|------|
| `-vn` | 不处理视频流 |
| `-map 0:a:0` | 使用第一个音频轨道 |
| `-ar 16000` | 采样率转成 16k |
| `-ac 1` | 转成单声道 |
| `-c:a pcm_s16le` | 输出标准 WAV PCM 音频 |
| `-y` | 覆盖已有输出文件 |

然后再把音频交给 faster-whisper 识别。实际使用时，我更推荐直接用下一节的批量脚本，省得手动维护每个文件的 WAV 和 SRT。

---

## 6. 批量转字幕脚本

下面这个脚本可以：

- 扫描当前目录中的 MP4
- 提取 WAV
- 生成 SRT
- 自动跳过已存在的字幕文件

脚本中的 `MODEL_PATH` 需要按自己的模型目录调整。如果模型放在 `~/models/medium`，并且当前用户不是 root，可以改成类似 `/home/用户名/models/medium`。

```python
from faster_whisper import WhisperModel
import subprocess
import time
from pathlib import Path

MODEL_PATH = "/root/models/medium"

model = WhisperModel(
    MODEL_PATH,
    device="cuda",
    compute_type="int8"
)

def format_time(seconds):
    ms = int(seconds * 1000)
    h = ms // 3600000
    ms %= 3600000
    m = ms // 60000
    ms %= 60000
    s = ms // 1000
    ms %= 1000
    return f"{h:02}:{m:02}:{s:02},{ms:03}"

def process(mp4):
    wav = mp4.with_suffix(".wav")
    srt = mp4.with_suffix(".srt")

    if not wav.exists():
        subprocess.run([
            "ffmpeg", "-y",
            "-i", str(mp4),
            "-vn", "-map", "0:a:0",
            "-ar", "16000",
            "-ac", "1",
            "-c:a", "pcm_s16le",
            str(wav)
        ])

    if srt.exists():
        print(f"[SKIP] {srt.name}")
        return

    start = time.time()

    segments, _ = model.transcribe(
        str(wav),
        language="zh",
        beam_size=5,
        vad_filter=False
    )

    with open(srt, "w", encoding="utf-8") as f:
        for i, seg in enumerate(segments, 1):
            f.write(f"{i}\n")
            f.write(f"{format_time(seg.start)} --> {format_time(seg.end)}\n")
            f.write(seg.text.strip() + "\n\n")

    print(f"{srt.name} 完成，用时 {int(time.time()-start)}s")

for mp4 in Path(".").glob("*.mp4"):
    process(mp4)
```

运行方式：

```bash
python transcribe.py
```

把脚本放在视频目录下运行后，会生成同名文件：

```text
video.mp4
video.wav
video.srt
```

如果只想保留字幕文件，确认 SRT 没问题后可以手动删除中间的 WAV。

---

## 7. 性能表现

以常见配置为例：

- GPU：GTX 1060
- 模型：medium
- compute_type：int8

表现大致如下：

```text
1 小时音频 → 10～20 分钟完成
```

相比 CPU 通常会有明显提升：

```text
3～8 倍
```

实际速度会受显卡、模型大小、音频长度、VAD 设置影响。

---

## 8. CPU 版本

如果当前机器没有可用 GPU，可以改成 CPU：

```python
model = WhisperModel(
    "small",
    device="cpu",
    compute_type="int8",
)
```

CPU 速度会慢一些，但对短视频、简单音频仍然够用。模型建议从 `small` 或 `base` 开始，确认流程没问题后再提高模型大小。

---

## 9. 常见问题

### GPU 显示 3D 占用高

这是正常现象，CUDA 计算在 Windows 任务管理器里可能会被归类到 3D。

判断是否真的在使用 GPU，还是看：

```bash
nvidia-smi
```

如果能看到 Python 进程和显存占用，就说明推理已经跑在 GPU 上。

### 字幕时间有漂移

常见原因：

- VAD 切分影响时间轴
- 视频本身时间戳不稳定
- 音频轨道选择不明确

可以先关闭 VAD：

```python
vad_filter=False
```

并且在提取音频时明确使用第一个音频轨道：

```bash
ffmpeg -vn -map 0:a:0 ...
```

### 模型下载慢

可以设置镜像：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

也可以像上面一样手动下载模型文件。

### 找不到 CUDA 或 cuDNN

如果安装了 CUDA 相关 Python 包后仍然报动态库找不到，可以检查虚拟环境中的库路径：

```bash
python -c "import nvidia.cublas, nvidia.cudnn; print(nvidia.cublas.__path__); print(nvidia.cudnn.__path__)"
```

必要时把相关 `lib` 目录加入 `LD_LIBRARY_PATH`。

### 识别语言错误

中文视频建议显式指定：

```python
language="zh"
```

这样可以减少自动语言检测带来的不稳定。

---

## 10. 完整流程

最终常用流程是：

```bash
source whisper-env/bin/activate
cd 视频目录
python transcribe.py
```

如果只处理单个文件，也可以拆成两步：

```bash
ffmpeg -i input.mp4 -vn -map 0:a:0 -ar 16000 -ac 1 -c:a pcm_s16le -y input.wav
python transcribe.py
```

得到的 `.srt` 可以直接和原视频放在同一目录下，大多数播放器会自动加载同名字幕。

---

## 总结

这套方案的优点是链路简单、数据不出本地，并且可以充分利用 WSL2 里的 Linux 工具和 NVIDIA GPU。

核心工具组合：

```text
ffmpeg + faster-whisper + WSL + GPU
```

适合：

- 视频字幕生成
- 直播录屏整理
- 本地语音转文本

对于个人视频整理、课程字幕、会议录音转写这类场景，`ffmpeg + faster-whisper + SRT` 已经足够实用。
