# 架构

`ai-toolkit` 采用分层、可复用的构建块组织：

1. `prompts/`、`policies/` 定义行为约束。
2. `skills/`、`hooks/`、`integrations/` 提供能力。
3. `mcp/`、`agents/` 封装运行时组件。
4. `workflows/` 编排多步骤执行流程。
5. `evaluations/` 验证质量和回归安全性。
6. `examples/` 展示实际组合方式。

设计目标：

- 跨框架兼容性
- 模块间最小耦合
- 组件可测试、可版本化
