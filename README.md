# Slack → ClickUp daily routine

Two scheduled Claude Code on the web sessions that watch your Slack channels for
commitments and deliveries, mirror them into ClickUp, and ping you in a private
Slack channel every evening so you can correct anything Claude got wrong.

| When | What |
|---|---|
| **04:00 Asia/Jakarta, Mon–Fri** | Scan last 48h of every channel you're active in, classify candidates, post a **review table** in `#claude-tasks`. Does NOT write to ClickUp. |
| **18:00 Asia/Jakarta, daily** | Light nudge if rows are still pending. Skipped silently if queue is empty. |
| **On-demand** (`slack-push-now`) | Reads your reply commands in the digest thread, pushes approved rows to ClickUp, posts confirmation. Fire it whenever you've replied with `push` / `skip` / `inbox` / `undo`. |
| **On-demand** (`slack-command`) | Reads any plain-text instructions you've posted in `#claude-tasks` and acts on them — create a recurring task, update a channel mapping, undo a task, edit config, ask "what's pending". Marks each handled message with `:claude_done:` so it isn't re-processed. |

## Reply-command syntax

In the morning digest thread, reply with any combination of:

- `push 1 3 5` — push those rows to ClickUp using their mapped list
- `push all` — push every pending row
- `push all high` / `push all low` — confidence filter
- `inbox 4` — push row 4 but force list = **Claude Inbox**
- `skip 2 6` — drop those rows from the queue
- `skip all low` — discard everything low-confidence
- `undo <task-link>` — revert a previously-pushed task (deletes a CREATE, reopens a COMPLETE)

Multiple replies are unioned; later commands override earlier ones for the same row. Then fire `slack-push-now` to apply.

Unactioned rows roll into tomorrow's digest with a fresh row index. After **7 days** with no command, they expire silently.

## Repo layout
```
triggers/morning-review.md     Prompt for the 04:00 scheduled session (table + queue)
triggers/push-now.md           Prompt for the on-demand push trigger
triggers/slack-command.md      Prompt for the on-demand "talk to your automation" trigger
triggers/evening-reminder.md   Prompt for the 18:00 scheduled nudge
config/channels.yml            Slack channel ID → ClickUp list mapping
config/identity.yml            Your Slack user ID, teammate IDs, ClickUp workspace + Claude Inbox list IDs
state/processed.jsonl          Append-only ledger of (slack ts → clickup task id) for dedup + audit
state/pending-review.json      All rows awaiting your push / skip command
state/last-digest.json         Today's digest message ts + row→pending_id map, used by push-now
```

State files live in git on branch `claude/slack-reader-workflow-t1njI` because the
container is ephemeral — each session commits state at the end so the next one
sees it.

## One-time setup

### 1. Create the review channel in Slack
- Create private channel **`#claude-tasks`** with just you as the human member.
- Add the Claude app to the channel (`/invite @Claude` or via channel settings).
- Copy the channel ID (Channel details → bottom of the About tab) into
  `config/identity.yml` → `review_channel_id`.

### 2. Create a "Claude Inbox" list in ClickUp
- Create a list called **"Claude Inbox"** anywhere in your workspace. This is
  the fallback when a Slack channel hasn't been mapped to a specific list yet.

### 3. Bootstrap session (run once, manually)
Open a Claude Code on the web session against this repo and paste:

```
Run the bootstrap routine:

1. Call slack_search_users for my handle (ask me for it if you don't have it) and
   record my Slack user ID. Ask me for the user IDs of my internal teammates
   (people on "my side" whose "yes" should count as a commitment).
2. Call clickup_get_workspace_hierarchy. Find the "Claude Inbox" list and any
   other lists I want auto-routed. Ask me which Slack channels should map to
   which ClickUp lists.
3. Write all of this into config/identity.yml and config/channels.yml, commit,
   and push to claude/slack-reader-workflow-t1njI.
```

### 4. Create the three triggers in Claude Code on the web
On the web app, create three triggers on this repo + branch:

| Trigger name | Schedule | Prompt source |
|---|---|---|
| `slack-morning-review` | Cron `0 4 * * 1-5` (Asia/Jakarta) | `triggers/morning-review.md` |
| `slack-push-now` | **On-demand** (no cron) | `triggers/push-now.md` |
| `slack-evening-reminder` | Cron `0 18 * * *` (Asia/Jakarta) | `triggers/evening-reminder.md` |
| `slack-command` | **On-demand** (no cron) | `triggers/slack-command.md` |

### 5. First-run check
Fire `slack-morning-review` manually once and confirm it posts a table in `#claude-tasks` and **does not** write to ClickUp. Reply with a test command (e.g. `skip 1`) and fire `slack-push-now` to confirm the round-trip works. Schedule the crons after that.

## Day-to-day use
- **Morning:** glance at the digest table in `#claude-tasks` on your phone. Reply with `push 1 3` / `skip 2` / `inbox 4` whenever.
- **Push:** fire `slack-push-now` and Claude does the ClickUp writes + posts a confirmation back in the thread.
- **Ad-hoc requests:** write plain-text instructions in `#claude-tasks` (e.g. *"create a recurring Wednesday standup task for Revic"*) and fire `slack-command`. Claude reads the channel, acts, and reacts `:claude_done:` on the message.
- **Evening nudge:** if you forgot to push, the 18:00 trigger sends a short reminder.
- **Correction:** to revert a previously-pushed task, reply `undo <task-link>` and re-fire `slack-push-now`.

### A note on `@Claude` mentions
If you have Anthropic's official Slack app installed in your workspace, mentioning `@Claude` will trigger a generic chat reply from that app — it has **no access** to this repo, your ClickUp, or the automation state. The right way to give the automation instructions is to post plain text in `#claude-tasks` and fire `slack-command`. No mention required.

### Recurring tasks
The ClickUp MCP tool doesn't expose ClickUp's native recurrence field. When you ask for a recurring task, Claude will create it once, embed the recurrence rule in the task description, tag it `recurring:pending-setup`, and remind you to flip the "Repeat" toggle in ClickUp's right sidebar once. After that, ClickUp handles all future occurrences natively.

## Stopping the workflow
Pause or delete the four triggers in the Claude Code on the web UI. The repo and state stay intact, so you can resume any time.
