# Client Meta Ads reports — deploy pack

Three self-refreshing client reports. Each folder is one client and contains two files:

```
halcyon/index.html   the report page   (upload once, rarely changes)
halcyon/data.json    the numbers       (replaced daily/live — this is the only file that changes)
mana/…
elixir/…
.htaccess            tells the browser never to cache data.json, and tells crawlers to stay out
robots.txt            belt-and-suspenders: tells search engines not to index anything under /reports
```

`index.html` fetches `data.json` from its own folder when the page opens. So a refresh only has to
replace the small JSON file for that client — the page itself stays put and loads instantly.

Client URLs end up as:

```
https://yourdomain.com/reports/halcyon/
https://yourdomain.com/reports/mana/
https://yourdomain.com/reports/elixir/
```

**Halcyon's report is the richer one.** It's the dedicated report built for sending straight to
Halcyon: a live "today" bar, 7-day calls/spend/CTR summary, call-ads detail, website-leads detail
(with cost per lead and average response time worked out from their own lead sheet), and an "All
leads" tab listing every lead with name, phone number, and status. Mana and Elixir are on the
simpler generic template (results + trends + campaign table) since they don't have a CRM feed.

---

## Important — Halcyon's report contains personal data

The All-leads tab shows real names and phone numbers pulled from Halcyon's own lead sheet. That's
a materially bigger exposure than Mana's or Elixir's report, both of which only show campaign
numbers. Before you put this on a public URL:

- **Password-protect the `halcyon` folder specifically** (step 4 below) with a password you don't
  reuse elsewhere, and only hand it to people at Halcyon who need it.
- The page already ships with `<meta name="robots" content="noindex,nofollow,noarchive">` and the
  repo's `robots.txt` disallows the whole `/reports` path, so a well-behaved search engine won't
  index or cache it. That's a safety net, not a substitute for the password — robots.txt is only a
  request, and a guessed or leaked URL still works without the folder password.
- If you'd rather not have names/numbers on a hosted page at all, say so and the leads tab can be
  dropped from this version — it'd stay available only in files sent to you directly.

---

## One-time setup on Hostinger Cloud (Git auto-deploy)

Your plan supports Git deployment, which is the cleanest route — nothing to upload by hand after this.

**1. Put this folder in a private Git repo**

```bash
cd site
git init
git add .
git commit -m "Client Meta Ads reports"
git remote add origin git@github.com:YOURNAME/ads-reports.git
git push -u origin main
```

Keep the repo **private**. The JSON holds client spend figures, and Halcyon's also holds lead
names and phone numbers.

**2. Connect the repo in hPanel**

hPanel → your website → **Advanced → GIT** → *Create new repository*:

- Repository: your repo's SSH or HTTPS URL
- Branch: `main`
- Install path: `public_html/reports`

If it's a private repo, hPanel shows an SSH key — add it to GitHub under
*Settings → Deploy keys* on that repo.

**3. Turn on auto-deploy**

Still in hPanel → GIT, copy the **webhook URL** shown next to the repository. In GitHub go to
*Settings → Webhooks → Add webhook*, paste it, content type `application/json`, save.

From now on every `git push` deploys within seconds. No FTP, no file manager.

**4. Password-protect each folder (do this for Halcyon at minimum)**

hPanel → **Files → Password Protect Directories**. Protect `public_html/reports/halcyon` (and
`mana`/`elixir` too, if you want) — a separate username and password per client. Without this,
anyone who has or guesses the URL can read the page.

---

## Keeping it "real time"

This is a static page reading a JSON file, not a live database connection — Meta's own API doesn't
push data instantly either, so "real time" in practice means *refreshed often enough that no one
notices the gap*. Two ways to run the refresh:

**A — Scheduled task (recommended).** Ask Claude to *"set up the scheduled refresh for the Halcyon
report"* and it will run on a schedule unattended: pull the latest figures from Meta and the lead
sheet, regenerate `halcyon/data.json` (and `mana`/`elixir` if wanted), commit, and push. Hostinger's
webhook deploys it within seconds of the push. This needs two things from you first:

- **A GitHub repo** for this `site/` folder (step 1 above), kept private.
- **A GitHub personal access token** with push (`repo`) access to that one repo, so the scheduled
  task can push on your behalf. Create one at github.com → Settings → Developer settings → Personal
  access tokens → Fine-grained tokens, scoped to just this repo, and share it with Claude when
  asked. Treat it like a password — it's what lets an automated job update your reports.

Pick a cadence: hourly is the most "live" this can realistically get, since Meta doesn't finalize
same-day numbers until the day ends anyway — the today-bar on Halcyon's report already stays
current within the current day for calls/leads/spend even between refreshes, as long as the
refresh has run at least once that day.

**B — Manual, whenever you remember.** Come back to this session (or start a new one) and say
*"refresh the Halcyon report"* — Claude pulls fresh Meta + sheet data, rebuilds `data.json`, and
pushes. Takes a couple of minutes, no ongoing setup required.

**Note on timing:** Meta does not finalize the current day's numbers until it ends, so anything
pulled mid-day is genuinely live-but-partial — that's what the "still updating" badge on the
today-bar communicates. A refresh run first thing in the morning (after ~6am IST) is the most
accurate point to catch up on the prior day in full.

---

## If you ever want to move off Git

The pages are plain static files. Any of these work equally well:

- **Manual:** upload the folders once via hPanel File Manager, then each refresh replace only the
  relevant `data.json` file(s).
- **Cron + PHP:** publish the JSON somewhere fetchable and have a Hostinger cron job pull it
  periodically into each folder. Useful if the refresh job cannot reach your server directly.
- **SFTP:** Cloud plans include SSH, so an `rsync` or `scp` of the JSON files also works.

---

## Notes on the reports themselves

- Figures are whatever Meta reported for that ad account, plus (Halcyon only) whatever's logged in
  the lead sheet. Amounts are in INR.
- **Results** means each campaign's own objective — phone calls for Mana and Halcyon, app installs
  for Elixir. Cost per result divides only the spend of the campaigns chasing that outcome.
- **People reached** is de-duplicated by Meta and cannot be summed across days. The Mana/Elixir
  reports show Meta's exact figure for the full 30-day and last-7-day views, and mark any other
  custom range `APPROX`.
- Halcyon's cost-per-lead and response-time figures come from the lead sheet, not Meta's own pixel
  count — the report explains why in its own copy (Meta's pixel misses leads on 3 of 4 website
  campaigns). Response time only covers leads marked one at a time; batch-updated leads are labelled
  "marked in a batch" rather than given a misleading number.
- Each file contains **only that client's data**. The build script fails if one client's name
  appears in another's file.
- No agency name appears anywhere. If you want one added later it goes in the footer — search
  `About these figures` in each `index.html`.
