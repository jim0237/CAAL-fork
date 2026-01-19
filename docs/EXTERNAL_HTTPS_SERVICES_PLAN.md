# CAAL External STT/TTS Integration with Self-Signed Certificates

## Executive Summary

**Goal**: Configure CAAL to use user's existing Kokoro TTS and Whisper STT services running on a network server with HTTPS and self-signed certificates.

**User's Setup**:
- Kokoro TTS: `https://10.30.11.45:8055` (HTTPS only, self-signed cert)
- Whisper STT: `https://10.30.11.45:8060` (HTTPS only, self-signed cert)
- Ollama: `http://10.30.11.45:11434` (HTTP, no SSL)

**Problem**: Services reject HTTP connections ("Empty reply from server"), and CAAL's default SSL verification rejects self-signed certificates.

**Solution**: Configure CAAL to trust custom CA certificate for HTTPS connections to external STT/TTS services.

---

## CAAL's Default Setup vs Your Setup

### What CAAL Expects Out-of-the-Box

CAAL ships with **bundled Docker containers** for STT and TTS that run alongside the main agent:

**Default Services (from docker-compose.yaml):**

1. **STT - Speaches Container** (`caal-speaches`)
   - Image: `ghcr.io/speaches-ai/speaches:latest-cuda-12.6.3`
   - Runs: Faster-Whisper with GPU acceleration
   - URL: `http://speaches:8000` (internal Docker network)
   - Protocol: **HTTP** (no SSL, containers on same network)
   - Port exposed: 8000 (localhost only by default)

2. **TTS - Kokoro Container** (`caal-kokoro`)
   - Image: `ghcr.io/remsky/kokoro-fastapi-gpu:v0.2.4-master`
   - Runs: Kokoro TTS with GPU acceleration
   - URL: `http://kokoro:8880` (internal Docker network)
   - Protocol: **HTTP** (no SSL, containers on same network)
   - Port exposed: 8880 (localhost only by default)

3. **LLM - Ollama** (NOT bundled)
   - User must provide separately
   - URL: Configured via `OLLAMA_HOST` environment variable
   - Protocol: **HTTP** typically (user's choice)
   - Example: `http://192.168.1.100:11434`

**Why HTTP (no SSL) by default?**
- Services communicate on Docker's internal network (isolated)
- No external network exposure = no encryption needed
- Simpler configuration for most users
- Lower latency (no SSL handshake overhead)

### Your Setup: External HTTPS Services

You're running services on a **separate network server** instead of bundled containers:

**Your Services:**

1. **STT - Your Whisper Server**
   - Location: `https://10.30.11.45:8060`
   - Protocol: **HTTPS** (encrypted network traffic)
   - Certificate: Self-signed (not from public CA)

2. **TTS - Your Kokoro Server**
   - Location: `https://10.30.11.45:8055`
   - Protocol: **HTTPS** (encrypted network traffic)
   - Certificate: Self-signed (not from public CA)

3. **LLM - Your Ollama Server**
   - Location: `http://10.30.11.45:11434`
   - Protocol: **HTTP** (no SSL)

**Why this creates a problem:**
- CAAL expects HTTP connections with no certificate validation
- Your services require HTTPS and reject HTTP (`curl: (52) Empty reply from server`)
- Python's default SSL verification rejects self-signed certificates
- Result: `SSL: CERTIFICATE_VERIFY_FAILED` errors

**What needs to change:**
- Configure CAAL to make HTTPS requests instead of HTTP
- Configure CAAL to trust your self-signed CA certificate
- Disable bundled containers (avoid running duplicate services)

---

## Technical Background

### How CAAL Integrates with STT/TTS

CAAL uses **OpenAI-compatible HTTP APIs** (not MCP servers):

**STT (Whisper/Speaches)**:
- Endpoint: `POST {SPEACHES_URL}/v1/audio/transcriptions`
- Protocol: HTTP REST with multipart form-data
- Request: Audio file + model name
- Response: JSON with transcribed text

**TTS (Kokoro)**:
- Endpoint: `POST {KOKORO_URL}/v1/audio/speech`
- Protocol: HTTP REST with JSON
- Request: `{"model": "kokoro", "input": "text", "voice": "am_puck"}`
- Response: MP3/audio stream

**Voice Discovery**:
- Endpoint: `GET {KOKORO_URL}/v1/audio/voices`
- Response: List of available voices

### Current SSL Handling

CAAL uses the LiveKit `openai.STT` and `openai.TTS` plugins, which internally use:
- `openai.AsyncClient` (Python OpenAI SDK)
- `httpx.AsyncClient` (HTTP library)
- Default SSL verification (rejects self-signed certificates)

**Current code** ([voice_agent.py:243-254](voice_agent.py)):
```python
session = AgentSession(
    stt=openai.STT(
        base_url=f"{SPEACHES_URL}/v1",
        api_key="not-needed",
        model=WHISPER_MODEL,
    ),
    tts=openai.TTS(
        base_url=f"{KOKORO_URL}/v1",
        api_key="not-needed",
        model=TTS_MODEL,
        voice=runtime["tts_voice"],
    ),
)
```

**Key Discovery**: Both `openai.STT` and `openai.TTS` accept a `client` parameter for custom `openai.AsyncClient` configuration. This is our solution pathway.

---

## Implementation Plan

### Phase 1: Certificate Preparation

**User Action Required**: Export the CA certificate that signed your Kokoro and Whisper service certificates.

**Steps**:
1. Obtain the CA certificate (`.crt`, `.pem`, or `.cer` format)
2. Create `certs/` directory in CAAL project root
3. Save certificate as `certs/ca-certificate.crt`

### Phase 2: Code Modifications

#### 2.1 Update voice_agent.py

**File**: [voice_agent.py](voice_agent.py)
**Location**: Lines 241-256 (STT/TTS initialization)

**Changes**:
1. Import SSL and httpx libraries
2. Create SSL context from CA certificate
3. Create custom `httpx.AsyncClient` with SSL configuration
4. Create custom `openai.AsyncClient` instances
5. Pass custom clients to `openai.STT` and `openai.TTS`

**New Configuration**:
```python
import ssl
import httpx
from openai import AsyncOpenAI

# SSL Configuration
CA_CERT_PATH = os.getenv("CA_CERT_PATH", "/app/certs/ca-certificate.crt")
SSL_VERIFY = os.getenv("SSL_VERIFY", "true").lower() == "true"

# Determine SSL verification config
if SSL_VERIFY and os.path.exists(CA_CERT_PATH):
    ssl_context = ssl.create_default_context(cafile=CA_CERT_PATH)
    verify_config = ssl_context
elif not SSL_VERIFY:
    verify_config = False  # Disable verification (dev only)
else:
    verify_config = True  # Default behavior

# Create custom HTTP clients
stt_http_client = httpx.AsyncClient(verify=verify_config)
tts_http_client = httpx.AsyncClient(verify=verify_config)

# Create OpenAI clients with custom SSL
stt_openai_client = AsyncOpenAI(
    base_url=f"{SPEACHES_URL}/v1",
    api_key="not-needed",
    http_client=stt_http_client
)

tts_openai_client = AsyncOpenAI(
    base_url=f"{KOKORO_URL}/v1",
    api_key="not-needed",
    http_client=tts_http_client
)

# Create session with custom clients
session = AgentSession(
    stt=openai.STT(client=stt_openai_client, model=WHISPER_MODEL),
    llm=ollama_llm,
    tts=openai.TTS(
        client=tts_openai_client,
        model=TTS_MODEL,
        voice=runtime["tts_voice"]
    ),
    vad=silero.VAD.load(),
)
```

#### 2.2 Update webhooks.py

**File**: [src/caal/webhooks.py](src/caal/webhooks.py)
**Locations**: Lines 417-421 (get_voices), 448-452 (get_models)

**Changes**: Apply same SSL configuration to httpx calls in webhook endpoints.

**Implementation**:
```python
# Helper function at module level
def get_ssl_context():
    """Get SSL context for httpx clients."""
    CA_CERT_PATH = os.getenv("CA_CERT_PATH", "/app/certs/ca-certificate.crt")
    SSL_VERIFY = os.getenv("SSL_VERIFY", "true").lower() == "true"

    if SSL_VERIFY and os.path.exists(CA_CERT_PATH):
        return ssl.create_default_context(cafile=CA_CERT_PATH)
    elif not SSL_VERIFY:
        return False
    return True

# Update get_voices() function
async with httpx.AsyncClient(verify=get_ssl_context()) as client:
    response = await client.get(f"{kokoro_url}/v1/audio/voices", timeout=10.0)

# Update get_models() function
async with httpx.AsyncClient(verify=get_ssl_context()) as client:
    response = await client.get(f"{ollama_host}/api/tags", timeout=10.0)
```

#### 2.3 Optional: Update n8n.py (if using HTTPS n8n)

**File**: [src/caal/integrations/n8n.py](src/caal/integrations/n8n.py)
**Location**: Line 129 (aiohttp.ClientSession)

**Only needed if**: Your n8n server also uses HTTPS with self-signed certificate.

**Changes**:
```python
import ssl
import aiohttp

# In execute_n8n_workflow function
ssl_context = get_ssl_context()  # Shared helper
connector = aiohttp.TCPConnector(ssl=ssl_context if ssl_context is not False else False)
async with aiohttp.ClientSession(connector=connector) as session:
    async with session.post(webhook_url, json=arguments) as response:
        # ... existing code
```

### Phase 3: Configuration Updates

#### 3.1 Update .env.example

**File**: [.env.example](.env.example)

**Add new section** after line 72 (after KOKORO_URL):

```bash
# =============================================================================
# SSL/TLS Configuration (for external HTTPS services)
# =============================================================================
# Enable/disable SSL certificate verification (true/false)
SSL_VERIFY=true

# Path to custom CA certificate bundle (for self-signed certificates)
# Mount your CA certificate to /app/certs/ca-certificate.crt
CA_CERT_PATH=/app/certs/ca-certificate.crt
```

#### 3.2 Update docker-compose.yaml

**File**: [docker-compose.yaml](docker-compose.yaml)
**Location**: Lines 79-83 (agent service volumes)

**Add certificate mount**:
```yaml
agent:
  # ... existing config ...
  volumes:
    - caal-memory:/app/data
    - ./settings.json:/app/settings.json
    - ./prompt:/app/prompt
    - ./mcp_servers.json:/app/mcp_servers.json
    - ./certs:/app/certs:ro  # NEW: Mount certificates directory
```

**Disable bundled STT/TTS containers** (since using external services):

**Option A**: Comment out services (lines 95-149):
```yaml
# speaches:
#   image: ghcr.io/speaches-ai/speaches:latest-cuda-12.6.3
#   # ... rest of config

# kokoro:
#   image: ghcr.io/remsky/kokoro-fastapi-gpu:v0.2.4-master
#   # ... rest of config
```

**Option B**: Create `docker-compose.override.yaml` (cleaner):
```yaml
services:
  # Disable bundled STT/TTS services
  speaches:
    deploy:
      replicas: 0

  kokoro:
    deploy:
      replicas: 0

  # Update agent dependencies
  agent:
    depends_on:
      livekit:
        condition: service_healthy
      # Remove speaches and kokoro health checks
```

#### 3.3 Create User .env File

**File**: Create `.env` (not in git)

**Content**:
```bash
# Network Configuration
CAAL_HOST_IP=192.168.1.150  # User's machine IP

# LiveKit
LIVEKIT_URL=ws://localhost:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret

# External STT (User's Whisper)
SPEACHES_URL=https://10.30.11.45:8060

# External TTS (User's Kokoro)
KOKORO_URL=https://10.30.11.45:8055
TTS_VOICE=am_puck

# SSL Configuration
SSL_VERIFY=true
CA_CERT_PATH=/app/certs/ca-certificate.crt

# Ollama
OLLAMA_HOST=http://10.30.11.45:11434
OLLAMA_MODEL=ministral-3:8b
OLLAMA_THINK=false
OLLAMA_TEMPERATURE=0.7
OLLAMA_NUM_CTX=8192
OLLAMA_MAX_TURNS=20
TOOL_CACHE_SIZE=3

# Timezone
TIMEZONE=America/Los_Angeles
TIMEZONE_DISPLAY=Pacific Time
```

### Phase 4: Documentation Updates

#### 4.1 Update LAUNCH_PLAN.md

**File**: [LAUNCH_PLAN.md](LAUNCH_PLAN.md)

**Add new section** after configuration section:

```markdown
## Using External STT/TTS Services with HTTPS

If you're using external Whisper/Kokoro services with HTTPS and self-signed certificates:

### 1. Export Your CA Certificate

Export the CA certificate that signed your service certificates:
- Format: PEM (.crt, .pem, or .cer)
- Save to: `certs/ca-certificate.crt` in project root

### 2. Update .env Configuration

```bash
# External services (HTTPS)
SPEACHES_URL=https://your-server:8060
KOKORO_URL=https://your-server:8055

# SSL Configuration
SSL_VERIFY=true
CA_CERT_PATH=/app/certs/ca-certificate.crt
```

### 3. Disable Bundled Containers

Create `docker-compose.override.yaml`:
```yaml
services:
  speaches:
    deploy:
      replicas: 0
  kokoro:
    deploy:
      replicas: 0
  agent:
    depends_on:
      livekit:
        condition: service_healthy
```

### 4. Launch

```bash
docker compose up -d
```

### Development/Testing Option

For quick testing in a trusted network, you can disable SSL verification:
```bash
SSL_VERIFY=false
```

⚠️ **Warning**: This disables certificate validation and should only be used in trusted environments.
```

---

## Critical Files to Modify

1. **[voice_agent.py](voice_agent.py)** (lines 241-256)
   - Add SSL context creation
   - Create custom httpx/OpenAI clients
   - Pass clients to STT/TTS plugins

2. **[src/caal/webhooks.py](src/caal/webhooks.py)** (lines 417, 448)
   - Add SSL helper function
   - Update httpx.AsyncClient calls with verify parameter

3. **[.env.example](.env.example)** (after line 72)
   - Add SSL_VERIFY and CA_CERT_PATH configuration

4. **[docker-compose.yaml](docker-compose.yaml)** (lines 79-83)
   - Mount certs directory to agent container
   - Document disabling bundled STT/TTS services

5. **[LAUNCH_PLAN.md](LAUNCH_PLAN.md)**
   - Add external HTTPS services section
   - Document certificate setup process

---

## Verification Steps

### 1. Test Certificate Mount

```bash
# Start containers
docker compose up -d

# Verify certificate is accessible
docker exec caal-agent ls -la /app/certs/
docker exec caal-agent cat /app/certs/ca-certificate.crt
```

### 2. Test HTTPS Connections

```bash
# Check agent logs for SSL errors
docker logs caal-agent

# Should see successful connections to:
# - https://10.30.11.45:8060 (Whisper)
# - https://10.30.11.45:8055 (Kokoro)
```

### 3. Test Voice Endpoints

```bash
# Test voice discovery endpoint
curl http://localhost:8889/voices

# Should return list of voices from Kokoro
```

### 4. Test End-to-End Voice

1. Open web UI: `http://<CAAL_HOST_IP>:3000`
2. Click microphone button
3. Speak: "Hello, can you hear me?"
4. Verify:
   - STT transcribes correctly
   - LLM responds
   - TTS plays audio response

### 5. Monitor for SSL Errors

```bash
# Watch logs for SSL/certificate errors
docker logs -f caal-agent 2>&1 | grep -i "ssl\|certificate\|verify"

# Should see no SSL errors
```

---

## Troubleshooting

### Error: "SSL: CERTIFICATE_VERIFY_FAILED"

**Cause**: CA certificate not trusted or not mounted correctly.

**Solutions**:
1. Verify certificate file exists: `ls -la certs/ca-certificate.crt`
2. Check Docker mount: `docker exec caal-agent ls -la /app/certs/`
3. Verify certificate format (must be PEM)
4. Check CA_CERT_PATH in .env matches mount path

### Error: "Connection refused" or "Empty reply"

**Cause**: Services may only accept HTTPS on specific ports.

**Solutions**:
1. Confirm service URLs are correct
2. Test from host: `curl -k https://10.30.11.45:8055/health`
3. Check network connectivity from container

### Error: "No module named 'httpx'" or "No module named 'ssl'"

**Cause**: Missing dependencies (should not happen with official image).

**Solutions**:
1. Rebuild Docker image: `docker compose build agent`
2. Check Dockerfile includes Python standard library

### Testing with SSL Verification Disabled

For troubleshooting, temporarily disable verification:

```bash
# In .env
SSL_VERIFY=false
```

Then restart: `docker compose restart agent`

If this fixes the issue, the problem is with the certificate trust configuration.

---

## Alternative: HTTP Proxy Approach (Not Recommended)

If certificate configuration proves difficult, you could:

1. Run an HTTP proxy on the host that terminates SSL
2. Point CAAL to `http://localhost:PORT`
3. Proxy forwards to `https://10.30.11.45:8055/8060`

**Tools**: nginx, envoy, or simple Python proxy

**Not recommended because**: Adds complexity, another failure point, and defeats the purpose of HTTPS.

---

## Security Considerations

### Using SSL_VERIFY=false

**When it's acceptable**:
- Local network only (no internet exposure)
- Trusted environment (home/office network)
- Development/testing purposes
- Services on same physical hardware

**When it's NOT acceptable**:
- Internet-exposed services
- Untrusted networks
- Production deployments
- Compliance requirements (HIPAA, PCI, etc.)

### Self-Signed Certificates on Local Network

Using HTTPS with self-signed certificates on a local network (like your `10.30.11.45` setup) is a **reasonable security practice**:

✅ **Pros**:
- Encrypts traffic (prevents eavesdropping on LAN)
- Protects against accidental misconfiguration
- Good security hygiene
- Free (no CA fees)

❌ **Cons**:
- Requires certificate management
- Trust configuration needed
- More complex setup

**Verdict**: Your approach (HTTPS with self-signed certs) is **more secure** than plain HTTP for local network services. The complexity is worth it if:
- Multiple people use the network
- Guest devices connect to your network
- You handle sensitive data (email, calendar, etc.)

For a home network with only trusted devices, HTTP would be simpler but HTTPS is better practice.

---

## Summary

**Changes Required**:
- ✅ Modify voice_agent.py for custom SSL configuration
- ✅ Modify webhooks.py for webhook SSL handling
- ✅ Update .env.example with SSL settings
- ✅ Update docker-compose.yaml to mount certificates
- ✅ Update LAUNCH_PLAN.md with external HTTPS instructions
- ✅ User provides CA certificate in certs/ca-certificate.crt

**Estimated Effort**:
- Code changes: 1-2 hours
- Testing: 30 minutes
- Documentation: 30 minutes
- **Total**: 2-3 hours

**Risk Level**: Low
- Changes are isolated to HTTP client configuration
- Fallback available (SSL_VERIFY=false for testing)
- No architectural changes needed

**User Impact**: Positive
- Enables use of existing STT/TTS infrastructure
- Maintains security with proper certificate validation
- Avoids running duplicate GPU workloads
