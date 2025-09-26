# 🔎 Code-Scoped Analysis Prompt — ODR (main/src/open_deep_research only)

**Role**: Act as a senior AI engineer with deep LangGraph/LangChain + MCP experience. Your job is to read code like a graph engineer and surface (1) existing capabilities & integration points and (2) preferred approaches to rapidly prototype MCP tools—**strictly from the code under** `https://github.com/langchain-ai/open_deep_research/tree/main/src/open_deep_research` on the **main branch**.

**Hard Scope Rule (must follow)**

* **Use only files inside** `src/open_deep_research/**` on **main**.
* **Do not rely on** README, examples, tests, issues, PRs, wiki, external websites, or any file outside that path.
* If the code imports external libs, you may infer intent **only from call sites and signatures in this codebase**. No external docs.

**Before You Begin**

1. Record repo **commit SHA** and **timestamp** for the version you analyze.
2. Enumerate the **file tree under** `src/open_deep_research/**` and list key modules you’ll read.
3. If any referenced file is **outside scope**, mark as *external dependency* and continue without it.

---

## Deliverables (in order)

1. **Executive Summary (8–12 bullets)**
   What the agent can do today, where the seams are, and what is configurable.

2. **Capability & Integration Matrix (table)**
   Columns: *Area* (LLM, search, tools, MCP, memory/state, prompts, observability, limits), *What exists* (file path + 1-line), *Integration seam* (how to extend without surgery), *Risks/constraints*, *Test hook* (how to validate).

3. **Graph Model of the System**

   * **Adjacency list**: `NodeA -> NodeB [condition=…]`
   * **Mermaid diagram** (high-level flow)
   * **State I/O map**: which nodes **read/write** which state keys; how messages and notes evolve.

4. **Graph Manifest (JSON)** — fill exactly:

```json
{
  "repo": {"url": "https://github.com/langchain-ai/open_deep_research", "path": "src/open_deep_research", "branch": "main", "commit": "<sha>", "analyzed_at": "<UTC ISO>"},
  "graph": {
    "state_keys": [{"name": "supervisor_messages", "type": "list[MessageLike]", "rw": "R/W", "notes": ""}],
    "nodes": [{"name": "researcher", "reads": ["researcher_messages"], "writes": ["compressed_research","raw_notes"], "file": "<relative_path>"}],
    "edges": [{"source": "supervisor", "target": "supervisor_tools", "condition": "tool_calls_present"}],
    "loops": [{"name": "research_loop", "exit_condition": "iterations >= limit || ResearchComplete"}]
  },
  "tools": {
    "builtin": [{"name": "think_tool", "file": "<relative_path>"}],
    "search": [{"name": "tavily_search|web_search", "file": "<relative_path>"}],
    "mcp": [{"client": "MultiServerMCPClient", "transport": "streamable_http", "tools": "<from config>", "file": "<relative_path>"}]
  },
  "integration_points": [
    {"surface": "tool_loading", "file": "<relative_path>", "pattern": "get_all_tools()/load_mcp_tools()", "best_practice": "config-driven injection"}
  ]
}
```

5. **MCP Rapid-Prototype Plan (YAML)** — least-surgery path:

```yaml
prototype:
  scope_rule: "Only modify files under src/open_deep_research; prefer config-based injection."
  goal: "Expose one new MCP tool and wire it into researcher flow without graph surgery."
  server:
    name: demo_tools
    transport: streamable_http
    auth: "optional|bearer"
    # illustrative; align with client code you locate:
    endpoint: "https://localhost:3000/mcp"
    tools:
      - name: classify_text
        input_schema: {type: object, properties: {text: {type: string}}, required: [text]}
        output_schema: {type: object, properties: {label: {type: string}}}
  client_wiring:
    config_surface: "<file:line> where MCPConfig/Configuration is read"
    discovery: "existing MultiServerMCPClient path"
    selection: "respect existing tool-name filter; avoid name collisions"
  prompts:
    edits:
      - file: "<prompts module>"
        change: "Mention availability and intended scope of `classify_text` so agent uses it correctly."
  acceptance:
    checks:
      - "Client discovers MCP tool at startup"
      - "Researcher invokes tool on suitable queries"
      - "State updates reflect tool outputs"
  rollout:
    dev: "local stdio/http"
    prod: "auth header via tokens store; retry/backoff"
```

6. **Minimal Code Diffs (inline blocks)**

   * **Config**: where to add/extend MCP server entry and tool allow-list.
   * **Tool load path**: how your tool clears the existing name-collision checks and gets wrapped (auth/error handling) if present.
   * **Prompts**: minimal edits to advertise tool affordances.

7. **Acceptance & Runbook (checklist + commands)**

   * How to set env/config, run the graph, and confirm end-to-end tool usage.
   * Show which **state keys** changed and **which messages** carried tool outputs.

---

## Method (how to read the code)

* **State first**: Identify **state models** and custom reducers; list keys and which nodes/edges mutate them.
* **Nodes next**: For each runnable/node function, capture: inputs, outputs, tool binding, retry/timeout, and exit conditions.
* **Tooling last**: Map search/MCP tool registration, name-collision logic, auth wrapping, and error paths.
* **Prompt surfaces**: Locate system/user prompts that govern supervisor/researcher behavior; note slots for MCP context.
* **Limits**: Extract iteration ceilings, token budgets, and concurrency caps; call out where token-limit recovery occurs.

**Evidence discipline**

* Quote **relative file paths** and **line spans** for key claims.
* If a symbol is referenced but not defined in-scope, tag it as *external dependency* (no guessing).
* If a config is read from env or `RunnableConfig`, specify the **exact key names** and where they’re consumed.

**Style**

* Deterministic: keep temperature low.
* Be concise but precise; prefer tables and manifests over prose.
* If something is ambiguous, add a **TODO/Question** section at the end.

---

## Output Format & Accuracy Requirements

**Analysis Output Structure:**
* Create structured documentation files following the pattern: `docs/odr-analysis-YYYY-MM-DD.md`, `docs/odr-graph-manifest-YYYY-MM-DD.json`, `docs/odr-mcp-prototype-YYYY-MM-DD.yaml`
* Use markdown formatting with clear section headers matching the deliverable structure above
* Include the full JSON manifest and YAML prototype plan as formatted code blocks
* Provide specific file paths and line numbers for all claims (e.g., `deep_researcher.py:700-719`)
* Preserve the analysis as permanent documentation rather than ephemeral conversation content

**Critical Accuracy Notes (apply these corrections to avoid common mistakes):**

1. **Tool name collisions**: MCP tools with naming conflicts are **not silently ignored** - they trigger `warnings.warn()` and are skipped with explicit warnings (`utils.py:510-514`)

2. **Web search metadata**: Native search tool descriptors are **static constants** (e.g., `"web_search_20250305"`, `"max_uses": 5`), not dynamic provider discovery (`utils.py:540-546`)

3. **Iteration counter ownership**: The `tool_call_iterations` counter is incremented by the `researcher` node (`deep_researcher.py:421-423`), not by `researcher_tools` - be precise about which node owns which state mutations

4. **Observability scope**: LangSmith integration uses **tags only** (`"langsmith:nostream"`) with no custom tracing/metrics hooks - avoid overstating observability capabilities (`deep_researcher.py:395-399, 530-531, 630-631`)

5. **State reducer behavior**: The `override_reducer` allows both replacement via `{"type": "override", "value": [...]}` and additive updates via direct list append - clearly distinguish these semantics (`state.py:55-60`)

**Evidence Discipline:**
* Quote **exact line spans** for all architectural claims
* Distinguish between **static tool descriptors** vs **dynamic tool discovery**
* Mark **external dependencies** explicitly when referenced but not defined in scope
* Verify **loop exit conditions** and **iteration ownership** precisely

---

**Notes to the model:**

* Your findings must be **entirely derivable from** the code in `src/open_deep_research/**` on **main**.
* Do **not** introduce behaviors or APIs not evidenced by this code path.
* Apply the accuracy corrections above to ensure analysis matches code reality.
* If requested to persist analysis, suggest structure: `docs/odr-analysis-YYYY-MM-DD.md`, `docs/odr-graph-manifest.json`, `docs/odr-mcp-prototype.yaml`

---

Relevant in-scope files you'll inspect in current main branch:  https://github.com/langchain-ai/open_deep_research/tree/main/src/open_deep_research

- utils.py
- state.py
- prompts.py
- deep_researcher.py
- configuration.py

---

