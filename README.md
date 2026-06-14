# Dead-Web Index — data

A reachability census of the most popular domains on the web. Every domain in the
[DomCop top-10M](https://www.domcop.com/top-10-million-websites) popularity list (this
release: the **top 500,000**) is probed and labelled **alive / redirect / blocked / dead**
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

| file | rows | description |
| --- | --- | --- |
| `data/polite.jsonl.gz` | 500,000 | one row per domain, polite arm |
| `data/reachability.jsonl.gz` | ~500,000 | one row per domain, reachability arm |
| `data/sample.jsonl` | 1,000 | uncompressed preview (both arms) |
| `data/summary.json` | — | run-level aggregates (outcome split, by-reason, by-TLD) |

Each line is one JSON record:

```json
{"domain":"example.com","tld":"com","rank":12345,"mode":"polite","outcome":"alive","reason":"ok","first_status":200,"final_status":200,"final_url":"https://example.com/","scheme":"https","hops":0,"parked":false,"run_id":"top500k-20260614","probed_at":"2026-06-14T..."}
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
| `run_id` | scan run (this release: `top500k-20260614`) |
| `probed_at` | RFC3339 timestamp |

## Method (brief)

HTTPS-first with HTTP fallback; cross-resolver DNS retry (8 public resolvers); redirects
followed (max 10); a retry on transient/timeout failures with an escalating deadline; a
raw TCP-connect check to separate "up but unresponsive" (blocked) from "gone" (dead);
parking-page detection. Homepage-level, from a datacenter vantage — a lower bound on
"alive" (deep pages and residential vantages reach more). Built with the scanner at
[Crawlora-org/ten-million-domains](https://github.com/Crawlora-org/ten-million-domains).

## This release

- **run_id**: `top500k-20260614` · top 500,000 domains (DomCop)
- Headline numbers are added with the data after the run completes.

## License

Data is licensed **CC BY 4.0** — free to use, share and adapt with attribution to
**Crawlora** (https://crawlora.net). See [`LICENSE`](LICENSE).
