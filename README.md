<div align="center">

# SimQbit

**Internal SMS Gateway — turn a real Android SIM into a programmable primitive for the fleet.**

[![CI](https://github.com/kariemSeiam/simqbit/actions/workflows/ci.yml/badge.svg)](https://github.com/kariemSeiam/simqbit/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Docker](https://img.shields.io/badge/docker-compose-2496ED?logo=docker&logoColor=white)](docker-compose.yml)
[![Go](https://img.shields.io/badge/go-1.25%2B-00ADD8?logo=go&logoColor=white)](https://github.com/android-sms-gateway/server)
[![Upstream stars](https://img.shields.io/github/stars/capcom6/android-sms-gateway?label=upstream%20stars&color=5C2D91)](https://github.com/capcom6/android-sms-gateway)

**[Why](#why) · [Quickstart](#quickstart) · [Pairing a device](#pairing-a-device) · [Sending a message](#sending-a-message) · [Architecture](#architecture) · [Comparison](#why-this-fork) · [FAQ](#faq)**

</div>

---

> [!IMPORTANT]
> No Android device is paired yet. The server runs and passes health
> checks, but no real SMS flows until a phone is registered — see
> [Status](#status).

## Why

Any internal project (Hvar, gaffer, freelance-venture) needs to send
OTPs, order confirmations, or alerts by SMS. The usual answer is a
per-message API like Twilio — expensive, and often unavailable for
local numbers outside a handful of supported countries.

| Instead of… | You get… |
|---|---|
| Paying per-message to Twilio/MessageBird | Your own SIM's carrier plan — often flat-rate or unlimited |
| A virtual number that doesn't exist for your country | Your real SIM, your real local number |
| Trusting a third party with message content | Self-hosted, AES-256 end-to-end encryption to the device |
| AGPL-licensed forks that leak into your product's license | Apache-2.0 throughout — no copyleft obligation |
| A managed cloud dependency for a purely internal tool | Bound to `127.0.0.1`, reachable only from the fleet's own network |

The name: **Sim** (the physical SIM/Android device layer) + **Qbit**
(each message/device treated as a routable, software-controlled unit).

## Architecture

```mermaid
flowchart LR
    A[Fleet project<br/>Hvar / gaffer / etc.] -->|POST /messages<br/>private_token| B[SimQbit server<br/>Go + MariaDB]
    B -->|FCM push| C[Android app<br/>Kotlin/Ktor]
    C -->|SmsManager| D[Carrier network]
    D --> E[Recipient phone]
    E -.->|reply SMS| C
    C -.->|webhook| B
    B -.->|delivery status| A
```

Built on [`android-sms-gateway/server`](https://github.com/android-sms-gateway/server)
(Go, Apache-2.0) — the highest-starred, most actively engineered
open-source Android SMS gateway backend available (see
[Why this fork](#why-this-fork)). Deploys as two Docker containers: the
gateway server itself and its MariaDB store.

## Status

| Component | State |
|---|---|
| Server | Running, healthy (`docker compose ps`) |
| Database | Migrated, healthy |
| Android device | Not yet registered — no real SMS flows until one is paired |
| Public exposure | None — bound to `127.0.0.1:3900`, internal-only |

## Quickstart

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
choosing `android-sms-gateway` as the base for this deployment —
researched, not assumed (verified stars/license/topics via the GitHub
API at time of writing):

| | SimQbit (on `android-sms-gateway`) | `NdoleStudio/httpsms` | `textbee/textbee` |
|---|---|---|---|
| License | Apache-2.0 | AGPL-3.0 | MIT |
| Stars (upstream) | 5,600+ | 4,600+ | 3,000+ |
| Local mode (zero cloud dependency) | ✅ | ❌ | ❌ |
| Multi-SIM support | ✅ | ❌ | ✅ (Pro tier) |
| MMS support | ✅ | ❌ | ❌ |
| Rate limiting per device | ✅ | ❌ | ❌ |
| Prometheus/Grafana shipped in-repo | ✅ | ❌ | ❌ |
| JWT with token revocation | ✅ | Basic auth only | API key only |
| End-to-end encryption | AES-256-CBC/PBKDF2 | ❌ | ❌ |
| Server language | Go | Go | Node.js/NestJS |
| App stack | Kotlin/Ktor (modern) | — | Kotlin + Java (mixed legacy) |

<details>
<summary>Why AGPL-3.0 mattered enough to rule out httpsms</summary>

AGPL requires that anyone who runs a modified version of the software
as a network service must publish their modified source. For an
internal fleet tool that might later wrap this in a paid or
client-facing product, that's a real constraint — Apache-2.0 carries
no such obligation. This is a licensing decision, not a quality
judgment on httpsms itself, which is a solid, actively maintained
project.

</details>

## Repository layout

```text
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
- GitHub secret scanning, push protection, and Dependabot security
  updates enabled on this repo.

## FAQ

**Does my carrier limit how many messages I can send this way?**
Yes — this uses `SmsManager` over your real SIM, so normal carrier
rate limits and anti-spam policies apply. It's built for
low-to-moderate internal traffic (OTPs, order confirmations, alerts),
not bulk marketing sends. For high volume, use multiple paired devices.

**What happens if the phone loses connectivity or dies?**
Messages queue server-side (`messages:list`/`messages:cancel` API) and
the health endpoint reports device online/offline state. There's no
automatic failover to a second device yet — pairing a second phone is
supported by the upstream server, but SimQbit's current deployment
targets one device.

**Why not just use Twilio and eat the cost?**
For a project needing a handful of Egyptian numbers reachable at
volume, most virtual-number providers either don't support local
numbers or charge per-message well above local carrier rates. This
trades that recurring cost for one spare phone and a SIM you already
pay for.

**Is this production-hardened, or a prototype?**
The upstream (`android-sms-gateway/server`) is: 5,600+ stars, JWT auth
with token revocation, rate limiting, Prometheus/Grafana shipped
in-repo, active commit history with real performance work. This repo
is the deployment wrapper (Docker Compose + MariaDB + secrets
discipline) around that — it has not yet been used to send a single
real message (see [Status](#status)).

## Acknowledgments

Built on top of [`android-sms-gateway`](https://github.com/capcom6/android-sms-gateway)
(Android app) and [`android-sms-gateway/server`](https://github.com/android-sms-gateway/server)
(backend) by [capcom6](https://github.com/capcom6) and contributors —
both Apache-2.0. This repository packages a private deployment
(Docker Compose + MariaDB) for internal fleet use; it does not modify
their source.

## License

[Apache License 2.0](LICENSE) — same license as the upstream projects
this deployment is built on.
