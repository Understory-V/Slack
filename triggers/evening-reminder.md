# Evening reminder (18:00 Asia/Jakarta, daily)

Lightweight nudge so you don't forget pending rows. No ClickUp writes; no
state changes.

---

## Step 1 — Load state

Read:
- `config/identity.yml` — `review_channel_id`.
- `state/pending-review.json`.
- `state/last-digest.json` (for the thread ts to nudge inside).

## Step 2 — Decide whether to nudge

If `pending-review.json` is empty (length 0), **do nothing**. Don't post.
End the run cleanly. The whole point of this trigger is to nudge — silence
when there's nothing to do is correct.

## Step 3 — Post the nudge

If `last-digest.json` exists from today, post **as a thread reply** under
`last-digest.message_ts`. Otherwise, post a new top-level message.

```
:waning_crescent_moon: Evening nudge — {N} rows still pending.

{high_count} 🟢 high · {low_count} 🟡 low

Reply in this thread with `push 1 3` / `skip 2` / `inbox 5`, then fire the
`slack-push-now` trigger.
```

Skip listing the rows themselves — the morning digest table is right above
this in the same thread. Keep the nudge short.

## Step 4 — End

No commit. No state writes. Done.
