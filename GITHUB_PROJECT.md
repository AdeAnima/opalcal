# GitHub Project Workflow — Opal

Playbook for agents interacting with Opal's GitHub Project via `gh` CLI. Read this when you need to query, create, or update roadmap items.

**Maintenance:** when you learn something useful (a new recipe, a constraint, a gotcha), update this doc. Keep recipes copy-paste-ready. Date-stamp new sections in the changelog at the bottom.

---

## Project Identity

| Field | Value |
|-------|-------|
| Project name | OpalCal Roadmap |
| Project number | `3` |
| Project ID (node) | `PVT_kwHOAYLQYc4BU7dQ` |
| Owner | `AdeAnima` (personal account) |
| Repo | `AdeAnima/opalcal` (this repo) |
| URL | https://github.com/users/AdeAnima/projects/3 |
| Visibility | Private |

**Hard rule:** never create GitHub projects, issues, labels, milestones, or repos in the `h&w` or `alphalist` orgs. Opal lives in `AdeAnima/opalcal` only.

### Legacy code reference

The pre-fork custom codebase still lives at `AdeAnima/opalcal-legacy` for historical reference (research notes, design docs, the CalDAV sync engine prototype). Its GitHub Project was deleted on 2026-04-18 — all items still relevant to the cal.diy fork have been re-created in this project (look for `migrated from opalcal-legacy#N` in the body). Code-only history remains accessible via the legacy repo.

---

## Milestones (phases)

| # | Name | Purpose |
|---|------|---------|
| 1 | Phase 0 — Fork Bootstrap | Cal.diy → Opal rebrand, deploy infrastructure, basic CI/CD |
| 2 | Phase 1 — Alpha (v0.3) | Shareable alpha — multi-tenant, OAuth sign-in, GDPR groundwork, onboarding |
| 3 | Phase 2 — v1.0 | Production-ready EU self-host. Microsoft Graph, payments, ops dashboards, security hardening |
| 4 | Phase 3 — v2.0 | Differentiated scheduling — mutual overlay, group solver, polls, AI agent, MCP/CLI |
| 5 | Phase 4 — v3.0 | Power features + native — standing-group fairness, SwiftUI app, App Clip, compliance |

**Querying milestones**

```bash
gh api repos/AdeAnima/opalcal/milestones --jq '.[] | "\(.number)\t\(.title)"'
```

---

## Labels

**Theme** (pick one per issue, describes the subsystem): `theme:auth`, `theme:payments`, `theme:calendar`, `theme:scheduling`, `theme:hygiene`, `theme:observability`, `theme:security`, `theme:ux`, `theme:ops`, `theme:privacy`

**Type** (pick one, describes the nature of work): `type:feature`, `type:correctness`, `type:infra`, `type:upstream-port`, `type:rebrand`. Plus GitHub defaults: `bug`, `enhancement`, `documentation`.

**Effort** (sizing guide, one per issue): `effort:XS` (<1 day), `effort:S` (1–3 days), `effort:M` (3–10 days), `effort:L` (2–4 weeks), `effort:XL` (1 month+).

**Special**: `idea` (backlog, no milestone), `human-task` (user does this, not code), `downstream-only` (don't upstream to cal.diy).

**Rule:** every issue gets at least one `theme:*` + either a `type:*` label or `idea` / `human-task`.

---

## Custom fields

| Field | Type | ID | Options (name → id) |
|-------|------|----|---------------------|
| Status | single-select | `PVTSSF_lAHOAYLQYc4BU7dQzhK8zcw` | Todo=`f75ad846`, In Progress=`47fc9ee4`, Done=`98236657` |

(Effort lives as a label, not a custom field, in project #3.)

Built-in fields also available: Title, Assignees, Labels, Milestone, Repository, Reviewers, Parent issue, Sub-issues progress, Linked pull requests.

**Query field IDs and options (refresh after adding fields):**

```bash
gh project field-list 3 --owner AdeAnima --format json | jq '.fields[]'
```

---

## Status automation

The project has the built-in workflow rules enabled — you almost never need to set Status by hand:

| Trigger | Auto-effect |
|---------|-------------|
| Issue added to project | Status → **Todo** |
| PR opened with `Closes #N` (or `Fixes #N` / `Resolves #N`) | Linked issue → **In Progress** |
| Issue closes (manually or via PR-merge keyword) | Status → **Done** |
| Status set to **Done** manually | Linked issue auto-closes |

**What you must still do:**
1. **Add new issues to the project at creation** — `gh project item-add 3 --owner AdeAnima --url <url>`. Without this, the rules don't fire.
2. **Include `Closes #N`** in every PR body for each issue the change resolves.
3. **Set effort** by adding an `effort:*` label at creation.

---

## Core recipes

### Query current state

```bash
# All open issues
gh issue list --repo AdeAnima/opalcal --state open --limit 100

# Current-phase work
gh issue list --repo AdeAnima/opalcal --milestone "Phase 1 — Alpha (v0.3)"

# Issues by theme
gh issue list --repo AdeAnima/opalcal --label theme:auth

# Ideas backlog
gh issue list --repo AdeAnima/opalcal --label idea

# Human-action tasks
gh issue list --repo AdeAnima/opalcal --label human-task

# Full project board
gh project item-list 3 --owner AdeAnima --format json --limit 200 \
  | jq -r '.items[] | [.content.number, .status, .content.title] | @tsv'
```

### Create a new issue (phase-bound)

```bash
URL=$(gh issue create --repo AdeAnima/opalcal \
  --title "Short descriptive title" \
  --label "theme:X,type:Y,effort:M" \
  --milestone "Phase 1 — Alpha (v0.3)" \
  --body-file - <<'EOF'
**Context**
Why this matters.

**Scope**
- concrete bullet
- another bullet

**Acceptance**
- verifiable outcome

**Blocked by / Blocks:** (optional)
EOF
)
gh project item-add 3 --owner AdeAnima --url "$URL"
```

### Create a backlog idea (no milestone)

```bash
URL=$(gh issue create --repo AdeAnima/opalcal \
  --title "Title" \
  --label "idea,theme:X" \
  --body "Short description + reason to consider + rough complexity.")
gh project item-add 3 --owner AdeAnima --url "$URL"
```

### Set Status manually (override only — prefer automation)

```bash
ITEM_ID=$(gh project item-list 3 --owner AdeAnima --format json --limit 200 \
  | jq -r '.items[] | select(.content.number == ISSUE_NUM) | .id')

gh project item-edit \
  --project-id PVT_kwHOAYLQYc4BU7dQ \
  --id "$ITEM_ID" \
  --field-id PVTSSF_lAHOAYLQYc4BU7dQzhK8zcw \
  --single-select-option-id 47fc9ee4    # In Progress
```

### Assign or change a milestone

```bash
gh issue edit NUM --repo AdeAnima/opalcal --milestone "Phase 2 — v1.0"
```

### Close an issue

```bash
gh issue close NUM --repo AdeAnima/opalcal --reason completed --comment "Done in PR #..."
# Or: --reason "not planned"
```

Closing auto-moves the project item to **Done**.

### Sub-issue relationships (parent / child)

```bash
PARENT=$(gh issue view 7 --repo AdeAnima/opalcal --json id --jq .id)
CHILD=$(gh issue view 8 --repo AdeAnima/opalcal --json id --jq .id)
gh api graphql -f query='
mutation($parent: ID!, $child: ID!) {
  addSubIssue(input: { issueId: $parent, subIssueId: $child }) {
    issue { number }
  }
}' -f parent="$PARENT" -f child="$CHILD"
```

---

## Issue hygiene rules

When creating or updating an issue:

1. **Always** add at least one `theme:*` label
2. **Always** add either a `type:*` label or `idea` / `human-task`
3. **Add to project** immediately after creation (`gh project item-add 3 --owner AdeAnima --url <url>`)
4. **Set Effort** by adding an `effort:*` label
5. **Milestone** if it belongs to a phase, otherwise leave bare and tag `idea` / `human-task`
6. **Body structure** — use Context / Scope / Acceptance / Dependencies
7. **Reference file paths with line numbers** when discussing specific code (`packages/features/auth/...:65`)
8. **Use `Closes #N` in PR/commit messages** to trigger auto-Done on merge
9. **For migrated items**, reference the source: `migrated from opalcal-legacy#37`

---

## When NOT to edit the project

- When executing a plan you're already running — stay in code, only update Status at start/finish
- When the user is mid-brainstorm — create issues only after decisions land
- Never duplicate an existing open issue — search first: `gh issue list --repo AdeAnima/opalcal --search "<keyword>"`

---

## Related Docs

- `CLAUDE.md` → Project conventions for AI agents
- `docs/opalcal/ROADMAP.md` → high-level future direction (UX vision, AI integration)
- `docs/opalcal/decisions/` → ADR-style records for deferred / chosen alternatives
- `docs/opalcal/specs/` → design specs per subsystem

---

## Changelog

- **2026-04-18** — Migrated from `opalcal-legacy` to this repo. New project #3 (`PVT_kwHOAYLQYc4BU7dQ`) replaces legacy #2 (deleted). Closed fork-bootstrap issues #2/#3/#4/#6 (rebrand, GHCR, CI, deploy — all done as of commits cb4848035e and 6624276bc4). Added 41 new issues: 6 future-vision (tiered UI, templates, onboarding, AI agent, MCP, CLI), 3 deferred decisions (bun, Better Auth, passkeys), 3 tech-debt (Docker cache, app-store trim, designed wordmark), and 29 features migrated from legacy spanning scheduling (overlay, polls, mutual/group/standing solvers), platform (SwiftUI, App Clip, web-component embed), payments (deposit holds, retry states, Stripe Connect), security (Turnstile, audit trails, source-available license), observability (dashboards, alerts, heatmap), and ops (managed event types, host-side change detection, billing placeholder, Opal GitHub org).
