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
- [x] 统一入口 `SKILL.md` — 意图识别 + 路由逻辑
- [x] 进化机制 `evolution/SKILL.md` — 知识审计 + 变更检测 + 更新流程
- [x] 5 个子 skill stub: driver-dev, build-debug, sample-gen, code-review, component-dev
- [x] 知识库初始化（`knowledge/` 统一管理）:
  - `knowledge/sdk-patterns.md` — SDK 通用设计模式
  - `knowledge/driver-patterns.md` — 驱动模板
  - `knowledge/build-issues.md` — 构建问题速查
  - `knowledge/evolution-log.md` — 进化记录
  - `knowledge/sdk-architecture.md` — 架构全景图 + 启动流程 + 构建系统
  - `knowledge/conventions.md` — 编码约定速查
  - `knowledge/online-docs.md` — 在线文档 URL 索引

**产出版本**: v0.1.0 (2026-02-26)

### Phase 1.5: 框架重构 ✅ 已完成 (2026-03-02)

**目标**: 提升 skill 文件对 LLM 的友好度，明确文件职责

**已完成的工作**:
- [x] 主 `SKILL.md` 瘦身（143→47 行）：移除预加载、质量检查、反馈收集，精简为纯路由器
- [x] 新增 `references/index.md` — 系统全景图，ROUTER/WORKER/REFERENCE/META 角色分类
- [x] `driver-dev/` 展平：`lisa_xxx/SKILL.md` → `lisa_xxx.md`，移除 frontmatter 和静态知识
- [x] 驱动 worker 统一为"资源索引 + 执行指令"格式（19 行/文件），无内嵌 API 说明
- [x] 修复 `evolution/SKILL.md` 路径引用错误
- [x] 完善 `SKILL_DESIGN_PRINCIPLES.md`（原则六、七、八更新，自检清单分类）

**产出版本**: v0.2.0 (2026-03-02)

---

### Phase 2: 核心 Skill 实现 🔄 进行中

**目标**: 将 driver-dev 和 build-debug 两个高频 skill 从 stub 升级为完整实现

#### 2.1 driver-dev 完整实现 ✅ 已完成（方式调整，2026-03-02）

原计划内嵌代码模板和静态 API 说明，与 Phase 1.5 确立的设计原则冲突（原则二：用活数据源替代静态知识）。

**实际完成方式**:
- [x] 20 个设备 worker 统一采用"资源索引 + 并行加载指令"格式
- [x] 不内嵌 API 说明，改为 `Read 头文件 + WebFetch 在线文档 + Glob 示例目录`
- [x] 开发细节由 AI 结合资源自行判断，只确认无法推断的硬件参数（引脚、实例等）
- [x] 覆盖全部 20 种设备：GPIO/UART/SPI/I2C/Flash/ADC/PWM/HWTIMER/RTC/WDT/SDMMC/I2S/Display/Touch/Audio/QSPILCD/DVP/Camera/RGB/Pinmux

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

### DD-002: 知识存储 — knowledge/ 统一目录

**决策** (v3, 2026-03-02 更新): 所有知识文件统一放在工程内 `knowledge/` 目录，由各 worker 按需加载。
- `knowledge/` — 所有知识文件（sdk-patterns, driver-patterns, build-issues, evolution-log, sdk-architecture, conventions, online-docs）
- `references/index.md` — 系统全景图，描述所有文件角色和加载时机

**原因**:
- 工程内文件可 git 管理和团队共享
- `references/index.md` 作为轻量系统地图，替代旧的 `memory/MEMORY.md` 索引角色
- Worker 文件引用知识文件，不内嵌知识内容（原则七）

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
| Phase 1.5: 框架重构 | ✅ 完成 | 100% |
| Phase 2: 核心 Skill | ✅ 完成 | 100% (driver-dev ✅, build-debug ✅, sample-gen ✅) |
| Phase 3: 扩展 Skill | 🔄 进行中 | 0% |
| Phase 4: 集成优化 | 📋 计划中 | 0% |

**当前焦点**: Phase 3.1 — code-review 完整实现

---

## 下一步行动

### 即刻启动: Phase 3.1 code-review

实现要点（遵循设计原则）：
1. **审查范围识别** — 支持 git diff / 指定文件 / PR 三种输入方式
2. **知识加载** — 读取 `knowledge/conventions.md`，按代码类型按需加载 `driver-patterns.md` 或 `sdk-patterns.md`
3. **可验证 checklist** — 每条审查项均可通过 Grep 确认，不写"应该"类模糊项
4. **结构化报告** — 按维度输出问题列表 + 修改建议
