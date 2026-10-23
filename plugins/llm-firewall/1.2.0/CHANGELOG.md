# Changelog

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
