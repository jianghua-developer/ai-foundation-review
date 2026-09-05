# fullstack-bridge `refactor/nunit-topology` 分支审查报告

- 审查日期：2026-09-04
- 审查对象：`~/project/fullstack-bridge` @ `refactor/nunit-topology`
- 基线：相对 main 13 提交，merge-base `ee8f3f9`
- 状态：36 单测 + e2e 全绿；`check --all` 绿；生成回归通过；含 2 条阻塞 + 4 条建议 + 3 条 nit

## 审查范围

核心代码改动面：

| 文件 | 改动 |
|---|---|
| `cli.py`（新增） | Click 统一入口：`generate <combo> <project>` + `check`，选项由底座 params.json 数据驱动 |
| `bridge/combos.py` | units/edges 泛化 + 统一 clone/baked + `param_schema`（derived 暴露规则） |
| `bridge/integrate/answers.py` | `merge_answers_by` per-edge 属主合并 |
| `bridge/check/*` | 仅 ruff 格式化（逻辑迁入 cli.py） |
| `integrate.py` / `check.py` / `integrate.spec` | 删除（被 cli.py / bridge.spec 取代） |
| `.github/scripts/clone-bases.py` / `bridge.spec` | 缓存 clone + params.json 烘焙 |
| `templates/project-README/` / `combos.yaml` / workflows / README | units-key 数据驱动 |
| `docs/generation-architecture.md` / `docs/refactor-step-design.md` | 规划产物（合 main 后循例删除） |

## 已执行验证（副作用均已还原到 pinned）

- `uv run pytest` → **36 passed**
- `cli.py check --all` → ✅ 全部对齐（仅 1 条 advisory：auth_mode `none` 经 else 兜底）
- `cli.py generate python-react` 冒烟 → 结构回归（`frontend/ backend/ docs/CONTRACT.md README.md`），answers 无跨端污染，CONTRACT/README 渲染正确；`--with-child-app true` 等 bool 取值形态可解析
- 漂移检测复现实验：缓存 base 人为推进到含新参数 `zz_fake_drift` 的“上游”提交 `f852d64` → `check --combo python-react` 仍报 `✓ 对齐基线 0034ec9 / ✅ 全部对齐`
- 模拟 unit source = git URL → `SystemExit: combos.yaml bases 未配置「…」的 git 地址`

---

## 🔴 阻塞（架构/正确性）

### B1. `check` 的漂移检测自我抵消——`--combo`/`--all` 与 `bridge-gate` 检测失效

**位置**：`cli.py:56-88`（eager 建组）、`cli.py:86-87`、`cli.py:173-180`（读工作树）、`combos.py:95-102`（checkout）

统一 CLI 建组时 `_build_generate_group` 对**每个** combo 立即求 schema，`param_schema` 内 `_ensure_base(source, version)` 在 HEAD≠pinned 时把共享缓存 clone `git checkout` 回 combos.yaml 钉的 version。随后 `_check_drift` 的“当前工作树”参照 = `resolve_base(src)` 的工作树（`cur_ref=None` 时 `read_params` 读工作树 params.json）——于是 old（`git show pinned`）与 new（工作树）是**同一 commit**，恒等、永远“对齐”。

复现输出（该次调用开头两次 `HEAD is now at <pinned>` 即 eager checkout 先行证据）：

```
HEAD is now at 0034ec9 …        ← 建 generate group 时先把缓存钉回 pinned
✓ [python-react frontend] vite-react-spa-template 对齐基线 0034ec9   ← 上游已前进但看不见
✅ 全部对齐
```

影响：手动 / `--all` / `bridge-gate.yml` 的“底座前进、combos 未 bump”检测（检查 1a）失效并给出虚假保证；仅 check-drift 的显式 `--base-repo/--base-version` 路径（`git show` 比对）仍有效。README 声称 check 据此检测漂移，不成立。

修复方向：漂移“当前”参照改为远端 ref（`origin/HEAD` / fetch 后 origin 默认分支），不要用会被本进程 checkout 掉的共享工作树；或让 schema 惰性化（见 B2）。

### B2. 建组即全量 param_schema：`check` 也被迫联网/改缓存；任一坏 combo 拖死整 CLI

**位置**：`cli.py:262-267`（`build_bridge_group`）

`build_bridge_group` 无条件先建 generate group → 每次 `check`/`--help`/`generate` 都先 clone/checkout **所有**底座。后果：

- 只读 `check` 被强加网络依赖与对共享缓存 HEAD 的写副作用（多次运行把缓存来回搬动）；
- 任一 combo schema 失败（缺 params.json / version 不可 checkout / source 异常）→ SystemExit 在建组期抛出，`bridge check` 与 `generate` **整体不可用**——只想查 python-react 也会被 python-vue 拖死。

建议：per-combo 惰性建命令（首次 `generate <combo>` 才求其 schema），或建组期对单 combo 失败降级跳过。

---

## 🟡 建议

### S1. unit source 的“显式 git 地址/本地路径”名存实亡且自相矛盾

**位置**：`combos.yaml:4`、`combos.py:5-6`（宣称支持）vs `combos.py:179`（param_schema 直调 `_ensure_base`）、`combos.py:87-89`（SystemExit）

`combos.yaml` 注释与 `combos.py` 模块 docstring 声明 units.source 可为“显式 git 地址/本地路径（逃生保留）”，`resolve_template`/`resolve_base`（`combos.py:111-126`）确实支持；但 `param_schema` 绕开 `resolve_*` 直调 `_ensure_base`，非裸名 source 查 bases 注册表失败 → SystemExit。与 generation-architecture Q4“彻底删逃生舱、多端只认注册组合”相悖。

建议：删掉 combos.yaml/combos.py 的 URL/路径承诺及 resolve_* 分支（最贴合定案），或让 param_schema 对非裸名 source 直接读其 params.json / 明确拒绝。

### S2. 共享同名参数的 schema 取自“先声明”（consumer）端，而非 edge provider

**位置**：`combos.py:188-190`（保留首现 spec）、`answers.py:29-31`（provider 赢）`cli.py:106-110`（broadcast）

`param_schema` 去重“保留首现 spec”，`unit_key` 记为首个声明端；合并语义是 provider 属主、后者赢。python-react 因两端 auth_mode 默认/choices 恰好一致未显形。一旦未来组合共享参数两端默认或启用 choices 不同，CLI 默认值（首现端）会被 broadcast 成各 unit copier 强值，静默覆盖非首现端“自身默认”，契约随之偏离其本义。

建议：共享参数 default/choices 由 edge provider 端决定，并在 param_schema 对多端一致性（默认 + enabled choices）显式校验/告警。

### S3. “N 不包括 1”不变量无代码校验

**位置**：`combos.py:51-60`（`edge_pairs` 仅校验 key 存在）

未校验 units≥2、edges 必填非空、链边数=N-1。1-unit 或断边组合会照常进入生成链跑契约/README 机器（正是桥不该背的）。

建议：load/校验层加形态约束（units≥2、edges 存在且非空、边端点合法）。

### S4. 缓存 clone 从不 fetch——钉到新 sha 时旧缓存 raw 崩溃

**位置**：`combos.py:91-94`（仅 .git 缺失时 clone）、`combos.py:102`（checkout 不 fetch）、`clone-bases.py:58-69`

对本地没有的 pinned sha 无 fetch，`git checkout` 直接抛 `CalledProcessError` 无友好提示。step-design §1.3 开发循环（改底座→push→bump→桥验证）第二次迭代即踩中，须手动 fetch/删缓存。

建议：checkout 前 fetch（或失败时 `git fetch origin` 重试 + 明确报错）；CI 若日后加 actions/cache 同理。

---

## ⚪ nit

- **N1** `combos.py:22-23` 与 `clone-bases.py:23` 双写同一 `BASE_CACHE`（step-design §1.2 说导出共享常量）；`_BASE_CACHE` 别名“保留旧引用兼容”已无外部引用，属死别名。
- **N2** 规划文档与本轮实际漂移（dc2b13f）：step-design §4“不改 templates/project-README/”、§2.1 示例保留顶层 `stack: {...} # 不变`，实际已把 stack 平铺进 `units.{app,stack}` 并通用化 project-README（units_desc）；根 README 文件表仍写“combos.yaml … + stack 技术栈”（顶层 stack 已删）。规划文档若在合并前继续作评审依据会误导，循例合后删除即消解。
- **N3** step-design §6 验收命令 `uv run bridge check …` 本仓无 `[project.scripts]`/`bridge` 入口（实测落到系统 `bridge` 命令）；开发态统一是 `uv run python cli.py`，仅验收行写法不符。

---

## 结论

改动方向（units/edges 泛化、per-edge 属主合并、Click 数据驱动 schema、clone/baked、spec 烘焙 params.json）落地干净，测试与生成回归全部验证通过。

**合入前建议优先处理 B1**：它在不报错的情况下把 check 漂移检测（1a）静默置空，是本次重构引入的实质性回归；B2 与其同源，一并修（惰性 schema）即可兼治。S1–S4 属一致性与健壮性缺口，当前单组合配置下均不触发，但都是“注册表再长一个组合”就会咬人的隐患。
