# Dead-Web Index — data

A reachability census of the most popular domains on the web. Every domain in the
[DomCop top-10M](https://www.domcop.com/top-10-million-websites) popularity list (this
release: the **full top 10 million**) is probed and labelled **alive / redirect / blocked / dead**
— once by an honest *polite* bot and once by a browser-like *reachability* client.

It is the dataset behind the Crawlora **Dead-Web Index** explorer.

## What "dead" means

A domain is **dead only if it is genuinely unreachable** — no DNS resolution, a
refused/reset connection, or nothing accepting a TCP connection. A server that answers
*any* HTTP status (including 404, 5xx, or a Cloudflare 522) or merely accepts a TCP
connection is **alive** or **blocked**, not dead: the host is up, it just may not serve a
homepage or may refuse a bot. Many entries in a popularity list are asset/CDN/API hosts
(e.g. `static.cloudflareinsights.com`, `fonts.googleapis.com`) that 404 or tarpit a bare
`GET /` yet are very much alive.

- **alive** — a usable HTTP response (2xx, or a 4xx/5xx the server answered).
- **redirect** — ended on an unresolved redirect.
- **blocked** — host up but would not serve us: anti-bot / auth / rate-limit, or it
  accepts a TCP connection yet will not complete HTTP (tarpit/timeout, strict TLS).
- **dead** — no DNS, connection refused/reset, or nothing listening.

The two **modes** are the point: a *polite* plain-HTTP bot vs a *reachability* client
with a real Chrome TLS/JA3 fingerprint. The gap — a domain dead/blocked to the bot but
alive to the browser — is the slice of the web that is reachable with better tooling.

## Files

The full per-arm data ships as **release assets** on the
[latest release](https://github.com/Crawlora-org/dead-web-index-data/releases/latest), split into
**10 chunks per arm** partitioned by DomCop rank (~1,000,000 domains each) and compressed with
**zstd**. Every chunk is a self-contained `.jsonl.zst` (~20 MB) — independently decodable, so you
can grab just the popularity band you need, resume a download, or fetch in parallel. The repo
carries a small preview, the run-level aggregates, and a checksummed manifest.

| file | rows | where | description |
| --- | --- | --- | --- |
| `polite.part-01.jsonl.zst` … `part-10` | 9,992,781 total | release assets | polite arm, 10 rank-partitioned chunks (`part-01` = ranks 1–1,000,000 … `part-10` = 9,000,001–10,000,000) |
| `reachability.part-01.jsonl.zst` … `part-10` | 9,997,315 total | release assets | reachability (browser) arm, same 10-chunk rank partition |
| `data/manifest.json` | — | in repo | per-chunk rows, rank range, byte size, and **sha256** |
| `data/sample.jsonl` | 1,000 | in repo | uncompressed preview (both arms) |
| `data/summary.json` | — | in repo | run-level aggregates (outcome split, by-reason, by-TLD) |

### Getting the data

```bash
base=https://github.com/Crawlora-org/dead-web-index-data/releases/download/v1.0.0
# grab one arm's 10 chunks, then decode + concatenate into a single JSONL stream:
for i in $(seq -w 1 10); do curl -fL -O "$base/polite.part-$i.jsonl.zst"; done
zstd -dc polite.part-*.jsonl.zst > polite.jsonl        # needs `zstd` (apt/brew install zstd)
```

zstd-compressed JSONL is read directly by DuckDB (`read_json_auto('polite.part-*.jsonl.zst')`)
and Polars; pandas needs the `zstandard` package (or just decode with `zstd -d` first). Verify
each download against the `sha256` in [`data/manifest.json`](data/manifest.json).

Each line is one JSON record:

```json
{"domain":"example.com","tld":"com","rank":12345,"mode":"polite","outcome":"alive","reason":"ok","first_status":200,"final_status":200,"final_url":"https://example.com/","scheme":"https","hops":0,"parked":false,"run_id":"top10m-20260615","probed_at":"2026-06-15T..."}
```

| field | meaning |
| --- | --- |
| `domain` | the probed host |
| `tld` | last dot-label (coarse TLD) |
| `rank` | DomCop popularity rank |
| `mode` | `polite` or `reachability` |
| `outcome` | `alive` / `redirect` / `blocked` / `dead` |
| `reason` | finer reason (`ok`, `dns_failed`, `timeout`, `not_found`, `forbidden`, `rate_limited`, `server_error`, `tls_error`, `connection_refused`, …) |
| `first_status` | HTTP status of the first hop (0 if none) |
| `final_status` | HTTP status after redirects (0 if no response) |
| `final_url` | URL of the final response |
| `scheme` | `https` or `http` |
| `hops` | redirects followed |
| `parked` / `park_vendor` | parked/for-sale domain + vendor |
| `run_id` | scan run (this release: `top10m-20260615`) |
| `probed_at` | RFC3339 timestamp |

## Method (brief)

HTTPS-first with HTTP fallback; cross-resolver DNS retry (8 public resolvers); redirects
followed (max 10); a retry on transient/timeout failures with an escalating deadline; a
raw TCP-connect check to separate "up but unresponsive" (blocked) from "gone" (dead);
parking-page detection. Homepage-level, from a datacenter vantage — a lower bound on
"alive" (deep pages and residential vantages reach more). The classification logic — DNS /
TCP / HTTP probing and the alive / redirect / blocked / dead rules — is open-sourced as a CLI
you can run on your own domains:
**[crawlora-deadweb](https://github.com/Crawlora-org/crawlora-deadweb)**
(`go install github.com/Crawlora-org/crawlora-deadweb@latest`).

## This release

- **run_id**: `top10m-20260615` · the full top 10,000,000 domains (DomCop), June 2026
- **polite**: 14.1% dead · 8.9% blocked · 76.6% alive · 0.3% redirect (9,992,781 domains)
- **reachability**: 14.1% dead · 8.2% blocked · 77.5% alive · 0.2% redirect (9,997,315 domains)
- Genuinely-dead (**14.1%**) is identical across both arms — no-DNS / no-connect does not depend on the client fingerprint. The browser-fingerprint arm clears more anti-bot walls, so its *blocked* rate is lower (8.9% → 8.2%): a real Chrome TLS/JA3 fingerprint reaches **~72,000** sites the polite bot is shut out of.

## How to cite

This repository ships a [`CITATION.cff`](CITATION.cff), so GitHub shows a **“Cite this
repository”** button that generates BibTeX/APA for you. Plain text:

> Crawlora (2026). *Crawlora Dead-Web Index: a reachability census of the top 10 million
> domains* (v1.0.0) [Data set]. https://github.com/Crawlora-org/dead-web-index-data

## Data collection & ethics

- **Public infrastructure only.** Each record is the result of an unauthenticated HTTP
  `GET /` (plus DNS/TCP checks) to a domain's public homepage — the same request any
  browser makes. No authentication was bypassed, no login-only or `robots.txt`-disallowed
  paths were fetched, and no page content is republished. The dataset records only the
  *reachability state* of a public endpoint.
- **No personal data.** Records contain domain names and their observed HTTP/DNS status —
  no WHOIS registrant data, no emails, no personal identifiers. Domain names in a
  popularity ranking are public records of (overwhelmingly) organisations, not individuals.
- **Point-in-time.** This is a single snapshot (see `run_id`); a domain dead today may
  resolve tomorrow and vice-versa. Treat it as a census, not a live status feed.
- **A lower bound on “alive.”** Probing is homepage-level from a datacenter vantage. Deep
  pages, residential vantages, and per-geo content reach more than a bare `GET /` does, so
  the true “alive” share is somewhat higher than reported.

## License

Data is licensed **CC BY 4.0** — free to use, share and adapt with attribution to
**Crawlora** (https://crawlora.net). See [`LICENSE`](LICENSE).
