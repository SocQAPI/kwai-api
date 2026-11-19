<!-- Generated from the SocQ API catalog and docs. Do not edit by hand. -->
# Kwai API

[![SocQ Kwai API](https://img.shields.io/badge/API-SocQ%20Kwai%20API-F64C31)](https://socq.ai/apis/kwai?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api)
[![API documentation](https://img.shields.io/badge/Docs-docs.socq.ai-111827)](https://docs.socq.ai/api-manual/kwai/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-2563EB)](LICENSE)
[![Check examples](https://github.com/SocQAPI/kwai-api/actions/workflows/check.yml/badge.svg)](https://github.com/SocQAPI/kwai-api/actions/workflows/check.yml)

3 endpoints for public Kwai data through one SocQ API key, with runnable cURL, Node.js and Python examples and a valid request body for every endpoint.

[Browse Kwai APIs](https://socq.ai/apis/kwai?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api) · [Get an API key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api) · [Documentation](https://docs.socq.ai/api-manual/kwai/overview) · [All SocQ examples](https://github.com/SocQAPI/socq-examples)

## Endpoints

Prices are in SocQ credits; 1 credit = $0.005.

| Endpoint | publicId | What it returns | Price | Launched |
| --- | --- | --- | --- | --- |
| [Kwai Profile API](#kwai-profile-api) | `kwai/profile` | Retrieve one public Kwai profile by username or supported profile URL with normalized identity, profile imagery, account flags, visible counts, and collection context. | 0.19 credits ($0.00095) per call | 2026-08-15 |
| [Kwai User Posts API](#kwai-user-posts-api) | `kwai/user-posts` | Collect one page of public Kwai posts from a username or supported profile URL, returning text, author, media, hashtags, timestamps, and visible engagement as normalized records. | 0.19 credits ($0.00095) per call | 2026-08-14 |
| [Kwai Post API](#kwai-post-api) | `kwai/post` | Retrieve one public Kwai post from a known kwai.com or kwai-video.com URL, returning normalized text, author, media, timestamps, and visible engagement. | 0.19 credits ($0.00095) per call | 2026-08-14 |

## Quick start

1. Create an API key at [socq.ai/dashboard/api-key](https://socq.ai/dashboard/api-key?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api).
2. Export it. The examples read only `SOCQ_API_KEY`.

```bash
git clone https://github.com/SocQAPI/kwai-api.git
cd kwai-api
export SOCQ_API_KEY="your-api-key"
```

3. Run any endpoint by its publicId. Without a second argument the script sends `payloads/<slug>.json`; pass a JSON file or an inline JSON object to change the input.

```bash
# Node.js 20+
node examples/node/socq.mjs kwai/profile

# Python 3.8+ (standard library only)
python3 examples/python/socq.py kwai/profile payloads/profile.json

# cURL + jq
bash examples/curl/socq.sh kwai/profile '{"url":"https://www.kwai.com/@ShortShortz8"}'
```

Each script submits the task, polls it until it succeeds or fails, follows every result page, and writes the records to `output/<slug>-<task_id>.json`. Run `npm run check` to validate the payloads and scripts offline.

Never expose `SOCQ_API_KEY` in browser code, mobile apps, public repositories, screenshots, fixtures or logs.

## How the API works

Base URL `https://api.socq.ai`. Every request sends the API key as a bearer token:

```http
Authorization: Bearer <SOCQ_API_KEY>
Content-Type: application/json
```

All endpoints are asynchronous:

```text
POST /v1/<publicId>  ->  data.task_id  ->  GET /v1/tasks/{task_id}  ->  queued | running | succeeded | failed
```

1. **Submit** the JSON body to `POST https://api.socq.ai/v1/<publicId>`. The response returns `data.task_id` immediately; it means the task was created, not that data is ready.
2. **Poll** `GET https://api.socq.ai/v1/tasks/{task_id}` until `data.status` is `succeeded` or `failed` (a failed task carries `data.error_message`).
3. **Read results.** A succeeded task returns normalized records in `data.results.items`. Use `limit` (default 50, max 500) and pass `data.results.next_cursor` as `cursor` while `data.results.has_more` is true. `fields` selects up to 50 comma-separated fields or dot paths.
4. **Raw files (optional).** `GET https://api.socq.ai/v1/tasks/{task_id}/files` lists raw JSONL exports.

```bash
curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100" \
  -H "Authorization: Bearer $SOCQ_API_KEY"

curl "https://api.socq.ai/v1/tasks/$TASK_ID?limit=100&cursor=$NEXT_CURSOR" \
  -H "Authorization: Bearer $SOCQ_API_KEY"
```

`callback_url` (where an endpoint accepts it) makes SocQ POST to your URL when the task reaches a terminal state; keep polling as a fallback.

**Billing.** A submit may reserve credits from `results_limit` or the number of submitted inputs; when the task finishes SocQ settles the charge from the actual result count and returns unused reserved credits. `402` means insufficient credits.

**Errors.** `400` invalid body or unsupported field, `401` missing or invalid key, `402` insufficient credits, `403` IP allowlist or task owned by another account, `404` endpoint or task not found, `422` input validation failed, `429` rate or credit-window limit (respect `Retry-After`), `500` submission failed. A client-side timeout does not cancel a task: resume polling the same `task_id` instead of submitting again.

Reference: [Authentication](https://docs.socq.ai/api-manual/agent-api/authentication) · [Task status and results](https://docs.socq.ai/api-manual/agent-api/status) · [Task files](https://docs.socq.ai/api-manual/agent-api/files) · [Callbacks](https://docs.socq.ai/api-manual/agent-api/callbacks) · [Errors](https://docs.socq.ai/api-manual/agent-api/errors)

## Endpoint reference

### Kwai Profile API

Retrieve one public Kwai profile by username or supported profile URL with normalized identity, profile imagery, account flags, visible counts, and collection context.

- **Request:** `POST https://api.socq.ai/v1/kwai/profile`
- **publicId:** `kwai/profile`
- **Price:** 0.19 credits ($0.00095) per call
- **Launched:** 2026-08-15
- **Example body:** [`payloads/profile.json`](payloads/profile.json)

Fetch normalized profiles data from Kwai.

Successful tasks are billed per task.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | One of* | A non-empty profile reference whose parsed hostname is kwai.com, kwai-video.com, or a subdomain of either host. Any scheme is accepted when an authority and hostname are present; protocol-relative references are also accepted. |
| `username` | string | One of* | A Kwai username with one optional leading @ followed by 1 to 64 ASCII letters, digits, periods, underscores, or hyphens, for a maximum total length of 65 characters. (length 1 to 65; pattern `^@?[A-Za-z0-9._-]{1,64}$`) |

*Required rule: send at least one of `url`, `username`.

```bash
curl -X POST "https://api.socq.ai/v1/kwai/profile" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://www.kwai.com/@ShortShortz8"
}'
```

Full docs: [https://docs.socq.ai/api-manual/kwai/profile](https://docs.socq.ai/api-manual/kwai/profile) · Landing page: [socq.ai/apis/kwai/profile](https://socq.ai/apis/kwai/profile?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api)

### Kwai User Posts API

Collect one page of public Kwai posts from a username or supported profile URL, returning text, author, media, hashtags, timestamps, and visible engagement as normalized records.

- **Request:** `POST https://api.socq.ai/v1/kwai/user-posts`
- **publicId:** `kwai/user-posts`
- **Price:** 0.19 credits ($0.00095) per call
- **Launched:** 2026-08-14
- **Example body:** [`payloads/user-posts.json`](payloads/user-posts.json)

Fetch normalized user posts data from Kwai.

Successful tasks are billed per task.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | One of* | A non-empty profile reference whose parsed hostname is kwai.com, kwai-video.com, or a subdomain of either host. Any scheme is accepted when an authority and hostname are present; protocol-relative references are also accepted. |
| `username` | string | One of* | A Kwai username with one optional leading @ followed by 1 to 64 ASCII letters, digits, periods, underscores, or hyphens, for a maximum total length of 65 characters. (length 1 to 65; pattern `^@?[A-Za-z0-9._-]{1,64}$`) |

*Required rule: send at least one of `url`, `username`.

```bash
curl -X POST "https://api.socq.ai/v1/kwai/user-posts" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://www.kwai.com/@ShortShortz8"
}'
```

Full docs: [https://docs.socq.ai/api-manual/kwai/user-posts](https://docs.socq.ai/api-manual/kwai/user-posts) · Landing page: [socq.ai/apis/kwai/user-posts](https://socq.ai/apis/kwai/user-posts?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api)

### Kwai Post API

Retrieve one public Kwai post from a known kwai.com or kwai-video.com URL, returning normalized text, author, media, timestamps, and visible engagement.

- **Request:** `POST https://api.socq.ai/v1/kwai/post`
- **publicId:** `kwai/post`
- **Price:** 0.19 credits ($0.00095) per call
- **Launched:** 2026-08-14
- **Example body:** [`payloads/post.json`](payloads/post.json)

Fetch normalized posts data from Kwai.

Successful tasks are billed per task.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | Yes | A non-empty post reference whose parsed hostname is kwai.com, kwai-video.com, or a subdomain of either host. Any scheme is accepted when an authority and hostname are present; protocol-relative references are also accepted. |

```bash
curl -X POST "https://api.socq.ai/v1/kwai/post" \
  -H "Authorization: Bearer $SOCQ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://www.kwai.com/@ShortShortz8/photo/5235021727595785285"
}'
```

Full docs: [https://docs.socq.ai/api-manual/kwai/post](https://docs.socq.ai/api-manual/kwai/post) · Landing page: [socq.ai/apis/kwai/post](https://socq.ai/apis/kwai/post?utm_source=github&utm_medium=readme&utm_campaign=socq-kwai-api)

## Repository contents

| Path | Purpose |
| --- | --- |
| [`examples/curl/socq.sh`](examples/curl/socq.sh) | Submit, poll and paginate with cURL and jq |
| [`examples/node/socq.mjs`](examples/node/socq.mjs) | Same workflow in Node.js 20+ (no dependencies) |
| [`examples/python/socq.py`](examples/python/socq.py) | Same workflow in Python 3.8+ (standard library) |
| [`payloads/`](payloads) | One documented request body per endpoint |
| [`endpoints.json`](endpoints.json) | Endpoint list, prices and request schemas used by the scripts and checks |
| [`fixtures/`](fixtures) | Synthetic submit and task response shapes |
| [`docs/production-notes.md`](docs/production-notes.md) | Retries, timeouts, pagination and data handling |
| [`scripts/check.mjs`](scripts/check.mjs) | Offline check: payloads, syntax and secrets |

## Responsible use and platform scope

- Use only publicly accessible Kwai data and the fields supported by each endpoint.
- Do not use the examples to access restricted content, login-only surfaces, authentication controls or non-public personal information.
- SocQ is not an official API of the represented platform and is not affiliated with or endorsed by it. Platform names and trademarks belong to their respective owners.
- Before production use, assess the laws, platform terms, privacy obligations and retention requirements that apply to your organization and use case.
- Collect only the fields needed for a defined purpose, restrict access, set retention periods, and support correction or deletion workflows where required.

This section describes the public-data boundary; it is not legal advice.

## License

[MIT](LICENSE)
