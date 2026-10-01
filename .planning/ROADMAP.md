# Roadmap: FitAcademy

## Overview

FitAcademy v1 is built as a series of vertical slices. Each one ships end to end (FastAPI backend, React Native + Expo mobile UI, deployed) so the author can use every capability on their own phone as soon as it lands. Phase 1 stands up the deployed walking skeleton: a secure EN/RO account on the author's phone, backed by an EU-hosted backend with tested backups. Onboarding then produces safe, explainable targets. Meal logging comes next, and daily dogfooding starts there. Barcode scanning and fast-logging shortcuts follow, then workout logging. The explainable rule-based coach closes the core loop, and a minimal web admin lets the author curate foods and exercises. v1 counts as "done" when the backend is deployed and the author has logged meals and workouts daily for 2+ weeks. That gate is checked at milestone audit, before any public launch.

## Foundational Decisions (lock in the first phase that needs them)

| Decision | Locked in | Why it can't wait |
|----------|-----------|-------------------|
| Module boundaries enforced by import-linter; one Postgres schema per module; UoW + outbox events | Phase 1 | Boundary erosion is invisible until it is expensive |
| i18n conventions (API returns codes + params; clients own UI strings; EN/RO plural keys) | Phase 1 | Retrofitting translations touches every screen |
| Clock abstraction + DST test fixtures; `/api/v1` + client minimum version | Phase 1 | Every later phase depends on them |
| IANA timezone on profile; targets as append-only history (`effective_from`) | Phase 2 | Past days must be judged against the target in force then |
| `log_date` frozen at write time; nutrient snapshots on log entries; food provenance columns | Phase 3 | Rewriting history or re-bucketing days corrupts all summaries and coaching |
| Structured coach insight payload (rule_id + version + params + evidence, rendered at read time) | Phase 6 | Stored prose cannot be re-localized or audited |

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Walking Skeleton & Secure Accounts** - Deployed EU backend + app on the author's phone with EN/RO sign-up, consent, sessions, password reset, and tested backups
- [ ] **Phase 2: Profile, Goals & Safe Targets** - Onboarding that produces guarded, explainable calorie/macro targets with history, plus bodyweight trend
- [ ] **Phase 3: Food Search & Daily Meal Log** - Search EN/RO foods, log meals, see daily totals vs targets and past days (dogfooding begins)
- [ ] **Phase 4: Fast Logging & Barcode Scanning** - Barcode scan with Open Food Facts fallback, custom foods, favorites/recents, quick-add, copy meal/day
- [ ] **Phase 5: Workout Logging & Progress** - Exercise library, offline-safe session logging with rest timer, history, and per-exercise progress
- [ ] **Phase 6: Explainable AI Coach** - Deterministic daily insight cards and weekly reviews with "why", safety rules, and EN/RO text
- [ ] **Phase 7: Web Admin Curation** - Next.js admin to create, edit, and archive foods and exercises with translations

## Phase Details

### Phase 1: Walking Skeleton & Secure Accounts
**Goal**: The author can install FitAcademy on their own phone, create an account against the deployed EU backend, and stay securely signed in, in English or Romanian
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05, AUTH-06, PLAT-01, PLAT-02, PLAT-03
**Success Criteria** (what must be TRUE):
  1. On the author's phone, a new user can register with email and password against the backend running in an EU cloud region, giving a separate, explicit health-data consent during signup that they can later withdraw from settings
  2. A signed-in user stays signed in across app restarts without re-entering credentials, can log out from settings, and can regain access through an emailed password-reset link
  3. The user can switch the app between English and Romanian and every screen built so far (signup, login, consent, settings, error messages) appears fully in the chosen language; a missing translation key fails CI
  4. Attempts to read or change another user's data, or to call admin-only endpoints without the admin role, are refused, as proven by automated tests running in CI
  5. The production database is backed up nightly to off-server storage, and a restore of that backup into a fresh database has been performed and documented
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Spike first (gates the mobile stack): real-device `expo-camera` barcode scan on the author's phone using an Expo development build; fallback is Flutter + `mobile_scanner`. Decide the author's phone platform (iOS vs Android) here.
- Spike: Procrastinate + async SQLAlchemy (needs an ADR, because it departs from the vision doc's Redis job queues; fallback Taskiq/Celery + Redis). The password-reset email is the first real background job, so the worker gets proven in production during this phase.
- Lock here: repo layout from the vision doc (`apps/api`, `apps/mobile`, `apps/web-admin`, `workers/`, `infra/`, `docs/adr`); import-linter in CI; Alembic multi-schema; UoW + outbox-lite events; stable error contract (`code` + `params`); i18n key conventions with `_one/_few/_other` plurals; clock abstraction + DST fixtures (2026-10-25 Europe/Bucharest is a 25-hour day); `/api/v1` + `client-config` minimum version; typed OpenAPI client for mobile.
- ADRs: job runner, hosting (EU VPS + Docker Compose + Caddy + off-box backups), refresh-token rotation (family reuse detection, single-flight client refresh, short server grace window), admin auth/CORS (httpOnly cookies, used in Phase 7).
- Keep health payloads out of logs and error tracking from day one. In-app account deletion (ACCT-01) is v2 but blocks any store launch. Keep all module data user-scoped so a deletion cascade can be added later without restructuring.

### Phase 2: Profile, Goals & Safe Targets
**Goal**: The user completes onboarding and gets safe, explainable daily calorie and macro targets for their goal, and can track their bodyweight trend
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: PROF-01, PROF-02, PROF-03, PROF-04, PROF-05, PROF-06, PROF-07, PROF-08
**Success Criteria** (what must be TRUE):
  1. During onboarding, and later from settings, the user can set timezone, language, units, sex, height, date of birth, goal (fat loss / maintenance / muscle gain), and activity level
  2. The user sees daily calorie, protein, fat, and carb targets, plus a "how we calculated this" screen that shows each step (BMR, activity multiplier, goal adjustment, macro split)
  3. Computed targets never fall below the calorie floor or exceed the safe weekly rate of change. A user under 18 or underweight cannot pick fat loss and sees a supportive explanation instead
  4. The user can manually override calorie and macro targets. A target change applies from that day forward, and the target recorded for any earlier date stays unchanged
  5. The user can log bodyweight and see their entries with a smoothed trend line over time, in their chosen units
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Lock here: IANA timezone on the profile (drives every `log_date` from now on); targets stored as append-only history with `effective_from`.
- ADR: calorie floor (convention ~1200 kcal F / ~1500 kcal M) and weekly rate cap, with cited sources; consider a pregnancy screen.
- Target pipeline (Mifflin-St Jeor -> TDEE x activity -> goal adjustment; protein first, then fat, carbs fill) covered by hypothesis property tests.

### Phase 3: Food Search & Daily Meal Log
**Goal**: The author can log everything they eat on their phone and see how each day adds up against their targets. Daily dogfooding starts here
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: NUTR-01, NUTR-02, NUTR-03, NUTR-04, NUTR-05, NUTR-11, NUTR-12
**Success Criteria** (what must be TRUE):
  1. The user can search foods by name in English or Romanian, with or without diacritics (e.g. "branza" finds "brânză"), across a seeded catalog of generic foods and common Romanian staples
  2. The user can log a food to breakfast, lunch, dinner, or snack using a serving or a gram amount, and can edit or delete any logged entry
  3. The user sees today's calories and macros against their targets and can browse previous days, each compared against the target that was in force on that day
  4. A food logged late in the evening counts toward that local day in the user's timezone, including on DST-change days, and an entry never moves to another day after it is logged
  5. Editing a food in the catalog never changes the nutrition values or totals of entries already logged
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Lock here: nutrient snapshots on each log entry (NUMERIC, canonical g/kcal units); `log_date` frozen at write time; food provenance columns (`source`, `source_ref`, `license`, `attribution`) from the first food migration; per-entity translation tables with fallback; archive, never delete, referenced foods.
- Seed USDA FoodData Central Foundation + SR Legacy (CC0) plus a curated Romanian core set. Source adapter + Atwater sanity validator (quarantine bad rows) + golden tests.
- Search: `unaccent` + `pg_trgm`, Romanian cedilla (ş ţ) -> comma-below (ș ț) normalization; measure search latency on the deployed box.

### Phase 4: Fast Logging & Barcode Scanning
**Goal**: Logging a meal takes seconds. The user can scan a barcode, tap a favorite or recent food, quick-add numbers, or copy a previous meal or day
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: NUTR-06, NUTR-07, NUTR-08, NUTR-09, NUTR-10, BARC-01, BARC-02, BARC-03, BARC-04, BARC-05
**Success Criteria** (what must be TRUE):
  1. The user can scan a packaged product's barcode with the phone camera and log it, and common Romanian supermarket products match instantly from the local catalog
  2. A barcode missing locally is looked up in Open Food Facts through the server and is found instantly on the next scan. A barcode found nowhere opens a pre-filled form (barcode already set) to add the product
  3. The user can create custom foods with their own macros, mark foods as favorites, and sees favorite, recent, and frequent foods first when logging
  4. The user can quick-add calories and macros without choosing a food, and can copy a single meal or an entire previous day into today
  5. Each food shows its data source (Open Food Facts, USDA, or user), and an in-app attribution screen credits the data providers
**Plans**: TBD
**UI hint**: yes
**Notes**:
- ADR before importing any Open Food Facts data: ODbL share-alike stance (provenance, copy-on-write corrections, attribution; legal review before public launch).
- Bulk Romania seed from the OFF Parquet dump via DuckDB, plus a scheduled refresh; measure real coverage of complete-macro rows after the first pass (~31.9k products claimed, unverified).
- Live lookup: GTIN-13 normalization + checksum, Redis negative cache, token-bucket throttle under OFF limits (15 reads/min/IP), custom User-Agent. The app never calls OFF directly.
- Acceptance test: scan ~30 real Romanian supermarket products. Dogfood target: median meal-log time under 30 seconds.

### Phase 5: Workout Logging & Progress
**Goal**: The user can log a full gym session on their phone, even with poor signal, and see how each exercise is progressing
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: WORK-01, WORK-02, WORK-03, WORK-04, WORK-05, WORK-06, WORK-07, WORK-08, WORK-09, WORK-10
**Success Criteria** (what must be TRUE):
  1. The user can browse the exercise library filtered by muscle group and equipment, with names in English or Romanian, and can create custom exercises
  2. The user can log a session with exercises and sets (reps, weight), mark sets as warm-up or working, use a rest timer between sets, and see what they did last time for each exercise while logging
  3. A workout in progress survives the app being closed or killed and periods without connectivity, and syncs once back online without duplicating sets
  4. The user can browse workout history and start a new workout by repeating a previous one
  5. For each exercise, the user can see estimated 1RM, personal records, and volume over time
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Independent of nutrition: can run in parallel with Phases 3-4 once Phase 2 is done.
- Research before seeding: exercise dataset licensing (free-exercise-db vs wger) and how to source EN/RO names.
- Offline: `expo-sqlite` outbox on device + idempotency keys on the server. Sessions get a `log_date` from the profile timezone, same as meals.

### Phase 6: Explainable AI Coach
**Goal**: Every morning the user gets one clear, explainable insight about yesterday, and every week a review of their trends. Both come from deterministic, safe rules and appear in the user's own language
**Mode:** mvp
**Depends on**: Phase 3, Phase 5
**Requirements**: COACH-01, COACH-02, COACH-03, COACH-04, COACH-05, COACH-06, COACH-07
**Success Criteria** (what must be TRUE):
  1. Each morning in their local time, the user sees a daily insight card about the previous complete day that names one concrete change to make
  2. Each week the user receives a review covering consistency, macro balance, training volume, and bodyweight trend, with 2-3 suggestions
  3. Every recommendation can be expanded to show why it was made (the user's numbers and the versioned rule behind it), and the same logged data always yields the same stored insight
  4. Incompletely logged days produce no advice and the user is told why. Sustained very low intake produces a supportive safety message rather than praise for the deficit
  5. Coach text appears in the user's language (English or Romanian, with correct plural forms) and uses neutral, non-medical wording
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Lock here: structured insight payload (`rule_id` + version + params + evidence) rendered at read time in the user's locale, never stored as prose; immutable CoachRun/CoachInsight records (input snapshot, supersede on rerun, suppressed findings kept with a reason).
- Pure, I/O-free engine (enforced by import-linter); a FeatureBuilder is its only window into other modules. A 15-minute idempotent sweeper schedules per-user jobs in each user's local time.
- Research during planning: initial rule set (6-12 rules) and thresholds; eating-disorder-safe wording and Romanian support resources; banned-phrase template lint in CI; hypothesis + time-machine tests across DST.
- Rule-based only. The LLM text layer is v2 (COACH-10) and never makes decisions.

### Phase 7: Web Admin Curation
**Goal**: An admin can curate the food and exercise catalogs, including Romanian translations, from a browser instead of touching the database
**Mode:** mvp
**Depends on**: Phase 4, Phase 5
**Requirements**: ADMN-01, ADMN-02
**Success Criteria** (what must be TRUE):
  1. An admin can sign in to the web admin in a browser, and accounts without the admin role cannot access it
  2. An admin can create, edit, and archive foods with Romanian translations. Changes appear in mobile search, while past log entries keep their original values
  3. An admin can create, edit, and archive exercises with translations. Archived items stop appearing for new logs but stay intact in history
**Plans**: TBD
**UI hint**: yes
**Notes**:
- Next.js 16 + Tailwind 4 + shadcn, sharing the typed OpenAPI client; separate `/api/admin/v1` surface with httpOnly-cookie auth per the Phase 1 ADR.
- Can run in parallel with Phase 6. User management, the admin audit log, and the review queue for user-submitted foods are v2 (ADMN-03..05).

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 (Phase 5 may run alongside 3-4; Phase 7 may run alongside 6)

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Walking Skeleton & Secure Accounts | 0/TBD | Not started | - |
| 2. Profile, Goals & Safe Targets | 0/TBD | Not started | - |
| 3. Food Search & Daily Meal Log | 0/TBD | Not started | - |
| 4. Fast Logging & Barcode Scanning | 0/TBD | Not started | - |
| 5. Workout Logging & Progress | 0/TBD | Not started | - |
| 6. Explainable AI Coach | 0/TBD | Not started | - |
| 7. Web Admin Curation | 0/TBD | Not started | - |
