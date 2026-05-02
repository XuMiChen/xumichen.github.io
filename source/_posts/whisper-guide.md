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
description: 记录在 WSL2 中使用 ffmpeg 和 faster-whisper 搭建本地语音识别流程，将视频转成中文字幕 SRT 文件
---

最近想把一些视频资料自动生成字幕，优先考虑的是：

- 不上传音频，完全本地处理
- 能在 WSL2 里跑，方便接入现有 Linux 工具链
- 有 NVIDIA GPU 时尽量使用 GPU 加速
- 输出标准 SRT 字幕，后续可以直接导入播放器或剪辑软件

最终方案是：

```
MP4 视频
  ↓ ffmpeg 提取音频
WAV 音频
  ↓ faster-whisper 识别
SRT 中文字幕
```

---

## 1. 环境准备

本文环境以 WSL2 + Ubuntu 为例，Windows 侧需要先安装 NVIDIA 驱动，并确保 WSL 能识别 GPU。

在 WSL 中检查：

```bash
nvidia-smi
```

如果能看到显卡信息，说明 WSL 已经可以调用 GPU。

安装基础工具：

```bash
sudo apt update
sudo apt install -y python3-pip python3-venv ffmpeg aria2
```

创建 Python 虚拟环境：

```bash
python3 -m venv whisper-env
source whisper-env/bin/activate
```

安装 faster-whisper：

```bash
pip install faster-whisper
```

如果要使用 GPU，可以继续安装 CUDA 相关运行库：

```bash
pip install nvidia-cublas-cu12 nvidia-cudnn-cu12
```

---

## 2. 选择模型

faster-whisper 支持多个 Whisper 模型，常见选择如下：

| 模型 | 速度 | 准确率 | 适合场景 |
|------|------|--------|----------|
| tiny | 很快 | 较低 | 快速测试 |
| base | 快 | 一般 | 简单字幕 |
| small | 中等 | 较好 | 日常视频 |
| medium | 较慢 | 更好 | 中文字幕推荐 |
| large-v3 | 慢 | 最好 | 对准确率要求高 |

如果显存足够，中文视频可以优先从 `medium` 开始尝试。速度不够时再降到 `small`。

---

## 3. 下载模型

可以直接让程序自动下载模型，也可以提前下载到本地目录。为了避免重复下载，我更倾向于单独放到 `~/models`。

示例：下载 `medium` 模型到本地。

```bash
mkdir -p ~/models/faster-whisper-medium
cd ~/models/faster-whisper-medium

aria2c -x 16 -s 16 -c -o model.bin \
  https://hf-mirror.com/Systran/faster-whisper-medium/resolve/main/model.bin
```

如果只下载 `model.bin` 还不够，需要补齐模型目录里的配置文件。更简单的方式是使用 Git LFS 或者让 `faster-whisper` 第一次运行时自动拉取：

```python
WhisperModel("medium", device="cuda", compute_type="float16")
```

后续缓存命中后，就不需要重复下载。

---

## 4. 提取音频

先使用 `ffmpeg` 从视频中提取单声道 16k WAV 音频：

```bash
ffmpeg -i input.mp4 -vn -ac 1 -ar 16000 -y input.wav
```

参数说明：

| 参数 | 作用 |
|------|------|
| `-vn` | 不处理视频流 |
| `-ac 1` | 转成单声道 |
| `-ar 16000` | 采样率转成 16k |
| `-y` | 覆盖已有输出文件 |

Whisper 可以直接处理视频或音频文件，但先显式提取音频更容易排查问题。

---

## 5. 生成 SRT 字幕

新建脚本 `transcribe.py`：

```python
from pathlib import Path
from faster_whisper import WhisperModel


def format_timestamp(seconds: float) -> str:
    milliseconds = round(seconds * 1000)
    hours = milliseconds // 3_600_000
    milliseconds %= 3_600_000
    minutes = milliseconds // 60_000
    milliseconds %= 60_000
    secs = milliseconds // 1000
    milliseconds %= 1000
    return f"{hours:02d}:{minutes:02d}:{secs:02d},{milliseconds:03d}"


def write_srt(segments, output_path: Path) -> None:
    with output_path.open("w", encoding="utf-8") as f:
        for index, segment in enumerate(segments, start=1):
            start = format_timestamp(segment.start)
            end = format_timestamp(segment.end)
            text = segment.text.strip()
            f.write(f"{index}\n{start} --> {end}\n{text}\n\n")


audio_path = Path("input.wav")
output_path = Path("input.srt")

model = WhisperModel(
    "medium",
    device="cuda",
    compute_type="float16",
)

segments, info = model.transcribe(
    str(audio_path),
    language="zh",
    vad_filter=True,
    beam_size=5,
)

print(f"detected language: {info.language}, probability: {info.language_probability:.2f}")
write_srt(segments, output_path)
print(f"subtitle saved to: {output_path}")
```

运行：

```bash
python transcribe.py
```

生成结果：

```text
input.srt
```

---

## 6. CPU 版本

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

## 7. 常见问题

### 找不到 CUDA 或 cuDNN

如果安装了 CUDA 相关 Python 包后仍然报动态库找不到，可以检查虚拟环境中的库路径：

```bash
python -c "import nvidia.cublas, nvidia.cudnn; print(nvidia.cublas.__path__); print(nvidia.cudnn.__path__)"
```

必要时把相关 `lib` 目录加入 `LD_LIBRARY_PATH`。

### 字幕断句不理想

可以尝试调整：

```python
vad_filter=True
beam_size=5
```

如果音频中停顿不明显，自动断句可能仍然需要人工微调。

### 识别语言错误

中文视频建议显式指定：

```python
language="zh"
```

这样可以减少自动语言检测带来的不稳定。

---

## 8. 完整流程

最终常用命令可以压缩成两步：

```bash
ffmpeg -i input.mp4 -vn -ac 1 -ar 16000 -y input.wav
python transcribe.py
```

得到的 `input.srt` 可以直接和原视频放在同一目录下，大多数播放器会自动加载同名字幕。

---

## 总结

这套方案的优点是链路简单、数据不出本地，并且可以充分利用 WSL2 里的 Linux 工具和 NVIDIA GPU。  
对于个人视频整理、课程字幕、会议录音转写这类场景，`ffmpeg + faster-whisper + SRT` 已经足够实用。
