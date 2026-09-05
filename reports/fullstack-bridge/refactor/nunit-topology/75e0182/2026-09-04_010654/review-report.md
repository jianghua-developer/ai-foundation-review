# fullstack-bridge `refactor/nunit-topology` 二轮审查报告（修复复核）

- 审查日期：2026-09-04
- 审查对象：`~/project/fullstack-bridge` @ `refactor/nunit-topology`
- 本轮基线：上一轮 tip `dc2b13f` → 本轮 tip `75e0182`（1 修复提交）
- 修复范围：B1/B2/S1–S4 + N1（提交信息自述，逐项对拍复核）
- 状态：39 测试全绿；漂移检测/惰性零触网经实测复现；无阻塞残留

## 验证记录（副作用已还原）

- `uv run pytest` → **39 passed**（上轮 36 + 净增 3：validate_combo 4 项 + 惰性降级/check-help 等，删除旧 is_url/ensure_git_repo 用例）
- `check --combo python-react` → 不再打印 `HEAD is now at …`（惰性组生效，check 零触网/零 checkout）
- **漂移复现**：把缓存 base 远端 main 指向含新参数 `zz_fake_drift` 的模拟上游提交 → `check` 正确检出 `⚠️ 漂移（基线 0034ec9 vs 远端 origin/HEAD）`、退出判 drift；还原 origin/main 后恢复对齐
- `generate python-react` 冒烟 → 惰性路径正常建 schema（触发时 clone），产物结构回归
- 树内残留引用扫描 → 仅剩 docs 两处（见 S1 残留）

---

## 🟢 一轮问题处置对拍

| 一轮编号 | 处置 | 复核结论 |
|---|---|---|
| B1 漂移自我抵消 | `_check_drift` 当前参照改 `origin/HEAD`（信号路径仍 `base_version`），回退工作树仅作兜底 | ✅ 修复。实测模拟上游 params 变化已检出漂移；check 不再被 eager checkout 污染 |
| B2 eager 建组拖死整 CLI | `LazyGenerateGroup`：`get_command` 惰性建 schema、建败告警跳过；`list_commands` 只列名 | ✅ 修复。`check`/`--help` 零触网；`test_generate_broken_combo_degrades` 覆盖降级 |
| S1 URL/显式路径承诺 | 删 `is_url`/`ensure_git_repo`/resolve_* 的 URL/本地路径分支；combos.yaml 注释改「只认注册裸名」 | ✅ 代码与 combos.yaml 一致（遗留 onboarding 文档，见 S1-残） |
| S2 共享参数 consumer 优先 | param_schema 先收 per_unit → 共享 spec 取 provider 端；跨端 default/choices 不一致告警；身份参数（project_name/title）豁免 | ✅ 单 provider 情形正确（python-react：backend 覆盖 frontend）。多 provider 残留见 S2-残 |
| S3 N≥2 不变量 | 新增 `validate_combo`（units≥2、edges 必填、端点合法）+ `validate_all_combos` | ✅ 基本满足。source 注册性与链形未校验，见 S3-残 |
| S4 缓存无 fetch | `_checkout_or_fetch`：本地缺 sha → `git fetch origin` 后重试，仍失败友好报错 | ✅ 修复 |
| N1 常量双写/死别名 | clone-bases 改从 `bridge.combos` 导入 `BASE_CACHE`；删 `_BASE_CACHE` | ✅ 修复 |

---

## 🟡 建议（二轮新发现 / 修复残留）

### R1. `validate_all_combos` 挂在 generate 组 invoke，坏**兄弟** combo 会阻断健康 combo 的生成
`LazyGenerateGroup.invoke`（cli.py）先 `validate_all_combos()` 再派发子命令。某组合形态非法（units<2/缺 edges）时，`generate <健康combo>` 也会被该兄弟 combo 的 SystemExit 拦下——与「坏 combo 不拖死整 CLI」的意图相悖（schema 构建失败已降级，但形态校验仍整体硬卡）。
建议：generate 只校验目标 combo；整注册表形态校验放 `check`。

### R2. 手动 `check` 的漂移参照依赖缓存 origin/HEAD 新鲜度——无 fetch 时静默过期
改为 origin/HEAD 后比“工作树”正确，但 origin/HEAD 只在 clone/fetch 时更新；桥在 `check` 路径仍不 fetch。CI（每次 fresh clone）与 base CI 派发的 check-drift 路径（`--base-version` 信号，`git show`）不受影响；但**本机缓存 clone 长期未 fetch 时，手动 `check` 会静默漏报上游漂移**。
另：信号路径下若 `git show <base_version>` 失败（sha 不在本地），现在会**静默回退工作树**而非报错，可能掩盖派发载荷问题。
建议：check 有网时先 `git fetch origin`（或 `git ls-remote`）刷新 origin/HEAD；信号 ref 读取失败应显式报错而非回退。

### R3. docs/base-onboarding.md:108 仍宣称「显式 git 地址/本地路径亦可」——与 S1 定案及代码矛盾
```
> source 裸名一律 clone 到缓存（...），显式 git 地址/本地路径亦可（无契约治理）。
```
onboarding 是**长期文档**（非将删的规划产物），新组合作者照此会误配 units.source。建议改为「source 须为 bases 注册裸名；多端只认注册组合（Q4），单模板走能力层 generate_single」。规划文档（refactor-step-design/generation-architecture）中的 URL/逃生舱措辞属历史计划，合 main 后删除即可，无需回改。

### R4. 多 provider 时共享参数取「声明序首个 provider」，与 merge_order「最深 provider 赢」不同构（三单元组合时）
`param_schema` 覆盖条件 `key in providers and old.unit_key not in providers`（combos.py）——ui→bff→api 链中 providers={bff,api}，共享参数会保留**先声明**的 bff spec；而 merge_order 展平后 api（最深）才赢。二者不一致时（两端 default/choices 不同）告警文案「取 provider 端」与实际保留对象不符（实际取的是首个而非最深 provider）。
当前单 edge（python-react，唯一 provider=backend）不受影响。建议：多 provider 平局按 merge_order 尾者（最深 provider）定，或至少在注释/告警中写明平局规则。

### R5. `validate_combo` 未校验 unit.source ∈ bases 注册表 / 链边数 = units-1
source 笔误（不在 bases）在 `generate` 才于 param_schema 报 SystemExit、在 `check --all` 时于 resolve_base 抛错——不如 load 期给出「未注册 source」的清晰提示。链形（N-1 边）亦未约束（4 单元 1 边可通过）。建议 validate 期一并查 source 注册性。

---

## ⚪ nit

- **N1**（沿用一轮 N2 的残留）：根 README 文件表「combos.yaml … + stack 技术栈」——顶层 stack 已平铺进 units.{app,stack}，措辞过时。
- **N2**：规划文档（step-design §6 `uv run bridge` 等）与最终代码仍有出入，属合 main 前即删的历史产物，不影响合入。

---

## 结论

**上轮 2 项阻塞（B1/B2）均已按预期修复并经实测确认，本轮无阻塞残留，具备合入条件。** 剩余 R1–R5 均为组合规模增长前不触发的健壮性/一致性建议：R2、R3 建议合入前顺手处理（分别影响 check 语义可信度与 onboarding 误导），R1/R4/R5 可在接入第二个组合（python-vue / ui-bff-api）时一并落实。
