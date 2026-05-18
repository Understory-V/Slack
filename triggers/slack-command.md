# Slack command (on-demand)

Fired manually whenever you've written an instruction in `#claude-tasks` for
Claude to act on. Reads unprocessed messages in the channel, interprets them,
executes, and reacts `:claude_done:` so the same message isn't re-processed.

This is the "talk to your automation" surface. Examples of things it handles:
- *"Create a recurring task every Wednesday for the Revic standup"*
- *"Map `#ext-understory-growthx` to the GrowthX GTMe Tasks list"*
- *"Delete task 868jnwe67"* / *"Reopen it"*
- *"What's pending?"* / *"Show me high-confidence rows"*
- *"Push row 1 and 3"* (delegates to push-now logic)
- *"Add Andre to my internal_teammate_ids"*

Note: the Anthropic Slack `@Claude` app (if installed in your workspace) will
*also* respond to `@Claude` mentions with a generic chat reply. **Ignore that.**
This trigger is your automation surface — it reads any plain-text message
(no @-mention needed) and only processes messages that haven't been reacted
with `:claude_done:` yet.

---

## Step 1 — Load state and config

Read:
- `config/identity.yml` — `my_slack_user_id`, `review_channel_id`,
  `clickup_workspace_id`, `claude_inbox_list_id`.
- `config/channels.yml` — current mappings.
- `state/pending-review.json`, `state/last-digest.json`,
  `state/processed.jsonl`.

## Step 2 — Fetch unprocessed channel messages

1. `slack_read_channel` on `review_channel_id`, `limit = 30`.
2. Keep messages where:
   - Author is `my_slack_user_id` (skip your own bot's posts and Anthropic's
     `@Claude` app replies).
   - Message is not threaded under the morning digest (those are handled by
     `push-now`). To detect: skip if `thread_ts == last-digest.message_ts`
     AND the message has a recognized push-now keyword (`push`, `skip`,
     `inbox`, `undo`).
3. For each candidate message, call `slack_get_reactions`. Skip if the
   `:claude_done:` reaction is already present (from a previous run).

Result: an ordered list of *unprocessed instructions*, oldest first.

## Step 3 — Interpret and execute each instruction

For each unprocessed message, decide the intent and act. Common intents and
how to execute them are below. Be conservative: if intent is unclear, post a
clarifying question in the thread and **do not** react `:claude_done:` (so it
shows up again next run after you reply).

### Intent: create a one-off ClickUp task
Parse: target list, name, due date, assignees, priority.
- Resolve target list: explicit name → `clickup_get_list` to find ID; otherwise
  use the channel mapping or Claude Inbox.
- `clickup_create_task` with `tags: ["source:slack", "auto:claude", "via:command"]`.
- Reply in thread with the task link.

### Intent: create a recurring task
ClickUp's MCP `clickup_create_task` does NOT expose the recurrence field
directly. The workflow:
1. Create the task once via `clickup_create_task`. In the markdown description,
   prepend a clearly-marked block:
   ```
   > **Recurring:** every Wednesday at 10:00 Asia/Jakarta
   > (Toggle the "Repeat" setting in ClickUp's right sidebar to activate the
   > recurrence — Claude can't set it via the API.)
   ```
2. Tag the task with `recurring:pending-setup` so you can find ones still
   awaiting the toggle.
3. Reply in the digest thread with the task link AND a one-line reminder:
   *"Open the task once and click Repeat → {parsed schedule}. After that
   ClickUp handles all future occurrences."*

(If a future ClickUp MCP version exposes recurrence, switch this branch to
set it directly.)

### Intent: update channel → list mapping
Parse: channel name, target list name (or "create new list <name> in <space>").
- If creating a new list: `clickup_create_list` (or `clickup_create_list_in_folder`)
  with the parsed space/folder. Capture the new `list_id`.
- Look up the Slack channel ID via `slack_search_channels`.
- Edit `config/channels.yml`:
  ```yaml
  mappings:
    <channel_id>:
      list_id: "<list_id>"
      list_name: "<list_name>"
  ```
- Reply with confirmation.

### Intent: delete / undo a ClickUp task
- Parse task ID or URL. If a URL, strip to the ID.
- Look up in `processed.jsonl` to confirm Claude created it. **Refuse** if
  it wasn't Claude-created — reply *"I can only undo tasks I created. Delete
  it manually in ClickUp."*
- If a CREATE: `clickup_delete_task`.
- If a COMPLETE: `clickup_update_task` status → open + add a comment
  *"Reopened via slack-command."*
- Append `UNDO` entry to `processed.jsonl`.

### Intent: status query (what's pending, show high-conf, etc.)
- Read `pending-review.json` (filter by confidence if requested).
- Reply with a short markdown summary in the thread. Don't post a new
  parent message — quote-reply to the instruction.

### Intent: edit config
- *"Add `Uxxxxx` to my internal_teammate_ids"* → edit `config/identity.yml`.
- *"Change the review channel to `Cxxx`"* → edit `config/identity.yml`.
- *"Set lookback to 72h"* → edit `triggers/morning-review.md`'s
  `LOOKBACK_HOURS` constant.
- Reply confirming the change and the file modified.

### Intent: delegate to push-now
If the instruction is a clean push-now command (`push N`, `skip N`, etc.) but
posted as a top-level message rather than a digest reply, re-post it as a
reply to `last-digest.message_ts` and react `:claude_done:` so the user can
fire `slack-push-now` next.

### Intent: unclear / multi-step / risky
Post a clarifying reply in the thread quoting back what you understood:
*"You asked me to {paraphrase}. Confirm by reacting :white_check_mark: or
restate."* Do NOT react `:claude_done:` until they confirm — keep it in the
queue.

## Step 4 — Mark processed

For each successfully-handled message, add the `:claude_done:` reaction via
the Slack reaction tool. This is the *only* signal preventing re-processing
next run.

If a message failed (ClickUp API error, etc.), post the error in a reply
and leave the reaction off so the next run retries.

## Step 5 — Commit and push

If `config/*` or `triggers/*` were edited, or `state/processed.jsonl` /
`pending-review.json` changed:

```
git add config/ triggers/ state/
git commit -m "slack-command run $(date -u +%Y-%m-%dT%H:%MZ): {one-line summary}"
git push -u origin claude/slack-reader-workflow-t1njI
```

If nothing changed, no commit. Done.

---

## Safety rules
- Never act on a message you can't fully interpret — clarify first.
- Never touch a ClickUp task Claude didn't create (no deleting other people's
  work, no reopening tasks from outside this workflow).
- Never push secrets / credentials / API keys into the repo, even if
  instructed.
- Never broaden config (e.g. adding a teammate, expanding the scope) without
  echoing the change back and waiting for confirmation on first-time
  expansions. Add a `:warning:` to the reply if the change widens what gets
  auto-tracked.
