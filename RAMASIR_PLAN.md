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
| D5 | **Separate database.** Same EC2 box, same S3 bucket, but RamaSir gets its own SQLite file. *(Rama, 2026-10-25.)* |

**Supersedes** the July 2026 "Rama Sir" plan (build-from-scratch Next.js + Postgres, Rama codes, Claude guides, for interview prep). That was a *learning* project and was never started — no folder was ever created. If interview prep is still wanted, it should be a separate, smaller exercise; mixing it into a product you need working will deliver neither.

---

## 3. Architecture - shared box, separate database

```
   ramasir.in    --> nginx -->  node :3002  -->  ramasir.sqlite   --+
                                                                    +--> same S3 bucket
 sevabook.co.in  --> nginx -->  node :3001  -->  database.sqlite  --+     (separate prefixes)
                       one EC2 box, one Agora account
```

**One box, one S3 bucket, two databases, two Node processes.** RamaSir runs the same codebase against its own SQLite file, so its data is fully independent of SevaBook's - no shared rows, no migration needed if the two are ever split, no risk of a SevaBook query touching RamaSir's students.

The cost of that independence is a second process to deploy, back up and watch. Accepted deliberately.

### What separating the DB actually requires

A separate database is not a config toggle - two deployments sharing one S3 bucket and one Agora account will **collide in three places**, because both databases start their auto-increment ids at 1. All three are silent failures, which is what makes them dangerous.

| # | Collision | Consequence | Fix |
|---|---|---|---|
| C1 | `db.js:4` hardcodes `database.sqlite` | One process can only ever open one DB | `DB_PATH` from env, defaulting to the current path |
| C2 | S3 key `sevabook/inst_<id>/course_<id>/batch_<id>/...` carries **no random token** | RamaSir's `inst_1/course_1/batch_1/` **is the same folder** as SevaBook's. Recordings interleave - and the delete path lists by prefix, so one brand's cleanup can **delete the other's files** | `S3_PREFIX` from env (`sevabook` / `ramasir`), applied to every key builder |
| C3 | Agora channel `sevabook_class_<batchId>_<date>` | Both DBs have a `batch 1`. Same batch id + same date = **the same Agora channel**: a RamaSir student can land in a SevaBook class | `CHANNEL_PREFIX` from env |

The `mp4` / `poster` / `subtitle` keys are safe - they already carry a random token from the September security fix. The folder-style prefixes and the channel names do not.

**Both new env vars must default to `sevabook`**, so SevaBook's behaviour stays byte-identical and the 101 already-re-keyed recordings keep resolving.

### Untested: creating a SevaBook database from scratch

RamaSir's DB starts empty and runs all 95 migrations. But `ARCHITECTURE_MAP.md` records a **schema split-brain**: `db.js` defines ~6 tables and ~13 `class_sessions` columns inline that have **no migration file**. A fresh database is therefore bootstrapped by migrations *and* by `db.js` - a path nobody has exercised in a long time.

**This must be proven locally before it is attempted on the box.** It is the single most likely thing to go wrong in Phase 1.

### Resource reality

The box is a **t3.small - 2 vCPU, 1.9 GB RAM** already running `sevabook`, `swadeshi-api` and `swadeshi-admin` (~274 MB between them, ~1.2 GB free). A fourth Node process at idle is fine. The pressure point is **ffmpeg**: recording transcode runs on the API host, and each process keeps its own concurrency-1 queue - so two brands can now transcode **simultaneously on 2 vCPU**. Likely fine given they teach at different hours; worth watching after the first real RamaSir class.

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
| N6 | **`DB_PATH` / `S3_PREFIX` / `CHANNEL_PREFIX` env plumbing** (in the SevaBook repo) | Required by D5. ~25 call sites; all default to today's values so SevaBook is unchanged. |
| N7 | **Second pm2 app + port 3002 + its own `.env` and backup** | A separate DB means a separate process. |

Note N1 can pull live data from the **existing public API** — `/api/public/institutes/:slug`, `/api/public/marketplace/batches`, `/api/public/courses` — so the landing page needs no backend of its own.

---

## 5. Phases

### Phase 1 - Teaching live on the web, ~1.5-2 weeks
1. Buy domain, point DNS at `13.238.175.144`
2. **Env plumbing (C1-C3)** in the SevaBook repo, defaults unchanged, deployed and verified to be a no-op for SevaBook
3. **Prove a from-scratch database locally** - all 95 migrations on an empty file, then boot and smoke it
4. Second pm2 app on :3002 with `ramasir.sqlite`, `S3_PREFIX=ramasir`, `CHANNEL_PREFIX=ramasir`
5. nginx vhost + Let's Encrypt for `ramasir.in`
6. Create the RamaSir institute; Rama as owner + teacher
7. Build + deploy the RamaSir landing page; serve the app shell for login / dashboards
8. First course + batch + a **real live class end to end**, confirming the recording lands under the `ramasir/` S3 prefix

**At the end of Phase 1 Rama can teach and take students.** Students use the browser.

### Phase 2 — The app · ~1 week
7. Flutter flavor: `com.ramasir.app`, name, icon, splash, colours
8. **New upload keystore** — generated once, backed up immediately (losing it = locked out of updates forever)
9. Build AAB, install on a real phone, test live class + recording + enrol

### Phase 3 — Play Store · ~3 weeks, mostly Google's clock
10. Store listing, screenshots, privacy policy URL, data safety, content rating
11. ⚠️ **Closed testing: 12 testers × 14 days.** This applied to SevaBook's submission and is expected to apply again — **verify in Play Console before planning around it**, since it is the single longest item here and nothing can shorten it.
12. Review (~3–7 days) → production

**Teaching starts: ~1.5-2 weeks. App live: ~5-6 weeks.** The separate database adds roughly 3-5 days over the shared-DB design (the prefix plumbing and proving a fresh DB); the app timeline is unchanged because it is gated by Google, not by us.

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
| Q5 | Does RamaSir show in SevaBook's public marketplace too? | Moot now - separate databases, so SevaBook's marketplace cannot see RamaSir's batches at all. |
| Q6 | Who are the 12 testers? | Needed before Phase 3 can start; worth lining up during Phase 1 |

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Play Store 14-day gate** is the critical path and cannot be compressed | Start Phase 2 early; teach on web meanwhile |
| **Fresh-DB bootstrap is an untested path** (schema split-brain) | Prove it locally in Phase 1 step 3, before the box is touched |
| **S3 / Agora namespace collision** silently mixing two brands' recordings and live classes | C2 + C3 env prefixes, verified by checking the first RamaSir recording's actual S3 key |
| **Two ffmpeg transcodes at once on 2 vCPU** | Watch after the first real class; vertical scale is a restart |
| **t3.small, 1.9 GB RAM**, now serving two brands' live classes | Watch during the first real class; vertical scale is a one-click restart |
| **New keystore lost** = permanently locked out of app updates | Back it up to two places the moment it is generated; never commit it |
| Two brands on one box means **one outage takes down both** | Accepted for now; revisit if RamaSir grows |
| **Two databases = two backup routines.** A snapshot discipline that only covers SevaBook silently leaves RamaSir unprotected | Back up both files in the same step, from day one |

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
