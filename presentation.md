# Docker for LLMs, Agents & MCP

> Containerising the AI Stack from Inference to Orchestration

**Brendan Lynskey | 2026-04-14**

`LLM` → `Agent` → `MCP Server` → `Gateway` → `Client`

---

## 01 — Why Containerise LLMs?

### Reproducibility
Pin CUDA, Python, and library versions. No more "works on my machine" for inference.

### GPU Isolation
Assign specific GPUs per container. Run multiple models on one host without conflicts.

### Dependency Hell
torch, transformers, CUDA, cuDNN — each model needs its own universe. Containers solve this.

### Portability
Build once, deploy anywhere — laptop, on-prem GPU server, or cloud VM with identical behaviour.

### Security
Isolate model serving from host. Limit network, filesystem, and process access with container boundaries.

---

## 02 — GPU Passthrough with Docker

### nvidia-container-toolkit Setup

```bash
# Install the NVIDIA Container Toolkit
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify GPU access inside a container
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

### Key Flags

| Flag | Effect |
|------|--------|
| `--gpus all` | Expose every GPU to the container |
| `--gpus '"device=0,1"'` | Expose only GPU 0 and 1 |
| `--runtime=nvidia` | Use NVIDIA runtime (alternative to --gpus) |
| `--shm-size=1g` | Increase shared memory for large model loading |
| `NVIDIA_VISIBLE_DEVICES` | Environment variable alternative to --gpus |

---

## 03 — Running Ollama in Docker

### Quick Start

```bash
# CPU only
docker run -d -p 11434:11434 \
  --name ollama \
  -v ollama_data:/root/.ollama \
  ollama/ollama

# With GPU
docker run -d --gpus all \
  -p 11434:11434 \
  --name ollama \
  -v ollama_data:/root/.ollama \
  ollama/ollama
```

### Pulling & Running Models

```bash
# Pull a model
docker exec ollama ollama pull llama3.1:8b

# Run inference
curl http://localhost:11434/api/generate \
  -d '{
    "model": "llama3.1:8b",
    "prompt": "Explain Docker in one sentence",
    "stream": false
  }'
```

**Key insight:** Mount a named volume at `/root/.ollama` so model weights persist across container restarts. Multiple Ollama containers can share the same volume read-only.

---

## 04 — vLLM, TGI & llama.cpp in Docker

### vLLM

```bash
docker run --gpus all \
  -p 8000:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  vllm/vllm-openai:latest \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --max-model-len 8192 \
  --tensor-parallel-size 2
```

OpenAI-compatible API. PagedAttention for throughput.

### Text Generation Inference (TGI)

```bash
docker run --gpus all \
  -p 8080:80 \
  -v ~/models:/data \
  ghcr.io/huggingface/text-generation-inference:latest \
  --model-id meta-llama/Llama-3.1-8B-Instruct \
  --quantize awq
```

HuggingFace's production server. Flash Attention, AWQ support.

### llama.cpp (GGUF)

```bash
docker run --gpus all \
  -p 8080:8080 \
  -v ~/models:/models \
  ghcr.io/ggerganov/llama.cpp:server \
  -m /models/llama-3.1-8b.Q4_K_M.gguf \
  --n-gpu-layers 35 \
  --ctx-size 4096
```

CPU-friendly. GGUF quantised models. Great for edge.

---

## 05 — Model Weight Management

### Volume Strategies

- **Named volumes** — `docker volume create models`
- **Bind mounts** — `-v ~/models:/models`
- **Shared cache** — Mount HuggingFace cache read-only across containers
- **Multi-stage builds** — Download weights at build time, bake into image

### Best Practices

- Never bake large model weights into images (10GB+ layers)
- Use `:ro` flag for shared model volumes
- Set `HF_HOME` and `TRANSFORMERS_CACHE` to the mounted path
- Pre-pull models in an init container or entrypoint script
- Use a model registry (MLflow, DVC) for version control

```yaml
# docker-compose.yml — shared model cache
volumes:
  hf_cache:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/huggingface_cache

services:
  vllm:
    image: vllm/vllm-openai:latest
    volumes:
      - hf_cache:/root/.cache/huggingface:ro
```

---

## 06 — Multi-Model Serving Patterns

### Sidecar Pattern
- Each agent container gets its own LLM sidecar
- Communication via localhost
- Simple but resource-heavy

### Shared Service Pattern
- Centralised LLM service on the Docker network
- Nginx or Traefik as reverse proxy
- Route by model name to specific containers
- Better GPU utilisation

```yaml
# Route by model name
services:
  llm-small:
    image: ollama/ollama
    deploy:
      resources:
        reservations:
          devices: [{ capabilities: [gpu], count: 1 }]
  llm-large:
    image: vllm/vllm-openai:latest
    command: ["--model", "meta-llama/Llama-3.1-70B-Instruct", "--tensor-parallel-size", "4"]
    deploy:
      resources:
        reservations:
          devices: [{ capabilities: [gpu], count: 4 }]
```

---

## 07 — Why Agents Need Containerisation

### Sandboxing
Agents execute arbitrary code — file writes, shell commands, API calls. Containers limit the blast radius.

### Tool Isolation
Each tool (code interpreter, browser, database) runs in its own container with minimal permissions.

### Reproducibility
Agent behaviour depends on installed packages, API keys, and system state. Containers freeze all of it.

### Scaling
- Spin up agent instances per user/request
- Horizontal scaling with Docker Compose `--scale`
- Ephemeral containers for stateless agents

### Security Boundaries
- Read-only filesystems (`--read-only`)
- No network access (`--network=none`)
- Drop all capabilities (`--cap-drop=ALL`)
- Resource limits (`--memory`, `--cpus`)

---

## 08 — Containerising LangChain / LangGraph Agents

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY agent/ ./agent/
COPY tools/ ./tools/

ENV OPENAI_API_KEY=""
ENV LANGCHAIN_TRACING_V2=true

EXPOSE 8000
CMD ["uvicorn", "agent.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### LangGraph Agent Server

```python
# agent/server.py
from fastapi import FastAPI
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

app = FastAPI()
llm = ChatOpenAI(model="gpt-4o")
agent = create_react_agent(llm, tools=[...])

@app.post("/invoke")
async def invoke(query: str):
    result = await agent.ainvoke({"messages": [("user", query)]})
    return result
```

**Tip:** Use `langgraph deploy` for production — it generates an optimised Dockerfile with checkpointing and streaming built in.

---

## 09 — Containerising CrewAI Multi-Agent Systems

### CrewAI Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app

RUN pip install crewai crewai-tools
COPY crew_config/ ./crew_config/
COPY main.py .

CMD ["python", "main.py"]
```

### Multi-Agent Crew

```python
# main.py
from crewai import Agent, Task, Crew

researcher = Agent(
    role="Researcher",
    goal="Find accurate information",
    llm="ollama/llama3.1:8b",
    llm_config={"base_url": "http://ollama:11434"}
)
writer = Agent(
    role="Writer",
    goal="Write clear reports",
    llm="ollama/llama3.1:8b",
    llm_config={"base_url": "http://ollama:11434"}
)
crew = Crew(agents=[researcher, writer], tasks=[...])
crew.kickoff()
```

**Pattern:** CrewAI agents point to `http://ollama:11434` — the Docker Compose service name. No localhost, no IP addresses.

---

## 10 — Agent Stack: Ollama + ChromaDB + Agent

```yaml
version: "3.9"
services:
  ollama:
    image: ollama/ollama
    ports: ["11434:11434"]
    volumes: [ollama_data:/root/.ollama]
    deploy:
      resources:
        reservations:
          devices: [{ capabilities: [gpu], count: 1 }]

  chromadb:
    image: chromadb/chroma:latest
    ports: ["8000:8000"]
    volumes: [chroma_data:/chroma/chroma]
    environment:
      - ANONYMIZED_TELEMETRY=false

  agent:
    build: ./agent
    depends_on: [ollama, chromadb]
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - CHROMA_HOST=chromadb
      - CHROMA_PORT=8000
    ports: ["8080:8080"]

volumes:
  ollama_data:
  chroma_data:
```

Flow: `Agent :8080` → `Ollama :11434` ← `ChromaDB :8000`

---

## 11 — What is MCP (Model Context Protocol)?

MCP is an **open standard** (by Anthropic) that lets LLMs interact with external systems through a unified protocol.

### Tools
Functions the model can call — run SQL, search the web, call APIs, execute code.

### Resources
Data the model can read — files, database rows, API responses, live metrics.

### Prompts
Reusable prompt templates exposed by the server for common workflows.

Flow: `LLM / Agent` → `MCP Client` → `MCP Server` → `External System`

**Key idea:** MCP replaces bespoke tool integrations with a single protocol. One client can talk to any MCP server — like USB-C for AI tools.

---

## 12 — Why Run MCP Servers in Docker?

### Isolation & Security
- Each MCP server gets its own filesystem and network namespace
- A compromised filesystem tool cannot access the database server
- Secrets are scoped per container via environment variables
- Read-only filesystem for servers that only need to read

### Portability & Ops
- Ship MCP servers as images — no Node/Python/Rust install required
- Version-pin server images for reproducible deployments
- Health checks and auto-restart via Docker Compose
- Logging via `docker logs` with structured JSON output

### Transport Modes

| Transport | Docker Approach | Use Case |
|-----------|----------------|----------|
| **stdio** | `docker run -i` (interactive, pipe stdin/stdout) | Local dev, single client |
| **SSE** | `docker run -p 3001:3001` (HTTP endpoint) | Remote clients, multi-tenant |
| **Streamable HTTP** | `docker run -p 3001:3001` (HTTP endpoint) | Modern MCP, bidirectional |

---

## 13 — Containerising MCP Servers

### Dockerfile for an MCP Server

```dockerfile
FROM node:22-slim
WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY src/ ./src/

# SSE transport on port 3001
ENV MCP_TRANSPORT=sse
ENV MCP_PORT=3001
EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -f http://localhost:3001/health || exit 1

CMD ["node", "src/index.js"]
```

### MCP Client Configuration

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-v", "/data:/data:ro",
        "mcp/filesystem-server",
        "/data"
      ]
    },
    "postgres": {
      "url": "http://localhost:3002/sse",
      "env": {
        "POSTGRES_URL": "postgresql://..."
      }
    }
  }
}
```

**stdio via Docker:** Use `docker run -i` so the MCP client can pipe JSON-RPC over stdin/stdout. The container acts like a local binary.

---

## 14 — Docker Compose for MCP Server Stacks

```yaml
version: "3.9"
services:
  mcp-filesystem:
    image: mcp/filesystem-server
    volumes:
      - ./workspace:/data:ro
    environment:
      - MCP_TRANSPORT=sse
      - MCP_PORT=3001
    ports: ["3001:3001"]

  mcp-postgres:
    image: mcp/postgres-server
    environment:
      - POSTGRES_URL=postgresql://user:pass@db:5432/mydb
      - MCP_TRANSPORT=sse
      - MCP_PORT=3002
    ports: ["3002:3002"]
    depends_on: [db]

  mcp-brave-search:
    image: mcp/brave-search
    environment:
      - BRAVE_API_KEY=${BRAVE_API_KEY}
      - MCP_TRANSPORT=sse
      - MCP_PORT=3003
    ports: ["3003:3003"]

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=pass
    volumes: [pgdata:/var/lib/postgresql/data]

volumes:
  pgdata:
```

---

## 15 — What is an MCP Gateway?

An MCP Gateway sits between clients and MCP servers, providing a **single endpoint** that aggregates, secures, and routes tool calls across multiple backends.

Flow:
```
Claude / Agent  →  MCP Gateway  →  MCP: Filesystem
                                →  MCP: Database
                                →  MCP: Search API
```

### Aggregation
Combine tools from multiple MCP servers into a single tool list for the client.

### Auth & Access Control
API keys, OAuth, RBAC — enforce who can call which tools.

### Protocol Bridge
Convert stdio servers to SSE/HTTP. Translate between transport modes.

---

## 16 — Supergateway & SSE Bridges

### Supergateway

Wraps any **stdio MCP server** and exposes it as an **SSE endpoint**.

```bash
docker run -p 3001:3001 \
  supercorp/supergateway \
  --stdio "npx -y @modelcontextprotocol/server-filesystem /data" \
  --port 3001
```

- Turns local-only servers into network-accessible services
- No code changes to the MCP server needed
- Supports multiple simultaneous client connections

### mcp-proxy

Lightweight proxy that bridges between SSE and stdio transports.

```bash
# SSE to stdio bridge
docker run -p 3002:3002 \
  sparfenyuk/mcp-proxy \
  --sse-port 3002 \
  --command "python mcp_server.py"

# Connect remote SSE from a stdio client
mcp-proxy --sse-url http://remote:3001/sse
```

**Why bridges matter:** Claude Desktop and Cursor only support stdio. Bridges let you run MCP servers anywhere and connect them to local clients.

---

## 17 — Docker Compose: Gateway + MCP Servers

```yaml
version: "3.9"
services:
  gateway:
    image: supercorp/supergateway
    ports: ["3000:3000"]
    environment:
      - GATEWAY_PORT=3000
      - AUTH_TOKEN=${GATEWAY_AUTH_TOKEN}
    depends_on: [mcp-filesystem, mcp-postgres, mcp-search]

  mcp-filesystem:
    image: mcp/filesystem-server
    volumes: [./workspace:/data:ro]
    environment: { MCP_TRANSPORT: sse, MCP_PORT: "3001" }
    expose: ["3001"]

  mcp-postgres:
    image: mcp/postgres-server
    environment:
      POSTGRES_URL: postgresql://user:pass@db:5432/app
      MCP_TRANSPORT: sse
      MCP_PORT: "3002"
    expose: ["3002"]
    depends_on: [db]

  mcp-search:
    image: mcp/brave-search
    environment:
      BRAVE_API_KEY: ${BRAVE_API_KEY}
      MCP_TRANSPORT: sse
      MCP_PORT: "3003"
    expose: ["3003"]

  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: pass }
    volumes: [pgdata:/var/lib/postgresql/data]

volumes:
  pgdata:
```

**Note:** Internal MCP servers use `expose` (not `ports`) — only the gateway is publicly accessible.

---

## 18 — Gateway Auth & Access Control

### Authentication Patterns
- **Bearer tokens** — Simple API key in Authorization header
- **OAuth 2.0 / OIDC** — MCP spec supports OAuth flows natively
- **mTLS** — Client certificates for service-to-service
- **API gateway** — Kong, Traefik, or Envoy in front of MCP

### Access Control
- **Tool-level RBAC** — User A can read files, User B can write
- **Rate limiting** — Prevent runaway agent tool loops
- **Audit logging** — Log every tool call with user, input, output
- **Allowlists** — Restrict which tools are exposed per client

```yaml
# Gateway config with auth
gateway:
  auth:
    type: bearer
    tokens:
      - token: ${ADMIN_TOKEN}
        role: admin
        allowed_tools: ["*"]
      - token: ${READONLY_TOKEN}
        role: reader
        allowed_tools: ["filesystem.read", "search.*"]
  rate_limit:
    requests_per_minute: 60
    burst: 10
```

---

## 19 — Production MCP Gateway Patterns

### Gateway Playground

```yaml
services:
  gateway:
    image: supercorp/supergateway
    ports: ["3000:3000"]

  web-ui:
    image: mcp/inspector
    ports: ["5173:5173"]
    environment:
      - MCP_SERVER_URL=http://gateway:3000/sse

  mcp-everything:
    image: mcp/everything-server
    expose: ["3001"]
```

Use **MCP Inspector** as a web UI to test and debug tool calls through the gateway.

### High Availability
- **Replicas** — `deploy: { replicas: 3 }` for gateway instances
- **Load balancer** — Traefik or HAProxy in front of gateway replicas
- **Health checks** — Gateway pings each MCP server; removes unhealthy ones
- **Circuit breakers** — Fail fast if an MCP server is down
- **Graceful degradation** — Return partial tool list if some servers are unavailable

### Observability Stack
Add Prometheus + Grafana to monitor tool call latency, error rates, and token usage across all MCP servers. Export logs to Loki for centralised search.

---

## 20 — Health Checks & Monitoring

### LLM Container Health Checks

```yaml
services:
  ollama:
    image: ollama/ollama
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### GPU Monitoring

```bash
# Real-time GPU stats
docker run --gpus all --rm \
  nvidia/cuda:12.4.0-base-ubuntu22.04 \
  nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total,temperature.gpu \
  --format=csv -l 5

# DCGM Exporter for Prometheus
docker run --gpus all -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:latest
```

### Key Metrics to Monitor

| Category | Metrics |
|----------|---------|
| **LLM Inference** | Tokens/second, Time to first token, Queue depth |
| **MCP Servers** | Tool call latency, Error rate, Active connections |
| **GPU** | VRAM usage %, Compute utilisation %, Temperature |

---

## 21 — Scaling & Cost Optimisation

### Scaling Strategies

| Strategy | Tool | Best For |
|----------|------|----------|
| Horizontal replicas | Compose `--scale` | Stateless agents |
| GPU scheduling | Kubernetes + GPU operator | Multi-model clusters |
| Serverless inference | KNative / Cloud Run GPU | Bursty workloads |
| Model sharding | vLLM tensor parallelism | Large models |

### Cost Optimisation

- **Right-size GPU allocation** — Use `nvidia-smi` to find actual VRAM usage, then constrain
- **Quantisation** — AWQ/GPTQ/GGUF reduce VRAM by 50-75%
- **Model caching** — Shared volumes avoid redundant downloads
- **Auto-scaling to zero** — Scale down idle inference containers
- **Spot/preemptible instances** — Use for batch inference workloads

```yaml
# Kubernetes GPU scheduling example
resources:
  limits:
    nvidia.com/gpu: 1
    memory: "16Gi"
  requests:
    nvidia.com/gpu: 1
    memory: "12Gi"
nodeSelector:
  gpu-type: "a100-40gb"
```

---

## 22 — Summary & Key Takeaways

### LLMs in Docker
- Use `--gpus` + nvidia-container-toolkit
- Mount model weights as volumes, never bake into images
- Ollama, vLLM, and TGI all have official Docker images

### Agents in Docker
- Sandbox agent tool execution in containers
- Docker Compose for agent + LLM + vector DB stacks
- Use service names for inter-container networking

### MCP in Docker
- stdio via `docker run -i`, SSE via port mapping
- Isolate each MCP server in its own container
- Environment variables for secrets and configuration

### MCP Gateways
- Single entry point for all MCP servers
- Auth, rate limiting, and audit logging at the gateway
- Supergateway bridges stdio servers to SSE

`LLM` → `Agent` → `MCP Server` → `Gateway` → `Client`

---

## 23 — Further Reading & Resources

### LLM Serving
- [ollama.com](https://ollama.com) — Ollama documentation
- [docs.vllm.ai](https://docs.vllm.ai) — vLLM docs
- [HuggingFace TGI](https://huggingface.co/docs/text-generation-inference) — Text Generation Inference
- [llama.cpp](https://github.com/ggerganov/llama.cpp) — CPU/GPU inference

### MCP Ecosystem
- [modelcontextprotocol.io](https://modelcontextprotocol.io) — MCP specification
- [MCP Servers repo](https://github.com/modelcontextprotocol/servers) — Official server implementations
- [Supergateway](https://github.com/supercorp-ai/supergateway) — stdio to SSE bridge

### Agent Frameworks
- [LangGraph](https://langchain-ai.github.io/langgraph/) — Agent orchestration
- [CrewAI](https://docs.crewai.com) — Multi-agent framework
- [Claude API](https://docs.anthropic.com) — Anthropic documentation

### Docker & GPU
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit)
- [Docker Compose](https://docs.docker.com/compose/) — Multi-container apps
- [Kubernetes GPU scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

---

*Brendan Lynskey — 2026*
