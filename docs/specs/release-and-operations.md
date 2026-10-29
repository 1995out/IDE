# 发布与运维规格（S7 · Release、凭据保险库、线上问题回流）

> **地位**：实施口径，从属 [ADR-0017](../adr/0017-release-lifecycle-after-merge.md)、[ADR-0014](../adr/0014-three-agent-topology-open-supervisor.md) §3（凭据保险库）、[ADR-0006](../adr/0006-local-first-no-tamper-proof.md)（本地优先）。
> **上游意图**：`q17`「部署」；`q14`「必须绑定凭据保险库（Vault）一起实施，账密禁止进入对话上下文」（[`../scheme.md`](../scheme.md) acc-9）。
> **施工卡**：TASK-L.14（凭据保险库）、L.15（发布生命周期与两类目标）、L.16（线上问题回流）。
> **状态**：设计定稿 · 待施工。全库 deploy / release / health_check / vault 为 0 命中。

---

## 0. 一句话

合并的代码要变成能用的软件：Daemon 用用户自己的目标与凭据构建、部署、探活。探活失败自动回到上一个能用的版本。密钥只在保险库和目标进程里出现。

---

## 1. Release 生命周期

```text
RELEASE_DRAFT ──用户点「发布」──▶ BUILDING ──▶ DEPLOYING ──▶ CHECKING_HEALTH ──▶ LIVE
      │                              │             │                │               │
      └──取消──▶（删除草稿）          └──失败──▶ FAILED ◀──失败──┘      └──失败──▶ ROLLING_BACK ──▶ ROLLED_BACK
                                                                                     │
                                                       LIVE ──新发布 LIVE──▶ SUPERSEDED
```

| 状态 | 进入条件 | 退出 |
| :--- | :--- | :--- |
| `RELEASE_DRAFT` | 用户在画布选中一个已合并提交与一个目标 | 用户点「发布」→ `BUILDING`；取消 → 删除 |
| `BUILDING` | 发布被点击 | 构建命令退出码 0 → `DEPLOYING`；否则 → `FAILED` |
| `DEPLOYING` | 构建成功 | 目标适配器 `deploy` 成功 → `CHECKING_HEALTH`；否则 → `FAILED` |
| `CHECKING_HEALTH` | 部署成功 | 探活通过 → `LIVE`；连续 3 次失败或超时 → `ROLLING_BACK`（存在上一个 LIVE 时），否则 → `FAILED` |
| `ROLLING_BACK` | 探活失败 | 停掉新版本，用上一个 LIVE 的导出目录重新部署，探活通过 → 本发布 `ROLLED_BACK`，上一个 LIVE 的记录保持 LIVE、进程重新运行；仍失败 → `FAILED`，并弹「两个版本都起不来」意外卡 |
| `LIVE` | 探活通过 | 同目标的新发布进入 LIVE → `SUPERSEDED` |
| `FAILED` / `ROLLED_BACK` / `SUPERSEDED` | — | 终态 |

**不变式**：
- R-INV-1：同一目标同一时刻至多一个 `LIVE`。
- R-INV-2：同一目标同一时刻至多一个非终态发布。发布不创建 git worktree：检出用 `git archive <sha> | tar -x` 导出到 `.stagent/releases/<release_id>/checkout`，只读 git 对象，不触碰索引与 worktree 元数据，因此不与合同施工争用 git 锁（ADR-0002 背景）。
- R-INV-4：**手动回滚 = 以上一个 LIVE 的 `commit_sha` 新建一次发布**，走完整生命周期；不新增状态迁移。
- R-INV-3：发布只能引用 `MERGED` 合同产生的、位于保护分支上的提交。

---

## 2. 数据模型（`src/lib/release/release.ts`，新建）

```ts
export interface ReleaseRecord {
  schema_version: "1.0.0";
  release_id: string;                 // "rel-20261105-01"
  target_id: string;                  // 见 §3
  commit_sha: string;                 // 保护分支上的提交
  contract_ids: string[];             // 自上一个 LIVE 以来合入的合同（由 Trailer 计算）
  state: ReleaseState;
  previous_live_release_id: string | null;
  steps: Array<{ step: "build" | "deploy" | "health" | "rollback"; argv: string[]; exit_code: number | null; started_at: string; ended_at: string | null; log_tail: string }>;  // log_tail 已脱敏
  created_at: string;
  triggered_by: "USER";
}
```

存储：`.stagent/releases/<release_id>.json`；目标配置：`.stagent/release-targets.json`。

---

## 3. 部署目标

### 3.1 适配器接口（`daemon/release/target-adapter.ts`，新建）

```ts
export interface ReleaseTargetAdapter {
  kind: "local_process" | "user_command";
  build(ctx: ReleaseContext): Promise<StepResult>;
  deploy(ctx: ReleaseContext): Promise<StepResult>;
  health(ctx: ReleaseContext): Promise<StepResult>;
  stop?(ctx: ReleaseContext): Promise<StepResult>;   // local_process 用
}

export interface ReleaseContext {
  release: ReleaseRecord;
  checkoutDir: string;        // git archive 导出的干净目录：.stagent/releases/<release_id>/checkout
  env: Record<string, string>; // deploy / health 步骤：保险库按 env_keys 注入（§4）；install / build 步骤：只有目标配置的明文 build_env
}
```

一次性命令（install、build、`user_command` 部署、健康命令）经 `daemon/runner/command-runner.ts` 执行：argv 形式、`shell: false`、白名单、超时、进程树回收、输出脱敏。`local_process` 的长驻进程用 `daemon/worker/cli-host/process-host.ts::spawnManaged`，因为 `runCommand` 只在进程退出时返回。

**构建前安装**：BUILDING 先运行检测到的安装命令（`npm ci` / `pip install -r requirements.txt` 等），复用全局包缓存的只读侧（ADR-0002）。发布是 Daemon 的对外动作，安装步骤允许访问包管理器注册表。

**密钥只进 deploy 与 health**：install 与 build 运行的是 Worker 写过的脚本，保险库值不注入这两步。

### 3.2 首版两类目标

| 类型 | 用途 | build | deploy | health |
| :--- | :--- | :--- | :--- | :--- |
| `local_process` | 在本机把应用当服务跑起来（个人工具、内网应用、演示） | `package.json` 的 `build` 脚本（无则跳过） | 顺序：停当前 LIVE 进程 → 以 `start` 脚本启动新版本，进程组由 Daemon 保活；端口取目标配置（不与 4100～4110、4200～4210 冲突） | HTTP GET 目标配置的健康路径（默认 `/`），2xx / 3xx 即通过 |
| `user_command` | 用户已有的部署方式（`npm run deploy`、`fly deploy`、`vercel --prod` 等） | 同上 | 执行目标配置中的部署脚本（必须是仓库 `package.json` 中存在的脚本名，或用户在目标配置中显式登记的 argv） | 目标配置的健康 URL（HTTP GET）或健康命令（退出码 0） |

目标配置示例（`.stagent/release-targets.json`）：

```json
{
  "targets": [
    { "target_id": "local", "kind": "local_process", "port": 4300, "health_path": "/" },
    { "target_id": "prod", "kind": "user_command", "deploy_script": "deploy",
      "health_url": "https://example.com/healthz", "env_keys": ["DATABASE_URL", "API_TOKEN"] }
  ]
}
```

`env_keys` 只写变量名；值在保险库里（§4）。

### 3.3 回滚

**自动回滚**（探活失败时）= 用上一个 LIVE 的导出目录重新走 deploy → health。**手动回滚** = 以上一个 LIVE 的 `commit_sha` 新建一次发布（R-INV-4）。

- `local_process`：先停当前进程组，再启动旧版本。
- `user_command`：依赖用户脚本可重复执行。不能的项目在目标配置里声明 `rollback_supported: false`，此时探活失败只进 `FAILED`，并弹意外卡说明需要手动处理。

---

## 4. 凭据保险库（TASK-L.14，ADR-0014 §3）

| 规则 | 实现要求 |
| :--- | :--- |
| V-1 只在保险库存值 | 新建 `daemon/vault/`。存取 API 只在 Daemon 进程内，不经 HTTP 返回明文；前端只能「设置值」「列出变量名」「删除」 |
| V-2 不进模型 | 保险库的值不得进入任何 Worker 指令、Supervisor 输入、右栏会话。注入点只有两个：发布步骤的 `ReleaseContext.env`，以及（M5 时）外网摄取代理（L.17） |
| V-3 不进合同与证据 | 合同只能引用变量名（`execution_permissions` 既有的环境变量名字段）。证据与日志中的值，经现有 `daemon/runner/command-runner.ts::redactSecrets(text, secretValues)` 传入保险库全部值做精确遮蔽；`daemon/core/debug-log.ts` 中另一份 `redactSecrets` 同样改为接受保险库值。现有实现跳过 6 个字符以下的值，所以保险库拒绝存储短于 8 个字符的值（`E_VAULT_VALUE_TOO_SHORT`） |
| V-4 仓库只放声明 | 仓库保留 `.env.example`（变量名 + 说明）。Daemon 在发布前比对：目标 `env_keys` 中有未在保险库设值的 → 发布前置失败，白话列出缺哪几个 |
| V-5 用户输入 | 画布「设置」→「密钥」中输入，掩码显示；不得通过右栏聊天输入。用户在右栏贴出疑似密钥时，右栏不拦截（它是自由对话），但 Supervisor 编译时不得把它写进方案，并在画布提示「请到密钥设置里保存，不要发在聊天里」 |

**OQ-REL-1（L.14 开工前定）**：存储后端二选一：

- **A. 系统钥匙串**：macOS Keychain / Windows Credential Manager / Linux Secret Service；
- **B. 本地加密文件**：`.stagent/vault.enc`，密钥派生自用户口令。

推荐 A：不在磁盘留可被复制的密文，也不需要用户记口令。代价是 Linux 无桌面环境时不可用，需要回退到 B。

---

## 5. 线上问题回流（TASK-L.16）

| 来源 | 进入方式 | 去向 |
| :--- | :--- | :--- |
| `local_process` 进程退出或崩溃 | Daemon 捕获退出码与 stderr 尾部（脱敏） | 画布「线上」卡：「<目标> 停了，最后的错误是…」；用户点「让 AI 看看」 |
| 健康检查周期性失败（LIVE 后每 60 秒探活一次，可关闭） | 同上 | 同上 |
| 用户在右栏说「线上这里不对」 | supervisor §16.5 分流，新增类别 E「线上问题」：消息中出现 LIVE 目标名或「线上 / 上线后 / 生产」等词，且当前无施工中合同 | 同上 |

「让 AI 看看」= 新建一份 `kind: "feature"` 的微合同桩，插到计划最前，并带上：`release_id`、`commit_sha`、脱敏日志尾部、关联的 `contract_ids`。之后走正常合同流程。**禁止**直接在 LIVE 目标上热修，也禁止在 `MERGED` 合同上返工（supervisor §5，「合并后新缺陷开新微合同」）。

---

## 6. 用户触点

| 触点 | 画布位置 | 显示 | 不显示 |
| :--- | :--- | :--- | :--- |
| 选目标 | 「发布」页 | 目标白话名（「本机」「正式站」）+ 上次发布时间 | argv、端口以外的技术参数 |
| **发布** | 同上 | 「把 <N 份改动> 发到 <目标>」+ 缺失密钥提示 | 密钥值 |
| 进度 | 同上 | 构建中 / 部署中 / 检查中 / 已上线 | 原始日志（可展开查看脱敏尾部） |
| 回滚 | LIVE 或 FAILED 卡上 | 「回到 <上一个版本时间>」（= 以上一个 LIVE 的提交新建一次发布） | — |

---

## 7. 测试矩阵

| ID | 场景 | 期望 |
| :--- | :--- | :--- |
| RO-01 | `local_process`：build、start、health 全部成功 | `LIVE`；`steps` 三条退出码 0 |
| RO-02 | 健康检查连续 3 次失败，存在上一个 LIVE | `ROLLING_BACK` → `ROLLED_BACK`；上一个 LIVE 的记录仍为 LIVE，且其进程已重新启动并通过探活 |
| RO-03 | 健康检查失败且无上一个 LIVE | `FAILED` |
| RO-04 | 发布引用非保护分支提交 | 拒绝创建（R-INV-3） |
| RO-05 | 同目标已有 `DEPLOYING` 时再发布 | 拒绝（R-INV-2） |
| RO-06 | 目标 `env_keys` 有一项未设值 | 发布前置失败，白话列出缺失项 |
| RO-07 | 部署脚本把保险库中的值打印到 stdout | `log_tail` 与证据中该值被遮蔽为 `[REDACTED]` |
| RO-08 | Worker 指令、Supervisor 输入全文搜索保险库值 | 0 命中 |
| RO-09 | `user_command` 且 `rollback_supported: false`，探活失败 | `FAILED` + 意外卡；不尝试回滚 |
| RO-10 | LIVE 进程崩溃 | 画布「线上」卡出现；点「让 AI 看看」后，计划在 `nextStub` 所在里程碑最前多出一份 `origin: "system"`、`context.release_id` 已填的合同桩 |
| RO-11 | 构建脚本打印环境变量 | 输出中没有任何保险库值（build 未被注入） |
| RO-12 | 保险库设置 5 个字符的值 | `E_VAULT_VALUE_TOO_SHORT` |

---

## 8. 不在首版

- 云平台原生适配器（容器镜像构建、Kubernetes、Serverless 等）：v0.3+，以新 `kind` 接入 §3.1 接口。
- 数据库迁移编排：首版由用户的 `deploy` 脚本自行处理；不可逆迁移在 S4 已经升级为用户决策（technical-design-and-knowledge §3）。
- 多环境晋级（staging → prod）：首版用两个独立目标表达，不做自动晋级。
