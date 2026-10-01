# FitAcademy

## What This Is

FitAcademy is a personal wellness platform that combines nutrition tracking, workout logging, AI-driven coaching, and (later) social motivation and rewards in one mobile app. The goal is not just to track data but to actively help users make better daily decisions through personalized, explainable insights. It is built as a serious, production-oriented system: a professional portfolio piece first, an app the author and friends use daily second, and a candidate for public launch later.

## Core Value

A user can quickly log what they eat and how they train, and get back clear, explainable coaching that tells them what to change, instead of just raw numbers.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can register, log in, and stay authenticated (JWT access + refresh tokens, roles: user / trainer / admin)
- [ ] User can set a profile and goal (fat loss, maintenance, muscle gain) with calorie/macro targets
- [ ] User can search a food database (macros + calories) and log meals with daily and historical summaries
- [ ] User can add foods manually, save favorites, and add foods by scanning a barcode
- [ ] User can browse an exercise library (muscle group, equipment)
- [ ] User can log workout sessions with sets, reps, and weights, and view history and progress
- [ ] User receives a daily insight card from the AI Coach (e.g. calorie/protein deviation, why it matters, one concrete change)
- [ ] User receives a weekly review (consistency, macro balance, training volume, 2–3 suggestions)
- [ ] Every coach recommendation includes a clear explanation of why it was made
- [ ] Coach insights are generated asynchronously (daily/weekly jobs) and stored for transparency/auditability
- [ ] Admin can manage the food and exercise databases and users via a minimal web admin
- [ ] App UI is available in English and Romanian (i18n from day 1)
- [ ] Backend is deployed to the cloud and the mobile app runs on the author's phone

### Out of Scope

- Social & community (profiles/following, feed, likes, comments, gym check-ins & reviews) — deferred past v1; core loop first
- Gamification (points, streaks, badges) — deferred past v1; motivation layer comes after the core loop is proven
- Rewards tied to partner gyms/brands — future; needs partners and anti-abuse rules
- Trainer Q&A (tickets, later real-time chat/scheduling) — deferred past v1
- Gym subscription wallet & renewal reminders — deferred past v1
- LLM-generated coaching text — v1 uses rules + text templates; LLM language layer added later (LLMs never make health decisions directly)
- Full web admin (moderation, partners, analytics) — v1 admin covers only foods, exercises, users
- Medical diagnosis or health treatment — non-goal; safety and liability
- Real-time biometric/wearable device integration — non-goal for initial phase
- Fully automated meal planning / personalized meal plans — non-goal until validated with safeguards
- Unverified health advice generation — non-goal; coach is rule-based and explainable
- Storing raw payment card data — never

## Context

- Source vision document: "FitAcademy Structure / Health Companion – Project Overview" (Google Doc). The body text calls the product "Health Companion"; the chosen name is **FitAcademy**.
- Architecture is pre-decided as a **modular monolith** with clear internal boundaries:
  - Client layer: mobile app + Next.js web admin
  - API layer: FastAPI (Python)
  - Domain modules: auth, users, nutrition, barcode, workouts, coach (v1); social, gamification, rewards, trainer, wallet, notifications (later)
  - Data layer: PostgreSQL (transactional), Redis (cache, rate limiting, job queues), S3-compatible object storage (images/media)
  - Async processing: background workers (daily coach insights, barcode ingestion, etc.)
- Each module follows the same internal layers: API (routes/validation) → Service (use cases) → Repository (DB access) → Domain models → Schemas/DTOs. Modules communicate through an internal event bus (e.g. `MealLogged`, `WorkoutLogged`).
- The vision doc includes a detailed starting repo layout (`apps/api`, `apps/mobile`, `apps/web-admin`, `workers/`, `infra/`, `docs/` with ADRs) — use it as the reference structure.
- AI Coach design: deterministic rule-based scoring engine considers user goals, recent behavior, and nutritional balance; outputs are explainable; prompt templates are versioned for the future LLM layer.
- Barcode lookup: local DB first, with external fallback (Open Food Facts is the doc's suggested provider). The food data source strategy (OFF vs USDA FoodData Central vs combination) is to be decided by research.
- Mobile framework is undecided. The author would like Python involved somewhere in the stack and is waiting for a recommendation; Python is already the whole backend + coach engine. Research should compare React Native (Expo), Flutter, and Python-native mobile options (Flet/Kivy/BeeWare), especially on camera/barcode support and app-store readiness.
- Audience progression: portfolio showcase → author + friends daily use → potential public launch. Decisions should not block a later public launch.

## Constraints

- **Tech stack**: Python + FastAPI backend, PostgreSQL, Redis, S3-compatible storage, Next.js web admin — set by the vision document
- **Architecture**: Modular monolith, no premature microservices — keeps deployment simple while allowing later extraction (e.g. AI Coach)
- **Team / timeline**: Solo developer working with Claude, no deadline — quality, correctness, and maintainability over speed
- **Safety**: Coach must be deterministic and explainable; no medical advice — health-related liability and trust
- **Security**: JWT + refresh tokens, RBAC (user/trainer/admin), ownership validation on all user data, rate limiting on sensitive endpoints, clear public/private data separation
- **i18n**: English + Romanian from day 1 — avoids a costly retrofit
- **Quality**: Clean, testable code, explicit domain boundaries, minimal magic, documentation as a first-class citizen (ADRs)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Product name: FitAcademy | Chosen over "Health Companion" | — Pending |
| v1 = core loop only (auth, nutrition + barcode, workouts, rule-based coach) | Prove the core value before adding motivation/social layers | — Pending |
| Modular monolith on FastAPI | Simple deployment, clear boundaries, future service extraction | — Pending |
| Coach: daily cards + weekly review | Daily nudges for action, weekly for trends | — Pending |
| Coach text via templates in v1, LLM later | Determinism and safety first; LLM is a presentation layer only | — Pending |
| Minimal web admin in v1 (foods, exercises, users) | Need a way to curate data without raw DB access | — Pending |
| English + Romanian i18n from day 1 | Serves local friends and broader portfolio audience | — Pending |
| Mobile framework: TBD after research (Python preference noted) | Needs a camera/barcode and app-store readiness comparison | — Pending |
| Food data source: TBD after research | Coverage vs licensing vs quality trade-off | — Pending |
| v1 "done" = backend deployed + author logs meals/workouts daily for 2+ weeks | Real usage validates the core loop | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-10-01 after initialization*
