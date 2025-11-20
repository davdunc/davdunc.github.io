---
layout: post
title: "Introducing Colosseum: A Multi-Agent Investment Committee Framework"
date: 2025-11-11 10:00:00 -0500
categories: [trading, ai]
tags: [python, langchain, langgraph, mcp, ai-agents]
---

I've been building [Colosseum](https://github.com/davdunc/colosseum), a multi-agent framework that simulates an investment committee using AI agents. Instead of a single AI making trading decisions, Colosseum orchestrates multiple specialized agents that collaborate like a real investment team.

## The Concept

Traditional algorithmic trading systems follow rigid rules or single-model predictions. But real investment committees work differently - they have:

- A technical analyst examining charts and patterns
- A fundamental analyst reviewing financials
- A news analyst monitoring market sentiment
- A risk manager ensuring position limits

Colosseum brings this collaborative approach to AI-driven investing.

## Architecture Overview

The framework is built on modern Python infrastructure:

```
┌─────────────────────────────────────────┐
│  Colosseum Multi-Agent Framework        │
├─────────────────────────────────────────┤
│  ┌──────────────┐  ┌─────────────────┐  │
│  │  PostgreSQL  │  │  Agent Services │  │
│  │   Database   │←─┤  - Supervisor   │  │
│  │  (State)     │  │  - Plugins      │  │
│  └──────────────┘  │  - MCP Clients  │  │
│                    └─────────────────┘  │
│  Podman Quadlets + systemd              │
└─────────────────────────────────────────┘
```

### Core Technologies

- **LangChain & LangGraph**: Modern agent orchestration using the `create_react_agent` pattern
- **Model Context Protocol (MCP)**: Secure, standardized communication between agents and external services
- **PostgreSQL 16**: Production-grade state persistence and conversation history
- **Podman Quadlets**: Container orchestration via systemd (replacing Kubernetes)

## The SupervisorAgent

At the heart of Colosseum is the `SupervisorAgent` - a coordinator that manages tool registration and MCP server connections:

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

class SupervisorAgent:
    def __init__(self, config):
        self.llm = ChatOpenAI(
            model="gpt-4",
            temperature=config.get("temperature", 0.7)
        )
        self.tools = []
        self.mcp_servers = {}

    def register_tool(self, tool):
        """Register a tool for agent use"""
        self.tools.append(tool)

    def create_agent(self):
        """Create the ReAct agent with registered tools"""
        return create_react_agent(
            self.llm,
            self.tools
        )
```

This supervisor pattern allows dynamic tool registration and maintains a registry of all available MCP servers (broker APIs, news feeds, market data).

## Model Context Protocol Integration

One of Colosseum's key innovations is using MCP for broker integration. MCP provides:

1. **Standardized interfaces** - All brokers implement the same base protocol
2. **Security boundaries** - Clear separation between agent logic and broker access
3. **Auditability** - All broker interactions are logged and traceable

Here's the MCP architecture from `colosseum/mcp/base.py`:

```python
class MCPServer(ABC):
    """Base class for all MCP servers"""

    @abstractmethod
    async def connect(self) -> bool:
        """Establish connection to the service"""
        pass

    @abstractmethod
    async def disconnect(self) -> None:
        """Clean disconnect"""
        pass

class BrokerMCPServer(MCPServer):
    """MCP server for broker platforms"""

    @abstractmethod
    async def get_account_info(self) -> dict:
        pass

    @abstractmethod
    async def place_order(self, symbol: str, quantity: int,
                          order_type: str) -> dict:
        pass

    @abstractmethod
    async def get_positions(self) -> list:
        pass
```

Current broker integrations include:
- **Interactive Brokers** (`ib.py`)
- **E*TRADE** (`etrade.py`)
- **DAS Trader** (`dastrader.py`)

## State Persistence

Multi-agent systems need to remember conversations and maintain state across sessions. Colosseum uses SQLAlchemy 2.0 with three core models:

```python
class AgentState(Base):
    """Stores individual agent state"""
    __tablename__ = 'agent_states'

    id = Column(Integer, primary_key=True)
    agent_id = Column(String, unique=True, nullable=False)
    state_data = Column(JSON)  # Flexible JSON storage
    updated_at = Column(DateTime, default=datetime.utcnow)

class ConversationHistory(Base):
    """Multi-agent conversation tracking"""
    __tablename__ = 'conversation_history'

    id = Column(Integer, primary_key=True)
    session_id = Column(String, nullable=False)
    agent_id = Column(String, nullable=False)
    message = Column(Text)
    timestamp = Column(DateTime, default=datetime.utcnow)

class MCPServerState(Base):
    """Cache MCP server connection state"""
    __tablename__ = 'mcp_server_states'

    id = Column(Integer, primary_key=True)
    server_id = Column(String, unique=True, nullable=False)
    connection_state = Column(JSON)
    last_connected = Column(DateTime)
```

The database supports both PostgreSQL (production) and SQLite (development), with automatic schema creation and health checks.

## Configuration System

Colosseum follows the XDG Base Directory Specification for Linux-native configuration management:

```python
# Configuration locations (in priority order):
# 1. ~/.config/colosseum/config.yaml (user-specific)
# 2. /var/lib/colosseum/config.yaml (service-specific)
# 3. /etc/colosseum/config.yaml (system-wide)

def load_config():
    config_paths = [
        Path.home() / '.config' / 'colosseum' / 'config.yaml',
        Path('/var/lib/colosseum/config.yaml'),
        Path('/etc/colosseum/config.yaml'),
    ]

    for path in config_paths:
        if path.exists():
            with open(path) as f:
                return yaml.safe_load(f)
```

Example configuration:

```yaml
agent:
  model: "gpt-4"
  temperature: 0.7

database:
  url: "postgresql://user:pass@localhost:5432/colosseum"

mcp_servers:
  - type: "broker"
    provider: "interactive_brokers"
    config_file: "/etc/colosseum/ib_config.json"

  - type: "news"
    provider: "etrade"
    config_file: "/etc/colosseum/etrade_config.json"
```

## Deployment with Podman Quadlets

Initially, I used Kubernetes (KIND), but ran into localhost networking issues - a notorious problem with KIND. The solution? Podman Quadlets with systemd.

Quadlets provide:
- Native Linux integration via systemd
- Simpler networking (bridge mode with proper DNS)
- No Kubernetes complexity for single-node deployments
- Better developer experience

The deployment system (`quadlet_deploy.py`) manages:
- Service installation and configuration
- Lifecycle management (start/stop/restart)
- Log viewing via journalctl
- Container shell access
- Database utilities

Services include:
- `colosseum-network.network`: Bridge network for container communication
- `colosseum-db.container`: PostgreSQL 16 Alpine
- `colosseum-agent.container`: Fedora-based agent runtime
- Persistent volumes for data and state

## What's Next

Current development focus:

1. **Enhanced agent specialization** - Creating dedicated technical, fundamental, and sentiment analysis agents
2. **Backtesting framework** - Historical simulation capabilities
3. **Risk management layer** - Position limits, exposure checks, kill switches
4. **Web UI** - Dashboard for monitoring agent decisions and portfolio state
5. **Paper trading integration** - Live testing without real money

## Why This Matters

Multi-agent systems represent a shift from monolithic AI to collaborative intelligence. Rather than trying to build one perfect model, we can:

- Specialize agents for specific tasks
- Enable debate and consensus building
- Maintain audit trails of decision-making
- Adapt by swapping or adding agents

Colosseum makes this practical for quantitative trading.

---

*Interested in multi-agent trading systems? Check out the [Colosseum repository](https://github.com/davdunc/colosseum) or reach out on [GitHub](https://github.com/davdunc).*
