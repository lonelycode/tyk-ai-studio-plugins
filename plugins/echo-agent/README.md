# Echo Agent Plugin

A simple test agent that wraps LLM responses with custom prefix and suffix markers.

## Description

The Echo Agent is a demonstration plugin that intercepts LLM responses and wraps them with configurable prefix and suffix markers (default: `<<` and `>>`). This plugin is primarily used for:

- **Testing**: Verify that the plugin system is working correctly
- **Development**: Example of how to build agent plugins
- **Debugging**: Easily identify when plugin processing has occurred

## Features

- Wraps LLM responses with custom prefix/suffix markers
- Configurable prefix and suffix strings
- Optional metadata inclusion in responses
- Supports custom LLM selection
- Model override capability

## Configuration

The plugin accepts the following configuration options:

```json
{
  "prefix": "<<",
  "suffix": ">>",
  "include_metadata": false,
  "default_llm_id": 1,
  "model": "gpt-4"
}
```

### Configuration Options

- **prefix** (string): Text to prepend before LLM responses (default: `"<<"`)
- **suffix** (string): Text to append after LLM responses (default: `">>"`)
- **include_metadata** (boolean): Include metadata about the LLM call (default: `false`)
- **default_llm_id** (integer): ID of the default LLM to use
- **model** (string): Specific model to use (e.g., 'gpt-4', 'claude-3-5-sonnet')

## Permissions

This plugin requires the following permissions:

- **llms.proxy**: Access to call LLM services via the AI Studio proxy

## Installation

### From Marketplace

1. Navigate to **Marketplace** in AI Studio
2. Search for "Echo Agent"
3. Click **Install**
4. Review and approve permissions
5. Configure the plugin settings
6. Complete the installation

### Manual Installation

```bash
# Using OCI reference
oci://ghcr.io/tyk-technologies/plugins/echo-agent:0.1.0
```

## Usage

Once installed and configured:

1. Create or select an Agent that uses this plugin
2. Start a chat conversation
3. The agent will wrap all LLM responses with your configured prefix/suffix

Example output:
```
User: Hello, how are you?
Agent: << I'm doing well, thank you for asking! How can I help you today? >>
```

## Requirements

- AI Studio version: 0.1.0 or higher
- OCI plugin support enabled
- LLM provider configured

## Source Code

The source code for this plugin is available in the main Tyk AI Studio repository:
https://github.com/TykTechnologies/midsommar/tree/main/examples/plugins/studio/echo-agent

## License

Apache 2.0

## Support

- Community Forum: https://community.tyk.io
- Issues: https://github.com/TykTechnologies/midsommar/issues
