---
layout: post
title: "Secure Broker Integration with Model Context Protocol (MCP)"
date: 2025-10-25 11:15:00 -0500
categories: [trading, security]
tags: [mcp, api-integration, security, python, interactive-brokers]
---

One of the biggest challenges in building AI trading agents is securely connecting them to broker APIs. How do you give an AI agent access to real money without creating a security nightmare?

Enter the Model Context Protocol (MCP) - a standardized way for AI agents to securely interact with external services. Here's how I'm using it in [Colosseum](https://github.com/davdunc/colosseum).

## The Security Problem

Traditional approaches to broker integration have issues:

### Approach 1: Direct API Keys in Agent Prompts
```python
# DON'T DO THIS
agent_prompt = f"""
You are a trading agent. Your API key is {API_KEY}.
Use it to place orders via the broker API.
"""
```

Problems:
- API keys exposed in agent memory
- Agent could leak credentials in responses
- No audit trail of what the agent did
- Difficult to revoke access

### Approach 2: Custom Agent Tools
```python
def place_order_tool(symbol, quantity, side):
    # Agent calls this directly
    broker_client.place_order(symbol, quantity, side)
```

Better, but:
- No standardization across brokers
- Hard to swap broker implementations
- Limited observability
- Credential management still complex

## Model Context Protocol Solution

MCP creates a standardized protocol between agents and services:

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Agent     │ ◄─────► │ MCP Server  │ ◄─────► │   Broker    │
│  (LLM)      │   MCP   │  (Gateway)  │   API   │  (IB/E*Trade)│
└─────────────┘         └─────────────┘         └─────────────┘
      ▲                        │
      │                        │
      │                   ┌────▼────┐
      └───────────────────┤  Logs   │
                          └─────────┘
```

Benefits:
1. **Security boundary** - Credentials never reach the agent
2. **Standardization** - All brokers implement the same interface
3. **Auditability** - All interactions logged
4. **Flexibility** - Easy to swap broker implementations

## MCP Architecture in Colosseum

### Base Protocol

Every MCP server implements the base protocol:

```python
from abc import ABC, abstractmethod

class MCPServer(ABC):
    """Base class for all MCP servers"""

    def __init__(self, server_id: str, config: dict):
        self.server_id = server_id
        self.config = config
        self.connected = False

    @abstractmethod
    async def connect(self) -> bool:
        """Establish connection to the service"""
        pass

    @abstractmethod
    async def disconnect(self) -> None:
        """Clean disconnect"""
        pass

    @abstractmethod
    async def health_check(self) -> dict:
        """Check if connection is healthy"""
        pass
```

### Broker-Specific Protocol

Broker servers extend the base with trading operations:

```python
class BrokerMCPServer(MCPServer):
    """MCP server for broker platforms"""

    @abstractmethod
    async def get_account_info(self) -> dict:
        """Retrieve account balances and buying power"""
        pass

    @abstractmethod
    async def get_positions(self) -> list[dict]:
        """Get current positions"""
        pass

    @abstractmethod
    async def get_orders(self) -> list[dict]:
        """Get active and recent orders"""
        pass

    @abstractmethod
    async def place_order(
        self,
        symbol: str,
        quantity: int,
        side: str,  # 'BUY' or 'SELL'
        order_type: str,  # 'MARKET', 'LIMIT', etc.
        limit_price: Optional[float] = None,
    ) -> dict:
        """Place a new order"""
        pass

    @abstractmethod
    async def cancel_order(self, order_id: str) -> bool:
        """Cancel an existing order"""
        pass

    @abstractmethod
    async def get_market_data(
        self,
        symbol: str,
        data_type: str  # 'quote', 'bars', 'trades'
    ) -> dict:
        """Retrieve market data"""
        pass
```

### Interactive Brokers Implementation

Here's a real example from `colosseum/mcp/ib.py`:

```python
from ib_insync import IB, MarketOrder, LimitOrder, Stock
from .base import BrokerMCPServer

class InteractiveBrokersMCP(BrokerMCPServer):
    """MCP server for Interactive Brokers"""

    def __init__(self, server_id: str, config: dict):
        super().__init__(server_id, config)
        self.ib = IB()
        self.host = config.get('host', '127.0.0.1')
        self.port = config.get('port', 7497)
        self.client_id = config.get('client_id', 1)

    async def connect(self) -> bool:
        """Connect to IB Gateway or TWS"""
        try:
            await self.ib.connectAsync(
                self.host,
                self.port,
                clientId=self.client_id
            )
            self.connected = True
            return True
        except Exception as e:
            self.connected = False
            raise ConnectionError(f"Failed to connect to IB: {e}")

    async def disconnect(self) -> None:
        """Disconnect from IB"""
        if self.connected:
            self.ib.disconnect()
            self.connected = False

    async def get_account_info(self) -> dict:
        """Get account information"""
        account_values = self.ib.accountValues()
        return {
            'net_liquidation': self._get_value(
                account_values, 'NetLiquidation'
            ),
            'buying_power': self._get_value(
                account_values, 'BuyingPower'
            ),
            'cash': self._get_value(
                account_values, 'TotalCashValue'
            ),
        }

    async def place_order(
        self,
        symbol: str,
        quantity: int,
        side: str,
        order_type: str,
        limit_price: Optional[float] = None,
    ) -> dict:
        """Place order through IB"""
        # Create contract
        contract = Stock(symbol, 'SMART', 'USD')

        # Create order
        if order_type == 'MARKET':
            order = MarketOrder(side, quantity)
        elif order_type == 'LIMIT':
            order = LimitOrder(side, quantity, limit_price)
        else:
            raise ValueError(f"Unsupported order type: {order_type}")

        # Place order
        trade = self.ib.placeOrder(contract, order)
        await self.ib.sleep(1)  # Give time to process

        return {
            'order_id': trade.order.orderId,
            'status': trade.orderStatus.status,
            'filled': trade.orderStatus.filled,
            'remaining': trade.orderStatus.remaining,
        }

    async def get_positions(self) -> list[dict]:
        """Get current positions"""
        positions = self.ib.positions()
        return [
            {
                'symbol': pos.contract.symbol,
                'quantity': pos.position,
                'avg_cost': pos.avgCost,
                'market_value': pos.marketValue,
            }
            for pos in positions
        ]

    def _get_value(self, account_values, tag: str) -> float:
        """Extract specific account value by tag"""
        for item in account_values:
            if item.tag == tag:
                return float(item.value)
        return 0.0
```

## Configuration Management

MCP servers are configured via JSON files, never in code:

`/etc/colosseum/ib_config.json`:
```json
{
  "server_id": "ib_main",
  "type": "broker",
  "provider": "interactive_brokers",
  "config": {
    "host": "127.0.0.1",
    "port": 7497,
    "client_id": 1,
    "account": "DU123456"
  },
  "credentials_file": "/etc/colosseum/secrets/ib_credentials.json"
}
```

Credentials are stored separately with restricted permissions:

`/etc/colosseum/secrets/ib_credentials.json`:
```json
{
{
  "username": "<USERNAME>",
  "password": "<ENCRYPTED_PASSWORD_HASH>"
}
```

```bash
# Restrict access
chmod 600 /etc/colosseum/secrets/ib_credentials.json
chown colosseum:colosseum /etc/colosseum/secrets/ib_credentials.json
```

## Loading MCP Servers

The `MCPLoader` dynamically instantiates servers:

```python
from colosseum.mcp.loader import MCPLoader
from colosseum.mcp.base import create_mcp_server

class MCPLoader:
    @staticmethod
    def load_from_config(config_path: str) -> MCPServer:
        """Load MCP server from config file"""
        with open(config_path) as f:
            config = json.load(f)

        # Factory pattern creates appropriate server
        return create_mcp_server(
            server_type=config['type'],
            provider=config['provider'],
            server_id=config['server_id'],
            config=config['config']
        )

# Usage
ib_server = MCPLoader.load_from_config('/etc/colosseum/ib_config.json')
await ib_server.connect()

# Now agents can use it
account_info = await ib_server.get_account_info()
```

## Agent Integration

Agents interact with MCP servers through registered tools:

```python
from langchain.tools import Tool

def create_broker_tools(mcp_server: BrokerMCPServer) -> list[Tool]:
    """Create LangChain tools from MCP server"""

    async def get_positions_tool() -> str:
        positions = await mcp_server.get_positions()
        return json.dumps(positions, indent=2)

    async def place_order_tool(
        symbol: str,
        quantity: int,
        side: str
    ) -> str:
        result = await mcp_server.place_order(
            symbol=symbol,
            quantity=quantity,
            side=side,
            order_type='MARKET'
        )
        return json.dumps(result, indent=2)

    return [
        Tool(
            name="get_positions",
            func=get_positions_tool,
            description="Get current portfolio positions"
        ),
        Tool(
            name="place_market_order",
            func=place_order_tool,
            description=(
                "Place a market order. "
                "Args: symbol (str), quantity (int), side (BUY/SELL)"
            )
        ),
    ]

# Register with supervisor
supervisor = SupervisorAgent(config)
ib_tools = create_broker_tools(ib_server)
for tool in ib_tools:
    supervisor.register_tool(tool)
```

Now the agent can trade, but never sees credentials:

```
Agent: I want to buy 100 shares of AAPL
Tool Call: place_market_order(symbol="AAPL", quantity=100, side="BUY")
MCP Server: → Validates request
            → Checks risk limits
            → Logs transaction
            → Calls IB API
            → Returns order confirmation
Agent: Order placed successfully, ID: 12345
```

## Audit Trail

Every MCP interaction is logged:

```python
class BrokerMCPServer(MCPServer):
    async def place_order(self, symbol, quantity, side, order_type, **kwargs):
        # Log the request
        self.logger.info(
            "Order request",
            extra={
                'server_id': self.server_id,
                'action': 'place_order',
                'symbol': symbol,
                'quantity': quantity,
                'side': side,
                'order_type': order_type,
                'timestamp': datetime.utcnow().isoformat(),
            }
        )

        try:
            result = await self._execute_order(symbol, quantity, side, order_type)

            # Log success
            self.logger.info(
                "Order placed",
                extra={
                    'server_id': self.server_id,
                    'order_id': result['order_id'],
                    'status': result['status'],
                }
            )

            return result

        except Exception as e:
            # Log failure
            self.logger.error(
                "Order failed",
                extra={
                    'server_id': self.server_id,
                    'error': str(e),
                }
            )
            raise
```

View logs:

```bash
journalctl -u colosseum-agent | grep "place_order"
```

## Multi-Broker Support

The beauty of MCP is swapping brokers without changing agent code:

```yaml
# config.yaml
mcp_servers:
  - server_id: "ib_main"
    type: "broker"
    provider: "interactive_brokers"
    config_file: "/etc/colosseum/ib_config.json"

  - server_id: "etrade_backup"
    type: "broker"
    provider: "etrade"
    config_file: "/etc/colosseum/etrade_config.json"

  - server_id: "das_trading"
    type: "broker"
    provider: "dastrader"
    config_file: "/etc/colosseum/das_config.json"
```

The agent doesn't care which broker it uses - they all implement the same MCP interface.

## Risk Management Layer

MCP servers can enforce risk controls:

```python
class BrokerMCPServer(MCPServer):
    def __init__(self, server_id: str, config: dict):
        super().__init__(server_id, config)
        self.risk_limits = config.get('risk_limits', {})

    async def place_order(self, symbol, quantity, side, **kwargs):
        # Check position limits
        positions = await self.get_positions()
        current_position = sum(
            p['quantity'] for p in positions if p['symbol'] == symbol
        )

        max_position = self.risk_limits.get('max_position_size', 1000)
        if abs(current_position + quantity) > max_position:
            raise ValueError(
                f"Order would exceed position limit of {max_position}"
            )

        # Check order size
        max_order_size = self.risk_limits.get('max_order_size', 500)
        if quantity > max_order_size:
            raise ValueError(
                f"Order size {quantity} exceeds limit of {max_order_size}"
            )

        # Check buying power
        account = await self.get_account_info()
        estimated_cost = quantity * await self._get_price(symbol)
        if estimated_cost > account['buying_power']:
            raise ValueError("Insufficient buying power")

        # All checks passed, place order
        return await self._execute_order(symbol, quantity, side, **kwargs)
```

Configure limits per server:

```json
{
  "server_id": "ib_main",
  "config": {
    "host": "127.0.0.1",
    "port": 7497
  },
  "risk_limits": {
    "max_position_size": 1000,
    "max_order_size": 500,
    "max_daily_trades": 100,
    "allowed_symbols": ["AAPL", "MSFT", "GOOGL"]
  }
}
```

## Testing with Mock Servers

For development, use mock MCP servers:

```python
class MockBrokerMCP(BrokerMCPServer):
    """Mock broker for testing"""

    def __init__(self, server_id: str, config: dict):
        super().__init__(server_id, config)
        self.mock_positions = []
        self.mock_orders = []
        self.mock_cash = 100000.0

    async def place_order(self, symbol, quantity, side, **kwargs):
        order_id = f"MOCK_{len(self.mock_orders)}"
        order = {
            'order_id': order_id,
            'symbol': symbol,
            'quantity': quantity,
            'side': side,
            'status': 'FILLED',
        }
        self.mock_orders.append(order)
        return order

    async def get_positions(self):
        return self.mock_positions

    async def get_account_info(self):
        return {
            'cash': self.mock_cash,
            'buying_power': self.mock_cash,
            'net_liquidation': self.mock_cash,
        }
```

## Conclusion

Model Context Protocol provides:

1. **Security** - Credentials isolated from agent
2. **Standardization** - Uniform interface across brokers
3. **Observability** - Complete audit trail
4. **Flexibility** - Easy to swap implementations
5. **Safety** - Enforce risk limits at the gateway

For AI trading systems, MCP is essential infrastructure. It creates clear security boundaries while enabling agent flexibility.

---

*Building AI trading agents? Check out [Colosseum on GitHub](https://github.com/davdunc/colosseum) for a complete MCP implementation.*
