# Pitfalls Research

**Domain:** Personal fitness and nutrition tracking mobile app (FitAcademy): FastAPI modular monolith, rule-based explainable coach, EN/RO, EU/Romania users, solo developer, portfolio -> friends -> possible public launch
**Researched:** 2026-10-01
**Confidence:** MEDIUM overall. Licensing, store-policy, rate-limit and token-rotation facts were cross-checked against official or primary pages (tagged VERIFIED). Domain-failure patterns (food data quality, timezone, retention, coach safety, over-engineering) come from established engineering and domain knowledge (tagged EXPERIENCE) and should be validated in phase-level research where flagged.

Confidence tags used below:
- **VERIFIED**: confirmed on an official or primary source during this research session.
- **EXPERIENCE**: well-established domain knowledge, not re-verified this session.
- **VERIFY-LATER**: details that change often (store policy numbers, legal thresholds). Re-check at the phase that depends on them.

Indicative phase names used for mapping (the roadmapper may rename or merge them):

| # | Phase | Scope |
|---|-------|-------|
| P1 | Foundation | Repo, CI, module-boundary enforcement, DB and migrations, deploy skeleton, i18n skeleton, logging/observability |
| P2 | Auth and Account Lifecycle | Register/login, JWT + refresh, RBAC, consent, export, delete |
| P3 | Profile, Goals and Targets | Profile, units, goal, calorie/macro targets with safety guardrails |
| P4 | Food DB and Meal Logging | Food data model, search, logging, daily/historical totals |
| P5 | Barcode and Food Ingestion | Barcode scan, OFF fallback, bulk import, missing-product flow |
| P6 | Workouts | Exercise library, sessions/sets, history and progress |
| P7 | AI Coach | Rules engine, daily card, weekly review, stored explainable insights |
| P8 | Web Admin | Foods, exercises, users |
| P9 | Store Readiness and Dogfood | TestFlight/Play testing, store compliance, 2-week real use |

The mobile client is built vertically alongside P2 to P7. It is not a separate late phase.

---

## Critical Pitfalls

### Pitfall 1: Trusting crowd-sourced food data (per-100g vs per-serving, kJ vs kcal, missing vs zero)

**What goes wrong:**
Logged calories are off by 2x to 10x, and the user (and the coach) never notices. Typical mechanisms:
- Open Food Facts exposes `*_100g`, `*_serving` and `*_prepared` variants. Mixing them up, or applying a serving quantity to a per-100g value, corrupts the entry.
- `energy_100g` is in kJ and `energy-kcal_100g` is in kcal. Reading the wrong key gives a 4.18x error.
- Sodium vs salt (salt = sodium x 2.5).
- Liquids are per 100 ml, not per 100 g.
- Missing nutrient fields are treated as 0 (a product with no fat listed looks like zero fat).
- Impossible rows exist: more than 900 kcal/100 g, macros summing to more than 100 g, kcal that disagrees wildly with 4P + 4C + 9F.
- USDA entries differ between raw, cooked and "as consumed". Branded entries are label-derived and can be stale.
- Sources disagree on the same food and nobody chose a winner.

**Why it happens:**
Developers map one external schema to one internal `calories` column and ship. OFF itself says the data is provided voluntarily with no assurance of accuracy, completeness or reliability (VERIFIED, OFF API docs).

**How to avoid:**
- Define one canonical internal model: nutrients stored per 100 g (or per 100 ml, with an explicit `basis` enum) plus explicit `serving` definitions (name, gram/ml amount). Every logged entry stores `quantity_g` and computed macros.
- Write a source adapter per provider (OFF, USDA, manual/admin) with a single normalization function and golden-file tests using real ugly payloads.
- Prefer `energy-kcal_100g`. Derive kcal from kJ only as a fallback.
- Model nutrients as nullable, never defaulted to 0. Keep `is_estimated` and `completeness` flags.
- Add an ingest-time sanity validator (Atwater consistency within a tolerance, per-100 g bounds, macro sum <= ~100 g). Failing rows go to a `needs_review` state surfaced in the admin, not into search results.
- Rank search by trust tier: admin-verified > USDA Foundation/SR > OFF with complete data > OFF incomplete > user-created.
- Do not "fix" label kcal that differs slightly from the Atwater sum (fiber, polyols and rounding explain it). Only flag large deviations.

**Warning signs:**
- A 30 g snack shows 1,200 kcal.
- Totals jump after switching serving.
- Beverage entries showing 0 kcal.
- You catch yourself editing foods by hand to correct entries.
- Search top results are junk rows with no macros.

**Phase to address:** P4 (model and validator), P5 (OFF adapter), P8 (review queue UI).

---

### Pitfall 2: Mutable food rows rewriting history (no snapshot at log time)

**What goes wrong:**
An admin edits a food (or a re-import updates OFF data) and every historical daily total changes silently. Weekly reviews and stored coach insights then contradict the numbers the user sees today.

**Why it happens:**
The meal log stores `food_id + quantity` and joins live to the food table.

**How to avoid:**
- Snapshot computed `kcal/protein/carbs/fat` (plus fiber, sugar, salt if tracked) and the food name/serving label onto the log entry at write time.
- Keep `food_id` as a reference only.
- Make food rows versioned or append-only for nutrient changes.
- Never hard-delete foods referenced by logs. Archive them instead.
- Store log numbers as `NUMERIC` (or integer milli-units), never float. Round only at display and reconcile totals as sums of stored entries.

**Warning signs:**
Yesterday's total changes after a re-import. Sum of displayed rows != displayed total (rounding).

**Phase to address:** P4 (schema decision, expensive to retrofit).

---

### Pitfall 3: Duplicate and polluted food catalogue (including Romanian coverage and diacritics)

**What goes wrong:**
Search returns 14 variants of "Lapte 1.5%". User-created foods leak into the global catalogue. OFF/USDA coverage of Romanian products and traditional foods (mici, sarmale, cozonac, telemea, zacusca) is thin, so the user's most common foods are missing and logging feels broken. Searching "paine" does not find "pâine", and searching "ciorbă" misses "ciorba". Romanian legacy cedilla forms (ş, ţ) vs correct comma-below forms (ș, ț) fail to match each other.

**Why it happens:**
No dedupe key, no scope separation, and search implemented as `ILIKE '%q%'`.

**How to avoid:**
- Dedupe on barcode (normalized GTIN) first, then normalized (name + brand + basis).
- User-created foods are private (`owner_id`) by default. Promotion to global is an explicit admin action.
- Normalize all text to NFC and map cedilla variants to comma-below. Search with `unaccent` + `pg_trgm` (GIN index) and a simple ranking: exact/prefix > trigram > popularity.
- Put the user's own favorites and recents first, before any global result.
- Seed a small curated, verified Romanian staples set (a few hundred foods) manually. This is higher value than importing millions of rows.
- Store food names in a translation table or JSONB (`name.en`, `name.ro`) from the start.

**Warning signs:**
Same food appears multiple times with trivial name differences. The author keeps creating custom foods for everyday meals.

**Phase to address:** P4 (search and model), P5 (import dedupe), P8 (admin merge tool).

---

### Pitfall 4: Barcode edge cases (format normalization, missing products, regional variants, rate limits)

**What goes wrong:**
- UPC-A (12 digits) and EAN-13 (leading 0) of the same product do not match.
- No checksum validation, so scan misreads (a common camera outcome) become "product not found".
- In-store variable-weight barcodes (prefixes 02/2x) are treated as products.
- A scan miss dead-ends the user, who abandons logging for that meal.
- The same brand sells different recipes by country under different or reused GTINs, so a German formulation gets matched for a Romanian product.
- All backend OFF calls share one server IP, and OFF limits are 15 req/min/IP for product reads and 10 req/min/IP for search, with a required descriptive User-Agent (VERIFIED, OFF API docs). A handful of concurrent users exhausts the budget. Live calls from every scan also leak what users scan to a third party.
- OFF asks anyone needing more than a few hundred products to use the data dump instead of the API (VERIFIED).

**Why it happens:**
"Local first, OFF fallback" is designed as a happy path with no miss path and no capacity plan.

**How to avoid:**
- Normalize all barcodes to one canonical form (GTIN-13, zero-padded) and validate the checksum client-side before any lookup. Reject or flag variable-weight prefixes.
- Lookup order: local DB -> negative cache (miss TTL in Redis, e.g. 24 h, so repeated scans do not hammer OFF) -> throttled OFF call (queue with a token bucket under the published limits, custom `FitAcademy/<ver> (<contact email>)` User-Agent) -> graceful miss flow.
- Bulk-import an OFF dump filtered to Romania and nearby EU countries (country tags) with quality filters. Use the live API only as a long-tail fallback.
- Miss flow is mandatory UX: "Not found: add it" with the barcode pre-filled, optional label photo, and manual macro entry. The result is saved to the local DB (private first, reviewable by admin).
- Prefer the localized name field (`product_name_ro`) and fall back to `product_name`. Show the product's country and last-modified date so regional mismatch is visible.
- Never call OFF directly from the mobile app. Keep the User-Agent, throttling, caching and attribution server-side.

**Warning signs:**
OFF 429s or IP ban in logs. Scan-to-result latency above 2 s. More than ~20% of scans end in "not found" in your own testing in a Romanian supermarket.

**Phase to address:** P5. Run a "scan 30 products in a Romanian supermarket" acceptance test before closing P5.

---

### Pitfall 5: Open Food Facts ODbL compliance (attribution and share-alike)

**What goes wrong:**
OFF data is copied into your database and shown or exposed with no attribution. Or OFF-derived rows are blended with other sources in one table with no provenance, so you cannot tell later what must be attributed or shared. A public launch then exposes you to the share-alike obligation.

**Facts (VERIFIED, OFF data page and API docs; license text is legally authoritative):**
- OFF database: ODbL. Individual contents: Database Contents License. Product images: CC-BY-SA.
- Public use of the database or works produced from it requires attribution (e.g. "Contains data from Open Food Facts, available under the Open Database License").
- Publicly using an adapted database requires offering that adapted database under the ODbL.
- Individual query results shown to users are generally "Produced Works" (attribution, no share-alike). A normalized copy with substantial OFF content is a "Derivative Database" (share-alike if publicly used) (MEDIUM, practitioner analysis, not legal advice).
- USDA FoodData Central is public domain (CC0), with no such obligations (EXPERIENCE, VERIFY-LATER: confirm current FDC terms in P4/P5 research).

**How to avoid:**
- Keep per-row provenance: `source` (`off`, `usda`, `admin`, `user`), `source_id`, `source_license`, `source_url`, `imported_at`. Build a small license registry, not one global license string.
- Show an attribution line in the app (About/Credits and next to OFF-sourced food detail) and in exports.
- Do not rehost OFF images as your own assets. Link to them with CC-BY-SA attribution or skip images in v1.
- Decide in an ADR how the OFF-derived subset is handled for public launch (e.g. keep it as a separable `food_off` dataset that you could publish as an ODbL export). Do this before importing, not after.
- Contribute back (write API) is optional. If done, use the required app identification params.

**Warning signs:**
No credits screen. No `source` column. Food rows merged and edited in place with no record of origin.

**Phase to address:** P5 (import design), P9 (credits and legal pass before any public release).

---

### Pitfall 6: Timezone and day-boundary bugs in daily totals

**What goes wrong:**
- Daily totals computed with `date(created_at)` in server/UTC time. A user in Romania (UTC+2/+3) who eats at 00:30 sees dinner counted on the wrong day.
- Travelling users or DST changes shift entries between days. Romania leaves DST on the last Sunday of October (2026-10-25), giving a 25-hour day. Spring gives a 23-hour day.
- The daily coach job runs at a fixed UTC time for everyone, so it analyses a partial or wrong local day.
- Weekly review week boundaries (Mon vs Sun start) differ from user expectation. Late-night eating the "day before" feels wrong.
- Backdated logs ("I forgot lunch yesterday") land on today.

**Why it happens:**
Everything is stored as UTC `timestamptz`, which is right, but the "which day does this belong to" question is answered at query time with the wrong zone.

**How to avoid:**
- Store `eaten_at` (UTC `timestamptz`) and a materialized `local_date` (DATE) computed at write time from the user's IANA zone at that moment (`Europe/Bucharest`), plus the `tz` used. Daily totals group by `local_date`, which is stable and indexable: `(user_id, local_date)`.
- The user selects the date explicitly for backfill. The client sends `local_date` and the server validates it (reasonable window, not far future).
- The profile holds IANA tz. The client updates it when the device zone changes (with a confirmation if it differs from the stored one).
- Optional "day starts at 04:00" setting for late eaters.
- Coach jobs: run an hourly sweeper that selects users whose local time just passed the target hour (e.g. 06:00) and who have no insight yet for `(user_id, local_date, kind)`. A unique constraint makes it idempotent and gives crash catch-up for free.
- Fix the week definition (ISO Monday start) and document it in an ADR. Make it a profile setting later.
- Use `zoneinfo`, never naive datetimes, and avoid `datetime.utcnow()`.
- Tests: freeze time at 2026-10-24T22:30Z through 2026-10-25T02:00Z in `Europe/Bucharest`, and spring-forward 2027-03-28. Test midnight-boundary logs, traveller zone change, and late-night log with a 04:00 day start.

**Warning signs:**
Totals differ between app and a SQL query. The coach card says "you ate 0 kcal today" at 06:00. Bug reports clustering after midnight.

**Phase to address:** P4 (schema and totals), P7 (scheduler), P1 (time/clock abstraction and test fixtures).

---

### Pitfall 7: Unsafe targets and medical-sounding coach advice

**What goes wrong:**
The target calculator, given a goal like "lose 15 kg in 4 weeks" for a small or already-lean user, outputs 900 to 1,000 kcal/day. The coach then praises "great deficit!" on days with severe under-eating. Other failure modes:
- Users under 18 get weight-loss targets.
- Pregnancy or breastfeeding is ignored.
- Protein recommended without an upper bound, with no mention of kidney disease.
- Text drifts into diagnosis or treatment ("your diet is causing...", "this will fix your blood pressure").
- Weight typos (800 vs 80.0, lb vs kg) feed the BMR formula and produce nonsense targets.
- An underweight (BMI < 18.5) user is set a fat-loss goal.

**Why it happens:**
A calculator is built first and guardrails are an afterthought. Rule engines optimize toward "user's goal" without a notion of "the goal is unsafe".

**How to avoid (implement as hard, tested constraints in the target service, not in templates):**
- **Calorie floor**: never output a target below a conservative floor (widely used convention: ~1,200 kcal women / ~1,500 kcal men, and never below estimated BMR). Treat these as configuration values with an ADR citing the source. Clamp and tell the user why (EXPERIENCE, VERIFY-LATER: have the author choose and document sources such as NHS/WHO/dietetic guidance).
- **Rate cap**: deficit <= ~20-25% of TDEE and weight loss pacing <= ~0.5-1% of bodyweight per week. Surplus capped for muscle gain.
- **BMI guard**: no fat-loss goal when BMI < 18.5 (or when the target weight would be below it). Offer maintenance or gain and a "talk to a professional" note.
- **Age gate**: v1 is 18+ only, enforced with date of birth (not a checkbox). Minors need different guidance and parental-consent regimes, so exclude them rather than half-support them.
- **Pregnancy/breastfeeding/medical conditions**: ask at onboarding. If yes, suppress deficit targets and show a "consult your doctor/dietitian" message. The app never targets a clinical condition.
- **Input validation**: plausible ranges for height, weight, age, plus unit-aware entry and an outlier check ("you entered 8.0 kg, did you mean 80.0?").
- **Language lint**: a banned-phrase test over all coach templates (diagnose, treat, cure, prevent, detox, "burn off", "earn your food", "good/bad food"). Templates use "consider", "typical guidance", and "not medical advice" framing. The coach never names conditions.
- Protein upper bound (e.g. ~2.2 g/kg) as a config constant. Calorie targets and macro targets must reconcile (kcal ~ 4P + 4C + 9F).
- Regulatory framing: keep marketing, store text and templates inside general wellness. Claims of treating or preventing disease risk medical-device classification under EU MDR (MEDIUM, VERIFY-LATER before public launch).

**Warning signs:**
Any code path where `target_kcal` is computed without passing through a clamp function. Templates containing medical terms. No unit tests around extreme inputs.

**Phase to address:** P3 (calculator and guardrails, with property-based tests), P7 (language lint and rule guards). Make guardrail tests a gate on P3 completion.

---

### Pitfall 8: Eating-disorder risk amplification

**What goes wrong:**
Calorie counting can be a risk factor or maintenance factor for disordered eating in vulnerable people. In one survey of people with eating disorders who used a calorie-tracking app, 73% said it contributed to their eating disorder (reported figure, secondary source: LOW-MEDIUM). A controlled trial in low-risk undergraduate women found one month of tracking did not increase risk, so the risk is concentrated in vulnerable users, not universal (published RCT, MEDIUM). App design that rewards restriction makes it worse: red "over budget" numbers, "you have X kcal left" countdowns, celebrating very low intake, streak pressure, weight-loss celebration, moralizing food language.

**Why it happens:**
Engagement design for dieters is the default pattern, and a rule-based coach has no sense of "low intake is a problem, not a win".

**How to avoid:**
- Treat sustained very-low intake as a safety signal, not an achievement. Rule: if logged intake (on days marked complete) is under the floor for N days, or weight drops faster than the rate cap, the coach stops issuing deficit or "good job" messaging, shows a neutral, supportive message with a suggestion to speak to a professional, and links to an EU/Romania-appropriate resource (verify resource in P7).
- Calm UI: no red for exceeding calories, no punitive counters. Use neutral colors and "remaining" language with an option to hide numbers or weight.
- No streak guilt (gamification is already deferred, and keep it deferred; see Pitfall 19).
- Avoid "clean/junk", "cheat", "earn", "burn off" wording (covered by the language lint in Pitfall 7).
- Allow maintenance/"just track" and non-weight goals. Make weigh-ins optional and never forced.
- Provide a visible "pause calorie tracking / delete my data" path (links with GDPR deletion).
- Do not show absolute weekly-loss celebration messaging.

**Warning signs:**
Any template that congratulates a deficit without checking the floor. Test personas with extreme inputs getting cheerful output.

**Phase to address:** P3 (guardrails), P7 (safety-signal rules, supportive-message templates). Needs focused phase research in P7 (resource list, wording review with a native-Romanian reader).

---

### Pitfall 9: Coach acts on incomplete data ("garbage in, confident advice out")

**What goes wrong:**
A user logs only a coffee by 15:00, or forgets to log for 3 days. The coach declares "you are 1,800 kcal under, eat more protein!" or praises a huge deficit. The advice is wrong and erodes trust in the product's whole premise.

**Why it happens:**
Rules treat "logged total" as "eaten total".

**How to avoid:**
- Introduce a day-completeness concept: an explicit "mark day complete" (or a heuristic: at least N meals or at least X% of target logged) before a day counts for deviation insights.
- Daily cards evaluate the previous complete local day. Same-day mid-day nudges use a different, softer rule family ("so far today").
- Weekly review needs a minimum number of complete days (e.g. 4 of 7). Below that, the review says "not enough data" and nudges logging, not diet.
- Every insight records a `data_confidence` and the number of days used.
- Cold start: for the first ~7 days, only onboarding-style guidance.

**Warning signs:**
Insights fire on days with 1 to 2 entries. Review text cites weekly averages computed over 2 logged days.

**Phase to address:** P4 (day-complete model), P7 (rule gating).

---

### Pitfall 10: Explainability that is decorative (explanation disconnected from the decision)

**What goes wrong:**
The "why" shown to the user is a static template string that does not reflect the actual inputs. Or the explanation is regenerated on read with current data and no longer matches what triggered the insight. Auditability ("stored for transparency") is claimed but cannot reproduce a decision. Multiple rules fire and the user gets five cards. The same insight repeats daily. Explanations say "because your protein is low" with no numbers, window, or threshold.

**Why it happens:**
Decision logic and text generation are written separately, and thresholds are hard-coded in several places.

**How to avoid:**
- A rule returns a structured result, not text: `{rule_id, rule_version, reason_codes[], inputs{values, window, local_dates}, thresholds{...}, severity, recommended_action_code, params{}}`. The explanation is rendered from the same object that drove the decision, so it cannot diverge.
- Persist the structured payload as the source of truth (immutable row, `generated_at`, `engine_version`, `template_version`). Render text at read time into the user's current language from payload + template (this also fixes i18n staleness, see Pitfall 17). Optionally also store the rendered text snapshot for audit.
- Pick one concrete action per card: a priority/arbitration layer picks the top rule. Add cooldown and dedupe (same `rule_id` not within N days unless severity rises). Enforce the "one concrete change" requirement with a test.
- Explanations always show the user's own numbers, the window, and the target they are compared with.
- Centralize thresholds in versioned config. Add golden fixtures (user-day histories -> expected insight) as regression tests. A decision replay command can re-run any stored insight from stored inputs.
- Do not claim causation ("because you ate X your energy dropped"). Phrase as observed pattern vs target.

**Warning signs:**
Template text contains no interpolated numbers. Rule thresholds appear as literals in more than one file. Cannot answer "why did the user get this card on Tuesday" from the DB alone.

**Phase to address:** P7 (design the payload before writing any rule). The P1 ADR should reserve the `insights` table shape.

---

### Pitfall 11: Logging friction kills retention (the real product risk)

**What goes wrong:**
Manual food logging is the single biggest reason people abandon trackers. If logging a usual breakfast takes more than ~20 to 30 seconds, the author's own 2-week dogfood (the v1 success criterion) fails, regardless of how good the coach is. The coach has nothing to analyse.

**Why it happens:**
Developers build the data model and the search endpoint, test with 3 foods, and never time a real day of logging. Network latency, no recents, per-item serving pickers and full-screen modals add up.

**How to avoid:**
- Treat time-to-log as a tracked metric from the first working build: tap count and seconds for "log my usual breakfast", "scan a product", "log a workout set".
- Must-have friction reducers in P4, not "later": recents and frequents at the top of search (before typing), favorites, "copy meal/day from yesterday", remember the last-used serving per food, quick-add calories/macros, user-defined meals or recipes (composite foods), default to the most common serving, one-tap re-log.
- Search latency budget < 300 ms server-side, debounce on the client, show local recents instantly (no spinner).
- Offline-first logging in the mobile client with a local queue and client-generated UUIDs (idempotent create). Gyms and basements have poor signal, and workout logging must never block on the network.
- Workouts: prefill the previous session's sets/weights/reps, increment buttons, a rest timer that survives backgrounding.
- Onboarding: ask only what the target calculator needs, defer everything else, and let the user log before finishing optional setup.
- Coach notifications: at most one per day, with user-controlled time.

**Warning signs:**
The author skips logging by day 3. More than 6 taps to log a repeated meal. Search needs the network for every keystroke.

**Phase to address:** P4 and P6 (friction reducers, offline queue contract), P9 (dogfood measurement). Define the dogfood success metrics in P1 so they are not invented after the fact.

---

### Pitfall 12: Refresh-token and JWT mistakes

**What goes wrong:**
- Long-lived access tokens (days) that cannot be revoked.
- Refresh tokens stored in plaintext in the DB, or not stored server-side at all (so logout and "revoke all" do nothing).
- No rotation, or rotation without reuse detection.
- **Rotation race**: the mobile app fires parallel requests, all get 401, all call `/refresh` with the same token. The second use looks like theft, the whole token family is revoked, and the user is randomly logged out.
- Algorithm not pinned (`alg=none`/HS-RS confusion), weak or committed signing secret, no `iss`/`aud`/`exp` validation.
- Roles embedded in the JWT go stale (a demoted admin keeps access until expiry).
- Tokens kept in AsyncStorage/plain storage on the phone.
- Password hashing with a deprecated library, no login rate limit, user enumeration on login or reset.
- Rate limiting keyed on the proxy IP (all users share one bucket) or on spoofable `X-Forwarded-For`.

**What is VERIFIED:** Rotation issues a new refresh token on each use and invalidates the previous one. Reuse detection revokes the entire token family (all tokens descending from one login). Keep a fixed absolute max lifetime from initial issuance and do not extend it on rotation. Mobile has Keychain/Keystore for secure storage. (Multiple 2026 practitioner guides, consistent with the IETF OAuth browser-based apps guidance.)

**How to avoid:**
- Access token 10 to 15 minutes. Refresh token opaque random (not a JWT), stored server-side as a hash (SHA-256), tied to `family_id`, `device_id`, `expires_at` (absolute cap e.g. 30 to 60 days), `rotated_at`, `revoked_at`.
- Rotate on every refresh. On reuse of a rotated token outside a short grace window (e.g. 10 to 30 s to absorb network retries), revoke the family. Client side: a single-flight refresh mutex so parallel 401s share one refresh call.
- Pin algorithms explicitly in verification, validate `exp`, `iss`, `aud`, `sub`, and use a strong secret from the environment (or asymmetric keys if a second verifier appears).
- Check admin/trainer role against the DB on privileged routes (or keep TTL very short). Do not trust JWT role claims alone for destructive admin actions.
- Mobile: `expo-secure-store` / Keychain / Keystore only (adapt to the chosen framework).
- Password hashing: argon2id (e.g. `argon2-cffi` or `pwdlib`). Avoid unmaintained libraries. Prefer PyJWT over unmaintained JWT libs (VERIFY-LATER: check current FastAPI security tutorial recommendations when implementing).
- Password change or reset revokes all of a user's refresh families. Generic error messages. Single-use, short-lived reset tokens. Login/reset/register rate limits in Redis keyed on a correctly resolved client IP (configure trusted proxy headers) plus account identifier.
- Next.js admin uses httpOnly + Secure + SameSite cookies, with CSRF protection for mutating requests. Do not store tokens in localStorage there.
- If Google/Facebook login is added later, Apple requires an equivalent privacy-preserving login option such as Sign in with Apple (VERIFY-LATER; Guideline 4.8). Email/password only avoids this in v1.

**Warning signs:**
Users get logged out "randomly". A refresh table with raw tokens. Tests never exercise parallel refresh. `jwt.decode` call without an `algorithms` argument.

**Phase to address:** P2. Include an explicit concurrency test (N parallel refreshes) and a stolen-token replay test.

---

### Pitfall 13: Broken object-level authorization and weak ownership checks

**What goes wrong:**
`GET /meals/{id}` returns another user's meal because the query filters only by `id`. Admin or trainer roles see all users' health data by default. A future trainer role gets implicit access.

**Why it happens:**
Authorization is checked at the route layer but repositories accept bare IDs.

**How to avoid:**
- Repositories take `user_id` as a required parameter for every user-owned table. Provide no unscoped getters on user data (only explicit admin-scope functions).
- A two-user test harness: every user-data endpoint has a test proving user B gets 404 (not 403, to avoid enumeration) on user A's resource.
- Use non-guessable IDs (UUIDv4/v7) for user-owned resources.
- The admin UI shows counts and aggregate data by default. Viewing an individual user's health data requires a deliberate action and is audit-logged.
- Keep the `trainer` role in the role enum but build no trainer features in v1. If trainer-client access arrives later, make it a per-user explicit grant.

**Warning signs:**
Any repository method named `get_by_id(id)` for user data. No cross-user tests.

**Phase to address:** P2 (pattern), enforced in every later phase by review checklist.

---

### Pitfall 14: GDPR and health-data privacy (EU/Romania users)

**What goes wrong:**
- Treating weight, food logs, goals and training data as ordinary personal data. GDPR Article 9 classes data concerning health as special-category data, so processing needs an Article 9 basis, typically explicit consent (VERIFIED, multiple sources). Explicit consent is more than a ticked ToS box: it needs a clear statement naming the health data categories and cannot be bundled into general terms.
- Assuming "it's only friends" means GDPR does not apply. Once the author operates a service that stores other people's data, the household/personal exemption does not cover the author as operator (EXPERIENCE, VERIFY-LATER: check with the supervisory authority guidance; the Romanian authority is ANSPDCP).
- No data export and no true deletion. Deletion that leaves rows in `coach_insights`, S3, logs, caches, backups or job queues. "Soft delete" or "deactivate" used as erasure (Google Play also rejects deactivation as deletion, VERIFIED).
- Health data in logs and error trackers (request bodies in Sentry, access logs with query strings), and in analytics/crash SDKs that ship data to US processors without a DPA/transfer mechanism.
- Staging/dev databases containing real friends' data.
- Hosting in a non-EU region or EU data replicated outside the EU by default.
- Sending food logs to a third-party LLM later without a processor agreement and explicit disclosure.
- No privacy policy in Romanian and English. No breach-response plan (72-hour notification duty to the authority where applicable).
- Age: Romania's digital-consent age is 16 (Law 190/2018, VERIFY-LATER). The 18+ gate in Pitfall 7 avoids this problem in v1.

**How to avoid:**
- **Consent record**: separate, unbundled, withdrawable health-data consent screen at registration, stored with consent text version, timestamp, language. Registration is blocked without it, because the app cannot function without health data. Withdrawal triggers the account deletion flow.
- **Data map** (one page, in `docs/`): every table, bucket, cache, log and third party that holds user data, with retention. This is the checklist for deletion and export.
- **Deletion**: an in-app "Delete account" flow (required by both stores, VERIFIED) that hard-deletes via a `UserDeleted` event handled by every module (nutrition, workouts, coach, auth, storage), purges S3 objects and Redis keys, and documents backup retention (e.g. backups roll off within 30 days). Test deletion with a script that asserts zero rows remain for a user across all tables.
- **Export**: JSON/CSV export of all user data (portability).
- **Minimize**: collect only what the calculator and coach need (DOB or birth year, sex for BMR, weight, height, goal). No ads SDKs and no analytics that include health content. Prefer self-hosted or EU-region analytics, or none in v1.
- **Logging hygiene**: scrub request bodies, never log health payloads, set Sentry `send_default_pii=False` and use `before_send` scrubbing.
- Choose an EU region for DB, object storage, backups and any managed service. Sign DPAs with processors.
- Synthetic seed data for dev/staging only.
- Write a short DPIA before any public launch (systematic processing of health data), even if the author concludes it is a light one. Keep the privacy policy accurate and plain-language, in RO and EN.

**Warning signs:**
No deletion test. Logs show weights or food names. A new SDK added without a processor review. Real friend data in staging.

**Phase to address:** P2 (consent, deletion/export skeleton), then each data-owning phase adds itself to the deletion/export contract. P9 (privacy policy, DPIA, store privacy labels).

---

### Pitfall 15: App-store review problems for health apps

**What goes wrong (VERIFIED unless noted):**
- **Apple**: Guideline 1.4.1 applies extra scrutiny to apps that could provide inaccurate health information. Accuracy claims need disclosed methodology. Guideline 5.1.3 treats health/fitness data as especially sensitive. Apps that support account creation must offer in-app account deletion and explain retention and revocation. A reviewer cannot log in (no demo account), the backend is down or on a sleeping free tier, or content is placeholder, and the app is rejected under the 2.1 completeness rules (EXPERIENCE).
- **Google Play**: every developer must complete the Health apps declaration form, even apps with no health features. Apps with account creation need an in-app deletion path plus a public web URL for deletion requests, and the Data safety form includes deletion questions. Deactivation does not count.
- **Both stores (EU)**: distributing as an individual in the EU requires a Digital Services Act trader/non-trader declaration, and trader details become public (MEDIUM, VERIFY-LATER).
- Store listing or screenshots with outcome claims ("lose 10 kg in a month") invite rejection and violate your own safety stance.
- Privacy labels/Data safety answers that do not match real behavior (SDKs collecting data) lead to rejection or removal.
- iOS privacy manifest (`PrivacyInfo.xcprivacy`) and required-reason API declarations for you and for any third-party SDK (EXPERIENCE, VERIFY-LATER).
- Camera permission usage strings that are vague.
- New Google Play personal accounts have had closed-testing requirements (a minimum number of testers for a minimum period) before production access (MEDIUM, VERIFY-LATER: policy changes often). Apple's updated age-rating questionnaire (VERIFY-LATER).
- If social/UGC arrives later, Apple requires report/block/moderation mechanisms (Guideline 1.2). This is another reason social stays deferred.

**How to avoid:**
- Build account deletion in P2, not at submission time.
- Use TestFlight internal testing and the Play internal track for the friends phase. Start the store-account setup (Apple Developer Program, Play Console, DSA declaration) early in P9 so waiting periods do not block you.
- Keep a seeded demo account and an always-on backend for review. Do not use free tiers that sleep.
- Write store copy with a "general wellness, not medical advice" posture and no outcome promises. Keep the in-app disclaimer on the first-run and target screens.
- Audit every dependency for data collection before filling labels. Keep the SDK count low.

**Warning signs:**
Delete-account is still a TODO at P9. Store setup not started when the app is otherwise done.

**Phase to address:** P2 (deletion), P9 (everything else). A "store compliance checklist" artifact belongs in P1 docs so it informs design early.

---

### Pitfall 16: Choosing the mobile framework by language preference

**What goes wrong:**
Picking a Python-native mobile stack (Flet, Kivy, BeeWare) because the author wants Python. These options have thin ecosystems for camera barcode scanning, secure storage, push notifications, offline storage and store-ready packaging compared with React Native/Expo or Flutter (EXPERIENCE, VERIFY-LATER: this is a stack-research question). The barcode scanning flow, the main logging accelerator, ends up the weakest part of the app, and the store submission becomes a build-system project.

**How to avoid:**
Decide on measured criteria: camera/barcode quality and speed, secure token storage, offline queue support, build/store pipeline, and how much of it a solo developer can maintain. Python stays where it is strongest (backend, coach engine, workers, data ingestion scripts). Treat "Python in the stack" as satisfied by the backend and note it in an ADR.

**Warning signs:**
Barcode scanning prototype takes more than a day to work on a real phone. Unresolved packaging issues for iOS.

**Phase to address:** Stack decision before P1 closes. Run a one-day barcode spike on a real device before committing.

---

### Pitfall 17: i18n retrofit pain (EN/RO)

**What goes wrong:**
"i18n from day 1" is claimed but only UI labels are covered. Pain appears later in:
- **Server strings**: API error messages as English text the client displays.
- **Coach text**: templates built by string concatenation. Romanian has three plural categories (1 zi, 2 zile, 20 **de** zile), so "{n} days" breaks. Romanian also has grammatical gender and articles.
- **Stored rendered text**: insights saved in the language active at generation time (async job, no request context). When the user switches language, history stays in the old language.
- **Content in the DB**: exercise names, muscle groups, equipment, food names with no translation tables.
- **Numbers and units**: decimal comma input ("1,5") rejected or misparsed by the backend. Dates and 24 h clock. Metric default.
- **Diacritics**: see Pitfall 3.
- **Machine-translated health text** shipped unreviewed.

**How to avoid:**
- The API returns error codes and structured payloads, never user-facing prose. The client owns all UI strings.
- Coach templates use ICU MessageFormat with plural/select (via a library on the client, or Babel/ICU on the server). The structured insight payload (Pitfall 10) is rendered at read time in the user's language. Store `preferred_language` on the profile so async jobs know it.
- Translation tables (`*_translation`) or JSONB `name` fields for exercises, muscle groups, equipment, foods from the first migration.
- Parse numeric input locale-aware on the client and send canonical numbers. Store metric canonical values and format at the edge.
- Pseudo-localization and long-string testing in the client. A CI check that every key exists in both locales. Human review of the Romanian coach copy (the author can do this).
- Keep the web admin English-only in v1 to bound scope.

**Warning signs:**
`f"You ate {n} meals"` in Python. Hard-coded strings in screens. Exercise names in a single `name` column.

**Phase to address:** P1 (conventions and tooling), P4/P6 (content translation tables), P7 (ICU templates).

---

### Pitfall 18: Over-engineering the modular monolith as a solo developer

**What goes wrong:**
Weeks go to scaffolding rather than the core loop:
- Eight or more module skeletons (including empty social/gamification/rewards/trainer/wallet/notifications) before any feature works.
- A full API -> Service -> Repository -> Domain -> DTO stack with four mapping layers for trivial CRUD.
- An event bus with its own infrastructure (a message broker) where a function call suffices, introducing eventual-consistency bugs (a `MealLogged` handler that updates totals, so the UI shows stale numbers).
- Kubernetes/Terraform/multi-service compose for one developer and zero users.
- Async/sync SQLAlchemy mixed, blocking calls inside `async def` endpoints.
- Monorepo tooling fights (Expo/Metro in a pnpm workspace) consuming days.
- ADR ceremony for every minor choice.

**Why it happens:**
The architecture doc is detailed and portfolio-motivated, so the temptation is to implement the diagram rather than the product.

**How to avoid:**
- Build vertical slices. Create a module when its first feature is built, not before. Keep a `docs/` placeholder (not code) for deferred modules.
- Enforce boundaries mechanically and cheaply: `import-linter` or `tach` contracts in CI (modules import only each other's public `service`/`events` interface; no cross-module table access). This gives the "modular monolith" claim real teeth at near-zero cost.
- Start with an in-process synchronous event dispatcher. Use a transactional outbox plus Redis worker only for things that are actually async (coach jobs, imports). Do not introduce Kafka/RabbitMQ. Choose one worker library (e.g. arq, Dramatiq or RQ) and stay with it.
- Keep the repository layer thin: use it where queries are non-trivial or where user scoping must be enforced (Pitfall 13). Do not duplicate every ORM model into a DTO twin where Pydantic schemas suffice.
- Pick sync or async SQLAlchemy once (ADR) and do not mix.
- Deploy to the simplest always-on target (a single small VM or PaaS with managed Postgres in an EU region, plus Redis). No Kubernetes.
- Generate the mobile API client from OpenAPI to avoid contract drift. Version the API (`/v1`) and expose a min-supported-app-version check, since mobile clients cannot be force-updated instantly.
- Time-box ADRs to one page. Write them for decisions that are expensive to reverse (timezone model, food snapshot model, token model, insight payload, i18n approach).
- Keep the web admin minimal. Consider whether a server-rendered or off-the-shelf admin would serve the first weeks, and only build the Next.js app when the data-curation pain is real.

**Warning signs:**
More than 3 weeks in with no meal logged on a phone. More lines in `__init__`/wiring/DTO mappers than in business rules. Module folders with only stubs.

**Phase to address:** P1 (conventions and CI boundary checks), reviewed at each phase transition.

---

### Pitfall 19: Scope creep into social/gamification/LLM before the core loop works

**What goes wrong:**
Streaks, badges, feeds, trainer chat, partner rewards, wallets and an LLM "coach voice" get built because they are fun or portfolio-attractive, while the actual loop (log fast -> trustworthy totals -> explainable insight) is unproven. Specific hazards:
- Streaks built on unreliable data punish users for the app's own bugs and are a known driver of unhealthy compulsion (see Pitfall 8).
- Social and UGC trigger moderation, reporting, privacy and store-review obligations (Pitfall 15).
- An LLM generating health text breaks the project's own determinism and safety rule.
- Trainer features implied by the `trainer` role creep into v1.

**How to avoid:**
- Keep the existing Out of Scope list as law. Add a written gate in `PROJECT.md`: "Core loop proven" = author (and 2 to 3 friends) log meals and workouts on 14 of 14 days, median meal log time under 30 s, zero total-mismatch bugs open, coach cards rated useful. Only then open the next milestone.
- Allow no new module directories for deferred areas before that gate.
- When the LLM layer is eventually added, it only rephrases structured payloads and never changes decisions, with the safety lint running on its output.
- Budget the portfolio value on depth (tests, ADRs, CI, observability, safety guardrails, explainability) rather than breadth of features.

**Warning signs:**
A branch named after a deferred feature. New tables for badges/follows/chat. Time spent on animations while search is slow.

**Phase to address:** Roadmap-level (explicit milestone gate), reinforced at every phase transition.

---

## Moderate Pitfalls

### Pitfall 20: Duplicate writes and offline sync conflicts
**What goes wrong:** Retried POSTs create two copies of the same meal or set. Edits made offline overwrite newer server edits.
**Prevention:** Client-generated UUIDs as idempotency keys on all create endpoints (unique constraint, return the existing row on duplicate). Last-write-wins with `updated_at` for single-user data is acceptable, but document it. Soft-delete tombstones are needed for sync of deletions, while GDPR erasure remains a hard delete of the account.
**Phase:** P4, P6.

### Pitfall 21: Workout data model that cannot grow
**What goes wrong:** Weights stored in lb/kg ambiguously, no distinction between template and performed sets, exercises hard-deleted (breaking history), duplicate exercise names ("Bench Press" x 5), progress charts that mislead (estimated 1RM from sets of 20 reps).
**Prevention:** Store weights in kg canonical with display conversion. Model `exercise` (archivable) -> `session` -> `set(reps, weight, rpe?, order)`. Never delete exercises used in history (archive them). Curate a small seed library from a properly licensed source (check licenses: several open exercise datasets are CC-BY-SA or have attribution rules, and scraped images are not licensed; VERIFY-LATER) and apply a rename/merge tool in the admin. Show progress as best set and volume, with plain definitions.
**Phase:** P6.

### Pitfall 22: Incorrect or overconfident TDEE and macro maths
**What goes wrong:** Activity multipliers overestimate TDEE for most people. Macro grams and kcal do not reconcile. The "goal" is changed and old targets are overwritten, so history comparisons break.
**Prevention:** Use a standard BMR equation (Mifflin-St Jeor) with conservative activity multipliers, label targets as estimates, and version targets with an effective date (`target_effective_from`) so a day is compared against the target that applied then. Property-based tests: macros reconcile to kcal within tolerance, floor/cap always hold.
**Phase:** P3.

### Pitfall 23: Background jobs that double-run, skip, or silently die
**What goes wrong:** Two scheduler instances create duplicate insights. A worker outage means no cards for days with no alert. Jobs depend on request context (language, timezone).
**Prevention:** Unique `(user_id, local_date, kind)` constraint, idempotent handlers, a single scheduler, catch-up on restart (the hourly sweeper in Pitfall 6), a heartbeat/health alert if no insight has been generated in 26 h, and jobs that load all context from the DB.
**Phase:** P7.

### Pitfall 24: Hosting choices that sabotage the UX
**What goes wrong:** Free tiers that cold-start after inactivity make the first request of the day take 30+ s (and trigger refresh failures). Redis/Postgres not in the same region as the app add latency to search. Untested backups. Secrets committed to the repo.
**Prevention:** Always-on small instance in an EU region, same region for DB/Redis/storage. Automated daily Postgres backups plus one restore drill before dogfood. Secrets via environment/secret manager. Pre-commit secret scanning.
**Phase:** P1 (deploy skeleton), P9 (restore drill).

### Pitfall 25: Large OFF import breaking a small database
**What goes wrong:** The full OFF dump (millions of products) is loaded into the same small Postgres instance, bloats indexes, slows search and blows through the disk quota. `ILIKE` search over millions of rows is slow.
**Prevention:** Import via a staging table, filter by country tags and a completeness/quality threshold, keep only needed columns, build `pg_trgm` GIN indexes on the search column, and cap catalogue size (tens of thousands of relevant rows is plenty for v1). Run the import as an idempotent worker job with progress logging.
**Phase:** P5.

### Pitfall 26: Image and media handling
**What goes wrong:** User-uploaded label photos or progress photos stored in a public bucket, or without EXIF stripping (location), or with no deletion on account removal.
**Prevention:** Private bucket with signed URLs, size/type limits, EXIF stripping, included in the deletion contract. Defer progress photos entirely in v1 (they are the most sensitive data you could collect).
**Phase:** P5 (label photo only if built), P2 (deletion contract).

---

## Minor Pitfalls

### Pitfall 27: Unit and numeric formatting slips
**What goes wrong:** Showing 2,345.6000000001 kcal, inconsistent rounding between list and total, mixing g/ml/oz.
**Prevention:** Round at presentation only, one shared formatting utility, integer kcal in the UI, one decimal for grams.

### Pitfall 28: Notification fatigue and timing
**What goes wrong:** Daily push at a fixed UTC time arrives at 4 a.m. local, or every day regardless of context.
**Prevention:** Local-time scheduling, user-controlled time, opt-in, one per day, silent when the user already logged.

### Pitfall 29: Admin tool exposing too much
**What goes wrong:** The admin UI lists users with their weights and meals by default.
**Prevention:** Aggregate/identity-only user list, explicit view action with audit log, admin accounts with MFA.

### Pitfall 30: Seed data and fixtures drift
**What goes wrong:** Demo/test accounts and seed foods diverge from the real schema, so demos break.
**Prevention:** Seeds as versioned scripts run in CI against migrations, with one canonical demo user used for store review.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Join log -> live food row (no snapshot) | Simpler schema | History silently changes, coach audits break | Never |
| Store only UTC timestamp, derive day at query | Less schema | Wrong-day totals, impossible to fix cleanly later | Never |
| Calories as `float` | Convenient | Rounding drift, totals not summing | Never (use NUMERIC/integers) |
| Coach text hard-coded in rule functions | Fast first card | No i18n, no explanation consistency, hard to test | Never (templates + structured payload from day 1) |
| English-only strings "for now" | Faster first screen | Retrofit across screens and API | Never, given the stated EN/RO requirement. A basic key-based setup costs under a day |
| Raw refresh token in DB | Quick to implement | Token theft on DB leak, no family revocation | Never |
| Skip rotation reuse detection | Simpler auth | Stolen token valid for the full lifetime | Only for the first internal build, before friends use it. Must be done before P9 |
| Full OFF dump into the main table, no provenance | Fast import | Licensing ambiguity, bloat, dedupe pain | Never. Use a staging table + provenance |
| Free-tier sleeping hosting | Zero cost | Cold-start UX, refresh failures | Only during early dev, not during dogfood |
| Skip offline queue in the client | Less client code | Logging fails in poor signal, retention loss | Only for the first working vertical slice. Needed before dogfood |
| Hard-delete / skip tombstones | Simple | Sync issues later | Acceptable in v1 if the client is online-first with a refetch strategy |
| In-process synchronous event dispatch | Zero infrastructure | Coupling if handlers grow heavy | Preferred for v1. Move only the slow handlers to the worker |
| Skip a DPIA/privacy policy until "later" | Saves time | Blocks public launch and store submission | Acceptable for author-only use. Needed before friends use it with real data (privacy policy) and before any public launch (DPIA) |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Open Food Facts API | No custom User-Agent, calling per scan from every client, ignoring 15/10 req/min/IP limits, hot-linking images, no attribution | Server-side only, descriptive `AppName/Version (email)` UA, throttled queue + cache + negative cache, bulk import from the dump for coverage, attribution everywhere, provenance columns |
| Open Food Facts data fields | Using `energy_100g` (kJ) as kcal, treating missing as 0, mixing `_serving` and `_100g` | Use `energy-kcal_100g`, nullable nutrients, a normalization adapter with golden tests and a sanity validator |
| USDA FoodData Central | Mixing Branded, Foundation and SR Legacy without trust ranking, raw vs cooked confusion, API key committed | Use Foundation/SR Legacy for generic foods, Branded only as a fallback, `food_category` and preparation state captured, API key in env, bulk-download JSON/CSV for import |
| Expo/React Native (or the chosen framework) camera | Barcode scanning that is slow in low light, no checksum validation, requests permission without context | Debounced scanning, checksum + format filter, permission rationale text, manual-entry fallback |
| Redis (rate limit/cache/queue) | One Redis for all with no key namespaces or TTLs, rate limits keyed on a shared proxy IP | Namespaced keys, TTLs everywhere, resolve client IP via trusted proxy config |
| Sentry/crash/analytics SDKs | Sending health payloads or PII, US-region default, undeclared in privacy labels | Scrub in `before_send`, EU region/DPA, or omit in v1, declare in labels |
| Apple/Google store consoles | Starting setup (accounts, DSA trader status, privacy labels, health declaration, testers) when the app is finished | Start account/compliance setup early in P9, use TestFlight internal/Play internal tracks for friends |
| S3-compatible storage | Public buckets, no lifecycle, missing from the deletion contract | Private buckets, signed URLs, lifecycle rules, deletion hooks |
| Future LLM provider | Sending identifiable health logs, letting output change decisions | Pseudonymized structured payload only, DPA, output lint, deterministic decision stays in rules |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| `ILIKE '%q%'` food search | Slow typing feedback, CPU spikes | `pg_trgm` GIN index + `unaccent`, prefix-first ranking, recents shortcut, Redis cache for hot queries | ~50k to 100k food rows, noticeable well before 1M |
| Daily total computed by scanning all logs | Slow home screen | Index `(user_id, local_date)`. Compute on the fly from entries (cheap). Add pre-aggregation only if measured | Not before ~100k entries per user, so do not pre-aggregate in v1 |
| N+1 queries when listing meals with foods | Slow history screen | Eager loading/joins, snapshot columns on the log entry avoid the join | History lists above ~100 rows |
| Weekly review computing over raw logs per request | Slow endpoint | Generated by the job and stored (already required) | Immediate if computed on read |
| OFF calls in the request path | Scan latency and 429s | Local-first, queue + cache, timeouts and circuit breaker | 2 to 3 concurrent users on one server IP |
| Blocking calls inside async FastAPI endpoints | Whole API stalls under small load | One consistent sync/async approach, run blocking libs in a threadpool, argon2 hashing in threadpool | A few concurrent logins |
| Hourly sweeper scanning all users | Slow as users grow | Index on `next_insight_due_at` or tz-bucketed queries | Thousands of users (irrelevant for v1, avoid premature work) |
| Full dump import on small DB | Disk/IO exhaustion | Filtered staging import, capped catalogue | At import time |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| IDOR on user resources (bare `get_by_id`) | Cross-user health data disclosure (high) | Mandatory `user_id` scoping in repositories, two-user tests |
| Admin sees all user health data by default | Privacy breach, GDPR exposure (high) | Aggregates by default, audited explicit access, admin MFA |
| Refresh tokens in plaintext / no family revocation | Persistent account takeover (high) | Hashed opaque tokens, rotation, reuse detection, absolute lifetime |
| JWT algorithm not pinned / weak secret | Token forgery (high) | Explicit `algorithms=[...]`, strong secret from env, validate `exp/iss/aud` |
| Health data in logs/error tracker | Special-category data leak (high) | Scrubbing, no request-body logging, review log lines |
| Public or guessable media URLs | Exposure of label/user photos (medium) | Private bucket, signed URLs, UUID keys |
| Login/reset/register without rate limits; enumeration | Credential stuffing, account discovery (medium) | Redis rate limits (IP + account), generic responses |
| Rate limiting on the wrong IP behind proxy | All users throttled or limit bypass (medium) | Configure trusted proxy headers |
| CORS wildcard + cookie auth on the admin | CSRF/session abuse (medium) | Strict origins, SameSite cookies, CSRF tokens |
| Deletion that leaves data in backups/caches/queues | GDPR erasure failure (high) | Data map, deletion script test, documented backup roll-off |
| Dependencies with typosquat/abandoned auth libs | Supply-chain/auth bugs (medium) | Use maintained libs (PyJWT, argon2-cffi/pwdlib), lock files, Dependabot/pip-audit |
| Debug endpoints / docs exposed in prod | Information disclosure (low-medium) | Disable `/docs` and debug in prod or protect with auth |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Logging a repeated meal takes many taps | Abandonment within days | Recents/frequents/favorites, copy yesterday, remembered servings, quick-add |
| Barcode miss dead-ends | User gives up on the meal | Prefilled "add this product" flow, label photo optional, save to local DB |
| Search requires exact Romanian diacritics | "Not found" for obvious foods | `unaccent` + trigram search, normalization |
| Red "over budget" numbers and deficit praise | Shame, restrictive behavior, ED risk | Neutral colors, "remaining" wording, safety-signal rules, hide-numbers option |
| Generic insight ("eat more protein") | Looks like horoscope, not coaching | Show the user's numbers, window, target, and one concrete change |
| Insights on incomplete days | Wrong advice, lost trust | Day-complete gating, "not enough data" states |
| Daily push at fixed UTC time | Annoyance, opt-outs | Local-time, opt-in, one per day |
| Long onboarding before first log | Drop-off before value | Minimum inputs for targets, let the user log immediately |
| Workout logging needs network | Lost sets in the gym | Offline queue, prefilled previous values, background-safe rest timer |
| Language switch leaves history in the old language | Feels half-translated | Render insights at read time from structured payload |
| Decimal comma rejected ("1,5") | Entry errors | Locale-aware numeric input |
| Mixed units (lb/kg) with no clear indicator | Wrong weights recorded | Unit label on every field, canonical metric storage |

## "Looks Done But Isn't" Checklist

- [ ] **Food logging:** Often missing snapshot-at-log-time of nutrients. Verify that editing a food does not change yesterday's total.
- [ ] **Daily totals:** Often missing a stored `local_date`. Verify with the 2026-10-25 25-hour day and a 00:30 log in `Europe/Bucharest`.
- [ ] **Barcode scan:** Often missing the miss path and negative cache. Verify scanning an unknown barcode produces a prefilled add flow, and repeated scans do not call OFF again.
- [ ] **Barcode normalization:** Often missing UPC-A/EAN-13 equivalence and checksum validation. Verify the same product scanned in both forms hits one record.
- [ ] **OFF integration:** Often missing the User-Agent, throttling, and attribution. Verify headers in a captured request and the credits screen.
- [ ] **Food data:** Often missing a sanity validator. Verify a row with 1,200 kcal/100 g and one with missing fat are quarantined, not searchable.
- [ ] **Targets:** Often missing clamps. Verify property tests: no output under the floor or over the rate cap for any valid input, and no fat-loss goal when BMI < 18.5.
- [ ] **Age gate:** Often a checkbox. Verify DOB-based rejection under 18.
- [ ] **Coach insights:** Often missing the stored inputs/threshold/version. Verify a stored insight can be replayed from the DB alone.
- [ ] **Coach safety:** Often missing the banned-phrase lint and the low-intake safety rule. Verify the 7-day under-floor persona gets the supportive message, not praise.
- [ ] **Coach on incomplete data:** Verify a day with only a coffee logged does not produce a deviation insight.
- [ ] **Auth refresh:** Often missing the race handling. Verify 10 parallel refresh calls do not log the user out and replaying an old token revokes the family.
- [ ] **Auth storage:** Verify tokens are in Keychain/Keystore and refresh tokens are hashed server-side.
- [ ] **Ownership:** Verify every user-data endpoint has a test where user B gets 404 on user A's data.
- [ ] **Account deletion:** Often missing S3, Redis, insights, and job queues. Verify a script finds zero rows/objects for the deleted user across all stores.
- [ ] **Data export:** Verify the JSON export includes every data-owning module.
- [ ] **Consent:** Verify a stored consent record (text version, timestamp, language) exists for every user and withdrawal triggers deletion.
- [ ] **Logs:** Verify no weights, meals, or request bodies in logs/Sentry.
- [ ] **i18n:** Often missing plural forms and server strings. Verify Romanian "1 zi / 2 zile / 20 de zile", the CI key-parity check, and that API errors are codes.
- [ ] **Language switch:** Verify old insights render in the new language.
- [ ] **Offline logging:** Verify airplane-mode logging syncs without duplicates after reconnect.
- [ ] **Scheduler:** Verify running two scheduler instances produces one insight per user-day.
- [ ] **Store readiness:** Verify in-app delete, Play Health declaration, DSA declaration, privacy labels vs real SDKs, demo account, always-on backend.
- [ ] **Backups:** Verify a restore drill was actually performed.
- [ ] **Dogfood:** Verify the written "core loop proven" gate is measured (days logged, median log time), not felt.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| No snapshot on log entries | HIGH | Add snapshot columns, backfill from current food rows (accept inexact history for old entries), freeze mutable food edits, add a migration test |
| Missing `local_date` | MEDIUM | Add column, backfill from `eaten_at` using profile tz (best effort), switch all totals to `local_date`, re-run affected insights |
| ODbL ambiguity discovered late | MEDIUM | Add provenance columns by re-import tracing `source_id`, add attribution, separate OFF-derived subset, decide on publishing the derived dataset |
| Bad food data in catalogue | LOW to MEDIUM | Run the sanity validator over all rows, quarantine failures, admin review queue, re-rank search by trust tier |
| Refresh tokens stored raw | MEDIUM | Rotate signing secret, force global logout, migrate to hashed opaque tokens |
| Rotation race log-outs | LOW | Add the grace window server-side and the single-flight mutex client-side |
| Unsafe targets already shown to users | MEDIUM | Ship clamps, recompute targets for all users, show a one-time explanation notice, review stored insights for unsafe messaging and supersede them |
| i18n retrofit (hard-coded strings/prose in API) | HIGH | Extract strings by screen, change API to codes, migrate stored insights to structured payloads, backfill translation tables |
| GDPR gaps (no deletion/export/consent) | MEDIUM to HIGH | Build the data map, add the deletion event contract, implement export, collect fresh consent from existing users at next login |
| Health data leaked in logs/Sentry | MEDIUM | Purge/rotate affected logs and project, add scrubbing, document the incident per the breach runbook |
| Store rejection for missing deletion/declarations | LOW | Implement the missing item, resubmit (days of delay), keep the checklist |
| Over-engineered scaffolding | MEDIUM | Delete empty modules, collapse trivial layers, keep CI boundary contracts, refocus on the vertical slice |
| Scope creep into social/gamification | MEDIUM | Revert to the gate, park branches, restate Out of Scope in `PROJECT.md` |
| Cold-start hosting killing UX | LOW | Move to an always-on small instance in an EU region |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1 Crowd-sourced data quality | P4 (model/validator), P5 (OFF adapter), P8 (review queue) | Golden-payload adapter tests, validator quarantines bad rows |
| 2 Mutable food rows rewriting history | P4 | Edit-food test: historical totals unchanged |
| 3 Duplicates, Romanian coverage, diacritics | P4, P5, P8 | Search tests for "paine/pâine", cedilla vs comma, dedupe by GTIN, curated RO staples seeded |
| 4 Barcode edge cases | P5 | Supermarket acceptance test, UPC/EAN equivalence test, negative-cache test |
| 5 ODbL compliance | P5, P9 | `source` columns present, credits screen, ADR on derived subset |
| 6 Timezone/day boundary | P1 (clock), P4 (schema), P7 (scheduler) | DST 2026-10-25 and 2027-03-28 tests, midnight tests, idempotent sweeper |
| 7 Unsafe targets/medical advice | P3, P7 | Property tests on clamps, banned-phrase lint in CI, 18+ DOB gate |
| 8 Eating-disorder amplification | P3, P7 | Low-intake persona gets supportive message, no deficit praise, wording reviewed |
| 9 Coach on incomplete data | P4, P7 | Coffee-only day produces no deviation insight |
| 10 Decorative explainability | P7 (shape reserved in P1 ADR) | Replay a stored insight from DB, numbers appear in every explanation |
| 11 Logging friction | P4, P6, P9 | Time-to-log metrics defined in P1, measured in dogfood |
| 12 JWT/refresh mistakes | P2 | Parallel refresh test, replay revokes family, algorithm pinned |
| 13 BOLA / ownership | P2, every phase | Two-user tests per endpoint |
| 14 GDPR/health privacy | P2 (consent, deletion/export), all data phases, P9 (policy/DPIA) | Deletion script zero-rows test, export test, log scrub check |
| 15 App-store review | P2 (deletion), P9 | Store checklist all green, demo account works, always-on backend |
| 16 Framework chosen by language | Before/at P1 | One-day barcode spike on real device |
| 17 i18n retrofit | P1, P4, P6, P7 | CI key parity, ICU plural tests, error codes only |
| 18 Over-engineering | P1, reviewed each phase | `import-linter`/`tach` in CI, no empty module folders, first meal logged by end of early phase |
| 19 Scope creep | Roadmap-level gate | Written "core loop proven" gate measured before new milestone |
| 20 Duplicate writes / sync | P4, P6 | Idempotent create test, airplane-mode test |
| 21 Workout model | P6 | Archive-not-delete test, kg canonical storage |
| 22 TDEE/macro maths | P3 | Reconciliation property tests, versioned targets |
| 23 Background job robustness | P7 | Two schedulers -> one insight, heartbeat alert |
| 24 Hosting | P1, P9 | Always-on EU deployment, restore drill |
| 25 Large OFF import | P5 | Filtered staging import with index and size budget |
| 26 Media handling | P2, P5 | Private bucket, deletion hook |

## Phases Most Likely to Need Deeper Phase-Level Research

| Phase | Why |
|-------|-----|
| P3 (targets/guardrails) | Choose and cite authoritative calorie-floor, rate-cap and BMI-guard sources. Decide Romanian-appropriate safety resources. Domain-safety critical. |
| P5 (barcode and ingestion) | OFF dump format and size, country filtering, current rate limits, ODbL derived-database ADR, USDA terms, Romanian coverage measurement. |
| P7 (coach) | Rule catalogue design, insight payload schema, ED safety-signal rules and wording, ICU template tooling. |
| P9 (store/legal) | Policies change often (Play testing requirements, DSA trader status, Apple age rating and privacy manifests). Re-verify at the time. Also the DPIA and the digital-consent age. |
| Stack decision (mobile) | Barcode/camera spike on a real device before committing. |
| P1, P2, P4, P6, P8 | Standard patterns. Little extra research needed beyond the pitfalls above. |

## Sources

Verified this session (confidence MEDIUM for web results, upgraded where cross-checked with a primary page):
- Open Food Facts API introduction (rate limits 15/min product read and 10/min search per IP, custom User-Agent `AppName/Version (email)`, ODbL/DbCL/CC-BY-SA licensing, data provided voluntarily with no accuracy assurance, bulk download for more than a few hundred products): https://openfoodfacts.github.io/openfoodfacts-server/api/
- Open Food Facts data page (license terms, attribution wording): https://world.openfoodfacts.org/data
- Practitioner analysis of ODbL for commercial nutrition apps (Produced Work vs Derivative Database, provenance registry, caching): https://dev.to/dietly/what-odbl-means-for-commercial-nutrition-apps-4hl7 (LOW-MEDIUM, secondary; not legal advice)
- Apple App Review Guidelines (1.4.1, 5.1.3, in-app account deletion): https://developer.apple.com/app-store/review/guidelines/
- Google Play: account deletion requirements https://support.google.com/googleplay/android-developer/answer/13327111?hl=en ; Health apps declaration https://support.google.com/googleplay/android-developer/answer/14738291?hl=en ; Health content and services https://support.google.com/googleplay/android-developer/answer/16679511?hl=en
- GDPR Article 9 and health-data consent guidance (secondary summaries; the regulation text itself is authoritative): https://www.themomentum.ai/blog/gdpr-consent-requirements-health-data ; https://www.legiscope.com/blog/health-data-article-9-gdpr.html ; https://fitsociety.io/blog/gdpr-for-fitness-apps
- Refresh token rotation and reuse detection (token families, absolute lifetime, Keychain/Keystore): https://dev.to/mukesh_13/refresh-token-rotation-under-the-hood-how-auth0-catches-a-stolen-token-before-its-ever-replayed-3n1l ; https://codecondo.com/jwt-refresh-token-rotation/ ; https://guptadeepak.com/ciam-compass/guides/token-lifetime-best-practices/
- Calorie-tracking and eating-disorder evidence: RCT in undergraduate women (no increased risk in low-risk group) https://www.sciencedirect.com/science/article/abs/pii/S2212267221007346 ; Stronger by Science overview https://www.strongerbyscience.com/diet-tracking/ ; the "73% believed the app contributed" figure comes from secondary reporting (https://firststepsed.co.uk/blog/the-dangers-of-myfitnesspal-when-calorie-counting-takes-control/) and should not be quoted as authoritative.

Not re-verified this session (EXPERIENCE, flagged VERIFY-LATER where policy-dependent): food-data field semantics (`energy_100g` in kJ, `energy-kcal_100g`), UPC-A/EAN-13 equivalence, timezone/DST design, Romanian plural categories and diacritic normalization, calorie floor conventions, Romania digital-consent age (16), DSA trader declaration, Play closed-testing requirements, exercise dataset licenses, FastAPI/JWT library recommendations, framework maturity of Python-native mobile options.

---
*Pitfalls research for: personal fitness and nutrition tracking app (FitAcademy), FastAPI modular monolith, rule-based explainable coach, EU/Romania*
*Researched: 2026-10-01*
