# 交付全链路审计：从「模糊琐碎需求」到「高质量可交付软件」

> **日期**：2026-10-30（Asia/Hong_Kong）· **基线**：`main` @ `d7e00f5`（含 PR #1、ADR-0014 与 q12～q16 裁决）
> **标尺**：用户 2026-10-30 提出的目标能力（原话见 [`../scheme.md`](../scheme.md) §0 `q17`～`q20`）——把需求理解、技术方案、功能实现、测试、部署串起来；把大型 AI 软件项目变成一组**能理解、能验收、能追踪、能维护**的可控任务；复杂度超过个人脑容量后，一个人仍能掌握整个系统。
> **方法**：通读 `docs/` 全部文档；对每个环节核对代码事实（`file:line` 均以本基线为准）；抽查任务卡能否被 Agent 直接执行。
> **本报告是 L4 证据层**：它只陈述现状与缺口。补全内容的权威落点见 §6。

---

## 0. 结论

1. **没有写全。** 现有文档把**一轮改动**（一份合同）从意图到合并写得很深，深度在业界少见。但目标能力要求的是**一个项目**的全生命周期。项目级分解、技术方案、测试义务、发布部署、系统知识维护这五环，文档要么空白，要么只有一句话。
2. **Agent 现在不能照任务无脑开发。** 原因不是卡写得差（J.2 这类卡已达到可执行标准），而是三件事：
   - **卡与已裁决的 ADR-0014 冲突**。J.3、J.5、J.6、J.7 仍按旧口径设计：开工前逐项表态，右栏插入系统消息。照卡做会造出被否决的东西。
   - **主路径上有五环没有卡**：项目分解、技术方案、测试义务、发布、知识库。Agent 只能现场发明。
   - **没有「卡就绪」标准**。卡的质量全凭作者自觉，Agent 无法判断一张卡是否可以开工。
3. **按影响排序的五个致命缺口**：
   - (1) 新功能默认不带测试：网关拦截测试文件写入。
   - (2) 没有项目级计划：大需求无法拆成可控的合同序列。
   - (3) 目标项目没有系统知识库：复杂度一过个人脑容量，就失去对系统的掌握。
   - (4) 合并之后没有发布：软件停在本地分支，到不了能用的环境。
   - (5) 预览不运行用户的应用：试玩看到的是静态页或占位页。

---

## 1. 评估标尺

**七个环节**（用户原话「需求理解、技术方案、功能实现、测试和部署」展开）：

| 环 | 名称 | 用户原话依据 |
| :--- | :--- | :--- |
| S1 | 需求摄取（右栏脑暴 + 原话留存） | q17「需求理解」 |
| S2 | 方案编译（意图 → 可审阅方案） | q17「需求理解」 |
| S3 | 项目分解（方案 → 里程碑 → 合同序列） | q18「变成一组…可控任务」 |
| S4 | 技术方案（每份合同的设计、接口、数据变更） | q17「技术方案」 |
| S5 | 施工（Worktree 内实现） | q17「功能实现」 |
| S6 | 验证与验收（测试、证据、试玩、合并） | q17「测试」、q18「能够验收」 |
| S7 | 发布与运维（构建、环境、部署、回滚、线上问题回流） | q17「部署」 |

**四个控制属性**（q18）：能理解、能验收、能追踪、能维护。另有一条贯穿要求（q19、q20）：系统复杂度超过个人脑容量后，用户仍能掌握全局。

**「无脑开发」的判定**：一张卡能被 Agent 直接执行，当且仅当它满足 [`../specs/task-card-standard.md`](../specs/task-card-standard.md) 的就绪定义（DoR）。DoR 要求卡写明：输入、改哪些文件、接口签名、测试断言、PASS/FAIL、依赖、非目标；并且卡不与任何 ACCEPTED 的 ADR 冲突。

---

## 2. 逐环核查

图例：✅ 写全且有代码 · 🟡 写了但落不了地 / 有冲突 · ❌ 空白

| 环 | 文档覆盖 | 代码事实 | 判定 | 缺口 |
| :--- | :--- | :--- | :---: | :--- |
| S1 需求摄取 | supervisor §13.2、§14.5；ADR-0014 §1；J.1 | 右栏 Claude Code 只读（`daemon/worker/cli-host/claude-code-adapter.ts` `PARTNER_TOOLS`）。`BaselineChatAdapter` 写死 system 提示（`daemon/chat/baseline-adapter.ts` `system:`）；保护清单拼进用户消息（`daemon/chat/session-manager.ts` `buildProtectionPromptNotice`） | 🟡 | DEC-004 已裁决剥离，但**没有卡**落地 |
| S2 方案编译 | supervisor §14（`SchemeDraftV2` / `SchemePatch` / 台账）；intent-compiler-prompt；scheme-patch-engine | `rebuildSchemeFromCorpus` 只靠 6 条 `MODULE_HINTS` 正则（`src/lib/journey/scheme-draft.ts:114`）；无模型编译；引擎文件不存在 | 🟡 | 规格完备，实现 0%（J.9、J.10）；J.10 与 DEC-004 冲突（`partner_hint`） |
| S2→合同 | supervisor §15（`ScopePlan`）；runtime §1～§2 | **`ScopePlan` / `plan_digest` 全库 0 命中**。合同范围由 `buildL0Plan` 从聊天文本推出（`daemon/supervisor/rules-l0.ts:388`），方案模块**不进入**合同 | 🟡 | J.2～J.6 未施工；J.3/J.5/J.6 与 ADR-0014 §2 冲突 |
| S3 项目分解 | **无**。RFC v0.2 §4 有 `SubTaskSpec` 设想，状态 DEFERRED | 任务图一份合同一张，节点从合同路径与验收条目生成（`src/lib/journey/proposal-from-contract.ts:42`）；epic / milestone / roadmap / project plan **0 命中** | ❌ | 没有项目级计划、合同序列、跨合同依赖、进度视图 |
| S4 技术方案 | **无**。方案只到产品层（模块、流程、实体）；技术选型散落在 Worker 自由发挥 | 无设计产物；`daemon/arch/*` 只做按需依赖图展示（`GET /api/arch/graph`），不落盘 | ❌ | 没有设计说明、接口先行、数据变更规则、技术决策记录 |
| S5 施工 | runtime §1～§5；ADR-0003、0007、0013；W 系列 | `AiWorker`（OpenAI）经网关写盘（`daemon/worker/ai-worker.ts:205`）；`AdapterRegistry` **未接线**，Claude Code 只作右栏 | ✅（单合同）/ 🟡 | 指令不含验证命令（`daemon/worker/session-task-injector.ts:227`）；Claude Code Worker 未接线 |
| S6 测试 | ADR-0005；evidence-and-acceptance-criteria（草案）；playtest-self-check | 验证命令来自合同，L0 按标记文件推断（`rules-l0.ts:31`）。**写测试文件默认被拦**（`daemon/mcp/gateway.ts:1089` `E_TEST_FILE_WRITE_BLOCKED`）；无测试生成；回归清单只针对 Stagent 自身（`daemon/pipeline/mainline-regression.ts`） | ❌ | 新功能默认无测试；验收条目不派生测试义务；目标项目无回归积累 |
| S6 试玩 | ADR-0008；supervisor §16 | Demo 静态服务（4100～4110）；真机预览只服务静态目录或登记已有 URL，否则写占位页（`daemon/supervisor/preview/real-preview-server.ts`，4200～4210；`daemon/server.ts:861`）。**没有任何代码在 Worktree 里启动用户应用** | 🟡 | 预览不跑用户应用；无健康探测 |
| S6 合并 | ADR-0012；merger | squash 合并 + 9 个 Trailer（`daemon/pipeline/merger.ts:204`） | ✅ | `DELIVERED` 仍由 `sign-gc.ts` 写入（DOC-6） |
| S7 发布与运维 | **无**（全库仅模板提到「发布」） | deploy / docker / artifact / health_check / vault **0 命中** | ❌ | 合并即终点；无构建、环境、密钥、部署、回滚、线上问题回流 |

---

## 3. 控制属性核查

| 属性 | 现有支撑 | 缺口 |
| :--- | :--- | :--- |
| **能理解** | 方案画布、白话五块、四幕旅程 | 只覆盖「这一轮」。用户看不到整个系统的模块地图、技术决策与演进历史；目标项目没有任何持续维护的系统描述 |
| **能验收** | 证据绑定 SHA、`can_merge` 由退出码派生、试玩 | 验收条目与测试没有绑定；新功能可以「零测试通过」；`manual_only` 项目只能靠试玩 |
| **能追踪** | Trailer 把提交连到合同与证据；方案条目带原话证据 | 链条在「方案条目 → 合同」处断开（范围从聊天文本推断）；没有「这段代码为什么存在」的反查入口 |
| **能维护** | ADR 机制（只用于 Stagent 自身）；records 审计包 | 目标项目没有 ADR / 模块地图；线上问题没有回流路径；没有「改这里会影响什么」的分析 |
| **超过脑容量仍可掌握** | — | 依赖以上四项。四项都只在单合同尺度成立，项目尺度全部缺位 |

---

## 4. 任务卡可执行性抽查

| 卡 | 就绪？ | 理由 |
| :--- | :---: | :--- |
| J.2 `ScopePlan` 纯函数 | ✅ | 文件、函数签名、收敛规则、三类夹具、PASS/FAIL 齐全；可作 DoR 样板 |
| J.3 确认谓词纳入 `ScopePlan` | 🟡 | 「有未表态敏感项 → `E_SCOPE_APPROVAL_REQUIRED`」与 ADR-0014 §2「只保留两次点击」冲突 |
| J.5 确认接线 | 🟡 | 依赖 J.3 的逐项表态 |
| J.6 范围区块 | ❌ | 「开工前看清并能调整」「每项可以/不行」与 ADR-0014 §2「审阅后置」直接冲突 |
| J.7 交接与右栏解锁 | 🟡 | 「右栏插入系统消息」与 DEC-005「中栏仅经画布交互」冲突 |
| J.10 意图编译器 | 🟡 | 依赖 `partner_hint` 双调用设计，DEC-004 已裁决剥离 |
| S3、S4、S6 测试义务、S7 | ❌ | 没有卡 |

**修正方式**：本次在各卡加「ADR-0014 修订」块，写明改什么。新增 L 系列卡覆盖空白环，见 §6。

---

## 5. 对 PR #3 草案的事实核对

PR #3（`docs/complete-stagent-delivery-pipeline`）与本审计方向一致：都认为 S3、S7 空白，预览不跑用户应用。但它有几处与代码事实不符，若合入会把 Agent 带偏：

| PR #3 的写法 | 代码事实 |
| :--- | :--- |
| 合并引擎 `daemon/pipeline/merge-orchestrator.ts` | 不存在；实际是 `daemon/pipeline/merger.ts` |
| Worktree `.stagent/worktrees/wt-<id>/` | 实际是 `.stagent/worktrees/task-<contractId>`，分支 `task/<id>`（`daemon/git/worktree-manager.ts:216`） |
| 证据 `.stagent/evidence/evidence-<id>.json` | 实际是 `.stagent/evidence/<contractId>-<evidenceId>.json`（`daemon/pipeline/orchestrator.ts:1555`） |
| 合同状态 `PENDING_USER_APPROVAL` | 生命周期 9 态里没有；对应的是 `PENDING_APPROVAL` + approval `PENDING_REVIEW` |
| 新建 `SoftwareContractV21`（`schema_version: "2.1.0"`） | 现行 `contract.schema.ts` 为 1.3.1。另起一套并行 Schema 会造成双真相源；正确做法是在现有 Schema 上演进 |
| 预览端口池 4100～4199 | 与 Demo（4100～4110）、真机预览（4200～4210）现有端口段冲突 |
| 「彻底旁路掉 `ScopePlan`」 | ADR-0014 §2 只后置**审阅**，没有取消范围计算。`writable_paths` 与 `never_touch` 仍须物理生效，而它们正来自 `ScopePlan` |

本次补全没有基于 PR #3 改写，而是按上述代码事实重写。PR #3 中「Software Contract 四要素」「黄金切片」两个好想法已吸收：前者进 [`../specs/delivery-lifecycle.md`](../specs/delivery-lifecycle.md) §3「可编译判定」，后者成为里程碑 M2 的退出标准。

---

## 6. 本次补全

| 缺口 | 补全落点 |
| :--- | :--- |
| 七环没有一条主干把它们串起来 | [`../specs/delivery-lifecycle.md`](../specs/delivery-lifecycle.md)：七环的输入、输出、门禁、用户触点、负责方；项目 → 里程碑 → 合同 → 节点四级模型 |
| S3 项目分解 | [`../specs/project-decomposition.md`](../specs/project-decomposition.md) + [ADR-0015](../adr/0015-contract-as-unit-of-delivery.md) |
| S4 技术方案 + 系统知识库 | [`../specs/technical-design-and-knowledge.md`](../specs/technical-design-and-knowledge.md) + [ADR-0016](../adr/0016-system-knowledge-base-in-repo.md) |
| S6 测试义务 | [`../specs/verification-strategy.md`](../specs/verification-strategy.md) |
| S7 发布与运维 | [`../specs/release-and-operations.md`](../specs/release-and-operations.md) + [ADR-0017](../adr/0017-release-lifecycle-after-merge.md) |
| 卡就绪没有标准 | [`../specs/task-card-standard.md`](../specs/task-card-standard.md) + [`../templates/05-task-card.md`](../templates/05-task-card.md) |
| 卡与 ADR-0014 冲突 | [`../specs/development-tasks.md`](../specs/development-tasks.md) J.3 / J.5 / J.6 / J.7 / J.10 的「ADR-0014 修订」块 |
| 空白环没有卡 | [`../specs/development-tasks.md`](../specs/development-tasks.md) §3.4d L 系列（L.1～L.17）+ 里程碑 M2～M5 |

## 7. 补全后仍不能「无脑」的部分（诚实声明）

- **ADR-0015～0017 需要用户确认。** 它们随本 PR 提交，合入即生效。若否决其中任何一篇，对应 L 卡自动阻断。
- **OP-1（确认路径 `occurrence` 洗白）仍待用户裁决。** J.9 开工前必须定。
- **SA-1（ADR-0014 下敏感项如何授权）采用推荐默认值**：只有用户原话点名的敏感项才进入授权清单，并入「确认开工」一次点击，授权存在 `ScopePlan` 里。见 delivery-lifecycle §5。用户可在 PR 中改判。
- **部署目标首版只覆盖「本地运行」与「用户自有命令」两类**。云平台适配器留到 v0.3+，见 release-and-operations §8。
- **凭据保险库的存储后端（系统钥匙串 vs 本地加密文件）留作 OQ-REL-1**：L.14 开工前定，但不阻断 M2、M3。
