# Remi — Build spec

**Remi is an AI keeper for a team's operations.** It learns how a team works across its tools, guides new hires through each task with an on-screen cursor in the browser, and stays on after onboarding for the rare, risky and undocumented tasks.

Remi is Heirloom repurposed. Heirloom's agents, knowledge store, citation checks and learning loop become Remi's brain. What's new is a browser extension that records workflows and draws the guide cursor, plus a sleeker interface built around it.

> **Source of truth.** This file is the spec for the Remi repo. References like "Heirloom SPEC §6" point to Heirloom's `docs/SPEC.md`, which this spec reuses. Where the two disagree for Remi, this file wins. Anything marked *(check against Heirloom)* is an assumption: read Heirloom's code and confirm before building on it, and ask if it doesn't match.

## 1. What carries over from Heirloom

Copy Heirloom's code into the new repo, then change it in place. Don't rewrite what already works: the backend rules and citation checks are the hardest parts to get right, and they carry over almost unchanged.

| Heirloom part | In Remi | What changes |
| --- | --- | --- |
| Single-agent answers (Heirloom M2) | Adapt | Answers come from playbooks and return guide steps, not just text |
| Agent-to-agent (Heirloom M3) | Keep | Becomes "Ask the owner": one team's agent asks another's on the employee's behalf |
| Learning loop (Heirloom M4) | Keep | An owner's answer is saved into the playbook it was about |
| Insights (Heirloom M7) | Adapt | Becomes the team view: ramp time, most-asked questions, one-person workflows |
| Behavior rules (Heirloom SPEC §6) | Keep | Same rules, plus new ones for recordings and guide steps (see §9) |
| `services/api/citations.py` | Keep | Also checks that every playbook step quotes a real recording or source |
| Design tokens and `components/ui/` | Keep | Same fonts, colors and primitives; new layouts on top |
| Seed data (Heirloom SPEC §14) | Replace | A fictional support team instead of Heirloom's teams (see §11) |
| Web app home | Adapt | Becomes Remi Studio, the admin console; employees mostly use the extension |
| Browser extension | New | Records workflows and draws the guide cursor |

## 2. MVP scope

The MVP proves one claim: Remi can guide someone through a process that crosses tools, using knowledge the team never wrote down.

**In the MVP**

- **Record.** An expert records one run of a workflow in Chrome.
- **Playbook.** Remi turns the recording, plus related tickets and Slack threads, into a playbook: steps, owner, the reason for each step, and edge cases. Every step cites its source.
- **Sign-off.** The workflow's owner approves the playbook before anyone is guided through it.
- **Ask.** An employee asks a question in the side panel in plain words.
- **Guide.** The cursor walks them through the playbook across tabs and stops at approvals.
- **Precedent.** "How did we handle this last time?" answered from past tickets, with citations.
- **Ask the owner.** Questions Remi can't answer go to the workflow's owner, and the answer is saved.
- **Team view.** Ramp time, most-asked questions, and one-person workflows.

**Not in the MVP (don't build):** practice runs, pre-submit checks, automatic drift alerts, role paths, desktop apps outside the browser, and browsers other than Chrome.

## 3. Interface

Remi has three surfaces. Employees live in the first two and rarely open the third.

| Surface | Who uses it | What's on it |
| --- | --- | --- |
| Side panel (Chrome's built-in side panel) | Every employee | An ask box, the current playbook as a step list with the active step highlighted, source chips under each answer, and an "Ask the owner" button |
| Guide overlay (drawn on the web page) | Every employee | The Remi cursor, a soft ring around the target element, and a small step card: step number, the action, and one line on why |
| Remi Studio (the web app) | Owners and managers | Recordings, a sign-off queue for new and changed playbooks, the playbook library, and the team view |

**Visual direction.** Sleek here means quiet: lots of white space, one accent, no decoration. The cursor is the brand, so it gets the strongest color and everything else steps back.

- Reuse Heirloom's tokens and primitives as they are: Hanken Grotesk for interface text, Newsreader for Studio headings, the navy-950 rail in Studio, pill tone pairs for status (draft, signed off, needs review), and team colors on avatars.
- The side panel is narrow, so it uses one column, no rail, and the same Card, Pill and Button primitives.
- The step card is the smallest thing Remi draws: one action line, one reason line, Next and Back buttons.
- Keep Heirloom's rules: sentence case, 44px touch targets, a 2px red focus outline, no hex values in components.
- With reduced motion on, the cursor appears at its target instead of flying there.

## 4. The guide cursor

**Reference: how Clicky does it.** Clicky (github.com/farzaa/clicky, MIT) is a macOS menu bar app with a full-screen transparent overlay window. It takes a screenshot, and Claude replies with `[POINT:x,y:label:screenN]` tags. The overlay parses them, maps them to the right monitor, and animates its own blue cursor along a bezier arc. It never moves the real system cursor and never steals focus.

**Why Remi does it differently.** Pointing from screenshots works anywhere but guesses pixels. Remi lives in the browser, so it reads the page's structure and points at the exact element. That makes it precise and lets it know when the person clicked the right thing. Screenshots are the fallback, not the main path.

**The guide loop**

```
Read the next step -> Find the element -> Fly cursor to it -> Wait for the click
        ^                    |                                       |
        |              tries in order:                               |
        |              1. recorded selector                          |
        |              2. role + accessible name                     |
        |              3. screenshot to Claude                       |
        +---------------- clicked: go to the next step --------------+
```

If all three ways of finding the element fail, the step card says so and offers "Ask the owner".

**How each piece works**

1. **The overlay.** A content script adds one host element to the page and renders the cursor, ring and step card inside a **closed Shadow DOM**, so page styles and extension styles can't collide. The layer is `position: fixed` over the viewport with `pointer-events: none`, so clicks pass through to the real page. Only the step card's buttons take clicks (`pointer-events: auto` on the card).
2. **Finding the element** (`finder.ts`). Each recorded step stores a locator set. Try in order:
   1. the CSS selector, accepted only if it matches exactly one visible element;
   2. role + accessible name, re-resolved against the current page (exactly one match = accept; several = use the recorded position to pick; none = next tier);
   3. the screenshot fallback: capture the visible tab (`chrome.tabs.captureVisibleTab`), send it with the step's instruction to the API, which asks Claude for a `[POINT:x,y]` tag. Check the result with `document.elementFromPoint(x, y)` and accept only an interactive element.
3. **Flying the cursor.** Read the target's position with `getBoundingClientRect()`, scroll it into view if needed, then animate along a quadratic bezier with `requestAnimationFrame` over about 400–600ms. Draw the ring around the element's rectangle.
4. **Staying attached.** Pages scroll, resize and re-render. Recompute the position on `scroll` and `resize`, watch the target with a `ResizeObserver` and the page with a `MutationObserver`. If the element disappears, run the finder again.
5. **Knowing the step is done.** Listen for clicks in the capture phase (`addEventListener('click', fn, true)`). When the click lands inside the target, or the URL changes to the step's expected page, move to the next step. Never call `preventDefault` on the page's events.
6. **Crossing tabs.** The extension's service worker holds the active guide session. On `chrome.tabs.onActivated` and `chrome.tabs.onUpdated` (status complete), it sends the current step to that tab's content script. The side panel stays open across tabs, so the step list never disappears.

**Limits to plan around.** A content script in the top page can't reach inside iframes from other domains, so declare the content script with `all_frames: true` and let each frame's script draw its own targets. Extensions can't run on `chrome://` pages. Apps drawn entirely on a canvas have no elements to find, so only the screenshot fallback works there.

**Starter code for the flight:**

```ts
// apps/extension/src/content/overlay.ts — inside the closed shadow root
export function flyTo(cursor: HTMLElement, target: Element, ms = 500) {
  target.scrollIntoView({ block: "center", behavior: "smooth" });
  const r = target.getBoundingClientRect();
  const end = { x: r.left + r.width / 2, y: r.top + r.height / 2 };
  const start = cursor.getBoundingClientRect();
  const from = { x: start.left, y: start.top };
  // control point above the straight line gives the arc
  const ctrl = { x: (from.x + end.x) / 2, y: Math.min(from.y, end.y) - 80 };
  if (matchMedia("(prefers-reduced-motion: reduce)").matches) ms = 0;
  const t0 = performance.now();
  const step = (now: number) => {
    const t = ms ? Math.min(1, (now - t0) / ms) : 1;
    const x = (1 - t) ** 2 * from.x + 2 * (1 - t) * t * ctrl.x + t ** 2 * end.x;
    const y = (1 - t) ** 2 * from.y + 2 * (1 - t) * t * ctrl.y + t ** 2 * end.y;
    cursor.style.transform = `translate(${x}px, ${y}px)`;
    if (t < 1) requestAnimationFrame(step);
  };
  requestAnimationFrame(step);
}
```

## 5. Recording workflows

An expert presses **Record** in the side panel, does the task once as normal, and presses **Stop**. Remi turns that run into a draft playbook.

**What the extension captures for each action**

- The action: click, field change, form submit, or page change.
- The page: its domain and path, with IDs and query strings masked.
- The target's locator set: a CSS selector, its role, its accessible name or visible label, and its position on screen.
- A small cropped screenshot around the target, for the playbook and for the screenshot fallback.
- An optional typed note from the expert, such as "we always check the order date first because…". These notes are where the *why* comes from.

**What it never captures.** What the expert types into a field is never stored. Remi records which field was filled ("typed into Refund amount"), not the value. Password fields are skipped entirely. Recording only runs on domains an admin has allowed in Studio.

**From recording to playbook.** The backend groups the actions into steps, then a Heirloom agent writes each step's instruction and reason using the expert's notes plus related tickets and Slack threads. Every reason must cite a source: a note, a ticket or a thread. `citations.py` checks each quote exists before the draft reaches the owner for sign-off.

## 6. Architecture

One backend serves two front ends. The backend is Heirloom's `services/` with new endpoints for recordings, playbooks and guide sessions. The extension and Studio never call Claude directly; every model call goes through the API so the behavior rules and logging apply.

```
+-------------------------+      +--------------------------+      +----------------+
| Chrome extension (new)  | <--> | Remi API (from Heirloom) | <--> | Remi Studio    |
|  - Side panel           |      |  - Agents                |      | (apps/web)     |
|  - Page overlay         |      |  - Rules and citations   |      +----------------+
|  - Service worker       |      |  - Store                 | ---> | Claude API     |
+-------------------------+      +--------------------------+      +----------------+
```

- **Extension to API.** The side panel sends questions and recordings. The API returns answers with citations and, when a playbook fits, the steps with their locator sets. The service worker holds the active session and hands each step to the overlay.
- **Studio to API.** Owners review drafts, sign off, and edit steps. Managers read the team view.
- **API to Claude.** Text calls for answers and playbook drafts, and vision calls for the screenshot fallback.
- **Store.** Keep Heirloom's existing database and retrieval setup *(check against Heirloom)*, and add the tables in §8.

## 7. Repo layout

Start the empty repo by copying in Heirloom's `apps/web` and `services/`, then add the extension. Move the design tokens and UI primitives into a shared package so Studio and the extension look the same.

```
remi/
  CLAUDE.md                 # rules for coding agents
  docs/SPEC.md              # this file
  apps/
    web/                    # Remi Studio, from heirloom/apps/web
    extension/              # new: Chrome extension (Manifest V3)
      manifest.json         # sidePanel, scripting, tabs, storage; all_frames content script
      src/background/       # service worker: guide session, tab handoff, recording buffer
      src/content/
        overlay.tsx         # cursor, ring, step card in a closed shadow root
        finder.ts           # selector -> role + name -> screenshot fallback
        recorder.ts         # captures actions and locator sets
      src/sidepanel/        # React side panel: ask, steps, sources
  packages/
    design/                 # tailwind tokens + components/ui (Pill, Avatar, Card,
                            # Segmented, Button, NavItem), moved from apps/web
  services/
    api/                    # from Heirloom
      citations.py          # kept; now also checks playbook steps
      recordings.py         # new
      playbooks.py          # new: draft, sign-off, versions
      guide.py              # new: match a question to a playbook, return steps
  seed/
    apps/                   # three mock tools for the demo (see §11)
                            # + fictional team, tickets, threads, recordings
```

For the extension, use a framework that handles content-script UI. Plasmo renders React components into a Shadow DOM for you, which saves the setup in the overlay step. A plain Vite build with a hand-made shadow root is fine too.

## 8. Data model

Keep Heirloom's people, teams, sources and logs *(check against Heirloom)*. Add these tables:

| Table | Key fields | Notes |
| --- | --- | --- |
| `workflow` | id, team_id, name, owner_id, frequency | One per task the team does, like "Refund a damaged item" |
| `playbook` | id, workflow_id, version, status, signed_off_by, signed_off_at | Status is `draft`, `signed_off`, or `needs_review`; only signed-off versions are used to guide |
| `step` | id, playbook_id, order, instruction, reason, domain, path_pattern, locator_set, is_approval | `is_approval` makes the guide stop and name who signs off |
| `locator_set` | selector, role, name, bbox, crop_image_id | Stored on each step, tried in that order by the finder |
| `recording` | id, workflow_id, recorded_by, started_at, actions | Raw actions from one expert run; never contains typed values |
| `citation` | id, step_id or answer_id, source_id, quote, verified | `verified` is set by `citations.py`; unverified quotes are dropped |
| `guide_session` | id, person_id, playbook_id, current_step, started_at, finished_at | Feeds ramp time in the team view |
| `owner_question` | id, asked_by, workflow_id, question, answer, answered_by, saved_to_step_id | The learning loop: an answer becomes part of a step |

## 9. Behavior rules enforced in the backend

As in Heirloom, these live in `services/`, not the UI. The extension and Studio only show what the backend decided.

**Kept from Heirloom (SPEC §6):** hop limits and loop prevention, share-level filtering and withheld lists, certainty labels, expiry, expert caps, dedupe, timeouts and escalation, logging of every agent call, and personal-data rejection.

**New for Remi:**

1. **Only signed-off playbooks guide.** A draft can't drive the cursor. A question that only matches a draft gets an answer labeled as unreviewed, with no guide.
2. **Every reason is cited.** A step's reason needs at least one verified citation. If the quote can't be found in its source, the reason is dropped and the step is flagged for the owner.
3. **No typed values, ever.** The API rejects any recording that contains field values or password fields, even if the extension failed to strip them.
4. **Allowed domains only.** Recordings and guide sessions from domains not on the team's allowlist are rejected.
5. **Remi never acts.** The API returns instructions and locators, never actions for the extension to perform. The person always clicks.
6. **Screenshots are temporary.** Screenshots sent for the vision fallback are used for that one call and not stored.
7. **Team view is aggregate.** No endpoint returns one person's questions, mistakes or sessions to anyone but that person.

## 10. Milestones

Build in this order, and demo each before starting the next. **Milestones 2, 4, 5 and 6 are the demo.** If time is short, cut or simplify the others, never these. Seed playbooks start signed off, so the guide works before the sign-off queue exists.

| # | Milestone | Done when | Protected |
| --- | --- | --- | --- |
| 1 | Repo and rebrand | Studio runs from the new repo with Remi's name, the shared design package, and the new seed team | |
| 2 | Ask in the side panel | A question in the side panel returns a cited answer from Heirloom's agents | Yes |
| 3 | Record to draft | An expert records a run and gets a draft playbook with verified citations | |
| 4 | Guide on one page | The cursor flies to each step's element on one page and advances on the right click | Yes |
| 5 | Guide across tools | One guide session continues across three tabs and stops at an approval step | Yes |
| 6 | Ask the owner | An unanswered question reaches the owner, and the answer is saved into the playbook | Yes |
| 7 | Sign-off queue | Owners approve, edit or reject drafts in Studio | |
| 8 | Team view | Ramp time, most-asked questions and one-person workflows, all aggregate | |
| 9 | Screenshot fallback | The guide still points correctly after a step's selector breaks | |

## 11. Seed data and demo script

All seed data is fictional: no real companies, people or customer records.

**Mock tools, not real ones.** A live demo can't depend on real help desk, store or payment accounts. Build three small mock web apps in `seed/apps/`, each a few pages with realistic forms: a help desk, a store admin, and a payments dashboard. Serve them on separate local ports so they act as three different tools and the cross-tool guide is real. Give their buttons and fields real labels and roles so the finder's role + name tier works.

**The seed team.** A fictional online furniture store's support team:

- **Dana**, support lead. Owns refunds, exchanges and chargebacks.
- **Sam**, new hire on day two.
- **Marcus**, finance. Owns approvals for refunds over $200.
- About 40 fictional tickets and 20 Slack-style threads, including three past partial-refund cases.
- Two signed-off playbooks (damaged-item refund, order lookup) and one draft (chargeback dispute).

**Demo script, about three minutes**

1. **Record.** Show Dana's recorded refund run becoming a draft playbook, with each reason cited to her notes or a ticket.
2. **Guide.** As Sam, ask "how do I refund a damaged table?" The cursor walks through the help desk, the store admin and the payments dashboard, then stops: refunds over $200 need Marcus.
3. **Precedent.** Sam asks about refunding one chair from a set. Remi shows the three past cases with citations.
4. **Ask the owner.** Sam sends the question to Dana. Dana answers in Studio, and the answer appears in the playbook for the next person.
5. **Team view.** Show ramp time, the most-asked questions, and chargebacks flagged as a one-person workflow.

## 12. Open questions (ask before deciding)

- Confirm every *(check against Heirloom)* item: the database and retrieval setup, the existing tables, and the Heirloom SPEC section numbers cited here.
- Expert notes: typed only for the MVP (default), or voice with speech-to-text?
- Extension framework: Plasmo (default) or plain Vite with a hand-made shadow root?
