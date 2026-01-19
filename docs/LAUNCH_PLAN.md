# CAAL-FORK Launch Plan

## Quick Reference

**Your Setup:**
- ✅ Ollama running at: `http://10.30.11.45:11434`
- ✅ Models available: Llama 3.1, 3.2, Ministral

**What you need:**
- Docker with NVIDIA GPU support (or Apple Silicon)
- n8n instance with MCP enabled (optional but recommended)
- 12GB+ VRAM for GPU-accelerated STT/TTS

**Launch time:** ~10-15 minutes (first run, includes model downloads)

---

## Table of Contents

1. [Prerequisites Check](#1-prerequisites-check)
2. [Configuration](#2-configuration)
3. [Launch (Docker)](#3-launch-docker)
4. [Verification](#4-verification)
5. [First Voice Test](#5-first-voice-test)
6. [Troubleshooting](#troubleshooting)
7. [Next Steps](#next-steps)

---

## 1. Prerequisites Check

### Required

**✅ You have:**
- Ollama server running at `http://10.30.11.45:11434`
- Llama 3.1, 3.2, Ministral models loaded

**⚠️ You need:**

**A. Docker with NVIDIA GPU Support (Linux)**

Check if installed:
```bash
# Check Docker
docker --version

# Check NVIDIA Container Toolkit
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu20.04 nvidia-smi
```

If not installed, see: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html

**B. Find Your LAN IP**

```bash
# Linux/Mac
ip addr show | grep "inet " | grep -v 127.0.0.1
# or
ifconfig | grep "inet " | grep -v 127.0.0.1

# Windows (PowerShell)
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike "127.*"}
```

**Example output:** `192.168.1.150` (you'll need this for `.env`)

---

## 2. Configuration

### Step 1: Create `.env` File

```bash
cd C:\Users\jbeasley\Desktop\Projects\CAAL-fork
cp .env.example .env
```

### Step 2: Edit `.env` with Your Settings

Open `.env` in your editor and update these values:

```bash
# =============================================================================
# CRITICAL SETTINGS - UPDATE THESE
# =============================================================================

# Your computer's LAN IP (where Docker is running)
# Find with: ip addr show | grep "inet " | grep -v 127.0.0.1
CAAL_HOST_IP=192.168.1.150  # ← CHANGE TO YOUR IP

# Your Ollama server (ALREADY RUNNING)
OLLAMA_HOST=http://10.30.11.45:11434  # ← CORRECT (your server)

# Ollama model - Choose ONE:
OLLAMA_MODEL=ministral-3:8b   # ← Recommended (fast + accurate)
# OLLAMA_MODEL=llama3.1:8b    # ← Alternative (good for reasoning)
# OLLAMA_MODEL=llama3.2:3b    # ← Alternative (fastest, less accurate)

# =============================================================================
# OPTIONAL SETTINGS (defaults are fine)
# =============================================================================

# Disable thinking mode for lower latency (keep this)
OLLAMA_THINK=false
OLLAMA_TEMPERATURE=0.7
OLLAMA_NUM_CTX=8192

# Whisper model for speech-to-text
WHISPER_MODEL=Systran/faster-whisper-medium

# TTS voice (male voice, change if you want)
TTS_VOICE=am_puck
# Other options: af_heart, af_bella, af_sarah (female), am_adam (male)

# Timezone
TIMEZONE=America/Los_Angeles
TIMEZONE_DISPLAY=Pacific Time

# =============================================================================
# n8n (OPTIONAL - Skip for now)
# =============================================================================
# If you have n8n running, uncomment and configure:
# N8N_MCP_URL=http://YOUR_N8N_IP:5678/mcp-server/http
# N8N_MCP_TOKEN=your_token_here

# =============================================================================
# Wake Word (OPTIONAL - Skip for now)
# =============================================================================
# Get from https://console.picovoice.ai/ if you want "Hey Cal" wake word
# PORCUPINE_ACCESS_KEY=
```

### Step 3: Save `.env`

Your `.env` should now have:
- `CAAL_HOST_IP` = your computer's LAN IP
- `OLLAMA_HOST` = `http://10.30.11.45:11434`
- `OLLAMA_MODEL` = `ministral-3:8b` (or your choice)

---

## 3. Launch (Docker)

### First-Time Launch

```bash
# Navigate to project directory
cd C:\Users\jbeasley\Desktop\Projects\CAAL-fork

# Start all services
docker compose up -d
```

**What's happening:**
1. **LiveKit** container starts (WebRTC server)
2. **Speaches** container starts (downloads Whisper model ~1-2 GB, may take 5-10 min)
3. **Kokoro** container starts (downloads TTS model ~200 MB, may take 2-3 min)
4. **Frontend** container starts (Next.js web UI)
5. **Agent** container starts (connects to your Ollama server)

**Progress monitoring:**

```bash
# Watch logs (Ctrl+C to exit)
docker compose logs -f

# Check status
docker compose ps

# Should see all containers "healthy" or "running"
```

### Expected First-Run Timeline

| Service | Status | Time | Download Size |
|---------|--------|------|---------------|
| LiveKit | Starting | 10s | ~100 MB |
| Speaches (STT) | Downloading model | 5-10 min | ~1-2 GB |
| Kokoro (TTS) | Downloading model | 2-3 min | ~200 MB |
| Frontend | Building | 30s | ~50 MB |
| Agent | Connecting to Ollama | 10s | ~20 MB |

**Total first-run time:** ~10-15 minutes

**Subsequent launches:** ~30 seconds (models cached)

---

## 4. Verification

### Step 1: Check Container Status

```bash
docker compose ps
```

**Expected output:**
```
NAME                 STATUS              PORTS
caal-agent          Up 2 minutes        0.0.0.0:8889->8889/tcp
caal-frontend       Up 2 minutes (healthy)  0.0.0.0:3000->3000/tcp
caal-kokoro         Up 2 minutes (healthy)  0.0.0.0:8880->8880/tcp
caal-livekit        Up 2 minutes (healthy)  Multiple ports
caal-speaches       Up 2 minutes (healthy)  0.0.0.0:8000->8000/tcp
```

All should show "Up" and "(healthy)" status.

### Step 2: Check Logs

```bash
# Check agent logs (should see "Connected to Ollama")
docker compose logs agent | tail -20

# Check for errors
docker compose logs | grep -i error
```

### Step 3: Verify Ollama Connection

```bash
# Test from agent container
docker exec caal-agent curl -s http://10.30.11.45:11434/api/tags

# Should return JSON with your models (llama3.1, llama3.2, ministral)
```

### Step 4: Access Web UI

Open your browser:
```
http://<YOUR_CAAL_HOST_IP>:3000
```

Example: `http://192.168.1.150:3000`

**Expected:** You should see the CAAL web interface with:
- Microphone button (may show "not supported" if using HTTP from another device)
- Text input field
- Settings icon (top right)

---

## 5. First Voice Test

### Option A: From the Host Machine (Easiest)

1. Open `http://localhost:3000` in your browser
2. Click the microphone button
3. Say: **"What's the weather?"**
4. Wait 1-2 seconds
5. You should hear a voice response

### Option B: From Another Device (Requires HTTPS)

**HTTP limitation:** Browsers block microphone access on HTTP except from localhost.

**For now, use text chat:**
1. Open `http://<YOUR_CAAL_HOST_IP>:3000` from another device
2. Type in the text input: **"What's the weather?"**
3. Press Enter
4. You should see a text response

**To enable voice from other devices:** See [HTTPS Setup](#https-setup-optional) below.

### Expected First Response

**User:** "What's the weather?"

**CAAL:**
- May say "I don't have access to weather tools yet" (expected without n8n setup)
- Or may use web_search tool to look it up (if DuckDuckGo search is working)

**Response time:** ~1-2 seconds total
- 500-800ms: Tool detection (Ollama)
- 300-500ms: Tool execution (web search)
- 500ms: TTS generation

---

## Troubleshooting

### Problem: "Cannot connect to Ollama"

**Check logs:**
```bash
docker compose logs agent | grep -i ollama
```

**Fix 1: Verify Ollama is accessible from Docker**
```bash
# Test from agent container
docker exec caal-agent curl -v http://10.30.11.45:11434/api/tags

# If this fails, check firewall or Docker network settings
```

**Fix 2: Update OLLAMA_HOST in `.env`**
```bash
# Make sure it's the network-accessible IP, not localhost
OLLAMA_HOST=http://10.30.11.45:11434  # ✓ Network IP
# NOT: http://localhost:11434           # ✗ Won't work from Docker
```

Restart after changing `.env`:
```bash
docker compose restart agent
```

---

### Problem: Speaches container stuck downloading

**Check progress:**
```bash
docker compose logs speaches -f
```

**If download is slow:**
- First run downloads ~1-2 GB model from HuggingFace
- Patience! This is normal for first launch
- Model is cached in `speaches-cache` volume for future runs

**If download fails:**
```bash
# Restart the container
docker compose restart speaches

# Check if it recovers
docker compose logs speaches -f
```

---

### Problem: Kokoro container errors

**Check logs:**
```bash
docker compose logs kokoro | tail -30
```

**Common issue: GPU not detected**

If you see "No GPU found" or "CUDA not available":

```bash
# Check GPU is accessible
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu20.04 nvidia-smi

# If this fails, install NVIDIA Container Toolkit
# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
```

---

### Problem: Frontend won't load

**Check status:**
```bash
docker compose logs frontend | tail -20
```

**Common issue: Port 3000 already in use**

```bash
# Check what's using port 3000
sudo netstat -tlnp | grep 3000

# Kill the process or change CAAL port
```

To use a different port, edit `docker-compose.yaml`:
```yaml
frontend:
  ports:
    - "3001:3000"  # External:Internal
```

Then access at `http://<YOUR_IP>:3001`

---

### Problem: Voice doesn't work from other devices

**Expected behavior:** HTTP mode only supports voice from localhost.

**Solutions:**

**Option 1: Use text chat** (immediate)
- Type queries instead of using microphone
- Works from any device

**Option 2: Enable HTTPS** (15-30 min setup)
- See [HTTPS Setup](#https-setup-optional) below
- Enables voice from any device

---

### Problem: Slow responses (>5 seconds)

**Check which model you're using:**
```bash
grep OLLAMA_MODEL .env
```

**Fast models (recommended):**
- `ministral-3:8b` - Best balance (500-800ms)
- `llama3.2:3b` - Fastest (300-500ms)

**Slower models (not recommended for voice):**
- `llama3.1:70b` - Very slow (5-10s+)
- `qwen3:14b` - Moderate (1-2s)

**Change model:**
```bash
# Edit .env
OLLAMA_MODEL=ministral-3:8b

# Restart agent
docker compose restart agent
```

---

## HTTPS Setup (Optional)

**Why?** Browsers require HTTPS for microphone access from non-localhost devices.

### Option 1: mkcert (Local Network)

**Best for:** Using CAAL on your local network only

**Steps:**

1. **Install mkcert:**
```bash
# Arch/Manjaro
sudo pacman -S mkcert

# Ubuntu/Debian
sudo apt install mkcert

# macOS
brew install mkcert

# Windows (Scoop)
scoop install mkcert
```

2. **Install local CA:**
```bash
mkcert -install
```

3. **Generate certificates:**
```bash
# Replace with YOUR CAAL_HOST_IP
mkcert 192.168.1.150

# Move certs to project
mkdir -p certs
mv 192.168.1.150.pem certs/server.crt
mv 192.168.1.150-key.pem certs/server.key
chmod 644 certs/server.key
```

4. **Update `.env`:**
```bash
CAAL_HOST_IP=192.168.1.150
HTTPS_DOMAIN=192.168.1.150
```

5. **Rebuild frontend:**
```bash
docker compose --profile https build frontend
```

6. **Launch with HTTPS:**
```bash
docker compose down
docker compose --profile https up -d
```

7. **Access:**
```
https://192.168.1.150
```

**Note:** Other devices need the mkcert CA installed to avoid certificate warnings.

---

### Option 2: Tailscale (Remote Access)

**Best for:** Accessing CAAL from anywhere (not just local network)

**Prerequisites:**
- Tailscale account and installed on your machine

**Steps:**

1. **Get Tailscale hostname:**
```bash
tailscale status | head -1
# Example: your-machine.tailnet.ts.net
```

2. **Generate Tailscale certs:**
```bash
tailscale cert your-machine.tailnet.ts.net

# Move to project
mkdir -p certs
mv your-machine.tailnet.ts.net.crt certs/server.crt
mv your-machine.tailnet.ts.net.key certs/server.key
chmod 644 certs/server.key
```

3. **Update `.env`:**
```bash
CAAL_HOST_IP=100.x.x.x  # Your Tailscale IP (run: tailscale ip -4)
HTTPS_DOMAIN=your-machine.tailnet.ts.net
```

4. **Rebuild and launch:**
```bash
docker compose --profile https build frontend
docker compose down
docker compose --profile https up -d
```

5. **Access from anywhere:**
```
https://your-machine.tailnet.ts.net
```

---

## Next Steps

### 1. Configure n8n Integration (Optional)

**What it enables:**
- Home Assistant control ("Turn on the lights")
- Calendar integration ("What's on my schedule?")
- Email automation ("Send email to...")
- Any n8n workflow becomes a tool

**Setup:**

1. **Install n8n** (if you don't have it):
```bash
docker run -d --name n8n -p 5678:5678 n8nio/n8n
```

2. **Enable MCP in n8n:**
- Settings → MCP Access → Enable
- Copy the access token

3. **Update `.env`:**
```bash
N8N_MCP_URL=http://YOUR_N8N_IP:5678/mcp-server/http
N8N_MCP_TOKEN=your_token_here
```

4. **Restart agent:**
```bash
docker compose restart agent
```

5. **Test:**
- Say: "List my n8n workflows"
- CAAL should discover and list available workflows

---

### 2. Add More MCP Servers (Optional)

**Available MCP servers:**
- `@modelcontextprotocol/server-filesystem` - File operations
- `@modelcontextprotocol/server-github` - GitHub operations
- `@modelcontextprotocol/server-brave-search` - Web search
- Community servers: https://github.com/modelcontextprotocol/servers

**Setup:**

1. **Copy example config:**
```bash
cp mcp_servers.json.example mcp_servers.json
```

2. **Edit `mcp_servers.json`:**
```json
{
  "filesystem": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/documents"],
    "env": {}
  }
}
```

3. **Restart agent:**
```bash
docker compose restart agent
```

---

### 3. Port Your Tools from CLI Project

If you're porting from your CLI project (Gmail, Calendar, Notes):

**See:** [BUILD_VS_BUY_DECISION.md](BUILD_VS_BUY_DECISION.md) - Migration Path section

**Quick guide:**

1. **Convert to @function_tool format:**
```python
# Your CLI tool
def search_emails(query: str, max_results: int = 10):
    # Gmail API code

# CAAL format
@function_tool
async def search_emails(self, query: str, max_results: int = 10) -> dict:
    """Search Gmail messages using query syntax.

    Args:
        query: Gmail search query (from:, subject:, after:, before:)
        max_results: Maximum number of results (1-50)
    """
    # Your Gmail API code (same as CLI)
```

2. **Add to agent class** (in `voice_agent.py` or separate file)

3. **Restart agent**

**Effort:** ~4-6 hours to port all tools

---

### 4. Test Different Models

**Try different models to find best balance:**

```bash
# Edit .env
OLLAMA_MODEL=llama3.2:3b  # Fastest
# or
OLLAMA_MODEL=ministral-3:8b  # Best balance (recommended)
# or
OLLAMA_MODEL=llama3.1:8b  # Better reasoning

# Restart
docker compose restart agent
```

**Test latency:**
- Say same query with each model
- Compare response speed
- Choose based on your priorities (speed vs accuracy)

---

### 5. Monitor Performance

**View logs in real-time:**
```bash
# All services
docker compose logs -f

# Just agent
docker compose logs -f agent

# Filter for errors
docker compose logs -f | grep -i error
```

**Check resource usage:**
```bash
docker stats
```

---

## Common Commands

### Start/Stop

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart specific service
docker compose restart agent

# View logs
docker compose logs -f agent
```

### Updates

```bash
# Pull latest code
git pull

# Rebuild containers
docker compose build

# Restart with new code
docker compose down && docker compose up -d
```

### Cleanup

```bash
# Stop and remove containers
docker compose down

# Remove volumes (DELETES cached models!)
docker compose down -v

# Remove images
docker compose down --rmi all
```

---

## System Requirements

### Minimum (GPU-Accelerated)

- **CPU:** 4+ cores
- **RAM:** 8 GB
- **GPU:** NVIDIA GPU with 12GB+ VRAM
  - Speaches (STT): ~4-6 GB VRAM
  - Kokoro (TTS): ~2-3 GB VRAM
  - Ollama (LLM): Already running on your server ✓
- **Disk:** 10 GB free (for Docker images + model cache)
- **Network:** 10+ Mbps for model downloads

### Recommended (Optimal Performance)

- **CPU:** 6+ cores
- **RAM:** 16 GB
- **GPU:** NVIDIA RTX 3060 or better (12-24 GB VRAM)
- **Disk:** 20 GB free
- **Network:** 50+ Mbps

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  Your Browser (http://YOUR_IP:3000)                 │
│  - Voice input (microphone)                         │
│  - Voice output (speakers)                          │
└─────────────────┬───────────────────────────────────┘
                  │ WebSocket (WebRTC)
                  ↓
┌─────────────────────────────────────────────────────┐
│  Docker Compose Stack (Your Machine)                │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │  Frontend  │  │  LiveKit   │  │   Agent      │  │
│  │  Next.js   │  │  WebRTC    │  │   Python     │  │
│  │  :3000     │  │  :7880     │  │   :8889      │  │
│  └────────────┘  └────────────┘  └───────┬──────┘  │
│                                           │         │
│  ┌────────────┐  ┌────────────┐          │         │
│  │  Speaches  │  │  Kokoro    │          │         │
│  │  STT+GPU   │  │  TTS+GPU   │          │         │
│  │  :8000     │  │  :8880     │          │         │
│  └────────────┘  └────────────┘          │         │
└───────────────────────────────────────────┼─────────┘
                                           │
                    ┌──────────────────────┼─────────┐
                    │                      │         │
                    ▼                      ▼         ▼
            ┌──────────────┐      ┌────────────┐  ┌────────┐
            │   Ollama     │      │    n8n     │  │  MCP   │
            │ 10.30.11.45  │      │  :5678     │  │Servers │
            │   :11434     │      │ (optional) │  │(opt.)  │
            └──────────────┘      └────────────┘  └────────┘
```

---

## FAQ

### Q: Do I need to run Ollama in Docker?

**A:** No! You already have Ollama running at `http://10.30.11.45:11434`. CAAL will connect to it.

### Q: Can I use models other than Ministral?

**A:** Yes! Any Ollama model that supports function calling:
- `llama3.1:8b` - Good reasoning
- `llama3.2:3b` - Fastest
- `ministral-3:8b` - Best balance (recommended)
- `qwen3:8b` - Good alternative

Avoid large models (70B+) for voice - too slow.

### Q: Why can't I use voice from my phone/tablet?

**A:** HTTP mode only allows voice from localhost (security restriction). Use HTTPS mode for voice from other devices.

### Q: How much GPU memory do I need?

**A:** ~12 GB minimum:
- Speaches (STT): 4-6 GB
- Kokoro (TTS): 2-3 GB
- Buffer: 2-3 GB

Your Ollama server handles LLM separately ✓

### Q: Can I run without GPU?

**A:** Not recommended. CPU-only STT/TTS is very slow (10-30s latency). Consider:
- Renting GPU cloud instance
- Using Apple Silicon (M1/M2/M3/M4) with mlx-audio

### Q: Where are models stored?

**A:** Docker volumes:
- `caal-speaches-cache` - Whisper models (~1-2 GB)
- `caal-kokoro-cache` - TTS models (~200 MB)
- `caal-memory` - Agent data (conversation logs, etc.)

Delete with: `docker compose down -v` (WARNING: removes all cached data)

---

## Support & Resources

**Documentation:**
- Full README: [README.md](README.md)
- Build vs Buy: [BUILD_VS_BUY_DECISION.md](BUILD_VS_BUY_DECISION.md)
- Tool Calling Deep Dive: [TOOL_CALLING_DEEP_DIVE.md](TOOL_CALLING_DEEP_DIVE.md)

**Useful Links:**
- LiveKit Agents: https://docs.livekit.io/agents/
- Ollama: https://ollama.ai/
- Speaches: https://github.com/speaches-ai/speaches
- Kokoro TTS: https://github.com/remsky/Kokoro-FastAPI

**Original Project:**
- GitHub: https://github.com/CoreWorxLab/caal

---

**Last Updated:** 2026-01-06
**Version:** CAAL-FORK
