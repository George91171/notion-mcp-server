# Notion MCP Server — Connect Claude, Cursor & VS Code to Notion

[![npm version](https://img.shields.io/npm/v/notion-mcp-server)](https://www.npmjs.com/package/notion-mcp-server)
![NPM Downloads](https://img.shields.io/npm/dw/notion-mcp-server)
![License](https://img.shields.io/badge/license-MIT-green)
![Model Context Protocol](https://img.shields.io/badge/MCP-Streamable_HTTP_+_stdio-purple)
![Stars](https://img.shields.io/github/stars/awkoy/notion-mcp-server)

Give your AI read/write access to Notion with one token and one command. Claude Code, Claude Desktop, Cursor, VS Code, Cline, Zed, anything that speaks MCP: it can create pages, query databases, append blocks, apply templates, comment and upload files, in plain language.

Notion ships its own MCP server. Where this one differs:

- **It authenticates with a token, so it runs headless.** Notion's hosted MCP is OAuth-only and someone has to click "Authorize". This one works in CI, cron jobs, background agents and self-hosted deployments.
- **It doesn't spend your context on tool schemas.** The official open-source server loads 24 endpoint schemas into the model's context at connection: 17,163 tokens, re-sent with every request for the rest of the session. This one loads three tools, 1,005 tokens — 94% less, 17× smaller — and fetches an operation's schema only when a task actually touches it.
- **It doesn't spend your context on answers either.** Reading the same pages through both servers, pulling a page's content costs **82% less** (26,071 → 4,568 tokens on an 88-block page), a 25-row database query **81% less**, a page object **68% less**. Notion's raw JSON is mostly `id`/`type` wrappers, `annotations`, and `created_by`/`parent`/`icon` blocks, and none of it reaches the model. That is the half that compounds, because a tool surface is paid once and responses are paid on every call. [Measured against a reproducible fixture, with the caveats stated →](./benchmarks#part-2--the-responses)

Nothing is lost to get there: a database query returns flat name → value rows instead of Notion's raw `properties` bags (5.3× lighter in the benchmark), and `verbose: true` gives you the untouched SDK shape whenever you want it — within 4 tokens of what the official server returns, which is how the benchmark proves both are reading the same thing. Batched mutations with atomic rollback, idempotency keys, retry on rate limits and self-healing validation errors are built in, and the [comparison below](#which-notion-mcp-should-you-use) has the rest.

<a href="https://glama.ai/mcp/servers/zrh07hteaa">
  <img width="380" height="200" src="https://glama.ai/mcp/servers/zrh07hteaa/badge" alt="Notion MCP Server on Glama" />
</a>

## Quick start

**1. Get a Notion token.** Open **[app.notion.com/developers/tokens](https://app.notion.com/developers/tokens)** → **+ New token** → name it, pick your workspace → **Create token** → copy the `ntn_…` value. A Personal Access Token sees everything *you* can see, with no per-page sharing. (Page missing or empty? Your admin disabled PATs — see [auth alternatives](#token-pat-or-internal-integration).)

<img src="https://raw.githubusercontent.com/awkoy/notion-mcp-server/main/assets/notion-pat-page.png" width="640" alt="Notion developer portal — the Personal access tokens page with the + New token button in the top right">

**2. Install it.**

```bash
npx add-mcp notion-mcp-server --env NOTION_TOKEN=ntn_paste_your_token_here
```

[`add-mcp`](https://github.com/neon-solutions/add-mcp) finds the MCP clients on your machine and writes the config for the ones you pick: Claude Code, Claude Desktop, Cursor, VS Code, Codex, Gemini CLI, Cline, Windsurf, Zed and a dozen others. Add `-g` to install at user level instead of the current project, `-a claude-code` to skip the picker, `--all` to write every client at once.

> Keep the `--env` flag. Without it the entry is written without a token, and the server starts and then fails every call with an auth error.

<details>
<summary><b>Or install it by hand: JSON config, Claude Code, Cursor, VS Code, Gemini CLI, Claude Desktop, Docker</b></summary>

**Any client that reads an `mcpServers` block** (Cursor's `~/.cursor/mcp.json`, Claude Desktop's `claude_desktop_config.json`, Cline's settings, Zed, Continue…):

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "notion-mcp-server"],
      "env": { "NOTION_TOKEN": "ntn_paste_your_token_here" }
    }
  }
}
```

**Claude Code:**

```bash
claude mcp add notion -s user \
  -e NOTION_TOKEN=ntn_paste_your_token_here \
  -- npx -y notion-mcp-server
```

Claude Code speaks the 2025-era protocol over stdio unless told otherwise. Set `MCP_PROTOCOL_NEGOTIATION=auto` in its environment and it probes for MCP 2026-07-28 (stateless requests, cache hints on every list). The server serves both.

**Cursor:** [![Install MCP Server](https://cursor.com/deeplink/mcp-install-dark.svg)](https://cursor.com/install-mcp?name=notion&config=eyJjb21tYW5kIjoibnB4IiwiYXJncyI6WyIteSIsIm5vdGlvbi1tY3Atc2VydmVyIl0sImVudiI6eyJOT1RJT05fVE9LRU4iOiJZT1VSX05PVElPTl9UT0tFTiJ9fQ==) — click, then replace `YOUR_NOTION_TOKEN` in the generated entry.

**VS Code (Copilot agent mode):** [![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Notion_MCP-0098FF?logo=githubcopilot)](https://insiders.vscode.dev/redirect/mcp/install?name=notion&inputs=%5B%7B%22id%22%3A%22notion_token%22%2C%22type%22%3A%22promptString%22%2C%22description%22%3A%22Notion%20Personal%20Access%20Token%20(ntn_...)%22%2C%22password%22%3Atrue%7D%5D&config=%7B%22command%22%3A%22npx%22%2C%22args%22%3A%5B%22-y%22%2C%22notion-mcp-server%22%5D%2C%22env%22%3A%7B%22NOTION_TOKEN%22%3A%22%24%7Binput%3Anotion_token%7D%22%7D%7D) — VS Code prompts for the token and stores it as a secret input.

**Gemini CLI:**

```bash
gemini extensions install https://github.com/awkoy/notion-mcp-server
```

The repo ships a `gemini-extension.json`, so this installs as an extension: it asks for the token once, keeps it in your system keychain, and starts the server with `npx`.

**Claude Desktop, without Node.js:** download [`notion-mcp-server.mcpb` from the latest release](https://github.com/awkoy/notion-mcp-server/releases/latest/download/notion-mcp-server.mcpb) and double-click it (or drag it into Settings → Extensions), then paste your token when prompted. Never edited a config file before? The [step-by-step walkthrough](./docs/claude-desktop-walkthrough.md) assumes nothing.

**Docker / Podman / OrbStack:**

```bash
claude mcp add notion -s user \
  -e NOTION_TOKEN=ntn_paste_your_token_here \
  -- docker run --rm -i -e NOTION_TOKEN ghcr.io/awkoy/notion-mcp-server:latest
```

The `-i` flag is required for stdio. The image is OCI-compliant, so Podman, OrbStack, colima, Rancher Desktop, Finch and nerdctl take the same flags. For a long-running HTTP container see [Remote / HTTP transport](#remote--http-transport).

</details>

**3. Try it.** In a new chat:

> *"Use Notion to make a page called 'Hello from my agent' and add a checklist of three things to try today."*

Your AI calls `notion_write` and replies with a live page link.

## What your AI can do with it

- *"Find every row in my Tasks database where Status is 'Doing' and tell me which are overdue."* — typed `where` filters, flattened rows
- *"Rename these 50 pages to the new convention."* — one batched call, 10-way parallel, idempotent retry
- *"Create a page from my 'Weekly review' template and fill in this summary."*
- *"Rewrite that spec page: fix the headings and add a code sample."* — markdown round-trip via `get_page_markdown` → edit → `update_page_markdown`
- *"Comment on yesterday's meeting notes with a one-paragraph summary."*
- *"Upload this diagram to the design page."* — single- and multi-part uploads
- *"Look at the screenshot on that bug report and tell me what's wrong."* — `get_image` hands the model the picture itself

The full catalog is the [operations menu](#operations-menu-47-ops-plus-one-alias): 47 operations behind three tools.

## Which Notion MCP should you use?

| | Best for | Auth | Headless / CI | Notes |
| --- | --- | --- | --- | --- |
| **[Notion hosted MCP](https://developers.notion.com/guides/mcp/get-started-with-mcp)** (`mcp.notion.com`) | Interactive chat in claude.ai, ChatGPT, Cursor | OAuth (a human must click; Notion says non-interactive auth is in the works) | ❌ | First-party, ~34 markdown tools (11 of them Custom Agent session tools that need Notion AI), some plan-gated |
| **[Official open-source server](https://github.com/makenotion/notion-mcp-server)** | — | Token | ✅ | Notion calls it deprecated and "no longer actively maintained"; the repo says it "may sunset" it and that issues and PRs are not actively monitored |
| **This server** | Agents, automation, CI, self-hosting, token-sensitive workloads | Token (PAT) | ✅ | Actively maintained, agent-first design |

To chat with your Notion in claude.ai's web UI, use Notion's hosted connector: it's one click. Use this server when the agent runs unattended, when context cost matters, or when you want batch and idempotent semantics and your own host.

<details>
<summary><b>Detailed comparison vs. the official open-source server</b></summary>

| Capability | Official Notion MCP (open source) | **This server** |
| --- | --- | --- |
| **Tool surface** | 24 tools (one per endpoint), 17,163 tokens loaded into context | **3 tools**, 1,005 tokens — [94% less schema at connection](./benchmarks) |
| **Response size** | Full Notion envelope on every read | [**82% less** reading a page's blocks](./benchmarks#part-2--the-responses), 81% on a 25-row database query, 68% on a page object, 71% on a search — same objects, both servers, matched pairs |
| **Operations covered** | ~24 endpoints | **47 operations** (plus a `trash_page` alias) across pages, blocks, databases, data sources, views, templates, comments, users, files |
| **Batch mutations** | Not documented | ✅ Universal `{ items: [...] }` envelope; up to **10 in parallel** |
| **Atomic batches + rollback** | Not documented | ✅ `atomic: true` aborts on first failure, best-effort archives entities created earlier |
| **Idempotency** | Not documented | ✅ `idempotency_key` — same key + op returns the cached result for 5 minutes |
| **Rate-limit handling** | 429s bubble up | ✅ Token-bucket limiter (3 req/s default) + exponential backoff, honors `Retry-After` |
| **Response shapes** | Raw Notion SDK JSON | **Slim shapers** drop noise by default; `verbose: true` opts out and returns the raw shape |
| **Database queries** | Raw `properties` bag per row | **Flattened** name → primitive map (all 20+ property types) — 16,629 → 3,143 tokens on the benchmark's 25-row query |
| **Writing properties** | Full Notion property JSON | Plain values: `{ Status: "Done", Due: "2026-10-01", Tags: ["a"] }`, typed from the data source schema (cached 5 min); wrong names and options rejected with the valid ones |
| **Filters** | Raw Notion filter JSON | Typed `where` shorthand — `{ Status: "Done", Priority: { in: [...] }, OR: [...] }` and `sorts: ["-Due Date"]`; raw filters still accepted |
| **Unknown fields** | Rejected | Ignored with a `warnings` entry naming the field and the accepted ones, so the call still runs |
| **Pagination** | Manual cursors | Opt-in `paginate: true` walks `next_cursor` (cap ≈ 1000 items) |
| **Wire format** | Default SDK serialization | **Compact JSON** — ~30% smaller payloads |
| **Markdown** | Page-level markdown tools | ✅ Accepted by `create_page` / `append_blocks` / `update_block` / comments, plus full round-trip (`get_page_markdown` / `update_page_markdown`), full GFM |
| **Templates** | — | ✅ `create_page` from a Notion template + `list_data_source_templates` discovery |
| **Database views** | — | ✅ list / get / query / create / update / delete; `query_view` runs a view's stored filters and sorts and returns hydrated rows |
| **File uploads** | Not in the documented tool surface | ✅ Single- and multi-part (5 MB chunks), MIME inferred |
| **Validation errors** | Plain error string | **Self-healing**: `{ code, message, path, issues, schema, example, fix }` — corrected in one round-trip |
| **Notion API version** | — | Pinned `2026-03-11` (data sources, views, templates) |

What that buys you in practice: renaming 50 pages is one `notion_write` call with `{ items: [...], concurrency: 10 }` rather than 50 trips through the agent's reasoning loop, and the prompt-token savings are the bigger half of the win. The [benchmark](./benchmarks) has the method, the tokenizer, the control that validates it, an honest worst case, and the limits of its own sample.

</details>

## Configuration

### Token: PAT or internal integration

Both go in the same `NOTION_TOKEN` env var; only where you get them differs.

| | **Personal Access Token** (recommended) | **Internal Integration** (scoped) |
| --- | --- | --- |
| Where | [app.notion.com/developers/tokens](https://app.notion.com/developers/tokens) → **+ New token** | [app.notion.com/developers/connections](https://app.notion.com/developers/connections) → **+ New connection** |
| Scope | Everything **you** can see | Only pages where you clicked **• • • → Connect → \<integration\>** |
| Friction | None | A Connect step per page or database |
| Use when | Default: personal and team workspaces, prototyping | An admin requires explicit per-resource scoping, or for shared production bots |

> 💡 Most `object_not_found` errors are the wrong auth choice rather than a bug: an Internal Integration token that was never Connected to the page. Switch to a PAT.

<details>
<summary><b>PAT details: capabilities, expiry, revocation, admin-disabled fallback</b></summary>

**Can:** read every page you have access to; create and update pages and databases where you have edit rights; comment as you; upload files.
**Can't:** reach pages you can't see, bypass workspace permissions, act as another user, or change admin settings. A PAT's scope is your account, so if you lose access to a page, so does the PAT. Issue separate tokens per teammate.

**Expiry:** PATs expire **1 year after creation** ([Notion docs](https://developers.notion.com/guides/get-started/personal-access-tokens)). Set a reminder for month 11.

**Revoking:** [app.notion.com/developers/tokens](https://app.notion.com/developers/tokens) → **Revoke** next to the token, effective immediately. Workspace admins can revoke anyone's from **Settings & members → Connections → All personal access tokens**.

**Admin disabled PATs?** Ask them to enable it, or create an Internal Integration at [app.notion.com/developers/connections](https://app.notion.com/developers/connections) (**+ New connection**) and **• • • → Connect** it to every page the agent should touch. Same `NOTION_TOKEN` env var.

Official reference: [PAT guide](https://developers.notion.com/guides/get-started/personal-access-tokens) · [Authorization overview](https://developers.notion.com/docs/authorization).

</details>

### Environment variables

| Env var | Required | Default | Meaning |
| --- | --- | --- | --- |
| `NOTION_TOKEN` | ✅ | — | PAT (`ntn_…`, recommended) or Internal Integration secret (`secret_…` / `ntn_…`) |
| `NOTION_PAGE_ID` | — | — | Default parent for `create_page` / `create_database` when no `parent` is passed (page → Share → Copy link; the whole URL or the bare 32-char id both work) |
| `NOTION_RATE_LIMIT` | — | `3` | Requests/second for the shared limiter (Notion's documented per-integration limit) |
| `NOTION_READ_ONLY` | — | — | `true`/`1`/`yes` disables every write operation in one switch |
| `NOTION_ALLOWED_OPERATIONS` | — | all | Comma-separated allowlist of operations or group presets — see [Restricting operations](#restricting-operations) |
| `NOTION_BLOCKED_OPERATIONS` | — | — | Comma-separated blocklist (same vocabulary); wins over the allowlist |
| `NOTION_CONFIRM_DESTRUCTIVE` | — | — | `true`/`1` keeps destructive operations enabled but asks you first — see [Restricting operations](#restricting-operations) |
| `NOTION_UPLOAD_ROOT` | — | — | Confine `upload_file`'s `path` source to one directory — see [Files](#files) |
| `NOTION_FILE_URLS` | — | `full` | `ref` replaces Notion's signed file URLs (~1,650 chars, valid for an hour) in slim responses with short `notion-file:` refs — see [Files](#files) |
| `HTTPS_PROXY` / `HTTP_PROXY` | — | — | Route all outbound traffic — Notion API calls and the downloads in `get_image` and `upload_file`'s `url` source — through an HTTP(S) proxy (standard env vars, lowercase also accepted) |
| `NOTION_DAILY_LOG_PAGE_ID` | — | — | Only used by the daily-log MCP prompt |

HTTP-transport variables (`MCP_TRANSPORT`, `PORT`, `HOST`, `MCP_AUTH_TOKEN`, …) are in [Remote / HTTP transport](#remote--http-transport).

> **Upgrading from v1.x or v2.x?** Every env var still works unchanged. The break is the tool surface: v1's five tools, then v2's `notion_execute`, became `notion_read` + `notion_write`, and `notion_describe` is as it was. Modern clients rediscover tools automatically. Details in [MIGRATION.md](./MIGRATION.md).

### Restricting operations

`NOTION_ALLOWED_OPERATIONS` (allowlist) and `NOTION_BLOCKED_OPERATIONS` (blocklist) each take a comma-separated list of **group presets** or exact **operation names**.

| Preset | Expands to |
| --- | --- |
| `read` | every non-mutating operation |
| `write` | every mutating operation |
| `destructive` | operations whose purpose is removal (`archive_page`/`trash_page`, `delete_block`, `batch_mixed_blocks`, `delete_comment`, `delete_view`) |
| `pages` `blocks` `databases` `data_sources` `views` `comments` `users` `files` | every operation in that family, read **and** write |

```jsonc
{ "env": { "NOTION_ALLOWED_OPERATIONS": "read" } }                       // read-only, the common case
{ "env": { "NOTION_BLOCKED_OPERATIONS": "destructive" } }                // everything except removals
{ "env": { "NOTION_ALLOWED_OPERATIONS": "read,append_blocks,add_page_comment" } }
```

Names are case-insensitive, unknown tokens are ignored with a warning, the blocklist wins, and an allowlist that resolves to zero operations disables everything (fail-closed). Disabled operations vanish from the tools' `operation` enums, from `notion_describe` and from the `notion://operations` menu, so naming one fails validation before it runs; when no write operation is enabled, `notion_write` is not advertised at all. One line on stderr at startup says what resolved. Check it first when the config doesn't behave:

```text
Operation access: 22/48 enabled (allow=read; block=(none))
```

**Confirm instead of block.** `NOTION_CONFIRM_DESTRUCTIVE=true` keeps destructive operations available and makes `notion_write` ask you before running one, through [MCP elicitation](https://modelcontextprotocol.io/specification/2025-11-25/client/elicitation): an `elicitation/create` request on 2025-era clients, an `input_required` round trip on MCP 2026-07-28 clients, where the retry carries a sealed `requestState` that only matches the call it was minted for. You get a yes/no dialog naming the operation and its target (the page, database, data source or block title when one retrieve can fetch it within 5 s, otherwise the id; for a batch, how many items).

Restores (`restore_page`, `delete_database` / `delete_data_source` with `in_trash: false`) and a `batch_mixed_blocks` call with no `delete` entry never prompt, and a blocked operation is still rejected with `operation_not_allowed` before anyone is asked. Decline, cancel or answer no and the call returns `confirmation_declined`; the server instructions tell the model not to retry and to ask you instead. A client that hasn't declared the elicitation capability gets `confirmation_unavailable` rather than a silent run. Use a client that supports elicitation, unset the variable, or block destructive operations outright.

<details>
<summary><b>Per-operation reference & limitations</b></summary>

| Domain | Read |