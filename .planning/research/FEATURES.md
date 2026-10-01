# Feature Research

**Domain:** Personal nutrition + workout tracking mobile app with a rule-based, explainable coach (FitAcademy)
**Researched:** 2026-10-01
**Confidence:** MEDIUM overall

- Target-calculation science (Mifflin-St Jeor, activity multipliers, protein g/kg, Atwater factors): HIGH. Established, widely corroborated, and used the same way across competitors.
- Competitor feature and pricing specifics: MEDIUM-LOW. They come from WebSearch results (review and comparison blogs). The confidence seam classifies `websearch` as LOW unless verified. Claims below are cross-corroborated across several independent pages, but none were checked against first-party docs. Re-verify any specific price or paywall claim before it drives a decision.
- Store-policy and GDPR notes: MEDIUM (training knowledge, not re-fetched this session). Verify at the phase that implements auth and privacy.

## Feature Landscape

### Table Stakes (Users Expect These)

Missing any of these makes the app feel broken next to MyFitnessPal, Cronometer, Lose It!, MacroFactor, Strong or Hevy.

**Auth, account, profile**

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Email/password register, login, persistent session (JWT access + refresh) | Users never want to re-login on a phone | MEDIUM | Already in PROJECT.md. Refresh rotation and revocation on logout. Roles user/trainer/admin are in the token from day 1. |
| Password reset by email | Every consumer app has it, and friends will forget passwords | LOW-MEDIUM | Needs a transactional email provider. Pulls a minimal "notifications/email" seam into v1. |
| Delete account (in-app) and export my data | App Store requires in-app account deletion for apps with account creation. GDPR applies because RO is in the EU and health data is a special category. | MEDIUM | Hard-delete or anonymize cascade across all modules. Design the schema around `owner_id` cascades from the start. |
| Explicit consent for health-data processing, plus privacy policy | Legal baseline for a public launch in the EU | LOW | A consent record with timestamp and policy version. Cheap now, painful to retrofit. |
| Profile: sex, birth date, height, current weight, activity level, units (metric/imperial), language, timezone | Inputs for target math and for localization | LOW | Store canonical metric units. Store the user's timezone and compute the "logical day" from it. Streaks, daily summaries and daily jobs all depend on this. |
| Goal selection: fat loss / maintenance / muscle gain | Every competitor does this at onboarding | LOW | Optional goal pace (slow/standard/fast). |
| Calorie and macro targets auto-computed from profile and goal, shown with a breakdown, and editable | All apps compute them. MFP paywalls gram-based macro editing, so free editable targets are a visible advantage. | MEDIUM | See "How targets are typically computed" below. |
| Bodyweight logging with trend line | Needed for progress, for the weekly review, and for any later target adjustment. **Gap: not in the current Active list.** | LOW | Weight, date, optional note. Show a smoothed trend (EMA), not just raw points. Daily weight is noisy and users panic at raw swings. |

**Nutrition logging**

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Food search (name, brand) returning kcal/protein/carbs/fat | Core of the category | HIGH | The hard parts are search quality (prefix + trigram + ranking by popularity and verified status) and data coverage for Romanian foods. See Pitfalls and the data-source decision. |
| Log food to a day and meal slot (breakfast/lunch/dinner/snack) with serving size/unit/quantity | Universal | MEDIUM | Foods need a per-100g base plus named servings (1 slice, 1 cup, 1 serving). Log entries should **snapshot nutrients at log time**, so later food edits or admin fixes don't rewrite history. |
| Edit and delete log entries, change quantity | Users mis-log constantly | LOW | |
| Daily summary: kcal and macros consumed vs target, remaining, progress rings or bars | The main screen of every tracker | LOW-MEDIUM | |
| History: browse previous days, weekly and monthly totals/averages | Needed for the weekly review, and users expect it | MEDIUM | |
| Manual (custom) foods | Always needed because databases are incomplete, especially for Romanian products | LOW-MEDIUM | Private to the creator by default. Required fields: name, serving, kcal, P/C/F. Optional: fiber, sugar, sodium, saturated fat. |
| Favorites and recent foods | Cuts logging time more than any other feature | LOW | "Recent" is nearly free (query the log). Favorites are a join table. |
| Copy/repeat meal or day ("log yesterday's breakfast") | Lose It!'s "Add Yesterday's meal" and MacroFactor's copy/paste are cited as the main speed features. People eat the same things. **Gap: not in the current Active list.** | LOW-MEDIUM | Likely the highest-ROI logging feature after search. |
| Quick-add calories/macros without a food | Lose It!, MFP and MacroFactor all have it. It is the escape hatch for restaurant meals. | LOW | Store as an entry with no food reference. |
| Barcode scanning with lookup, plus a "not found, create it" path | Table stakes in 2026. MFP paywalled it in 2022 and users resent that, so a free scanner is a selling point for a free app. | MEDIUM-HIGH | Local DB first, then Open Food Facts fallback. A **miss flow** (create a custom food with that barcode attached) is essential, because OFF coverage is patchy outside Western Europe. |

**Workouts**

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Exercise library filterable by muscle group and equipment, with search | Strong, Hevy and Fitbod all have it | MEDIUM | Seed from an open dataset (check licenses, e.g. free-exercise-db or wger; unverified here). Exercise names and instructions need EN+RO. |
| Custom exercises | Every logger has them. Libraries are never complete. | LOW | Private per user. |
| Log a workout session: exercises, sets with weight x reps, add/remove sets and exercises, finish and save | Core of Strong and Hevy | MEDIUM | Model: Workout -> WorkoutExercise -> Set (weight, reps, set type, optional RPE, done flag). Support kg/lb. |
| Show previous performance for each exercise while logging ("last time: 60 kg x 8") | The signature UX of Strong and Hevy, and what makes progressive overload possible. **Gap: not in the current Active list.** | LOW-MEDIUM | One query per exercise: the last session's sets. |
| Set types: warm-up vs working (drop/failure optional) | Warm-ups pollute volume and PR stats if not distinguished | LOW | At minimum warm-up vs normal. |
| Rest timer | Present in both Strong and Hevy | LOW-MEDIUM | Client-side. Needs background and notification handling on mobile (a platform-specific trap). |
| Workout history list and detail | Universal | LOW | |
| Per-exercise progress: estimated 1RM, best set, volume over time, personal records | Both apps track PRs and 1RM, and show charts | MEDIUM | Epley/Brzycki estimated 1RM. Compute from set data (derived, never stored as truth). |
| Repeat last workout / save workout as routine (template) | Hevy free allows 4 routines and Strong 3, so they are a core mechanic. Without them, every session starts from an empty page. | MEDIUM | The current Active list says only "log sessions". A minimal "repeat previous workout" is cheap. Full routine management can be v1.x. |
| Offline-tolerant logging (workout in progress survives lost signal or app kill) | Gyms have poor reception. Losing a half-logged session is the fastest way to lose a user. | MEDIUM-HIGH | At minimum persist the in-progress workout locally and sync on finish. Full offline-first sync is much harder and should be deferred. |

**Coach (the stated core value)**

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Daily insight card: what happened, why it matters, one concrete change | The core promise in PROJECT.md | MEDIUM | See the Differentiators table for structure. This is table stakes *for FitAcademy* but differentiating in the market. |
| Weekly review: consistency, macro balance, training volume, 2-3 suggestions | MacroFactor's weekly check-in is the benchmark | MEDIUM-HIGH | Needs aggregation queries, rule prioritization and cooldowns, and a stored result. |

**Platform and admin**

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| EN + RO UI strings, locale-aware number/date formatting, unit preferences | Stated requirement. Romanian uses decimal commas, and users may type "1,5". | MEDIUM | Use ICU-style messages. Romanian has three plural categories (one/few/other), so naive "s" pluralization breaks. Coach text templates must be translated and pluralized too. |
| Content i18n: food names and exercise names/instructions in EN and RO | A Romanian user searching "pui" or "piept de pui" must find chicken breast | MEDIUM | This is data, not UI strings. Plan for translation tables or fields, and for search across both languages. |
| Kcal and kJ handling | EU labels show both, and OFF stores energy in kJ | LOW | Always display kcal. Convert on ingestion. |
| Admin: food CRUD (with search), exercise CRUD (muscle group, equipment, translations), user list (role change, disable, delete) | Required to curate data without raw DB access | MEDIUM | Also needed: an admin review queue for user-created foods or barcodes, and an audit log of admin actions. The audit log is cheap now and expensive later. |
| Loading, empty, error and offline states, plus an onboarding flow | Basic polish that separates a product from a demo | MEDIUM | Onboarding doubles as the target-setup flow. |

### How calorie and macro targets are typically computed

This is the standard pipeline across MyFitnessPal, Lose It!, Cronometer and the initial targets of MacroFactor. Implement it as a pure, unit-tested function, because its output feeds the coach rules and must be reproducible. (HIGH confidence on the formulas; the exact deficit percentages differ by app.)

1. **BMR (resting energy):** Mifflin-St Jeor (1990). It is the de facto default.
   - Male: `10*kg + 6.25*cm - 5*age + 5`
   - Female: `10*kg + 6.25*cm - 5*age - 161`
   - Katch-McArdle (`370 + 21.6*leanMassKg`) is only better when a *reliable* body-fat % is known. A bad body-fat input gives worse results than Mifflin. Skip it in v1, or make it an optional override.
2. **TDEE:** BMR x activity multiplier. Sedentary 1.2, light 1.375, moderate 1.55, very active 1.725, extra 1.9. Users overestimate activity, so label the choices by *daily life* ("desk job") and not by gym frequency, to avoid double-counting logged workouts.
3. **Goal adjustment:**
   - Fat loss: roughly -10% to -25% of TDEE. A common safe pace is 0.5-1% of bodyweight per week. Offer slow/standard/fast.
   - Maintenance: 0.
   - Muscle gain: roughly +5% to +15% (about +250-500 kcal).
   - Apply **hard guardrails**: a minimum kcal floor (commonly ~1200 for women, ~1500 for men; pick and document values), a cap on the deficit size, and refuse or soften targets for under-18s.
4. **Protein first, then fat, then carbs fill the remainder:**
   - Protein: about 1.6-2.2 g/kg bodyweight for muscle gain or resistance trainees (ISSN-range literature). Use the upper end (about 2.0-2.4) in a deficit to protect lean mass. About 1.2-1.6 g/kg for general maintenance.
   - Fat: about 20-35% of kcal, with a floor around 0.6-0.8 g/kg.
   - Carbs: remaining kcal.
   - Atwater factors: 4 kcal/g protein, 4 kcal/g carbs, 9 kcal/g fat.
5. **Show the work.** Display "BMR -> TDEE -> goal adjustment -> macro split" in the UI with each step. Allow manual override (MacroFactor's Coached/Collaborative/Manual idea, simplified to "accept or edit"). This explanation screen is the first place the product delivers on its explainability promise.
6. **Static vs adaptive:** MFP, Lose It! and Cronometer compute once and stay static. MacroFactor re-estimates TDEE continuously from logged intake and the weight trend, then proposes weekly changes (it needs about 2-4 weeks of consistent data). That is the market's main upgrade path, and it is why bodyweight logging belongs in v1 even though adaptive targets don't.

### Differentiators (Competitive Advantage)

Align with the Core Value: "clear, explainable coaching that tells them what to change." Don't differentiate on everything.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Explained coach recommendations** (observation -> evidence numbers -> why it matters -> one action) | Competitors show numbers, or give opaque guidance. Fitbod's model is not inspectable. Hevy Trainer at least publishes its progression rule. A transparent "because" on every card is FitAcademy's strongest wedge and fits the portfolio angle. | MEDIUM | Each insight stores `rule_id`, `rule_version`, the input snapshot (the numbers used), the template id and version, locale, and the rendered text. This gives auditability, a clean path to an LLM wording layer later, and one-click regeneration. |
| One-action-per-day insight with prioritization and cooldown | Avoids notification-wall fatigue. Most apps dump data or generic tips. | MEDIUM | Priority-ranked rule set, dedupe (don't repeat the same rule within N days), suppress with insufficient data. |
| Rule library tied to goal (e.g. protein shortfall, kcal over/under target for 3+ days, weekend drift, low logging completeness, training-volume imbalance across muscle groups, missed planned sessions) | Concrete and actionable, which is what "tells them what to change" means | MEDIUM-HIGH | Start with about 8-12 well-tested rules rather than 50. Each rule is a pure function of aggregates. |
| "Why these targets" explanation at onboarding and on every target change | Competitors apply formulas silently | LOW-MEDIUM | Nearly free once the target function returns its intermediate steps. |
| Weekly review with a prioritized, bounded suggestion set (2-3) | MacroFactor's weekly check-in is its signature. Pairing nutrition *and* training in one review is rarer. Strong and Hevy don't do nutrition, and MFP/Cronometer don't coach training volume. | HIGH | Sections: adherence (days logged, days in range), average kcal/protein vs target, macro balance, training sessions and volume per muscle group, PRs, weight trend if available. |
| Free barcode scanning and free gram-level macro targets | MFP paywalls both, so "no paywall on core features" differentiates on trust. | LOW (policy) | A product stance, not engineering. |
| Accept/modify/reject target-adjustment proposals (based on weight trend and adherence) | MacroFactor-style loop, simplified and rule-based. Keeps the user in control and stays explainable. | MEDIUM-HIGH | Suggest v1.x. Needs bodyweight trend, an adherence gate (don't adjust on incomplete logging), and a minimum data window. |
| Rule-based progression hints for lifts (e.g. double progression: hit top of the rep range on all sets, then add weight) | Hevy Trainer uses this style, and it is explainable, unlike Fitbod's opaque model. A natural fit for the rules engine. | MEDIUM | v1.x. Depends on workout history and "previous performance". |
| Bilingual coach voice (EN + RO) | Few competitors serve Romanian well. Genuinely useful to the target friends and a portfolio differentiator. | MEDIUM | Native-quality RO templates, not machine-translated. Plurals matter. |

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| LLM-written coaching or health decisions | "AI coach" is the market pitch | Non-deterministic, unauditable, a liability for health advice. Explicitly out of scope. | Template-based text now. Store structured insight payloads so an LLM can later *rephrase* them, never decide. |
| "Eat back" exercise calories (adding burned kcal to the daily budget, as MFP does) | Feels motivating and fair | Exercise calorie estimates are inaccurate and encourage over-eating. They also confuse the explanation chain. | Keep targets independent of logged workouts. Activity level covers baseline. Adaptive TDEE (later) handles the real expenditure. |
| Meal-photo AI recognition (MFP made it Premium-only in May 2026) | Looks magical | Accuracy and portion errors, cost, and non-determinism that undermines the "explainable" story. | Fast search, favorites, copy-meal, barcode. |
| Automated meal plans or personalized diets | Obvious next step for coaching | Explicit non-goal until validated. Medical and liability risk. | Insights that point at macro gaps, without prescribing menus. |
| Wearable/biometric integration (Apple Health, Garmin, etc.) | Users ask for step sync | Explicit non-goal. Large integration surface and noisy data. | Manual weight entry. Revisit after v1 validation. |
| Micronutrient tracking (Cronometer's 80+ nutrients) | Power users love it | Needs high-quality verified data you don't have. It would dominate data-curation effort. | Store optional fiber, sugar, sodium and saturated fat if the source provides them. Keep the nutrient schema extensible. |
| Social feed, following, likes, comments, leaderboards | Hevy has a social feed | Out of scope. Adds moderation, privacy and abuse surface. | Keep data private by default. See "Deferred areas". |
| Streaks, points, badges | Retention mechanic | Out of scope. Streaks can also drive guilt and disordered-eating patterns if introduced before the core loop is proven. | Weekly consistency summary in the review. |
| Shaming, red/green "failure" language, extreme calorie warnings, body-judgment copy | Common in competitors' UX | Harmful for users prone to disordered eating. Conflicts with the "no medical advice" stance. | Neutral, behavior-focused language. Enforce a calorie floor. Provide a disclaimer and a safe-guard policy for under-18s. |
| Recipe URL import and full recipe builder | Cronometer, Lose It! and MacroFactor have recipes | High effort (parsing, ingredient matching, servings math) with modest return at v1 | Manual foods plus copy-meal. Later: "saved meal" combos, then recipes. |
| Water, sleep, mood, step tracking | "Wellness app" scope creep | Dilutes the core loop. | Not in v1. |
| Third-party sign-in (Google/Apple) in v1 | Convenience | If any third-party login is offered, App Store rules (to verify) require an equivalent such as Sign in with Apple. It also adds account-linking complexity. | Email/password first. Add social login deliberately in the public-launch phase. |
| Real-time chat with trainers | Natural next step | Deferred. WebSockets, moderation and presence add heavy infrastructure. | See "Deferred areas". |
| Adaptive/ML-based TDEE in v1 | MacroFactor's headline feature | Needs weeks of data and a robust estimator. Easy to get subtly wrong and unsafe. | Static target in v1, with bodyweight logging in place to enable it later. |

## Feature Dependencies

```
Auth (register/login/refresh/roles)
    └──requires──> Profile (sex, age, height, weight, activity, units, tz, locale)
                       └──requires──> Goal + Target computation (pure function)
                                          └──requires──> Daily summary (consumed vs target)

Food data (seeded DB + search)
    ├──requires──> Food i18n (EN/RO names) + serving/unit model
    ├──enhances──> Manual foods (private foods share the same model)
    └──requires──> Meal logging (nutrient snapshot at log time)
                       ├──requires──> Favorites / Recent / Copy-meal / Quick-add
                       ├──requires──> Daily summary + History
                       └──requires──> Barcode scan (local -> OFF fallback -> create-custom miss flow)

Exercise library (seeded + i18n)
    └──requires──> Workout logging (sets/reps/weight)
                       ├──requires──> Previous performance, set types, rest timer
                       ├──requires──> Workout history + per-exercise progress (1RM, PRs)
                       └──enhances──> Routines / repeat workout

Bodyweight logging ──enhances──> Weekly review ──enhances──> Target-adjustment proposals (v1.x)

Event bus (MealLogged, WorkoutLogged, WeightLogged)
    └──requires──> Aggregation layer (daily/weekly rollups, user-timezone days)
                       └──requires──> Rule engine (pure rules + priority + cooldown)
                                          └──requires──> Insight store (rule_id, version, inputs, template, locale)
                                                             ├──requires──> Template i18n (EN/RO + plurals)
                                                             ├──requires──> Async daily/weekly workers
                                                             └──enhances──> Explanations (rendered from the same stored inputs)

Admin (foods/exercises/users) ──requires──> Auth roles + audit log
                              ──enhances──> Food data quality (review queue for user-created foods and barcodes)

Adaptive targets ──conflicts──> Silent auto-change of targets (always require user accept)
"Eat back" exercise calories ──conflicts──> Explainable static targets
Streaks/points ──conflicts──> Shame-free coach tone (if introduced before the core loop is proven)
```

### Dependency Notes

- **Targets require Profile, and Coach requires Targets.** Nearly every deviation rule is "intake vs target", so the target function and its stored inputs must be correct and versioned first. Record which formula version produced each target set.
- **The coach requires aggregation, not raw rows.** Build daily and weekly rollups (per user timezone) once. Rules, the weekly review and later gamification streaks all consume them.
- **Logging completeness gates the coach.** A half-logged day looks like a huge deficit. Coach rules need a "day is sufficiently logged" signal (a user-marked "finished logging", or a threshold), or the daily card will give wrong advice and destroy trust.
- **Explanations require stored inputs.** Persist the numbers used by each rule, so the explanation can be shown, audited and regenerated, and so an LLM layer can later wrap the same payload.
- **Barcode requires local food storage first.** A scan result should be saved as a local food, so repeat scans are fast and independent of OFF availability. This also supports admin curation.
- **Nutrient snapshots in log entries** decouple history from food edits. They are required for correct history and admin fixes.
- **Previous performance, PRs and progress require consistent exercise identity.** Don't let duplicate or renamed exercises fragment history. Admin merge/dedupe becomes valuable.
- **Content i18n requires data-model decisions early.** Retrofitting translations onto foods and exercises after launch is costly.

## MVP Definition

### Launch With (v1)

Core loop, in dependency order, scoped to the "author logs daily for 2+ weeks" success criterion.

- [ ] Auth: register, login, refresh, logout, roles, password reset — foundation for everything
- [ ] Profile, goal and computed targets with a "how we calculated this" breakdown and manual override — onboarding and the first explainability moment
- [ ] Bodyweight logging with trend (gap-fill for the PROJECT.md list) — needed for progress and the weekly review
- [ ] Food search (EN+RO), meal logging with servings and meal slots, edit/delete, daily summary, history — the core nutrition loop
- [ ] Manual foods, favorites, recent foods, copy-meal/yesterday, quick-add — the speed features that decide retention
- [ ] Barcode scan with local-first lookup, OFF fallback and create-on-miss — requested v1 feature, and a free-tier differentiator
- [ ] Exercise library (muscle group, equipment, EN+RO) plus custom exercises — prerequisite for logging
- [ ] Workout logging (sets/reps/weight, set types, previous performance, rest timer, repeat last workout), history, per-exercise progress and PRs — the core training loop
- [ ] Daily insight card with a small rule set (about 6-10 rules), prioritization, cooldown, logging-completeness gate and stored explanation
- [ ] Weekly review (adherence, macro balance, training volume, weight trend if available, 2-3 suggestions)
- [ ] Minimal admin: foods, exercises, users, plus audit log and a user-submitted-food review queue
- [ ] EN+RO UI and coach templates, with plural handling
- [ ] Account deletion/data export and a consent record — store and legal requirement for public launch; cheap now
- [ ] In-progress workout persists locally across app restarts and poor signal

### Add After Validation (v1.x)

- [ ] Routines/templates management (beyond "repeat last") — trigger: the author finds themselves re-building the same sessions
- [ ] Rule-based target-adjustment proposals (weight trend + adherence, accept/modify/reject) — trigger: 3-4 weeks of weight and intake data exist
- [ ] Rule-based lift progression hints (double progression) — trigger: workout history is rich enough
- [ ] Saved meals/combos (precursor to recipes) — trigger: copy-meal is not enough
- [ ] Push notifications for daily insight and logging reminders — trigger: opens drop. Requires the notifications module.
- [ ] Admin: bulk CSV import for foods and exercise translations, duplicate merge, basic usage stats
- [ ] Email verification, social sign-in — trigger: heading toward public launch
- [ ] Exercise media (images/animations via S3) — trigger: library polish

### Future Consideration (v2+)

- [ ] Social and community (profiles, following, feed, gym check-ins/reviews) — needs moderation and privacy design
- [ ] Gamification (streaks, points, badges) — only after the core loop is proven
- [ ] Trainer Q&A (tickets first, chat/scheduling later)
- [ ] Gym subscription wallet and renewal reminders
- [ ] Partner rewards
- [ ] LLM wording layer (rephrases structured insights, never decides)
- [ ] Adaptive TDEE estimation
- [ ] Recipes with URL import; micronutrients; wearables; meal-photo logging

### Deferred areas: what they look like, and how v1 avoids painting itself into a corner

| Deferred area | What competitors do | Cheap v1 design choices that keep the door open |
|---------------|--------------------|-----------------------------------------------|
| **Social** (Hevy: follow, feed, likes, comments, shared workouts) | Opt-in public profiles, workout sharing, activity feed | Use UUID keys. Every user-owned row has `owner_id` and ownership checks in the service layer. Keep a clear public/private split (a separate public-profile concept, never reuse the private profile row). Default all content to private. Emit domain events (`WorkoutLogged`, `MealLogged`) that a future feed can subscribe to. |
| **Gamification** (streaks, points, badges, challenges) | Streak counters, badges, leaderboards | Event log with timestamps and the user's timezone. Compute the "logical day" consistently. Never hard-delete events without a trace. Streaks then become a derived view over events, not a data migration. |
| **Rewards (gym/brand partners)** | Partner discounts, points redemption | Needs anti-abuse rules (logs are user-entered and easy to fake). Keep points ledger design out of v1. Leave an event-driven hook, nothing more. |
| **Trainer Q&A** (tickets, later chat and scheduling) | Coach-client messaging (Trainerize-style products), paid plans | The `trainer` role already exists in RBAC. Later add a trainer-client relationship table with **explicit user consent** for data sharing. Keep insight `author_type` extensible (rule / trainer / llm). Keep trainer data access out of v1 admin. |
| **Gym subscription wallet and renewal reminders** | Membership managers, reminders | Separate module. Never store raw card data. Needs a notification abstraction: build the email seam for password reset in a way that a push/notification service can extend. |
| **LLM language layer** | Chatty "AI coach" features | The structured insight payload (rule id, inputs, template version, locale) is the stable contract. Store rendered text separately, so text can be regenerated or replaced. |

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Auth + profile + target computation with breakdown | HIGH | MEDIUM | P1 |
| Food search + meal logging + daily summary | HIGH | HIGH | P1 |
| Copy-meal / recent / favorites / quick-add | HIGH | LOW | P1 |
| Manual foods | HIGH | LOW | P1 |
| Barcode scan (with miss flow) | HIGH | MEDIUM-HIGH | P1 |
| Bodyweight logging + trend | HIGH | LOW | P1 |
| Exercise library + custom exercises | HIGH | MEDIUM | P1 |
| Workout logging + previous performance + rest timer | HIGH | MEDIUM | P1 |
| Workout history + progress + PRs | HIGH | MEDIUM | P1 |
| Daily insight card (rules + explanation + completeness gate) | HIGH | MEDIUM-HIGH | P1 |
| Weekly review | HIGH | HIGH | P1 |
| EN+RO i18n (UI, content, coach templates) | HIGH | MEDIUM | P1 |
| Minimal admin + audit log | MEDIUM | MEDIUM | P1 |
| Account deletion/export + consent | MEDIUM (HIGH at launch) | MEDIUM | P1 |
| Offline-tolerant in-progress workout | MEDIUM-HIGH | MEDIUM | P1 (basic persist), P3 (full sync) |
| Routines/templates | MEDIUM-HIGH | MEDIUM | P2 |
| Target-adjustment proposals | HIGH | MEDIUM-HIGH | P2 |
| Lift progression hints | MEDIUM-HIGH | MEDIUM | P2 |
| Push notifications | MEDIUM | MEDIUM | P2 |
| Saved meals | MEDIUM | LOW-MEDIUM | P2 |
| Social, gamification, trainer Q&A, wallet, rewards | MEDIUM | HIGH | P3 |
| LLM text layer | LOW-MEDIUM | MEDIUM | P3 |
| Adaptive TDEE | MEDIUM-HIGH | HIGH | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Future consideration

## Competitor Feature Analysis

(Competitor specifics are MEDIUM-LOW confidence: aggregated web sources, not first-party docs.)

| Feature | MyFitnessPal | Cronometer / Lose It! | MacroFactor | Strong / Hevy / Fitbod | Our Approach |
|---------|--------------|----------------------|-------------|------------------------|--------------|
| Target setup | Formula-based once. Gram-based macro editing is Premium. | Formula-based. Cronometer adds micronutrient targets. | Algorithmic, with Coached/Collaborative/Manual modes | n/a | Formula-based with a visible breakdown, editable grams, and a later adaptive path |
| Adaptive TDEE | No | No | Yes, the headline feature. Needs about 2-4 weeks of data. | n/a | v1.x proposals via rules. Adaptive estimator deferred. |
| Food DB | Very large, crowd-sourced (noisy) | Cronometer: verified (USDA etc.), 80+ nutrients. Lose It!: large, fewer micros. | Curated, "verified" emphasis | n/a | Seeded and curated, with admin review, user-private custom foods, and RO coverage as the focus |
| Barcode | Paid (since 2022) | Available | Available | n/a | Free, local-first, OFF fallback, create-on-miss |
| Logging speed | Search, recents, copy | Lose It!: "add yesterday's meal", recipes. Cronometer: copy/paste, recipes. | Fastest logger (published benchmark), copy/paste at item/multi/day level, quick add | n/a | Search + recents + favorites + copy-meal + quick-add |
| Recipes | Yes | Yes (Cronometer imports from URL) | Yes | n/a | Deferred |
| Photo meal scan | Premium-only (May 2026) | Cronometer has photo logging | Some AI features | n/a | Anti-feature in v1 |
| Weekly review/check-in | Reports | Reports/charts | Weekly check-in with accept/modify/reject | Hevy: progress review | Weekly review spanning nutrition + training, with explanations |
| Exercise library + filters | Basic | Basic | n/a (separate app for workouts) | Muscle/equipment filters, custom exercises | Muscle group + equipment, EN+RO, custom exercises |
| Workout logging | Minimal | Minimal | n/a | Fast set logging, rest timer (Strong: separate warm-up/working timers), supersets, previous performance | Core set logging, set types, previous performance, rest timer. Supersets later. |
| PRs, 1RM, charts | No | No | n/a | Both: PRs, estimated 1RM, volume charts. Hevy free has 3-month history window. | PRs, 1RM, volume charts, no history paywall |
| Routines/templates | No | No | n/a | Hevy 4 free, Strong 3 free (custom) | Repeat-last in v1, routines in v1.x |
| Auto-programming/progression | No | No | No | Fitbod: model-driven, opaque. Hevy Trainer: published progression rule. | Rule-based, explainable hints (v1.x) |
| Explainability | Low | Low | Medium (explains target changes) | Fitbod low, Hevy Trainer medium | **Primary differentiator**: every recommendation explained from stored inputs |
| Social | Community forums | Some | Minimal | Hevy: follow/feed | Deferred |
| Pricing | Free tier degraded, Premium about $80/yr | Free + paid tiers | Paid subscription | Free with limits plus Pro | Free, no paywalled core |

## Sources

- MacroFactor reviews and comparisons (calorierankings.com, best-nutrition-apps.com, nutriscan.app, gainframe.app, amyfoodjournal.com): adaptive TDEE, weekly check-in, Coached/Collaborative/Manual modes. MEDIUM-LOW.
- MacroFactor first-party pages (macrofactor.com/macrofactors-algorithms-and-core-philosophy, /new-food-logger, /10-macrofactor-features): algorithm philosophy, copy/paste, logging speed. Surfaced in search results and not opened directly. MEDIUM-LOW.
- MyFitnessPal paywall coverage (xda-developers.com, fitbudd.com, nutrola.app, thenutritionmagazine.com, intakenutrition.io): barcode paywall since 2022, Premium/Premium+ pricing, May 2026 meal-scan paywall. MEDIUM-LOW (cross-corroborated, prices may have changed).
- Hevy vs Strong vs Fitbod comparisons (grabgains.com, setgraph.app, push-pull.app, sensai.fit, repreturn.com), plus Fitbod help center (help.fitbod.me) and Hevy progressive overload page (hevyapp.com): rest timer, routines caps, PRs, 1RM, history limits, Hevy Trainer published rule vs Fitbod opaque model. MEDIUM-LOW.
- Cronometer vs MacroFactor vs Lose It! comparisons (welling.ai, calorierankings.com, nutriscan.app, caleye.fit, loseit.zendesk.com): verified data and micronutrients, recipes, copy/paste, "Add Yesterday's meal", quick add. MEDIUM-LOW.
- BMR/TDEE formula references (calculator.net, omnicalculator.com, caleye.fit, calctypes.com): Mifflin-St Jeor default, Katch-McArdle only with reliable body fat. Formulas, multipliers, Atwater factors and ISSN protein ranges also reflect established literature known to the researcher. HIGH for formulas, MEDIUM for exact deficit and surplus percentages and kcal floors (these vary by app and need a documented product decision).
- Open Food Facts coverage: search results indicated strong coverage in France and Western Europe and patchier elsewhere. No Romania-specific coverage figures were found. **Must be measured empirically** (sample 50-100 common Romanian barcodes) before committing to OFF as the primary source. LOW.
- App Store account-deletion and Sign in with Apple rules, and GDPR special-category health data: training knowledge, not re-fetched. MEDIUM. Verify at the auth/privacy phase.

---
*Feature research for: personal nutrition + workout tracking app with rule-based explainable coach (FitAcademy)*
*Researched: 2026-10-01*
