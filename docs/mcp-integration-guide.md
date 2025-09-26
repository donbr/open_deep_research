# Open Deep Research - MCP Integration Guide

This guide explains how to integrate new Model Context Protocol (MCP) tools into the Open Deep Research system using the minimal-surgery prototype framework. It provides a complete workflow from development to production deployment.

**Source**: [`odr-mcp-prototype.yaml`](odr-mcp-prototype.yaml)
**Philosophy**: Leverage existing architectural patterns to add new tools without modifying the core graph structure.

---

## 🎯 Integration Philosophy

### Core Principle: "No Graph Surgery"
The Open Deep Research system is designed for extensibility through **configuration-driven tool injection** rather than architectural modifications. The MCP integration framework follows these principles:

- **Minimal Code Changes**: Only modify configuration files and prompts
- **Existing Pattern Leverage**: Use established tool discovery and error handling patterns
- **Configuration-Driven**: New tools added via configuration rather than code changes
- **Collision Detection**: Automatic handling of tool name conflicts
- **Authentication Wrapper**: Built-in OAuth and error handling for MCP servers

---

## 🛠️ Example Integration: Text Classification Tool

### Tool Specification

The prototype demonstrates integrating a `classify_text` tool with the following specification:

```yaml
name: classify_text
description: "Categorize and label text content for research organization"
input_schema:
  type: object
  properties:
    text:
      type: string
      description: "Text content to classify"
  required: [text]
output_schema:
  type: object
  properties:
    label:
      type: string
      description: "Classification label"
    confidence:
      type: number
      description: "Confidence score (0.0-1.0)"
    categories:
      type: array
      items:
        type: string
      description: "List of relevant categories"
```

### MCP Server Configuration

```yaml
server:
  name: demo_tools
  transport: streamable_http
  auth: "optional"  # Set to "required" for production OAuth flow
  endpoint: "https://localhost:3000/mcp"
```

---

## 🔌 Integration Architecture

### Tool Flow Pipeline

The MCP tool integration follows a 9-step pipeline leveraging existing system patterns:

1. **Tool Request**: `researcher` node calls `get_all_tools(config)` (`deep_researcher.py:384`)
2. **MCP Loading**: `get_all_tools()` calls `load_mcp_tools()` (`utils.py:594`)
3. **Client Creation**: `load_mcp_tools()` creates `MultiServerMCPClient` (`utils.py:500`)
4. **Tool Discovery**: Client fetches tools from MCP server via HTTP transport
5. **Authentication Wrapper**: Tools wrapped with auth handler (`utils.py:521`)
6. **Tool Registration**: `classify_text` added to researcher's tool arsenal
7. **Execution**: `researcher_tools` node executes tool calls (`deep_researcher.py:474-479`)
8. **Output Capture**: Tool outputs captured in `ToolMessage` format
9. **Synthesis**: Results flow to `compress_research` for synthesis

### State Management Integration

| State Key | Role in MCP Integration |
|-----------|------------------------|
| `researcher_messages` | Contains tool calls and ToolMessage responses |
| `tool_call_iterations` | Incremented after each MCP tool use (`deep_researcher.py:422`) |
| `compressed_research` | Synthesized output includes MCP tool results |
| `raw_notes` | Unprocessed tool outputs preserved for final report |

### Error Handling Patterns

| Error Type | Handling Strategy | Implementation |
|------------|------------------|----------------|
| Connection Failures | Empty list returned from `load_mcp_tools()` | Graceful degradation |
| Authentication Failures | `ToolException` with user-friendly message | OAuth wrapper |
| Tool Execution Errors | Wrapped in `execute_tool_safely()` | `deep_researcher.py:427` |
| Name Collisions | `warnings.warn()` with skip behavior | `utils.py:510-514` |

---

## 📝 Required Code Changes

### 1. Configuration Update

**File**: `src/open_deep_research/configuration.py`
**Lines**: 27-30

```python
# Before
tools: Optional[List[str]] = Field(default=None, optional=True)

# After
tools: Optional[List[str]] = Field(default=["classify_text"], optional=True)
```

### 2. Prompt Enhancement

**File**: `src/open_deep_research/prompts.py`
**Lines**: 146-152

```python
# Before
<Available Tools>
You have access to two main tools:
1. **tavily_search**: For conducting web searches to gather information
2. **think_tool**: For reflection and strategic planning during research

# After
<Available Tools>
You have access to these main tools:
1. **tavily_search**: For conducting web searches to gather information
2. **think_tool**: For reflection and strategic planning during research
3. **classify_text**: For categorizing and labeling text content
```

### 3. No Utils Changes Required

The existing patterns in `utils.py` automatically handle new MCP tools:
- `load_mcp_tools()` discovers tools from server automatically
- Collision detection (`lines 510-514`) prevents name conflicts
- Auth wrapper (`lines 385-447`) handles MCP errors
- `get_all_tools()` (`lines 569-597`) assembles complete toolkit

---

## ✅ Validation & Testing Framework

### Development Environment Validation

#### 1. MCP Server Connection
```python
# Test: MultiServerMCPClient instantiation succeeds
from open_deep_research.utils import load_mcp_tools
config = {'configurable': {'mcp_config': {'url': 'http://localhost:3000'}}}
tools = await load_mcp_tools(config, set())
assert len(tools) > 0, "MCP server connection failed"
```

#### 2. Tool Discovery
```python
# Test: classify_text appears in get_all_tools() output
from open_deep_research.utils import get_all_tools
import asyncio

async def test():
    config = {
        'configurable': {
            'mcp_config': {
                'url': 'http://localhost:3000',
                'tools': ['classify_text']
            }
        }
    }
    tools = await get_all_tools(config)
    assert any(t.name == 'classify_text' for t in tools), 'classify_text tool not found'

asyncio.run(test())
```

#### 3. Collision Detection
**Existing Tools**: `think_tool`, `tavily_search`, `web_search`, `ConductResearch`, `ResearchComplete`
**Validation**: No `warnings.warn()` calls for tool name conflicts

#### 4. State Integration
- Tool calls appear in `researcher_messages` state
- `ToolMessage` with results in message history
- Tool outputs included in `compressed_research` synthesis
- `tool_call_iterations` increments correctly (`deep_researcher.py:422`)

---

## 🚀 Deployment Workflow

### Development Environment

**Mock Server Setup**:
```bash
# Mock MCP server for development
docker run -p 3000:3000 mock-mcp-server:latest
```

**Configuration** (`.env.local`):
```env
MCP_SERVER_URL=http://localhost:3000/mcp
MCP_AUTH_REQUIRED=false
```

**Testing**:
```bash
# Start LangGraph dev server
uvx langgraph dev --allow-blocking

# Test in LangGraph Studio
# URL: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
# Input: {"messages": [{"role": "human", "content": "Classify this text: AI is transforming healthcare"}]}
```

### Production Environment

**Authentication Flow**: OAuth token exchange via Supabase tokens

**Server Setup**:
```bash
export MCP_SERVER_URL=https://mcp-production.example.com/mcp
export MCP_AUTH_REQUIRED=true
```

**Configuration** (`.env.production`):
```env
GET_API_KEYS_FROM_CONFIG=true
SUPABASE_KEY=your-supabase-key
SUPABASE_URL=your-supabase-url
```

**Deployment**:
```bash
# Deploy to LangGraph Platform
langgraph deploy --env-file .env.production
```

---

## 💡 Usage Examples

### Example 1: Text Classification During Research

**Scenario**: User asks to research AI applications in healthcare

**Workflow**:
1. Supervisor delegates research task via `ConductResearch`
2. Researcher uses `tavily_search` to find healthcare AI content
3. Researcher calls `think_tool` to analyze search results
4. Researcher uses `classify_text` to categorize findings by medical domain
5. Researcher continues search based on classification insights
6. `compress_research` synthesizes all findings with classifications
7. Final report includes categorized insights

### Example 2: Content Organization

**Scenario**: Large research corpus needs categorization

**Workflow**:
1. Multiple parallel researchers gather diverse content
2. Each researcher uses `classify_text` to label their findings
3. Compression phase organizes content by classification labels
4. Final report structured around content categories

---

## 📊 Monitoring & Observability

### LangSmith Integration
- Tool calls tagged with `'langsmith:nostream'` (`deep_researcher.py:396`)
- MCP tool usage appears in experiment traces
- Error patterns visible in LangSmith dashboards

### Key Logging Points
- MCP client connection success/failure
- Tool discovery and collision warnings
- Authentication token refresh events
- Tool execution duration and success rates

---

## 📈 Scaling Considerations

### Concurrency Management
- MCP tools respect `max_concurrent_research_units` limit (1-20)
- Server-side rate limits may impact parallel research
- Consider tool output caching for expensive operations

### Fallback Strategies
- Graceful degradation when MCP server unavailable
- Empty tool list returned on connection failure
- Research continues with available tools

---

## 🔒 Security Framework

### Authentication
- OAuth token exchange prevents unauthorized access
- Supabase integration for token management
- Token refresh and expiration handling

### Input/Output Security
- Tool schemas enforce input constraints
- Tool outputs processed through `ToolMessage` wrapper
- HTTPS transport required for production MCP servers

### Network Security
- TLS/SSL for all MCP server communications
- Authentication headers for secure tool access
- Rate limiting and DDoS protection considerations

---

## 🔧 Development Commands

### Quick Tool Testing
```bash
# Test MCP tool discovery
python -c "
from open_deep_research.utils import get_all_tools
import asyncio
config = {'configurable': {'mcp_config': {'url': 'http://localhost:3000', 'tools': ['classify_text']}}}
print(asyncio.run(get_all_tools(config)))
"
```

### Integration Validation
```bash
# Start dev server with MCP config
uvx langgraph dev --config mcp_config.url=http://localhost:3000

# Test tool availability
curl -X POST http://127.0.0.1:2024/assistants/deep_researcher/invoke \
  -H "Content-Type: application/json" \
  -d '{"input": {"messages": [{"role": "human", "content": "Classify this research topic"}]}}'
```

---

## 🎯 Best Practices

### Tool Design Principles
1. **Single Responsibility**: Each tool should have a focused, well-defined purpose
2. **Clear Schemas**: Input/output schemas should be comprehensive and typed
3. **Error Handling**: Tools should provide meaningful error messages
4. **Performance**: Consider caching and optimization for expensive operations

### Integration Guidelines
1. **Test Locally First**: Always test with mock servers before production
2. **Monitor Collisions**: Watch for tool name conflicts in logs
3. **Validate Schemas**: Ensure input/output schemas match expectations
4. **Error Recovery**: Plan for MCP server downtime and failures

### Production Readiness
1. **Authentication**: Always use OAuth in production environments
2. **Rate Limiting**: Plan for API rate limits and implement backoff
3. **Monitoring**: Set up alerting for MCP server health and performance
4. **Documentation**: Maintain clear documentation for all custom tools

This guide provides a complete framework for integrating MCP tools into the Open Deep Research system while maintaining architectural integrity and production readiness.