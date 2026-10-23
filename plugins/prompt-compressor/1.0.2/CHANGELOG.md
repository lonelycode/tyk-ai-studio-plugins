# Changelog

## [1.0.2] - 2026-10-23

Requires AI Studio 2.3.0.

### Added
- **`compress_server_tool_results`** (per policy, off by default): compresses
  some tool results the vendor produced like the client's own: Anthropic web
  search, web fetch and (bash) code execution results, and Responses file
  search and code interpreter results. Other vendor results are always left
  as sent. Level 3 never masks a vendor's result.

### Fixed
- **Bedrock and Google requests are priced.** The model is now read from the
  request path when the body names none (Bedrock's `/model/{modelId}/`, an
  escaped ARN or inference profile included, and Google's `models/{model}:`),
  so the savings dashboard prices those records. Before, a Bedrock record had
  no model and was never priced.
- **Documentation:** the coverage table now lists `/ai/`, `/v1`, `/router/`
  and `/anthropic/` to Bedrock as covered from AI Studio 2.3, where the plugin
  runs on the gateway's internal `/llm/call/` hop on the Converse body.

## [1.0.1] - 2026-10-21

First release. Needs Tyk AI Studio 2.3.0 and its Microgateway. (1.0.0 was a
pre-review build and was never listed.)

### Added
- Compression of LLM request bodies on the Microgateway (`post_auth`), starting with tool outputs: OpenAI Chat Completions and Responses, Anthropic, Google AI, Vertex, Bedrock Converse and Ollama.
- Levels: 1 lossless (the default), 2 noise reduction for test runs, logs, listings and long JSON arrays, keeping every failure, and 3 size budgets, optional masking and external engines (Headroom, LLMLingua-2). Levels 2 and 3 are opt-in.
- Cache-safe output: the same text and policy always give the same bytes, so the vendor's prompt cache keeps hitting.
- `audit` (the default) and `enforce` modes; every failure forwards the request unchanged.
- Named policies bound per LLM or App, and per App through the `prompt_compressor.policy` metadata key.
- `drop_tools`: removes named tool definitions from requests to save tokens (not an access control).
- Response headers `X-Tyk-Compression`, `-Tokens-Saved` and `-Policy`; an optional client bypass header, off by default.
- Studio pages: savings dashboard (tokens and cost saved at input and cache-read price, cache-read share, policy-change markers, cache-drop warning, tool-definition cost), Policies editor, and a preview tool. Each has its own permission.
- Savings records sent from the edges to Studio carry counts and IDs only, never prompt or tool text.
