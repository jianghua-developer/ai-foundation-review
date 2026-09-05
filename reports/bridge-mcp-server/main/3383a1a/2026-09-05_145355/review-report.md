# bridge-mcp-server `main`@3383a1a 审查报告（能力层生成面首版）

- 审查日期：2026-09-05
- 审查对象：`~/project/bridge-mcp-server` @ `main`（tip `3383a1a`，P3 生成面 server 首版）
- 事实源：docs/DESIGN.md（§3.1/§4-§6 契约）、docs/implementation-plan.md（P0–P4 / M0–M3）
- 状态：**无阻塞，可合入**；25 测试全绿（含单端/多端真实 e2e）；3 建议 + 3 nit（多为 DESIGN 返回契约与实现未对齐的功能缺口）

## 验证记录（副作用已还原）

- **测试**：`BRIDGE_EXE=…/bridge uv run pytest` → **25 passed**（单测 mock 桥/copier + e2e：generate_single 打真实 copier、generate_multi shell-out 真实 dist/bridge，临时目录清理）
- **行为探测**（对照 DESIGN 契约）：
  - 无协议地址（无 params.json）→ `FileNotFoundError`（DESIGN/plan 要求回退 copier.yml，未实现，见 A1）
  - show-combo params spec 无 `shared` 标注（DESIGN §6.6 要求，见 A2）
  - get_template_params `derived` 仅为名字列表，无「由哪些输入算出」（DESIGN §6.2 要求，数据无支撑，见 A3）
- 单端 e2e 离线用本机 python-fastapi 本地仓做 clone 源；多端 e2e 用已打包 bridge

## 审查要点（正向确认）

- **架构一致**：双路径×双链落地——单端（clone→协议直读→spec 校验→copier）零注册、多端纯 cli shell-out（`bridge_mcp/bridge_cli.py`，--json 稳健解析、容忍 warning 前缀）；**六生成面工具、无 check/治理**（DESIGN §6/§10 更新后一致）；一面一 server（servers/generation 独立入口，governance 仅占位）
- **薄封装守界**：多端不直读 combos.yaml、不 clone 底座内省；server 只给菜单/校验/确定性执行/结构化返回；L1 过滤在 server、L2/L3 留壳（guide Prompt 强制 L3 确认）
- **spec 严格校验（单端）**：派生/未知键拒绝、类型/choices、必填（无 default 原生必须给）、target_dir 非空拒绝覆盖——e2e 覆盖
- **selection 单一真源**：templates.yaml 薄菜单不抄 selection；能力层对底座 selection 未知字段显式告警（`selection_warning`，S3 落地）；get_combo_params 直传桥合并 selection
- **资产齐备**：resources（templates:// combos://）+ 玩法 Prompt（引用工具不抄清单）+ skills/generate-project 仓库 canonical（壳侧只引用工具）

---

## 🟡 建议

### A1. 单端链缺「无协议底座回退 copier.yml」——⑥其它/零注册地址直生成承诺对非协议地址不成立
DESIGN §5.1「base 兼容：非系列/未接入协议的裸地址允许回退解析 template/copier.yml 拿参数」、§6.1 校验同、plan P3 任务与风险节「未接入协议的单端地址不得报错——单端回退 copier.yml」均要求回退；实现 `protocol.load_params_json` 直接 `raise FileNotFoundError`（实测确认，无回退）。
影响：系列 6 形态里「⑥其它单模板」及「不在册 git 地址照常 generate_single（§4.8）」目前仅对**已接入协议**的底座成立；任意通用 copier 模板（无 params.json）会失败。现 templates.yaml 只有三协议底座、未触发，但这是文档承诺的缺口。
建议：补 copier.yml 内省回退（读 template/copier.yml 问题声明得原生参数/默认，无 selection），或收窄 DESIGN/templates 表述为「⑥其它待回退实现」，二者择一，勿留双向承诺。

### A2. DESIGN §6.6「shared: true 标注」未落到链（桥 show-combo 未加、server 直传）
DESIGN §6.6（U1 定案写入）要求 get_combo_params 返回清单标注 `shared: true`（= 全链一致单值、填一个广播全链）；实现为桥 show-combo 的 params 直传，spec 无 shared 键（实测 auth_mode 无标注）。U1 语义目前只靠「参数去重成单选项」隐式表达，壳/agent 无法从返回程序化得知「这是整链单一决策、无逐端取值」。
建议：桥 `show-combo` 对「出现于 ≥2 units 的原生参数」加 `shared: true`（顺带把 internal 同理标注），server 透传即得；或明确降 DESIGN §6.6 措辞为「去重即隐式共享」。

### A3. DESIGN §6.2「派生含『由哪些输入算出』」无数据支撑
DESIGN §6.2 返回契约列派生为 `name / derived:true / 由哪些输入算出`；params.json 的 derived spec 无 computed_from 元数据（协议未定义），实现只能给名字列表（实测 `['child_apps']`，与桥一致）。
建议：二选一——协议 params.json 派生 spec 增 `computed_from`（gen-params 从 copier.yml when 表达式/人工注记），或把 DESIGN §6.2 措辞降为「派生名字列表（输入见底座模板/契约）」。勿让返回契约承诺无数据支撑的字段。

---

## ⚪ nit

- **N1** 返回契约缺「后续提示」字段（DESIGN §6.1/§6.4 返回含 `后续提示`）；现由 server instructions/玩法 Prompt 承担该职责，功能无缺但返回结构与 DESIGN 不一致——补字段或降 DESIGN。
- **N2** `generate_multi` 未做 target_dir 非空预检（单端 `generate_single` 有），靠桥 copier 中途报错，错误面与单端不一致——可加同款预检。
- **N3** DESIGN §3.1 树写 `servers/governance/README.md`（占位说明），实现为 `__init__.py`——占位形式不一致，无实质影响。

## 结论

**无阻塞，可合入。** P3 生成面 server 与 DESIGN/plan 高度一致（架构守界、spec 严格、资源/玩法/壳侧 skill 齐备、25 测含真实 e2e 全绿）。3 建议均为「DESIGN 返回契约 vs 实现」的缺口：A1 影响 ⑥其它/通用地址承诺（现不触发，建议在纯单端底座入册前定案补实现或收窄文档），A2/A3 是文档写了无数据/未落地的字段，随下次跨仓改动（协议升级或桥内省扩展）一并处理即可。
