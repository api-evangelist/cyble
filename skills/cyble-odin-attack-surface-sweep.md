---
name: ODIN attack-surface sweep for a domain
description: >-
  Given a domain, enumerate its subdomains and WHOIS posture, pivot to the hosts and open services
  behind them, and pull the CVEs and known exploits affecting those services — the core external
  attack-surface reconnaissance flow on the ODIN API.
api: openapi/cyble-odin-openapi.yml
base_url: https://api.odin.io/
auth: X-API-Key header
operations:
  - POST /v1/domain/subdomain/count
  - POST /v1/domain/subdomain/search
  - GET /v1/domain/whois/{domain-name}
  - POST /v1/hosts/search
  - GET /v1/hosts/{ip}
  - GET /v1/hosts/cve/{ip}
  - GET /v1/hosts/exploits/{ip}
  - GET /v1/fields/hosts/{category}
---

# ODIN attack-surface sweep for a domain

ODIN is a read-only search index. Every step below is a query — nothing you do here changes state,
and there is no write surface to make idempotent.

## Before you start

- Put your key in the `X-API-Key` request header on every call. There is no OAuth and no bearer
  token. Get a key from the ODIN console under **API and Query Limits**.
- Call over HTTPS only. Plain HTTP requests fail, as do requests missing the key.
- Budget is metered in **credits, not requests**. There are no rate-limit headers to read. You will
  discover exhaustion only when a call returns **402 Payment Required** mid-run — treat 402 as a
  hard stop for the whole job, not as a retryable error on one call.

## Steps

1. **Size the subdomain footprint before you fetch it.**
   `POST /v1/domain/subdomain/count` with `{"domain": "<target>"}`. The response `data.count` tells
   you how many records exist. Use this to decide whether the sweep is affordable before spending
   credits on pages.

2. **Page the subdomains.**
   `POST /v1/domain/subdomain/search` with `{"domain": "<target>", "limit": 100}`.
   Pagination is cursor-based: read `pagination.last` from the response and send it back as `start`
   on the next call. Stop when `pagination.last` is absent or you have reached `pagination.total`.
   Each record carries `domain`, `subdomain` and a parsed `ext_dns_name` (`fld`, `tld`).

3. **Pull the registration posture.**
   `GET /v1/domain/whois/{domain-name}` returns `registrar`, `name_servers`, `domain_status`,
   `created_date`, `expires_date` and the four contact blocks. If you need to see registrar or
   nameserver changes over time, follow with
   `GET /v1/domain/whois/{domain-name}/historical`.

4. **Learn the queryable fields before writing a host query.**
   `GET /v1/fields/hosts/{category}` returns the field registry, including which fields are
   `is_locked` on your plan. Querying a locked or unknown field returns **400**, so read the
   registry rather than guessing field names.

5. **Find the hosts.**
   `POST /v1/hosts/search` with a Lucene query. Match on what you learned above, for example
   `{"query": "domains.name:\"<target>\" AND services.port:443", "limit": 50}`.
   The 400+ queryable fields support ranges (`services.port:[8000 TO 9000]`), booleans, and
   fuzzy/regex matching. Page with `pagination.last` -> `start` exactly as in step 2.

6. **Expand each interesting IP.**
   `GET /v1/hosts/{ip}` returns the full host record: `services[]` (port, protocol, product,
   version, banners, `softwares[]`), `asn`, `location`, `hostnames[]`, `domains[]`, `whois`,
   `tags[]`, and an `is_vuln` flag.

7. **Pull vulnerabilities and exploits.**
   `GET /v1/hosts/cve/{ip}` returns the CVEs on that host with `score`, `severity`,
   `vector_string`, `weakness` and the affected `services`. Then
   `GET /v1/hosts/exploits/{ip}` returns known exploits (`platform`, `type`, `url`, `file`).
   For a single CVE on a single host use `GET /v1/hosts/cves/{ip}/{cve}` and
   `GET /v1/hosts/exploits/{ip}/{cve}`.
   For a paged full CVE dump on one IP use `GET /v1/cves/all/{ip}/{page}` — this operation pages by
   an integer path segment, not by the cursor used elsewhere.

## Handling responses

Every successful response is `{ "success": true, "data": ..., "pagination": {...} }`. Read `data`,
not the top level.

Errors are `{ "success": false, "message": "<free text>" }` served as `application/json`. There is
no error-code registry and no RFC 9457 `problem+json` — you get an HTTP status and a sentence.

| Status | Meaning | What to do |
|---|---|---|
| 400 | Malformed Lucene query, unknown or locked field | Fix the query. Re-read the field registry. Do not retry unchanged. |
| 401 | Missing, disabled, deleted or invalid API key | Stop. Not retryable. Not declared in the OpenAPI, but documented and real. |
| 402 | Credits exhausted or dataset not on your plan | Stop the entire job. Retrying burns nothing but wastes wall time. |
| 408 | Request timed out server-side | Usually a query too broad to complete. Narrow the query or lower `limit`, then retry once. |
| 500 | Server-side fault | Retry once with backoff. If it persists, report it — there is no status page component covering `api.odin.io`. |

## Cautions

- `GET /v1/ping` is the only unauthenticated operation. Use it to check reachability without
  spending a credit.
- Query results describe third-party infrastructure. Enumerating a domain you do not own or have
  authorization to test may be unlawful in your jurisdiction, and ODIN's terms govern use.
