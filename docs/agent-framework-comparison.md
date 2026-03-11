# OpenClaw vs LangGraph：Agent 开发框架对比分析

> 本文档深入对比 OpenClaw 的 Agent 架构与 LangGraph 的状态机实现方式，帮助有 LangGraph 背景的开发者理解 OpenClaw 的设计哲学。

---

## 目录

1. [核心概念对比](#核心概念对比)
2. [架构模式对比](#架构模式对比)
3. [状态管理对比](#状态管理对比)
4. [控制流对比](#控制流对比)
5. [多 Agent 协作对比](#多-agent-协作对比)
6. [代码示例对比](#代码示例对比)
7. [设计哲学差异](#设计哲学差异)
8. [适用场景建议](#适用场景建议)

---

## 核心概念对比

### LangGraph 的核心概念

```python
# LangGraph 的核心是 "StateGraph"
from langgraph.graph import StateGraph, END

# 1. State - 状态模式定义
class AgentState(TypedDict):
    messages: list
    next_node: str
    tool_calls: list

# 2. Node - 状态节点（函数）
def agent_node(state: AgentState):
    # 执行逻辑，返回状态更新
    return {"messages": [...], "next_node": "tool_node"}

# 3. Edge - 状态转移（条件或无条件）
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_edge("agent", "tool_node")  # 或条件边
graph.add_conditional_edges("agent", router_function)
```

### OpenClaw 的核心概念

```typescript
// OpenClaw 的核心是 "Session" + "Event Stream"
// 没有显式的图结构，而是通过以下概念组织：

// 1. Session - 会话上下文（类似 State，但更丰富）
interface SessionContext {
  messages: AgentMessage[];        // 对话历史
  sessionKey: string;              // 会话标识
  tools: AnyAgentTool[];           // 可用工具
  memory: MemorySearchResult[];    // 检索记忆
}

// 2. Pi Runtime - 运行循环（隐含状态机）
// 不是显式节点，而是通过 while 循环 + LLM 自主决策
async function runEmbeddedPiAgent(params) {
  while (runLoopIterations < MAX_RUN_LOOP_ITERATIONS) {
    // 构建请求载荷
    const payload = buildEmbeddedRunPayloads({ ... });
    // 执行单次尝试
    const attempt = await runEmbeddedAttempt({ ... });
    // 处理上下文溢出/故障转移
    if (contextOverflowError) { ... }
    if (shouldFailover(error)) { ... }
  }
}

// 3. Tool System - 工具即边（隐式转移）
// LLM 通过 function calling 决定"转移到"哪个工具
const tools = [readTool, writeTool, bashTool, browserTool];
```

---

## 架构模式对比

### LangGraph：显式状态机（Explicit State Machine）

```
┌─────────────────────────────────────────────────────────────┐
│                     LangGraph 架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐        │
│   │  START   │─────▶│  Agent   │─────▶│  Router  │        │
│   └──────────┘      └──────────┘      └────┬─────┘        │
│                                             │               │
│                              ┌──────────────┼──────────┐   │
│                              ▼              ▼          ▼   │
│                         ┌────────┐    ┌────────┐   ┌──────┐│
│                         │ Tool A │    │ Tool B │   │ END  ││
│                         └───┬────┘    └───┬────┘   └──────┘│
│                             │             │                │
│                             └──────┬──────┘                │
│                                    ▼                       │
│                              ┌──────────┐                  │
│                              │  Agent   │ (循环)            │
│                              └──────────┘                  │
│                                                             │
│  特点：                                                      │
│  - 图结构是显式定义的（add_node, add_edge）                    │
│  - 状态流转由开发者编码控制                                   │
│  - 每个节点是一个函数，输入/输出 State                         │
│  - 条件边通过 router function 决定下一跳                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### OpenClaw：隐式反应循环（Implicit ReAct Loop）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          OpenClaw 架构                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        Pi Runtime 核心循环                           │  │
│   │                                                                    │  │
│   │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │  │
│   │   │  接收    │───▶│ 构建     │───▶│ LLM      │───▶│ 解析     │    │  │
│   │   │  Prompt  │    │ Payload  │    │ 推理     │    │ 响应     │    │  │
│   │   └──────────┘    └──────────┘    └──────────┘    └────┬─────┘    │  │
│   │                                                        │          │  │
│   │                              ┌─────────────────────────┘          │  │
│   │                              ▼                                    │  │
│   │   ┌──────────┐    ┌──────────┐    ┌──────────┐                    │  │
│   │   │ 总结     │◀───│ 新消息   │◀───│ 工具     │◀───┐               │  │
│   │   │ 输出     │    │ 历史     │    │ 执行     │    │               │  │
│   │   └──────────┘    └──────────┘    └──────────┘    │               │  │
│   │                                                    │               │  │
│   │                              ┌─────────────────────┘               │  │
│   │                              │  (无工具调用，或达到最大轮数)         │  │
│   │                              ▼                                    │  │
│   │                        ┌──────────┐                               │  │
│   │                        │   END    │                               │  │
│   │                        └──────────┘                               │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   特点：                                                                     │
│   - 没有显式的图结构定义                                                     │
│   - "下一跳"由 LLM 通过 function calling 自主决定                              │
│   - 状态转换是隐式的（通过消息历史和工具结果驱动）                              │
│   - 循环由运行时管理，直到满足终止条件                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 状态管理对比

### LangGraph：集中式状态

```python
# LangGraph 的状态是集中管理的 TypedDict
# 所有节点共享同一个 state 对象

def agent_node(state: AgentState):
    # 读取状态
    messages = state["messages"]
    
    # 修改状态（返回部分更新）
    response = llm.invoke(messages)
    return {
        "messages": messages + [response],
        "next_node": "tool_node"  # 显式控制流转
    }

def tool_node(state: AgentState):
    # 工具执行
    tool_calls = state["tool_calls"]
    results = execute_tools(tool_calls)
    return {
        "messages": state["messages"] + results,
        "next_node": "agent"  # 返回 agent 节点
    }

# 图编译后运行
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tool", tool_node)
graph.add_conditional_edges("agent", lambda s: s["next_node"])
app = graph.compile()
result = app.invoke({"messages": [user_input]})
```

### OpenClaw：分布式上下文

```typescript
// OpenClaw 的状态分布在多个层级

// 1. Session 级别状态（持久化）
interface SessionEntry {
  sessionId: string;
  sessionKey: string;     // "agent:main:main"
  messages: AgentMessage[];  // 对话历史
  acp?: SessionAcpMeta;   // ACP 控制平面元数据
}

// 2. Runtime 级别状态（运行时缓存）
interface CachedRuntimeState {
  runtime: AcpRuntime;
  handle: AcpRuntimeHandle;
  backend: string;
  agent: string;
  mode: string;
}

// 3. Turn 级别状态（单次交互）
interface ActiveTurnState {
  runtime: AcpRuntime;
  handle: AcpRuntimeHandle;
  abortController: AbortController;
}

// 4. 订阅级别状态（流式处理）
interface EmbeddedPiSubscribeState {
  assistantTexts: string[];      // 累积的助手消息
  toolMetas: ToolMeta[];         // 工具调用元数据
  blockState: {                  // 块级解析状态
    thinking: boolean;
    final: boolean;
    inlineCode: InlineCodeState;
  };
  // ... 更多运行时状态
}

// 状态不是集中返回，而是通过事件流传递
function subscribeEmbeddedPiSession(params) {
  const state: EmbeddedPiSubscribeState = { ... };
  
  // 订阅事件流，边接收边处理
  const sessionUnsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );
}
```

---

## 控制流对比

### LangGraph：显式控制流

```python
# LangGraph 的控制流是显式编程的

# 1. 条件边决定下一跳
def router(state: AgentState) -> Literal["tool_node", "end"]:
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tool_node"  # 有工具调用，去工具节点
    return "end"  # 没有工具调用，结束

graph.add_conditional_edges("agent", router)

# 2. 循环通过显式的边实现
graph.add_edge("tool_node", "agent")  # 工具执行完返回 agent

# 3. 并行执行通过 Send 实现
def fan_out(state: AgentState):
    return [Send("tool_node", {"tool": t}) for t in state["tool_calls"]]

graph.add_conditional_edges("agent", fan_out)
```

### OpenClaw：隐式控制流

```typescript
// OpenClaw 的控制流是隐式的，由 LLM + 运行时共同决定

// 1. 没有显式路由函数
// LLM 通过 function calling "决定"下一步
// 系统提示词告诉 LLM 可用工具，LLM 输出 tool_calls

// 2. 运行循环处理隐式流转
async function runEmbeddedAttempt(params) {
  // 调用 LLM
  const response = await llm.invoke(payload);
  
  // 解析响应
  if (response.tool_calls && response.tool_calls.length > 0) {
    // LLM 决定调用工具 -> "隐式"转移到工具执行
    const results = await executeTools(response.tool_calls);
    
    // 工具结果加入历史
    messages.push(...results);
    
    // 继续循环（隐式返回 agent "节点"）
    return { continue: true };
  }
  
  // LLM 决定不调用工具 -> 结束
  return { continue: false, final_response: response.content };
}

// 3. 外层循环管理生命周期
while (runLoopIterations < MAX_RUN_LOOP_ITERATIONS) {
  const attempt = await runEmbeddedAttempt({ ... });
  
  if (!attempt.continue) {
    // 自然结束
    return attempt.final_response;
  }
  
  // 继续下一轮（隐式循环）
  runLoopIterations++;
}

// 4. 子代理通过工具调用创建（不是图节点）
// spawnSubagentTool 是一个普通工具，内部启动新会话
async function spawnSubagentTool(params) {
  const childSessionKey = generateSubagentSessionKey(parentKey, depth + 1);
  return runEmbeddedPiAgent({
    sessionKey: childSessionKey,
    prompt: params.prompt,
    spawnedBy: parentKey,
  });
}
```

---

## 多 Agent 协作对比

### LangGraph：子图 + Supervisor

```python
# LangGraph 的多 Agent 通过子图实现

# 1. 每个 Agent 是一个子图
researcher_graph = create_researcher_agent()
writer_graph = create_writer_agent()

# 2. Supervisor 节点调度
class SupervisorState(TypedDict):
    messages: list
    next: Literal["researcher", "writer", "FINISH"]

def supervisor_node(state: SupervisorState):
    # LLM 决定哪个 Agent 执行
    decision = llm.invoke([system_prompt, *state["messages"]])
    return {"next": decision}

# 3. 构建多 Agent 图
graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor_node)
graph.add_node("researcher", researcher_graph)
graph.add_node("writer", writer_graph)

# 条件边根据 supervisor 决定调度
graph.add_conditional_edges("supervisor", lambda s: s["next"])
```

### OpenClaw：会话树 + 生命周期管理

```typescript
// OpenClaw 的多 Agent 是会话树结构

// 1. 子代理注册表管理生命周期
const subagentRuns = new Map<string, SubagentRunRecord>();

interface SubagentRunRecord {
  runId: string;
  requesterSessionKey: string;   // 父会话
  childSessionKey: string;       // 子会话
  task: string;                  // 任务描述
  depth: number;                 // 嵌套深度
  startedAt: number;
  endedAt?: number;
  outcome?: SubagentRunOutcome;
  cleanup: "delete" | "keep";
}

// 2. sessions-spawn 工具创建子代理
async function sessionsSpawnTool(params) {
  // 检查深度限制
  const currentDepth = getSubagentDepthFromSessionStore(parentSessionKey);
  if (currentDepth >= MAX_SUBAGENT_DEPTH) {
    throw new Error(`Maximum subagent depth exceeded`);
  }
  
  // 创建子会话
  const childSessionKey = generateSubagentSessionKey(parentKey, currentDepth + 1);
  
  // 注册到子代理注册表
  const runId = registerSubagentRun({
    requesterSessionKey: parentKey,
    childSessionKey,
    task: params.prompt,
    depth: currentDepth + 1,
  });
  
  // 启动子代理（异步）
  runEmbeddedPiAgent({
    sessionKey: childSessionKey,
    prompt: params.prompt,
    spawnedBy: parentKey,
  }).then(result => {
    // 完成通知
    completeSubagentRun({ runId, outcome: result });
  });
  
  return { runId, childSessionKey };
}

// 3. 生命周期事件系统
function ensureListener() {
  listenerStop = onAgentEvent((evt) => {
    if (evt.stream === "lifecycle" && evt.data.phase === "end") {
      // 子代理结束，通知父代理
      const entry = subagentRuns.get(evt.runId);
      if (entry) {
        announceSubagentCompletion(entry);
      }
    }
  });
}

// 4. 没有显式 Supervisor
// 父代理通过 "sessions_list" 工具查看子代理状态
// 父代理通过 "sessions_send" 工具给子代理发消息
// 子代理通过生命周期事件向父代理报告
```

---

## 代码示例对比

### 场景：实现一个能调用工具的 Agent

#### LangGraph 实现

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

# 1. 定义状态
class State(TypedDict):
    messages: Annotated[list, add_messages]

# 2. 定义工具
@tool
def search(query: str) -> str:
    """搜索信息"""
    return f"搜索结果: {query}"

@tool
def calculate(expression: str) -> str:
    """计算表达式"""
    return str(eval(expression))

tools = [search, calculate]

# 3. 定义节点
def agent_node(state: State):
    """Agent 节点：调用 LLM"""
    model = ChatOpenAI().bind_tools(tools)
    response = model.invoke(state["messages"])
    return {"messages": [response]}

def router(state: State) -> Literal["tools", "__end__"]:
    """路由函数：决定下一跳"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return "__end__"

# 4. 构建图
graph = StateGraph(State)
graph.add_node("agent", agent_node)
graph.add_node("tools", ToolNode(tools))

graph.set_entry_point("agent")
graph.add_conditional_edges("agent", router)
graph.add_edge("tools", "agent")

# 5. 编译运行
app = graph.compile()
result = app.invoke({"messages": [("human", "搜索 OpenAI 的最新新闻并计算 2+2")]})
```

#### OpenClaw 实现

```typescript
// OpenClaw 的实现方式完全不同

// 1. 工具定义（与 LangGraph 类似）
const searchTool = createTool({
  name: "web_search",
  description: "搜索信息",
  parameters: z.object({ query: z.string() }),
  execute: async ({ query }) => {
    return `搜索结果: ${query}`;
  },
});

const calculateTool = createTool({
  name: "calculate",
  description: "计算表达式",
  parameters: z.object({ expression: z.string() }),
  execute: async ({ expression }) => {
    return String(eval(expression));
  },
});

// 2. 没有显式图定义！
// 工具通过系统提示词告诉 LLM

// 3. 运行函数（没有节点概念）
async function runAgent(prompt: string) {
  // 构建工具列表
  const tools = [searchTool, calculateTool];
  
  // Pi 运行时处理循环
  const result = await runEmbeddedPiAgent({
    sessionKey: "agent:main:test",
    prompt,
    tools,
    // 所有控制流是隐式的：
    // - LLM 决定调用哪个工具
    // - 运行时处理工具执行
    // - 循环继续直到 LLM 不再调用工具
  });
  
  return result;
}

// 4. 运行
const result = await runAgent("搜索 OpenAI 的最新新闻并计算 2+2");

// 如果要实现多 Agent（类似 LangGraph 的 supervisor）
// OpenClaw 使用 sessions-spawn 工具
async function runMultiAgent(prompt: string) {
  // 父代理启动子代理处理子任务
  const spawnTool = createSessionsSpawnTool({
    onSpawn: async (params) => {
      // 启动子代理会话
      return runEmbeddedPiAgent({
        sessionKey: params.childSessionKey,
        prompt: params.prompt,
        spawnedBy: params.parentSessionKey,
      });
    },
  });
  
  return runEmbeddedPiAgent({
    sessionKey: "agent:supervisor:main",
    prompt,
    tools: [searchTool, calculateTool, spawnTool],
  });
}
```

---

## 设计哲学差异

| 维度 | LangGraph | OpenClaw |
|------|-----------|----------|
| **抽象层级** | 显式图结构（节点+边） | 隐式反应循环（ReAct） |
| **控制流** | 开发者编码控制 | LLM 自主决策 + 运行时管理 |
| **状态管理** | 集中式 TypedDict | 分布式（Session + Runtime + Turn） |
| **多 Agent** | 子图 + Supervisor 显式调度 | 会话树 + 生命周期事件隐式协调 |
| **流式处理** | 需要额外配置 | 原生支持，事件驱动 |
| **工具调用** | 通过边路由到 ToolNode | 运行时拦截 tool_calls 并执行 |
| **可观测性** | 检查图结构即可理解流程 | 需要跟踪消息历史和事件流 |
| **灵活性** | 结构化，适合复杂工作流 | 动态，适合开放式对话 |

### LangGraph 的优势场景

1. **复杂工作流**：需要严格控制的业务流程（审批、多步骤 ETL）
2. **多 Agent 协作**：明确的分工和调度策略
3. **可解释性要求**：需要可视化和调试完整执行路径
4. **人机协作**：需要明确的人工介入点

### OpenClaw 的优势场景

1. **开放式对话**：Agent 自主决定如何解决问题
2. **工具丰富环境**：大量工具，动态选择
3. **长会话**：需要上下文压缩和故障转移
4. **多渠道部署**：同一 Agent 同时服务多个渠道
5. **子代理动态创建**：根据任务动态分解

---

## 适用场景建议

### 选择 LangGraph 如果：

- 你需要 **可视化** 和 **精确控制** Agent 的执行流程
- 业务流程是 **结构化的**、有明确步骤的
- 需要 **人机协作**，在特定节点等待人工确认
- 团队成员更熟悉 **传统编程** 思维

### 选择 OpenClaw 如果：

- 你希望 Agent **自主规划** 解决方案
- 工具数量 **很多**，组合方式 **动态变化**
- 需要 **长会话** 和 **高可用**（故障转移、上下文压缩）
- 需要 **多渠道** 部署（Telegram、Discord、Slack 等）
- 需要 **子代理动态创建** 处理子任务

---

## 总结

LangGraph 和 OpenClaw 代表了两种不同的 Agent 开发范式：

| | LangGraph | OpenClaw |
|---|-----------|----------|
| **范式** | 显式编程（Explicit Programming） | 隐式反应（Implicit ReAct） |
| **类比** | 工作流引擎（如 Airflow） | 对话运行时（如操作系统进程） |
| **心智模型** | "我编排 Agent 的行为" | "我为 Agent 提供能力，它自主决策" |

对于 LangGraph 用户理解 OpenClaw 的关键转换：

1. **忘掉图的节点和边** → 关注消息历史和工具定义
2. **忘掉显式路由** → 信任 LLM 的 function calling
3. **忘掉集中式 State** → 理解 Session + Event Stream 的分布式状态
4. **忘掉 Supervisor 节点** → 使用 sessions-spawn 工具创建子代理

两者并非互斥，LangGraph 适合**工作流编排**，OpenClaw 适合**对话运行时**。在复杂系统中，可以结合使用：LangGraph 处理业务流程，OpenClaw 处理具体任务的对话执行。

---

*文档生成时间：2026-03-10*
*基于 OpenClaw 源码和 LangGraph 文档对比分析*
