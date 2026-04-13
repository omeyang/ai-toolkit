---
name: reqloop-lite
description: 轻量版需求自验收 — 反向 spec-driven 验收（code → spec 裁决），不依赖企业 ALM / CI / 内部工具。仅需 git + 语言 test runner。保留核心能力：EARS 结构化反讲 + 人工 confirm 硬门禁 + 判定三元组 + 决策链 + 负向影响分析 + 安全例外。适合开源项目、单仓库团队、外部协作场景。触发：用户提供 PR 描述 / issue 正文希望做需求验收，或 `/reqloop-lite <id>`。
version: 0.2.0
---

# reqloop-lite — 轻量需求自验收

## 定位

reqloop 完整版需要 qianliu-ipd / qianliu-gitlab / qianliu-ci / qianliu-aitest 等企业内部工具。
本 skill 剥离这些依赖，**核心能力不降级**，适合：

- 开源项目的 PR 验收
- 小团队 / 单仓库场景
- 无 ALM 系统（Spira / DOORS / Jama）的企业
- 与 GitHub Issues / PR 一体化工作流

## 核心能力（与完整版一致）

1. **EARS 结构化反讲** + 人工 confirm 硬门禁
2. **三源冲突裁决**（默认代码为准；安全关键词走 PRD 为准例外）
3. **判定三元组**：verdict + confidence + evidence
4. **决策链 decisions.jsonl**（append-only + inputs_hash，支持重放）
5. **负向影响分析**（反向调用链 + 隐式行为变更阻塞）

## 降级部分

| 能力 | Lite 实现 |
|------|---------|
| 需求来源 | PR 描述 / issue 正文 / PRD 文件路径（用户提供）|
| 代码采集 | 当前 git 仓库 `git diff base..HEAD` 或 `gh pr view` |
| CI 触发 | 本地 `go test` / `make test` / 用户命令 |
| e2e 回归 | 跳过或人工贴结果 |
| 缺陷回写 | `gh issue create` 或本地 Markdown 清单 |
| 依赖图 | `go list -deps` / tree-sitter（深度 ≤2）|
| 跨仓库影响 | 标记 `unknown`，提示升级到完整版 |

## 依赖

**必需**：`git`
**推荐**：`gh` CLI、语言对应 test runner、ripgrep
**可选**：tree-sitter（多语言依赖图）、`go list`（Go 项目）

## 使用

```
/reqloop-lite <id-or-PR>
```

首次调用交互获取：需求文本、代码范围、验证命令、e2e 策略。详见主文档 `../reqloop/LITE.md`。

## 与完整版的关系

本 skill 复用 `../reqloop/` 的流程定义、阶段文件、模板。**不重复维护**：

- 主流程：`../reqloop/WORKFLOW.md`（本版执行时忽略 qianliu 相关段落）
- 阶段指令：`../reqloop/stages/*.md`（按 LITE.md 的降级规则执行）
- 轻量模式约定：`../reqloop/LITE.md`
- 决策链契约：`../reqloop/stages/DECISIONS.md`

执行时**先读 `../reqloop/LITE.md`**，再读 `../reqloop/WORKFLOW.md`，遇到企业工具引用时按 LITE.md 的映射表替换为本地实现。

## 相对业界的差异化

- 正向 SDD（Spec Kit / Kiro / Tessl）全部"spec → code"，无反向验收开源工具
- 本 skill 是**目前唯一能跑在开源项目上的反向 SDD 验收闭环**
- 填补业界公认空白：推理级 traceability（decisions.jsonl）、隐式行为变更检测
