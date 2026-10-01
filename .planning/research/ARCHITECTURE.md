# Architecture Research

**Domain:** Personal fitness and nutrition tracking mobile app with a rule-based, explainable AI coach (FastAPI modular monolith)
**Researched:** 2026-10-01
**Confidence:** MEDIUM overall. The structural patterns (module boundaries, outbox, snapshotting, local-date storage) are well-established and HIGH confidence from engineering practice. Specific tool choices (job queue, boundary linter) were cross-checked via web search only. Confidence is tagged per claim, and tooling syntax needs a spike (see "Research flags").

Scope note: STACK.md owns library selection. This file recommends libraries only where they shape architecture (boundary linter, job queue, i18n mechanism).

---

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                               CLIENTS                                     │
│   ┌────────────────────────┐              ┌───────────────────────────┐  │
│   │ Mobile app (TBD fw)    │              │ Next.js web admin         │  │
│   │ typed client from      │              │ (foods, exercises, users) │  │
│   │ OpenAPI, /api/v1       │              │ /api/admin/v1             │  │
│   └───────────┬────────────┘              └─────────────┬─────────────┘  │
└───────────────┼─────────────────────────────────────────┼────────────────┘
                │ HTTPS + JWT                             │ HTTPS + JWT (role=admin)
┌───────────────▼─────────────────────────────────────────▼────────────────┐
│                  ENTRYPOINTS (delivery mechanisms, no business logic)     │
│   ┌──────────────────────────────┐   ┌────────────────────────────────┐  │
│   │ API process (FastAPI)        │   │ Worker process (same codebase) │  │
│   │ routers v1 + admin v1        │   │ scheduler + job handlers +     │  │
│   │ auth/locale/tz middleware    │   │ outbox relay                   │  │
│   └──────────────┬───────────────┘   └───────────────┬────────────────┘  │
│                  └───────────────┬───────────────────┘                   │
│                      calls ONLY module public facades                    │
├──────────────────────────────────▼───────────────────────────────────────┤
│                        DOMAIN MODULES (one Python package each)           │
│                                                                          │
│   coach ──────────────► nutrition.public / workouts.public / users.public│
│     │                                                                    │
│   barcode ───────────► nutrition.public                                  │
│   nutrition   workouts     (peers, never import each other)              │
│        │          │                                                      │
│   auth ──► users (profile, timezone, locale, goal, targets history)      │
│        │          │                                                      │
│        └────┬─────┘                                                      │
│             ▼                                                            │
│   platform  (db/UoW, event bus + outbox, i18n, errors, storage, config)  │
├──────────────────────────────────────────────────────────────────────────┤
│                               DATA LAYER                                  │
│  ┌─────────────────────┐ ┌───────────────┐ ┌───────────────────────────┐│
│  │ PostgreSQL          │ │ Redis         │ │ S3-compatible storage     ││
│  │ one schema/module   │ │ cache, rate   │ │ images/media (keys in DB) ││
│  │ + job queue tables  │ │ limit ONLY    │ │ presigned upload/download ││
│  │ + outbox_event      │ │ (not durable) │ │                           ││
│  └─────────────────────┘ └───────────────┘ └───────────────────────────┘│
└──────────────────────────────────────────────────────────────────────────┘
   External: Open Food Facts (barcode fallback, behind barcode module only)
```

Dependency direction is strictly downward. Coach is the sink: it reads everything, nothing reads coach except the API layer (and, later, notifications via events).

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| `platform` | Cross-cutting kernel: async DB session and Unit of Work, event base classes, bus, outbox relay, i18n/locale resolution, error model, S3 client, settings, clock | Plain Python package, no domain knowledge. Every module may import it. |
| `users` | User identity row, profile (timezone IANA, locale, units, birth data), goal, nutrition target history (append-only, with `effective_from`) | Owns `users.*` tables. Exposes `public.get_profile`, `public.list_active_subjects` (id, tz, locale). |
| `auth` | Credentials, JWT access + refresh rotation (hashed refresh tokens with family id for reuse detection), roles, RBAC dependencies | Depends on `users`. Exposes `CurrentUser` and `require_role` FastAPI dependencies from `platform`-level security interface. |
| `nutrition` | Food catalog (+ translations, servings, barcodes link), custom foods, favorites, meals and meal items with nutrient snapshots, daily summaries | Owns `nutrition.*`. Emits `MealLogged`, `MealUpdated`, `MealDeleted`. |
| `barcode` | Barcode lookup: local DB first, then Open Food Facts, with write-through import into the food catalog; bulk ingestion jobs | Depends on `nutrition.public.upsert_external_food`. Only module that talks to OFF. |
| `workouts` | Exercise catalog (+ translations), workout sessions, exercises-in-session, sets, history and progress queries | Owns `workouts.*`. Emits `WorkoutLogged`, `WorkoutUpdated`, `WorkoutDeleted`. |
| `coach` | Feature builder, pure rule engine, rule registry, CoachRun and CoachInsight storage, template rendering, schedule sweeper | Owns `coach.*`. Reads other modules only through their `public` query facades. |
| API entrypoint | HTTP routing, validation, auth, versioning, OpenAPI | FastAPI `apps/api`; routers are thin and call services via facades. |
| Worker entrypoint | Runs scheduled sweepers, per-user jobs, outbox relay, ingestion | Same package, separate process (`workers/` in the vision layout is a launcher, not a module). |
| Web admin | Foods, exercises (incl. translations), user management | Next.js calling `/api/admin/v1`; admin routers live inside each module and are mounted under the admin prefix. |

---

## Recommended Project Structure

Reference layout from the vision doc (`apps/api`, `apps/mobile`, `apps/web-admin`, `workers/`, `infra/`, `docs/`) is kept. Inside `apps/api`:

```
apps/api/
├── pyproject.toml                  # incl. [tool.importlinter] or .importlinter
├── alembic/                        # ONE migration history, multi-schema
│   └── versions/                   # filename prefix = module (nutrition_0003_...)
├── src/fitacademy/
│   ├── main.py                     # FastAPI app factory (composition root #1)
│   ├── worker.py                   # worker entry (composition root #2)
│   ├── bootstrap.py                # registers event subscribers + job handlers
│   ├── api/
│   │   ├── v1/router.py            # mounts each module's v1 router
│   │   └── admin_v1/router.py      # mounts each module's admin router
│   ├── platform/
│   │   ├── db.py  uow.py           # engine, session factory, UnitOfWork
│   │   ├── events/                 # DomainEvent, bus, outbox model + relay
│   │   ├── i18n/                   # locale resolution, Babel catalogs, translate()
│   │   ├── errors.py               # stable error codes + params
│   │   ├── storage.py              # S3 presign helpers
│   │   └── clock.py                # injectable now()
│   └── modules/
│       ├── users/  auth/  nutrition/  barcode/  workouts/  coach/
│       │   ├── public.py           # THE ONLY importable surface for other modules
│       │   ├── events.py           # events this module publishes (public contract)
│       │   ├── subscribers.py      # handlers for events from modules it may depend on
│       │   ├── api/                # routes_v1.py, routes_admin.py (thin)
│       │   ├── service/            # use cases; owns the transaction via UoW
│       │   ├── repository/         # SQLAlchemy queries; ALWAYS user-scoped where applicable
│       │   ├── models/             # SQLAlchemy persistence models (schema = module name)
│       │   └── schemas/            # Pydantic DTOs (request/response), never ORM objects
│       └── coach/
│           ├── engine/             # PURE: rules, scoring, selection (no I/O imports)
│           ├── features.py         # FeatureBuilder (the only place coach reads other modules)
│           ├── templates/          # template keys -> locale catalogs (en, ro)
│           └── scheduler.py        # sweeper
├── tests/
│   ├── architecture/               # import-linter run + custom invariants
│   └── modules/<module>/
└── locales/{en,ro}/LC_MESSAGES/    # backend catalogs (coach text, emails, push)
```

### Structure Rationale

- **`public.py` + `events.py` per module:** these two files are the module's whole external contract. Everything else is private by lint rule, not by convention. [HIGH, practice; see deadislove template and breadcrumbscollector article]
- **`models/` are SQLAlchemy persistence models, not a separate pure-domain layer.** A parallel DDD domain model is overkill for a solo CRUD-heavy app. Put real business rules in `service/` or in pure functions (nutrient scaling, volume calc, the coach engine). The one place a truly pure core pays off is `coach/engine/`.
- **Composition roots (`main.py`, `worker.py`, `bootstrap.py`):** subscribers are registered here, so modules never import each other just to wire events.
- **Admin routers inside each module:** admin CRUD for foods lives next to the food service; it cannot drift from validation rules. They are mounted under a separate prefix with `require_role("admin")`.
- **One Alembic history, one Postgres schema per module:** a single deploy unit means a single migration order, but schemas (`nutrition`, `workouts`, `coach`, `users`, `auth`, `platform`) make cross-module SQL joins visibly wrong and make later extraction (coach) a `pg_dump --schema` away. [HIGH, practice]

---

## Architectural Patterns

### Pattern 1: Enforced module boundaries (public facade + linter in CI)

**What:** Other modules may import only `modules.<x>.public` and `modules.<x>.events`. The dependency graph between modules is a declared DAG. Both rules are checked in CI.
**When to use:** From the first commit. Retrofitting boundaries after a year of cross-imports is the classic modular-monolith failure.
**Trade-offs:** Small ceremony (a facade file per module, DTOs returned instead of ORM rows). Pays back when the coach needs a different view of data than the UI does.

Recommended tool: **import-linter** (pure Python, contract types verified from its docs: `forbidden`, `protected`, `layers`, `independence`, `acyclic_siblings`, `custom`). Alternative to spike in Phase 0: **Tach** (Rust-based, `tach.toml`, native "public interface" declarations, explicit per-module dependency lists). Pick one in the first phase and keep it; do not run both. [MEDIUM: both confirmed to exist and match the need via web docs; Tach's maturity not independently assessed]

```ini
# .importlinter  (INI example form confirmed in import-linter docs; pyproject TOML form and
# the exact `layers` sibling syntax should be verified against the installed version)
[importlinter]
root_package = fitacademy

# 1) Module DAG: higher can import lower, never the reverse. Peers independent.
[importlinter:contract:module-dag]
name = Module dependency direction
type = layers
layers =
    fitacademy.modules.coach
    fitacademy.modules.barcode
    fitacademy.modules.nutrition | fitacademy.modules.workouts
    fitacademy.modules.auth
    fitacademy.modules.users
    fitacademy.platform

# 2) Cross-module access only via public/events (one forbidden contract per target module)
[importlinter:contract:nutrition-internals-private]
name = Nutrition internals are private
type = forbidden
source_modules =
    fitacademy.modules.coach
    fitacademy.modules.barcode
    fitacademy.modules.workouts
forbidden_modules =
    fitacademy.modules.nutrition.service
    fitacademy.modules.nutrition.repository
    fitacademy.modules.nutrition.models
    fitacademy.modules.nutrition.schemas
    fitacademy.modules.nutrition.api

# 3) Layering inside every module (one contract per module, or use `containers`)
[importlinter:contract:nutrition-layers]
name = Nutrition layers
type = layers
layers =
    fitacademy.modules.nutrition.api
    fitacademy.modules.nutrition.service
    fitacademy.modules.nutrition.repository
    fitacademy.modules.nutrition.models

# 4) Engine purity: coach engine imports nothing with I/O
[importlinter:contract:coach-engine-pure]
name = Coach engine is pure
type = forbidden
source_modules = fitacademy.modules.coach.engine
forbidden_modules =
    sqlalchemy
    fastapi
    redis
    httpx
```

Supporting rules (not lintable, enforce by review and tests):
- No SQLAlchemy `relationship()` across modules; reference by ID only. Cross-module relationships create import cycles and hidden lazy loads.
- No cross-schema JOINs in repositories. Exception: FKs from any module's `user_id` to `users.user(id)` (identity is the shared kernel; gives `ON DELETE CASCADE` for account deletion, which matters for EU users under GDPR).
- Public facades return frozen Pydantic/dataclass DTOs, never ORM instances.
- An architecture test that fails if any `modules/*/public.py` is missing or if a new module is not listed in the layers contract.

### Pattern 2: Service-owned transactions and user-scoped repositories

**What:** The service layer opens a Unit of Work (one SQLAlchemy `AsyncSession` transaction), calls repositories, collects domain events, commits. Routes never commit. Repository methods that touch user data take `user_id` as a required argument and put it in the `WHERE`.
**When to use:** Always. Satisfies the "ownership validation on all user data" constraint structurally: an unscoped `get_meal(meal_id)` simply does not exist, so IDOR is not an opt-in check.
**Trade-offs:** Slightly more verbose signatures. Admin code uses explicit `admin_*` repository methods so the escape hatch is greppable.

```python
class MealService:
    def __init__(self, uow_factory, clock, bus):
        ...

    async def log_meal(self, user_id: UUID, cmd: LogMealCommand) -> MealDTO:
        async with self._uow() as uow:
            foods = await uow.foods.get_many(cmd.food_ids)           # catalog read
            meal = Meal.create(user_id=user_id, log_date=cmd.log_date, ...)
            for item in cmd.items:
                meal.add_item(snapshot_item(foods[item.food_id], item))  # Pattern 4
            await uow.meals.add(meal)
            uow.record(MealLogged(event_id=uuid7(), user_id=user_id,
                                  meal_id=meal.id, log_date=meal.log_date))
            await uow.commit()          # writes meal + outbox rows atomically
        return MealDTO.from_model(meal)
```

Note for SQLAlchemy async: set `expire_on_commit=False`, use explicit eager loading (`selectinload`), and never rely on lazy loading in async code. [HIGH]

### Pattern 3: Event bus = in-process dispatch backed by an outbox table ("outbox-lite")

**What:** Services record events on the Unit of Work. On commit, events are inserted into `platform.outbox_event` in the SAME transaction as the business rows. After commit, the in-process dispatcher immediately delivers to registered subscribers and marks the row dispatched. A worker-side relay sweeps rows still undispatched (crash, handler error) using `SELECT ... FOR UPDATE SKIP LOCKED` and redelivers. Delivery is at-least-once; subscribers must be idempotent.
**When to use:** Recommended from day one. Cost is one table and one insert per event; benefit is that "MealLogged happened but the subscriber never ran" cannot occur silently, and the same table later feeds notifications, streaks, social feed with no redesign. No broker (Kafka, Redis Streams, RabbitMQ) is justified in a single-process monolith.
**Trade-offs:** More moving parts than a bare `dict[type, list[handler]]`. If you want the smallest v1, ship the in-process bus behind the same `uow.record()` interface and add the outbox table in the next phase; the call sites do not change. But do not skip idempotent handlers. [MEDIUM: pattern HIGH, "outbox-lite" hybrid is a recommendation; outbox + SKIP LOCKED mechanics confirmed by multiple sources]

Rules:
1. **Thin events:** carry IDs plus the few facts routers need (`user_id`, `meal_id`, `log_date`). Consumers call the producer's `public` facade for details. Prevents event payloads becoming a second, unversioned API.
2. **Events are a public contract:** live in `<module>/events.py`, include `schema_version`, are only ever extended additively once persisted.
3. **Subscribers live in the consumer module** (`coach/subscribers.py` imports `nutrition.events`), registered in `bootstrap.py`. Direction of the import equals direction of dependency.
4. **Never run handlers before commit and never inside the producer's transaction.** Handler failure must not roll back or fail the user's meal log.
5. **Idempotency:** handlers key on `event_id` (a `platform.event_delivery(event_id, handler)` unique row, or naturally idempotent upserts).
6. **Do not use FastAPI `BackgroundTasks` or bare `asyncio.create_task` for events:** both are lost on crash/restart and have no retry. [HIGH]

What consumes events in v1 (be honest about this in the roadmap): (a) coach marks `(user_id, log_date)` stale so a later regeneration is offered; (b) cache invalidation of daily summary if Redis caching is added. Everything else (streaks, feed, notifications) is future. That is why the bus must exist but must stay tiny.

### Pattern 4: Nutrient snapshotting at log time (reference AND snapshot)

**What:** A `meal_item` stores `food_id` (nullable FK, `ON DELETE SET NULL`, for navigation and "log again") AND an immutable snapshot of what the user actually ate: name, per-100 g nutrients at that moment, the entered quantity, and computed absolute nutrients for that quantity.
**Why:** (1) Admin corrections and Open Food Facts refreshes must not silently rewrite a user's history or last week's coach inputs. (2) Daily totals become a plain `SUM` over one table with no join, and coach reads need no catalog access. (3) Auditability: a CoachRun's inputs are reproducible. (4) Foods can be archived or merged without breaking history.
**Trade-offs:** Duplicated data (trivial size). Genuine data-error corrections do not propagate automatically; provide an explicit admin "recompute affected meal items" job when a catalog error is serious, never an implicit join.

```sql
-- nutrition schema (sketch)
CREATE TABLE nutrition.food (
  id uuid PRIMARY KEY,
  source text NOT NULL,                 -- 'off' | 'usda' | 'admin' | 'user'
  source_id text,                       -- external id; UNIQUE(source, source_id) where not null
  owner_user_id uuid,                   -- NULL = global catalog, set = private custom food
  brand text,
  default_locale text NOT NULL DEFAULT 'en',
  -- per-100 g, canonical units: kcal, g, mg (convert kJ -> kcal at import)
  kcal numeric(9,3) NOT NULL, protein_g numeric(9,3), carbs_g numeric(9,3),
  fat_g numeric(9,3), fiber_g numeric(9,3), sugar_g numeric(9,3), sodium_mg numeric(9,3),
  micronutrients jsonb NOT NULL DEFAULT '{}',   -- long tail, keyed, fixed canonical unit per key
  is_verified boolean NOT NULL DEFAULT false,
  archived_at timestamptz, updated_at timestamptz NOT NULL
);
CREATE TABLE nutrition.food_translation (
  food_id uuid REFERENCES nutrition.food ON DELETE CASCADE,
  locale text, name text NOT NULL, search_text text NOT NULL,   -- normalized (see Pattern 9)
  PRIMARY KEY (food_id, locale)
);
CREATE TABLE nutrition.food_serving (            -- "1 slice" = 30 g
  id uuid PRIMARY KEY, food_id uuid REFERENCES nutrition.food, grams numeric(9,3) NOT NULL,
  label_key text, is_default boolean NOT NULL DEFAULT false
);
CREATE TABLE nutrition.food_barcode (barcode text PRIMARY KEY, food_id uuid NOT NULL REFERENCES nutrition.food);

CREATE TABLE nutrition.meal (
  id uuid PRIMARY KEY,                           -- may be client-generated (idempotent retries)
  user_id uuid NOT NULL REFERENCES users.user ON DELETE CASCADE,
  log_date date NOT NULL,                        -- diary day, user-chosen (Pattern 6)
  meal_type text NOT NULL,                       -- breakfast|lunch|dinner|snack (closed vocab)
  logged_at timestamptz NOT NULL DEFAULT now(),
  note text
);
CREATE INDEX ON nutrition.meal (user_id, log_date);

CREATE TABLE nutrition.meal_item (
  id uuid PRIMARY KEY,
  meal_id uuid NOT NULL REFERENCES nutrition.meal ON DELETE CASCADE,
  food_id uuid REFERENCES nutrition.food ON DELETE SET NULL,
  quantity_g numeric(9,3) NOT NULL,              -- canonical amount
  serving_id uuid, serving_qty numeric(9,3),     -- what the user entered, for display/edit
  food_snapshot jsonb NOT NULL,                  -- {name, brand, source, per_100g{...}, food_updated_at}
  kcal numeric(9,2) NOT NULL, protein_g numeric(9,2), carbs_g numeric(9,2),
  fat_g numeric(9,2), fiber_g numeric(9,2), sugar_g numeric(9,2), sodium_mg numeric(9,2),
  nutrients_extra jsonb NOT NULL DEFAULT '{}'
);
```

Design notes:
- **Typed columns for the macros + JSONB for the long tail.** Coach and summaries hit macros constantly (fast `SUM`, simple indexes); a normalized USDA-style nutrient dictionary + `food_nutrient` EAV table is more rigorous but overkill for v1. If micronutrient tracking becomes a feature, migrate the JSONB keys then. [MEDIUM, recommendation]
- Editing a quantity recomputes absolute values from `food_snapshot.per_100g` (not from the live food).
- Store energy as kcal; Open Food Facts exposes `*_100g` fields (energy, fat, saturated fat, carbs, sugars, proteins, fiber, salt) normalized per 100 g/ml. Salt vs sodium and kJ vs kcal conversion happen in the import adapter, once. [MEDIUM, OFF data model confirmed via search]
- `meal` is a container for one logging act/slot; it exists mainly so `MealLogged` has a natural aggregate. Do not build recipes or meal plans (out of scope).
- Barcodes are a separate table (many barcodes per food; OFF imports).
- Custom foods are rows in the same table with `owner_user_id`; search queries filter `owner_user_id IS NULL OR owner_user_id = :user`. Never let another user's custom foods appear.

### Pattern 5: Exercises, sessions, sets

```sql
CREATE TABLE workouts.exercise (
  id uuid PRIMARY KEY, slug text UNIQUE NOT NULL,            -- stable key, never reused
  owner_user_id uuid,                                        -- NULL = global
  primary_muscle text NOT NULL, equipment text NOT NULL,     -- closed vocab keys (client-translated)
  secondary_muscles text[] NOT NULL DEFAULT '{}',
  logging_type text NOT NULL,        -- weight_reps | reps_only | duration | distance_duration
  media_key text, archived_at timestamptz
);
CREATE TABLE workouts.exercise_translation (
  exercise_id uuid REFERENCES workouts.exercise ON DELETE CASCADE, locale text,
  name text NOT NULL, instructions text, search_text text NOT NULL,
  PRIMARY KEY (exercise_id, locale)
);
CREATE TABLE workouts.workout_session (
  id uuid PRIMARY KEY, user_id uuid NOT NULL REFERENCES users.user ON DELETE CASCADE,
  log_date date NOT NULL, started_at timestamptz NOT NULL, ended_at timestamptz,
  name text, notes text, status text NOT NULL            -- in_progress | completed
);
CREATE INDEX ON workouts.workout_session (user_id, log_date);
CREATE TABLE workouts.workout_exercise (
  id uuid PRIMARY KEY, session_id uuid REFERENCES workouts.workout_session ON DELETE CASCADE,
  exercise_id uuid NOT NULL REFERENCES workouts.exercise ON DELETE RESTRICT,
  position int NOT NULL, exercise_name_snapshot text NOT NULL, notes text
);
CREATE TABLE workouts.workout_set (
  id uuid PRIMARY KEY, workout_exercise_id uuid REFERENCES workouts.workout_exercise ON DELETE CASCADE,
  set_index int NOT NULL, set_type text NOT NULL DEFAULT 'working',  -- warmup|working|drop|failure
  reps int, weight_kg numeric(7,3), duration_s int, distance_m int, rpe numeric(3,1),
  completed_at timestamptz
);
```

- **Canonical units in storage (kg, m, s, g, kcal); unit preference is display-only** (profile `units`). Converting at write and at read once avoids mixed-unit history.
- **Exercises are never hard-deleted:** archive (`archived_at`) and merge via an admin job that re-points `workout_exercise.exercise_id`. `ON DELETE RESTRICT` enforces this.
- **Derived metrics are computed, not stored:** volume = sum(reps x weight), estimated 1RM, personal records. Cache only if profiling shows a need.
- **`logging_type` drives validation and UI** (a plank has no reps/weight; a run has distance/duration). Retrofitting this is painful; include it from the start. [MEDIUM, domain practice]
- `log_date` = local date of `started_at` (a session crossing midnight belongs to the start day).
- Accept client-generated UUIDs as primary keys for meals, sessions, sets so a retry after a flaky gym connection is idempotent (`INSERT ... ON CONFLICT DO NOTHING`, then return existing). Prefer UUIDv7 for index locality. [MEDIUM]

### Pattern 6: Timezone-correct "daily" via a stored diary date

**What:** Three rules.
1. Store instants as `timestamptz` (UTC) in `logged_at`, `started_at`, `completed_at`.
2. Store a **`log_date date`** on every loggable aggregate, frozen at write time. For meals it is the diary day the user is logging to (users backfill yesterday's dinner; the client sends the date they are looking at). For workouts it is the local date of `started_at`. All "daily" aggregation is `GROUP BY log_date`.
3. The profile holds the IANA zone (`Europe/Bucharest`). It determines "today" defaults, server-side job timing, and weekly boundaries. It is NOT used to re-derive dates from timestamps at query time.
**Why:** Recomputing with `AT TIME ZONE profile.tz` at query time reshuffles history when a user travels or changes zone. Sources on Postgres multi-timezone grouping recommend keeping `timestamptz` as truth, storing the zone, and resolving the local date once at write time. [MEDIUM, web sources agree; matches common diary-app behavior]
**Trade-offs:** A client bug could send a wrong date; validate server-side (`log_date` within, say, today-30d..today+1d in the profile zone) and reject the rest with a stable error code.

Related decisions:
- **Targets are history, not a single row:** `users.nutrition_target(user_id, effective_from date, kcal, protein_g, carbs_g, fat_g, goal)`, append-only. Adherence for 2026-09-20 is judged against the target in force then, and the coach run snapshots the target it used.
- **Weeks:** ISO weeks, Monday start by default (store `week_start` in profile only if users ask). Weekly runs key on `period_start` = Monday local date.
- **DST:** compute "is it past 06:00 for this user?" with `zoneinfo` in Python, never with fixed UTC offsets. Install the `tzdata` package in slim container images, otherwise `zoneinfo` fails with no system tz database. [HIGH]
- **Zone changes:** client detects device zone change and prompts to update the profile; past `log_date`s are unaffected by design.
- **Daily summary:** compute on read (`SELECT sum(...) FROM meal_item JOIN meal ... WHERE user_id=:u AND log_date BETWEEN ...`). At roughly 10-20 items per user-day this is trivially fast with `(user_id, log_date)` indexes. Do NOT build a `daily_summary` projection table in v1; add it only when measured. This also means `MealLogged` has no mandatory v1 projection consumer.

### Pattern 7: Coach = pure engine + versioned rules + immutable run/insight records

**What:** Separate three things that tutorials blur together.

```
FeatureBuilder (I/O)  ->  Engine (pure, deterministic)  ->  Persistence + Rendering
reads nutrition/          evaluate(features, ruleset)       CoachRun + CoachInsight rows,
workouts/users via        -> list[Finding]                  template_key + params,
public facades            select(findings) -> top N         rendered per locale at read time
```

- `CoachFeatures` is a frozen, serializable dataclass: targets in force, daily totals for the window, training sessions/volume/frequency, days-logged count, goal, data-sufficiency flags. It is the only input the engine sees.
- `engine.evaluate(features, ruleset)` has no clock, no DB, no randomness. Same input + same ruleset version -> identical output. Enforced by the import-linter purity contract and by golden-file tests.
- A **rule** is code with metadata: `rule_id` (`PROTEIN_LOW`), `version` (int, bump on any threshold or logic change), `category`, `min_data` requirements, `safety_tier`, and `evaluate(features) -> Finding | None`. Thresholds live in versioned code/config, not admin-editable DB rows in v1 (editable thresholds destroy replayability).
- A **ruleset version** (`2026.10.1`) identifies the registered rule set; the run also stores `engine_version` (git SHA at deploy).
- **Selection** (rank, cooldown so the same advice is not repeated three days running, cap at 1 card/day and 2-3 weekly suggestions) is part of the engine and is also recorded. Store ALL fired findings with `selected` and `suppressed_reason`; "why wasn't I told X?" is as auditable as "why was I told Y?".
- **Insufficient data is a first-class outcome** (`status='skipped'`, reason `insufficient_data`), producing at most a "log your meals" nudge. Prevents bogus advice from one logged snack.
- **Store keys and params, not rendered prose:** `template_key`, `template_params`, `explanation_key`, `action_key`. Render with the user's current locale at read time. This is what makes English/Romanian switching, template edits, and the future LLM presentation layer safe (the LLM would consume the structured insight, and store its output in a separate rendering table with `prompt_template_version`).

```sql
CREATE TABLE coach.coach_run (
  id uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users.user ON DELETE CASCADE,
  kind text NOT NULL CHECK (kind IN ('daily','weekly')),
  period_start date NOT NULL, period_end date NOT NULL,   -- local dates, inclusive
  timezone text NOT NULL,                                 -- zone used for this run
  trigger text NOT NULL,                                  -- schedule | manual | backfill
  status text NOT NULL,                                   -- pending|running|succeeded|skipped|failed
  ruleset_version text NOT NULL, engine_version text NOT NULL,
  inputs jsonb NOT NULL,                                  -- serialized CoachFeatures (full snapshot)
  inputs_hash text NOT NULL,
  error text, scheduled_for timestamptz, started_at timestamptz, finished_at timestamptz,
  superseded_by uuid REFERENCES coach.coach_run,
  created_at timestamptz NOT NULL DEFAULT now()
);
-- THE idempotency lock: exactly one live run per user/kind/period
CREATE UNIQUE INDEX coach_run_one_live
  ON coach.coach_run (user_id, kind, period_start) WHERE superseded_by IS NULL;

CREATE TABLE coach.coach_insight (
  id uuid PRIMARY KEY, run_id uuid NOT NULL REFERENCES coach.coach_run ON DELETE CASCADE,
  user_id uuid NOT NULL,                                  -- denormalized for scoped reads
  rule_id text NOT NULL, rule_version int NOT NULL,
  category text NOT NULL, severity text NOT NULL, priority int NOT NULL,
  selected boolean NOT NULL, suppressed_reason text,
  evidence jsonb NOT NULL,       -- {metric, actual, target, deviation, window_days, days_with_data}
  template_key text NOT NULL, template_params jsonb NOT NULL,
  explanation_key text NOT NULL, action_key text, action_params jsonb,
  user_state text NOT NULL DEFAULT 'unseen',              -- unseen|seen|dismissed
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON coach.coach_insight (user_id, created_at DESC) WHERE selected;
```

- **Immutability:** runs and insights are never updated except `user_state` and run status transitions. A re-run (rules improved, late edits to an old day) creates a NEW run and sets `superseded_by` on the old one. History of what the user was actually shown stays intact.
- **Snapshot inputs inside the run** so replay needs neither the live data nor the old catalog values. A replay test loads `inputs`, runs the ruleset named in `ruleset_version`, and asserts the same findings. This requires keeping old rule versions importable (or at minimum recording the git SHA); decide the retention policy in the coach phase.
- **Every insight carries its "why":** `evidence` is structured (actual vs target vs window), and `explanation_key` renders it ("You averaged 62 g protein over 5 logged days; your target is 120 g"). That satisfies the "every recommendation explains why" requirement by construction, since a rule cannot emit a Finding without evidence.
- **Safety:** rules are whitelisted outputs; calorie targets have hard floors in the users/target logic, not in coach prose; no medical wording in templates; coach never writes to nutrition/workouts/users. [MEDIUM, aligned with project constraint]
- Later hook: coach emits `CoachInsightCreated` through the same outbox, so a future notifications module can push without coach knowing about it.

### Pattern 8: Scheduling = frequent idempotent sweeper, fan-out per user (not a cron per user)

**What:** One periodic job (every 15 minutes) lists active users, decides in Python which are "due" in THEIR local time, and attempts to claim `(user, kind, period)` by inserting a `coach_run` row (`ON CONFLICT DO NOTHING` against the partial unique index). Only inserts that succeed enqueue a per-user job. The per-user job loads the run, builds features, evaluates, persists insights, marks success.

```python
async def coach_sweeper(now_utc: datetime):
    for s in await users.public.list_active_subjects():          # (user_id, tz, locale)
        local = now_utc.astimezone(ZoneInfo(s.tz))
        if local.time() >= DAILY_INSIGHT_LOCAL_TIME:
            period = local.date() - timedelta(days=1)            # previous complete local day
            await coach.service.claim_and_enqueue(s.user_id, "daily", period, tz=s.tz)
        if local.weekday() == 0 and local.time() >= WEEKLY_INSIGHT_LOCAL_TIME:
            await coach.service.claim_and_enqueue(s.user_id, "weekly",
                                                  local.date() - timedelta(days=7), tz=s.tz)
    # catch-up: also consider the last N (e.g. 3) missed days, never older
```

**Why this shape:**
- Self-healing: worker down for six hours -> next sweep catches up. Duplicate sweeps or two workers -> the unique index absorbs it. No exactly-once queue needed, only an idempotency key.
- DST/travel-safe because "due" is computed from `zoneinfo`, not stored UTC cron strings.
- Per-user retries are isolated: one user's failing job does not block the batch. Failed runs stay visible as `status='failed'` with `error` for debugging.
- No cross-module SQL: the sweeper asks `users.public` for subjects instead of joining `users.profile` itself.
- Run on the PREVIOUS complete local day at the user's morning (inputs immutable, no half-day advice). If product wants an evening "today so far" card, add it as a separate run kind with `period_complete=false`; this is a product decision flagged for the coach phase.

**Job runner choice (affects the worker phase):** Recommend **Procrastinate** (Postgres-backed queue; periodic tasks, retries, locks; no extra broker) as the default, with **Celery + Redis** as the conventional fallback. Reasoning: jobs are durable alongside the data and backups; Redis stays a disposable cache/rate-limit store instead of a durable-queue dependency; arq is in maintenance-only mode (maintainer issue #510), so avoid it for a new long-lived project. Caveat to spike: Procrastinate's SQLAlchemy connector is psycopg2-based and for deferring only; workers use their own psycopg pool, and atomic "defer in the same transaction as business data" is only simple if you share a psycopg connection. Because the sweeper is idempotent, you do NOT need transactional enqueue for coach jobs; use the outbox for events and the sweeper for coach. [MEDIUM: capabilities confirmed from Procrastinate docs and search; async-SQLAlchemy interplay not verified, flagged for Phase 0 spike]

Other workers on the same runner: outbox relay sweep, barcode/OFF ingestion, expired refresh-token cleanup, orphaned S3 object cleanup.

### Pattern 9: i18n data model (closed vocab in catalogs, open content in translation tables)

**What:** Two different problems, two mechanisms.

| Kind of text | Examples | Mechanism |
|--------------|----------|-----------|
| Closed, developer-owned vocabulary | muscle groups, equipment, meal types, goals, units, error codes, coach categories | API returns stable **keys**; mobile and admin translate from their own catalogs. Backend catalogs only where backend renders text (coach, email, push). |
| Open, admin/user/import-owned content | food names, brands (sometimes), exercise names and instructions | **Per-entity translation table** `(entity_id, locale, name, search_text)` with composite PK. |
| Coach text | insight headline, explanation, action | Stored as `template_key + params`, rendered at read time in the user's locale (Pattern 7). |

- **Prefer per-entity translation tables** over a generic polymorphic `translations(entity_type, entity_id, field, ...)` (no FKs, awkward queries) and over a JSONB `names` column (hard to index for search, no per-locale uniqueness). The per-entity table gives FK integrity, simple admin CRUD, and a per-locale search index. [HIGH, standard practice]
- **Fallback chain:** requested locale -> language-only -> `food.default_locale` -> any available. Implement once in the repository query/service helper; never in routers.
- **Search must span locales.** A Romanian user may type "pui" while the UI is English, and OFF imports carry whatever language the product was entered in. Search all translations, rank the user's locale first.
- **Search normalization:** store `search_text` = lowercase + `unaccent`ed + trimmed, index with `pg_trgm` GIN. Romanian pitfall: legacy cedilla forms (`ş`, `ţ`) vs correct comma-below (`ș`, `ț`) are distinct code points in user input and imported data; normalize both to the same folded form before indexing and before querying. [MEDIUM]
- **Plurals:** Romanian has three CLDR plural categories (one, few, other). Do NOT hand-roll `n == 1` logic in coach templates. Use gettext/Babel catalogs with proper plural forms or ICU-style plural messages; test Romanian templates with 1, 2, 19, 20, 21 values. [HIGH on the plural rule; Babel choice is MEDIUM, defer final library to STACK.md]
- **Locale resolution order:** `users.profile.locale` > `Accept-Language` > `en`. Unauthenticated endpoints use the header only.
- **Do not localize numbers or units server-side:** API sends canonical numbers and keys; clients format (Romanian decimal comma, kg vs lb). Fewer bugs, smaller API.
- **Admin UX:** a food/exercise cannot be published without at least the default-locale name; missing `ro` name is allowed and falls back to `en` (flag "needs translation" in admin).

### Pattern 10: API versioning and mobile compatibility

**What:** Mobile clients cannot be force-updated instantly (store review lag, users who never update), so the API is a long-lived contract.
- **URL major version:** `/api/v1/...` for the app, `/api/admin/v1/...` for the web admin (separate CORS and auth policy; admin can break independently since it deploys with the backend).
- **Additive-only inside a major version:** new optional fields, new endpoints, new optional query params. Never rename, remove, retype, or tighten validation in v1. [MEDIUM: consistent with sources]
- **Clients MUST tolerate unknown fields and unknown enum values** (coach `category`, `severity`, `meal_type`, `logging_type` will grow). Document this as a client contract and test it in the mobile app.
- **Routers per version, schemas versioned lazily:** each module has `api/routes_v1.py`; `api/v1/router.py` mounts them. Create `routes_v2.py` and v2 schemas only for the routes that actually break, mounting both. Do not copy the whole API on day one.
- **Client compatibility channel:** client sends `X-App-Version`, `X-Platform`, `Accept-Language`; server exposes `GET /api/v1/meta/client-config` returning `{min_supported_version, latest_version, update_url, message_key}`. Below minimum -> `426 Upgrade Required` with a stable error code. Use force-update sparingly (security/data-corruption fixes). Support N-1 major versions; announce removals with `Deprecation` and `Sunset` response headers. [MEDIUM: pattern from mobile API-versioning guidance]
- **Contract safety net:** commit the generated OpenAPI schema, run a breaking-change diff (e.g. `oasdiff`) in CI against the last released schema, and generate the typed mobile client from it (works for any framework chosen: TS client for React Native, generated Dart for Flutter).
- **Stable error contract:** `{ "error": { "code": "meal.log_date_out_of_range", "params": {...}, "request_id": "..." } }`. Clients translate codes; server messages are for logs.
- **Pagination:** cursor-based for history lists (`log_date desc, id`); never offset on growing logs.
- **Idempotency:** accept `Idempotency-Key` or client-generated UUID ids on create endpoints (meals, sessions, sets).

---

## Data Flow

### Request Flow

```
Mobile: POST /api/v1/nutrition/meals
    │  headers: Authorization, Accept-Language, X-App-Version
    ▼
Middleware: request_id, version gate (426?), locale resolve, rate limit (Redis)
    ▼
Router (nutrition/api/routes_v1)  -- validate Pydantic schema, CurrentUser dependency
    ▼
MealService.log_meal(user_id, cmd)  -- UoW begin
    ├─► foods repo: read catalog rows (nutrition schema)
    ├─► build snapshots, compute absolute nutrients
    ├─► meals repo: insert meal + items (user-scoped)
    └─► uow.record(MealLogged); commit  (meal rows + outbox_event atomic)
    ▼
After commit: in-process dispatcher -> subscribers (idempotent)
    ▼
Response: MealDTO (+ updated day totals optionally)
```

### State Management

Server is the source of truth; the mobile app caches the diary for the viewed date and sends writes with client-generated ids so a retry is safe. Coach cards are read-only server state (`user_state` is the only mutable field).

### Key Data Flows

1. **Log a meal:** as above. `MealLogged` carries `(user_id, meal_id, log_date)`. Coach subscriber marks the day stale (idempotent upsert); nothing else is required in v1.
2. **Log a workout:** session created `in_progress`, exercises and sets appended (each set write idempotent), session completed -> `WorkoutLogged`. Same outbox path.
3. **Daily insight:** worker sweeper (every 15 min) -> for each user due in local time, claim `coach_run(user, 'daily', yesterday)` via unique index -> enqueue job -> FeatureBuilder reads `nutrition.public.get_daily_totals`, `workouts.public.get_sessions_summary`, `users.public.get_targets_for(range)` -> engine -> persist findings/insights, snapshot `inputs` -> run `succeeded` (or `skipped`). Mobile `GET /api/v1/coach/insights/today` reads selected insights and renders templates in the user's locale.
4. **Weekly review:** same machinery, `kind='weekly'`, Monday local, window = previous Monday-Sunday; feature builder aggregates the daily data over 7 days.
5. **Barcode scan:** `GET /api/v1/barcode/{code}` -> barcode service checks `nutrition.public.find_by_barcode` -> miss -> OFF client (custom User-Agent, timeout, circuit breaker) -> map to canonical nutrients -> `nutrition.public.upsert_external_food` (write-through) -> return food. Negative lookups are cached briefly in Redis so a missing barcode does not hammer OFF.
6. **Admin edits a food:** admin router -> `FoodAdminService.update` (same validation as user custom foods) -> new catalog values affect only FUTURE logs; history keeps snapshots. Optional "recompute meal items" job for serious data errors.
7. **Account deletion:** delete user -> `ON DELETE CASCADE` across module tables -> `UserDeleted` event -> worker removes S3 media keys. Do this properly early (EU users).

---

## Build Order (dependencies between components)

```
Phase 0  Platform skeleton ─┬─► boundary linter + architecture tests in CI
                            ├─► DB session/UoW, Alembic multi-schema, error model, i18n infra
                            ├─► event bus interface + outbox table + relay
                            ├─► job runner + empty scheduler, worker entrypoint
                            └─► /api/v1 + /api/admin/v1 skeleton, OpenAPI export, deploy pipeline
Phase 1  users + auth ───────► profile (tz, locale, units), targets history, RBAC
                               (deploy the walking skeleton here so dogfooding can start early)
Phase 2  nutrition catalog ──► food + translations + servings + search + admin CRUD (+ seed import)
Phase 3  nutrition logging ──► meal/meal_item snapshots, diary + daily summary, favorites,
                               custom foods; emits MealLogged
Phase 4  workouts ───────────► exercise catalog (+translations, admin), sessions/sets, history/progress;
                               emits WorkoutLogged   (independent of 2-3; can run in parallel)
Phase 5  barcode ────────────► needs Phase 2 (+3 to be useful end-to-end); OFF adapter, ingestion job
Phase 6  coach ──────────────► needs 1, 3, 4. Order inside: FeatureBuilder + public facades ->
                               pure engine + 3-5 rules -> CoachRun/Insight storage ->
                               template rendering (en/ro) -> sweeper + daily -> weekly ->
                               cooldown/selection tuning
Phase 7  web admin UI ───────► needs admin routers from 1-4; UI work can start once Phase 2 lands
Phase 8  hardening ──────────► rate limiting policy, backups/restore drill, observability, GDPR export/delete
```

Ordering rationale:
- The linter, UoW, outbox table and i18n infrastructure must exist BEFORE the first module. They are nearly free at the start and expensive to retrofit.
- Users/targets come before nutrition because every daily total is judged against a target and every date depends on the profile timezone.
- Nutrition logging and workouts both precede coach: coach needs real logged data to be testable, and the public query facades they expose are coach's only input.
- Barcode is a leaf: it depends on the catalog but nothing depends on it. It can slip without affecting the core loop.
- Admin API endpoints are built WITH each module (catalogs need curation to be usable); only the Next.js UI is deferred.

**Research flags for phases:**
- Phase 0: spike import-linter vs Tach on a 3-module toy; spike Procrastinate with async SQLAlchemy/asyncpg (shared pool, worker pool, periodic task, outbox relay) vs Celery. Needs hands-on verification.
- Phase 2: food data source decision (Open Food Facts vs USDA FoodData Central, licensing, Romanian coverage, per-100 g normalization) lives in other research; the import adapter design depends on it.
- Phase 6: rule catalog content and thresholds (nutrition science plus safety review) needs its own research; architecture here is settled, content is not.
- Phases 1, 3, 4, 7, 8: standard patterns, unlikely to need deeper research.

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-1k users (portfolio, friends) | One API container, one worker container, one Postgres, small Redis. Everything above fits. No read replicas, no projections, no partitioning. |
| 1k-100k users | Add indexes driven by `EXPLAIN` (`meal(user_id, log_date)` already). Sweeper may batch and run per-timezone buckets. Cache hot catalog search and daily summaries in Redis (invalidate on `MealLogged`). Run multiple API replicas and 2+ workers (idempotent design already allows it). |
| 100k+ users | Partition `meal_item`/`workout_set` by range on `log_date` (or hash on user). Add read replica for history/progress queries. Extract the coach as its own worker service: it already has its own schema, a pure engine, and reads only via facades (turn facades into internal HTTP/gRPC or replicate via outbox). |

### Scaling Priorities

1. **First bottleneck:** food search latency (trigram search over a large OFF-derived catalog with multi-locale rows). Fix with scoped indexes, limit search to verified or locale-relevant rows by default, then a dedicated search index only if measured.
2. **Second bottleneck:** coach sweep cost (one job per user per day plus weekly). Fix with timezone-bucketed sweeps, job concurrency limits, and skipping users with no activity in the window.

Data volume reality check: about 15 meal items per user-day is roughly 5k rows per user-year. Even 10k active users is about 55M rows/year, which Postgres handles without partitioning for a long time.

---

## Anti-Patterns

### Anti-Pattern 1: Reference-only meal items (join to live food row)

**What people do:** `meal_item(food_id, grams)`, with totals computed by joining the catalog.
**Why it's wrong:** An admin typo fix or OFF refresh silently rewrites every past day and every stored coach input. Archiving or merging foods breaks history. Coach needs a catalog join.
**Do this instead:** Pattern 4: keep `food_id` for navigation, snapshot per-100 g + computed absolutes at log time.

### Anti-Pattern 2: Deriving daily buckets from `timestamptz` with the user's current timezone at query time

**What people do:** `date_trunc('day', logged_at AT TIME ZONE profile.tz)` in every summary query.
**Why it's wrong:** Changing zone or travelling moves meals between days; users cannot log to a previous day; DST edge bugs in aggregations.
**Do this instead:** Pattern 6: frozen `log_date` per row, explicit from the client for meals.

### Anti-Pattern 3: Handlers executed inside the request transaction (or fire-and-forget tasks)

**What people do:** Call subscribers synchronously before commit, or `asyncio.create_task` / `BackgroundTasks` for "events".
**Why it's wrong:** A subscriber bug fails the user's meal save; or work is silently lost on a crash/redeploy; no retries; hidden coupling and ordering dependencies.
**Do this instead:** Pattern 3: outbox row in the same commit, dispatch after commit, relay for retries, idempotent handlers.

### Anti-Pattern 4: Cron per user, or one big nightly UTC job

**What people do:** Schedule insights at 03:00 UTC for everyone, or create per-user schedules.
**Why it's wrong:** Users get yesterday's card before their day ends (or hours late); DST breaks schedules; a single failure blocks everyone; per-user cron entries are unmanageable.
**Do this instead:** Pattern 8: frequent idempotent sweeper with a unique-index claim.

### Anti-Pattern 5: Coach reads other modules' tables or ORM models directly

**What people do:** `from modules.nutrition.models import MealItem` inside coach for "just one query".
**Why it's wrong:** Boundary erosion in week two; schema changes in nutrition break coach silently; extraction later becomes a rewrite.
**Do this instead:** `nutrition.public` DTO facades consumed only by `coach/features.py`; linter forbids anything else.

### Anti-Pattern 6: Storing rendered coach prose (or mutating insights in place)

**What people do:** Save the English sentence in `insight.text`; "fix" a bad insight by updating the row.
**Why it's wrong:** Cannot localize, cannot re-render after template fixes, cannot audit what was said, cannot compare rule versions.
**Do this instead:** Pattern 7: keys + params + structured evidence; new run supersedes old; immutable history.

### Anti-Pattern 7: Admin-editable coach thresholds in the DB (v1)

**What people do:** Put rule thresholds in a settings table so they "can tune without deploy".
**Why it's wrong:** Destroys replayability and ruleset versioning; changes are unreviewed health-adjacent logic.
**Do this instead:** Thresholds in versioned code/config; bump `ruleset_version`; revisit when there is a review workflow.

### Anti-Pattern 8: Redis as the durable queue or source of truth

**What people do:** Put job state or event delivery only in Redis.
**Why it's wrong:** Persistence misconfiguration or eviction loses jobs/events silently; second place to back up.
**Do this instead:** Postgres owns durable state (jobs, outbox, runs). Redis = cache, rate limiting, short-TTL negative lookups.

### Anti-Pattern 9: Hand-rolled plural and unit logic in templates, localized numbers from the server

**What people do:** `"day" if n == 1 else "days"` and pre-formatted "1.234,5 kcal" strings in API responses.
**Why it's wrong:** Romanian breaks on `few` forms; clients cannot reformat; stale strings after locale change.
**Do this instead:** Catalog plurals via gettext/Babel, send canonical numbers and keys, format on the client.

### Anti-Pattern 10: Premature DDD ceremony and speculative modules

**What people do:** Separate domain/ORM model hierarchies, generic repositories, stub modules for social/gamification/wallet.
**Why it's wrong:** Triples the code for a solo developer; stub modules rot and dilute the boundary rules.
**Do this instead:** Persistence models + services + pure functions where logic is non-trivial; add a module (with its linter contract) only when its phase starts.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Open Food Facts | `barcode` module adapter: httpx client, custom User-Agent, short timeout, retry/backoff, circuit breaker; map to canonical per-100 g schema; write-through into `nutrition.food` | Respect their published rate limits and terms; cache negative results; keep attribution/licence requirements from the licence (ODbL) in mind for stored data. Bulk dataset import is an ingestion job, not a request path. Source decision is in FEATURES/STACK research. |
| S3-compatible storage | Presigned PUT/GET issued by `platform.storage`; DB stores keys only; lifecycle rules for temp uploads | Validate content type and size on the presign request; clean orphans via worker. |
| Redis | Rate limiting, catalog/summary cache, negative barcode cache | Treat as disposable; app must function (slower) with it cold. |
| Push / email (later) | `notifications` module subscribes to `CoachInsightCreated` etc. via outbox | Out of v1; keep events thin so this drops in. |
| LLM (later) | Presentation layer on structured insights; separate rendering table with `prompt_template_version` | Never feeds decisions back into the engine. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| api -> module | Direct call to `service` (same module), or to `public` facade (other module) | Routers thin; no repository access from routers. |
| coach -> nutrition / workouts / users | Synchronous in-process calls to `public` read facades, DTOs out | Only in `coach/features.py`. |
| nutrition / workouts -> coach | Events only (`MealLogged`, `WorkoutLogged`) via outbox | Producer does not know consumers. |
| barcode -> nutrition | `nutrition.public.upsert_external_food` / `find_by_barcode` | One-way. |
| auth -> users | `users.public` | Credentials separate from profile. |
| any module -> platform | Direct import (UoW, events, i18n, errors, storage) | Platform has no domain imports (linter-enforced). |
| API/worker -> modules | Via `service` and `public`; composition roots wire subscribers | Worker tasks are thin wrappers around service calls. |

---

## Open Decisions to Confirm (architecture-affecting)

1. Diary day semantics: user-chosen `log_date` for meals (recommended) and whether a "day starts at 04:00" setting is wanted (recommended: no, defer).
2. Daily insight timing: previous complete day at local morning (recommended) vs also an evening "today" card.
3. Job runner: Procrastinate (recommended) vs Celery; settle in the Phase 0 spike.
4. Boundary linter: import-linter (recommended) vs Tach; settle in the Phase 0 spike.
5. Outbox at day one (recommended) vs in-process bus first with outbox in the next phase (acceptable if call sites use `uow.record()`).
6. Rule version retention: keep old rule versions importable for exact replay, or rely on stored `inputs` + `engine_version` git SHA only.
7. Mobile framework is undecided; the API design above is framework-neutral, but the typed-client generation tool depends on it.

## Sources

- Import Linter docs, contract types and INI config example: https://import-linter.readthedocs.io/en/stable/ (fetched; sub-pages for TOML/layers returned 404, so exact `layers` sibling syntax and pyproject form are UNVERIFIED here) [MEDIUM]
- Tach (public interfaces, `tach.toml`, layered architecture): https://docs.gauge.sh/usage/interfaces/ , https://github.com/tach-org/tach [MEDIUM, search snippets]
- FastAPI modular monolith template with import-linter in CI and public/private modules: https://github.com/deadislove/fastapi-modular-monolith-template [MEDIUM]
- Modular monolith in Python (public API files, events): https://breadcrumbscollector.tech/modular-monolith-in-python/ [MEDIUM]
- Transactional outbox with Postgres and `FOR UPDATE SKIP LOCKED`: https://dev.to/gabrielanhaia/outbox-pattern-in-postgres-end-to-end-producer-relayer-consumer-2k75 , https://www.npiontko.pro/2025/05/19/outbox-pattern , https://github.com/modern-python/faststream-outbox [MEDIUM, multiple sources agree]
- Procrastinate (Postgres task queue, periodic tasks, transactional deferral, SQLAlchemy connector caveats): https://procrastinate.readthedocs.io/ , https://procrastinate.readthedocs.io/en/stable/howto/basics/connector.html [MEDIUM]
- ARQ maintenance-only status: https://github.com/python-arq/arq (issue #510 per search), https://github.com/tastyware/streaq/discussions/108 [MEDIUM]
- Task queue comparison (Procrastinate vs arq vs Celery): https://stevenyue.com/blogs/exploring-python-task-queue-libraries-with-load-test [LOW-MEDIUM]
- Postgres timezone grouping and storing local date at write time: https://tunasakara.com/en/blog/multi-timezone-grouping-postgresql , https://oneuptime.com/blog/post/2026-01-25-postgresql-timezone-handling/view [MEDIUM]
- Open Food Facts nutriments per 100 g data model: https://static.openfoodfacts.org/data/data-fields.txt , https://blog.openfoodfacts.org/en/news/data-in-open-food-facts [MEDIUM]
- Mobile API versioning (N-1, version-check endpoint, force update sparingly): https://yrkan.com/blog/api-versioning-mobile/ , https://oneuptime.com/blog/post/2026-01-24-fix-api-versioning-compatibility-issues/view [MEDIUM]
- Practice-derived (no single source; HIGH within domain consensus): snapshot-vs-reference for transactional line items, per-entity translation tables, CLDR plural categories for Romanian, immutable audit records with superseding runs, zoneinfo/tzdata container requirement.

---
*Architecture research for: FitAcademy (fitness and nutrition tracking, FastAPI modular monolith, rule-based coach)*
*Researched: 2026-10-01*
