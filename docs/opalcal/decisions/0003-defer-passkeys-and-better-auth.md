# 0003 — Defer passkeys and Better Auth migration

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Martin
**Related:** 0001 — Stay on NextAuth v4

## Context

Two related questions came up while standing up the first deploy:

1. Add **passkey / WebAuthn** sign-in?
2. Migrate from **NextAuth v4** to **Better Auth** (which has passkeys built in)?

Verified state of the codebase: cal.diy ships zero WebAuthn/passkey code (no `@simplewebauthn/*` deps, no `Authenticator` Prisma model, no callbacks). Adding passkeys means either (a) building it bespoke, (b) bolting on Better Auth dual-stack for that one feature, or (c) full Better Auth migration.

A sub-agent inventory of the NextAuth surface area:
- `packages/features/auth/lib/next-auth-options.ts` — **1,235 lines** of multi-tenant identity logic (auto-merge, profile switcher, `upId` semantics, impersonation, SAML legacy decode)
- `packages/features/auth/lib/next-auth-custom-adapter.ts` — 170 lines, custom Prisma adapter, `User`/`Account` only (JWT-only sessions, no `Session`/`VerificationToken` tables)
- 22 server-side files import `next-auth`
- 67 client-side files import `next-auth/react`
- 64 sites use `getServerSession`, 53 use `useSession`, 16 use `getToken`
- `apps/api/v2` (NestJS) decodes the NextAuth JWT directly
- 7+ Playwright e2e files hardcode the NextAuth session-cookie name
- `User.id` is `Int` everywhere — Better Auth defaults to `String`

## Decision

**Defer both.** No passkeys, no Better Auth migration in 2026.

For the team's first deploy, magic link + Google OAuth + email/password (as decided in `0002`) cover the use cases.

## Rationale — why not passkeys

Useful but not urgent. Magic link gives a passwordless UX that meets the bar for a 5-person team. Building bespoke WebAuthn into NextAuth v4 (no first-class support) is 3–5 days of work plus ongoing maintenance of a custom credentials provider. The juice doesn't justify the squeeze right now.

If passkeys become a real ask, the cleanest path is `@simplewebauthn/server` directly with cal.com's existing user model — not Better Auth. Better Auth's WebAuthn plugin assumes its own adapter shape, and our `User.id: Int` makes that adapter a custom-fork problem.

## Rationale — why not Better Auth migration (now)

Effort estimates from sub-agent inventory:

| Bucket | Scope | Days |
|---|---|---|
| Minimum-viable | Credentials + magic link + Google. Schema mapped to existing User/Account. Custom session callback ported. Cookie compat shim. tRPC ctx + middleware swapped (~120 sites). Playwright fixtures fixed. No Azure, no impersonation, no passkeys. | 15–20 |
| Feature parity | Above + Azure AD + impersonation rebuilt on BA admin plugin + API v2 NestJS strategy rewritten + identity-merge logic ported + e2e green | 25–35 |
| Full upgrade | Above + passkeys + edge middleware + replace bespoke org code with BA organization plugin (touches Membership/Profile/Team/billing — biggest risk) | 45–60+ |

Plus ~10 days if also migrating `User.id: Int` → `String` (BA's default).

Big specific risks: cookie name change → mass logout unless a parallel-cookie shim is written; the 1,235-line auth-options file has multi-tenant logic with no BA equivalent (auto-identity-merge, `upId`, profile switcher); cross-org impersonation is an [open issue in BA](https://github.com/better-auth/better-auth/issues/3056).

Non-trivial weeks of work for zero user-visible feature gain. NextAuth v4 still gets security patches; upstream cal.com is also still on v4.

## Trigger conditions to revisit

Re-open this decision when **any** of:

1. NextAuth v4 announces EOL or drops a CVE without a patch
2. Upstream cal.com migrates (eliminates merge-friction argument; copy their work)
3. Concrete product need: passkeys, edge-runtime auth on a hot path, or Redis-backed sessions
4. Better Auth ships cross-org impersonation **and** documents an `Int`-ID escape hatch (closes the two biggest specific risks)

## Action items

- [ ] None right now. Magic link + Google + password is the plan.
- [ ] If passkeys come back as a real ask: draft a separate spec using `@simplewebauthn/server` directly (skip Better Auth).

## Sources

- [Better Auth migration guide](https://better-auth.com/docs/guides/next-auth-migration-guide)
- [Better Auth Organization plugin](https://better-auth.com/docs/plugins/organization)
- [Better Auth issue #3056 — cross-org admin impersonation](https://github.com/better-auth/better-auth/issues/3056)
- [Auth.js → Better Auth handover, Discussion #13252](https://github.com/nextauthjs/next-auth/discussions/13252)
- [`@simplewebauthn/server`](https://simplewebauthn.dev/docs/packages/server)
- Local: `docs/opalcal/decisions/0001-stay-on-next-auth-v4.md`, `docs/opalcal/decisions/0002-auth-providers-first-deploy.md`
