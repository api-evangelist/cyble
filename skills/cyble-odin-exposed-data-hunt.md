---
name: ODIN exposed bucket and file hunt
description: >-
  Find publicly exposed cloud storage buckets for an organization and drill into the sensitive files
  inside them, using ODIN's AI/ML labelling to prioritise credential, PII and financial exposure
  ahead of everything else.
api: openapi/cyble-odin-openapi.yml
base_url: https://api.odin.io/
auth: X-API-Key header
operations:
  - GET /v1/fields/exposed/buckets
  - POST /v1/exposed/buckets/count
  - POST /v1/exposed/buckets/summary
  - POST /v1/exposed/buckets/search
  - GET /v1/fields/exposed/files
  - POST /v1/exposed/files/count
  - POST /v1/exposed/files/summary
  - POST /v1/exposed/files/search
---

# ODIN exposed bucket and file hunt

`searchExposedBuckets` is the only operation in the published ODIN OpenAPI that carries an
`operationId`. Every other operation is addressed by method and path.

## Before you start

- `X-API-Key` header on every request, HTTPS only.
- Credits, not request rate. A **402** anywhere means the job is over.
- Buckets and files are two separate datasets joined by **bucket name** — `exposed.File.bucket`
  carries `exposed.Bucket.name`. There is no numeric foreign key.

## Vocabulary

These are the published enumerations. Do not invent values outside them.

- **Providers**: `aws`, `gcp`, `do`, `linode`, and others listed in the field reference.
- **Bucket/file labels**: `credential`, `financial`, `pii`, `legal`, `ip`, `medical`, `hr`,
  `report`, `confidential`, `backup`, `compromised`, `vulnerable`.
- **Bucket file categories**: `img`, `aud`, `vid`, `font`, `doc`, `src`, `web`, `bkup`, `dbdump`.
- **File categories**: `img`, `aud`, `vid`, `font`, `txt`, `doc`, `src`, `db`, `march`, `arch`,
  `3d`, `exec`, `key`, `cert`.

## Steps

1. **Read the field registry first.**
   `GET /v1/fields/exposed/buckets` and `GET /v1/fields/exposed/files`. Each returns
   `{name, category, display_category, is_locked}`. `is_locked` tells you which fields your plan
   cannot query — hitting one returns 400, not an empty result.

2. **Shape the query before you spend on it.**
   `POST /v1/exposed/buckets/count` with `{"query": "<lucene>"}` returns just a count.
   `POST /v1/exposed/buckets/summary` with `{"query": "<lucene>", "field": "provider", "limit": 20}`
   returns an aggregate breakdown by any field — use it to see where the exposure concentrates
   before paging results.

3. **Search buckets, highest signal first.**
   `POST /v1/exposed/buckets/search` (`operationId: searchExposedBuckets`) with:
   ```json
   {
     "query": "labels:credential AND provider:aws AND open:true",
     "limit": 50,
     "sort_by": "ins_at",
     "sort_dir": "desc"
   }
   ```
   `sort_by` and `sort_dir` are available on this dataset. Each `exposed.Bucket` carries `name`,
   `provider`, `region`, `owner_id`, `open`, `deleted`, `labels[]`, `categories[]`, `files`,
   `new_files`, `sensitive_files`, `file_cat_count`, `age_in_days` and the scan timestamps
   (`ins_at`, `scan_at`, `prev_scan_at`, `last_found_at`, `last_open_at`).
   Prioritise on `sensitive_files` and `labels`, not on raw `files`.

4. **Page with the cursor.**
   Read `pagination.last` and send it back as `start` on the next call — it is an array of sort-key
   values, treat it as opaque. `pagination.total` bounds the run.

5. **Drill into the files of a bucket that matters.**
   `POST /v1/exposed/files/search` with `{"query": "bucket:\"<bucket-name>\" AND sensitive:true", "limit": 50, "sort_by": "mod_at", "sort_dir": "desc"}`.
   Each `exposed.File` carries `name`, `path`, `bucket`, `provider`, `region`, `url`, `ext`,
   `ext_desc`, `category`, `label`, `type`, `size`, `etag`, `accessible`, `sensitive` and the
   `ins_at` / `mod_at` / `scan_at` timestamps.
   Use `POST /v1/exposed/files/count` and `POST /v1/exposed/files/summary` the same way as step 2.

## Handling responses

Envelope is `{ "success": true, "data": ..., "pagination": {...} }`. Errors are
`{ "success": false, "message": "..." }`.

- **400** — bad Lucene, unknown field, or a value outside the published enumerations. Fix, do not retry.
- **402** — credits exhausted. Stop the job.
- **408** — query too broad. Narrow it or lower `limit`.
- **500** — retry once with backoff.

There are no rate-limit headers and no `Retry-After`. Pace your own calls.

## Cautions

- `url` on an `exposed.File` points at live third-party data. Recording that a file is exposed is
  reconnaissance; retrieving its contents may not be lawful. Read the label, do not fetch the object.
- `accessible` and `open` are point-in-time scan observations, not a live check. Re-scan timestamps
  are on every record — check `scan_at` before acting on a finding.
