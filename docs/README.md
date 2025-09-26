# Open Deep Research - Technical Documentation

This directory contains comprehensive technical analysis and integration documentation for the Open Deep Research system.

## 📋 Documentation Index

### 🔎 [Code-Scoped Analysis](odr-analysis-2025-01-25.md)
**Complete architectural analysis of the LangGraph-based research workflow**

- Executive summary of the 8-node, 3-subgraph architecture
- Capability & integration matrix for all system components
- Graph model with adjacency lists and Mermaid diagrams
- State I/O mapping with precise line references
- Tool integration analysis (search, MCP, built-in tools)
- Error handling & token limit detection patterns
- Minimal code diffs for MCP prototype integration
- Production deployment runbook with validation checklist

### 📊 [Graph Manifest](odr-graph-manifest.json)
**Machine-readable graph structure and metadata**

- Complete node and edge definitions with line references
- State key specifications with reducer types
- Tool integration patterns and collision handling
- Configuration field mappings and limits
- Loop detection and exit condition logic

### 🛠️ [MCP Prototype Plan](odr-mcp-prototype.yaml)
**Step-by-step framework for integrating new MCP tools**

- Minimal code surgery approach using existing patterns
- Authentication and error handling workflows
- Development to production deployment pipeline
- Validation checklist and monitoring setup
- Example usage patterns and scaling considerations

## 🎯 Quick Start Guide

### For Developers
1. **Understanding the Architecture**: Start with [odr-analysis-2025-01-25.md](odr-analysis-2025-01-25.md) sections 1-3
2. **Integration Points**: Review the capability matrix in section 2
3. **Adding Tools**: Follow [odr-mcp-prototype.yaml](odr-mcp-prototype.yaml) for MCP tool integration

### For System Architects
1. **Graph Structure**: Review [odr-graph-manifest.json](odr-graph-manifest.json) for complete system topology
2. **State Management**: Focus on state_keys and reducers for data flow understanding
3. **Scaling Patterns**: Review concurrency and iteration management patterns

### For DevOps/Platform Engineers
1. **Deployment**: Section 7 of [odr-analysis-2025-01-25.md](odr-analysis-2025-01-25.md)
2. **Configuration**: Environment variables and runtime config patterns
3. **Monitoring**: LangSmith integration and observability setup

## 🏗️ Architecture Overview

The Open Deep Research system is built on **LangGraph** with a sophisticated parallel architecture:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   User Input    │ -> │  Clarification   │ -> │ Research Brief  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         v
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Final Report   │ <- │   Supervisor     │ -> │ Parallel        │
│   Generation    │    │   Subgraph       │    │ Researchers     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

### Key Components
- **8 Core Nodes** across main workflow and subgraphs
- **Parallel Research Units** (1-20 configurable concurrent researchers)
- **Multi-Model Support** (different LLMs for each research phase)
- **Extensible Tool Integration** (search APIs, MCP servers, native tools)

## 🔧 Integration Patterns

### Tool Loading Pattern
```python
# utils.py:569-597
tools = [research_tools] + search_tools + mcp_tools
# Collision detection with warnings for conflicts
```

### State Management Pattern
```python
# state.py:55-60
def override_reducer(current, new):
    if new.get("type") == "override":
        return new.get("value")  # Complete replacement
    else:
        return current + new      # Additive update
```

### MCP Integration Pattern
```python
# utils.py:449-524
client = MultiServerMCPClient(server_config)
tools = await client.get_tools()
wrapped_tools = [wrap_mcp_authenticate_tool(t) for t in tools]
```

## 🎛️ Configuration Reference

### Model Configuration
- `summarization_model`: Default `"openai:gpt-4.1-mini"`
- `research_model`: Default `"openai:gpt-4.1"`
- `compression_model`: Default `"openai:gpt-4.1"`
- `final_report_model`: Default `"openai:gpt-4.1"`

### Research Limits
- `max_concurrent_research_units`: 1-20 (default: 5)
- `max_researcher_iterations`: 1-10 (default: 6)
- `max_react_tool_calls`: 1-30 (default: 10)

### Search APIs
- **Tavily**: Full-featured search with summarization
- **OpenAI Native**: Web search preview functionality
- **Anthropic Native**: Web search with usage limits (max_uses: 5)

## 📝 Usage Examples

### Basic Research Query
```bash
# Via LangGraph Studio
curl -X POST http://127.0.0.1:2024/assistants/deep_researcher/invoke \
  -H "Content-Type: application/json" \
  -d '{"input": {"messages": [{"role": "human", "content": "Research the latest developments in quantum computing applications"}]}}'
```

### MCP Tool Integration
```python
# Add new tool to configuration
mcp_config = {
    "url": "https://your-mcp-server.com/mcp",
    "tools": ["classify_text", "sentiment_analysis"],
    "auth_required": True
}
```

## 🚀 Development Workflow

### Local Development
```bash
# Start development server
uvx langgraph dev --allow-blocking

# Access LangGraph Studio
open https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

### Code Quality
```bash
# Linting and type checking
ruff check && mypy
```

### Evaluation
```bash
# Run Deep Research Bench evaluation
python tests/run_evaluate.py
```

## 📈 Performance Notes

- **Token Limits**: Progressive truncation with provider-specific detection
- **Concurrency**: Parallel research units with configurable limits
- **Error Recovery**: Comprehensive retry logic with exponential backoff
- **Caching**: Tool output caching for expensive operations

## 🔒 Security Considerations

- **API Keys**: Environment-based or config-based key management
- **MCP Authentication**: OAuth token exchange with Supabase integration
- **Input Validation**: Tool schema enforcement
- **Network Security**: HTTPS required for production MCP servers

## 📊 Monitoring & Observability

### LangSmith Integration
- Automatic experiment tracking with tags
- Performance metrics and cost analysis
- Error pattern analysis and debugging

### Key Metrics to Monitor
- Research completion rates
- Token usage per research session
- Tool execution success rates
- MCP server response times

## 🤝 Contributing

When extending the Open Deep Research system:

1. **Follow Existing Patterns**: Use the integration seams documented in the analysis
2. **Maintain Type Safety**: All state updates should use typed reducers
3. **Handle Errors Gracefully**: Implement provider-specific error detection
4. **Document Integration Points**: Update this documentation for new patterns

## 📚 Related Documentation

- **Main README**: [../README.md](../README.md) - Project overview and quickstart
- **Legacy Documentation**: [../src/legacy/legacy.md](../src/legacy/legacy.md) - Alternative implementations
- **Configuration Schema**: [../src/open_deep_research/configuration.py](../src/open_deep_research/configuration.py) - Complete config reference

---

**Last Updated**: January 25, 2025
**Analysis Version**: ODR Code-Scoped Analysis v1.0
**Commit**: b419df8d33b4f39ff5b2a34527bb6b85d0ede5d0