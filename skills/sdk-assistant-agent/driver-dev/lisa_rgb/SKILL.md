---
name: lisa_rgb
description: lisa_rgb 驱动使用与移植辅助。涵盖 RGB LED 灯带控制、Bounce Buffer 模式、WS2812 协议时序、DMA 输出。
---

# lisa_rgb 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_rgb/lisa_rgb.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_rgb/README.html`
- **示例目录**: `samples/drivers/devices/lisa_rgb/`（rgb_bounce_buffer）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_rgb/lisa_rgb.h
   → API 函数签名、颜色数据结构体（R/G/B 字段顺序）、灯珠数量/协议类型枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_rgb/README.html
   → prj.conf 配置（CONFIG_LISA_RGB_DEVICE）、支持的 LED 协议（WS2812B/SK6812等）、Bounce Buffer 说明

③ Read  samples/drivers/devices/lisa_rgb/rgb_bounce_buffer/main.c
   → 初始化 → 填充颜色数据 → 触发 DMA 输出的完整流程
```

### Step 2: 回答规则

- `lisa_rgb` 使用 **Bounce Buffer** 模式：应用填充颜色数据 → 驱动将颜色编码为 WS2812 波形 → DMA 输出
- 颜色字节顺序（RGB/GRB/RGBW）因 LED 型号而异，从头文件或文档确认，不要假设
- 灯珠数量在初始化时配置，运行时不可更改（影响 DMA buffer 大小）
- DMA 传输期间不要修改颜色 buffer，等待上一帧传输完成后再更新

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_rgb/lisa_rgb.h       → vtable 接口（lisa_rgb_api_t）及颜色/配置结构体
② Read  drivers/lisa_rgb/lisa_rgb_arcs.c  → 平台适配：WS2812 波形编码、DMA 配置、时序精度保证
```

### RGB LED 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `init(dev, cfg)` — 初始化控制器，分配 Bounce Buffer（大小 = LED数量 × 每颗LED的字节数 × 编码倍率）
- `update(dev, pixels, count)` — 提交颜色数据，触发 DMA 输出
- `get_update_buffer(dev)` — 获取可写的颜色 buffer 地址（Bounce Buffer 机制）

**WS2812 时序要求**：
- WS2812B 协议：每个 bit 用特定高低电平时长编码（T0H/T0L/T1H/T1L ≈ 300-900ns）
- 帧间复位时间：>50μs 的低电平，LED 才锁存颜色数据
- 移植时通常用 SPI/I2S/PWM+DMA 模拟 WS2812 时序，需精确计算波特率与 bit 编码映射

**Bounce Buffer 机制**：
- 应用层填充 RGB 颜色数组（每颗 LED 3/4 字节）
- 驱动层将颜色数据展开为 WS2812 波形数据（每 bit 扩展为多个 SPI bit）
- DMA 输出展开后的波形数据
- "Bounce" 指驱动内部维护展开用的中间 buffer，应用不直接看到

**已知注意点**：
- WS2812 时序对精度要求高（±150ns），DMA 是唯一可靠方式，禁止用 CPU 循环模拟（中断会破坏时序）
- 颜色字节顺序：WS2812B 是 GRB，SK6812 是 RGB，RGBW 款多一个白通道；移植时务必查阅目标 LED 规格书
- 帧率受限于 `LED数量 × 30μs/LED`，100颗 LED 最大约 333fps（通常足够）
