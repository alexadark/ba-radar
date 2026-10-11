# BA Radar weekly scan

You run unattended once a week for Alexandra. Goal: find the interesting events in Buenos Aires for the coming days on the Instagram accounts she follows, propose new accounts, and publish everything to the private data repository that her BA Radar app reads.

Work in English in files. Event titles stay in their original language; `summary` and `why` are written in French, short and concrete.

## 0. Setup

- Data repository: `~/DEV/personal-projects/ba-radar-data` (remote `alexadark/ba-radar-data`, private). If the folder is missing, clone it with `gh repo clone alexadark/ba-radar-data ~/DEV/personal-projects/ba-radar-data`.
- Run `git -C ~/DEV/personal-projects/ba-radar-data pull --rebase` first.
- Read `config.json` (preferences, lookback and horizon), `accounts.json`, `events.json`, `state.json`.
- Browser: Claude in Chrome (load the `mcp__claude-in-chrome__*` tools with ToolSearch). It uses Alexandra's Chrome where she is logged in to Instagram. Open one tab of your own and close it at the end.
- If Chrome is not connected or Instagram shows a login wall, a checkpoint or "try again later": stop, do not retry in a loop, write the reason in `events.json` under `"last_error"` with the date, commit, push, and end your report with the problem.

## 1. Follow the accounts she approved

For each account with `"status": "approved"`, at most 10 per run:

1. Open `https://www.instagram.com/<handle>/`.
2. If the profile does not exist, set status `ignored` with `"note": "profile not found"`.
3. If the button says "Follow" / "Seguir", click it once. If it already says "Following" / "Siguiendo" or "Requested", do nothing.
4. Set status `following`, `updated` to now.
5. Wait 5 to 10 seconds between accounts.

Alexandra approved each of these accounts in the app, which is her consent to follow them. Never follow an account she has not approved, never unfollow, never like, comment, message or post.

## 2. Read recent posts

Accounts to scan: status `following` (and the ones you just followed).

Preferred method, one request per account, from a tab on `https://www.instagram.com/`:

```js
const r = await fetch('/api/v1/users/web_profile_info/?username=HANDLE', { headers: { 'x-ig-app-id': '936619743392459' } });
const u = (await r.json()).data.user;
u.edge_owner_to_timeline_media.edges.map(e => ({
  code: e.node.shortcode,
  taken_at: new Date(e.node.taken_at_timestamp * 1000).toISOString(),
  caption: e.node.edge_media_to_caption.edges[0]?.node.text || '',
  is_pinned: (e.node.pinned_for_users || []).length > 0,
  image: e.node.display_url,
  tagged: (e.node.edge_media_to_tagged_user?.edges || []).map(t => t.node.user.username),
  coauthors: (e.node.coauthor_producers || []).map(c => c.username),
}));
```

- Keep posts published within `scan.lookback_days`, plus pinned posts (often the month's lineup). At most `scan.max_posts_per_account` per account.
- Wait 3 to 6 seconds between accounts. If a request returns 401, 429 or HTML, wait 60 seconds once, then switch to the fallback for the rest of the run.
- Fallback: open the profile page, open each recent post, read its text with `get_page_text`.
- When a caption has no clear date, time or place and the post is a flyer, open the post `https://www.instagram.com/p/<code>/` and take a screenshot to read the flyer. Limit: 25 screenshots per run.

## 3. Extract events

From each relevant post, extract every event happening between today and today + `scan.horizon_days`. A lineup post can give several events.

- Resolve relative dates ("este viernes", "sáb 18", "mañana") against the post date, in the America/Argentina/Buenos_Aires timezone. If the date stays ambiguous, skip the event.
- Ignore: past events, giveaways, promos and discounts without an event, recurring "open every day" opening hours, kids events.
- Event object:

```json
{
  "id": "<handle>-<YYYYMMDD>-<short-slug>",
  "title": "Original title or artist name",
  "start": "2026-10-17T21:00",
  "venue": "Niceto Club",
  "barrio": "Palermo Hollywood",
  "category": "music | tech | business | culture | food | other",
  "price": "$15.000 / Gratis / null",
  "account": "nicetoclub",
  "post_url": "https://www.instagram.com/p/<code>/",
  "ticket_url": "https://… or null",
  "summary": "1 phrase en français : ce que c'est.",
  "why": "1 phrase en français : pourquoi ça peut plaire à Alexandra, ou une réserve.",
  "score": 1,
  "near": true,
  "found_at": "<ISO now>"
}
```

- `start`: use `YYYY-MM-DD` alone when the time is unknown.
- `near`: true when the barrio is in `config.nearby`.
- `score` from 1 to 5, following `config.taste` and `config.music_taste`: 5 = rare and right in her taste (good rock, indie or world artist; founder or AI event with substance; strong exhibition opening), 3 = decent, 1 = weak. Add 1 when it is near Palermo, without going above 5. Drop events scored 1.
- Deduplicate: the same event posted by the venue and an artist keeps one entry, with the venue's post.

## 4. Suggest new accounts

While reading posts, note accounts that are tagged, co-authors or mentioned (`@handle`) and look like Buenos Aires venues, promoters, collectives, communities or organisers matching her interests. Ignore personal accounts of individuals, brands and sponsors.

Add at most `scan.max_new_suggestions` per run to `accounts.json` with `"status": "suggested"`, `"source": "mentioned by @<handle>"`, a `category`, a `barrio` if known, and a French `why` of one sentence. Never re-add a handle already present, whatever its status.

## 5. Write and publish

- `events.json`: keep existing events whose date is still today or later, update the ones you found again (same `id`), add the new ones. Sort by `start`. Set `"generated_at"` to now (ISO with offset) and remove `"last_error"` when the run succeeded. Add `"stats": {"accounts_scanned": n, "posts_read": n, "events": n, "new_suggestions": n}`.
- Never modify `state.json` (it belongs to the app) except to drop ids of events that no longer exist in `events.json`.
- Validate both JSON files with `python3 -m json.tool`, then commit only `events.json` and `accounts.json` with the message `Weekly scan YYYY-MM-DD`, and push. If the push is rejected, `git pull --rebase` and push again.
- Close the tab you opened.

## 6. Report

End with a short French report: number of accounts followed and scanned, events found, the 5 best events of the week (date, title, place), new suggestions, and any problem.
