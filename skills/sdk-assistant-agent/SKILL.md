---
name: sdk-assistant-agent
description: >
  ARCS SDK / LISTENAI SDK 开发助手统一入口。当用户在此 SDK 工程中进行任何开发任务时触发，包括但不限于：
  驱动开发（LISA_DEVICE_REGISTER、API vtable、外设驱动、UART/SPI/I2C/GPIO/Flash/Display）、
  构建调试（CMake 编译错误、链接失败、Kconfig/menuconfig、map 文件、内存不足）、
  样例生成（怎么用 xxx、写个 demo/sample、使用示例）、
  组件开发（lisa_os、lisa_evt_pub、work_queue、新组件、新模块）、
  代码审查（review、检查代码、PR）、
  SDK 架构问题（SYS_INIT、FreeRTOS、ELF 段、链接脚本、自定义段）。
  凡涉及 arcs_mini、arcs_evb 板型，或 listenai_library、listenai_add_executable 等 CMake 宏的问题，均应触发此 skill。
---

# SDK 开发助手

统一入口 Agent，负责意图识别、skill 路由、执行编排和知识反馈。

> **编写或迭代任何 skill 文件前，先读取 `SKILL_DESIGN_PRINCIPLES.md`。**

## 工作流程

### Step 1: 加载上下文

读取以下知识文件，建立当前会话的知识基础：

1. 读取 `memory/MEMORY.md` — 全局索引和 SDK 概览（已自动加载）
2. 根据用户问题的领域，按需读取对应的详细知识文件：
   - 驱动相关 → `knowledge/driver-patterns.md`
   - 架构模式 → `knowledge/sdk-patterns.md`
   - 构建问题 → `knowledge/build-issues.md`
3. 涉及具体驱动/组件使用时，参考 `knowledge/online-docs.md` 中的 URL 映射，
   用 WebFetch 获取对应的在线文档页面，获取 API 细节、配置要求和使用注意事项。

### Step 2: 意图识别

分析用户输入，判断属于以下哪个领域（可多选）：

| 领域 | 触发信号 | 对应 Skill |
|------|----------|------------|
| **驱动开发** | 新驱动、设备注册、LISA_DEVICE、API vtable、平台适配、外设 | 读取 `driver-dev/SKILL.md` 执行 |
| **构建调试** | 编译错误、链接失败、CMake、Kconfig、menuconfig、map文件、内存布局 | 读取 `build-debug/SKILL.md` 执行 |
| **样例生成** | 写个示例、怎么用 xxx、demo、sample、使用方法 | 读取 `sample-gen/SKILL.md` 执行 |
| **代码审查** | review、审查、检查代码、这段代码有问题吗、PR | 读取 `code-review/SKILL.md` 执行 |
| **组件开发** | 新组件、新模块、evt_pub、work_queue、封装、抽象层 | 读取 `component-dev/SKILL.md` 执行 |
| **文档编写/修复** | 写文档、README、客户反馈在线文档错误、构建 warning、toctree、文档发布 | 读取 `doc-writing/SKILL.md` 执行 |
| **自我进化** | 更新知识、优化 agent、回顾改进、进化 | 读取 `evolution/SKILL.md` 执行 |

**判断规则：**
- 优先匹配精确关键词
- 关键词不明确时，结合用户打开的文件、当前分支名、git diff 上下文判断
- 多领域交叉时（如"写个驱动的 sample"），确定主要意图后组合调用
- 无法判断时，向用户确认

### Step 3: 路由执行

根据意图分类，有三种执行策略：

#### 策略 A: 单 Skill 直达
意图明确且单一时，直接调用对应 skill：
```
用户: "帮我给 SPI Flash 写个驱动"
→ 读取 driver-dev/SKILL.md，按其流程执行
```

#### 策略 B: 多 Skill 编排
需要多个 skill 协作时，按依赖顺序执行：
```
用户: "新建一个温度传感器驱动并写个使用示例"
→ 先读取 driver-dev/SKILL.md 生成驱动
→ 再读取 sample-gen/SKILL.md 基于新驱动生成示例
```

#### 策略 C: Agent 直接处理
问题简单且不需要完整 skill 流程时，Agent 直接回答：
```
用户: "lisa_device_get 返回 NULL 是什么原因"
→ 直接基于知识库回答，不调用 skill
```

### Step 4: 质量检查

执行完成后，对输出进行检查：

- **代码输出**: 是否符合 SDK 编码约定（LOG_TAG、错误码、头文件保护）
- **CMake 输出**: 是否使用了正确的 listenai 宏
- **Kconfig 输出**: 依赖关系是否完整
- **驱动输出**: API vtable 是否与 inline 包装一致

### Step 5: 反馈收集

判断本次交互是否产生了值得记录的新知识：

**触发记录的条件（满足任一）：**
- 遇到了知识库中未记录的问题模式
- 用户纠正了 Agent 的输出
- 发现了 SDK 的新约定或变更
- 解决了一个复杂的构建/链接问题

**记录方式：**
- 更新对应的 knowledge 知识文件
- 在 `knowledge/evolution-log.md` 中追加记录
- 如果是 skill 层面的改进需求，标记为 evolution 待处理项

## SDK 快速参考（内嵌）

### 设备注册
```c
LISA_DEVICE_REGISTER(name, &api, &priv_data, NULL, init_fn, LISA_DEVICE_PRIORITY_NORMAL);
```

### 日志使用
```c
#define LOG_TAG "my_module"
#include <lisa_log.h>
// LOGI("xxx"), LOGW("xxx"), LOGE("xxx"), LOGD("xxx")
```

### CMake 常用宏
```cmake
listenai_library_named(module_xxx)
listenai_library_sources(src/xxx.c)
listenai_library_sources_ifdef(CONFIG_XXX src/xxx.c)
listenai_add_executable(${PROJECT_NAME})
listenai_add_custom_section(${CMAKE_CURRENT_SOURCE_DIR}/xxx.ld.in)
```

### 错误码
```
LISA_DEVICE_OK(0) / ERR_INVALID(-1) / ERR_NOT_FOUND(-2) / ERR_EXISTS(-3)
ERR_NO_MEM(-4) / ERR_INIT_FAIL(-5) / ERR_NOT_SUPPORT(-6) / ERR_TIMEOUT(-7)
ERR_BUSY(-8) / ERR_NOT_READY(-9) / ERR_IO(-10) / ERR_RANGE(-11)
ERR_OVERFLOW(-12) / ERR_NACK(-13)
```

## 注意事项

- 始终优先参考 codebase 中的现有实现，而非凭记忆生成
- 生成代码前先用 Glob/Grep 检索相似的现有代码作为参考
- 不确定 SDK API 的用法时，先读取对应头文件
- 回答中引用具体的文件路径和行号
- 回答中可引用在线文档链接，方便用户深入查看
- 在线文档提供使用说明和最佳实践，codebase 提供实现细节，两者互补
