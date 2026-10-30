# Remi

The full build spec is **[docs/SPEC.md](docs/SPEC.md)**. It is the source of truth for product, behavior, UI, architecture, data, seed content, and demo. Read the relevant sections before starting any task. If the spec and this file disagree, ask.

Remi is built from Heirloom's code (SPEC §1). Copy Heirloom's `apps/web` and `services/` in first, then change existing code in place before writing anything new. Where the spec says *(check against Heirloom)*, read Heirloom's code to confirm, and ask if it doesn't match.

## Rules

1. **Build in milestone order (SPEC §10).** Finish and demo each milestone before starting the next. Don't build ahead.
2. **Protect milestones 2, 4, 5, and 6.** They are the demo (ask in the side panel, guide on one page, guide across tools, ask the owner). If time is short, cut or simplify other milestones first, never these.
3. **Behavior rules go in backend code, not just the UI (SPEC §9).** Heirloom's rules plus Remi's (signed-off playbooks only, cited reasons, no typed values, allowed domains only, Remi never acts, temporary screenshots, aggregate team view) are enforced in `services/`. The extension and Studio only display what the backend decided.
4. **Verify citations in code.** `services/api/citations.py` checks that every quote exists in its source, for answers and for playbook steps. Drop or flag any that don't. Never trust a model-provided quote unchecked.
5. **The extension never clicks, types, or submits for the user.** It points, explains, and waits. Never call `preventDefault` on the page's events.
6. **Render all in-page UI inside a closed shadow root** (SPEC §4), with `pointer-events: none` on the overlay layer and `pointer-events: auto` only on the step card.
7. **Use the design tokens and primitives in `packages/design`** for Studio, the side panel, and the overlay: Hanken Grotesk + Newsreader, navy-950 rail, pill tone pairs, team colors. No hex values in components; reuse Pill, Avatar, Card, Segmented, Button, NavItem. Sentence case, 44px touch targets, 2px red focus outline, respect reduced motion.
8. **All seed data is fictional.** Use only the people, teams, tools, and tickets in SPEC §11. Demo against the mock apps in `seed/apps/`, never real accounts.
