# 05. MCP & Tool Integration

[← Назад к списку тем](README.md)

---

## Model Context Protocol (MCP)

### What is MCP?

```
┌─────────────────────────────────────────────────────────────────────┐
│              Model Context Protocol (MCP)                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MCP = Standard protocol for LLM ↔ Tool communication               │
│  Created by Anthropic, open specification                           │
│                                                                     │
│  ┌─────────────┐         MCP          ┌─────────────┐              │
│  │   Client    │ ◄─────────────────► │   Server    │              │
│  │  (Claude,   │   JSON-RPC 2.0      │  (Tools,    │              │
│  │   Agent)    │                      │   Data)     │              │
│  └─────────────┘                      └─────────────┘              │
│                                                                     │
│  Key concepts:                                                      │
│  - Resources: data sources (files, DBs, APIs)                       │
│  - Tools: executable functions                                      │
│  - Prompts: reusable prompt templates                               │
│  - Sampling: LLM can request completions from server                │
│                                                                     │
│  Benefits:                                                          │
│  - Standardized tool interface                                      │
│  - Security boundaries                                              │
│  - Composable servers                                               │
│  - Language agnostic                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### MCP Server Implementation (Python)

```python
from mcp.server import Server, NotificationOptions
from mcp.server.models import InitializationOptions
import mcp.server.stdio
import mcp.types as types

# Create server
server = Server("my-tools-server")

# Define tools
@server.list_tools()
async def handle_list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="search_database",
            description="Search the product database",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "Search query"
                    },
                    "limit": {
                        "type": "integer",
                        "default": 10
                    }
                },
                "required": ["query"]
            }
        ),
        types.Tool(
            name="get_user_info",
            description="Get user information by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "user_id": {"type": "string"}
                },
                "required": ["user_id"]
            }
        )
    ]

# Handle tool calls
@server.call_tool()
async def handle_call_tool(
    name: str,
    arguments: dict | None
) -> list[types.TextContent]:

    if name == "search_database":
        results = await database.search(
            query=arguments["query"],
            limit=arguments.get("limit", 10)
        )
        return [types.TextContent(
            type="text",
            text=json.dumps(results)
        )]

    elif name == "get_user_info":
        user = await database.get_user(arguments["user_id"])
        return [types.TextContent(
            type="text",
            text=json.dumps(user)
        )]

    raise ValueError(f"Unknown tool: {name}")

# Define resources
@server.list_resources()
async def handle_list_resources() -> list[types.Resource]:
    return [
        types.Resource(
            uri="file:///config/settings.json",
            name="Application Settings",
            mimeType="application/json"
        )
    ]

@server.read_resource()
async def handle_read_resource(uri: str) -> str:
    if uri == "file:///config/settings.json":
        return json.dumps(load_settings())
    raise ValueError(f"Unknown resource: {uri}")

# Run server
async def main():
    async with mcp.server.stdio.stdio_server() as (read, write):
        await server.run(
            read,
            write,
            InitializationOptions(
                server_name="my-tools",
                server_version="1.0.0"
            )
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### MCP Server (TypeScript/Node.js)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-tools-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// List available tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_weather",
      description: "Get weather for a location",
      inputSchema: {
        type: "object",
        properties: {
          location: { type: "string", description: "City name" },
        },
        required: ["location"],
      },
    },
  ],
}));

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "get_weather") {
    const weather = await fetchWeather(args.location);
    return {
      content: [{ type: "text", text: JSON.stringify(weather) }],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### MCP Client Usage

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def use_mcp_tools():
    # Connect to MCP server
    server_params = StdioServerParameters(
        command="python",
        args=["my_mcp_server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize
            await session.initialize()

            # List available tools
            tools = await session.list_tools()
            print(f"Available tools: {[t.name for t in tools.tools]}")

            # Call a tool
            result = await session.call_tool(
                "search_database",
                {"query": "python tutorials", "limit": 5}
            )
            print(f"Result: {result}")

            # Read a resource
            resource = await session.read_resource(
                "file:///config/settings.json"
            )
            print(f"Settings: {resource}")
```

---

## Tool Design Patterns

### Tool Schema Best Practices

```python
# Good tool definition
good_tool = {
    "name": "create_calendar_event",
    "description": """Create a new calendar event.
Use this when the user wants to schedule a meeting or reminder.
Returns the event ID and confirmation.""",
    "inputSchema": {
        "type": "object",
        "properties": {
            "title": {
                "type": "string",
                "description": "Event title (required)",
                "minLength": 1,
                "maxLength": 100
            },
            "start_time": {
                "type": "string",
                "format": "date-time",
                "description": "Start time in ISO 8601 format"
            },
            "duration_minutes": {
                "type": "integer",
                "description": "Duration in minutes",
                "minimum": 15,
                "maximum": 480,
                "default": 60
            },
            "attendees": {
                "type": "array",
                "items": {"type": "string", "format": "email"},
                "description": "List of attendee email addresses"
            }
        },
        "required": ["title", "start_time"]
    }
}

# Bad tool definition (too vague)
bad_tool = {
    "name": "do_stuff",
    "description": "Does things",
    "inputSchema": {
        "type": "object",
        "properties": {
            "data": {"type": "string"}  # What data? What format?
        }
    }
}
```

### Error Handling

```python
from enum import Enum
from dataclasses import dataclass

class ToolErrorCode(Enum):
    INVALID_INPUT = "invalid_input"
    NOT_FOUND = "not_found"
    PERMISSION_DENIED = "permission_denied"
    RATE_LIMITED = "rate_limited"
    INTERNAL_ERROR = "internal_error"

@dataclass
class ToolResult:
    success: bool
    data: any = None
    error_code: ToolErrorCode = None
    error_message: str = None

async def safe_tool_call(name: str, args: dict) -> ToolResult:
    """Execute tool with proper error handling."""
    try:
        # Validate inputs
        if not validate_args(name, args):
            return ToolResult(
                success=False,
                error_code=ToolErrorCode.INVALID_INPUT,
                error_message="Invalid arguments provided"
            )

        # Execute tool
        result = await execute_tool(name, args)

        return ToolResult(success=True, data=result)

    except NotFoundException as e:
        return ToolResult(
            success=False,
            error_code=ToolErrorCode.NOT_FOUND,
            error_message=str(e)
        )
    except PermissionError as e:
        return ToolResult(
            success=False,
            error_code=ToolErrorCode.PERMISSION_DENIED,
            error_message="Access denied"
        )
    except Exception as e:
        logger.exception("Tool execution failed")
        return ToolResult(
            success=False,
            error_code=ToolErrorCode.INTERNAL_ERROR,
            error_message="An unexpected error occurred"
        )
```

### Tool Composition

```python
class CompositeToolServer:
    """Compose multiple tool providers into one server."""

    def __init__(self):
        self.providers = {}

    def register_provider(self, name: str, provider):
        """Register a tool provider."""
        self.providers[name] = provider

    async def list_tools(self) -> list:
        """Aggregate tools from all providers."""
        all_tools = []
        for name, provider in self.providers.items():
            tools = await provider.list_tools()
            # Namespace tools by provider
            for tool in tools:
                tool.name = f"{name}.{tool.name}"
                all_tools.append(tool)
        return all_tools

    async def call_tool(self, name: str, args: dict):
        """Route tool call to appropriate provider."""
        provider_name, tool_name = name.split(".", 1)
        provider = self.providers.get(provider_name)

        if not provider:
            raise ValueError(f"Unknown provider: {provider_name}")

        return await provider.call_tool(tool_name, args)

# Usage
server = CompositeToolServer()
server.register_provider("calendar", CalendarTools())
server.register_provider("email", EmailTools())
server.register_provider("database", DatabaseTools())

# Tools: calendar.create_event, email.send, database.query, etc.
```

---

## Security Considerations

### Input Validation

```python
from pydantic import BaseModel, Field, validator
from typing import Optional

class SearchInput(BaseModel):
    query: str = Field(..., min_length=1, max_length=500)
    limit: int = Field(default=10, ge=1, le=100)
    filters: Optional[dict] = None

    @validator('query')
    def sanitize_query(cls, v):
        # Remove potential injection attempts
        dangerous = ['DROP', 'DELETE', 'UPDATE', ';', '--']
        for d in dangerous:
            if d.lower() in v.lower():
                raise ValueError(f"Invalid query content")
        return v

    @validator('filters')
    def validate_filters(cls, v):
        if v is None:
            return v
        allowed_keys = {'category', 'date_range', 'status'}
        if not set(v.keys()).issubset(allowed_keys):
            raise ValueError(f"Invalid filter keys")
        return v

async def search_tool(args: dict):
    # Validate with Pydantic
    try:
        validated = SearchInput(**args)
    except ValidationError as e:
        return {"error": str(e)}

    return await execute_search(validated)
```

### Permission Scopes

```python
from enum import Flag, auto

class ToolPermission(Flag):
    READ = auto()
    WRITE = auto()
    DELETE = auto()
    ADMIN = auto()

class SecureToolServer:
    def __init__(self):
        self.tool_permissions = {
            "search": ToolPermission.READ,
            "create_record": ToolPermission.WRITE,
            "delete_record": ToolPermission.DELETE,
            "admin_action": ToolPermission.ADMIN
        }

    async def call_tool(
        self,
        name: str,
        args: dict,
        user_permissions: ToolPermission
    ):
        required = self.tool_permissions.get(name)

        if required and not (required & user_permissions):
            raise PermissionError(
                f"Tool '{name}' requires {required.name} permission"
            )

        return await self._execute_tool(name, args)
```

---

## На интервью

### Типичные вопросы

1. **What is MCP?**
   - Model Context Protocol by Anthropic
   - Standard for LLM ↔ tool communication
   - JSON-RPC based, language agnostic

2. **MCP vs function calling?**
   - Function calling: model-specific APIs
   - MCP: standardized protocol, portable
   - MCP supports resources, prompts, sampling

3. **How to design good tools?**
   - Clear, specific descriptions
   - Well-defined schemas with constraints
   - Meaningful error messages
   - Single responsibility

4. **Tool security best practices?**
   - Input validation (Pydantic, JSON Schema)
   - Permission scopes
   - Rate limiting
   - Audit logging

5. **When to use MCP vs direct API?**
   - MCP: reusable tools, multi-client support
   - Direct: simple integrations, performance-critical

6. **Handling tool failures?**
   - Structured error responses
   - Retry logic for transient failures
   - Graceful degradation

---

[← Назад к списку тем](README.md)
