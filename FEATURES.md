# Orchestration Membrane – Project Board

Goal: A Go “membrane” service that:
- Talks to an orchestration model
- Exposes tools (MCP / OpenAI tools)
- Routes calls to other local models (Ollama + friends) and system tools
- Runs on DGX Spark via Docker Compose

---

## 0. Meta / Housekeeping

### META-01 – Create repo + base layout
- [ ] Create Git repo: `dgx-orchestrator` (or similar)
- [ ] Add folders: `cmd/`, `internal/`, `deploy/`, `docs/`
- [ ] Add `go.mod`

### META-02 – Add TASKS.md and basic conventions
- [ ] Add this board as `TASKS.md`
- [ ] Define simple label scheme in comments (e.g. `[core] [infra] [ux]`)
- [ ] Decide branch naming (e.g. `feature/<id>-short-name`)

---

## 1. Core Domain Design (Calls, Models, Tools)

### CORE-01 – Define core `Call` interface
- [ ] Create `internal/calls/call.go`
- [ ] Define `CallKind`, `Call` interface (`Name()`, `Kind()`, `Run(...)`)
- [ ] Add `CallRouter` with `Register()` + `Handle()`

### CORE-02 – Add `ModelCall` and `ToolCall` markers
- [ ] Extend `call.go` with `ModelCall` and `ToolCall` marker interfaces
- [ ] Add helpers `ModelCalls()` and `ToolCalls()` on `CallRouter` (optional)

### CORE-03 – Implement `CodeModelCall`
- [ ] Create `internal/calls/code_model_call.go`
- [ ] Define `CodeModelArgs` struct (prompt, system, temperature, model override)
- [ ] Implement `CodeModelCall` that calls a model via OpenAI-style client

### CORE-04 – Implement `HTTPGetCall`
- [ ] Create `internal/calls/http_get_call.go`
- [ ] Define `HTTPGetArgs` (url, headers)
- [ ] Implement `HTTPGetCall` using shared `*http.Client`

---

## 2. Ollama Integration (Model Discovery + Model-Tools)

### OLL-01 – Basic Ollama client
- [ ] Create `internal/ollama/client.go`
- [ ] Implement `ListModels(baseURL string) (map[string]struct{}, error)` using `/api/tags`
- [ ] Add small log helper to print discovered models on startup

### OLL-02 – Define `ModelToolSpec`
- [ ] Create `internal/modeltools/spec.go`
- [ ] Define `ModelToolSpec { ToolName, ModelName, Description, System }`
- [ ] Add `DefaultModelTools` slice with a few logical tools:
  - code_assistant → `llama3:8b` (example)
  - embedding_model → `nomic-embed-text`
  - vision_captioner → `llava:7b` (or placeholder)

### OLL-03 – Implement `OllamaModelCall`
- [ ] Create `internal/modeltools/ollama_model_call.go`
- [ ] Implement `OllamaModelCall` (implements `ModelCall`)
- [ ] POST to `/api/chat` with `model`, `messages` array
- [ ] Return `{ "response": "<content>" }` as `map[string]any`

### OLL-04 – Startup wiring: register existing model tools
- [ ] In `cmd/server/main.go`, after `ListModels`:
  - [ ] Loop over `DefaultModelTools`
  - [ ] For each spec where model exists, create `OllamaModelCall`
  - [ ] Register with `CallRouter`
- [ ] Log: `"registered model-call tool %q -> model %q"`

### OLL-05 – Periodic refresh (optional)
- [ ] Add ticker to refresh Ollama model list every N minutes
- [ ] Log any newly available model tools

---

## 3. Orchestrator Integration (Big Brain)

> This is the main “reasoning” model (could be OpenAI API now, later DGX local).

### ORCH-01 – Define orchestrator client interface
- [ ] Create `internal/orch/client.go`
- [ ] Define interface `OrchestratorClient` with `Chat(...)` method
- [ ] Provide basic OpenAI-backed implementation

### ORCH-02 – Define tool schema for `code_assistant` etc.
- [ ] Create `internal/orch/toolschema.go`
- [ ] Define Go structs for tool definitions (name, description, JSON schema)
- [ ] Generate `[]ToolDefinition` from `ModelToolSpec` + other calls

### ORCH-03 – Orchestrator call loop (simple version)
- [ ] Create `internal/orch/loop.go`
- [ ] Implement function:
  - Takes user message + tool list
  - Sends to orchestrator model
  - If response includes tool_call:
    - Call `CallRouter.Handle(...)`
    - Feed result back as tool output
  - Return final text

> Don’t overcomplicate this at first; just get 1 tool working E2E.

---

## 4. API Layer (Expose Membrane as HTTP Service)

### API-01 – Minimal HTTP server
- [ ] Create `cmd/server/main.go` with `net/http` or `chi`
- [ ] Add `/healthz` endpoint
- [ ] Add `/v1/chat` endpoint (internal format, not OpenAI yet)

### API-02 – Wire `/v1/chat` to orchestrator loop
- [ ] Define request struct: `{ "input": "string", "session_id": "optional" }`
- [ ] Call orchestrator loop with:
  - user message
  - current tool list from `CallRouter`
- [ ] Return response as `{ "output": "string", "trace": "optional" }`

### API-03 – OpenAI-compatible shim (optional)
- [ ] Add `/v1/chat/completions` endpoint
- [ ] Map OpenAI-style request → internal `ChatRequest`
- [ ] Return OpenAI-style response so UIs like AnythingLLM can plug in

---

## 5. MCP Integration (Optional but Future-Proof)

### MCP-01 – Add basic MCP server
- [ ] Create `cmd/mcp-server/main.go`
- [ ] Add dependency on `github.com/modelcontextprotocol/go-sdk/mcp`
- [ ] Create MCP server with implementation name `dgx-membrane`

### MCP-02 – Expose `Call`s as MCP tools
- [ ] Add adapter that:
  - Iterates over `CallRouter`’s calls
  - Registers each as MCP tool with `Name()` + description
  - Handler calls `CallRouter.Handle(...)`

### MCP-03 – Document how to connect a MCP client
- [ ] Add section in `docs/mcp.md`
- [ ] Explain transport (stdio / TCP)
- [ ] Show sample config for a MCP-aware agent (e.g. OpenAI Agents, Claude desktop)

---

## 6. Docker & DGX Spark Setup

### DOCKER-01 – Dockerfile for Go membrane
- [ ] Create `Dockerfile` for Go server (multi-stage build)
- [ ] Expose port (e.g. 8080)
- [ ] Set minimal env variables (OLLAMA_BASE_URL, ORCH_BASE_URL, etc.)

### DOCKER-02 – docker-compose.yml (local)
- [ ] Add services:
  - `ollama`
  - `go-membrane`
- [ ] Set `depends_on: [ollama]` for `go-membrane`
- [ ] Add basic healthcheck for `go-membrane` and `ollama`

### DOCKER-03 – DGX-specific notes
- [ ] Add `docs/dgx.md`
- [ ] Document:
  - GPU visibility (`NVIDIA_VISIBLE_DEVICES`)
  - How orchestrator model container will be added later
  - Any NVIDIA NIM integration ideas

---

## 7. Observability & Safety

### OBS-01 – Structured logging
- [ ] Use a logging lib or standard logger
- [ ] Include call name, kind, and duration per call
- [ ] Log external HTTP errors clearly

### OBS-02 – Basic metrics (optional)
- [ ] Add simple counters / histograms (if you want Prometheus)
- [ ] Expose `/metrics` (optional)

### SAFE-01 – Timeouts & limits
- [ ] Add context timeouts when calling Ollama and orchestrator
- [ ] Add max prompt size / response size checks

---

## 8. Nice-to-Haves / Future Ideas (Backlog)

- [ ] UI to show:
  - available model tools
  - their backing models
- [ ] Hot-reload of `ModelToolSpec` from config file
- [ ] Per-session memory / context store
- [ ] Per-tool rate limiting (so a bad prompt can’t hammer one model)
- [ ] Automatic “capabilities card” generator for the orchestrator prompt (“You have access to these tools…”)

---

## Working Style Suggestions

- Keep tickets **tiny** — if a task in this list feels big, split it.
- Try to finish **1–3 tickets per coding session** so you feel momentum.
- If you change direction, add a new section to `TASKS.md` called `IDEAS / BRAIN DUMP` and throw raw thoughts in there so they don’t clog your head.
