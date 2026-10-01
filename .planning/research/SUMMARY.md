# Project Research Summary

**Project:** FitAcademy — Personal nutrition & workout tracking with explainable rule-based coach
**Domain:** Health & fitness mobile app (portfolio → author + friends → possible public launch)
**Researched:** 2026-10-01
**Confidence:** MEDIUM-HIGH overall (STACK and ARCHITECTURE high, FEATURES medium-low on competitor claims, PITFALLS experience-based)

## Executive Summary

FitAcademy is a production-oriented, portfolio-visible personal wellness platform combining nutrition tracking, workout logging, and explainable coaching. **Two open PROJECT.md decisions were resolved by research:**

1. **React Native + Expo SDK 57 (TypeScript) for mobile** — barcode scanning built into `expo-camera`, app-store maturity, code sharing with the Next.js admin via a typed OpenAPI client. Python stays server-side (FastAPI API, Procrastinate workers, pure-Python coach engine, food ingestion CLIs, future LLM layer). Runner-up: Flutter. Python-native mobile (Flet/Kivy/BeeWare) rejected: no built-in barcode decoding, limited mobile wheels, maturity concerns.
2. **Hybrid food data with per-row provenance** — curated USDA FoodData Central Foundation + SR Legacy core (CC0, optionally CIQUAL 2025) for generic foods; Open Food Facts for barcodes (bulk Romania seed via DuckDB from the OFF Parquet dump + throttled server-side live lookup); user foods. ODbL share-alike handled conservatively (provenance columns, copy-on-write corrections, attribution, legal review before public launch).

Recommended architecture: **modular monolith on FastAPI** with deterministic rule-based coaching, structured insights (`template_key + params + evidence`, rendered at read time), and module boundaries enforced by import-linter from day one.

**Non-negotiable decisions (lock in the first phases — expensive to retrofit):**
- Timezone-aware daily buckets (`log_date` frozen at write time, per user's IANA zone)
- Nutrient snapshots at log time (macros stored on the log entry, not fetched from mutable food rows)
- Targets as append-only history (`effective_from`; past days compared to the target in force then)
- Structured insights (rule_id + version + evidence + params, never stored prose)
- GDPR consent + hard deletion contract (`UserDeleted` cascade per module, tested)

**Key risk: engineering ceremony vs. solo-dev velocity.** Modular monolith + outbox events + i18n from day 1 cost more than plain CRUD, but pay off (extractable coach, testable boundaries, clean data model). Counter-risks: scope creep (social/gamification/LLM) and logging friction killing retention. **Gate:** the core loop must be proven by 2+ weeks of daily use before any public launch.

## Key Findings

### Recommended Stack

- **Mobile:** React Native + Expo SDK 57, TypeScript, Expo Router, `expo-camera` barcode scanning, development builds from day one; TanStack Query 5 with persisted cache; `expo-sqlite` outbox for offline logging; `openapi-typescript` + `openapi-fetch`
- **Backend:** Python 3.14, FastAPI 0.142.x (pin minor), Pydantic 2.13, Uvicorn 0.54, SQLAlchemy 2.1 (asyncio) + Alembic 1.20, psycopg 3.3, PostgreSQL 18 (`pg_trgm`, `unaccent`), Redis 8 / Valkey 9 (cache, rate limits, idempotency keys only)
- **Jobs:** Procrastinate 3.10 (Postgres-backed: transactional enqueue, cron, durable history) — deviates from vision doc's "Redis job queues" → needs ADR + spike; fallback Taskiq/Celery
- **Auth:** hand-rolled PyJWT 2.15 + pwdlib[argon2]; rotating hashed refresh tokens with family-based reuse detection (avoid fastapi-users, python-jose, passlib)
- **Tooling:** uv, Ruff, mypy strict, pytest 9 + pytest-asyncio + testcontainers (real Postgres), hypothesis (coach rules), time-machine, schemathesis, respx, import-linter (mandatory)
- **Admin:** Next.js 16 on Node 24 LTS, Tailwind 4, shadcn
- **i18n (EN + RO from day 1):** i18next/react-i18next with `_one/_few/_other` plural keys; coach insights stored language-neutral and rendered at read time; server-side Babel only later for push/email
- **Hosting:** single Hetzner Cloud VPS (EU), Docker Compose + Caddy, Cloudflare R2 object storage, nightly off-box Postgres backups with tested restore; Garage/SeaweedFS for local S3 (not MinIO); Render/Railway/Fly as fallback

### Expected Features

**Must-have v1:** auth (register/login/refresh/roles, password reset), profile (IANA timezone, locale, units, DOB), goal + computed targets (Mifflin-St Jeor → TDEE × activity → goal adjustment; protein first, fat, carbs fill; calorie floor + rate cap; "how we calculated this" screen), food search (recents/favorites first), meal logging with snapshots, barcode scan (local-first, OFF fallback, create-on-miss), workouts (sets/reps/weight, previous performance, set types, rest timer, repeat-last), daily + weekly coach insights (structured, explainable), account deletion/export + consent, EN+RO UI and content.

**Candidate v1 additions (not yet in PROJECT.md Active list):** bodyweight logging with trend (needed by weekly review), copy-meal / "log yesterday" + quick-add calories (major retention drivers), password reset, account deletion & data export, in-progress workout surviving restarts/poor signal, minimal admin audit log.

**Defer past v1:** routines, target-adjustment proposals, lift-progression hints, push notifications, saved meals/recipes, social, gamification, trainer Q&A, wallet, rewards, LLM text layer, adaptive TDEE, wearables, meal-photo AI, micronutrients.

**Anti-features:** LLM-written health advice, "eat back" exercise calories, streaks/shame copy before the core loop is proven (eating-disorder risk), social login in v1.

### Architecture Approach

- **Modules:** users, auth, nutrition, barcode, workouts, coach + platform kernel + API/worker entrypoints
- **Boundaries:** each module exposes only `public.py` (DTO facades) and `events.py`; import-linter forbids access to internals and keeps the coach engine I/O-free; one Postgres schema per module, single Alembic history, no cross-module ORM relationships/JOINs
- **Events (outbox-lite):** services record events on the Unit of Work → same commit writes `outbox_event` + business rows → in-process dispatch after commit → worker relay retries with `FOR UPDATE SKIP LOCKED`; at-least-once, idempotent handlers, thin events (IDs + `log_date`)
- **Nutrient snapshots:** `meal_item` stores `food_id` + snapshot name/per-100g + absolute macros; admin edits never rewrite history; NUMERIC not float; canonical units (g, kg, kcal)
- **Timezones:** `log_date` (DATE) + `logged_at` (timestamptz); group by `log_date`; profile IANA zone drives "today" defaults, job timing and week boundaries; `tzdata` in containers
- **Coach:** FeatureBuilder (only place coach reads other modules) → pure deterministic engine with versioned rules → immutable `CoachRun`/`CoachInsight` (input snapshot, supersede on rerun, all findings stored with `selected`/`suppressed_reason`) → template render in user locale
- **Scheduling:** 15-minute idempotent sweeper computes "due in user local time", claims via partial unique index, fans out per-user jobs; daily card covers previous complete local day, delivered in the user's morning
- **i18n data:** per-entity translation tables for foods/exercises with fallback chain; search normalized with `unaccent` + `pg_trgm`, Romanian cedilla (ş ţ) → comma-below (ș ț)
- **API:** `/api/v1` (app) and `/api/admin/v1`; additive-only changes; `client-config` endpoint with min version (426 for outdated clients); OpenAPI breaking-change diff in CI; stable error contract (`code` + `params`)
- **Security:** user-scoped repositories (user_id required parameter → IDOR structurally impossible); admin roles checked against DB

### Critical Pitfalls

1. **Crowd-sourced food data quality** (kJ vs kcal, missing = zero, per-100g vs per-serving, diacritics) → adapter per source, golden tests, Atwater sanity validator with quarantine, trust-tier ranking (verified > USDA > OFF complete > OFF incomplete > user)
2. **Mutable food rows rewriting history** → snapshot at log time, archive never delete referenced foods
3. **Barcode edge cases & OFF limits** (15 reads/min/IP, 10 searches/min/IP, regional variants, missing products) → GTIN-13 normalization + checksum, bulk seed, Redis negative cache, token-bucket throttle, custom User-Agent, never call OFF from client, graceful "not found, add it" flow
4. **Timezone/day-boundary bugs** → freeze `log_date` at write, per-user sweeper, DST fixtures (2026-10-25 Europe/Bucharest is a 25-hour day)
5. **Unsafe targets & eating-disorder risk** → hard guardrails in the target service (kcal floor ~1200F/1500M, rate cap, BMI < 18.5 blocks fat-loss, 18+ age gate, pregnancy screen), banned-phrase template lint, low-intake detection rule, day-completeness gate (no advice from half-logged days)
6. **Refresh-token rotation races** → single-flight refresh on client + short server grace window
7. **GDPR Art. 9 health data + store rules** → explicit unbundled withdrawable consent, in-app deletion (Apple + Google), Play Health apps declaration, health payloads out of logs/Sentry, EU hosting
8. **Solo-dev over-engineering & scope creep** → create modules only when their first feature is built, measured "core loop proven" gate

## Implications for Roadmap

Indicative 9-phase structure (roadmapper may rename/merge):

| Phase | Name | Rationale | Delivers | Research flags |
|-------|------|-----------|----------|----------------|
| P1 | Foundation & Skeleton | Enforce boundaries and patterns before business logic | Monorepo + tooling, Alembic multi-schema, UoW + event bus + outbox, import-linter CI, error model, i18n skeleton, clock abstraction + DST tests, versioned API skeleton, deploy pipeline (EU VPS), backups, observability | Spike: Procrastinate + async SQLAlchemy; import-linter vs Tach; real-device Expo barcode spike |
| P2 | Auth & Account Lifecycle | Identity, consent, deletion are structural; GDPR non-negotiable | Register/login/logout, JWT + rotating refresh, RBAC, password reset (email seam), health-data consent, account deletion (`UserDeleted` cascade), data export skeleton | — |
| P3 | Profile, Goals & Targets | Targets feed every coach rule and daily summary | Profile (timezone, locale, units, DOB), target computation, append-only target history, hard guardrails, "show the work" screen, manual override, bodyweight logging | Choose and cite kcal floors/caps (ADR); property tests |
| P4 | Food Data & Meal Logging | Core loop; highest retention and data-quality risk | Food catalog + translations + servings, search (pg_trgm, recents/favorites), meal logging with snapshots, daily summary, history, favorites, custom foods, quick-add, copy-meal/yesterday, USDA seed + curated RO core, provenance | Measure OFF Romania coverage; normalization golden tests; pg_trgm latency |
| P5 | Barcode & Food Ingestion | Leaf module depending on P4 | Scan → normalize → local → negative cache → throttled OFF → write-through → miss flow; bulk OFF Romania import + weekly refresh; review queue; ODbL attribution | ODbL ADR; supermarket acceptance test (~30 RO products) |
| P6 | Workouts | Independent of nutrition; can run parallel to P4–P5 | Exercise library (translations, logging_type), custom exercises, sessions with local persistence, sets (types, rest timer), previous performance, history, per-exercise progress, repeat-last | Exercise dataset licensing (free-exercise-db, wger) |
| P7 | AI Coach Engine | Core value; needs nutrition + workouts data | FeatureBuilder, pure engine, 6–12 versioned rules, selection/cooldown, sufficiency gate, CoachRun/CoachInsight storage, EN/RO template render, safety constraints, daily sweeper, weekly review | Rule thresholds + safety review; Romanian plurals on device; ED wording review |
| P8 | Web Admin UI | Curation beyond raw DB access | Foods CRUD + translation + review queue, exercises CRUD, users (roles, disable, delete), audit log | — |
| P9 | Store Readiness & Dogfood | Validate the core loop before any public launch | TestFlight + Play internal testing, privacy policy (EN+RO), store compliance, restore drill, metrics: median meal-log < 30 s, 14/14 days logged, zero total-mismatch bugs, coach cards rated useful | Apple/Play fees and policies (verify) |

**Ordering rationale:** P1–P3 are foundations that can't be retrofitted → P4 and P6 can proceed in parallel → P5 is a leaf of P4 → P7 needs nutrition + workout data → P8 after the admin API exists → P9 after the loop works. Deploy the walking skeleton early so dogfooding starts as soon as possible.

**Cross-cutting decisions to lock early (ADRs + migrations):**
1. Timezone model (`log_date` frozen at write, IANA zone)
2. Nutrient snapshots
3. Structured insight payload
4. Targets as history
5. i18n conventions (API returns codes only; clients own UI strings; translation tables for content)
6. Account deletion cascade
7. Food provenance columns (source, source_ref, license, attribution)

### Research Flags

- **Needs deeper research during planning:** Procrastinate cron fan-out with async SQLAlchemy; coach rule content and thresholds; eating-disorder safety wording and RO resources; OFF Romania real coverage; exercise dataset licensing; Apple/Play policy details
- **Standard patterns (skip research):** admin CRUD, auth with PyJWT, pg_trgm search

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Versions from PyPI/npm registries and endoflife.date (2026-10-01); mobile decision evidence-based on barcode support, store readiness, code sharing |
| Features | MEDIUM | Table stakes uncontroversial; competitor specifics from aggregated web sources; BMR/TDEE formulas HIGH |
| Architecture | MEDIUM-HIGH | Standard modular-monolith, outbox, timezone and snapshot patterns; Procrastinate integration needs a spike |
| Pitfalls | MEDIUM | OFF limits, store deletion rules, token rotation verified against primary docs; ED guidance and calorie floors are conventions requiring an ADR |
| **Overall** | **MEDIUM-HIGH** | Ready for requirements definition |

### Gaps to Address

1. **Real-device barcode spike** with `expo-camera` early in P1; fall back to Flutter + `mobile_scanner` if it fails
2. **OFF Romania coverage** (~31.9k products, unverified) — measure complete-macro rows after the first DuckDB pass
3. **Coach rule content and thresholds** — product decisions, researched in P7
4. **Eating-disorder guardrails** — wording review, RO support resources
5. **Procrastinate + async SQLAlchemy** — 1-day spike; fallback Taskiq/Celery + Redis
6. **Author's phone platform (iOS vs Android)** — affects distribution and Apple Developer fee
7. **Admin auth/CORS** — httpOnly cookies + separate rate limits; small ADR in P1
8. **ODbL stance for public launch** — publish OFF-derived subset or keep internal; legal review

## Sources

- `.planning/research/STACK.md` — stack versions, mobile framework comparison, food data source comparison, hosting
- `.planning/research/FEATURES.md` — table stakes, differentiators, anti-features, target-computation pipeline, competitor comparison
- `.planning/research/ARCHITECTURE.md` — module boundaries, outbox events, data modeling, coach design, scheduling, i18n, API versioning
- `.planning/research/PITFALLS.md` — 30 domain pitfalls with prevention strategies and phase mapping

---
*Research completed: 2026-10-01*
*Ready for roadmap: yes*
