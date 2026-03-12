# 如何使用 Symphony

本指南将详细介绍如何设置和使用 Symphony 来自动化您的开发工作流程。

## 目录

- [前置要求](#前置要求)
- [快速开始](#快速开始)
- [详细配置](#详细配置)
- [工作流程](#工作流程)
- [常见问题](#常见问题)

## 前置要求

### 1. 代码库准备

Symphony 在已采用 [Harness Engineering](https://openai.com/index/harness-engineering/) 实践的代码库中效果最好。确保您的代码库具备：

- **完善的文档**：清晰的 README、架构文档、API 文档
- **自动化测试**：完整的测试套件（单元测试、集成测试）
- **CI/CD 流程**：自动化构建和测试流程
- **明确的架构约束**：清晰的代码组织和设计模式

### 2. 工具和账号

您需要准备以下工具和账号：

1. **Linear 账号**：用于任务跟踪
   - 在 Linear 中创建或加入一个团队
   - 创建一个项目用于管理任务
   - 获取 Linear API 密钥：Settings → Security & access → Personal API keys

2. **OpenAI Codex**：编码代理
   - 确保已安装 `codex` CLI 工具
   - 配置好 Codex 认证

3. **运行环境**（选择一种）：
   - **选项 A**：使用 Elixir 参考实现（推荐用于快速试用）
   - **选项 B**：根据 SPEC.md 实现您自己的版本

## 快速开始

### 选项 1：使用 Elixir 参考实现

#### 步骤 1：安装依赖

推荐使用 [mise](https://mise.jdx.dev/) 来管理 Elixir/Erlang 版本：

```bash
# 克隆 Symphony 仓库
git clone https://github.com/openai/symphony
cd symphony/elixir

# 安装 mise（如果还没有安装）
curl https://mise.run | sh

# 信任并安装项目依赖
mise trust
mise install

# 验证 Elixir 安装
mise exec -- elixir --version
```

#### 步骤 2：配置环境变量

```bash
# 设置 Linear API 密钥
export LINEAR_API_KEY="lin_api_your_linear_api_key_here"
```

#### 步骤 3：配置工作流

复制示例工作流文件并根据您的需求进行配置：

```bash
# 复制工作流模板到您的项目目录
cp WORKFLOW.md ~/my-project/WORKFLOW.md

# 编辑配置
vim ~/my-project/WORKFLOW.md
```

最小化配置示例：

```markdown
---
tracker:
  kind: linear
  project_slug: "your-project-slug"  # 从 Linear 项目 URL 获取
  active_states:
    - Todo
    - In Progress
  terminal_states:
    - Done
    - Closed
    - Cancelled

workspace:
  root: ~/code/symphony-workspaces  # 工作空间根目录

hooks:
  after_create: |
    git clone git@github.com:your-org/your-repo.git .
    # 添加其他初始化命令，如依赖安装

agent:
  max_concurrent_agents: 10  # 最大并发代理数
  max_turns: 20              # 单次运行的最大轮次

codex:
  command: codex app-server
---

You are working on a Linear issue {{ issue.identifier }}.

Title: {{ issue.title }}
Description: {{ issue.description }}

[在这里添加您的具体工作流程指令...]
```

#### 步骤 4：构建并运行

```bash
cd symphony/elixir

# 设置并构建项目
mise exec -- mix setup
mise exec -- mix build

# 启动 Symphony
mise exec -- ./bin/symphony ~/my-project/WORKFLOW.md
```

可选参数：

```bash
# 自定义日志目录
./bin/symphony ~/my-project/WORKFLOW.md --logs-root ./custom-logs

# 启用 Web 仪表板
./bin/symphony ~/my-project/WORKFLOW.md --port 4000
```

### 选项 2：自定义实现

如果您想用其他编程语言实现 Symphony，可以参考规范：

```bash
# 让您的编码代理根据规范实现
# 提供以下提示给代理：
"根据以下规范实现 Symphony：
https://github.com/openai/symphony/blob/main/SPEC.md"
```

## 详细配置

### WORKFLOW.md 配置详解

`WORKFLOW.md` 文件包含两部分：

1. **YAML 前置内容**：配置参数
2. **Markdown 正文**：发送给 Codex 的提示模板

#### tracker 配置

```yaml
tracker:
  kind: linear                    # 问题跟踪系统类型（目前仅支持 linear）
  api_key: $LINEAR_API_KEY       # API 密钥（使用环境变量）
  project_slug: "your-slug"      # Linear 项目标识
  active_states:                  # 活跃状态列表
    - Todo
    - In Progress
    - Human Review
    - Merging
    - Rework
  terminal_states:                # 终止状态列表
    - Done
    - Closed
    - Cancelled
    - Duplicate
```

**获取项目 slug**：
- 在 Linear 中右键点击项目
- 复制 URL
- slug 是 URL 中的项目标识符

#### workspace 配置

```yaml
workspace:
  root: ~/code/symphony-workspaces  # 工作空间根目录
  # 支持路径展开：
  # - ~ 会被展开为用户主目录
  # - $VAR 会从环境变量读取
```

每个问题会在此目录下创建独立的工作空间：
```
~/code/symphony-workspaces/
  ├── PROJECT-1/     # 问题 PROJECT-1 的工作空间
  ├── PROJECT-2/     # 问题 PROJECT-2 的工作空间
  └── ...
```

#### hooks 配置

生命周期钩子允许您在工作空间的不同阶段执行自定义脚本：

```yaml
hooks:
  after_create: |
    # 工作空间创建后执行
    git clone --depth 1 git@github.com:your-org/your-repo.git .
    npm install
    # 或其他初始化命令

  before_remove: |
    # 工作空间删除前执行
    # 可用于清理、备份等
```

**使用 mise 的示例**：

```yaml
hooks:
  after_create: |
    git clone --depth 1 https://github.com/your-org/your-repo.git .
    if command -v mise >/dev/null 2>&1; then
      mise trust
      mise install
      mise exec -- npm install
    fi
```

#### agent 配置

```yaml
agent:
  max_concurrent_agents: 10  # 最大并发运行的代理数量
  max_turns: 20              # 单个代理会话的最大轮次
```

- `max_concurrent_agents`：控制同时运行的代理数量，避免资源过载
- `max_turns`：防止单个任务陷入无限循环

#### codex 配置

```yaml
codex:
  command: codex app-server                    # Codex 启动命令
  approval_policy: never                       # 审批策略
  thread_sandbox: workspace-write              # 线程沙箱模式
  turn_sandbox_policy:                         # 每轮沙箱策略
    type: workspaceWrite
```

**审批策略选项**：
- `never`：不需要审批（高信任环境）
- `on-request`：代理请求时需要审批
- `on-failure`：失败时需要审批
- `untrusted`：所有操作都需要审批
- 对象形式：`{"reject": {"sandbox_approval": true, "rules": true, "mcp_elicitations": true}}`

**沙箱模式**：
- `read-only`：只读访问
- `workspace-write`：可以写入工作空间（推荐）
- `danger-full-access`：完全访问（不推荐）

**高级命令配置**：

```yaml
codex:
  command: |
    codex
      --config shell_environment_policy.inherit=all
      --config model_reasoning_effort=xhigh
      --model gpt-5.3-codex
      app-server
```

#### polling 配置

```yaml
polling:
  interval_ms: 5000  # 轮询间隔（毫秒）
```

Symphony 会按此间隔检查 Linear 中的新任务或状态变化。

### 提示模板

YAML 配置后的 Markdown 正文是发送给 Codex 的提示模板。可以使用以下变量：

```markdown
---
# ... YAML 配置 ...
---

You are working on a Linear issue {{ issue.identifier }}.

## Issue Details

- **Identifier**: {{ issue.identifier }}
- **Title**: {{ issue.title }}
- **Status**: {{ issue.state }}
- **Labels**: {{ issue.labels }}
- **URL**: {{ issue.url }}

## Description

{{ issue.description }}

## Instructions

1. 分析问题描述和需求
2. 设计解决方案
3. 实现代码变更
4. 编写和运行测试
5. 创建 Pull Request
6. 等待审查

## Guidelines

- 遵循代码库的编码规范
- 确保所有测试通过
- 编写清晰的提交信息
- 在 PR 中提供详细的变更说明
```

**可用的模板变量**：
- `{{ issue.identifier }}`：问题 ID（如 PROJECT-123）
- `{{ issue.title }}`：问题标题
- `{{ issue.description }}`：问题描述
- `{{ issue.state }}`：当前状态
- `{{ issue.labels }}`：问题标签
- `{{ issue.url }}`：Linear 中的问题 URL
- `{{ attempt }}`：重试次数（从 1 开始）

**条件逻辑示例**：

```markdown
{% if attempt %}
这是第 {{ attempt }} 次尝试。请从当前工作空间状态继续，不要从头开始。
{% endif %}

{% if issue.description %}
{{ issue.description }}
{% else %}
没有提供描述。
{% endif %}
```

## 工作流程

### 典型使用流程

1. **在 Linear 中创建任务**
   - 创建新的 issue
   - 设置状态为 `Todo`
   - 添加详细的描述和需求

2. **Symphony 自动检测**
   - Symphony 定期轮询 Linear
   - 检测到新的 `Todo` 任务
   - 创建独立的工作空间

3. **代理自动执行**
   - 在工作空间中克隆代码库
   - 启动 Codex 代理
   - 发送工作流提示
   - 代理分析问题并实现解决方案

4. **代理提交工作**
   - 创建 git 分支
   - 提交代码变更
   - 创建 Pull Request
   - 更新 Linear 任务状态为 `Human Review`

5. **人工审查**
   - 工程师审查 PR
   - 提供反馈或批准
   - 如需修改，将任务状态改为 `Rework`

6. **合并或重做**
   - 批准后：任务状态改为 `Merging`，代理自动合并 PR
   - 需要修改：代理根据反馈重新工作
   - 完成后：任务状态改为 `Done`

### 状态流转图

```
Backlog (不处理)
    ↓
  Todo ──→ Symphony 检测
    ↓
In Progress ──→ 代理执行工作
    ↓
Human Review ──→ 等待人工审查
    ↓         ↙
  Merging   Rework ──→ 代理重做
    ↓         ↓
  Done ←──────┘
```

### 使用技能（Skills）

Symphony 提供了一组可选的技能，可以复制到您的项目中：

```bash
# 复制技能到您的项目
cp -r /path/to/symphony/.codex/skills ~/my-project/.codex/
```

可用的技能：

1. **commit**：创建规范的 git 提交
   ```
   使用会话历史创建良好格式的 git 提交
   ```

2. **push**：推送变更到远程仓库
   ```
   保持远程分支与本地同步
   ```

3. **pull**：拉取最新代码
   ```
   在开始工作前同步最新的 main 分支
   ```

4. **land**：安全合并 PR
   ```
   执行完整的合并流程，包括检查和清理
   ```

5. **linear**：与 Linear 交互
   ```
   更新任务状态、添加评论、上传文件等
   ```

6. **debug**：调试辅助
   ```
   帮助调试问题和错误
   ```

这些技能会在工作流提示中被引用，代理可以在需要时调用它们。

## 高级用法

### 使用 Web 仪表板

启动带有 Web 仪表板的 Symphony：

```bash
./bin/symphony ./WORKFLOW.md --port 4000
```

然后在浏览器中访问 `http://localhost:4000`，您可以：

- 查看所有活跃代理的实时状态
- 监控工作空间使用情况
- 查看日志和错误
- 手动触发刷新

API 端点：
- `GET /`：Web 仪表板
- `GET /api/v1/state`：获取完整状态
- `GET /api/v1/<issue_identifier>`：获取特定问题的状态
- `POST /api/v1/refresh`：手动触发刷新

### 自定义日志位置

```bash
./bin/symphony ./WORKFLOW.md --logs-root /var/log/symphony
```

日志结构：
```
/var/log/symphony/
  ├── symphony.log          # 主日志
  ├── PROJECT-1/            # 问题 PROJECT-1 的日志
  │   ├── agent.log
  │   └── workspace.log
  └── PROJECT-2/
      └── ...
```

### 使用环境变量

创建 `.env` 文件：

```bash
# .env
LINEAR_API_KEY=lin_api_xxx
SYMPHONY_WORKSPACE_ROOT=/home/dev/workspaces
SOURCE_REPO_URL=git@github.com:org/repo.git
CODEX_BIN=/usr/local/bin/codex
```

在 `WORKFLOW.md` 中引用：

```yaml
tracker:
  api_key: $LINEAR_API_KEY
workspace:
  root: $SYMPHONY_WORKSPACE_ROOT
hooks:
  after_create: |
    git clone --depth 1 "$SOURCE_REPO_URL" .
codex:
  command: "$CODEX_BIN app-server"
```

### 多项目管理

为不同项目创建不同的工作流文件：

```bash
# 项目 A
./bin/symphony ~/workflows/project-a.md --port 4000

# 项目 B（在另一个终端）
./bin/symphony ~/workflows/project-b.md --port 4001
```

### SSH 远程工作节点

Symphony 支持在远程机器上运行代理。配置示例：

```yaml
# WORKFLOW.md
worker:
  type: ssh
  hosts:
    - user@worker1.example.com
    - user@worker2.example.com
  ssh_key_path: ~/.ssh/symphony_workers
```

这样可以：
- 利用更强大的远程机器
- 分散工作负载
- 隔离不同项目的环境

## 最佳实践

### 1. 工作流设计

**明确的指令**：
- 提供清晰、具体的任务描述
- 定义明确的验收标准
- 说明测试要求

**分阶段执行**：
```markdown
## Phase 1: Analysis and Planning
1. 阅读相关代码
2. 设计解决方案
3. 创建任务计划

## Phase 2: Implementation
1. 实现核心功能
2. 编写测试
3. 验证功能

## Phase 3: Quality Assurance
1. 运行完整测试套件
2. 检查代码质量
3. 更新文档
```

### 2. 任务管理

**任务粒度**：
- 保持任务适度大小（1-4 小时的工作量）
- 过大的任务应拆分为多个子任务
- 每个任务应有单一、明确的目标

**依赖关系**：
- 在 Linear 中使用 `blocks` 关系标记依赖
- 确保依赖任务先完成

**标签使用**：
```
# 推荐的标签体系
- priority: high/medium/low
- type: feature/bug/refactor/docs
- area: frontend/backend/api/database
- complexity: simple/medium/complex
```

### 3. 代码库准备

**文档要求**：
```
your-repo/
  ├── README.md              # 项目概述和快速开始
  ├── CONTRIBUTING.md        # 贡献指南
  ├── docs/
  │   ├── architecture.md    # 架构设计
  │   ├── api.md            # API 文档
  │   └── development.md    # 开发指南
  └── .codex/
      ├── skills/           # 自定义技能
      └── AGENTS.md         # 代理专用文档
```

**AGENTS.md 示例**：
```markdown
# 代理开发指南

## 代码组织

- `src/`：源代码
- `tests/`：测试文件
- `config/`：配置文件

## 编码规范

- 使用 ESLint 和 Prettier
- 遵循 Airbnb 风格指南
- 所有函数必须有 JSDoc 注释

## 测试要求

- 单元测试覆盖率 > 80%
- 所有新功能必须有测试
- 运行测试：`npm test`

## 常见任务

### 添加新的 API 端点
1. 在 `src/routes/` 创建路由文件
2. 在 `src/controllers/` 创建控制器
3. 在 `tests/` 添加测试
4. 更新 API 文档

### 修复 Bug
1. 先编写复现测试
2. 修复代码
3. 确保测试通过
4. 检查是否影响其他功能
```

### 4. 监控和调试

**日志监控**：
```bash
# 实时查看主日志
tail -f log/symphony.log

# 查看特定问题的日志
tail -f log/PROJECT-123/agent.log

# 搜索错误
grep ERROR log/symphony.log
```

**状态检查**：
```bash
# 使用 API 检查状态
curl http://localhost:4000/api/v1/state | jq .

# 检查特定问题
curl http://localhost:4000/api/v1/PROJECT-123 | jq .
```

**调试代理**：
- 查看工作空间内容：`ls ~/code/symphony-workspaces/PROJECT-123/`
- 查看 git 历史：`cd ~/code/symphony-workspaces/PROJECT-123 && git log`
- 手动测试命令：进入工作空间并手动运行代理的命令

### 5. 安全考虑

**沙箱配置**：
```yaml
# 生产环境推荐配置
codex:
  approval_policy: on-request  # 需要人工批准
  thread_sandbox: workspace-write  # 限制写入范围
  turn_sandbox_policy:
    type: workspaceWrite
```

**敏感信息**：
- 使用环境变量存储密钥，不要硬编码
- 不要在 Linear 问题描述中包含敏感信息
- 定期轮换 API 密钥

**代码审查**：
- 始终人工审查代理生成的 PR
- 检查是否有意外的文件变更
- 验证测试覆盖率和质量

## 常见问题

### Q1: Symphony 无法启动

**问题**：运行 `./bin/symphony` 时报错

**解决方法**：
```bash
# 检查 Elixir 版本
mise exec -- elixir --version

# 重新安装依赖
mise exec -- mix deps.get
mise exec -- mix compile

# 检查 WORKFLOW.md 语法
mise exec -- mix workflow.validate ./WORKFLOW.md
```

### Q2: 代理没有检测到 Linear 任务

**问题**：任务在 Linear 中设置为 Todo，但 Symphony 没有处理

**可能原因和解决方法**：

1. **项目 slug 不正确**
   ```yaml
   # 确认 project_slug 正确
   tracker:
     project_slug: "your-correct-slug"  # 从 Linear URL 获取
   ```

2. **状态名称不匹配**
   ```yaml
   # 确认状态名称与 Linear 中完全一致
   tracker:
     active_states:
       - Todo  # 注意大小写
       - In Progress
   ```

3. **API 密钥无效**
   ```bash
   # 测试 API 密钥
   curl -H "Authorization: $LINEAR_API_KEY" \
     https://api.linear.app/graphql \
     -d '{"query": "{ viewer { id name } }"}'
   ```

### Q3: 代理执行失败

**问题**：代理开始工作但中途失败

**调试步骤**：

1. **查看代理日志**
   ```bash
   tail -f log/<issue-id>/agent.log
   ```

2. **检查工作空间状态**
   ```bash
   cd ~/code/symphony-workspaces/<issue-id>
   git status
   git log
   ```

3. **手动重现**
   ```bash
   # 进入工作空间
   cd ~/code/symphony-workspaces/<issue-id>

   # 手动运行代理会执行的命令
   npm test
   npm build
   ```

4. **检查钩子执行**
   ```bash
   # 查看钩子是否成功执行
   grep "after_create" log/<issue-id>/workspace.log
   ```

### Q4: 工作空间清理问题

**问题**：工作空间占用过多磁盘空间

**解决方法**：

```bash
# 手动清理终止任务的工作空间
cd ~/code/symphony-workspaces
find . -type d -name "PROJECT-*" -exec du -sh {} \;

# 删除特定工作空间
rm -rf ~/code/symphony-workspaces/PROJECT-123
```

在 `WORKFLOW.md` 中配置自动清理：
```yaml
hooks:
  before_remove: |
    # 清理大文件
    find . -name "node_modules" -type d -exec rm -rf {} +
    find . -name "*.log" -delete
```

### Q5: 如何处理需要人工干预的情况

**场景**：任务需要访问特殊权限或资源

**解决方法**：

1. **使用 `Human Review` 状态**
   ```markdown
   # 在提示中说明
   如果需要特殊权限（如生产数据库访问），请：
   1. 在 Linear 中添加评论说明需要什么权限
   2. 将任务状态改为 Human Review
   3. 等待人工提供权限后继续
   ```

2. **配置阻塞条件**
   ```markdown
   ## 阻塞条件

   如果遇到以下情况，请停止并标记为 Human Review：
   - 需要访问生产环境
   - 需要外部服务的 API 密钥
   - 需要修改基础设施配置
   ```

### Q6: 多个代理冲突

**问题**：多个代理同时修改同一文件导致冲突

**解决方法**：

1. **合理规划任务**
   - 避免多个任务同时修改相同文件
   - 使用 Linear 的依赖关系标记

2. **调整并发数**
   ```yaml
   agent:
     max_concurrent_agents: 3  # 降低并发数
   ```

3. **使用任务队列**
   - 将相关任务放在同一项目中
   - Symphony 会按顺序处理

### Q7: 如何自定义代理行为

**场景**：需要代理遵循特定的工作流程

**解决方法**：

在 `WORKFLOW.md` 的提示部分详细说明：

```markdown
---
# ... YAML 配置 ...
---

# 自定义工作流程

## 第一步：理解需求
1. 仔细阅读问题描述
2. 识别关键需求和约束
3. 列出需要修改的文件

## 第二步：设计方案
1. 设计技术方案
2. 考虑边界情况
3. 制定测试策略

## 第三步：实现
1. 创建功能分支：`git checkout -b feature/{{ issue.identifier }}`
2. 实现核心功能
3. 编写单元测试
4. 运行测试套件：`npm test`

## 第四步：质量检查
1. 运行 linter：`npm run lint`
2. 检查代码覆盖率
3. 手动测试关键路径

## 第五步：提交
1. 使用 commit 技能创建规范提交
2. 使用 push 技能推送到远程
3. 创建 PR 并链接到此问题
4. 将状态改为 Human Review

## 代码规范
- 使用 TypeScript 严格模式
- 所有函数必须有类型注解
- 遵循项目的 ESLint 规则
- 使用 Prettier 格式化代码

## 测试要求
- 新功能必须有单元测试
- 测试覆盖率不低于 80%
- 所有测试必须通过
- 包含集成测试（如适用）
```

### Q8: 性能优化

**问题**：Symphony 运行缓慢或资源占用高

**优化建议**：

1. **调整轮询间隔**
   ```yaml
   polling:
     interval_ms: 10000  # 从 5 秒增加到 10 秒
   ```

2. **限制并发代理**
   ```yaml
   agent:
     max_concurrent_agents: 5  # 根据机器资源调整
   ```

3. **优化工作空间**
   ```yaml
   hooks:
     after_create: |
       # 使用浅克隆节省时间和空间
       git clone --depth 1 --single-branch git@github.com:org/repo.git .
   ```

4. **使用本地缓存**
   ```yaml
   hooks:
     after_create: |
       git clone git@github.com:org/repo.git .
       # 使用共享的依赖缓存
       ln -s /shared/cache/node_modules ./node_modules
   ```

## 进阶主题

### 集成到 CI/CD

将 Symphony 集成到您的 CI/CD 流程：

```yaml
# .github/workflows/symphony.yml
name: Symphony
on:
  schedule:
    - cron: '*/5 * * * *'  # 每 5 分钟运行

jobs:
  symphony:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Elixir
        uses: erlef/setup-beam@v1
        with:
          elixir-version: '1.14'
          otp-version: '25'
      - name: Run Symphony
        env:
          LINEAR_API_KEY: ${{ secrets.LINEAR_API_KEY }}
        run: |
          cd elixir
          mix deps.get
          mix compile
          timeout 4m ./bin/symphony ../WORKFLOW.md || true
```

### 监控和告警

设置监控和告警系统：

```bash
# 使用 Prometheus 监控 Symphony
# 在 WORKFLOW.md 中启用指标端点
server:
  port: 4000
  metrics_enabled: true
```

配置 Prometheus：
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'symphony'
    static_configs:
      - targets: ['localhost:4000']
```

### 分布式部署

在多台机器上部署 Symphony：

```bash
# 机器 1：处理项目 A
./bin/symphony ~/workflows/project-a.md

# 机器 2：处理项目 B
./bin/symphony ~/workflows/project-b.md

# 机器 3：处理项目 C
./bin/symphony ~/workflows/project-c.md
```

或使用负载均衡：

```nginx
# nginx.conf
upstream symphony {
    server symphony1.internal:4000;
    server symphony2.internal:4000;
    server symphony3.internal:4000;
}
```

## 故障排查清单

遇到问题时，按照此清单逐项检查：

- [ ] Linear API 密钥是否有效
- [ ] WORKFLOW.md 语法是否正确
- [ ] 项目 slug 是否匹配
- [ ] 状态名称是否与 Linear 一致
- [ ] Codex 是否正确安装和配置
- [ ] 工作空间目录是否有写权限
- [ ] 网络连接是否正常
- [ ] 日志中是否有错误信息
- [ ] 磁盘空间是否充足
- [ ] 系统资源（CPU/内存）是否足够

## 获取帮助

如果您遇到问题或需要帮助：

1. **查看日志**：首先检查 Symphony 日志文件
2. **查阅文档**：阅读 [SPEC.md](https://github.com/openai/symphony/blob/main/SPEC.md) 获取详细规范
3. **社区支持**：在 GitHub Issues 中搜索类似问题
4. **提交问题**：如果是新问题，在 GitHub 上创建 Issue

## 总结

Symphony 通过自动化编排编码代理，使团队能够专注于管理工作而非监督代理。关键要点：

✅ **准备工作**：确保代码库已采用 Harness Engineering 实践
✅ **配置得当**：根据项目需求定制 WORKFLOW.md
✅ **任务清晰**：在 Linear 中提供明确的任务描述和验收标准
✅ **持续监控**：使用日志和仪表板监控代理活动
✅ **人工审查**：始终审查代理生成的代码
✅ **迭代改进**：根据使用经验不断优化工作流配置

开始您的 Symphony 之旅，让 AI 代理为您的团队工作！
