---
name: component-dev
description: ARCS SDK 组件开发辅助。创建新组件骨架代码，包含头文件、实现、CMakeLists.txt、Kconfig，支持事件发布集成。当用户要开发新组件、封装抽象层时触发。
---

# ARCS 组件开发辅助

帮助开发者创建符合 ARCS SDK 规范的功能组件。

## 触发条件

- 用户要创建新的 component
- 用户需要封装一个抽象层/中间件
- 用户要设计组件间通信（evt_pub 集成）

## 执行流程

### Step 1: 确认组件定位

了解组件功能、对外 API、依赖的驱动或其他组件。

### Step 2: 参考现有组件

读取 `components/` 下最相关的组件作为参考。

### Step 3: 生成骨架

1. `include/lisa_xxx.h` — 公共 API
2. `lisa_xxx.c` — 实现
3. `CMakeLists.txt` — 构建配置
4. `Kconfig` — 配置选项

### Step 4: 验证

检查依赖完整性、API 一致性、构建集成正确性。

## 状态: STUB — 待 Phase 3 完整实现
