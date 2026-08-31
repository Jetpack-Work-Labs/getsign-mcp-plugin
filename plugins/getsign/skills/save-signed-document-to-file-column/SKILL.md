---
name: "save-signed-document-to-file-column"
description: "Save a signed document into a monday.com File column: check whether the workflow's Signature collection or Generate document settings already do this automatically, and if not, download the signed PDF and hand it off to a connected monday API tool to attach it to the item's File column. Use when someone says: 'save the signed document in the files column'; 'save this to the file column'; 'attach the signed pdf to this item'; 'put the signed contract back on the board'; 'upload the signed doc to the files column'."
when_to_use: "Starts with `getsign_download_signed_documents`. Needs `envelope_id`, `item_id` to begin; ask for what is missing rather than guessing."
license: "MIT"
allowed-tools:
  - mcp__plugin_getsign_getsign__getsign_download_signed_documents
  - mcp__getsign__getsign_download_signed_documents
metadata:
  source_flow: "save_signed_document_to_file_column"
  generated_from: "FLOWS in getsign_mcp/services/skills.py"
  getsign_mcp_version: "0.1.0"
---

# Save a signed document into a monday File column

Someone finishes a signature request and asks to have the signed copy land in a
File column on the board, alongside (or instead of) sending a download link.
Whether that is possible with a tool call, or already happening on its own,
depends entirely on how the workflow is configured.

## Check for the automatic path first

Two workflow settings make GetSign's own backend write the file there, with no
tool call needed on your part:

- **Signature collection** (`signatureCollection.isEnabled` +
  `fileColumnId`) — writes the signed file to that File column the moment
  signing completes. This is the setting that does it; Sign anywhere
  (`enableSignAnywhere`) is a sub-option of the same feature about signing
  without predefined signature pads, and has no bearing on the file write.
- **Generate document** (`outputColumnId`) — writes a generated file to that
  column when a status column reaches its configured label.

`getsign_get_workflow`'s `display_summary` shows whether either is on. If one
is, and it's pointed at the column the user means, there is nothing to do —
it already happened or will happen automatically.

**Neither setting is retroactive.** Turning one on now does not backfill a
document that was already signed before the setting existed. Do not enable a
feature toggle on the user's behalf just to satisfy this request — that
decision belongs to them, same as any other workflow setting (see the
`create-or-update-workflow` skill).

## No setting configured, or the document is already signed

This is the common case when someone asks for it after the fact. It is not one
tool call — it is a hand-off between two servers, because GetSign MCP has no
monday-write capability of its own:

## Steps

1. `getsign_download_signed_documents`
   - requires `envelope_id`, `item_id`
   - next `getsign_status`
   - failures: Fails if signing is not complete, no signed documents exist, or the signed snapshot is not ready yet. Links expire quickly and must not be shared publicly.

That first step only gets you a presigned download link — `data.download_url`
(or `download_urls` for multiple files), short-lived and meant for a browser.
From there, a connected monday API MCP finishes the job:

1. Download the file from that URL (e.g. to a temp path).
2. Call that server's asset-upload tool to mint a presigned upload URL
   (commonly named something like `get_asset_upload_url`), giving it the file
   name, content type, and size.
3. PUT the downloaded bytes to the returned URL, and keep the `ETag` response
   header.
4. Call the finalize step (commonly `finalize_asset_upload`) with the upload
   id, that `ETag`, and the target `board_id` / `item_id` / `column_id` — this
   is what actually attaches the file to the item's File column.

## If there is no monday connector in the session

Say so. There is no fallback inside GetSign MCP — no endpoint here or in
`getsign-backend` takes an arbitrary file plus an item and column id and
uploads it on demand. Do not invent one, and do not silently drop the request;
tell the user this needs a monday-capable tool connected alongside GetSign to
complete.

## Gotchas

See [references/gotchas.md](../../references/gotchas.md), rule 13a.
