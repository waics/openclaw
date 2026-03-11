# Pi Runtime 运行循环深度解析

> 本文档深入剖析 OpenClaw Pi Runtime 的核心运行循环，重点关注：
> 1. While 循环的退出机制
> 2. 子代理的嵌套实现
> 3. 政务机器人的设计借鉴

---

## 一、核心运行循环架构

### 1.1 双层循环结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         外层循环：故障恢复与重试                              │
│                    (run.ts: runEmbeddedPiAgent)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   while (true) {                                                            │
│     // 1. 检查最大迭代次数                                                    │
│     if (runLoopIterations >= MAX_RUN_LOOP_ITERATIONS) {                     │
│       return { error: "retry_limit" };                                     │
│     }                                                                       │
│                                                                             │
│     // 2. 执行单次尝试（内层循环在 attempt 中）                               │
│     const attempt = await runEmbeddedAttempt({ ... });                     │
│                                                                             │
│     // 3. 处理各种错误情况                                                    │
│     if (contextOverflowError) {                                            │
│       // 尝试压缩上下文                                                      │
│       if (canCompact) { compact(); continue; }                             │
│       return { error: "context_overflow" };                                │
│     }                                                                       │
│                                                                             │
│     if (promptError) {                                                     │
│       // 尝试故障转移到其他 auth profile                                    │
│       if (canFailover) { rotateAuthProfile(); continue; }                  │
│       throw promptError;                                                   │
│     }                                                                       │
│                                                                             │
│     if (failoverFailure) {                                                 │
│       // 尝试故障转移到其他模型                                              │
│       if (fallbackConfigured) {                                            │
│         throw new FailoverError(...);  // 外层捕获并切换模型                │
│       }                                                                    │
│     }                                                                       │
│                                                                             │
│     // 4. 成功完成 → 退出循环                                                 │
│     return { payloads, meta };                                             │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         内层循环：LLM 交互与工具调用                           │
│                    (attempt.ts: runEmbeddedAttempt)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   // 这不是显式循环，而是由 pi-coding-agent SDK 管理的隐式循环                │
│   // 关键：SessionManager 处理多轮 tool_calls                                 │
│                                                                             │
│   const sessionManager = new SessionManager(...);                          │
│   const session = sessionManager.createSession(...);                       │
│                                                                             │
│   // 订阅事件流（包含多轮工具调用的完整交互）                                   │
│   const unsubscribe = session.subscribe(eventHandler);                     │
│                                                                             │
│   // 事件类型：                                                              │
│   // - text_delta: 流式文本输出                                              │
│   // - toolUse: LLM 请求调用工具                                             │
│   // - toolResult: 工具执行结果                                              │
│   // - message_end: 单条消息结束                                             │
│   // - session_end: 整个会话结束（无更多工具调用）                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 源码位置

| 文件 | 职责 |
|------|------|
| `src/agents/pi-embedded-runner/run.ts:807` | 外层 while 循环起点 |
| `src/agents/pi-embedded-runner/run.ts:808-839` | 最大迭代次数检查 |
| `src/agents/pi-embedded-runner/run.ts:849` | 调用内层 attempt |
| `src/agents/pi-embedded-runner/run/attempt.ts:746` | 内层 attempt 实现 |
| `src/agents/pi-embedded-subscribe.ts:34` | 事件订阅处理 |

---

## 二、循环退出机制详解

### 2.1 正常退出条件

```typescript
// run.ts:1425-1537
// 情况 1：成功完成（最常用路径）

const usage = toNormalizedUsage(usageAccumulator);
const agentMeta: EmbeddedPiAgentMeta = {
  sessionId: sessionIdUsed,
  provider: lastAssistant?.provider ?? provider,
  model: lastAssistant?.model ?? model.id,
  usage,
  // ...
};

// 构建最终响应 payload
const payloads = buildEmbeddedRunPayloads({
  assistantTexts: attempt.assistantTexts,
  toolMetas: attempt.toolMetas,
  lastAssistant: attempt.lastAssistant,
  // ...
});

// 返回成功结果 → 退出 while 循环
return {
  payloads: payloads.length ? payloads : undefined,
  meta: {
    durationMs: Date.now() - started,
    agentMeta,
    aborted,
    systemPromptReport: attempt.systemPromptReport,
  },
};
```

**正常退出的本质**：
- 内层 attempt 完成（LLM 不再输出 tool_calls）
- 所有工具执行完毕
- 返回最终响应

### 2.2 异常退出条件

```typescript
// 情况 2：达到最大迭代次数（run.ts:808-839）
if (runLoopIterations >= MAX_RUN_LOOP_ITERATIONS) {
  return {
    payloads: [{
      text: "Request failed after repeated internal retries...",
      isError: true,
    }],
    meta: { error: { kind: "retry_limit", message } },
  };
}

// 情况 3：上下文溢出（run.ts:986-1157）
if (contextOverflowError) {
  // 尝试 3 次压缩
  if (overflowCompactionAttempts < MAX_OVERFLOW_COMPACTION_ATTEMPTS) {
    await compactEmbeddedPiSession({ ... });
    continue;  // 重试
  }
  // 压缩失败 → 退出
  return {
    payloads: [{
      text: "Context overflow: prompt too large...",
      isError: true,
    }],
    meta: { error: { kind: "context_overflow", message: errorText } },
  };
}

// 情况 4：超时（run.ts:1467-1490）
if (timedOut && !timedOutDuringCompaction && payloads.length === 0) {
  return {
    payloads: [{
      text: "Request timed out before a response was generated...",
      isError: true,
    }],
    meta: { aborted, ... },
  };
}

// 情况 5：用户中断（通过 AbortSignal）
// 在 attempt 中检查 signal.aborted，抛出 AbortError
```

### 2.3 故障转移机制

```typescript
// 情况 6：Auth Profile 故障转移（run.ts:1246-1254）
if (promptFailoverFailure && await advanceAuthProfile()) {
  continue;  // 使用新的 auth profile 重试
}

// 情况 7：模型故障转移（run.ts:1414-1420）
if (fallbackConfigured && promptFailoverFailure) {
  throw new FailoverError(errorText, {
    reason: promptFailoverReason ?? "unknown",
    provider,
    model: modelId,
  });
  // 外层捕获 FailoverError，切换到备用模型重新调用 runEmbeddedPiAgent
}
```

---

## 三、子代理嵌套机制详解

### 3.1 子代理创建流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         父代理调用 sessions_spawn                            │
│                    (src/agents/tools/sessions-spawn-tool.ts)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   execute: async (_toolCallId, args) => {                                  │
│     // 1. 参数解析                                                           │
│     const task = args.task;  // 子代理任务描述                               │
│     const runtime = args.runtime ?? "subagent";                            │
│     const mode = args.mode ?? "run";  // "run" | "session"                  │
│                                                                             │
│     // 2. 深度检查                                                           │
│     const callerDepth = getSubagentDepthFromSessionStore(parentKey);       │
│     if (callerDepth >= maxSpawnDepth) {                                    │
│       return { status: "forbidden", error: "max depth exceeded" };         │
│     }                                                                       │
│                                                                             │
│     // 3. 子数量检查                                                          │
│     const activeChildren = countActiveRunsForSession(parentKey);           │
│     if (activeChildren >= maxChildren) {                                   │
│       return { status: "forbidden", error: "max children exceeded" };      │
│     }                                                                       │
│                                                                             │
│     // 4. 创建子会话密钥                                                       │
│     const childSessionKey = `agent:${targetAgentId}:subagent:${uuid}`;     │
│                                                                             │
│     // 5. 注册到子代理注册表                                                    │
│     const runId = registerSubagentRun({                                    │
│       requesterSessionKey: parentKey,                                      │
│       childSessionKey,                                                     │
│       task,                                                                │
│       depth: callerDepth + 1,                                              │
│     });                                                                     │
│                                                                             │
│     // 6. 启动子代理（异步，不阻塞父代理）                                        │
│     spawnSubagentDirect({ ... });  // 返回 runId 和 childSessionKey         │
│                                                                             │
│     return { status: "accepted", runId, childSessionKey };                 │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 子代理生命周期管理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      子代理注册表 (subagent-registry.ts)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   // 内存中的子代理运行记录                                                   │
│   const subagentRuns = new Map<string, SubagentRunRecord>();               │
│                                                                             │
│   interface SubagentRunRecord {                                            │
│     runId: string;                                                         │
│     requesterSessionKey: string;    // 父会话                                │
│     childSessionKey: string;        // 子会话                                │
│     task: string;                   // 任务描述                              │
│     depth: number;                  // 嵌套深度                              │
│     startedAt: number;                                                     │
│     endedAt?: number;                                                      │
│     outcome?: SubagentRunOutcome;   // 完成结果                              │
│     cleanup: "delete" | "keep";                                            │
│     frozenResultText?: string | null;  // 冻结的结果                         │
│   }                                                                         │
│                                                                             │
│   // 生命周期事件监听                                                        │
│   function ensureListener() {                                              │
│     listenerStop = onAgentEvent((evt) => {                                 │
│       if (evt.stream === "lifecycle" && evt.data.phase === "end") {        │
│         const entry = subagentRuns.get(evt.runId);                         │
│         if (entry) {                                                       │
│           // 子代理完成，通知父代理                                            │
│           completeSubagentRun({                                            │
│             runId: evt.runId,                                              │
│             outcome: { status: "ok" },                                     │
│             reason: SUBAGENT_ENDED_REASON_COMPLETE,                        │
│           });                                                               │
│         }                                                                   │
│       }                                                                     │
│     });                                                                     │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 父子代理通信机制

```typescript
// 方式 1：父代理通过工具查看子代理状态
// src/agents/tools/sessions-list-tool.ts
async function sessionsListTool() {
  const runs = listRunsForRequester(parentSessionKey);
  return runs.map(r => ({
    sessionKey: r.childSessionKey,
    status: r.outcome ? 'completed' : 'running',
    outcome: r.outcome,
  }));
}

// 方式 2：父代理通过工具给子代理发消息
// src/agents/tools/sessions-send-tool.ts
async function sessionsSendTool(params) {
  const { targetSessionKey, message } = params;
  // 发送消息到子会话
  await sendMessageToSession(targetSessionKey, message);
}

// 方式 3：子代理完成后通知父代理（事件驱动）
// src/agents/subagent-announce.ts
async function runSubagentAnnounceFlow(params) {
  const { requesterSessionKey, childSessionKey, outcome } = params;
  
  // 构建完成消息
  const completionMessage = buildSubagentCompletionMessage({
    childSessionKey,
    task: params.task,
    result: outcome.status === 'ok' ? outcome.result : outcome.error,
  });
  
  // 发送给父代理（作为用户消息）
  await sendToParent(requesterSessionKey, completionMessage);
}
```

---

## 四、政务机器人设计借鉴

### 4.1 政务业务的特点

| 特点 | 技术挑战 | OpenClaw 解决方案 |
|------|----------|------------------|
| 多步骤表单填写 | 状态管理复杂 | Session 持久化 + 消息历史 |
| 材料审核 | 可能中断和恢复 | 上下文压缩 + 会话恢复 |
| 多部门协同 | 跨系统查询 | 子代理并行处理 |
| 业务规则复杂 | 需要精确控制 | System Prompt + 工具策略 |
| 高可用要求 | 不能失败 | 故障转移 + 重试机制 |

### 4.2 借鉴 OpenClaw 的 LangGraph 政务机器人设计

```python
# 方案：结合 LangGraph 的结构化和 OpenClaw 的灵活性

from typing import TypedDict, Annotated, Optional, Literal
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_core.messages import SystemMessage, AIMessage
import operator

class GovServiceState(TypedDict):
    # 用户基本信息
    user_id: str
    service_type: Optional[str]  # 业务类型（社保/公积金/户籍等）
    
    # 渐进式收集的信息
    collected_info: Annotated[dict, operator.or_]  # 已收集的字段
    missing_fields: list  # 还缺的字段
    
    # 业务状态
    stage: Literal["intake", "verification", "processing", "completed", "error"]
    
    # 子任务（借鉴 OpenClaw 的子代理）
    sub_tasks: list  # 并行子任务
    pending_sub_tasks: int    # 等待中的子任务数
    
    # 对话历史
    messages: Annotated[list, operator.add]
    
    # 容错相关
    retry_count: int
    last_error: Optional[str]


# ============================================================
# 1. 渐进式信息收集（借鉴 OpenClaw 的 ReAct 循环）
# ============================================================

async def information_collector(state: GovServiceState):
    """
    借鉴 OpenClaw 的 runEmbeddedAttempt：
    - LLM 自主决定需要询问哪些信息
    - 通过工具调用来查询外部系统验证
    - 循环直到信息收集完整
    """
    
    system_prompt = f"""
    你是政务办理助手，正在为用户办理：{state['service_type']}
    
    已收集信息：{state['collected_info']}
    还缺字段：{state['missing_fields']}
    
    你的任务是：
    1. 如果信息不完整，向用户询问最必要的一个或几个字段
    2. 如果用户提供了信息，使用 verify_info 工具验证
    3. 如果信息完整，调用 complete_intake 工具进入下一阶段
    
    注意：每次只询问最必要的信息，避免一次性询问太多。
    """
    
    # 调用 LLM（绑定工具）
    from langchain_openai import ChatOpenAI
    model = ChatOpenAI().bind_tools([
        verify_info,      # 验证信息
        complete_intake,  # 完成收集
        ask_clarification # 请求澄清
    ])
    
    response = await model.ainvoke([
        SystemMessage(content=system_prompt),
        *state['messages']
    ])
    
    # 处理工具调用（类似 OpenClaw 的工具执行）
    if response.tool_calls:
        return {
            "messages": [response],
            # 让 LangGraph 路由到 ToolNode
        }
    
    # 自然语言回复
    return {"messages": [response]}


# ============================================================
# 2. 子任务并行处理（借鉴 OpenClaw 的子代理）
# ============================================================

async def create_sub_tasks(state: GovServiceState):
    """
    借鉴 OpenClaw 的 sessions_spawn：
    将大任务分解为多个独立的子任务并行执行
    """
    import asyncio
    from uuid import uuid4
    
    if state['service_type'] == '社保转移':
        # 创建并行子任务（类似 spawnSubagentDirect）
        sub_tasks = [
            {
                "task_id": f"query_source_{uuid4()}",
                "task_type": "query",
                "status": "pending",
                "params": {"action": "query_source_social_security", "user_id": state['user_id']}
            },
            {
                "task_id": f"query_target_{uuid4()}",
                "task_type": "query", 
                "status": "pending",
                "params": {"action": "query_target_capacity", "region": state['collected_info'].get('target_region')}
            },
            {
                "task_id": f"verify_identity_{uuid4()}",
                "task_type": "verify",
                "status": "pending",
                "params": {"action": "verify_real_name", "info": state['collected_info']}
            }
        ]
        
        # 启动所有子任务（异步并行）
        for task in sub_tasks:
            asyncio.create_task(run_sub_task(task, state['user_id']))
        
        return {
            "sub_tasks": sub_tasks,
            "pending_sub_tasks": len(sub_tasks),
            "stage": "processing"
        }


async def run_sub_task(task: dict, user_id: str):
    """
    子任务执行器（类似 OpenClaw 的子代理运行）
    """
    try:
        # 更新状态为运行中
        task['status'] = 'running'
        
        # 执行任务
        result = await execute_sub_task(task)
        
        # 任务完成（类似 OpenClaw 的 completeSubagentRun）
        task['status'] = 'completed'
        task['result'] = result
        
    except Exception as e:
        # 任务失败
        task['status'] = 'failed'
        task['result'] = {'error': str(e)}


async def wait_for_sub_tasks(state: GovServiceState):
    """
    等待所有子任务完成（类似 OpenClaw 的子代理生命周期管理）
    """
    
    # 检查是否所有任务都完成
    completed = sum(1 for t in state['sub_tasks'] if t['status'] in ['completed', 'failed'])
    
    if completed < len(state['sub_tasks']):
        # 还有任务未完成，等待一段时间后重试
        # 借鉴 OpenClaw 的异步事件驱动，但 LangGraph 中需要轮询
        return {
            "pending_sub_tasks": len(state['sub_tasks']) - completed,
            # 使用 LangGraph 的延迟边
        }
    
    # 所有任务完成，汇总结果
    all_success = all(t['status'] == 'completed' for t in state['sub_tasks'])
    
    if all_success:
        return {
            "stage": "completed",
            "messages": [AIMessage(content="所有验证通过，业务办理成功！")]
        }
    else:
        # 有任务失败，决定重试或报错
        failed_tasks = [t for t in state['sub_tasks'] if t['status'] == 'failed']
        return {
            "stage": "error",
            "last_error": f"以下任务失败：{[t['task_id'] for t in failed_tasks]}"
        }


# ============================================================
# 3. 容错和重试机制（借鉴 OpenClaw 的故障转移）
# ============================================================

async def error_handler(state: GovServiceState):
    """
    借鉴 OpenClaw 的多层故障恢复
    """
    
    error = state['last_error']
    retry_count = state.get('retry_count', 0)
    
    # 第一层：简单重试（类似 auth profile 轮换）
    if retry_count < 3 and is_transient_error(error):
        return {
            "retry_count": retry_count + 1,
            "stage": state['stage'],  # 回到原阶段重试
            "messages": [AIMessage(content=f"系统临时错误，正在重试（{retry_count + 1}/3）...")]
        }
    
    # 第二层：降级处理（类似 OpenClaw 的 fallback model）
    if state['stage'] == 'verification':
        # 自动审核失败，转人工审核
        return {
            "stage": "manual_review",
            "messages": [AIMessage(content="自动审核遇到问题，已转人工审核，请等待工作人员联系。")]
        }
    
    # 第三层：优雅失败
    return {
        "stage": "error",
        "messages": [AIMessage(content=f"系统遇到错误：{error}。请稍后重试或联系客服。")]
    }


def is_transient_error(error: str) -> bool:
    """判断是否为临时性错误"""
    transient_keywords = ['timeout', 'rate limit', 'connection', 'temporary']
    return any(kw in error.lower() for kw in transient_keywords)


async def execute_sub_task(task: dict):
    """模拟子任务执行"""
    import random
    await asyncio.sleep(random.uniform(0.5, 2))
    return {"status": "success", "data": {}}


# 工具函数占位
def verify_info(**kwargs):
    pass

def complete_intake(**kwargs):
    pass

def ask_clarification(**kwargs):
    pass


# ============================================================
# 4. 构建完整的政务机器人图
# ============================================================

def build_gov_service_graph():
    graph = StateGraph(GovServiceState)
    
    # 节点
    graph.add_node("intake", information_collector)
    graph.add_node("tools", ToolNode([verify_info, complete_intake]))
    graph.add_node("create_sub_tasks", create_sub_tasks)
    graph.add_node("wait_sub_tasks", wait_for_sub_tasks)
    graph.add_node("finalize", lambda s: {"messages": [AIMessage(content="业务办理完成")]})
    graph.add_node("error_handler", error_handler)
    
    # 边
    graph.set_entry_point("intake")
    
    # intake -> tools -> intake 循环（信息收集阶段）
    def route_from_intake(state):
        last_msg = state['messages'][-1]
        if hasattr(last_msg, 'tool_calls') and last_msg.tool_calls:
            return "tools"
        elif state.get('stage') == 'processing':
            return "create_sub_tasks"
        return "intake"
    
    graph.add_conditional_edges("intake", route_from_intake, {
        "tools": "tools",
        "create_sub_tasks": "create_sub_tasks",
        "intake": "intake"
    })
    graph.add_edge("tools", "intake")
    
    # 子任务处理
    graph.add_edge("create_sub_tasks", "wait_sub_tasks")
    
    def route_from_wait(state):
        if state.get('pending_sub_tasks', 0) == 0:
            return "finalize"
        return "wait_sub_tasks"
    
    graph.add_conditional_edges("wait_sub_tasks", route_from_wait, {
        "finalize": "finalize",
        "wait_sub_tasks": "wait_sub_tasks"
    })
    
    return graph.compile()
```

### 4.3 关键借鉴点总结

| OpenClaw 机制 | 政务机器人应用 |
|--------------|---------------|
| **外层 while 循环** | LangGraph 的图循环 + 错误处理节点 |
| **内层隐式循环（ReAct）** | `intake` 节点的工具调用循环 |
| **子代理（sessions_spawn）** | `create_sub_tasks` + `wait_sub_tasks` 节点 |
| **深度限制** | `max_concurrent_tasks` 配置 |
| **生命周期事件** | `run_sub_task` 的异步回调 + 状态更新 |
| **故障转移** | `error_handler` 节点的多层恢复策略 |
| **上下文压缩** | `collected_info` 的摘要存储 |

---

## 五、核心代码片段索引

### 5.1 循环退出相关

| 功能 | 文件 | 行号 |
|------|------|------|
| 最大迭代检查 | `run.ts` | 808-839 |
| 上下文溢出处理 | `run.ts` | 986-1157 |
| 超时处理 | `run.ts` | 1467-1490 |
| 成功返回 | `run.ts` | 1508-1537 |
| Auth Profile 轮换 | `run.ts` | 633-658 |
| 模型故障转移 | `run.ts` | 1414-1420 |

### 5.2 子代理相关

| 功能 | 文件 | 行号 |
|------|------|------|
| sessions_spawn 工具 | `sessions-spawn-tool.ts` | 62-197 |
| 子代理直接启动 | `subagent-spawn.ts` | 237-400+ |
| 深度检查 | `subagent-spawn.ts` | 314-322 |
| 子数量检查 | `subagent-spawn.ts` | 324-331 |
| 注册子代理 | `subagent-registry.ts` | 64+ |
| 生命周期监听 | `subagent-registry.ts` | 752-807 |
| 完成通知 | `subagent-announce.ts` | 多个函数 |

---

## 六、总结

### OpenClaw 循环设计的精髓

1. **双层循环分离**：
   - 外层：故障恢复、资源管理、生命周期
   - 内层：LLM 交互、工具调用、流式处理

2. **隐式 vs 显式**：
   - 不控制 LLM "走哪条边"
   - 通过工具可用性和 System Prompt 引导行为

3. **子代理即工具**：
   - `sessions_spawn` 是一个普通工具
   - 异步执行，事件驱动通知
   - 深度和数量限制防止滥用

4. **失败是常态**：
   - 上下文溢出 → 压缩重试
   - API 失败 → 换账号/模型重试
   - 超时 → 优雅降级

### 政务机器人应用建议

1. **渐进式信息收集**：不要一次性问太多，像 OpenClaw 一样让 LLM 自主决定问什么
2. **子任务并行**：复杂业务分解为独立子任务（查询、验证、提交）
3. **容错设计**：每层都要有降级方案（自动→人工、实时→异步）
4. **状态持久化**：会话可能中断，设计可恢复的 State 结构

---

*文档生成时间：2026-03-10*
*基于 OpenClaw 源码深度分析*
