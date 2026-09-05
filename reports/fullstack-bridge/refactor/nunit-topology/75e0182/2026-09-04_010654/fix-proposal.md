# fullstack-bridge `refactor/nunit-topology` 二轮审查修改意见

> 对应：[review-report.md](review-report.md)（2026-09-04 二轮，0 阻塞；R1–R5 建议）
> 上轮 B1/B2/S1–S4 已✅修复。本文件给出 R1–R5 处置建议。
> 报告结论：R2/R3 建议**合入前顺手处理**；R1/R4/R5 可在接入第二组合（python-vue / ui-bff-api）时一并落实。

---

## 建议合入前处理（R2 / R3 / N1-根README）

### R2. 手动 check 漂移参照依赖 origin/HEAD 新鲜度 + 信号失败静默回退

**现状**（cli.py `_check_drift`）：
- 无信号时读 `origin/HEAD`，但 check 路径不 fetch → 本机缓存长期未 fetch 时静默漏报上游漂移；
- 信号路径 `git show <base_version>` 失败会**静默回退工作树**，可能掩盖派发载荷问题。

**修改方向**：
```python
# 1) check 有网时先刷新 origin/HEAD：对涉及底座 git fetch origin（或 ls-remote）
#    —— 放 check 命令入口（_build_check_command 回调），非 _check_drift 每 unit 重复
# 2) 信号 ref 读取失败 → 显式报错（SystemExit 或 ✗ 标记），不静默回退
```

**落地约束**：
- fetch 放哪：`check` 回调开头对 `_select_targets` 涉及的底座统一 `git fetch origin`（一次，非每 unit）。CI 每次 fresh clone 无需 fetch（本地已是新）——fetch 幂等无副作用，可统一加。
- **离线场景**：若 fetch 失败（无网），应回退用现有 origin/HEAD 而非中断——区分「fetch 网络错」（可继续用旧 ref + 提示）与「信号 ref 不存在」（真错误，须报）。
- 信号路径：`git show <base_version>` 失败 = 派发 sha 不在本地 → **报错**，不静默回退工作树（工作树是另一个语义，不该混作信号的兜底）。

### R3. onboarding:108 残留 URL/本地路径承诺

**现状**：`docs/base-onboarding.md:108`「显式 git 地址/本地路径亦可（无契约治理）」——与 S1 定案及代码矛盾，新组合作者照此会误配 units.source。

**修改方向**：改为——
```
> source 须为 bases 注册裸名；多端只认注册组合（Q4），单模板形态走能力层 generate_single。
> 底座一律 clone 到缓存（~/.cache/fullstack-bridge/bases）。
```
（规划文档 refactor-step-design/generation-architecture 的 URL/逃生舱措辞属历史计划，合 main 后删除即可，不回改。）

### N1. 根 README combos.yaml 行「+ stack 技术栈」

**现状**：README.md:25「combos.yaml … units≥2 + edges + version 基线 + stack 技术栈」——顶层 stack 已平铺进 units.{app,stack}，措辞过时（units 示例已用 keyed app/stack，但文件表这行没同步）。

**修改方向**：文件表该行 `+ stack 技术栈` → 删除或改 `units.{key}.{app,stack}（README 元数据）`。

---

## 建议第二组合落地时处理（R1 / R4 / R5）

### R1. validate_all_combos 挂 generate 组 invoke，坏兄弟 combo 阻断健康 combo

**现状**：`LazyGenerateGroup.invoke` 先 `validate_all_combos()`——某 combo 形态非法会 SystemExit 拦下 `generate <健康combo>`，与「坏 combo 不拖死」意图相悖（schema 构建已降级，形态校验仍整体硬卡）。

**修改方向**：generate 只校验**目标 combo**（`get_command` 命中后对单 combo validate）；整注册表校验留给 `check`（check 本就遍历全部，天然该全量卡）。

实现：`invoke` 里的 `validate_all_combos()` 移除；`get_command` 构建 combo 前先 `validate_combo(cmd_name, combo)`——单 combo 非法仅该 combo 不可生成，兄弟健康 combo 不受影响。

### R4. 多 provider 时共享参数取「声明序首个 provider」，与 merge_order「最深 provider 赢」不同构

**现状**：`param_schema` 覆盖条件 `key in providers and old.unit_key not in providers`（combos.py:224）——ui→bff→api 链 providers={bff,api}，共享参数保留**先声明**的 bff spec；merge_order 展平后 api（最深）才赢。当前单 provider（python-react）不受影响。

**修改方向**：多 provider 平局按 **merge_order 尾者（最深 provider）** 定，与合并语义同构：
```python
# providers 有多个时，共享参数的最终 spec 取 merge_order 中序最末的 provider
deepest_provider = merge_order(combo) 最后出现的 provider key
# 合并去重循环结束后，对共享参数若存在更深 provider 的同名 spec → 以其覆盖
```
或简化为：**收集阶段不覆盖，合并阶段按 merge_order 尾 provider 决定**——把「取哪端 spec」与「合并序」统一到同一规则（`merge_order` 尾者）。

告警文案「取 provider 端」相应改为「取最深 provider（merge_order 尾者）」。

### R5. validate_combo 未校验 unit.source ∈ bases / 链边数 = units-1

**现状**：source 笔误（不在 bases）到 `generate` 才于 param_schema 报 SystemExit、`check --all` 于 resolve_base 抛错——不如 load 期清晰提示「未注册 source」；链形（4 单元 1 边）亦未约束。

**修改方向**：`validate_combo` 补两项：
```python
def validate_combo(combo_name, combo):
    ...现有 units≥2 / edges 必填 / 端点合法...
    bases = _load_bases()
    for _, unit in combo["units"].items():
        if unit.get("source") not in bases:
            raise SystemExit(f"❌ 组合 {combo_name} unit {unit} 的 source 未在 bases 注册表")
    # 链形：edges 数应 = units-1（单链拓扑；星型/环等非法——目前系列只支持链式契约）
    if len(combo["edges"]) != len(combo["units"]) - 1:
        raise SystemExit(f"❌ 组合 {combo_name} edges 数应 = units-1（链形）")
```
> 注意：链形约束是「当前只支持链式契约」的体现——若未来支持非链拓扑需放宽；加注释说明。

---

## 实施顺序建议

1. **R2 + R3 + N1**（合入前批）：check fetch 刷新 origin/HEAD + 信号失败显式报错；onboarding 措辞对齐；根 README 行修正。
2. **R5**（低成本，可与上批同做）：validate_combo 补 source 注册性 + 链形——source 笔误从 generate 期提前到 load 期。
3. **R1 + R4**（第二组合落地时，与 python-vue / ui-bff-api 接入同批）：generate 单目标校验；多 provider 取 merge_order 尾者。

R1-R5 修复后复跑：39 测试 + 漂移实验（fetch 后 origin/HEAD）+ 构造坏 combo 生成降级。
