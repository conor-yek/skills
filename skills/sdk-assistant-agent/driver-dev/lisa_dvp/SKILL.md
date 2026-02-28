---
name: lisa_dvp
description: lisa_dvp 驱动使用与移植辅助。涵盖 DVP（Digital Video Port）摄像头输入、普通模式/乒乓缓冲模式、帧同步回调。
---

# lisa_dvp 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_dvp/lisa_dvp.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_dvp/README.html`
- **示例目录**: `samples/drivers/devices/lisa_dvp/`（normal_mode / pingpong_mode）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_dvp/lisa_dvp.h
   → API 函数签名、lisa_dvp_config_t 结构体（分辨率/格式/帧率）、帧回调原型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_dvp/README.html
   → prj.conf 配置（CONFIG_LISA_DVP_DEVICE）、支持的像素格式（YUV/RGB/JPEG）、VSYNC/HSYNC 极性

③ Glob  samples/drivers/devices/lisa_dvp/**
   → 按场景选 main.c：
     - 单缓冲捕获（采完停止）    → normal_mode/main.c
     - 双缓冲连续捕获（乒乓）    → pingpong_mode/main.c
```

### Step 2: 回答规则

- **normal vs pingpong**：normal 模式捕获一帧后停止，适合拍照；pingpong 持续采集，适合实时预览/视频
- DVP 是摄像头的接口层，通常配合 `lisa_camera`（更高层抽象）或直接操作
- 帧回调在中断上下文执行，通知应用处理帧数据时应使用信号量/消息队列，不要在回调中做图像处理
- 像素格式（YUV422/RGB565/JPEG）需与摄像头 Sensor 的输出格式配置一致

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_dvp/lisa_dvp.h       → vtable 接口（lisa_dvp_api_t）及帧缓冲结构体
② Read  drivers/lisa_dvp/lisa_dvp_arcs.c  → 平台适配：DVP 控制器时序、DMA 帧传输、VSYNC/HREF 中断
```

### DVP 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `configure(dev, cfg)` — 设置分辨率、像素格式、帧率、同步信号极性
- `set_frame_buffer(dev, buf, size)` — 设置帧接收缓冲区（普通模式）
- `set_frame_buffer_pingpong(dev, buf0, buf1, size)` — 设置乒乓缓冲区
- `start(dev, cb)` — 启动捕获，注册帧完成回调
- `stop(dev)` — 停止捕获

**乒乓缓冲机制**：
- DVP 控制器每采完一帧，触发 VSYNC 中断，自动切换到另一个缓冲区继续采集
- 上一帧的 buffer 在切换后交给应用处理，应用处理完后需归还给驱动（部分实现需要显式归还）

**时序配置**：
- DVP 接口需配置 PCLK 极性（上升/下降沿采样）、VSYNC/HREF 有效极性
- 移植时这些参数必须与摄像头 Sensor 数据手册中的 DVP 时序完全匹配

**已知注意点**：
- DVP 是并行接口（8/10/12 位数据线 + 时钟 + VSYNC + HREF），引脚数量较多，pinmux 配置复杂
- 帧率由摄像头 Sensor 的 PCLK 和行/帧消隐时序决定，DVP 控制器被动接收，不主动控制帧率
- 大分辨率（如 1080p）帧数据量大，DMA 传输时间长，确保 DMA buffer 地址对齐（通常 4 字节）
