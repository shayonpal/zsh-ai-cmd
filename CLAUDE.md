# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

zsh-ai-cmd is a zsh plugin that translates natural language to shell commands using LLM APIs. User types a description, presses `Ctrl+Z`, and sees the suggestion as ghost text. Tab accepts, typing dismisses.

## Architecture

The main plugin lives in @zsh-ai-cmd.plugin.zsh with provider implementations in `providers/`.

### Supported Providers

 | Provider   | File                      | Default Model               | API Key Env Var     |
 | ---------- | ------                    | ---------------             | -----------------   |
 | Anthropic  | `providers/anthropic.zsh` | `claude-haiku-4-5-20251001` | `ANTHROPIC_API_KEY` |
 | OpenAI     | `providers/openai.zsh`    | `gpt-5.2-2025-12-11`        | `OPENAI_API_KEY`    |
 | Gemini     | `providers/gemini.zsh`    | `gemini-3-flash-preview`    | `GEMINI_API_KEY`    |
 | DeepSeek   | `providers/deepseek.zsh`  | `deepseek-chat`             | `DEEPSEEK_API_KEY`  |
 | Ollama     | `providers/ollama.zsh`    | `mistral-small`             | (none - local)      |

Set provider via `ZSH_AI_CMD_PROVIDER='openai'` (default: `anthropic`).

### Provider Implementation

Each provider file exports two functions:
- `_zsh_ai_cmd_<provider>_call "$input" "$prompt"` - Makes API call, prints command to stdout
- `_zsh_ai_cmd_<provider>_key_error` - Prints setup instructions when API key missing

All providers use structured outputs (JSON schema) where supported for reliable command extraction. The system prompt is shared across providers via `$_ZSH_AI_CMD_PROMPT` from `prompt.zsh`.

### Core Components

**Core Flow:**
- **Widget function** `_zsh_ai_cmd_suggest`: Main entry point bound to keybinding. Captures buffer text, shows spinner, calls API, displays result as ghost text via `POSTDISPLAY`.
- **API call** `_zsh_ai_cmd_call_api`: Background curl with animated braille spinner. Uses ZLE redraw for UI updates during blocking wait.
- **Key retrieval** `_zsh_ai_cmd_get_key`: Lazy-loads API key from env var or macOS Keychain.

**Ghost Text System:**
- **`_zsh_ai_cmd_show_ghost`**: Displays suggestion in `POSTDISPLAY`. If suggestion extends current buffer, shows suffix only. Otherwise shows ` → suggestion`.
- **`_zsh_ai_cmd_clear_ghost`**: Clears `POSTDISPLAY` and resets suggestion state.
- **`_zsh_ai_cmd_update_ghost_on_edit`**: Called on keystroke. Clears ghost if user's edits diverge from suggestion.
- **`_zsh_ai_cmd_accept`**: Tab handler. Accepts suggestion into buffer or falls through to normal tab completion.

## Testing

API response validation tests live in @test-api.sh:

```sh
# Test default provider (anthropic)
./test-api.sh

# Test specific provider
./test-api.sh --provider openai
./test-api.sh --provider gemini
./test-api.sh --provider ollama
./test-api.sh --provider deepseek
```

Manual testing:
```sh
source ./zsh-ai-cmd.plugin.zsh
# Type natural language, press Ctrl+Z
list files modified today<Ctrl+Z>
```

Enable debug logging:
```sh
ZSH_AI_CMD_DEBUG=true
tail -f ${ZSH_AI_CMD_LOG:-/tmp/zsh-ai-cmd.log}
```

## Code Conventions

- Uses `command` prefix (e.g., `command curl`, `command jq`) to bypass user aliases
- All configuration via `typeset -g` globals with `ZSH_AI_CMD_` prefix
- Internal functions/variables use `_zsh_ai_cmd_` or `_ZSH_AI_CMD_` prefix
- Pure zsh where possible; external deps limited to `curl`, `jq`, `security` (macOS)

## ZLE Widget Constraints

When modifying the spinner or UI code:
- `zle -R` forces redraw within widget context
- `zle -M` shows messages in minibuffer
- Background jobs need `NO_NOTIFY NO_MONITOR` to suppress job control noise
- `read -t 0.1` provides non-blocking sleep without external deps

**Widget Wrapping:**
The plugin wraps `self-insert` and `backward-delete-char` to intercept keystrokes and update ghost text. This uses `zle .self-insert` (with dot prefix) to call the original widget. The idempotency guard at the top prevents double-wrapping if the plugin is sourced multiple times.
