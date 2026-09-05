# fullstack-bridge `feat/introspection-selection` 回归审查报告

- 审查日期：2026-09-05
- 审查对象：`~/project/fullstack-bridge` @ `feat/introspection-selection`
- 基线：上轮 tip `978cd75` → 本轮 tip `b655c96`（1 fix 提交）
- 修复范围：上轮 S1（内省面==落参面）、S2（注册但 schema 坏的错误面）；S3 落为注记 + 能力层对齐
- 状态：**S1/S2 关闭、无阻塞、无建议残留**；53 测试 + ruff 全绿、实测一致

## 验证记录（副作用已还原）

- **测试**：`uv run pytest` → **53 passed**（上轮 51 + S1/S2 各新增 1：params==generate 选项键集、registered-but-broken 明确报错）
- **ruff**：All checks passed
- **实测一致**：`show-combo python-react --json` params 16 键（已剔 project_name/project_title）+ `internal: ['project_name','project_title']` + `derived: ['child_apps']`；`generate python-react --help` 选项 16（剔 skip-tasks/--help）——两键集相等，内省面==落参面
- 工作区 git clean

## 上轮问题处置对拍

| 上轮 | 处置 | 复核结论 |
|---|---|---|
| S1 show-combo params 含 internal 身份参数 | show-combo 剔除 `project_name/project_title` 进 `params`，另加 `internal` 标注；新增测试断言 `params` 键集 == generate 可接受选项键集 | ✅ 关闭。实测两键集一致（16==16）；MCP 落参不会给出 generate 拒绝的键 |
| S2 坏 schema 的注册 combo 误报 no-such-command | `get_command` 区分：真未注册 → `None`（no-such-command）；已注册但 schema 构建失败 → `click.UsageError`（「已注册但 schema 构建失败」） | ✅ 关闭。两类错误面语义分离；测试覆盖未注册仍 None、已注册坏 schema 明确报错且不含 "No such command" |
| S3 SELECTION_FIELDS 双份/静默丢字段 | combos.py 补**单一真源注记**（指向协议 SCHEMA）；功能侧按结论留**能力层对齐**（系列待办已登记 bridge-mcp-server P3） | ✅ 按建议闭环。注记消除「不知会漂」的盲点；功能对齐待协议 SCHEMA 单一真源化时一并做 |

## 本轮新观察（无遗留）

- S2 语义变化与 B2 初衷不冲突：惰性组仍只建**目标** combo（兄弟坏 combo 不拖死 CLI），仅对「用户显式请求的坏 combo」从静默降级改为明确报错——错误面向 MCP 更诚实。

## 结论

**上轮 S1/S2 修复到位并经实测确认，S3 按建议落为注记 + 跨仓待办，具备合入 main 条件。** 无阻塞、无建议残留。
