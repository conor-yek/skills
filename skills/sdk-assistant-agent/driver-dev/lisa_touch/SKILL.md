---
name: lisa_touch
description: lisa_touch 驱动使用与移植辅助。涵盖触摸坐标获取、轮询/中断两种模式、多点触控、与 LVGL 集成。
---

# lisa_touch 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_touch/lisa_touch.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_touch/README.html`
- **示例目录**: `samples/drivers/devices/lisa_touch/`（polling_mode / interrupt_mode）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_touch/lisa_touch.h
   → API 函数签名、lisa_touch_data_t / lisa_touch_point_t 结构体、触摸事件类型枚举

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_touch/README.html
   → prj.conf 配置（CONFIG_LISA_TOUCH_DEVICE）、I2C 地址配置、中断引脚说明

③ Glob  samples/drivers/devices/lisa_touch/**
   → 按场景选 main.c：
     - 定时轮询坐标     → polling_mode/main.c
     - 中断触发读取     → interrupt_mode/main.c
```

### Step 2: 回答规则

- **轮询 vs 中断**：轮询适合资源充裕场景（每 20-50ms 查询一次）；中断模式功耗更低，响应更及时
- 触摸控制器通常通过 I2C 连接，坐标读取前需确认 I2C 设备已初始化
- **LVGL 集成**：LVGL 通过 `lv_indev_drv_t.read_cb` 回调读取触摸数据，给出对应适配代码
- 坐标原点（左上/右下/旋转）需与显示方向对齐，必要时在驱动层做坐标变换

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_touch/lisa_touch.h       → vtable 接口（lisa_touch_api_t）及数据结构体
② Read  drivers/lisa_touch/lisa_touch_arcs.c  → 平台适配：I2C 读取时序、中断引脚配置、坐标校准
```

### Touch 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `get_point(dev, data, point_count)` — 读取当前所有触摸点坐标（返回实际有效点数）
- `set_callback(dev, cb)` / `get_callback(dev)` — 注册触摸中断回调（中断模式）
- `get_capabilities(dev, caps)` — 查询最大触摸点数、分辨率

**多点触控**：
- `lisa_touch_data_t` 包含一个 `lisa_touch_point_t` 数组，每个元素代表一个触摸点
- `point_count` 表示当前实际触摸点数（1=单点，>1=多点）
- 移植时从 IC 寄存器读取所有有效点数据，填入数组后返回

**I2C 通信适配**：
- 大多数触摸 IC（GT9xx / FT5xxx / CST8xx 等）通过 I2C 读取寄存器报告坐标
- 移植时需查阅触摸 IC 数据手册，实现正确的寄存器读取和数据解析

**坐标变换**：
- 若触摸 IC 输出坐标与显示方向不一致（镜像/旋转），在 `get_point` 实现中做变换
- 变换公式：水平镜像 `x' = max_x - x`；垂直镜像 `y' = max_y - y`

**已知注意点**：
- 中断模式下，INT 引脚通常需要配置为下降沿触发，在 GPIO 中断回调中给信号量，由任务调用 `get_point`（不要在 ISR 中直接做 I2C 操作）
- 部分触摸 IC 的 INT 引脚兼具复位功能，上电序列需按 datasheet 要求操作
