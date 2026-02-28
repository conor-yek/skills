---
name: lisa_audio
description: lisa_audio 驱动使用与移植辅助。涵盖录音/播放 pipeline、AEC 回声消除对齐、Codec 抽象接口、多声道配置。
---

# lisa_audio 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_audio/lisa_audio.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_audio/README.html`
- **示例目录**: `samples/drivers/devices/lisa_audio/`（record_playback / record_echo_alignment）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_audio/lisa_audio.h
   → API 函数签名、录音/播放 pipeline 结构体、声道配置枚举、AEC 相关参数

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_audio/README.html
   → prj.conf 配置（CONFIG_LISA_AUDIO_DEVICE）、Codec 型号配置、采样率支持列表、AEC 使能选项

③ Glob  samples/drivers/devices/lisa_audio/**
   → 按场景选 main.c：
     - 录音同时播放               → record_playback/main.c
     - 录音 + AEC 回声对齐        → record_echo_alignment/main.c
```

### Step 2: 回答规则

- `lisa_audio` 是比 `lisa_i2s` 更高层的抽象，封装了 Codec 控制 + I2S 数据传输
- **AEC（回声消除）**：需要录音数据和播放数据的精确时间对齐，`record_echo_alignment` 示例展示标准方式；不要自行设计对齐逻辑
- 多声道配置（麦克风阵列通道数、播放声道数）从 `lisa_audio_config_t` 中设置，需与实际硬件匹配
- 提醒用户：Codec 型号在 prj.conf 中通过 Kconfig 选择，不同 Codec 需不同配置

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_audio/lisa_audio.h       → vtable 接口（lisa_audio_api_t）及 pipeline 结构体
② Read  drivers/lisa_audio/lisa_audio_arcs.c  → 平台适配：Codec I2C 控制序列、I2S 时钟配置、DMA 链路
```

### Audio 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `open(dev, cfg)` — 初始化 Codec 和 I2S 控制器，配置采样率/位宽/声道
- `read(dev, buf, size)` — 读取一帧录音 PCM 数据（阻塞等待 DMA 完成）
- `write(dev, buf, size)` — 写入一帧播放 PCM 数据（非阻塞提交到队列）
- `ioctl(dev, cmd, arg)` — 控制 Codec 参数（音量、静音、增益、AEC 配置）
- `close(dev)` — 停止 pipeline，关闭 Codec

**Codec 适配层**：
- `lisa_audio` 通过 I2C 控制 Codec 寄存器（音量、路由、ADC/DAC 配置）
- 移植新 Codec 时需实现 Codec 的 init / gain_set / route_set 等操作，注意寄存器时序（上电顺序）

**AEC 时间对齐**：
- 录音（MIC）和参考信号（Speaker 输出回采）需在同一时钟域同步采样
- 移植时确保录音 DMA 和播放 DMA 使用同一 I2S 时钟源，避免时钟漂移

**已知注意点**：
- 录音和播放同时运行时，`read` 和 `write` 的 buffer 大小应保持一致（通常为 10ms 帧长）
- Codec 上电到可用需要时间（通常 10-100ms），`open` 中需包含适当等待
- 采样率切换需重新初始化（`close` + `open`），不支持运行时动态切换
