# CLAUDE.md

本文档为 Claude Code (claude.ai/code) 在此代码仓库中工作提供指导。

## 仓库概览

Superspec2 是一个 monorepo，包含两个互补的 AI 原生开发框架：

| 项目 | 用途 | 位置 |
|------|------|------|
| **OpenSpec** | 基于 delta 规范的规范驱动开发和变更追踪 | `OpenSpec/` |
| **Superpowers** | AI 代理的可组合技能库（TDD、调试、协作） | `superpowers/` |

两个项目均使用 MIT 许可证，既可协同工作也可独立使用。

---

## OpenSpec


### 架构

```
OpenSpec/
├── src/
│   ├── cli/                  # CLI 入口 (基于 commander)
│   ├── core/                 # 核心业务逻辑
│   │   ├── artifact-graph/   # 产物依赖图
│   │   ├── command-generation/# AI 工具技能生成器
│   │   ├── completions/      # Shell 自动完成生成器
│   │   ├── schemas/          # 工作流模式定义
│   │   ├── templates/        # 技能和工作流模板
│   │   └── parsers/          # Markdown/规范解析器
│   └── commands/             # CLI 命令实现
│       └── workflow/         # OPSX 工作流命令
├── schemas/                  # 工作流模式 (spec-driven 等)
├── openspec/                 # 自吃狗粮：OpenSpec 自身的变更
├── docs/                     # 用户文档
└── dist/                     # 构建输出
```

### 核心概念

**Delta 规范：** 变更指定相对于基础规范的 ADDED（新增）/MODIFIED（修改）/REMOVED（移除）需求。

**产物 DAG：** 产物形成依赖图（proposal → specs/design → tasks）。依赖是使能条件，而非关卡。

**模式驱动工作流：** 默认的 `spec-driven` 模式：

```yaml
proposal (无依赖) → specs (需要 proposal)
                  → design (需要 proposal)
                  → tasks (需要 specs + design)
```

### CLI 命令

| 命令 | 用途 |
|------|------|
| `openspec init` | 在项目中初始化 |
| `openspec list` | 列出变更/规范 |
| `openspec validate` | 验证变更 |
| `openspec archive` | 归档已完成的变更 |
| `openspec status` | 产物完成状态 |
| `openspec instructions` | 获取产物创建指令 |
| `openspec schemas` | 列出可用工作流模式 |
| `openspec schema init/fork/validate/which` | 模式管理命令 |

### Slash 命令 (OPSX)

运行 `openspec init` 后，AI 工具获得 slash 命令。**Core 模式**（默认）：
- `/opsx:propose` — 创建变更及所有产物
- `/opsx:explore` — 在提交前调查研究
- `/opsx:apply` — 实现任务
- `/opsx:archive` — 归档并合并规范

**Expanded 模式**（需配置）额外提供：
- `/opsx:new` — 创建变更脚手架
- `/opsx:continue` — 逐个创建产物
- `/opsx:ff` — 一次性创建所有产物
- `/opsx:verify` — 验证实现与产物的一致性
- `/opsx:sync` — 同步 delta 规范到主规范
- `/opsx:bulk-archive` — 批量归档多个变更
- `/opsx:onboard` — 端到端引导教程

---

### OPSX 工作流架构

**核心思想：** "动作而非阶段" — 命令是你可以随时执行的动作，而非必须按顺序经历的阶段。

```
传统线性工作流：PLANNING → IMPLEMENTING → DONE
OPSX 流体工作流： proposal → specs → design → tasks → implement
                   (依赖是使能条件，非关卡)
```

**架构组件：**

1. **Schema 引擎** (`schemas/spec-driven/schema.yaml`)
   - 定义产物类型、生成路径、依赖关系
   - 模板目录提供每个产物的 Markdown 指令

2. **产物图系统** (`src/core/artifact-graph/`)
   - 拓扑排序决定产物创建顺序
   - 状态检测：blocked → ready → done
   - 通过文件系统存在性判断完成状态

3. **命令生成器** (`src/core/command-generation/`)
   - 从单一源生成 AI 工具技能文件
   - 支持 Claude Code、Cursor、Windsurf 等 20+ 工具

4. **Delta 规范合并**
   - 归档时将 ADDED/MODIFIED/REMOVED 合并到主规范
   - 支持并行变更而不冲突

**工作流模式：**
```
Quick Feature:  /opsx:propose → /opsx:apply → /opsx:archive
Exploratory:    /opsx:explore → /opsx:propose → /opsx:apply
Parallel:       多个 /opsx:new 同时存在，分别 /opsx:apply
```

### OPSX 优缺点分析

**优点：**
- **流体迭代** — 实现过程中可随时更新产物，支持"实现→发现设计问题→更新设计→继续实现"的自然循环
- **棕色场优先** — Delta 规范专为修改现有代码设计，避免重写完整规范
- **Schema 可扩展** — 通过 YAML 自定义工作流，无需修改 TypeScript 源码
- **并行变更安全** — 多个变更文件夹独立存在，归档时检测并解决冲突
- **审计追踪完整** — 归档保留 proposal（为什么）、design（如何）、tasks（做了什么）
- **渐进严谨度** — 简单变更可跳过 design.md，复杂变更可添加研究阶段

**缺点：**
- **学习曲线陡峭** — 需理解变更、规范、Delta、产物、Schema 等多个抽象概念
- **文档碎片化** — 核心概念分散在 `opsx.md`、`workflows.md`、`concepts.md`、`cli.md` 中
- **探索流程未完善** — 探索如何过渡到正式变更仍在讨论中（参见 `openspec/explorations/explore-workflow-ux.md`）
- **工具耦合** — 依赖 AI 助手的技能文件生成，手动维护较繁琐
- **命令过载** — Expanded 模式有 9 个命令 + 5 个 CLI 子命令，新用户易混淆
- **状态管理隐式** — 产物状态依赖文件系统检测，无集中状态文件

### 最佳实践

- **保持变更聚焦** — 一个逻辑工作单元对应一个变更
- **善用 `/opsx:explore`** — 需求不明确时先探索，再决定是否需要 `/opsx:propose`
- **归档前验证** — 使用 `/opsx:verify` 检查实现是否匹配产物
- **命名清晰** — `add-dark-mode` 优于 `feature-1`，便于 `openspec list` 浏览
- **何时更新 vs 新建** — 意图/范围未变时更新现有变更，意图根本改变时新建

---

## Superpowers

### 结构

```
superpowers/
├── skills/                   # 技能定义
│   ├── brainstorming/
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   ├── subagent-driven-development/
│   ├── using-git-worktrees/
│   ├── writing-plans/
│   └── ...
├── commands/                 # 快速命令
├── hooks/                    # 事件钩子
└── .claude-plugin/           # 插件元数据
```

### 核心技能

| 技能 | 用途 |
|------|------|
| `brainstorming` | 编码前的苏格拉底式设计优化 |
| `test-driven-development` | RED-GREEN-REFACTOR 循环执行 |
| `systematic-debugging` | 4 阶段根本原因分析 |
| `subagent-driven-development` | 并行子代理执行与审查 |
| `using-git-worktrees` | 隔离开发分支 |
| `writing-plans` | 详细实现计划 |
| `verification-before-completion` | 确保修复实际有效 |

### 技能调用

技能根据上下文自动触发。`using-superpowers` 技能确保在任何任务开始之前（甚至澄清问题之前）调用相关技能。

### 理念

- 测试驱动开发（始终先写测试）
- 系统性优于临时性
- 复杂度降低
- 证据优于声明

---

## 开发约定

### Git

- 约定式提交：`type(scope): subject`
- Changesets 版本管理（OpenSpec 中的 `.changeset/`）

---

## 相关文档

- `OpenSpec/docs/opsx.md` — OPSX 工作流架构详解（流体动作 vs 线性阶段）
- `OpenSpec/docs/workflows.md` — 工作流模式指南（何时使用哪个命令）
- `OpenSpec/docs/concepts.md` — 核心概念：规范、变更、Delta、产物依赖图
- `OpenSpec/docs/cli.md` — CLI 命令完整参考（人类命令 vs Agent 命令）
- `OpenSpec/docs/customization.md` — 自定义 Schema 和模板
- `OpenSpec/openspec/explorations/explore-workflow-ux.md` — 探索流程 UX 设计讨论
- `superpowers/README.md` — Superpowers 概览和安装
- `docs/plan/purrfect-hatching-bachman.md` — Superspec 集成设计文档
