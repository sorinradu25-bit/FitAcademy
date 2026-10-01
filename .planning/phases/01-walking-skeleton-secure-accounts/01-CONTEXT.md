# Phase 1: Walking Skeleton & Secure Accounts - Context

**Gathered:** 2026-10-02
**Status:** Ready for planning

<domain>
## Phase Boundary

The author can install FitAcademy on their own Android phone, create an account against the backend deployed in an EU region, give (and later withdraw) explicit health-data consent, stay securely signed in, log out, and reset a forgotten password by email, all in English or Romanian. The phase also lays the hard-to-retrofit foundations listed in ROADMAP.md (module boundaries, outbox, i18n conventions, clock/DST fixtures, `/api/v1` + client-config, nightly backups with a tested restore).

Requirements: AUTH-01…06, PLAT-01…03.

Not in this phase: profile fields beyond email/password (Phase 2), any health logging (Phase 3+), account deletion and data export (v2), social login (out of scope).

</domain>

<decisions>
## Implementation Decisions

### Signup & Email Verification
- **D-01:** The signup screen collects **email + password only**. Name, date of birth, sex, height, goal, etc. are collected in Phase 2 onboarding. The 18+ age gate therefore lives in Phase 2, not at signup.
- **D-02:** Email verification is **requested but non-blocking**: the user enters the app immediately, a persistent banner asks them to confirm their address, and a "resend" action exists. The account stores `email_verified_at`.
- **D-03:** **Password-reset emails are sent only to verified addresses.** For unverified or unknown addresses, the reset endpoint returns the same generic response (no account enumeration).

### Health-Data Consent (GDPR Art. 9)
- **D-04:** Consent is a **separate screen shown right after signup**, not a checkbox on the signup form. It contains a short plain-language explanation (EN/RO), a link to the privacy policy, and an **unticked** checkbox. Consent is unbundled from terms acceptance. — **Reversibility:** costly — consent records feed every health module's access checks from Phase 3 onward.
- **D-05:** Consent is stored as an **append-only consent record** (type, policy version, granted/withdrawn, timestamp, locale), not a boolean flag, so grant/withdraw history is auditable. — **Reversibility:** one-way — consent history is a GDPR record; collapsing it to a flag later would lose required evidence.
- **D-06:** **Declining or withdrawing consent puts the account into a read-only, health-locked mode**: the account remains, health features (logging, coach — later phases) show a locked state with an explanation and a "give consent" button. Existing health data is **kept** (deletion arrives with ACCT-01 in v2). Withdrawal is available from Settings. Backend must enforce this (not just the UI): health-module endpoints reject writes without active consent.

### Sessions
- **D-07:** **Sliding 30-day inactivity window.** Short-lived access token (~10–15 min) + rotating refresh token; each successful refresh extends the session by 30 days. A user who opens the app at least once a month is never forced to log in again. No absolute session lifetime in v1.
- **D-08:** **Unlimited concurrent devices.** Each device login creates its own refresh-token family; logout revokes only that device's family. **No session-list / remote-logout UI in v1.**
- **D-09:** Refresh-token reuse detection revokes the affected family (research: single-flight refresh on the client + short server-side grace window to avoid false logouts from parallel 401s).

### Passwords & Reset
- **D-10:** Password policy: **minimum 10 characters, no composition rules**, reject very common / known-breached passwords (NIST SP 800-63B style). Validation errors are returned as stable codes + params and translated on the client (EN/RO).
- **D-11:** The reset link opens a **simple server-rendered web page** (EN/RO, chosen from the link/request locale) where the new password is set; it works on any device, with or without the app installed. **No deep linking in Phase 1.** Reset tokens are single-use, short-lived, and stored hashed.
- **D-12:** The password-reset (and verification) email is sent via a **background job**: this is the first real worker job and proves the job runner in production.

### Phone, Install & Barcode Spike
- **D-13:** The author has **both iPhone and Android**. **Phase 1 develops and verifies on Android first**; iOS verification is deferred until the author decides whether to pay for Apple Developer (~$99/yr). Code stays cross-platform (Expo); just don't block on iOS checks.
- **D-14:** Distribution to the author's phone: **APK built with EAS Build** (cloud), installed via link/QR. Development build from day one (not Expo Go). No Google Play account needed in this phase.
- **D-15:** **Barcode spike acceptance:** on the author's Android phone, using an Expo development build with `expo-camera`, scan **10 real household products** (EAN-13, including glossy/curved packaging). **Pass = at least 9/10 read correctly in under 2 seconds each.** Fail → switch mobile stack to Flutter + `mobile_scanner` before further mobile work. This spike runs first in the phase.

### Claude's Discretion
- **Default app language:** follow the device locale (RO if the phone is Romanian, otherwise EN), with a language switch in Settings that persists per account.
- **Email provider:** pick an EU-friendly transactional email service with a free tier suitable for a handful of users (research to recommend; behind a provider interface so it can be swapped).
- **Domain:** the API and the reset page need an HTTPS hostname. If the author has no domain yet, research should propose the cheapest sensible option (buy a short domain vs. a free subdomain) and flag it as an author action.
- **Login brute-force protection:** rate limiting on login, signup, reset and verification endpoints (Redis), with values chosen by research.
- **Effect of a password change/reset on existing sessions:** standard practice (revoke other devices' refresh families) unless research finds a reason otherwise.
- Exact token lifetimes within the bounds above; error-code naming; screen layout of auth screens.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project scope & requirements
- `.planning/PROJECT.md` — vision, constraints (security, i18n, EU hosting), key decisions
- `.planning/REQUIREMENTS.md` — AUTH-01…06, PLAT-01…03 (this phase); ACCT-01 (account deletion, v2 launch blocker)
- `.planning/ROADMAP.md` §Phase 1 — goal, 5 success criteria, notes (spikes, ADRs, locked foundations)

### Research
- `.planning/research/SUMMARY.md` — consolidated decisions and phase implications
- `.planning/research/STACK.md` — versions: FastAPI, SQLAlchemy 2.1, Alembic, psycopg 3, Procrastinate, PyJWT + pwdlib[argon2], Expo SDK 57, i18next, hosting (Hetzner + Caddy + R2)
- `.planning/research/ARCHITECTURE.md` — module boundaries (`public.py`, import-linter), outbox-lite events, error contract, API versioning + `client-config`, i18n conventions
- `.planning/research/PITFALLS.md` — refresh-token rotation races, GDPR Art. 9 consent, health data out of logs, backups/restore, solo-dev over-engineering

### Original vision
- Google Doc "FitAcademy Structure / Health Companion – Project Overview" (repo layout under "Starting Point" is reflected in ROADMAP Phase 1 notes)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None: greenfield repository (only planning docs, README, `docs/REZUMAT-PROIECT.md`).

### Established Patterns
- Commit style: Conventional Commits (`docs:`, `chore:`, `feat:`, `fix:`). **Do not add `Co-Authored-By: Claude` or other Claude attribution to commits or PRs in this repo** (author request; portfolio repo).
- `.gitignore` excludes `.claude/`, research cache, and discuss checkpoints.

### Integration Points
- Target layout from the vision doc: `apps/api` (FastAPI), `apps/mobile` (Expo), `apps/web-admin` (Next.js, Phase 7), `workers/`, `infra/`, `docs/adr/`.
- GitHub remote: `github.com/sorinradu25-bit/FitAcademy` (public). CI should run on GitHub Actions.

</code_context>

<specifics>
## Specific Ideas

- The author wants the repo to read well in job interviews: ADRs in `docs/adr/`, clean per-phase history, a PR per phase, and a `phase-1` tag at the end of the phase.
- The author writes in Romanian; user-facing project docs may be bilingual, code and identifiers stay English.

</specifics>

<deferred>
## Deferred Ideas

- Session list with remote logout per device: possible v2 settings feature.
- In-app deep link for password reset (universal/app links): revisit when a domain and store builds exist.
- iOS verification and Apple Developer account: decide before any iOS testing or App Store work.
- Google Play internal testing track: belongs with store-readiness work (v2, alongside ACCT-01).

</deferred>

---

*Phase: 01-walking-skeleton-secure-accounts*
*Context gathered: 2026-10-02*
