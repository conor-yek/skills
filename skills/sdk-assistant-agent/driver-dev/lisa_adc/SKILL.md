---
name: lisa_adc
description: lisa_adc 驱动使用与移植辅助。涵盖多通道采样、温度传感器通道、采样精度配置、连续/单次采样模式。
---

# lisa_adc 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_adc/lisa_adc.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_adc/README.html`
- **示例目录**: `samples/drivers/devices/lisa_adc/`（read_basic / read_temperature_channel）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_adc/lisa_adc.h
   → API 函数签名、lisa_adc_channel_cfg_t 结构体、增益/参考/分辨率枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_adc/README.html
   → prj.conf 配置（CONFIG_LISA_ADC_DEVICE）、通道编号说明、电压参考说明

③ Glob  samples/drivers/devices/lisa_adc/**
   → 按场景选 main.c：
     - 普通外部通道采样  → read_basic/main.c
     - 片内温度传感器    → read_temperature_channel/main.c
```

### Step 2: 回答规则

- ADC 采样前需先 `channel_setup` 配置通道参数（增益、参考电压、分辨率、通道 ID）
- **温度通道**：温度传感器映射到特定通道 ID，从文档或 read_temperature_channel 示例确认通道号
- 原始值转电压/温度的换算公式从文档或示例代码中获取，不要自行推导
- 提醒用户：ADC 引脚通常需要 pinmux 配置，参考 `drivers/lisa_pinmux/README.html`

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_adc/lisa_adc.h       → vtable 接口（lisa_adc_api_t）及通道配置结构体
② Read  drivers/lisa_adc/lisa_adc_arcs.c  → 平台适配：时钟配置、DMA/中断触发、通道寄存器映射
```

### ADC 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `channel_setup(dev, channel_cfg)` — 配置指定通道的增益、参考、分辨率、差分/单端模式
- `read(dev, sequence, async)` — 执行采样序列（支持多通道批量采样）
- `ref_internal_mv(dev)` — 返回内部参考电压值（mV），用于电压换算

**采样序列设计**：
- `lisa_adc_sequence` 包含 buffer + buffer_size + channels mask + oversampling 配置
- 多通道采样结果按通道 ID 顺序存入 buffer，移植时注意结果排列顺序与硬件实际顺序对应

**温度传感器通道**：
- 通常是一个特殊的内部通道 ID（如 channel 8 或 channel 16），平台相关
- 移植时需在 arcs.c 中处理该通道的特殊初始化路径（开启温度传感器电源域）

**已知注意点**：
- `channel_setup` 必须在 `read` 之前对每个使用的通道调用，否则结果未定义
- 采样分辨率（8/10/12/16 bit）影响原始值范围，换算公式必须与配置的分辨率匹配
- 连续采样模式（若支持）的触发方式从 arcs.c 确认，部分实现只支持单次触发
