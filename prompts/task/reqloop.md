# 需求自验收任务（reqloop）

按 `workflows/reqloop/WORKFLOW.md` 执行「需求自验收闭环」。
完整阶段定义与取舍已在 WORKFLOW.md 中，本文件只作为任务入口。

## 调用方式

```
/reqloop <需求ID> [--prd <路径>] [--env <环境>]
```

- `<需求ID>`：`IPD-xxx` 或 `OR-xxx`，必填
- `--prd`：可选，附加 PRD 文档路径
- `--env`：可选，指定运行时校验环境（默认询问用户）

## 执行顺序（严格遵守 WORKFLOW.md）

1. **gather** — `qianliu-ipd` / `qianliu-or` 拉需求（可选附 PRD）
2. **collect-code** — `qianliu-gitlab` 多仓库拉 MR / diff
3. **backspec** — 三源反推生成反讲文档 ★ **HARD GATE**：等待 `.reqloop/confirmed-{id}.md` 人工签字
4. **review** — `code-review-graph` 补全局 + `code-reviewer` Agent 审增量
5. **runtime** — `qianliu-ci` 指定环境 build/test/lint + 业务边界覆盖矩阵
6. **e2e** — `qianliu-aitest` / `orbitest` 回归
7. **report** — 汇总 + `qianliu-td` 建缺陷 + 产出 `.reqloop/acceptance-{id}.md`

## 强约束（违反即中止）

1. **阶段 3 未 confirm 严禁进入阶段 4**
   - 读取 `.reqloop/confirmed-{id}.md`
   - 校验 `confirmed: true` 且 `confirmed_by` 非空
   - 校验反讲文档所有 🔴🟡🟠 差异均有处置决定
   - 任一不满足 → 明确告知用户"已中止，等待人工 confirm"，**不继续**

2. **三源冲突不得调和**
   - 代码 ↔ 需求单 ↔ PRD 有差异时，默认以代码为准
   - 在反讲文档中标红列出，**不自行判断哪方正确**
   - 裁决权完全交给开发人工

3. **阶段 5 不下"符合业务"结论**
   - 只输出业务边界覆盖矩阵（场景 × 代码 × 测试 × 数据 × 环境）
   - 有 ❌ 或 ❓ 即标"未闭环"，不擅自放行

4. **不跨范围 review**
   - 阶段 4 的 review 仅限反讲已 confirm 的变更
   - 超范围建议归入 `out-of-scope` 段，不计入验收结论

## 产物约定

所有产物写入执行时工作目录下 `.reqloop/`，按需求 ID 隔离。
详细结构见 WORKFLOW.md「产物目录约定」章节。

## 断点续跑

当用户再次执行 `/reqloop <需求ID>`：

1. 扫描 `.reqloop/` 下已存在的产物文件
2. 从最后一个完成的阶段的**下一阶段**开始
3. 若停在阶段 3 门禁，检查 confirm 状态决定放行或继续等待
4. 用户可显式指定从某阶段重跑：`/reqloop <需求ID> --from 4`

## 交付总结格式（最终给用户）

简洁，不重复报告内容，只给导航：

```
reqloop {id} — {结论}

- Critical: N  运行时失败: N  e2e 失败: N  未闭环: N
- 报告: .reqloop/acceptance-{id}.md
- 缺陷: TD-xxx, TD-yyy
- 后续: [1-3 条建议]
```
