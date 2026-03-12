# Session State 与存档快照规范

> 版本：v0.1 | 状态：初版定稿

## 1. 概述

Session State 是一次游玩过程的完整运行时状态。存档快照 (Save Snapshot) 是 Session State 在某个时间点的序列化副本。

## 2. Session State 结构

```typescript
interface SessionState {
  /** 会话元信息 */
  meta: SessionMeta;
  /** 玩家信息 */
  player: PlayerState;
  /** 剧情进度 */
  plot: PlotState;
  /** 世界状态 */
  world: WorldState;
  /** 记忆系统 */
  memory: MemoryState;
  /** 存档点资源 */
  savePoints: SavePointState;
}

interface SessionMeta {
  sessionId: string;
  storyId: string;
  createdAt: string;          // ISO 8601
  updatedAt: string;          // ISO 8601
  totalTurns: number;         // 总交互轮数
  playtimeSeconds: number;    // 累计游玩时间
  engineVersion: string;      // 引擎版本，用于兼容性检查
  schemaVersion: number;      // 数据结构版本号
}

interface PlayerState {
  roleId: string;             // 对应 Story Package 中的角色 ID
  displayName: string;
  customTraits: Record<string, string>;  // 玩家自定义的角色特征
}

interface PlotState {
  currentChapter: number;
  currentScene: string;
  currentBeatIndex: number;
  completedMilestones: MilestoneRecord[];
  activeMilestoneId: string | null;      // 当前目标里程碑
  chapterHistory: ChapterSummary[];
}

interface MilestoneRecord {
  milestoneId: string;
  completedAt: string;        // ISO 8601
  turnNumber: number;         // 在第几轮完成的
  triggerContext: string;      // 触发时的简要上下文
}

interface ChapterSummary {
  chapter: number;
  summary: string;            // AI 生成的章节摘要
  keyDecisions: string[];     // 玩家在该章的关键决策
}

interface WorldState {
  flags: Record<string, boolean>;
  variables: Record<string, number | string>;
  relationships: RelationshipMap;
  activeCharacters: string[];  // 当前场景中活跃的角色 ID
}

interface RelationshipMap {
  /** key 格式: "characterA->characterB", value: -100 到 100 */
  [key: string]: number;
}

interface MemoryState {
  vivid: VividMemoryEntry[];
  faded: FadedMemoryEntry[];
  forgotten: ForgottenIndex[];
  permanent: PermanentMemoryEntry[];
}

interface VividMemoryEntry {
  turnNumber: number;
  role: "player" | "narrator" | string;  // string 为 NPC 角色 ID
  content: string;
  timestamp: string;
}

interface FadedMemoryEntry {
  turnRange: [number, number];   // 覆盖的轮次范围
  summary: string;
  tokenCount: number;
  relatedCharacters: string[];
}

interface ForgottenIndex {
  turnRange: [number, number];
  tags: string[];                // 极简关键词索引
}

interface PermanentMemoryEntry {
  turnNumber: number;
  type: "milestone" | "betrayal" | "alliance" | "death" | "promise" | "custom";
  summary: string;
  relatedCharacters: string[];
}

interface SavePointState {
  available: number;            // 当前可用的手动存档点
  usedThisChapter: number;      // 本章已用的存档点
  totalUsed: number;            // 累计已用的存档点
}
```

## 3. 存档快照结构

```typescript
interface SaveSnapshot {
  /** 快照头信息 */
  header: SnapshotHeader;
  /** 完整的 Session State */
  state: SessionState;
  /** 校验信息 */
  integrity: IntegrityInfo;
}

interface SnapshotHeader {
  snapshotId: string;
  sessionId: string;
  storyId: string;
  saveType: "auto_anchor" | "manual";
  createdAt: string;
  turnNumber: number;
  chapterNumber: number;
  sceneName: string;
  displayLabel: string;        // 给玩家看的存档描述
  schemaVersion: number;
}

interface IntegrityInfo {
  checksum: string;            // SHA-256 of state JSON
  engineVersion: string;
}
```

## 4. 存档操作规则

### 创建存档

```
自动锚点：里程碑完成 → 自动创建 SaveSnapshot(saveType: "auto_anchor")
手动存档：玩家触发 → 检查 available > 0 → 扣减 → 创建 SaveSnapshot(saveType: "manual")
```

### 读取存档

```
选择存档 → 反序列化 SaveSnapshot → 校验 checksum → 检查 schemaVersion 兼容性
  → 用 state 替换当前 SessionState → 从 state.plot.currentBeatIndex 恢复推演
```

### 存档兼容性

- `schemaVersion` 递增管理
- 向前兼容：新版本可以读取旧版本存档（通过 migration 函数）
- 不保证向后兼容：旧版本不一定能读取新版本存档

## 5. 序列化格式

### 原型阶段 (Web)

- 格式：JSON
- 存储：`localStorage` 或 IndexedDB
- 不加密

### 产品阶段 (Tauri/Desktop)

- 格式：JSON → zstd 压缩 → AES-256-GCM 加密
- 存储：本地文件系统，`~/.game-name/saves/`
- 加密密钥由 Rust Core 管理

---

# AI 输出 JSON 协议

> 版本：v0.1 | 状态：初版定稿

## 1. 概述

每次 AI 推演返回一个结构化 JSON 对象，而非纯文本。前端/Core 根据此结构渲染叙事、更新状态、判断里程碑。

## 2. TurnResult 结构

```typescript
interface TurnResult {
  /** 本轮生成的叙事节拍列表 */
  beats: Beat[];
  /** 状态变更指令 */
  stateChanges: StateChanges;
  /** 是否触发了里程碑 */
  milestoneTriggered: MilestoneTrigger | null;
  /** AI 生成的建议选项（在最后一个 Beat 后展示） */
  suggestions: Suggestion[];
  /** 场景转换指令 */
  sceneTransition: SceneTransition | null;
  /** 介入点类型（本轮最后应暂停等待玩家吗） */
  interventionType: "reactive" | "forced" | "none";
  /** 调试/元信息 */
  debug?: DebugInfo;
}
```

## 3. Beat 结构

```typescript
interface Beat {
  type: "narration" | "dialogue" | "event" | "transition";
  content: string;
  /** dialogue 类型时必填 */
  speaker?: string;           // 角色 ID
  emotion?: string;           // 情绪标签
  /** 展示控制 */
  displayHint?: "normal" | "dramatic" | "whisper" | "shout";
}
```

## 4. StateChanges 结构

```typescript
interface StateChanges {
  /** 关系值变更 */
  relationships?: Record<string, number>;  // key: "A->B", value: 变化量（正负）
  /** Flag 变更 */
  flags?: Record<string, boolean>;
  /** 变量变更 */
  variables?: Record<string, number | string>;
  /** 记忆标记：哪些内容应进入永久记忆 */
  permanentMemoryTags?: PermanentMemoryEntry[];
  /** 角色进出场景 */
  characterEnter?: string[];
  characterExit?: string[];
}
```

## 5. Suggestion 结构

```typescript
interface Suggestion {
  id: string;
  text: string;               // 展示给玩家的选项文本
  tone: string;               // 基调标签：如 "cautious", "aggressive", "diplomatic"
  /** 选项的内部行动描述，用于回传给 AI 上下文 */
  actionDescription: string;
}
```

## 6. 辅助类型

```typescript
interface MilestoneTrigger {
  milestoneId: string;
  summary: string;            // 里程碑完成的简要描述
}

interface SceneTransition {
  targetScene: string;
  transitionType: "fade" | "cut" | "dissolve";
  timeSkipDescription?: string;  // 如 "三天后..."
}

interface DebugInfo {
  promptTokens: number;
  completionTokens: number;
  modelId: string;
  latencyMs: number;
}
```

## 7. 玩家输入结构

```typescript
interface PlayerAction {
  /** 输入类型 */
  type: "suggestion_select" | "free_text" | "silence" | "proactive_intervene";
  /** suggestion_select 时为选中的 suggestion ID */
  suggestionId?: string;
  /** free_text 时为玩家输入的原始文本 */
  text?: string;
  /** 当前轮次号 */
  turnNumber: number;
}
```

## 8. 协议约束

- AI 输出必须是可解析的 JSON，不可包含 JSON 外的文本
- `beats` 数组至少包含 1 个 Beat
- `suggestions` 数组在 `interventionType` 为 `reactive` 或 `forced` 时必须非空
- 关系值变更为增量（+5 表示增加 5），不是绝对值
- 角色 ID 必须匹配 Story Package 中定义的角色
