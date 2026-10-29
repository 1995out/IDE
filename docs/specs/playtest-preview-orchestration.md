# 试玩预览与环境编排工程规格 (playtest-preview-orchestration.md)

> **状态**：PROPOSED — 本文为补充提案，尚未与既有 Schema/ADR 完成冲突裁决
> **关联架构**：`docs/adr/0008-demo-isolated-origin.md`, `docs/specs/playtest-self-check.md`

---

## 1. 目标与定位

代码改完并不代表交付完成。为了实现“一个人能够驾驭 AI 构建的复杂软件”，系统必须在中栏提供**真实可交互、可直接点击把玩的高保真预览环境**，并与主开发环境物理隔离。

---

## 2. 预览集成的待决契约

已批准的安全边界见 `docs/adr/0008-demo-isolated-origin.md` 与 `docs/specs/system-architecture.md#§4.4`：独立 Origin、随机 loopback 端口、`iframe sandbox="allow-scripts"`、不授予 `allow-same-origin`、严格 CSP 与 Origin 拒绝。任何预览编排都必须继承这些限制。

当前需要先作出并验证的工程决定：

1. 中栏试玩是否复用现有 Demo 预览，还是增加“Worker Worktree 真机预览”这一不同模式；
2. 若新增 Worktree 预览，如何安全构建、启动、探活、关闭，如何与 Evidence 中的代码快照绑定；
3. 端口分配/代理、生命周期状态、失败可观测性与资源回收的现有实现分别在哪里，是否已有代码。

本节不指定端口池、反向代理或跨项目框架约定，避免与 ADR-0008 冲突或把未批准实现伪装成既定事实。

### 2.1 最低验收问题

- 预览 URL 明确绑定到本轮合同、Worktree、commit/snapshot 与 Evidence；
- 用户能实际交互，且 iframe 不获得控制台或 Daemon API 权限；
- 预览进程退出、超时、崩溃和用户结束本轮时可观测并可清理；
- 预览所用依赖、数据与环境不得写入主工作区或真实生产数据。

具体实现方式须先与当前 `src/lib/ipc/real-preview-origin.ts`、`src/components/studio/DemoPreviewPane.tsx` 和 ADR-0008 核对。

---

## 3. 前后端联调与 Mock 降级（待与项目支持边界对齐）

对于涉及后端 API 或外部服务的复杂项目，预览环境支持三级降级联调策略：

在确定 Stagent 支持的项目类型后，再定义全栈本地运行、服务 Mock 与测试数据隔离策略。当前不得假定所有被打开的仓库都是 TanStack/Next，也不得自动复制或修改未知数据库。每种 Stub/Mock 必须记录模拟范围、未覆盖行为与转真实实现条件。
