# Changelog

## [1.2.0] - 2026-10-23

Requires AI Studio 2.3.0. Built on the 2.3 Plugin SDK (64 MiB plugin message
limit).

### Fixed
- **Model rules match Bedrock.** The model is read with the Plugin SDK's
  `RequestModel`, so Bedrock requests, whose model is in the
  `/model/{modelId}/` path, match rules by model ID, inference profile or ARN,
  escaped or not. Before, a Bedrock request was matched by its LLM's slug.
- **Bedrock Converse documents are inspected.** A document in a text format
  (`txt`, `md`, `csv`, `html`) is decoded and inspected, with its name and
  context; before, its base64 bytes were matched as they were, so a phrase in
  the document was missed. The bytes of Converse images, videos and binary
  documents no longer count as text, for matching or for `fail_closed`.
- **`hook_phase` is honoured.** The gateway calls both hooks the plugin
  declares, and each checked the request, so every request was checked twice
  whatever `hook_phase` said. The second check, in `post_auth`, saw the body
  as earlier `post_auth` plugins (such as the Prompt Compressor) had
  rewritten it. Now only the configured phase checks: `pre_auth` by default,
  and an unknown value falls back to it. `hook_phase` is in the configuration
  form.

## [1.1.1] - 2026-10-14

### Added
- `fail_closed` option: block requests whose body cannot be parsed or yields no text, instead of letting them through

### Fixed
- The block response is built as JSON, so a configured message with quotes or newlines can no longer break it
- Prompt text is inspected wherever the vendor reads it: system prompts and `system_instruction`, Responses API `input` and `instructions`, tool definitions and arguments, tool results, nested content blocks and text documents

### Changed
- Requires AI Studio 2.3.0

## [1.1.0] - 2026-09-02

### Fixed
- Align the plugin module with the parent module's Go version

### Changed
- Minimum AI Studio version is now 2.1, declared explicitly for the first time

## [1.0.0] - 2025-12-04

### Added
- Initial release of LLM Firewall
- Content filtering for requests and responses
- Prompt injection detection
- Policy enforcement capabilities
