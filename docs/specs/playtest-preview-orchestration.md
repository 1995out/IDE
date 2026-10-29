# 试玩预览与环境编排工程规格 (playtest-preview-orchestration.md)

> **修订状态**：ACTIVE  
> **关联架构**：`docs/adr/0008-demo-isolated-origin.md`, `docs/specs/playtest-self-check.md`

---

## 1. 目标与定位

代码改完并不代表交付完成。为了实现“一个人能够驾驭 AI 构建的复杂软件”，系统必须在中栏提供**真实可交互、可直接点击把玩的高保真预览环境**，并与主开发环境物理隔离。

---

## 2. 动态端口编排与反向代理机制

```
                                  浏览器 / 用户工作台
                                          │
                                          ▼ :3000 (前端主控)
                   ┌──────────────────────────────────────────────┐
                   │               Daemon Gateway                 │
                   │      /preview/:worktree_id/* 反向代理         │
                   └──────┬────────────────────────────────┬──────┘
                          │                                │
                          ▼                                ▼
            Worktree A 隔离进程 (端口 4101)    Worktree B 隔离进程 (端口 4102)
            ┌───────────────────────────┐    ┌───────────────────────────┐
            │   Vite / TanStack Start   │    │        Next.js / API      │
            │   (无外部网络，本地环境变量)  │    │   (无外部网络，本地环境变量) │
            └───────────────────────────┘    └───────────────────────────┘
```

### 2.1 端口分配与生命周期
1. **端口池管理**：Daemon 维护一个动态预览端口池（默认范围：`4100 ~ 4199`）。
2. **启动与探活（Readiness Probe）**：
   - 当 Worker 施工完成且静态测试通过后，Daemon 在该任务对应的 Worktree 目录下执行合同声明的 `preview_command`；
   - Daemon 每隔 500ms 探测本地端口，直到接收到 HTTP 200 或超过 15 秒超时；
   - 探活成功后，Daemon 将状态置为 `PREVIEW_READY`，并将反向代理挂载至中栏 iframe。
3. **隔离与销毁**：
   - 预览进程与主 Daemon 进程解耦，运行于独立进程组；
   - 用户点击“合入主线”或“放弃本轮”时，Daemon 向进程组发送 `SIGTERM`（超时 3 秒升级为 `SIGKILL`），回收端口与资源。

---

## 3. 前后端联调沙盒与 Mock 降级

对于涉及后端 API 或外部服务的复杂项目，预览环境支持三级降级联调策略：

1. **同构全栈（Fullstack Local）**：若项目为 TanStack Start / Next.js 全栈项目，直接在单一进程中拉起前端与 Server Functions；
2. **API 拦截沙盒（Service Mocking）**：对于需要第三方外部调用的场景，Daemon 自动注入 Mock Service Worker (MSW) 或轻量代理拦截器，保证在断网环境下预览界面依然能够响应交互；
3. **只读数据隔离（Isolated SQLite/LocalDB）**：施工 Worktree 复制一份测试专用本地数据库，杜绝预览改动污染用户主分支数据。
