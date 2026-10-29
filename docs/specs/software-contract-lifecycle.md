# Software Contract 核心第一性原理与生命周期契约 (software-contract-lifecycle.md)

> **状态**：PROPOSED — 概念说明，不取代 `SoftwareContractV13`、`SoftwareContractV20` 或 `SchemeDraftV2`；版本关系须另行裁决。
> **关联架构**：`docs/adr/0014-three-agent-topology-open-supervisor.md`, `docs/rfc/v0.2-multi-agent-pipeline.md`

---

## 1. 核心第一性原理 (The Stagent Axiom)

> **“任何满足「可观察、可边界、可冻结、可验证」的业务意图，都可以被编译为 Software Contract；Software Contract 可以授权 Agent 在受控范围内生成、修改、验证软件，并输出 Evidence Bundle。”**

### 1.1 四要素的操作化问题（不是已批准的充要谓词）

RFC v0.2 使用这四个词描述可编译意图，但当前权威文档没有给出完整的充要谓词。下表是供规格收敛使用的候选定义，不可作为已实现的阻断规则：

| 要素 | 核心含义 | 形式化判定谓词 (Compiler Predicate) | 反例（不可编译） |
| :--- | :--- | :--- | :--- |
| **可观察** (Observable) | 变更必须对外界（UI、API、系统输出、退出码）产生可直接感知的物理表现。 | `has_observable_output === true`<br>（明确存在可见 UI 状态、HTTP 状态码/响应体、日志流或退出码） | “把代码写得更好看一点”、“重构一下让逻辑更优雅”（无物理可观测表现） |
| **可边界** (Bounded) | 改动范围在空间（路径白名单）和资源（时间、命令执行次数、模型预算）上可严格限定。 | `writable_paths.length > 0 && budget_limits.is_finite === true`<br>（显式白名单与熔断预算） | “把整个系统查一遍随便改改”、“遇到问题能调的都调一下”（无限范围） |
| **可冻结** (Freezable) | 关键业务主张与接口类型已闭环，不存在影响施工拓扑的核心未决分歧（Blocking Conflicts）。 | `unresolved_blocking_conflicts.length === 0`<br>（分歧已由用户白话拍板） | “到底用 MySQL 还是 Mongo 还没定，你看着办先写”（底层拓扑未决） |
| **可验证** (Verifiable) | 存在确定性的程序化断言手段，能够由 Daemon 沙盒独立运行判定成败。 | `verification_commands.length > 0 || playtest_spec !== null`<br>（自动化命令或试玩验收断言） | “帮我看看感觉对不对”、“人工体验一下好不好用”（缺乏确定性判据） |

---

## 2. Software Contract 的三重法定属性

Contract 目标上承接用户意图、施工授权与确定性验收，但仓库当前同时存在 `SoftwareContractV13`、RFC 中的 `SoftwareContractV20` 和 UI 层 `SchemeDraftV2`。它们是否同一对象、如何映射、哪一个是唯一运行时授权事实源，必须由后续 ADR/Schema 变更决定；本文件不宣称已完成统一。

```
                    ┌─────────────────────────┐
                    │    自然语言需求 / 脑暴    │
                    └────────────┬────────────┘
                                 │ 意图编译组件
                                 ▼
                    ┌─────────────────────────┐
                    │   Software Contract     │
                    │   (唯一的中心授权 IR)   │
                    └────┬────────┬────────┬──┘
                         │        │        │
         ┌───────────────┘        │        └────────────────┐
         ▼                        ▼                         ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  给用户的确认单   │    │ 给 Worker 的沙盒 │    │  给 Daemon 的    │
│  (意图与验收标准) │    │ 权限字典与 DAG   │    │  验收考卷与基准  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

1. **对用户（法律确认单）**：  
   用户在开工前仅需审阅并确认“意图描述是否准确、预期验收表现是否符合心智”。**审阅内容分级委托**：技术路径、依赖图、编译参数后置，用户签字即完成开工法定授权。
2. **对 Worker（沙盒执行凭证）**：  
   Worker 在 Worktree 隔离区启动时，注入其上下文的唯一硬契约。Worker 绝不能自行修改合同；其 `allowed_files` 为文件系统物理硬边界，超出即被系统层掐断。
3. **对 Daemon 裁判（确定性考卷）**：  
   所有子任务执行完毕后，Daemon 依据合同内锁定的 `verification_spec` 启动真实 Runner 执行命令，严禁任何 LLM 参与批卷。

---

## 3. Schema 演进待决项（本文件不新增运行时 Schema）

当前可查到的基线：

- `SoftwareContractV13`：`src/lib/contract.schema.ts` 与 `docs/specs/system-architecture.md#§9.1`；
- `SoftwareContractV20`：`docs/rfc/v0.2-multi-agent-pipeline.md#§4.1` 的目标演进；
- `SchemeDraftV2` / `SchemePatch`：Supervisor 的方案 UI 与意图编译表示。

在完成兼容策略前，不应新增名为 `SoftwareContractV21` 的第二套运行时类型，也不应让它替代既有合同 Schema。需在后续决策中明确：迁移版本、草案到已签合同的映射、用户授权记录、子任务授权子集、验证命令和 Evidence 的合同绑定。
