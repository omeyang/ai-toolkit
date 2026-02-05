# skills

用于代理编码工作流的可复用技能包。

## 当前状态

- 已从本地 Claude 安装迁移基线技能包。
- 完整列表请参阅 `skills/CATALOG.md`。

## 建议的模块结构

```text
skills/<skill-name>/
├── SKILL.md
├── scripts/
├── assets/
└── references/
```

## 规则

- 每个目录一个技能包。
- `SKILL.md` 为必需文件。
- 仅包含工作流所需的引用。
- 保持前置字段（`name`、`description`、`user-invocable`）一致。
