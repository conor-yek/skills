# ARCS SDK 编码约定速查

## 文件命名
- 驱动: `lisa_xxx.h` / `lisa_xxx_arcs.c`
- 组件: `lisa_xxx.h` / `lisa_xxx.c`
- 模块前缀: `module_lisa_xxx` (CMake target name)

## 头文件保护
```c
#pragma once
// 或
#ifndef __LISA_XXX_H__
#define __LISA_XXX_H__
// ...
#endif
```

## 日志
```c
#define LOG_TAG "module_name"  // 必须在 include 之前
#include <lisa_log.h>
LOGI("message");              // Info
LOGW("message");              // Warning
LOGE("message");              // Error
LOGD("message");              // Debug
LOGV("message");              // Verbose
LOGH("tag", buf, len);        // Hex dump
```

## 错误处理
- 返回 0 表示成功，负数表示错误
- 使用 `LISA_DEVICE_ERR_*` 预定义错误码
- inline 包装函数必须检查 dev 和 dev->api 是否为 NULL

## C/C++ 兼容
```c
#ifdef __cplusplus
extern "C" {
#endif
// ... declarations ...
#ifdef __cplusplus
}
#endif
```

## API 设计
- 配置类: `xxx_configure(dev, config)` — 使用 const config 指针
- 同步操作: `xxx_write_sync(dev, buf, len, timeout_ms)`
- 异步操作: 通过事件回调 `xxx_callback_t`
- 使能/禁用: `xxx_enable(dev)` / `xxx_disable(dev)`

## Kconfig 约定
- 驱动选项: `LISA_XXX_DEVICE`，依赖 `LISA_DEVICE`
- 组件选项: `LISA_XXX`
- 子选项用 `if/endif` 包裹
