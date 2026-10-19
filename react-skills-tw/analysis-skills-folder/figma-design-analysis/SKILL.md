---
name: figma-design-analysis
description: Use this skill to reconcile mobile, tablet and desktop Figma Design Intent JSONs into a responsive design intent contract, and analyse Figma outputs to enrich component planning, responsive behaviour, RTL, and design-system reuse. Reads already-fetched Figma context files; performs no fetching. Triggers include Figma analysis, Figma reconciliation, responsive design intent, mobile tablet desktop reconciliation, or responsive strategy.
---

# Figma Design Analysis

## Purpose
This skill **reconciles and analyses** Figma Design Intent JSONs already fetched.
- ✅ **Reconcile** mobile, tablet and desktop Design Intent JSONs into `responsive_design_intent.json` (conditional)
- ✅ **Analyse** the Figma outputs to enrich component planning, responsive behaviour, RTL, and design-system reuse
- ❌ **No fetching.** Never connect to Figma MCP, never open a Figma node, never re-scan the JIRA story for URLs.

## Inputs

| Input | Source | Required? |
| --- | --- | --- |
| Figma Design Intent JSONs | `./figma-output/figma_design_{nodeId}_{viewport}-context.json` | Validated present in Phase 4 |
| Existing reconciliation | `./figma-output/responsive_design_intent.json` | If already present |

## ⚠️ VIEWPORTS — READ FIRST

The project has three viewports, matching the Tailwind theme (`--breakpoint-md: 768px`, `--breakpoint-lg: 1024px`):

| Viewport | Width | Tailwind | Grid |
| --- | --- | --- | --- |
| mobile | 390 – 767px | base (no prefix) | 4 columns |
| tablet | 768 – 1023px | `md:` | 8 columns |
| desktop | ≥ 1024px | `lg:` | 12 columns, content max width 1700px |

390px is the minimum supported width.

### Identify each file's viewport from its content
```text
1. Read screenMetadata.viewport  → "mobile" | "tablet" | "desktop" | "unclassified"
2. If absent (older files), read screenMetadata.frameWidth / frameDimensions
   and classify against the table above.
3. "unclassified" or still ambiguous → do not use the file for reconciliation.
   Record it in DEV_REVIEW.md §4. Do NOT guess.
```
The CONTEXT_MANIFEST's VIEWPORT COVERAGE block states which viewports are present — use it to decide whether reconciliation is required before opening the files.

---

# PART 1 — Reconciliation (Conditional)

⚠️ **Run when two or more viewports (mobile / tablet / desktop) exist for the same logical component or screen.**
- Only one viewport present → **skip reconciliation**, go to Part 2.
- `responsive_design_intent.json` already exists with `"schemaVersion": "3-viewport"` → **do NOT re-reconcile**. Use it.
- `responsive_design_intent.json` exists without that schemaVersion (older mobile/desktop file) → it is stale; reconcile again and overwrite it.

## Role
You are a Figma Reconciliation Agent. Compare the available viewport Design Intent JSONs and produce a lean responsive context JSON.

## What NOT to Output
- Full mobile, tablet or desktop JSON
- Raw Figma node tree or raw component hierarchy dump
- Token definitions, variable definitions, full style objects
- Raw colour definitions unless no token or semantic reference exists
- SVG paths, image binaries, repeated low-value metadata
- Every single child node if it does not affect implementation

## Reconciliation Rules

**Rule 1 — Reconcile by semantic intent, not node names.**
- `Forgot Username Button` and `Forgot username?` may both represent the same action.
- `Mobile/Top Navigation` and `Desktop Header` may both represent header areas with breakpoint-specific structures.
- Keep the Figma `figmaName` and `nodeId` of each matched node under `sourceNodes`. Never rename nodes and never use Sitecore, BFF or code names here.

**Rule 2 — Keep shared components lean.** For each shared component include only: Name, What it contains, Mobile behaviour, Tablet behaviour, Desktop behaviour, Key responsive differences, Design-system reuse guidance.

**Rule 3 — Identify breakpoint-specific components.** Classify as:
- `shared-responsive`
- `same-purpose-different-structure`
- `same-component-different-variant`
- `adaptive` — same function, different UI pattern (e.g. modal ↔ bottom sheet, flyout ↔ bottom sheet, sidebar ↔ hamburger drawer, table ↔ card list)
- visible only in some viewports — record `visibleIn: ["mobile" | "tablet" | "desktop"]`

If a component exists in only some viewport designs, **do not invent its behaviour for the others** — mark it hidden there.

**Rule 4 — Convert Figma dimensions into implementation intent.**
- Mobile fill container → fluid width
- Desktop fixed form width → max-width, not hard fixed width
- Column layout on mobile/tablet + row layout on desktop → responsive direction switching
- Record column span per viewport on that viewport's grid (4 / 8 / 12).

**Rule 5 — Missing tablet design.** If mobile and desktop exist but tablet does not, tablet follows the **mobile** structure and tokens on the 8-column grid (full-width spans stay full width). Mark every such value `"source": "derived"` and record it in DEV_REVIEW.md §4. If tablet exists but mobile or desktop does not, the missing viewport follows the tablet structure (mobile on 4 columns; desktop with no change above 1024px, content max 1700px), also marked derived.

**Rule 6 — Always include RTL expectations.**
- Use start and end, not hardcoded left and right
- Use logical spacing and alignment
- Do not mirror brand logos
- Mirror only directional icons when needed
- Ensure Arabic text can expand without clipping
- Prefer `text-start` and `text-end`
- Ensure button icon placement works in RTL

## Reconciliation Output
Save as `./figma-output/responsive_design_intent.json`.

```json
{
  "schemaVersion": "3-viewport",
  "componentName": "",
  "componentType": "page | component | pattern",
  "sourceViewports": {
    "mobile":  { "file": "", "width": "390px",  "source": "figma | derived | missing" },
    "tablet":  { "file": "", "width": "768px",  "source": "figma | derived | missing" },
    "desktop": { "file": "", "width": "1700px", "source": "figma | derived | missing" }
  },
  "responsiveStrategy": "",
  "implementationModel": "single-responsive-component | single-responsive-page | responsive-composition-with-breakpoint-specific-subcomponents",
  "sharedComponents": [
    {
      "name": "",
      "sourceNodes": { "mobile": { "nodeId": "", "figmaName": "" }, "tablet": null, "desktop": { "nodeId": "", "figmaName": "" } },
      "responsibility": "",
      "contains": [],
      "classification": "shared-responsive | same-purpose-different-structure | same-component-different-variant | adaptive",
      "mobile":  { "layout": "", "columnSpan": 4, "width": "", "spacing": "", "typography": {}, "visibility": "visible", "source": "figma" },
      "tablet":  { "layout": "", "columnSpan": 8, "width": "", "spacing": "", "typography": {}, "visibility": "visible", "source": "derived" },
      "desktop": { "layout": "", "columnSpan": 6, "width": "", "spacing": "", "typography": {}, "visibility": "visible", "source": "figma" },
      "responsiveHandling": []
    }
  ],
  "breakpointSpecificComponents": [
    { "name": "", "visibleIn": ["desktop"], "reason": "", "implementationHandling": "" }
  ],
  "layoutRules": {
    "mobile":  { "rootDirection": "", "grid": "4 columns", "horizontalMargin": "", "contentPriority": [], "containerBehaviour": "" },
    "tablet":  { "rootDirection": "", "grid": "8 columns", "horizontalMargin": "", "contentPriority": [], "containerBehaviour": "" },
    "desktop": { "rootDirection": "", "grid": "12 columns", "horizontalMargin": "", "contentPriority": [], "containerBehaviour": "max-width 1700px, centred" },
    "transformation": []
  },
  "keyResponsiveDifferences": [{ "area": "", "mobile": "", "tablet": "", "desktop": "" }],
  "stateAndInteractionGuidance": [
    { "element": "", "stateOrInteraction": "", "codingGuidance": "" }
  ],
  "rtlGuidance": [""]
}
```

---

# PART 2 — Analysis and Enrichment

## Source Authority

**If `responsive_design_intent.json` exists (two or more viewports were provided):**

It is the **authoritative source** for responsive interpretation.
- Do NOT perform viewport reconciliation again.
- Do NOT reclassify shared, adaptive or breakpoint-specific components.
- Do NOT derive new responsive rules from the raw context JSONs.

Use it as the source of truth for: responsiveStrategy, implementationModel, sharedComponents, breakpointSpecificComponents, layoutRules, keyResponsiveDifferences, rtlGuidance.

Use the mobile, tablet and desktop Design Intent JSONs **only as supporting references** for:
- Confirming exact design-system component references
- Checking component variant names, state names, text labels, token names
- Checking source node names or hierarchy when relevant
- Validating whether a required element exists in the source design

If a context JSON conflicts with the reconciliation JSON, **preserve the reconciliation JSON as the source of truth** and report the conflict in DEV_REVIEW.md §3.

**If only one viewport is provided (no reconciliation):**

Use the single Design Context JSON directly for component planning, design-system reuse planning, and identifying implementation risks. Record in DEV_REVIEW.md §2 that the other two viewports were not provided; the Coding Agent applies the same layout on the other viewports' grids.

## ⚠️ ANALYSIS DEPTH vs OUTPUT WIDTH
Perform every analysis check below in full. What is emitted into ANALYSIS_PLAN.md is narrower.

| Work Performed (always, in full) | Emitted To |
| --- | --- |
| Target frame/node identification | Internal |
| Layer hierarchy review | Internal → enriches **§5** hierarchy |
| Semantic component identification | Internal → enriches **§5**, **§13.1** |
| Text/copy ownership inference (CMS vs API vs system) | **§12** per-prop Source |
| Icon reference capture | Internal → enriches **§12** props |
| Auto-layout / responsive interpretation | **§13.2** exceptions |
| Adaptive components (different pattern per viewport) | **§13.2** — one row per adaptive component |
| Derived viewport values (missing tablet / mobile / desktop design) | **DEV_REVIEW.md §4** |
| Design-system primitive identification | **§13.1** reuse decisions |
| "Do not create" component identification | **§8** Things NOT to Implement |
| Figma-only interactions | **§9** — continue INT-xxx numbering |
| Design-driven states (hover, focus, transitioning) | **§10** — continue STATE-xxx numbering |
| RTL requirements | **§13.2** — story-specific exceptions only |
| Design ambiguity | **DEV_REVIEW.md §4** |
| Content the design requires but no source provides | Flag for the next phase gap sweep |

⚠️ Do **not** emit a standalone Figma analysis section into ANALYSIS_PLAN.md. Figma findings feed the existing sections above. Baseline RTL/responsive/a11y rules are already embedded in the Coding Agent's skills — emit only story-specific exceptions.

## Enrichment Duties (From Phase 3 Drafts)
Phase 3 ran on story text alone. Complete what it could not know:

| Draft from Phase 3 | Enrichment here |
| --- | --- |
| **§9 interactions** — story-stated only | Add Figma-only interactions. **Continue INT-xxx numbering; never renumber.** If an interaction differs by viewport, keep one row and state the per-viewport behaviour. Record Figma-derived provenance in DEV_REVIEW.md §4. |
| **§10 states** — story-stated only | Add design-driven visual states (hover, focus, transitioning, paused). **Continue STATE-xxx numbering.** |
| **§12 prop model** — Source Detail marked To be resolved | Resolve Sitecore-sourced props where the design confirms the field. API-sourced props are resolved in API Phase 6. |
| **§13.2 NFR exceptions** | Add design-driven exceptions — e.g. a carousel needing `rtl:rotate-180` on pager arrows, a fixed-height scroll region, or an adaptive component (modal on desktop, bottom sheet on mobile/tablet). |
| **§8 Things NOT to Implement** | Add design elements deliberately not built in this story. |

## Feeding the API Gap Sweep
While analysing the design, note every piece of **content the design displays** — labels, values, media, counts, status text. Phase 6 runs next and compares these against the actual contracts. Anything the design requires but no contract provides becomes a recorded gap.

⚠️ Do not attempt the gap analysis here — the API contracts have not been read yet. Simply ensure design-required content is captured in the §12 prop model so the gap sweep has something to check against.

## Figma Analysis Checklist

| Check | What to Validate |
| --- | --- |
| Figma context read | Confirm Figma data was available |
| Viewport identified per file | From `screenMetadata.viewport`, not filename |
| Viewport coverage recorded | Mobile / tablet / desktop each Figma, Derived or Missing |
| Target frame/node identified | Node name and type captured |
| Layer hierarchy reviewed | Understand visible UI structure |
| Text/copy ownership inferred carefully | Determine whether labels come from CMS or API/system |
| Icon references noted | Capture icon names only, not actual assets |
| Auto-layout/layout hints considered | Use dimensions/layout only as guidance, not hardcoded implementation |
| Design ambiguity captured | Missing/unclear component intent → DEV_REVIEW.md §4 |
| Design-required content captured | Present in §12 for the Phase 6 gap sweep |

## Guardrails for Analysis
- ✅ Use the Responsive Reconciliation JSON as the authoritative responsive contract
- ✅ Use mobile/tablet/desktop context JSONs only as detail references
- ✅ Identify shared, adaptive and breakpoint-specific components
- ✅ Identify responsive layout transformations across all three viewports
- ✅ Identify RTL requirements
- ✅ Identify design-system primitives and semantic components to reuse
- ✅ Identify what must not be created as a new component → feeds §8
- ❌ Do not treat desktop as the default source of truth
- ❌ Do not treat mobile as a scaled-down version of desktop
- ❌ Do not perform reconciliation again if a current (`3-viewport`) `responsive_design_intent.json` exists
- ❌ Do not use context JSONs to override the Responsive Reconciliation JSON
- ❌ Do not hardcode Figma dimensions — convert to implementation intent
- ❌ Do not recreate design-system components from Figma visuals

## Guardrails

### Always Do
- Read only the Figma files already written in Phase 2.
- Identify viewport from `screenMetadata`, never from the filename.
- Reconcile only when two or more viewports are present and no current reconciliation file exists.
- Treat the reconciliation JSON as authoritative once produced.
- Mark every derived viewport value and record it in DEV_REVIEW.md §4.
- Enrich Phase 3's drafts — §5, §8, §9, §10, §12, §13.
- **Continue existing INT-xxx and STATE-xxx numbering; never renumber.**
- Capture design-required content in §12 so the Phase 6 gap sweep can check it.
- Record design ambiguity in DEV_REVIEW.md §4 and conflicts in §3.

### Never Do
- **Never fetch from Figma MCP** — acquisition was Phase 2.
- **Never re-scan the JIRA story for Figma URLs** — use the CONTEXT_MANIFEST.
- **Never re-derive or silently change the component classification.** If the design contradicts it, record an analysis risk in DEV_REVIEW.md §1.
- **Never invent design intent, node names, tokens, or component structure.**
- Never rename Figma nodes or label them with Sitecore, BFF or code names.
- Never invent behaviour for a component in a viewport whose design does not show it.
- Never reuse a `responsive_design_intent.json` that lacks `"schemaVersion": "3-viewport"`.
- Never emit a standalone Figma section into ANALYSIS_PLAN.md.
- Never emit baseline RTL/responsive rules — only story-specific exceptions.
- **Never renumber IDs assigned in Phase 3.**
- Never perform API gap analysis here — the contracts have not been read yet.

## Gate: This Skill Completes When
- [ ] CONTEXT_MANIFEST consulted for viewport coverage.
- [ ] Viewport identified per file from `screenMetadata`.
- [ ] Reconciliation performed **only** when two or more viewports present and no current reconciliation file.
- [ ] `responsive_design_intent.json` written with `schemaVersion: 3-viewport` (or correctly skipped).
- [ ] Derived viewport values marked and recorded in DEV_REVIEW.md §4.
- [ ] All analysis checklist rows completed.
- [ ] Phase 3 drafts enriched: §5, §8, §9, §10, §12, §13.
- [ ] Figma-only interactions added with continued INT-xxx numbering.
- [ ] Design-driven states added with continued STATE-xxx numbering.
- [ ] Design-required content captured in §12 for the gap sweep.
- [ ] Classification left unchanged; any contradiction recorded as a risk.
- [ ] No standalone Figma section produced.
- [ ] Design ambiguity → DEV_REVIEW.md §4; conflicts → DEV_REVIEW.md §3.
