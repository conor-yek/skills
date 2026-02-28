# SDK Assistant Agent 开发计划

> 贯穿整个项目的开发蓝图，跨会话持久化进度追踪。

## 项目总览

**目标**: 构建一个专业的 ARCS SDK 开发助手 Agent，能够：
- 自动识别开发意图并路由到专业 skill
- 基于深度 SDK 知识生成高质量代码
- 持续自我进化，积累经验知识
- 覆盖驱动开发、构建调试、样例生成、代码审查、组件开发全流程

**交付物**: `.claude/skills/sdk-assistant-agent/` 下的完整 skill 体系 + `knowledge/` 下的知识库

---

## Phase 分解

### Phase 1: 基础框架搭建 ✅ 已完成

**目标**: 建立 Agent 骨架和知识基线

**已完成的工作**:
- [x] 统一入口 `SKILL.md` — 意图识别 + 路由逻辑 + 质量检查 + 反馈收集
- [x] 进化机制 `evolution/SKILL.md` — 知识审计 + 变更检测 + 更新流程
- [x] 5 个子 skill stub: driver-dev, build-debug, sample-gen, code-review, component-dev
- [x] 知识库初始化（已迁移至 `knowledge/` 统一管理）:
  - `knowledge/sdk-patterns.md` — 6 大设计模式
  - `knowledge/driver-patterns.md` — 驱动模板与陷阱
  - `knowledge/build-issues.md` — 构建问题速查
  - `knowledge/evolution-log.md` — 进化记录
  - `knowledge/sdk-architecture.md` — 架构全景图 + 启动流程 + 构建系统
  - `knowledge/conventions.md` — 编码约定速查
  - `memory/MEMORY.md` — 轻量索引（auto memory，自动加载）

**产出版本**: v0.1.0 (2026-02-26)

---

### Phase 2: 核心 Skill 实现 🔄 进行中

**目标**: 将 driver-dev 和 build-debug 两个高频 skill 从 stub 升级为完整实现

#### 2.1 driver-dev 完整实现
- [ ] 深入分析 3-5 个现有驱动的实际代码，提取精确模板
  - 参考目标: `lisa_uart`, `lisa_spi`, `lisa_i2c`, `lisa_flash`, `lisa_display`
  - 关注: 实际的 API 签名、私有数据结构、初始化流程、中断处理
- [ ] 完善 SKILL.md 中的代码生成模板
  - 头文件完整模板（含 C++ extern "C" 保护）
  - 平台适配层完整模板（含私有数据、中断处理、DMA 模式）
  - 针对不同设备类型的模板变体（传感器 / 通信 / 存储 / 显示）
- [ ] 添加交互式需求收集流程
  - 设备类型选择 → 通信方式 → 功能列表 → 回调需求
- [ ] 添加生成后自动验证 checklist
- [ ] 用一个实际驱动验证完整流程

#### 2.2 build-debug 完整实现 ✅ 已完成 (2026-02-27)
- [x] 建立错误模式库（从 codebase 和构建系统中提取）
  - CMake 阶段错误分类与诊断树（含 listenai-cmake-config.cmake 实际错误）
  - 编译错误诊断树
  - 链接错误诊断树（含链接脚本 ASSERT、ABI 不匹配、relocation 错误）
  - Kconfig 依赖问题诊断（含 Kconfig 优先级说明）
- [x] 完善诊断流程
  - 自动从错误输出中提取关键信息（SKILL.md 错误分析树）
  - 匹配已知模式 → 给出解决方案
  - 未知模式 → 引导式排查流程
- [x] 添加内存布局分析能力
  - ARCS 芯片完整内存区域表（ROM/RAM/ILM/DLM/PSRAM/WIFI_RAM 等）
  - CONFIG_PRINT_MEMORY_USAGE 使用指引
  - map 文件解读指引（grep 命令 + .symb 文件）
  - 自定义段 .ld.in 模板与说明
  - 代码/数据放置属性速查表
- [x] 集成 build-issues.md 的自动更新机制（Step 4 知识沉淀流程）

#### 2.3 sample-gen 完整实现 ✅ 已完成 (2026-02-27)
- [x] 分析现有 samples 的结构模式
  - 驱动类 sample 模式（lisa_gpio, lisa_uart, lisa_spi 等）
  - 网络类 sample 模式（http, wifi）
  - 蓝牙类 sample 模式（ble_peripheral）
  - 模块类 sample 模式（cjson, sqlite3 等）
- [x] 完善代码生成模板
  - 3 种 CMakeLists.txt 模板（单源/多源/子目录+库）
  - prj.conf 推断规则表（按驱动/模块/网络/BLE 分类）
  - main.c 模板（含版权头、LOG_TAG、设备获取、pinmux 覆盖、错误处理）
  - Kconfig 固定模板（osource）
  - sample.yaml 模板（含 test key 命名规范）
- [x] 添加 README 生成（遵循 Devices_Samples_Spec.md 规范，含基础/高级两种模板）
- [x] 完整的 5 步流程：需求确认→参考样例→生成文件→质量检查→输出说明

---

### Phase 3: 扩展 Skill 实现 📋 计划中

**目标**: 完成 code-review 和 component-dev skill

#### 3.1 code-review 完整实现
- [ ] 定义审查规则集（基于 SDK 约定）
  - 命名规范检查
  - 错误处理完整性
  - 线程安全审查（ISR 安全、锁使用）
  - 内存安全审查（缓冲区边界、DMA 对齐）
  - API 设计规范性（vtable 完整性、inline 包装）
- [ ] 设计审查报告格式
- [ ] 支持对 git diff / PR 的增量审查

#### 3.2 component-dev 完整实现
- [ ] 分析现有组件的结构模式
  - 参考: lisa_os, lisa_log, lisa_evt_pub, lisa_kv, work_queue
- [ ] 完善组件骨架生成模板
- [ ] 添加 evt_pub 集成指引
- [ ] 支持组件间依赖分析

---

### Phase 4: 集成优化 📋 计划中

**目标**: 提升 Agent 整体质量和使用体验

- [ ] 路由规则调优（基于实际使用数据）
- [ ] 多 Skill 编排优化（驱动+样例、组件+审查等组合场景）
- [ ] 知识库完整性审计和补充
- [ ] 进化机制实战验证和调优
- [ ] 用户反馈闭环机制完善

---

## 设计决策记录

### DD-001: Agent 架构 — 统一入口 + 子 Skill 路由

**决策**: 使用单一入口 SKILL.md 做意图识别和路由，子 skill 作为内部指令文件（不独立注册到系统）。

**原因**:
- 避免用户需要记忆多个 skill 名称
- 统一入口可以做跨 skill 编排
- 子 skill 之间可以共享知识上下文

**影响**: 所有子 skill 通过 `读取 xxx/SKILL.md 执行` 的方式被调用，而非系统级 skill 注册。

### DD-002: 知识存储 — knowledge/ 统一目录 + memory/ 轻量索引

**决策** (v2, 2026-02-27 更新): 所有详细知识文件统一放在工程内 `knowledge/` 目录，`memory/MEMORY.md` 仅作为轻量索引指向工程目录。
- `knowledge/` — 所有知识文件（sdk-patterns, driver-patterns, build-issues, evolution-log, sdk-architecture, conventions）
- `memory/MEMORY.md` — 精简索引，自动加载到每次对话上下文

**原因**:
- 工程内文件可 git 管理和团队共享，auto memory 不在 git 中
- memory/ 受 MEMORY.md 200 行限制，仅适合做索引
- 演进更新时只需修改工程内文件，路径统一

### DD-003: 进化机制 — 独立 Skill + 记录日志

**决策**: 进化机制作为独立 skill，使用 evolution-log.md 记录每次变更。

**原因**:
- 进化是显式触发的（非自动），需要用户确认
- 日志记录便于追溯知识变更历史
- 独立 skill 可以有完整的诊断→验证→更新流程

### DD-005: Skill 设计原则 — LLM 程序而非人类文档

**决策**: 所有 skill 文件遵循 `SKILL_DESIGN_PRINCIPLES.md` 中定义的 9 条原则。

**核心认知**: Skill 文件的读者是 LLM，写法更接近代码而非说明书：
- 指令映射到具体工具调用（Glob/Grep/WebFetch/Read）
- 用活的数据源（WebFetch + codebase Grep）替代静态知识
- 决策树替代散文指引，checklist 项可机器验证

**影响**: 编写新 skill 或迭代现有 skill 前需阅读 `SKILL_DESIGN_PRINCIPLES.md`。

---

### DD-004: Skill 实现顺序 — 按使用频率优先

**决策**: Phase 2 优先实现 driver-dev、build-debug、sample-gen；Phase 3 实现 code-review、component-dev。

**原因**:
- 驱动开发和构建调试是嵌入式开发最高频的场景
- 样例生成与驱动开发强关联，可复用 Phase 2 的知识积累
- code-review 和 component-dev 相对独立，可延后

---

## 当前进度

| 阶段 | 状态 | 完成度 |
|------|------|--------|
| Phase 1: 基础框架 | ✅ 完成 | 100% |
| Phase 2: 核心 Skill | 🔄 进行中 | 67% (build-debug ✅, sample-gen ✅) |
| Phase 3: 扩展 Skill | 📋 计划中 | 0% |
| Phase 4: 集成优化 | 📋 计划中 | 0% |

**当前焦点**: Phase 2 — build-debug ✅, sample-gen ✅，下一步: driver-dev 完整实现

---

## 下一步行动

### 即刻启动: Phase 2 规划

1. **driver-dev 深度分析** — 读取 3-5 个现有驱动的完整源码，提取精确的代码模式
2. **build-debug 错误库** — 分析构建系统，建立错误诊断树
3. **sample-gen 模式提取** — 分析现有 samples 结构

### Phase 2 执行策略

建议按以下顺序逐个完成：
1. driver-dev（最复杂，需要最多 codebase 分析）
2. build-debug（可复用 driver-dev 分析中积累的构建知识）
3. sample-gen（可复用前两者的模板和模式知识）

每个 skill 完成后通过实际使用场景验证，发现的问题反馈到进化机制。
