# Symphony 项目说明与 Harness Engineering 的意义

## 项目概述

**Symphony** 是一个长期运行的自动化编排服务，它能够持续读取项目管理工具（如 Linear）中的工作项，为每个任务创建隔离的工作空间，并在工作空间内运行自主的编码代理（coding agent）会话。

简单来说，Symphony 让团队可以**管理工作**而不是**监督编码代理**。

## 核心功能

Symphony 解决了四个运维问题：

1. **标准化执行流程**：将任务执行转变为可重复的守护进程工作流，而不是手动脚本
2. **工作空间隔离**：在每个任务专属的工作空间中隔离代理执行，确保代理命令只在任务目录内运行
3. **配置版本化**：将工作流策略保存在仓库中（`WORKFLOW.md`），使团队可以将代理提示词和运行时设置与代码一起版本控制
4. **可观测性**：提供足够的可观测性来操作和调试多个并发代理运行

## 工作原理

1. **轮询 Linear** 获取待处理工作
2. **为每个任务创建工作空间**
3. **在工作空间内启动 Codex**（使用 App Server 模式）
4. **向 Codex 发送工作流提示**
5. **持续让 Codex 处理任务，直到工作完成**

如果任务移动到终止状态（`Done`、`Closed`、`Cancelled` 或 `Duplicate`），Symphony 会停止该任务的活动代理并清理对应的工作空间。

## 与 Harness Engineering 的关系

### 什么是 Harness Engineering？

Harness Engineering 是 OpenAI 提出的一种工程实践方法，旨在让代码库更适合与 AI 编码代理协作。详见：https://openai.com/index/harness-engineering/

### Symphony 的定位

README 中明确指出：

> Symphony works best in codebases that have adopted harness engineering. Symphony is the next step -- moving from managing coding agents to managing work that needs to get done.

**Symphony 是 Harness Engineering 的下一步演进**：

- **Harness Engineering 阶段**：让代码库适配 AI 代理，学习如何管理编码代理
- **Symphony 阶段**：从管理编码代理升级到管理需要完成的工作本身

## 对 Harness Engineering 的意义

### 1. 自动化工作流编排

传统的 Harness Engineering 实践中，开发者需要：
- 手动启动编码代理
- 监督代理执行
- 管理工作空间
- 处理代理状态

Symphony 将这些操作自动化，让团队可以：
- 在 Linear 中规划工作
- 让 Symphony 自动分配和执行任务
- 通过仪表板监控进度
- 专注于高层次的工作管理而非低层次的代理监督

### 2. 标准化的执行环境

Symphony 提供：
- **隔离的工作空间**：每个任务在独立的目录中执行，避免相互干扰
- **版本化的工作流**：`WORKFLOW.md` 文件定义了代理的行为和配置
- **可重复的执行**：相同的任务配置产生一致的执行结果

### 3. 企业级运维能力

Symphony 实现了生产级别的特性：
- **并发控制**：可配置的并发代理数量限制
- **重试机制**：支持指数退避的失败重试
- **状态协调**：自动同步任务状态和代理执行状态
- **可观测性**：结构化日志、实时仪表板、API 接口

### 4. 团队协作模式的演进

使用 Symphony 后，团队的工作方式发生根本变化：

**传统模式**：
```
开发者 → 手动运行代理 → 监督执行 → 提交代码 → 代码审查
```

**Symphony 模式**：
```
产品经理/团队 → 在 Linear 创建任务 → Symphony 自动执行 → 代理完成工作并提供证明 → 人工审查结果 → 批准合并
```

在演示视频中可以看到，Symphony 监控 Linear 看板，自动生成代理来处理任务。代理完成任务后提供工作证明：CI 状态、PR 审查反馈、复杂度分析和演示视频。批准后，代理安全地合并 PR。

## 技术架构特点

### 语言选择：Elixir/OTP

Symphony 选择 Elixir 作为实现语言，原因包括：
- **Erlang/BEAM/OTP** 擅长监督长期运行的进程
- 丰富的生态系统工具和库
- 支持热代码重载，无需停止活动的子代理，对开发非常有用

### 核心组件

1. **Workflow Loader**：读取和解析 `WORKFLOW.md`
2. **Issue Tracker Client**：与 Linear 集成，获取任务
3. **Orchestrator**：拥有轮询循环和运行时状态
4. **Workspace Manager**：管理每个任务的工作空间
5. **Agent Runner**：启动和管理编码代理会话
6. **Status Surface**（可选）：提供人类可读的状态信息

## 实际应用价值

### 对于已采用 Harness Engineering 的团队

1. **提高效率**：从手动监督到自动化编排，释放开发者时间
2. **标准化流程**：统一的工作流定义，减少团队间的差异
3. **更好的可见性**：实时仪表板展示所有进行中的工作
4. **风险控制**：工作空间隔离和配置化的安全策略

### 对于计划采用 Harness Engineering 的团队

Symphony 提供了一个**参考实现**和**规范**（`SPEC.md`），团队可以：
- 学习如何构建代理编排系统
- 了解需要考虑的技术问题
- 根据自己的需求实现定制版本

README 中建议：
```
Tell your favorite coding agent to build Symphony in a programming language of your choice:
> Implement Symphony according to the following spec:
> https://github.com/openai/symphony/blob/main/SPEC.md
```

## 安全与信任边界

Symphony 明确定义了信任边界：
- 这是一个**低调的工程预览版**，用于在受信任环境中测试
- 支持配置化的审批策略和沙箱模式
- 工作空间路径验证和隔离是基本的安全控制

## 总结

Symphony 对 Harness Engineering 的意义在于：

1. **实践验证**：证明了 AI 代理可以在生产环境中自主完成端到端的开发工作
2. **范式转变**：从"人工监督 AI 编码"到"人工管理工作，AI 自主执行"
3. **工程化框架**：提供了一套完整的、可参考的工程化实现
4. **生产就绪**：展示了如何将 AI 代理集成到真实的软件开发流程中

Symphony 不仅仅是一个工具，它代表了软件工程团队与 AI 协作的新模式。在这个模式中，人类专注于定义"做什么"（工作需求），而 AI 负责"怎么做"（具体实现），两者通过标准化的接口和工作流进行协作。

这正是 Harness Engineering 实践的终极目标：**让 AI 成为团队中可靠的、自主的工程师，而不仅仅是需要持续监督的工具**。
