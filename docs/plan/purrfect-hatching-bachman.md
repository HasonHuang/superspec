# Superspec 架构方案分析

## 背景

用户提出两种整合 OpenSpec 和 Superpowers 的方案：

1. **组合层方案（推荐）**：superspec 作为技能组合层，引用/组合 OpenSpec 和 Superpowers 的技能
2. **融合方案（备用）**：将两套技能融合/复制到 superspec 中

---

## 现状分析

### 已有基础

Superspec 项目已存在初始设计和技能实现：

```
superspec/
├── skills/
│   ├── superspec-core/        # 元技能：工作流编排 ✓
│   ├── superspec-propose/     # 提案生成 ✓
│   ├── superspec-apply/       # 实现执行 ✓
│   ├── superspec-verify/      # 验证审查 ✓
│   └── superspec-debug/       # 调试支持 (需要完善)
└── SUPERSPEC_DESIGN.md        # 设计文档 ✓
```

### 依赖关系

当前 superspec 技能依赖于 Superpowers 技能：

| Superspec 技能 | 依赖的 Superpowers 技能 |
|---------------|------------------------|
| `superspec-core` | 编排所有子技能 |
| `superspec-propose` | `brainstorming`, `writing-plans` |
| `superspec-apply` | `subagent-driven-development`, `test-driven-development`, `using-git-worktrees` |
| `superspec-verify` | `verification-before-completion`, `requesting-code-review` |
| `superspec-debug` | `systematic-debugging` |

---

## 方案一：组合层方案（推荐）

### 架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Superspec 组合层                                   │
│                                                                             │
│  skills/                                                                    │
│  ├── superspec-core/       → 编排逻辑，调用外部技能                          │
│  ├── superspec-propose/    → 引用 Superpowers brainstorming/writing-plans   │
│  ├── superspec-apply/      → 引用 Superpowers subagent-driven-development   │
│  ├── superspec-verify/     → 引用 Superpowers verification/code-review      │
│  └── superspec-debug/      → 引用 Superpowers systematic-debugging          │
└─────────────────────────────────────────────────────────────────────────────┘
                              ↓ 组合/引用
┌─────────────────────────────────────────────────────────────────────────────┐
│                        外部依赖技能库                                         │
│                                                                             │
│   ┌─────────────────────────┐   ┌─────────────────────────────────────────┐ │
│   │    Superpowers          │   │    OpenSpec                             │ │
│   │  - brainstorming        │   │  - /opsx:* 命令                          │ │
│   │  - writing-plans        │   │  - opsx-propose                         │ │
│   │  - subagent-driven-dev  │   │  - opsx-apply                           │ │
│   │  - TDD                  │   │  - opsx-verify                          │ │
│   │  - verification-*       │   │  - opsx-archive                         │ │
│   │  - using-git-worktrees  │   │                                        │ │
│   │  - systematic-debugging │   │                                        │ │
│   └─────────────────────────┘   └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 技术实现

Superspec 技能通过 Skill 工具引用外部技能：

```markdown
---
name: superspec-apply
description: "使用子代理执行 OpenSpec 任务"
---

# Superspec Apply

## 依赖技能

**必须同时加载：**
- `superpowers:subagent-driven-development`
- `superpowers:test-driven-development`
- `superpowers:using-git-worktrees`

## 工作流程

1. 调用 `superpowers:using-git-worktrees` 创建隔离环境
2. 读取 OpenSpec tasks.md
3. 对每个任务调用 `superpowers:subagent-driven-development`
4. ...
```

### 安装要求

用户需要安装所有依赖：

```bash
# 1. 安装 OpenSpec CLI
npm install -g @fission-ai/openspec@latest
openspec init

# 2. 安装 Superpowers 插件
/plugin install superpowers

# 3. 安装 Superspec 插件
/plugin install superspec
```

### 优点

| 优点 | 说明 |
|------|------|
| **独立更新** | OpenSpec/Superpowers 可独立迭代，superspec 自动获益 |
| **代码复用** | 不重复实现已有技能逻辑 |
| **轻量维护** | superspec 只维护编排逻辑和整合规则 |
| **清晰边界** | 每个技能库职责明确 |
| **社区贡献** | 底层技能库的改进自动惠及 superspec |

### 缺点

| 缺点 | 说明 | 缓解方案 |
|------|------|----------|
| **依赖复杂** | 用户需安装多个插件 | 提供一键安装脚本 |
| **版本兼容** | 需管理跨技能库版本兼容 | 使用语义化版本 + 兼容性测试 |
| **加载顺序** | 需确保依赖技能先加载 | 在 SKILL.md 中明确声明依赖 |
| **技能命名** | 可能混淆（如 `brainstorming` vs `superspec-propose`） | 使用命名空间前缀区分 |

### 技能命名空间设计

```
superpowers:*  → 原始 Superpowers 技能
  - brainstorming
  - writing-plans
  - subagent-driven-development
  - test-driven-development
  - ...

opsx:*  → OpenSpec CLI 命令（slash commands）
  - /opsx:propose
  - /opsx:apply
  - /opsx:verify
  - /opsx:archive

superspec:*  → Superspec 组合技能
  - superspec-core
  - superspec-propose
  - superspec-apply
  - superspec-verify
  - superspec-debug
```

---

## 方案二：融合方案（备用）

### 架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Superspec 融合技能库                                 │
│                                                                             │
│  skills/                                                                    │
│  ├── superspec-core/                                                        │
│  ├── superspec-propose/    ← 融合 brainstorming + writing-plans 逻辑         │
│  ├── superspec-apply/      ← 融合 subagent-driven-development 逻辑          │
│  ├── superspec-verify/     ← 融合 verification + code-review 逻辑           │
│  ├── superspec-debug/      ← 融合 systematic-debugging 逻辑                 │
│  │                                                                          │
│  ├── (融合的 Superpowers 技能)                                              │
│  │   ├── brainstorming/    ← 复制 Superpowers 源代码                        │
│  │   ├── writing-plans/                                                     │
│  │   ├── subagent-driven-development/                                       │
│  │   ├── test-driven-development/                                           │
│  │   └── ...                                                                │
│  │                                                                          │
│  └── (OpenSpec 整合层)                                                      │
│      ├── opsx-propose/     ← OpenSpec 命令的技能封装                        │
│      ├── opsx-apply/                                                        │
│      └── ...                                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 技术实现

复制 Superpowers 技能源码到 superspec：

```bash
# 复制 Superpowers 技能
cp -r superpowers/skills/brainstorming superspec/skills/
cp -r superpowers/skills/writing-plans superspec/skills/
cp -r superpowers/skills/subagent-driven-development superspec/skills/
# ...

# 修改 SKILL.md 中的引用
# 添加 OpenSpec 整合逻辑
```

### 安装要求

```bash
# 1. 安装 OpenSpec CLI
npm install -g @fission-ai/openspec@latest
openspec init

# 2. 安装 Superspec 插件（包含所有技能）
/plugin install superspec
```

### 优点

| 优点 | 说明 |
|------|------|
| **简单依赖** | 用户只需安装 superspec 一个插件 |
| **自包含** | 所有技能逻辑都在一个地方 |
| **版本控制** | 不需要管理跨库版本兼容 |
| **定制灵活** | 可以修改底层技能逻辑以适应整合 |

### 缺点

| 缺点 | 说明 | 影响 |
|------|------|------|
| **代码重复** | Superpowers 更新时需手动同步 | 维护负担重，容易过时 |
| **失去独立性** | 无法从底层技能库的改进中自动获益 | 需要主动追踪上游更新 |
| **许可证问题** | 需确保兼容和保留原作者信息 | 法律风险 |
| **技能冲突** | 可能与用户已安装的 Superpowers 冲突 | 用户体验问题 |
| **体积膨胀** | 技能库变大，加载时间增长 | 性能影响 |

---

## 推荐方案：组合层 + 可选融合

### 混合架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Superspec 发行版                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  核心层 (superspec 原创)：                                                   │
│  ├── superspec-core/       [原创编排逻辑]                                    │
│  ├── superspec-propose/    [原创整合逻辑]                                    │
│  ├── superspec-apply/      [原创整合逻辑]                                    │
│  ├── superspec-verify/     [原创整合逻辑]                                    │
│  └── superspec-debug/      [原创整合逻辑]                                    │
│                                                                             │
│  组合层 (引用外部)：                                                         │
│  ├── .superspec-deps.yaml  [声明依赖的技能]                                  │
│  │   dependencies:                                                         │
│  │     - superpowers:brainstorming                                         │
│  │     - superpowers:subagent-driven-development                           │
│  │     - superpowers:test-driven-development                               │
│  │     - ...                                                               │
│  │                                                                           │
│  可选：融合核心 Superpowers 技能 (用于简化安装)：                             │
│  ├── _vendor/                [vendor 目录，明确标记为第三方]                  │
│  │   └── superpowers/                                                        │
│  │       ├── brainstorming/    ← 复制 + 保留原作者信息                       │
│  │       ├── TDD/                                                            │
│  │       └── ...                                                             │
│                                                                             │
│  安装脚本：                                                                  │
│  ├── install.sh              [一键安装：检测/安装依赖]                        │
│  └── install.ps1                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 实施策略

#### 阶段 1：纯组合层（MVP）

先实现纯组合层方案，验证核心价值：

1. 完善现有 superspec 技能（core/propose/apply/verify/debug）
2. 在 SKILL.md 中明确声明依赖
3. 提供安装说明和依赖检查

**交付物：**
- 完整的 superspec 技能组
- 文档：如何安装依赖
- 依赖检查脚本

#### 阶段 2：Vendor 可选分发

如果用户反馈依赖安装太复杂，添加 vendor 目录：

1. 复制 Superpowers 核心技能到 `_vendor/superpowers/`
2. 保留原作者信息和许可证
3. 安装脚本检测是否已安装 Superpowers
   - 已安装：使用已安装版本
   - 未安装：使用 vendor 版本或提示安装

**交付物：**
- Vendor 目录
- 一键安装脚本
- 版本同步机制（定期同步上游）

#### 阶段 3：OpenSpec 整合增强

与 OpenSpec 项目协作，可能的话：

1. 在 OpenSpec CLI 中添加 `--superspec` 标志
2. 生成增强版技能（包含 TDD 和子代理执行）
3. 或提供 superspec 专用的 schema/templates

---

## 技术细节

### 依赖声明格式

创建 `.superspec-deps.yaml` 文件：

```yaml
# .superspec-deps.yaml
name: superspec
version: 0.1.0

# 声明依赖的外部技能
dependencies:
  # Superpowers 技能库
  - name: superpowers
    min_version: "5.0.0"
    skills:
      - brainstorming
      - writing-plans
      - subagent-driven-development
      - test-driven-development
      - using-git-worktrees
      - verification-before-completion
      - requesting-code-review
      - systematic-debugging
      - finishing-a-development-branch

  # OpenSpec CLI（提供 slash commands）
  - name: openspec
    min_version: "1.0.0"
    commands:
      - /opsx:propose
      - /opsx:apply
      - /opsx:verify
      - /opsx:archive

# 可选：vendor 目录中的技能（如果提供）
vendor:
  enabled: true
  source: _vendor/
  fallback: true  # 如果未安装外部依赖，使用 vendor 版本
```

### 安装脚本示例

```bash
#!/bin/bash
# superspec-install.sh

SUPERPOWERS_INSTALLED=$(/plugin list | grep -c superpowers)
OPENSCRIPT_INSTALLED=$(npm list -g @fission-ai/openspec | grep -c @)

echo "=== Superspec 依赖检查 ==="

if [ "$SUPERPOWERS_INSTALLED" -eq 0 ]; then
    echo "⚠ Superpowers 未安装"
    echo "  安装：/plugin install superpowers"
    echo "  或使用 vendor 版本：./superspec-install.sh --use-vendor"
else
    echo "✓ Superpowers 已安装"
fi

if [ "$OPENSCRIPT_INSTALLED" -eq 0 ]; then
    echo "⚠ OpenSpec CLI 未安装"
    echo "  安装：npm install -g @fission-ai/openspec"
else
    echo "✓ OpenSpec CLI 已安装"
fi

# 继续安装 superspec...
```

### 技能加载顺序

在 Claude Code 中，技能加载顺序很重要。确保：

1. **依赖技能先加载**：Superpowers 技能 → Superspec 技能
2. **使用绝对引用**：在 SKILL.md 中使用完整技能名

```markdown
---
name: superspec-apply
---

## Required Skills

**必须预先加载：**
- `superpowers:subagent-driven-development`
- `superpowers:test-driven-development`
```

---

## 版本兼容性

### 版本矩阵

| Superspec | Superpowers | OpenSpec CLI |
|-----------|-------------|--------------|
| 0.1.x | 5.0.x | 1.x |
| 0.2.x | 5.1.x | 1.x |
| 1.0.x | 5.x | 2.x |

### 兼容性测试

```bash
# 测试脚本
./test-compatibility.sh \
  --superpowers-version 5.0.6 \
  --openspec-version 1.2.0
```

---

## 许可证考虑

### Superpowers (MIT)

```
MIT License - 允许复制、修改、分发，需保留原作者信息
```

Vendor 方案需要：
1. 保留 `LICENSE` 文件
2. 在每个 vendor 技能的 SKILL.md 中注明原作者
3. 明确标记为第三方代码

### OpenSpec (MIT)

同样需要保留原作者信息和许可证。

---

## 总结与推荐

### 推荐：组合层方案 + Vendor 可选

**短期（MVP）：**
- 纯组合层，用户自行安装依赖
- 完善 superspec 核心技能
- 提供清晰的安装文档

**中期：**
- 添加 vendor 目录作为可选
- 一键安装脚本
- 定期同步上游更新

**长期：**
- 与 OpenSpec/Superpowers 项目协作
- 可能的官方整合

### 决策理由

| 标准 | 组合层 | 融合 | 混合 |
|------|--------|------|------|
| 维护成本 | 低 | 高 | 中 |
| 用户便利 | 中 | 高 | 高 |
| 独立性 | 高 | 低 | 高 |
| 可持续 | 是 | 否 | 是 |
| 许可证合规 | 是 | 需注意 | 是 |

**混合方案**平衡了维护成本、用户便利和长期可持续性。

---

## 下一步行动

### 阶段 1：组合层 MVP

- [ ] 完善 `superspec-debug` 技能
- [ ] 在所有 SKILL.md 中添加依赖声明
- [ ] 创建 `.superspec-deps.yaml`
- [ ] 编写安装文档
- [ ] 创建依赖检查脚本

### 阶段 2：Vendor 支持

- [ ] 复制 Superpowers 核心技能到 `_vendor/`
- [ ] 添加许可证和原作者信息
- [ ] 创建一键安装脚本
- [ ] 实现依赖检测逻辑

### 阶段 3：OpenSpec 协作

- [ ] 联系 OpenSpec 维护者
- [ ] 讨论官方整合可能
- [ ] 探索 CLI 扩展点
