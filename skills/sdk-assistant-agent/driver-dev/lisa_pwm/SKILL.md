---
name: lisa_pwm
description: lisa_pwm 驱动使用与移植辅助。涵盖 PWM 输出基础配置、频率/占空比设置、多通道管理、极性控制。
---

# lisa_pwm 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_pwm/lisa_pwm.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_pwm/README.html`
- **示例目录**: `samples/drivers/devices/lisa_pwm/`（output_basic）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_pwm/lisa_pwm.h
   → API 函数签名、lisa_pwm_dt_spec_t / lisa_pwm_channel_t 结构体、标志位枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_pwm/README.html
   → prj.conf 配置（CONFIG_LISA_PWM_DEVICE）、通道编号与引脚映射、时钟精度说明

③ Read  samples/drivers/devices/lisa_pwm/output_basic/main.c
   → 标准设置周期和占空比的完整流程
```

### Step 2: 回答规则

- PWM 周期（period）和脉冲宽度（pulse）单位通常为**纳秒（ns）**，注意单位换算
- 占空比 = pulse / period，范围 [0, period]；`pulse = 0` 为全低，`pulse = period` 为全高
- 极性（INVERTED flag）：正常极性高电平有效；若硬件信号逻辑相反，设置反转标志
- 提醒用户：PWM 引脚需 pinmux 配置，参考板级 `lisa_pwm_pinmux()` 或 `drivers/lisa_pinmux/README.html`

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_pwm/lisa_pwm.h       → vtable 接口（lisa_pwm_api_t）及参数类型
② Read  drivers/lisa_pwm/lisa_pwm_arcs.c  → 平台适配：定时器时钟源、分频配置、通道寄存器映射
```

### PWM 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `set_cycles(dev, channel, period_cycles, pulse_cycles, flags)` — 以时钟周期数设置 PWM（底层接口）
- `get_cycles_per_sec(dev)` — 返回 PWM 时钟频率（Hz），用于 ns → cycles 换算
- 高层便捷函数（如 `pin_set(dev, channel, period_ns, pulse_ns, flags)`）通常在头文件中以 inline 实现，内部调用 `set_cycles`

**多通道设计**：
- 一个 PWM 设备实例通常管理多个通道（channel 0/1/2/…）
- 各通道共享同一时钟源和分频器，频率通常一致，仅占空比独立配置
- 若需要不同频率的 PWM，需使用不同 PWM 设备实例

**周期精度**：
- 实际周期受时钟频率和分频限制，只能达到有限精度（time_per_cycle = 1/clock_freq）
- 移植时 `get_cycles_per_sec` 返回的时钟频率必须与实际硬件时钟配置一致

**已知注意点**：
- `set_cycles` 中 `pulse_cycles` 不能大于 `period_cycles`，否则行为未定义
- 输出 50% 占空比：`pulse_cycles = period_cycles / 2`
- 停止 PWM 输出：设置 `pulse_cycles = 0`（持续输出低电平）
