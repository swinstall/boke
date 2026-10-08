---
title: "显式状态图 vs 事件流服务层：LangGraph 与 AgentScope 的取舍边界"
date: 2026-10-08T19:13:43+08:00
tags: ["langgraph", "agentscope", "multi-agent", "orchestration", "state"]
author: "swinstall"
---

## 一、先问一个让架构师睡不着的问题

你的多 agent 系统上了生产。某天凌晨，一个 agent 在调用写库工具前被权限系统拦下，需要人工确认——但人已经睡了。两小时后服务重启，它能从「停泊点」继续吗？这个「停泊点」由谁记录？框架自己兜底，还是要你手写 Redis 读写？答案正好划出两个框架的分野。

一句话结论：LangGraph 把编排显式建模成一张有状态的状态图（state graph），持久化与暂停是图运行时的一等公民；AgentScope 2.0 把编排收敛成无状态 agent 的事件流（Msg/Event）+ 一层 FastAPI 多租户服务，持久化与暂停由服务层与权限系统驱动。前者给你可控与可审计，代价是概念多、要自己管 thread/checkpoint；后者给你开箱即用的服务，代价是 2.x 破坏性演进频繁（4 个月发 10 个 2.x 版本）。

本文对比依据：langgraph 1.2.14、langchain 1.4.3、agentscope 2.0.9（2026-10-08 PyPI 实测，来源 https://pypi.org/pypi/langgraph/json 、https://pypi.org/pypi/agentscope/json）。

## 二、抽象模型：显式状态图 vs message passing

LangGraph 的运行时内核是 message passing + Pregel/BSP 超级步（super-step）。官方定义三要素：State 是「当前应用快照」、Node 是「编码 agent 逻辑的函数」、Edge 是「决定下一步执行谁的函数」，原话是 *nodes do the work, edges tell what to do next*（https://docs.langchain.com/oss/python/langgraph/graph-api）。State 由 schema（TypedDict/dataclass/Pydantic）+ reducer 共同定义；节点在 `inactive` 状态起步，收到入边（channel）消息才变 `active`，超级步结束时无在途消息的节点投票 `halt`，全部 inactive 才终止（https://docs.langchain.com/oss/python/langgraph/graph-api 、https://docs.langchain.com/oss/python/langgraph/pregel）。控制流是反转的——图决定「下一步谁跑」，不是 agent 自己决定。

AgentScope 2.0 的官方定义是「a **stateless** reasoning-acting loop engine」，把模型、工具、权限、HITL、context、middleware、事件系统统一进一个 `Agent` 接口（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/agent/overview）。它没有中心编排器：agent 之间靠 `observe()` 向对方 context 注入 `Msg`（不触发推理），Agent Team 场景下走 Redis 消息总线 + wakeup 唤醒目标 session（https://docs.agentscope.io/en/versions/2.0.9/deploy/agent-team）。关键区分是 Message 与 Event 两套粒度：`Msg` 是跨 agent 通信与持久化的完整一轮，Event 是流式交互与 HITL 的增量更新，一次 `reply` 产生的整串事件恰好聚合成一条 assistant `Msg`（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/message-and-event）。

一句话对照：LangGraph 用 node/edge/channel 拼图，AgentScope 用 agent-loop + Msg/Event 拼服务。

## 三、关键能力逐项对比

**状态与持久化**。LangGraph 两套互补机制：checkpointer 把 thread 的图状态在每个超级步边界落成 checkpoint（短期/线程内记忆，支撑断点续跑、HITL、time travel），store 持久化图状态之外的应用数据（长期/跨线程记忆）。`thread_id` 是主键，必须写进 `config={"configurable": {"thread_id": ...}}`；节点级 pending writes 也持久化，同超级步内别的节点失败时，已成功节点恢复后不重跑。后端 `InMemorySaver` 进程重启即丢，`SqliteSaver` 供开发，`PostgresSaver` 供生产（https://docs.langchain.com/oss/python/langgraph/persistence 、https://docs.langchain.com/oss/python/langgraph/checkpointers）。AgentScope 的 `AgentState` 可整体序列化 JSON「在一个进程暂停、在另一进程恢复」，内置 `RedisStorage`，key 按 `(user_id, agent_id, session_id)` 分层，走 `get_session`/`update_session_state`；首轮必须先 `upsert_session` 建记录，否则 `update_session_state` 抛 `KeyError`（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/agent/run-agent）。差别在：LangGraph 的持久化是图执行的自动产物，AgentScope 的持久化要调用方自己在 reply 前后读写 storage——`AgentState` 不自动落盘。

**人机协同**。LangGraph 的 `interrupt()` 放在任意 node 内、动态可条件触发，触发时用 persistence 层保存状态并无限期等待，恢复只需 `Command(resume=...)`；但恢复时该 node 从头重跑（interrupt 之前的代码再执行一遍），副作用必须自己幂等（https://docs.langchain.com/oss/python/langgraph/interrupts）。AgentScope 的暂停是策略驱动而非代码显式放置：工具被权限系统判 ASK → agent 停泊并 emit `RequireUserConfirmEvent`，人回传 `UserConfirmResultEvent(reply_id=...)` 续跑，`reply_id` 充当续跑游标；接受 `rules` 会写进权限引擎，后续同类调用自动放行（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/agent/human-in-the-loop）。这是设计哲学差异，不是能力强弱。

**并行与多智能体**。LangGraph 同超级步内被调度节点并行执行（https://docs.langchain.com/oss/python/langgraph/pregel）；多 agent pattern 的官方文档挂在 LangChain 侧（https://docs.langchain.com/oss/python/langchain/multi-agent），且 `langgraph-supervisor` 已停止维护、要求迁到 subagents pattern（https://docs.langchain.com/oss/python/migrate/langgraph-supervisor）。AgentScope 的 `TeamPipeline`（2.0.9 新增）是 leader + member：leader 派活，member 在自己 context 里跑、只回传最终结果（不污染 leader 上下文），同一轮并发、member 间不直接对话、支持逐 member 的 HITL；官方标注 pipeline 模块 experimental（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/pipeline/team）。

**可观测性**。LangGraph 官方指定 LangSmith，`LANGSMITH_TRACING=true` 即开，并有专门的错误码文档页（https://docs.langchain.com/oss/python/langgraph/observability 、https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT ）。AgentScope 2.0 的 tracing 在中间件一章以 `TracingMiddleware`（OpenTelemetry）小节形式提供，**没有独立可观测性章节**（1.x 才有 task_tracing，接 Langfuse/Phoenix/ARMS 的具体做法只能引 1.x 文档）——官方 2.0 文档未说明更多（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/middleware）。

**部署与语言生态**。LangGraph 需 Python ≥3.10，另有 JS/TS 版；多租户托管走 LangSmith Cloud，另有 hybrid / standalone server / self-hosted with control plane 三种 LangSmith 部署选项，开源侧不自带多租户（https://docs.langchain.com/oss/python/langgraph/deploy）。AgentScope 需 Python ≥3.11，`agentscope.app` 直接给 FastAPI 服务 + 预置 Web UI，多租户/多会话/分布式是 first-class，还有独立但 API 不互通的 TypeScript/Java 实现（https://docs.agentscope.io/en/versions/2.0.9/deploy/agent-service 、https://docs.agentscope.io/en/versions/2.0.9/others/faq）。

**学习曲线**。LangGraph 自述「very low-level」，概念集中在 state/reducer/super-step/channel/checkpointer/interrupt/Command（https://docs.langchain.com/oss/python/langgraph/overview）。AgentScope 上手面窄——配置全走 `Agent(...)` 构造参数，5 行可跑，但 2.0 是 breaking release、文档按版本分目录强制你写出版本号（https://docs.agentscope.io/en/versions/2.0.9/building-blocks/agent/configure-agent）。

## 四、同一任务的两段写法

同一个「两步流程 + 人工介入 + 状态持久化」任务，两边各一段最小代码（均按官方示例整理，标注依赖版本，未在本机执行）。

**LangGraph**（langgraph==1.2.14，Python ≥3.10）：

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command

class State(TypedDict):
    topic: str
    draft: str
    approved: bool

def draft_node(state: State) -> State:
    return {"draft": f"draft about {state['topic']}"}

def approval_node(state: State) -> State:
    approved = interrupt({"question": "approve?", "draft": state["draft"]})
    return {"approved": bool(approved)}

def publish_node(state: State) -> State:
    return {"draft": state["draft"] + " [published]"}

builder = StateGraph(State)
builder.add_node("draft", draft_node)
builder.add_node("approval", approval_node)
builder.add_node("publish", publish_node)
builder.add_edge(START, "draft")
builder.add_edge("draft", "approval")
builder.add_edge("approval", "publish")
builder.add_edge("publish", END)

graph = builder.compile(checkpointer=InMemorySaver())  # 生产换 PostgresSaver
config = {"configurable": {"thread_id": "thread-1"}}
first = graph.invoke({"topic": "ice cream"}, config=config)  # 挂在 approval，状态已落 checkpoint
second = graph.invoke(Command(resume=True), config=config)    # 恢复
```

这段为什么重要：它把「持久化」和「暂停」压缩成两个正交钩子——`checkpointer` 与 `interrupt()`，恢复只有一行 `Command(resume=...)`。代价也暴露无遗：interrupt 恢复时节点从头重跑，节点副作用必须幂等。

**AgentScope**（agentscope==2.0.9，Python ≥3.11，需 DASHSCOPE_API_KEY）：

```python
import asyncio, os
from agentscope.agent import Agent
from agentscope.credential import DashScopeCredential
from agentscope.event import ConfirmResult, RequireUserConfirmEvent, UserConfirmResultEvent
from agentscope.message import UserMsg
from agentscope.model import DashScopeChatModel
from agentscope.state import AgentState
from agentscope.app.storage import RedisStorage

USER_ID, AGENT_ID, SESSION_ID = "u1", "a1", "s1"

async def main():
    async with RedisStorage(host="localhost", port=6379) as storage:
        record = await storage.get_session(USER_ID, AGENT_ID, SESSION_ID)
        state = record.state if record else AgentState()
        agent = Agent(name="my_agent", system_prompt="You are helpful.",
                      model=DashScopeChatModel(
                          credential=DashScopeCredential(api_key=os.environ["DASHSCOPE_API_KEY"]),
                          model="qwen-max"),
                      state=state)
        async for event in agent.reply_stream(UserMsg(name="user", content="Run the two-step task.")):
            if isinstance(event, RequireUserConfirmEvent):
                results = [ConfirmResult(confirmed=True, tool_call=tc, rules=tc.suggested_rules)
                           for tc in event.tool_calls]
                result = await agent.reply(UserConfirmResultEvent(
                    reply_id=event.reply_id, confirm_results=results))
                print(result.get_text_content())
        await storage.update_session_state(USER_ID, AGENT_ID, SESSION_ID, state=agent.state)

asyncio.run(main())
```

这段为什么重要：人工介入不是外挂 API，而是 agent loop 的一个出口——`reply_stream` 把 `RequireUserConfirmEvent` 当一等事件抛出来，恢复只是再 `reply()` 一个事件对象。代价是：恢复路径与权限系统强耦合，且持久化要调用方自己在前后读写 `RedisStorage`。

## 五、选型建议与适用边界

选 LangGraph，当你需要把确定性逻辑与 agentic 步骤混在一张可审计、可 time travel 的图里，或者要对图执行做精细控制（动态路由、节点级容错、checkpoint 语义）。接受它的概念负担，自己管 thread/checkpointer。

选 AgentScope 2.x，当你需要在开源栈内直接拿到多租户服务、IM 渠道、沙箱与团队协作，且愿意锁死版本、接受 breaking 升级节奏（2.0 升级包本身不会迁移既有代码，见 https://docs.agentscope.io/en/versions/2.0.9/others/faq ）。别拿 pipeline 这类 experimental 模块当主论据。

不选 LangGraph 当「多租户平台」：开源侧不自带，托管侧由 LangSmith Agent Server 承担，持久化由服务端代管、服务端内部实现细节不在开源文档中说明。不选 AgentScope 当「深控编排」：它刻意把编排收敛进事件流，你要的 super-step 级确定性它没有暴露。

## 六、小结

两个框架的分歧不在「能不能做持久化或 HITL」——两边都能落地——而在这些能力是**运行时内核的显式产物，还是服务层/权限系统的约定产物**。LangGraph 用显式性换可控与可审计；AgentScope 用约定性换开箱即用的服务面。别信「两者都很好」：你的选择取决于你更愿意在哪个地方付成本——在概念里，还是在升级节奏里。
