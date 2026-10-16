# Akkco Code

A modular, terminal-based agentic AI coding assistant for pair programming, repository inspection, and automated tool execution.

Built for the **Hacktoberfest Hack Day** hackathon (MLH), with model inference served through **DigitalOcean Serverless Inference** (OpenAI-compatible, `openai-gpt-oss-20b`).

Akkco Code is a complete harness rather than a thin client: it owns the agent loop, tool execution, context budgeting, conversation memory, and terminal presentation, while models stay behind a provider-neutral contract. The same runtime drives a small local Ollama model or a hosted cloud endpoint without changes to the domain packages.

For detailed system design, contracts, protocol recovery, and architectural specifications, see [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Hackathon Context

Built during the Hacktoberfest Hack Day hackathon as a demonstration that an agentic coding assistant can be provider-neutral, reliable with open-weight models, and fully inspectable end to end.

- **Inference**: DigitalOcean Serverless Inference (`https://inference.do-ai.run/v1`) serving OpenAI `openai-gpt-oss-20b` through native function calling.
- **Open-model reliability**: a strict textual tool protocol (Compatibility Protocol v1.1) with deterministic one-turn repair, so smaller local models without dependable native tool calling can drive the same agent loop.
- **Local-first memory**: every completed turn is indexed into SQLite (`node:sqlite`), and relevant past conversations are retrieved into a token-budgeted context in later sessions.

## Highlights

- **Provider-neutral agent runtime**: multi-turn model → tool → model loop with cycle detection, iteration limits, and cancellation.
- **Repository tools**: `read_file`, `list_files`, and `search_text` with path containment enforcement and Zod-validated inputs.
- **Bounded context compiler**: deterministic assembly order, explicit token budgeting, relevance-filtered conversation memory, and graceful degradation when retrieval or embeddings are unavailable.
- **Persistent conversation memory**: SQLite-backed storage with versioned migrations, checksum validation, and multi-process write locking.
- **Two tool execution modes**: `native` structured tool calling and `compatibility` textual protocol with protocol-violation detection and repair.
- **Terminal UI with PTY tests**: scrollback-safe Ink/React TUI, slash commands, prompt queueing, autocomplete, a plain piped renderer, and an end-to-end pseudo-terminal test harness.

---

## Workspace Structure

- **`apps/cli`**: Interactive Ink/React terminal application, command parser, PTY test harness, and plain piped renderer.
- **`packages/core`**: Core runtime, agent loop orchestration, session state, chronological transcript, context compiler, conversation memory, and loop reliability guards (cycle detection, error limits).
- **`packages/models`**: Provider-neutral contracts, item types, events, tool representations, and test fakes.
- **`packages/providers`**: Concrete model integrations, including OpenAI-compatible endpoints and strict textual tool compatibility protocols for local/small models.
- **`packages/retrieval`**: SQLite persistence (`node:sqlite`), versioned migrations, vector serialization, and similarity search primitives.
- **`packages/tools`**: Repository tools (`read_file`, `list_files`, `search_text`), path containment enforcement, Zod schemas, and model adapters.

---

## Getting Started

### Prerequisites

- Node.js 22.5+ (`node:sqlite`)
- npm
- Native build tools (`make`, `g++`, Python 3) to compile `node-pty` for the PTY test suite

### Installation

```bash
npm install
```

### Configuration

Create `apps/cli/.env` (loaded automatically by `npm run dev` via `tsx --env-file`):

```dotenv
AKKCO_BASE_URL=https://inference.do-ai.run/v1
AKKCO_API_KEY=<your DigitalOcean model access key>
AKKCO_MODEL=openai-gpt-oss-20b
AKKCO_TOOL_MODE=native
```

Environment variables set in your shell take precedence over values in the file, so per-run overrides work:

```bash
AKKCO_MODEL=openai-gpt-oss-120b npm run dev
```

### Running the CLI

```bash
npm run dev                    # interactive terminal UI (TTY)
echo "hello" | npm run dev     # plain stream renderer for piped input
```

### Configuration Reference

| Variable                    | Default                        | Description                                                                                             |
| --------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `AKKCO_BASE_URL`            | `http://localhost:11434/v1`    | OpenAI-compatible model endpoint (Ollama by default).                                                   |
| `AKKCO_API_KEY`             | —                              | Bearer token for the endpoint, if required.                                                             |
| `AKKCO_MODEL`               | `openai-gpt-oss-20b`           | Model name or ID passed to the endpoint.                                                                |
| `AKKCO_TOOL_MODE`           | `compatibility`                | `native` for models with reliable native tool calling (e.g. gpt-oss); `compatibility` for small models. |
| `AKKCO_MAX_TOOL_ITERATIONS` | `25`                           | Maximum tool loop iterations per turn; invalid values fail fast at startup.                             |
| `AKKCO_DB_PATH`             | `~/.akkco/akkco.db`            | SQLite conversation memory location (`:memory:` in test modes).                                         |
| `AKKCO_EMBEDDING_MODEL`     | —                              | Enables vector memory retrieval through an OpenAI-compatible `/embeddings` endpoint.                    |
| `AKKCO_EMBEDDING_BASE_URL`  | falls back to `AKKCO_BASE_URL` | Embedding endpoint base URL.                                                                            |
| `AKKCO_EMBEDDING_API_KEY`   | falls back to `AKKCO_API_KEY`  | Embedding endpoint API key.                                                                             |

When `AKKCO_TOOL_MODE` is omitted, models without dependable native tool calling still work through text envelopes, with automatic single-turn protocol repair if the model malforms a call.

---

## Demo Prompts

Both prompts complete against this repository with `openai-gpt-oss-20b` in `native` mode:

- `Read README.md and summarize Akkco Code in three sentences.` — quick single-tool end-to-end flow.
- `Trace how one tool call is executed in this repo, end to end. Cite file paths.` — multi-step tool loop across the codebase.

---

## Development & Testing

```bash
# Typecheck TypeScript code across all packages
npm run typecheck

# Run unit and component test suites
npm test

# Run interactive PTY end-to-end tests
npm run test:tui

# Check code formatting
npm run format:check

# Run complete verification pipeline
npm run verify
```

`AKKCO_TEST_PROVIDER=1` routes model calls to a deterministic offline provider for reproducible tests without external LLM dependencies.

---

## Architecture & Specifications

For comprehensive details on:

- Multi-turn runtime loops & session transcripts
- Context budgeting, memory retrieval, and graceful degradation
- Reliability v1.1 cycle detection (`detectRepeatedToolCycle`)
- Strict textual tool compatibility protocol & recovery
- Repository relative path discipline & containment protections
- Terminal UI layout, scrolling, and PTY testing harness

Please refer to [ARCHITECTURE.md](./ARCHITECTURE.md).
