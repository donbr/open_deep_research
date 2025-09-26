# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Open Deep Research

A configurable, fully open-source deep research agent that works across multiple model providers, search tools, and MCP servers. Performance is on par with popular deep research agents (#6 on Deep Research Bench leaderboard with 0.4309 RACE score).

## Commands

```bash
# Environment setup
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
uv sync

# Run LangGraph Studio development server
uvx --refresh --from "langgraph-cli[inmem]" --with-editable . --python 3.11 langgraph dev --allow-blocking
# Or shorter: uvx langgraph dev

# Code quality checks
ruff check              # Linting
mypy                   # Type checking
ruff check && mypy     # Both checks

# Testing & evaluation
python tests/run_evaluate.py  # Run Deep Research Bench evaluation (costs $20-$100)
python tests/extract_langsmith_data.py --project-name "EXPERIMENT" --model-name "MODEL" --dataset-name "deep_research_bench"

# Run a single test
python -m pytest tests/test_specific.py -v
python -m pytest tests/test_specific.py::test_function_name -v
```


## Architecture

### LangGraph Workflow
8 nodes across 3 subgraphs orchestrating research:

1. **clarify_with_user** - Analyzes queries and asks clarifying questions if scope unclear
2. **write_research_brief** - Creates structured research briefs and initializes supervisor context
3. **supervisor** (subgraph) - Manages research delegation with configurable concurrency (1-20 researchers)
4. **researcher** (subgraph) - Conducts focused research using available tools
5. **compress_research** - Consolidates findings from parallel units with citation management
6. **final_report_generation** - Produces comprehensive reports with retry logic for token limits

**Key Patterns**:
- Supervisor calls `ConductResearch` to spawn parallel researcher subgraphs
- Each researcher tracks `tool_call_iterations` (max 30), supervisor tracks `research_iterations` (max 10)
- Custom `override_reducer` supports both replacement (`{"type": "override", "value": [...]}`) and additive updates

### State Management
Typed state classes with custom reducers (`state.py`):
- `AgentState` - Main workflow state with messages and research data
- `SupervisorState` - Research task coordination with iteration tracking
- `ResearcherState` - Individual researcher execution context with tool call counting
- `ResearcherOutputState` - Structured output from research units

State flow: `messages` → `research_brief` → `supervisor_messages` → parallel `researcher_messages` → `compressed_research` → `final_report`

### Tool Integration
Pattern in `utils.py:569-597`:
```python
tools = [built_in_tools] + [search_tools] + [mcp_tools]
# Name collision detection with warnings.warn()
```

**Categories**:
- **Built-in**: `think_tool`, `ConductResearch`, `ResearchComplete`
- **Search**: Tavily (async with summarization), OpenAI/Anthropic native search
- **MCP**: MultiServerMCPClient with OAuth, auth wrapping, error handling

MCP details:
- OAuth token exchange via Supabase (`utils.py:352-383`)
- Auth error wrapper with user-friendly messages (`utils.py:385-447`)
- Automatic discovery from MCP servers with tool allowlists
- Name collision warnings and skipping (`utils.py:510-514`)

## Key Implementation Details

### Error Handling & Token Limits
Provider-specific token limit detection (`utils.py:665-785`):
- **OpenAI**: Detects `BadRequestError` with token keywords and `context_length_exceeded` codes
- **Anthropic**: Detects `BadRequestError` with "prompt is too long" messages
- **Google/Gemini**: Detects `ResourceExhausted` exceptions and API core errors
- **Recovery**: Progressive truncation with model-specific token limits (29+ models in lookup table)

### Iteration Management
- **Supervisor Loop**: `research_iterations` tracked in supervisor node, exits on max iterations OR `ResearchComplete` OR no tool calls
- **Researcher Loop**: `tool_call_iterations` tracked in researcher node (`deep_researcher.py:421-423`), exits on max calls OR completion
- **Parallel Execution**: Up to `max_concurrent_research_units` researchers run simultaneously with `asyncio.gather()`

### State Reducer Semantics
Override reducer pattern (`state.py:55-60`):
```python
# Complete replacement:  {"type": "override", "value": [...]}
# Additive update:       direct list append
```

### MCP Tool Integration
1. **Discovery**: `get_all_tools()` → `load_mcp_tools()` → `MultiServerMCPClient.get_tools()`
2. **Authentication**: OAuth token exchange if `auth_required=true`
3. **Collision Handling**: Name conflicts trigger `warnings.warn()` and tool skipping
4. **Error Wrapping**: MCP tools wrapped with auth error handler

## Configuration

Configuration via `configuration.py` with UI metadata:
- **Multi-Model Pipeline**: Different models for summarization (gpt-4o-mini), research (gpt-4o), compression (gpt-4o), and final reports (gpt-4o)
- **Search APIs**: Tavily with summarization, OpenAI native search, Anthropic native search (max_uses: 5)
- **Concurrency**: `max_concurrent_research_units` (1-20), `max_researcher_iterations` (1-10), `max_react_tool_calls` (1-30)
- **Error Resilience**: `max_structured_output_retries` (1-10) with exponential backoff

### Testing Different Configurations
```bash
# Test different model configurations
uvx langgraph dev --config research_model=anthropic:claude-sonnet-4

# Test with different concurrency settings
uvx langgraph dev --config max_concurrent_research_units=10
```

### MCP Tool Development
```bash
# Test MCP tool discovery
python -c "
from open_deep_research.utils import get_all_tools
import asyncio
config = {'configurable': {'mcp_config': {'url': 'http://localhost:3000', 'tools': ['your_tool']}}}
print(asyncio.run(get_all_tools(config)))
"
```

## Performance Notes
- **Token Management**: Model-specific limits with progressive truncation (GPT-4o: 1M tokens, Claude-3-opus: 200K tokens)
- **Concurrency Impact**: Higher concurrent researchers may hit rate limits; monitor API quotas
- **Cost Estimation**: Deep Research Bench evaluation costs $20-$100 depending on model selection
- **Current Benchmark**: 0.4309 RACE score on Deep Research Bench leaderboard (#6 ranking)
- **Memory Growth**: `raw_notes` accumulation can grow large; compression pipeline mitigates this

## Technical Documentation
For detailed architectural analysis and integration patterns, see:
- **Complete Technical Analysis**: [`docs/odr-analysis-2025-01-25.md`](docs/odr-analysis-2025-01-25.md) - Deep dive into graph architecture, state management, and integration points
- **Machine-Readable Graph Structure**: [`docs/odr-graph-manifest.json`](docs/odr-graph-manifest.json) - JSON manifest of nodes, edges, state keys, and tools
- **MCP Integration Framework**: [`docs/odr-mcp-prototype.yaml`](docs/odr-mcp-prototype.yaml) - Rapid prototype plan for MCP tool integration

## Legacy Implementations
Two alternative implementations in `src/legacy/` provide different architectural approaches:
1. **Plan-and-Execute** (`graph.py`) - Sequential processing with human-in-the-loop
2. **Multi-Agent** (`multi_agent.py`) - Supervisor-researcher architecture with parallel processing

These are less performant than the current implementation but offer alternative patterns for research automation.