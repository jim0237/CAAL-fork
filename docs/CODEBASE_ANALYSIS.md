# CAAL-FORK Codebase Analysis

## Quick Summary

**User Request:**
> Review the CAAL-FORK virtual assistant codebase focusing on:
> 1. Local LLM routing implementation
> 2. Specific prompts used
> 3. Tool calling technique
> 4. Schema hardening with Pydantic
> 5. Usage of Langchain, Langfuse, or Langgraph

**Key Findings:**

**1. Local LLM Routing:**
- ❌ No multi-provider routing - uses **Ollama exclusively**
- ✅ Model selection via configuration (`settings.json` or `.env`)
- ✅ Has tool execution routing instead (3-tier priority system)

**2. Specific Prompts:**
- ✅ **Yes** - Markdown-based template at [prompt/default.md](prompt/default.md)
- Includes tool priority guidelines, voice formatting rules, behavioral instructions
- Supports dynamic context injection like `{{CURRENT_DATE_CONTEXT}}`

**3. Tool Calling Technique:**
- ✅ **Hybrid multi-source system**:
  - Ollama native function calling (primary)
  - MCP (Model Context Protocol) servers
  - n8n webhook-based workflows
  - LiveKit @function_tool decorators
- Innovative tool data caching separate from chat history
- Two-phase approach: non-streaming detection + streaming response

**4. Schema Hardening with Pydantic:**
- ✅ **Yes** - Uses **Pydantic v2.12.4** with FastAPI
- 11 BaseModel classes in [src/caal/webhooks.py](src/caal/webhooks.py)
- Additional whitelist validation for settings
- All webhook API endpoints are schema-validated

**5. Framework Usage:**
- ❌ **Does NOT use Langchain, Langfuse, or Langgraph**
- ✅ Primary: **LiveKit Agents** for voice orchestration
- ✅ Also: Ollama Python SDK, FastAPI, DuckDuckGo search

---

## Executive Summary

CAAL-FORK is a voice-activated assistant built on **LiveKit Agents** framework with **Ollama** as the exclusive LLM backend. It does NOT use Langchain, Langfuse, or Langgraph. The project implements schema validation via **Pydantic v2.12.4** integrated with FastAPI, and uses a sophisticated hybrid tool calling system combining native Ollama function calling, MCP (Model Context Protocol), and n8n webhook workflows.

---

## Table of Contents

1. [Local LLM Routing](#1-local-llm-routing)
2. [Specific Prompts Used](#2-specific-prompts-used)
3. [Tool Calling Technique](#3-tool-calling-technique)
4. [Schema Hardening & Pydantic](#4-schema-hardening--pydantic)
5. [Framework Usage](#5-framework-usage)
6. [Project Architecture](#project-architecture)
7. [Critical Files Reference](#critical-files-reference)

---

## 1. Local LLM Routing

### Answer: No Traditional LLM Provider Routing

**CAAL uses a single-provider architecture** locked to Ollama. There is no multi-provider routing system (e.g., no switching between OpenAI, Anthropic, Ollama, etc.).

### What Exists Instead:

#### Model Selection Configuration

**File:** [src/caal/settings.py](src/caal/settings.py)

**Configuration Sources:**
- `settings.json` with fallback to `.env`
- Runtime configurable via webhook API

**Configurable Parameters:**
- `model`: Which Ollama model to use (e.g., "ministral-3:8b")
- `temperature`: 0.7 (default)
- `num_ctx`: Context window size (8192 default)
- `max_turns`: Conversation sliding window (20 turns default)

**Environment Variables (.env.example):**
```env
OLLAMA_HOST=http://192.168.1.100:11434
OLLAMA_MODEL=ministral-3:8b
OLLAMA_THINK=false  # Qwen3-specific thinking mode
OLLAMA_TEMPERATURE=0.7
OLLAMA_NUM_CTX=8192
```

### Tool Routing (Not LLM Routing)

The project implements **tool execution routing** with priority order:

**File:** [src/caal/llm/ollama_node.py:500-546](src/caal/llm/ollama_node.py#L500-L546)

**Priority System:**
```
1. Agent methods (@function_tool decorated)
2. n8n workflows (webhook-based execution)
3. MCP servers (with server_name__tool_name prefix)
```

**MCP Multi-Server Support:**
- Tools prefixed with `server_name__tool_name` format
- Example: `home_assistant__turn_on_light`
- Configured in `mcp_servers.json`

### Architecture Pattern

CAAL overrides LiveKit's LLM abstraction to make **direct Ollama API calls**:

**Why:**
- Lower latency (avoid LiveKit's LLM wrapper overhead)
- Custom parameters support (like `think=False` for Qwen3)
- Tool data caching implementation
- Sliding window context management

**Implementation:** Custom `llm_node()` method in VoiceAssistant class

**File:** [src/caal/llm/ollama_node.py:153-225](src/caal/llm/ollama_node.py#L153-L225)

---

## 2. Specific Prompts Used

### Location
**File:** [prompt/default.md](prompt/default.md)

### Prompt Structure

The system uses a **markdown-based prompt template** with dynamic context injection:

#### 1. Role Definition
"You are a helpful, conversational voice assistant"

#### 2. Tool Priority System
- **Tools first** - Device control, workflows, environment queries
- **Web search second** - Current events, time-sensitive data
- **General knowledge last** - Static facts only

#### 3. Voice Output Guidelines
- Plain text only (no markdown/symbols)
- Number/date/time formatting for TTS
- Brevity (1-2 sentences)
- Conversational tone with contractions

#### 4. Behavioral Rules
- Always call tools for actions
- Offer to create new tools when missing capabilities
- Retry on corrections
- Minimize filler phrases

### Dynamic Context Injection

**File:** [src/caal/settings.py](src/caal/settings.py)

**Features:**
- Variable substitution: `{{CURRENT_DATE_CONTEXT}}`
- Loaded via `load_prompt_with_context()` function
- Timezone-aware date/time injection

### Example Prompt Content

```markdown
# Voice Assistant

You are a helpful, conversational voice assistant. {{CURRENT_DATE_CONTEXT}}

# Tool Priority

Answer questions in this order:
1. **Tools** - Device control, workflows, environment queries
2. **Web search** - Current events, news, prices, hours, scores
3. **General knowledge** - Only for static facts that never change

# Voice Output

Responses are spoken via TTS. Write plain text only - no asterisks, markdown, or symbols.
- Numbers: "seventy-two degrees" not "72°"
- Keep responses to 1-2 sentences
- Be warm and use contractions

# Tool Usage

When the user requests an action:
1. Always call the appropriate tool
2. Wait for the tool response
3. Summarize the result conversationally

# Response Format

Tool responses often include a `message` field - use that for your spoken response.
```

---

## 3. Tool Calling Technique

### Answer: Hybrid Multi-Source System

CAAL implements a sophisticated tool calling architecture combining:

1. **Ollama Native Function Calling** (Primary)
2. **MCP (Model Context Protocol)** for external integrations
3. **n8n Webhook-based Workflows**
4. **LiveKit @function_tool Decorators**

### A. Tool Discovery

**File:** [src/caal/llm/ollama_node.py:311-389](src/caal/llm/ollama_node.py#L311-L389)

The `_discover_tools()` function aggregates from multiple sources:

```python
async def _discover_tools(agent) -> list[dict] | None:
    """Discover tools from agent methods and MCP servers."""

    # Return cached tools if available
    if hasattr(agent, "_ollama_tools_cache"):
        return agent._ollama_tools_cache

    ollama_tools = []

    # 1. Get @function_tool decorated methods from agent
    if hasattr(agent, "_tools") and agent._tools:
        for tool in agent._tools:
            # Extract function signature and convert to Ollama format
            # Creates tool definition with name, description, parameters

    # 2. Get MCP tools from configured servers (except n8n)
    if hasattr(agent, "_caal_mcp_servers"):
        for server_name, server in agent._caal_mcp_servers.items():
            if server_name == "n8n":
                continue  # n8n uses webhook-based execution

            mcp_tools = await _get_mcp_tools(server)
            # Prefix with server name: "server_name__tool_name"
            ollama_tools.extend(mcp_tools)

    # 3. Add n8n workflow tools (webhook-based)
    if hasattr(agent, "_n8n_workflow_tools"):
        ollama_tools.extend(agent._n8n_workflow_tools)

    # Cache and return
    agent._ollama_tools_cache = result
    return result
```

### B. Tool Schema Format

**Ollama's Native Function Calling Format:**

```python
tool = {
    "type": "function",
    "function": {
        "name": "tool_name",
        "description": "What the tool does",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {"type": "string"},
                "param2": {"type": "integer"}
            },
            "required": ["param1"]
        }
    }
}
```

**Example from web_search.py:**

```python
@function_tool
async def web_search(self, query: str) -> str:
    """Search the web for current events, news, prices, store hours,
    or any time-sensitive information not available from other tools.

    Args:
        query: What to search for on the web.
    """
    # Implementation details...
```

The `@function_tool` decorator automatically generates the tool schema from the function signature and docstring.

### C. LLM Call with Tools

**File:** [src/caal/llm/ollama_node.py:152-196](src/caal/llm/ollama_node.py#L152-L196)

**Two-phase approach:**

#### Phase 1: Non-streaming call to detect tool usage

```python
# If tools available, check for tool calls first (non-streaming)
if ollama_tools:
    response = ollama.chat(
        model=model,
        messages=messages,
        tools=ollama_tools,  # Pass all discovered tools
        think=think,
        stream=False,
        options=options,
    )

    if hasattr(response, "message"):
        # Check for tool calls
        if hasattr(response.message, "tool_calls") and response.message.tool_calls:
            tool_calls = response.message.tool_calls

            # Track tool usage for frontend indicator
            tool_names = [tc.function.name for tc in tool_calls]
            tool_params = [tc.function.arguments for tc in tool_calls]

            # Publish tool status immediately
            if hasattr(agent, "_on_tool_status"):
                asyncio.create_task(agent._on_tool_status(True, tool_names, tool_params))

            # Execute tools and get results
            messages = await _execute_tool_calls(
                agent, messages, tool_calls, response.message,
                tool_data_cache=tool_data_cache,
            )
```

#### Phase 2: Streaming follow-up response

```python
            # Stream follow-up response with tool results
            followup = ollama.chat(
                model=model,
                messages=messages,
                think=think,
                stream=True,
                options=options,
            )

            for chunk in followup:
                if hasattr(chunk, "message") and hasattr(chunk.message, "content"):
                    yield strip_markdown_for_tts(chunk.message.content)
```

### D. Tool Execution Routing

**File:** [src/caal/llm/ollama_node.py:500-545](src/caal/llm/ollama_node.py#L500-L545)

**Three-tier routing system:**

```python
async def _execute_single_tool(agent, tool_name: str, arguments: dict) -> Any:
    """Execute a single tool call.

    Routing priority:
    1. Agent methods (@function_tool decorated)
    2. n8n workflows (webhook-based execution)
    3. MCP servers (with server_name__tool_name prefix parsing)
    """

    # 1. Check if it's an agent method
    if hasattr(agent, tool_name) and callable(getattr(agent, tool_name)):
        result = await getattr(agent, tool_name)(**arguments)
        return result

    # 2. Check if it's an n8n workflow
    if (tool_name in agent._n8n_workflow_name_map
        and agent._n8n_base_url):
        workflow_name = agent._n8n_workflow_name_map[tool_name]
        result = await execute_n8n_workflow(
            agent._n8n_base_url, workflow_name, arguments
        )
        return result

    # 3. Check MCP servers (with multi-server routing)
    if "__" in tool_name:
        server_name, actual_tool = tool_name.split("__", 1)
    else:
        server_name, actual_tool = "n8n", tool_name

    if server_name in agent._caal_mcp_servers:
        server = agent._caal_mcp_servers[server_name]
        result = await _call_mcp_tool(server, actual_tool, arguments)
        if result is not None:
            return result

    raise ValueError(f"Tool {tool_name} not found")
```

### E. n8n Workflow Integration

**File:** [src/caal/integrations/n8n.py:24-111](src/caal/integrations/n8n.py#L24-L111)

n8n workflows are discovered via MCP but executed via webhooks:

**Convention:**
- All workflows use webhook triggers
- Webhook path = workflow name
- Workflow descriptions extracted from webhook node `notes` field

**Discovery Process:**

```python
async def discover_n8n_workflows(n8n_mcp, base_url: str):
    """Discover n8n workflows and create tool definitions.

    Convention: All workflows use webhook triggers with webhook path = workflow name.
    Workflow descriptions extracted from webhook node notes.
    """

    # Get list of workflows via MCP
    result = await n8n_mcp._client.call_tool("search_workflows", {})
    workflows = parse_mcp_result(result)["data"]

    for workflow in workflows:
        wf_name = workflow["name"]
        wf_id = workflow["id"]

        # Get detailed description from webhook notes
        details_result = await n8n_mcp._client.call_tool(
            "get_workflow_details",
            {"workflowId": wf_id}
        )
        workflow_details = parse_mcp_result(details_result)
        description = extract_webhook_description(workflow_details)

        # Create Ollama tool definition
        tool = {
            "type": "function",
            "function": {
                "name": sanitize_tool_name(wf_name),
                "description": description,
                "parameters": {
                    "type": "object",
                    "additionalProperties": True  # Flexible schema
                }
            }
        }
        tools.append(tool)
```

**Webhook Execution:**

```python
async def execute_n8n_workflow(base_url: str, workflow_name: str, arguments: dict):
    """Execute an n8n workflow via POST request.

    Convention: webhook URL = {base_url}/webhook/{workflow_name}
    """
    webhook_url = f"{base_url.rstrip('/')}/webhook/{workflow_name}"

    async with aiohttp.ClientSession() as session:
        async with session.post(webhook_url, json=arguments) as response:
            response.raise_for_status()
            return await response.json()
```

### F. n8n Workflow Example

**File:** [n8n-workflows/espn_get_nfl_scores.json](n8n-workflows/espn_get_nfl_scores.json)

Key elements:

1. **Webhook node** with `notes` field containing tool description:
```json
"notes": "Get live NFL scores and standings. Optional parameters: team (e.g., 'Packers', 'Bears'). Returns current scores, game status, and matchups."
```

2. **Response format** with `message` field for voice output:
```javascript
return {
    message: "Packers 24, Bears 17. Final.",
    games: filteredGames  // Structured data for follow-up
}
```

3. **Settings** to enable MCP discovery:
```json
"settings": {
    "availableInMCP": true
}
```

### G. Tool Data Caching

**File:** [src/caal/llm/ollama_node.py:36-67](src/caal/llm/ollama_node.py#L36-L67)

**Innovative context management system:**

```python
class ToolDataCache:
    """Caches recent tool response data for context injection.

    Tool responses often contain structured data (IDs, arrays) that the LLM
    needs for follow-up calls. This cache preserves that data separately
    from chat history and injects it into context on each LLM call.
    """

    def __init__(self, max_entries: int = 3):
        self._cache = []
        self.max_entries = max_entries

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
```

**Injection Before Each LLM Call:**

```python
# Build final message list
messages = []
if system_prompt:
    messages.append(system_prompt)

# Inject tool data context
if tool_data_cache:
    context = tool_data_cache.get_context_message()
    if context:
        messages.append({"role": "system", "content": context})

# Apply sliding window to chat history
messages.extend(chat_messages[-max_messages:])
```

### Key Patterns & Techniques

1. **Hybrid Tool Discovery**
   - LiveKit `@function_tool` decorators for agent methods
   - MCP protocol for external service discovery
   - n8n webhook convention for workflow integration

2. **Tool Schema Convention**
   - Uses Ollama's native function calling format
   - Flexible schemas with `additionalProperties: true` for workflows
   - Descriptions embedded in webhook node `notes` field

3. **Execution Strategy**
   - Non-streaming initial call to detect tool usage
   - Streaming follow-up response for natural conversation
   - Parallel tool execution support

4. **Context Management**
   - Tool data cache separate from chat history
   - Sliding window for conversation history (default 20 turns)
   - Dynamic system message injection

5. **Voice Optimization**
   - Markdown stripping for TTS (`strip_markdown_for_tts()`)
   - Message field prioritization in tool responses
   - Structured data separated from voice output

---

## 4. Schema Hardening & Pydantic

### Answer: YES - Using Pydantic v2.12.4

**CAAL uses Pydantic** integrated with FastAPI for schema validation on all webhook API endpoints.

### Pydantic Version

**File:** [uv.lock](uv.lock)
```
name = "pydantic"
version = "2.12.4"
dependencies = [
    { name = "annotated-types" },
    { name = "pydantic-core" },
    { name = "typing-extensions" },
    { name = "typing-inspection" },
]
```

Brought in as transitive dependency via FastAPI.

### Pydantic Models

**File:** [src/caal/webhooks.py](src/caal/webhooks.py)

**11 Pydantic BaseModel classes** for request/response validation:

**Import Statement (line 40):**
```python
from pydantic import BaseModel
```

#### Request Models (Input Validation)

**1. AnnounceRequest** (lines 62-66):
```python
class AnnounceRequest(BaseModel):
    """Request body for /announce endpoint."""

    message: str
    room_name: str = "voice_assistant_room"
```

**2. ReloadToolsRequest** (lines 69-74):
```python
class ReloadToolsRequest(BaseModel):
    """Request body for /reload-tools endpoint."""

    tool_name: str | None = None
    message: str | None = None
    room_name: str = "voice_assistant_room"
```

**3. WakeRequest** (lines 77-80):
```python
class WakeRequest(BaseModel):
    """Request body for /wake endpoint."""

    room_name: str = "voice_assistant_room"
```

**4. SettingsUpdateRequest** (lines 277-280):
```python
class SettingsUpdateRequest(BaseModel):
    """Request body for POST /settings endpoint."""

    settings: dict
```

**5. PromptUpdateRequest** (lines 291-294):
```python
class PromptUpdateRequest(BaseModel):
    """Request body for POST /prompt endpoint."""

    content: str
```

#### Response Models (Output Validation)

- **WakeResponse** (lines 83-87)
- **AnnounceResponse** (lines 90-94)
- **ReloadToolsResponse** (lines 97-102)
- **HealthResponse** (lines 105-109)
- **SettingsResponse** (lines 269-274)
- **PromptResponse** (lines 283-288)
- **VoicesResponse** (lines 297-300)
- **ModelsResponse** (lines 303-306)

### FastAPI Integration with Pydantic

**Example Endpoint Usage (lines 112-144):**

```python
@app.post("/announce", response_model=AnnounceResponse)
async def announce(req: AnnounceRequest) -> AnnounceResponse:
    """Make the agent speak a message."""
    # FastAPI automatically validates 'req' against AnnounceRequest schema
    # and serializes the return value using AnnounceResponse

    result = session_registry.get(req.room_name)
    if not result:
        raise HTTPException(status_code=404, detail=f"No active session")

    session, _agent = result
    await session.say(req.message)

    return AnnounceResponse(status="announced", room_name=req.room_name)
```

**Example with Settings Update (lines 327-359):**

```python
@app.post("/settings", response_model=SettingsResponse)
async def update_settings(req: SettingsUpdateRequest) -> SettingsResponse:
    """Update settings."""
    # req.settings is validated as a dict by Pydantic
    current = settings_module.load_settings()

    # Additional manual validation against DEFAULT_SETTINGS
    for key, value in req.settings.items():
        if key in settings_module.DEFAULT_SETTINGS:
            current[key] = value

    settings_module.save_settings(current)
    # ...
```

### Additional Validation Patterns

#### Manual Schema Validation

**File:** [src/caal/settings.py](src/caal/settings.py)

The settings module uses **whitelist validation** to ensure only known keys are accepted (lines 88-112):

```python
def save_settings(settings: dict) -> None:
    """Save settings to JSON file.

    Args:
        settings: Settings dict to save. Only keys in DEFAULT_SETTINGS are saved.
    """
    # Filter to only known keys
    filtered = {k: v for k, v in settings.items() if k in DEFAULT_SETTINGS}

    try:
        SETTINGS_PATH.parent.mkdir(parents=True, exist_ok=True)
        with open(SETTINGS_PATH, "w") as f:
            json.dump(filtered, f, indent=2)
        _settings_cache = None
        logger.info(f"Saved settings to {SETTINGS_PATH}")
    except Exception as e:
        logger.error(f"Failed to save settings: {e}")
        raise
```

**DEFAULT_SETTINGS schema** (lines 32-50):
```python
DEFAULT_SETTINGS = {
    "agent_name": "Cal",
    "tts_voice": "am_puck",
    "prompt": "default",
    "wake_greetings": [...],
    "temperature": 0.7,
    "model": "ministral-3:8b",
    "num_ctx": 8192,
    "max_turns": 20,
    "tool_cache_size": 3,
}
```

#### Config Validation in Setup Scripts

**File:** [n8n-workflows/setup.py](n8n-workflows/setup.py)

Lines 39-49 show manual validation of required configuration fields:

```python
# Validate required fields
missing = []
if not config.get("N8N_HOST"):
    missing.append("N8N_HOST")
if not config.get("N8N_API_KEY") or config.get("N8N_API_KEY") == "your_n8n_api_key_here":
    missing.append("N8N_API_KEY")

if missing:
    print(f"Error: Missing required config: {', '.join(missing)}")
    sys.exit(1)
```

#### MCP Schema Handling

**File:** [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py)

Lines 403-411 show schema conversion from MCP tool definitions:

```python
# Convert MCP schema to Ollama format
parameters = {"type": "object", "properties": {}, "required": []}
if hasattr(mcp_tool, "inputSchema") and mcp_tool.inputSchema:
    schema = mcp_tool.inputSchema
    if isinstance(schema, dict):
        parameters = schema
    elif hasattr(schema, "properties"):
        parameters["properties"] = schema.properties or {}
        parameters["required"] = getattr(schema, "required", []) or []
```

**File:** [src/caal/integrations/n8n.py](src/caal/integrations/n8n.py)

Lines 90-94 show flexible JSON Schema for n8n workflows:

```python
# Flexible schema - LLM uses description to determine parameters
parameters = {
    "type": "object",
    "additionalProperties": True  # Accept any properties
}
```

### Validation Scope & Coverage

**Validated Components:**
- ✅ All webhook API endpoints (8 endpoints total)
- ✅ Settings persistence (whitelist validation)
- ✅ Configuration files (manual validation)
- ✅ MCP tool schemas (JSON Schema format)

**Not Using Schema Validation:**
- ❌ Voice agent internal data structures
- ❌ LLM responses (validated implicitly by Ollama/LiveKit)
- ❌ n8n workflow payloads (flexible schema by design)

### Schema Hardening Assessment

**Strengths:**
- All external API inputs are validated through Pydantic models
- Response shapes are enforced for consistency
- Settings use whitelist validation to prevent injection of unknown keys
- Type hints provide static analysis support
- Modern Python type hints (`str | None`, `list[str]`, `dict`) throughout

**No Advanced Features Used:**
- No custom `@validator`, `@field_validator`, or constraint types (like `constr`, `conint`)
- No Pydantic's `Field()` for adding descriptions, examples, and constraints
- Basic validation only (type checking and required fields)

**Potential Improvements:**
- Could add Pydantic field validators for more granular validation (e.g., string length limits, regex patterns)
- Could add stricter type constraints on the `SettingsUpdateRequest.settings` dict
- Could use Pydantic's `Field()` for adding descriptions, examples, and constraints

**Overall Rating:** The project has **solid schema validation** for its API surface, appropriate for a voice assistant system with webhook integrations.

---

## 5. Framework Usage

### Does NOT Use:
- ❌ **Langchain**
- ❌ **Langfuse**
- ❌ **Langgraph**

### Frameworks Actually Used:

#### 1. LiveKit Agents (Primary Framework)

**File:** [pyproject.toml:36](pyproject.toml#L36)
```toml
livekit-agents[silero,turn-detector,mcp,groq]~=1.3
```

**Purpose:** Voice agent orchestration, session management, STT/TTS pipeline

**Key Features Used:**
- `AgentSession` - Manages voice conversations
- `Agent` base class - Extended by VoiceAssistant
- MCP (Model Context Protocol) integration
- Voice activity detection (VAD)
- Turn detection for conversation flow
- Silero VAD for voice detection

#### 2. Ollama Python SDK

**File:** [pyproject.toml:39](pyproject.toml#L39)
```toml
ollama>=0.3.0
```

**Usage:** Direct API calls to Ollama server for LLM inference

**Implementation:** [src/caal/llm/ollama_node.py:153-225](src/caal/llm/ollama_node.py#L153-L225)

**Features:**
- Synchronous and streaming chat completions
- Native function calling support
- Custom parameter support (think mode, temperature, context window)

#### 3. FastAPI

**File:** [pyproject.toml:47-48](pyproject.toml#L47-L48)
```toml
fastapi>=0.115.0
uvicorn>=0.32.0
```

**Purpose:** Webhook server for external triggers and settings management

**Endpoints:**
- `/announce` - Make agent speak a message
- `/wake` - Manually wake the agent
- `/reload-tools` - Reload tool cache
- `/health` - Health check
- `/settings` - Get/update runtime settings
- `/prompt` - Get/update system prompt
- `/voices` - Get available TTS voices
- `/models` - Get available Ollama models

#### 4. Other Key Dependencies

```toml
# Core utilities
python-dotenv>=1.0.0     # Environment configuration
PyYAML>=6.0              # YAML parsing
requests>=2.31.0         # HTTP client

# Web search
ddgs>=9.0.0              # DuckDuckGo search

# MCP integration
aiohttp>=3.9.0           # Async HTTP for webhooks
```

### Why No Langchain/Langfuse/Langgraph?

**Design Philosophy:**
CAAL-FORK implements its own lightweight abstractions for:

1. **Tool Discovery** - Custom aggregation from multiple sources
2. **Tool Execution Routing** - Three-tier priority system
3. **Context Management** - Tool data caching + sliding window
4. **Conversation History** - Optimized for voice interactions

**Benefits of Custom Implementation:**
- Lower latency (direct Ollama API calls)
- Voice-optimized (markdown stripping, TTS formatting)
- Flexible integration (MCP + n8n + native tools)
- Simpler debugging (no framework abstraction layers)
- Smaller dependency footprint

---

## Project Architecture

### Voice Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  Docker Compose Stack                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐         │
│  │  Frontend   │  │  LiveKit    │  │  Speaches    │         │
│  │  Next.js    │  │  Server     │  │  STT + GPU   │         │
│  │  :3000      │  │  :7880      │  │  :8000       │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬───────┘         │
│         │                │                │                 │
│         └────────────────┴────────────────┘                 │
│                          │                                  │
│                 ┌────────┴────────┐                         │
│                 │  Voice Agent    │                         │
│                 │  Python         │                         │
│                 │  :8889 webhooks │                         │
│                 └────────┬────────┘                         │
└──────────────────────────┼──────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
          ┌───▼───┐   ┌───▼───┐   ┌───▼────┐
          │Ollama │   │  n8n  │   │  MCP   │
          │  LLM  │   │Flows  │   │Servers │
          │:11434 │   │:5678  │   │        │
          └───────┘   └───────┘   └────────┘
```

### Tool Discovery & Execution Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Tool Discovery (Startup)                                 │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
        ┌─────────────────────────────────────┐
        │  _discover_tools()                  │
        │  src/caal/llm/ollama_node.py:311   │
        └──────────────┬──────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌─────────────┐ ┌──────────────┐
│@function_tool│ │MCP Servers  │ │n8n Workflows │
│decorated     │ │(filesystem, │ │(via webhooks)│
│methods       │ │home_assist.)│ │              │
└──────┬───────┘ └──────┬──────┘ └──────┬───────┘
       │                │               │
       └────────────────┼───────────────┘
                        ▼
            ┌──────────────────────┐
            │ Ollama Tool Schema   │
            │ (JSON Function Def)  │
            └──────────┬───────────┘
                       │
┌─────────────────────────────────────────────────────────────┐
│ 2. LLM Call with Tools                                      │
└─────────────────────────────────────────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────┐
        │  ollama.chat()              │
        │  tools=ollama_tools         │
        │  stream=False               │
        └──────────┬──────────────────┘
                   │
         ┌─────────┴──────────┐
         │                    │
         ▼                    ▼
   ┌─────────┐          ┌──────────────┐
   │No tools │          │ Tool calls   │
   │needed   │          │ detected     │
   └────┬────┘          └──────┬───────┘
        │                      │
        │                      ▼
        │          ┌──────────────────────┐
        │          │ _execute_tool_calls()│
        │          └──────────┬───────────┘
        │                     │
        │          ┌──────────┴──────────────┐
        │          │                         │
        │          ▼                         ▼
        │   ┌──────────────┐      ┌──────────────────┐
        │   │ Agent method │      │ n8n webhook      │
        │   │ execution    │      │ POST request     │
        │   └──────────────┘      └──────────────────┘
        │          │                         │
        │          │          ┌──────────────┘
        │          │          │
        │          │          ▼
        │          │   ┌──────────────────┐
        │          │   │ MCP server call  │
        │          │   │ (server__tool)   │
        │          │   └──────────────────┘
        │          │          │
        │          └──────────┴────────┐
        │                              │
        │                              ▼
        │                   ┌─────────────────────┐
        │                   │ Tool results added  │
        │                   │ to messages         │
        │                   └─────────┬───────────┘
        │                             │
        └─────────────────────────────┘
                       │
                       ▼
        ┌─────────────────────────────┐
        │  ollama.chat()              │
        │  stream=True                │
        │  (with tool results)        │
        └──────────┬──────────────────┘
                   │
                   ▼
        ┌─────────────────────────────┐
        │  Stream response to user    │
        │  (strip markdown for TTS)   │
        └─────────────────────────────┘
```

### Directory Structure

```
C:\Users\jbeasley\Desktop\Projects\CAAL-fork\
├── voice_agent.py              # Main entry point
├── pyproject.toml              # Project dependencies
├── uv.lock                     # Locked dependency versions
├── .env.example                # Environment variables template
├── settings.json               # Runtime configuration
├── mcp_servers.json.example    # MCP server configuration
│
├── src/caal/
│   ├── llm/
│   │   ├── ollama_llm.py      # LiveKit LLM plugin
│   │   └── ollama_node.py     # Custom LLM node with tool routing
│   │
│   ├── integrations/
│   │   ├── mcp_loader.py      # Multi-MCP server configuration
│   │   ├── n8n.py             # n8n workflow discovery & execution
│   │   └── web_search.py      # DuckDuckGo + Ollama summarization
│   │
│   ├── settings.py            # Runtime configuration management
│   ├── webhooks.py            # FastAPI webhook endpoints
│   └── session_registry.py    # Active session tracking
│
├── prompt/
│   └── default.md             # System prompt template
│
├── frontend/                   # Next.js web UI
│   ├── components/
│   ├── app/
│   └── package.json
│
├── mobile/                     # Flutter mobile app
│   ├── lib/
│   ├── android/
│   └── pubspec.yaml
│
├── n8n-workflows/             # Example workflow definitions
│   ├── setup.py               # Workflow import script
│   ├── espn_get_nfl_scores.json
│   └── ...
│
└── docker-compose.yml         # Full stack deployment
```

### Component Interactions

```
┌──────────────────────────────────────────────────────────────┐
│ User Interaction Layer                                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐            │
│  │ Web UI     │  │ Mobile App │  │ External    │            │
│  │ (Next.js)  │  │ (Flutter)  │  │ Webhooks    │            │
│  └─────┬──────┘  └─────┬──────┘  └──────┬──────┘            │
│        │               │                │                   │
└────────┼───────────────┼────────────────┼───────────────────┘
         │               │                │
         └───────────────┴────────────────┘
                         │
┌────────────────────────┼───────────────────────────────────┐
│ Voice Agent Layer      │                                   │
├────────────────────────┼───────────────────────────────────┤
│                        ▼                                   │
│         ┌──────────────────────────┐                       │
│         │ LiveKit Agent Session    │                       │
│         │ (VoiceAssistant)         │                       │
│         └──────────┬───────────────┘                       │
│                    │                                       │
│      ┌─────────────┼─────────────┐                         │
│      │             │             │                         │
│      ▼             ▼             ▼                         │
│  ┌──────┐    ┌─────────┐   ┌──────────┐                   │
│  │ STT  │    │   LLM   │   │   TTS    │                   │
│  │Speach│    │ Custom  │   │ Speach   │                   │
│  │  es  │    │  Node   │   │   es     │                   │
│  └──────┘    └────┬────┘   └──────────┘                   │
│                   │                                        │
└───────────────────┼────────────────────────────────────────┘
                    │
┌───────────────────┼────────────────────────────────────────┐
│ Tool Execution     │                                       │
├───────────────────┼────────────────────────────────────────┤
│                   │                                        │
│      ┌────────────┼────────────┐                           │
│      │            │            │                           │
│      ▼            ▼            ▼                           │
│  ┌────────┐  ┌───────┐  ┌──────────┐                      │
│  │Agent   │  │  n8n  │  │   MCP    │                      │
│  │Methods │  │Webhook│  │ Servers  │                      │
│  │        │  │       │  │          │                      │
│  │• web_  │  │HTTP   │  │• filesys │                      │
│  │ search │  │POST   │  │• home_   │                      │
│  │        │  │       │  │ assist   │                      │
│  └────────┘  └───────┘  └──────────┘                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
                    │
┌───────────────────┼────────────────────────────────────────┐
│ External Services │                                        │
├───────────────────┼────────────────────────────────────────┤
│                   │                                        │
│      ┌────────────┼────────────┐                           │
│      │            │            │                           │
│      ▼            ▼            ▼                           │
│  ┌────────┐  ┌───────┐  ┌──────────┐                      │
│  │ Ollama │  │  n8n  │  │Home Asst.│                      │
│  │:11434  │  │:5678  │  │:8123     │                      │
│  └────────┘  └───────┘  └──────────┘                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Critical Files Reference

### LLM & Prompting

| File | Description | Key Features |
|------|-------------|--------------|
| [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py) | Core LLM logic, tool calling | Direct Ollama API, tool discovery, two-phase execution |
| [src/caal/llm/ollama_llm.py](src/caal/llm/ollama_llm.py) | LiveKit LLM plugin | Framework integration layer |
| [src/caal/settings.py](src/caal/settings.py) | Configuration management | Runtime settings, whitelist validation |
| [prompt/default.md](prompt/default.md) | System prompt template | Tool priority, voice guidelines, behavior rules |

### Tool Calling

| File | Lines | Description |
|------|-------|-------------|
| [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py) | 311-389 | Tool discovery from multiple sources |
| [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py) | 500-545 | Tool execution routing (3-tier) |
| [src/caal/llm/ollama_node.py](src/caal/llm/ollama_node.py) | 36-67 | ToolDataCache implementation |
| [src/caal/integrations/n8n.py](src/caal/integrations/n8n.py) | 24-111 | n8n workflow discovery & execution |
| [src/caal/integrations/mcp_loader.py](src/caal/integrations/mcp_loader.py) | - | MCP server configuration loading |
| [src/caal/integrations/web_search.py](src/caal/integrations/web_search.py) | - | DuckDuckGo search + Ollama summarization |

### Schema Validation

| File | Lines | Description |
|------|-------|-------------|
| [src/caal/webhooks.py](src/caal/webhooks.py) | 40-309 | Pydantic models for all API endpoints |
| [src/caal/settings.py](src/caal/settings.py) | 88-112 | Settings whitelist validation |
| [src/caal/settings.py](src/caal/settings.py) | 32-50 | DEFAULT_SETTINGS schema definition |

### Configuration Files

| File | Purpose |
|------|---------|
| [pyproject.toml](pyproject.toml) | Project dependencies and metadata |
| [uv.lock](uv.lock) | Locked dependency versions |
| [.env.example](.env.example) | Environment variables template |
| [settings.json](settings.json) | Runtime configuration |
| [mcp_servers.json.example](mcp_servers.json.example) | MCP server configuration |
| [docker-compose.yml](docker-compose.yml) | Full stack deployment |

### Example Workflows

| File | Description |
|------|-------------|
| [n8n-workflows/espn_get_nfl_scores.json](n8n-workflows/espn_get_nfl_scores.json) | Get live NFL scores via ESPN API |
| [n8n-workflows/setup.py](n8n-workflows/setup.py) | Import workflows into n8n |

---

## Summary

**CAAL-FORK is a voice assistant that:**

1. **Uses Ollama exclusively** (no LLM provider routing)
   - Model selection via configuration, not runtime routing
   - Direct API calls for lower latency
   - Custom parameters support (think mode, temperature, context window)

2. **Has sophisticated prompt templates** in markdown with dynamic context
   - Tool priority system (tools → web search → general knowledge)
   - Voice optimization guidelines
   - Behavioral rules for conversational AI

3. **Implements hybrid tool calling** (Ollama native + MCP + n8n webhooks)
   - Multi-source tool discovery
   - Three-tier execution routing
   - Tool data caching for context preservation
   - Two-phase execution (non-streaming detection + streaming response)

4. **Uses Pydantic v2.12.4** for schema validation on API endpoints
   - 11 BaseModel classes for webhooks
   - Whitelist validation for settings
   - FastAPI integration for automatic validation

5. **Does NOT use Langchain, Langfuse, or Langgraph**
   - Built on LiveKit Agents framework instead
   - Custom lightweight abstractions for tools and context
   - Voice-optimized architecture
   - Lower latency through direct integrations

The architecture prioritizes **voice optimization, low latency, and extensibility** through multiple tool integration patterns.

---

## Additional Insights

### Design Philosophy

CAAL-FORK demonstrates a **"just enough framework"** approach:

- Uses frameworks where they add value (LiveKit for voice, FastAPI for webhooks)
- Avoids heavy abstractions where direct implementations are clearer
- Prioritizes voice user experience (markdown stripping, TTS optimization)
- Extensible through multiple integration patterns (MCP, n8n, decorators)

### Performance Optimizations

1. **Tool Data Caching** - Preserves structured data separately from chat history
2. **Sliding Window** - Limits conversation history to configurable turns
3. **Direct Ollama API** - Bypasses LiveKit LLM wrapper for lower latency
4. **Non-blocking Tool Status** - Publishes tool usage to frontend immediately
5. **Streaming Responses** - Natural conversation flow with real-time output

### Extensibility Points

1. **Add new agent methods** - Decorate with `@function_tool`
2. **Add MCP servers** - Configure in `mcp_servers.json`
3. **Add n8n workflows** - Create webhook-based workflows with descriptions in notes
4. **Customize prompts** - Edit `prompt/default.md` or create new templates
5. **Add API endpoints** - Extend `src/caal/webhooks.py` with Pydantic models

### Production Considerations

**Current State:**
- Voice-optimized for real-time conversations
- Multi-source tool integration
- Runtime configuration via API
- Docker Compose deployment

**Potential Enhancements:**
- Add LLM provider routing for fallback/load balancing
- Implement more advanced Pydantic validation (field validators, constraints)
- Add observability/tracing (could integrate Langfuse here)
- Implement conversation persistence
- Add user authentication/authorization
- Rate limiting on webhook endpoints
- Metrics collection for tool usage

---

**Analysis Date:** 2026-01-05
**Analyzer:** Claude Sonnet 4.5 (via Claude Code)
