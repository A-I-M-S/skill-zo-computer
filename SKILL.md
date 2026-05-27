---
name: skill-zo-computer
description: "Forward a task to Zo Computer and return its reply. Use whenever the user prefixes a request with `zo ` (e.g. `zo find John Doe on LinkedIn`) or explicitly asks to delegate work to Zo Computer."
metadata:
  openclaw:
    requires:
      bins:
        - curl
        - jq
      env:
        - ZO_API_KEY
    primaryEnv: ZO_API_KEY
---

# Zo Computer Bridge

Use when the user says `zo <task>` or asks to "send this to Zo", "ask Zo", "have Zo do X". This skill delegates the task to a Zo Computer session via its `/zo/ask` HTTP API and returns the full reply.

## Requirements & Guardrails

- `ZO_API_KEY` must be set. Generate one at https://aloy.zo.computer → Settings → Advanced → Access Tokens.
- Pass the user's request verbatim — do not paraphrase or summarize before forwarding. Zo is itself an agent and works best with the original phrasing.
- Each call is a fresh, isolated Zo session with no memory of prior OpenClaw turns. If continuity matters, include relevant context in the `input` string.
- Calls are synchronous and can take 30s–several minutes depending on the task (web research, browser automation, etc.). Do not retry on timeout without asking the user.
- Never log or echo `ZO_API_KEY`.

## Workflow

1. **Initial Check**: Verify tools and key:
   ```sh
   command -v curl >/dev/null && command -v jq >/dev/null && [ -n "$ZO_API_KEY" ]
   ```
   If `ZO_API_KEY` is missing, ask the user to set it before proceeding.

2. **Extract task**: Take everything after the `zo ` trigger as `$TASK`. Preserve quoting and whitespace.

3. **Execute**: POST the task to Zo Computer and wait for the full response:
   ```sh
   curl -sS --max-time 600 https://api.zo.computer/zo/ask \
     -H "Authorization: Bearer $ZO_API_KEY" \
     -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     -d "$(jq -nc --arg input "$TASK" '{input:$input, model_name:"anthropic:claude-opus-4-7"}')"
   ```

4. **Parse**: Extract `.output` from the JSON response:
   ```sh
   echo "$RESPONSE" | jq -r '.output // .error // "No output returned."'
   ```

5. **Finalize**: Present Zo's reply to the user as-is (markdown preserved). Mention it came from Zo Computer. Ask if they want a follow-up `zo` call.

## Example

User: `zo find the LinkedIn profile of the CTO of Stripe and return name + URL`

Run:
```sh
TASK="find the LinkedIn profile of the CTO of Stripe and return name + URL"
curl -sS --max-time 600 https://api.zo.computer/zo/ask \
  -H "Authorization: Bearer $ZO_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "$(jq -nc --arg input "$TASK" '{input:$input, model_name:"anthropic:claude-opus-4-7"}')" \
  | jq -r '.output'
```

## Optional: Structured Output

When the caller needs machine-parseable JSON, add an `output_format` JSON Schema to the body:

```sh
curl -sS --max-time 600 https://api.zo.computer/zo/ask \
  -H "Authorization: Bearer $ZO_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "input": "Find LinkedIn profile of the CTO of Stripe.",
    "model_name": "anthropic:claude-opus-4-7",
    "output_format": {
      "type": "object",
      "properties": {
        "name": {"type": "string"},
        "title": {"type": "string"},
        "linkedin_url": {"type": "string"}
      },
      "required": ["name", "title", "linkedin_url"]
    }
  }'
```
`.output` will then be a JSON object instead of a string.

## Error Handling

- **HTTP 401 / 403** — `ZO_API_KEY` invalid or revoked. Tell the user to regenerate at Settings → Advanced → Access Tokens. Do not retry.
- **HTTP 429** — rate-limited. Wait 30s, retry once, then stop.
- **HTTP 5xx / network** — retry once after 5s. If it fails again, surface the error verbatim and stop.
- **Timeout (`curl` exit 28)** — task may still be running on Zo's side. Tell the user; do not auto-retry (would duplicate work/cost).
- **Empty `.output`** — print the raw JSON for debugging and ask the user how to proceed.
