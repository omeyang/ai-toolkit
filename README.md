# ai-toolkit

AI 构建块统一仓库：技能包、MCP 服务器、代理、钩子、工作流、策略、提示词、评估和集成。

## 范围

- `skills/`：可复用技能包（SKILL.md、脚本、资源）
- `mcp/`：MCP 服务器模块、配置、适配器和示例
- `agents/`：代理定义和可复用的角色/剧本规范
- `hooks/`：事件钩子（前置/后置动作、验证、护栏）
- `workflows/`：多步骤编排流程和自动化管道
- `policies/`：安全、权限、合规和治理规则
- `prompts/`：可复用提示词模板和组件
- `evaluations/`：测试集、基准测试、回归检查和评分
- `integrations/`：外部系统连接器（GitHub、Slack 等）
- `examples/`：端到端参考项目
- `docs/`：架构说明、规范和路线图

## 仓库结构

```text
ai-toolkit/
├── agents/
├── docs/
├── evaluations/
├── examples/
├── hooks/
├── integrations/
├── mcp/
├── policies/
├── prompts/
├── skills/
└── workflows/
```

## 快速开始

1. 将模块添加或迁移到相应的顶级目录
2. 在该目录（或模块）中添加 `README.md`，说明用法和约束
3. 在 `examples/` 或 `evaluations/` 下添加至少一个可运行或可测试的示例

## 迁移快照

- 初始 Claude 技能包迁移已完成，位于 `skills/`
- 当前清单跟踪在 `skills/CATALOG.md`

## 原则

- 保持模块小而可组合
- 优先显式契约而非隐式行为
- 每个可复用组件都应包含使用示例
