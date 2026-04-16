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
| `/reqloop review <id>` | 4a + 4b | 反向的 `/plan` | 代码审查 + 影响分析 |
| `/reqloop verify <id>` | 5 + 6 | 反向的 `/tasks + /implement` | 运行时 + e2e |
| `/reqloop report <id>` | 7 | — | 汇总报告 + 回写缺陷 |
| `/reqloop resolve <id>` | 7+ | — | 有条件通过→通过升级 |
| `/reqloop revalidate <id>` | 增量 | — | 需求/代码变更后增量重验 |
| `/reqloop batch <id1,id2,...>` | 全量 | — | 多需求并行验收 + 聚合报告 |
| `/reqloop stats [--dir <path>]` | — | — | 历史决策链度量统计 |
| `/reqloop export <id>` | 8 | — | 导出 OSLC/ReqIF |

每个子命令可独立重跑，产物互不覆盖。

## 执行规则

调用 reqloop skill，按 `WORKFLOW.md` 与 `stages/*.md` 顺序执行。

> **7 条硬约束详见 `WORKFLOW.md` §核心设计取舍**，此处不重复。
> 速查：①阶段 3 硬门禁 ②三源冲突不调和 ③三元组强制 ④决策链 append-only ⑤阶段 4b 隐式变更阻塞 ⑥阶段 5 不下业务结论 ⑦不跨范围 review

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
