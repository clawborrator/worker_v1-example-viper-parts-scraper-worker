# worker_v1-example-viper-parts-scraper-worker

Docker compose stack for the Facebook Dodge Viper parts scraper.
A read-only Playwright-equipped Claude Code agent that, every 6
hours, visits each configured Facebook Dodge Viper group,
scrolls the feed, extracts buy/sell candidate posts, dedups
against prior cycles, and notifies `@clauderemote` with new
matches.

Read-only. No posting, no commenting, no reactions, no DMs. The
wrapper has no write-side verbs at all.

## Companion repo

The agent's playbook, Playwright wrapper, and config live in:

```
https://github.com/clawborrator/worker_v1-example-viper-parts-scraper-repo
```

`docker compose up` clones it into `/workspace/repo`.

## Setup (one time)

### 1. Capture Facebook cookies

From a browser already logged in to the Facebook account that's
a member of the Dodge Viper groups you want to scrape:

- **Chrome / Edge:** install **Cookie-Editor** extension, open
  facebook.com, click the extension icon, Export, "Export as JSON"
- **Firefox:** install **Cookie Quick Manager**, search
  "facebook.com", Export selection

Save the result as:

```
./secrets/facebook.cookies.json
```

It's gitignored. Don't commit it.

### 2. Mint a clawborrator channel token

```bash
claw token mint --name viper-parts-scraper
```

Copy the `ck_live_...` value.

### 3. GitHub PAT for the listings log

The scraper commits `data/listings/<date>.json` to its repo each
cycle. Needs `repo` scope.

### 4. Configure .env

```bash
cp .env.example .env
$EDITOR .env
```

Fill in: `CLAUDE_CODE_OAUTH_TOKEN`, `CLAWBORRATOR_TOKEN`,
`REPO_PAT`, `GIT_USER_EMAIL`, `GIT_USER_NAME`.

### 5. Configure groups + keywords

After first boot the agent clones the playbook repo into the
container. To configure WHICH groups to scrape and WHICH
keywords to match, edit the repo's `config/groups.json` and
`config/keywords.json`, commit, and push. The next cron fire
picks up the changes (the agent re-reads config every cycle).

You can also edit these BEFORE first boot by cloning the
playbook repo locally and editing there.

### 6. Start

```bash
docker compose up -d
docker compose logs -f viper-parts-scraper
```

First warmup cycle hits in roughly 30 to 60 seconds. After that,
`CronCreate 0 */6 * * *` paces the recurring cycles (4 per day).

## What to expect in @clauderemote

After each cycle the scraper sends a past-tense digest like:

> Scraped 3 Dodge Viper groups. 7 new candidates since last
> cycle: 4 WTS (1996 GTS hardtop in Phoenix, set of OEM wheels,
> brake calipers, intake manifold), 2 WTB (looking for stock
> exhaust, looking for trunk emblem), 1 ambiguous (someone
> asking about parts compatibility). Full list:
> data/listings/2026-05-16-180000Z.json

If a cycle has zero new matches, the notification just says
"no new matches this round."

## Stopping / restarting

```bash
docker compose down                  # stop + remove
docker compose up -d                 # restart (re-clones repo)
docker compose logs -f viper-parts-scraper
```

## When cookies expire

Facebook invalidates sessions on suspicious activity, password
changes, IP changes, etc. With stealth + headed mode, realistic
cookie lifetime is days to weeks for a read-only session. Less
than a posting bot. Refresh by re-exporting from your browser
and `docker compose restart viper-parts-scraper`.

## When Facebook changes the DOM

Facebook's "Comet" UI hashes class names per build. The wrapper's
`SELECTORS` object in `specialists/fb-groups.js` is centralized
for one-line fixes. When a cycle reports `selectors_stale` or
returns 0 posts from groups you know have activity, that is the
signal.

## See also

- `../worker_v1-playwright/` is the image
- `../worker_v1-example-viper-parts-scraper-repo/` is the playbook
- `../worker_v1-example-reddit-engager-worker/` is the sibling
  pattern for Reddit (read + write engagement)
- `../worker_v1-example-linkedin-engager-worker/` is the sibling
  for LinkedIn
