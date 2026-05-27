# skill-zo-computer

An [OpenClaw](https://claw.aimeta.club) skill that forwards a task to [Zo Computer](https://zocomputer.com) via its `/zo/ask` HTTP API and returns the reply.

## Trigger

```
zo <any task>
```

Examples:
- `zo find the LinkedIn profile of the CTO of Stripe`
- `zo summarize my latest 10 emails`
- `zo draft a tweet announcing our new product`

## Setup

1. Generate an access token at https://aloy.zo.computer → **Settings → Advanced → Access Tokens**.
2. Export it in OpenClaw's environment:
   ```sh
   export ZO_API_KEY="zo_..."
   ```
3. Ensure `curl` and `jq` are available (both are standard on most systems).

No installation needed — the skill is pure shell + `curl`.

## How it works

Each `zo <task>` invocation spawns a fresh, fully-isolated Zo session with access to Zo's full toolset (web search, LinkedIn lookups, browser automation, email, calendar, file system, etc.). The call is synchronous and returns Zo's final reply as markdown (or structured JSON if `output_format` is provided).

See `SKILL.md` for the full spec.

## License

MIT
