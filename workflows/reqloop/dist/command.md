---
description: 需求自验收闭环 — 按 IPD/OR 需求 ID 执行反讲与验收全流程
argument-hint: <需求ID|子命令> [--prd <路径>] [--env <环境>] [--from <阶段号>] [--lite]
---

执行 reqloop 需求自验收工作流。

## 调用形式

### 全量执行（推荐）

```
/reqloop <需求ID> [--prd <路径>] [--env <环境>] [--from <阶段号>] [--lite]
```

- `<需求ID>`：`IPD-xxx` 或 `OR-xxx`（必填）
- `--prd <路径>`：附加本地 PRD 文档
- `--env <环境>`：指定阶段 5 运行时校验环境
- `--from <N>`：从第 N 阶段重跑（断点续跑，如 `--from 4` 跳过已 confirm 的反讲）
- `--lite`：走轻量模式（见 `LITE.md`），跳过 qianliu 全家桶依赖

### 子命令（对齐 Spec Kit / Kiro 心智）

与 GitHub Spec Kit 的 `specify → plan → tasks → implement` 形成**反向镜像**，
降低熟悉 SDD 的用户学习成本：

| 子命令 | 对应阶段 | Spec Kit 对照 | 用途 |
|--------|---------|--------------|------|
| `/reqloop gather <id>` | 1 + 2 | — | 拉需求单 + 代码 |
| `/reqloop backspec <id>` | 3 | 反向的 `/specify` | 反讲 + 人工 confirm |
| `/reqloop review <id>` | 4 + 4.5 | 反向的 `/plan` | 代码审查 + 影响分析 |
| `/reqloop verify <id>` | 5 + 6 | 反向的 `/tasks + /implement` | 运行时 + e2e |
| `/reqloop report <id>` | 7 | — | 汇总报告 + 回写缺陷 |
| `/reqloop export <id>` | 8 | — | 导出 OSLC/ReqIF |

每个子命令可独立重跑，产物互不覆盖。

## 执行规则

调用 reqloop skill，按 `WORKFLOW.md` 与 `stages/*.md` 顺序执行。严格遵守：

1. **阶段 3 人工 confirm 硬门禁**，未签字不进阶段 4
2. **三源冲突默认以代码为准**；命中安全关键词时走**安全例外**（PRD 为准 + security-auditor）
3. **判定三元组强制**：所有 AI 判定必须输出 `verdict + confidence + evidence`
4. **低置信度不独立结论**：`confidence: low` 路由到人工裁决段
5. **决策链 append-only**：每条判定写 `.reqloop/decisions-{id}.jsonl`，不得改写
6. **阶段 4.5 隐式行为变更阻塞**：未关联需求的调用链变更须人工确认
7. **阶段 5 不下"符合业务"结论**，只出覆盖矩阵
8. **Review 不跨反讲范围**，超出的入 out-of-scope 不影响结论

## 最终交付格式

```
reqloop {id} — {通过 / 有条件通过 / 不通过}
- Critical: N  运行时失败: N  e2e 失败: N  未闭环: N
- 隐式行为变更: N  低置信度待裁决: N  安全例外: {yes|no}
- 报告: .reqloop/acceptance-{id}.md
- 决策链: .reqloop/decisions-{id}.jsonl ({N} 条判定)
- 缺陷: TD-xxx, TD-yyy
- 后续: [1-3 条建议]
```
