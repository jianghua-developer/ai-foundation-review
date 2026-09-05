# bridge-mcp-server `main`@08a3011 回归审查报告（P3 评审建议落地）

- 审查日期：2026-09-05
- 审查对象：`~/project/bridge-mcp-server` @ `main`（tip `08a3011`）
- 基线：上轮 tip `3383a1a` → 本轮 tip `08a3011`（1 fix 提交：A1/N1/N2/N3/A3）
- 关联：桥侧 A2 由上游 fullstack-bridge main 解决（`c3f8257` shared 标注，可执行 15:07 重建）
- 状态：**A1/A2/A3/N1/N2/N3 全部闭环，无阻塞、无建议残留**；26 测试 + ruff 全绿、语义探测通过

## 验证记录（副作用已还原）

- **测试**：`BRIDGE_EXE=… uv run pytest` → **26 passed**（上轮 25 + A1 回退单测）；`ruff` All checks passed
- **A1 探测**：`protocol.introspect_copier` 对无 params.json 的通用 copier.yml 模板正确内省——无默认→必填、字面默认保留（str/int）、`when:false`+表达式默认→`derived:True` 不暴露；`_load_doc` 回退接入 get_template_params/generate_single
- **A2 探测（经 server 透传桥 show-combo）**：`shared=['auth_mode','project_description']`（≥2 units 原生参数）；`api_base_url`（仅前端）不标 shared；`project_name/project_title` 在 `internal`、不进 params 也不标 shared——与 DESIGN §6.6/U1 语义一致
- **governance 目录**：已 README-only（`README.md` 在，`__init__.py` 已删），符合 DESIGN §3.1
- 工作区 git clean

## 上轮问题处置对拍

| 上轮 | 处置 | 复核结论 |
|---|---|---|
| A1 无协议底座回退缺失 | `protocol.introspect_copier`（copier.yml 内省原生参数，无 selection）+ `tools_single._load_doc` 回退；DESIGN §6.1/§5.1 补注 | ✅ 关闭。⑥其它/通用 copier 地址零注册直生成承诺兑现；单测覆盖，探测正确（表达式默认省略→必填、when:false→derived） |
| A2 shared 标注缺失 | **桥侧解决**：fullstack-bridge main `c3f8257` param_schema 对 ≥2 units 原生参数标 `shared:true`（身份除外），show-combo 透传；server `get_combo_params` 直传即得 | ✅ 关闭（上游闭环）。语义探测：共享（auth_mode/project_description）标注、私有不标、internal 分离 |
| A3 派生「由哪些输入算出」无数据支撑 | DESIGN §6.2 措辞收敛为「派生名字列表（协议暂无 computed_from）」 | ✅ 关闭。不再承诺无数据字段 |
| N1 返回缺「后续提示」 | generate_single/generate_multi 返回加 `next_steps` | ✅ 关闭。DESIGN §6.1/§6.4 对齐 |
| N2 多端无 target 预检 | generate_multi 补 target_dir 非空预检（与单端一致错误面） | ✅ 关闭 |
| N3 governance 占位形式 | 删 `__init__.py`，DESIGN §3.1 README-only | ✅ 关闭（README.md 已在仓） |

## 本轮新观察（无遗留）

- A1 回退目前仅单测覆盖、无 e2e（⑥其它单端底座尚未入册，无真实载体）——待纯单端底座接入时补一条生成 e2e 即可，非阻塞。
- A2 依赖桥可执行含 `c3f8257`；`dist/bridge` 需在桥 main 更新后重建（CLAUDE 前置已注明），属运行部署约束非缺陷。

## 结论

**上轮 3 建议 + 3 nit 全部闭环（A2 由桥侧 c3f8257 完成、server 透传），无阻塞、无建议残留，具备合入条件。** 生成面六工具 + 内省 + 回退 + 玩法/资源/壳侧 skill 与 DESIGN §3-§10 一致。
