# 阶段 2 — 代码采集 (collect-code)

## 目标

拉取与需求关联的**所有**真实代码变更，作为反讲的**事实侧**输入源 B。
跨仓库是常态，不能遗漏。

## 输入

- 需求 ID
- （可选）关联 MR / commit 列表（若需求单未关联）

## 步骤

1. **从需求单反查关联 MR**
   - 阶段 1 产物中通常带有关联 MR 链接 / commit hash
   - 若无，向用户询问关联的仓库和分支 / MR 列表

2. **多仓库 diff 采集**
   - 调用 `qianliu-gitlab` 逐个仓库拉取：
     - MR 元信息（标题、描述、reviewer、状态）
     - 完整 diff（含新增文件全文，不只是 hunk）
     - commit 列表与每条 commit message

3. **归档结构**

   ```
   .reqloop/code-{id}/
   ├── INDEX.md              # 仓库 × MR 列表总览
   ├── repo-a/
   │   ├── mr-123.diff
   │   ├── mr-123.meta.md
   │   └── commits.md
   └── repo-b/
       └── ...
   ```

4. **INDEX.md 必含字段**
   - 仓库名 / MR 号 / 标题 / 状态 / 变更行数 / 关联需求 ID
   - 便于阶段 3 反讲时统一引用

## 取舍

- **只拉增量 diff，不拉全仓库**（阶段 4 由 code-review-graph 补全局视角）
- **新增文件例外**：新增文件必须拉全文，否则 review 时上下文不足
- **不跨需求采集**：即使 MR 里混入了其它需求，也按该 MR 整体归档，阶段 3 反讲时标注

## 失败处理

- 仓库访问失败 → 记录到 `INDEX.md` 并中止（缺代码不能反讲）
- MR 未合入 → 警告但继续（按 MR 当前 diff 处理，在 backspec 中标注状态）
