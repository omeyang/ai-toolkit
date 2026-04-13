# 决策链产物 (decisions.jsonl)

## 目的

填补业界 SDD 工具（Spec Kit / Kiro / Tessl）的共同空白：**推理级 traceability**。
Spec Kit 能告诉你"哪条需求对应哪段代码"，但说不清"AI 为什么这样判定"。
反向验收天然同时持有"代码事实"和"需求意图"，有条件产出完整决策链。

## 位置

`.reqloop/decisions-{id}.jsonl` —— 每行一个 JSON 对象，append-only，不回填不改写。

## 字段契约

```json
{
  "ts": "2026-04-13T10:23:45Z",
  "stage": "backspec | review | runtime | e2e | report",
  "decision_id": "D-0001",
  "req_id": "REQ-001",
  "finding_id": "F-001",
  "code_anchor": { "repo": "repo-a", "file": "pkg/foo/bar.go", "lines": "42-50", "commit": "abc1234" },
  "verdict": "covered | partial | missing | conflict | violates | compliant",
  "confidence": "high | medium | low",
  "reasoning": "一句话说明判定链路",
  "inputs_hash": {
    "req_text": "sha256:...",
    "code_diff": "sha256:...",
    "prd_section": "sha256:..."
  },
  "model": "claude-opus-4-6",
  "human_override": null
}
```

## 写入规则

- 阶段 3、4、5、6 的**每一条**判定都必须追加一行
- `inputs_hash` 让未来可以重放：若需求文本/代码哈希变化，决策自动作废
- 人工在 confirm 或后续 review 阶段可以 override，写入 `human_override: { by, at, new_verdict, reason }`，**不覆盖原行，追加新行**

## 消费场景

1. **阶段 7 最终报告**：统计每阶段 verdict/confidence 分布
2. **需求漂移审计**：跨迭代对比同一 REQ-ID 的决策历史
3. **AI 判定回溯**：出现争议时查 reasoning + inputs_hash，而非"AI 当时怎么想的"
4. **回写需求单**：`qianliu-td` 缺陷附决策链 hash，形成签名链

## 禁止事项

- AI 不得删除或改写已写入行（仅允许 append）
- 未填 `inputs_hash` 的行视为无效决策
- `confidence: low` 的决策不得单独支撑"通过"结论
