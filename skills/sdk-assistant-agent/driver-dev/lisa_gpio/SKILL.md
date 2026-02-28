---
name: lisa_gpio
description: lisa_gpio 驱动使用与移植辅助。涵盖 GPIO 输出控制、输入读取、中断回调、多实例（GPIOA/GPIOB）及 pinmux 配置。
---

# lisa_gpio 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_gpio/lisa_gpio.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_gpio/README.html`
- **示例目录**: `samples/drivers/devices/lisa_gpio/`（output_basic / input_basic / interrupt）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_gpio/lisa_gpio.h
   → API 函数签名、lisa_gpio_pin_t / lisa_gpio_flags_t 枚举、回调结构体定义

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_gpio/README.html
   → prj.conf 配置要求（CONFIG_LISA_GPIO_DEVICE）、注意事项

③ Glob  samples/drivers/devices/lisa_gpio/**
   → 按场景选 main.c：
     - 输出控制  → output_basic/main.c
     - 输入读取  → input_basic/main.c
     - 中断回调  → interrupt/main.c
```

### Step 2: 回答规则

- 引用头文件中的实际函数名与枚举值，不猜测
- 给出可编译代码片段（从 main.c 参考）
- **多实例**：同一 SoC 上通常有 GPIOA / GPIOB 等，通过 `lisa_device_get_binding("GPIOA")` 获取；提醒用户确认实际设备名
- **pinmux**：GPIO 引脚使用前通常需要 pinmux 配置，提醒用户参考板级 `lisa_gpio_pinmux()` 函数或 `drivers/lisa_pinmux/README.html`

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_gpio/lisa_gpio.h       → 了解 vtable 接口（lisa_gpio_api_t）
② Read  drivers/lisa_gpio/lisa_gpio_arcs.c  → 了解平台适配层实现方式
```

### GPIO 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `configure(dev, pin, flags)` — 设置引脚方向、上下拉、中断触发模式
- `get_raw(dev, value)` / `set_raw(dev, set_pins, clear_pins)` — 底层寄存器读写
- `manage_callback(dev, cb, set)` — 注册/注销中断回调
- `enable_callback(dev, pin)` / `disable_callback(dev, pin)` — 中断使能控制

**多实例设计**：
- 每个 GPIO Bank（GPIOA/GPIOB）对应一个 `LISA_DEVICE_REGISTER` 实例
- 私有数据中存储 Bank 基地址，由 cfg 结构体区分
- 回调链表通常在 priv 中维护，支持多个 handler 注册同一 pin

**中断回调适配**：
- 平台 ISR → 遍历回调链表 → 调用匹配 pin 的 handler
- 回调结构体通常包含 `pin_mask`（按位过滤）+ `handler` 函数指针

**已知注意点**：
- `flags` 参数组合（方向 | 电平 | 中断触发）需与头文件枚举严格对应，不要裸传数字
- 中断模式下需先 `manage_callback` 注册，再 `configure` 使能中断触发，顺序不能反
