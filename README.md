# cinemacity-watchdog

Watches the [Cinema City](https://www.cinemacity.cz) schedule and, when a new
showtime of **The Odyssey in IMAX** appears, opens an issue in this repo and
**assigns it to the repo owner**. GitHub turns that into an e-mail and a push
notification in the mobile app.

The assignment matters: by default e-mail is only sent for "Participating"
notifications (assignments, mentions, replies). Merely "Watching" a repo gives
a web/mobile notification only — e-mail for it is off under
Settings → Notifications until you turn it on.

It runs in GitHub Actions, so it works while your computer is off.

## How it works

- The workflow [`.github/workflows/watch.yml`](.github/workflows/watch.yml) is
  scheduled **every 5 minutes**. GitHub drops most scheduled runs under load,
  so in practice you get a check roughly every 15–30 minutes. The repo is
  public, so Actions minutes are free and unlimited.
- [`watch.py`](watch.py) downloads the schedule from the public JSON API of
  cinemacity.cz (`/cz/data-api-service/v1/quickbook/10101/…`) — no key, no login.
- The list of showtimes already seen lives in [`state/seen.json`](state/seen.json),
  which the workflow commits back after a run. So only additions are reported.
- New showtimes → an issue with time, auditorium, flags (70mm / subtitled /
  sold out) and a direct link to buy tickets. Showtimes that **disappeared**
  from the schedule (cancelled screenings) are reported too.
- The issue is **closed right after it is created**. It only serves as a
  delivery channel for the e-mail GitHub sends on creation — the list of open
  issues stays empty and nothing needs manual cleanup. The content stays
  readable under closed issues.
- Times are computed in the cinema's zone (`Europe/Prague`), not the runner's
  UTC. Otherwise a screening that just finished would look like a future one
  and be falsely reported as cancelled when it drops off the schedule.

One run is ~45 HTTP requests and takes ~20 seconds.

## What exactly is watched

Showtimes where the **film title** contains `odyss` **and** the **auditorium
name** contains `imax`. Currently that matches a single cinema in Czechia —
**Praha Flora**, auditorium `IMAX VOLVO`, where The Odyssey runs in 70mm with
Czech subtitles.

To avoid pulling the full schedule of all thirteen cinemas, the search runs in
two phases: first find which cinemas have an IMAX auditorium at all (one probe
on the nearest playing day plus an API hint via the `70-mm` attribute), then
walk only those in depth. If IMAX shows up at another cinema, it gets picked up
automatically.

Behaviour can be changed with environment variables in the workflow:

| Variable | Default | Meaning |
| --- | --- | --- |
| `FILM_PATTERN` | `odyss` | substring of the film title (case-insensitive) |
| `AUDITORIUM_PATTERN` | `imax` | substring of the auditorium name |
| `API_LANG` | `en_GB` | language of film/cinema names from the API; falls back to `cs_CZ` automatically if the API does not serve it |
| `HORIZON_DAYS` | `180` | how far ahead to ask |
| `HINT_ATTR` | `70-mm` | attribute for cheaply finding candidate cinemas |
| `REQUEST_DELAY` | `0.25` | pause between API requests (s) |

Watching anything else (say `FILM_PATTERN=dune`, `AUDITORIUM_PATTERN=4dx`)
means changing two variables and deleting `state/seen.json`. Note that
`FILM_PATTERN` is matched against the title in the `API_LANG` language.

## I want to watch it too (fork)

The watchdog needs no tokens or secrets — the Cinema City API is public and the
built-in `GITHUB_TOKEN` is enough to open issues. To get it running:

1. **Fork** this repo.
2. **Settings → General → Features → tick `Issues`.** Forks have issues turned
   off, and without them the watchdog has nowhere to report.
3. **Actions → "I understand my workflows, go ahead and enable them".** Then
   open the *Cinema City watchdog* workflow and click **Enable workflow** —
   GitHub does not run scheduled workflows in forks until you do.
4. Done. Issues are created and assigned to you, because the workflow uses
   `${{ github.repository_owner }}` — nothing to edit.

The state in `state/seen.json` is forked along, so you are not flooded with the
current schedule; the first message comes with the first new showtime. To see
what is playing right now, run the workflow manually with `force_report`.

## Manual run

**Actions → Cinema City watchdog → Run workflow**. Ticking *force_report*
reports all current showtimes, including known ones — handy for checking that
it is alive, or as "show me what's on".

```bash
gh workflow run watch.yml -f force_report=true
```

## Running locally

Plain Python 3, no dependencies:

```bash
python3 watch.py --state state/seen.json
```

Useful switches: `--seed` (only writes the state, reports nothing — good after
changing the filter), `--force-report` (prints everything regardless of state).

## Maintenance

- **Actions quota:** the repo is deliberately public — public repos get free,
  unlimited Actions minutes. If it were flipped to private, runs would count
  against the free 2,000 minutes per month and this cadence would blow through
  it; slow the cron down at the same time (e.g. `23 */2 * * *`).
- **60-day pause:** GitHub automatically disables the cron if nothing is pushed
  to the repo for 60 days. Not a risk while the film is playing — the workflow
  commits its own state.
- **Once The Odyssey finishes its run,** the watchdog simply stops reporting.
  Either switch it off (Actions → *Disable workflow*) or change `FILM_PATTERN`
  to the next film.
- If the Cinema City API changes, the workflow fails with an error and GitHub
  e-mails you about it.
