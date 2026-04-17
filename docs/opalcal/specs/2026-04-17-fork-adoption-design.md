# OpalCal via cal.diy fork — adoption design (Path 1-lite)

**Status:** Proposed
**Date:** 2026-04-17
**Author:** audit synthesis across 9 parallel subagents (auth, calendar, scheduling, payments/security/observability, ux, ops, architecture-fit, fork-survey, upstream-posture)
**Supersedes:** the prior "Launch Alpha (v0.3)" scope defined on top of the custom OpalCal codebase at `/Users/marten/Code/ade_anima/opalcal/`

---

## Summary

OpalCal pivots from a custom-built Next.js app to a **fork of `calcom/cal.diy`** at `AdeAnima/opalcal-next`. The current custom codebase becomes a reference implementation; the fork is the new source of truth.

- **Fork strategy:** tracking fork with monthly rebase. Structural changes avoided to keep rebase cost low.
- **Deployment:** prebuilt Docker image (`calcom.docker.scarf.sh/calcom/cal.diy`) to avoid the 6.9 GB source-build footprint on PaaS.
- **Upstream contribution:** CalDAV correctness + deployment-slimming PRs go upstream. Apple Sign-In, EU-privacy branding, OpalCal-specific UX stay in the fork.
- **Accept the architectural trade-off:** cal.diy treats CalDAV as a "Beta" integration. It works for CRUD (tsdav + RFC 5545 + VTIMEZONE + SCHEDULE-AGENT injection). It doesn't have incremental sync / ctag polling. This is good enough for alpha; revisit in v2.0.

---

## Why this shape (decision trail)

1. **Competitive audit:** cal.diy (MIT community fork of Cal.com) shipped April 2026. Clones 100+ of the OpalCal roadmap items for free. Our only remaining wedge is **Apple-first UX + EU-privacy** (no competitor owns either). "Open-source scheduler" and "CalDAV-capable scheduler" are both solved.
2. **Architectural audit:** rewriting cal.diy's schema to be "CalDAV-first" costs 6-8 weeks and guarantees weekly merge conflicts against upstream. Not worth it — "CalDAV works" is sufficient.
3. **Upstream-posture audit:** `calcom/cal.diy` is a rename of `calcom/cal.com` (same 41k-star repo, same PR numbering). Cal.com-employee PRs merge in 6–24h. Opinionated community PRs (Apple, EU, CalDAV improvements) stagnate 1–4 months or get rejected. Upstreaming generic fixes is realistic; branding/ideology features must stay downstream.
4. **Fork-survey audit:** of 12,824 forks, only `onehashai/Cal-ID` has productive divergence and it's AGPL+EE (unusable for MIT OpalCal). "Slim / CalDAV-first / EU-privacy" is uncontested whitespace.
5. **Deployment reality:** cal.diy's dev checkout is 6.9 GB (3.3 GB `node_modules`). Railway free tier (5 GB) is unworkable from source. Prebuilt image or a different host (Hetzner €4 Germany box, Fly.io Frankfurt) is required.

---

## What's effectively free after adoption

Items from the pre-pivot OpalCal roadmap that cal.diy already ships or mostly ships, with updated effort estimates:

| # | Title | Before | After | Evidence in cal.diy |
|---|---|---|---|---|
| #7 | Multi-tenant User + Org + query scoping | XL | **S (rename only)** | `User` + `Organization` + `Profile` + `Membership` all richer than planned (`packages/prisma/schema.prisma:401-765`) |
| #8 | Auth.js v5 + env-driven providers | XL | **S (add Apple only)** | NextAuth v4 + env-gated Google + Azure AD + magic-link + credentials already shipping (`packages/features/auth/lib/next-auth-options.ts`) |
| #17 | Public host profile `/@handle` | S | **XS** | `/[user]` route + `allowSEOIndexing` + full view (`apps/web/modules/users/views/users-public-view.tsx`) |
| #18 | Calendar overlay on booking pages | M | **S** | Provider-agnostic overlay with zustand store (`packages/features/bookings/Booker/components/OverlayCalendar/`) |
| #23 | Microsoft Graph + Microsoft sign-in | XL | **XS** | 600-LOC Office365 impl + Azure AD provider (`packages/app-store/office365calendar/`) |
| #24 | Stripe Connect dual-mode | M | **S** | Full Connect OAuth + PaymentService (`packages/app-store/stripepayment/`) — add single-key wrapper |
| #26 | Cloudflare Turnstile | S-M | **XS** | `packages/lib/server/checkCfTurnstileToken.ts` + component |
| #32 | CSP nonce + Stripe allowlisting | M | **XS** | `apps/web/lib/csp.ts` + `buildNonce.ts` |
| #33 | Event-level privacy (busy only) | M | **S** | `EventType.hideCalendarEventDetails` + CLASS:PRIVATE injection |
| #35 | Webhook V2 signed payloads | M | **XS** | HMAC-SHA256 ships (`sendPayload.ts:296-298`), add timestamp to prevent replay |
| #40 | Round-robin transparency | M | **XS** | Full `BookingAudit` + `AssignmentReason` + `WrongAssignmentReport` models |
| #41 | i18n EN + DE | L | **XS** | 44 locales fully translated, DE is 4,639 lines |
| #49 | Policy / audit / compliance | XL | **L** | `booking-audit` package covers ~60% (immutable, actor-typed, operationId) |
| #50 | Embed adoption | M | **S** | `@calcom/embed-core` + `@calcom/embed-react` + `@calcom/embed-snippet` |
| #58 | Web-component `<opalcal-booking>` | M | **XS** | Custom elements already registered, 90% done |
| #61 | Managed event types | M | **S** | `SchedulingType.MANAGED` + `parentId` self-relation ships |
| #63 | SMTP env vars | S | **XS** | Nodemailer + `EMAIL_SERVER_*` convention |
| #75 | Onboarding checklist | M | **S** | 5-step wizard exists — dashboard checklist layer remains |
| #77 | Billing page placeholder | S | **XS** | Settings scaffold exists |
| #79 | E2E cancel/reschedule/payment | M | **S** | Specs exist (`apps/web/playwright/reschedule.e2e.ts` + `payment.e2e.ts`) |
| #21 | iCloud setup E2E | M | **S** | Full template at `packages/platform/examples/base/tests/connect-atoms/apple-connect.e2e.ts` |
| #72 | Stripe Connect registration | (human) | (human) | Code done, human registration unchanged |
| #12 | Scheduling-limit helper + tz fix | M | **XS** | `packages/lib/intervalLimits/` has superior implementation |
| #13 | Structured logging | L | **M** | tslog + DistributedTracing + next-axiom wired; extend mask list |

**Estimated saved effort: ~3-4 months.**

## What stays greenfield (or near-greenfield)

Items where cal.diy offers no help — effort unchanged:

| # | Title | Effort | Notes |
|---|---|---|---|
| #9 | **Apple Sign-In** → chained iCloud app-password | L (~2 weeks) | Core wedge. Follow Azure AD branch in `next-auth-options.ts:600-727` as template. Adds new provider + guided iCloud setup wizard. |
| #15 | CalDAV poll reconciler + ctag / sync-token | L | cal.diy has **no** CalDAV caching or incremental sync. Deferred to v2.0 unless performance pain emerges. |
| #19 | Lightweight meeting polls (Doodle-style) | M | No polls in cal.diy. |
| #25 | Postgres-backed rate limiter | M | cal.diy uses Unkey SaaS with no-op fallback — worse than ours. Keep plan. |
| #34 | Office hours / drop-in event type | S-M | Extend `SchedulingType` enum. |
| #37 | 1:1 mutual overlay | L | Guest OAuth + busy intersection — not cal.diy's "add guests" CC'ing feature. |
| #38 | Group availability solver | L | Collective event type ≠ group solver. |
| #42 | Routing forms | L | **Ripped out of cal.diy** in migration `20260305043434`. Build fresh if needed. |
| #45 | Standing groups with fairness solver | L | Genuinely novel, no competitor ships this. |
| #46/#47 | Native SwiftUI + App Clip | XL / M | No native code in cal.diy. |
| #51 | Migration guides (Calendly → OpalCal) | M | No importer in cal.diy. |
| #52 | AI scheduling (BYOAI / LiteLLM) | XL | AI agents dropped from cal.diy. |
| #55 | Booking page templates | S | No template system. |
| #60 | Deposit-hold (Stripe manual capture) | M | `capture_method: "manual"` not used. |
| #62 | Host calendar heatmap | M | No host-side analytics UI. |
| #68/#71 | Privacy policy + DPA drafts | (human) | Cal.diy's `WEBSITE_PRIVACY_POLICY_URL` points to external cal.com — full drafts still needed. |
| #76 | Host data export (GDPR Art. 20) | S | No export path exists. |
| #78 | Telemetry opt-in | S | cal.diy ships Jitsu ON by default — opposite model. |
| #82 | Canonical public-auth origin resolver | M | cal.diy relies on static `WEBAPP_URL` env, no `trustHost` / `x-forwarded-host`. |

## New work created by the adoption

Items that don't exist in the pre-pivot roadmap but need to ship:

| Item | Effort | Notes |
|---|---|---|
| **Rebrand sweep** | M (~1 week) | sed-replace `Cal.diy` / `Cal.com` → `OpalCal` across ~400 files, `@calcom/*` → `@opalcal/*` npm scope |
| **Deployment slimming** | M (3-5 days) | Delete `apps/api/v2` (NestJS REST API), `apps/docs` (Mintlify), ~130 of 151 app-store integrations we don't need (keep caldavcalendar, applecalendar, googlecalendar, office365calendar, stripepayment, dailyvideo, a few more). Drops ~1.5–2 GB. |
| **Fork deploy pipeline** | S | Fork the docker-hub image under AdeAnima, or build and publish our own to GHCR. Set up Railway/Hetzner/Fly deploy docs. |
| **Upstream rebase workflow** | S | Document monthly `git fetch upstream && git rebase upstream/main` cadence. Set up a CI job that flags merge conflicts. |
| **Upstream PR watch list** | XS | Subscribe to `calcom/cal.diy` issues with `caldav`, `self-host`, `docker`, `i18n` labels. Set reminders to cherry-pick stalled PRs. |

## Upstream PRs to track (port if stalled >4 weeks)

Watch these open cal.diy PRs — port into our fork if upstream doesn't merge them:

- **#28725** — CalDAV allow same base URL with different credentials and dedupe by full identity (CalDAV correctness, aligns with our positioning)
- **#28796** — CalDAV `toJSDate` instead of `toString` fix for VTIMEZONE double-conversion (Apple/Fastmail/Posteo bug)
- **#28445** — CalDAV default calendar handling
- **#27911** — CalDAV RRULE fix
- **#26819** — CalDAV Roundcube compatibility
- **#26077** — Booking-page i18n namespace slimming (helps deployment size)
- **#26132** — TypeScript project references to reduce build time
- **#24622** — Improved app declarative handler (cleaner provider registration)
- **#27057** — EU region support (open 3+ months, likely a port candidate)
- **#26668** — Runtime privacy-URL replacement (EU alignment)
- **#26665** — Multi-arch Docker (deployment slimming)

## Things to contribute upstream to `calcom/cal.diy`

Generic improvements aligned with maintainer priorities (high-merge-probability):

1. **CalDAV sync-token / ctag / incremental sync** — fills a known gap, no competing PRs, 6+ open CalDAV PRs signal demand
2. **CalDAV bugfixes** — timezone, RRULE, Fastmail duplicate-invite patterns (match what merges: small, self-contained, correctness-focused)
3. **Docker slimming + env config improvements** — demand exists (open PRs), no Cal-team priority
4. **Apple-provider NextAuth config helper** — if we build it cleanly, the "add Apple Sign-In" shell might land upstream even if the OpalCal-branded iCloud-wizard UX stays downstream

## Things to keep exclusively in the fork

Opinionated / branded / un-mergeable-upstream:

1. **Apple Sign-In → chained iCloud app-password guided wizard UX** — zero traction history, touches auth layer Cal team guards tightly
2. **EU-privacy / GDPR defaults** — EU region pinning, subprocessor-list page, DPA template, privacy-policy-per-booking-page, German-first copy for EU customers
3. **OpalCal branding** — wordmark, favicon, OG image, email templates, domain references, i18n string overrides
4. **Opinionated booking-flow UX** — post-booking redirect URL (already shipped in pre-pivot OpalCal), buffer label rename (prep/recovery time), analytics dashboard with all statuses visible
5. **Minimal-deploy configuration** — slimmed docker-compose, ripped app-store integrations, Hetzner/Fly deployment guides

---

## Phased roadmap

### Phase 0 — Fork bootstrap (1–2 weeks)
**Goal:** `AdeAnima/opalcal-next` running locally + deployed to a staging instance with OpalCal branding applied. No new features yet.

- [ ] Verify full clone + upstream remote wiring
- [ ] Run deployment-slimming pass: drop `apps/api/v2`, `apps/docs`, unused app-store integrations
- [ ] Rebrand sweep: `Cal.diy` → `OpalCal`, `@calcom/*` → `@opalcal/*`, wordmark, favicon, OG, email templates
- [ ] Publish slimmed Docker image to GHCR
- [ ] Deploy staging to Hetzner (or Fly.io Frankfurt) with custom domain
- [ ] Document `git fetch upstream && git rebase` workflow
- [ ] Set up CI: run cal.diy test suite + linter in our fork

### Phase 1 — OpalCal Alpha (8–10 weeks)
**Goal:** shareable alpha at a public URL, Apple-first onboarding, EU-privacy-ready.

- [ ] #9 **Apple Sign-In** + chained iCloud app-password setup wizard — core wedge feature
- [ ] Apple Developer Program enrollment + Services ID + .p8 key (human-task)
- [ ] EU privacy doc drafts: #68 privacy policy, #71 DPA template
- [ ] Subprocessor-list page (new page, required for GDPR Art. 28)
- [ ] Privacy-policy-per-booking-page override (already in pre-pivot OpalCal — port to new fork)
- [ ] Post-booking redirect URL (already in pre-pivot — port)
- [ ] #75 onboarding-dashboard checklist + empty-state guidance (on top of cal.diy's 5-step wizard)
- [ ] Alpha invite-code gate — extend `ensureSignupIsEnabled` with `AlphaInvite` check
- [ ] #76 host data export (GDPR Art. 20) — new export pipeline
- [ ] Port CalDAV PRs #28725, #28796 if upstream stalls
- [ ] Telemetry-opt-in rewrite — replace Jitsu-on-by-default with daily-batched anonymous ping (opt-in)
- [ ] Domain registration (opalcal.com / .app / .eu?) — human-task
- [ ] Trademark search — human-task

### Phase 2 — v1.0 (2–3 months)
**Goal:** production-ready self-host for EU teams.

- [ ] OpalCal-specific German copy + German-first booking-page defaults
- [ ] Minimal deployment guides: Hetzner, Fly.io Frankfurt, self-hosted Docker
- [ ] #49 audit trail retention + policy controls (on top of cal.diy's booking-audit package)
- [ ] Buffer label rename to prep/recovery time (UX polish — already in pre-pivot)
- [ ] #82 canonical public-auth origin resolver (wrap `WEBAPP_URL` with request-aware resolver for reverse-proxy scenarios)
- [ ] Contribute upstream: CalDAV correctness fixes, Docker slimming improvements
- [ ] Operational dashboards + runbooks (#29, #30 — new, on top of cal.diy logging)
- [ ] Stripe Connect single-key wrapper for solo self-hosters (#24 dual-mode)

### Phase 3 — v2.0 (3–4 months)
**Goal:** differentiated features that cal.diy doesn't have.

- [ ] #15 CalDAV poll reconciler + ctag / sync-token (contribute upstream first; fallback to fork if rejected — likely rejected)
- [ ] #37 Phase A: 1:1 mutual overlay via invitation
- [ ] #19 Lightweight meeting polls (Doodle-style)
- [ ] #34 Office hours / drop-in event type
- [ ] #55 Booking page templates
- [ ] #62 Host calendar heatmap

### Phase 4 — v3.0 (6+ months)
**Goal:** power features, native app.

- [ ] #38 Group availability solver
- [ ] #45 Standing groups fairness solver (genuinely novel)
- [ ] #46 Native SwiftUI app (EventKit, Widgets, Live Activities, Watch)
- [ ] #47 App Clip (instant booking via QR/NFC)
- [ ] #52 AI scheduling (BYOAI via LiteLLM)

---

## Risks + mitigations

| Risk | Mitigation |
|---|---|
| Upstream rebase friction grows as fork diverges | Keep structural changes minimal (no schema rewrites). Additive features in new files. Monthly rebase cadence. |
| Cal.com changes license or kills cal.diy | MIT is irrevocable for already-published code. We'd freeze at the pre-change commit and hard-fork. Monitor monthly. |
| Apple Sign-In implementation rejected upstream | Expected — keep OpalCal-branded UX in fork, upstream only the generic provider config if useful |
| Deployment slimming breaks with upstream changes | Bound the rip to app-store integrations + `apps/docs` / `apps/api/v2`. Tree-shake rather than delete where possible. |
| "CalDAV works but isn't first-class" becomes a performance problem at scale | Revisit in Phase 3. cal.diy's 1–10 user performance is fine; throttling from iCloud appears at higher volumes. |
| Community perception: "just a cal.diy re-skin" | Lean hard on Apple-first UX + EU-privacy positioning. Public roadmap showing CalDAV contributions upstream signals genuine investment. |

## Data migration from pre-pivot OpalCal

The current OpalCal (single Next.js app) has no production users. Migration = none needed. Archive the repo, preserve the audit/design docs, start fresh on the fork.

## Success criteria for this pivot

- Phase 0 in 2 weeks: slimmed + rebranded + deployed staging
- Phase 1 in 10 weeks: shareable alpha with Apple Sign-In working
- Phase 1 monthly rebase from `calcom/cal.diy` completes in <1 day
- <5 rejected upstream PRs in Phase 1 (high merge rate signals we're contributing well)
- Docker image <2.5 GB (vs current 6.9 GB source tree)
