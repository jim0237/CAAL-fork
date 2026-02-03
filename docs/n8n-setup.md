# n8n MCP Server Setup

Connect CAAL to n8n for workflow automation tools.

## Prerequisites

- n8n version 1.70+ (MCP server support)
- MCP Server feature enabled in n8n

## 1. Enable MCP Server in n8n

1. Open n8n Settings > MCP Servers
2. Enable the MCP Server
3. Copy the **Server URL** (format: `http://HOST:PORT/mcp-server/http`)
4. Generate an **API Token** and copy it

## 2. Configure CAAL

Add to your `.env` file:

```env
N8N_MCP_URL=http://YOUR_N8N_IP:5678/mcp-server/http
N8N_MCP_TOKEN=your_jwt_token_here
```

### Important Notes

- The URL path must be `/mcp-server/http` (not `/mcp`)
- n8n uses **Streamable HTTP** transport (POST with SSE response)
- CAAL auto-configures this transport in `mcp_loader.py`

## 3. Rebuild and Restart

```bash
docker compose build agent
docker compose up -d agent
```

## 4. Verify Connection

Check agent logs for MCP initialization:

```bash
docker compose logs agent | grep -i mcp
```

Or test directly:

```bash
docker exec caal-agent /app/.venv/bin/python -c "
import asyncio
import os
os.chdir('/app')
from caal.integrations import load_mcp_config, initialize_mcp_servers

async def test():
    configs = load_mcp_config()
    servers = await initialize_mcp_servers(configs)
    print(f'Connected: {list(servers.keys())}')
    if 'n8n' in servers:
        tools = await servers['n8n'].list_tools()
        print(f'Tools: {len(tools)}')

asyncio.run(test())
"
```

## How Workflow Discovery Works

CAAL discovers workflows **from the n8n server** at runtime:

1. Calls n8n MCP `search_workflows` to list all workflows
2. For each workflow, calls `get_workflow_details` to extract descriptions
3. Creates tool definitions from webhook node notes
4. Exposes workflows as callable tools to the voice agent

**Convention:** Workflows must use webhook triggers. The webhook path becomes the tool name.

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Session terminated` | Wrong transport type | Ensure `transport="streamable_http"` in mcp_loader.py |
| `404 Not Found` | Wrong URL path | Use `/mcp-server/http` not `/mcp` |
| `406 Not Acceptable` | Client not accepting SSE | Transport issue - should not occur with streamable_http |
| `401 Unauthorized` | Invalid/expired token | Generate new token in n8n MCP settings |

### Test n8n endpoint directly

```bash
curl -X POST "http://YOUR_N8N_IP:5678/mcp-server/http" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

Expected: HTTP 200 with SSE response containing `protocolVersion` and `serverInfo`.

## Optional: Timeout Configuration

Default timeout is 10 seconds. Increase for slow networks:

```env
N8N_MCP_TIMEOUT=30.0
```
