# 0004 — Defer Bun migration

**Status:** Accepted
**Date:** 2026-04-18
**Deciders:** Martin

## Context

OpalCal is a Yarn 4 (berry) monorepo inherited from the cal.com fork. CI builds run ~20 minutes per push, and the question came up: would migrating from Yarn 4 to Bun meaningfully shorten the loop?

Current build time breakdown:

| Phase | Time | Bun helps? |
|---|---|---|
| `yarn install` | ~5 min | Yes (~1 min with bun) |
| Next.js + SWC + TypeScript compile | ~8 min | No (Next handles this, not the package manager) |
| Image build + push to GHCR | ~5 min | No (network + layer upload) |
| Other (lint, prisma, misc) | ~2 min | Marginal |
| **Total** | **~20 min** | **~16 min projected** |

So the realistic upside is ~4 minutes per build.

## Decision

**Defer the Bun migration.** Stay on Yarn 4. Pursue cheaper wins first.

## Rationale — cost vs. benefit

Migration cost: 2–4 days of focused work, with non-trivial breakage risk:

| Risk area | Detail |
|---|---|
| Yarn 4 patches | `.yarn/patches/` contains patches for `next-i18next`, `libphonenumber-js`, `dayjs`. Bun's patch format differs and each needs re-applying and re-verifying |
| Resolutions | ~30 pinned dependencies in `resolutions` block; bun's override syntax is different |
| Postinstall hooks | `husky install && turbo run post-install` chain needs to keep working |
| Dockerfile + CI | Every yarn invocation across Dockerfile and `.github/workflows/*` needs editing |
| Native deps | Bun has known compat issues with `@prisma/client`, `@googleapis/*`, and other native modules — possible runtime breakage |
| Upstream drift | Cal.com upstream stays on Yarn → permanent merge friction on lockfile, scripts, CI configs |

Trading 2–4 days of risky work plus ongoing merge tax for ~4 minutes per build is a bad ratio.

## Better alternatives, ranked

1. **GitHub Actions layer cache** — already configured in `.github/workflows/docker-build.yml`. The first run was the slow one; subsequent builds should hit warm cache for unchanged layers. Validate this is actually working before doing anything else.
2. **Trim `packages/app-store/`** — drop the ~145 unused integrations. Estimated ~30% smaller install + image, ~20% faster build. One-time cleanup, no merge friction (additions only come from upstream rarely).
3. **Turbo remote cache** (Vercel free tier) — caches Next.js build artifacts across runs. On a cache hit the compile phase drops from ~8 min to ~30 sec. Biggest single lever, low risk.

Any one of these beats the bun migration on ROI. All three together would likely push CI under 5 minutes — well past what bun alone can deliver.

## Trigger conditions to revisit

Re-open this decision when **any** of:

1. GHA layer cache + app-store trim + turbo remote cache are all in place and builds are still painful
2. We need bun's runtime perf for something specific (edge function, cold-start sensitive worker, etc.)
3. Upstream cal.com migrates to bun — eliminates the merge-friction argument; copy their work
4. A yarn-specific bug bites us (lockfile corruption, plugin breakage, perf regression)

## Action items

- [ ] Verify GHA Docker layer cache is actually being hit on subsequent builds (check cache hit/miss in workflow logs)
- [ ] Spec the app-store trim: enumerate which integrations OpalCal actually needs, draft a removal PR
- [ ] Evaluate Turbo remote cache on Vercel free tier — set up `TURBO_TOKEN` + `TURBO_TEAM` secrets, measure cache-hit build time
- [ ] No bun work until the above three are complete and measured

## Sources

- [Bun compatibility tracker](https://bun.sh/docs/runtime/nodejs-apis)
- [Turborepo remote caching](https://turborepo.com/docs/core-concepts/remote-caching)
- [Yarn 4 patch protocol](https://yarnpkg.com/cli/patch)
- Local: `.github/workflows/docker-build.yml`, `packages/app-store/`, `.yarn/patches/`
