# fullstack-bridge `refactor/nunit-topology` 三轮审查报告（修复复核）

- 审查日期：2026-09-04
- 审查对象：`~/project/fullstack-bridge` @ `refactor/nunit-topology`
- 本轮基线：二轮 tip `75e0182` → 本轮 tip `9067443`（1 修复提交）
- 修复范围：R2/R3/N1/R5（R1/R4 按系列既定暂缓，series-overview 已登记）
- 状态：42 测试全绿；R2/R3/N1/R5 复核通过；新增 2 条建议 + 2 nit，无阻塞

## 验证记录

- `uv run pytest` → **42 passed**（上轮 39 + 新增 validate source/链形等 3 项）
- `check --combo python-react` 实测 → fetch 刷新 origin/HEAD 正常（成功静默），输出对齐、无 checkout 痕迹
- **星型拓扑探测**：3 单元 2 边但非链（`[[a,b],[a,c]]`）通过 `validate_combo`——「链形」校验只数边数、不校验连通（见 A1）
- README/onboarding 残留措辞扫描 → 见 N2

---

## 🟢 上轮问题处置对拍

| 编号 | 处置 | 复核结论 |
|---|---|---|
| R2 漂移参照过期 | check 入口 `_fetch_bases_for(targets)` 逐 source `fetch origin` 刷新 origin/HEAD（离线失败仅警告沿用本地 ref）；信号 ref（base_version）读取失败**显式报错不回退** | ✅ 修复。fetch 静默成功；信号路径错误不再被工作树兜底掩盖 |
| R3 onboarding:108 | 表述改「source 须为 bases 注册裸名；多端只认注册组合（Q4）」 | ✅ 修复，与代码/定案一致 |
| N1 README combos.yaml 行 | 改「units.{key}.{app,stack} README 元数据」 | ✅ 修复（其下 templates 行仍有残留，见 N2） |
| R5 source 注册 + 链形 | `validate_combo` 补 source ∈ bases 注册表 + edges 数 = units-1 | ✅ 基本修复；链形仅数边数、非充分（见 A1） |
| R1 / R4 | 未改（按计划暂缓，series-overview「审查暂缓项」已登记） | ⏸ 接受；随第二组合接入时落实 |

---

## 🟡 建议（三轮新发现）

### A1. 「链形」校验不充分：边数 = units-1 是必要条件非充分条件，星型拓扑会通过
`validate_combo` 只校验 `len(edges) == len(units) - 1`（combos.py）。实测 3 单元星型 `[[a,b],[a,c]]`（非链）通过校验。若「仅支持链形」是硬约束（注释如是说），应校验边构成**单一路径**（每节点入/出度约束 + 连通），否则星型会静默进入 merge_order/契约渲染（语义未定义）。
当前无多 edge 组合，不触发；建议接入 ui-bff-api 时一并收紧（或把约束改为「单路径」并补测试）。

### A2. 结构校验只挂在 generate 路径，`check`（含 merge gate）无结构门
`validate_all_combos` 仅在 `LazyGenerateGroup.invoke` 触发；`check`/`bridge-gate`（`check --all`）不校验 combos.yaml 形态。后果：缺 edges/非链等结构坏组合在 gate 上**静默放行**，未注册 source 则在 check 半途于 resolve_base SystemExit（提示可读但非前置校验）。二轮 R1 定案「整注册表校验留给 check」尚未落到 check。
建议：`check` 命令入口也跑一次 `validate_all_combos`（结构门在 gate 生效）；generate 侧按 R1 收敛为只校验目标 combo。

---

## ⚪ nit

- **N1** 离线 fetch 失败仅警告，随后仍可能输出「✅ 全部对齐」而 refs 已过期——结论旁可带「（基于本地 ref，可能过期）」提示。低。
- **N2** README:27 `templates/project-README/` 行仍写「技术栈来自 combos.yaml stack」——实际机制已改为 cli.py 从 `units.{key}.{stack}` 装配 `units_desc` 注入模板（无顶层 stack），措辞过时。

---

## 结论

**上轮 R2/R3/N1/R5 修复到位，42 测试全绿，无阻塞，具备合入条件。** 新增 A1/A2 两条建议与 N1/N2 两个 nit：A2（check 结构门）与 R1 同源，建议随 R1 一并落地；A1（链形连通性）在接入 ui-bff-api 前不影响任何现有组合，可与 R4 同批处理。
