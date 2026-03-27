# OPSX 工作流深度分析

> 本文档分析 OpenSpec 的 OPSX 工作流架构、实现机制、优缺点及实际使用中的注意事项。

---

## 核心哲学

**"动作而非阶段"（Actions, not Phases）**

```
传统线性工作流：PLANNING → [门] → IMPLEMENTING → [门] → DONE
                  不能回头

OPSX 流体工作流：  proposal ↔ specs ↔ design ↔ tasks ↔ apply
                  双向箭头表示可随时编辑任何产物
```

---

## 架构组件

### 1. Schema 引擎

**位置：** `OpenSpec/schemas/spec-driven/schema.yaml`

**作用：** 定义产物类型、生成路径、依赖关系

```yaml
name: spec-driven
version: 1
description: Default OpenSpec workflow
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []              # 无依赖，始终可创建

  - id: specs
    generates: "specs/**/*.md"
    requires: [proposal]      # 仅需 proposal 完成

  - id: design
    generates: design.md
    requires: [proposal]      # 与 specs 并行，都只依赖 proposal

  - id: tasks
    generates: tasks.md
    requires: [specs, design] # 需要两者都完成

apply:
  requires: [tasks]
  tracks: tasks.md
```

**产物依赖图（DAG）：**

```
                    proposal
                   (root node)
                       │
         ┌─────────────┴─────────────┐
         │                           │
         ▼                           ▼
      specs                       design
   (requires:                  (requires:
    proposal)                   proposal)
         │                           │
         └─────────────┬─────────────┘
                       │
                       ▼
                    tasks
                (requires:
                specs, design)
                       │
                       ▼
                 ┌──────────────┐
                 │ APPLY PHASE  │
                 └──────────────┘
```

### 2. 产物图系统（Artifact Graph）

**位置：** `src/core/artifact-graph/`

**核心能力：**

| 能力 | 实现方式 |
|------|----------|
| **拓扑排序** | Kahn 算法计算有效构建顺序 |
| **状态检测** | 文件系统扫描（文件存在性） |
| **Ready 查询** | `getNextArtifacts()` 动态计算下一步 |
| **Blocked 查询** | `getBlocked()` 返回未满足的依赖 |

**状态检测逻辑：**

```typescript
// 完成状态通过文件系统存在性判断
type CompletedSet = Set<string>;  // 已完成的 artifact ID

// 检测方式：
- proposal 完成 ⇔ proposal.md 文件存在
- specs 完成 ⇔ specs/**/*.md 匹配到文件
- design 完成 ⇔ design.md 文件存在
- tasks 完成 ⇔ tasks.md 文件存在
```

**状态流转示例：**

```
初始状态：CompletedSet = {}
→ ready: [proposal]

用户创建 proposal.md 后：CompletedSet = {proposal}
→ ready: [specs, design]  ← 两者都 ready，可任选其一

用户先创建 specs 后：CompletedSet = {proposal, specs}
→ ready: [design]  ← design 仍可随时创建

用户创建 design.md 后：CompletedSet = {proposal, specs, design}
→ ready: [tasks]

用户创建 tasks.md 后：CompletedSet = {proposal, specs, design, tasks}
→ ready: [] (全部完成，现在可以 /opsx:apply)
```

### 3. 命令生成器

**位置：** `src/core/command-generation/`

**作用：** 从单一源生成 AI 工具技能文件

**支持的工具：** Claude Code、Cursor、Windsurf、GitHub Copilot 等 20+ 工具

### 4. Delta 规范合并

**机制：** 归档时将 ADDED/MODIFIED/REMOVED 合并到主规范

**示例：**

```markdown
# openspec/changes/add-auth/specs/auth/spec.md

## ADDED Requirements

### Requirement: 两因素认证
系统必须支持 TOTP -based 双因素认证。

## MODIFIED Requirements

### Requirement: 会话过期
系统必须在 15 分钟后使会话过期。（原来是 30 分钟）
```

---

## "任意顺序流转"的真实含义

### 可以灵活排序的场景

| 你希望的顺序 | 系统响应 |
|-------------|---------|
| proposal → specs → design → tasks | ✅ 标准流程 |
| proposal → design → specs → tasks | ✅ 完全合法 |
| proposal → (跳过 design) → tasks | ⚠️ tasks 会显示 blocked，等待 design |
| 直接 tasks | ⚠️ blocked，等待所有前置依赖 |

### 核心约束

> **依赖不可违反，但 DAG 本身不是线性的**

`specs`和 `design` 都只依赖 `proposal`，所以：
- 可以先 specs 后 design
- 可以先 design 后 specs
- 可以同时创建（一次生成两个）

---

## 迭代/回溯机制

### 文档声称

`opsx.md` 第 544-554 行：

```
/opsx:new ───► /opsx:continue ───► /opsx:apply ───► /opsx:archive
                     │                  │
                     │                  ├── "The design is wrong"
                     │                  │
                     │                  ▼
                     │            Just edit design.md
                     │            and continue!
                     │                  │
                     │                  ▼
                     │         /opsx:apply picks up
                     │         where you left off
```

`getting-started.md` 第 215 行：
> "During implementation, if you discover the design needs adjustment, just update the artifact and continue."

### 实际机制

**没有状态机跟踪"执行阶段"**

```
传统工作流引擎：
  PLANNING → [gate] → DESIGN → [gate] → IMPLEMENT
  状态机跟踪当前阶段，支持"回滚到上一节点重新执行"

OPSX：
  文件存在 → 命令读取 → 执行动作
  没有阶段概念，只有"文件当前内容是什么"
```

### 迭代流程

```
1. 用户在 apply 阶段发现设计问题
   ↓
2. 用户手动编辑 design.md（或用对话让 AI 修改）
   ↓
3. 用户继续 /opsx:apply
   ↓
4. apply 重新读取 tasks.md 和 design.md，继续执行
```

### 示例

```text
初始任务列表 (tasks.md)：
- [x] 1.1 创建 ThemeContext
- [ ] 1.2 添加 CSS 变量  ← 进行中

用户发现：design.md 中的技术方案不对

用户操作：编辑 design.md，改为使用 Tailwind

继续 /opsx:apply：
- AI 读取 tasks.md，看到 1.1 已完成 → 跳过 1.1
- AI 读取 design.md，看到新的 Tailwind 方案
- AI 继续执行 1.2，使用 Tailwind 而非 CSS 变量
```

---

## 关键限制：已完成任务的返工问题

### 问题场景

```text
初始状态：
  tasks.md:          design.md:
  - [x] 1.1 已完成    描述方案 A

修改后：
  tasks.md:          design.md:
  - [x] 1.1 已完成    修改为方案 B  ← 影响 1.1
```

**继续 `/opsx:apply` 时：**

```
AI 读取 tasks.md → "1.1 是 [x]，跳过"
AI 读取 design.md → "1.1 应该用方案 B"

结果：1.1 不会被重新执行，代码与设计不一致
```

### 解决方案（用户责任）

**方式 1：手动取消勾选**

```text
用户操作：
1. 修改 design.md
2. 同时修改 tasks.md:
   - [ ] 1.1 创建 ThemeContext (用新方案)  ← 取消勾选
   - [ ] 1.2 添加 CSS 变量
3. 继续 /opsx:apply → AI 重新执行 1.1
```

**方式 2：对话中明确要求**

```text
用户：我修改了 design.md，1.1 的实现方案变了，需要返工
AI：好的，我来帮你取消 1.1 的勾选并重新执行
    [AI 修改 tasks.md，将 1.1 改为 [ ]]
    [AI 重新执行 1.1]
```

**方式 3：使用 /opsx:verify 发现不一致**

```text
用户：/opsx:verify

AI:  Verifying add-dark-mode...

     COHERENCE
     ⚠ Design mentions "方案 B" but
       implementation uses 方案 A (task 1.1)

     Recommendations:
     1. Re-run task 1.1 with new approach
```

### 设计哲学选择

| 设计选择 | OPSX 的做法 | 传统引擎的做法 |
|----------|------------|----------------|
| 状态跟踪 | 文件复选框 `[x]` | 流程实例数据库 |
| 一致性保证 | 用户/AI 负责 | 引擎自动验证 |
| 返工机制 | 手动取消勾选 | 回滚到上一节点 |
| 灵活性 | 完全自由，可任意编辑 | 受流程定义约束 |
| 复杂度 | 极低 | 较高 |

---

## OPSX 优缺点分析

### 优点

| 优点 | 说明 |
|------|------|
| **流体迭代** | 实现过程中可随时更新产物，支持"实现→发现设计问题→更新设计→继续实现"的自然循环 |
| **棕色场优先** | Delta 规范专为修改现有代码设计，避免重写完整规范 |
| **Schema 可扩展** | 通过 YAML 自定义工作流，无需修改 TypeScript 源码 |
| **并行变更安全** | 多个变更文件夹独立存在，归档时检测并解决冲突 |
| **审计追踪完整** | 归档保留 proposal（为什么）、design（如何）、tasks（做了什么） |
| **渐进严谨度** | 简单变更可跳过 design.md，复杂变更可添加研究阶段 |

### 缺点

| 缺点 | 说明 |
|------|------|
| **学习曲线陡峭** | 需理解变更、规范、Delta、产物、Schema 等多个抽象概念 |
| **文档碎片化** | 核心概念分散在 `opsx.md`、`workflows.md`、`concepts.md`、`cli.md` 中 |
| **探索流程未完善** | 探索如何过渡到正式变更仍在讨论中 |
| **工具耦合** | 依赖 AI 助手的技能文件生成，手动维护较繁琐 |
| **命令过载** | Expanded 模式有 9 个命令 + 5 个 CLI 子命令，新用户易混淆 |
| **状态管理隐式** | 产物状态依赖文件系统检测，无集中状态文件 |
| **一致性需手动维护** | 设计变更影响已完成任务时，需手动取消勾选重新执行 |

---

## 最佳实践

### 保持变更聚焦
一个逻辑工作单元对应一个变更

### 善用 `/opsx:explore`
需求不明确时先探索，再决定是否需要 `/opsx:propose`

### 归档前验证
使用 `/opsx:verify` 检查实现是否匹配产物

### 命名清晰
`add-dark-mode` 优于 `feature-1`，便于 `openspec list` 浏览

### 何时更新 vs 新建

| 情况 | 建议 |
|------|------|
| 意图相同，执行细化 | 更新现有变更 |
| 范围缩小（MVP 优先） | 更新现有变更 |
| 学习驱动的修正 | 更新现有变更 |
| 意图根本改变 | 新建变更 |
| 范围爆炸到不同工作 | 新建变更 |
| 原始变更可独立"完成" | 新建变更 |

### 修改产物后的检查清单

```text
修改 design.md 后：

1. 思考：这个修改是否影响已完成的任务？
   ↓ 是
2. 手动将受影响的任务从 [x] 改为 [ ]
   ↓
3. 继续 /opsx:apply

或者更简单：
直接告诉 AI："我修改了 design，需要重新执行 1.1"
```

---

## 命令速查

### Core 模式（默认）

| 命令 | 用途 |
|------|------|
| `/opsx:propose` | 创建变更及所有产物 |
| `/opsx:explore` | 在提交前调查研究 |
| `/opsx:apply` | 实现任务 |
| `/opsx:archive` | 归档并合并规范 |

### Expanded 模式额外命令

| 命令 | 用途 |
|------|------|
| `/opsx:new` | 创建变更脚手架 |
| `/opsx:continue` | 逐个创建产物 |
| `/opsx:ff` | 一次性创建所有产物 |
| `/opsx:verify` | 验证实现与产物的一致性 |
| `/opsx:sync` | 同步 delta 规范到主规范 |
| `/opsx:bulk-archive` | 批量归档多个变更 |
| `/opsx:onboard` | 端到端引导教程 |

---

## 相关文档

- `OpenSpec/docs/opsx.md` — OPSX 工作流架构详解
- `OpenSpec/docs/workflows.md` — 工作流模式指南
- `OpenSpec/docs/concepts.md` — 核心概念：规范、变更、Delta、产物依赖图
- `OpenSpec/docs/cli.md` — CLI 命令完整参考
- `OpenSpec/openspec/specs/artifact-graph/spec.md` — 产物图系统规范
