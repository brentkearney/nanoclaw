---
name: add-memos
description: Add MemOS persistent memory backend to NanoClaw. Provides semantic similarity search, automatic deduplication, memory evolution, and graph-based knowledge via a self-hosted Docker stack. Entirely opt-in — zero impact when not configured.
---

# Add MemOS Memory Backend

This skill integrates [MemOS](https://memos.openmem.net) as a persistent memory backend for NanoClaw agents. When configured, agents get semantic search across past conversations, automatic memory capture, and explicit memory tools (`search_memories`, `add_memory`, `chat`).

## Phase 1: Pre-flight & Gather Info

Gather all required information up front so the remaining phases can run unattended.

### Check if already applied

Check if `src/memos-client.ts` exists. If it does, skip to Phase 3 (MemOS Stack Setup). The code changes are already in place.

### Check Docker

```bash
docker info > /dev/null 2>&1 && echo "Docker OK" || echo "Docker not available"
```

Docker is required for both the NanoClaw agent containers and the MemOS stack.

### Read assistant name

Read `ASSISTANT_NAME` from `.env` (defaults to "Andy" if not set). Use this name (lowercased) for defaults below.

### Gather user inputs

Use `AskUserQuestion` to collect all required configuration at once:

> I need a few details to set up MemOS. Please provide:
>
> **1. OpenAI-compatible API provider** — MemOS needs an OpenAI-compatible API for two purposes: generating embeddings (vectors for semantic search) and internal memory operations (summarization, deduplication, synthesis). This is separate from the Claude model that NanoClaw uses for agent conversations.
>   - **OpenRouter** (recommended) — supports many models without a direct OpenAI account. Base URL: `https://openrouter.ai/api/v1`
>   - **OpenAI directly** — Base URL: `https://api.openai.com/v1`
>   - **Other** — any OpenAI-compatible endpoint (e.g., local Ollama, LiteLLM, vLLM)
>
> **2. API key** for the provider above
>
> **3. Embedding model** — used for semantic search vectorization. Default: `openai/text-embedding-3-small`. Accept the default or provide a custom model name.
>
> **4. Chat model** — used by MemOS internally for memory summarization, deduplication, and the `chat` tool's synthesis. This is NOT the agent's model (which is Claude). Default: `openai/gpt-4o-mini`. Accept the default or provide a custom model name.
>
> **5. MemOS basic auth password** — used to secure the MemOS API via the reverse proxy. The username will default to `<assistant_name>`. I can generate a strong random password for you, or you can specify your own.
>
> **6. Where to clone MemOS** — the MemOS Docker stack will be cloned here (default: `../MemOS` relative to this project)
>
> **7. Migrate existing memories?** — Should I migrate your existing conversation history and group notes into MemOS? (yes/no)

Store all answers for use in subsequent phases.

### Validate embeddings API

Immediately test the user's embeddings API credentials before proceeding:

```bash
curl -s <OPENAI_BASE_URL>/embeddings \
  -H "Authorization: Bearer <OPENAI_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"input":"test","model":"<embedding_model>"}'
```

A successful response will contain an `embedding` array of floats. If you get a 401 (bad key), 404 (wrong endpoint), or 400 (unsupported model), inform the user what went wrong and loop back to "Gather user inputs" to collect corrected values. Do not proceed until this test passes — MemOS will silently fail to index memories otherwise.

No more user interaction is needed after this point.

## Phase 2: Apply Code Changes

### Ensure remote

```bash
git remote -v
```

If `memos` is missing, add it:

```bash
git remote add memos https://github.com/brentkearney/nanoclaw-memos.git
```

### Merge the skill branch

```bash
git fetch memos main
git merge memos/main || {
  git checkout --theirs package-lock.json
  git add package-lock.json
  git merge --continue
}
```

This merges in:
- `src/memos-client.ts` — HTTP client for MemOS search/add with graceful degradation
- `container/agent-runner/src/memos-mcp-stdio.ts` — MCP server giving agents `search_memories`, `add_memory`, `chat` tools
- `scripts/migrate-memories-to-memos.ts` — One-time migration tool for existing data
- Config exports (`MEMOS_API_URL`, `MEMOS_USER_ID`, `CONTAINER_NETWORK`) in `src/config.ts`
- Auto-recall and auto-capture in `src/index.ts`
- MemOS secrets passing, Docker network joining, and settings sync in `src/container-runner.ts`
- Conditional MemOS MCP server registration in `container/agent-runner/src/index.ts`
- MemOS environment variables in `.env.example`

If the merge reports conflicts, resolve them by reading the intent files in `.claude/skills/add-memos/modify/`:
- `src/config.ts.intent.md`
- `src/index.ts.intent.md`
- `src/container-runner.ts.intent.md`
- `container/agent-runner/src/index.ts.intent.md`
- `.env.example.intent.md`

### Validate code changes

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: MemOS Stack Setup

### Clone MemOS

MemOS runs as a Docker stack with 5 services: `memos-api`, `memos-mcp`, `neo4j`, `qdrant`, and `caddy` (reverse proxy).

Clone to the location specified in Phase 1:

```bash
git clone https://github.com/MemTensor/MemOS.git <clone_path>
cd <clone_path>
```

Refer to the [MemOS documentation](https://memos-docs.openmem.net) and the Docker Compose files in the repo for the available deployment options. The `docker/` directory contains compose configurations.

### Configure the MemOS stack

Create the MemOS `.env` file using values gathered in Phase 1:
- `OPENAI_API_KEY` — the user's embeddings API key
- `OPENAI_BASE_URL` — the user's embeddings API endpoint
- `MOS_CHAT_MODEL` — the chat model from Phase 1 (default: `openai/gpt-4o-mini`)
- `MOS_CUBE_PATH` — **must be a persistent path**, not `/tmp/`. Use a `data/` directory inside the MemOS clone (e.g., `./data/cubes`). This path stores per-user memory cubes. Ensure it is mounted as a Docker volume in `docker-compose.yml` so data survives container recreation.
- Caddy basic auth: username `<assistant_name>`, password from Phase 1

When configuring the Docker Compose file:

1. Add a volume mount for `MOS_CUBE_PATH` on the `memos-api` service so cube data persists
2. Set an explicit network name so it's predictable regardless of directory:

```yaml
services:
  memos-api:
    volumes:
      - ./data/cubes:/app/data/cubes  # persistent cube storage

networks:
  memos:
    name: memos_network
```

This ensures the network is always called `memos_network`. Set `CONTAINER_NETWORK=memos_network` in NanoClaw's `.env` to match.

### Start the stack

```bash
docker compose up -d
```

Verify all 5 services are running:

```bash
docker ps | grep memos
```

### Verify stack is healthy

```bash
curl -s http://localhost:8080/product/search -X POST -H "Content-Type: application/json" -d '{"query":"test","user_id":"test"}' | head -c 200
```

If using basic auth:

```bash
curl -s -u <assistant_name>:<password> http://localhost:8080/product/search -X POST -H "Content-Type: application/json" -d '{"query":"test","user_id":"test"}' | head -c 200
```

A JSON response (even with empty results) means the stack is healthy.

### MemOS API reference

The MemOS Product API uses these endpoints and field names:

- **Add memory**: `POST /product/add` — body: `{"memory_content": "...", "user_id": "..."}`
- **Search memories**: `POST /product/search` — body: `{"query": "...", "user_id": "..."}`
- **Response structure**: Search results are nested: `data.text_mem[].memories[]` — each memory has `memory` (text), `metadata.relativity` (relevance score)

Note: The field for adding is `memory_content`, NOT `text`. Using `text` will silently fail.

### Verify end-to-end memory storage and retrieval

Test the full pipeline — store a memory, search semantically, verify non-zero relevance:

```bash
# Store a test memory
curl -s -u <assistant_name>:<password> http://localhost:8080/product/add -X POST -H "Content-Type: application/json" -d '{"memory_content":"The sky is blue and water is wet","user_id":"test"}'

# Wait a few seconds for async ingestion, then search
sleep 5
curl -s -u <assistant_name>:<password> http://localhost:8080/product/search -X POST -H "Content-Type: application/json" -d '{"query":"what color is the sky","user_id":"test"}'
```

The search should return the memory with a non-zero relevance score. If results are empty or all scores are 0.00:

- Check the MemOS API logs: `docker logs memos-api --tail 50`
- Look for 400/401 errors from the embeddings provider
- Verify `OPENAI_API_KEY` and `OPENAI_BASE_URL` are set correctly in the MemOS stack `.env`

## Phase 4: NanoClaw Configuration

### Set environment variables

Add to NanoClaw's `.env` using the values from Phase 1:

```bash
# MemOS API endpoint (host-side, through reverse proxy)
MEMOS_API_URL=http://localhost:8080/product

# Basic auth credentials for the reverse proxy (user:password)
MEMOS_API_AUTH=<assistant_name>:<password>

# User namespace in MemOS (defaults to assistant name lowercased)
MEMOS_USER_ID=<assistant_name>

# Direct Docker network URL for container access (no auth needed)
# Falls back to MEMOS_API_URL if not set
MEMOS_CONTAINER_API_URL=http://memos-api:8000/product

# Docker network name — containers join this to reach MemOS services
CONTAINER_NETWORK=memos_network
```

### About the reverse proxy

We recommend Caddy because it provides automatic SSL/TLS — important since basic auth credentials travel in plaintext without encryption. Users can substitute Nginx, Traefik, or HAProxy depending on their infrastructure.

## Phase 5: Build & Verify

### Clear stale agent-runner copies

```bash
rm -r data/sessions/*/agent-runner-src 2>/dev/null || true
```

### Rebuild the container

```bash
cd container && ./build.sh
```

### Compile and restart

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

### Test

Tell the user:

> MemOS is connected! Send a message in your main channel. Check the logs for:
>
> - `Injected MemOS memories into prompt` — auto-recall is working
> - The agent should now have `search_memories`, `add_memory`, and `chat` tools
>
> After a conversation ends, the exchange is automatically stored in MemOS.

Monitor logs:

```bash
tail -f logs/nanoclaw.log | grep -iE "(memos|memor)"
```

## Phase 6: Migration (if requested)

Only run this if the user opted in during Phase 1.

### Dry run first

```bash
MEMOS_API_URL=http://localhost:8080/product npx tsx scripts/migrate-memories-to-memos.ts --dry-run
```

### Execute migration

```bash
MEMOS_API_URL=http://localhost:8080/product npx tsx scripts/migrate-memories-to-memos.ts
```

Options:
- `--dry-run` — Preview without sending
- `--user-id=<assistant_name>` — Override MemOS user ID

## Troubleshooting

### MemOS stack not responding

```bash
docker ps | grep memos
docker logs memos-api --tail 50
```

Check that all 5 services are running. Common issue: embedding API key not set or expired.

### Container can't reach MemOS

- Verify `CONTAINER_NETWORK` matches the actual Docker network: `docker network ls | grep memos`
- Test from inside a container: the URL should be `http://memos-api:8000/product` (internal port, no auth)
- Check that `MEMOS_CONTAINER_API_URL` is set in `.env`

### Auto-recall not working

- Verify `MEMOS_API_URL` is set in `.env`
- Check logs for `searchMemories` errors
- Test the API directly: `curl -u <assistant_name>:<password> http://localhost:8080/product/search -X POST -H "Content-Type: application/json" -d '{"query":"test","user_id":"<assistant_name>"}'`

### Agent doesn't have memory tools

- Clear stale agent-runner: `rm -r data/sessions/*/agent-runner-src 2>/dev/null || true`
- Rebuild container: `cd container && ./build.sh`
- Verify `memos-mcp-stdio.ts` is in `container/agent-runner/src/`

### Relevance scores are all 0.00

- Check that the embedding API is working (MemOS logs will show 400 errors if not)
- Verify `OPENAI_API_KEY` and `OPENAI_BASE_URL` in the MemOS stack `.env`
- Scores improve as more data is ingested

## Removal

1. Remove `.env` variables: `MEMOS_API_URL`, `MEMOS_API_AUTH`, `MEMOS_USER_ID`, `MEMOS_CONTAINER_API_URL`, `CONTAINER_NETWORK`
2. Delete `src/memos-client.ts`
3. Delete `container/agent-runner/src/memos-mcp-stdio.ts`
4. Delete `scripts/migrate-memories-to-memos.ts`
5. Revert MemOS changes in `src/config.ts`, `src/index.ts`, `src/container-runner.ts`, `container/agent-runner/src/index.ts` (use the intent.md files to identify what to remove)
6. Clear stale agent-runner copies: `rm -r data/sessions/*/agent-runner-src 2>/dev/null || true`
7. Rebuild: `cd container && ./build.sh && cd .. && npm run build`
8. Restart: `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `systemctl --user restart nanoclaw` (Linux)
