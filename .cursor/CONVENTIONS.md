# .cursor 配置规范

Rules 和 Skills 的命名、格式、新增流程。

---

## 1. Rules（编码规则）

### 定位

Rule = 针对特定领域的编码约束，Cursor 在匹配场景下自动注入上下文。

### 文件位置

```
.cursor/rules/{domain}.mdc
```

### 命名规则

| 规则 | 说明 |
|------|------|
| 格式 | `{domain}.mdc`，kebab-case |
| 精简 | 用最短的领域词，不加冗余限定词 |
| 不加角色 | 不用 "backend"、"frontend" 等除非必要消歧 |

**命名示例**：

```
✅ rust.mdc          — Rust 就是 Rust，不需要 "rust-backend"
✅ react.mdc         — React 本身就是前端，不需要 "react-frontend"
✅ inference.mdc     — inference 本身就涉及 AI，不需要 "ai-inference"
✅ tauri.mdc         — 框架名即领域
✅ animation.mdc     — 如果将来新增动画规则

❌ react-frontend.mdc
❌ ai-inference-layer.mdc
❌ rust-backend-coding-standards.mdc
```

### Frontmatter 格式

```yaml
---
description: 一句中文，说明此规则的适用范围
globs: "**/*.rs"        # 可选，按文件类型自动触发
alwaysApply: false       # 仅 core.mdc 为 true
---
```

| 字段 | 要求 |
|------|------|
| `description` | 一句中文，≤ 30 字，说明"这是什么规则" |
| `globs` | 可选。设置后编辑匹配文件时自动激活 |
| `alwaysApply` | 仅 `core.mdc` 设为 `true`，其余均为 `false` |

### 正文结构

```markdown
# {领域} 规则

## 第一个主题
- 要点
- 要点

## 第二个主题
- 要点

（代码示例用 BAD / GOOD 对比）
```

- 每个 rule 聚焦一个领域，不跨界
- 条目用祈使句，直接说"做什么"或"不做什么"
- 代码示例用 `// BAD` / `// GOOD` 对比，不写多余解释

### 新增 Rule 检查清单

1. 确认现有 rules 不已覆盖此内容
2. 文件名遵循 `{domain}.mdc` 命名
3. description 一句中文 ≤ 30 字
4. globs 设置正确（或不设，手动触发）
5. 更新 `core.mdc` 末尾的 rules 清单

---

## 2. Skills（专家技能）

### 定位

Skill = 特定领域的深度专家知识，Cursor 按 description 匹配触发，注入完整的专业能力。

### 目录位置

```
.cursor/skills/{domain}/SKILL.md
```

### 命名规则

| 规则 | 说明 |
|------|------|
| 格式 | `{domain}/`，kebab-case |
| 精简 | 用领域名，不用角色头衔 |
| 不加头衔 | 不用 "designer"、"architect"、"lead" |

**命名示例**：

```
✅ game-systems/      — 聚焦游戏系统设计
✅ narrative-design/   — 聚焦叙事内容设计
✅ visual-design/      — 聚焦视觉交互设计
✅ rust-engine/        — 聚焦 Rust 引擎实现
✅ tauri-ai/           — 聚焦 Tauri + AI 架构

❌ game-systems-designer/
❌ senior-rust-technical-lead/
❌ tauri-local-ai-architect/
```

### Frontmatter 格式

```yaml
---
name: {domain}          # 与目录名一致
description: >-
  {领域}: {关键能力}.
  Use when {触发场景}.
---
```

| 字段 | 要求 |
|------|------|
| `name` | 与目录名完全一致 |
| `description` | 3 行以内，英文。第一行 = 领域 + 关键词；第二行 = Use when 触发条件 |

**description 模板**：

```yaml
description: >-
  {领域描述}:
  {关键能力1}, {关键能力2}, {关键能力3}.
  Use when {场景1}, {场景2}, or {场景3}.
```

**实际示例**：

```yaml
# game-systems
description: >-
  Game mechanics & system balance for AI-driven text games:
  memory decay, save economy, pacing control, plot guardrails, replayability.
  Use when designing/balancing game systems, tuning parameters, or defining intervention logic.

# rust-engine
description: >-
  Rust implementation for game engine core:
  state machines, serialization, memory models, FFI/IPC, performance, security.
  Use when implementing Rust logic, save systems, Story Package parsers, or Rust-JS boundaries.
```

### 正文结构

```markdown
# 中文标题 / English Title

（一段话定义角色能力边界）

## 语言规范
（默认中文，术语双语对照）

## 核心能力
### 1. 能力一 (English)
### 2. 能力二 (English)
...

## 工作流程
（步骤列表）

## 输出规范
（格式要求）
```

- 正文保留专业深度，不因 frontmatter 精简而削减内容
- 核心能力按子领域分节，每节有具体的规范和示例
- 术语首次出现时给双语对照：`中文术语 (English Term)`

### 新增 Skill 检查清单

1. 确认现有 skills 不已覆盖此领域
2. 目录名遵循 `{domain}/` 命名，不含角色头衔
3. `name` 字段与目录名一致
4. `description` ≤ 3 行，遵循模板格式
5. 正文包含：角色定位、核心能力、工作流程、输出规范
6. 更新 `core.mdc` 末尾的 skills 清单

---

## 3. 当前清单

### Rules

| 文件 | 描述 | 触发方式 |
|------|------|---------|
| `core.mdc` | 核心通用规则 | alwaysApply |
| `rust.mdc` | Rust 核心引擎编码规范 | `**/*.rs` |
| `tauri.mdc` | Tauri 桌面应用架构规范 | `src-tauri/**` |
| `react.mdc` | React 前端开发规范 | `src/**/*.tsx,ts,jsx` |
| `inference.mdc` | AI 推理适配层规范 | 手动触发 |

### Skills

| 目录 | 领域 |
|------|------|
| `game-systems/` | 游戏机制设计与系统平衡 |
| `narrative-design/` | 叙事内容与 AI Prompt 模板 |
| `visual-design/` | 视觉风格与交互设计 |
| `rust-engine/` | Rust 引擎核心实现 |
| `tauri-ai/` | Tauri 桌面 + 本地 AI 架构 |
