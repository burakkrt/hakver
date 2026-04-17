# Hakver — Post-MVP Roadmap

Items deferred from MVP scope during the plan review rounds. Each item has a short rationale and the MVP entry point it attaches to so the follow-up PR knows exactly where to slot in.

---

## 1. Error Tracking (Sentry or equivalent)

**Why deferred:** MVP relies on Railway + Vercel dashboards plus Pino structured logs for visibility. These surface aggregate health but not unhandled exceptions tied to specific requests, so a background crash loop can run for hours before anyone notices.

**Follow-up scope:**
- Add Sentry free tier (or self-hosted GlitchTip / Bugsnag) covering backend and frontend
- Backend: `@sentry/node` + NestJS integration, capture unhandled exceptions + rejections, attach `userId` + `requestId` breadcrumbs
- Frontend: `@sentry/nextjs` with source map upload, error boundary integration, breadcrumb trail for navigation and API calls
- Release tracking wired to git SHA on deploy
- Alert rules: error-rate spike, new-issue first-seen, regression of resolved issue
- Redact PII (email, name, bearer tokens) before transport

**Attaches to:** Phase 17 "Monitoring and Error Tracking" section — replace the "to be added later" note with a concrete setup task.

---

## 2. Automated Database Backup

**Why deferred:** MVP keeps backup procedure manual (`pg_dump` + monthly restore test). Manual procedures decay; without automation a tail risk incident can cost weeks of user data.

**Follow-up scope:**
- Scheduled GitHub Actions workflow: weekly `pg_dump --format=custom` against `DIRECT_URL`
- Symmetric-key encryption (GPG or age) using a secret stored in GitHub Actions
- Upload to object storage (Backblaze B2 free tier, Cloudflare R2 free tier, or Supabase Storage)
- Retention policy: 12 weekly snapshots rolling
- Monthly restore verification step (spin up throwaway Postgres, `pg_restore`, run smoke query)
- Runbook committed at `docs/runbooks/backup-restore.md`
- (Optional upgrade) move to Supabase Pro for PITR when traffic warrants

**Attaches to:** Phase 17 "Database Backup Strategy" section — convert the manual procedure into an automated workflow while keeping the manual recipe as a fallback.

---

## 3. Full-text Search on Topic Content

**Why deferred:** MVP search uses pg_trgm on the `title` field only. Title search covers the common intent but misses users searching for a phrase they remember from the body of a topic.

**Follow-up scope:**
- Evaluate between two approaches and pick one per query latency / relevance trade-off
  - Option A: second pg_trgm GIN index on `content` with weighted ranking (`title` match > `content` match)
  - Option B: PostgreSQL `tsvector` with Turkish dictionary support (more accurate, steeper setup)
- Update `GET /topics?search=` service layer to query both fields; keep response shape unchanged
- Add snippet highlighting in the search results page (Phase 13/14)
- Measure p95 latency on the production dataset before and after

**Attaches to:** Phase 5 Topic Listing (Section 10) and Phase 13 Search Results Page.

---

## 4. Deleted-Account Cold Archive Pipeline

**Why deferred:** MVP anonymizes retired accounts immediately (Phase 4 Section 10 Stage 1), which is already sufficient to hide personal data from the product surface. A follow-up scheduled job carries the residual rows to a detached archive database so the hot Postgres stays free of any identifier tied to a retired human. This is a KVKK-maturity step, not a launch-blocker.

**Follow-up scope:**
- Nightly cron that selects `User` rows with `deletedAt < now() - interval '30 days'` and migrates their downstream references to an archive database
- Archive schema stores only the minimum needed to keep anonymous presentation working: `Topic.authorDisplay = "Silinmiş Kullanıcı"`, `Vote.topicId`, `AnonymousIdentity.displayName`. No personal fields, no FK back to the original User row
- After migration, the primary `User` row is hard-deleted
- Listings and content continue to render the "Silinmiş Kullanıcı" card, now sourced from the archive mirror
- Full KVKK runbook documenting retention, access, and destruction windows

**Attaches to:** Phase 4 Section 10 "Stage 2 — Cold archive + full personal-data removal" placeholder; the Stage 2 section already references this follow-up by name.

---

## 5. Resolved / Outcome Flair on Topics

**Why deferred:** MVP already has `updateNote` for post-window annotations. A dedicated "Çözüldü" status with a single outcome choice ("Haklıymışım" / "Haksızmışım" / "Belirsiz") gives threads a clear closing mechanic but expands the UI surface. Defer until MVP usage data shows closing is a recurring pattern.

**Follow-up scope:**
- New enum `TopicResolution`: `RIGHT_WAS_ME`, `WRONG_WAS_ME`, `UNCLEAR`
- New column `Topic.resolution: TopicResolution?` (nullable = unresolved)
- Endpoint `PATCH /topics/:id/resolution` (author-only, within and after edit window), optional explanatory note (reuse update-note pattern)
- Card badge renders the Turkish outcome label with the appropriate accent colour (success for "Haklıymışım", danger for "Haksızmışım", neutral for "Belirsiz")
- Filter option on listing: only resolved, only unresolved
- Resolved topics optionally lock further voting — decide during design based on comparable platform behaviour (Reddit flair does not lock voting)

**Attaches to:** Phase 5 topic schema and endpoints, Phase 13 topic card + filter UI.

---
