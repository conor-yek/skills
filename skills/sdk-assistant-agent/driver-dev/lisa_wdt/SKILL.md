---
name: lisa_wdt
description: lisa_wdt 驱动使用与移植辅助。涵盖看门狗基础喂狗、中断预警模式、超时配置及系统复位行为。
---

# lisa_wdt 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_wdt/lisa_wdt.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_wdt/README.html`
- **示例目录**: `samples/drivers/devices/lisa_wdt/`（basic / interrupt）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_wdt/lisa_wdt.h
   → API 函数签名、lisa_wdt_config_t 结构体、超时参数类型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_wdt/README.html
   → prj.conf 配置（CONFIG_LISA_WDT_DEVICE）、超时时间范围、中断模式说明

③ Glob  samples/drivers/devices/lisa_wdt/**
   → 按场景选 main.c：
     - 基础看门狗（超时直接复位）   → basic/main.c
     - 中断预警（超时前先触发中断）  → interrupt/main.c
```

### Step 2: 回答规则

- WDT 一旦启动（`install_timeout` + `setup`）通常**无法停止**，提醒用户确认这一点
- 喂狗（`feed`）必须在超时周期内执行，否则系统复位
- **中断模式**：支持在复位前先触发中断（用于保存状态/记录日志），中断回调执行后仍会复位
- 超时时间单位从头文件确认（ms / us / cycles），不要假设

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_wdt/lisa_wdt.h       → vtable 接口（lisa_wdt_api_t）
② Read  drivers/lisa_wdt/lisa_wdt_arcs.c  → 平台适配：时钟源、超时计算、中断使能
```

### WDT 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `setup(dev, options)` — 启动看门狗（通常设置 pause-in-sleep / pause-in-halt 选项）
- `disable(dev)` — 禁用看门狗（部分硬件一旦启动不可禁用）
- `install_timeout(dev, cfg)` — 配置超时时间和回调
- `feed(dev, channel_id)` — 喂狗（复位计数器）

**超时计算**：
- 硬件 WDT 计数器基于固定时钟源，超时配置转换为计数器装载值
- 移植时需将用户传入的 ms 时间转换为时钟周期数，注意上/下溢边界

**中断预警模式**：
- 部分硬件支持两级超时：第一级触发中断（预警），第二级触发复位
- 中断回调中通常只有有限时间（取决于两级超时差），不能做复杂操作

**已知注意点**：
- RTOS 多任务场景：喂狗通常放在独立的低优先级任务中；若高优先级任务死循环阻塞该任务，则无法喂狗 → 系统复位（这是 WDT 的设计目的）
- 调试时 WDT 可能干扰断点调试，`setup` 的 `options` 中通常有 `STALL_ON_HALT` 选项暂停调试时的计数
- 移植层 `disable` 函数若硬件不支持，应返回 `LISA_DEVICE_ERR_NOT_SUPPORT`
