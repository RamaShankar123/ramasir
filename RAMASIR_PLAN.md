# RamaSir — Alignment Doc

**Status:** 🟡 DRAFT — needs Rama's sign-off before any build work
**Date:** 2026-10-25 · **Repo:** `github.com/RamaShankar123/ramasir` · **Folder:** `D:\RamaSir`

---

## 1. What this is

A teaching platform for **one teacher — Rama**. His own domain, his own app, his own brand. Students find him, enrol, attend **live classes**, and watch **recorded ones**.

This is *not* SevaBook. SevaBook is a multi-tenant SaaS sold to coaching institutes. RamaSir is a single-teacher product, and it is Rama's own teaching business.

**The goal is to start teaching, fast.** Everything below is shaped around that, not around building something impressive.

---

## 2. Decisions locked (2026-10-25)

| # | Decision |
|---|---|
| D1 | Purpose = **teaching, quickly**. Not a learning/portfolio exercise. Claude writes the code. |
| D2 | **Own domain + own app.** Not a page inside sevabook.co.in. |
| D3 | **Live classes AND recorded videos.** Both. |
| D4 | Reuse the SevaBook codebase rather than building from scratch. |

**Supersedes** the July 2026 "Rama Sir" plan (build-from-scratch Next.js + Postgres, Rama codes, Claude guides, for interview prep). That was a *learning* project and was never started — no folder was ever created. If interview prep is still wanted, it should be a separate, smaller exercise; mixing it into a product you need working will deliver neither.

---

## 3. Architecture — one backend, two front doors

```
        ramasir.in     ──┐
                         ├──→   one EC2 · one Node API · one SQLite DB · one S3
     sevabook.co.in     ──┘      (RamaSir = one institute row inside it)

        RamaSir app   ──┐
                        ├──→   same API
        SevaBook app  ──┘
```

**Why not a second full deployment:** SevaBook's multi-tenancy exists precisely so one backend can serve many institutes. A second backend would mean two servers, two databases, two sets of secrets, two backup routines and two deploys — permanently, for one teacher. The prod box is a **t3.small with 1.9 GB RAM already running three apps** (`sevabook`, `swadeshi-api`, `swadeshi-admin`); adding a fourth Node process plus its ffmpeg transcoding is a real risk to the live platform.

**What this means in practice:** RamaSir is an `institutes` row. Rama is its owner + its teacher. Everything already keyed on `institute_id` — batches, enrolments, live classes, recordings, fees, attendance — works for him on day one with no new backend code.

**The trade-off, stated plainly:** RamaSir's data lives in the same SQLite file as SevaBook's other institutes. Rama owns both, so this is fine today. If RamaSir ever needs to be sold or split off, the data has to be migrated out — possible (every table carries `institute_id`), but real work. Accepting this now is what buys the fast start.

---

## 4. Reused vs new

### Reused as-is — no code written
- **Backend, entirely.** Auth, institutes, courses, batches, class sessions, live Agora tokens, recording pipeline, homework, quizzes, attendance, fees, notifications, parent dashboard.
- **Live classes + auto-recording.** Agora cloud recording → S3 → CloudFront, with the recovery machine already battle-tested.
- **Student / teacher / parent dashboards** (web + mobile screens).
- **Payments** — Razorpay, already integrated.
- **Admin CMS** — `app_config` + the App Home editor.

### Genuinely new — this repo
| # | Piece | Why it can't be reused |
|---|---|---|
| N1 | **RamaSir landing page** | SevaBook's landing sells SaaS to institute owners. RamaSir's must sell *one teacher* — his subjects, his batches, his credibility, enrol/demo CTA. Different content, different job. |
| N2 | **nginx vhost + SSL for ramasir.in** | New front door on the same box. |
| N3 | **Brand assets** | Logo, colours, app icon, splash. |
| N4 | **Flutter flavor** | New package id (`com.ramasir.app`), app name, icon, theme. Lives as a build flavor in the mobile codebase; config + assets tracked here. |
| N5 | **Play Store listing** | New app = new listing, screenshots, privacy policy on the new domain, data-safety form, content rating. |

Note N1 can pull live data from the **existing public API** — `/api/public/institutes/:slug`, `/api/public/marketplace/batches`, `/api/public/courses` — so the landing page needs no backend of its own.

---

## 5. Phases

### Phase 1 — Teaching live on the web · ~1 week
1. Buy domain, point DNS at `13.238.175.144`
2. nginx vhost + Let's Encrypt for `ramasir.in`
3. Create the RamaSir institute; Rama as owner + teacher
4. Build + deploy the RamaSir landing page
5. Serve the existing app shell under `ramasir.in` for login / dashboards
6. First course + batch + a test live class end to end

**At the end of Phase 1 Rama can teach and take students.** Students use the browser.

### Phase 2 — The app · ~1 week
7. Flutter flavor: `com.ramasir.app`, name, icon, splash, colours
8. **New upload keystore** — generated once, backed up immediately (losing it = locked out of updates forever)
9. Build AAB, install on a real phone, test live class + recording + enrol

### Phase 3 — Play Store · ~3 weeks, mostly Google's clock
10. Store listing, screenshots, privacy policy URL, data safety, content rating
11. ⚠️ **Closed testing: 12 testers × 14 days.** This applied to SevaBook's submission and is expected to apply again — **verify in Play Console before planning around it**, since it is the single longest item here and nothing can shorten it.
12. Review (~3–7 days) → production

**Teaching starts: ~1 week. App live: ~5–6 weeks.** The gap is Google's process, not development.

---

## 6. Cost

| Item | Cost |
|---|---|
| Domain `ramasir.in` | ~₹800–1,500 / year |
| Hosting | **₹0 extra** — same box, same DB |
| Play Store | ₹0 — developer account already paid |
| Agora (live classes) | per-minute, scales with teaching hours |
| S3 + CloudFront (recordings) | small, grows with the recording library |

Separate EC2 instead would add ~₹1,400/month plus the ops burden. Not recommended.

---

## 7. Open decisions — need Rama

| # | Question | Recommendation |
|---|---|---|
| Q1 | Domain name — `ramasir.in`, `ramasir.co.in`, something else? | Check availability first; `.in` is cheap and right for the audience |
| Q2 | Android package id — permanent, can never change | `com.ramasir.app` |
| Q3 | Do students **pay on the platform**, or offline? | Offline first (fees module tracks it). Online payment pulls in the Razorpay/PA-licence question — defer it. |
| Q4 | Is RamaSir's landing page a **separate small site**, or the SevaBook SPA re-skinned? | Separate small site. One teacher's page should not carry a SaaS app's bundle, and it keeps SevaBook untouched. |
| Q5 | Does RamaSir show in SevaBook's public marketplace too? | No — keep `is_public = 0`. Separate brands. |
| Q6 | Who are the 12 testers? | Needed before Phase 3 can start; worth lining up during Phase 1 |

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Play Store 14-day gate** is the critical path and cannot be compressed | Start Phase 2 early; teach on web meanwhile |
| **Shared DB with SevaBook** — a bad migration or hard-delete affects both | Same discipline already in place: snapshot before destructive migrations |
| **t3.small, 1.9 GB RAM**, now serving two brands' live classes | Watch during the first real class; vertical scale is a one-click restart |
| **New keystore lost** = permanently locked out of app updates | Back it up to two places the moment it is generated; never commit it |
| Two brands on one box means **one outage takes down both** | Accepted for now; revisit if RamaSir grows |

---

## 9. What lives in this repo

```
D:\RamaSir
  RAMASIR_PLAN.md      ← this doc
  web/                 ← landing page (N1)
  infra/               ← nginx vhost, deploy script (N2)
  brand/               ← logo, icons, splash, colour tokens (N3)
  app/                 ← Flutter flavor config + store assets (N4, N5)
```

The backend and the shared app code stay in their own repos. Nothing is forked.

---

## 10. Next step

Rama reads this and signs off — or changes it. **No build work starts until then**, except anything explicitly approved.

First thing after sign-off: **buy the domain** (Q1). DNS propagation is slow and everything in Phase 1 waits on it, so it should be started before any code.
