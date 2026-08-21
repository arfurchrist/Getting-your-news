# Getting-your-news
Fingerprinting and Content Manipulation Mitigation

Browser fingerprints - Search their characteristics if needed

If a Browser can be uniquely identified, which it can, and there is a method to manipulate content, which there is. It follows bespoke content can be delivered to a specific group or single browser instance. The bespoke content may be a targeted headline or a repeating attack vector across your browser viewing of online news. There are various techniques in this space.

If you look across the various projects in this repo you may observe there is a repeating occurrence of an entity which appears to act as a conduit. This entity has had access to live sport, live news, online news, printed news. To anchor the proposition. The hosts (live news and sport) have to be complicit in those techniques, which would indicate most probable knowledge of and active involvement in this method (fingerprinting and crafted content delivery). The leopard does not change its spots. What can be further deduced, if you progress along this path, is the makings of and quite probable existence of mass social coercion and control, in at least one 'democratic' nation, with strongly invasive surveillance powers and regular accusations of being the most secretive democracy in existence.  

This is small collection of tools which pulls RSS news feeds and presents them locally. This mitigates the above combination of identification and content manipulation attack. It also provides a local copy, which can be used as a reference.

If you give the build-brief.md to an agent it will build the tools for you locally on Linux. You can also create a VM using open source Linux virtualization and then build it inside the VM.

# Self-Hosted Full-Text News RSS Aggregator

A self-hosted news reader that pulls RSS feeds from AU / UK / international sources,
enriches them to **full article text**, and presents them in one web UI and two desktop
clients.

> Addresses and ports below are **placeholders** — substitute your own. Examples use the
> `10.20.30.0/24` space and ports in the `60xx` range.

## 1. Where it runs

A single Linux host running the **system Docker engine** (Debian 13). Two containers on a
private docker network; read via the web UI or native desktop clients on the LAN.

```
┌─ Host  (example 10.20.30.10) ───────────────────────────────────────┐
│  Docker engine (system, enabled at boot)                            │
│  docker network: feeds-net (bridge)                                 │
│  ┌────────────────────────┐   internal HTTP   ┌───────────────────┐ │
│  │ freshrss               │ ────────────────► │ fivefilters       │ │
│  │ freshrss/freshrss      │  (hostname on     │ full-text-rss     │ │
│  │ web :6052 → 80         │   feeds-net)      │ :6050 → 80        │ │
│  │ vol freshrss-data      │                   │ vol ff-cache      │ │
│  └────────────────────────┘                   └───────────────────┘ │
│        ▲  web UI + GReader API                                      │
└────────┼────────────────────────────────────────────────────────────┘
         │  http://10.20.30.10:6052   (browser, Newsflash, Fluent Reader)
```

## 2. Components

| Component                     | Image / package                           | Port                     | Role                                                         |
| ----------------------------- | ----------------------------------------- | ------------------------ | ------------------------------------------------------------ |
| **FreshRSS**                  | `freshrss/freshrss:latest`                | `6052 → 80`              | RSS aggregator, web UI, refresh cron, storage (SQLite in `freshrss-data`) |
| **FiveFilters Full-Text RSS** | `heussd/fivefilters-full-text-rss:latest` | `6050 → 80` (host-local) | Fetches each article and extracts full body text             |
| **feeds-net**                 | docker bridge network                     | —                        | Lets FreshRSS reach FiveFilters by hostname `fivefilters`    |
| **Newsflash** *(viewer)*      | flatpak `io.gitlab.news_flash.NewsFlash`  | —                        | Desktop client; syncs via FreshRSS Google Reader API         |
| **Fluent Reader** *(viewer)*  | flatpak `me.hyliu.fluentreader`           | —                        | Desktop client; syncs via FreshRSS Google Reader API         |

Both viewers connect to `http://<host>:6052/api/greader.php` (enable *Allow API access* +
set a per-user API password in FreshRSS). Refresh cron runs every 15 min.

## 3. Data flow

1. FreshRSS cron fires (every 15 min).
2. For each feed it requests a **FiveFilters wrapper**, not the raw source:
   `http://fivefilters/makefulltextfeed.php?url=<urlencoded-source>&max=20&links=preserve`
   (resolved over `feeds-net`).
3. FiveFilters fetches the source RSS, visits each article, extracts the **full body**, and
   returns an enriched feed.
4. FreshRSS stores the full-text articles in the `freshrss-data` volume.
5. You read them at `http://<host>:6052` — in the browser, or in Newsflash / Fluent Reader
   over the GReader API — grouped by country category.

> Every feed depends on FiveFilters; a source that blocks scraping (e.g. returns HTTP 403)
> fails the full-text step even though its raw RSS is fine.

## 4. Feed sources — by country

- **🇦🇺 Australia** — ABC News (Top Stories, Around Australia, Just In, Business, Politics,
  Sport, World, NSW/QLD/VIC/WA); Sydney Morning Herald (Latest, National, National/NSW,
  Business, Politics/Federal, World); The Age (Latest, National/Victoria); news.com.au
  (National, World); Crikey.
- **🇬🇧 United Kingdom** — BBC News; The Guardian; The Independent; Daily Mail (News, Sport,
  World); Daily Mirror; Daily Express; Evening Standard; Metro.
- **🌍 International** — Al Jazeera; CNBC (International Top News, US Top News); CBNNews.com.
