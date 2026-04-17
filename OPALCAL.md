# OpalCal

**OpalCal is a tracking fork of [`calcom/cal.diy`](https://github.com/calcom/cal.diy)** — the MIT community edition of Cal.com. This file documents what OpalCal does differently and how we stay in sync with upstream.

## The wedge

OpalCal targets what cal.diy doesn't cover:
- **Apple/iCloud-first UX** — Sign in with Apple → guided iCloud CalDAV setup (not "paste an app-password")
- **EU-privacy positioning** — GDPR-first defaults, subprocessor transparency page, DPA template, German-first copy, EU-region deployment guides
- **Opinionated self-host UX** — slimmed Docker image, post-booking redirects, privacy policy per booking page, plain `prep/recovery time` labels instead of "buffer time"

Everything else comes from upstream cal.diy. We avoid structural divergence.

## Relationship to upstream

- `origin` = `https://github.com/AdeAnima/opalcal.git` (our fork)
- `upstream` = `https://github.com/calcom/cal.diy.git` (MIT community edition)
- Monthly rebase cadence. Minimize structural changes (schema, core booking flow) to keep rebase cheap.

## Contribution flow

| Kind of change | Where it lands |
|---|---|
| CalDAV correctness fixes | Upstream (PR to `calcom/cal.diy`) |
| Docker/self-host slimming | Upstream |
| Small bug fixes | Upstream |
| Apple Sign-In auth provider (plumbing) | Try upstream first; fallback to fork |
| Apple Sign-In branded onboarding UX | Fork only |
| EU-privacy defaults / subprocessor page / DPA | Fork only |
| OpalCal branding (wordmark, copy, emails) | Fork only |
| Post-booking redirects, buffer rename, etc. | Fork only |

## Rebase workflow (monthly)

```bash
git fetch upstream
git rebase upstream/main
# resolve conflicts — keep OpalCal customizations
bun test             # or yarn test
bun run build        # or yarn build
git push --force-with-lease
```

If the rebase takes more than a day, something has diverged structurally — stop and reassess.

## Deployment

Do **not** build from source on PaaS free tiers — the dev checkout is ~6.9 GB. Use one of:

1. **Prebuilt Docker image** — fastest path to a running instance
2. **Hetzner Cloud** (€4/month Germany box, 40 GB disk) — fits the EU-privacy positioning
3. **Fly.io Frankfurt region** — generous free tier, multi-region ready

A slimmed `docker-compose` for OpalCal lives at `docs/opalcal/deploy/` (added in Phase 0).

## Project tracker

Live roadmap: [OpalCal Roadmap on GitHub Projects](https://github.com/users/AdeAnima/projects) — see the linked board from issues.

Phase overview + historical context: [`docs/opalcal/specs/2026-04-17-fork-adoption-design.md`](./docs/opalcal/specs/2026-04-17-fork-adoption-design.md)

Legacy repo (pre-fork custom Next.js app, archived): [`AdeAnima/opalcal-legacy`](https://github.com/AdeAnima/opalcal-legacy)

## Hard rules

- Never create GitHub resources in the `h&w` or `alphalist` orgs. OpalCal lives in `AdeAnima/opalcal` only.
- Never skip pre-commit hooks (`--no-verify`). Fix the underlying issue.
- Never force-push to `main`.
