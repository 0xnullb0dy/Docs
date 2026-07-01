# Scaling Guide

This bot has a small number of genuinely separate scaling axes. Most "will this scale?" worry gets pointed at the wrong one, so this doc separates them out with concrete numbers, not just "should be fine."

## The four axes, and which ones your current setup actually stresses

|Axis|What it affects|Your situation|
|---|---|---|
|Members in _one_ guild|Reaction event volume, role-add API calls|Large — this is your real load|
|Number of guilds the bot is in|Sharding requirement, gateway connections|One guild — not a factor for you|
|Streamers tracked|Twitch API call volume|~10 — trivial|
|Reaction panels / bindings|In-memory cache size, DB row count|Small — a handful of messages|

Your stated scale (one big guild, ~10 streamers) only really stresses the **first row**. The other three have headroom you're nowhere near using. The rest of this doc goes through each component and says, plainly, the point at which it would actually become a problem.

---

## Reaction roles

**Current design:** raw gateway events checked against an in-memory dict; a database write only happens when _you_ run `/reactionrole bind`/`unbind`, never on a member reacting. Member reactions never touch SQLite at all.

**What actually has a ceiling here:** not your code — Discord's own role-update rate limit. Each `add_roles`/`remove_roles` call is one HTTP request. Discord enforces both a global limit (50 requests/second per bot) and per-route limits. `discord.py`'s HTTP client queues requests against these automatically — calls don't fail under burst, they just queue and drain.

**Concretely:** if a giveaway-style event causes 5,000 people to react within a few seconds, that's 5,000 queued role-add calls. They'll all eventually succeed, but the _last_ person to react might get their role tens of seconds to a couple of minutes after reacting, not instantly. That's a Discord platform ceiling, not something architecture changes on your end can remove — the fix, if you ever hit it, is operational (e.g. stagger a mass-ping instead of dropping it all at once), not code.

**When to revisit:** only if you start seeing complaints about role-grant delay during specific high-traffic moments (raids, giveaways). Day-to-day reacting on a large-but-normal server won't come close to this.

---

## Twitch notifications

**Current design:** one batched Helix API call per poll interval (default 60s), covering all tracked streamers in that call (up to 100 per request).

**Numbers:** Twitch's app-token rate limit is 800 points/minute, and a `/streams` call costs 1 point regardless of how many `user_login` filters you pack into it. At 10 streamers in one call every 60 seconds, you're using roughly 1 point/minute — about 0.1% of the limit.

**When polling stops being the right tool:**

- Tracking 50–100+ streamers _and_ wanting near-real-time alerts (polling latency = your poll interval, on average half of it).
- Running this as a multi-tenant bot across many servers where the combined unique streamer list grows into the hundreds.

At that point, switch to **Twitch EventSub** (webhook-push instead of polling) — Twitch notifies you the moment a stream starts, no polling loop at all. The trade-off is needing a public HTTPS endpoint with a valid cert, which is real infrastructure to stand up and keep running. For 10 streamers on one server, that trade isn't worth making — polling is simpler, has zero moving parts to keep alive, and isn't anywhere near its limit.

---

## Database (SQLite)

**Why it's not the bottleneck people assume it is:** this bot's data size tracks _configuration_ (panels, bindings, tracked streamers), not member count. A 50,000-member guild with 10 streamers and 3 panels has a database roughly the same size as a 500-member guild with the same setup — a few hundred rows, total.

**Write frequency in practice:** a handful of writes per hour (streamer state changes when someone goes live/offline) plus whatever you write when running setup commands. Reaction adds/removes never write to the DB. This is nowhere near write-contention territory for SQLite in WAL mode, which handles far higher write rates than this bot will ever generate for a single guild.

**When to actually migrate to Postgres:**

- Running multiple bot _processes_ against the same dataset (horizontal scaling — e.g. you split features into separate services).
- Operating as a multi-tenant bot across many large guilds from one codebase, where you want concurrent writers and remote/managed backups.

If that day comes, the migration is contained: `database.py` is the only file with SQL in it. Every other file calls methods like `db.add_tracked_streamer(...)` — swap the implementation behind those signatures and nothing else changes.

---

## Sharding

Discord requires sharding once a bot is in **2,500+ guilds total** — this is about _guild count_, not members in any single guild. A single very large guild does not need sharding; `discord.py` won't even suggest it until you cross that guild-count threshold. Since this bot lives in one server, this isn't something to plan for unless you eventually open the bot up publicly to other servers.

---

## Quick reference: what would make me revisit this design

|Symptom|Likely cause|Fix|
|---|---|---|
|Role grants visibly lag during raids/giveaways|Discord's per-bot rate limit, not the bot|Operational (stagger events), not code|
|Notifications feel "late" after a streamer goes live|Poll interval|Lower `TWITCH_POLL_INTERVAL`, or move to EventSub if tracking many streamers|
|Bot is added to hundreds/thousands of other servers|Guild count, not member count|Enable sharding (`discord.py` handles this; ask and I'll wire it up)|
|Running more than one bot process against the same data|Concurrent writers|Migrate `database.py` to Postgres|

None of these apply at your current scale. This table is here so that when something _does_ change, you know which lever to actually pull instead of guessing.