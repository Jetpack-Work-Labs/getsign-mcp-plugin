# GetSign for Claude Code

E-signatures on monday.com boards, driven from chat.

## Install

```
/plugin marketplace add Jetpack-Work-Labs/getsign-mcp-plugin
/plugin install getsign@getsign
```

## Authenticate — required, and not automatic

Installing the plugin wires up the MCP server but does **not** connect your
monday.com account. No browser opens on install. In a fresh session:

```
/mcp
```

Select `getsign`, choose **Authenticate**, and complete the monday.com
consent screen. You are done when `/mcp` shows `✔ Connected`.

If your monday.com account has never had the GetSign app installed, an account
admin has to click through monday's install screen once. The
`/getsign:install-getsign-on-new-account` skill produces that link and
walks the rest.

## Skills

- `/getsign:install-getsign-on-new-account` — Set up GetSign on a monday.com account for the first time: check whether the app is installed, hand the account admin a single link that covers both monday's install screen and OAuth consent, then confirm the connection works and add a GetSign board view.
- `/getsign:send-new-document` — Send a document that has no GetSign template yet out for signature from a monday.com board: upload it, attach it to a signing workflow, map the signature fields, point them at a signer column, send, then track through to the signed copy.
- `/getsign:board-pending-signatures` — Find out who still hasn't signed across a monday.com board, then drill into one item's signing history and full audit trail to see where it stalled and who to chase.

Claude also loads these on its own when what you ask matches — you do not have
to type the slash command.

## Already connected to GetSign MCP directly?

You can keep that connection. Claude Code deduplicates MCP servers by endpoint,
so you will see one `getsign` server rather than two, and the skills
work either way — they pre-approve both the plugin-scoped and direct tool names.

## Reporting issues

These files are generated from the GetSign MCP server's flow registry. File
issues on this repo; fixes land in the server repo and are regenerated here.
