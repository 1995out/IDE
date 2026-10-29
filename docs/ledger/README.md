# docs/ledger · 台账（还没解决的事）

> **建立**：2026-10-27 以 `docs/backlog/` 建立（裁决 D-8）；2026-10-29 文档治理改名 `ledger/`，并把原 `specs/consistency-audit.md` 并入。
> **对应 Supervisor 产物**：证据台账里「没定的、没修的、被挂起的」那一部分——方案的 `conflicts` / `open_questions` 落盘后，若一轮内解决不了，就进这里。
> **分界**：`ledger/` 不是权威。它只回答「还欠什么、谁来定、做完的判据是什么」。与 `specs/`、`adr/` 冲突时，以后者为准。

---

## 清单

| 文件 | 内容 | 状态 |
| :--- | :--- | :--- |
| [`consistency-audit.md`](./consistency-audit.md) | 规格一致性审计台账：`C-` 待裁决矛盾、`M-` 机械修复、`OP-` 挂账项（OP-1～OP-35）、权威分层表 | 持续追加；OP-1、OP-13 待用户裁决 |
| [`w3-known-gaps.md`](./w3-known-gaps.md) | W.3 环境 / 秘密 / 网络守卫的已知缺口 | 7 项中 2 项未修（`tmpRoot` GC、`splitShellSegments` 引号边界） |

## 文档治理挂账（2026-10-29 新增）

本轮通读 `docs/` 时发现、但**故意没在本 PR 里改**的问题。不改的理由都是同一条：动这些需要改规格正文或任务卡正文，属于「改条款」，要用户先拍板（见 [`../scheme.md`](../scheme.md) §12）。

| 编号 | 问题 | 位置 | 建议 | 阻断级别 |
| :--- | :--- | :--- | :--- | :--- |
| DOC-1 | `development-tasks.md` 约 460KB：约 65% 是已完成卡历史，版本头 84 条状态回写与 §6 修订流水重复，卡内样板句重复上百次 | `specs/development-tasks.md` | 拆成卡号索引 + 活跃卡 + 已完成卡 + 修订流水，卡号不变（[`../scheme.md`](../scheme.md) dec-4） | NON_BLOCKER |
| DOC-2 | 任务卡状态矛盾：C.14f、C.15a、C.15c、C.15d 矩阵写 COMPLETED、卡内写 NEW_TASK；J.1 代码已合入（PR #29 / #30）但卡仍为 NEW_TASK | 同上 | 由卡 owner 按 supervisor §10 判定口径回写 | NON_BLOCKER |
| DOC-3 | 任务卡文件内三处「当前基线」互相矛盾（`6415fbd` / `4280de6` / `f14c8cb`） | 同上 L28、L121、§4 | 只保留一处，其余改指针 | NON_BLOCKER |
| DOC-4 | 约 24 处引用已删除的 `docs/archive/*`，约 98 处引用已删除的「走查报告 §6.x」 | 同上 | 统一改为「见 Git 历史」或删除 | NON_BLOCKER |
| DOC-5 | 路径三方不一致：Worktree 目录（`task-{id}` vs `{id}`）、Demo 目录（`.stagent/demo/` vs `.stagent/prototypes/`）、Git Trailer 键名与长度上限 | `system-architecture.md` §5、`constants-and-paths.md` §2/§3、`runtime-contract-and-safety.md` §5.3、代码 | 以代码为准回写规格，或立卡改代码 | NON_BLOCKER |
| DOC-6 | `delivery-state.ts` 仍有 `DELIVERED`，与 ADR-0012、`state-machine-contract.md` 冲突；两套状态机（生命周期 9 态 / 交付轨 A）关系未交代 | `state-machine-contract.md`、runtime §7、§9 | 在 state-machine-contract 补一节两轨关系 | NON_BLOCKER |
| DOC-7 | `state-machine-contract.md` 标「目标、待测试固化」，但 `src/lib/state-machine.ts` 已实现其迁移表；`constants-and-paths.md` 同为草案 | `specs/` 两份草案 | 核对后升格或合并入 runtime 契约 | NON_BLOCKER |
| DOC-8 | RFC v0.2 阶段 0「右栏可写主干」、§3.1「删除旁听」与现行 SSOT 冲突；附录 A/B/C 是 UI 候选清单，不是 RFC | `rfc/v0.2-multi-agent-pipeline.md` | 已加取代横幅；正文改写待 v0.2 立项 | NON_BLOCKER |
| DOC-9 | RFC-003 内两份 `SchemeSnapshot` 定义并存；Ed25519「防篡改」措辞与 ADR-0006 冲突 | `rfc/scheme-snapshot-and-attestation-model.md` | v0.2 立项时一并修 | NON_BLOCKER |
| DOC-10 | 三份实施口径（引擎 / 墓地门禁 / 试玩自查）把 K-1～K-13、G-MATCH、墓地交叉逻辑各写一遍，已出现 K-9 漂移 | `specs/scheme-patch-engine.md` 等 | 只在 supervisor §14.11 保留一份，其余改指针 | NON_BLOCKER |
| DOC-11 | `supervisor.md` 内部旧节号（§9.5 写「§3.1–§3.5」、L694「§5.4」）与计数（§18 写 35 条、实为 39 条）；intent-compiler few-shot 例 6 用 `remove_item` 处理否决，与 §4.5 冲突 | `specs/supervisor.md`、`specs/intent-compiler-prompt.md` | 机械修复 | NON_BLOCKER |

---

## 纪律

1. 每一项都必须能回答「**谁来做、做完的判据是什么**」。没有判据的不得放进来。
2. 一项被修完后不删除——标记已修复并注明 commit / PR，或随任务卡归档进 Git 历史。
3. 立卡施工时，把卡号回填到对应条目。

## 架构决议落盘记录（2026-10-30 用户拍板）

依据用户在 2026-10-30 的终审裁决（`q12`～`q16`），关闭以下冲突并立项：

| 裁决编号 | 涉及冲突 | 裁决核心 | 落地归宿 |
| :--- | :--- | :--- | :--- |
| **DEC-001** | `conf-1` | 中栏 Supervisor 意图编译组件化，形态开放（不锁死 Claude Code，允许轻量实现） | 立项并合入 [`../adr/0014-three-agent-topology-open-supervisor.md`](../adr/0014-three-agent-topology-open-supervisor.md)；关闭 conf-1 |
| **DEC-002** | `conf-5` | 审阅内容（路径树/命令明细）后置验收幕；保留“确认开工”与“合入”两次点击授权；物理写门禁与保护清单不动 | 回写 `scheme.md` §7；更新运行时签署规范；关闭 conf-5 |
| **DEC-003** | `conf-6` | 允许 Worker 受控只读外网通道，但必须强绑定凭据保险库（Vault）一并实施，账密严禁进入对话上下文 | 登记为 Vault 前置依赖；关闭 conf-6；闭环 goal-2 / oq-3 |
| **DEC-004** | `conf-2, 4` | 右栏剥离写死的架构师人设与 `partner_hint` / `buildProtectionPromptNotice` 前缀，恢复纯粹自由对话 | 关闭 conf-2、conf-4；更新 Claude/Baseline Adapter |
| **DEC-005** | `conf-3, 7` | 中栏仅经画布交互；右栏维持精确只读白名单防穿透 | 关闭 conf-3、conf-7 |
