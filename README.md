# mlx-serve

mlx-serve is an OpenAI-compatible server for MLX language models on Apple
Silicon. It supports streaming chat and completions, embeddings, continuous
batching, structured JSON output, prompt caching, multiple loaded models,
tool/function calling, and structured reasoning extraction.

## Requirements

- A Mac with Apple Silicon
- Homebrew

## Install via Homebrew

```sh
brew tap yannick/tap
brew trust yannick/tap
brew install mlx-serve
```

Upgrade later with `brew update && brew upgrade mlx-serve`.

## Quick start

```sh
mlx-serve setup
mlx-serve tui --config ./mlx-serve.yaml
```

Setup creates `./mlx-serve.yaml`, discovers compatible models in LM Studio,
the Hugging Face cache, and `~/mlx-models`, then offers to download the small
`mlx-community/Qwen2.5-0.5B-Instruct-4bit` test model. The default model
directory is `~/mlx-models`.

Press `s` in the TUI to start the server, or start it directly:

```sh
mlx-serve serve --config ./mlx-serve.yaml
```

The API listens on `http://127.0.0.1:8080` by default. Use
`mlx-serve setup --yes` for unattended setup or `--no-download` to create the
config and discover existing models only.

## TUI controls

The dashboard includes Hugging Face search, discovered models, configured
models, editable YAML, streaming chat, and live machine/server statistics.

- `1`–`5`: switch screens
- `s`: start or gracefully stop the server
- `/`: search Hugging Face
- `o` / `O`: change or reverse the search sort order
- `d` / `x`: start or stop a resumable download
- `a`: add a discovered model to the config
- `Space`, then `l` / `u`: load or unload selected models
- `e`, then `Ctrl-S`: edit, validate, and save the config
- `Enter`: send a chat message
- `Tab`: change the loaded chat model
- `/clear` or `Ctrl-N`: start a new chat
- `q`: stop active work and quit

## Discover and download models

```sh
# Scan common locations.
mlx-serve scan --config ./mlx-serve.yaml

# Include additional locations.
mlx-serve scan --dirs /Volumes/models,/opt/models \
  --config ./mlx-serve.yaml

# Download real Granite checkpoints.
mlx-serve download mlx-community/granite-4.1-8b-4bit \
  --config ./mlx-serve.yaml
mlx-serve download mlx-community/granite-4.1-8b-8bit \
  --config ./mlx-serve.yaml
mlx-serve download \
  usermma/Granite-4.1-30B-Claude-4.6-Opus-Thinking-Charles-Xavier-mlx-2Bit \
  --config ./mlx-serve.yaml
```

`HF_TOKEN` authenticates private or gated Hugging Face downloads. Interrupted
TUI downloads retain partial data and resume when selected again.

```sh
mlx-serve models list --cache-dir ~/mlx-models
mlx-serve models gc --cache-dir ~/mlx-models
```

Garbage collection is a dry run unless `--yes` is supplied.

## Configuration

Setup writes a complete configuration. A compact example is:

```yaml
server:
  host: 127.0.0.1
  port: 8080
  auth_token: null

cache:
  dir: ~/mlx-models
  offline: false

models:
  - id: granite
    repo: mlx-community/granite-4.1-8b-4bit
    revision: main
    context: 8192
    slots: 4
    load_on_start: true
    load_on_demand: true
    tool_format: json
    reasoning_format: none
    enable_thinking: false

limits:
  max_queue_depth: 64
  max_tokens_default: 1024
  request_timeout_s: 300
  prefix_cache:
    enabled: true
    max_entries: 256

logging:
  level: info
  format: json
  request_log: true

metrics:
  enabled: true
```

Validate edits without starting the server:

```sh
mlx-serve config-check --config ./mlx-serve.yaml
```

`load_on_start` loads a model during startup. `load_on_demand` allows an API
request to load it when needed. Both may be enabled together. If both are
false, load the model manually from the TUI or API.

## OpenAI-compatible API

The main endpoints are:

- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/embeddings`
- `GET /v1/models`
- `POST /v1/models/{id}/load`
- `DELETE /v1/models/{id}/load`
- `GET /health/live` and `GET /health/ready`
- `GET /metrics` when metrics are enabled

```sh
curl http://127.0.0.1:8080/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{
    "model":"granite",
    "messages":[{"role":"user","content":"Say hello briefly."}],
    "stream":false
  }'
```

Set `"stream": true` for Server-Sent Events. `response_format` supports
`json_object` and compatible `json_schema` requests for constrained JSON.

Multiple configured models can be resident when memory permits:

```sh
curl -X POST http://127.0.0.1:8080/v1/models/granite/load
curl -X DELETE http://127.0.0.1:8080/v1/models/granite/load
```

If authentication is enabled, set `auth_token: env:MLX_SERVE_TOKEN` and send
`Authorization: Bearer $MLX_SERVE_TOKEN` with `/v1` requests.

## Tool calling and reasoning

Tool and reasoning formats are explicit per model. Granite and Qwen 2.5 use
`tool_format: json`; Qwen 3.5 models with nested `<function>` and `<parameter>`
tags use `tool_format: qwen35`.

```yaml
models:
  - id: granite-thinking
    repo: usermma/Granite-4.1-30B-Claude-4.6-Opus-Thinking-Charles-Xavier-mlx-2Bit
    load_on_start: true
    load_on_demand: true
    tool_format: json
    reasoning_format: think_tags
    enable_thinking: true
```

Chat Completions accepts `tools`, `tool_choice`, legacy `functions` and
`function_call`, assistant `tool_calls`, and `tool` result messages. Responses
provide stable call IDs and `finish_reason: "tool_calls"`; extracted thinking
is returned separately as `reasoning_content` in regular and streaming output.

The client executes each requested function and returns its result with the
full chat history. `tool_choice: "auto"` and `"none"` are supported. Required
or forced named calls, strict tool schemas, and `parallel_tool_calls: false`
are currently rejected.

## Metrics and request logging

Prometheus metrics are enabled by default:

```sh
curl http://127.0.0.1:8080/metrics
```

Write complete requests and answers as one JSON object per line with:

```sh
mlx-serve serve --config ./mlx-serve.yaml \
  --request-log-file ./requests.jsonl
```

Each record contains timestamps, request, answer, status, and completion state.
Streaming answers are stored as ordered SSE chunks. These logs contain prompts
and generated content, so treat them as sensitive data.

Run `mlx-serve --help` or `<command> --help` for the complete CLI reference.
