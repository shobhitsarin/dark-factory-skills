---
name: defect-fix
description: End-to-end FE defect fix for a triaged Jira defect. One fixed path - understand, trace, reproduce, map impact, fix, prove, gate, report. Covers the reported scenario, scenarios the ticket did not mention, scenarios the fix itself could break, and all three viewports (mobile, tablet, desktop). Use for every defect run.
---

# Defect Fix  (skill version: v8)

## Inputs (from your task prompt, never from this file)
`ticket_id`, `parent_ticket_id`, workspace root (contains `.agents/` and `src/`). The repo is `src/`.
Defect ticket: `src/.SS_WF/<ticket_id>_JIRA_OUTPUT.json`. Missing or unreadable → stop, verdict BLOCKED.
A ticket number inside the title (e.g. `ABC-123 ||`) is a label, not the parent.

## Viewports (project Tailwind theme)
| Viewport | Width | Tailwind | Grid |
| --- | --- | --- | --- |
| Mobile | 390 – 767px | base (no prefix) | 4 columns |
| Tablet | 768 – 1023px | `md:` | 8 columns |
| Desktop | ≥ 1024px | `lg:` | 12 columns, content max 1700px |

## Rules (override every other skill)
1. **The defect is real.** Outcomes: FIXED (gate passed) or BLOCKED (with evidence). Never "not a defect" / "already fixed".
2. **Context in layers.** Ticket → parent code-gen file map → source → analysis plan / figma-output only for a specific question. Search large docs with `grep -n`.
3. **Trace every hop** from user action to what the user sees, including where data enters the code. A root cause with "if/may/possibly" is unfinished.
4. **Mock only the outer boundary** (`fetch` / HTTP client / CMS props) with real payload shapes. Never mock a hook, service, mapper or helper on the traced path.
5. **A passing reproduction means the test is wrong.** Never delete a reproduction test; it becomes the regression test.
6. **Defect overrides:** coding skills give standards only. Source may change, tests must run, nothing on the traced path is mocked, nothing is regenerated. UI reproductions may assert classes, tokens, `dir`, variants and breakpoint prefixes.
7. **Dev notes** are the top instruction after the Expected Result. A defect dev note beats a parent-story dev note.
8. **Never weaken, skip or delete an existing test.** Change one only if it asserts the buggy behaviour, and report why.
9. **No hardcoding to fix.** Copy from Sitecore, values from constants, routes from links. A hardcoded fallback is not a fix.
10. **Never copy ticket credentials or personal data** into code, tests or reports. Use synthetic values.
11. **Responsive fixes use only base / `md:` / `lg:`.** Never `sm:`, `xl:`, `2xl:`, `max-*:`, arbitrary breakpoints or `window.innerWidth` for layout. Never create a separate per-viewport copy of a component.
12. **Stay at the workspace root.** Never `cd`. Use `git -C src`. Run tests only with the commands below.

---

## Stage 1 — Understand
Per issue record **TRIGGER**, **OBSERVED**, **EXPECTED**, **SIGNALS**, **dev notes** (verbatim), and the **viewport(s)** where it was observed (mobile / tablet / desktop / all / not stated).
- Expected missing/vague → use the AC the ticket cites, else the matching plan AC. None → BLOCKED: expected behaviour undefined. Never invent.
- Viewport not stated → treat as all three; Stage 4 proves each.
- Attachments only (screenshots) → record "attachment not reviewed"; no steps in text → BLOCKED: needs steps.
- Per contract (BFF, Sitecore, Figma): does the ticket bring **new data, a reference, or a described change in words**?
  - None → existing contract is correct; the fault is in the code.
  - Data / described change → code must satisfy the ticket's version; build fixtures from it.
  - Reference only → load `context-gathering`, fetch that item only. A new Figma link applies only to the viewport it shows.
- Ticket Expected conflicts with AC / plan / design → **ticket wins**; record the conflict.
- Expected needs different backend/CMS output (not different FE handling) → that part is BLOCKED: outside FE.
- Split issues only when root cause, layer or expected behaviour differ.

## Stage 2 — Locate and trace
1. `grep -n` the parent code-gen document for the file map. Read its **Defect Change Log** for the same files — earlier fixes and their regression tests must keep passing; do not repeat a rejected approach (also check ticket comments for earlier attempts).
2. Open each hop on the TRIGGER path. Note input shape, branch taken, state set, what later clears or overrides it, render condition.
3. Open more only when a hop needs it:

| You need | Read |
| --- | --- |
| Real API response/error shape | repo mock files; else plan (`grep -n` endpoint/code) |
| Sitecore field / message entry | plan Sitecore section |
| AC wording not quoted in code-gen doc | plan AC section |
| Layout per viewport (spans, order, visibility, adaptive pattern) | `src/figma-output/responsive_design_intent.json` (authoritative across viewports) |
| Design detail for one viewport (tokens, spacing, variant) | `src/figma-output/figma_design_{nodeId}_{viewport}-context.json` for that viewport |
| Shared util/hook behaviour | open it; never assume |

- No parent or no code-gen doc → search the repo for the ticket signals (screen, label, code).
- Code differs from the code-gen doc → the code is the truth; record the drift.
- Trace leads outside the parent story's files → follow it; record "fault outside story scope".
- A value marked `"source": "derived"` in the reconciliation (no design for that viewport) is still the expected behaviour unless the ticket says otherwise.

Exit: mechanism in 1–2 sentences, no "if/may".

## Stage 3 — Reproduce
Test at the level the user observes, next to the source (`<Source>.test.tsx` or a `describe('<ticket_id> …')` block in the existing file).
```bash
python3 ./.agents/skills/run-test-cases/scripts/run-tests.py --files <test-file> --label <ticket_id>-<ISSUE-n>-reproduction --expect fail
```
- `EXPECTED_FAIL`/`FAIL` for the OBSERVED reason → continue.
- `PASSED_UNEXPECTEDLY` → make it realistic (real payload, real CMS data, real sequence, fake timers, controlled promise resolution, repeated/concurrent actions); re-run.
- `FAILED_FOR_WRONG_REASON`/`NOT_COLLECTED` → fix setup, never the assertion.
- By defect type:
  - **Visual/RTL/responsive** → jsdom renders no layout, so assert what the design requires: the class/token, `dir`, variant, presence, and the breakpoint prefix (`md:…` / `lg:…`) for the affected viewport. Not expressible → BLOCKED: needs visual check.
  - **Adaptive (different pattern per viewport)** → mock the project's existing viewport hook/prop and assert which presentation renders.
  - **Missing behaviour (AC never built)** → assert the AC behaviour; the fix adds code.
  - **Timing/race** → fake timers, deferred promises, double clicks, unmount mid-request.
  - **Browser/device/performance-only** → BLOCKED with evidence unless a render/call-count assertion expresses it.
- 3 realistic attempts without a valid failure → BLOCKED with every hypothesis and run folder. No source change.

## Stage 4 — Impact map (before changing code)
Run the related tests once, before any change, to record the baseline:
```bash
python3 ./.agents/skills/run-test-cases/scripts/run-tests.py --related <root-cause-file> --label <ticket_id>-<ISSUE-n>-baseline
```
Write `src/.SS_WF/Agent/TEST_RUNS/<ticket_id>-<ISSUE-n>-impact.md` using `references/impact-map.md`. It must list:
- **A. Reported scenario** — from the ticket.
- **B. Same behaviour, not in the ticket** — every input that reaches the changed code (other codes/branches, success, empty/null/partial data, missing CMS entry, boundary values), every state and transition (initial, loading, error → retry → success, repeated failures, timers start/end/reset, unmount, navigate away/back), locale/RTL, and **each of mobile, tablet and desktop** if UI.
- **C. Related ACs** — every AC that reads or writes the same state/behaviour (thresholds, counters, expiry, disabled states).
- **D. Consumers** — every importer of the code you will change (`grep -rn` the symbol/file), across packages.
- **E. Introduced by the fix** — what your planned change could break: guards removed, defaults changed, new state not reset, timers not cleared, effects re-running, type/shape changes for importers, shared component used elsewhere, earlier defect fixes on these files, and **other viewports** (a base-class change also applies to tablet and desktop unless `md:`/`lg:` overrides it).

Each row: scenario · viewport (if UI) · expected (source: ticket / AC / plan / reconciliation / existing behaviour) · status now (`correct` / `broken-reported` / `broken-found` / `unknown`).
Open the code for every `unknown` until it is `correct` or `broken-found`.

## Stage 5 — Fix
Design one change that makes **every** row in the impact map correct. Load coding skills for the area you touch (rule 6 wins):

| Change touches | Load |
| --- | --- |
| hooks, services, containers, mappers, API errors/toasts | `frontend-logic-integration` |
| state, forms, validation, timers, counters | `frontend-state-and-form-management` |
| markup, styles, layout, RTL, responsive, a11y | `presentational-ui-generation` |
| Sitecore mapping, rendering, CMS messages | `sitecore-rendering-integration` |
| images, icons, media | `frontend-media-integration` |
| creating/moving/renaming files | `repository-structure-governance` |
| test conventions | `frontend-test-generation` |

Fix rules:
- **Root layer.** Fix where the behaviour is wrong; no downstream patch to hide it.
- **Ticket vs AC.** If meeting the ticket would break a C-row AC, find a change that meets both. If they truly conflict, meet the ticket and record the AC conflict.
- **Shared code.** Fixing a shared component, util, type or mapper means every D-row consumer stays correct; changing a public API means updating every caller. No local override that hides a shared bug.
- **One viewport, not the others.** Fix a viewport-specific defect at its own prefix (base / `md:` / `lg:`); prove the other two viewports are unchanged. Fix a defect seen on all viewports at the base and check no `md:`/`lg:` override still carries the bug.
- **No hardcoding.** Rule 9.
- **Several issues, one change.** Allowed; each issue still needs its own failing assertion and its own gate pass.
- **Conflicting issues.** Fix the higher priority (dev notes, then ticket order); the other is BLOCKED with the conflict.
- **Existing test encodes the bug.** Update it, record old assertion → new assertion → reason.
- **broken-found rows (not in ticket).** Fix in this run if they are the same behaviour and the same files, each with its own failing-first assertion. Otherwise record as "found, not fixed" with evidence — no scope creep into other features.
- Remove temporary logging.

Run the reproduction again:
```bash
python3 ./.agents/skills/run-test-cases/scripts/run-tests.py --files <same-test-file> --label <ticket_id>-<ISSUE-n>-after-fix
```
Must be `PASS`.

## Stage 6 — Prove
Update every impact-map row with its evidence:
- `new assertion` — named `it(...)` that fails on the old code (required for every changed behaviour, every broken-found row you fixed, and every E-row your change touched — including each other viewport for a responsive fix).
- `existing test` — file + `it(...)` you actually opened.
- `found, not fixed` / `outside FE` / `AC conflict` — with evidence.
No row may stay `unknown` or `unverified`.

Run related tests on **every** changed source file:
```bash
python3 ./.agents/skills/run-test-cases/scripts/run-tests.py --related <changed-source-file> --label <ticket_id>-<ISSUE-n>-related
```
Failures already present in the baseline are not yours; any **new** failure is a regression your fix introduced → back to Stage 5.

## Stage 7 — Evidence gate (per issue)
```bash
python3 ./.agents/skills/defect-fix/scripts/verify-evidence.py --ticket <ticket_id> --issue <ISSUE-n>
```
`GATE=PASS` → FIXED. `GATE=FAIL` → go to the stage owning the listed reason. Same reason twice → BLOCKED with the gate output. Never edit, skip or delete a test to pass.

Before reporting, **revert any change made only for a BLOCKED issue** (`git -C src checkout -- <file>` / remove its new files).

## Stage 8 — Report
Follow `./.agents/skills/defect-fix/references/reporting.md`. Runs even if every issue is BLOCKED.

## Stage 9 — Final output (exact shape)
```text
DEFECT_VERDICT=<FIXED|PARTIAL|BLOCKED>
<ticket_id>  <n> issues  <n> fixed  <n> blocked   skill=v8
  ISSUE-001  <FIXED|BLOCKED>  <file → symbol>: <mechanism, one line>
    Viewports:    <mobile|tablet|desktop|all|n/a>
    Trace:        <hop> → <hop> → ...
    Reproduction: <test file> :: <it name>   before: <verdict>  after: <verdict>
    Impact map:   <n> rows  (<n> reported, <n> found & fixed, <n> found not fixed, <n> AC conflicts)
    Regressions:  none | <test>
    Gate:         <GATE line>
    Extra context: <file: question> | none
    Skills:       <loaded skills>
  git -C src diff --stat:
    <pasted>
  Code-gen doc: <updated|not updated: reason>   Learning: <written|candidate|none>
```
