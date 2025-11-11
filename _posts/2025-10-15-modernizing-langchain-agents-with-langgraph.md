---
layout: post
title: "Modernizing LangChain Agents: From initialize_agent to LangGraph"
date: 2025-10-15 16:00:00 -0500
categories: [ai, software-engineering]
tags: [langchain, langgraph, python, ai-agents, refactoring]
---

LangChain's `initialize_agent()` is deprecated. If you're still using it, you're building on a dead API. Here's how I migrated [Colosseum](https://github.com/davdunc/colosseum) to modern LangGraph patterns and why you should too.

## The Old Way: initialize_agent()

My original agent implementation looked like this:

```python
from langchain.agents import initialize_agent, AgentType
from langchain.llms import OpenAI

llm = OpenAI(temperature=0.7)
tools = [search_tool, calculator_tool]

agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True
)

response = agent.run("What is 25 * 4?")
```

This worked, but:
- ⚠️ `initialize_agent()` is deprecated
- ⚠️ Limited control over agent behavior
- ⚠️ Hard to customize the prompt template
- ⚠️ No structured output parsing
- ⚠️ Difficult to debug agent reasoning

## The New Way: LangGraph

LangGraph is LangChain's modern agent framework. It represents agents as graphs with:
- **Nodes**: Agent actions (LLM calls, tool executions)
- **Edges**: Control flow (conditional routing)
- **State**: Data passed between nodes

### Basic Migration

Here's the equivalent LangGraph implementation:

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

llm = ChatOpenAI(model="gpt-4", temperature=0.7)
tools = [search_tool, calculator_tool]

agent = create_react_agent(llm, tools)

# Invoke the agent
result = agent.invoke({
    "messages": [("user", "What is 25 * 4?")]
})

# Access the response
print(result["messages"][-1].content)
```

Key differences:
1. `ChatOpenAI` instead of `OpenAI` (supports chat models)
2. `create_react_agent()` instead of `initialize_agent()`
3. Messages passed as conversation list
4. Structured result with full message history

## The ReAct Pattern

LangGraph's `create_react_agent` implements the ReAct pattern:
- **Re**asoning: Agent thinks about what to do
- **Act**ion: Agent executes a tool
- **Observe**: Agent sees tool results
- **Repeat**: Until task is complete

This is the same pattern `ZERO_SHOT_REACT_DESCRIPTION` used, but with better implementation.

## Real-World Example: Colosseum SupervisorAgent

Here's how I refactored Colosseum's agent supervisor:

### Before (Deprecated API)

```python
from langchain.agents import initialize_agent, AgentType
from langchain.llms import OpenAI

class SupervisorAgent:
    def __init__(self, config):
        self.llm = OpenAI(
            temperature=config.get("temperature", 0.7),
            model_name=config.get("model", "gpt-4")
        )
        self.tools = []
        self.agent = None

    def register_tool(self, tool):
        self.tools.append(tool)
        # Need to recreate agent every time tools change
        self._rebuild_agent()

    def _rebuild_agent(self):
        self.agent = initialize_agent(
            tools=self.tools,
            llm=self.llm,
            agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
            verbose=True,
            max_iterations=10,
        )

    def execute(self, task: str) -> str:
        if not self.agent:
            raise ValueError("No tools registered")
        return self.agent.run(task)
```

Problems:
- Rebuilding agent on every tool registration is expensive
- Limited access to intermediate steps
- Hard to add custom behavior
- `run()` returns just a string, losing context

### After (Modern LangGraph)

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from typing import Optional

class SupervisorAgent:
    def __init__(self, config):
        self.llm = ChatOpenAI(
            model=config.get("model", "gpt-4"),
            temperature=config.get("temperature", 0.7)
        )
        self.tools = []
        self.agent = None
        self.config = config

    def register_tool(self, tool):
        """Register a tool - agent created on first execution"""
        self.tools.append(tool)
        self.agent = None  # Mark for recreation

    def _ensure_agent(self):
        """Lazy agent creation"""
        if self.agent is None:
            self.agent = create_react_agent(
                self.llm,
                self.tools,
                state_modifier=self._get_system_message()
            )

    def _get_system_message(self) -> str:
        """Custom system prompt"""
        return """You are a supervisor agent in a multi-agent investment committee.

Your role is to:
1. Coordinate information gathering from specialized agents
2. Synthesize multiple perspectives into coherent analysis
3. Make final investment decisions based on consensus
4. Maintain risk management standards

Available tools allow you to:
- Query market data
- Get positions from broker accounts
- Place trades (with risk limits)
- Access news and sentiment data

Think carefully before taking actions, especially trades.
"""

    def execute(self, task: str) -> dict:
        """Execute task and return full result"""
        self._ensure_agent()

        result = self.agent.invoke({
            "messages": [("user", task)]
        })

        return {
            "output": result["messages"][-1].content,
            "messages": result["messages"],
            "tool_calls": self._extract_tool_calls(result["messages"]),
        }

    def _extract_tool_calls(self, messages) -> list[dict]:
        """Extract all tool calls from message history"""
        tool_calls = []
        for msg in messages:
            if hasattr(msg, 'tool_calls') and msg.tool_calls:
                for tc in msg.tool_calls:
                    tool_calls.append({
                        'tool': tc['name'],
                        'args': tc['args'],
                    })
        return tool_calls

    async def execute_async(self, task: str) -> dict:
        """Async execution for better performance"""
        self._ensure_agent()

        result = await self.agent.ainvoke({
            "messages": [("user", task)]
        })

        return {
            "output": result["messages"][-1].content,
            "messages": result["messages"],
            "tool_calls": self._extract_tool_calls(result["messages"]),
        }
```

Improvements:
- ✅ Lazy agent creation (only when needed)
- ✅ Custom system message via `state_modifier`
- ✅ Full message history returned
- ✅ Tool call extraction for auditing
- ✅ Async support for better performance
- ✅ Structured output dictionary

## Custom System Prompts

One of LangGraph's best features is easy prompt customization:

```python
def create_technical_analyst_agent(tools):
    """Create agent specialized in technical analysis"""
    llm = ChatOpenAI(model="gpt-4", temperature=0.3)

    system_message = """You are a technical analysis specialist.

Your expertise:
- Chart patterns (head and shoulders, triangles, flags)
- Technical indicators (RSI, MACD, moving averages)
- Support and resistance levels
- Volume analysis

When analyzing securities:
1. Always check multiple timeframes
2. Look for confluence of signals
3. Consider volume confirmation
4. Note key support/resistance levels

Provide specific entry/exit levels and rationale.
"""

    return create_react_agent(
        llm,
        tools,
        state_modifier=system_message
    )
```

This is much cleaner than the old way:

```python
# Old way - had to manually construct the prompt
from langchain.prompts import ZeroShotAgent

prefix = """You are a technical analysis specialist..."""
suffix = """Begin!\n\nQuestion: {input}\n{agent_scratchpad}"""

prompt = ZeroShotAgent.create_prompt(
    tools,
    prefix=prefix,
    suffix=suffix,
    input_variables=["input", "agent_scratchpad"]
)

# Then pass to agent
agent = initialize_agent(..., agent_kwargs={"prompt": prompt})
```

## Streaming Responses

LangGraph makes streaming trivial:

```python
async def stream_agent_response(agent, task: str):
    """Stream agent responses as they're generated"""
    async for event in agent.astream_events(
        {"messages": [("user", task)]},
        version="v1"
    ):
        if event["event"] == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            if chunk.content:
                print(chunk.content, end="", flush=True)
```

This provides real-time feedback as the agent thinks and acts.

## Debugging with LangGraph

LangGraph provides much better debugging:

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(llm, tools)

# Execute and get full state
result = agent.invoke({
    "messages": [("user", "Analyze AAPL")]
})

# Inspect each message
for i, msg in enumerate(result["messages"]):
    print(f"\n--- Message {i} ---")
    print(f"Type: {msg.type}")
    print(f"Content: {msg.content}")

    if hasattr(msg, 'tool_calls') and msg.tool_calls:
        print(f"Tool calls: {msg.tool_calls}")
```

Output:
```
--- Message 0 ---
Type: human
Content: Analyze AAPL

--- Message 1 ---
Type: ai
Content: I'll analyze Apple stock using market data and news.
Tool calls: [{'name': 'get_market_data', 'args': {'symbol': 'AAPL'}}]

--- Message 2 ---
Type: tool
Content: {"price": 150.23, "change": +1.5%, ...}

--- Message 3 ---
Type: ai
Content: Based on the data, AAPL is showing...
```

## Migration Checklist

If you're still using `initialize_agent()`, here's your migration path:

1. **Update imports**
   ```python
   # Old
   from langchain.llms import OpenAI
   from langchain.agents import initialize_agent, AgentType

   # New
   from langchain_openai import ChatOpenAI
   from langgraph.prebuilt import create_react_agent
   ```

2. **Switch to chat models**
   ```python
   # Old
   llm = OpenAI(temperature=0.7)

   # New
   llm = ChatOpenAI(model="gpt-4", temperature=0.7)
   ```

3. **Update agent creation**
   ```python
   # Old
   agent = initialize_agent(
       tools, llm, agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION
   )

   # New
   agent = create_react_agent(llm, tools)
   ```

4. **Change invocation**
   ```python
   # Old
   result = agent.run("Do something")

   # New
   result = agent.invoke({"messages": [("user", "Do something")]})
   output = result["messages"][-1].content
   ```

5. **Update to async if needed**
   ```python
   # Async version
   result = await agent.ainvoke({"messages": [("user", "Do something")]})
   ```

## Why This Matters

The migration to LangGraph isn't just about avoiding deprecation warnings. It's about:

1. **Better control** - Fine-grained control over agent behavior
2. **Observability** - Full message history and tool call tracking
3. **Performance** - Async support and streaming
4. **Flexibility** - Easy to customize prompts and add logic
5. **Future-proof** - LangGraph is the future of LangChain agents

## Advanced: Custom Agent Graphs

Beyond `create_react_agent`, you can build completely custom agent graphs:

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated, Sequence
import operator

class AgentState(TypedDict):
    messages: Annotated[Sequence, operator.add]
    risk_check_passed: bool
    analysis_complete: bool

def analyst_node(state):
    # Run market analysis
    analysis = analyst_agent.invoke(state["messages"])
    return {"messages": analysis["messages"], "analysis_complete": True}

def risk_check_node(state):
    # Check if proposed trade passes risk limits
    passed = check_risk_limits(state["messages"])
    return {"risk_check_passed": passed}

def execution_node(state):
    # Execute trade if risk check passed
    if state["risk_check_passed"]:
        result = execute_trade(state["messages"])
        return {"messages": [result]}
    else:
        return {"messages": [("system", "Trade rejected by risk management")]}

# Build custom graph
workflow = StateGraph(AgentState)
workflow.add_node("analyst", analyst_node)
workflow.add_node("risk_check", risk_check_node)
workflow.add_node("execution", execution_node)

workflow.set_entry_point("analyst")
workflow.add_edge("analyst", "risk_check")
workflow.add_conditional_edges(
    "risk_check",
    lambda x: "execution" if x["risk_check_passed"] else END
)
workflow.add_edge("execution", END)

app = workflow.compile()
```

This level of control is impossible with `initialize_agent()`.

## Conclusion

Migrating from `initialize_agent()` to LangGraph:
- ✅ Takes ~30 minutes for basic agents
- ✅ Provides much better debugging
- ✅ Enables advanced customization
- ✅ Future-proofs your codebase

If you haven't migrated yet, now's the time. The old API won't be around forever.

---

*Migrating LangChain agents? Questions about LangGraph? Find me on [GitHub](https://github.com/davdunc).*
