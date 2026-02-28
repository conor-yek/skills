---
name: lisa_qspilcd
description: lisa_qspilcd 驱动使用与移植辅助。涵盖 QSPI 接口 LCD 驱动、基础传输模式、DMA 加速刷新、命令/数据时序。
---

# lisa_qspilcd 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_qspilcd/lisa_qspilcd.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_qspilcd/README.html`
- **示例目录**: `samples/drivers/devices/lisa_qspilcd/`（basic / dma）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_qspilcd/lisa_qspilcd.h
   → API 函数签名、命令/数据传输结构体、QSPI 线宽/模式枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_qspilcd/README.html
   → prj.conf 配置（CONFIG_LISA_QSPILCD_DEVICE）、LCD 型号配置、初始化命令序列说明

③ Glob  samples/drivers/devices/lisa_qspilcd/**
   → 按场景选 main.c：
     - CPU 直接传输像素   → basic/main.c
     - DMA 加速刷新       → dma/main.c
```

### Step 2: 回答规则

- `lisa_qspilcd` 专为 QSPI 接口（4 线 SPI）LCD 设计，比普通 SPI LCD 有更高带宽
- 像素数据发送前通常需要先发送"设置显示区域"命令（column address / row address set），再发像素
- **DMA 模式**：DMA 刷新期间 CPU 不参与数据搬运，适合大面积刷新；buffer 需在 DMA 传输期间保持有效
- 询问用户使用的 LCD 型号（ST7789/ILI9341/GC9A01 等），初始化命令序列与型号相关

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_qspilcd/lisa_qspilcd.h       → vtable 接口（lisa_qspilcd_api_t）
② Read  drivers/lisa_qspilcd/lisa_qspilcd_arcs.c  → 平台适配：QSPI 控制器配置、DMA 链路、DC 引脚控制
```

### QSPILCD 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `init(dev)` — 初始化 QSPI 控制器，发送 LCD 初始化命令序列（复位 → 配置 → 开启显示）
- `write_cmd(dev, cmd, data, len)` — 发送 LCD 命令（1字节命令 + 可选参数）
- `write_pixel(dev, x, y, w, h, buf, len)` — 写入像素数据到指定区域
- `write_pixel_dma(dev, x, y, w, h, buf, len, cb)` — DMA 方式写像素，完成后回调

**QSPI 传输模式**：
- 命令阶段：通常 1 线（SPI 单线），1 字节命令
- 数据阶段：可选 1/2/4 线，像素数据通过 4 线传输速度最快
- 移植时在 QSPI 控制器配置中设置各阶段的线宽

**LCD 初始化序列**：
- 不同 LCD IC 有不同初始化寄存器序列，通常以 `{cmd, delay_ms, data_len, data[]}` 数组表示
- 移植新 LCD 时，将 IC 提供的初始化序列转换为驱动内部格式

**已知注意点**：
- DC（Data/Command）引脚：发命令时拉低，发数据时拉高；QSPI 控制器通常可硬件自动控制，需确认
- 像素格式（RGB565/RGB666/RGB888）需与 LCD IC 配置一致，通过初始化命令中的色彩格式寄存器设置
- DMA 传输完成回调在中断上下文，不要在回调中做复杂操作
