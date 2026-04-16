# reqloop-lite — 轻量验收模式

## 目标场景

企业外 / 开源项目 / 单仓库团队：没有 qianliu-ipd / qianliu-gitlab / qianliu-ci / qianliu-aitest 这些内部工具，但同样需要 AI 做需求验收闭环。

触发方式：`/reqloop <id> --lite` 或 `/reqloop lite <id>`。

## 与完整版的差异

| 能力 | 完整版 | Lite |
|------|--------|------|
| 需求来源 | qianliu-ipd / qianliu-or 自动拉取 | 人工贴 PR 描述 + 可选 PRD 文件路径 |
| 代码采集 | qianliu-gitlab 多仓库 | 当前 git 仓库 + `git diff` / PR base |
| CI 触发 | qianliu-ci 指定环境 | 本地 `make test` / `go test` / 用户自定义命令 |
| e2e 回归 | qianliu-aitest / orbitest | **跳过或人工执行** |
| 缺陷回写 | qianliu-td | 生成 Markdown 清单 + 可选 `gh issue create` |
| 依赖图 | code-review-graph（企业镜像）| `go list -json` / tree-sitter 简化版 |

核心能力**不降级**：反讲 + 硬门禁 + 判定三元组 + 决策链 + 负向影响分析 全部保留。

## Lite 版输入契约

### Zero-config 智能推断（首选）

Lite 模式优先自动推断所有输入，**仅在推断失败时才交互询问**：

| 输入项 | 推断策略 | 推断失败时 |
|--------|---------|-----------|
| 需求描述 | ① `gh pr view --json body` 提取当前 PR body ② 最近 N 条 commit message 拼接 ③ 检查 `.github/ISSUE_TEMPLATE` | 询问用户粘贴 |
| 代码范围 | ① 检测当前分支是否有 upstream → `git diff main..HEAD` ② 检测 `gh pr view` → PR diff | 询问用户选择 |
| 验证命令 | ① `Makefile` 含 `test` target → `make test` ② `go.mod` 存在 → `go test ./... && go vet ./...` ③ `package.json` 含 `test` script → `npm test` ④ `Cargo.toml` → `cargo test` | 询问用户输入 |
| e2e 测试 | 默认跳过（结论最高"有条件通过"）| — |

**推断结果须在执行前展示给用户确认**（一次性确认，不逐条问）：

```
/reqloop PR-42 --lite

AI: Lite 模式 — 自动推断的输入如下，回车确认或修改：
  需求描述: [PR #42 body 摘要前 3 行...]
  代码范围: git diff main..HEAD (12 files changed)
  验证命令: go test ./... && go vet ./... && golangci-lint run
  e2e: 跳过
  > 确认 (Y/n) 或输入修改项编号:
```

### 手动模式（完整交互）

用户可通过 `--interactive` 强制逐项交互：

```
/reqloop <id> --lite --interactive

AI: Lite 模式，请提供：
  1. 需求描述（直接粘贴 PR 描述 / issue 正文 / PRD 路径）：
  2. 代码范围：
     - [a] 当前分支 vs main 的 diff
     - [b] 指定 commit 范围: <from>..<to>
     - [c] 指定 PR 号（用 gh pr view）
  3. 验证命令（可多条，逐条执行）：
     默认: go test ./... && go vet ./... && golangci-lint run
  4. e2e 测试：
     - [a] 跳过
     - [b] 人工执行后我会贴结果
     - [c] 本地脚本: <path>
```

## 阶段行为差异

### 阶段 1-2 采集（合并为一步）

- `.reqloop/req-{id}.md` 由用户提供的文本 + git log 组合生成
- `.reqloop/code-{id}/` 由 `git diff --name-only base..HEAD` 填充
- **跨模块 vs 跨仓库**（重要区分）：
  - **Monorepo / Go workspace / Nx workspace**：单仓多模块，Lite **完全支持**
    - Go workspace：`go list -m` 枚举模块 → 按模块归档 diff → 模块间 impact analysis 正常执行
    - Nx/Turborepo：`nx affected` 或 `turbo run --filter` 确定受影响模块
    - 通用 monorepo：按目录前缀拆分 diff，每个顶级目录视为一个逻辑模块
  - **真正的跨仓库**（不同 git remote）：Lite 不支持，提示升级到完整版

### 阶段 3 反讲（不变）

EARS schema + 硬门禁 + 安全例外全部保留。
**这是 Lite 版的核心价值** —— 即使没有企业工具链，反讲门禁依然生效。

### 阶段 4 review（降级但保留核心）

- 不依赖 code-review-graph；改用 `go list -deps` + 简易 importer 生成一阶调用关系
- 跨语言项目降级为"仅分析同语言边界"，标注 `partial_graph: true`
- 判定三元组、决策链照常产出

### 阶段 4b 影响分析（降级）

- 深度限制降为 2（完整版是 3）
- **Monorepo 模块间影响**：正常分析（Go workspace 用 `go list -deps`，其他用目录级 import 分析）
- **跨仓库影响**（不同 git remote）：标记为 `unknown`，提示人工

### 阶段 5 runtime（本地执行）

- AI 执行用户提供的验证命令
- 覆盖矩阵照常生成
- 失败日志写入 `.reqloop/runtime-{id}.md`，不需要 CI 平台

### 阶段 6 e2e（可选）

若用户选 [a] 跳过：矩阵中 e2e 列固定为 "skipped"，结论降级最高"有条件通过"。

### 阶段 7 report（调整回写方式）

- 不调用 qianliu-td
- 若仓库是 GitHub 且有 `gh` CLI，可生成 `gh issue create` 命令清单，用户确认后执行
- 否则只输出本地 Markdown 缺陷清单

### 阶段 8 export（不变）

OSLC / ReqIF / CSV 导出与完整版一致 —— Lite 用户最需要这个能力回流到 Jira/Spira。

## 依赖清单（最小集）

**必需**：
- `git`（diff / log）
- 语言对应的 test runner（用户提供命令即可）

**可选增强**：
- `gh` CLI —— GitHub PR 采集与 issue 回写
- `tree-sitter` —— 多语言反向调用链
- `go list` —— Go 项目依赖分析
- `ripgrep` —— 代码锚点定位加速

## 开源分发路径

Lite 版适合作为独立 skill 发布到 agentskills.io：
- 包：`reqloop-lite`（不含 qianliu / 内部工具引用）
- 入口：`SKILL.md` + `command.md` + `stages/`
- 文档强调"不依赖企业 ALM，也能做 AI 需求验收闭环"

这是 ai-toolkit 目前对外最有差异化的素材 —— 业界 Spec Kit / Kiro 都做"正向"，没有等价的反向验收开源工具。
