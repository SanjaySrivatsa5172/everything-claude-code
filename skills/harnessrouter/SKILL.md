---
name: harnessrouter
description: Deploy, configure, and call HarnessRouter — the unified agent harness infrastructure layer. Routes tasks to Claude Code, Codex, Hermes, and 23 other harnesses through one OpenAI Responses-compatible API. Use when the user is building a product that needs agent harnesses as pluggable backends, wants to benchmark or switch harnesses, needs persistent sessions, file I/O, streaming progress, or task cancellation without per-harness integration work.
origin: ECC
---

# HarnessRouter

HarnessRouter is the infrastructure layer that exposes existing agent harnesses (Claude Code, Codex, Hermes, Gemini CLI, DeepSeek Harness, and 21 others) behind one OpenAI Responses-compatible API via the Unified Harness Protocol (UHP). One integration drives any harness; swap them without touching product code.

**License**: Apache 2.0  
**Source**: https://github.com/HarnessRouter/harnessrouter  
**Protocol spec**: https://unifiedharnessprotocol.org  
**Cloud option**: https://harnessrouter.ai

## When to Activate

- Building a product that uses agent harnesses (Claude Code, Codex, etc.) as AI backends
- Need to switch or compare harnesses without rewriting product integration
- Want persistent sessions, file attachment, streaming progress, task cancellation via a single API
- Benchmarking harness cost and latency across configurations
- Self-hosting agent infrastructure with your own provider keys
- Integrating agent tasks into CI, sheets, slides, dashboards, or any product surface

Do NOT activate for:
- Single-model inference (use the Claude API directly)
- MCP tool development (HarnessRouter uses MCP internally but is not an MCP server)
- Tasks that don't need persistent session state or file I/O

## Architecture

```
Product backend
      │
      ▼  POST /v1/responses  (OpenAI Responses-compatible)
┌─ HarnessRouter container ───────────────────────────────┐
│  Console :3000   ← only published port                  │
│       │ same-origin proxy                               │
│       ▼                                                 │
│  Gateway :8080   UHP + Responses API + lifecycle        │
│       │ loopback                                        │
│       ▼                                                 │
│  Runner  :8081   runs harnesses in session workspaces   │
│                                                         │
│  /data volume  database · files · secrets · workspaces  │
└─────────────────────────────────────────────────────────┘
      │
      ├── Claude Code harness
      ├── Codex harness
      ├── Hermes harness
      └── 23 more harness backends
```

Sessions use separate workspaces and OS users, not separate containers. The Console is the entry point for both UI and API.

## Self-Host Quickstart

**Requirements**: Docker, ~4 GB disk, a provider API key. No HarnessRouter account.

### 1. Start the container

```bash
docker run -d --name harnessrouter \
  -p 127.0.0.1:3000:3000 \
  -v harnessrouter:/data \
  harnessrouter/harnessrouter
```

Follow logs until ready:
```bash
docker logs -f harnessrouter
# Wait for: [harnessrouter] ready on :3000
```

### 2. Sign in and connect a provider

Open http://localhost:3000. Default credentials: `harnessrouter` / `harnessrouter`. **Change the password immediately in Profile.**

Under **Bring your own key**, add your provider API key (Anthropic, OpenAI, etc.).

### 3. Set up an agent (Console path)

Open **Start**, describe what your agent does, press **Continue**. HarnessRouter selects harness, model, and skills automatically. Send the example request to run the first task.

### 4. Get an API key (product integration path)

Open **API Keys** in the Console rail, name the key, press **Create**. Store as `HARNESSROUTER_API_KEY`. This key is separate from your Console password and provider key.

## API Usage

Base URL: `http://localhost:3000/api/harness`

### Run a task (non-streaming)

```bash
curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/responses" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "input": "Write a 3-sentence summary of transformer attention.",
    "metadata": {"harness_id": "claude-code"},
    "model": "claude-sonnet-4-6",
    "stream": false
  }'
```

### Run a task (streaming SSE)

```bash
curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/responses" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "input": "Analyze this codebase and propose three refactoring targets.",
    "metadata": {"harness_id": "claude-code"},
    "model": "claude-sonnet-4-6",
    "stream": true
  }'
```

### Continue a session

```bash
curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/responses" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "input": "Now apply the first refactor.",
    "previous_response_id": "<response_id_from_prior_call>",
    "metadata": {"harness_id": "claude-code"},
    "model": "claude-sonnet-4-6",
    "stream": false
  }'
```

### Attach a file

```bash
# Upload first
FILE_ID=$(curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/files" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -F "file=@myfile.csv" | jq -r .id)

# Reference in task
curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/responses" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -H 'Content-Type: application/json' \
  -d "{
    \"input\": [{\"type\": \"text\", \"text\": \"Analyze this CSV\"},
                {\"type\": \"input_file\", \"file_id\": \"$FILE_ID\"}],
    \"metadata\": {\"harness_id\": \"claude-code\"},
    \"model\": \"claude-sonnet-4-6\",
    \"stream\": false
  }"
```

### Cancel a task

```bash
curl --fail-with-body -sS -X DELETE \
  "$HARNESSROUTER_BASE_URL/v1/responses/$RESPONSE_ID" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY"
```

### Idempotent retries

Use `Idempotency-Key` to retry without duplicating execution. Same key + same body returns the cached result. Same key + different body returns `409`.

```bash
IDEM_KEY=$(python3 -c 'import uuid; print(uuid.uuid4())')
curl --fail-with-body -sS \
  "$HARNESSROUTER_BASE_URL/v1/responses" \
  -H "Authorization: Bearer $HARNESSROUTER_API_KEY" \
  -H 'Content-Type: application/json' \
  -H "Idempotency-Key: $IDEM_KEY" \
  -d '{ ... }'
```

## Harness Selection

`metadata.harness_id` selects the backend. Leave on `auto` to let HarnessRouter choose by cost and latency.

| harness_id | Base harness | Notes |
|---|---|---|
| `claude-code` | Claude Code | Best for filesystem tasks, PRs, long coding sessions |
| `codex` | OpenAI Codex | Fast, low-cost code tasks |
| `hermes` | Hermes | Strong multi-step reasoning |
| `gemini-cli` | Gemini CLI | Gemini models |
| `deepseek-harness` | DeepSeek | Cost-optimized |
| `aider` | Aider | Git-native editing |
| `openhands` | OpenHands | Web browsing + coding |
| `auto` | HarnessRouter picks | Recommended for benchmarking |

Discover all configured harnesses: `GET /v1/harnesses`

## Secrets and Environment

Pass sensitive values through the harness environment, not the prompt. The agent reads `$VAR_NAME`; the value is redacted from all logs, responses, and traces.

```json
{
  "input": "Deploy using $DEPLOY_TOKEN",
  "metadata": {
    "harness_id": "claude-code",
    "env": {
      "DEPLOY_TOKEN": "vault:my-deploy-token"
    }
  }
}
```

Store secrets first: `PUT /v1/mcp-secrets/my-deploy-token` with the value. The `vault:` prefix reads them at runtime. Direct env values (`"ENV_VAR": "literal-value"`) work too but won't be redacted.

Never put tokens, hashes, or IDs in the prompt string — agents retype what they read, and models corrupt long exact strings.

## Distributed Tracing

Send W3C `traceparent`/`tracestate` headers on the request. HarnessRouter propagates a child span to model calls, joining your trace. Invalid W3C syntax is silently dropped, never causes a failure.

## Node.js Client Pattern

```javascript
const resp = await fetch(`${process.env.HARNESSROUTER_BASE_URL}/v1/responses`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.HARNESSROUTER_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    input: task,
    metadata: { harness_id: 'claude-code' },
    model: 'claude-sonnet-4-6',
    stream: false,
  }),
});
const result = await resp.json();
```

For streaming, set `stream: true` and read the `text/event-stream` response body as SSE. Each event is a JSON object; the `delta` event carries incremental output.

## Docker Compose (multi-container setup)

```yaml
# docker-compose.yml
services:
  harnessrouter:
    image: harnessrouter/harnessrouter
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - harnessrouter_data:/data
    environment:
      HR_SECRET_KEY: "${HR_SECRET_KEY}"   # for at-rest encryption; required for Cloud upload

  app:
    build: .
    environment:
      HARNESSROUTER_BASE_URL: http://harnessrouter:3000/api/harness
      HARNESSROUTER_API_KEY: "${HARNESSROUTER_API_KEY}"
    depends_on:
      - harnessrouter

volumes:
  harnessrouter_data:
```

## Moving to HarnessRouter Cloud

1. Set `HR_SECRET_KEY` on your local instance (encrypts the stored destination key).
2. In Console **Settings**, select **Upload to Cloud**.
3. Connect a destination using its Cloud workspace API key.
4. Change `HARNESSROUTER_BASE_URL` to the Cloud endpoint.

Uploads copy harness configuration only — not provider keys, sessions, or generated files.

## Unified Harness Protocol (UHP)

HarnessRouter implements UHP, the open standard for running agent harnesses. UHP is shaped like the OpenAI Responses API but adds:

- **Harness selection** via `metadata.harness_id`
- **Persistent sessions** via `previous_response_id`
- **Structured file I/O** (upload + artifacts)
- **Task lifecycle** (submit, stream, cancel, retrieve)
- **Error taxonomy** (structured failures, not status-code guessing)
- **Idempotency** via `Idempotency-Key`

UHP is NOT:
- A model API (it doesn't replace Anthropic/OpenAI directly)
- MCP (MCP connects agents to tools; UHP connects a product to a full harness)
- A2A (A2A connects agents to agents)

Full spec: `protocol/versions/2026-10-19/` in the repository, or https://unifiedharnessprotocol.org

## API Reference Location

- OpenAPI JSON: `http://localhost:3000/api/harness/v1/openapi.json`
- Browsable docs: `http://localhost:3000/api/harness/v1/docs`
- UHP spec: `protocol/versions/2026-10-19/`

## Common Mistakes

- **Putting tokens in the prompt**: use `env` with `vault:` references instead
- **Ignoring idempotency**: retry network failures with the same `Idempotency-Key` to avoid double execution
- **Hard-coding `harness_id`**: use `auto` when you don't need a specific harness
- **Running without `HR_SECRET_KEY`**: required before uploading to Cloud or storing secrets
- **Exposing the API key in client-side code**: `HARNESSROUTER_API_KEY` must stay server-side only
- **Adding `--user` to the Docker run command**: entrypoint needs root for per-session user management
