# GetSign MCP Plugin

Claude Code plugin distribution repo for [GetSign](https://getsign.io) — e-signatures on monday.com boards, driven from chat.

> [!IMPORTANT]
> **Everything under `plugins/getsign/` and `.claude-plugin/` in this repo is generated** from the flow registry in the private [`getsign-mcp-server`](https://github.com/Jetpack-Work-Labs/getsign-mcp-server) repo and republished here by a one-way sync script. Nothing here is hand-edited except this file, `LICENSE`, and `.github/workflows/validate.yml`.
>
> **File issues on this repo — pull requests here will be silently overwritten by the next publish.** Fixes belong in `getsign-mcp-server` and flow back here through the normal generate → review → publish loop.

## Install

```
/plugin marketplace add Jetpack-Work-Labs/getsign-mcp-plugin
/plugin install getsign@getsign
```

See [`plugins/getsign/README.md`](plugins/getsign/README.md) for authentication, the skills this plugin ships, and what to do if you're already connected to the GetSign MCP server directly.

## What's in this repo

- `.claude-plugin/marketplace.json` — the marketplace manifest Claude Code reads for `/plugin marketplace add`.
- `plugins/getsign/` — the plugin itself: `.claude-plugin/plugin.json`, `.mcp.json`, skills, and reference docs.

Both are generated; see the note above.

## License

MIT — see [`LICENSE`](LICENSE).
