# External Service Requirements for CAAL

## Protocol Requirements

**REQUIRED:** HTTP only (no HTTPS)
- CAAL expects plain HTTP connections
- Self-signed HTTPS certificates require code modifications to CAAL
- Use HTTP for simplest integration

**Services Required:**
- STT (Speech-to-Text): Whisper/Speaches
- TTS (Text-to-Speech): Kokoro
- LLM: Ollama
- Workflows (Optional): n8n

## STT Service Requirements (Whisper/Speaches)

**Base URL Format:** `http://<ip>:<port>`

**Required Endpoint:**
```
POST /v1/audio/transcriptions
Content-Type: multipart/form-data

Parameters:
- file: <audio_file> (required)
- model: <model_name> (required)

Response:
{
  "text": "transcribed text"
}
```

**Example Configuration:**
```bash
SPEACHES_URL=http://10.30.11.45:8060
```

---

## TTS Service Requirements (Kokoro)

**Base URL Format:** `http://<ip>:<port>`

**Required Endpoint:**
```
POST /v1/audio/speech
Content-Type: application/json

Body:
{
  "model": "kokoro",
  "input": "text to speak",
  "voice": "am_puck"
}

Response:
- Audio stream (MP3 or WAV)
- Content-Type: audio/mpeg or audio/wav
```

**Optional Endpoint (for voice discovery):**
```
GET /v1/audio/voices

Response:
{
  "voices": [
    {"id": "am_puck", ...},
    {"id": "af_heart", ...}
  ]
}
```

**Example Configuration:**
```bash
KOKORO_URL=http://10.30.11.45:8055
TTS_VOICE=am_puck
```

---

## API Compatibility

Both services must implement **OpenAI-compatible APIs**:
- STT: OpenAI Speech-to-Text API format
- TTS: OpenAI Text-to-Speech API format

---

## Testing

**Test STT:**
```bash
curl -X POST http://10.30.11.45:8060/v1/audio/transcriptions \
  -F "file=@sample.wav" \
  -F "model=Systran/faster-whisper-small"
```

**Test TTS:**
```bash
curl -X POST http://10.30.11.45:8055/v1/audio/speech \
  -H "Content-Type: application/json" \
  -d '{"model":"kokoro","input":"Hello world","voice":"am_puck"}' \
  --output test.mp3
```

**Test Voice Discovery:**
```bash
curl http://10.30.11.45:8055/v1/audio/voices
```

---

## Ollama Service Requirements

**Base URL Format:** `http://<ip>:<port>`

**Required Endpoint:**
```
POST /api/chat

Standard Ollama API format
```

**Example Configuration:**
```bash
OLLAMA_HOST=http://10.30.11.45:11434
OLLAMA_MODEL=ministral-3:8b
OLLAMA_THINK=false
```

**Testing:**
```bash
curl http://10.30.11.45:11434/api/tags
```

---

## n8n Service Requirements (Optional)

**Base URL Format:** `http://<ip>:<port>`

**Required Endpoint:**
```
POST /mcp-server/http

MCP (Model Context Protocol) server endpoint
```

**Required:**
- MCP server feature enabled in n8n
- MCP access token generated from n8n Settings

**Example Configuration:**
```bash
N8N_MCP_URL=http://10.30.10.39:5678/mcp-server/http
N8N_MCP_TOKEN=<your-token-from-n8n-settings>
```

**Getting MCP Token:**
1. Access n8n web interface
2. Go to Settings > MCP Access (or similar)
3. Generate/copy MCP access token
4. Add to CAAL `.env` file

**Testing:**
```bash
curl http://10.30.10.39:5678/mcp-server/http
```

Expected: Valid response (not 404)

**Note:** If n8n MCP endpoint returns 404, enable MCP server feature in n8n settings first.

---

## CAAL Configuration

Once all services are running with HTTP:

1. Create `.env` file (copy from `.env.example`)
2. Set service URLs:
   ```bash
   # Your machine's LAN IP
   CAAL_HOST_IP=192.168.1.150

   # External STT
   SPEACHES_URL=http://10.30.11.45:8060

   # External TTS
   KOKORO_URL=http://10.30.11.45:8055
   TTS_VOICE=am_puck

   # External Ollama
   OLLAMA_HOST=http://10.30.11.45:11434
   OLLAMA_MODEL=ministral-3:8b
   OLLAMA_THINK=false

   # External n8n (optional)
   N8N_MCP_URL=http://10.30.10.39:5678/mcp-server/http
   N8N_MCP_TOKEN=<your-token>
   ```
3. Disable bundled STT/TTS containers:

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

4. Launch: `docker compose up -d`

**No code changes to CAAL required.**
