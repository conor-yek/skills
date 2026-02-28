---
name: lisa_uart
description: lisa_uart 驱动使用与移植辅助。涵盖轮询输出、中断接收、DMA 异步发送、多实例配置。
---

# lisa_uart 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_uart/lisa_uart.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_uart/README.html`
- **示例目录**: `samples/drivers/devices/lisa_uart/`（poll_out / recv_sync_int / send_async_dma 等）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_uart/lisa_uart.h
   → API 函数签名、lisa_uart_config_t 结构体、波特率/奇偶校验/停止位枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_uart/README.html
   → prj.conf 配置要求、引脚复用说明

③ Glob  samples/drivers/devices/lisa_uart/**
   → 按场景选 main.c：
     - 轮询发送单字符   → poll_out/main.c
     - 中断方式接收     → recv_sync_int/main.c
     - DMA 异步批量发送 → send_async_dma/main.c
```

### Step 2: 回答规则

- 引用头文件中的实际枚举值（波特率、数据位、奇偶校验等），不要硬编码数字
- **多实例**：UART0/UART1/… 通过 `lisa_device_get_binding("UART0")` 获取；提醒用户确认实际设备名与板级 pinmux
- **模式区分**：轮询/中断/DMA 是完全不同的 API 路径，引导用户先确认使用哪种模式

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_uart/lisa_uart.h       → vtable 接口（lisa_uart_api_t）及配置结构体
② Read  drivers/lisa_uart/lisa_uart_arcs.c  → 平台适配层：波特率计算、FIFO 管理、DMA 通道配置
```

### UART 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `configure(dev, cfg)` — 设置波特率/数据位/停止位/奇偶校验
- `poll_in(dev, c)` / `poll_out(dev, c)` — 阻塞式单字节收发
- `fifo_read(dev, buf, len)` / `fifo_fill(dev, buf, len)` — FIFO 批量操作
- `irq_callback_set(dev, cb)` — 设置中断回调
- `irq_rx_enable(dev)` / `irq_tx_enable(dev)` — 使能收发中断
- `irq_tx_ready(dev)` / `irq_rx_ready(dev)` — 在回调中判断事件来源

**DMA 模式**：
- DMA 路径通常独立于 IRQ 路径，通过异步 API 触发（从 send_async_dma 示例确认具体接口）
- DMA 完成回调需在回调函数中处理缓冲区状态

**多实例设计**：
- 每个 UART 实例（UART0/1/…）有独立的基地址 + 中断号，通过 cfg 结构体区分
- IRQ handler 通过 `DEVICE_GET(uart_arcs_N)` 获取设备实例后调用公共处理逻辑

**已知注意点**：
- `poll_out` 是阻塞调用，在实时任务中慎用
- 中断接收流程：`irq_callback_set` → `irq_rx_enable` → 在 callback 中检查 `irq_rx_ready` → `fifo_read`，顺序不能乱
- DMA 发送期间缓冲区必须保持有效（不可在栈上分配发送 buffer）
