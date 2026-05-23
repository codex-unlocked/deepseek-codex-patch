# Codex DeepSeek Patch 🚀

> **Credits:** All major credit goes to [0xSero's codex-shim](https://github.com/0xSero/codex-shim) who did the vast majority of the heavy lifting to build the proxy shim! This repository simply provides AI-friendly patches and an automated installation runbook to make the setup flawless and accessible.

A complete, one-shot AI-automated patch to make the official **Codex Desktop** app fully compatible with **DeepSeek V4/V3** models (via BYOK).

## What This Does
Out of the box, Codex Desktop strictly blocks custom models from its UI. Under the hood, it uses OpenAI's new `developer` role (which DeepSeek rejects), and it completely breaks DeepSeek's "thinking mode" (reasoning) during multi-turn conversations by mangling the `reasoning_content` requirement.

This repository contains an **AI Runbook**. You do not need to manually run any complicated commands or python scripts. You simply give the runbook to your AI coding assistant (like Cursor, Cline, or Copilot), and it will automatically:
1. Set up a local API shim to translate Codex requests to DeepSeek perfectly.
2. Mathematically fix the cross-turn `reasoning_content` dropping bug using sentinel-based message merging.
3. Patch the Codex Desktop UI to permanently bypass the hardcoded model allowlist.
4. Calculate new cryptographic ASAR integrity hashes and properly ad-hoc re-sign the app on macOS from the inside-out so it doesn't crash on launch.

## Requirements
- macOS (Apple Silicon or Intel)
- The official Codex Desktop App installed in `/Applications/Codex.app`
- An AI Coding Assistant (Cursor, Cline, GitHub Copilot Workspace, etc.) with workspace file-editing and terminal execution capabilities.

## How to Use (1-Minute Setup)

**1. Clone this repository**
```bash
git clone https://github.com/codex-unlocked/deepseek-codex-patch.git
cd deepseek-codex-patch
```

**2. Prompt your AI Assistant**
Open your favorite AI coding assistant in this directory, and give it the exact following prompt:

> "Please read the `AI_RUNBOOK_CODEX_DEEPSEEK.md` file in this workspace. Execute every single instruction and code patch in that runbook exactly as written to set up codex-shim and patch my Codex Desktop app."

**3. Launch Codex**
Once your AI confirms it has finished the runbook, launch the patched app!
```bash
open ~/Applications/Codex.app
```
*(Do not open the original `/Applications/Codex.app` or the custom models will stay hidden!)*

