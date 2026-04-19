# 小智 Linux — 全志T113S3 · LVGL8 版本

> 基于全志 T113S3 的嵌入式 AI 语音交互机器人  
> 利用 Qwen、DeepSeek 等大模型实现端云协同智能对话，LVGL8 驱动屏幕交互界面

---

## 📖 项目简介

小智 Linux 是一款运行在 ARM Linux 设备上的 AI 语音交互入口，基于桌面智慧屏项目移植适配。  
系统完成从音频采集 → 压缩编码 → 网络传输 → 云端 AI 推理 → 解码播放 → 屏幕显示的全链路闭环，  
为嵌入式智能终端提供低延时、高可用的本地化 AI 交互能力。

---

## ✨ 功能特性

| 模块 | 说明 |
|------|------|
| 🎙 音频采集 | 基于 ALSA 驱动麦克风实时采集 PCM 音频 |
| 🗜 压缩编码 | Opus 编解码，降低网络带宽占用 |
| 📡 网络传输 | WebSocket 实现端云双向流式通信 |
| 🤖 AI 推理 | 接入 Qwen / DeepSeek 等大模型完成 ASR → LLM → TTS |
| 🔊 解码播放 | 接收 TTS 音频流，ALSA 异步解码播放，支持语音打断 |
| 🖥 屏幕显示 | LVGL8 驱动 MIPI 屏，实时展示对话状态与内容 |

---

## 🏗 系统架构

```
┌─────────────────────────────────────────────────────┐
│                  全志 T113S3 (ARM Cortex-A7)         │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌───────────────┐  │
│  │ 音频采集  │───▶│ Opus编码 │───▶│  WebSocket发送 │  │
│  │  ALSA    │    │          │    │               │  │
│  └──────────┘    └──────────┘    └──────┬────────┘  │
│                                         │           │
│  ┌──────────┐    ┌──────────┐    ┌──────▼────────┐  │
│  │ ALSA播放  │◀───│ Opus解码 │◀───│  WebSocket接收 │  │
│  │ 异步播放器 │    │          │    │               │  │
│  └──────────┘    └──────────┘    └───────────────┘  │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │         LVGL8 UI 状态显示 (MIPI屏)            │   │
│  │   待机 → 聆听 → 思考 → 回复 → 打断           │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                          │  WebSocket / HTTPS
                          ▼
             ┌────────────────────────┐
             │        云端服务         │
             │  ASR → LLM → TTS       │
             │  Qwen / DeepSeek       │
             └────────────────────────┘
```

---

## 🔄 状态机流程

```
        ┌─────────┐
        │  IDLE   │◀─────────────────────┐
        └────┬────┘                      │
             │ 检测到唤醒词 / 按键          │
             ▼                           │
        ┌─────────┐                      │
        │LISTENING│ ── 采集音频 + 编码     │
        └────┬────┘                      │
             │ VAD检测到静音              │
             ▼                           │
        ┌─────────┐                      │
        │THINKING │ ── 等待云端响应        │
        └────┬────┘                      │
             │ 收到TTS流                  │
             ▼                           │
        ┌─────────┐   用户打断            │
        │ PLAYING │──────────────────────┘
        └─────────┘  打断延迟 < 100ms
```

---

## 🛠 技术栈

| 类别 | 技术 |
|------|------|
| **主控平台** | 全志 T113S3 · ARM Cortex-A7 · Linux 5.4 |
| **编程语言** | C / C++14 |
| **音频** | ALSA · Opus 编解码 |
| **网络** | WebSocket · HTTP · JSON 解析 |
| **并发** | 多线程 · 有限状态机 · 事件队列 |
| **UI框架** | LVGL 8.x · MIPI 屏驱动 |
| **构建工具** | CMake · 交叉编译工具链 |
| **外设** | I2C · GPIO · UART · SPI |
| **AI服务** | Qwen · DeepSeek · 标准 WebSocket 协议接入 |

---
## ⚙ 快速开始

### 环境依赖

```bash
# Ubuntu 编译主机
sudo apt install cmake build-essential
sudo apt install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf
sudo apt install libopus-dev  # 本地调试用
```

### 本地编译（Ubuntu 调试）

```bash
git clone https://github.com/yourname/xiaozhi-linux-t113.git
cd xiaozhi-linux-t113
mkdir build && cd build
cmake ..
make -j4
```

### 交叉编译（T113S3目标板）

```bash
mkdir build-t113 && cd build-t113
cmake .. -DCMAKE_TOOLCHAIN_FILE=../toolchain-t113.cmake
make -j4
```

### 部署到开发板

```bash
# 拷贝可执行文件
scp xiaozhi root@192.168.1.x:/usr/local/bin/

# 拷贝配置文件
scp config/config.json root@192.168.1.x:/etc/xiaozhi/

# 板子上运行
ssh root@192.168.1.x
export QT_QPA_PLATFORM=linuxfb  # 如使用Qt，否则LVGL直接操作framebuffer
/usr/local/bin/xiaozhi
```

### 开机自启（systemd）

```ini
# /etc/systemd/system/xiaozhi.service
[Unit]
Description=XiaoZhi AI Voice Robot
After=network.target

[Service]
ExecStart=/usr/local/bin/xiaozhi
Restart=always
RestartSec=3
Environment="DISPLAY=:0"

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable xiaozhi
systemctl start xiaozhi
```

---

## 🔧 配置说明

编辑 `config/config.json`：

```json
{
  "server": {
    "websocket_url": "wss://your-server.com/ws",
    "api_key": "your_api_key_here"
  },
  "audio": {
    "capture_device": "hw:0,0",
    "playback_device": "hw:0,0",
    "sample_rate": 16000,
    "channels": 1,
    "opus_bitrate": 24000
  },
  "ai": {
    "model": "qwen-turbo",
    "language": "zh-CN",
    "vad_silence_ms": 800
  },
  "ui": {
    "screen_width": 800,
    "screen_height": 480,
    "brightness": 80
  }
}
```

---

## 🖥 UI 界面说明

| 状态 | 显示内容 |
|------|---------|
| 待机 IDLE | 时钟 + 天气 + 待机动画 |
| 聆听 LISTENING | 麦克风波形动画 + "我在听..." |
| 思考 THINKING | 旋转加载动画 + "思考中..." |
| 回复 PLAYING | 对话气泡 + TTS 播放进度 |
| 错误 ERROR | 错误提示 + 自动重试倒计时 |

---

## 📌 开发注意事项

- MIPI 屏驱动需修改设备树节点，注意初始化时序问题
- LVGL8 多页面切换使用 `lv_obj_clean()` 统一清理，避免内存泄漏
- 音频播放与 UI 刷新在独立线程中运行，通过事件队列通信
- WebSocket 断线后自动重连，重连间隔指数退避
- 语音打断通过检测音频能量阈值触发，打断延迟目标 < 100ms

---

## 📄 License

MIT License © 2025 陈梅庆

---

## 🙏 致谢

- [小智AI](https://github.com/78/xiaozhi-esp32) — 原始项目灵感来源
- [LVGL](https://lvgl.io) — 嵌入式 GUI 框架
- [Opus](https://opus-codec.org) — 音频编解码库
- 全志 T113S3 SDK 团队
