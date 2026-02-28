---
name: lisa_rtc
description: lisa_rtc 驱动使用与移植辅助。涵盖时间读写、闹钟配置、时区处理、低功耗唤醒。
---

# lisa_rtc 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_rtc/lisa_rtc.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_rtc/README.html`
- **示例目录**: `samples/drivers/devices/lisa_rtc/`（time_basic / alarm）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_rtc/lisa_rtc.h
   → API 函数签名、lisa_rtc_time_t 结构体（年月日时分秒字段）、闹钟回调原型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_rtc/README.html
   → prj.conf 配置（CONFIG_LISA_RTC_DEVICE）、时钟源要求（32.768kHz 晶振）

③ Glob  samples/drivers/devices/lisa_rtc/**
   → 按场景选 main.c：
     - 基础时间读写  → time_basic/main.c
     - 闹钟触发     → alarm/main.c
```

### Step 2: 回答规则

- `lisa_rtc_time_t` 结构体字段含义从头文件确认（年份是否从 1900 偏移、月份是否从 0 起）
- RTC 上电后若无初始化，返回的时间可能是随机值，需先调用 `set_time` 设置正确时间
- 闹钟回调在中断上下文执行，不要在回调中做阻塞操作
- 提醒用户：RTC 需要 32.768kHz 低速晶振才能在系统掉电后保持时间

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_rtc/lisa_rtc.h       → vtable 接口（lisa_rtc_api_t）及时间结构体
② Read  drivers/lisa_rtc/lisa_rtc_arcs.c  → 平台适配：BCD/二进制转换、闹钟中断配置
```

### RTC 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `set_time(dev, timeptr)` — 设置当前时间
- `get_time(dev, timeptr)` — 读取当前时间
- `alarm_set_time(dev, id, mask, timeptr)` — 配置闹钟（id 为闹钟编号，mask 指定匹配字段）
- `alarm_set_callback(dev, id, cb, user_data)` — 绑定闹钟回调
- `alarm_get_time(dev, id, mask, timeptr)` — 读取已配置的闹钟时间

**时间编码**：
- 部分硬件 RTC 使用 BCD（Binary Coded Decimal）编码存储时间，移植层需做 BCD ↔ 二进制转换
- `lisa_rtc_time_t` 通常以二进制存储，底层驱动处理格式转换

**闹钟 mask 机制**：
- mask 指定哪些字段（秒/分/时/日/月）参与匹配，未设置字段视为"任意匹配"
- 例：只匹配分和秒 → 每小时的固定时刻触发；仅匹配秒 → 每分钟触发

**已知注意点**：
- RTC 掉电保持需要独立电源域（VBAT），移植时确认硬件是否支持掉电保持
- 系统重启后应在应用层确认 RTC 时间有效性（如检查年份是否合理）
- 若 SoC 没有独立硬件 RTC，可用 AON Timer 模拟（精度和掉电保持能力有限）
