# CAAL Startup Guide for Dockge

This guide covers deploying CAAL on server `10.30.11.45` using Dockge, with external STT, TTS, and Ollama services.

## Prerequisites

**Already running on 10.30.11.45:**
- ✅ STT (Whisper/Speaches): `http://10.30.11.45:8060`
- ✅ TTS (Kokoro): `http://10.30.11.45:8055`
- ✅ Ollama: `http://10.30.11.45:11434`

**Available on 10.30.10.39 (optional, for later):**
- n8n: `http://10.30.10.39:5678`

## Deployment via Dockge

### Step 1: Create the Stack in Dockge

1. Open Dockge web UI
2. Click **"+ Compose"** to create a new stack
3. Name it: `caal`
4. Paste the following docker-compose configuration:

```yaml
# CAAL Voice Framework - External Services Configuration
# STT, TTS, and Ollama are external (already running)

services:
  # ===========================================================================
  # LiveKit Server - WebRTC Infrastructure
  # ===========================================================================
  livekit:
    image: livekit/livekit-server:latest
    container_name: caal-livekit
    command: --config /etc/livekit.yaml --node-ip 10.30.11.45
    ports:
      - "7880:7880"
      - "7881:7881"
      - "7881:7881/udp"
      - "50000-50100:50000-50100/udp"
    volumes:
      - ./livekit.yaml:/etc/livekit.yaml:ro
    networks:
      - caal-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:7880"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  # ===========================================================================
  # CAAL Agent - Voice Processing Pipeline
  # ===========================================================================
  agent:
    image: ghcr.io/jim0237/caal-agent:latest
    container_name: caal-agent
    ports:
      - "8889:8889"
    environment:
      # LiveKit (internal)
      - LIVEKIT_URL=ws://livekit:7880
      - LIVEKIT_API_KEY=devkey
      - LIVEKIT_API_SECRET=secret
      # External STT
      - SPEACHES_URL=http://10.30.11.45:8060
      - WHISPER_MODEL=Systran/faster-whisper-medium
      # External TTS
      - KOKORO_URL=http://10.30.11.45:8055
      - TTS_VOICE=am_puck
      # External Ollama
      - OLLAMA_HOST=http://10.30.11.45:11434
      - OLLAMA_MODEL=ministral-3:8b
      - OLLAMA_THINK=false
      - OLLAMA_TEMPERATURE=0.7
      - OLLAMA_NUM_CTX=8192
      - OLLAMA_MAX_TURNS=20
      - TOOL_CACHE_SIZE=3
      # Timezone
      - TIMEZONE=America/Chicago
      - TIMEZONE_DISPLAY=Central Time
    volumes:
      - caal-memory:/app/data
    networks:
      - caal-network
    depends_on:
      livekit:
        condition: service_healthy
    restart: unless-stopped

  # ===========================================================================
  # Frontend - Next.js Web Interface
  # ===========================================================================
  frontend:
    image: ghcr.io/jim0237/caal-frontend:latest
    container_name: caal-frontend
    ports:
      - "3000:3000"
    environment:
      - LIVEKIT_URL=ws://livekit:7880
      - LIVEKIT_API_KEY=devkey
      - LIVEKIT_API_SECRET=secret
      - WEBHOOK_URL=http://agent:8889
    networks:
      - caal-network
    depends_on:
      livekit:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://127.0.0.1:3000"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

networks:
  caal-network:
    driver: bridge

volumes:
  caal-memory:
    name: caal-memory
```

### Step 2: Create LiveKit Configuration

In the same Dockge stack directory, create `livekit.yaml`:

```yaml
port: 7880
rtc:
  port_range_start: 50000
  port_range_end: 50100
  tcp_port: 7881
  use_external_ip: false
keys:
  devkey: secret
logging:
  level: info
```

**In Dockge:** Use the file editor or SSH to create this file in the stack directory.

### Step 3: Deploy

1. Click **"Deploy"** in Dockge
2. Watch the logs for startup progress

## Alternative: Build from Source

If pre-built images aren't available, you'll need to build locally:

### Option A: Build on the server

```bash
# SSH to server
ssh user@10.30.11.45

# Clone repository
git clone https://github.com/jim0237/CAAL-fork.git
cd CAAL-fork

# Copy your .env file (from Windows machine or create manually)
# The .env file should already exist if you synced it

# Build and run
docker compose up -d --build
```

### Option B: Build images and push to registry

On your development machine:

```bash
# Build images
docker compose build

# Tag for your registry
docker tag caal-agent:latest your-registry/caal-agent:latest
docker tag caal-frontend:latest your-registry/caal-frontend:latest

# Push
docker push your-registry/caal-agent:latest
docker push your-registry/caal-frontend:latest
```

Then update the Dockge compose to use your registry images.

## Verification

### Check Container Status

```bash
docker ps | grep caal
```

Expected output:
```
caal-frontend   Up (healthy)   0.0.0.0:3000->3000/tcp
caal-agent      Up             0.0.0.0:8889->8889/tcp
caal-livekit    Up (healthy)   0.0.0.0:7880->7880/tcp
```

### Check Logs

```bash
# All services
docker compose logs -f

# Agent only (most useful)
docker logs -f caal-agent
```

### Test Connectivity from Agent Container

```bash
# Test Ollama
docker exec caal-agent curl -s http://10.30.11.45:11434/api/tags

# Test STT
docker exec caal-agent curl -s http://10.30.11.45:8060/health

# Test TTS
docker exec caal-agent curl -s http://10.30.11.45:8055/v1/audio/voices
```

### Access Web UI

Open in browser: `http://10.30.11.45:3000`

**Note:** Voice input only works from:
- The server itself (localhost)
- HTTPS connections (requires additional setup)

Text input works from any device on the network.

## Troubleshooting

### "Cannot connect to Ollama"

```bash
# Check agent can reach Ollama
docker exec caal-agent curl -v http://10.30.11.45:11434/api/tags

# If fails, check if Ollama is bound to all interfaces
# Ollama should be started with: OLLAMA_HOST=0.0.0.0 ollama serve
```

### "STT/TTS connection refused"

```bash
# Verify services are running
curl http://10.30.11.45:8060/health
curl http://10.30.11.45:8055/v1/audio/voices

# Check Docker network mode
# If using bridge network, ensure external services bind to 0.0.0.0
```

### Agent can't reach external services

Try using host network mode:

```yaml
agent:
  # ... other config ...
  network_mode: host
  # Remove: networks: - caal-network
```

### Frontend shows "Disconnected"

1. Check LiveKit is healthy: `docker logs caal-livekit`
2. Verify WebSocket port is accessible: `curl http://10.30.11.45:7880`
3. Check browser console for errors

## Adding n8n Later

When ready to enable n8n MCP integration:

1. Enable MCP in n8n:
   - Open n8n UI: `http://10.30.10.39:5678`
   - Go to Settings → MCP Servers
   - Enable and copy the access token

2. Add environment variables to agent service:
   ```yaml
   environment:
     # ... existing vars ...
     - N8N_MCP_URL=http://10.30.10.39:5678/mcp-server/http
     - N8N_MCP_TOKEN=your_token_here
   ```

3. Redeploy the stack

## Quick Reference

| Service | Internal URL | External URL |
|---------|-------------|--------------|
| Frontend | http://frontend:3000 | http://10.30.11.45:3000 |
| Agent | http://agent:8889 | http://10.30.11.45:8889 |
| LiveKit | ws://livekit:7880 | ws://10.30.11.45:7880 |
| STT | - | http://10.30.11.45:8060 |
| TTS | - | http://10.30.11.45:8055 |
| Ollama | - | http://10.30.11.45:11434 |
| n8n | - | http://10.30.10.39:5678 |
