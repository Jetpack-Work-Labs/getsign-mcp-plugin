---
name: "install-getsign-on-new-account"
description: "Set up GetSign on a monday.com account for the first time: check whether the app is installed, hand the account admin a single link that covers both monday's install screen and OAuth consent, then confirm the connection works and add a GetSign board view. Use when someone says: 'set up getsign'; 'install getsign on monday'; 'connect my monday account to getsign'; 'getsign isn't installed'; 'onboard my account'."
when_to_use: "Starts with `getsign_account`. Needs no prior context — safe to run cold."
license: "MIT"
allowed-tools:
  - mcp__plugin_getsign_getsign__getsign_account
  - mcp__getsign__getsign_account
  - mcp__plugin_getsign_getsign__getsign_connect
  - mcp__getsign__getsign_connect
  - mcp__plugin_getsign_getsign__getsign_list_workflows_for_board
  - mcp__getsign__getsign_list_workflows_for_board
  - mcp__plugin_getsign_getsign__getsign_ensure_board_view
  - mcp__getsign__getsign_ensure_board_view
  - mcp__plugin_getsign_getsign__getsign_create_workflow
  - mcp__getsign__getsign_create_workflow
metadata:
  source_flow: "install_getsign_on_new_account"
  generated_from: "FLOWS in getsign_mcp/services/skills.py"
  getsign_mcp_version: "0.1.0"
---

# Set up GetSign on a monday.com account

Use this when GetSign is not connected yet — a new account, a fresh workspace,
or a session where the tools come back unauthorized.

Everything here runs from chat except one click, which has to be made by a
monday.com **account admin** in a browser. Establish early whether the person
you are talking to is one; if not, the deliverable is a link for them to
forward, not a dead end.

## Two questions that look like one

Keep these apart — they fail differently and the fixes are unrelated. Both are
`getsign_account`, on different actions:

- **Does this session's token work?** `action=preflight` (the default, and the
  recommended first call) — checks a token is configured, then hits `/account`.
  `action=status` is the weaker version: it only reports whether a token exists,
  with no network call.
- **Is the app installed on the monday account?** `action=install_state`. Pass
  `account_id` for an account that has never connected.

A working token does **not** prove the app is installed: when an account
uninstalls, the account record survives and its tokens keep working. And
installing never authorizes GetSign — the OAuth round trip always has to happen
afterwards. That is why the check appears on both sides of the install below.

## Reading the install state honestly

`installed` is **tri-state**. `null` means not known — report it as unknown, not
as either answer.

Read `install_state_source` before trusting a `true`:

- `app_event` — a webhook from monday. Authoritative.
- `oauth_callback` — proves an install as of that timestamp, no later.
- `event_log_backfill` — reconstructed from the event log. Good.
- `legacy_account` — inferred from the account row merely existing, dated at its
  creation. A strong hint, not proof. `state_is_inferred` will be `true`.

If you need certainty on an inferred `true`, run `getsign_connect` with
`action=install` — the link it mints carries monday's `force_install_if_needed`,
which shows the install screen exactly when the app is not installed and skips
straight to consent when it is. That makes the link self-verifying, at the cost
of one browser tab.

## Connecting

`getsign_connect` covers both paths:

- `action=oauth` (default) — a secure GetSign login. It returns
  `already_connected` with no URL when the session already works, so it will not
  pointlessly re-auth; pass `force=true` when you actually want a fresh login.
- `action=install` — for an account that has never installed the app. Returns a
  single URL covering monday's install screen *and* OAuth consent, so one admin
  click covers both.

Prefer this over asking anyone to paste a monday session token.

`action=install` also returns advisory `install_app` mutation guidance. That is
for a caller with its own monday admin API access (a monday MCP connector, say)
that wants an account-wide install. Two things to know: the mutation is absent
from monday's public API reference, so it carries no compatibility guarantee and
non-admins get `UNAUTHORIZED_EXCEPTION`; and **it cannot confirm its own
result** — `created: false` covers both "already installed" and "an old record
reactivated". Never report the app as installed on the strength of it.
Completing the URL is the confirmation.

If a non-admin opens the link they land on monday's red "App is not installed"
box with a **disabled** Authorize button. That is a permissions problem, not a
broken link — route it to an admin.

## Steps

1. `getsign_account`
   - next `getsign_connect`
   - failures: If auth is missing, preflight does not call authenticated backend routes and returns getsign_connect as the next step. installed is tri-state — null means not known. Never infer install state from action=status or a successful /account.
2. `getsign_connect`
   - next `getsign_account`
   - failures: The user must approve access in the browser, then run getsign_account again. A valid token is never proof the app is installed — use getsign_account action=install_state. Installing ≠ authorizing.
3. `getsign_account`
   - next `getsign_connect`
   - failures: If auth is missing, preflight does not call authenticated backend routes and returns getsign_connect as the next step. installed is tri-state — null means not known. Never infer install state from action=status or a successful /account.
4. `getsign_list_workflows_for_board`
   - requires `board_id`
   - next `getsign_get_workflow`, `getsign_ensure_board_view`, `getsign_create_workflow`
5. `getsign_ensure_board_view`
   - requires `board_id`
   - next `getsign_list_workflows_for_board`, `getsign_get_workflow`
   - failures: Needs a live AppFeatureBoardView named 'getsign board view' on the installed GetSign app version, and Monday scopes that allow creating board views.
6. `getsign_create_workflow`
   - **conditional — ask before calling this when a workflow already exists.** The `getsign_list_workflows_for_board` step above is a branch, not a formality: if it returned any workflow, do NOT silently reuse one and do NOT silently call this tool either. Present the existing workflow(s) (name + `envelope_id`) alongside a 'create a new workflow' option and ask the user to choose — call `getsign_get_workflow` on any they're considering to surface its settings first. Only call this tool if the user picks create; every call creates a brand-new envelope on the board with no dedup by name, so calling it unasked litters the board with duplicate workflows. Call it directly, without asking, only when the list came back empty.
   - requires `board_id`, `workflow_name`
   - next `getsign_ensure_board_view`, `getsign_select_template_for_workflow`, `getsign_get_workflow`
   - failures: MISSING_WORKFLOW_NAME: workflow_name was blank or whitespace-only, so nothing was created. Ask the user what to name the workflow — don't pick one yourself — then retry. The name shows on the monday board, so suggest the source document or agreement type (e.g. 'NDA - Acme Corp').

## Once connected

Add the GetSign board view as soon as a board is named for signing — it is
idempotent and safe to call early, so do not wait until a workflow exists. Gate
it on actual signing intent though: it writes a visible tab, and stamping one
onto a board the user is only browsing is noise.

Check `getsign_list_workflows_for_board` before creating anything. If a
workflow already exists, ask the user whether to reuse it or create a new
one — present both options rather than picking for them.

Creating a workflow needs the board only — never ask for an item to get one.

When you do create or fetch one, it comes back with a large settings blob that
already has real defaults applied: reminders, whether the sender gets a copy of
the signed document, OTP enforcement, approval rules, the email sender identity.
Summarize the ones a sender would care about and ask whether they want anything
changed before moving on. Do not proceed silently on defaults the user never
saw.

The feature toggles — Generate document, Signature collection, Share and track,
Form filler — are **optional and off by default**. A workflow with all of them
off is complete and can send. Present them as options if asked; do not turn one
on unprompted and do not tell the user they need one to send.

## Gotchas

See [references/gotchas.md](../../references/gotchas.md).
