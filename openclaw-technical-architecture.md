# OpenClaw 技术架构详解

## 1. 总体架构

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

## 2. Agent 运行时架构

### 2.1 Pi 运行时集成

OpenClaw 使用 **Pi (Prompt Interface)** 作为核心 Agent 引擎，以 **Embedded Mode** 集成：

```typescript
// 嵌入式运行方式
import { createAgentSession } from "@mariozechner/pi-coding-agent";

const { session } = await createAgentSession({
  model: resolvedModel,
  tools: builtInTools,      // Pi 的基础工具
  customTools: openClawTools, // OpenClaw 自定义工具
  sessionManager,
  settingsManager,
  resourceLoader,
});

// 直接调用 session.prompt() 触发 Agent 循环
await session.prompt(userPrompt, { images });
```

**关键特性：**
- **非 RPC 模式**：直接函数调用，无进程间通信开销
- **自定义工具注入**：完全替换 Pi 的默认工具集
- **事件订阅机制**：通过 `session.subscribe()` 监听流式事件
- **会话持久化**：JSONL 格式的对话历史管理

### 2.2 Agent 循环流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent 执行循环                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────┐                                               │
│   │  用户输入    │                                               │
│   └──────┬──────┘                                               │
│          ▼                                                      │
│   ┌─────────────────────────┐                                   │
│   │ 1. 构建 System Prompt   │  ← 动态构建（含 Skills/Sandbox）  │
│   │ 2. 加载 Tools           │  ← 根据策略过滤后的工具集          │
│   │ 3. 准备 History         │  ← 加载会话历史                    │
│   └───────────┬─────────────┘                                   │
│               ▼                                                 │
│   ┌─────────────────────────┐                                   │
│   │  Pi Agent Loop          │  ← 由 pi-agent-core 处理           │
│   │                         │                                   │
│   │   ┌─────────────────┐   │                                   │
│   │   │ 调用 LLM API    │   │                                   │
│   │   └────────┬────────┘   │                                   │
│   │            ▼            │                                   │
│   │   ┌─────────────────┐   │                                   │
│   │   │ 响应解析        │   │                                   │
│   │   │ - 文本内容      │   │                                   │
│   │   │ - tool_calls   │   │                                   │
│   │   └────────┬────────┘   │                                   │
│   │            │            │                                   │
│   │   ┌────────┴────────┐   │                                   │
│   │   │ 有 tool_calls?  │───┼─── No ───▶ 返回最终响应           │
│   │   └────────┬────────┘   │                                   │
│   │            │ Yes        │                                   │
│   │            ▼            │                                   │
│   │   ┌─────────────────┐   │                                   │
│   │   │ 执行工具调用    │   │  ← OpenClaw 工具在此执行          │
│   │   │ - 参数解析      │   │                                   │
│   │   │ - 权限检查      │   │                                   │
│   │   │ - 实际执行      │   │                                   │
│   │   │ - 结果格式化    │   │                                   │
│   │   └────────┬────────┘   │                                   │
│   │            │            │                                   │
│   │            └────────────┼─── 结果加入历史 ───▶ 继续循环      │
│   │                         │                                   │
│   └─────────────────────────┘                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 工具系统架构

### 3.1 工具分层结构

```
Tool System
├── Core Tools (from @mariozechner/pi-coding-agent)
│   ├── read       - 文件读取（被 OpenClaw 包装）
│   ├── write      - 文件写入（被 OpenClaw 包装）
│   ├── edit       - 文件编辑（被 OpenClaw 包装）
│   ├── bash       - 被替换为 exec/process
│   └── apply_patch - OpenAI Codex 补丁工具
│
├── OpenClaw Core Tools
│   ├── browser    - 浏览器自动化（Playwright）
│   ├── canvas     - 实时画布渲染
│   ├── cron       - 定时任务管理
│   ├── exec       - 命令执行（替换 bash）
│   ├── process    - 后台进程管理
│   ├── image      - 图像生成/处理
│   ├── memory_search - 内存语义搜索
│   ├── message    - 消息发送
│   ├── nodes      - 节点执行
│   ├── pdf        - PDF 处理
│   ├── sessions_* - 会话管理工具组
│   ├── subagents  - 子代理管理
│   ├── tts        - 文本转语音
│   ├── web_fetch  - 网页获取
│   └── web_search - 网络搜索
│
├── Channel Action Tools
│   ├── discord_*  - Discord API 操作（30+ actions）
│   ├── telegram_* - Telegram API 操作
│   ├── slack_*    - Slack API 操作
│   └── whatsapp_* - WhatsApp API 操作
│
└── Plugin Tools (动态加载)
    └── 扩展提供的自定义工具
```

### 3.2 工具注册机制

#### 3.2.1 注册流程

```typescript
// src/agents/pi-tools.ts
export function createOpenClawCodingTools(options?: {...}): AnyAgentTool[] {
  // Step 1: 获取 Pi 基础工具并替换
  const base = (codingTools as AnyAgentTool[]).flatMap((tool) => {
    if (tool.name === readTool.name) {
      // 根据沙箱状态创建对应版本
      return sandboxRoot 
        ? createSandboxedReadTool({ root, bridge })
        : createOpenClawReadTool(freshReadTool, {...});
    }
    if (tool.name === "bash" || tool.name === execToolName) {
      return []; // 移除 bash，后续添加 exec
    }
    // ... write/edit 同理
    return [tool];
  });

  // Step 2: 创建 OpenClaw 特有工具
  const tools: AnyAgentTool[] = [
    ...base,
    execTool,           // 替换 bash
    processTool,
    ...createOpenClawTools({...}), // OpenClaw 核心工具
  ];

  // Step 3: 应用策略管道过滤
  const subagentFiltered = applyToolPolicyPipeline({
    tools,
    steps: [
      { policy: profilePolicy, label: "profile" },
      { policy: groupPolicy, label: "group" },
      { policy: sandbox?.tools, label: "sandbox" },
      { policy: subagentPolicy, label: "subagent" },
    ],
  });

  // Step 4: 参数标准化和 Hook 包装
  const normalized = subagentFiltered.map((tool) =>
    normalizeToolParameters(tool, { modelProvider, modelId })
  );
  
  const withHooks = normalized.map((tool) =>
    wrapToolWithBeforeToolCallHook(tool, {
      agentId, sessionKey, runId, loopDetection,
    })
  );

  // Step 5: AbortSignal 包装
  return options?.abortSignal
    ? withHooks.map((tool) => wrapToolWithAbortSignal(tool, options.abortSignal))
    : withHooks;
}
```

#### 3.2.2 插件工具加载

```typescript
// src/plugins/tools.ts
export function resolvePluginTools(params: {...}): AnyAgentTool[] {
  // 加载插件注册表
  const registry = loadOpenClawPlugins({ config, workspaceDir });

  for (const entry of registry.tools) {
    // 调用插件工厂函数创建工具
    const resolved = entry.factory(params.context);
    
    // 检查名称冲突
    if (existing.has(tool.name)) {
      // 报错或跳过
    }
    
    // 注册工具元数据
    pluginToolMeta.set(tool, {
      pluginId: entry.pluginId,
      optional: entry.optional,
    });
    
    tools.push(tool);
  }
  
  return tools;
}
```

#### 3.2.3 工具策略管道

```typescript
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

### 3.3 工具调用协议

#### 3.3.1 接口定义

```typescript
// 核心接口（来自 @mariozechner/pi-agent-core）
interface AgentTool<TParams, TResult> {
  name: string;           // 工具唯一标识
  label?: string;         // 显示名称
  description: string;    // 功能描述（给 LLM 看）
  parameters: JSONSchema; // JSON Schema 参数定义
  ownerOnly?: boolean;    // 是否仅限所有者
  
  execute: (
    toolCallId: string,                    // 调用 ID
    params: TParams,                       // 解析后的参数
    signal?: AbortSignal,                  // 取消信号
    onUpdate?: AgentToolUpdateCallback     // 流式更新回调
  ) => Promise<TResult>;
}

// 工具执行结果
interface AgentToolResult<T> {
  content: Array<{
    type: "text" | "image";
    text?: string;        // 文本内容
    data?: string;        // 图片 base64
    mimeType?: string;    // 图片 MIME
  }>;
  details?: T;            // 详细数据
}

// 流式更新回调
type AgentToolUpdateCallback<T> = (update: {
  type: "update";
  content: Array<{ type: "text"; text: string }>;
}) => void;
```

#### 3.3.2 Pi 适配层

Pi 使用 `ToolDefinition` 接口，与 `AgentTool` 略有不同，需要适配：

```typescript
// src/agents/pi-tool-definition-adapter.ts
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    
    execute: async (...args) => {
      const { toolCallId, params, onUpdate, signal } = splitToolExecuteArgs(args);
      
      // 执行 beforeToolCall hook（策略检查）
      const hookOutcome = await runBeforeToolCallHook({...});
      if (hookOutcome.blocked) {
        throw new Error(hookOutcome.reason);
      }
      
      // 调用实际工具
      const rawResult = await tool.execute(
        toolCallId,
        hookOutcome.params,
        signal,
        onUpdate
      );
      
      // 标准化结果
      return normalizeToolExecutionResult({ toolName, result: rawResult });
    },
  }));
}
```

#### 3.3.3 调用时序图

```
┌─────────┐    ┌─────────────┐    ┌─────────────────┐    ┌──────────────┐
│   LLM   │    │  Pi Agent   │    │  Tool Adapter   │    │  Tool Impl   │
└────┬────┘    └──────┬──────┘    └────────┬────────┘    └──────┬───────┘
     │                │                     │                    │
     │ 1. 生成 tool_call                    │                    │
     │────────────────▶                     │                    │
     │                │                     │                    │
     │                │ 2. 匹配 ToolDefinition                    │
     │                │────────────────────▶│                    │
     │                │                     │                    │
     │                │ 3. 调用 execute()   │                    │
     │                │────────────────────▶│                    │
     │                │                     │                    │
     │                │                     │ 4. 参数解析/验证    │
     │                │                     │                    │
     │                │                     │ 5. 权限检查         │
     │                │                     │                    │
     │                │                     │ 6. 调用 AgentTool   │
     │                │                     │───────────────────▶│
     │                │                     │                    │
     │                │                     │ 7. 执行业务逻辑     │
     │                │                     │◀───────────────────│
     │                │                     │                    │
     │                │                     │ 8. 返回结果        │
     │                │                     │                    │
     │                │ 9. 标准化结果       │                    │
     │                │◀────────────────────│                    │
     │                │                     │                    │
     │ 10. 加入历史   │                     │                    │
     │◀───────────────│                     │                    │
     │                │                     │                    │
```

---

## 4. 渠道接入层架构

### 4.1 渠道分层

| 类型 | 数量 | 代表渠道 | 实现位置 |
|------|------|----------|----------|
| **核心内置** | 9 | Telegram, Discord, Slack, Signal, iMessage, WhatsApp, IRC, Google Chat, LINE | `src/<channel>/` |
| **扩展渠道** | 15+ | Matrix, MS Teams, Feishu, Zalo, Mattermost, Nextcloud Talk, Nostr, Twitch, BlueBubbles, Tlon, Synology Chat | `extensions/<channel>/` |
| **其他扩展** | 10+ | voice-call, memory-core, phone-control, device-pair | `extensions/<feature>/` |

**总计：约 25+ 消息平台**

### 4.2 渠道注册机制

```typescript
// src/channels/registry.ts
export const CHAT_CHANNEL_ORDER = [
  "telegram",
  "whatsapp",
  "discord",
  "irc",
  "googlechat",
  "slack",
  "signal",
  "imessage",
  "line",
] as const;

// 扩展渠道通过 package.json 注册
// extensions/matrix/package.json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "matrix",
      "label": "Matrix",
      "selectionLabel": "Matrix (plugin)",
      "docsPath": "/channels/matrix",
      "order": 70
    }
  }
}
```

### 4.3 渠道消息处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      渠道消息处理流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │ 渠道消息接收  │  Telegram: webhook/long polling              │
│  │              │  Discord: websocket gateway                   │
│  │              │  Slack: Socket Mode                           │
│  └──────┬───────┘                                               │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │ 1. 消息标准化     │  转换为内部统一格式                        │
│  │ 2. 发送者识别     │  解析 senderId, senderName                │
│  │ 3. 会话解析       │  生成 sessionKey (agent:channel:id)       │
│  └────────┬─────────┘                                           │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ 4. 允许列表检查   │  dmPolicy: pairing | open                 │
│  │ 5. 命令解析       │  检测 /command 或自然语言指令              │
│  └────────┬─────────┘                                           │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ 6. 路由决策       │  确定目标 agentId                         │
│  │ 7. 上下文加载     │  加载历史消息、记忆                         │
│  └────────┬─────────┘                                           │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ 8. 触发 Agent    │  调用 runEmbeddedPiAgent()                │
│  │ 9. 流式响应      │  实时发送增量内容                          │
│  └────────┬─────────┘                                           │
│           ▼                                                     │
│  ┌──────────────────┐                                           │
│  │ 10. 渠道适配输出  │  转换为渠道特定格式                        │
│  │ 11. 发送消息      │  调用渠道 API 发送                         │
│  └──────────────────┘                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 渠道动作工具

每个渠道可注册特定的 Action Tools：

```typescript
// src/agents/tools/discord-actions.ts
const messagingActions = new Set([
  "react", "reactions", "sticker", "poll", "permissions",
  "fetchMessage", "readMessages", "sendMessage", "editMessage", "deleteMessage",
  "threadCreate", "threadList", "threadReply",
  "pinMessage", "unpinMessage", "listPins", "searchMessages",
]);

const guildActions = new Set([
  "memberInfo", "roleInfo", "emojiList", "emojiUpload",
  "channelCreate", "channelEdit", "channelDelete",
  "categoryCreate", "categoryEdit", "categoryDelete",
]);

const moderationActions = new Set(["timeout", "kick", "ban"]);
```

---

## 5. 核心数据流

### 5.1 单次请求完整数据流

```
User Input
    │
    ▼
┌────────────────────────────────────────────────────────────────┐
│ Channel Adapter (渠道特定实现)                                    │
│ - 接收原始消息                                                   │
│ - 标准化为 InternalMessage 格式                                   │
│ - 提取 metadata (sender, chat, timestamp)                        │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Message Router                                                   │
│ - resolveSessionKey()  → "agent:telegram:+1234567890"           │
│ - checkAllowlist()     → 验证发送者权限                          │
│ - resolveAgentId()     → 确定处理 Agent                          │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Session Manager                                                  │
│ - loadSessionFile()    → ~/.openclaw/agents/<agent>/sessions/   │
│ - limitHistoryTurns()  → 根据渠道类型限制历史长度                  │
│ - injectMemory()       → memory_search 结果注入                  │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Agent Runner (runEmbeddedPiAgent)                                │
│ - resolveModel()       → 选择模型 (provider/model)               │
│ - createTools()        → 根据上下文创建工具集                     │
│ - buildSystemPrompt()  → 动态构建系统提示                         │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Pi Agent Loop                                                    │
│ - session.prompt()     → 触发 Pi 的 Agent 循环                   │
│ - LLM API Call         → 流式调用                               │
│ - Tool Execution       → 执行工具调用                            │
│   - beforeToolCall hook                                          │
│   - Tool.execute()                                               │
│   - afterToolCall hook                                           │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Stream Handler                                                   │
│ - subscribeEmbeddedPiSession()                                   │
│ - 处理 message_start/end/update                                  │
│ - 处理 tool_execution_start/end                                  │
│ - 提取 <think> / <final> 块                                      │
│ - 块级消息切分 (block chunking)                                  │
└────────────────────┬───────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Output Adapter                                                   │
│ - 解析 reply directives ([[media:url]], [[voice]])              │
│ - 转换为渠道特定格式                                             │
│ - sendMessage() 调用渠道 API                                     │
└────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键配置文件

| 文件 | 用途 |
|------|------|
| `src/channels/registry.ts` | 核心渠道注册表 |
| `src/agents/pi-tools.ts` | 工具创建与策略应用 |
| `src/plugins/tools.ts` | 插件工具加载 |
| `src/agents/pi-tool-definition-adapter.ts` | Pi 工具接口适配 |
| `src/agents/pi-embedded-runner/run.ts` | Agent 主运行循环 |
| `src/agents/pi-embedded-subscribe.ts` | 流式事件订阅处理 |

---

## 7. 扩展点

### 7.1 添加新渠道

1. 在 `extensions/<channel>/` 创建扩展
2. 实现 Channel Plugin 接口
3. 在 `package.json` 中注册 `openclaw.channel`
4. 可选：添加渠道特定的 Action Tools

### 7.2 添加新工具

1. 在 `src/agents/tools/` 创建工具实现
2. 实现 `AgentTool` 接口
3. 在 `createOpenClawTools()` 中注册
4. 或通过 Plugin 机制动态加载

### 7.3 自定义策略

通过配置文件控制工具可用性：

```json
{
  "agents": {
    "main": {
      "tools": {
        "allow": ["read", "write", "browser"],
        "deny": ["exec"]
      }
    }
  }
}
```

---

## 8. 性能考量

| 方面 | 优化策略 |
|------|----------|
| **工具加载** | 惰性加载 + 缓存策略管道结果 |
| **会话历史** | 自动 compaction + 智能截断 |
| **流式处理** | 增量渲染 + 块级去重 |
| **多模型** | 认证配置文件轮换 + 故障转移 |
| **沙箱** | 容器复用 + 文件系统桥接 |

---

*文档基于 OpenClaw 源码分析生成*
