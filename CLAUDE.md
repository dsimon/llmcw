# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository contains the `llmcw` shell script — an LLM cowork launcher for macOS. It presents an interactive menu to select from local models (via LM Studio) or cloud models, writes the appropriate provider configuration to macOS preferences (`com.anthropic.claudefordesktop`), launches Claude Desktop (Cowork tab), and restores the original preferences on exit.

Currently supported tools:
- `claude` — Claude Cowork (Anthropic)

The framework is designed to support additional cowork tools from other vendors as they emerge.

## Usage

```
llmcw [claude]   # default: claude
```

Option `0` (default, press Enter) launches with no model override.

## How Provider Configuration Works

Claude Cowork reads provider settings from the `enterpriseConfig` key inside `~/Library/Application Support/Claude/claude_desktop_config.json`. `llmcw` saves the existing `enterpriseConfig` block at startup, writes new keys for the selected model, and restores the originals when the user signals the session is done.

### Preference keys managed

| Key | Purpose |
|-----|---------|
| `inferenceProvider` | `gateway` for local/3P, unset for account login |
| `inferenceGatewayBaseUrl` | Gateway base URL |
| `inferenceGatewayApiKey` | Gateway API key |
| `inferenceGatewayAuthScheme` | `bearer` or `x-api-key` |
| `inferenceModels` | JSON array of model IDs to use |
| `inferenceGatewayHeaders` | JSON array of extra headers (reserved) |

### Local models (LM Studio)
LM Studio v0.4.1+ exposes an Anthropic-compatible `/v1/messages` endpoint on `http://127.0.0.1:1234`, which Cowork can consume directly as a gateway.

**Note**: Claude Desktop's gateway URL validator requires `http://127.0.0.1` (not `http://localhost`) for local HTTP endpoints.

1. `llmcw` starts the LM Studio daemon if not running (`lms daemon up`)
2. Loads the selected model (`lms load <key>`)
3. Sets `inferenceProvider=gateway`, `inferenceGatewayBaseUrl=http://127.0.0.1:1234`
4. Launches Claude Desktop

### Cloud — Anthropic direct
Clears the gateway prefs so Cowork uses the user's account login. Sets `inferenceModels` to the selected model ID to override the default.

### Cloud — openai_compatible providers
Starts a LiteLLM proxy (`uv tool install 'litellm[proxy]'` required) on a random localhost port. The proxy exposes `/v1/messages` and translates to the upstream OpenAI chat/completions endpoint. Cowork's gateway is pointed at the proxy.

## Configuration File

Cloud models and credentials live in **`~/.llmcw/config.json`** — never committed to git.

### `providers`
Named credential blocks. Each has:
- `type`: `"anthropic"` or `"openai_compatible"`
- `api_key`: provider key (unused for `anthropic` type — auth is via account login)
- `base_url`: endpoint (openai_compatible only)

### `models`
Per-tool ordered lists. Each entry:
```json
{"provider": "<provider-key>", "model": "<model-id>", "display": "<menu label>"}
```

The menu shows `"<display> [<provider.name>]"`. Model lists use a `list` array nested in an object (to allow `_comment` keys alongside it).

### Model ID conventions
| Provider | Format | Example |
|----------|--------|---------|
| `anthropic` | Anthropic canonical IDs or aliases | `claude-opus-4-7`, `opus` |
| `openrouter` | `provider/model` with dots for versions | `anthropic/claude-opus-4.7` |
| `deepinfra` | HuggingFace-style `Org/Model` | `Qwen/Qwen2.5-Coder-32B-Instruct` |

**OpenRouter note**: version numbers use dots, not dashes. Verify IDs at [openrouter.ai/models](https://openrouter.ai/models).

## Dependencies

| Dependency | Required for |
|------------|-------------|
| `python3` | Config loading, model parsing |
| `curl` | LM Studio API + LiteLLM health check |
| `bc` | Human-readable model size formatting |
| `lms` | LM Studio daemon + model load/unload (optional) |
| `litellm` | openai_compatible providers |

Install LiteLLM: `uv tool install 'litellm[proxy]'`

## Files

| Path | Purpose |
|------|---------|
| `llmcw` | Main script |
| `~/.llmcw/config.json` | Cloud model config and credentials (gitignored) |
| `/tmp/llmcw-litellm.*.log` | LiteLLM proxy log (cleaned up on exit) |
| `/tmp/llmcw-litellm.*.yaml` | Ephemeral LiteLLM config (cleaned up on exit) |

## Adding a New Cowork Backend

Search the script for `claude)` to find the three places each backend appears: the `case` in argument parsing, `cloud_label` display, and the final `launch_cowork` call pattern. The `launch_cowork` function is shared — backends differ only in how they configure the provider preferences before calling it.
