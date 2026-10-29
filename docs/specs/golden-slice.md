# Phase 2 黄金切片施工契约 (golden-slice.md)

> **修订状态**：APPROVED (严格遵守 ADR-0014 宪法，作为当前代码改造第一优先级)  
> **关联架构**：`docs/adr/0014-three-agent-topology-open-supervisor.md`, `docs/audits/PATH-AUDIT.md`

---

## 1. 切片场景与边界定义

### 1.1 场景描述
**用户在右栏输入一句话需求（如“在顶栏加一个深色模式切换按钮”） ➔ 点击一次“确认开工” ➔ Worktree 内 Worker 施工改码 ➔ 中栏直接拉起可点击试玩预览 ➔ 用户点击一次“合入主线”原子合入。**

### 1.2 零外部依赖
- 严禁调用真实外网爬虫工具（WebFetch/WebSearch 保持断开）；
- 严禁涉及账号密码凭据库（Vault）；
- 仅操作当前本地 Vite/TanStack 仓库。

---

## 2. 模块真实度与 Stub / Mock 矩阵

为了快速跑通主干骨架，本切片严格划分真实工作与模拟存根模块：

| 模块环节 | 实现策略 | 假在哪里 (Stub / Mock 细节) | 何时必须换成真实现 (Next Phase) |
| :--- | :---: | :--- | :--- |
| **右栏自由对话** | **真实** | 彻底剥离写死的架构师人设与保护清单消息前缀，保持白板对话。 | 当前切片即为真实现。 |
| **中栏意图编译** | **轻量真实** | 采用单次 LLM 结构化输出（JSON Schema 提取），不依赖多轮复杂交互。 | 当出现跨模块多分歧冲突时，升级为交互式编译。 |
| **前置开工审批** | **真实重写** | **审阅内容彻底后置**：彻底旁路掉老的 `ScopePlan` 路径树弹窗与敏感路径勾选；保留“确认开工”原子点击。 | 当前切片即为终态标准。 |
| **Worktree 分配** | **真实** | 复用现有的 `WorktreeManager`，在独立工作区检出分支。 | 当前切片即为真实现。 |
| **Worker 施工** | **真实** | 依据子任务 prompt 修改指定前端组件，物理限制在 `src/` 白名单内。 | 当前切片即为真实现。 |
| **确定性验证** | **真实** | Daemon 沙盒真实执行 `npm run typecheck` 或轻量测试，捕获退出码。 | 当前切片即为真实现。 |
| **中栏试玩预览** | **轻量真实** | Daemon 在 Worktree 下拉起当前 Vite 开发端口，中栏 iframe 直接映射本地 URL。 | 遇到复杂多端口全栈或后端微服务时升级为反向代理池。 |
| **原子合入** | **真实** | 复用现有的 `MergeOrchestrator` 执行 Squash Merge。 | 当前切片即为真实现。 |

---

## 3. 验收标准与可观测物理证据 (Acceptance Criteria)

整个切片推进过程中，每一步必须产出确定性物理证据：

1. **意图编译落盘证据**：
   - 用户发送一句话后，`.stagent/contracts/contract-<id>.json` 生成，状态为 `PENDING_USER_APPROVAL`，包含 `writable_paths: ["src/**"]`。
2. **授权流转证据**：
   - 用户在 UI 上仅点击一次按钮，合同 `approval.status` 转为 `APPROVED`，不抛出 `E_SCOPE_APPROVAL_REQUIRED`。
3. **施工隔离证据**：
   - `.stagent/worktrees/wt-<id>/` 成功创建并包含独立 git 分支，Worker 仅在此目录下写入。
4. **退出码证据**：
   - 验证执行器产出 `.stagent/evidence/evidence-<id>.json`，其中 `exit_code: 0` 且 `verdict.can_merge === true`。
5. **试玩可交互证据**：
   - 中栏渲染 iframe，加载 `http://localhost:<preview_port>`，用户可实际点击切换按钮。
6. **最终合入证据**：
   - 点击“合入”后，本地 `main` 分支生成 Squash commit，包含 `Stagent-Contract-ID` 审计 Trailer。

---

## 4. 实施顺序：4 个原子递进 PR 路线图

- **PR-1: 审批门禁降级与开工 Fast-Path (契约去阻断)**  
  - 改造 `evaluateSchemeConfirmation` 与状态机，跳过 `ScopePlan` 必须传参校验，实现审阅内容后置；
  - **验收语句**：前端仅传入合同 ID 即可成功调用开工 API，返回 200 且状态推进为 `EXECUTING`。
- **PR-2: 右栏纯白板适配器与轻量意图编译挂接**  
  - 清理 `daemon/chat/session-manager.ts` 的前缀注入，接入轻量意图结构化提取器；
  - **验收语句**：右栏输入单句需求后，中栏 3 秒内自动渲染出包含可观察目标的一键开工卡片。
- **PR-3: Worktree 本地预览拉起与中栏试玩挂载**  
  - 实现本地开发服务自动探活并在中栏以 iframe 呈现试玩界面；
  - **验收语句**：Worker 施工完成后，中栏自动切换为可点击交互的组件预览。
- **PR-4: 端到端黄金闭环集成测试**  
  - 编写一条完整的自动化端到端测试用例，串联从意图输入到 Git 合入的完整流水线；
  - **验收语句**：单测试用例执行完毕后，主干分支产生对应 Git 提交且整个过程无需人工技术决策。
