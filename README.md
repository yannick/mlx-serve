# mlx-serve — binary releases mirror

OpenAI-compatible continuous-batching server for MLX models on Apple
Silicon.

This GitHub repository hosts **binary releases only** (macOS arm64,
Developer ID signed + Apple-notarized).

no telemetry, no phone home

Install via Homebrew:

```sh
brew tap yannick/tap
brew trust yannick/tap
brew install mlx-serve
```




━━ Quickstart

  The default config location is ./mlx-serve.yaml, relative to the directory where
  the server is started. After installing, the interactive setup command creates
  that file, discovers compatible models in LM Studio, the Hugging Face cache, ~/
  mlx-models, and any --dirs paths, then offers to download the small
  mlx-community/Qwen2.5-0.5B-Instruct-4bit test model:


  just build
  ./bin/mlx-serve setup
  ./bin/mlx-serve serve --config ./mlx-serve.yaml


  Scan extra locations later with mlx-serve scan --dirs /dir/a,/dir/b; it
  preserves existing entries and adds newly discovered checkpoints. Both commands
  accept --config/-c when the file should live elsewhere. For unattended setup,
  use mlx-serve setup --yes; --no-download performs discovery and config creation
  only. The complete annotated config remains available in mlx-serve.example.yaml.
  A new setup/scan config persists ~/mlx-models as cache.dir when no model
  directory was supplied.

  The download command reads cache.dir and cache.offline from the supplied config.
  --cache-dir and --offline override those values. An 8-bit Granite checkpoint is
  also available as mlx-community/granite-4.1-8b-8bit.

  just build creates the production MLX-C release at ./bin/mlx-serve. Use just
  release-mock to build the same path with the mock engine instead.

  ── Interactive dashboard

  mlx-serve tui opens a tui-rs dashboard around the configured server. It can
  start and gracefully stop the server, search Hugging Face for MLX-tagged models,
  show model-card descriptions plus parameter/download sizes, and download through
  mlx-serve's checksum-verifying cache. The YAML view can be edited, validated,
  and saved in place. A local-model view scans LM Studio, Hugging Face, the
  configured cache, and ~/mlx-models; its add dialog writes explicit context,
  slot, task, quantization, startup, and on-demand parameters. A configured-model
  view can mark multiple rows, load or unload them, and shows ready models in
  green with aligned serving-parameter and resident-memory columns; full model
  names determine the width of the name column. A fifth view provides streaming
  chat whenever a generation model is loaded. It sends the complete conversation
  history on each turn, supports /clear and Ctrl-N for a new conversation, and
  uses Tab to switch among loaded generation models. CPU, GPU, and network
  throughput use bar time series with the newest sample on the right; request and
  token rates remain compact gauges. The shared network scale follows standard
  link-speed ceilings (10/100 Mbit, 1/2.5/10/25/ 100 Gbit) based on the highest
  observed rate, with cumulative traffic alongside.

  When metrics.enabled: true (the default), Prometheus-format metrics are
  available over HTTP at GET /metrics:


  curl http://127.0.0.1:8080/metrics



  ./bin/mlx-serve tui --config ./mlx-serve.yaml


  Press s to start/stop and 1–5 for Hugging Face, discovered local models,
  configured models, YAML config, and chat. In search, o cycles the sort column, O
  reverses it, d downloads, and x cancels while retaining resumable partial files.
  In local models, a opens the explicit-parameter add dialog. In configured
  models, Space marks rows and l/u loads or unloads all marked models. In config,
  e edits and Ctrl-S validates/saves. In chat, Enter sends, Tab changes the loaded
  model, /clear or Ctrl-N starts a new chat, Esc returns to the dashboard, and
  Ctrl-C quits. Elsewhere, q drains the server, stops any download safely, and
  quits.

  Configured models have independent residency policies: load_on_start makes a
  model ready during bootstrap, while load_on_demand permits an inference request
  to load an unloaded model. If both are false, the model is manual-only.
  Operators and clients can explicitly manage residency with POST /v1/models/{id}/
  load and DELETE /v1/models/{id}/load; unloading removes the model from admission
  immediately and drains in-flight requests before its memory is released.

  ── Tool calling and reasoning

  Tool support is enabled per model because output envelopes differ between
  families. Granite and other models that emit JSON inside <tool_call> use:


  models:
    - id: granite-thinking
      repo: usermma/Granite-4.1-30B-Claude-4.6-Opus-Thinking-Charles-Xavier-mlx-2Bit
      tool_format: json
      reasoning_format: think_tags
      enable_thinking: true


  Qwen 3.5 models using nested <function>/<parameter> tags instead set
  tool_format: qwen35. The Chat Completions endpoint accepts tools, legacy
  functions/function_call, assistant tool_calls, and tool result messages.
  Responses use OpenAI-compatible tool_calls with finish_reason: "tool_calls";
  extracted reasoning is returned separately as reasoning_content in JSON and SSE
  deltas. The server returns tool requests to the client—it never executes them.

  One complete client-owned tool round trip looks like this (the second request
  uses the returned call id and includes the full history):


  curl http://127.0.0.1:8080/v1/chat/completions \
    -H 'content-type: application/json' \
    -d '{
      "model":"granite-thinking",
      "messages":[{"role":"user","content":"Read README.md"}],
      "tools":[{"type":"function","function":{"name":"read_file","description":"Read a UTF-8 file","parameters":{"type":"object","properties":{"path":{"type":"string"}},"required":["path"]}}}]
    }'

  # After executing read_file for the returned call_... id:
  curl http://127.0.0.1:8080/v1/chat/completions \
    -H 'content-type: application/json' \
    -d '{
      "model":"granite-thinking",
      "messages":[
        {"role":"user","content":"Read README.md"},
        {"role":"assistant","content":null,"tool_calls":[{"id":"call_...","type":"function","function":{"name":"read_file","arguments":"{\"path\":\"README.md\"}"}}]},
        {"role":"tool","tool_call_id":"call_...","content":"# mlx-serve ..."}
      ],
      "tools":[{"type":"function","function":{"name":"read_file","parameters":{"type":"object","properties":{"path":{"type":"string"}},"required":["path"]}}}]
    }'


  tool_choice: "auto" and "none" are supported. required, forced named tools,
  parallel_tool_calls: false, and strict: true schemas currently return a clear
  400 because reliably enforcing those modes requires tool-aware constrained
  decoding.

  Full request/answer auditing is available to both serve and tui:


  ./bin/mlx-serve serve --config ./mlx-serve.yaml \
    --request-log-file ./requests.jsonl


  Every authenticated /v1 response is appended as one JSON object containing
  start/end Unix-millisecond timestamps, method, path, status, request, answer,
  and explicit completion/truncation flags. Streaming answers are stored as their
  ordered SSE JSON chunks. Captures are bounded to 1 MiB each, and a bounded
  writer queue backpressures response completion if storage falls behind. The file
  contains prompts and generated content, so protect it as sensitive application
  data.
