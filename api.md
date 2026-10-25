# API Reference

Base URL: `https://kirkova.cc/v1`.

There are two separate audiences for this API:

- **Public endpoints** (`/public/*`) — no key required, IP rate-limited, used by the marketing
  website. Read-only and lower-detail than the authenticated versions.
- **Authenticated endpoints** — everything else. Require a bearer JWT obtained by exchanging an
  API key (see [Authentication](#authentication)). Scopes on the key determine which endpoints
  and fields you can reach.

All responses are JSON. Timestamps are ISO-8601 UTC strings.

## Authentication

There are two ways to end up with a JWT, depending on who you are.

### 1. API consumers (you have a long-lived API key)

Exchange your API key for a short-lived JWT, then send that JWT as a bearer token on every
request:

```bash
curl -X POST https://kirkova.cc/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk_live_..."}'
# => {"access_token": "eyJ...", "token_type": "bearer", "expires_in": 900}

curl https://kirkova.cc/v1/games/upcoming?sport=nba \
  -H "Authorization: Bearer eyJ..."
```

The JWT expires quickly (see `expires_in`, seconds) — request a new one when it does rather than
caching it long-term. Tokens carry the scopes baked into your API key at the time of exchange.

### 2. Website accounts (email/password, for the account dashboard and billing)

```bash
curl -X POST https://kirkova.cc/v1/account/signup \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "at-least-8-chars"}'
# => {"access_token": "eyJ...", "token_type": "bearer", "expires_in": ...}
```

Signup auto-provisions one API key for the account, scoped to the `free` tier. Use
`POST /v1/account/login` on subsequent visits. This session token is a different `type` of JWT
than the API-key exchange above and only works on `/account/*` and `/billing/*` routes — use it to
manage your account, not to call prediction endpoints. For programmatic access, pull your
provisioned key from `GET /v1/account/me` and use flow #1.

### Scopes and tiers

| Tier | Scopes granted |
|------|-----------------|
| `free` | `predictions:read`, `models:read` |
| `pro`  | `predictions:read`, `models:read`, `props:read` |

Upgrading via Stripe checkout flips your tier and updates your existing API key's scopes in
place — no new key issued, and no need to re-authenticate for the change to take effect on
`/account/*` calls (tier is re-read from the database on every request). A JWT obtained via
`/auth/token` before the upgrade still reflects the old scopes until you exchange it again.

## Rate limits

Every route is limited per client IP:

- Public endpoints (`/public/*`): 30 requests/minute.
- Authenticated endpoints: subject to the account-wide default limit (see response headers
  `X-RateLimit-Limit` / `X-RateLimit-Remaining` on any response).

Exceeding a limit returns `429 Too Many Requests` with a `Retry-After` header.

## Errors

Errors are returned as:

```json
{ "detail": "human-readable message" }
```

Common status codes: `401` (missing/invalid/expired token), `403` (valid token, missing scope),
`404` (not found), `409` (conflict, e.g. duplicate signup email), `422` (validation error), `429`
(rate limited).

## Endpoints

### Public (no auth, IP rate-limited)

| Method & path | Description |
|---|---|
| `GET /public/sports` | List supported sports (`nba`, `mlb`). |
| `GET /public/games/upcoming?sport=nba\|mlb&hours=48` | Upcoming games in the next `hours` (1–168). |
| `GET /public/predictions/game/{game_id}` | Game winner probability from the active model. |
| `GET /public/models/active?sport=nba\|mlb` | Active model versions, metrics, and feature spec hash. |

### Games (requires `predictions:read`)

| Method & path | Description |
|---|---|
| `GET /games/upcoming?sport=nba\|mlb&hours=48` | Upcoming games with full team/venue detail. |
| `GET /games/{game_id}` | Single game detail. |

### Predictions

| Method & path | Scope | Description |
|---|---|---|
| `GET /predictions/game/{game_id}` | `predictions:read` | Game winner probability. |
| `GET /predictions/props/{game_id}` | `props:read` (Pro) | All player prop predictions for a game. |
| `GET /predictions/player/{player_id}?game_id=` | `props:read` (Pro) | Props for one player in one game. |

### Models (requires `models:read` / `predictions:read`)

| Method & path | Scope | Description |
|---|---|---|
| `GET /sports` | `predictions:read` | List supported sports. |
| `GET /models/active?sport=nba\|mlb` | `models:read` | Active model versions and metrics. |

### Account

| Method & path | Auth | Description |
|---|---|---|
| `POST /account/signup` | — | Create a website account; auto-provisions a free-tier API key. |
| `POST /account/login` | — | Exchange email/password for a session token. |
| `GET /account/me` | session token | Email, tier, masked API key, recent request log. |
| `POST /account/api-key/rotate` | session token | Revoke the current key and issue a new one for the account's tier. |

### Billing

| Method & path | Auth | Description |
|---|---|---|
| `POST /billing/checkout` | session token | Create a Stripe Checkout session for the Pro plan. |
| `POST /billing/portal` | session token | Create a Stripe Billing Portal session (manage/cancel). |
| `POST /billing/webhook` | Stripe signature | Webhook receiver — updates tier + API key scopes. Not for direct use. |

### Health

| Method & path | Description |
|---|---|
| `GET /health` | Liveness/readiness — Postgres, Redis, data-ingest freshness, model freshness. |

## Example: full flow with an API key

```bash
# 1. Exchange your key for a token
TOKEN=$(curl -s -X POST https://kirkova.cc/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"api_key": "sk_live_..."}' | jq -r .access_token)

# 2. Call an endpoint
curl https://kirkova.cc/v1/predictions/game/12345 \
  -H "Authorization: Bearer $TOKEN"
```

```python
import httpx

api_key = "sk_live_..."
base = "https://kirkova.cc/v1"

token = httpx.post(f"{base}/auth/token", json={"api_key": api_key}).json()["access_token"]
headers = {"Authorization": f"Bearer {token}"}

games = httpx.get(f"{base}/games/upcoming", params={"sport": "nba"}, headers=headers).json()
pred = httpx.get(f"{base}/predictions/game/{games[0]['id']}", headers=headers).json()
print(pred["home_win_probability"])
```
