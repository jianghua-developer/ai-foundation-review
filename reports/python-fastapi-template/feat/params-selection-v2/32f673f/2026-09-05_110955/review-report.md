# python-fastapi-template `feat/params-selection-v2` 审查报告

- 审查日期：2026-09-05
- 审查对象：`~/project/python-fastapi-template` @ `feat/params-selection-v2`
- 基线：main → tip `32f673f`（2 提交：`782d182` params.json v2 selection 接入 + `32f673f` 全链钉 copier 9.17.1）
- 关联真源：fullstack-param-protocol（协议仓，main 领先 origin 2 提交：`db2e9a8` v2 + `10dcbed` copier 钉）
- 状态：**无阻塞**；3 建议 + 1 nit；vendor 与协议仓完全一致、验证全绿

## 验证记录（副作用已还原）

- **vendor 一致性**：`bin/gen-params.py` 与协议仓 `gen-params.py@HEAD` 逐字节相同；`bin/GEN_PARAMS_VERSION` = 该文件内容 sha256（`60bf6ac…`），三方（base 文件 / base 标记 / 协议仓 HEAD）一致
- **协议仓单测**：`uv run pytest` → 7 passed（v2 两区 + 轮转保留用例）
- **底座 `--verify`**：绿（copier==9.17.1）
- **regen 幂等**：对提交的 params.json 跑 generate → git 零改动、内容一致（selection 轮转保留稳定）
- **copier 钉版本**：base hook / base CI / 协议 pyproject 均 `copier==9.17.1`；base 无 9.17.0 残留
- **桥兼容性（推理）**：v2 改动全在 params.json 顶层（schema_version / selection / generated_by），`params` 区未变 → 桥 `diff_params`（只比 `params.*`）不受影响，main 合入后无需 bump combos.yaml；verify 不因 selection 触发契约漂移

## 审查结论摘要

作为协议 v2（两区）的**首个示范底座**，本分支是干净、忠实的接入：gen-params.py 从协议仓原样 vendor、钩子/CI 与协议模板同构并升级到双区语义、params.json 迁移正确、selection 策展内容 schema 合法。未发现阻塞项。以下为低风险一致性与防线建议。

---

## 🟡 建议

### A1. `bin/GEN_PARAMS_VERSION` 无任何消费方——vendor 漂移没有自动防线
文件只存一行内容 sha256，base 全仓（含 CI/钩子）与协议仓均无读取它的脚本/文档。作用仅是“人工记住当前 vendored 内容指纹”，既不检测「协议仓更新后本底座未同步」，也不检测「本底座改了文件却没更新标记」（当前三者一致，属侥幸维持）。

建议：在 params-check.yml 的 verify job 顺带加一行低成本自检，例如 `test "$(sha256sum bin/gen-params.py | cut -d' ' -f1)" = "$(cat bin/GEN_PARAMS_VERSION)"`，使 vendor 内容与标记漂移在提交时即暴露；或移除该死标记，改用 sync 文档写明「升级 gen-params.py = 从协议仓复制 + 更新标记」。

### A2. 协议仓「接入模板」描述仍停 v1，底座示范复制件已超前——模板/底座双源文案漂移
协议仓 `hooks/pre-commit` 与 `workflows/params-check.yml`（供新底座复制的 canonical 模板）头注仍写 v1 语义（“自检 params.json == copier.yml”）；本底座复制件已改为 v2 双区文案并删「见 pyproject」（见 .githooks/pre-commit:4、params-check.yml:2-5）。功能一致（gen-params `--verify` 本身已双区），但：后续底座从协议模板复制将拿到过时描述；协议模板将来更新时 base 的 diff 会带文案噪音。

建议：把协议仓两模板头注回写为 v2 双区语义（单一真源在协议仓），底座不再改注释。

### A3. pre-commit 兜底路径未校验 copier 版本——非 9.17.1 环境会静默改脏 params.json
`.githooks/pre-commit:19-20` 的兜底解析器指向 `/home/jeff/.local/share/uv/tools/copier/bin/python`，仅注释「须同版本」（:4）而无版本检查。若该工具实际为 9.17.0，best-effort regen 会把 params.json 的 `generated_by` 改写为 `@9.17.0` 并留在工作区；开发者如未察觉直接提交，CI（钉 9.17.1）verify 因全文本比对必然失败。uv 主路径本机存在故概率低，但兜底是对“无 uv 环境”的让步，那里反而最不可控。

建议：兜底运行前先查 copier 版本，非 9.17.1 则跳过 regen 并提示（而非用错版本静默改写）。

---

## ⚪ nit

- **N1**（A2 同源）base hook 与协议模板单行注释措辞差异（base 去「见 pyproject」、补「须同版本」）——若 A2 模板回写即消除，不留独立。

---

## 结论

**无阻塞，可合入 main。** 三条建议均为“防线/一致性”级：A1 补 vendor 自检或去死标记、A2 回写协议模板头注（单一真源归位）、A3 兜底版本检查。合入后桥侧无需改动（params 区未变），系列其余底座（vite-react / vite-vue）可循本分支作示范顺序接入 v2。
