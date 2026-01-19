# CAAL-FORK Tool Calling Deep Dive: How It Achieves Accurate Function Calling

## Executive Summary

CAAL-FORK achieves accurate tool calling through a sophisticated **two-phase execution strategy** combined with **Ollama's native function calling** (which supports models like Llama 3.1 and Ministral). The key to accuracy lies in:

1. **Non-streaming initial LLM call** to detect and parse tool calls reliably
2. **Structured schema conversion** from multiple sources (MCP, n8n, decorators)
3. **Three-tier execution routing** with error handling
4. **Tool data caching** to provide context for follow-up calls
5. **Streaming follow-up response** for natural conversation flow

This document provides a detailed technical breakdown of how the project accomplishes accurate tool calling, which may be particularly relevant for your two-stage router using Llama 3.1 and Ministral.

---

## Table of Contents

1. [The Two-Phase Execution Strategy](#1-the-two-phase-execution-strategy)
2. [Tool Schema Generation & Accuracy](#2-tool-schema-generation--accuracy)
3. [Tool Discovery from Multiple Sources](#3-tool-discovery-from-multiple-sources)
4. [Tool Execution Routing](#4-tool-execution-routing)
5. [Tool Data Caching for Multi-Turn Accuracy](#5-tool-data-caching-for-multi-turn-accuracy)
6. [Error Handling & Recovery](#6-error-handling--recovery)
7. [Key Techniques for Accuracy](#7-key-techniques-for-accuracy)
8. [Comparison with Your Two-Stage Router](#8-comparison-with-your-two-stage-router)

---

## 1. The Two-Phase Execution Strategy

### Overview

CAAL-FORK's most critical technique for accurate tool calling is the **two-phase execution pattern**:

**Phase 1: Non-Streaming Detection**
- Initial LLM call with `stream=False`
- Waits for complete response before processing
- Parses tool calls from structured response
- Executes all tool calls sequentially

**Phase 2: Streaming Response**
- Follow-up LLM call with `stream=True`
- Tool results injected into context
- Natural conversational response streamed to user

### Implementation

**File:** [src/caal/llm/ollama_node.py:152-196](src/caal/llm/ollama_node.py#L152-L196)

```python
async def ollama_llm_node(agent, chat_ctx, model, think, temperature, ...):
    """Custom LLM node using Ollama directly."""

    # Build messages with tool data context
    messages = _build_messages_from_context(chat_ctx, tool_data_cache, max_turns)

    # Discover all available tools
    ollama_tools = await _discover_tools(agent)

    # PHASE 1: Non-streaming call to detect tool usage
    if ollama_tools:
        response = ollama.chat(
            model=model,
            messages=messages,
            tools=ollama_tools,      # Pass all tool schemas
            think=think,
            stream=False,            # ← KEY: Wait for complete response
            options=options,
        )

        # Check if LLM decided to call any tools
        if hasattr(response, "message"):
            if hasattr(response.message, "tool_calls") and response.message.tool_calls:
                tool_calls = response.message.tool_calls
                logger.info(f"Ollama returned {len(tool_calls)} tool call(s)")

                # Track which tools are being called (for UI feedback)
                tool_names = [tc.function.name for tc in tool_calls]
                tool_params = [tc.function.arguments for tc in tool_calls]

                # Publish tool status immediately to frontend
                if hasattr(agent, "_on_tool_status"):
                    asyncio.create_task(
                        agent._on_tool_status(True, tool_names, tool_params)
                    )

                # Execute tools and append results to message history
                messages = await _execute_tool_calls(
                    agent, messages, tool_calls, response.message,
                    tool_data_cache=tool_data_cache,
                )

                # PHASE 2: Stream follow-up response with tool results in context
                followup = ollama.chat(
                    model=model,
                    messages=messages,       # ← Includes tool results
                    think=think,
                    stream=True,             # ← Stream for natural UX
                    options=options,
                )

                for chunk in followup:
                    if hasattr(chunk, "message") and hasattr(chunk.message, "content"):
                        if chunk.message.content:
                            yield strip_markdown_for_tts(chunk.message.content)
                return

            # No tool calls - return content directly
            elif hasattr(response.message, "content") and response.message.content:
                yield strip_markdown_for_tts(response.message.content)
                return

    # No tools available - stream directly
    response_stream = ollama.chat(
        model=model,
        messages=messages,
        tools=None,
        think=think,
        stream=True,
        options=options,
    )

    for chunk in response_stream:
        if hasattr(chunk, "message") and hasattr(chunk.message, "content"):
            if chunk.message.content:
                yield strip_markdown_for_tts(chunk.message.content)
```

### Why This Approach Works

**1. Reliable Tool Detection**
- `stream=False` ensures complete response is received
- Structured `tool_calls` array is fully populated
- No partial parsing or streaming JSON issues

**2. Sequential Execution**
- Tools execute in order specified by LLM
- Dependencies between tools are respected
- Each tool result is captured before next execution

**3. Context Preservation**
- Tool results added to message history
- LLM sees exact tool outputs in follow-up call
- Natural language summary generated with full context

**4. Error Isolation**
- Tool execution errors caught individually
- Error messages passed back to LLM for recovery
- Follow-up response can explain failures gracefully

---

## 2. Tool Schema Generation & Accuracy

### The Critical Role of Schema Quality

Accurate tool calling depends heavily on **high-quality tool schemas**. The LLM needs:
- Clear tool names
- Descriptive function descriptions
- Precise parameter types
- Required vs optional parameter indicators

CAAL-FORK generates schemas from three sources, each with different conversion logic.

### A. @function_tool Decorated Methods

**File:** [src/caal/llm/ollama_node.py:323-362](src/caal/llm/ollama_node.py#L323-L362)

Python functions are introspected to generate schemas:

```python
async def _discover_tools(agent) -> list[dict] | None:
    """Discover tools from agent methods and MCP servers."""

    ollama_tools = []

    # Get @function_tool decorated methods from agent
    if hasattr(agent, "_tools") and agent._tools:
        for tool in agent._tools:
            if hasattr(tool, "__func__"):
                func = tool.__func__
                name = func.__name__                    # Function name
                description = func.__doc__ or ""        # Docstring as description
                sig = inspect.signature(func)           # Python signature
                properties = {}
                required = []

                # Introspect parameters
                for param_name, param in sig.parameters.items():
                    if param_name == "self":
                        continue

                    # Map Python types to JSON Schema types
                    param_type = "string"  # default
                    if param.annotation != inspect.Parameter.empty:
                        if param.annotation == str:
                            param_type = "string"
                        elif param.annotation == int:
                            param_type = "integer"
                        elif param.annotation == float:
                            param_type = "number"
                        elif param.annotation == bool:
                            param_type = "boolean"

                    properties[param_name] = {"type": param_type}

                    # Parameters without defaults are required
                    if param.default == inspect.Parameter.empty:
                        required.append(param_name)

                # Build Ollama tool schema
                ollama_tools.append({
                    "type": "function",
                    "function": {
                        "name": name,
                        "description": description,
                        "parameters": {
                            "type": "object",
                            "properties": properties,
                            "required": required,
                        },
                    },
                })
```

**Example Agent Method:**

```python
@function_tool
async def web_search(self, query: str) -> str:
    """Search the web for current events, news, prices, store hours,
    or any time-sensitive information not available from other tools.

    Args:
        query: What to search for on the web.
    """
    # Implementation...
```

**Generated Schema:**

```json
{
  "type": "function",
  "function": {
    "name": "web_search",
    "description": "Search the web for current events, news, prices, store hours,\nor any time-sensitive information not available from other tools.\n\nArgs:\n    query: What to search for on the web.",
    "parameters": {
      "type": "object",
      "properties": {
        "query": {"type": "string"}
      },
      "required": ["query"]
    }
  }
}
```

### B. MCP Server Tools

**File:** [src/caal/llm/ollama_node.py:392-429](src/caal/llm/ollama_node.py#L392-L429)

MCP tools already have JSON Schema definitions that are converted to Ollama format:

```python
async def _get_mcp_tools(mcp_server) -> list[dict]:
    """Get tools from an MCP server in Ollama format."""
    tools = []

    if not mcp_server or not hasattr(mcp_server, "_client"):
        return tools

    try:
        # Get tools via MCP protocol
        tools_result = await mcp_server._client.list_tools()

        if hasattr(tools_result, "tools"):
            for mcp_tool in tools_result.tools:
                # Convert MCP schema to Ollama format
                parameters = {"type": "object", "properties": {}, "required": []}

                if hasattr(mcp_tool, "inputSchema") and mcp_tool.inputSchema:
                    schema = mcp_tool.inputSchema

                    # Handle both dict and object schemas
                    if isinstance(schema, dict):
                        parameters = schema
                    elif hasattr(schema, "properties"):
                        parameters["properties"] = schema.properties or {}
                        parameters["required"] = getattr(schema, "required", []) or []

                tools.append({
                    "type": "function",
                    "function": {
                        "name": mcp_tool.name,
                        "description": getattr(mcp_tool, "description", "") or "",
                        "parameters": parameters,
                    },
                })

        if tools:
            tool_names = [t["function"]["name"] for t in tools]
            logger.info(f"Loaded {len(tools)} MCP tools: {', '.join(tool_names)}")

    except Exception as e:
        logger.warning(f"Error getting MCP tools: {e}")

    return tools
```

**Key Feature:** MCP tools are **prefixed with server name** to avoid collisions:

```python
# In _discover_tools()
for server_name, server in agent._caal_mcp_servers.items():
    if server_name == "n8n":
        continue  # n8n uses webhook discovery

    mcp_tools = await _get_mcp_tools(server)

    # Prefix tools with server name
    for tool in mcp_tools:
        original_name = tool["function"]["name"]
        tool["function"]["name"] = f"{server_name}__{original_name}"

    ollama_tools.extend(mcp_tools)
```

**Example:**
- Original: `turn_on_light`
- Prefixed: `home_assistant__turn_on_light`

### C. n8n Workflow Tools

**File:** [src/caal/integrations/n8n.py:24-111](src/caal/integrations/n8n.py#L24-L111)

n8n workflows are discovered via MCP but use **flexible schemas**:

```python
async def discover_n8n_workflows(n8n_mcp, base_url: str):
    """Discover n8n workflows and create tool definitions.

    Convention: All workflows use webhook triggers with webhook path = workflow name.
    Workflow descriptions extracted from webhook node notes.
    """
    tools = []
    workflow_name_map = {}

    # Get list of active workflows via MCP
    result = await n8n_mcp._client.call_tool("search_workflows", {})
    workflows = parse_mcp_result(result)

    if not workflows or "data" not in workflows:
        return tools, workflow_name_map

    for workflow in workflows["data"]:
        wf_name = workflow["name"]
        wf_id = workflow["id"]

        # Get detailed description from webhook notes
        details_result = await n8n_mcp._client.call_tool(
            "get_workflow_details",
            {"workflowId": wf_id}
        )
        workflow_details = parse_mcp_result(details_result)
        description = extract_webhook_description(workflow_details)

        # Sanitize workflow name for use as function name
        sanitized_name = sanitize_tool_name(wf_name)
        workflow_name_map[sanitized_name] = wf_name

        # Create Ollama tool definition with FLEXIBLE schema
        tool = {
            "type": "function",
            "function": {
                "name": sanitized_name,
                "description": description or f"Execute workflow: {wf_name}",
                "parameters": {
                    "type": "object",
                    "additionalProperties": True  # ← KEY: Accept any parameters
                }
            }
        }
        tools.append(tool)

    return tools, workflow_name_map
```

**Why Flexible Schema?**

n8n workflows are highly dynamic and can accept arbitrary JSON payloads. Rather than trying to infer strict schemas, CAAL relies on:

1. **Rich descriptions** in webhook node `notes` field
2. **LLM reasoning** about appropriate parameters
3. **Flexible acceptance** via `additionalProperties: True`

**Example n8n Tool Schema:**

```json
{
  "type": "function",
  "function": {
    "name": "espn_get_nfl_scores",
    "description": "Get live NFL scores and standings. Optional parameters: team (e.g., 'Packers', 'Bears'). Returns current scores, game status, and matchups.",
    "parameters": {
      "type": "object",
      "additionalProperties": true
    }
  }
}
```

The LLM reads the description and infers that it can pass `{"team": "Packers"}`.

### Schema Quality Best Practices

**From CAAL's Implementation:**

1. **Use detailed docstrings** - They become tool descriptions
2. **Type annotate parameters** - Enables accurate type mapping
3. **Mark required parameters** - No defaults = required
4. **Prefix multi-server tools** - Avoid name collisions
5. **Use flexible schemas for dynamic endpoints** - Trust LLM reasoning
6. **Extract descriptions from metadata** - n8n webhook notes, MCP descriptions

---

## 3. Tool Discovery from Multiple Sources

### Unified Tool Discovery

**File:** [src/caal/llm/ollama_node.py:311-389](src/caal/llm/ollama_node.py#L311-L389)

```python
async def _discover_tools(agent) -> list[dict] | None:
    """Discover tools from agent methods and MCP servers.

    Tools are cached on the agent instance after first discovery to avoid
    redundant MCP API calls on every user utterance.
    """
    # Return cached tools if available
    if hasattr(agent, "_ollama_tools_cache") and agent._ollama_tools_cache is not None:
        return agent._ollama_tools_cache

    ollama_tools = []

    # SOURCE 1: @function_tool decorated methods
    if hasattr(agent, "_tools") and agent._tools:
        for tool in agent._tools:
            # Convert Python function to Ollama schema (see Section 2A)
            ollama_tools.append(...)

    # SOURCE 2: MCP servers (except n8n)
    if hasattr(agent, "_caal_mcp_servers") and agent._caal_mcp_servers:
        for server_name, server in agent._caal_mcp_servers.items():
            if server_name == "n8n":
                continue  # n8n uses webhook discovery

            mcp_tools = await _get_mcp_tools(server)

            # Prefix tools with server name to avoid collisions
            for tool in mcp_tools:
                original_name = tool["function"]["name"]
                tool["function"]["name"] = f"{server_name}__{original_name}"

            ollama_tools.extend(mcp_tools)
            logger.info(f"Added {len(mcp_tools)} tools from MCP server: {server_name}")

    # SOURCE 3: n8n workflow tools (webhook-based)
    if hasattr(agent, "_n8n_workflow_tools") and agent._n8n_workflow_tools:
        ollama_tools.extend(agent._n8n_workflow_tools)

    # Cache tools on agent instance
    result = ollama_tools if ollama_tools else None
    agent._ollama_tools_cache = result

    return result
```

### Tool Caching Strategy

**Key Insight:** Tool discovery happens **once per agent session**, then cached.

**Why Cache?**
- Avoids redundant MCP API calls
- Reduces latency on every user message
- Tools rarely change during a conversation

**Cache Invalidation:**
- Manual via `/reload-tools` webhook endpoint
- Automatic on agent restart

**File:** [src/caal/webhooks.py:147-180](src/caal/webhooks.py#L147-L180)

```python
@app.post("/reload-tools", response_model=ReloadToolsResponse)
async def reload_tools(req: ReloadToolsRequest) -> ReloadToolsResponse:
    """Reload tool cache for an active session."""
    result = session_registry.get(req.room_name)
    if not result:
        raise HTTPException(status_code=404, detail="No active session")

    session, agent = result

    # Clear tool cache
    if hasattr(agent, "_ollama_tools_cache"):
        agent._ollama_tools_cache = None

    # Optionally announce the reload
    if req.message:
        await session.say(req.message)

    return ReloadToolsResponse(
        status="reloaded",
        room_name=req.room_name,
        tool_name=req.tool_name
    )
```

---

## 4. Tool Execution Routing

### Three-Tier Routing System

**File:** [src/caal/llm/ollama_node.py:500-545](src/caal/llm/ollama_node.py#L500-L545)

```python
async def _execute_single_tool(agent, tool_name: str, arguments: dict) -> Any:
    """Execute a single tool call.

    Routing priority:
    1. Agent methods (@function_tool decorated)
    2. n8n workflows (webhook-based execution)
    3. MCP servers (with server_name__tool_name prefix parsing)
    """

    # TIER 1: Check if it's an agent method
    if hasattr(agent, tool_name) and callable(getattr(agent, tool_name)):
        logger.info(f"Calling agent tool: {tool_name}")
        result = await getattr(agent, tool_name)(**arguments)
        logger.info(f"Agent tool {tool_name} completed")
        return result

    # TIER 2: Check if it's an n8n workflow
    if (
        hasattr(agent, "_n8n_workflow_name_map")
        and tool_name in agent._n8n_workflow_name_map
        and hasattr(agent, "_n8n_base_url")
        and agent._n8n_base_url
    ):
        logger.info(f"Calling n8n workflow: {tool_name}")
        workflow_name = agent._n8n_workflow_name_map[tool_name]
        result = await execute_n8n_workflow(
            agent._n8n_base_url, workflow_name, arguments
        )
        logger.info(f"n8n workflow {tool_name} completed")
        return result

    # TIER 3: Check MCP servers (with multi-server routing)
    if hasattr(agent, "_caal_mcp_servers") and agent._caal_mcp_servers:
        # Parse server name from prefixed tool name
        # Format: server_name__actual_tool (double underscore separator)
        if "__" in tool_name:
            server_name, actual_tool = tool_name.split("__", 1)
        else:
            # Unprefixed tools default to n8n server
            server_name, actual_tool = "n8n", tool_name

        if server_name in agent._caal_mcp_servers:
            server = agent._caal_mcp_servers[server_name]
            result = await _call_mcp_tool(server, actual_tool, arguments)
            if result is not None:
                return result

    # Tool not found in any tier
    raise ValueError(f"Tool {tool_name} not found")
```

### Routing Logic Explanation

**Priority Order:**

1. **Agent Methods** - Fastest, in-process execution
2. **n8n Workflows** - HTTP webhooks, flexible automation
3. **MCP Servers** - Protocol-based, multi-server support

**Why This Order?**

- **Performance:** Agent methods have zero network overhead
- **Flexibility:** n8n workflows handle complex automation
- **Extensibility:** MCP supports unlimited external services

### Multi-Server MCP Routing

**Tool Name Format:** `server_name__actual_tool`

**Example:**
- `home_assistant__turn_on_light` → Routes to `home_assistant` server, calls `turn_on_light`
- `filesystem__read_file` → Routes to `filesystem` server, calls `read_file`

**Configuration:** [mcp_servers.json.example](mcp_servers.json.example)

```json
{
  "home_assistant": {
    "command": "mcp-server-home-assistant",
    "args": ["--url", "http://homeassistant.local:8123"],
    "env": {
      "HA_TOKEN": "your_token_here"
    }
  },
  "filesystem": {
    "command": "mcp-server-filesystem",
    "args": ["/home/user/documents"]
  }
}
```

---

## 5. Tool Data Caching for Multi-Turn Accuracy

### The Problem

When tools return structured data (IDs, arrays, objects), the LLM needs that data for follow-up questions:

**Example Conversation:**

```
User: "Get the NFL scores"
Tool: Returns JSON with game IDs, teams, scores

User: "Tell me more about the Packers game"
LLM: Needs to reference game ID from previous tool call
```

**Challenge:** Chat history has a sliding window (default 20 turns), so old tool results may be trimmed.

### The Solution: ToolDataCache

**File:** [src/caal/llm/ollama_node.py:36-67](src/caal/llm/ollama_node.py#L36-L67)

```python
class ToolDataCache:
    """Caches recent tool response data for context injection.

    Tool responses often contain structured data (IDs, arrays) that the LLM
    needs for follow-up calls. This cache preserves that data separately
    from chat history and injects it into context on each LLM call.
    """

    def __init__(self, max_entries: int = 3):
        self.max_entries = max_entries
        self._cache: list[dict] = []

    def add(self, tool_name: str, data: Any) -> None:
        """Add tool response data to cache."""
        entry = {"tool": tool_name, "data": data, "timestamp": time.time()}
        self._cache.append(entry)
        if len(self._cache) > self.max_entries:
            self._cache.pop(0)  # Remove oldest

    def get_context_message(self) -> str | None:
        """Format cached data as context string for LLM injection."""
        if not self._cache:
            return None
        parts = ["Recent tool response data for reference:"]
        for entry in self._cache:
            parts.append(f"\n{entry['tool']}: {json.dumps(entry['data'])}")
        return "\n".join(parts)

    def clear(self) -> None:
        """Clear the cache."""
        self._cache.clear()
```

### Cache Population

**File:** [src/caal/llm/ollama_node.py:476-481](src/caal/llm/ollama_node.py#L476-L481)

```python
async def _execute_tool_calls(agent, messages, tool_calls, response_message,
                               tool_data_cache=None):
    """Execute tool calls and append results to messages."""

    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        arguments = tool_call.function.arguments or {}

        try:
            tool_result = await _execute_single_tool(agent, tool_name, arguments)

            # Cache structured data if present
            if tool_data_cache and isinstance(tool_result, dict):
                # Look for common data fields, otherwise cache the whole result
                data = tool_result.get("data") or tool_result.get("results") or tool_result
                tool_data_cache.add(tool_name, data)
                logger.debug(f"Cached tool data for {tool_name}")

            messages.append({
                "role": "tool",
                "content": str(tool_result),
                "tool_call_id": getattr(tool_call, "id", None),
            })
        except Exception as e:
            error_msg = f"Error executing tool {tool_name}: {e}"
            logger.error(error_msg, exc_info=True)
            messages.append({
                "role": "tool",
                "content": error_msg,
                "tool_call_id": getattr(tool_call, "id", None),
            })

    return messages
```

### Cache Injection

**File:** [src/caal/llm/ollama_node.py:232-308](src/caal/llm/ollama_node.py#L232-L308)

```python
def _build_messages_from_context(chat_ctx, tool_data_cache=None, max_turns=20):
    """Build Ollama messages with sliding window and tool data context.

    Message order:
    1. System prompt (always first, never trimmed)
    2. Tool data context (injected from cache)
    3. Chat history (sliding window applied)
    """
    system_prompt = None
    chat_messages = []

    # Extract system prompt and chat messages
    for msg in chat_ctx.messages:
        role = msg.role
        content = msg.content

        if role == "system":
            system_prompt = {"role": "system", "content": content}
        else:
            chat_messages.append({"role": role, "content": content})

    # Build final message list
    messages = []

    # 1. System prompt (always first)
    if system_prompt:
        messages.append(system_prompt)

    # 2. Inject tool data context as system message
    if tool_data_cache:
        context = tool_data_cache.get_context_message()
        if context:
            messages.append({"role": "system", "content": context})

    # 3. Apply sliding window to chat history
    # max_turns * 2 accounts for user + assistant pairs
    max_messages = max_turns * 2
    if len(chat_messages) > max_messages:
        trimmed = len(chat_messages) - max_messages
        chat_messages = chat_messages[-max_messages:]
        logger.debug(f"Sliding window: trimmed {trimmed} old messages")

    messages.extend(chat_messages)
    return messages
```

### Example Injected Context

```
System: You are a helpful voice assistant...

System: Recent tool response data for reference:
espn_get_nfl_scores: {"games": [{"id": "401547432", "home": "Packers", "away": "Bears", "homeScore": 24, "awayScore": 17}]}

User: Tell me more about the Packers game
```

The LLM can now reference game ID `401547432` even if the original tool call was trimmed from chat history.

---

## 6. Error Handling & Recovery

### Tool Execution Error Handling

**File:** [src/caal/llm/ollama_node.py:488-495](src/caal/llm/ollama_node.py#L488-L495)

```python
try:
    tool_result = await _execute_single_tool(agent, tool_name, arguments)

    # Cache and append success message
    messages.append({
        "role": "tool",
        "content": str(tool_result),
        "tool_call_id": tool_call.id,
    })
except Exception as e:
    # Catch errors and pass back to LLM for recovery
    error_msg = f"Error executing tool {tool_name}: {e}"
    logger.error(error_msg, exc_info=True)

    messages.append({
        "role": "tool",
        "content": error_msg,  # ← LLM sees the error
        "tool_call_id": tool_call.id,
    })
```

**Key Insight:** Errors are **passed back to the LLM** as tool responses, allowing it to:
- Explain the error to the user
- Suggest alternatives
- Retry with corrected parameters

### MCP Tool Error Handling

**File:** [src/caal/llm/ollama_node.py:548-582](src/caal/llm/ollama_node.py#L548-L582)

```python
async def _call_mcp_tool(mcp_server, tool_name: str, arguments: dict) -> Any | None:
    """Call a tool on an MCP server."""

    if not mcp_server or not hasattr(mcp_server, "_client"):
        return None

    try:
        logger.info(f"Calling MCP tool: {tool_name}")
        result = await mcp_server._client.call_tool(tool_name, arguments)

        # Check for MCP protocol errors
        if result.isError:
            text_contents = []
            for content in result.content:
                if hasattr(content, "text") and content.text:
                    text_contents.append(content.text)
            error_msg = f"MCP tool {tool_name} error: {text_contents}"
            logger.error(error_msg)
            return error_msg  # Return error as tool result

        # Extract text content from MCP response
        text_contents = []
        for content in result.content:
            if hasattr(content, "text") and content.text:
                text_contents.append(content.text)

        return "\n".join(text_contents) if text_contents else "Tool executed successfully"

    except Exception as e:
        logger.warning(f"Error calling MCP tool {tool_name}: {e}")
        return None
```

### Top-Level Error Handling

**File:** [src/caal/llm/ollama_node.py:227-229](src/caal/llm/ollama_node.py#L227-L229)

```python
except Exception as e:
    logger.error(f"Error in ollama_llm_node: {e}", exc_info=True)
    yield f"I encountered an error: {e}"
```

Even catastrophic failures result in spoken error messages to the user.

---

## 7. Key Techniques for Accuracy

### Summary of Accuracy Techniques

| Technique | Purpose | Implementation |
|-----------|---------|----------------|
| **Non-streaming initial call** | Reliable tool call detection | `stream=False` in Phase 1 |
| **Structured schema generation** | Clear tool definitions for LLM | Introspection, MCP conversion |
| **Tool name prefixing** | Avoid collisions across servers | `server_name__tool_name` |
| **Tool data caching** | Multi-turn context preservation | ToolDataCache class |
| **Sliding window** | Manage conversation context | `max_turns` parameter |
| **Error propagation** | LLM-driven error recovery | Errors as tool responses |
| **Tool caching** | Consistent tool availability | Cache on agent instance |
| **Flexible schemas** | Handle dynamic endpoints | `additionalProperties: true` |
| **Rich descriptions** | LLM reasoning about parameters | Docstrings, webhook notes |
| **Sequential execution** | Tool dependency handling | Await each tool before next |

### Best Practices Derived from CAAL

1. **Always use non-streaming for tool detection**
   - Streaming JSON parsing is error-prone
   - Complete response ensures all tool calls are captured

2. **Generate high-quality schemas**
   - Detailed descriptions help LLM select correct tool
   - Accurate type mapping reduces parameter errors
   - Clear required/optional distinction

3. **Implement tool result caching**
   - Prevents context loss in long conversations
   - Enables accurate follow-up questions
   - Separates structured data from conversation history

4. **Use namespacing for multi-source tools**
   - Prevents name collisions
   - Makes tool source explicit
   - Enables routing based on prefix

5. **Pass errors back to LLM**
   - Enables graceful error explanations
   - Allows LLM to suggest corrections
   - Better UX than hard failures

6. **Cache tool schemas**
   - Reduces discovery overhead
   - Ensures consistency within session
   - Improves latency

---

## 8. Comparison with Your Two-Stage Router

### Your Approach: Two-Stage Router with Llama 3.1 & Ministral

Based on your description, you use:
- **Stage 1:** Verify user request through first LLM
- **Stage 2:** Verify through second LLM (different model)

This is a **validation-based approach** to ensure accuracy.

### CAAL's Approach: Two-Phase Execution

CAAL uses:
- **Phase 1:** Non-streaming LLM call to detect tools
- **Phase 2:** Streaming LLM call to generate response

This is an **execution-based approach** optimized for UX.

### Comparison Table

| Aspect | Your Two-Stage Router | CAAL Two-Phase Execution |
|--------|----------------------|--------------------------|
| **Purpose** | Verification of user intent | Tool detection + response generation |
| **Stage 1** | Llama 3.1 verifies request | Ollama (any model) with `stream=False` detects tools |
| **Stage 2** | Ministral verifies request | Ollama (same model) with `stream=True` generates response |
| **Model Selection** | Two different models | Same model, different streaming modes |
| **Tool Execution** | ? (after verification) | Between Phase 1 and Phase 2 |
| **Output** | Verified intent/parameters | Natural language response to user |
| **Latency** | Two full LLM calls before action | One call for detection, streaming for response |

### Potential Hybrid Approach

You could combine both techniques:

```
Stage 1 (Llama 3.1): Verify user intent
    ↓
Stage 2 (Ministral): Verify parameters
    ↓
Phase 1 (Ollama non-streaming): Detect tool calls
    ↓
Execute Tools
    ↓
Phase 2 (Ollama streaming): Generate response
```

**Benefits:**
- **Verification** from your two-stage router
- **Reliable tool detection** from CAAL's non-streaming approach
- **Natural UX** from streaming follow-up

**Trade-off:** Higher latency (4 LLM calls total)

### Recommendations for Your Use Case

**If your primary concern is accuracy:**
1. **Adopt CAAL's non-streaming detection** in your Stage 2
   - After Ministral verifies parameters
   - Use `stream=False` for reliable tool call parsing
   - Execute tools sequentially

2. **Implement tool data caching** for multi-turn accuracy
   - Preserve structured data from tool responses
   - Inject as context in subsequent calls
   - Prevents loss of critical IDs/references

3. **Use high-quality tool schemas**
   - Detailed descriptions reduce ambiguity
   - Accurate type mapping reduces errors
   - Clear required/optional parameters

4. **Pass tool errors back to LLM**
   - Let LLM explain failures gracefully
   - Enable retry with corrected parameters
   - Better UX than hard failures

**If latency is also a concern:**
- Consider **single-model** approach (like CAAL)
- Use **tool result verification** instead of pre-verification
- Implement **confidence scoring** from tool call responses

---

## Conclusion

CAAL-FORK achieves accurate tool calling through:

1. **Two-Phase Execution Strategy**
   - Non-streaming for reliable detection
   - Streaming for natural UX

2. **High-Quality Schema Generation**
   - From Python introspection
   - From MCP protocol
   - From n8n workflow metadata

3. **Intelligent Tool Routing**
   - Three-tier priority system
   - Multi-server MCP support
   - Flexible webhook integration

4. **Context Preservation**
   - Tool data caching
   - Sliding window for chat history
   - Structured data injection

5. **Graceful Error Handling**
   - Errors passed to LLM
   - Individual tool error isolation
   - User-friendly explanations

This architecture enables reliable tool calling with **Ollama models like Llama 3.1 and Ministral**, which you're also using in your two-stage router. The key insight is that **non-streaming initial calls** dramatically improve tool detection reliability, while **tool data caching** enables accurate multi-turn conversations.

---

## Additional Resources

### Key Files to Study

| File | Focus Area |
|------|------------|
| [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py) | Core tool calling logic |
| [src/caal/integrations/n8n.py](src/caal/integrations/n8n.py) | Flexible schema approach |
| [src/caal/integrations/mcp_loader.py](src/caal/integrations/mcp_loader.py) | Multi-server configuration |
| [src/caal/settings.py](src/caal/settings.py) | Runtime configuration |
| [prompt/default.md](prompt/default.md) | Tool usage instructions for LLM |

### Configuration Examples

- [.env.example](.env.example) - Ollama configuration
- [mcp_servers.json.example](mcp_servers.json.example) - MCP server setup
- [n8n-workflows/](n8n-workflows/) - Example workflow definitions

---

**Analysis Date:** 2026-01-05
**Analyzer:** Claude Sonnet 4.5 (via Claude Code)
