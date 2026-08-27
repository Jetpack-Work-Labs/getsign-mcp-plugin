---
name: "send-new-document"
description: "Send a document that has no GetSign template yet out for signature from a monday.com board: upload it, attach it to a signing workflow, map the signature fields, point them at a signer column, send, then track through to the signed copy. Use when someone says: 'send this for signature'; 'get this signed'; 'send the contract out for signing'; 'send this NDA to the client to sign'; 'email this document for a signature'."
when_to_use: "Starts with `getsign_create_template`. Needs `file_name`, `content_type` to begin; ask for what is missing rather than guessing."
license: "MIT"
allowed-tools:
  - mcp__plugin_getsign_getsign__getsign_create_template
  - mcp__getsign__getsign_create_template
  - mcp__plugin_getsign_getsign__getsign_list_workflows_for_board
  - mcp__getsign__getsign_list_workflows_for_board
  - mcp__plugin_getsign_getsign__getsign_ensure_board_view
  - mcp__getsign__getsign_ensure_board_view
  - mcp__plugin_getsign_getsign__getsign_create_workflow
  - mcp__getsign__getsign_create_workflow
  - mcp__plugin_getsign_getsign__getsign_get_editor_url
  - mcp__getsign__getsign_get_editor_url
  - mcp__plugin_getsign_getsign__getsign_get_signer_data
  - mcp__getsign__getsign_get_signer_data
  - mcp__plugin_getsign_getsign__getsign_get_preview_url
  - mcp__getsign__getsign_get_preview_url
  - mcp__plugin_getsign_getsign__getsign_send_signature_request
  - mcp__getsign__getsign_send_signature_request
  - mcp__plugin_getsign_getsign__getsign_status
  - mcp__getsign__getsign_status
  - mcp__plugin_getsign_getsign__getsign_download_signed_documents
  - mcp__getsign__getsign_download_signed_documents
metadata:
  source_flow: "send_new_document"
  generated_from: "FLOWS in getsign_mcp/services/skills.py"
  getsign_mcp_version: "0.1.0"
---

# Send a new document for signature

Use this when the user has a document that GetSign has never seen before — a
PDF or DOCX on disk, in a monday.com File column, or one they are about to
produce — and wants it signed by someone.

This is not the flow for a document that already exists as a GetSign template.
If the user says "the usual NDA" or "our standard contract", call
`getsign_list_template_gallery` first and attach the existing template with
`getsign_select_template_for_workflow` instead of uploading a second copy.

## Before you start

Ask for whatever is missing — do not guess:

- **The board.** Everything is board-scoped. If the user names a board by title
  rather than id, resolve it before proceeding.
- **The document.** Either a file path, or confirmation that it is already in a
  monday File column on the item.
- **Who signs it.** Signers come only from an email, people, or mirror-email
  column that already exists on the board. Establish early that the board has
  one — if it does not, a human has to add it in monday before the send step can
  work, and that is much better surfaced at the start than after the upload.

An item id is *not* needed to create the workflow or attach the document. Only
bring one in at the field-mapping and send steps.

## Uploading: `getsign_create_template` is called TWICE

This is the part that trips people up. It is a two-phase presigned upload:

1. **Call 1** — `file_name` + `content_type`, no `storage_key`. Returns
   `data.url` (a presigned PUT) and `data.storage_key`.
2. **PUT the raw bytes** straight from disk to `data.url` —
   `curl -X PUT --data-binary @path -H "Content-Type: <type>" "<data.url>"`, or
   `curl -T path`. This is a direct request to file storage, not an MCP call.
   Never base64 or inline the file through chat here; that defeats the whole
   point of the presigned path.

   **It must be a `PUT`, sending `Content-Type` and no other header.** Drop
   `-X PUT` and curl sends a POST, which S3 refuses with
   `403 SignatureDoesNotMatch` — the URL is signed for `PutObject`. Add any
   `x-amz-*` header and it refuses with `403 AccessDenied — There were headers
   present in the request which were not signed`. The `x-amz-checksum-crc32`
   sitting in the query string is the checksum of an *empty* body; it is already
   signed and wants nothing from you, so never compute one. Keep the error body
   (never `-o /dev/null`): a `403` from S3 is a malformed request, not the
   unreachable-storage case below.
3. **Call 2** — same `file_name` / `content_type` / `board_id`, plus
   `storage_key` from call 1. This registers the template.

Pass `envelope_id` + `item_id` on call 2 and it auto-attaches, with no separate
`getsign_select_template_for_workflow` step needed.

**If your client cannot make that outbound PUT** — a sandboxed agent, a CI
runner behind an allowlist proxy, no browser — the PUT dies at the proxy and
then surfaces one step later as a confusing "key does not exist" error. There
is **no base64-over-MCP fallback** — one existed and was deliberately removed,
because a client with no outbound HTTP also has no shell to encode the file
with, and base64 written out token by token truncates into a corrupt document.
The bytes have to reach storage from somewhere that can make the request — a
local shell, or a human uploading through the GetSign UI.

First make sure it really is the egress case: a `403` from S3 itself is a
malformed request (see the PUT rules above), not unreachable storage. Blocked
egress dies at the proxy's CONNECT and never reaches AWS at all.

## Mapping the signature fields — offer both paths

Once the document is attached it has no signature fields yet. There are two ways
to add them, and the user should choose:

- **Manual** — `getsign_get_editor_url` opens the PDF editor and they drag
  fields where they want them.
- **AI-assisted** — `getsign_detect_placeholders_ai` finds the likely fields,
  then `getsign_save_placeholders` persists them.

Detection saves nothing on its own. Always follow it with the save step, and
only then open the editor to review. Jumping from detection straight to the
editor silently discards every detected field.

Pass `getsign_detect_placeholders_ai`'s `data.placeholders` through to
`getsign_save_placeholders` **unmodified** except for `field_assignments`. Never
hand-author `x`/`y`/`width`/`height` — the save route accepts a malformed
placeholder without complaint, and it then fails to render and fails to resolve
a signer.

Keep `edit_template` on `getsign_get_editor_url` consistent with
`is_template_update` on `getsign_save_placeholders`. Both default to
template-level, which propagates to every item using that template; pass `false`
on both only when the user explicitly wants the change confined to one item.

If the document has fields for more than one signer, call the save tool once per
signer with only that signer's placeholders — not the full detected array with a
new column each time.

## Steps

1. `getsign_create_template`
   - requires `file_name`, `content_type`
   - next `getsign_create_template`, `getsign_list_template_gallery`, `getsign_list_envelope_documents`
   - failures: storage_key from call 1 must be reused verbatim in call 2 — it names the S3 object the PUT just wrote to. Skipping the PUT step (or PUT-ing to the wrong url) makes call 2 register a template pointing at empty/missing content with no error at either step.
2. `getsign_list_workflows_for_board`
   - requires `board_id`
   - next `getsign_ensure_board_view`, `getsign_create_workflow`, `getsign_get_workflow`
3. `getsign_ensure_board_view`
   - requires `board_id`
   - next `getsign_list_workflows_for_board`, `getsign_get_workflow`
   - failures: Needs a live AppFeatureBoardView named 'getsign board view' on the installed GetSign app version, and Monday scopes that allow creating board views.
4. `getsign_create_workflow`
   - requires `board_id`, `workflow_name`
   - next `getsign_ensure_board_view`, `getsign_select_template_for_workflow`, `getsign_get_workflow`
   - failures: MISSING_WORKFLOW_NAME: workflow_name was blank or whitespace-only, so nothing was created. Ask the user what to name the workflow — don't pick one yourself — then retry. The name shows on the monday board, so suggest the source document or agreement type (e.g. 'NDA - Acme Corp').
5. `getsign_get_editor_url`
   - requires `envelope_id + item_id, or template_id + file_id`, `optional file_id on item`, `optional edit_template`
   - next `getsign_detect_placeholders_ai`, `getsign_get_signer_data`, `getsign_send_signature_request`
   - failures: Requires GETSIGN_APPLICATION_URL and a connected user session. Rejects both ID shapes or neither. If no item documents exist, attach or generate a document first.
6. `getsign_get_signer_data`
   - requires `envelope_id`, `item_id`
   - next `getsign_send_signature_request`, `getsign_get_editor_url`
7. `getsign_get_preview_url`
   - requires `envelope_id + item_id, or template_id + file_id`, `optional file_id on item`
   - next `getsign_send_signature_request`, `getsign_get_editor_url`
   - failures: Requires GETSIGN_APPLICATION_URL and a connected user session. If already fully signed, refuses with next_tool=getsign_download_signed_documents.
8. `getsign_send_signature_request`
   - requires `envelope_id`, `item_id`
   - next `getsign_status`, `getsign_status`, `getsign_get_signer_data`
   - failures: CONFIRMATION_REQUIRED: show summary_text, then re-call with confirm=true and confirmation_token (same params). INVALID_CONFIRMATION_TOKEN: start over without confirm. MISSING_SIGNATURE_FIELDS: response includes next_tool=getsign_get_editor_url — call it and share the editor URL.
9. `getsign_status`
   - requires `envelope_id+item_id for history, or board_id for board actions`
   - next `getsign_download_signed_documents`, `getsign_generate_signing_link`
10. `getsign_download_signed_documents`
   - requires `envelope_id`, `item_id`
   - next `getsign_status`
   - failures: Fails if signing is not complete, no signed documents exist, or the signed snapshot is not ready yet. Links expire quickly and must not be shared publicly.

## Before you send

`getsign_send_signature_request` is gated. It will come back asking for
confirmation with a summary of what is about to go out and to whom — show that
summary to the user and get a real answer before retrying with the token. This
is the last reversible moment.

If it fails on missing signature fields, the document has no mapped fields yet
or the signer column resolved to nothing on that item. Check the item's email
column actually has a value.

## After you send

Do not stop at "sent". `getsign_status` tracks it — `action=history` for this
item, `action=activity` for the board-level event feed. On completion `history`
returns a `next_tool` pointing at `getsign_download_signed_documents`; offer the
full audit trail alongside the download link, not just the link.

Nothing here writes back to a monday status column. If the user expects the
board to update on completion, say so plainly — that needs a monday connector.

## Gotchas

See [references/gotchas.md](../../references/gotchas.md).
