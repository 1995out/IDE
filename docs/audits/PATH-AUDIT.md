# Stagent 全链路实现路径审计报告 (PATH-AUDIT.md)

> **审计基线**：Commit `d7e00f5` (PR #2 合并后基线，包含 ADR-0014 与 SchemeDraftV2 确权)  
> **审计标尺**：Stagent 核心使命——从“模糊琐碎的需求”到“高质量可交付软件”的全链路打通，使“一个人 + AI”能驾驭超越个人脑容量的复杂软件系统。

---

## 结论摘要（照当前文档施工，开发 Agent 在哪一环会停摆或做错？）

**当前文档路径在【第 2 环：编译为 Contract】与【第 4 环：隔离施工】之间直接断裂，在【第 6 环：验证与交付】之后完全空白。Agent 绝对无法照现有任务无脑开发。**

具体原因：
1. **意图编译与任务卡断层**：ADR-0014 规定“中栏形态开放、审阅后置、仅保留开工与合入两次点击”，但底层 `specs/`（状态机 D7 门禁）与 `development-tasks.md`（J.2~J.6）依然强制要求用户在开工前审阅并勾选 `ScopePlan` 文件树，开发 Agent 会陷入“按 ADR 简化 vs 按状态机阻断”的死锁。
2. **前后端联调与接口冻结完全空白**：文档只有单 Worker 隔离修改单个文件的设想，对于复杂系统必需的“Schema-First 接口声明 → 前后端并行/依赖施工 → 真实通信联调”没有任何物理落地规范。
3. **真实部署与环境适配完全空白**：文档将“Git Squash Merge”当成了系统终态，对真实软件交付必需的容器化、端口映射、环境配置注入、构建产物部署与健康检查只字未提。
4. **如果现在放任开发**：Agent 会把大量时间花在实现已被废止的反人类前端审批表单上，而在真正的工程联调、测试断言与部署闭环上全靠现场幻觉瞎猜。

---

## 全链路逐环穿透核查矩阵

| 链路环节 | 理论定义与要求 | 状态判定 | 现有设计与代码现状 | 断裂/冲突指针 (file#section) | 落地阻断原因与修复要求 |
| :--- | :--- | :---: | :--- | :--- | :--- |
| **环 1：意图摄取** (Requirement Capture) | 右栏只读自由脑暴、原话完整留存、自然语言提炼 | **[有冲突]** | • 规格声明自由对话（ADR-0014）<br>• 但代码存在写死架构师人设与保护清单强制拼入消息前缀 | • `docs/adr/0014-three-agent-topology-open-supervisor.md#§1`<br>• `daemon/chat/session-manager.ts:226`<br>• `daemon/chat/baseline-adapter.ts:42` | ADR-0014 已经定性剥离人设，但尚未翻译为具体的适配器过滤接口定义；现有代码中非 Claude 适配器仍会污染用户原话。 |
| **环 2：编译为 Contract** (Contract Compilation) | 将满足「可观察、可边界、可冻结、可验证」的意图编译为结构化 SoftwareContract (IR) | **[写了但落不了地]** | • 设计了庞大的 51 码校验与字面子串匹配<br>• 代码事实仅为 6 条正则硬编码 (`MODULE_HINTS`) 与单次 API 调用 | • `docs/specs/intent-compiler-prompt.md#§3`<br>• `docs/specs/scheme-patch-engine.md#§2.2`<br>• `src/lib/journey/scheme-draft.ts:77` | 形式化过于沉重导致实际无法运行，实际运行又过于简陋无法支撑复杂系统拆解；缺乏清晰的“结构化意图编译（Structured Outputs）”最小落地规范。 |
| **环 3：用户授权** (Sign-off & Watershed) | 审阅内容后置，仅保留“确认开工”原子点击授权；安全门禁实时守底 | **[有冲突]** | • ADR-0014 明确“两次点击、审阅后置”<br>• 但历史 specs 强制在开工前校验 `ScopePlan` 并弹窗 | • `docs/adr/0014-three-agent-topology-open-supervisor.md#§3`<br>• `docs/specs/supervisor.md#§15.3, §17 D7`<br>• `docs/specs/state-machine-contract.md#§2` | 状态机代码若未增加 Fast-Path 旁路，前端一点击开工就会因未传 `ScopePlan` 报 `E_SCOPE_APPROVAL_REQUIRED`，按钮死锁。 |
| **环 4：隔离施工** (Orchestrated Execution) | Worktree 物理隔离、Worker 依据子任务 DAG 独立改码；支持接口先行与前后端协作 | **[写了但落不了地]** | • 现有 Worktree 隔离机制完备（ADR-0003）<br>• 但只支持单一 Worker 改动，完全缺失多模块/前后端协同与接口声明机制 | • `docs/rfc/v0.2-multi-agent-pipeline.md#§3.3, §4.1`<br>• `docs/specs/system-architecture.md#§5.1`<br>• `daemon/worker/worktree-manager.ts` | 复杂系统由多个文件和模块组成。若无“Task-0 静态类型先行”与子任务依赖图，单一 Worker 会在前后端代码之间来回涂抹导致破坏性重构。 |
| **环 5：验证出证据** (Deterministic Verification) | 物理执行真实命令、捕获退出码与 stdout/stderr、生成 Evidence Bundle、支持中栏试玩预览 | **[写了但落不了地]** | • 命令行 exit_code 证据链完整（ADR-0005）<br>• 但中栏试玩原型（Playtest Preview）如何动态分配端口、注入环境配置、启动子进程完全未定义 | • `docs/adr/0005-deterministic-verification-gate.md`<br>• `docs/adr/0008-demo-isolated-origin.md`<br>• `docs/specs/playtest-self-check.md#§3` | 单元测试绿了不等于软件可用。对于业务用户，必须在中栏能直接点击交互原型；现有文档缺少“预览生命周期管理与健康检查（Probe）”的工程落地规范。 |
| **环 6：合入交付** (Atomic Git Merge) | 用户点“合入主线”触发原子 Squash Merge，追加审计信息 | **[已写全]** | • Git 引擎原子合入机制完备，仅由 Daemon 裁判执行，封锁主线污染 | • `docs/adr/0012-merged-as-terminal-state.md`<br>• `docs/specs/system-architecture.md#§5.3`<br>• `daemon/pipeline/merge-orchestrator.ts` | 本环是全链路中唯一架构完整且代码已落地的环节。 |
| **环 7：部署与环境适配** (Deployment & Lifecycle) | 从本地合并到真实环境可运行（容器打包、环境变量隔离、持续交付、环境可观测） | **[完全空白]** | • 整个 `docs/` 43 份文档中，“部署”一词仅在模板填空指南中出现一次，没有任何系统架构和工程规格 | • 搜索 `docs/` 全文：零系统架构，零规格描述 | 软件工程的终点是部署可用。完全缺失导致用户依然要手动配置 Docker、服务器、环境变量和域名，击穿了“一个人驾驭复杂软件”的定位。 |

---

## 支撑“一个人驾驭复杂软件”最致命的 3 个设计空白

1. **空白 1：接口契约先行（Schema-First）与前后端联调机制**  
   - **痛点**：单人团队最怕“后端改了字段前端白屏”、“联调只能靠肉眼试错”。  
   - **现状**：现有文档只有粗颗粒度的文件读写权限，没有规定如何从意图中提取 TypeScript/Zod DTO 契约并作为所有子任务的共同依赖（Single Source of Truth）。
2. **空白 2：试玩预览（Playtest Preview）的物理隔离与动态端口编排**  
   - **痛点**：用户点开工后，代码改完了，但中栏依然只是干巴巴的文本报告，看不到也玩不到。  
   - **现状**：ADR-0008 声明了“Demo 隔离源”，但 Daemon 到底如何在后台起 Vite/Next 服务、如何代理端口、如何避免多任务端口冲突、挂了如何拉起，完全是空白。
3. **空白 3：软件部署、环境配置与健康自愈流水线**  
   - **痛点**：本地测试全通，一上线就环境不兼容、数据库连不上、配置丢失。  
   - **现状**：系统只管到 Git Merge 为止。Merge 之后软件如何打包、如何配置生产环境 Secrets、如何执行迁移脚本、如何探活，文档完全缺位。
