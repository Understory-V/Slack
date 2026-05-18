# Slack → ClickUp daily routine

Two scheduled Claude Code on the web sessions that watch your Slack channels for
commitments and deliveries, mirror them into ClickUp, and ping you in a private
Slack channel every evening so you can correct anything Claude got wrong.

| When | What |
|---|---|
| **04:00 Asia/Jakarta, Mon–Fri** | Read last 48h of every channel you're in. Auto-create tasks for high-confidence commitments; auto-complete tasks for high-confidence deliveries; queue ambiguous items for evening review. |
| **18:00 Asia/Jakarta, daily** | Post a digest in `#claude-tasks`: today's auto-actions + the ambiguous items with ✅ / ❌ / 📥 reaction prompts. Apply your reactions to ClickUp. |

## Repo layout
```
triggers/morning-review.md     Prompt for the 04:00 session
triggers/evening-reminder.md   Prompt for the 18:00 session
config/channels.yml            Slack channel ID → ClickUp list mapping
config/identity.yml            Your Slack user ID, teammate IDs, ClickUp workspace + Claude Inbox list IDs
state/processed.jsonl          Append-only ledger of (slack ts → clickup task id) for dedup
state/pending-review.json      Low-confidence items waiting for 18:00 review
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

### 4. Schedule the two triggers in Claude Code on the web
On the web app, create two scheduled triggers on this repo + branch:

| Trigger name | Cron (Asia/Jakarta) | Prompt source |
|---|---|---|
| `slack-morning-review` | `0 4 * * 1-5` | contents of `triggers/morning-review.md` |
| `slack-evening-reminder` | `0 18 * * *` | contents of `triggers/evening-reminder.md` |

### 5. Dry-run before going live
Before scheduling the morning trigger, run `triggers/morning-review.md` manually
once with `DRY_RUN=true` set at the top of the prompt. It will log what it
*would* do without touching ClickUp, and post the same summary to
`#claude-tasks` so you can sanity-check.

## Day-to-day use
- **Morning:** glance at the `#claude-tasks` summary on your phone.
- **Evening:** the digest will tag low-confidence items. React:
  - ✅ → confirm and create/complete in ClickUp
  - ❌ → discard
  - 📥 → send to Claude Inbox for later triage
- **Correction:** if the morning run created a bad task, reply in the digest
  thread with `undo <task-link>`. The next 18:00 run will revert it.

## Stopping the workflow
Pause or delete the two scheduled triggers in the Claude Code on the web UI.
The repo and state stay intact, so you can resume any time.
