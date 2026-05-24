# ONE-SHOT AI IMPLEMENTATION RUNBOOK: CODEX BYOK WITH DEEPSEEK

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
Replace the entire `responses_to_chat` function with the following bulletproof logic. This decodes the `encrypted_content`, adds a fallback to extract text from `summary`, attaches it as `reasoning_content`, ensures proper tool_call message sequencing, and prevents DeepSeek crashes by padding missing reasoning blocks.
```python
def responses_to_chat(body: dict[str, Any], upstream_model: str) -> dict[str, Any]:
    import sys, json
    print("DEBUG INCOMING BODY:", json.dumps(body), file=sys.stderr)
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
            else:
                for summary in m.get("summary") or []:
                    text = summary.get("text") if isinstance(summary, dict) else None
                    if text:
                        pending_reasoning += text
            # Insert sentinel so merge logic does not fuse assistant
            # messages from different conversation turns.
            messages.append({"_SENTINEL": True, "role": "_sentinel"})
            continue
            
        if pending_reasoning:
            if m.get("role") == "assistant":
                m["reasoning_content"] = pending_reasoning
                pending_reasoning = ""
            # If role is not assistant (e.g. tool, user), keep
            # pending_reasoning for the next assistant message.
            
        # Only merge when the *immediate* predecessor is an assistant
        # message.  If a _SENTINEL sits between them, the two assistant
        # messages belong to different reasoning phases and MUST NOT be
        # merged — fusing their tool_calls would break DeepSeek's
        # requirement that every tool_calls message is followed by
        # matching tool-result messages.
        merge_idx = len(messages) - 1
        if merge_idx >= 0 and not messages[merge_idx].get("_SENTINEL") and messages[merge_idx].get("role") == "assistant" and m.get("role") == "assistant":
            # Combine them
            if m.get("content"):
                if messages[merge_idx].get("content"):
                    messages[merge_idx]["content"] += "\n" + m["content"]
                else:
                    messages[merge_idx]["content"] = m["content"]
            if m.get("tool_calls"):
                if "tool_calls" not in messages[merge_idx]:
                    messages[merge_idx]["tool_calls"] = []
                # Deduplicate by id: only add tool_calls whose id is not
                # already present in the merged message.  Without this,
                # two assistant messages referencing the same call_id
                # (common with the "call_0" fallback) produce duplicate
                # entries that the downstream validation misses because it
                # only checks set membership, not count.  This causes
                # "insufficient tool messages" from strict APIs (DeepSeek).
                existing_ids = {tc["id"] for tc in messages[merge_idx]["tool_calls"] if tc.get("id")}
                for tc in m["tool_calls"]:
                    if tc.get("id") not in existing_ids:
                        messages[merge_idx]["tool_calls"].append(tc)
                        if tc.get("id"):
                            existing_ids.add(tc["id"])
            if m.get("reasoning_content"):
                if "reasoning_content" not in messages[merge_idx]:
                    messages[merge_idx]["reasoning_content"] = m["reasoning_content"]
                else:
                    messages[merge_idx]["reasoning_content"] += "\n" + m["reasoning_content"]
        else:
            # Deduplicate tool_calls within a single message too.
            # The "call_0" fallback in _responses_input_to_messages can
            # produce multiple entries with the same id.
            if m.get("tool_calls"):
                seen = set()
                deduped = []
                for tc in m["tool_calls"]:
                    if tc.get("id") not in seen:
                        seen.add(tc["id"])
                        deduped.append(tc)
                m["tool_calls"] = deduped
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

    # Validate and cleanup tool_calls to satisfy strict APIs (like DeepSeek).
    # Every tool_call in an assistant message MUST have a corresponding tool message.
    # Use Counter (not set) so N tool_calls sharing the same id require N tool
    # messages.  A plain set would let duplicate ids all pass when fewer tool
    # messages exist, triggering "insufficient tool messages" errors.
    from collections import Counter
    for i, msg in enumerate(messages):
        if msg.get("role") == "assistant" and msg.get("tool_calls"):
            available = Counter[str]()
            for j in range(i + 1, len(messages)):
                if messages[j].get("role") == "tool":
                    if "tool_call_id" in messages[j]:
                        available[messages[j]["tool_call_id"]] += 1
                elif messages[j].get("role") not in ("tool", "_sentinel"):
                    break

            consumed = Counter[str]()
            valid_tool_calls = []
            for tc in msg["tool_calls"]:
                tc_id = tc.get("id") or ""
                if tc_id and available.get(tc_id, 0) > consumed.get(tc_id, 0):
                    consumed[tc_id] += 1
                    valid_tool_calls.append(tc)
            if not valid_tool_calls:
                del msg["tool_calls"]
                if not msg.get("content") and not msg.get("reasoning_content"):
                    msg["content"] = "..." # dummy content to prevent empty message error
            else:
                msg["tool_calls"] = valid_tool_calls

    # Remove sentinel entries and orphaned tool messages before sending to upstream API.
    final_messages = []
    valid_tool_call_ids = set()
    for m in messages:
        if m.get("_SENTINEL"):
            continue
        if m.get("role") == "assistant" and "tool_calls" in m:
            for tc in m["tool_calls"]:
                valid_tool_call_ids.add(tc.get("id"))
        if m.get("role") == "tool":
            if m.get("tool_call_id") not in valid_tool_call_ids:
                continue # drop orphaned tool message
                
        # Ensure deepseek models always have reasoning_content in assistant messages
        if m.get("role") == "assistant" and "deepseek" in upstream_model.lower():
            if "reasoning_content" not in m:
                m["reasoning_content"] = ""
                
        final_messages.append(m)
    
    messages = final_messages

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
        
    import sys
    print("DEBUG OUTGOING CHAT PAYLOAD:", json.dumps(chat), file=sys.stderr)
    return chat
```

**Action 3: Preserve `<think>` Blocks in Non-Streaming Responses**
Replace `chat_completion_to_response` to ensure that non-streaming background actions (like commit generation) don't permanently drop the reasoning content from the chat history:
```python
def chat_completion_to_response(payload: dict[str, Any], requested_model: str) -> dict[str, Any]:
    choice = (payload.get("choices") or [{}])[0]
    message = choice.get("message") or {}
    output: list[dict[str, Any]] = []
    
    raw_content = message.get("content") or ""
    text = strip_think(raw_content)
    
    reasoning = message.get("reasoning_content")
    if not reasoning:
        match = THINK_RE.search(raw_content)
        if match:
            reasoning = match.group(0)
            reasoning = reasoning.removeprefix("<think>").removesuffix("</think>").strip("\n")
            
    if reasoning:
        import base64
        import json
        payload_data = {"type": "thinking", "thinking": reasoning, "signature": ""}
        raw = json.dumps(payload_data, separators=(",", ":")).encode("utf-8")
        encrypted = "anthropic-thinking-v1:" + base64.urlsafe_b64encode(raw).decode("ascii")
        output.append(
            {
                "id": "rs_0",
                "type": "reasoning",
                "status": "completed",
                "summary": [{"type": "summary_text", "text": reasoning}],
                "encrypted_content": encrypted,
            }
        )

    if text:
        output.append(
            {
                "id": "msg_0",
                "type": "message",
                "status": "completed",
                "role": "assistant",
                "content": [{"type": "output_text", "text": text, "annotations": []}],
            }
        )
    for call in message.get("tool_calls") or []:
        fn = call.get("function") or {}
        output.append(
            {
                "id": call.get("id", "call_0"),
                "type": "function_call",
                "status": "completed",
                "call_id": call.get("id", "call_0"),
                "name": fn.get("name", ""),
                "arguments": fn.get("arguments", ""),
            }
        )
    return {
        "id": payload.get("id", "resp_chat"),
        "object": "response",
        "created_at": payload.get("created", 0),
        "status": "completed",
        "model": requested_model,
        "output": output,
        "usage": payload.get("usage"),
    }
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

---

## APPENDIX: Troubleshooting OpenAI API Error "insufficient tool messages following tool_calls message"

### 1. What Is This Error?

This is a **message-ordering validation error** thrown by OpenAI's Chat Completions API. It means the `messages` array you sent to the API has a structural violation: an **assistant message containing `tool_calls`** that isn't immediately and completely followed by matching **`tool`-role response messages**.

OpenAI enforces a strict parent-child relationship in the conversation history:

```
assistant (with tool_calls)  →  tool (for call_1)  →  tool (for call_2)  →  ...  →  assistant/user/next message
```

If even one tool call ID in that assistant message has no corresponding `tool` message, the entire request is rejected with this error.

### 2. Why Does It Happen? (Root Causes)

The error is caused by one of these scenarios:

| Cause | What Happens |
|-------|-------------|
| **Missing tool response** | The assistant asked for 2 function calls, but you only added 1 `tool` message to the history before the next request. |
| **Tool responses appended after another message** | You inserted a `user`, `system`, or another `assistant` message between the assistant's tool_calls message and the tool responses. The API requires tool responses **immediately** after the assistant message that requested them. |
| **Streaming truncation** | You're using `stream=True`. Tool call deltas arrive across multiple chunks. If you stop collecting chunks early or assemble the final assistant message incorrectly, some `tool_call_id` values get lost. |
| **Race conditions / parallel execution** | You fire multiple tool calls in parallel but one fails silently. You then push only the successful response(s) into the messages array, leaving orphaned call IDs. |
| **Manual message construction** | You hand-build the messages array and forget to attach a tool response before the next API call. |
| **Logging / replay** | You save messages to a database, trim or filter them, and replay a subset that breaks the pairing. |

### 3. How to Fix It

#### Step 1: Validate the pairing before every API call

```python
def validate_tool_call_pairing(messages):
    """Check that every assistant tool_call has a matching tool response."""
    for i, msg in enumerate(messages):
        if msg.get("role") == "assistant" and msg.get("tool_calls"):
            expected_ids = {tc["id"] for tc in msg["tool_calls"]}
            found_ids = set()
            j = i + 1
            # Collect all immediate tool responses
            while j < len(messages) and messages[j]["role"] == "tool":
                found_ids.add(messages[j]["tool_call_id"])
                j += 1
            missing = expected_ids - found_ids
            if missing:
                raise ValueError(f"Missing tool responses for: {missing}")
```

#### Step 2: Always pair tool calls with responses immediately

```python
# Correct pattern
response = client.chat.completions.create(model="gpt-4o", messages=messages)

msg = response.choices[0].message

if msg.tool_calls:
    # 1. Append the assistant message FIRST
    messages.append(msg.model_dump())

    # 2. THEN append each tool response BEFORE any other message
    for tool_call in msg.tool_calls:
        result = execute_tool(tool_call.function.name, tool_call.function.arguments)
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": json.dumps(result)
        })

    # 3. NOW you can call the API again
    response = client.chat.completions.create(model="gpt-4o", messages=messages)
```

#### Step 3: Handle streaming correctly

When streaming, tool calls arrive incrementally — accumulate them fully before appending:

```python
collected_tool_calls = {}  # index → {id, name, arguments}

for chunk in stream:
    delta = chunk.choices[0].delta
    for tc in delta.tool_calls or []:
        idx = tc.index
        if idx not in collected_tool_calls:
            collected_tool_calls[idx] = {"id": tc.id or "", "function": {"name": "", "arguments": ""}}
        if tc.id:
            collected_tool_calls[idx]["id"] = tc.id
        if tc.function:
            if tc.function.name:
                collected_tool_calls[idx]["function"]["name"] += tc.function.name
            if tc.function.arguments:
                collected_tool_calls[idx]["function"]["arguments"] += tc.function.arguments

# Now build the assistant message with ALL tool calls intact
assistant_msg = {
    "role": "assistant",
    "content": None,
    "tool_calls": list(collected_tool_calls.values())
}
messages.append(assistant_msg)

# Then add all tool responses
for idx in sorted(collected_tool_calls):
    tc = collected_tool_calls[idx]
    result = execute_tool(tc["function"]["name"], tc["function"]["arguments"])
    messages.append({
        "role": "tool",
        "tool_call_id": tc["id"],
        "content": json.dumps(result)
    })
```

### 4. Other Important Things to Know

#### Invalid Ordering (breaks)
```
[{"role": "assistant", "tool_calls": [{"id": "call_1"}, {"id": "call_2"}]}]
[{"role": "tool", "tool_call_id": "call_1", "content": "..."}]
[{"role": "user", "content": "Hey!"}]          ← ❌ "call_2" never responded, AND a user message was injected
```

#### Valid Ordering (works)
```
[{"role": "assistant", "tool_calls": [{"id": "call_1"}, {"id": "call_2"}]}]
[{"role": "tool", "tool_call_id": "call_1", "content": "..."}]
[{"role": "tool", "tool_call_id": "call_2", "content": "..."}]
[{"role": "user", "content": "Hey!"}]           ← ✅ All tool calls resolved first
```

#### Edge Cases

- **Parallel tool calls with failures**: If one tool execution throws an exception, you must **still** provide a `tool` message for that call ID. Return the error as the content, e.g., `{"error": "timeout"}`. Never leave it out.
- **Assistant messages with `content` + `tool_calls`**: The assistant message may have both text `content` and `tool_calls`. The same rule applies — every tool_call_id still needs a response.
- **Re-submitting old messages**: If you trim conversation history for cost/context reasons, never cut between an assistant tool_calls message and its tool responses. Trim in complete "blocks."
- **API version differences**: This validation was tightened around early 2024. Older libraries (e.g., `openai<1.0`) might not enforce it client-side, but the server will reject the request.

#### Quick Debugging Snippet

Add this right before your API call:

```python
for i, m in enumerate(messages):
    if m.get("role") == "assistant" and m.get("tool_calls"):
        for tc in m["tool_calls"]:
            found = any(
                msg.get("tool_call_id") == tc["id"]
                for msg in messages[i+1:i+1+len(m["tool_calls"])]
            )
            if not found:
                print(f"⚠️ Orphaned tool_call_id: {tc['id']} (index {i})")
```

### 5. TL;DR Summary

| Aspect | Detail |
|--------|--------|
| **What the error means** | An assistant message with `tool_calls` exists in your messages array, but not all those call IDs have a corresponding `tool`-role response message. |
| **Why it happens** | Missing, misordered, or truncated tool responses; streaming bugs; silent failures; bad message trimming. |
| **The fix** | Ensure every `tool_call_id` gets a `tool` message placed **immediately** after the assistant message, with no other messages in between, before the next API call. |
| **Golden rule** | Treat `assistant(tool_calls)` + its `tool` responses as an **atomic, inseparable block**. Never split this block. |
