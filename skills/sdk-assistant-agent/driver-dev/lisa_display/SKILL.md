---
name: lisa_display
description: lisa_display 驱动使用与移植辅助。涵盖帧缓冲刷新、显示格式配置、LVGL 集成、背光控制。
---

# lisa_display 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_display/lisa_display.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_display/README.html`
- **示例目录**: `samples/drivers/devices/lisa_display/`（display_flush）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_display/lisa_display.h
   → API 函数签名、lisa_display_capabilities_t / lisa_display_buffer_descriptor_t 结构体、像素格式枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_display/README.html
   → prj.conf 配置（CONFIG_LISA_DISPLAY_DEVICE）、分辨率参数、像素格式支持列表、LVGL 集成说明

③ Read  samples/drivers/devices/lisa_display/display_flush/main.c
   → 完整的初始化 → 获取显示参数 → 帧缓冲刷新流程
```

### Step 2: 回答规则

- 刷新前需先获取显示能力（`get_capabilities`）确认分辨率和支持的像素格式
- `write` 接口参数包含目标区域（x, y, width, height）和数据 buffer，支持局部刷新
- **LVGL 集成**：LVGL 通过 `lv_disp_drv_t.flush_cb` 回调调用 `lisa_display_write`，确认 LVGL 版本与驱动接口的适配层
- 背光控制通常通过独立 GPIO 或 PWM 驱动实现，不在 `lisa_display` vtable 内

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_display/lisa_display.h       → vtable 接口（lisa_display_api_t）
② Read  drivers/lisa_display/lisa_display_arcs.c  → 平台适配：底层 LCD 控制器或 SPI/QSPI 传输
```

### Display 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `write(dev, x, y, desc, buf)` — 将 buf 中的像素数据刷新到屏幕 (x,y) 起始的矩形区域
- `read(dev, x, y, desc, buf)` — 从屏幕读取像素数据（可选，部分硬件不支持回读）
- `blanking_on(dev)` / `blanking_off(dev)` — 开启/关闭显示（黑屏省电 vs 正常显示）
- `get_capabilities(dev, caps)` — 查询分辨率、支持的像素格式、帧缓冲对齐要求

**像素格式适配**：
- 常见格式：RGB565（2字节/像素）、RGB888（3字节）、ARGB8888（4字节）
- 移植时在 `get_capabilities` 中准确返回硬件实际支持的格式列表
- 若 LVGL 使用的格式与硬件不同，需在适配层做格式转换

**局部刷新优化**：
- 只刷新脏区域（dirty region）可显著降低总线带宽和 CPU 占用
- 移植层收到 `write` 调用后，只向 LCD 控制器发送对应矩形区域的像素命令序列

**已知注意点**：
- `buf` 指针在 `write` 返回前必须保持有效（DMA 传输期间不能释放）
- LVGL 的 `flush_ready()` 必须在硬件传输完成后才调用，否则 LVGL 过早复用 buffer 导致画面撕裂
- SPI LCD 移植时注意 DC（数据/命令）引脚的切换时序
