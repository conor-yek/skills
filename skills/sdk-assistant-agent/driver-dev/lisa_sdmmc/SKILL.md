---
name: lisa_sdmmc
description: lisa_sdmmc 驱动使用与移植辅助。涵盖 SD/MMC 卡初始化、块读写、卡状态检测及文件系统集成。
---

# lisa_sdmmc 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_sdmmc/lisa_sdmmc.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_sdmmc/README.html`
- **示例目录**: `samples/drivers/devices/lisa_sdmmc/`（simple_use）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_sdmmc/lisa_sdmmc.h
   → API 函数签名、卡信息结构体（容量/块大小/卡类型）、错误码定义

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_sdmmc/README.html
   → prj.conf 配置（CONFIG_LISA_SDMMC_DEVICE）、支持的卡类型（SD/SDHC/MMC）、电压要求

③ Read  samples/drivers/devices/lisa_sdmmc/simple_use/main.c
   → 卡初始化 → 获取卡信息 → 块读写的完整流程
```

### Step 2: 回答规则

- SDMMC 以**块（block）**为最小读写单元，通常 512 字节/块；读写地址和长度必须块对齐
- 卡操作前需先检查卡是否插入并初始化成功，失败应优雅处理（卡未插/接触不良）
- 若用户需要文件系统（FatFS），提醒 SDMMC 驱动仅提供块 IO，FatFS 集成需要额外配置

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_sdmmc/lisa_sdmmc.h       → vtable 接口（lisa_sdmmc_api_t）及卡信息结构体
② Read  drivers/lisa_sdmmc/lisa_sdmmc_arcs.c  → 平台适配：SDIO/SPI 模式选择、时钟配置、DMA 传输
```

### SDMMC 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `init(dev)` — 初始化控制器并探测卡（发送 CMD0/CMD8/ACMD41 等初始化序列）
- `read(dev, buf, sector, count)` — 从指定扇区读取 count 个块
- `write(dev, buf, sector, count)` — 向指定扇区写入 count 个块
- `ioctl(dev, cmd, buf)` — 获取磁盘信息（总扇区数、扇区大小、块大小）

**SDIO vs SPI 模式**：
- 高速 SDIO 模式：使用专用 SDIO 控制器，4bit 数据线，速度快（25/50MHz）
- SPI 模式：通过 SPI 控制器模拟 SD 协议，速度慢但兼容性好
- 移植时选择对应的控制器实现路径

**DMA 传输**：
- 块读写通常通过 DMA 实现，buffer 需满足 cache 对齐要求（通常 32 字节对齐）
- 移植时确认 DMA 完成等待机制（轮询 vs 信号量）

**已知注意点**：
- 卡初始化速率（400kHz）与数据传输速率（25MHz+）不同，驱动内部需在初始化后切换时钟
- 写操作后需等待卡内部写完成（发送 CMD13 查询 R1 状态），不能立即关断电源
- Hot-plug（热插拔）支持需要检测 GPIO（CD 引脚），移植层按需实现
