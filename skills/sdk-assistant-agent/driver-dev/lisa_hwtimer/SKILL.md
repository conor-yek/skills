---
name: lisa_hwtimer
description: lisa_hwtimer 驱动使用与移植辅助。涵盖 GPT 通用定时器、AON（Always-On）低功耗定时器、双定时器模式，超时回调配置。
---

# lisa_hwtimer 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_hwtimer/lisa_hwtimer.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_hwtimer/README.html`
- **示例目录**: `samples/drivers/devices/lisa_hwtimer/`（gpt_timer / aon_timer / dual_timer）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_hwtimer/lisa_hwtimer.h
   → API 函数签名、lisa_hwtimer_config_t 结构体、时间单位、回调原型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_hwtimer/README.html
   → prj.conf 配置（CONFIG_LISA_HWTIMER_DEVICE）、GPT/AON 区别、精度说明

③ Glob  samples/drivers/devices/lisa_hwtimer/**
   → 按场景选 main.c：
     - 通用高精度定时     → gpt_timer/main.c
     - 低功耗睡眠定时     → aon_timer/main.c
     - 两个定时器级联或并行 → dual_timer/main.c
```

### Step 2: 回答规则

- **GPT vs AON**：GPT 精度高（纳秒级），AON 工作于低功耗状态（精度较低但可在 sleep 中运行）；根据用户场景引导选择
- 超时时间单位从头文件确认（ns / us / ms / tick），不要假设
- 超时回调在中断上下文执行，提醒用户不要在回调中做阻塞操作（`lisa_os_sleep`、`printf`等）
- 单次（one-shot）vs 周期（periodic）模式：从 config 结构体确认配置方式

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_hwtimer/lisa_hwtimer.h       → vtable 接口（lisa_hwtimer_api_t）
② Read  drivers/lisa_hwtimer/lisa_hwtimer_arcs.c  → 平台适配：硬件定时器寄存器、时钟源配置
```

### HWTimer 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `init(dev, cfg, cb, user_data)` — 初始化定时器，绑定超时回调
- `start(dev, ticks)` — 启动定时，ticks 值的时间基准由驱动时钟确定
- `stop(dev)` — 停止定时
- `get_value(dev)` — 读取当前计数值（用于测量经过时间）
- `set_top_value(dev, cfg)` — 设置计数上限（周期定时器模式）

**GPT 定时器**：
- 时钟源通常来自高速总线时钟，精度可达纳秒级
- 适合精确延时、PWM 辅助计时、脉冲测量

**AON 定时器**：
- 时钟源为低速时钟（32kHz 或 RC 振荡器），在系统 sleep 时仍可运行
- 精度约 31-100us/tick，不适合高精度计时
- 适合低功耗模式下的唤醒定时

**已知注意点**：
- 定时器回调在中断上下文执行，若需在回调中通知任务，使用 `lisa_os_sem_give_from_isr` 等 ISR 安全 API
- AON 定时器时钟频率随温度/电压漂移，长时间精度不可依赖
- 移植不同硬件时注意寄存器读写顺序（先清标志再处理逻辑，避免中断重入）
