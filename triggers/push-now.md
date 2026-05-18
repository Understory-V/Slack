# Push-now (on-demand)

Fired manually from the Claude Code on the web UI whenever you've replied to
the morning digest with `push` / `skip` / `inbox` / `undo` commands. Reads
your replies, applies them to `state/pending-review.json`, and performs the
ClickUp writes.

---

## Step 1 — Load state

Read:
- `config/identity.yml`, `config/channels.yml`.
- `state/last-digest.json` — `{message_ts, channel_id, rows: [...]}`.
- `state/pending-review.json` — the full pending queue.
- `state/processed.jsonl` — for `undo` lookups.

If `last-digest.json` is missing or empty, post in `review_channel_id`:
*"No active digest — nothing to push. Run `slack-morning-review` first."*
and stop.

## Step 2 — Read the digest thread

`slack_read_thread` on `last-digest.message_ts` in `channel_id`,
`limit = 1000`. Keep replies authored by `my_slack_user_id` only.

## Step 3 — Parse reply commands

Case-insensitive. Tokens separated by whitespace or commas. Across multiple
reply messages, take the union; for the same row, later commands override
earlier ones.

Supported syntax:
- `push N [M K …]` — push those rows.
- `push all` — push every pending row.
- `push all high` — push every row with `confidence = "high"`.
- `push all low` — push every row with `confidence = "low"`.
- `inbox N [M …]` — push those rows but force list = `claude_inbox_list_id`.
- `skip N [M …]` — drop those rows from the queue (no ClickUp write).
- `skip all` / `skip all high` / `skip all low` — same shape as push.
- `undo <task_id_or_url>` — revert a previously-pushed task (see Step 5).
- Free-text in replies that doesn't match any command → ignored.

Build three sets: `to_push`, `to_inbox`, `to_skip`, plus `to_undo` (list of
task IDs / URLs). Row numbers refer to `row_index` in `pending-review.json`.

If `to_push ∪ to_inbox ∪ to_skip ∪ to_undo` is empty, post:
*"No commands found in thread replies. Reply with `push 1 3` etc. and fire
this trigger again."* and stop.

## Step 4 — Apply pushes

For each row in `to_push ∪ to_inbox`:

- **CREATE action:**
  - `list_id` = `would_be_list_id` (or `claude_inbox_list_id` if row was in
    `to_inbox`).
  - `clickup_create_task`:
    - `name` = `title`
    - `markdown_description` = full description (ask quote, affirmation,
      requester, Slack permalink, channel name, queued_at).
    - `due_date` = `due_date` if present.
    - `tags` = `["source:slack", "auto:claude", "conf:{confidence}"]`.
  - Append to `processed.jsonl`:
    `{ts: thread_ts, channel_id, action: "CREATE", task_id, confidence,
      run_at: "<iso8601>", run_type: "push-now",
      list_override: "inbox" if from to_inbox else null}`.

- **COMPLETE action:**
  - `clickup_update_task` on `match_task_id` setting status to `complete`
    (workspace-default complete status).
  - `clickup_create_task_comment` on `match_task_id`:
    `"Delivered: {delivery_permalink}"`.
  - Append to `processed.jsonl` with `action: "COMPLETE"`.

If a ClickUp call fails: log the row, leave it in `pending-review.json`,
include it in the failures summary below.

## Step 5 — Apply undos

For each entry in `to_undo`:
1. Normalize to a task ID (strip the ClickUp URL prefix if present).
2. Search `processed.jsonl` for the most recent matching entry.
3. If `action: "CREATE"` → `clickup_delete_task`.
4. If `action: "COMPLETE"` → `clickup_update_task` setting status to `open`
   (or the workspace default open status), and add a comment:
   *"Reopened via undo command."*
5. Append an `UNDO` entry to `processed.jsonl` referencing the original.

If the task ID isn't in `processed.jsonl`, refuse to undo and flag it in the
confirmation message — don't touch tasks Claude didn't create.

## Step 6 — Apply skips

Remove every row in `to_skip` from `pending-review.json`. Append a `SKIP`
entry to `processed.jsonl` for each so the same thread doesn't re-surface
tomorrow:
`{ts: thread_ts, channel_id, action: "SKIP", run_at, run_type: "push-now"}`.

## Step 7 — Rebuild pending-review.json

Remove every successfully-pushed and skipped row. Keep failures + rows you
didn't react to. Renumber `row_index` only on the next morning run (not here)
to avoid breaking ongoing replies.

## Step 8 — Post confirmation

Reply in the digest thread:

```
✅ Pushed {N} to ClickUp:
• Row 1 — Revic ZoomInfo takeout → <task link>
• Row 3 — Zapier review → <task link>
• Row 5 — Andrew GitHub org → <task link> (Claude Inbox via `inbox 5`)

🗑️ Skipped {M}: rows 2, 4

↩️ Undone {K}: <task id> (was CREATE)

⚠️ Failed {F}: rows X, Y — left in queue
{error detail}

📥 {remaining} rows still pending. Reply more commands and re-run
   `slack-push-now`, or wait for tomorrow's digest.
```

## Step 9 — Commit and push

```
git add state/processed.jsonl state/pending-review.json
git commit -m "push-now $(date -u +%Y-%m-%dT%H:%MZ): pushed {N}, skipped {M}, undone {K}"
git push -u origin claude/slack-reader-workflow-t1njI
```

Skip the commit if no changes. Done.
