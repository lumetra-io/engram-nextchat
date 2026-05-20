# engram-nextchat

[NextChat](https://github.com/ChatGPTNextWeb/NextChat) integration for [Engram](https://lumetra.io) — durable, explainable memory for the ~80k-star open-source chat client.

Adds six MCP tools to NextChat — `store_memory`, `query_memory`, `list_memories`, `list_buckets`, `delete_memory`, `clear_memories` — backed by the hosted Engram MCP server. Works via the `mcp-remote` stdio bridge, because NextChat's MCP client is stdio-only (see [`app/mcp/client.ts`](https://github.com/ChatGPTNextWeb/NextChat/blob/main/app/mcp/client.ts) — it constructs a `StdioClientTransport` with no SSE/HTTP fallback).

## Setup

### 1. Get an Engram API key

Sign up at <https://lumetra.io> — free tier, no card. You'll see an `eng_live_…` token in your dashboard.

### 2. Configure a BYOK provider key

Engram is bring-your-own-key end-to-end for the LLM that handles extraction and synthesis. Configure one provider at <https://lumetra.io/models>. DeepSeek is what we recommend — cheap and fast. Without a provider key, every `store_memory` / `query_memory` returns HTTP 412.

### 3. Install Node.js (one-time)

`mcp-remote` runs via `npx`, which needs Node.js LTS. Install from <https://nodejs.org/> if you don't have it.

### 4. Enable MCP in NextChat

MCP support in NextChat is **off by default** and must be turned on at build/run time via an env var. Pick the path that matches how you run NextChat:

#### Docker

```bash
docker run -d -p 3000:3000 \
  -e ENABLE_MCP=true \
  -v "$PWD/mcp_config.json:/app/app/mcp/mcp_config.json" \
  yidadaa/chatgpt-next-web
```

#### From source

```bash
git clone https://github.com/ChatGPTNextWeb/NextChat.git
cd NextChat
ENABLE_MCP=true yarn build
ENABLE_MCP=true yarn start
```

#### Vercel / hosted

Set `ENABLE_MCP=true` in your environment variables and redeploy. Note: stdio MCP servers spawn child processes, which means the host running NextChat needs `npx` available and outbound network. Most serverless platforms (Vercel functions, edge runtimes) will **not** work — use Docker on a VM, Fly.io with a persistent machine, Railway, or self-hosted.

### 5. Drop the config file into NextChat's working directory

NextChat reads MCP servers from `app/mcp/mcp_config.json` relative to its working directory (see [`app/mcp/actions.ts`](https://github.com/ChatGPTNextWeb/NextChat/blob/main/app/mcp/actions.ts)):

```js
const CONFIG_PATH = path.join(process.cwd(), "app/mcp/mcp_config.json");
```

Copy [`mcp_config.example.json`](./mcp_config.example.json) to that path:

```bash
cp mcp_config.example.json /path/to/NextChat/app/mcp/mcp_config.json
```

…and replace `eng_live_...` with your real key. The file contents:

```json
{
  "mcpServers": {
    "engram": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@0.1.5",
        "https://mcp.lumetra.io/mcp/sse",
        "--header",
        "Authorization:Bearer eng_live_..."
      ]
    }
  }
}
```

> **Why `mcp-remote@0.1.5`?** NextChat's official Docker image ships Node 18.20.8, and `mcp-remote@0.1.6+` pulls `undici@7` which requires Node 20+. Using the latest `mcp-remote` crashes the bridge at startup with `ReferenceError: File is not defined`. `0.1.5` is the last release that works on Node 18.

For Docker, mount the file as shown in step 4. For Vercel/hosted, NextChat's UI also has an MCP server panel (Settings → MCP Market) that writes to the same file — you can paste the same JSON there if filesystem editing isn't available.

### 6a. Claude 4.x users — pick the right model

NextChat sends both `temperature` and `top_p` in every Anthropic request, which **Claude 4.x rejects** (`temperature and top_p cannot both be specified`). Workarounds, in order of least invasive:

1. **Use Claude 3.x** (`claude-3-5-sonnet-latest`, `claude-3-5-haiku-latest`) — they accept both params without complaint. This is the simplest fix.
2. **Use a non-Anthropic model** — OpenAI / Gemini / DeepSeek aren't picky about the param combo.
3. **Patch NextChat** — strip `top_p` from the request payload before it hits Anthropic. Tracked upstream as a NextChat bug; we'll update this recipe once it's fixed.

### 6. Restart NextChat and verify

Restart the server, open the chat UI, and look for the MCP indicator next to the message input. The `engram` server should report **active**. Then ask:

> Use the engram store_memory tool to remember "nextchat verification". Then call engram list_memories on bucket "default" with limit 3. Print the raw tool outputs.

You should see the stored `memory_id` and the bucket listing rendered inline.

## Tools exposed

| Tool | What it does |
|---|---|
| `store_memory(content, bucket?)` | Save a fact to a bucket (defaults to `"default"`). |
| `query_memory(question, bucket?)` | Hybrid retrieval + synthesized answer with citations. |
| `list_memories(bucket, limit?)` | Newest-first list of memories in a bucket. |
| `list_buckets(limit?, offset?)` | Paginated list of all buckets in your tenant. |
| `delete_memory(memory_id, bucket)` | Remove a single memory. |
| `clear_memories(bucket)` | Empty a bucket. Destructive. |

## Why the stdio bridge

NextChat's MCP client wraps the official `@modelcontextprotocol/sdk` but only wires up `StdioClientTransport` — no SSE or Streamable HTTP. Engram's hosted endpoint is SSE, so we use [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) as a stdio↔SSE bridge — same pattern as Claude Desktop, Cline, AnythingLLM, and other stdio-only clients.

If NextChat upstream adds SSE/HTTP support later, the config simplifies to a single `"url"` entry and the `npx` shim goes away.

## Troubleshooting

- **No MCP panel in the UI** — `ENABLE_MCP=true` wasn't set when NextChat started. The flag is read at build time for the static UI and at runtime for the API routes; both need it.
- **`engram` shows status `error`** — check NextChat's stderr for `mcp-remote` output. The most common failures: `npx` not on PATH inside the container, or outbound HTTPS to `mcp.lumetra.io` blocked.
- **HTTP 401 from Engram** — re-check the `Authorization:Bearer ...` header. The colon between `Authorization` and `Bearer` is required by `mcp-remote`'s header parser (no space after the colon, single space between `Bearer` and the token).
- **HTTP 412 from Engram** — you haven't configured a BYOK provider key at <https://lumetra.io/models>.
- **`File is not defined`** — your Node.js is older than 20. Pin `mcp-remote@0.1.5` for Node 18 compatibility: replace `"mcp-remote"` with `"mcp-remote@0.1.5"` in `args`.

## Source & contact

- Source: <https://github.com/lumetra-io/engram-nextchat>
- Issues: <https://github.com/lumetra-io/engram-nextchat/issues>
- Lumetra: <https://lumetra.io> · <support@lumetra.io>

## License

MIT — Lumetra
