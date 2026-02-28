---
name: code-review
description: ARCS SDK 代码审查。对照 SDK 编码约定检查代码风格、错误处理、线程安全、内存安全、API 设计规范性。当用户提交代码需要 review 或检查时触发。
---

# ARCS 代码审查

按 ARCS SDK 规范对代码进行审查。

## 触发条件

- 用户要求审查代码
- 用户提交了 PR 需要 review
- 用户问"这段代码有没有问题"

## 审查维度

1. **SDK 约定**: LOG_TAG、错误码、头文件保护、命名规范
2. **API 设计**: vtable 完整性、inline 包装正确性、参数校验
3. **线程安全**: ISR 中是否使用了非安全 API、共享资源保护
4. **内存安全**: 缓冲区边界、DMA 对齐、内存泄漏
5. **构建集成**: CMake 条件编译、Kconfig 依赖完整性

## 状态: STUB — 待 Phase 3 完整实现
