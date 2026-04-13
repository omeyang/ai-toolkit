---
name: reqloop
description: 需求自验收闭环 — 以多 skill 编排完成"需求 → 代码 → 反讲 → 验收"。区别于普通 code review：先对齐需求再审代码；以代码为准（安全例外除外），但回写需求。支持 EARS 结构化反讲 + 判定三元组 + 决策链 + 负向影响分析。当用户要求"需求自验收"、"反讲需求"、"验收 IPD/OR 需求"、或提供需求 ID（IPD-xxx / OR-xxx）希望做闭环验收时调用。支持 `--lite` 轻量模式（不依赖企业 ALM）。
version: 0.2.0
---

# reqloop — 需求自验收闭环

## 能做什么

按流水线完成需求自验收：

1. **gather** — 拉需求单（可选附 PRD）
2. **collect-code** — 拉多仓库 MR / commit / diff
3. **backspec** — 三源反推生成 EARS 结构化反讲文档 ★ 人工 confirm 硬门禁
4. **review** — 全局依赖视角 + 三元组判定（verdict + confidence + evidence）
5. **impact-radius**（4.5）— 反向调用链 + 隐式行为变更阻塞
6. **runtime** — 指定环境 build/test/lint + 业务边界覆盖矩阵
7. **e2e** — 端到端回归
8. **report** — 建缺陷 + 最终验收报告
9. **export**（可选）— 导出 OSLC / ReqIF / CSV / Xray，对接 DOORS / Spira / Jama / Jira

## 触发方式

- 用户说"自验收 IPD-xxx"、"反讲这个需求"、"验收 OR-xxx"、"/reqloop <id>" 时调用
- 子命令：`/reqloop {gather|backspec|review|verify|report|export} <id>`
- 轻量模式：`/reqloop <id> --lite`（无 qianliu / CI / ALM 依赖）

## 执行约束（硬性，违反即中止）

1. **阶段 3 未 confirm 严禁进入阶段 4**
   - 校验 `.reqloop/confirmed-{id}.md` 中 `confirmed: true` + `confirmed_by` 非空
   - 反讲 §1 YAML 通过 schema 校验（status 枚举、covered 必须有 code_anchors、low confidence 必须有 reason）
   - 所有 diff hunks 至少被一条 requirement 的 code_anchors 引用
   - 所有 🔴🟡🟠 差异均有处置决定
2. **三源冲突不得调和** — 默认代码为准；命中安全关键词（鉴权/审计/限流/PII/加密/幂等/注入）走 **安全例外**：PRD 为准 + security-auditor 介入 + 阻塞验收
3. **判定三元组强制** — 所有 AI 判定输出 `verdict + confidence + evidence`，`confidence: low` 不独立结论
4. **决策链 append-only** — 每条判定写 `.reqloop/decisions-{id}.jsonl`，含 `inputs_hash`，不得改写已有行
5. **阶段 4.5 隐式行为变更阻塞** — 未关联 REQ-ID 的反向调用链变更须人工确认
6. **阶段 5 不下"符合业务"结论** — 只出覆盖矩阵，❌/❓ 即标未闭环
7. **不跨范围 review** — 超出反讲的入 `out-of-scope`，不影响结论

## 详细指令

执行时**必须先读 `WORKFLOW.md`**，再按阶段读 `stages/*.md`：

- `WORKFLOW.md` — 主流程与核心取舍
- `stages/01-gather.md` ~ `stages/07-report.md` — 主流水线
- `stages/04.5-impact-radius.md` — 负向影响分析
- `stages/08-export.md` — OSLC/ReqIF 导出（可选）
- `stages/DECISIONS.md` — 决策链产物契约（跨阶段共用）
- `templates/backspec.md` — EARS 反讲模板
- `templates/acceptance-report.md` — 最终报告模板
- `LITE.md` — 轻量模式差异说明

## 产物约定

所有产物写入执行时工作目录下 `.reqloop/`，按需求 ID 隔离：

```
.reqloop/
├── req-{id}.md
├── code-{id}/
├── backspec-{id}.md           # EARS 结构化
├── confirmed-{id}.md          # 人工签字
├── review-{id}.md             # 三元组 findings
├── impact-{id}.md             # 负向影响
├── runtime-{id}.md
├── e2e-{id}.md
├── decisions-{id}.jsonl       # 决策链（append-only）
├── security-{id}.md           # 安全例外触发时
├── export/                    # OSLC/ReqIF/CSV（可选）
└── acceptance-{id}.md
```

断点续跑：再次调起时扫描已存在产物，从最后完成阶段的下一阶段开始。

## 相对业界工具的差异化

- **Spec Kit / Kiro / Tessl** 是正向 SDD（spec → code）；reqloop 是**反向验收**（code → spec 裁决）
- 填补业界公认空白：**推理级 traceability**（decisions.jsonl + inputs_hash）
- **隐式行为变更检测**：正向 SDD 工具无法发现，反向验收天然产出
