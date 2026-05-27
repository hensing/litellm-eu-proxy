# litellm-european-proxy

A Docker-based [LiteLLM](https://github.com/BerriAI/litellm) proxy that aggregates two European AI providers behind a single OpenAI-compatible API endpoint. Designed for use with [paperless-gpt](https://github.com/icereed/paperless-gpt) and similar tools that require an OCR or LLM backend.

**Author:** Dr. Henning Dickten ([@hensing](https://github.com/hensing))
**License:** MIT

---

## Providers & Models

| Model name (proxy) | Provider | Underlying model | Use case |
|--------------------|----------|------------------|----------|
| `lighton-ocr` | [LightOn](https://lighton.ai) | LightOnOCR2 | Document OCR — **primary paperless-gpt endpoint** |
| `mistral-small` | [Mistral AI](https://mistral.ai) | Mistral Small 4 | Tagging, classification, n8n, paperless-gpt |
| `mistral-large` | Mistral AI | Mistral Large 3 | Complex reasoning, best-in-class quality |
| `mistral-embed` | Mistral AI | mistral-embed | Multilingual embeddings |
| `mistral-tts` | Mistral AI | Voxtral TTS | Text-to-speech — EN + DE |
| `mistral-stt` | Mistral AI | Voxtral Mini Transcribe V2 | Speech-to-text — EN + DE (batch) |

All providers are OpenAI-compatible — no custom Python adapter is required.

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Compose (v2+)
- API keys for the providers you want to use (see [Configuration](#configuration))

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/hensing/litellm-european-proxy.git
cd litellm-european-proxy

# 2. Copy the environment template and fill in your keys
cp .env.example .env
$EDITOR .env

# 3. Start the proxy (Ollama starts automatically as a dependency)
docker compose up -d

# 4. Pull the LightOnOCR-2 model into Ollama (one-time, ~1 GB)
docker exec litellm-ollama ollama pull maternion/LightOnOCR-2

# 5. Verify it is running
curl http://localhost:4000/health
```

The proxy is now available at `http://localhost:4000` with an OpenAI-compatible API.

---

## Ollama — Local Model Setup

The `lighton-ocr` model runs locally inside an **Ollama** container that starts automatically with `docker compose up -d`. The model weights are not bundled — you pull them once after the first start.

### Pull the model (required once)

```bash
docker exec litellm-ollama ollama pull maternion/LightOnOCR-2
```

The download is roughly 1 GB and is cached in `./data/ollama/`, so it survives container restarts and updates.

### Verify the model is available

```bash
docker exec litellm-ollama ollama list
```

You should see `maternion/LightOnOCR-2` in the output.

### Memory usage (CPU-only hosts)

The default configuration in `docker-compose.yml` is already tuned for low RAM:

| Environment variable | Value | Effect |
|----------------------|-------|--------|
| `OLLAMA_NUM_PARALLEL` | `1` | Processes one request at a time — prevents memory spikes under concurrent load |
| `OLLAMA_MAX_LOADED_MODELS` | `1` | Keeps at most one model in memory at once |
| `OLLAMA_KEEP_ALIVE` | `5m` | Unloads the model from RAM after 5 minutes of inactivity |

With these settings LightOnOCR-2 (Q4 quantised, ~1 GB weights) typically uses **1.5–2 GB RAM** while processing a document and drops back to near zero between requests.

### Update to a newer version

```bash
docker exec litellm-ollama ollama pull maternion/LightOnOCR-2
docker compose restart litellm
```

### GPU acceleration (NVIDIA, optional)

To enable NVIDIA GPU support, uncomment the `deploy` block in `docker-compose.yml`:

```yaml
ollama:
  # ...
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: all
            capabilities: [gpu]
```

Then recreate the container:

```bash
docker compose up -d --force-recreate ollama
```

The [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) must be installed on the host before enabling GPU passthrough.

---

## Configuration

### Environment variables (`.env`)

Copy `.env.example` to `.env` and set the following:

| Variable | Description |
|----------|-------------|
| `LIGHTON_API_KEY` | API key from [paradigm.lighton.ai](https://paradigm.lighton.ai) |
| `MISTRAL_API_KEY` | API key from [console.mistral.ai](https://console.mistral.ai) → API Keys |
| `LITELLM_MASTER_KEY` | Any secret string — clients use this as the Bearer token |

### Model names

Model slugs (the identifier sent to the upstream provider) are configured in `config/litellm_config.yaml`. You can verify available models for each provider:

```bash
# LightOn
curl https://paradigm.lighton.ai/api/v2/models \
  -H "Authorization: Bearer $LIGHTON_API_KEY"

# Mistral
curl https://api.mistral.ai/v1/models \
  -H "Authorization: Bearer $MISTRAL_API_KEY"
```

---

## paperless-gpt Integration

Configure paperless-gpt to use this proxy as its LLM backend:

```yaml
# paperless-gpt configuration
LLM_PROVIDER: openai
LLM_MODEL: lighton-ocr
OPENAI_API_KEY: your_master_key_here      # value of LITELLM_MASTER_KEY
OPENAI_BASE_URL: http://localhost:4000/v1 # or your server's address
```

The `lighton-ocr` model routes to **LightOnOCR2**, a multilingual vision-language model optimised for document OCR. For tagging and metadata generation, pair it with `mistral-small`.

---

## API Usage Examples

All requests use standard OpenAI API format. Replace `your_master_key` with the value of `LITELLM_MASTER_KEY`.

### List available models

```bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer your_master_key"
```

### Document OCR (LightOnOCR2)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer your_master_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "lighton-ocr",
    "messages": [{
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "Extract all text from this document. Return only the raw text, preserving line breaks."
        },
        {
          "type": "image_url",
          "image_url": {
            "url": "data:image/jpeg;base64,<BASE64_ENCODED_IMAGE>"
          }
        }
      ]
    }]
  }'
```

### Chat / Tagging (Mistral Small)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer your_master_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-small",
    "messages": [{"role": "user", "content": "Classify this document: invoice from Acme Corp, dated 2024-01-15."}],
    "response_format": {"type": "json_object"}
  }'
```

### Embeddings (Mistral)

```bash
curl http://localhost:4000/v1/embeddings \
  -H "Authorization: Bearer your_master_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-embed",
    "input": "The quick brown fox jumps over the lazy dog"
  }'
```

### Speech-to-text (Voxtral Transcribe V2)

```bash
curl http://localhost:4000/v1/audio/transcriptions \
  -H "Authorization: Bearer your_master_key" \
  -F "model=mistral-stt" \
  -F "file=@recording.mp3" \
  -F "language=de"
```

### Text-to-speech (Voxtral TTS)

```bash
curl http://localhost:4000/v1/audio/speech \
  -H "Authorization: Bearer your_master_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-tts",
    "input": "Hallo, wie geht es Ihnen?",
    "voice": "alloy"
  }' \
  --output speech.mp3
```

---

## Production Notes

### Enable persistent storage (PostgreSQL)

Uncomment the `postgres` service block in `docker-compose.yml` and add to `.env`:

```dotenv
POSTGRES_PASSWORD=a_strong_database_password
DATABASE_URL=postgresql://litellm:a_strong_database_password@postgres:5432/litellm
STORE_MODEL_IN_DB=true
```

This enables the LiteLLM UI at `http://localhost:4000/ui` for managing virtual API keys and viewing usage metrics.

### Reverse proxy with Nginx Proxy Manager (recommended)

The default `docker-compose.yml` does **not** expose port 4000 to the host. Instead, the `litellm` container joins an external Docker network named `proxy` — the same network that Nginx Proxy Manager (NPM) uses. NPM can then reach the container directly by its service name.

**NPM must exist first.** If you already run NPM with a `proxy` network, no extra steps are needed. If not, create it once:

```bash
docker network create proxy
```

**NPM Proxy Host settings:**

| Field | Value |
|-------|-------|
| Domain name | `ai.example.com` |
| Scheme | `http` |
| Forward Hostname | `litellm` (Docker service name) |
| Forward Port | `4000` |
| SSL | Enable — request a Let's Encrypt certificate |

NPM handles TLS termination; communication between NPM and litellm stays on the internal Docker network (unencrypted, but never leaves the host).

**Security layers in this setup:**

1. **Network isolation** — Port 4000 is not bound to the host. The container is only reachable via the internal `proxy` network.
2. **TLS** — NPM provides HTTPS with automatic Let's Encrypt certificates.
3. **Bearer token** — Every request must include `Authorization: Bearer <LITELLM_MASTER_KEY>`. Requests without a valid key are rejected by LiteLLM before reaching any provider.

For local testing without NPM, temporarily uncomment the `ports` block in `docker-compose.yml` to bind to localhost only:

```yaml
ports:
  - "127.0.0.1:4000:4000"
```

### Security

- Set `LITELLM_MASTER_KEY` to a long random string: `openssl rand -hex 32`
- Never commit `.env` to version control — it is listed in `.gitignore`.
- Do not expose port `4000` directly to the internet when running behind NPM.

---

## Troubleshooting

**Proxy does not start:**
Check that `.env` exists and `LITELLM_MASTER_KEY` is set:
```bash
docker compose logs litellm
```

**`lighton-ocr` returns an error or times out on first request:**
The model may not have been pulled yet. Run:
```bash
docker exec litellm-ollama ollama pull maternion/LightOnOCR-2
```
Check Ollama logs if the pull fails: `docker compose logs ollama`

**Model returns 404 / unknown model:**
The model slug sent to the upstream provider may differ from the assumed value. Query the provider's `/models` endpoint (see [Configuration](#configuration)) and update `config/litellm_config.yaml` accordingly, then restart with `docker compose restart`.

**Voxtral audio model returns 404:**
The model slug for Voxtral TTS/STT is versioned (e.g. `voxtral-tts-26-03`). Query `https://api.mistral.ai/v1/models` to get the current list and update `config/litellm_config.yaml` if a newer version was released.

---

## Repository Structure

```
.
├── config/
│   └── litellm_config.yaml   # Model routing configuration
├── .env.example              # Environment variable template
├── .gitignore
├── docker-compose.yml
├── LICENSE
└── README.md
```
