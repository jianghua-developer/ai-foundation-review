# fullstack-bridge `refactor/nunit-topology` 审查修改意见

> 对应：[review-report.md](review-report.md)（2026-09-04，2 阻塞 + 4 建议 + 3 nit）
> 本文件给出每条问题的**修改方向与落地约束**，供实施时对照。阻塞项按报告结论建议优先处理。

---

## 🔴 B1 + B2（同源，建议一并修）

两条同根于「建 generate group 时 eager 求每 combo schema」，而 schema 求值 = clone/checkout 底座。拆开看各自修复：

### B1 漂移检测自我抵消

**根因**：`check` 的「当前底座」参照 = 共享缓存**工作树**，但该工作树在本进程建组时已被 `_ensure_base` checkout 回 pinned → old(new) 恒等。

**修改方向**：漂移「当前」参照改为**远端 ref**，不再用会被 checkout 污染的工作树。

```python
def _check_drift(...):
    for key, unit in iter_units(combo):
        src, pinned = unit["source"], unit.get("version")
        base_dir = resolve_base(src)
        old, err = read_params(base_dir, pinned)                    # git show pinned
        cur_ref = base_version if (base_repo and src == base_repo) else "origin/HEAD"
        new, err = read_params(base_dir, cur_ref)                    # git show origin/HEAD
        ...
```

**落地约束**：
- `read_params` 已支持任意 git ref（`git show <ref>:params.json`），`origin/HEAD` 是合法 ref → 改动小。
- `origin/HEAD` 需存在：clone 后默认有；但要反映「底座最近推进」，check 前须 `git fetch`。放哪？
  - 折中 a：`check` 命令开头对涉及底座 `git fetch origin`（联网，但 check 本就该看远端真实状态）；
  - 折中 b：复用 `clone-bases.py --all`（CI 已在 check 前跑它）——CI 路径 fetch 由它负责，本仓 check 保持轻；
  - 本仓 e2e/手动 check 因底座未推进、origin/HEAD==pinned，仍报对齐——语义正确（本地无新底座状态可查）。
- **默认分支名**：`origin/HEAD` 符号引用对 github clone 指向默认分支，够用；严谨可用 `git symbolic-ref refs/remotes/origin/HEAD` 解析。
- **副作用**：cache 被 generate 留在 pinned，check 不再依赖工作树后即解耦——generate 与 check 可各自独立工作。

### B2 建组即全量 param_schema（check 被迫联网 + 坏 combo 拖死全局）

**根因**：Click 的 `add_command` 必须在解析前完成，group 建组即遍历全部 combo 求 schema。

**修改方向（务实折中，Click 无法真正按需惰建子命令）**：拆分「轻量路径」与「schema 路径」——

- `bridge` 顶层 group 下，`check` 与 `generate` 是**兄弟命令**。Click 建组顺序：先 `add_command(check)`，`generate` 组**延迟到用户真正调用 generate 时**再建。
  - 实现：`generate` 不预挂子命令，用 `click.Group(invoke_without_command=True)` + 在 `generate` 的 callback 里 `add_command` 后转发？**Click 不支持运行时加命令再 dispatch**。
  - **可行替代**：把 `generate` 从「子命令 group」降为「一个命令 + combo 是 `click.argument`」？选项又依赖 combo，回到原点。
- **真正可行**：给 `param_schema` 加**缓存 + 失败降级**，让建组廉价且健壮：
  1. **schema 快照缓存**：首次求值后写 `~/.cache/fullstack-bridge/schema/<combo>.json`，后续建组读缓存不 clone（见 S4 联动：缓存失效需 fetch）。
  2. **单 combo 失败降级**：`make_cmd` 对 `param_schema` 抛错的 combo `continue`（不建子命令），打印告警——坏 combo 不再拖死整 CLI，`check` 仍可用。

- **报告建议「per-combo 惰性」**：若坚持，方向是把入口拆成两个可执行（`bridge-generate` / `bridge-check`）或 `cli.py generate` 与 `cli.py check` 各自独立建组——`bridge check` 入口根本不建 generate group。**这是最干净的解法**：CLI 拆 `generate` 与 `check` 两个顶层入口，各自只建自己需要的命令树。代价是失去单入口（本重构原目标），需权衡是否值得为 check 的独立保住单入口。

**建议**：优先做「降级 + 顶层拆 check 建组」，保住单入口且 check 不触网。

---

## 🟡 S1–S4

### S1 unit source 的 URL/本地路径承诺与 param_schema 矛盾

**修改方向**：与 generation-architecture Q4（删逃生舱）对齐——**删掉 URL/路径承诺**，多端只认注册裸名。

- `combos.yaml:4` 注释、`combos.py` 模块 docstring 里的「显式 git 地址/本地路径」表述删除；
- `resolve_template`/`resolve_base`/`ensure_git_repo` 的 is_url/路径分支删除（裸名唯一路径），`param_schema` 直调 `_ensure_base` 即一致；
- 若担心未来真需要本地路径调试，另开能力层 `generate_single`（那是它的域），不进 combos.yaml。

### S2 共享同名参数 schema 取自 consumer 端而非 provider

**修改方向**：`param_schema` 去重规则从「保留首现」→「provider 端优先」。

```python
# 合并去重时：同名参数，若当前端是某 edge 的 provider，则覆盖首现 spec
def param_schema(...):
    providers = {p for _, p in edge_pairs(combo)}   # 所有 provider key
    for key, unit in iter_units(combo):
        for name, spec in unit_params.items():
            if spec.get("derived"):
                continue
            if name in schema and key not in providers:
                continue            # consumer 端不覆盖
            schema[name] = {**spec, "unit_key": key}
```

- **加一致性校验**：对跨端共享参数（多端同名）若 enabled choices 或 default 不同 → `print` 告警（提示两端底座默认漂移），不静默取一。
- python-react 现状（两端 choices/default 一致）不受影响。

### S3 「N 不包括 1」无代码校验

**修改方向**：`edge_pairs` 或独立校验函数加形态约束：

```python
def validate_combo(combo_name, combo):
    units = combo.get("units", {})
    if len(units) < 2:
        raise SystemExit(f"❌ 组合 {combo_name} units < 2（桥只管多单元，N≥2）")
    edges = combo.get("edges")
    if not edges:
        raise SystemExit(f"❌ 组合 {combo_name} 缺 edges（必填显式）")
    # edge_pairs 已校验端点存在
```

`load_combos` 或 CLI 建组时对每 combo 调一次。

### S4 缓存 clone 从不 fetch → 钉新 sha 崩溃

**修改方向**：`_ensure_base` checkout 前先 fetch 或失败重试：

```python
if version and cur != version:
    # 本地无该 sha：fetch origin 后再 checkout
    r = subprocess.run(["git", "-C", dest, "checkout", version], capture_output=True)
    if r.returncode != 0:
        subprocess.run(["git", "-C", dest, "fetch", "origin"], check=True)
        subprocess.run(["git", "-C", dest, "checkout", version], check=True)
```

- `clone-bases.py` 同步（其 `--collect-params` 路径已 checkout，补 fetch 同理）；
- 报错文案友好（含「缓存旧、需 fetch」提示）。

---

## ⚪ N1–N3

- **N1**：删 `_BASE_CACHE` 别名，`clone-bases.py` 改从 `bridge.combos import BASE_CACHE`（或共享常量文件）。step-design §1.2 说导出共享常量——落实它。
- **N2**：规划文档与本轮漂移——按既有先例，**合 main 后删除 docs 规划产物**即消解，无需回改文档追平（规划文档保留在重构分支即其本义）。但根 README 文件表「combos.yaml … stack 技术栈」若描述的是当前实现则应核对（顶层 stack 已删）——确认根 README 不残留过时行。
- **N3**：step-design §6 验收命令 `uv run bridge …` → 改 `uv run python cli.py …`（开发态无 `[project.scripts]` 入口）。规划文档合后删除同样消解。

---

## 实施顺序建议

1. **B2 降级 + 顶层拆分 check 建组**（check 不触网、坏 combo 不拖死）——先做，解锁可靠手测；
2. **B1 漂移参照改 origin/HEAD + fetch**（恢复漂移检测 1a）——同一批；
3. **S3 校验 + S1 删 URL/路径承诺**（形态约束，防「注册表多组合后咬人」）；
4. **S4 fetch 重试 + N1 常量共享**（健壮性/整洁）；
5. S2 一致性告警、N2/N3 文档核正随文档批次。

B1+B2 修复后，重跑报告「已执行验证」节实验（漂移复现、坏 combo 降级、check 无网）作回归确认。
