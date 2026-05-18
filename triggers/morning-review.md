# Morning review (04:00 Asia/Jakarta, Mon–Fri)

You are running as a scheduled Claude Code on the web session. Your job is to
scan the last 48 hours of every Slack channel the user is a member of, mirror
high-confidence commitments and deliveries into ClickUp, and queue anything
ambiguous for the 18:00 review.

**Config knobs (edit the values, not the keys):**
- `LOOKBACK_HOURS = 48`
- `DRY_RUN = false`  ← set to `true` for a no-write dry run

---

## Step 1 — Load state and config

Read these files from the repo:
- `config/identity.yml` — `my_slack_user_id`, `internal_teammate_ids`,
  `clickup_workspace_id`, `claude_inbox_list_id`, `review_channel_id`.
- `config/channels.yml` — `mappings:` (Slack channel ID → ClickUp list ID).
- `state/processed.jsonl` — append-only ledger; build a set of already-handled
  `thread_ts` values from entries in the last 14 days. Ignore older entries.
- `state/pending-review.json` — current low-confidence queue.

If any of the config files are missing the bootstrap values, stop and post a
message in `review_channel_id` asking the user to run the bootstrap routine
(see `README.md`).

## Step 2 — Enumerate channels

Call `mcp__789bdec3-74f9-472f-9121-b2864a62e576__slack_search_channels` to list
all channels the user is a member of. Keep entries where the channel type is
`public_channel` or `private_channel`. **Drop** `im` and `mpim` — no DMs.

## Step 3 — Pull last 48 hours per channel

For each channel:
1. Compute `oldest_ts = now − LOOKBACK_HOURS hours` as a Slack ts string.
2. Call `slack_read_channel` with `oldest = oldest_ts`.
3. For any message with `reply_count > 0`, call `slack_read_thread` to get the
   full thread. Treat the **thread (root + replies)** as the unit of analysis,
   not individual messages.
4. Skip threads whose `root.ts` is in the processed-set from Step 1.

Build a list of `{channel_id, channel_name, root_ts, permalink, messages[]}`
records.

## Step 4 — Classify each thread

For each thread, decide one of CREATE, COMPLETE, or LOW-CONF using this rubric.

### CREATE (high-confidence)
All of:
- A party makes a **concrete ask** with a specific deliverable
  ("can you send the proposal?", "please review the doc", "we need the deck").
- Someone else in the thread replies with a **clear affirmative**: "yes",
  "will do", "on it", "sending now", "by Friday", an explicit date, or
  "done by EOD".
- The affirmative is **after** the ask in the thread, and the affirmer is
  **not the same user** as the asker.
- Affirmer scope: if `internal_teammate_ids` is `"*"`, any user other than the
  asker counts. Otherwise, the affirmer must be `my_slack_user_id` or in the
  `internal_teammate_ids` list.

Extract:
- `title`: imperative phrasing of the ask, max 80 chars.
- `due_date`: ISO date if mentioned, else null.
- `requester`: display name of the asker.
- `assignee_hint`: who on your side said yes (for the description).
- `permalink`: thread root permalink from Slack.

### COMPLETE (high-confidence)
All of:
- Look up open ClickUp tasks in the channel's mapped list (or Claude Inbox)
  via `clickup_filter_tasks` filtered by tag `source:slack`. Find one whose
  description contains a matching slack permalink OR whose title closely
  matches the original ask in this thread.
- The latest message from the *affirmer* (same `internal_teammate_ids`
  scope as CREATE) contains a **delivery signal**: "sent", "done",
  "shipped", "deployed", "uploaded", "here you go", or includes a
  file/URL attachment that plausibly satisfies the ask.

Extract:
- `task_id`: the matching ClickUp task ID.
- `delivery_permalink`: permalink of the delivery message.

### LOW-CONF (everything else that *might* be actionable)
- Conditional language: "maybe", "if we can", "let me check", "I'll see".
- Ask without a clear yes from your side.
- Yes without a clear ask (e.g. internal teammates chatting).
- Ambiguous match against existing tasks.

Extract:
- `summary`: one-line description of what was discussed.
- `suggested_action`: CREATE or COMPLETE.
- `would_be_list`: which ClickUp list it'd go to.
- `permalink`.

Purely social / off-topic threads with no ask and no delivery are **dropped**
silently (don't queue them).

## Step 5 — Resolve channel → ClickUp list

For each CREATE/LOW-CONF item:
- If `channel_id` is in `config/channels.yml` `mappings:`, use that list ID.
- Otherwise, fall back to `claude_inbox_list_id` from `config/identity.yml`.

## Step 6 — Act on high-confidence

**Skip this entire step if `DRY_RUN = true`.** Instead, log intended actions
to a `dry_run_report` array.

For each CREATE:
- `clickup_create_task` with:
  - `name` = title
  - `description` = ask + `\n\nFrom: {requester} in #{channel_name}\nThread: {permalink}\nAffirmed by: {assignee_hint}`
  - `due_date` if present
  - `tags` = `["source:slack", "auto:claude"]`
- Append to `state/processed.jsonl`:
  `{"ts": "<root_ts>", "channel": "<channel_id>", "action": "CREATE", "task_id": "<id>", "confidence": "high", "run_at": "<iso8601>"}`

For each COMPLETE:
- `clickup_update_task` on `task_id` setting status to `complete`
  (use whatever "complete" status the user's workspace uses; if unknown, the
  built-in `closed` / `complete` value).
- `clickup_create_task_comment` on `task_id` with:
  `"Delivered: {delivery_permalink}"`
- Append to `state/processed.jsonl`:
  `{"ts": "<root_ts>", "channel": "<channel_id>", "action": "COMPLETE", "task_id": "<id>", "confidence": "high", "run_at": "<iso8601>"}`

If any ClickUp call fails: log the error to the morning summary, do **not**
append to `processed.jsonl` (so the next run retries), and continue.

## Step 7 — Update pending-review

Replace `state/pending-review.json` with the full current low-confidence queue:
- Start from the existing pending items (skip any whose `thread_ts` you handled
  in Step 6 — they've been resolved).
- Add all new LOW-CONF items from this run, each as:
  ```json
  {
    "thread_ts": "...",
    "channel_id": "...",
    "channel_name": "...",
    "permalink": "...",
    "summary": "...",
    "suggested_action": "CREATE" | "COMPLETE",
    "would_be_list_id": "...",
    "queued_at": "<iso8601>"
  }
  ```
- Drop items older than 7 days (stale).

## Step 8 — Post morning summary

Send a single message to `review_channel_id` via `slack_send_message` with:

```
:sunrise: Morning review — {date}
• Created: {N} (links: ...)
• Completed: {M} (links: ...)
• Queued for evening review: {K}
• Errors: {E}  (if > 0, include details)

{If DRY_RUN: prepend "🧪 DRY RUN — no ClickUp writes performed."}
```

Keep the links to ClickUp tasks inline so you can tap them on mobile.

## Step 9 — Commit and push state

Run (via Bash):
```
git add state/processed.jsonl state/pending-review.json
git commit -m "morning review run $(date -u +%Y-%m-%dT%H:%MZ)"
git push -u origin claude/slack-reader-workflow-t1njI
```

If there's nothing to commit (no changes), skip the commit but still end the
run cleanly. Done.
