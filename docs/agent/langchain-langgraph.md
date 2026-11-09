# LangChain 与 LangGraph：用一个客服 Agent 看懂它们

## 先看一个具体问题

用户问：“我的订单 12345 到哪了？”

系统需要识别物流意图、查询订单、把真实结果交给模型组织回答。如果缺少订单号，就先追问；如果遇到复杂问题，就转人工。模型负责理解和表达，查询函数负责读取业务数据，应用负责校验和执行控制。LangChain 和 LangGraph 可以帮助开发者把这些部分接起来。

## LangChain：把模型和工具接成一个 Agent

先把“查询订单”写成 Python 函数，再告诉 Agent 这个函数叫什么、需要什么参数、能做什么。LangChain 的工具装饰器会把函数信息整理成 Agent 可使用的工具定义；Agent 执行器把模型、提示和工具组成一次工具调用循环。

~~~python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def lookup_order(order_id: str) -> str:
    """查询订单当前的物流状态。"""
    # 教学示例：实际项目中这里会查订单系统或调用物流 API
    return f"订单 {order_id} 已发货，正在配送中。"

agent = create_agent(
    model="openai:替换成你配置的模型",
    tools=[lookup_order],
    system_prompt="你是客服助手。涉及订单状态时，先查询工具，再根据结果回答。",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "我的订单 12345 到哪了？"}]
})

print(result["messages"][-1].content)
~~~

运行时发生的事情：

1. 用户消息交给模型。
2. 模型判断需要查询订单，返回一个结构化工具调用，例如参数 order_id=12345。
3. Agent 执行器调用 Python 函数。
4. 函数返回物流状态，执行器把结果放回对话。
5. 模型根据结果生成最终回答。

工具装饰器不会自动连接订单数据库。它主要把函数的名称、参数类型和说明暴露给 Agent；函数体里的业务逻辑仍由开发者实现。工具的权限和参数也需要应用校验。

### 把“工具箱”说得更具体

LangChain 不只是工具函数集合，它还提供了常用的连接接口和 Agent 执行器：

- Model 接口：用相近的方式接入不同模型服务。
- Tool：把 Python 函数、API 或查询能力描述成模型可以选择的操作。
- Prompt 和 Message：组织发给模型的指令和对话。
- Retriever 和 Document：连接知识检索与文档表示。
- Output parser 和结构化输出：把模型回复约束成程序需要的字段。
- Agent 执行器：把模型、提示、工具和中间件组合成可运行的 Agent。

“工具箱”是一个好记的比喻；更准确地说，LangChain 还提供了把这些组件组合起来的接口和执行抽象。

## LangGraph：把有分支的客服流程变成路线图

也可以把客服流程提前规定好：

~~~text
收到问题
  ↓
识别意图
  ├─ 物流问题且有订单号 → 查询订单 → 组织回答 → 结束
  ├─ 物流问题但没有订单号 → 询问订单号 → 等待用户回复
  └─ 其他复杂问题 → 转人工
~~~

LangGraph 把流程拆成几个直观的部分：

| 名称 | 客服例子里的内容 |
| --- | --- |
| State（状态） | 当前问题、识别出的意图、订单号、查询结果、回答草稿 |
| Node（节点） | 识别意图、查询订单、生成回答、转人工等一步一步的函数 |
| Edge（边） | 查询后去生成回答；缺少订单号则去追问；复杂问题则转人工 |

下面的代码展示状态、节点和条件分支。为了让流程容易阅读，意图识别和订单查询使用了示例逻辑；真实系统会把它们替换为模型调用和订单 API。

~~~python
from typing_extensions import TypedDict
from langgraph.graph import END, START, StateGraph

class SupportState(TypedDict):
    question: str
    order_id: str
    intent: str
    order_info: str
    answer: str

def classify(state: SupportState):
    question = state["question"]
    intent = "shipping" if ("物流" in question or "订单" in question) else "other"
    return {"intent": intent}

def choose_next(state: SupportState):
    if state["intent"] == "shipping" and state["order_id"]:
        return "lookup"
    if state["intent"] == "shipping":
        return "ask_order_id"
    return "handoff"

def lookup(state: SupportState):
    # 教学示例：实际系统在这里请求订单服务
    return {"order_info": f"订单 {state['order_id']} 正在配送中。"}

def answer(state: SupportState):
    return {"answer": state["order_info"]}

def ask_order_id(state: SupportState):
    return {"answer": "请告诉我订单号，我来帮你查询物流。"}

def handoff(state: SupportState):
    return {"answer": "我把这个问题转给人工客服继续处理。"}

builder = StateGraph(SupportState)
builder.add_node("classify", classify)
builder.add_node("lookup", lookup)
builder.add_node("answer", answer)
builder.add_node("ask_order_id", ask_order_id)
builder.add_node("handoff", handoff)

builder.add_edge(START, "classify")
builder.add_conditional_edges(
    "classify",
    choose_next,
    {
        "lookup": "lookup",
        "ask_order_id": "ask_order_id",
        "handoff": "handoff",
    },
)
builder.add_edge("lookup", "answer")
builder.add_edge("answer", END)
builder.add_edge("ask_order_id", END)
builder.add_edge("handoff", END)

support_flow = builder.compile()
result = support_flow.invoke({
    "question": "我的订单 12345 到哪了？",
    "order_id": "12345",
    "intent": "",
    "order_info": "",
    "answer": "",
})
print(result["answer"])
~~~

读代码时抓住这些动作：

- SupportState 是所有步骤共用的一张“工作单”。节点可以读取它，并返回要更新的字段。
- add_node 把函数登记为一个步骤。
- add_edge 连接固定的前后步骤。
- add_conditional_edges 根据当前状态选择下一条路线。
- compile 准备好这张流程图；invoke 用初始状态运行一次。

状态像随流程更新的工作单，节点像处理工作单的岗位，边像“下一步交给谁”的规则。更复杂的流程还可以保存进度、暂停等待人工、恢复执行和重试。

## 两者的关系：可以组合，也可以分别使用

LangChain 的 create_agent 适合快速搭起“模型根据需要调用工具”的 Agent。LangGraph 适合把“先做什么、什么条件下走哪条路、何时等待人工、失败后是否重试”明确写成流程。

当前 LangChain Agent 使用了 LangGraph 的运行机制。开发者可以使用高层的 create_agent，也可以直接用 LangGraph 精细搭建流程。LangGraph 节点可以调用 LangChain 的模型或工具，也可以调用普通 Python 函数、其他 SDK 或自建服务。

可以按具体场景选择：

- 用户问题开放，需要模型在少量工具中决定下一步：从 LangChain Agent 开始。
- 业务有固定关卡、条件分支、审核或恢复流程：用 LangGraph 表达流程。
- 流程只有几步确定的 Python 调用：直接写普通函数也很合适。

## 和你的两个项目怎么对应

### EchoMind

EchoMind 的 Agent 主链使用自建 AgentOrchestrator 和 Anthropic SDK 的 tool_use，而不是 LangChain 或 LangGraph 的 Agent 执行器。项目显式处理工具 schema、白名单和参数校验、handler 调用、tool_result 回填，并限制单 Agent 最多 3 轮。

它和上面的 LangChain 客服例子在“模型请求工具、应用执行工具、把结果交回模型”的基本循环上相似；差别在于，EchoMind 的循环和权限控制由自己的 Orchestrator 实现。

### Enterprise Insight Agent

项目主链由自建 IntelligenceWorkflow 调用 GPT Researcher，负责研究任务、网页检索、证据整理、报告生成和主张审查。核心研究工作流不是 LangGraph 图。

仓库使用了部分 LangChain 生态组件，例如 langchain_core.documents.Document、文本切分器和可选向量存储路径。准确描述是使用了部分 LangChain 组件；主工作流仍由项目自己的流程编排。

## 面试回答

### LangChain 和 LangGraph 分别是什么？

> LangChain 把模型接入、工具描述、检索和 Agent 执行封装成可组合的组件。比如我可以把订单查询函数注册成工具，让模型判断何时调用，再由 Agent 执行器把结果交回模型。LangGraph 进一步把有状态流程表示成节点和边，明确表达分支、循环、暂停和恢复。LangChain Agent 可以使用 LangGraph 的运行机制；需要更明确控制流程时，也可以直接构建 LangGraph。

### 为什么 EchoMind 没用 LangChain 或 LangGraph？

> EchoMind 的核心 Agent 流程规模适合自建：我直接使用 Anthropic tool_use，在 Orchestrator 中处理 schema、白名单、参数校验、工具执行、结果回填和最大轮数。这样项目的工具权限和执行边界比较直接。现有流程如果增加长任务恢复、人工审批或更复杂的分支，再评估 LangGraph 的状态图能力。

## 参考资料

- [LangChain Agents：模型、工具和 Agent 执行器](https://docs.langchain.com/oss/python/langchain/agents)
- [LangGraph Graph API：State、Node、Edge 与编译执行](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [Thinking in the Graph API：客服流程案例](https://docs.langchain.com/oss/python/langgraph/thinking-in-graph-api)
- [LangChain 学习资源：Agent 基于 LangGraph primitives](https://docs.langchain.com/oss/python/learn)
