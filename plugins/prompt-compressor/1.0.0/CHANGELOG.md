# Changelog

## [1.0.0] - 2026-10-21

First release. Needs Tyk AI Studio 2.3.0 and its Microgateway.

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
