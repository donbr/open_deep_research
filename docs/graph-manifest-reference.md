# Open Deep Research - Graph Manifest Reference

This document provides a comprehensive overview of the LangGraph architecture based on the machine-readable graph manifest. It serves as a technical reference for understanding the system's structure, data flow, and integration patterns.

**Source**: [`odr-graph-manifest.json`](odr-graph-manifest.json)
**Commit**: b419df8d33b4f39ff5b2a34527bb6b85d0ede5d0
**Analysis Date**: 2025-01-25

---

## 🏗️ Graph Architecture Overview

The Open Deep Research system implements a **3-subgraph, 8-node architecture** with sophisticated state management and tool integration:

```
Main Graph (AgentState)
├── clarify_with_user
├── write_research_brief
├── research_supervisor (Supervisor Subgraph)
│   ├── supervisor
│   └── supervisor_tools
├── researcher_subgraph (Researcher Subgraph)
│   ├── researcher
│   ├── researcher_tools
│   └── compress_research
└── final_report_generation
```

---

## 📊 State Management System

### State Keys Architecture

The system uses **10 core state keys** with different access patterns and reduction strategies:

| State Key | Type | Access | Reducer | Purpose |
|-----------|------|--------|---------|---------|
| `messages` | `list[MessageLikeRepresentation]` | R/W | MessagesState default | Main conversation thread |
| `supervisor_messages` | `list[MessageLikeRepresentation]` | R/W | override_reducer | Supervisor conversation with override semantics |
| `researcher_messages` | `list[MessageLikeRepresentation]` | R/W | operator.add | Individual researcher conversations |
| `research_brief` | `str` | R/W | default | Structured research question |
| `raw_notes` | `list[str]` | R/W | override_reducer | Unprocessed research findings |
| `notes` | `list[str]` | R/W | override_reducer | Processed research findings |
| `compressed_research` | `str` | W | default | Synthesized findings from researchers |
| `research_iterations` | `int` | R/W | default | Supervisor iteration counter |
| `tool_call_iterations` | `int` | R/W | default | Researcher tool call counter |
| `final_report` | `str` | W | default | Final formatted report |

### State Reducer Patterns

**Override Reducer** (`state.py:55-60`):
```python
# Complete replacement
{"type": "override", "value": [...]}

# Additive update
direct_list_append
```

**Critical State Flow**:
```
messages → research_brief → supervisor_messages (override) →
parallel researcher_messages → compressed_research → final_report
```

---

## 🔄 Node Architecture

### Main Workflow Nodes

#### 1. Input Processing Layer

**`clarify_with_user`** (`deep_researcher.py:60-115`)
- **Reads**: `messages`
- **Writes**: `messages`
- **Purpose**: Analyze user messages and ask clarifying questions if research scope unclear
- **Routing**: Either proceeds to research brief OR ends with user question

**`write_research_brief`** (`deep_researcher.py:118-175`)
- **Reads**: `messages`
- **Writes**: `research_brief`, `supervisor_messages`
- **Purpose**: Transform user messages into structured research brief and initialize supervisor

#### 2. Research Coordination Layer

**`supervisor`** (`deep_researcher.py:178-223`)
- **Reads**: `supervisor_messages`
- **Writes**: `supervisor_messages`, `research_iterations`
- **Purpose**: Lead research supervisor that plans strategy and delegates to researchers
- **Tools**: `ConductResearch`, `ResearchComplete`, `think_tool`

**`supervisor_tools`** (`deep_researcher.py:225-349`)
- **Reads**: `supervisor_messages`, `research_iterations`
- **Writes**: `supervisor_messages`, `raw_notes`, `notes`
- **Purpose**: Execute tools called by supervisor including research delegation and thinking
- **Parallel Execution**: Spawns up to `max_concurrent_research_units` researcher subgraphs

#### 3. Individual Research Layer

**`researcher`** (`deep_researcher.py:365-424`)
- **Reads**: `researcher_messages`
- **Writes**: `researcher_messages`, `tool_call_iterations`
- **Purpose**: Individual researcher conducting focused research on specific topics
- **Tools**: Search tools, MCP tools, `think_tool`

**`researcher_tools`** (`deep_researcher.py:435-509`)
- **Reads**: `researcher_messages`, `tool_call_iterations`
- **Writes**: `researcher_messages`
- **Purpose**: Execute tools called by researcher including search and strategic thinking

**`compress_research`** (`deep_researcher.py:511-585`)
- **Reads**: `researcher_messages`
- **Writes**: `compressed_research`, `raw_notes`
- **Purpose**: Compress and synthesize research findings into concise structured summary

#### 4. Report Generation Layer

**`final_report_generation`** (`deep_researcher.py:607-697`)
- **Reads**: `messages`, `research_brief`, `notes`
- **Writes**: `final_report`, `messages`, `notes`
- **Purpose**: Generate final comprehensive research report with retry logic for token limits

---

## 🔀 Edge System & Control Flow

### Primary Flow Edges

| Source | Target | Condition | Implementation |
|--------|--------|-----------|----------------|
| START | `clarify_with_user` | always | `deep_researcher.py:714` |
| `clarify_with_user` | `write_research_brief` | need_clarification=false | `deep_researcher.py:77` |
| `clarify_with_user` | END | need_clarification=true | `deep_researcher.py:106-109` |
| `write_research_brief` | `research_supervisor` | always | `deep_researcher.py:163-174` |
| `research_supervisor` | `final_report_generation` | research_complete | `deep_researcher.py:715` |
| `final_report_generation` | END | always | `deep_researcher.py:716` |

### Supervisor Subgraph Edges

| Source | Target | Condition | Implementation |
|--------|--------|-----------|----------------|
| `supervisor` | `supervisor_tools` | always | `deep_researcher.py:217-222` |
| `supervisor_tools` | `supervisor` | continue_research=true | `deep_researcher.py:346-348` |
| `supervisor_tools` | END | research_complete OR max_iterations OR no_tool_calls | `deep_researcher.py:255-262` |

### Researcher Subgraph Edges

| Source | Target | Condition | Implementation |
|--------|--------|-----------|----------------|
| `researcher` | `researcher_tools` | always | `deep_researcher.py:418-423` |
| `researcher_tools` | `researcher` | continue_research=true | `deep_researcher.py:505-508` |
| `researcher_tools` | `compress_research` | max_iterations OR research_complete OR no_tool_calls | `deep_researcher.py:498-502` |
| `compress_research` | END | always | `deep_researcher.py:602` |

---

## 🔄 Loop Management

### Supervisor Loop
- **Nodes**: `supervisor` → `supervisor_tools`
- **Exit Condition**: `iterations >= max_researcher_iterations || ResearchComplete || no_tool_calls`
- **Iteration Variable**: `research_iterations`
- **Configuration**: `max_researcher_iterations` (default: 6, max: 10)

### Researcher Loop
- **Nodes**: `researcher` → `researcher_tools`
- **Exit Condition**: `tool_call_iterations >= max_react_tool_calls || ResearchComplete || no_tool_calls`
- **Iteration Variable**: `tool_call_iterations`
- **Configuration**: `max_react_tool_calls` (default: 10, max: 30)

---

## 🛠️ Tool Ecosystem

### Built-in Tools

| Tool | Implementation | Purpose |
|------|---------------|---------|
| `think_tool` | `utils.py:219-244` | Strategic reflection tool for research planning and decision-making |
| `ConductResearch` | `state.py:15-19` | Structured tool for research delegation to sub-agents |
| `ResearchComplete` | `state.py:21-22` | Signal tool to indicate research completion |

### Search Tools

| Tool | Implementation | Purpose | Constraints |
|------|---------------|---------|-------------|
| `tavily_search` | `utils.py:43-136` | Fetch and summarize search results from Tavily search API | Async with summarization |
| `web_search_20250305` | `utils.py:540-546` | Anthropic's native web search with usage limits | max_uses: 5 |
| `web_search_preview` | `utils.py:548-550` | OpenAI's web search preview functionality | - |

### MCP Tool Integration

**Client**: `MultiServerMCPClient`
- **Transport**: streamable_http
- **Configuration**: via `mcp_config.tools`
- **Authentication**: OAuth token exchange (`utils.py:352-383`)
- **Error Handling**: MCP-specific wrapper (`utils.py:385-447`)
- **Discovery**: Automatic from configured MCP servers

---

## 🔌 Integration Points

### Tool Loading Pattern
**Surface**: `utils.py:569-597`
```python
def get_all_tools():
    tools = [built_in_tools] + [search_tools] + [mcp_tools]
    # Name collision detection with warnings.warn()
```
- **Pattern**: `get_all_tools()`
- **Best Practice**: Config-driven injection with collision detection
- **Collision Handling**: `utils.py:510-514` with `warnings.warn()`

### MCP Integration Pattern
**Surface**: `utils.py:449-524`
- **Pattern**: `load_mcp_tools()`
- **Authentication**: OAuth token exchange via `fetch_tokens()`
- **Best Practice**: Auth wrapper + collision detection
- **Error Handling**: User-friendly MCP error messages

### Model Configuration Pattern
**Surface**: `configuration.py:236-247`
- **Pattern**: `Configuration.from_runnable_config()`
- **Multi-Model Support**: Different models for summarization, research, compression, final_report
- **Best Practice**: Environment variable fallbacks

### State Management Pattern
**Surface**: `state.py:55-60`
- **Pattern**: `override_reducer`
- **Semantics**: Supports both override (`{'type': 'override'}`) and additive updates
- **Best Practice**: Typed state with custom reducers

### Error Handling Pattern
**Surface**: `utils.py:665-785`
- **Pattern**: Provider-specific token limit detection
- **Providers**: OpenAI, Anthropic, Google/Gemini
- **Best Practice**: Progressive truncation with model token limits

---

## ⚙️ Configuration Schema

### Model Configuration
- `summarization_model`: For processing search results
- `research_model`: For conducting research
- `compression_model`: For synthesizing findings
- `final_report_model`: For generating final reports

### Operational Limits

| Configuration | Default | Maximum | Purpose |
|---------------|---------|---------|---------|
| `max_concurrent_research_units` | 5 | 20 | Parallel researcher limit |
| `max_researcher_iterations` | 6 | 10 | Supervisor loop limit |
| `max_react_tool_calls` | 10 | 30 | Researcher tool call limit |
| `max_structured_output_retries` | 3 | 10 | LLM retry limit |

### Search API Options
- **anthropic**: Native web search with usage limits
- **openai**: Web search preview functionality
- **tavily**: Full-featured search with summarization
- **none**: No search functionality

### MCP Configuration
- **url**: MCP server endpoint URL
- **tools**: List of tool names to enable from server
- **auth_required**: Boolean for OAuth requirement

---

## 🚀 Execution Flow Summary

1. **User Input** → `clarify_with_user` → (optional clarification) → `write_research_brief`
2. **Research Planning** → `research_supervisor` → `supervisor` loop with `ConductResearch` delegation
3. **Parallel Research** → Multiple `researcher` subgraphs executing concurrently with tool calls
4. **Research Synthesis** → `compress_research` consolidates findings from all researchers
5. **Report Generation** → `final_report_generation` creates comprehensive final output

**Key Characteristics**:
- **Parallel Execution**: Up to 20 concurrent researchers
- **Iterative Refinement**: Supervisor and researcher loops with configurable limits
- **Error Resilience**: Token limit detection and progressive truncation
- **Tool Extensibility**: MCP integration with collision detection
- **State Integrity**: Custom reducers with override and additive semantics

This architecture provides a robust, scalable foundation for automated research with sophisticated control flow, error handling, and extensibility mechanisms.