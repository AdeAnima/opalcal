# Opal Roadmap

Future work, not yet scheduled. Items here are intentionally not specced — they capture intent so we don't lose it.

## UX Customization & Declutter

Inspired by OrcaSlicer's tiered settings (Basic / Advanced / Expert).

- **User-tier UI complexity levels**: let users pick how much surface area they see
  - Basic: just enough to take a booking
  - Advanced: workflows, routing, integrations
  - Expert: every knob the codebase exposes
- **Templates**: opinionated starting points for common setups (1:1 coach, sales team round-robin, office hours, interview panel, …) so users don't face a blank slate
- **Guided onboarding**: walk new users through first event type / availability / first booking. Current cal.diy onboarding drops users into a complex shell with no guidance
- **In-product AI agent**: chat surface inside the UI that can read current state and perform actions for the user (create event types, adjust availability, set up workflows, debug "why isn't this booking working")

## AI Agent Integration

- **MCP server**: expose Opal as an MCP server so external AI agents (Claude, etc.) can manage bookings, event types, availability, and contacts on a user's behalf
- **CLI**: scriptable interface for the same operations, useful for automations and for power users

## Other deferred items

See `docs/opalcal/decisions/` for already-decided deferrals:
- 0001 — stay on NextAuth v4 for now
- 0003 — passkeys & Better Auth migration deferred
- 0004 — bun migration deferred
