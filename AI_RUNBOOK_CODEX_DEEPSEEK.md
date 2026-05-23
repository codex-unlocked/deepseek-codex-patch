# ONE-SHOT AI IMPLEMENTATION RUNBOOK: CODEX BYOK WITH DEEPSEEK

> **Credits:** All primary credit goes to [0xSero's codex-shim](https://github.com/0xSero/codex-shim) who did the heavy lifting to build the core proxy server! This runbook exists strictly to automate the installation of that shim using AI, while applying a few necessary edge-case patches for DeepSeek compatibility and Codex UI restrictions.

**TARGET AUDIENCE:** AI Coding Assistants (Cursor, Copilot, etc.)  
**MISSION:** You are tasked with patching the `codex-shim` and the Codex Desktop app to fully support DeepSeek BYOK models. The user wants this done flawlessly on the first try. You MUST follow these exact steps and code snippets. Do not deviate.

---

## CONTEXT & ISSUES TO SOLVE
Codex Desktop does not natively support DeepSeek endpoints or the new `developer` role. It exhibits three major issues:
1. It hard-filters custom models from the UI dropdown.
2. It uses the `developer` role, which DeepSeek rejects with `invalid_request_error`.
3. It splits "thinking" blocks and tool calls into consecutive assistant messages, causing DeepSeek V4 to crash because the second message lacks `reasoning_content`.

You must implement the following 4 steps perfectly.

---

## STEP 1: INSTALL & FIX THE CODEX SHIM DAEMON
The user uses [codex-shim](https://github.com/0xSero/codex-shim/).
1. Clone the repository and set up a `.venv` (install requirements).
2. **CRITICAL DAEMON FIX**: In `codex_shim/cli.py`, the `start()` function spawns a daemon using `sys.executable`, which often resolves to the system python and fails to import `aiohttp`.

**Action:** Add this helper above `start()` in `codex_shim/cli.py`:
```python
def _python_exe() -> str:
    import sys
    venv_python = PROJECT_ROOT / ".venv" / "bin" / "python3"
    if venv_python.exists():
        return str(venv_python)
    return sys.executable
```
And modify `subprocess.Popen` inside `start()` to use `_python_exe()` instead of `sys.executable`. Also add `env["VIRTUAL_ENV"] = str(PROJECT_ROOT / ".venv")` to the `env` dictionary.

---

## STEP 2: PATCH SHIM TRANSLATIONS (`translate.py`)
You must heavily patch `codex_shim/translate.py` to fix the `developer` role and `reasoning_content` crashes.

**Action 1: Map the Developer Role**
In `_responses_input_to_messages`, find the block handling messages. Update it to convert `developer` to `system`:
```python
        if item_type in {"message", None} and "role" in item:
            flush_pending_assistant_tool_calls()
            role = item.get("role", "user")
            if role == "developer":
                role = "system"
            messages.append({"role": role, "content": _content_to_text(item.get("content", ""))})
```

**Action 2: Reattach & Combine Reasoning Content**
Replace the entire `responses_to_chat` function with the following bulletproof logic. This decodes the `encrypted_content`, attaches it as `reasoning_content`, AND strictly merges consecutive assistant messages (which DeepSeek demands):
```python
def responses_to_chat(body: dict[str, Any], upstream_model: str) -> dict[str, Any]:
    messages = []
    instructions = body.get("instructions")
    if instructions:
        messages.append({"role": "system", "content": _content_to_text(instructions)})
    
    pending_reasoning = ""
    for raw_m in _responses_input_to_messages(body.get("input")):
        m = dict(raw_m)
        if m.get("_reasoning_only"):
            decoded = _decode_thinking_blob(m.get("encrypted_content"))
            if decoded and isinstance(decoded, dict) and decoded.get("thinking"):
                pending_reasoning += decoded["thinking"]
            # Insert sentinel so merge logic does not fuse assistant
            # messages from different conversation turns.
            messages.append({"_SENTINEL": True, "role": "_sentinel"})
            continue
            
        if pending_reasoning:
            if m.get("role") == "assistant":
                m["reasoning_content"] = pending_reasoning
                pending_reasoning = ""
            # If role is not assistant, keep pending_reasoning for the next assistant message.
            
        # Find real merge target: skip sentinel entries.
        merge_idx = len(messages) - 1
        while merge_idx >= 0 and messages[merge_idx].get("_SENTINEL"):
            merge_idx -= 1

        if merge_idx >= 0 and messages[merge_idx].get("role") == "assistant" and m.get("role") == "assistant":
            # Combine consecutive assistant messages to satisfy DeepSeek
            if m.get("content"):
                if messages[merge_idx].get("content"):
                    messages[merge_idx]["content"] += "\n" + m["content"]
                else:
                    messages[merge_idx]["content"] = m["content"]
            if m.get("tool_calls"):
                if "tool_calls" not in messages[merge_idx]:
                    messages[merge_idx]["tool_calls"] = []
                messages[merge_idx]["tool_calls"].extend(m["tool_calls"])
            if m.get("reasoning_content"):
                if "reasoning_content" not in messages[merge_idx]:
                    messages[merge_idx]["reasoning_content"] = m["reasoning_content"]
                else:
                    messages[merge_idx]["reasoning_content"] += "\n" + m["reasoning_content"]
        else:
            messages.append(m)

    if pending_reasoning:
        # Find the last non-sentinel assistant message.
        tail_idx = len(messages) - 1
        while tail_idx >= 0 and messages[tail_idx].get("_SENTINEL"):
            tail_idx -= 1
        if tail_idx >= 0 and messages[tail_idx].get("role") == "assistant":
            messages[tail_idx]["reasoning_content"] = pending_reasoning
        else:
            messages.append({"role": "assistant", "content": "", "reasoning_content": pending_reasoning})

    # Remove sentinel entries before sending to upstream API.
    messages = [m for m in messages if not m.get("_SENTINEL")]

    chat: dict[str, Any] = {
        "model": upstream_model,
        "messages": messages or [{"role": "user", "content": ""}],
        "stream": bool(body.get("stream", False)),
    }
    _copy_if_present(body, chat, "temperature")
    _copy_if_present(body, chat, "top_p")
    _copy_if_present(body, chat, "max_output_tokens", "max_tokens")
    _copy_if_present(body, chat, "max_tokens")
    _copy_if_present(body, chat, "parallel_tool_calls")

    tools = _responses_tools_to_chat_tools(body.get("tools"))
    if tools:
        chat["tools"] = tools
        _copy_if_present(body, chat, "tool_choice")
    return chat
```

---

## STEP 3: PATCH THE CODEX UI (`app.asar`)
Codex hides custom models. You must patch the frontend code. Because of macOS System Integrity Protection (SIP), you MUST copy the app to `~/Applications` first.

**Action: Execute this exact Bash script:**
```bash
APP=~/Applications/Codex.app
rm -rf "$APP"
cp -a /Applications/Codex.app "$APP"
rm -rf /tmp/codex-asar-patch
mkdir -p /tmp/codex-asar-patch
cd /tmp/codex-asar-patch
npx --yes @electron/asar extract "$APP/Contents/Resources/app.asar" extracted
PATCH_FILE=$(grep -RIl 'useHiddenModels' extracted/webview/assets/model-queries-*.js | head -n1)
sed -i.bak -E 's/let u=c\.useHiddenModels&&o!==`amazonBedrock`,d;/let u=!1,d;/' "$PATCH_FILE"
npx --yes @electron/asar pack extracted app.asar.new
cp app.asar.new "$APP/Contents/Resources/app.asar"
```

---

## STEP 4: FIX ASAR INTEGRITY & APPLE CODE SIGNING (CRITICAL)
If you skip this, the app will instantly crash with `EXC_BREAKPOINT`. You must update the integrity hash and recursively ad-hoc sign the app with custom entitlements from the inside-out.

**Action 1: Execute this exact Python script to update `Info.plist`:**
*(Note: ensure the path points to `~/Applications/Codex.app`)*
```python
import plistlib, struct, hashlib, os

app_dir = os.path.expanduser('~/Applications/Codex.app')
app_asar = os.path.join(app_dir, 'Contents/Resources/app.asar')
plist_path = os.path.join(app_dir, 'Contents/Info.plist')

with open(app_asar, 'rb') as f:
    data_size, header_size, _, json_size = struct.unpack('<4I', f.read(16))
    header_json = f.read(json_size)
new_hash = hashlib.sha256(header_json).hexdigest()

with open(plist_path, 'rb') as f:
    plist = plistlib.load(f)

if 'ElectronAsarIntegrity' not in plist:
    plist['ElectronAsarIntegrity'] = {}
plist['ElectronAsarIntegrity']['Resources/app.asar'] = {
    'algorithm': 'SHA256',
    'hash': new_hash
}

with open(plist_path, 'wb') as f:
    plistlib.dump(plist, f)

print('Successfully updated ASAR integrity hash.')
```

**Action 2: Execute this exact Bash script for recursive codesigning:**
```bash
cat << 'EOF' > /tmp/custom_ent.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "https://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.security.app-sandbox</key><false/>
	<key>com.apple.security.cs.allow-jit</key><true/>
	<key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
	<key>com.apple.security.device.audio-input</key><true/>
	<key>com.apple.security.files.user-selected.read-write</key><true/>
	<key>com.apple.security.network.client</key><true/>
	<key>com.apple.security.cs.disable-library-validation</key><true/>
</dict>
</plist>
EOF

find ~/Applications/Codex.app -type d -name "*.framework" -o -name "*.app" | \
awk '{ print length($0) " " $0; }' | sort -r -n | cut -d ' ' -f 2- | while read bundle; do
    codesign --force --sign - --entitlements /tmp/custom_ent.plist "$bundle"
done
codesign --force --sign - --entitlements /tmp/custom_ent.plist ~/Applications/Codex.app
echo "Successfully code-signed app."
```

## STEP 5: BOOT UP
1. Run `codex-shim restart` to ensure the daemon is running with your new `translate.py` changes.
2. Ensure `~/.factory/settings.json` is configured with DeepSeek credentials.
3. Launch the app: `open ~/Applications/Codex.app`.
4. DeepSeek will now perfectly handle function calling, reasoning, and context!
