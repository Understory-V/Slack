# Morning review (04:00 Asia/Jakarta, Mon–Fri)

Scheduled session that scans the last 48 hours of every Slack channel you're
active in, classifies candidate tasks with a confidence tag, and posts a single
approval table in `#claude-tasks`. **It does not write to ClickUp.** Pushes
happen later via the `slack-push-now` trigger after you reply with commands.

**Config knobs (edit the values, not the keys):**
- `LOOKBACK_HOURS = 48`
- `EXPIRY_DAYS = 7`   ← pending rows older than this drop out silently

---

## Step 1 — Load state and config

Read from the repo:
- `config/identity.yml` — `my_slack_user_id`, `internal_teammate_ids`,
  `clickup_workspace_id`, `claude_inbox_list_id`, `review_channel_id`.
- `config/channels.yml` — `mappings:` (Slack channel ID → ClickUp list ID).
- `state/processed.jsonl` — build a dedup set of `thread_ts` values from
  entries in the last 14 days (any `action`, including `expired`).
- `state/pending-review.json` — carryover items from prior days.

If any config value is missing, post a setup-needed message in
`review_channel_id` and stop.

## Step 2 — Enumerate active channels

To avoid scanning every channel in the workspace, find channels with recent
activity involving you:

1. `slack_search_public_and_private` with
   `query = "from:<@{my_slack_user_id}> after:{date-2-days-ago}"`,
   `channel_types = "public_channel,private_channel"`, `limit = 20`,
   `include_context = false`, `sort = "timestamp"`.
2. Run a second search with `query = "to:<@{my_slack_user_id}> after:{date}"`
   in case you were tagged but didn't reply.
3. Take the unique channel names from both result sets. Drop `im` and `mpim`.
4. Resolve each name to a channel ID via `slack_search_channels`.

## Step 3 — Pull last 48h per channel

For each channel:
1. `slack_read_channel` with `limit = 50`.
2. Filter the returned messages to those with `ts >= now - LOOKBACK_HOURS`.
3. For any message with `reply_count > 0`, call `slack_read_thread` to get
   the full thread.
4. Skip threads whose `root.ts` is in the Step 1 dedup set OR already a
   `thread_ts` in `state/pending-review.json` (carryover handled separately
   in Step 6).
5. The unit of analysis is the **thread** (root + replies).

## Step 4 — Classify each thread

Decide one of CREATE, COMPLETE, LOW-CONF, or DROP using this rubric.

### HIGH-CONF CREATE
All of:
- A party makes a **concrete ask** with a specific deliverable.
- Someone *else* in the thread replies with a **clear affirmative**: "yes",
  "will do", "on it", "sending now", "by Friday", an explicit date, or
  "done by EOD". The affirmer ≠ asker.
- Affirmer scope: if `internal_teammate_ids = "*"`, any user other than the
  asker counts. Otherwise the affirmer must be `my_slack_user_id` or in the
  `internal_teammate_ids` list.
- The ask is **not fully fulfilled** within the same thread inside the
  48h window (skip ask + same-window-delivery cycles — those drop).

Extract: title (imperative, ≤80 chars), due_date (ISO or null), requester,
assignee_hint (who said yes), permalink (thread root).

### HIGH-CONF COMPLETE
All of:
- An open ClickUp task exists matching this thread. Look it up via
  `clickup_filter_tasks` filtered by tag `source:slack` scoped to the
  mapped list (or Claude Inbox). Match by slack permalink in description,
  or close title match.
- The latest message from the *affirmer side* contains a delivery signal:
  "sent", "done", "shipped", "deployed", "uploaded", "here you go", or a
  file/URL attachment.

Extract: `task_id`, `delivery_permalink`.

### LOW-CONF (still surfaced for review)
- Conditional ("maybe", "if we can", "let me check").
- Ask without an explicit yes.
- Yes without a clear ask.
- Reverse-direction (Vilca asking external party — track as a blocker).
- Internal admin chatter when there's *some* commitment signal.
- Ambiguous match against existing tasks.

Extract: `summary` (one line), `suggested_action` (CREATE | COMPLETE),
`would_be_list_id`, permalink.

### DROP (silently)
- Purely social / off-topic.
- Bot notifications (Instantly.ai lead alerts, etc.).
- Channel joins, file shares with no commitment language.
- Ask + delivery fully resolved in-thread within the 48h window (no open work).

## Step 5 — Resolve channel → ClickUp list

For each surfaced row:
- If `channel_id` is in `config/channels.yml` `mappings:`, use that list ID.
- Otherwise fall back to `claude_inbox_list_id`.

## Step 6 — Build the new pending queue

Start with `state/pending-review.json`. Process:
1. **Expire** any item with `queued_at` older than `EXPIRY_DAYS` days. For
   each, append `{ts, channel_id, action: "expired", confidence, run_at}` to
   `processed.jsonl`.
2. **Carryover** — keep the rest. They'll get new row indices below.
3. **Append** all new HIGH-CONF and LOW-CONF rows from Step 4.
4. **Re-index** the entire list. `row_index` starts at 1 in display order:
   high-confidence first, then low-confidence; within each group sort by
   channel name then queued_at ascending.

Each pending item record:
```json
{
  "row_index": 1,
  "thread_ts": "...",
  "channel_id": "...",
  "channel_name": "...",
  "permalink": "...",
  "confidence": "high" | "low",
  "action": "CREATE" | "COMPLETE",
  "summary": "...",
  "title": "...",                    // for CREATE
  "due_date": "YYYY-MM-DD" | null,
  "would_be_list_id": "...",
  "would_be_list_name": "...",
  "match_task_id": "..." | null,     // for COMPLETE
  "delivery_permalink": "..." | null,
  "queued_at": "<iso8601>",
  "first_seen_run": "<iso8601>"      // unchanged on carryover
}
```

## Step 7 — Post the digest table

Send a single message to `review_channel_id` with this structure:

```
:sunrise: **Morning digest — {date}** ({total} rows pending)
{carryover_count} carried over · {new_count} new · {expired_count} expired

| # | Conf | Channel | Action | Summary | List | Slack |
|---|------|---------|--------|---------|------|-------|
| 1 | 🟢 high | #revic | CREATE | Launch ZoomInfo takeout (16,680 cos), due Wed | Claude Inbox | [open]({permalink}) |
| 2 | 🟡 low | #revic | CREATE | Martech copies pending Morgan approval | Claude Inbox | [open]({permalink}) |
…

**To act:** reply in this thread with commands. Then fire the
`slack-push-now` trigger.

Commands:
• `push 1 3 5` — push those rows to ClickUp
• `push all high` / `push all low` — confidence filter
• `inbox 4` — push but force to Claude Inbox
• `skip 2 6` — drop those rows
• `undo <task-link>` — revert a previously-pushed task
```

**That's it — no threaded replies.** The table summary plus the inline slack link
is enough context. The user wants to scan the table and reply directly in the
thread with commands like `push 1 3 5` or `skip 2`. Don't pollute the thread
with one-paragraph-per-row context replies.

If a row's summary in the table can't fit the key context in ≤80 chars, prefer
truncating with `…` over splitting into a threaded reply.

## Step 8 — Persist state

Write `state/last-digest.json`:
```json
{
  "message_ts": "<parent message ts>",
  "channel_id": "<review_channel_id>",
  "posted_at": "<iso8601>",
  "rows": [
    {"row_index": 1, "thread_ts": "<slack source thread ts>"},
    ...
  ]
}
```
(No per-row `reply_ts` — there are no automatic threaded replies. The
push-now trigger reads user replies under `message_ts` and parses
`push 1 3 5` etc. against `row_index`.)

Write the rebuilt `state/pending-review.json` (full carryover + new rows).

## Step 9 — Commit and push

```
git add state/processed.jsonl state/pending-review.json state/last-digest.json
git commit -m "morning digest $(date -u +%Y-%m-%dT%H:%MZ)"
git push -u origin claude/slack-reader-workflow-t1njI
```

If nothing changed (no new candidates, no expirations, no carryover), still
post the digest but with "0 rows pending" and skip the commit. Done.
