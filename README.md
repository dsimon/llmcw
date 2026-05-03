# llmcw

A macOS shell script launcher for LLM cowork tools. Presents an interactive menu to select from local models (via LM Studio) or cloud models, configures Claude Cowork's inference provider via macOS preferences, launches Claude Desktop, and restores the original settings when you're done.

## Supported tools

| Command | App |
|---------|-----|
| `llmcw` or `llmcw claude` | [Claude Cowork](https://claude.ai/product/cowork) (Anthropic) |

The framework is built to support additional cowork tools from other vendors as they emerge.

## What it does

- **Local models**: manages the full LM Studio lifecycle — starts the daemon (`lms daemon up`) if needed, lists available LLMs with architecture, quantization, size, and context info, loads the selected model, then points Cowork's gateway at LM Studio's Anthropic-compatible `/v1/messages` endpoint.
- **Cloud models — Anthropic**: clears the gateway so Cowork uses your account login; sets `inferenceModels` to the selected model ID.
- **Cloud models — 3P providers**: starts a local [LiteLLM](https://github.com/BerriAI/litellm) proxy that exposes `/v1/messages` and translates to the upstream OpenAI-compatible endpoint, then points Cowork's gateway at the proxy.
- **Clean preferences**: saves all Cowork inference preferences before launch and restores them when you press Enter to end the session.

## Installation

```bash
# Clone
git clone https://github.com/dsimon/llmcw.git
cd llmcw

# Make executable and put on PATH
chmod +x llmcw
ln -s "$PWD/llmcw" /usr/local/bin/llmcw
```

**Dependencies:**

| Dependency | Required for |
|------------|-------------|
| `bash` 3+ | macOS ships bash 3 |
| `python3` | Config loading, LM Studio model parsing |
| `curl` | LM Studio API calls, LiteLLM health check |
| `bc` | Human-readable model size formatting |
| `lms` | LM Studio daemon management and model load/unload (optional; from [LM Studio](https://lmstudio.ai)) |
| `litellm` | openai_compatible providers |

Install LiteLLM:
```bash
uv tool install 'litellm[proxy]'
```

## Configuration

On first run, `~/.llmcw/config.json` is created from a template. Edit it to add your API keys and model list.

```
~/.llmcw/
└── config.json    # credentials + model list — gitignored, never leaves your machine
```

### Provider types

```jsonc
{
  "providers": {
    "anthropic": {
      "name": "Anthropic",
      "type": "anthropic",       // uses Cowork account login; no API key needed
      "api_key": ""
    },
    "openrouter": {
      "name": "OpenRouter",
      "type": "openai_compatible", // routes through LiteLLM proxy gateway
      "api_key": "your-key",
      "base_url": "https://openrouter.ai/api/v1"
    },
    "deepinfra": {
      "name": "DeepInfra",
      "type": "openai_compatible",
      "api_key": "your-key",
      "base_url": "https://api.deepinfra.com/v1/openai"
    }
  }
}
```

### Adding models

Models are listed per tool. Each entry references a provider and uses that provider's model ID format:

| Provider | Model ID format | Example |
|----------|----------------|---------|
| `anthropic` | Anthropic canonical or alias | `claude-opus-4-7`, `opus` |
| `openrouter` | `provider/model` with dots for versions | `anthropic/claude-opus-4.7` |
| `deepinfra` | HuggingFace-style `Org/Model` | `Qwen/Qwen2.5-Coder-32B-Instruct` |

**OpenRouter note**: version numbers use dots, not dashes. Verify IDs at [openrouter.ai/models](https://openrouter.ai/models).

Model lists support `_comment` keys for documentation — these are ignored by the parser.

## Usage

```
llmcw [claude]
```

```
  0    Default model (no override)

Local LLMs (from LM Studio):
  1    Qwen2.5-Coder-32B   Qwen    Q4_K_M (4bit)   19.8 GB   32768
  ...

Claude Cowork models:
  2    Claude Opus 4.7 [Anthropic]          (claude-opus-4-7)
  3    Claude Sonnet 4.6 [Anthropic]        (claude-sonnet-4-6)
  4    Claude Opus 4.7 [OpenRouter]         (anthropic/claude-opus-4.7)
  ...

  Q/E  Quit / Exit

Choose a model (0-4, Q/E to quit) [0]:
```

After selection, Claude Desktop opens. Press Enter in the terminal when your Cowork session is done — the script restores the original inference preferences and cleans up.

**If Claude Desktop is already running**, the preference changes take effect after a restart (Cmd+Q then reopen).

## How provider routing works

Cowork reads inference settings from macOS preferences (`com.anthropic.claudefordesktop`). `llmcw` writes to those keys before launching and restores them on exit.

| Selection | `inferenceProvider` | How it works |
|-----------|--------------------|-|
| Local (LM Studio) | `gateway` → `http://localhost:1234` | LM Studio v0.4.1+ exposes Anthropic-compatible `/v1/messages` |
| Anthropic (direct) | unset | Cowork uses account login; `inferenceModels` sets the model |
| openai_compatible | `gateway` → `http://localhost:PORT` | LiteLLM proxy translates Anthropic→OpenAI chat/completions |

## Maintenance

### Updating model lists

Edit `~/.llmcw/config.json` directly. No script changes needed — the config is reloaded on every run.

**Finding model IDs:**
- Anthropic: [platform.claude.com/docs/about-claude/models](https://platform.claude.com/docs/about-claude/models/overview)
- OpenRouter: [openrouter.ai/models](https://openrouter.ai/models)
- DeepInfra: [deepinfra.com/models](https://deepinfra.com/models)

### Updating LiteLLM

```bash
uv tool upgrade litellm
```

### Adding a new cowork tool backend

The script has a straightforward pattern for each backend in three places: the `case` in argument parsing, `cloud_label` display, and the provider-configuration block before `launch_cowork`. Search for `claude)` as the reference.
