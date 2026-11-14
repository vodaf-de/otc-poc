# OTC Desk — Private OTC Settlement Platform (POC)

**BTC / USDT → Naira, with on-chain verification and an admin console.**

> Version 0.1.0 — **proof of concept**. This repository implements the functional skeleton of the
> *Private OTC Settlement Platform v1.0* specification (Phase 2, "MVP") in a deliberately compact
> form: a single Flask process, SQLite, and no mandatory external service.
> It is **not** production-ready: see [Limits](#13-limits--out-of-scope).

---

## Table of contents

1. [Overview](#1-overview)
2. [Functional scope](#2-functional-scope)
3. [Architecture](#3-architecture)
4. [Quick start](#4-quick-start)
5. [Demo accounts](#5-demo-accounts)
6. [Demo walkthrough](#6-demo-walkthrough)
7. [Deal lifecycle](#7-deal-lifecycle)
8. [On-chain verification](#8-on-chain-verification)
9. [Security](#9-security)
10. [Tests](#10-tests)
11. [Deployment](#11-deployment)
12. [Configuration](#12-configuration)
13. [Limits & out of scope](#13-limits--out-of-scope)
14. [Roadmap to v1.0](#14-roadmap-to-v10)
15. [Design system](#15-design-system)

---

## 1. Overview

The desk is not an exchange: every transaction is a **deal** negotiated between two
counterparties, which the platform structures, verifies and records.

```
Client                    Desk (admin)                     Chain
  │  creates the deal         │                               │
  ├──────────────────────────>│  deposit instructions         │
  │  sends the crypto         │                               │
  ├──────────────────────────────────────────────────────────>│
  │  submits the txid         │  checks amount / address /    │
  ├──────────────────────────>│  confirmations ──────────────>│
  │                           │  releases NGN (bank reference)│
  │  printable receipt        │  closes + audit trail         │
  │<──────────────────────────┤                               │
```

Three principles are wired into the code:

- **Nothing moves without verification.** The "Release NGN" action is refused (`409`) while the
  status is not `CRYPTO_CONFIRMED`; the check compares amount, destination address and
  confirmation depth against the deal record.
- **The desk stays in control.** Every state transition goes through an identified actor and
  writes an audit row (actor, IP, before/after).
- **Strict role separation.** A client only sees their own deals (403 otherwise), cannot reach
  `/admin/*` (403), and cannot trigger desk-only actions.

## 2. Functional scope

| Area | Implemented in this POC |
|---|---|
| Authentication | Email / password login (Werkzeug hash), Flask-Login sessions, CSRF on every form |
| Roles | `client`, `admin` (desk). `admin` = operator + supervisor for this POC |
| Deals | Created by a client, internal counterparty, terms (asset, network, amount, rate, fee, fiat rail, bank instructions) |
| Lifecycle | 10 states, guarded transitions, per-deal history and message thread |
| On-chain verification | BTC mainnet (Blockstream), USDT TRC-20 / ERC-20 / BEP-20, deterministic demo mode |
| Fiat settlement | Bank reference entry, receipt upload (png/jpg/webp/pdf), move to `FIAT_SENT` |
| Receipts | Printable settlement receipt (print CSS), transfer reference, verification fingerprint |
| Client portal | Dashboard, deals, whitelisted wallets, statements + CSV export |
| Admin console | Dashboard (30-day volume, P&L, exposure, 14-day chart), queue, on-chain console, clients (KYC, limits, risk flag), audit log |
| Audit | Append-only journal of every state change and sensitive action |

**Flow direction covered:** the client **sells crypto** and **receives NGN**. The data model
(`side`) allows the opposite direction, but the UI and the verification engine only implement
this one, to stay within the POC scope.

## 3. Architecture

```
otc-repo/
├── run.py                  # dev entrypoint (flask run / python run.py)
├── wsgi.py                 # gunicorn entrypoint
├── setup.py                # packaging (pip install -e .)
├── Makefile                # install / run / seed / reset
├── requirements.txt
├── deploy/
│   ├── otcdesk.service     # ready-to-use systemd unit
│   └── nginx.conf.example  # reverse proxy + TLS example
├── otc/
│   ├── __init__.py         # create_app, Jinja filters, context processors
│   ├── config.py           # 100% env-driven configuration
│   ├── models.py           # User, Deal, ChainCheck, WalletAddress, Rate, AuditLog…
│   ├── security.py         # csrf, login_manager, admin_required
│   ├── cli.py              # flask seed / reset-db
│   ├── formatting.py       # fmt_ngn, fmt_crypto, timeago
│   ├── blueprints/
│   │   ├── auth.py         # /login, /logout
│   │   ├── portal.py       # client portal
│   │   ├── admin.py        # desk console
│   │   └── api.py          # JSON endpoints (deal status)
│   ├── services/
│   │   ├── chain.py        # BTC / EVM / TRON providers + simulator
│   │   ├── verification.py # check orchestration + persistence
│   │   ├── deals.py        # state machine, quotes, limits
│   │   ├── audit.py        # journaling
│   │   └── files.py        # receipt uploads
│   ├── templates/          # Jinja (portal/, admin/, errors/)
│   └── static/             # app.css (design system), app.js
├── tests/smoke_test.py     # 24 end-to-end HTTP checks
└── var/                    # SQLite + uploads (git-ignored)
```

- **Backend:** Flask 3, Flask-SQLAlchemy, Flask-Login, Gunicorn.
- **Database:** SQLite by default (`var/otc.sqlite3`), overridable through `DATABASE_URL`
  (PostgreSQL works without code changes, once the driver is installed).
- **Frontend:** Jinja + hand-written CSS, no JS dependency. Dark "desk terminal" theme.
- **Realtime:** none (no WebSocket in this POC); status refreshes on reload.

## 4. Quick start

Requirement: Python ≥ 3.11.

```bash
git clone https://github.com/vodaf-de/otc-poc.git otc-poc && cd otc-poc

make install                 # venv + dependencies
.venv/bin/pip install -e .   # installs the local package (setup.py) in editable mode
cp .env.example .env         # then edit SECRET_KEY
make seed                    # creates tables + the demo book

make run                     # http://127.0.0.1:5055
# or, as a server:
.venv/bin/gunicorn -w 1 -b 0.0.0.0:5055 wsgi:app
```

The `-e .` flag installs the package in editable mode, which is required for the local CLI
tools to work correctly. Run it before `make seed` and `make run`.

`SECRET_KEY` must be unique per environment:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Rebuild the demo book: `make seed` (adds `--reset`). Wipe the database: `make reset`.

### Open it in your editor

**VS Code**

```bash
git clone https://github.com/vodaf-de/otc-poc.git && cd otc-poc && code .
```

**Cursor**

```bash
git clone https://github.com/vodaf-de/otc-poc.git && cd otc-poc && cursor .
```

## 5. Demo accounts

| Role | Email | Password | Context |
|---|---|---|---|
| Admin (desk) | `admin@otcdesk.test` | `Admin#2026` | Full access to `/admin` |
| Admin (ops) | `ops@otcdesk.test` | `Ops#2026` | Second operator, same scope |
| Client | `trading@northwind.test` | `Client#2026` | KYC verified, 2 deals |
| Client | `treasury@helios.test` | `Client#2026` | KYC verified, 2 deals |
| Client | `desk@acme.test` | `Client#2026` | KYC **pending** (tests the block) |

The demo book contains 6 deals covering every state (draft, negotiation, crypto pending,
settled, fiat sent, dispute), 4 desk treasury addresses and 2 reference rates.

> These credentials are public seed data and intentionally weak: **do not reuse them** on an
> exposed environment.

## 6. Demo walkthrough

1. **Client** — sign in with `trading@northwind.test`, open *New deal*, pick BTC, enter an
   amount, accept the quoted rate, fill in the bank instructions.
   The deal is created as `CREATED` with an `OTC-YYYYMM-XXXXXX` reference.
2. **Client** — *Send to counterparty* → `NEGOTIATION`.
3. **Admin** — `/admin/queue`: the deal shows up. *Publish instructions* generates the deposit
   address (desk wallet snapshot) and moves the deal to `AWAITING_DEPOSIT`.
4. **Client** — *Submit txid* with a transaction hash. In demo mode the simulator returns a
   deterministic confirmation count and the deal moves to `CRYPTO_PENDING` or
   `CRYPTO_CONFIRMED`.
5. **Admin** — `/admin/verify`: rerun a check, read the detail (amount, destination,
   confirmations). Then *Release NGN* with the transfer reference and, optionally, the bank
   receipt → `FIAT_SENT`. **Without** `CRYPTO_CONFIRMED`, the action returns `409`.
6. **Client** — the printable receipt is available; the statement and CSV export include the row.
7. **Admin** — *Settle* → `SETTLED`. Every step appears in `/admin/audit`.

## 7. Deal lifecycle

```
CREATED ──send──> NEGOTIATION ──accept──> AGREED ──publish──> AWAITING_DEPOSIT
   │                  │                     │                      │
   │                  └──────────┬──────────┴──────────────────────┘
   │                             ▼
   │                    CRYPTO_PENDING ──confirmations──> CRYPTO_CONFIRMED
   │                                                        │
   │                                                   release NGN
   │                                                        ▼
   └──reject/dispute──────────────────────────────────>  FIAT_SENT ──settle──> SETTLED
                                                              │
                                                        DISPUTED / REJECTED
```

| State | UI label | Who acts |
|---|---|---|
| `CREATED` | Draft terms | Client (send) |
| `NEGOTIATION` | Negotiating | Counterparty (accept) |
| `AGREED` | Terms agreed | Desk (publish instructions) |
| `AWAITING_DEPOSIT` | Awaiting crypto | Client (deposit + txid) |
| `CRYPTO_PENDING` | Crypto unconfirmed | Desk (recheck) / client (new txid) |
| `CRYPTO_CONFIRMED` | Crypto confirmed | Desk (release NGN) |
| `FIAT_SENT` | Fiat sent | Desk (close) |
| `SETTLED` | Settled | — (archive) |
| `DISPUTED` / `REJECTED` | — | Desk (arbitration) |

Allowed transitions are centralised in `otc/services/deals.py` (`can_transition`), not in
templates.

## 8. On-chain verification

| Network | Provider | Required confirmations (default) |
|---|---|---|
| Bitcoin mainnet | Blockstream API (`/tx/:txid`, `/blocks/tip/height`) | 2 |
| USDT TRC-20 | Tronscan / TronGrid (`TRON_API_KEY`) | 19 |
| USDT ERC-20 | Etherscan-compatible (`ETHERSCAN_API_KEY`) | 12 |
| USDT BEP-20 | Etherscan-compatible (BSC) | 15 |

Each check compares:

- **the amount** transferred against the deal's expected amount (dust tolerance),
- **the destination address** against the deposit address frozen on the deal,
- **the confirmation depth** against the network threshold.

Possible results: `confirmed`, `pending`, `not_found`, `mismatch`, `error`. Every attempt is
stored in `chain_checks` (provider, raw JSON, timestamp): the history is visible in
`/admin/verify` and quoted on the receipt.

### Demo mode

`CHAIN_DEMO=true` (default) enables `demo-simulator`: no network request, deterministic result
(`sha256(network:txid)`) and **always flagged `simulated`** in the UI ("simulated" badge, note on
the receipt). This is the mode used for the demonstration.

To switch to real verification:

```dotenv
CHAIN_DEMO=false
BLOCKSTREAM_URL=https://blockstream.info/api
ETHERSCAN_API_KEY=...
TRON_API_KEY=...
```

Known limits: no RBF / double-spend detection, no multi-provider fallback, no automatic polling
(checks are triggered manually), no token support beyond USDT.

## 9. Security

In place in the POC:

- hashed passwords (Werkzeug `pbkdf2`), signed HttpOnly sessions, `SameSite=Lax`;
- **CSRF**: a session token is required on every POST (form field or `X-CSRF-Token` header);
- **RBAC**: `admin_required`, deal ownership checked on every client route (403);
- **append-only audit log**: actor, IP, entity, before/after values;
- receipt uploads limited (4 MB, png/jpg/jpeg/webp/pdf, randomised filename);
- per-client monthly limit and risk flag enforced at deal creation;
- basic security headers and dedicated error pages.

Deliberately absent (see §13): 2FA, IP allow-listing, encryption at rest, secret rotation,
rate limiting, account lockout, signed receipts, ops/supervisor role split.

## 10. Tests

The HTTP smoke test covers 24 checks: login, cross-user RBAC, API blocking, deal creation,
verification → `CRYPTO_PENDING`, release guard (`409`), fiat release → `FIAT_SENT`, receipt,
close → `SETTLED`.

```bash
.venv/bin/python tests/smoke_test.py                     # http://127.0.0.1:5055
.venv/bin/python tests/smoke_test.py http://host:5055    # remote target
```

The test creates its own deals: run it against a demo database, never against real data.

## 11. Deployment

Reference target: Ubuntu 24.04, systemd service, Gunicorn, Nginx in front.

```bash
# on the server (as a sudo-capable user)
sudo apt install -y python3-venv git
sudo useradd --system --home-dir /opt/otc-desk --shell /usr/sbin/nologin otcdesk
sudo git clone <repo-url> /opt/otc-desk
sudo chown -R otcdesk:otcdesk /opt/otc-desk
cd /opt/otc-desk

sudo -u otcdesk python3 -m venv .venv
sudo -u otcdesk .venv/bin/pip install -U pip
sudo -u otcdesk .venv/bin/pip install -r requirements.txt

sudo -u otcdesk cp .env.example .env
sudo -u otcdesk sed -i "s|^SECRET_KEY=.*|SECRET_KEY=$(openssl rand -hex 32)|" .env
sudo -u otcdesk .venv/bin/flask --app run.py seed

sudo cp deploy/otcdesk.service /etc/systemd/system/otcdesk.service
sudo systemctl daemon-reload
sudo systemctl enable --now otcdesk
systemctl status otcdesk --no-pager
```

The provided unit listens on `0.0.0.0:5055` with 1 worker / 4 threads (SQLite = a single writer).
To expose the service properly, put Nginx + TLS in front — see
`deploy/nginx.conf.example`:

```nginx
server {
    listen 80;
    server_name desk.example.com;

    location / {
        proxy_pass http://127.0.0.1:5055;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then set `SESSION_COOKIE_SECURE=true` in `.env` and restart the service. With a reverse proxy,
you can also bind Gunicorn to `127.0.0.1:5055` only.

Backup: `var/otc.sqlite3` + `var/uploads/` (both are needed to restore a receipt).

## 12. Configuration

All variables are read from the environment (`.env` is loaded by python-dotenv).

| Variable | Default | Purpose |
|---|---|---|
| `SECRET_KEY` | dev key | Session and CSRF token signing — **must be changed** |
| `DATABASE_URL` | `sqlite:///var/otc.sqlite3` | SQLAlchemy connection |
| `OTC_BRAND` | `OTC Desk` | Displayed brand name |
| `OTC_ENV_NAME` | `POC` | Environment badge |
| `FEE_BPS` | `60` | Desk fee in basis points (0.60%) |
| `CHAIN_DEMO` | `true` | Deterministic simulator vs. real providers |
| `BLOCKSTREAM_URL` | `https://blockstream.info/api` | BTC endpoint |
| `ETHERSCAN_API_KEY` | empty | ERC-20 / BEP-20 |
| `TRON_API_KEY` | empty | TRC-20 |
| `CHAIN_HTTP_TIMEOUT` | `8` | Provider call timeout (s) |
| `SESSION_COOKIE_SECURE` | `false` | `true` behind HTTPS |
| `OTC_DATA_DIR` | `./var` | SQLite database + uploads |

## 13. Limits & out of scope

This POC is a **flow demonstration**, not a settlement system:

- SQLite + a single process: no high availability, no real concurrency;
- no 2FA, IP allow-listing, rate limiting, account lockout or secret management;
- no RBF / double-spend detection, no polling or on-chain webhooks;
- no real bank connectivity: the transfer reference and receipt are entered by hand;
- no cryptographic receipt signing, no certified accounting trail;
- no automated KYC, no AML/sanctions screening;
- no multi-tenancy, no ops / compliance / supervisor split;
- amounts and rates are not rounded according to production ledger rules;
- single flow direction (client sells crypto for NGN).

Out of scope of the v1.0 specification and therefore absent here: exchange engine, order book,
market making, custody / cold wallet, regulatory licensing.

## 14. Roadmap to v1.0

Suggested priority order to move from the POC to Phase 2/3 of the specification:

1. **PostgreSQL + migrations** (Alembic), Redis + worker for confirmation polling.
2. **Hardened auth**: TOTP 2FA, admin IP allow-list, password policy, rate limiting.
3. **Verification**: multi-provider fallback, RBF/double-spend detection, automatic polling,
   confirmation thresholds scaled to deal size.
4. **Fiat rails**: bank integration (instant transfer / NIP), automatic reference reconciliation,
   settlement statuses.
5. **Compliance**: KYC/AML, sanctions screening, evidence retention, regulator export.
6. **Frontend**: Next.js + WebSocket for realtime tracking, transactional notifications.
7. **Operations**: Docker, TLS everywhere, tested backups, observability (logs, metrics),
   load tests and external security review.

## 15. Design system

Dark "desk terminal" theme, gold (value) and teal (confirmed) accents.

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#07090c` | Background |
| `--surface` | `#0e131a` | Cards |
| `--line` | `#1d2835` | Borders |
| `--text` | `#e7ecf3` | Text |
| `--gold` | `#e9b949` | Primary accent, amounts |
| `--teal` | `#37d6a6` | Success, confirmations |
| `--red` | `#f2707f` | Disputes, errors |
| `--blue` | `#7aa7ff` | Information |

Typography: Inter (UI), JetBrains Mono (references, addresses, technical amounts).
Components (cards, KPIs, status badges, stepper, tables, forms, printable receipt) are defined in
`otc/static/css/app.css`, without a CSS framework.

---

*Internal POC — demo data only. Do not use with real funds.*
