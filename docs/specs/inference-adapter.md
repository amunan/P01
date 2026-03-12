# 推理适配层规范 (Inference Adapter Specification)

> 版本：v0.1 | 状态：初版定稿

## 1. 概述

推理适配层 (Inference Adapter) 是游戏核心层与 LLM 之间的抽象层。它屏蔽底层推理实现差异，使 Story Engine 不关心模型来自哪里。

## 2. 核心接口

### TypeScript 接口（原型阶段）

```typescript
interface InferenceAdapter {
  /** 执行一轮推演 */
  runTurn(request: TurnRequest): Promise<TurnResult>;

  /** 生成记忆摘要（用于记忆衰退压缩） */
  compressMemory(entries: VividMemoryEntry[]): Promise<string>;

  /** 生成建议选项（可独立于 runTurn 调用，用于主动介入） */
  generateSuggestions(context: SuggestionContext): Promise<Suggestion[]>;

  /** 健康检查 */
  healthCheck(): Promise<BackendStatus>;

  /** 后端类型标识 */
  backendType: BackendType;
}

type BackendType = "mock" | "cloud" | "external_local" | "builtin_local" | "replay";

interface BackendStatus {
  available: boolean;
  latencyMs?: number;
  modelId?: string;
  error?: string;
}
```

### Rust trait（产品阶段）

```rust
#[async_trait]
pub trait InferenceBackend: Send + Sync {
    async fn run_turn(&self, request: TurnRequest) -> Result<TurnResult, InferenceError>;
    async fn compress_memory(&self, entries: Vec<VividMemoryEntry>) -> Result<String, InferenceError>;
    async fn generate_suggestions(&self, context: SuggestionContext) -> Result<Vec<Suggestion>, InferenceError>;
    async fn health_check(&self) -> Result<BackendStatus, InferenceError>;
    fn backend_type(&self) -> BackendType;
}
```

## 3. TurnRequest 结构

```typescript
interface TurnRequest {
  /** 系统 Prompt（由 Prompt Composer 组装好的完整 System Prompt） */
  systemPrompt: string;

  /** 角色上下文（当前场景活跃角色的 Profile 摘要） */
  characterContext: string;

  /** 剧情上下文（当前章节、目标里程碑、世界状态摘要） */
  plotContext: string;

  /** 记忆上下文（分层记忆拼接后的文本） */
  memoryContext: string;

  /** 玩家本轮输入 */
  playerAction: PlayerAction;

  /** 输出格式指令 */
  outputFormatInstruction: string;

  /** 运行时配置 */
  config: InferenceConfig;
}

interface InferenceConfig {
  maxOutputTokens: number;     // 最大输出 token 数
  temperature: number;         // 温度参数（0.0 - 2.0）
  topP?: number;
  stopSequences?: string[];
  streamOutput: boolean;       // 是否流式输出
}
```

## 4. 后端实现

### 4.1 MockBackend

用于开发调试和自动化测试。

```typescript
class MockBackend implements InferenceAdapter {
  backendType = "mock" as const;

  constructor(private fixtures: Map<string, TurnResult>) {}

  async runTurn(request: TurnRequest): Promise<TurnResult> {
    // 查找匹配的 fixture，或返回默认响应
  }

  async healthCheck(): Promise<BackendStatus> {
    return { available: true, modelId: "mock-v1" };
  }
}
```

特性：
- 从 fixture 文件加载预设响应
- 支持基于关键词匹配的简单路由
- 延迟模拟（模拟真实推理耗时）
- 零网络依赖

### 4.2 CloudBackend

连接远程 LLM API。

```typescript
class CloudBackend implements InferenceAdapter {
  backendType = "cloud" as const;

  constructor(private config: CloudConfig) {}
}

interface CloudConfig {
  provider: "openai" | "anthropic" | "custom";
  apiKey: string;
  baseUrl?: string;
  model: string;
  timeout: number;
  retryCount: number;
}
```

特性：
- 支持 OpenAI 兼容 API（大多数云端服务兼容此格式）
- 流式输出支持（SSE）
- 自动重试 + 指数退避
- Token 用量统计

### 4.3 ExternalLocalBackend

连接用户本地运行的推理服务。

```typescript
class ExternalLocalBackend implements InferenceAdapter {
  backendType = "external_local" as const;

  constructor(private config: ExternalLocalConfig) {}
}

interface ExternalLocalConfig {
  endpoint: string;            // 如 "http://localhost:11434/api" (Ollama)
  model: string;
  connectionTimeout: number;
  protocol: "ollama" | "openai_compat" | "lm_studio";
}
```

特性：
- 自动检测本地服务是否运行
- 支持 Ollama / LM Studio / 任何 OpenAI 兼容 API
- 离线可用（只要本地服务在运行）
- 模型列表查询

### 4.4 BuiltinLocalBackend（后续阶段）

游戏内嵌的推理引擎，产品阶段实现。

```typescript
class BuiltinLocalBackend implements InferenceAdapter {
  backendType = "builtin_local" as const;
  // Rust 侧通过 llama.cpp binding 实现
  // TS 侧通过 Tauri command 桥接
}
```

特性：
- 用户无需安装额外软件
- 模型随游戏安装或首次运行时下载
- 通过 Tauri command 调用 Rust 侧推理
- 硬件检测：启动时检查 GPU/内存是否满足要求

### 4.5 ReplayBackend

回放录制的推理结果，用于确定性测试和调试。

```typescript
class ReplayBackend implements InferenceAdapter {
  backendType = "replay" as const;

  constructor(private recording: TurnResult[]) {}

  async runTurn(): Promise<TurnResult> {
    return this.recording[this.cursor++];
  }
}
```

特性：
- 完全确定性，相同输入始终相同输出
- 从录制文件加载
- 用于自动化测试和 bug 复现

## 5. 推理路由 (Inference Router)

管理多个后端的优先级和降级策略。

```typescript
class InferenceRouter implements InferenceAdapter {
  private backends: InferenceAdapter[];
  private activeBackend: InferenceAdapter | null = null;

  constructor(private config: RouterConfig) {}

  async runTurn(request: TurnRequest): Promise<TurnResult> {
    // 按优先级尝试可用后端
    // 失败时自动降级到下一个
  }

  async selectBackend(): Promise<InferenceAdapter> {
    for (const backend of this.backends) {
      const status = await backend.healthCheck();
      if (status.available) return backend;
    }
    throw new InferenceError("no_backend_available");
  }
}

interface RouterConfig {
  /** 后端优先级（从高到低） */
  priority: BackendType[];
  /** 健康检查间隔 */
  healthCheckIntervalMs: number;
  /** 切换后端时是否通知用户 */
  notifyOnSwitch: boolean;
}
```

### 默认优先级链

| 平台 | 优先级（高 → 低） |
|------|------------------|
| Desktop (Tauri) | builtin_local → external_local → cloud |
| Web | cloud → external_local |
| 小程序 | cloud |
| 开发调试 | mock → external_local → cloud |

## 6. 错误处理

```typescript
class InferenceError extends Error {
  constructor(
    public code: InferenceErrorCode,
    message: string,
    public retryable: boolean = false
  ) {
    super(message);
  }
}

type InferenceErrorCode =
  | "no_backend_available"
  | "connection_failed"
  | "timeout"
  | "invalid_response"       // AI 返回的 JSON 无法解析
  | "output_validation_failed" // 结构校验失败
  | "content_filtered"       // 内容安全过滤
  | "rate_limited"
  | "model_not_found"
  | "insufficient_hardware";
```

## 7. 流式输出协议

当 `streamOutput: true` 时，推理结果通过事件流逐步推送。

```typescript
interface StreamEvent {
  type: "token" | "beat_complete" | "turn_complete" | "error";
  /** token 事件：增量文本片段 */
  token?: string;
  /** beat_complete 事件：一个完整 Beat 已生成 */
  beat?: Beat;
  /** turn_complete 事件：整轮推演完成 */
  result?: TurnResult;
  /** error 事件 */
  error?: InferenceError;
}
```

前端订阅事件流，逐段渲染叙事文本，提升感知速度。

## 8. 原型 vs 产品阶段差异

| 维度 | 原型阶段 (Web/TS) | 产品阶段 (Tauri/Rust) |
|------|------------------|---------------------|
| 接口语言 | TypeScript interface | Rust trait |
| 前端调用方式 | 直接调用 TS 类 | 通过 Tauri command 调用 Rust |
| 后端实现 | Mock + ExternalLocal + Cloud | 全部 5 种 |
| Prompt 组装 | TS 侧 | Rust 侧 |
| 输出校验 | TS 侧 JSON Schema 校验 | Rust 侧 serde 反序列化 + 校验 |
| 流式输出 | fetch SSE / WebSocket | Tauri event channel |
