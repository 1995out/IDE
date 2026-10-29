# Software Contract 核心第一性原理与生命周期契约 (software-contract-lifecycle.md)

> **修订状态**：ACTIVE (依据 ADR-0014 补充，确立 Stagent 全链路基石)  
> **关联架构**：`docs/adr/0014-three-agent-topology-open-supervisor.md`, `docs/rfc/v0.2-multi-agent-pipeline.md`

---

## 1. 核心第一性原理 (The Stagent Axiom)

> **“任何满足「可观察、可边界、可冻结、可验证」的业务意图，都可以被编译为 Software Contract；Software Contract 可以授权 Agent 在受控范围内生成、修改、验证软件，并输出 Evidence Bundle。”**

### 1.1 业务意图形式化四要素充要判定

一个业务意图只有同时满足以下四项判定谓词，意图编译器才允许将其固化为 `SoftwareContract`：

| 要素 | 核心含义 | 形式化判定谓词 (Compiler Predicate) | 反例（不可编译） |
| :--- | :--- | :--- | :--- |
| **可观察** (Observable) | 变更必须对外界（UI、API、系统输出、退出码）产生可直接感知的物理表现。 | `has_observable_output === true`<br>（明确存在可见 UI 状态、HTTP 状态码/响应体、日志流或退出码） | “把代码写得更好看一点”、“重构一下让逻辑更优雅”（无物理可观测表现） |
| **可边界** (Bounded) | 改动范围在空间（路径白名单）和资源（时间、命令执行次数、模型预算）上可严格限定。 | `writable_paths.length > 0 && budget_limits.is_finite === true`<br>（显式白名单与熔断预算） | “把整个系统查一遍随便改改”、“遇到问题能调的都调一下”（无限范围） |
| **可冻结** (Freezable) | 关键业务主张与接口类型已闭环，不存在影响施工拓扑的核心未决分歧（Blocking Conflicts）。 | `unresolved_blocking_conflicts.length === 0`<br>（分歧已由用户白话拍板） | “到底用 MySQL 还是 Mongo 还没定，你看着办先写”（底层拓扑未决） |
| **可验证** (Verifiable) | 存在确定性的程序化断言手段，能够由 Daemon 沙盒独立运行判定成败。 | `verification_commands.length > 0 || playtest_spec !== null`<br>（自动化命令或试玩验收断言） | “帮我看看感觉对不对”、“人工体验一下好不好用”（缺乏确定性判据） |

---

## 2. Software Contract 的三重法定属性

`SoftwareContract` 是系统从“自然语言对话”进入“受控代码施工”的**唯一中介表示 (IR)**，它具有三位一体的法定属性：

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

## 3. 契约数据结构定义 (`SoftwareContractV21`)

```typescript
export interface SoftwareContractV21 {
  schema_version: "2.1.0";
  contract_id: string;             // 唯一合同编号，如 "contract-20261030-darkmode"
  title: string;                   // 业务功能白话标题
  intent: {
    user_prompt_summary: string;  // 用户原始意图提炼
    q_refs: string[];              // 绑定的原话引用编号（q12, q13...）
    observable_behaviors: string[];// 可观察行为清单
  };
  authorization: {
    granted_by: "USER";
    granted_at: string;            // ISO 8601 时间戳
    status: "APPROVED" | "PENDING" | "REJECTED";
    writable_paths: string[];      // 物理写权限白名单（Glob 格式）
    forbidden_paths: string[];     // 物理绝对禁区（如 .git, daemon/, keys）
  };
  task_dag: {
    tasks: SubTaskDefinition[];    // 细粒度子任务拓扑（Schema-First 机制）
  };
  verification_spec: {
    commands: Array<{
      cmd: string;
      argv: string[];
      timeout_seconds: number;
    }>;
    playtest: {
      required: boolean;
      preview_command?: string;    // 如 "npm run dev"
      preview_port?: number;       // 动态分配的端口
      route_path?: string;         // 验收 URL 路径
      expected_elements: string[]; // 试玩验收断言提示
    };
  };
  budget_limits: {
    max_duration_seconds: number;
    max_model_tokens: number;
    max_cost_cents?: number;
  };
}

export interface SubTaskDefinition {
  task_id: string;                 // 如 "task-0-types", "task-1-frontend"
  title: string;
  kind: "SCHEMA" | "BACKEND" | "FRONTEND" | "TEST" | "CONFIG";
  depends_on: string[];            // 前置依赖 ID 数组
  worker_prompt: string;           // 针对该独立 Worker 的精准施工提示
  assigned_paths: string[];        // 本子任务严格限制的写路径
}
```
