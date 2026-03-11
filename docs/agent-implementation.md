# OpenClaw Agent 实现与 AI 原生程序技术方案

本文档深入剖析 OpenClaw 项目中 Agent（智能代理）的完整实现架构，涵盖 Pi 运行时、ACP 控制平面、工具系统、内存系统等核心组件。

---

## 目录

1. [架构概览](#架构概览)
2. [核心概念](#核心概念)
3. [Pi 代理运行时](#pi-代理运行时)
4. [ACP 控制平面](#acp-控制平面)
5. [工具系统](#工具系统)
6. [内存与上下文管理](#内存与上下文管理)
7. [多代理路由](#多代理路由)
8. [AI 原生程序设计哲学](#ai-原生程序设计哲学)

---

## 架构概览

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Telegram │  │ Discord  │  │  Slack   │  │ WhatsApp │  │   WebChat    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┼───────────┘
        │             │             │             │               │
        └─────────────┴─────────────┴─────────────┴───────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            网关层 (Gateway)                                  │
│                     ws://127.0.0.1:18789                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  统一消息路由  │  会话管理  │  事件总线  │  WebSocket 广播               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ACP 控制平面 (Agent Control Plane)                     │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Session Manager  │  Identity Resolver  │  Persistent Bindings       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Pi 代理运行时 (@mariozechner/pi-*)                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  Agent Core  │  Tool Registry  │  Stream Wrappers  │  Compaction     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────┐ ┌───────────────────────────┐
│      工具系统          │ │    内存系统        │ │      沙箱系统             │
│  ┌─────────────────┐  │ │  ┌─────────────┐  │ │  ┌─────────────────────┐  │
│  │ Read/Write/Edit │  │ │  │ Embeddings  │  │ │  │ Docker/Podman       │  │
│  │ Exec/Process    │  │ │  │ Vector Store│  │ │  │ Browser Automation  │  │
│  │ Browser/Canvas  │  │ │  │ Hybrid FTS  │  │ │  │ FS Bridge           │  │
│  │ Channel Actions │  │ │  │ sqlite-vec  │  │ │  │ Security Policy     │  │
│  └─────────────────┘  │ │  └─────────────┘  │ │  └─────────────────────┘  │
└───────────────────────┘ └───────────────────┘ └───────────────────────────┘
```

---

## 核心概念

### 1. Agent 定义

在 OpenClaw 中，**Agent** 是一个能够接收消息、执行工具调用、并返回响应的智能实体。关键特征：

- **会话隔离**：每个 Agent 运行在独立的会话上下文
- **工具能力**：通过 Tool Registry 动态注册可用工具
- **多模态**：支持文本、图像、音频等多种输入输出
- **流式响应**：支持增量式内容生成（streaming）

### 2. 会话（Session）

```typescript
// 会话标识符结构
sessionKey: "agent:main:main"     // agentId:sessionLabel:channel
sessionId: "uuid-v4"              // 运行时生成的唯一 ID
```

会话是 Agent 状态管理的边界：
- 消息历史隔离
- 工具调用上下文
- 内存向量存储隔离
- 配置覆盖（per-session config）

### 3. 运行模式

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| **Embedded** | Pi 运行时嵌入在 OpenClaw 进程中 | 本地 CLI、生产环境 |
| **External** | 通过 ACP 协议连接外部 Agent | IDE 插件、远程 Agent |

---

## Pi 代理运行时

### 架构设计

Pi（Prompt Interface）是 OpenClaw 的核心 Agent 运行时，基于 `@mariozechner/pi-*` 系列包构建：

```
pi-embedded-runner/
├── run.ts              # 主运行循环入口
├── run/
│   ├── attempt.ts      # 单次尝试执行
│   ├── payloads.ts     # 请求载荷构建
│   └── compaction.ts   # 上下文压缩
├── model.ts            # 模型解析与选择
├── history.ts          # 消息历史管理
├── system-prompt.ts    # 系统提示词构建
└── skills-runtime.ts   # 技能运行时
```

### 核心运行循环

```typescript
// src/agents/pi-embedded-runner/run.ts
export async function runEmbeddedPiAgent(
  params: RunEmbeddedPiAgentParams
): Promise<EmbeddedPiRunResult> {
  // 1. 解析并验证模型配置
  const { model, error } = resolveModel(provider, modelId, agentDir, config);
  
  // 2. 处理认证配置文件轮换
  const profileCandidates = resolveAuthProfileOrder({ cfg, store, provider });
  
  // 3. 主运行循环（支持故障转移）
  while (runLoopIterations < MAX_RUN_LOOP_ITERATIONS) {
    // 3.1 构建请求载荷
    const payload = buildEmbeddedRunPayloads({ ... });
    
    // 3.2 执行单次尝试
    const attempt = await runEmbeddedAttempt({ ... });
    
    // 3.3 处理上下文溢出（自动压缩）
    if (contextOverflowError) {
      await compactEmbeddedPiSession({ ... });
      continue;
    }
    
    // 3.4 处理故障转移
    if (shouldFailover(error)) {
      await advanceAuthProfile();
      continue;
    }
    
    return attempt.result;
  }
}
```

### 关键特性

#### 1. 多模型故障转移（Failover）

```typescript
// 配置文件支持多认证配置文件
{
  "agents": {
    "main": {
      "authProfiles": ["anthropic-primary", "anthropic-backup", "openai-fallback"]
    }
  }
}
```

故障转移策略：
- **auth**: API Key 无效或过期
- **rate_limit**: 速率限制
- **overloaded**: 服务过载
- **billing**: 计费问题
- **timeout**: 请求超时

#### 2. 上下文窗口管理

```typescript
// 三层防护机制
1. 预检: evaluateContextWindowGuard()    // 运行前检查
2. 监控: context-window-guard.ts         // 运行时监控  
3. 压缩: compactEmbeddedPiSession()      // 溢出时压缩
```

压缩策略（Compaction）：
- **Summarization**: 将早期对话总结为摘要
- **Token Truncation**: 按优先级截断消息
- **Smart Pruning**: 保留关键工具调用结果

#### 3. 流式处理架构

```typescript
// src/agents/pi-embedded-subscribe.ts
export function subscribeEmbeddedPiSession(params) {
  const state: EmbeddedPiSubscribeState = {
    assistantTexts: [],      // 累积的助手消息
    toolMetas: [],           // 工具调用元数据
    blockState: {            // 块级解析状态
      thinking: false,       // <think> 块
      final: false,          // <final> 块
      inlineCode: {...}      // 代码跨度跟踪
    }
  };
  
  // 订阅 Pi Session 事件流
  const sessionUnsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );
}
```

---

## ACP 控制平面

### Agent Client Protocol (ACP)

ACP 是 OpenClaw 实现的开放协议，用于标准化 Agent 与客户端（如 IDE）的通信：

```
┌─────────────┐      NDJSON Stream      ┌─────────────┐
│   IDE/CLI   │  ◄──────────────────►  │  ACP Server │
│   Client    │   (stdin/stdout)       │  (Gateway)  │
└─────────────┘                        └──────┬──────┘
                                              │
                                         WebSocket
                                              │
                                        ┌─────┴─────┐
                                        │  Gateway  │
                                        │  Server   │
                                        └───────────┘
```

### 核心组件

#### 1. ACP Server

```typescript
// src/acp/server.ts
export async function serveAcpGateway(opts: AcpServerOptions): Promise<void> {
  // 1. 建立 Gateway WebSocket 连接
  const gateway = new GatewayClient({
    url: connection.url,
    token: creds.token,
    onEvent: (evt) => agent?.handleGatewayEvent(evt),
  });
  
  // 2. 启动 ACP 协议处理
  new AgentSideConnection((conn) => {
    agent = new AcpGatewayAgent(conn, gateway, opts);
    agent.start();
  }, stream);
}
```

#### 2. AcpGatewayAgent

```typescript
// src/acp/translator.ts
export class AcpGatewayAgent implements Agent {
  // 会话生命周期管理
  async newSession(params: NewSessionRequest): Promise<NewSessionResponse>
  async loadSession(params: LoadSessionRequest): Promise<LoadSessionResponse>
  async prompt(params: PromptRequest): Promise<PromptResponse>
  async cancel(params: CancelNotification): Promise<void>
  
  // 配置管理
  async setSessionMode(params: SetSessionModeRequest)
  async setSessionConfigOption(params: SetSessionConfigOptionRequest)
  
  // 事件处理
  async handleGatewayEvent(evt: EventFrame)
  async handleChatEvent(evt: EventFrame)
  async handleAgentEvent(evt: EventFrame)
}
```

#### 3. Session Manager

```typescript
// src/acp/control-plane/manager.ts
export class AcpSessionManager {
  // 会话生命周期
  async initializeSession(input: AcpInitializeSessionInput)
  async runTurn(input: AcpRunTurnInput)
  async closeSession(input: AcpCloseSessionInput)
  
  // 运行时控制
  private sessionActorQueue: SessionActorQueue
  private runtimeCache: AcpRuntimeCache
  
  // 身份协调
  async reconcileIdentity(): Promise<AcpStartupIdentityReconcileResult>
}
```

### ACP 能力声明

```typescript
// src/acp/translator.ts
async initialize(): Promise<InitializeResponse> {
  return {
    protocolVersion: PROTOCOL_VERSION,
    agentCapabilities: {
      loadSession: true,                    // 支持会话恢复
      promptCapabilities: {
        image: true,                        // 支持图像输入
        audio: false,                       // 暂不支持音频
        embeddedContext: true,              // 支持嵌入式上下文
      },
      mcpCapabilities: {
        http: false,                        // MCP HTTP 暂不支持
        sse: false,                         // MCP SSE 暂不支持
      },
      sessionCapabilities: {
        list: {},                           // 支持会话列表
      },
    },
  };
}
```

---

## 工具系统

### 工具架构

```
Tool System
├── Core Tools (pi-coding-agent)
│   ├── read      - 文件读取
│   ├── write     - 文件写入
│   ├── edit      - 文件编辑
│   ├── bash      - 命令执行
│   └── apply_patch - 补丁应用
├── OpenClaw Tools
│   ├── Browser   - 浏览器自动化
│   ├── Canvas    - 实时画布操作
│   ├── Memory    - 内存搜索
│   ├── Sessions  - 子代理管理
│   ├── Cron      - 定时任务
│   └── Channels  - 渠道操作
└── Plugin Tools
    └── 动态加载的第三方工具
```

### 工具创建流程

```typescript
// src/agents/pi-tools.ts
export function createOpenClawCodingTools(options: ToolOptions): AnyAgentTool[] {
  // 1. 获取基础编码工具
  const base = codingTools.flatMap((tool) => {
    if (tool.name === readTool.name) {
      // 2. 根据沙箱状态包装工具
      return sandboxRoot 
        ? createSandboxedReadTool({ root, bridge })
        : createOpenClawReadTool(freshReadTool, { ... });
    }
    // ...
  });
  
  // 3. 添加 OpenClaw 特有工具
  const tools: AnyAgentTool[] = [
    ...base,
    execTool,
    processTool,
    ...createOpenClawTools({ ... }),
  ];
  
  // 4. 应用策略管道过滤
  const filtered = applyToolPolicyPipeline({
    tools,
    steps: [
      { policy: profilePolicy, label: "profile" },
      { policy: groupPolicy, label: "group" },
      { policy: sandbox?.tools, label: "sandbox" },
      { policy: subagentPolicy, label: "subagent" },
    ]
  });
  
  // 5. 包装 Hook 和 AbortSignal
  return filtered
    .map(tool => wrapToolWithBeforeToolCallHook(tool, { ... }))
    .map(tool => options.abortSignal 
      ? wrapToolWithAbortSignal(tool, options.abortSignal) 
      : tool
    );
}
```

### 工具策略系统

```typescript
// src/agents/tool-policy.ts
interface ToolPolicy {
  allow?: string[];      // 允许列表
  deny?: string[];       // 拒绝列表
  alsoAllow?: string[];  // 额外允许（继承场景）
}

// 多层策略合并（优先级从低到高）
const policies = [
  globalPolicy,           // 全局默认
  globalProviderPolicy,   // 全局提供商策略
  agentPolicy,            // Agent 级别
  agentProviderPolicy,    // Agent 提供商级别
  profilePolicy,          // 认证配置文件级别
  groupPolicy,            // 群组/渠道级别
  sandbox?.tools,         // 沙箱限制
  subagentPolicy,         // 子代理限制
];
```

### 沙箱工具执行

```typescript
// src/agents/sandbox/fs-bridge.ts
export interface SandboxFsBridge {
  // 路径安全检查
  resolveSafePath(requested: string): SafePathResult;
  
  // 文件操作
  readFile(relPath: string): Promise<string>;
  writeFile(relPath: string, content: string): Promise<void>;
  editFile(relPath: string, oldStr: string, newStr: string): Promise<void>;
  
  // 命令执行（通过 Docker/Podman）
  exec(params: {
    command: string[];
    cwd?: string;
    env?: Record<string, string>;
  }): Promise<ExecResult>;
}
```

---

## 内存与上下文管理

### 内存架构

```
Memory System
├── Embedding Providers
│   ├── OpenAI (text-embedding-3-*)
│   ├── Gemini (embedding-001)
│   ├── Ollama (本地嵌入)
│   ├── Voyage AI
│   └── Mistral AI
├── Storage Backend
│   ├── SQLite (元数据)
│   ├── sqlite-vec (向量存储)
│   └── FTS5 (全文检索)
├── Index Sources
│   ├── memory/          # 项目文档
│   ├── sessions/        # 会话历史
│   └── extraPaths/      # 额外配置路径
└── Search Modes
    ├── Vector Only      # 纯语义搜索
    ├── FTS Only         # 纯关键词搜索
    └── Hybrid (MMR)     # 混合搜索 + 多样性
```

### 内存管理器实现

```typescript
// src/memory/manager.ts
export class MemoryIndexManager implements MemorySearchManager {
  private db: DatabaseSync;                    // SQLite 数据库
  private provider: EmbeddingProvider | null;   // 嵌入提供商
  private vector: {                             // 向量配置
    enabled: boolean;
    available: boolean | null;
    dims?: number;
  };
  private fts: {                                // 全文检索配置
    enabled: boolean;
    available: boolean;
  };
  
  // 混合搜索实现
  async search(query: string, opts?: {
    maxResults?: number;
    minScore?: number;
    sessionKey?: string;
  }): Promise<MemorySearchResult[]> {
    // 1. 同步检查（如有脏数据）
    if (this.dirty || this.sessionsDirty) {
      await this.sync({ reason: "search" });
    }
    
    // 2. 提取关键词
    const keywords = extractKeywords(cleaned);
    
    // 3. 并行执行向量和关键词搜索
    const [vectorResults, keywordResults] = await Promise.all([
      this.searchVector(queryVec, candidates),
      this.searchKeyword(cleaned, candidates),
    ]);
    
    // 4. 混合排序（支持 MMR 多样性）
    return this.mergeHybridResults({
      vector: vectorResults,
      keyword: keywordResults,
      vectorWeight: hybrid.vectorWeight,
      textWeight: hybrid.textWeight,
      mmr: hybrid.mmr,                    // Maximal Marginal Relevance
      temporalDecay: hybrid.temporalDecay, // 时间衰减
    });
  }
}
```

### 混合搜索算法

```typescript
// src/memory/hybrid.ts
export function mergeHybridResults(params: {
  vector: VectorResult[];
  keyword: KeywordResult[];
  vectorWeight: number;
  textWeight: number;
  mmr?: { enabled: boolean; lambda: number };
  temporalDecay?: { enabled: boolean; halfLifeDays: number };
}): MemorySearchResult[] {
  // 1. 分数归一化与加权
  const merged = new Map<string, MergedResult>();
  
  for (const v of params.vector) {
    const existing = merged.get(v.id);
    existing.vectorScore = v.score * params.vectorWeight;
  }
  
  for (const k of params.keyword) {
    const existing = merged.get(k.id);
    existing.textScore = k.textScore * params.textWeight;
  }
  
  // 2. 混合分数计算
  for (const entry of merged.values()) {
    entry.score = (entry.vectorScore ?? 0) + (entry.textScore ?? 0);
  }
  
  // 3. MMR 重排序（如果启用）
  if (params.mmr?.enabled) {
    return applyMMR([...merged.values()], params.mmr.lambda);
  }
  
  // 4. 时间衰减（如果启用）
  if (params.temporalDecay?.enabled) {
    return applyTemporalDecay([...merged.values()], params.temporalDecay.halfLifeDays);
  }
  
  return [...merged.values()].sort((a, b) => b.score - a.score);
}
```

---

## 多代理路由

### 子代理系统

```typescript
// src/agents/tools/sessions-spawn-tool.ts
async function spawnSubagent(params: {
  prompt: string;
  model?: string;           // 可选覆盖模型
  session?: string;         // 目标会话（可选）
  inherit?: boolean;        // 是否继承父代理上下文
  depth?: number;           // 嵌套深度限制
}): Promise<SpawnResult> {
  // 1. 检查深度限制
  const currentDepth = getSubagentDepthFromSessionStore(parentSessionKey);
  if (currentDepth >= MAX_SUBAGENT_DEPTH) {
    throw new Error(`Maximum subagent depth (${MAX_SUBAGENT_DEPTH}) exceeded`);
  }
  
  // 2. 创建子会话
  const childSessionKey = generateSubagentSessionKey(parentSessionKey, currentDepth + 1);
  
  // 3. 继承策略（工具允许列表等）
  const inheritedPolicy = params.inherit 
    ? resolveSubagentToolPolicy(config, currentDepth + 1)
    : getDefaultSubagentPolicy();
  
  // 4. 启动子代理运行
  return runEmbeddedPiAgent({
    sessionKey: childSessionKey,
    prompt: params.prompt,
    toolPolicy: inheritedPolicy,
    spawnedBy: parentSessionKey,
  });
}
```

### 路由决策流程

```
Inbound Message
       │
       ▼
┌──────────────────┐
│  Session Resolver │  ← 根据 sender/channel/agent 解析目标会话
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Policy Engine    │  ← 应用工具策略、沙箱策略、子代理策略
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Model Selector   │  ← 根据配置选择模型（支持覆盖）
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Pi Runner        │  ← 执行 Agent 运行循环
└──────────────────┘
```

---

## AI 原生程序设计哲学

### 1. 工具即接口（Tools as Interface）

传统程序：函数是能力的边界
```typescript
// 传统方式：硬编码调用链
const result = await analyzeData(await fetchData());
```

AI 原生：工具是 Agent 的能力扩展
```typescript
// AI 原生方式：Agent 自主决策调用哪些工具
const tools = [fetchDataTool, analyzeDataTool, visualizeTool];
const result = await agent.run({ prompt: "分析销售数据", tools });
// Agent 可能：fetch -> analyze -> visualize -> summarize
```

### 2. 上下文即状态（Context as State）

```typescript
// 消息历史作为核心状态
interface SessionContext {
  messages: AgentMessage[];        // 对话历史
  toolResults: ToolResult[];       // 工具调用结果
  memory: MemorySearchResult[];    // 检索到的记忆
  systemPrompt: string;            // 动态系统提示词
}

// 状态转换
context -> LLM -> { text, toolCalls } -> executeTools -> newContext
```

### 3. 流式即默认（Streaming by Default）

```typescript
// 传统：等待完整响应
const response = await llm.complete(prompt);  // 阻塞等待

// AI 原生：流式处理
const stream = llm.stream(prompt);
for await (const chunk of stream) {
  // 实时处理：渲染 UI、发送给用户、流式工具调用
  await handleChunk(chunk);
}
```

### 4. 故障即切换（Failure as Failover）

```typescript
// 内置多提供商容错
const runWithFailover = async () => {
  const providers = ["anthropic", "openai", "google"];
  for (const provider of providers) {
    try {
      return await runWithProvider(provider);
    } catch (err) {
      if (isRetryable(err)) continue;
      throw err;
    }
  }
};
```

### 5. 提示词即代码（Prompts as Code）

```typescript
// 类型安全的提示词工程
interface SystemPromptParams {
  skills: Skill[];
  sandboxInfo: EmbeddedSandboxInfo;
  ownerDisplay: string;
  toolUsageStyle: "sequential" | "parallel";
}

const createSystemPrompt = (params: SystemPromptParams): string => {
  return renderTemplate(SYSTEM_PROMPT_TEMPLATE, {
    skills: formatSkills(params.skills),
    sandbox: formatSandboxInfo(params.sandboxInfo),
    // ...
  });
};
```

### 6. 沙箱即安全（Sandbox as Security）

```typescript
// 多层安全防护
interface SandboxConfig {
  mode: "off" | "docker" | "podman";
  workspaceAccess: "none" | "ro" | "rw";
  networkMode: "none" | "host" | "custom";
  allowedTools: string[];
  securityOpts: {
    seccomp: string;      // 系统调用过滤
    apparmor: string;     // MAC 策略
    noNewPrivileges: boolean;
    capDrop: string[];    // 丢弃 Linux capabilities
  };
}
```

---

## 总结

OpenClaw 的 Agent 实现展示了一个生产级 AI 原生程序的完整架构：

1. **分层架构**：网关 → ACP → Pi 运行时 → 工具/内存/沙箱，每层职责清晰
2. **协议驱动**：ACP 协议实现标准化 Agent 通信，支持 IDE 集成
3. **容错设计**：多模型故障转移、上下文压缩、认证轮换，确保高可用
4. **安全优先**：多层工具策略、沙箱隔离、路径安全检查
5. **性能优化**：流式处理、混合搜索、智能缓存、批量嵌入

这套架构不仅支撑了 OpenClaw 的 25+ 渠道接入和复杂工具调用，也为构建可靠的 AI 原生应用提供了可复用的设计模式。

---

*文档生成时间：2026-03-10*
*基于 OpenClaw 源码分析*
