# LLM Firewall

**Block prompts containing disallowed phrases or patterns before they ever reach the LLM.**

| | |
|---|---|
| Plugin ID | `com.tyk.llm-firewall` |
| Tier | Community |
| Runs in | Microgateway (edge) and AI Studio's embedded gateway |
| Hooks | `pre_auth` (default) or `post_auth` — chosen when the plugin is registered |
| Admin UI | None (rules are configured via the plugin's settings form) |
| Requires | Microgateway ≥ 1.0.0 |

## What it does

The LLM Firewall inspects the text of incoming LLM requests (POST requests) — user messages, system prompts, and multi-part content — against a list of blocked phrases and regular expressions you define. If any pattern matches, the request is rejected with **HTTP 403** and never reaches the provider:

```json
{
  "error": {
    "message": "Request blocked by content policy",
    "type": "content_policy_violation"
  }
}
```

The message text is configurable. Matching is case-insensitive by default — this applies to regular expressions as well as plain phrases — and stops at the first match.

It understands the request formats of multiple vendors, so one rule set covers your whole model estate:

| Vendor | What is scanned |
|--------|-----------------|
| OpenAI | `messages` array (`role`/`content`) |
| Anthropic | `system` field + `messages`, including content blocks |
| Google AI / Vertex | `systemInstruction` + `contents`/`parts` (for these vendors, if the body has no `model` field the model name is read from the URL path) |
| Ollama | OpenAI-style `messages` |
| Anything else | Falls back to OpenAI-format parsing |

## When to use it

- **Prompt-injection defense** — block known jailbreak phrasings ("ignore previous instructions", "DAN mode", …) at the gateway, uniformly, instead of relying on every application to sanitize input.
- **Acceptable-use enforcement** — stop requests containing banned terms (competitor names in a support bot, profanity in a public-facing assistant) before tokens are spent.
- **Data egress tripwires** — regex patterns for things that should never appear in a prompt (internal hostnames, project codenames, credential-like strings).
- **Per-model policies** — stricter phrase lists for externally exposed models, looser ones for internal experimentation, using model glob patterns.

It is deliberately simple: fast, deterministic pattern matching with no ML component, which makes it predictable and cheap to run at the edge. For semantic/intent-based moderation, pair it with an AI Studio content filter — the two approaches complement each other.

## Rules and patterns

Configuration is a list of **rules**, each scoped to models by a glob pattern and holding a list of **phrases**:

```json
{
  "block_message": "Your request violates our usage policy",
  "case_sensitive": false,
  "rules": [
    {
      "model_pattern": "*",
      "enabled": true,
      "phrases": [
        { "pattern": "jailbreak", "is_regex": false, "description": "Block jailbreak keyword" },
        { "pattern": "ignore (all )?previous instructions", "is_regex": true, "description": "Prompt injection" }
      ]
    },
    {
      "model_pattern": "gpt-4*",
      "enabled": true,
      "phrases": [
        { "pattern": "DAN mode", "is_regex": false, "description": "GPT-specific jailbreak" }
      ]
    }
  ]
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `block_message` | `"Request blocked by content policy"` | Message returned in the 403 body |
| `case_sensitive` | `false` | Case-sensitive matching — applies to both plain phrases and regex patterns (with the default `false`, your regexes are made case-insensitive automatically) |
| `rules[].model_pattern` | *(required)* | Glob over model names: `*`, `claude-*`, `gpt-4o-*`, … |
| `rules[].enabled` | — | **Must be explicitly set to `true`** — a rule that omits this field is treated as disabled |
| `rules[].phrases[].pattern` | *(required)* | The phrase, or a regular expression |
| `rules[].phrases[].is_regex` | `false` | Treat `pattern` as a regex |
| `rules[].phrases[].description` | *(empty)* | Why this pattern exists — shown in block logs |

Every rule whose `model_pattern` matches the request's model is applied, so a `*` baseline rule combines naturally with model-specific additions. If the model name can't be determined for a request at all, only rules with a literal `*` pattern apply to it.

## Choosing the hook phase

The firewall can run in either of two places in the request pipeline. The phase is selected **when the plugin is registered** with the gateway (the manifest's primary hook, and therefore the usual default, is `pre_auth`) — it is not part of the plugin's configuration form:

- **`pre_auth`** (default) — checks run *before* authentication. Malicious requests are rejected as early and cheaply as possible. No user/app context is available.
- **`post_auth`** — checks run *after* authentication, when the authenticated app/user context exists. Choose this if you plan to correlate blocks with identities in your logs.

## Fail-open behavior

The plugin is designed never to take down legitimate traffic through a configuration mistake:

- Unparseable request body, or a request with no extractable text → request **allowed**
- No rule matches the model → request **allowed**
- Invalid regex in a phrase → that pattern is **skipped**; an invalid `model_pattern` glob skips the **whole rule** (both logged when the configuration is loaded)
- Unknown vendor → OpenAI-format parsing is attempted

## Privacy and logging

When a request is blocked, the log records the model, the *pattern* that matched, its description, and the request ID — **not** the prompt content itself. Prompt text is never written to logs.

## Limitations

- **Requests only** — provider responses are not scanned.
- **Text only** — images and other media in multi-modal prompts are not analyzed.
- **Pattern matching, not intent detection** — paraphrased attacks that avoid your exact patterns will pass; treat this as one layer of defense.
- Checking stops at the first matching pattern, so block logs show one match per request.
