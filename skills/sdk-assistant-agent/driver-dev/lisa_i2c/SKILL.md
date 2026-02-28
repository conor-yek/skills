---
name: lisa_i2c
description: lisa_i2c 驱动使用与移植辅助。涵盖 Master 读写传输、从机地址扫描、Repeated Start、7/10位地址模式。
---

# lisa_i2c 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_i2c/lisa_i2c.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_i2c/README.html`
- **示例目录**: `samples/drivers/devices/lisa_i2c/`（basic_write_read / slave_scan）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_i2c/lisa_i2c.h
   → API 函数签名、lisa_i2c_msg_t 结构体、flags 枚举（READ/WRITE/STOP/RESTART）

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_i2c/README.html
   → prj.conf 配置、时钟频率选项（100kHz/400kHz/1MHz）、注意事项

③ Glob  samples/drivers/devices/lisa_i2c/**
   → 按场景选 main.c：
     - 基础读写外设寄存器  → basic_write_read/main.c
     - 扫描总线上的从机    → slave_scan/main.c
```

### Step 2: 回答规则

- I2C 传输以 `lisa_i2c_msg_t` 数组表达，每个 msg 对应一次 START/RESTART 条件
- **7位地址**：函数参数中地址通常已左移（即传入 `0xXX`，不含 R/W bit），从头文件注释确认
- **写后读**（先发寄存器地址再读数据）：需要两条 msg，第一条 WRITE + RESTART flag，第二条 READ + STOP flag

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_i2c/lisa_i2c.h       → vtable 接口（lisa_i2c_api_t）及 msg 结构体
② Read  drivers/lisa_i2c/lisa_i2c_arcs.c  → 平台适配：时序控制、SCL stretch、中断/DMA 模式
```

### I2C 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `configure(dev, dev_config)` — 设置 Master/Slave 模式、时钟速率、地址位宽
- `transfer(dev, msgs, num_msgs, addr)` — 以 msg 数组执行一次完整 I2C 事务
- `recover_bus(dev)` — 总线死锁时强制恢复（9 个 CLK pulse + STOP）

**msg flags 设计**：
- `LISA_I2C_MSG_WRITE` / `LISA_I2C_MSG_READ` — 传输方向
- `LISA_I2C_MSG_STOP` — 本 msg 结束后发 STOP 条件
- `LISA_I2C_MSG_RESTART` — 本 msg 结束后发 Repeated START（不释放总线）
- 最后一条 msg 必须带 `MSG_STOP`，否则总线不释放

**时钟拉伸（Clock Stretch）**：
- 从机可能拉低 SCL 等待，平台适配层需设置超时以防死锁
- 移植时确认硬件是否支持自动 stretch 检测

**已知注意点**：
- 地址参数传值规范（是否含 R/W bit）因实现而异，务必从头文件注释确认，不要假设
- 总线 scan 时对每个地址做 0 字节写探测，注意区分 NACK（设备不存在）和 ACK（设备存在）
- `recover_bus` 为可选接口（vtable 中可为 NULL），移植时按需实现
