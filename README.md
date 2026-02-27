# ai-toolkit

AI 构建块统一仓库：技能包、MCP 服务器、代理、钩子、工作流、策略、提示词、评估和集成。

## 范围

- `skills/`：可复用技能包（19 个 Go 全栈技能）
- `mcp/`：MCP 服务器配置模板（K8s/MongoDB/ClickHouse/Redis/Kafka/OTel/GitHub）
- `agents/`：代理定义（Go 开发/代码审查/K8s 运维/数据库/安全审计）
- `hooks/`：事件钩子脚本（格式化/lint/测试/安全拦截/上下文注入）
- `workflows/`：编排流程（TDD/Code Review/Deploy）
- `prompts/`：CLAUDE.md 模板和提示词片段
- `policies/`：安全、权限、合规和治理规则
- `evaluations/`：测试集、基准测试、回归检查和评分
- `integrations/`：外部系统连接器
- `examples/`：端到端参考项目
- `docs/`：架构说明、规范和路线图

## 仓库结构

```text
ai-toolkit/
├── skills/          # 19 个技能包 (Go 全栈)
├── hooks/           # 6 个 Hook 脚本 + 2 个配置模板
│   ├── scripts/     # go-format, go-lint, go-test-async, block-dangerous, session-context, commit-lint
│   └── configs/     # settings.json (完整/最小)
├── mcp/             # MCP 配置模板
│   ├── configs/     # go-backend-full, go-backend-minimal, observability
│   └── servers/     # MCP 服务器参考清单
├── agents/          # 5 个 Agent 定义
│   ├── golang-pro/
│   ├── code-reviewer/
│   ├── k8s-devops/
│   ├── db-specialist/
│   └── security-auditor/
├── workflows/       # 3 个工作流
│   ├── tdd/
│   ├── code-review/
│   └── deploy/
├── prompts/         # 提示词模板
│   ├── system/      # CLAUDE.md 模板
│   ├── task/        # 任务提示词 (审查/排查/实现)
│   └── snippets/    # 代码片段 (错误处理/并发)
├── policies/
├── evaluations/
├── integrations/
├── examples/
└── docs/
```

## 快速开始

### 为 Go 项目配置 Claude Code

```bash
# 1. 复制 Hook 脚本
mkdir -p .claude/hooks
cp ai-toolkit/hooks/scripts/*.sh .claude/hooks/
chmod +x .claude/hooks/*.sh

# 2. 配置 Hook
cp ai-toolkit/hooks/configs/settings.json .claude/settings.json

# 3. 配置 MCP 服务器
cp ai-toolkit/mcp/configs/go-backend-full.json .mcp.json

# 4. 生成 CLAUDE.md
cp ai-toolkit/prompts/system/CLAUDE.md.tmpl CLAUDE.md
# 编辑 CLAUDE.md，替换 {{PLACEHOLDER}} 为实际值
```

### 单独使用某个模块

每个模块都可独立使用，详见各模块的 README.md。

## 原则

- 保持模块小而可组合
- 优先显式契约而非隐式行为
- 每个可复用组件都应包含使用示例
