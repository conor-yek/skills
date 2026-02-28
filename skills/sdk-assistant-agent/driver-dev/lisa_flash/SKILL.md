---
name: lisa_flash
description: lisa_flash 驱动使用与移植辅助。涵盖 Flash 读写擦除、单核/双核安全访问（halt_remote_core 模式）、页/块对齐要求。
---

# lisa_flash 驱动辅助

## 资源索引

- **头文件**: `drivers/lisa_flash/lisa_flash.h`
- **在线文档**: `https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_flash/README.html`
- **示例目录**: `samples/drivers/devices/lisa_flash/`（single_core / halt_remote_core）

---

## 使用说明

### Step 1: 并行加载上下文

同时执行（不要串行等待）：

```
① Read  drivers/lisa_flash/lisa_flash.h
   → API 函数签名、lisa_flash_pages_info_t 结构体、erase/write/read 参数类型

② WebFetch https://docs2.listenai.com/arcs-sdk/latest/zh/html/drivers/lisa_flash/README.html
   → prj.conf 配置（CONFIG_LISA_FLASH_DEVICE）、Flash 分区说明、双核注意事项

③ Glob  samples/drivers/devices/lisa_flash/**
   → 按场景选 main.c：
     - 单核场景（单 CPU 直接读写）  → single_core/main.c
     - 双核场景（需先暂停另一核）  → halt_remote_core/main.c
```

### Step 2: 回答规则

- **双核系统**：flash 操作期间必须暂停另一核（`halt_remote_core` 模式），否则另一核从 XIP Flash 取指令会导致 Hardfault；务必提醒用户
- 擦除地址和长度必须页对齐（从文档或头文件获取页大小，通常 4KB）
- 写入地址必须满足写对齐要求（通常 1 字节或 4 字节，从头文件确认）
- 读操作通常无对齐限制

---

## 移植说明

### 加载参考实现

并行读取：

```
① Read  drivers/lisa_flash/lisa_flash.h       → vtable 接口（lisa_flash_api_t）及参数类型
② Read  drivers/lisa_flash/lisa_flash_arcs.c  → 平台适配：QSPI/SPI 控制器操作、双核同步机制
```

### Flash 移植要点

**vtable 核心接口**（从头文件确认实际定义）：
- `read(dev, offset, data, len)` — 从 offset 读取 len 字节
- `write(dev, offset, data, len)` — 写入，offset 和 len 需满足写对齐
- `erase(dev, offset, size)` — 擦除，offset 和 size 必须页对齐
- `get_parameters(dev, params)` — 获取 Flash 参数（页大小、总大小等）
- `write_protection_set(dev, enable)` — 写保护控制（可选接口）

**双核安全访问设计**：
- ARCS 是双核 SoC，两个核共享 Flash（XIP 模式）
- Flash 擦写期间 Flash 总线被独占，XIP 取指必须停止
- `halt_remote_core` 示例展示了暂停/恢复远端核的标准流程
- 移植时在 `erase` / `write` 实现内部（或外部封装层）加入双核同步

**地址偏移说明**：
- `offset` 参数是相对 Flash 起始地址的偏移，不是绝对地址
- 移植层需将 offset 转换为实际物理地址（通常加上 Flash 基地址）

**已知注意点**：
- 擦除是最耗时操作（页擦除约 20-100ms），在 RTOS 任务中注意超时配置
- 写入前必须先擦除（NOR Flash 只能将 1 写为 0，擦除将整页恢复为 FF）
- 移植层实现 `get_parameters` 时页大小、写入大小、擦除大小要与实际芯片手册匹配
