---
name: lisa_spi
description: lisa_spi 驱动使用与移植辅助。涵盖 SPI Master 传输、Master-Slave 组合、DMA 模式及 CS 管理。
---

# lisa_spi 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_spi/lisa_spi.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_spi/README.html`
- **示例目录**: `samples/drivers/devices/lisa_spi/`（master / master_slave）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_spi/lisa_spi.h
   → API 函数签名、lisa_spi_config_t / lisa_spi_buf_t / lisa_spi_buf_set_t 结构体、模式枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_spi/README.html
   → prj.conf 配置、CS 配置方式、DMA 使能选项

③ Glob  samples/drivers/devices/lisa_spi/**
   → 按场景选 main.c：
     - 单 Master 发送/接收  → master/main.c
     - Master + Slave 回环  → master_slave/main.c
```

### Step 2: 回答规则

- SPI 传输使用 `buf_set`（tx_bufs + rx_bufs）结构，注意 `count` 字段代表 buf 数量，不是字节数
- CS（片选）可由驱动托管（GPIO CS）或软件手动控制，从文档确认当前支持方式
- **模式**：CPOL / CPHA 组合需与外设 datasheet 核对，提醒用户确认

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_spi/lisa_spi.h       → vtable 接口（lisa_spi_api_t）及数据结构
② Read  drivers/lisa_spi/lisa_spi_arcs.c  → 平台适配：时钟分频计算、DMA 通道配置、CS 控制
```

### SPI 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `transceive(dev, config, tx_bufs, rx_bufs)` — 同步全双工传输
- `transceive_async(dev, config, tx_bufs, rx_bufs, cb)` — 异步（DMA）传输 + 完成回调
- `release(dev, config)` — 释放 CS（锁定 CS 模式下使用）

**数据缓冲设计**：
- `lisa_spi_buf_set_t` 支持 scatter-gather（多段 buffer 串联传输），适合发送 header + payload 分离场景
- tx/rx buf 数量可不等（只发不收：rx_bufs 传 NULL 或 count=0）

**DMA 模式**：
- 异步传输回调在中断上下文执行，避免在回调中做耗时操作
- DMA buffer 需 cache 对齐（若 SoC 有 cache），移植时确认 DMA 对齐要求

**已知注意点**：
- `lisa_spi_config_t` 中的 `frequency` 是期望频率，实际频率受时钟分频限制，驱动内取最近可达值
- 全双工时 tx/rx buffer 长度应相同；半双工只发或只收时另一方传 NULL
- CS 默认高有效（active high），若外设要求低有效需在 config.operation 中设置对应标志位
