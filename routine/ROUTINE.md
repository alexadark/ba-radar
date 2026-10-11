# BA Radar weekly scan

You run unattended once a week for Alexandra. The run has two roles:

- **You, the collector** (steps 0, 1, 2, 4, 5): mechanical work. Drive the browser, collect raw posts, merge files, commit. Do not judge events yourself.
- **The judge** (step 3): a subagent you launch with the Agent tool, `model: "opus"`, `effort: "medium"`. It turns raw posts into scored events and account suggestions.

Goal: find the interesting events in Buenos Aires for the coming days on the Instagram accounts she follows, propose new accounts, and publish everything to the private data repository that her BA Radar app reads.

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

Browse like a person, page by page. Do not call Instagram's internal `/api/v1/` endpoints: they answer 429 and draw attention to the account.

For each account:

1. Navigate to `https://www.instagram.com/<handle>/`, then run with `javascript_tool`:

```js
await new Promise(r => setTimeout(r, 2500));
[...document.querySelectorAll('a[href*="/p/"], a[href*="/reel/"]')]
  .map(a => ({ href: a.getAttribute('href'), alt: a.querySelector('img')?.alt || '' }));
```

   The grid lists the newest posts first, after up to 3 pinned posts. `alt` often holds the start of the caption.
2. Open the posts in grid order (`https://www.instagram.com/p/<code>/`, keep the code from the href) and read each one:

```js
await new Promise(r => setTimeout(r, 2000));
({ time: document.querySelector('article time, main time')?.getAttribute('datetime'),
   caption: document.querySelector('meta[property="og:description"]')?.content });
```

   Stop for this account at the first non-pinned post older than `scan.lookback_days`, or after `scan.max_posts_per_account` posts. Keep every post you read, even if it looks irrelevant: the judge decides.
3. Append one record per post to the raw file `.scan/raw-YYYY-MM-DD.json` in the data repository (a JSON array, folder ignored by Git):
   `{"account": "<handle>", "post_url": "https://www.instagram.com/p/<code>/", "posted_at": "<time>", "pinned": true|false, "caption": "<og:description text>"}`
4. Wait 3 to 6 seconds between pages. If Instagram shows "Try again later", a checkpoint or a login page, stop scanning and go to step 5 with what you have, plus `"last_error"`.

Scan at most 60 accounts per run, starting with the ones not scanned for the longest time (keep `"last_scanned"` on each account in `accounts.json`).

## 3. Judge (delegated)

Launch one subagent with the Agent tool: `model: "opus"`, `effort: "medium"`, `run_in_background: false`, description `Judge BA events`. Give it this prompt, with the real paths and today's date filled in:

> You are the judge of the BA Radar weekly scan. Read the section "Judge rules" of /Users/webstantly/DEV/personal-projects/ba-radar/routine/ROUTINE.md and follow it. Inputs: the raw posts in `<raw file>`, and `config.json`, `accounts.json`, `events.json` in ~/DEV/personal-projects/ba-radar-data. Today is `<YYYY-MM-DD>`, timezone America/Argentina/Buenos_Aires. Write your output to `<data repo>/.scan/judged-YYYY-MM-DD.json` and reply with one line of counts.

Wait for it, then check that the judged file exists and parses. If the subagent fails, retry once; if it fails again, record `"last_error": "judge failed"` and go to step 5 without new events.

## Judge rules

These rules are for the judge subagent only.

Output file: `{"events": [...], "suggestions": [...]}`.

**Events.** From each raw post, extract every event happening between today and today + `scan.horizon_days`. A lineup post can give several events.

- Resolve relative dates ("este viernes", "sáb 18", "mañana") against `posted_at`, in the America/Argentina/Buenos_Aires timezone. If the date stays ambiguous, skip the event.
- When a caption has no clear date, time or place but the post looks like an event flyer, open `post_url` with Claude in Chrome (load `mcp__claude-in-chrome__*` with ToolSearch, open your own tab, close it after) and take a screenshot to read the flyer. Limit: 25 screenshots. Browse only; never click Follow, like or comment.
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
- Deduplicate: the same event posted by the venue and an artist keeps one entry, with the venue's post. Reuse the `id` of a matching event already in `events.json`.

**Suggestions.** From captions, note accounts that are mentioned (`@handle`) or co-authors and look like Buenos Aires venues, promoters, collectives, communities or organisers matching her interests. Ignore individuals' personal accounts, brands and sponsors. At most `scan.max_new_suggestions`, never a handle already in `accounts.json` whatever its status. Object: `{"handle", "name", "category", "barrio", "why" (one French sentence), "source": "mentioned by @<handle>"}`.

## 4. Merge suggestions

Add each judged suggestion to `accounts.json` with `"status": "suggested"` and `added`/`updated` set to now, skipping any handle already present.

## 5. Write and publish

- `events.json`: keep existing events whose date is still today or later, replace those with the same `id` by the judged version, add the new judged ones. Sort by `start`. Set `"generated_at"` to now (ISO with offset) and remove `"last_error"` when the run succeeded. Add `"stats": {"accounts_scanned": n, "posts_read": n, "events": n, "new_suggestions": n}`.
- Never modify `state.json` (it belongs to the app) except to drop ids of events that no longer exist in `events.json`.
- Validate both JSON files with `python3 -m json.tool`, then commit only `events.json` and `accounts.json` with the message `Weekly scan YYYY-MM-DD`, and push. If the push is rejected, `git pull --rebase` and push again.
- Close the tab you opened.

## 6. Report

End with a short French report: number of accounts followed and scanned, events found, the 5 best events of the week (date, title, place), new suggestions, and any problem.
