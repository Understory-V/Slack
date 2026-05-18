# Evening reminder (18:00 Asia/Jakarta, daily)

You are running as a scheduled Claude Code on the web session. Your job is to
post a daily digest in `#claude-tasks`, surface today's auto-actions for review,
present low-confidence items for the user to confirm/discard via reactions, and
apply yesterday's reactions.

---

## Step 1 — Load state and config

Read:
- `config/identity.yml` — `review_channel_id`, `claude_inbox_list_id`,
  `clickup_workspace_id`.
- `state/processed.jsonl` — for today's auto-actions (entries with `run_at`
  on today's date in Asia/Jakarta).
- `state/pending-review.json` — current low-confidence queue.
- `state/last-digest.json` (may not exist on first run) —
  `{"message_ts": "...", "items": [...]}` from yesterday's digest, so we can
  read its reactions.

## Step 2 — Apply reactions from yesterday's digest

If `state/last-digest.json` exists:

1. Call `mcp__789bdec3-74f9-472f-9121-b2864a62e576__slack_get_reactions` on
   `last-digest.message_ts` in `review_channel_id`.
2. For each item in `last-digest.items`, look at reactions on that item's
   numbered line (we encode the item index in the message — see Step 4).
   Reactions are read from the parent message, not per-line; instead, the
   user reacts to **replies in the thread** numbered `1.`, `2.`, etc.
3. Read the thread of `last-digest.message_ts` via `slack_read_thread`. Each
   reply corresponds to one numbered item. Read reactions on each reply.
4. For each reply with a reaction:
   - ✅ `white_check_mark` → apply the suggested action:
     - CREATE → `clickup_create_task` in `would_be_list_id` with name/description
       built from the item's summary + slack permalink. Append CREATE entry to
       `state/processed.jsonl`.
     - COMPLETE → find the candidate task and `clickup_update_task` to complete,
       plus `clickup_create_task_comment` with the permalink. Append COMPLETE
       entry to `processed.jsonl`.
   - ❌ `x` → discard. Don't touch ClickUp.
   - 📥 `inbox_tray` → `clickup_create_task` in `claude_inbox_list_id` regardless
     of suggested action. Append CREATE entry.
   - No reaction → leave the item in `pending-review.json` for tomorrow.
5. Remove resolved items from `pending-review.json`.

Also scan replies in the thread for any message that starts with `undo ` followed
by a ClickUp task URL or ID. For each such message:
- Look up the task. If it was created today (in `processed.jsonl` with `action:
  CREATE` and `run_at` = today), call `clickup_delete_task` and append an
  `UNDO` entry to `processed.jsonl`.
- If it was a COMPLETE action, revert the task to `open` status via
  `clickup_update_task` and append an `UNDO` entry.

## Step 3 — Build today's digest content

From `processed.jsonl`, filter entries with `run_at` on today's date.
Separate into `created_today`, `completed_today`, and `undone_today`.

From `pending-review.json`, take all items not yet resolved → `pending`.

## Step 4 — Post the digest

Send a single message to `review_channel_id`:

```
:waning_crescent_moon: Evening review — {date}

*Auto-actions today*
• Created: {count} — {clickup task links, comma-separated}
• Completed: {count} — {clickup task links}
• Undone (from your reactions): {count}

If any of the above are wrong, reply with: `undo <clickup-task-link>`

*Needs your call ({pending.length})*
React on the numbered replies below:  :white_check_mark: confirm  ·  :x: discard  ·  :inbox_tray: send to Claude Inbox
```

Then, **in the thread** of that message, post one reply per pending item:

```
{n}. [{suggested_action}] {summary}
    → would go to: {list_name}
    → {permalink}
```

Save `state/last-digest.json` as:
```json
{
  "message_ts": "<parent message ts>",
  "items": [
    {"index": 1, "reply_ts": "<reply ts>", "pending_item_ts": "<thread_ts from pending-review>"},
    ...
  ]
}
```

If `pending` is empty, post the parent message only (with "Needs your call (0)
— nothing to triage 🎉") and skip the thread replies. Still save
`state/last-digest.json` with `items: []` so tomorrow's run doesn't try to
read yesterday's reactions twice.

## Step 5 — Commit and push state

```
git add state/processed.jsonl state/pending-review.json state/last-digest.json
git commit -m "evening reminder $(date -u +%Y-%m-%dT%H:%MZ)"
git push -u origin claude/slack-reader-workflow-t1njI
```

Skip the commit cleanly if there are no changes. Done.
