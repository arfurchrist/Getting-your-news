# Build Brief — Self-Hosted Full-Text News RSS Aggregator

**Audience:** a Claude Code instance tasked with standing this up from scratch on a
**Debian 13 (trixie)** Linux box.
**Goal:** reproduce a self-hosted news reader that pulls RSS feeds from AU / UK /
international sources, enriches them to **full article text**, and presents them in one
web UI — then **generate its own documentation** so the whole thing is git-committable.

> Addresses and ports below are **placeholders** — substitute real values for the target
> host. Examples use the `10.20.30.0/24` space and ports in the `60xx` range.

---

## 1. Target environment

- **OS:** Debian 13 (trixie), amd64.
- **Runtime:** the **system Docker engine** (`docker-ce` + `docker-compose-plugin`) — NOT
  Docker Desktop. The system engine auto-starts at boot with no user session, which is the
  robust choice for an always-on service.
- **Access:** the box is reachable on the LAN (example `10.20.30.10`); the web UI is
  exposed on a `60xx` port.

## 2. Architecture

```
┌─ Debian 13 host  (example 10.20.30.10) ─────────────────────────────┐
│  Docker engine (system, enabled at boot)                            │
│  docker network: feeds-net (bridge)                                 │
│  ┌────────────────────────┐   internal HTTP   ┌───────────────────┐ │
│  │ freshrss               │ ────────────────► │ fivefilters       │ │
│  │ freshrss/freshrss       │  (hostname on    │ full-text-rss     │ │
│  │ web :6052 → 80          │   feeds-net)     │ :6050 → 80        │ │
│  │ vol freshrss-data       │                  │ vol ff-cache      │ │
│  └────────────────────────┘                   └───────────────────┘ │
│  Reader browses  http://10.20.30.10:6052                            │
└─────────────────────────────────────────────────────────────────────┘
```

**Two containers, one network:**

| Service | Image | Port (example) | Volume | Role |
|---------|-------|----------------|--------|------|
| `freshrss` | `freshrss/freshrss:latest` | `6052 → 80` | `freshrss-data` → `/var/www/FreshRSS/data` | RSS aggregator, web UI, refresh cron, storage (SQLite) |
| `fivefilters` | `heussd/fivefilters-full-text-rss:latest` | `127.0.0.1:6050 → 80` | `fivefilters-cache` → `/var/www/html/cache` | Fetches each article and extracts full body text |
| network `feeds-net` | bridge | — | — | lets FreshRSS reach FiveFilters by hostname `fivefilters` |

**Core design decision — full text for every feed:** FreshRSS does **not** subscribe to
source feeds directly. Each feed URL is wrapped by FiveFilters:

```
http://fivefilters/makefulltextfeed.php?url=<URL-ENCODED-SOURCE-FEED>&max=20&links=preserve
```

(`fivefilters` resolves over `feeds-net`.) This turns excerpt-only feeds into full articles.
Trade-off: FiveFilters becomes a single point of failure for all feeds, and any source that
blocks scraping will fail the full-text step (see §7).

## 2a. Local viewer clients

Besides the FreshRSS web UI, this solution is read through **two native desktop RSS
clients** that sync against the server over FreshRSS's **Google Reader-compatible API**:

| Viewer | Package (flatpak) | Sync method |
|--------|-------------------|-------------|
| **Newsflash** | `io.gitlab.news_flash.NewsFlash` | GReader API |
| **Fluent Reader** | `me.hyliu.fluentreader` | GReader API |

Both connect to: `http://<host>:6052/api/greader.php`, authenticating with the FreshRSS
username and its **API password** (set per-user in FreshRSS → Profile → *API management*).
Requirement: enable **Settings → Authentication → Allow API access** in FreshRSS.
Failure mode to document: if the server is unreachable or the API password changes, clients
report *"account needs to be logged in"* / fail to refresh their sidebar — re-authenticate to fix.

## 3. Build steps

1. **Install Docker (system engine) on Debian 13** and enable it at boot:
   - Add Docker's official apt repo for Debian trixie, install
     `docker-ce docker-ce-cli containerd.io docker-compose-plugin`.
   - `sudo systemctl enable --now docker`
   - Add the operator user to the `docker` group (optional, for non-root `docker`).
2. **Create the project** at e.g. `/opt/news-rss/` with the compose file in §4.
3. **Bring it up:** `docker compose up -d`.
4. **Configure FreshRSS** (first-run wizard at `http://<host>:6052`, or the `cli/` scripts):
   - DB: SQLite (default; simplest, single-volume backup).
   - Create the reader user (example `newsreader`).
   - Set timezone and refresh cron (example `TZ=<Area/City>`, `CRON_MIN=1,16,31,46` = every 15 min).
5. **Add feeds** grouped into per-country categories (§6). Either add them in the UI, or
   author an **OPML** file where each feed's `xmlUrl` is the FiveFilters-wrapped URL from §2,
   and import it via the UI / `cli/import-for-user.php`.
6. **Enable the API + connect the viewers:** turn on *Allow API access* in FreshRSS, set an
   API password for the user, then install the two flatpak clients (§2a) and add a FreshRSS /
   Google-Reader-API account in each pointing at `http://<host>:6052/api/greader.php`.
7. **Verify** (§8), then **generate the git docs** (§9).

## 4. `docker-compose.yml` (author this into the repo)

```yaml
services:
  freshrss:
    image: freshrss/freshrss:latest
    container_name: freshrss
    restart: unless-stopped
    ports:
      - "6052:80"                 # <host-port in 60xx> : 80
    environment:
      TZ: "Area/City"             # e.g. Australia/Sydney
      CRON_MIN: "1,16,31,46"      # refresh every 15 min
    volumes:
      - freshrss-data:/var/www/FreshRSS/data
    networks: [feeds-net]

  fivefilters:
    image: heussd/fivefilters-full-text-rss:latest
    container_name: fivefilters
    restart: unless-stopped
    ports:
      - "127.0.0.1:6050:80"       # VM-local only; reached internally as http://fivefilters
    volumes:
      - fivefilters-cache:/var/www/html/cache
    networks: [feeds-net]

networks:
  feeds-net:

volumes:
  freshrss-data:
  fivefilters-cache:
```

## 5. Feed URL pattern (per feed)

For each source feed `<SRC>`, subscribe FreshRSS to:
```
http://fivefilters/makefulltextfeed.php?url=<urlencode(SRC)>&max=20&links=preserve
```
Example (ABC Top Stories): `url=www.abc.net.au%2Fnews%2Ffeed%2F...%2Frss.xml`.

## 6. Feed sources — organise as categories by country

Create one FreshRSS category per country group and populate:

- **Australia** — ABC News (Top Stories, Around Australia, Just In, Business, Politics,
  Sport, World, and state feeds NSW / QLD / VIC / WA); Sydney Morning Herald (Latest,
  National, National/NSW, Business, Politics/Federal, World); The Age (Latest,
  National/Victoria); news.com.au (National, World); Crikey.
- **United Kingdom** — BBC News; The Guardian; The Independent; Daily Mail (News, Sport,
  World); Daily Mirror; Daily Express; Evening Standard; Metro.
- **International** — Al Jazeera; CNBC (International Top News, US Top News); CBNNews.com.

(Use each publisher's current public RSS URLs; wrap every one per §5.)

## 7. Known issues to design around

- **Sites that block scraping** (observed: news.com.au returns **HTTP 403** to the fetcher)
  will fail the full-text step. Mitigations: set a browser-like `User-Agent` in FiveFilters
  for that host, or subscribe to that source's **raw** RSS (skip the wrapper), or drop it.
- **FiveFilters is a single point of failure** for all feeds — keep it in the restart policy.

## 8. Verification

- `docker compose ps` → both `Up`.
- `curl -s -o /dev/null -w '%{http_code}' http://<host>:6052/` → `302` (login) = healthy.
- `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:6050/` → `200`.
- `docker logs --tail 50 freshrss` → confirm feeds actualize; note any per-source errors.
- After one cron cycle, articles appear in the UI grouped by country category.

## 9. Boot persistence

With the **system Docker engine** this is automatic: `systemctl enable docker` +
`restart: unless-stopped` on both services → the stack returns after any reboot, no user
login required. (Do **not** use Docker Desktop for this; its per-user model needs
`loginctl enable-linger` and is fragile for an always-on service.)

## 10. Deliverable — make it self-documenting for git

As the **final step**, the building instance MUST write, into the project repo, so the
whole solution can be committed and re-deployed:

1. **`docker-compose.yml`** (from §4, with real port/TZ values filled in).
2. **`feeds.opml`** — the exported feed list (`cli/export-opml-for-user.php --user <user>`),
   which captures every wrapped feed URL and its country category.
3. **`SOLUTION.md`** — a high-level design doc it authors covering: components table,
   an architecture diagram, the full-text data-flow, feed sources by country, the two local
   **viewer clients** (§2a) and how they connect via the GReader API, known issues, and an
   operations/quick-reference section. Keep host-specific addresses as placeholders or in a
   separate uncommitted `.env`.
4. **`README.md`** — quickstart: prerequisites, `docker compose up -d`, first-run config,
   how to add feeds, backup (the `freshrss-data` volume), and restore.

The result is a self-contained, reproducible, git-committable repo.

---

## Placeholder key (substitute real values)

| Placeholder | Meaning | Example |
|-------------|---------|---------|
| `10.20.30.0/24` | host LAN subnet | your network |
| `10.20.30.10` | Debian 13 host address | your host |
| `6052` | FreshRSS web port | any free `60xx` |
| `6050` | FiveFilters port (host-local) | any free `60xx` |
| `newsreader` | FreshRSS username | your choice |
| `Area/City` | timezone | e.g. `Australia/Sydney` |
