# Stagent 全链路实现路径审计报告 (PATH-AUDIT.md)

> **审计基线**：Commit `d7e00f5` (PR #2 合并后基线，包含 ADR-0014 与 SchemeDraftV2 确权)  
> **审计标尺**：Stagent 核心使命——从“模糊琐碎的需求”到“高质量可交付软件”的全链路打通，使“一个人 + AI”能驾驭超越个人脑容量的复杂软件系统。

---

## 结论摘要（照当前文档施工，开发 Agent 在哪一环会停摆或做错？）

**当前文档已经覆盖了合同、Worktree、验证证据与 Git 合并的目标架构，但不能说“照任务就能无脑开发”：主要阻塞在【意图编译与正式合同的对接】【ADR-0014 与旧审批规格/任务卡的收敛】【试玩预览从隔离 Demo 到 Worker Worktree 成果的界定】。合入主线后的真实部署不在现有系统规格中。**

具体原因：
1. **意图编译到可施工合同口径未收敛**：RFC 定义 `SoftwareContractV20` 与 DAG；架构代码基线仍定义 `SoftwareContractV13`，Supervisor 另有 `SchemeDraftV2`。三者映射及哪一个是运行时授权事实源，文档未统一。
2. **复杂项目的接口冻结与联调闭环不足**：RFC 已有 Schema-First 的 `Task-0` 与 DAG 设计，但尚未看到从业务意图抽取接口、锁定共享 DTO、联调失败归因到返工 Evidence 的完整可执行任务链。
3. **部署范围未定义**：文档的主闭环以 `MERGED` 为终态；部署模板只有占位提示，未定义 Stagent 是否代管发布、支持哪些目标环境，或怎样验证发布成功。
4. **如果现在按任务卡直接开发**：开发者可能继续实现旧版范围审阅 UI，也可能依 ADR-0014 去掉该 UI；由于任务卡与后续裁决未统一，结果不可预测。部署与运行环境问题则需另行定规格，不能由当前任务卡推断。

---

## 全链路逐环穿透核查矩阵

| 链路环节 | 理论定义与要求 | 状态判定 | 现有设计与代码现状 | 断裂/冲突指针 (file#section) | 落地阻断原因与修复要求 |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **环 1：意图摄取** (Requirement Capture) | 右栏需求对话与方案编译的角色、输入与可追溯性 | **[有冲突]** | • ADR-0014 声明右栏不应注入人设和消息前缀<br>• 代码中仍有保护提示注入；备用适配器也有固定提示词 | • `docs/adr/0014-three-agent-topology-open-supervisor.md#§1`<br>• `daemon/chat/session-manager.ts:233`<br>• `daemon/chat/baseline-adapter.ts`（备用适配器提示词，需按当前行号复核） | 设计决策与实现存在落差；此项需要代码施工，不是文档完全空白。 |
| **环 2：编译为 Contract** (Contract Compilation) | 将满足「可观察、可边界、可冻结、可验证」的意图编译为结构化 SoftwareContract (IR) | **[写了但落不了地]** | • RFC 定义 `SoftwareContractV20`、DAG 与授权/验证阶段<br>• 当前代码保留 `SoftwareContractV13` Schema、`SchemeDraftV2` 与意图编译实现，二者映射须核实 | • `docs/specs/intent-compiler-prompt.md#§3`<br>• `docs/specs/scheme-patch-engine.md#§2.2`<br>• `src/lib/journey/scheme-draft.ts`（规则模块提示）<br>• `docs/specs/supervisor.md#§19`（实现落差登记） | 规格包含大规模校验与补丁协议，代码侧可见规则提示与方案状态模型；但从对话到正式 SoftwareContract 的端到端映射、版本兼容和已实现边界没有在一份权威契约中收敛。当前材料不足以断言其“无法运行”或“只有正则”。 |
| **环 3：用户授权** (Sign-off & Watershed) | 审阅内容后置，仅保留“确认开工”原子点击授权；安全门禁实时守底 | **[有冲突]** | • ADR-0014 明确“两次点击、审阅后置”<br>• 但历史 specs 强制在开工前校验 `ScopePlan` 并弹窗 | • `docs/adr/0014-three-agent-topology-open-supervisor.md#§2.2`<br>• `docs/specs/supervisor.md#§15.3–§15.5, §17 D7`<br>• `docs/specs/state-machine-contract.md#§2–§3` | 文档口径不一致：ADR-0014 要求把技术审阅后置并保留安全门禁；supervisor 规格仍将 ScopePlan 纳入确认谓词。代码中是否会因此报错需结合当前请求路径与测试验证，本审计不将特定错误码当作已证实事实。 |
| **环 4：隔离施工** (Orchestrated Execution) | Worktree 物理隔离、Worker 依据子任务 DAG 独立改码；支持接口先行与前后端协作 | **[写了但落不了地]** | • Worktree 隔离、Worker 与 DAG 的目标已有设计<br>• RFC 明确 Schema-First 和 Task DAG；实际落地状态需对照代码与未完成任务卡 | • `docs/rfc/v0.2-multi-agent-pipeline.md#§3.3, §4.1`<br>• `docs/specs/system-architecture.md#§5.1`<br>• `daemon/git/worktree-manager.ts` | RFC 已提出 Task-0 与 DAG，并非完全空白；缺口在于现有实现是否执行同一接口契约，以及联调/失败返工是否有可验收路径。 |
| **环 5：验证出证据** (Deterministic Verification) | 物理执行真实命令、捕获退出码与 stdout/stderr、生成 Evidence Bundle、支持中栏试玩预览 | **[写了但落不了地]** | • Evidence Bundle、确定性验证与 Demo 独立 Origin 均有规格和代码支撑<br>• “试玩”包含静态/生成原型与 Worker Worktree 真机预览等不同含义，其边界与联动未收敛 | • `docs/adr/0005-deterministic-verification-gate.md`<br>• `docs/adr/0008-demo-isolated-origin.md#决策`<br>• `docs/specs/system-architecture.md#§4.4`<br>• `docs/specs/playtest-self-check.md#§3`<br>• `src/components/studio/DemoPreviewPane.tsx`<br>• `src/lib/ipc/real-preview-origin.ts` | 不能把现有 Demo 预览说成空白：已有独立 Origin 与 iframe 安全规范/实现。待决的是它是否必须承载 Worker Worktree 的真实构建成果，以及端口、启动、健康检查和清理怎样与合同生命周期集成。 |
| **环 6：合入交付** (Atomic Git Merge) | 用户点“合入主线”触发原子 Squash Merge，追加审计信息 | **[写了但落不了地]** | • Git 引擎原子合入机制完备，仅由 Daemon 裁判执行，封锁主线污染 | • `docs/adr/0012-merged-as-terminal-state.md`<br>• `docs/specs/system-architecture.md#§5.3`<br>• `daemon/pipeline/merger.ts` | 目标流程与部分合并实现有明确依据；但“已写全并完整运行”的断言仍需检查测试和实际运行证据，因此本审计保守判为未证明完全落地。 |
| **环 7：部署与环境适配** (Deployment & Lifecycle) | 从本地合并到真实环境可运行（容器打包、环境变量隔离、持续交付、环境可观测） | **[完全空白]** | • 未发现覆盖部署全生命周期的系统规格；现有模板仅含部署拓扑占位项 | • 在 docs 中未找到成体系的部署生命周期规格；模板只提供环境/部署拓扑填空项 | 文档没有定义 Stagent 是否负责应用发布部署、支持目标、凭据责任或验证标准；不能据此假定必须支持 Docker、服务器或特定云平台。 |

---

## 支撑“一个人驾驭复杂软件”最致命的 3 个设计空白

1. **空白 1：接口契约先行（Schema-First）与前后端联调机制**  
   - **痛点**：单人团队最怕“后端改了字段前端白屏”、“联调只能靠肉眼试错”。  
   - **现状**：RFC v0.2 §3.3/§4.1 已要求 `Task-0: Type & Contract Declarations` 并共享静态类型；缺的是贯穿业务意图、DTO 变更、前后端联调、失败证据与返工的可执行实现/测试链。
2. **空白 2：试玩预览（Playtest Preview）的物理隔离与动态端口编排**  
   - **痛点**：用户点开工后，代码改完了，但中栏依然只是干巴巴的文本报告，看不到也玩不到。  
   - **现状**：ADR-0008、系统架构 §4.4 与 DemoPreview 代码已定义 Demo 的独立 Origin 安全边界。要将 Worker Worktree 应用作为试玩目标，需明确它与现有隔离 Demo 是否同一功能；不得以新规格覆写 CSP、Origin、iframe sandbox 等安全约束。
3. **空白 3：软件部署、环境配置与健康自愈流水线**  
   - **痛点**：本地测试全通，一上线就环境不兼容、数据库连不上、配置丢失。  
   - **现状**：系统只管到 Git Merge 为止。Merge 之后软件如何打包、如何配置生产环境 Secrets、如何执行迁移脚本、如何探活，文档完全缺位。
