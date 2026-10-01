# Phase 1: Walking Skeleton & Secure Accounts - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-10-01 → 2026-10-02 (paused once, resumed from checkpoint)
**Phase:** 01-walking-skeleton-secure-accounts
**Areas discussed:** Signup & consent, Sessions & passwords, Phone & install

---

## Signup & consent

**What's on the signup screen itself?**

| Option | Description | Selected |
|--------|-------------|----------|
| Email + password only | Minimal friction; profile details come in Phase 2 onboarding | ✓ |
| Email + password + name | Display name from the start | |
| Email + password + DOB | Age 18+ check right at signup | |

**Must users verify their email before using the app?**

| Option | Description | Selected |
|--------|-------------|----------|
| Verify, but don't block | Straight into the app; banner asks to confirm; reset only to verified addresses | ✓ |
| Must verify first | App locked until the link is clicked | |
| No verification | Simplest; typos make password reset impossible | |

**How is the health-data consent presented at signup?**

| Option | Description | Selected |
|--------|-------------|----------|
| Separate screen after signup | Own screen, plain explanation, privacy link, unticked checkbox | ✓ |
| Checkbox on the signup form | Separate unticked checkbox under terms | |
| You decide | Whatever satisfies GDPR with least friction | |

**What happens if someone declines consent, or withdraws it later?**

| Option | Description | Selected |
|--------|-------------|----------|
| Read-only + clear path | Account stays; health features locked with explanation + consent button; data kept | ✓ |
| Block the app entirely | Only the consent screen is visible | |
| Lock and delete health data | Withdrawal also deletes logged health data after confirmation | |

---

## Sessions & passwords

**How long do you stay signed in without re-entering the password?**

| Option | Description | Selected |
|--------|-------------|----------|
| 30 days of inactivity | Opening the app at least monthly keeps you signed in | ✓ |
| 90 days, then re-login | Password required again after 90 days regardless | |
| You decide | Standard secure default | |

**Multiple devices at once?**

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, unlimited | Each device has its own session | ✓ |
| Yes + session list | See devices and log one out remotely | |
| Single device | New login logs out the old device | |

**Password rules?**

| Option | Description | Selected |
|--------|-------------|----------|
| Min 10 + common-password check | No composition rules; NIST style | ✓ |
| Min 8 + upper/digit/symbol | Classic rules | |
| Min 8 only | Simple, weaker | |

**Where is the new password chosen after clicking the reset link?**

| Option | Description | Selected |
|--------|-------------|----------|
| Simple web page | Server-hosted page (EN/RO), works on any device | ✓ |
| Directly in the app | Deep link; needs domain setup for iOS/Android | |
| 6-digit code | Code by email, entered in the app | |

---

## Phone & install

**Which phone do you have / want to test on?**

| Option | Description | Selected |
|--------|-------------|----------|
| iPhone (iOS) | Needs Apple Developer (~$99/yr) for lasting installs | |
| Android | Free APK install | |
| Both | Test on both | ✓ |

**Which phone is primary in Phase 1?**

| Option | Description | Selected |
|--------|-------------|----------|
| Android first | Free, no expiry; iPhone later | ✓ |
| iPhone first | Pay Apple Developer or rebuild every 7 days | |
| Both from the start | Every phase verified on both | |

**How does the app get onto the Android phone?**

| Option | Description | Selected |
|--------|-------------|----------|
| APK via EAS Build | Cloud build, install via link/QR | ✓ |
| Google Play Internal Testing | Needs Play account (~$25) | |
| Local build + USB | Android Studio on the Mac | |

**What counts as a successful barcode spike?**

| Option | Description | Selected |
|--------|-------------|----------|
| 10 real products, 9 of 10 | EAN-13 incl. glossy/curved; <2 s each; fail → Flutter | ✓ |
| Just read one code | Minimal test | |
| You decide | Reasonable threshold | |

---

## Claude's Discretion

- Default app language (device locale, switch in Settings)
- Transactional email provider
- Domain/hostname choice (flag as author action)
- Rate-limit values; session revocation on password change; token lifetimes; error-code naming; auth screen layout
- Area "Domain, email & language" was not discussed: user chose to write CONTEXT.md with standard defaults

## Deferred Ideas

- Session list with remote logout
- In-app deep link for password reset
- iOS verification / Apple Developer account
- Google Play internal testing track
