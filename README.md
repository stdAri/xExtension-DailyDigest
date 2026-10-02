# Daily Digest (xExtension-DailyDigest)

A [FreshRSS](https://freshrss.org) user extension that generates scheduled AI digests of your new articles.

At the configured times (default **08:00** and **20:00**), the articles added since the previous digest — across the categories you select (default: all) — are sent in a single request to any **OpenAI-compatible chat API**. The model writes a digest grouped by topic with numbered citations, which is stored as an unread article in a dedicated muted feed ("AI digest" feed, name is configurable). A **per-category** mode generates one digest per selected category instead, each in its own feed.

Each digest opens with the key stories of the period, followed by numbered topics. Citations read `[n original]`: the number opens the article **inside your FreshRSS**, “original” the original website. A collapsible source list is appended at the end of each digest.

## Features

- Combined digest (one feed) or per-category digests (one feed per category), with per-category windows and failure tracking.
- Any OpenAI-compatible endpoint: OpenAI, Gemini (OpenAI-compatible endpoint), Kimi, a local model, … Only standard fields (`model` / `messages` / `stream`) are sent.
- Optional **fallback model** (endpoint + key + model): used when the primary returns an empty completion (e.g. content-moderation aborts) or keeps failing after a retry.
- Configurable schedule, article cap, excerpt length, target language, prompt and feed name.
- Per-category prompt supplements and excerpt length overrides, applied only to per-category digests.
- Plain-markup layout (headings, lists, bold, dividers — no CSS), so digests look the same in the web UI and in reader apps that drop inline styles.
- Citations link into FreshRSS (`/i/?…` deep links) and to the original website; relative links are made absolute for API clients (Reeder, NetNewsWire, …) at display time.
- Retry policy with transient-error detection, per-slot attempt cap, and a lock file against concurrent runs.
- Optional: mark source articles as read after a digest is generated.
- Configuration page with status panel, **Test API connection** and **Generate now** buttons.
- English and Simplified Chinese UI.

## Requirements

- FreshRSS ≥ 1.28 (tested on 1.30).
- The [background refresh](https://freshrss.github.io/FreshRSS/en/admins/08_BackgroundTasks.html) (`app/actualize_script.php`) must run via cron for scheduled digests. Web page loads never trigger the LLM — only cron and the explicit "Generate now" button do.
- An API key for an OpenAI-compatible chat completions API.

## Installation

1. Copy (or clone) this directory into your FreshRSS `extensions/` directory as `xExtension-DailyDigest`.
2. In FreshRSS: **Settings → Extensions**, enable **Daily Digest** (it is a user-level extension, enable it for each user who wants digests).
3. Open its ⚙ configuration and fill in the API settings.

## Configuration

**Settings → Extensions → Daily Digest → ⚙**

Settings are ordered **API → Fallback → Output → Schedule and scope**, so the general prompt appears before the category settings.

- **API endpoint / key / model**: any OpenAI-compatible API; `/chat/completions` is appended to the endpoint automatically. A saved key is never displayed again.
- **Fallback model (optional)**: secondary endpoint / key / model — all three required, empty = disabled. Used when the primary model returns an empty completion (content-moderation abort) or still fails after one retry.
- **Output**: target language, general prompt and feed name.
- **Base URL for in-FreshRSS links**: leave empty for relative `/i/?…` links (works in the web UI over any host); third-party clients that cannot resolve relative links need the full URL of your FreshRSS, e.g. `https://freshrss.example.com` (applies to future digests only).
- **Mark source articles as read**: off by default.
- **Digest times**: comma-separated `HH:MM`, default `08:00,20:00`.
- **Mode**:
  - **Combined** (default): one digest over the selected categories, stored in the digest feed.
  - **Per category**: one digest per checked category, stored in feeds named `<feed name> · <category>` (auto-created, muted, placeholder URL `https://daily-digest.invalid/<user>/category-<id>`; recreated if deleted).
- **Categories**: in combined mode, none checked = all categories; in per-category mode, only checked categories get digests. The category holding the digest feeds is not listed.
- **Category settings**: expand ⚙ beside a checked category to add a prompt supplement and/or an excerpt length override (0–5000 characters). The panel shows “(set)” when either is configured. Both settings apply only in per-category mode.
- **Max articles per digest** (default 150, newest kept), **global excerpt length** (default 400 chars; 0 = titles only).
- Buttons: **Test API connection**, **Generate now** (manual run, 1–5 minutes; per-category mode generates each checked category in turn and reports per-category results on the page).

## How it works

- Scheduled generation hooks into `freshrss_user_maintenance`, but **only acts in the CLI** (cron running `actualize_script.php`), deferred until after that run's feed refresh has finished. Normal page loads never call the LLM.
- On each cron run, the most recent scheduled slot ≤ now is computed; if it has no digest yet, one is generated. When the extension is first enabled, past slots are not back-filled.
- Article window: everything added since the previous digest (read and unread alike; the first window covers the last 12 hours), excluding the digest feeds themselves, `[Summary]` / `AI Summary` articles, and archived (hidden) feeds.
- An empty window is skipped silently but the slot counts as done.
- Retries within one run (at most 3 API calls per digest):
  1. primary model;
  2. on a transient network error (curl errors, HTTP 429 / 5xx): wait 15 s, call the primary again;
  3. empty/moderated completion, or transient error persisting: call the fallback model once (if configured).
  Permanent errors (HTTP 400 / 401, …) fail immediately. The digest's header line and the log state which model produced it.
- If the API still fails, the slot is not advanced: the next cron run retries, up to 3 attempts per slot; afterwards the next slot takes over the same window.

### Per-category mode

- Shares the schedule with combined mode. One cron run calls the LLM once per checked category (N categories ≈ N calls, N× the tokens).
- **Prompt supplement** (`cat_prompts`, keyed by category ID): appended after the general prompt, with a heading naming the category. It takes precedence over conflicting general requirements. `{language}` is replaced in both prompts; the mandatory output format rules are always appended last and cannot be overridden.
- **Excerpt length override** (`cat_excerpts`, keyed by category ID): leave empty to use the global setting, or set 0–5000 characters (0 = titles only). Longer excerpts cost proportionally more input tokens for that category. The digest meta line indicates when a category prompt or excerpt override was used, including the overridden length.
- Each category tracks its own window (`last_entry_id`) and failure count: one failing category does not block the others, and only failed categories are retried on the next cron run (max 3 attempts per slot per category). After 2 consecutive category failures within one run, the rest is postponed to the next cron run, to avoid hammering a broken API. The slot counts as done once every category succeeded or gave up.
- A category's **first** digest covers the last 12 hours, but never articles already covered by a combined digest. Later digests go back at most 48 hours (e.g. after re-checking a long-unchecked category).
- Switching back to combined mode continues the combined window from the last per-category run, so already-digested articles are not repeated (articles from unchecked categories are not back-filled).
- Category → feed mapping is stored in `cat_feeds`; a lock file (`data/cache/dailydigest-<user>.lock`) prevents concurrent runs.
- Layout: key points (the 5–8 most important stories across all topics, unnumbered), then the topics. The extension numbers the topic headings (「一、二、…」 when the digest language is Chinese, `1.` `2.` otherwise; numbers written by the model are replaced) and puts a `<hr>` before each topic and before the source list. Only plain tags are used (`h3 ul li strong sup hr em`), since many reader apps drop inline CSS.
- The model refers to articles only by `[n]` numbers; the extension turns them into a superscript `[n original]`: the number links to the article **inside FreshRSS** (title as tooltip), “original” to the original website. Adjacent citations such as `[3][7]` become one group. A collapsible source list (title → inside FreshRSS, plus “original”) and a one-line legend are appended. The link format `/i/?get=f_<feed id>&state=3&search=e%3A<entry id>` searches one entry by id inside its feed (`state=3` = read + unread), yielding exactly one result; `f_` is used instead of `get=a` so entries of "category-only" feeds also open. Model output is filtered through a tag whitelist with all attributes stripped.
- Log lines are prefixed `Daily Digest:` (see **Settings → Logs**). API keys are never logged.

## Notes

- The configuration page uses `static/style.css`, registered with `Minz_View::appendStyle()`: FreshRSS’ `default-src 'self'` Content Security Policy ignores inline styles. The stylesheet controls the category settings panels and the general prompt textarea width; keep it with the extension when installing or updating.
- Digests are generated by the first cron run at or after each scheduled time. With a `*/45` cron, 08:00 and 20:00 land exactly on schedule, while e.g. 08:10 waits until 08:45.
- The digest feeds use placeholder URLs (`https://daily-digest.invalid/<user>`, per-category `…/category-<id>`) and are muted, so they are never refreshed automatically; pressing their "refresh" button manually shows an error you can ignore.
- Some providers abort on sensitive content mid-completion (empty content, `finish_reason: null`, zero tokens). The log reports this as a likely content-moderation abort; the fallback model (if configured) takes over for that digest, otherwise that digest fails.

### How in-FreshRSS links reach third-party clients

In-FreshRSS links are stored relative (`/i/?…`). When an entry is served through the Google Reader API, the extension absolutizes them against the current request's host: LAN access gets `http://<lan-host>/i/?…`, clients syncing through your public domain get `https://<public-domain>/i/?…`. The scheme prefers `X-Forwarded-Proto`; without it, IPs, hosts with ports and localhost are treated as http, other hostnames as https. Leaving "Base URL for in-FreshRSS links" empty is usually right; filling it hard-codes that base into newly generated digests instead.

## Uninstall

1. Disable Daily Digest under **Settings → Extensions**.
2. Delete the digest feeds ("AI digest" and "… · category" feeds) — optional.
3. Remove the `extensions/xExtension-DailyDigest` directory.
   The leftover `extensions → Daily Digest` entry in the user configuration is harmless.

## License

[AGPL-3.0](LICENSE), same as FreshRSS itself.
