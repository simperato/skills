---
name: strava
description: >-
  Access the Strava API v3 to list the authenticated athlete's uploaded
  activities (/athlete/activities) and download full details of a specific
  activity (/activities/{id}), managing OAuth 2.0 tokens via personal_memories
  and strictly minimizing API calls to respect Strava's low rate limits.
---

# Strava API Skill

Access Strava API v3 for two core read operations:
1. **List athlete activities**: `GET https://www.strava.com/api/v3/athlete/activities`
2. **Get full activity details**: `GET https://www.strava.com/api/v3/activities/{id}`

References:
- Authentication: https://developers.strava.com/docs/authentication/
- API Reference: https://developers.strava.com/docs/reference/

---

## 1. Credential Storage (`personal_memories`)

All Strava OAuth credentials, client IDs, and secrets MUST be stored and retrieved using the **`personal_memories`** skill (`personal_memories/SKILL.md`), never hardcoded in files or left in plain workspace scripts.

- **Subject (`soggetto`)**: The athlete/user's subject folder inside `Personal Memories/` (confirm the subject name with the user if not already known).
- **Topic (`argomento`)**: Use a dedicated, unambiguous topic name **`Strava OAuth`** so Strava credentials are never confused with OAuth keys for other services.
- **Security**: Strictly enforce `personal_memories` Restricted permissions (private, no `anyone`/`domain` access). Never print full `client_secret`, `access_token`, or `refresh_token` values in chat responses.

### Fields to store in `Strava OAuth`
Keep each entry clearly labeled as **Strava**:
- `Strava Client ID`: `<client_id>`
- `Strava Client Secret`: `<client_secret>`
- `Strava Access Token`: `<access_token>`
- `Strava Refresh Token`: `<refresh_token>`
- `Strava Token Expires At`: `<unix_epoch_seconds>` (and optional ISO-8601 UTC note)
- `Strava Scope`: `<e.g. read,activity:read_all>`
- `Strava Athlete ID` (optional): `<athlete_id>`

---

## 2. OAuth 2.0 Authentication & Token Lifecycle

### A. Initial Authorization (one-time setup if no tokens exist)
If `Strava Refresh Token` is not yet stored in `personal_memories`:
1. Retrieve (or ask the user for) `Strava Client ID` and `Strava Client Secret`, and store them in `personal_memories` under topic `Strava OAuth`.
2. Provide the user with the Strava authorization URL (default `redirect_uri` for local/manual setup is typically `http://localhost` or the Callback Domain configured in the user's Strava API Application):
   ```text
   https://www.strava.com/oauth/authorize?client_id=<CLIENT_ID>&redirect_uri=<REDIRECT_URI>&response_type=code&approval_prompt=auto&scope=read,activity:read_all
   ```
   > **Scope note**: `activity:read_all` is required to read **private** activities as well as public ones (`activity:read` omits private activities).
3. Ask the user to authorize in their browser and paste the `code` query parameter from the redirect URL.
4. Exchange the authorization `code` for tokens:
   ```bash
   curl -sS -X POST https://www.strava.com/oauth/token \
     -d client_id="<CLIENT_ID>" \
     -d client_secret="<CLIENT_SECRET>" \
     -d code="<AUTHORIZATION_CODE>" \
     -d grant_type=authorization_code
   ```
5. Save `access_token`, `refresh_token`, `expires_at`, `scope`, and `athlete.id` into `personal_memories` (`Strava OAuth`).

### B. Token Check & Refresh (before API calls)
Strava `access_token` values expire every **6 hours**.
1. Read `Strava Access Token`, `Strava Refresh Token`, `Strava Token Expires At`, `Strava Client ID`, and `Strava Client Secret` from `personal_memories` (`Strava OAuth`).
2. Compare the current Unix timestamp against `expires_at`.
3. **Only refresh when needed** (`now >= expires_at - 60`), to avoid unnecessary token requests:
   ```bash
   curl -sS -X POST https://www.strava.com/oauth/token \
     -d client_id="<CLIENT_ID>" \
     -d client_secret="<CLIENT_SECRET>" \
     -d grant_type=refresh_token \
     -d refresh_token="<REFRESH_TOKEN>"
   ```
4. **Always persist rotated tokens**: Strava may issue a new `refresh_token` on refresh. Immediately update `Strava Access Token`, `Strava Refresh Token`, and `Strava Token Expires At` in `personal_memories` (`Strava OAuth`).

---

## 3. Rate Limits & Frugal API Usage (Mandatory)

Strava enforces strict per-application rate limits:
- **Read (non-upload) limit**: **100 requests per 15 minutes** and **1,000 requests per day** (resets at `:00`, `:15`, `:30`, `:45` past each hour, and midnight UTC daily).
- **Overall limit**: **200 requests per 15 minutes** and **2,000 requests per day**.

### Rules to minimize API calls
1. **Always maximize page size**: Pass `per_page=200` (the maximum allowed) on `/athlete/activities` so each request fetches up to 200 activities instead of the default 30.
2. **Use time bounds (`after` / `before`)**: When the user asks for recent activities or a specific date range, pass Unix epoch timestamps in `after` and/or `before` instead of paging through the entire history.
3. **Do NOT fetch `/activities/{id}` unless needed**: `/athlete/activities` already returns summary fields (`id`, `name`, `sport_type`, `start_date`, `distance`, `moving_time`, `elapsed_time`, `total_elevation_gain`, `average_speed`, `max_speed`, `average_heartrate`, `max_heartrate`, `map.summary_polyline`, etc.). Only call `/activities/{id}` for specific activity IDs when full details (e.g., description, calories, gear, splits, laps, segment efforts) are requested.
4. **Never batch `/activities/{id}` blindly**: If a user asks for details on many activities, check how many calls it will consume first and confirm with the user before making a large batch of requests.
5. **Cache within the session**: Save fetched JSON responses to a temporary/local file (when appropriate) during a multi-step analysis so you never re-request the same page or activity ID twice.
6. **Monitor rate-limit headers**: Include `-i` or `-D -` in `curl` (or inspect headers in Python) and check:
   - `X-ReadRateLimit-Usage` / `X-ReadRateLimit-Limit`
   - `X-RateLimit-Usage` / `X-RateLimit-Limit`
7. **Stop on `429 Too Many Requests`**: Never retry in a tight loop. Report the limit/usage headers and wait until the next 15-minute reset window (`:00`, `:15`, `:30`, `:45`).

---

## 4. Supported Operations

### Operation 1: List Uploaded Activities (`GET /athlete/activities`)
Returns a list of `SummaryActivity` objects for the authenticated athlete, ordered newest-first (by default).

**Endpoint**:
```text
GET https://www.strava.com/api/v3/athlete/activities
```

**Query parameters**:
- `before` (integer, optional): Unix epoch timestamp to filter activities that took place **before** this time.
- `after` (integer, optional): Unix epoch timestamp to filter activities that took place **after** this time.
- `page` (integer, optional): Page number (starts at `1`).
- `per_page` (integer, optional): Items per page (default `30`, **always set `per_page=200`** when listing all or many activities).

**Example**:
```bash
curl -sS -D /tmp/strava_headers.txt \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  "https://www.strava.com/api/v3/athlete/activities?per_page=200&page=1"
```

**Pagination strategy for "all activities"**:
1. Start at `page=1` with `per_page=200` (and `after`/`before` if bounded).
2. Stop immediately as soon as a page returns fewer than `200` items (or `[]`), avoiding an extra empty-page request whenever `len(items) < 200`.
3. Check `X-ReadRateLimit-Usage` after each page.

---

### Operation 2: Download Full Activity Detail (`GET /activities/{id}`)
Returns a `DetailedActivity` object for a single activity.

**Endpoint**:
```text
GET https://www.strava.com/api/v3/activities/{id}
```

**Parameters**:
- `id` (integer, required, path): The activity ID.
- `include_all_efforts` (boolean, optional): Set to `true` only if all segment efforts are explicitly needed (default `false` keeps payload smaller).

**Example**:
```bash
curl -sS -D /tmp/strava_headers.txt \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  "https://www.strava.com/api/v3/activities/<ACTIVITY_ID>"
```

---

## 5. Verification Checklist

Before completing a Strava API task:
- [ ] Credentials were read from / updated in `personal_memories` under the `Strava OAuth` topic (and Restricted permissions were preserved).
- [ ] If the token was refreshed, the new `access_token`, `refresh_token`, and `expires_at` were saved back to `personal_memories`.
- [ ] `per_page=200` (and `after`/`before` when applicable) was used on `/athlete/activities` to minimize API calls.
- [ ] Rate-limit headers (`X-ReadRateLimit-Usage` / `X-RateLimit-Usage`) were checked and respected.
- [ ] No raw tokens or `client_secret` were exposed in the final response.
