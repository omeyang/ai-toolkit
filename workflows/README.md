# workflows

可组合的编排流程，协调技能包、代理、钩子和 MCP。

## 目录结构

```
workflows/
├── README.md
├── tdd/                 # 测试驱动开发工作流
│   └── WORKFLOW.md
├── code-review/         # 代码审查工作流
│   └── WORKFLOW.md
└── deploy/              # Kubernetes 部署工作流
    └── WORKFLOW.md
```

## 工作流清单

| 工作流 | 用途 | 核心组件 |
|--------|------|---------|
| `tdd` | 测试驱动开发 | go-test Skill + golang-pro Agent + 格式化 Hook |
| `code-review` | 结构化代码审查 | code-reviewer Agent + security-auditor Agent |
| `deploy` | K8s 部署上线 | k8s-devops Agent + k8s-go Skill + MCP |

## 设计原则

- **声明式**: 用流程图描述步骤，而非命令式脚本
- **可观测**: 每个步骤有明确的输入/输出和成功/失败标准
- **可组合**: 工作流引用 Skills、Agents、Hooks、MCP，不重复实现
- **可回退**: 关键步骤有回滚方案

## 编写规范

每个 WORKFLOW.md 必须包含：

1. **前置条件** — 工具版本要求 + 组件依赖列表（Agent/Skill/Hook/MCP）
2. **适用场景** — 何时使用此工作流
3. **流程定义** — ASCII 流程图 + 步骤说明
4. **Agent 调用方式** — 展示如何通过 Task 工具调用子 Agent，包括：
   - 完整的 prompt 模板（含 AGENT.md 引用 + 任务描述 + 约束）
   - 多 Agent 串联或并行的示例
   - 不同场景的调用变体
5. **使用的组件** — 组件清单表

## 组件交互

```
Workflow (编排)
    ├── Agent (执行主体)
    │     └── Skill (领域知识)
    ├── Hook (自动化守护)
    │     ├── PreToolUse  (拦截)
    │     └── PostToolUse (反馈)
    └── MCP (外部工具)
          ├── Kubernetes
          ├── Database
          └── GitHub
```
