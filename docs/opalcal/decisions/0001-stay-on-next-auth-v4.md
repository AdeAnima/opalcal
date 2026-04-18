# 0001 — Stay on NextAuth v4 (defer Auth.js v5 / Better Auth)

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Martin
**Supersedes:** —

## Context

The fork uses `next-auth@4.24.13` (verified in `apps/web/package.json` and `packages/features/auth/package.json`). The handler lives at `apps/web/pages/api/auth/[...nextauth].ts` (Pages Router style). Providers wired in `packages/features/auth/lib/next-auth-options.ts`: Credentials (always), Email magic link (always), Google (gated by `IS_GOOGLE_LOGIN_ENABLED`), Azure AD (gated by 3 env vars). Custom Prisma adapter (`CalComAdapter`) plus heavy `jwt` / `session` callbacks. SAML provider was removed in this fork; only legacy JWT-callback handling for SAML tokens remains.

We considered migrating to Auth.js v5 (`next-auth@5.x`) to gain edge-runtime auth in middleware, the unified `auth()` helper, env-var auto-detection, and stricter OIDC handling.

## Decision

**Stay on NextAuth v4. Patch-bump to latest `4.24.x`. Do not migrate to v5.**

When auth eventually becomes a real pain point, evaluate **Better Auth** as the migration target — not Auth.js v5.

## Rationale

1. **v5 is still officially beta after 2+ years**, with no GA date (see Discussion #13382 on `nextauthjs/next-auth`).
2. **September 2025: Auth.js was handed to the Better Auth team** and put into maintenance mode (Discussion #13252; HN thread #45389293). The ecosystem energy has moved to Better Auth; v5 is unlikely to receive significant new feature work.
3. **Migration cost is 2–4 engineering weeks** for this fork: handler refactor to App Router, custom Prisma adapter rewrite for the v5 `Adapter` interface, env-var renames (`NEXTAUTH_*` → `AUTH_*`), session-cookie rename (`next-auth.session-token` → `authjs.session-token`) which logs all users out unless a compat shim is written, hundreds of `getServerSession` → `auth()` swaps, split-config refactor (edge config without adapter, node config with adapter), full E2E across 4 providers + magic link.
4. **Marginal feature gain** for our use case. We don't use middleware-based auth heavily; OAuth-on-Vercel-previews isn't relevant; we're not blocked on env auto-detection.
5. **Upstream cal.com is also still on v4.24.x** — staying aligned reduces merge friction with upstream.
6. **v4 maintenance status is fine** through 2026 and likely 2027. No formal EOL announced. Security fixes still flow.

## Consequences

**Positive:**
- Zero migration work; engineering time spent elsewhere (deploy, rebrand, port real legacy gaps)
- Stays aligned with upstream cal.com → easier to pull merges
- Avoids landing on a beta library whose maintainers have publicly pivoted

**Negative / accepted trade-offs:**
- No edge-runtime auth in middleware (middleware uses JWT decode anyway, fine)
- No env-var auto-detection — we manage `NEXTAUTH_SECRET` / `NEXTAUTH_URL` explicitly
- We will eventually have to migrate; the cost grows the longer we wait. But the migration target is more likely to be Better Auth than v5, and Better Auth's migration tooling is improving.

## Action items

- [ ] Bump `next-auth` to latest `4.24.x` patch in a small PR (free, security only)
- [ ] Add "evaluate Better Auth" to the someday-list (not this quarter)
- [ ] Clean up dead SAML leftovers from removed feature: `@boxyhq/saml-jackson` dep in `apps/web/package.json`, `serverExternalPackages` entry in `apps/web/next.config.ts:233`, `SAML_DATABASE_URL` / `SAML_ADMINS` in `.env.example` (see Decision 0002 — branding & cleanup, when written)

## Sources

- [Auth.js v5 migration guide](https://authjs.dev/getting-started/migrating-to-v5)
- [Discussion #13252 — Auth.js is now part of Better Auth](https://github.com/nextauthjs/next-auth/discussions/13252)
- [HN: Auth.js is now part of Better Auth](https://news.ycombinator.com/item?id=45389293)
- [Discussion #13382 — How many more years of beta for v5?](https://github.com/nextauthjs/next-auth/discussions/13382)
- [Better Auth migration guide from Auth.js](https://better-auth.com/docs/guides/next-auth-migration-guide)
- [Migrating from NextAuth v4 to Auth.js v5 without logging out users](https://medium.com/@sajvanleeuwen/migrating-from-nextauth-v4-to-auth-js-v5-without-logging-out-users-c7ac6bbb0e51)
