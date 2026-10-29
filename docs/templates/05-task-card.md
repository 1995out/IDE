# 任务卡模板（满足就绪标准 DoR）

> 复制到 [`../specs/development-tasks.md`](../specs/development-tasks.md) 对应小节，并替换方括号内容。逐条对照 [`../specs/task-card-standard.md`](../specs/task-card-standard.md) §1 的 R-1～R-12；任何一条留空，这张卡都不能开工。样板见 `TASK-J.2`。

```markdown
#### 【Task X.N】[一句话标题] `[NEW_TASK]`

* **任务编号**：`TASK-X.N`
* **任务名称**：[名称]
* **对应规格**：[文件 §节号；scheme.md 条目 id]                       ← R-1
* **前置依赖**：[`TASK-…`（状态）] 或「无」                            ← R-3
* **优先级**：[P0～P5]
* **状态**：`[NEW_TASK]`
* **对照痛点（代码事实）**：[`path::symbol` 今天做了什么、错在哪]    ← R-4
* **一句话目标**：[做完后谁能看见什么变化]                             ← R-2
* **ADR 修订**：[无] 或 [按 ADR-NNNN §x 改：……]                         ← R-9
* **采用的默认值**：[无] 或 [scheme dec-N / OQ-… 的推荐值：……]           ← R-10
* **范围**：                                                             ← R-5
  1. 新建 / 修改 `[path]`：导出 `[symbol(signature)]`，[行为]。
  2. …
* **非目标**：[不做什么；与哪张卡分界]                                 ← R-6
* **不变声明**：[鉴权 / Demo Origin / 点亮三证 / 网关写范围：不变或变更及依据]   ← R-12
* **验收（PASS）**：新增 `[tests/…/x.test.ts]`：                         ← R-7
  - [断言 1]
  - [断言 2]
* **FAIL**：[出现即不合格的反例]                                         ← R-8
* **关键文件**：[path（新）、path（改）、path（只读复用）]
* **估算**：[S / M]（L 必须拆）                                          ← R-11
```
