# 部署、环境适配与持续交付工程规格 (deployment-and-environment.md)

> **修订状态**：ACTIVE  
> **关联架构**：`docs/adr/0012-merged-as-terminal-state.md`

---

## 1. 目标与定位

完成代码合并不是软件生命周期的终点。为了让“一个人 + AI”能真正交付工业级软件，Stagent 必须具备将合并后的代码自动化打包、注入环境变量、部署至目标环境并进行健康自愈的能力。

---

## 2. 部署拓扑与交付阶段

```
代码合入主线 (Squash Merge)
        │
        ▼
   构建产物打包 (Docker / Standalone Build)
        │
        ▼
   环境适配与 Secrets 安全注入 (From Vault)
        │
        ▼
   目标环境发布 (Local Daemon Service / Remote Cloud)
        │
        ▼
   自动化健康探活 (Smoke Test & Health Probe)
        │
        ├── 成功 ──> 交付完成 (Production Live)
        └── 失败 ──> 自动一键回滚 (Rollback to Previous Release)
```

### 2.1 环境变量与凭据注入规范 (Secrets Isolation)
1. **严禁明文进代码**：任何敏感配置（API Keys、数据库连接串、私钥）严禁通过 LLM 对话或 Git 提交传递；
2. **凭据安全绑定（Credential Vault 映射）**：
   - 项目根目录仅保留 `.env.example` 声明配置 Schema；
   - 部署执行器从宿主 Daemon 的安全 Vault 中以环境变量形式注入真实运行时凭据；
   - 部署日志中自动掩码所有匹配敏感正则的字符串（Masking）。

### 2.2 部署目标适配器接口 (Deployment Adapter Interface)

```typescript
export interface DeploymentTargetAdapter {
  target_name: "local_docker" | "static_site" | "cloud_run" | "serverless";
  build(projectRoot: string, contract: SoftwareContractV21): Promise<BuildResult>;
  deploy(artifact: BuildArtifact, envConfig: Record<string, string>): Promise<DeployResult>;
  health_check(endpointUrl: string): Promise<HealthCheckResult>;
  rollback(previousDeploymentId: string): Promise<RollbackResult>;
}
```

### 2.3 自动回滚机制
当部署完成并在健康探活阶段（Health Check）连续 3 次失败或未能在指定超时内响应时，部署管理器自动恢复上一个成功的构建产物，并在中栏输出带有明确故障归因（stdout/stderr 诊断日志）的失败交付卡，保证系统可用性不受破坏。
