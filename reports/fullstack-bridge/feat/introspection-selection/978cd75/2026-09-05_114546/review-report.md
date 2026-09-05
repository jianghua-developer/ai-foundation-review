# fullstack-bridge `feat/introspection-selection` 审查报告

- 审查日期：2026-09-05
- 审查对象：`~/project/fullstack-bridge` @ `feat/introspection-selection`
- 基线：main（`409f5a4`，重构已合入 + U1 共享参数作用域定案）→ tip `978cd75`（1 feat 提交）
- 上游依赖：DESIGN 跨仓 #3（内省面 list-combos/show-combo + selection 合并）；底座 v2 已合入各底座 main（vite-react `535f3d2` / python-fastapi `3bf48bb`，本分支 combos.yaml 已钉并可达）
- 状态：**无阻塞**；2 建议 + 2 nit；51 测试全绿、内省面实测通过

## 验证记录（副作用已还原）

- **全量测试**：`uv run pytest` → **51 passed**（含新增 test_introspection 9 例）
- **内省面实测**：
  - `list-combos --json` → python-react 行含 units/edges + 合并 selection（react 底座 3 + fastapi 底座 3 + combo 段 1，去重、声明序）
  - `show-combo python-react --json` → 原生参数 18（含 child_apps_raw、不含 child_apps）+ derived `['child_apps']` + selection 两键
  - `check --combo python-react` → 对齐新钉版 `535f3d2`/`3bf48bb`，全绿
- **钉版可达**：两底座 v2 提交均已 push/合入各自 main；桥缓存 clone 含对应 sha（cat-file commit 命中），无次序问题
- **工作区**：桥仓 git clean（未留改动）

## 审查要点（正向）

- **selection 合并符合 DESIGN §8**：单一真源在底座 params.json `selection` 区，combo 段只叠组合不可约事实（python-react 加了「一体交付」一条）；并集去重、声明序、全空→None；`_unit_params_doc` 统一 frozen/source 读参并复用进 `param_schema`
- **combo 段 selection 结构门**：仅 `suited_for`/`tradeoffs` 且 `list[str]`，未知字段拒绝（防拼错漂移）——挂在 `validate_combo`，check/generate 共用
- **R1/A2 归属落地**：generate 组 invoke 只校验**目标** combo（不再全量，坏兄弟 combo 不阻断）；check 入口整注册表结构门（`validate_all_combos` 先于选目标、离线）→ bridge-gate `check --all` / check-drift 自动生效
- **内省面形态**：--json 结构化（ensure_ascii=False 保 CJK），human 摘要可读；show-combo 校验目标、派生只读分列（防 agent 误传派生）

---

## 🟡 建议

### S1. show-combo 原生参数基线含 internal 身份参数，与 generate 落参面不一致
`show-combo` 的 `params` = `param_schema` 原样（含 `project_name`/`project_title`，均为原生 derived:false）；而 `generate` 的选项在 `_INTERNAL_PARAMS`（cli.py）剔除这两个身份参数——`generate python-react --help` 无 `--project-name/--project-title`，实测确认。

影响：作为「落参前参数基线」（DESIGN §6.6 / MCP `get_combo_params`→`generate_multi`）消费时，内省面会给出 generate 拒绝的键——若 agent 照单传入 `project_name`，shell-out `generate` 因无该选项直接 UsageError。

建议：show-combo 的 params 面与 generate 可接受面同构（剔除/标注 `project_name`/`project_title` 为「内部派生，勿传」），或至少在 JSON 中给个 `askable`/internal 标注，避免内省面 ≠ 落参面。

### S2. 生成侧 schema 构建失败的「注册 combo」报错为 no-such-command，语义不符
R1 后 generate 对 schema 构建失败（缺 params.json / source 异常）的 combo 走 `get_command` catch → 返回 None → Click 报 `No such command '<combo>'`，尽管该 combo 在注册表内；先行的 ⚠️ warning 给出真因，但最终错误面会误导 MCP（以为 combo 未注册）。低。

建议：区分「未注册」与「已注册但 schema 坏」两类错误面（如 catch 后抛带明确信息的 UsageError，而非返回 None）。

### S3. SELECTION_FIELDS 为跨仓双份（协议 gen-params + 桥 combos.py），需同步演进
协议仓 `gen-params.py` 与桥 `bridge/combos.py:26` 各持一份 `("suited_for","tradeoffs")`，且策略不对称：底座侧 validate_selection **容忍未知字段**（策展区可演进、轮转保留），桥侧 merge 只并已知字段、combo 段 validate **拒绝未知字段**。后果：协议日后新增 selection 字段 → 底座可合法携带，但桥 `merge_selection` 会**静默丢弃**该新字段（内省面比底座策展少），combo 段写该字段则被硬拒，需两仓同步改才一致。

建议：把字段集作为协议契约点写进 SCHEMA.md（单一真源），或在桥侧对「底座 selection 含桥未知字段」显式告警而非静默丢弃。低优先。

---

## ⚪ nit

- **N1** `list-combos` human 输出只列 `suited_for`，`tradeoffs` 仅 JSON 可见——菜单可接受，无碍。
- **N2** 校验归属后 generate 先建 schema 再校验形态：目标 combo 若「结构坏但 schema 可建」，validate 在 invoke 抛 `SystemExit`（信息清晰）；若「schema 都建不起」，落到 S2 的 no-such-command。两类失败错误面不统一（建议面已提）。

---

## 结论

**无阻塞，可合入。** 内省面、selection 合并、R1/A2 校验归属均实现正确且测试覆盖到位（51 全绿、实测通过）；版本钉到已合入的底座 v2 提交、可达无次序隐患。S1 建议合入前顺手处理（内省面与落参面同构，直接影响 MCP 消费正确性）；S2/S3 为低优一致性项，可在能力层（bridge-mcp-server）落地时一并对齐。
