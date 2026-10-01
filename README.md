# FitAcademy

> A personal wellness app that turns nutrition and workout logs into **clear, explainable coaching**, not just numbers.

**Status:** 🧭 Planning complete. Implementation starts with Phase 1. See the [roadmap](.planning/ROADMAP.md).

---

## What it is

FitAcademy combines meal logging (with barcode scanning), workout tracking, and a **deterministic, explainable coach**. The coach tells you what to change tomorrow and shows exactly *why*: the numbers it used and the rule that fired.

**Core value:** log quickly, get advice you can understand and trust.

v1 focuses on the core loop: accounts, goals and safe targets, food logging, barcode scanning, workouts, the coach, and a minimal admin. Social features, gamification, trainer Q&A and rewards are deliberately deferred until the core loop is proven by daily use.

## Architecture

```
┌────────────────────┐   ┌──────────────────────┐
│ Mobile app         │   │ Web admin            │
│ React Native+Expo  │   │ Next.js              │
└─────────┬──────────┘   └──────────┬───────────┘
          │  typed OpenAPI client   │
┌─────────▼─────────────────────────▼───────────┐
│ FastAPI — modular monolith  (/api/v1)         │
│  auth · users · nutrition · barcode ·         │
│  workouts · coach    (boundaries enforced     │
│                       by import-linter in CI) │
└──────┬───────────────┬────────────────┬───────┘
       │               │                │
  PostgreSQL 18     Redis/Valkey    S3 (R2)
  (schema/module,   (cache, rate    (media)
   outbox, jobs)     limits)
       │
  Procrastinate workers: daily/weekly coach runs,
  barcode ingestion, emails, outbox relay
```

### Key design decisions

| Decision | Why |
|----------|-----|
| **Modular monolith** (not microservices) | Simple to deploy for a solo developer, with clear boundaries; the coach can be extracted later |
| **Rule-based coach, no LLM decisions** | Deterministic, testable, auditable; an LLM may only rephrase text later |
| **Structured insights** (`rule_id` + version + params + evidence) rendered at read time | Localizable (EN/RO), auditable, replayable |
| **Nutrient snapshots** on every log entry | Editing a food never rewrites history |
| **`log_date` frozen in the user's timezone** | Correct daily totals, including across DST changes |
| **Transactional outbox** for domain events | No lost events, no message broker needed |
| **Hard safety guardrails** (calorie floor, rate cap, age/BMI gates) in code | Health safety does not depend on wording |
| **Hybrid food data**: USDA FDC (CC0) + Open Food Facts (ODbL), per-row provenance | Best coverage for EU barcodes, with licensing kept explicit |
| **EN + RO from day one** | Romanian has 3 plural forms, and i18n is costly to retrofit |
| **EU hosting** | GDPR Article 9: health data is special-category |

## Tech stack

- **Backend:** Python 3.14, FastAPI, Pydantic 2, SQLAlchemy 2.1, Alembic, PostgreSQL 18 (`pg_trgm`, `unaccent`), Procrastinate, PyJWT + Argon2
- **Mobile:** React Native + Expo (TypeScript), Expo Router, `expo-camera`, TanStack Query, i18next
- **Admin:** Next.js 16, Tailwind 4, shadcn/ui
- **Quality:** pytest + testcontainers (real Postgres), hypothesis, schemathesis, mypy strict, Ruff, import-linter
- **Infra:** Docker Compose, Caddy, Hetzner (EU), Cloudflare R2, nightly off-site backups with tested restore

## Roadmap

| Phase | Delivers |
|-------|----------|
| 1. Walking Skeleton & Secure Accounts | App on a real phone, EU-deployed backend, signup/consent/sessions, EN/RO, backups |
| 2. Profile, Goals & Safe Targets | Explainable calorie/macro targets with guardrails, bodyweight trend |
| 3. Food Search & Daily Meal Log | Bilingual food search, meal logging, daily totals vs targets |
| 4. Fast Logging & Barcode Scanning | Barcode scan with Open Food Facts fallback, quick-add, copy meal/day |
| 5. Workout Logging & Progress | Offline-safe workout logging, rest timer, progress charts |
| 6. Explainable AI Coach | Daily insight cards and weekly reviews with a "why" for every recommendation |
| 7. Web Admin Curation | Manage foods and exercises with translations |

## How this project is planned

The project is developed with a spec-driven workflow. Every decision is documented before code is written:

- [`.planning/PROJECT.md`](.planning/PROJECT.md): vision, scope, constraints, key decisions
- [`.planning/REQUIREMENTS.md`](.planning/REQUIREMENTS.md): 53 testable v1 requirements with IDs and phase traceability
- [`.planning/ROADMAP.md`](.planning/ROADMAP.md): phases with observable success criteria
- [`.planning/research/`](.planning/research/SUMMARY.md): stack, features, architecture and pitfalls research
- [`docs/REZUMAT-PROIECT.md`](docs/REZUMAT-PROIECT.md): full project walkthrough (Romanian)

## Non-goals

No medical diagnosis, no LLM-made health decisions, no "eat back" exercise calories, no wearables or meal-photo AI in v1.
