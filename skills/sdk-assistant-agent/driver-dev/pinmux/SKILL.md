---
name: pinmux
description: ARCS SDK pinmux 配置辅助。pinmux 不是 lisa_device 框架驱动，通过 IOMuxManager 接口配置引脚功能复用。涵盖 pad/pin/func 参数查询、覆盖函数模式、与其他驱动的配合。
---

# pinmux 配置辅助

> ⚠️ **特殊说明**：pinmux **不是** lisa_device 框架的设备驱动，没有 vtable 和 `LISA_DEVICE_REGISTER`。
> 通过 `IOMuxManager_PinConfigure(pad, pin, func)` 直接调用 HAL 层接口。

## 资源索引

- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_pinmux/README.html`
- **头文件**: 无独立 lisa_device 头文件，HAL 接口在 SoC HAL 层（参考 `soc/arcs/hal/`）
- **示例参考**: 各驱动 sample 中的 `lisa_<xxx>_pinmux()` 覆盖函数（与 lisa_gpio/lisa_uart 等配套）

---

## 使用说明

### Step 1: 加载 pinmux 文档

```
WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_pinmux/README.html
→ pad / pin / func 参数含义、引脚复用功能表、板级 pinmux 覆盖机制
```

同时查找目标驱动的参考 sample 中的 pinmux 设置：

```
Glob  samples/drivers/devices/lisa_<目标驱动>/**/main.c 或 board 目录
→ 找到 lisa_<xxx>_pinmux() 函数，了解同类外设的 pinmux 配置方式
```

### Step 2: 回答规则

- pinmux 参数（pad, pin, func）从文档的引脚复用功能表中查找，不要猜测数字
- **覆盖函数模式**：SDK 默认板级 pinmux 通过 `lisa_<xxx>_pinmux()` 弱符号提供，用户可在自己的代码中重新定义同名强符号覆盖默认实现
- 若用户问"怎么配 GPIO 引脚复用为 UART"等场景，给出 `IOMuxManager_PinConfigure` 调用示例 + 对应 pad/pin/func 值

---

## 移植说明

### pinmux 配置原理

pinmux 的职责：将物理引脚（PAD）的功能切换到目标外设（UART/SPI/I2C/GPIO/…）。

```c
// 基础调用方式
IOMuxManager_PinConfigure(pad_id, pin_func, extra_cfg);
// pad_id: 引脚编号（如 PAD_GPIO_A0）
// pin_func: 功能编号（如 FUNC_UART0_TX）
// extra_cfg: 额外属性（上下拉、驱动强度等）
```

### 新驱动移植中的 pinmux 处理

在新驱动的平台适配层（`lisa_<name>_arcs.c`）中：

**方式 A（弱符号覆盖）**：
```c
// 提供默认弱实现（用户可覆盖）
__attribute__((weak)) void lisa_<name>_pinmux(void)
{
    IOMuxManager_PinConfigure(PAD_XXX, FUNC_XXX, 0);
}
```

在驱动 init 函数中调用 `lisa_<name>_pinmux()`，用户可在应用层定义同名强函数覆盖默认配置。

**方式 B（Kconfig 控制）**：
部分驱动通过 Kconfig 选项选择引脚，在 `lisa_<name>_arcs.c` 中用 `#ifdef CONFIG_LISA_<NAME>_PIN_XXX` 区分。

### 已知注意点

- pad 编号与 GPIO 编号不是同一概念：PAD 是物理引脚，GPIO 是逻辑功能，一个 PAD 可以复用为多种功能
- 同一 PAD 同一时刻只能配置为一种功能，两个外设不能共享同一 PAD
- 部分 PAD 有上下拉默认值，若外设要求特定状态，需在 `IOMuxManager_PinConfigure` 中显式设置
- JTAG/SWD 调试引脚通常与 GPIO 复用，开发阶段若重配这些引脚会丢失调试连接
