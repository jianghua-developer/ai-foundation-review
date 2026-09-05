# python-fastapi-template `feat/params-selection-v2` 回归审查报告

- 审查日期：2026-09-05
- 审查对象：`~/project/python-fastapi-template` @ `feat/params-selection-v2`
- 基线：上轮 tip `32f673f` → 本轮 tip `3bf48bb`（1 fix 提交）
- 修复范围：上轮 A1/A2/A3/N1（协议仓同步推进至 `5f94b8d`，底座整份复制 canonical 模板）
- 状态：**全部关闭，无阻塞、无建议残留**；字节级对齐协议仓 + 实测全绿

## 验证记录（副作用已还原）

- **模板单一真源恢复**：base `.githooks/pre-commit`、`.github/workflows/params-check.yml` 与协议仓 `hooks/pre-commit`、`workflows/params-check.yml` @HEAD（5f94b8d）**逐字节一致**（HOOK-IDENTICAL / CI-IDENTICAL），双源文案漂移消除
- **vendor 指纹**：`sha256sum bin/gen-params.py` == `bin/GEN_PARAMS_VERSION` == `60bf6ac…`（协议仓 gen-params.py 未变，仍一致）
- **CI 指纹自检步骤**：同语义命令实测通过
- **双区 `--verify`**：绿（copier==9.17.1）
- **pre-commit 钩子实跑**（uv 主路径）：`✓ params.json 已与 copier.yml 同步`，钩子后 `git status` 干净（幂等）；`bash -n` 语法 OK

## 上轮问题处置对拍

| 上轮 | 处置 | 复核结论 |
|---|---|---|
| A1 GEN_PARAMS_VERSION 无消费方 | 协议仓 + 底座 CI 均加 **vendor 指纹自检** step（`sha256sum` vs 标记，失败 exit 1） | ✅ 关闭。commit 时即暴露 vendor 漂移 |
| A2 协议模板/底座双源文案漂移 | 协议仓回写模板为 v2 双区语义（5f94b8d），底座改为**整份字节复制** canonical 模板 | ✅ 关闭。单一真源归位协议仓；后续底座复制即得 v2 |
| A3 兜底 copier 版本无守卫 | hook 新增 `copier_is_pinned` 守卫（importlib.metadata 查 9.17.1），不符则跳过并提示 | ✅ 关闭。非钉版本不再静默改脏 params.json |
| N1 注释措辞差异 | 随 A2 字节对齐自然消除 | ✅ 关闭 |

## 本轮新观察（无遗留，仅记录）

- 钩子兜底路径 `/home/jeff/.local/share/uv/tools/copier/bin/python` 为**机器特定绝对路径**（沿用既有设计，非本轮引入）：无 uv + 该路径不存在 + 无其他 copier 时，钩子仅提示并跳过，CI 硬校验兜底——行为正确，仅提示语依赖该路径存在性。可接受。
- 指纹自检依赖 Linux runner 的 `sha256sum`（workflow 已跑 ubuntu），无跨平台顾虑。

## 结论

**上轮 3 建议 + 1 nit 全部按预期修复并经实测确认，字节级对齐协议仓 canonical 模板（单一真源归位），具备合入 main 条件。** 系列其余底座接入 v2 时，直接复制协议仓模板即可获得同等防线。
