---
layout: post
title: "Building Stateful Multi-Agent Systems: Database Design for AI Agents"
date: 2025-11-20 09:00:00 -0500
categories: [ai, architecture]
tags: [python, sqlalchemy, postgresql, database, multi-agent]
---

AI agents need memory. Without state persistence, every conversation starts from scratch, agents can't coordinate across sessions, and you lose all audit trails. Here's how I designed the database layer for [Colosseum](https://github.com/davdunc/colosseum).

## The Problem

Multi-agent systems face unique state management challenges:

1. **Agent State**: Each agent needs to remember configuration, session context, and intermediate results
2. **Conversation History**: Multi-turn conversations must persist across sessions
3. **MCP Server State**: Broker connections, authentication tokens, and cached data
4. **Coordination**: Multiple agents sharing state without conflicts

A simple key-value store isn't enough. We need structured data with relationships, efficient queries, and transactional guarantees.

## Schema Design

I designed three SQLAlchemy ORM models to handle these requirements:

### 1. AgentState - Individual Agent Memory

```python
from sqlalchemy import Column, Integer, String, DateTime, JSON, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime

Base = declarative_base()

class AgentState(Base):
    """Stores individual agent state and session data"""
    __tablename__ = 'agent_states'

    id = Column(Integer, primary_key=True)
    agent_name = Column(String(255), nullable=False, index=True)
    session_id = Column(String(255), nullable=False, index=True)
    state_data = Column(JSON, nullable=False, default=dict)
    created_at = Column(DateTime, default=datetime.utcnow, nullable=False)
    updated_at = Column(
        DateTime,
        default=datetime.utcnow,
        onupdate=datetime.utcnow
    )
```

Key design decisions:

- **JSON column for state_data**: Allows flexible state without schema migrations. An agent can store whatever it needs - tool results, intermediate calculations, user preferences.
- **Composite index on (agent_name, session_id)**: Most queries filter by both fields. This makes lookups fast.
- **Automatic timestamps**: `onupdate=datetime.utcnow` tracks when state was last modified without explicit updates.

### 2. ConversationHistory - Multi-Agent Conversations

```python
class ConversationHistory(Base):
    """Multi-agent conversation tracking"""
    __tablename__ = 'conversation_history'

    id = Column(Integer, primary_key=True)
    session_id = Column(String(255), nullable=False, index=True)
    agent_name = Column(String(255), nullable=False)
    role = Column(String(50), nullable=False)  # user, assistant, system
    content = Column(Text, nullable=False)
    metadata = Column(JSON, nullable=True, default=dict)
    timestamp = Column(
        DateTime,
        default=datetime.utcnow,
        nullable=False,
        index=True
    )
```

This tracks every message in the investment committee:

```python
# Technical analyst reports findings
save_conversation(
    session_id="session_123",
    agent_name="technical_analyst",
    role="assistant",
    content="AAPL showing bullish divergence on RSI...",
    metadata={"indicators": ["RSI", "MACD"], "timeframe": "1h"}
)

# Risk manager responds
save_conversation(
    session_id="session_123",
    agent_name="risk_manager",
    role="assistant",
    content="Position size should not exceed 5% of portfolio...",
    metadata={"risk_score": 0.3, "max_position": 500}
)
```

The `metadata` JSON column stores agent-specific context without polluting the schema.

### 3. MCPServerState - Connection Caching

```python
class MCPServerState(Base):
    """Cache MCP server connection state"""
    __tablename__ = 'mcp_server_states'

    id = Column(Integer, primary_key=True)
    server_name = Column(String(255), nullable=False, unique=True, index=True)
    server_type = Column(String(100), nullable=False)
    state_data = Column(JSON, nullable=False, default=dict)
    last_connected = Column(DateTime, nullable=True)
```

MCP servers (broker APIs, news feeds) have authentication state and cached data:

```python
# Cache Interactive Brokers session
{
    "server_name": "ib_main",
    "server_type": "broker",
    "state_data": {
        "account_id": "DU123456",
        "session_token": "encrypted_token_here",
        "cached_positions": [...],
        "last_sync": "2025-11-20T09:00:00Z"
    },
    "last_connected": "2025-11-20T09:00:00"
}
```

## Database Manager

The `DatabaseManager` class handles connections with production-ready patterns:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from pathlib import Path
import os

class DatabaseManager:
    def __init__(self, database_url: str = None):
        if database_url is None:
            # XDG Base Directory for development
            data_dir = Path(
                os.environ.get('XDG_DATA_HOME', Path.home() / '.local/share')
            ) / 'colosseum'
            data_dir.mkdir(parents=True, exist_ok=True)
            database_url = f"sqlite:///{data_dir}/state.db"

        self.engine = create_engine(
            database_url,
            pool_pre_ping=True,  # Handle stale connections
            echo=False
        )
        self.Session = sessionmaker(bind=self.engine)

    def create_tables(self):
        """Create all tables if they don't exist"""
        Base.metadata.create_all(self.engine)

    def get_session(self):
        """Get a new database session"""
        return self.Session()
```

### Connection Pooling

The `pool_pre_ping=True` setting is critical for production:

```python
self.engine = create_engine(
    database_url,
    pool_pre_ping=True,  # Test connections before use
    pool_size=5,         # Maintain 5 connections
    max_overflow=10,     # Allow 10 additional under load
    pool_recycle=3600,   # Recycle connections after 1 hour
)
```

This prevents the "connection already closed" errors that plague long-running applications.

### Dual Database Support

Development uses SQLite for simplicity:

```python
# Development (automatic)
db = DatabaseManager()
# Uses: ~/.local/share/colosseum/state.db
```

Production uses PostgreSQL for performance and concurrent access:

```python
# Production
db = DatabaseManager(
    "postgresql://colosseum:password@localhost:5432/colosseum"
)
```

The same code works for both - SQLAlchemy handles the differences.

## Helper Functions

I wrapped common operations in simple helper functions:

### Saving Agent State

```python
def save_agent_state(
    agent_name: str,
    session_id: str,
    state_data: dict
) -> None:
    """Save or update agent state"""
    db = get_database_manager()
    session = db.get_session()

    try:
        # Check if state exists
        existing = session.query(AgentState).filter_by(
            agent_name=agent_name,
            session_id=session_id
        ).first()

        if existing:
            existing.state_data = state_data
            # updated_at handled automatically by onupdate
        else:
            state = AgentState(
                agent_name=agent_name,
                session_id=session_id,
                state_data=state_data
            )
            session.add(state)

        session.commit()
    except Exception as e:
        session.rollback()
        raise
    finally:
        session.close()
```

### Loading Agent State

```python
from typing import Optional

def load_agent_state(
    agent_name: str,
    session_id: str
) -> Optional[dict]:
    """Load agent state, returns None if not found"""
    db = get_database_manager()
    session = db.get_session()

    try:
        state = session.query(AgentState).filter_by(
            agent_name=agent_name,
            session_id=session_id
        ).first()

        return state.state_data if state else None
    finally:
        session.close()
```

### Saving Conversation Messages

```python
def save_conversation(
    session_id: str,
    agent_name: str,
    role: str,
    content: str,
    metadata: dict = None
) -> None:
    """Save a conversation message"""
    db = get_database_manager()
    session = db.get_session()

    try:
        message = ConversationHistory(
            session_id=session_id,
            agent_name=agent_name,
            role=role,
            content=content,
            metadata=metadata or {}
        )
        session.add(message)
        session.commit()
    except Exception as e:
        session.rollback()
        raise
    finally:
        session.close()
```

### Loading Conversation History

```python
def load_conversation_history(
    session_id: str,
    limit: int = None
) -> list[dict]:
    """Load conversation history for a session"""
    db = get_database_manager()
    session = db.get_session()

    try:
        query = session.query(ConversationHistory).filter_by(
            session_id=session_id
        ).order_by(ConversationHistory.timestamp.asc())

        if limit:
            query = query.limit(limit)

        return [
            {
                'agent_name': msg.agent_name,
                'role': msg.role,
                'content': msg.content,
                'metadata': msg.metadata,
                'timestamp': msg.timestamp.isoformat()
            }
            for msg in query.all()
        ]
    finally:
        session.close()
```

## Usage in Practice

### Supervisor Agent with State

```python
class SupervisorAgent:
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.agent_name = "supervisor"

        # Load previous state if exists
        self.state = load_agent_state(
            self.agent_name,
            self.session_id
        ) or {
            'tasks_completed': [],
            'current_analysis': None,
            'risk_budget': 1.0
        }

    async def analyze_opportunity(self, symbol: str):
        # Save that we're analyzing this symbol
        self.state['current_analysis'] = symbol
        save_agent_state(
            self.agent_name,
            self.session_id,
            self.state
        )

        # Coordinate with other agents
        technical = await self.get_technical_analysis(symbol)
        fundamental = await self.get_fundamental_analysis(symbol)

        # Save conversation for audit
        save_conversation(
            self.session_id,
            self.agent_name,
            "assistant",
            f"Analysis complete for {symbol}",
            metadata={
                'symbol': symbol,
                'technical_score': technical['score'],
                'fundamental_score': fundamental['score']
            }
        )

        # Update state
        self.state['tasks_completed'].append({
            'task': 'analyze',
            'symbol': symbol,
            'timestamp': datetime.utcnow().isoformat()
        })
        save_agent_state(
            self.agent_name,
            self.session_id,
            self.state
        )
```

### Multi-Agent Coordination

```python
async def investment_committee_session():
    session_id = str(uuid.uuid4())

    # Initialize agents with shared session
    supervisor = SupervisorAgent(session_id)
    technical = TechnicalAnalyst(session_id)
    risk_mgr = RiskManager(session_id)

    # Run analysis - each agent saves its state
    await supervisor.analyze_opportunity("AAPL")

    # Later: load entire conversation for audit
    history = load_conversation_history(session_id)

    for msg in history:
        print(f"[{msg['agent_name']}] {msg['role']}: {msg['content']}")
```

## Performance Considerations

### Indexing Strategy

The schema includes indexes on frequently queried columns:

```python
agent_name = Column(String(255), nullable=False, index=True)
session_id = Column(String(255), nullable=False, index=True)
timestamp = Column(DateTime, index=True)
```

For the most common query pattern, add a composite index:

```python
from sqlalchemy import Index

# Add to AgentState model
__table_args__ = (
    Index('ix_agent_session', 'agent_name', 'session_id'),
)
```

### JSON Column Queries

PostgreSQL supports JSON queries if you need to filter by state_data:

```python
# PostgreSQL-specific: query inside JSON
from sqlalchemy import cast
from sqlalchemy.dialects.postgresql import JSONB

# Find all agents with specific tool results
session.query(AgentState).filter(
    AgentState.state_data['tool_results'].astext.contains('AAPL')
).all()
```

SQLite lacks this capability - another reason PostgreSQL is better for production.

### Connection Management

Always close sessions to return connections to the pool:

```python
# Good - using try/finally
session = db.get_session()
try:
    # ... do work
finally:
    session.close()

# Better - using context manager
from contextlib import contextmanager

@contextmanager
def get_session():
    session = db.get_session()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Usage
with get_session() as session:
    session.add(new_state)
    # Commits automatically on success
```

## Testing

Test with an in-memory SQLite database:

```python
import pytest
from colosseum.database import DatabaseManager, AgentState, Base

@pytest.fixture
def db():
    """Create test database"""
    manager = DatabaseManager("sqlite:///:memory:")
    manager.create_tables()
    return manager

def test_save_load_agent_state(db):
    session = db.get_session()

    # Save state
    state = AgentState(
        agent_name="test_agent",
        session_id="test_session",
        state_data={"key": "value"}
    )
    session.add(state)
    session.commit()

    # Load state
    loaded = session.query(AgentState).filter_by(
        agent_name="test_agent",
        session_id="test_session"
    ).first()

    assert loaded.state_data == {"key": "value"}
    session.close()

def test_conversation_ordering(db):
    session = db.get_session()

    # Add messages out of order
    for i in [3, 1, 2]:
        msg = ConversationHistory(
            session_id="test",
            agent_name="agent",
            role="assistant",
            content=f"Message {i}"
        )
        session.add(msg)

    session.commit()

    # Should return in timestamp order
    messages = session.query(ConversationHistory).filter_by(
        session_id="test"
    ).order_by(ConversationHistory.timestamp.asc()).all()

    # Timestamps should be monotonically increasing
    timestamps = [m.timestamp for m in messages]
    assert timestamps == sorted(timestamps)
    session.close()
```

## Production Deployment

In production, the PostgreSQL database runs as a Podman Quadlet:

```ini
# colosseum-db.container
[Container]
Image=docker.io/library/postgres:16-alpine
ContainerName=colosseum-db
Network=colosseum-network.network

# Persistent storage
Volume=colosseum-db-data.volume:/var/lib/postgresql/data:Z

# Configuration
Environment=POSTGRES_DB=colosseum
Environment=POSTGRES_USER=colosseum
Environment=POSTGRES_PASSWORD=secure_password_here

# Health check
HealthCmd=pg_isready -U colosseum -d colosseum
HealthInterval=10s
HealthRetries=5
HealthStartPeriod=30s

# Networking
PublishPort=127.0.0.1:5432:5432
```

The agent container connects via the internal network:

```python
DATABASE_URL=postgresql://colosseum:password@colosseum-db:5432/colosseum
```

## Why This Matters

Stateful agents enable:

1. **Continuity**: Resume sessions where they left off
2. **Audit Trails**: Complete history of agent decisions
3. **Coordination**: Multiple agents sharing information
4. **Debugging**: Inspect agent state when things go wrong
5. **Analysis**: Query patterns in agent behavior over time

Without proper state management, AI agents are just stateless API calls. With it, they become intelligent systems that learn and coordinate.

---

*Building stateful AI agents? The complete implementation is in the [Colosseum repository](https://github.com/davdunc/colosseum). Questions? Find me on [GitHub](https://github.com/davdunc).*
