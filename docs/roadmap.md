# 路线图

## 第一阶段：基础建设 (Done)

- [x] 建立目录契约和命名规范
- [x] 将现有技能包迁移到 `skills/`（18 个技能）
- [x] 架构文档和贡献指南

## 第二阶段：运行时 (Done)

- [x] Hook 脚本（go-format, go-lint, go-test-async, block-dangerous, session-context, commit-lint）
- [x] Hook 配置模板（完整版 + 最小版）
- [x] MCP 服务器配置模板（K8s, MongoDB, ClickHouse, Redis, Kafka, OTel, GitHub）
- [x] Agent 定义（golang-pro, code-reviewer, k8s-devops, db-specialist, security-auditor）
- [x] 工作流定义（TDD, Code Review, Deploy）
- [x] Prompt 模板（CLAUDE.md 模板, 任务提示词, 代码片段）

## 第三阶段：可靠性

- [ ] 添加评估套件和回归检查
- [ ] 添加 CI 结构和文档验证
- [ ] Hook 脚本单元测试
- [ ] 端到端示例项目

## 第四阶段：扩展

- [ ] Policies 模块（安全策略、权限边界）
- [ ] Integrations 模块（GitHub Actions、Slack 通知）
- [ ] 更多技能包（前端、DevOps、SRE）
- [ ] 多语言支持（Python、Rust）
