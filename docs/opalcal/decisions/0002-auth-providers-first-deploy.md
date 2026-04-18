# 0002 — Auth providers for first deploy

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Martin
**Related:** 0001 — Stay on NextAuth v4

## Context

For the first Opal team deployment we need a sign-in story that covers the team's mix of Google Workspace and non-Google addresses, with no engineering work beyond env-var configuration. Verified provider wiring in `packages/features/auth/lib/next-auth-options.ts`:

- Credentials (email/password) — always pushed (line 303)
- Email magic link — always pushed (line 359)
- Google OAuth — pushed only when `IS_GOOGLE_LOGIN_ENABLED` is true (line 317)
- Azure AD / Outlook — pushed when `OUTLOOK_LOGIN_ENABLED` + client ID + secret are set (line 334)
- SAML — removed in this fork (only legacy JWT-callback shims remain)

## Decision

Enable **email/password + magic link + Google OAuth** for the first deploy. Defer Microsoft/Outlook until a teammate actually needs it.

## Required env vars

| Provider | Env vars |
|---|---|
| Email/password | _(none — always on)_ |
| Magic link | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PWD`, `EMAIL_FROM` (or `SMTP_FROM_EMAIL`) |
| Google OAuth | `GOOGLE_LOGIN_ENABLED=true`, `GOOGLE_API_CREDENTIALS='{"web":{"client_id":"...","client_secret":"..."}}'` |

Plus core auth env: `NEXTAUTH_SECRET` (`openssl rand -base64 32`), `NEXTAUTH_URL=https://<domain>`, `CALENDSO_ENCRYPTION_KEY` (`openssl rand -base64 24`).

## Consequences

- **Zero code changes** required — only env config
- Magic link doubles as fallback when Google is unavailable / for non-Google addresses
- Email/password remains as a final fallback for service / admin accounts
- We will need an SMTP provider (Postmark, SES, Resend, etc.) for magic-link delivery — pick one as part of the deploy spec
- Google OAuth requires creating a Google Cloud project + OAuth consent screen + adding `<domain>/api/auth/callback/google` as a redirect URI
