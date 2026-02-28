---
name: lisa_camera
description: lisa_camera 驱动使用与移植辅助。涵盖摄像头高层抽象接口、Sensor 初始化、图像捕获、与 lisa_dvp 的分层关系。
---

# lisa_camera 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_camera/lisa_camera.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_camera/README.html`
- **示例目录**: `samples/drivers/devices/lisa_camera/`（平铺，无子目录）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_camera/lisa_camera.h
   → API 函数签名、lisa_camera_config_t 结构体、图像格式/分辨率枚举、帧缓冲结构体

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_camera/README.html
   → prj.conf 配置（CONFIG_LISA_CAMERA_DEVICE）、支持的 Sensor 型号（GC0308/OV2640等）、Kconfig 选择

③ Glob  samples/drivers/devices/lisa_camera/**
   → Read 找到的 main.c（示例平铺，无子目录结构）
```

### Step 2: 回答规则

- `lisa_camera` 是**高层抽象**，内部封装了 Sensor 的 I2C 配置控制 + DVP 接口数据传输
- 用户代码通常只与 `lisa_camera` 交互，不需要直接操作 `lisa_dvp`
- **Sensor 型号**：不同 Sensor 的初始化寄存器序列不同，通过 Kconfig 选择使用哪个 Sensor 驱动
- 图像数据格式（YUV422/RGB565/JPEG）从 `lisa_camera_config_t` 配置，需与 Sensor 支持的格式匹配

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_camera/lisa_camera.h       → vtable 接口（lisa_camera_api_t）
② Read  drivers/lisa_camera/lisa_camera_arcs.c  → 平台适配：DVP 控制器初始化、Sensor I2C 配置
```

### Camera 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `init(dev, cfg)` — 初始化 DVP 控制器 + 通过 I2C 配置 Sensor 寄存器（分辨率/格式/帧率）
- `capture(dev, buf, size, cb)` — 启动一次帧捕获，完成后回调
- `stream_on(dev, cb)` — 启动连续视频流（乒乓缓冲模式）
- `stream_off(dev)` — 停止视频流
- `ioctl(dev, cmd, arg)` — 运行时控制（曝光/增益/白平衡/对比度等）

**Sensor 驱动层次**：
```
lisa_camera（用户 API）
    ├── lisa_dvp（DVP 数据通路，负责像素接收）
    └── Sensor 驱动（I2C 配置，负责图像参数）
```
移植新 Sensor 时：实现 Sensor 的 `init`（寄存器初始化序列）和 `ioctl`（参数调整）

**已知注意点**：
- Sensor I2C 地址和寄存器格式因型号而异，移植时需查阅具体 Sensor 数据手册
- 部分 Sensor 需要 XCLK（外部时钟输入），需在移植层配置 SoC 输出对应频率的时钟（通常 24MHz）
- 帧率不稳定时，首先检查 DVP 的 PCLK 采样边沿和 VSYNC/HREF 极性是否与 Sensor 时序匹配
