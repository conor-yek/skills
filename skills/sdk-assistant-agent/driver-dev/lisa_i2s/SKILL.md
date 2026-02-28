---
name: lisa_i2s
description: lisa_i2s 驱动使用与移植辅助。涵盖 I2S Master 发送/接收、Slave 模式、DMA 音频流、采样率/位宽配置。
---

# lisa_i2s 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_i2s/lisa_i2s.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_i2s/README.html`
- **示例目录**: `samples/drivers/devices/lisa_i2s/`（master_tx / master_rx / slave_rx_tx 等）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_i2s/lisa_i2s.h
   → API 函数签名、lisa_i2s_config_t 结构体（采样率/位宽/声道/格式）、回调原型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_i2s/README.html
   → prj.conf 配置（CONFIG_LISA_I2S_DEVICE）、MCLK 要求、格式（标准I2S/左对齐/右对齐）

③ Glob  samples/drivers/devices/lisa_i2s/**
   → 按场景选 main.c：
     - Master 发送音频     → master_tx/main.c
     - Master 接收音频     → master_rx/main.c
     - Slave 收发双向      → slave_rx_tx/main.c
```

### Step 2: 回答规则

- I2S 配置参数（采样率、位宽、声道数、帧格式）必须与外部 Codec/DAC/ADC 完全匹配，否则音频失真
- **Master vs Slave**：Master 输出 BCLK 和 LRCLK；Slave 接收外部时钟，不能同时两个都是 Master
- DMA 音频流通常使用乒乓缓冲（ping-pong），一块在传输的同时另一块由应用填充/消费
- MCLK（主时钟）通常为采样率的 256 倍（如 16kHz → 4.096MHz），从文档确认是否需要单独配置

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_i2s/lisa_i2s.h       → vtable 接口（lisa_i2s_api_t）及配置/缓冲结构体
② Read  drivers/lisa_i2s/lisa_i2s_arcs.c  → 平台适配：DMA 链配置、时钟分频、FIFO 深度
```

### I2S 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `configure(dev, cfg)` — 设置采样率、位宽、声道、格式、Master/Slave 模式
- `read(dev, mem_block, size)` — 获取一块接收完成的 DMA 缓冲（阻塞等待）
- `write(dev, mem_block, size)` — 提交一块发送 DMA 缓冲（非阻塞提交）
- `trigger(dev, dir, cmd)` — 控制 TX/RX 流：START / STOP / DRAIN / DROP

**乒乓 DMA 设计**：
- 驱动内部通常维护 TX/RX 队列（环形 buffer pool）
- `write` 将 buffer 投入 TX 队列；`read` 从 RX 队列取出已填充的 buffer
- 移植时 DMA 中断切换 buffer 指针时需保证原子性

**时钟设计**：
- I2S 时钟链：系统PLL → MCLK 分频 → BCLK 分频 → LRCLK（BCLK / (2 × 位宽)）
- 移植时确认各分频值与目标采样率的对应关系，小数分频需特别处理

**已知注意点**：
- TX underrun（发送队列空）/ RX overrun（接收队列满）会导致音频断续，需确保应用侧及时消费/填充
- Slave 模式下需等待 Master 先输出时钟后才能启动传输，否则第一帧数据可能错位
- `DRAIN` 命令等待队列中已有的数据全部发送完毕；`DROP` 命令立即停止并丢弃队列
