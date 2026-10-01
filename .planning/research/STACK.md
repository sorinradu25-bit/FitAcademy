# Stack Research

**Domain:** Personal fitness & nutrition tracking mobile app with a rule-based, explainable AI coach (FitAcademy)
**Researched:** 2026-10-01
**Overall confidence:** MEDIUM-HIGH (see per-row confidence; method note below)

> **Method / confidence note.** Context7 MCP and the `ctx7` CLI were not available to this agent. Versions were pulled **directly from the PyPI and npm registry JSON APIs and the endoflife.date API on 2026-10-01** (primary sources, treated as HIGH for "what is current"). Behavioural/licensing claims come from official docs fetched via WebFetch/WebSearch (the GSD confidence seam classifies those providers as LOW by default). Where I cross-checked two or more independent sources I mark **MEDIUM-HIGH**. Anything that rests only on my training knowledge is marked **(training-only)** and should be verified in the relevant phase. Nothing below is legal advice (ODbL section especially).

---

## Executive Decisions (read this first)

| Decision | Verdict | One-line rationale |
|----------|---------|--------------------|
| **Mobile framework** | **React Native + Expo (SDK 57, TypeScript, Expo Router)** | Best barcode/app-store/EAS story, and it is the only option that shares a language, generated API types, validation schemas and i18n files with the Next.js admin. Python stays where it earns its keep: API, workers, coach engine, data ingestion. |
| **Python on the phone?** | **No.** Python lives in the backend, the coach engine, ingestion pipelines and the OpenAPI contract that generates the mobile client types. | The phone is a thin UI over a Python brain. On-device Python (Flet/Kivy/BeeWare) buys nothing (the coach runs server-side) and costs camera/scanner maturity, binary-wheel limits, app size and ecosystem. |
| **Runner-up** | Flutter 3.47 + `mobile_scanner` | Choose only if the author decides code sharing with Next.js does not matter and prefers Dart. |
| **Food data** | **Hybrid, three tiers: (1) USDA FDC Foundation Foods + optional CIQUAL 2025 as a curated generic-food core, (2) Open Food Facts for barcodes via server-side live lookup with write-through cache + a bulk-seeded Romania/EU subset, (3) user/admin-created foods.** Skip USDA Branded. | USDA is CC0 (zero obligations) but US-market and has no Romanian packaged goods; OFF is the only open source with real European/Romanian barcode coverage but is ODbL/crowd-sourced, so it must be isolated by `source` provenance, attributed, and never mutated in place. |
| **Job queue** | **Procrastinate 3.10 (Postgres-backed)** rather than a Redis queue; Redis kept for cache, rate limiting, idempotency/lock keys | Transactional enqueue gives outbox semantics for free, durable audit trail in SQL, built-in cron, no lost jobs on a Redis restart. *Deviates from the vision doc's "Redis job queues"; log it as an ADR. Fallback: Taskiq + Redis.* |
| **Hosting** | **One Hetzner Cloud VPS (EU) + Docker Compose + Caddy**, Cloudflare R2 for object storage, off-box Postgres backups | Cheapest, simplest mental model, EU latency to Romania. Managed PaaS (Render/Railway/Fly) is the escape hatch if ops becomes a distraction. |

---

## Recommended Stack

### Core Technologies (Backend)

| Technology | Version (2026-10-01) | Purpose | Why Recommended | Confidence |
|------------|---------------------|---------|-----------------|------------|
| Python | **3.14.x** (3.14.8 current; 3.13.16 acceptable fallback) | Runtime | 3.14 is 11 months old, EOL 2030-10; FastAPI, SQLAlchemy 2.1, pydantic-core all ship 3.14 wheels (also free-threaded wheels, but use the normal GIL build). SQLAlchemy 2.1 requires >=3.11. | HIGH |
| FastAPI | **0.142.x** (pin `~=0.142.2`) | API layer | Pre-decided. Note: still 0.x and had breaking changes in 0.125-0.132 and 0.137; **pin the minor and read release notes on every bump.** 0.142 adds native OpenTelemetry (2 days old; do not rely on it in v1). Pydantic v1 is gone. | HIGH |
| Pydantic | **2.13.5** + pydantic-settings **2.15.0** | DTOs/validation/config | FastAPI-native. The coach engine's `Insight`/`Reason` models should also be Pydantic so they serialize identically into the DB (JSONB) and the API. | HIGH |
| Uvicorn | **0.54.0** (`uvicorn[standard]`) | ASGI server | `uvicorn --workers N` behind Caddy is enough; no Gunicorn layer needed in 2026. (Granian 2.8.4 is a credible alternative; not needed.) | HIGH |
| PostgreSQL | **18.x** (18.6 current; 17.11 acceptable if a managed host lags) | Transactional store | PG 18 gives native `uuidv7()` (time-ordered UUIDs: good index locality and client-generatable-style IDs) *(training-only; verify on first migration)*. Supported to 2030-11. Extensions needed: `pg_trgm`, `unaccent`. | HIGH (version) / MEDIUM (uuidv7) |
| Redis | **8.x** (8.10.2 current) or Valkey 9.1 drop-in | Cache, rate limits, idempotency keys, locks | Per vision doc. With Procrastinate carrying jobs, Redis loss is non-fatal (cache only), which is the right blast radius for a solo-run box. Valkey (BSD) is the licence-simplest swap if the project goes public. | HIGH |
| SQLAlchemy | **2.1.1** (2026-09-25) | ORM + Core | Typed 2.x declarative (`Mapped[...]`), async, best-in-class Postgres support; repository layer from the vision doc maps cleanly. 2.1 makes **psycopg 3 the default Postgres driver**, drops Python 3.10, and **no longer installs greenlet by default: install `sqlalchemy[asyncio]`**. 2.1.1 is only ~1 week old; if a blocking bug appears, the 2.0.5x line is the safe fallback (Alembic 1.20 supports both). | MEDIUM-HIGH |
| Alembic | **1.20.0** (2026-09-11) | Migrations | Only serious choice with SQLAlchemy. 1.20 requires SA>=2.0 and has explicit 2.1 compat fixes. Use `alembic init -t async`; set a `MetaData(naming_convention=...)`; always hand-review autogenerate output. | HIGH |
| psycopg | **3.3.6** `[binary,pool]` | Postgres driver (async + sync) | One driver for SQLAlchemy 2.1 (default), Procrastinate (requires psycopg3) and psql-style scripts. asyncpg 0.31 is fast but has not released since 2025-11, and would mean two drivers. | HIGH |
| Procrastinate | **3.10.0** (2026-09-23) | Background jobs / cron | Postgres-native queue (`LISTEN/NOTIFY` + `SKIP LOCKED`), async-first, periodic (cron) tasks, retries, locks/queueing-lock for idempotent fan-out; requires Python 3.10+, PostgreSQL 13+, psycopg3. Enqueue inside the same transaction as the domain write (`MealLogged` outbox) - impossible with a Redis broker without a hand-built outbox. Jobs/results are queryable SQL, which fits the "insights stored for auditability" requirement. | MEDIUM-HIGH |
| PyJWT | **2.15.1** | JWT access tokens | What the official FastAPI tutorial now uses (python-jose is abandoned: do not use). Use `pyjwt[crypto]` if moving to EdDSA/RS256. | HIGH |
| pwdlib | **0.3.1** `[argon2]` | Password hashing | Official FastAPI tutorial replaced passlib with pwdlib + Argon2. | HIGH |
| redis-py | **8.1.0** | Redis client (async) | Standard. | HIGH |
| boto3 | **1.43.x** | S3-compatible storage (presigned URLs) | Only used to **generate presigned PUT/GET URLs** (pure CPU, no I/O), so no async wrapper needed. aioboto3 (last release 2025-10) is unnecessary. | HIGH |
| httpx | **0.28.1** | Outbound HTTP (OFF client), and test client | Note: last release 2024-12 but still the FastAPI-documented default; pair with `respx` for mocking. | HIGH |

**Auth design (hand-rolled, ~300 lines, deliberate):** short-lived access JWT (10-15 min, `sub`, `role`, `jti`), **rotating opaque refresh tokens** stored **hashed** (SHA-256) in Postgres with `family_id` + reuse detection (reuse = revoke family), Argon2id passwords, RBAC via a `require_role()` dependency, ownership checks in the repository layer (`WHERE user_id = :current_user`), rate limits on `/auth/*`. Do **not** adopt `fastapi-users` (15.0.5): it went into maintenance mode (announced 2025-10-25, no new features), is opinionated about its own user model/managers, and building this yourself is a portfolio-visible, test-covered security artefact. Managed IdP (Auth0/Keycloak) is overkill for v1; revisit only for social login at public launch.

### Core Technologies (Mobile client) - see "Decision 1" below

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| Expo SDK | **57** (`expo@57.0.26`; RN **0.86.3**, React **19.2**) | App framework/toolchain | Current stable (released 2026-06-30). New Architecture only (mandatory since SDK 55). **Do not start on SDK 58 (beta since 2026-09-15).** Always add packages via `npx expo install` so versions are SDK-pinned (npm `latest` for react-native is 0.87.x, which is *not* what SDK 57 wants). | HIGH |
| Expo Router | **57.x** | File-based navigation | Default Expo navigation; typed routes. | HIGH |
| expo-camera | **57.0.6** | Barcode scanning (EAN-13/EAN-8/UPC-A/UPC-E etc.) | Built in: `CameraView` + `barcodeScannerSettings` + `onBarcodeScanned`. Android uses Google's code scanner, iOS 16+ uses `DataScannerViewController`; works in Expo Go and dev builds. If you ever need custom frame processing, `react-native-vision-camera` 5.2.3 is the upgrade path. | HIGH |
| TypeScript | use what the Expo and Next templates install (npm `latest` is 7.0.2, the native-compiler line; **verify toolchain compatibility before bumping, else pin ~5.9**) | Types | Shared across mobile + admin. | MEDIUM |
| TanStack Query | **5.104.0** + `@tanstack/query-async-storage-persister` / `react-query-persist-client` 5.104.0 | Server state, cache, offline mutation queue | Persist the query cache to MMKV/SQLite so yesterday's log renders offline; `onlineManager` + paused mutations give a log-while-offline queue. | HIGH |
| expo-sqlite | **57.0.3** | Local store (outbox of unsynced logs, recent/favourite foods, cached food items) | First-party, works offline; sufficient for the "log offline, sync later" requirement without a sync engine. | HIGH |
| expo-secure-store | **57.0.4** | Refresh-token storage (Keychain/Keystore) | Never AsyncStorage for tokens. | HIGH |
| i18next / react-i18next / expo-localization | **26.4.2 / 17.0.15 / 57.0.2** | EN + RO i18n | Expo docs endorse react-i18next; uses `Intl.PluralRules`, needed for Romanian `one/few/other`. | HIGH |
| zustand | **5.0.15** | Tiny client UI state (active meal slot, scanner state) | Server state stays in TanStack Query; zustand only for ephemeral UI state. | MEDIUM-HIGH |
| react-hook-form + zod | **7.89.0 + 4.6.5** | Forms/validation (log meal, set, profile) | Same pair as admin; share zod schemas where DTOs overlap. | HIGH |
| @shopify/flash-list | **2.3.3** | Long lists (food search, history) | Standard for large RN lists. | HIGH |
| victory-native (XL) + @shopify/react-native-skia | **42.0.1 + 2.14.0** | Progress/trend charts | Skia-based, performant; `react-native-gifted-charts` 1.4.80 is the lighter no-Skia alternative if bundle size matters. Install Skia via `expo install`. | MEDIUM |
| openapi-typescript / openapi-fetch / openapi-react-query | **7.13.0 / 0.17.0 / 0.5.4** | **Generated, type-safe API client from FastAPI's OpenAPI** | The concrete "Python drives the client" mechanism: FastAPI emits OpenAPI -> CI regenerates `packages/api-client` -> mobile *and* admin compile against the same types; contract drift becomes a compile error. Chosen over `@hey-api/openapi-ts` (0.99, pre-1.0 churn) and `orval` (8.x, heavier codegen). | MEDIUM-HIGH |
| EAS Build / EAS CLI | **eas-cli 24.8.0** | Cloud builds, TestFlight/Play internal track, OTA updates | Free plan: 30 builds/month (max 15 iOS), 1,000 update MAU. Ample for a solo dev. | MEDIUM |
| Styling | **React Native `StyleSheet` + a small design-token module** for v1 | Styling | NativeWind's stable v4.2.7 pins you to Tailwind 3 while v5 (Tailwind 4) is still RC (`5.0.0-rc.0`); don't build the app's foundation on a mid-migration dependency. Revisit NativeWind v5 / Uniwind (1.12) once stable. | MEDIUM |

### Core Technologies (Web admin)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| Node.js | **24 LTS** (24.21.0; pin via `.nvmrc`/mise). Node 26 becomes LTS on 2026-10-28: evaluate after | Toolchain runtime | 24 is supported to 2028-04. | HIGH |
| pnpm (workspaces) | **12.8.1**; Turborepo **2.11.6** optional | Monorepo for `apps/mobile`, `apps/web-admin`, `packages/api-client`, `packages/i18n` | Expo supports pnpm monorepos; Turborepo only if build times hurt (not needed at this size). | MEDIUM-HIGH |
| Next.js | **16.3.8** (React 19.3, Node >=20.9) | Admin | Pre-decided. Use as a thin BFF: store the admin JWT in an httpOnly cookie via route handlers/Server Actions, never localStorage. | HIGH |
| Tailwind CSS + shadcn/ui | **4.3.3 + CLI 4.21.1** | Admin UI | Fastest route to a clean CRUD admin. | HIGH |
| TanStack Table | **9.2.4** | Foods/exercises/users grids | Headless; pairs with shadcn. (v9 is a recent major: if you hit friction, v8 remains widely documented.) | MEDIUM |
| next-intl | 4.14.8 | Admin i18n *if ever needed* | v1 admin can be English-only (requirement is app UI EN/RO). | MEDIUM |

### Database / Data

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| PostgreSQL 18 + `pg_trgm` + `unaccent` | 18.6 | Food/exercise search, all data | See "Search" below. Skip Elasticsearch/Meilisearch/Typesense for v1: the food table is O(10^5) rows of short strings, which `pg_trgm` GIN handles in milliseconds. |
| DuckDB | **1.5.6** | **Offline ingestion tool** (not a runtime dependency): read OFF Parquet/JSONL and USDA CSV, filter, normalise, emit to Postgres `COPY` | Reads the 4.8M-row OFF Parquet without loading it into memory; far simpler than a hand-rolled JSONL parser. `polars` 1.44.2 or `ijson` 3.5.1 are fine alternatives. |
| Cloudflare R2 (prod) | n/a | S3-compatible object storage (user photos later, exports) | 10 GB free, $0 egress, S3 API + presigned URLs. Hetzner Object Storage (~EUR 4.99-6.49/mo base) is the same-vendor alternative. |
| Garage or SeaweedFS (local dev) | n/a | Local S3 in docker-compose | **Do not use MinIO**: the community edition entered maintenance mode in Dec 2025, the repo was archived, and official community binaries/images are no longer published. |

### Infrastructure

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Hetzner Cloud VPS (CX23 ~EUR 5.49/mo, or CX33 for headroom) | n/a | Single host: `api`, `worker`, `web-admin`, `postgres:18`, `redis:8`, `caddy` in one `docker-compose.yml` | Prices rose 2026-06-15 but remain far below PaaS (a comparable API+worker+Postgres stack is ~USD 45-55 on a Hetzner box vs ~USD 70-120 on Render/Fly/Railway per 2026 comparisons). EU region = low latency to Romania. Avoid ARM (CAX) unless you build multi-arch images. |
| Caddy | 2.x | TLS (automatic Let's Encrypt), reverse proxy | Zero-config HTTPS; one 10-line Caddyfile. |
| GitHub Actions + GHCR | n/a | CI (lint, type-check, tests, OpenAPI export + client regen check, Alembic `upgrade head` on a throwaway PG) and deploy (`docker compose pull && up -d` over SSH) | Keep deploy boring and scripted; consider Coolify (v4.3 stable) *only* if SSH-deploy becomes painful. |
| Postgres backups | `pg_dump` nightly (or pgBackRest/WAL-G) -> R2 | **Backups off the box + a tested restore**, written as an ADR/runbook before real data exists | The single biggest solo-dev risk is a lost VPS disk. |
| Sentry (`sentry-sdk` 2.71.0, `@sentry/react-native` 8.29.0) | n/a | Errors (API, worker, mobile, admin) | Free tier is enough; catches the phone-only crashes you will never reproduce locally. |
| structlog 26.1.0 + asgi-correlation-id 5.0.1 | n/a | JSON logs with request/job IDs | Required to trace "why did the coach say that" across API -> job -> insight row. |

**Managed alternative (if ops is unwanted):** Render or Railway for API+worker, managed Postgres (Neon/Render PG), Upstash/Render Redis. Expect roughly 2-5x the monthly cost of the Hetzner box (2026 comparisons: ~USD 32 lean Render / ~USD 23 lean Railway / ~USD 14 Fly vs ~USD 10 Hetzner for hobby-sized stacks; ~USD 72-122 vs ~USD 45-55 for web+worker+PG). Fly.io has no free tier for new accounts.

### Supporting Libraries (Backend)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `slowapi` 0.1.10 *or* `fastapi-limiter` 0.2.0 | n/a | Rate limiting on `/auth/*`, barcode lookups, search | Redis-backed. Prefer `fastapi-limiter` (async redis-py). Both are lightly maintained; a ~40-line Redis `INCR`+`EXPIRE`/Lua dependency is an acceptable replacement. *(API not verified; spike in the auth phase.)* MEDIUM |
| Babel | 2.18.0 | Server-side rendering of notification/email text, CLDR plural rules for `ro` | Only for text rendered **on the server** (push, email). See i18n section. |
| tenacity | 9.1.4 | Retry/backoff for OFF HTTP calls | Wrap the OFF client; combine with a Redis token bucket (OFF allows 15 product reads/min/IP). |
| python-slugify / `unicodedata` (stdlib) | 9.1.2 | NFC-normalise + slug | Ingestion; also fix ş/ţ (cedilla) -> ș/ț (comma-below) before storing. |
| `openfoodfacts` (official SDK) | 5.3.0 (MIT, "beta") | Product lookup, dataset helpers | Optional. A ~60-line `httpx` client with a proper `User-Agent` gives you full control over caching/timeouts; use the SDK only for taxonomy helpers. |
| fastapi-pagination | 0.16.0 | List endpoints | Optional; **cursor pagination** for meal/workout history is simple enough to hand-roll with `(logged_at, id)` keysets. |
| import-linter | **2.15** | **Enforce modular-monolith boundaries in CI** (`nutrition` must not import `workouts` internals; modules talk via public service interfaces and events) | **Mandatory** for this architecture: without it the "modular" monolith decays into a ball of mud. |
| orjson 3.12.0 | n/a | Fast JSON responses | Optional micro-optimisation; skip in v1. |

### Development & Test Tools

| Tool | Version | Purpose | Notes |
|------|---------|---------|-------|
| uv | **0.12.21** | Python package/venv/lockfile manager | Replaces pip/poetry/pip-tools; used in the official FastAPI docs (`uv add`). |
| Ruff | **0.16.9** | Lint + format (replaces black/isort/flake8) | One tool. |
| mypy | **2.3.1** (strict) | Type checking | Pydantic and SQLAlchemy plugins are mature. `ty` (0.0.84) is still pre-release: do not gate CI on it. pyright 1.1.414 is an acceptable alternative. |
| pytest / pytest-asyncio | **9.1.1 / 1.4.0** | Test runner | `asyncio_mode = "auto"`. |
| testcontainers | **4.15.0** | **Real Postgres 18 (+ Redis) in tests** | Do not test against SQLite; `pg_trgm`, JSONB, `uuidv7`, and Procrastinate all need real Postgres. |
| polyfactory | 3.3.0 | Test data factories for Pydantic/SQLAlchemy models | |
| time-machine | 3.5.1 | Freeze time | Essential for daily/weekly jobs and timezone-boundary tests (`freezegun` 1.5.5 works but is slower). |
| hypothesis | 6.168.3 | **Property-based tests for the coach rules engine** | E.g. "no rule ever emits a recommendation below the safe calorie floor"; "every insight has >=1 reason". Pair with golden/snapshot fixtures for explanation text. |
| schemathesis | 4.28.0 | OpenAPI-driven fuzzing/contract tests | Cheap, portfolio-visible, catches 500s and auth holes. |
| respx | 0.23.1 | Mock `httpx` (OFF API) | |
| pytest-cov 7.1.0 / pytest-xdist 3.8.0 | | Coverage / parallel | |
| Mobile: jest-expo 57.0.5 + @testing-library/react-native 14.0.1 | | Component/unit tests | |
| Mobile E2E: Maestro (install via the official script; YAML flows) | 2.x *(verify)* | Smoke E2E: login -> scan -> log meal | Far less setup than Detox. Keep to 3-5 flows. |
| Admin: Vitest 5.0.3 + Playwright 1.63.0 | | Unit + one admin smoke flow | |
| Biome 2.5.15 *or* ESLint 10.11.0 + Prettier 3.9.9 | | JS/TS lint+format | Pick one; Biome is a single fast tool, ESLint has the broader RN/Next rule ecosystem (Expo's template ships ESLint config). Default: **ESLint + Prettier** to match templates. |

---

## Decision 1: Mobile Framework

### Verdict: React Native with Expo (SDK 57), TypeScript. Python is not on the device.

### Evidence-based comparison

| Criterion | **RN + Expo 57** | Flutter 3.47 | Flet 1.0.3 | Kivy 2.3.1 | BeeWare (Toga 0.5.7 / Briefcase 0.4.5) |
|-----------|------------------|--------------|------------|------------|-----------------------------------------|
| Camera + barcode (EAN-13/UPC) | **Built in** (`expo-camera`, 13 symbologies, Google code scanner on Android, `DataScannerViewController` on iOS 16+, works in Expo Go) | Excellent: `mobile_scanner` 7.4.2 (ML Kit / Apple Vision, 2.3k likes, 1.59M downloads) | `Camera` control (preview/capture/frame streaming, since 0.81) but **no documented built-in barcode decoding**; you'd decode frames in Python (needs mobile-built wheels, e.g. zxing-cpp, *unverified*) or write a Dart/Flutter extension (negating "pure Python") | No maintained mobile barcode story; stale (last release 2024-12) | No documented live-preview scanning *(unverified)* |
| App-store readiness | Mature: EAS Build -> TestFlight/Play, OTA updates | Mature | Docs show `flet build apk/aab/ipa`, signing, Play/App Store flows; iOS builds require macOS + Xcode. Brand-new 1.0 (2026-09-30 was 1.0.3); thin production track record | Buildozer/kivy-ios; non-native look | Pre-1.0 (0.x); native widgets |
| Embedded-Python store risk | n/a | n/a | Bundled CPython must satisfy Apple review (historical `itms-services` string auto-rejection in 3.12, cpython issue #120522: since patched but shows the class of risk) | same class | same class |
| i18n EN + RO (plurals) | **i18next + Intl.PluralRules**, Expo guide, per-app language via config plugin | First-party `gen-l10n` ARB + ICU plurals (arguably the best out-of-box) | DIY (gettext/Babel in Python on-device) | DIY | DIY |
| Offline-friendly logging | `expo-sqlite` + TanStack Query persist + outbox | `drift`/`sqflite` + queues | Python `sqlite3` works; sync model is custom | Same | Same |
| Ecosystem maturity | Very large; SDK every ~4 months (57 -> 58 beta already) | Very large; Material/Cupertino split into packages in 3.44 (recent churn) | Small; ~100 prebuilt binary wheels only; single-threaded event loop caveat | Aging | Small |
| **Code sharing with Next.js admin** | **High**: TS, generated OpenAPI client, zod schemas, i18n JSON, TanStack Query, same lint/test tooling | None (Dart) | None | None | None |
| Python already in stack (backend + coach) | Python via **OpenAPI contract** (types generated from FastAPI) | Same | Python UI + Python backend, but UI-side Python adds nothing the server doesn't already own | Same | Same |
| Solo-dev + AI-assisted velocity | High (largest corpus, Expo tooling) | High | Medium (small corpus, fast-moving 1.0) | Low | Low |
| Portfolio signal | Strong (TS full-stack + Python backend is a very common hiring profile) | Strong (Dart) | Novel but "toy" risk for reviewers | Low | Low |

### Why not Python-on-device (the honest answer to "I'd like Python involved")

1. **The valuable Python is server-side.** The coach engine, rules, scheduling and data ingestion are Python and run on the backend (daily/weekly async jobs, auditable). Nothing in the v1 mobile client needs Python's strengths; it renders screens, scans barcodes, and queues log writes.
2. **Scanner and platform-API gap.** Barcode scanning is a core v1 requirement. Flet 1.0 exposes a Camera control but I found no built-in scanner; closing the gap means native Flutter code or on-device binary wheels, which is exactly the work Python-only frameworks promise to avoid.
3. **Binary-wheel ceiling.** On-device Python can only use packages with pre-built iOS/Android wheels (Flet's index: "more than 100"). Any C/Rust dependency not in that list blocks you.
4. **Maturity and risk.** Flet 1.0 reached "production-ready" status only on release day this cycle; Kivy is stagnant; BeeWare is pre-1.0. A portfolio piece that may go public should not have its client on the least-proven layer.
5. **Where Python *is* involved (make this explicit in the roadmap):** FastAPI API; Procrastinate workers; the pure-Python `coach` package (no I/O, Pydantic in/out, property-tested); OFF/USDA/CIQUAL ingestion CLIs (DuckDB/polars); the OpenAPI spec that generates every client type; later the LLM language layer (Anthropic SDK, server-side, never deciding health matters); analysis/eval notebooks for tuning rules.

**Choose Flutter instead when:** the author prefers Dart/pixel-identical UI and accepts no code sharing with the admin. It is the strongest alternative (superior first-party i18n and scanner plugin).
**Choose Flet instead when:** the hard requirement becomes "no JS/TS in the repo", and the author accepts building scanning as a Flutter extension, plus a thinner ecosystem. Time-box a 2-day spike (camera + EAN scan + Play internal-track upload) before committing.

### Mobile practicalities to bake into phase plans

- **Use a development build (`expo-dev-client` 57.0.19) from day one**, not Expo Go: Expo Go is also pending App Store approval for SDK 57 at the time of writing, and you will add native modules.
- **Which phone does the author have?** An iPhone requires a paid Apple Developer account ($99/yr, *training-only*) for TestFlight/long-lived installs; Android sideload is free. Resolve before the "runs on my phone" milestone.
- **Offline logging pattern (no sync engine in v1):** client generates a UUID per log entry -> writes to SQLite outbox + optimistic cache -> `POST` with `Idempotency-Key`/client ID -> server upserts by client ID. Food *search* needs network; cache recents/favourites locally. Do not adopt PowerSync/WatermelonDB for v1.
- **Barcode flow:** scan -> `GET /foods/barcode/{ean}` (server: local DB -> OFF live lookup -> 404 "not found, add manually") -> never call OFF directly from the phone (shared server IP limit, cache control, attribution).
- **Hermes `Intl.PluralRules` for Romanian:** test `few` category rendering on a real device in the i18n phase; keep `@formatjs/intl-pluralrules` 6.3.15 as the polyfill fallback. *(Not verified on RN 0.86.)*

---

## Decision 2: Food Data Source

### Verdict: Hybrid with explicit tiers and provenance. Open Food Facts for barcodes (server-side, cached, isolated by `source`), USDA FDC Foundation Foods (and optionally CIQUAL 2025) for generic foods, user/admin foods on top. Skip USDA Branded.

### Comparison

| Criterion | Open Food Facts | USDA FoodData Central | CIQUAL 2025 (ANSES, FR) | FatSecret / Nutritionix / Edamam |
|-----------|-----------------|----------------------|--------------------------|----------------------------------|
| Coverage | **~4.79M products worldwide**; Romania-specific site shows roughly **31.9k** products *(search-derived, MEDIUM-LOW)*; barcodes are its core | ~600k+ records: Foundation Foods (tiny, high quality, 04/2026), SR Legacy (~7.8k generic, **final, frozen 2018**), FNDDS (survey/dishes), Branded (~US market, 2.9 GB CSV) | 3,484 generic foods x 74 components (French names) | US-centric; paid/free tiers with attribution |
| Barcodes | **Yes (primary use)**, European/Romanian packaged goods present | Branded has GTINs but **US products**; useless for Romanian shelves | No barcodes | FatSecret Basic (free): **US dataset only**, UPC scan, attribution required; regionalisation is Premier |
| Generic/whole foods | Weak (not its purpose) | **Excellent (Foundation, SR Legacy)** | **Excellent for European foods** | Mixed |
| Data quality | Crowd-sourced; a 2021 analysis found ~67% of entries had complete macros, <20% micronutrients beyond sodium *(single secondary source)*; per-serving vs per-100g mistakes common | Lab-analysed / government-curated | Government-curated, current (2025 edition) | Proprietary, variable |
| Licence | **ODbL** (database) + **DbCL** (contents) + **CC-BY-SA** (images); attribution + share-alike on derivative databases | **CC0 / public domain** (attribution requested, not required) | **Etalab Open Licence 2.0** (attribution, free reuse incl. commercial) | Restrictive: attribution, caching limits, no bulk collection (Edamam), commercial tiers USD 49-1,850+/mo |
| API limits | **15 req/min/IP product reads; 10 req/min/IP search**; custom `User-Agent` required; IP bans on abuse | **1,000 req/hour/IP** with free data.gov key (DEMO_KEY: 30/hr, 50/day) | Download only (Excel/XML via Zenodo / data.gouv) | 5,000/day (FatSecret Basic) / 200/day free (Nutritionix) |
| Bulk import | **Nightly** JSONL, CSV (~0.9 GB gz), **Parquet on Hugging Face**, MongoDB dump; **14-day daily deltas** | CSV/JSON zips per dataset (Foundation 04/2026, Branded 04/2026) | XLSX/XML (1.5-70 MB) | Not allowed / not offered |
| Romanian names | Many Romanian product names (user-contributed) | English only | French only | English |

### Why this combination

1. **Barcodes demand OFF.** It is the only open dataset with meaningful European/Romanian barcode coverage. USDA Branded is a US product catalogue; importing 2.9 GB of it adds noise and zero Romanian hits.
2. **Generic foods demand a curated core.** OFF is poor at "100 g boiled chicken breast / raw apple / cooked rice". USDA Foundation Foods (+ SR Legacy for breadth) and CIQUAL 2025 are authoritative, and the licences (CC0, Etalab) impose almost nothing. Seed a **~300-600 item "FitAcademy Core Foods" table** with `name_en` / `name_ro`, via bulk import then admin curation (Claude-assisted Romanian translation, human-reviewed in the admin app). Add Romanian staples (sarmale, mamaliga, ciorba, etc.) manually via admin; no open Romanian composition table was found.
3. **Commercial APIs are the wrong trade for this project.** Attribution banners, caching bans and per-month fees conflict with the "local DB first" architecture and a possible public launch.

### Import & lookup strategy

- **Tier A - generic core (batch, one-off + occasional refresh):** Python CLI (`apps/api/.../ingest/`) reads USDA Foundation CSV (+ SR Legacy), optionally CIQUAL XLSX -> normalises to per-100 g kcal/protein/carbs/fat/fibre/sugar/sodium (canonical units, store raw source payload JSONB) -> upsert into `foods` with `source in ('usda','ciqual')`, `license`, `attribution`.
- **Tier B - barcodes (hybrid):**
  1. **Bulk seed** with DuckDB over OFF Parquet: filter `countries_tags` containing `en:romania` (plus optionally neighbouring/EU markets you care about), require non-null energy + macros, drop obviously invalid rows (e.g. kcal > 900/100 g), `COPY` into Postgres. Weekly delta refresh via a Procrastinate periodic task using the 14-day delta files.
  2. **Live lookup on cache miss** (server-side only): local DB -> Redis negative cache -> `GET /api/v2/product/{ean}?fields=...` with `User-Agent: FitAcademy/<ver> (<contact email>)`; guard with a global Redis token bucket (<=12/min to stay under the 15/min/IP cap), `tenacity` backoff, and a circuit breaker. **Write-through** the result into `foods` (`source='off'`, `source_ref=<ean>`, `fetched_at`, `off_last_modified`). Cache "not found" for ~24 h.
  3. **Do not use OFF's search endpoint at runtime** (10/min/IP; no full-text server-side). Food *search* runs against **your** Postgres.
- **Tier C - user and admin foods:** `source in ('user','admin')`. **Copy-on-write:** if a user "fixes" an OFF item, create a new `user` row (`derived_from=<food_id>`); never mutate OFF rows in place.
- **Single `foods` table + provenance columns** (`source`, `source_ref`, `license`, `attribution_text`, `fetched_at`) rather than separate tables: simpler search, simple per-source export/delete. Keep OFF rows filterable so they can be exported or purged independently.
- **Contribute back:** OFF accepts edits through its write API. Pushing corrections upstream beats maintaining a divergent fork and is the cleanest answer to share-alike.

### ODbL share-alike: what it means for FitAcademy (not legal advice)

- ODbL text (opendatacommons.org, verified): a **Derivative Database** = any "translation, adaptation, arrangement, modification, or any other alteration" of the database; a **Produced Work** = output resulting from using a substantial part of the contents "via a search or other query"; a **Collective Database** = the original in *unmodified* form as part of a collection (not derivative). **Section 4.4:** any Derivative Database you **Publicly Use** must be under ODbL (or compatible). Section 4.6: you must offer recipients the whole derivative database or a file of your alterations.
- Practical reading: an app screen showing a calorie total is a *produced work* (attribution only). A **normalised Postgres copy of a substantial slice of OFF is likely a Derivative Database**, so once the app is public you should assume share-alike applies *to the OFF-derived rows*, **not to your own code, your own foods, or user data** (those are separate content in a collective database, the reason for the `source` column and copy-on-write).
- **Now (author + friends):** risk is negligible; do the cheap things anyway.
- **Compliance checklist to schedule before public launch:** (1) visible in-app attribution "Food data from Open Food Facts (ODbL)" with link, plus images' CC-BY-SA credit if you show product photos (or **don't store/serve OFF images**: hotlink OFF image URLs or skip photos in v1); (2) provenance columns as above; (3) a documented, reproducible **export of OFF-derived rows + alterations** (a `source='off'` dump script and a public `/open-data` page/repo), cheap to build if rows are filterable; (4) get a proper legal read for the public launch. *(Interpretation of "Publicly Use" for API-served apps is contested; sources disagree on whether network use triggers 4.4/4.6. Treat the conservative reading as the design target.)*
- USDA (CC0) and CIQUAL (Etalab 2.0, attribution) require no share-alike. Credit both in an in-app "Data sources" screen anyway (USDA asks for attribution "when possible").

### Search implementation (Postgres, v1)

- Extensions: `pg_trgm`, `unaccent`. `unaccent()` is not IMMUTABLE, so wrap it in a SQL `IMMUTABLE` function and index a **stored generated `search_text`** column: `lower(f_unaccent(name_en || ' ' || name_ro || ' ' || coalesce(brand,'')))` with `GIN (search_text gin_trgm_ops)`.
- Query: prefix match boost + `word_similarity(:q, search_text)` ordered desc, with boosts for the user's favourites/recents and `source` priority (core > user > off). Default `pg_trgm.word_similarity_threshold` is 0.6 (tune to ~0.3-0.4 for short queries and typos).
- **Romanian diacritics:** normalise to NFC and map legacy cedilla forms (ş, ţ) to comma-below (ș, ț) at ingest *and* at query time; `unaccent` then folds ă â î ș ț for matching. Postgres also ships a `romanian` text-search configuration if stemming is ever needed; trigram is the right default for short food names.
- Move to Meilisearch/Typesense only if search relevance becomes a measured problem. It won't in v1.

---

## i18n (EN + RO) - the part that is easy to get wrong

| Layer | Choice | Notes |
|-------|--------|-------|
| UI strings (mobile + admin) | **i18next 26 / react-i18next 17**, JSON catalogs in a shared `packages/i18n` | Use plural suffixes `_one/_few/_other` for Romanian. Dates/numbers via `Intl`, never hand-formatted. Romanian "20 **de** zile" falls into the CLDR `other` category and "2-19 zile" into `few`, so write separate keys. |
| **Coach insights (the important one)** | **Persist language-neutral, structured insights:** `{rule_id, rule_version, template_key, params, reasons:[{code, params}], generated_at}`; **render text at read time** on the client from catalogs | Explainable + auditable (store *why* in data, not baked prose); switch language without regenerating history; no LLM involved. Template catalogs are versioned alongside rule versions. |
| Server-rendered text (push, email) | **Babel 2.18** + gettext `.po` (CLDR plural rules, incl. `ro`) | Only needed once notifications exist (later milestone). Do not use `fastapi-babel` (1.0.0, last release 2024-12) or `python-i18n` (2020). |
| Food/exercise names | `name_en`, `name_ro` columns (or a translations table) + admin editing | Fall back to EN when RO is null. |
| API errors | Stable machine-readable `code` + params; clients translate | Never ship translated strings from the API. |

---

## Installation

```bash
# ---- Backend (apps/api) ----
uv init --python 3.14
uv add "fastapi~=0.142.2" "uvicorn[standard]" "pydantic>=2.13" pydantic-settings \
       "sqlalchemy[asyncio]~=2.1.1" "alembic~=1.20" "psycopg[binary,pool]~=3.3" \
       "procrastinate~=3.10" "redis~=8.1" pyjwt "pwdlib[argon2]" boto3 httpx tenacity \
       structlog asgi-correlation-id "sentry-sdk[fastapi]" babel python-multipart \
       fastapi-limiter
uv add --dev pytest pytest-asyncio pytest-cov pytest-xdist testcontainers polyfactory \
       time-machine hypothesis schemathesis respx ruff mypy import-linter
# Ingestion tooling (run offline)
uv add --group ingest duckdb polars openfoodfacts python-slugify

# ---- Monorepo / mobile (apps/mobile) ----
pnpm create expo-app apps/mobile --template default@sdk-57
cd apps/mobile
npx expo install expo-camera expo-router expo-sqlite expo-secure-store expo-localization \
     expo-dev-client expo-image expo-notifications @shopify/flash-list \
     @shopify/react-native-skia react-native-reanimated react-native-gesture-handler
pnpm add i18next react-i18next @tanstack/react-query \
     @tanstack/react-query-persist-client @tanstack/query-async-storage-persister \
     zustand react-hook-form zod victory-native openapi-fetch openapi-react-query \
     @sentry/react-native react-native-mmkv
pnpm add -D jest-expo @testing-library/react-native

# ---- Shared API client (packages/api-client): regenerate from FastAPI OpenAPI in CI ----
pnpm add -D openapi-typescript   # npx openapi-typescript ../../apps/api/openapi.json -o src/schema.d.ts

# ---- Web admin (apps/web-admin) ----
pnpm create next-app apps/web-admin --ts --tailwind --app
cd apps/web-admin
pnpm dlx shadcn@latest init
pnpm add @tanstack/react-query @tanstack/react-table react-hook-form zod openapi-fetch
pnpm add -D vitest @playwright/test
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| RN + Expo | Flutter 3.47 + `mobile_scanner` | Dart preferred, no admin code-sharing needed, want best first-party i18n and pixel-identical UI. |
| RN + Expo | Flet 1.0.3 | Non-negotiable "UI in Python"; accept custom scanner extension and young ecosystem; spike first. |
| Procrastinate (Postgres) | Taskiq 0.13 + Redis broker | You want the vision-doc "Redis job queues" literally; async-native, FastAPI integration (`taskiq-fastapi`), cron scheduler; pre-1.0. Needs an outbox table for transactional enqueue and AOF persistence on Redis. |
| Procrastinate | Celery 5.6.3 + Beat | Very large fan-out or you need the Celery ecosystem (Flower); sync-first, heavier ops, overkill here. |
| Procrastinate | Dramatiq 2.2.1 | Simpler than Celery, good retries; no native cron or transactional enqueue. |
| psycopg 3 | asyncpg 0.31 | Benchmarked hot paths only; stale release cadence, second driver needed for Procrastinate. |
| SQLAlchemy 2.1 | SQLAlchemy 2.0.5x | If 2.1.x shows a regression; Alembic 1.20 supports both. |
| SQLAlchemy | SQLModel 0.0.47 | Rejected: pre-1.0, couples API schemas to table models, which contradicts the module layering (API schemas/DTOs vs domain vs repository). |
| Hand-rolled JWT auth | fastapi-users 15.0.5 | Maintenance-only since 2025-10; acceptable if you need OAuth social login quickly, but not recommended as the foundation. |
| Hetzner + Compose | Render / Railway / Fly.io | You'd rather pay (2-5x) than own patching/backups. |
| Hetzner + Compose | Coolify (v4.3) on the same VPS | If scripted SSH deploys become tedious; adds a service to maintain. |
| Cloudflare R2 | Hetzner Object Storage / Backblaze B2 | Keep everything with one vendor. |
| pg_trgm | Meilisearch / Typesense / OpenSearch | Only after measured relevance or latency problems at much larger scale. |
| OFF (live + Romania seed) | Full OFF import (all 4.8M) | Never needed; huge ODbL surface, no benefit. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `python-jose`, `passlib` | Unmaintained; dropped from the official FastAPI tutorial | PyJWT + pwdlib[argon2] |
| `fastapi-users` as the auth foundation | Maintenance mode (Oct 2025), imposes its own user model | Own `auth` module |
| MinIO (community edition) for local S3 | Maintenance mode Dec 2025, repo archived, no official community images | Garage or SeaweedFS (dev), R2 (prod) |
| `arq` | Maintenance-only (last release 2026-04) | Procrastinate (or Taskiq) |
| Celery for this scale | Heavy config, sync-first, extra beat process, no transactional enqueue | Procrastinate |
| Runtime calls to OFF *search* / calling OFF from the phone | 10/15 req/min per IP; one shared server IP; no cache control; attribution scattered | Server-side barcode lookup with write-through cache + local search |
| USDA Branded Foods import | 2.9 GB of US-market products; no Romanian value | Foundation + SR Legacy + CIQUAL core, OFF for packaged goods |
| Commercial nutrition APIs as the primary source (FatSecret free = US-only; Nutritionix/Edamam restrictive and costly) | Licence/caching restrictions fight the local-DB-first design | OFF + USDA + own data |
| SQLite as the backend test DB | Hides `pg_trgm`/JSONB/uuidv7/Procrastinate behaviour | testcontainers Postgres |
| `AsyncStorage` for tokens | Unencrypted | `expo-secure-store` |
| Expo Go as the long-term dev environment | Will not carry your native module set; SDK-57 Expo Go release lagging | `expo-dev-client` development builds |
| Starting on Expo SDK 58 (beta) | Beta churn | SDK 57, upgrade ~1 month after SDK 58 GA |
| `fastapi-babel` (2024-12), `python-i18n` (2020) | Stale | Babel + gettext, or structured insights rendered client-side |
| Microservices / Kafka / Elasticsearch in v1 | Violates "modular monolith, no premature microservices" | Modules + import-linter + Procrastinate + pg_trgm |
| `ty` (0.0.84) as a CI gate | Pre-release type checker | mypy 2.3 strict |
| Free-threaded Python build in production | Ecosystem-wide thread-safety unknowns | Standard 3.14 build |
| Kivy / BeeWare for this app | Stale / pre-1.0, no mature live barcode scanner | RN + Expo |

## Stack Patterns by Variant

**If the author's phone is an iPhone:** budget the $99/yr Apple Developer account early (TestFlight); use EAS Build for iOS, no Mac strictly required for RN/Expo builds (it *is* required for `flet build ipa`).
**If the author's phone is Android:** sideload internal APK/AAB builds first; defer the Play Console ($25 one-time, *training-only*) until friends join.
**If OFF live lookup starts hitting the 15/min cap** (several friends scanning at once): extend the Romania bulk seed to more countries/categories, lengthen positive-cache TTL, and raise concurrency limits only after contacting OFF (they ask heavy users to reach out).
**If the app goes public:** move to a managed Postgres with PITR, add a second API instance behind Caddy/a load balancer (workers already stateless), swap hand-rolled rate limits for edge limits (Cloudflare), complete the ODbL compliance checklist and a legal review, and consider Redis -> Valkey for licence simplicity.
**If the LLM language layer is added later:** it lives in the Python backend (Anthropic SDK) as a *presentation-only* step over the already-persisted structured insight (`rule_id`, `reasons`, `params`); prompt templates are versioned next to rule versions. No stack change required, which validates the structured-insight storage decision above.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| SQLAlchemy 2.1.1 | Alembic 1.20.0, psycopg 3.3.x, Python >=3.11 | Needs `sqlalchemy[asyncio]` (greenlet no longer default); default PG driver is psycopg 3. Verified in SA 2.1 migration notes + Alembic 1.20 changelog. |
| Procrastinate 3.10 | psycopg 3 (required), PostgreSQL >=13 | Uses its own connection pool; share the DSN, not the SA engine. |
| FastAPI 0.142.x | Pydantic >=2 (v1 dropped), Starlette 1.6 | Pin minor; 0.x breaking changes appear in minor bumps (0.129, 0.131, 0.132, 0.137). |
| Expo SDK 57 | React Native 0.86.3, React 19.2, New Architecture only | `npx expo install --check` after every dependency add; do **not** use npm `latest` for RN/Reanimated/Gesture Handler/Skia. |
| expo-camera 57.x | Expo SDK 57 | Android scanning uses Google code scanner (Play Services required); iOS needs 16+ for DataScanner. |
| Next.js 16.3.x | React 19.x, Node >=20.9 | Run on Node 24 LTS. |
| Python 3.14 | FastAPI 0.142, SQLAlchemy 2.1, pydantic-core (cp314 wheels), psycopg 3.3 | Use normal GIL build. |
| Redis 8 / Valkey 9 | redis-py 8.1 | Interchangeable for this app's usage (cache, INCR, locks). |

## Open Questions / Verify During Phases

- **Procrastinate vs vision-doc Redis queue:** recommendation deviates; confirm via ADR early (auth/infra phase) with a 1-day spike of per-timezone daily fan-out (hourly periodic job -> per-user deferred tasks with `queueing_lock`).
- **Hermes Romanian plural rules** on RN 0.86 real devices (polyfill fallback ready).
- **Maestro current version/install path** and **fastapi-limiter API** were not verified; confirm in the phase that adds them.
- **Romania OFF coverage** (~31.9k) came from a search snippet; measure actual rows with complete macros after the first DuckDB pass before promising "Romanian barcode coverage".
- **Flet barcode feasibility** is inferred from absence in docs, not a proven negative; if Python-UI becomes a hard requirement, run the spike.
- **ODbL "Publicly Use" for API-served apps** is contested: treat the checklist as design target and get a legal read before public launch.
- **Phone OS (iOS vs Android)** affects the distribution plan and cost.

## Sources

- PyPI JSON API (`pypi.org/pypi/<pkg>/json`) and npm registry (`registry.npmjs.org/<pkg>`), queried 2026-10-01 - all version numbers - HIGH
- endoflife.date API (PostgreSQL 18.6, Python 3.14.8, Redis 8.10.2, Valkey 9.1.2, Node 24/26) - HIGH
- SQLAlchemy 2.1 migration notes (docs.sqlalchemy.org/en/21/changelog/migration_21.html) - Python >=3.11, psycopg default, greenlet extra - MEDIUM-HIGH
- Alembic 1.20 changelog (alembic.sqlalchemy.org) - SA 2.0 minimum, 2.1 compat - MEDIUM-HIGH
- FastAPI release notes (fastapi.tiangolo.com/release-notes) and OAuth2/JWT tutorial (PyJWT + pwdlib) - MEDIUM-HIGH
- fastapi-users maintenance-mode announcement (Oct 25, 2025) via search results - MEDIUM
- Procrastinate docs (procrastinate.readthedocs.io), Taskiq docs (taskiq-python.github.io), job-queue comparisons incl. arq maintenance status - MEDIUM
- PostgreSQL `pg_trgm` docs (postgresql.org/docs/current/pgtrgm.html) - HIGH
- Expo changelog + SDK 57 post (expo.dev/changelog/sdk-57: RN 0.86, React 19.2, released 2026-06-30), `expo-camera` docs (docs.expo.dev/versions/latest/sdk/camera/), Expo localization guide, New Architecture mandatory since SDK 55 - MEDIUM-HIGH
- EAS free-tier limits (30 builds/mo, 1,000 update MAU) - secondary pricing sites - MEDIUM
- Flutter 3.47.1 / Dart 3.13.1 (2026-08-19) via search; `mobile_scanner` 7.4.2 (pub.dev) - MEDIUM
- Flet 1.0 blog (flet.dev/blog/flet-1-0), Flet 0.81 Camera announcement, Flet Android/iOS publish docs - MEDIUM
- cpython issue #120522 (Python 3.12 `itms-services` App Store rejection) - MEDIUM
- Open Food Facts: API intro (openfoodfacts.github.io/openfoodfacts-server/api/: 15/10 req/min/IP, User-Agent), terms of use (ODbL/DbCL/CC-BY-SA, attribution), data page (JSONL/CSV/Parquet/Mongo, nightly, 14-day deltas), Hugging Face Parquet dataset (4.85M rows), world.openfoodfacts.org (4,786,737 products) - HIGH (official) / ro count MEDIUM-LOW
- ODbL 1.0 full text (opendatacommons.org/licenses/odbl/1-0/) - definitions, 4.4, 4.6 - HIGH; practitioner interpretation (dev.to "What ODbL Means for Commercial Nutrition Apps") - MEDIUM; *not legal advice*
- USDA FDC API guide (1,000 req/hr/IP, CC0) and download page (Foundation 04/2026, Branded 04/2026, SR Legacy final 04/2018) - HIGH (official)
- CIQUAL 2025 (anses.fr, Zenodo: 3,484 foods, 74 components, Etalab 2.0) - MEDIUM-HIGH
- FatSecret / Nutritionix / Edamam pricing and terms via comparison sites - MEDIUM-LOW
- Hosting comparisons (bex.co Sep 2026 PaaS ledgers, Hetzner price-adjustment 2026-06-15 docs, Coolify v4 reviews) - MEDIUM
- MinIO maintenance-mode coverage (Dec 2025) and R2 pricing - MEDIUM
- Training-only (flagged inline): PG 18 `uuidv7()`, Apple/Play developer account fees, Redis 8 AGPL licensing nuance.

---
*Stack research for: FitAcademy (fitness & nutrition tracker, FastAPI modular monolith, rule-based explainable coach)*
*Researched: 2026-10-01*
