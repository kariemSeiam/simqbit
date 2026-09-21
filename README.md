# SimQbit — Internal SMS Gateway

Private-mode SMS Gateway server, self-hosted for the fleet. Any internal
project (Hvar, gaffer, freelance-venture) can send/receive real SMS
through an Android phone acting as a carrier gateway — no Twilio, no
per-message fees, full control over the data path.

The name: **Sim** (the physical SIM/Android device layer) + **Qbit**
(each message/device treated as a routable, software-controlled unit) —
turning a physical SIM into a programmable primitive for the fleet.

Built on [`android-sms-gateway/server`](https://github.com/android-sms-gateway/server)
(Go, Apache-2.0) — the highest-starred, most actively engineered
open-source Android SMS gateway backend available (see
[Architecture notes](#why-this-fork)). Deploys as two Docker containers:
the gateway server itself and its MariaDB store.

## Status

| Component | State |
|---|---|
| Server | Running, healthy (`docker compose ps`) |
| Database | Migrated, healthy |
| Android device | Not yet registered — no real SMS flows until one is paired |
| Public exposure | None — bound to `127.0.0.1:3900`, internal-only |

## Quick start

```bash
git clone <this-repo-url> simqbit
cd simqbit
cp configs/config.example.yml configs/config.yml
cp .env.example .env   # fill in real secrets, chmod 600 .env
docker compose up -d
curl http://127.0.0.1:3900/api/3rdparty/v1/health/live
```

## Pairing a device

1. Install the APK from
   [`capcom6/android-sms-gateway` releases](https://github.com/capcom6/android-sms-gateway/releases)
   on a spare Android phone with a real SIM.
2. In the app, switch to **Cloud mode** and point it at this server's
   address (reachable from the phone's network — not `127.0.0.1`, use
   the VPS's internal/Tailscale address).
3. Register the device using the `private_token` from `configs/config.yml`.
4. Send a test message via the API (see below) and confirm it lands on
   the phone.

## Sending a message

```bash
curl -X POST http://127.0.0.1:3900/api/3rdparty/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <device-token>" \
  -d '{
    "textMessage": {"text": "Hello from SimQbit"},
    "phoneNumbers": ["+201234567890"]
  }'
```

Full API surface: `/api/3rdparty/v1/{messages,devices,webhooks,inbox,settings,logs}`.
OpenAPI schema served when `http.openapi.enabled: true` (already set in
`configs/config.example.yml`).

## Why this fork

Evaluated against `NdoleStudio/httpsms` and `textbee/textbee` before
choosing `android-sms-gateway`:

- **License** — Apache-2.0, not AGPL-3.0 (httpsms). No obligation to
  open-source anything built on top commercially.
- **Local mode** — the Android app can run a fully offline local server
  with zero cloud dependency, a capability neither competitor has.
- **Architecture** — Kotlin/Ktor on the app side (clean modules: gateway,
  localserver, encryption, incoming, health), Go with clean
  handler→service→repository layering on the server side, Prometheus +
  Grafana dashboards shipped in-repo, per-device rate limiting, JWT with
  token revocation, AES-256-CBC/PBKDF2 end-to-end encryption.
- **Stars/activity** — 5,600+ stars, active commit history with real
  performance work (index optimization, query denormalization), not
  feature-only churn.

## Repository layout

```
simqbit/
├── docker-compose.yml       # server + MariaDB, secrets via .env (gitignored)
├── configs/
│   ├── config.example.yml   # template, secrets redacted
│   └── config.yml           # real config, gitignored
└── .env                     # DB credentials for compose, gitignored, chmod 600
```

## Security notes

- All secrets are generated with `openssl rand`, never hand-typed.
- `docker-compose.yml` references `${DB_ROOT_PASS}`/`${DB_PASS}` only —
  no literal credentials committed, ever.
- Server bound to `127.0.0.1` — not reachable from outside the host
  unless deliberately proxied (e.g. via Tailscale or a reverse proxy
  with its own auth).
