---
name: context-gathering
description: Use to acquire all external context for a story — Sitecore API JSON, BFF API specs, and Figma Design Intent JSONs for mobile, tablet and desktop — and write them to disk for downstream analysis phases. Performs acquisition only; no reconciliation and no analysis. Triggers include context gathering, fetch context, Figma fetch, Sitecore API fetch, BFF spec fetch, or gather story context.
---

# Context Gathering

## Purpose

This skill is a **single acquisition phase**. It reads the JIRA story once, fetches every external artefact the story depends on, and writes them to fixed locations on disk.

It performs **acquisition only**:

- ✅ Extract endpoints and URLs from the JIRA story
- ✅ Fetch Sitecore API JSON, BFF API specs, and Figma Design Intent JSONs (one per viewport design)
- ✅ Record each Figma design's viewport (mobile / tablet / desktop) as a fact
- ✅ Write each artefact to its canonical path
- ✅ Report what was fetched, what was missing, and what failed
- ❌ No reconciliation across viewports (Phase 5)
- ❌ No contract analysis, no Sitecore-to-Figma mapping (analysis phases)
- ❌ No component planning, no story interpretation, no validation judgement

⚠️ **This skill does not judge whether a missing artefact or a missing viewport matters.** It records outcomes; Phase 4 judges them.

## Output Locations (Canonical)

| Artefact                      | Path                                                                                                                              |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Sitecore API JSON             | `.SC_API_SPEC/sitecore-api.json`                                                                                                  |
| BFF API spec (per operation)  | `.BFF_API_SPEC/{operationId}.json`                                                                                                |
| Figma Design Intent (per URL) | `figma-output/figma_design_{nodeId}_{viewport}-context.json` — `{viewport}` = `mobile` \| `tablet` \| `desktop` \| `unclassified` |

These folders already exist; use them, never create new ones:
`./src/.SC_API_SPEC/`, `./src/.BFF_API_SPEC/`, `./src/figma-output/`

---

## ⚠️ SITECORE FETCH — USE THE PROJECT SCRIPT, NOT THE HTTP TOOL

```bash
bash scripts/fetch-sitecore-api.sh "<complete-endpoint-url>"
```

**Do NOT use the generic http tool for Sitecore.** Do NOT construct an ad-hoc curl command.

### Why a script

| Concern                 | How the script handles it                                                          |
| ----------------------- | ---------------------------------------------------------------------------------- |
| Self-signed certificate | `-k` is applied inside the script. Never pass TLS flags from the caller.           |
| Silent-but-loud errors  | `-sS` is applied inside the script.                                                |
| Timeouts                | Connect 15s / total 60s.                                                           |
| Redirects               | Followed, capped at 5.                                                             |
| Partial writes          | Staged to a temp file; written only after HTTP status and JSON validity both pass. |
| Consistency             | Identical flags every run.                                                         |

### Usage

```bash
bash scripts/fetch-sitecore-api.sh "https://cm.dev.internal.example.net/sitecore/api/layout/render/jss?item=/path&sc_apikey=XXX"
bash scripts/fetch-sitecore-api.sh "<endpoint>" ".SC_API_SPEC/sitecore-api.json"   # explicit path, rarely needed
```

Pass the complete endpoint exactly as in the story (scheme, host, path, query string).

### Exit codes → manifest

| Exit | Meaning                    | Manifest status                 |
| ---- | -------------------------- | ------------------------------- |
| 0    | Valid JSON written         | Fetched                         |
| 1    | Missing/malformed endpoint | Fetch Failed — invalid endpoint |
| 2    | Network/transport failure  | Fetch Failed — transport        |
| 3    | Non-2xx                    | Fetch Failed — HTTP <status>    |
| 4    | Empty or unparseable JSON  | Fetch Failed — invalid payload  |
| 5    | Could not write            | Fetch Failed — filesystem       |

On any non-zero exit the file is not created. Record the exit code and the script's stderr in the manifest, then continue with BFF and Figma.

---

## ⚠️ TASK-LEVEL FAIL-STOP RULE

Three independent tasks: Sitecore, BFF, Figma. If one fails, **stop that task only** and record it; the other two still run.
Within Figma, each URL is independent: one failed viewport never stops the others.
"SKIP the whole task" in the prompt below means **the BFF task only**.
**Never** invent an endpoint, field, node, token, component, or viewport design to fill a gap.

---

## ⚠️ FIGMA VIEWPORTS (mobile / tablet / desktop)

Stories provide **up to three Figma designs: mobile, tablet and desktop**. Each URL is extracted independently into its own file.

### Viewport ranges (from the project Tailwind theme)

The theme defines `--breakpoint-md: 768px` and `--breakpoint-lg: 1024px`.

| Viewport | Root frame width                    | Tailwind prefix    | Expected grid |
| -------- | ----------------------------------- | ------------------ | ------------- |
| mobile   | < 768px                             | none (base styles) | 4 columns     |
| tablet   | 768px – 1023px                      | `md:`              | 8 columns     |
| desktop  | ≥ 1024px (content max width 1700px) | `lg:`              | 12 columns    |

If the theme breakpoints change, update this table to match.

### Classify each URL's viewport (a fact, not a judgement)

1. **JIRA label first.** If the story labels the link (e.g. `Desktop -`, `Tablet -`, `Mobile -`), use that label.
2. **Verify with the root frame width** against the table above.
3. **No label:** classify by width alone.
4. **Label and width disagree, or width fits no range:** set viewport `unclassified`; record both the label and the width in `screenMetadata` and the manifest. Never guess.
5. **Two URLs classify to the same viewport:** write both files (each has its own nodeId) and list the duplicate in the manifest. Never drop or merge one.

### Per-viewport extraction

- Run Steps 1–11 of the prompt **fully and separately for every viewport**. Never copy values from one viewport's file into another.
- Never synthesise a missing viewport (e.g. derive tablet from mobile + desktop). A missing design is recorded as missing.
- `screenMetadata` must include `viewport`, `tailwindPrefix` (`base` | `md` | `lg` | `null`), `deviceType`, `frameWidth`, `frameHeight`, `jiraLabel` (or `null`) and `sourceUrl`. This is the only viewport signal downstream.
- `layoutGrid` must include `columnCount`, `gutterWidth`, `margins`, `maxWidth` (if set) for that viewport. Record what Figma shows, even if it differs from the expected 4 / 8 / 12 columns; Phase 4 judges the mismatch.

---

## ⚠️ NAMING (hard rule)

Figma output must be a faithful record of the design, so Phase 5 can match the same element across mobile, tablet and desktop.

For every node in `componentHierarchy`:

- `figmaName` — the Figma layer/frame name **exactly** as in Figma, including generic names like `Frame 427`.
- `nodeId` — the Figma node id. Always present.
- `componentRef` — the main component name from the Figma library for instances, else `null`.
- `variant` / `state` — from the instance (Step 7).
- `semanticType` — the Step 9 classification.

Never rename, merge, or split Figma nodes. **Never use Sitecore, BFF, or code names in Figma output**, even if they are already in context. Mapping Figma to Sitecore happens in analysis, not here.

---

## THE CONTEXT SYNTHESISOR PROMPT (Use Verbatim)

Execute the following prompt exactly as written.

You are context Context synthesisor.
Read the JIRA User story to extract out the -

- Sitecore API Endpoints - Analyse the story to fetch the complete endpoint of Sitecore API. Always use the complete endpoint.
  and then save the output in .SC_API_SPEC/sitecore-api.json
- BFF Endpoints - use open-api-spec MCP Tool to get data for all the operation ids mentioned in the Jira story. Save the output inside .BFF_API_SPEC/{operationId}.json. If NO Backend API details are provided, do not invent API fields or endpoints, THEN SKIP the whole task and output with just "API NOT FOUND". Never use Yaml Spec file name as the operationId or Endpoint. Endpoint or operationId is different from name of Yaml spec file and is mentioned individually.

FOLDER / FILE STRUCTURE VIOLATIONS (HARD RULES — Zero Exceptions):

- Do NOT create a new src folder — it already exists at the repository root; use the existing one
- The skill never writes any file outside "src" folder. Paths of files to be created are relative to src folder.

From the data of JIRA Story, find all the Figma URL endpoints which are mentioned in it, along with any viewport label next to each (Mobile / Tablet / Desktop).

Your responsibility is to connect to the Figma MCP Server, analyse all provided Figma designs, and generate one Design Intent JSON file per Figma URL, optimised for downstream frontend code generation.

Mention all the FIGMA URLs found, with their viewport.

HARD RULES -
The output MUST focus on design intent and implementation guidance.
DO NOT return raw Figma node data.
DO NOT return unnecessary MCP payloads.
DO NOT return variable definitions, design token definitions, full vector data, or verbose style definitions.
Record names exactly as in Figma; never rename nodes or use Sitecore/BFF/code names.
Your objective is to minimise tokens while maximising implementation accuracy.

### INPUT

READ the JIRA Data to extract all the FIGMA URLS and repeat the steps for all URLs one by one, independently for each viewport.

### STEP 1 - CONNECT TO FIGMA

Use the Figma MCP server to:
Open the supplied node.
Traverse all nested children.
Identify screen hierarchy.
Identify reusable components (by their Figma library component name).
Extract only implementation-relevant metadata.

### STEP 2 - ANALYSE SCREEN STRUCTURE

Identify:
Page Name
Screen Name
Node ID
Frame Dimensions
Viewport (mobile | tablet | desktop | unclassified) and Tailwind prefix (base | md | lg) — per the FIGMA VIEWPORTS rules
Device Type
Primary Layout Pattern
Classify screen as:
Page, Modal, Drawer, Bottom Sheet, Tab View, Wizard, Dashboard, List View, Form, Detail View, Other

### STEP 3 - EXTRACT LAYOUT GRID INFORMATION

Extract all layout guide details.
Capture:
Grid Type
Column Count
Row Count
Gutter Width
Margins
Max Width (if defined)
Grid Alignment
Grid Width Behaviour
Stretch Behaviour
This information is mandatory.
The Coding Agent will use this to recreate responsive layouts.
Extract how all components are structured and fit on this viewport's grid in terms of columns (column start and span per component).

### STEP 4 - EXTRACT RESPONSIVE BEHAVIOUR

For every frame and component extract:

#### Constraints

Horizontal Constraint
Vertical Constraint
Examples: Left, Right, LeftRight, TopBottom, Centre, Scale

#### Resizing Behaviour

Fixed Width, Fixed Height, Hug Content, Fill Container

#### Size Limits

Capture when defined: Min Width, Max Width, Min Height, Max Height
This information is mandatory.

### STEP 5 - EXTRACT AUTO LAYOUT INFORMATION

For every auto-layout container capture:
Direction, Padding, Gap, Cross-Axis Alignment, Main-Axis Alignment, Wrap Setting, Fill/Hug Settings
Represent using implementation-friendly values.
The Coding Agent should be able to directly map this to Flexbox or CSS Grid.

### STEP 6 - BUILD COMPONENT HIERARCHY

Generate complete parent-child hierarchy.
Capture:
figmaName (exact Figma layer name)
nodeId
componentRef (library component name for instances, else null)
Component Type
Parent Component
Child Components
Text/Name if present on that component
Preserve nesting.
Never flatten the structure.
Never rename, merge or split nodes.
The hierarchy should represent actual component composition.
Instead of mentioning ids and leaving coding agent to connect them and read its complete data; add nested component with details as per hierarchy inside childComponents array.

### STEP 7 - EXTRACT COMPONENT INSTANCE INFORMATION

Capture only:
Referenced Component Name
Variant Selection
Component State (Default, Hover, Focus, Active, Disabled, Error, Loading)
DO NOT return the full component instance payload.

### STEP 8 - EXTRACT DESIGN TOKEN USAGE

DO NOT extract variable definitions, token definitions, or token values.
Extract only token references used by components (e.g. colour-primary-500, spacing-4, font-body-md, radius-md).
Capture usage under: Colours, Typography, Spacing, Border Radius, Shadows, Elevation, Borders

- NEVER hardcode any value - Use only token names.
- Values without a token name go into missing_tokens.

### STEP 9 - IDENTIFY SEMANTIC COMPONENT TYPES

For every component, based on COMPONENT INSTANCE INFORMATION, NAME and TYPE, infer the most likely semantic type.
For example, an element which is visually a link but is built using the Button component should be "Button", with the design tokens and variants from its instance information.
Classifications: Button, Link, Input, Text Area, Select, Radio, Checkbox, Switch, Card, Table, Tabs, Badge, Chip, Banner, Alert, Header, Footer, Navigation, Breadcrumb, Pagination, Modal, Drawer, Accordion, Carousel, Skeleton, Progress Indicator
Assign one primary semantic type. Record it in semanticType; never use it to rename the node.

### STEP 10 - TYPOGRAPHY INTENT

Capture: Semantic Role, Token Name, Text Style Usage (e.g. Heading XL, Heading LG, Body MD, Caption SM)
DO NOT capture verbose text style metadata.

### STEP 11 - ICON INVENTORY

Capture: Icon Name, Semantic Meaning
DO NOT export binary image content.
DO NOT export SVG path data.

### OUTPUT FORMAT

Return a single JSON structure per Figma URL.
{
"screenMetadata": { "viewport": "", "tailwindPrefix": "", "deviceType": "", "frameWidth": 0, "frameHeight": 0, "jiraLabel": null, "sourceUrl": "", "nodeId": "" },
"layoutGrid": { "columnCount": 0, "gutterWidth": "", "margins": "", "maxWidth": null },
"responsiveRules": {},
"componentHierarchy": [ { "figmaName": "", "nodeId": "", "componentRef": null, "semanticType": "", "gridPlacement": { "columnStart": 0, "columnSpan": 0 }, "childComponents": [] } ],
"missing_tokens": []
}

### DO NOT INCLUDE

Raw MCP response, Raw Figma JSON, Variable definitions, Variable values, Token definitions, Complete text content, Full vector geometry, Raw paint definitions, Raw effect definitions, Image binaries, SVG paths, Unused metadata.
Only return information that directly improves frontend code generation quality.
Repeat the above steps for all the Figma URLs found and save each output in the figma-output folder with name: figma*design*{nodeId}\_{viewport}-context.json

### END OF VERBATIM PROMPT

---

## File Naming

| Artefact | Written as                                                                       |
| -------- | -------------------------------------------------------------------------------- |
| Sitecore | `.SC_API_SPEC/sitecore-api.json` — one file                                      |
| BFF      | `.BFF_API_SPEC/{operationId}.json` — one per operationId                         |
| Figma    | `figma-output/figma_design_{nodeId}_{viewport}-context.json` — one per Figma URL |

The viewport in the filename and `screenMetadata.viewport` must match. `unclassified` files are still written.

---

## Required Output: CONTEXT_MANIFEST

Phase 4 depends on this; Phases 5 and 6 use it to avoid re-scanning the story.

```text
CONTEXT MANIFEST — {{ticket_id}}

SITECORE
  Status:      Fetched | Not Found in Story | Fetch Failed
  Endpoint:    [complete endpoint used, or: None found in story]
  Fetched via: scripts/fetch-sitecore-api.sh
  Script exit: [0–5]
  File:        .SC_API_SPEC/sitecore-api.json | Not written
  Note:        [exact failure reason from stderr, if applicable]

BFF
  Status:      Fetched | API NOT FOUND | Fetch Failed
  Operation IDs found in story: [id1, id2, …] | None
  Files written:
    - .BFF_API_SPEC/{operationId}.json
  Not retrieved: [operationIds with reason]

FIGMA
  Status:      Fetched | Partially Fetched | No Figma URLs in Story | Fetch Failed
  URLs found:
    - <url>  label: <Mobile|Tablet|Desktop|none>
  Files written:
    - figma-output/figma_design_{nodeId}_{viewport}-context.json  (viewport: …, prefix: base|md|lg, frame: NNNpx, grid: N cols)
  Not retrieved: [URLs with reason]
  Classification notes: [label/width mismatches, unclassified widths, duplicate viewports] | None

VIEWPORT COVERAGE
  Mobile  (<768px,  base): Present | Missing
  Tablet  (768–1023, md:): Present | Missing
  Desktop (≥1024,    lg:): Present | Missing
  Unclassified files: [list] | None
  Reconciliation required in Phase 5: Yes (2+ viewports present) | No (single viewport)
```

Report counts accurately — Phase 4 compares URLs found against files written.

---

## Guardrails

### Always Do

- Read the JIRA story **once** and extract all Sitecore endpoints, BFF operationIds, and Figma URLs with their viewport labels.
- Fetch Sitecore via `scripts/fetch-sitecore-api.sh` with the complete endpoint; capture exit code and stderr.
- Write every artefact to its canonical path; name BFF specs after their operationId.
- Classify every Figma URL against the 768 / 1024 breakpoints; extract each viewport separately and fully.
- Populate `screenMetadata` (viewport, prefix, width, label, URL) and `layoutGrid` for each Figma file.
- Record Figma names and nodeIds exactly as in Figma.
- List every URL, endpoint and operationId found, including failed ones.
- Report the CONTEXT_MANIFEST with VIEWPORT COVERAGE for all three viewports.
- On any fetch failure, stop that task (or that URL) only and record the reason.

### Never Do

- Never invent an endpoint, field, node, token, component, or viewport design.
- Never synthesise a missing viewport from the others, or copy values between viewport files.
- Never guess a viewport when label and width disagree — use `unclassified`.
- Never rename, merge, or split Figma nodes, or use Sitecore/BFF/code names in Figma output.
- Never abort the phase because one task or one URL failed.
- Never treat "SKIP the whole task" as covering Sitecore or Figma.
- Never judge whether a missing artefact or viewport is acceptable — that is Phase 4's job.
- Never use the generic http tool or an ad-hoc curl for Sitecore; never pass `-k`, `-sS` or TLS flags to the script.
- Never write `.SC_API_SPEC/sitecore-api.json` by hand or from a partial response; never treat a non-zero exit as success.
- Never use a YAML spec file name as a BFF operationId or endpoint.
- Never reconcile mobile, tablet and desktop here — that is Phase 5.
- Never analyse contracts, plan components, map Figma to Sitecore, or interpret the story here.
- Never return raw MCP payloads, raw Figma node data, token definitions, SVG paths, or image binaries.
- Never create a new src folder or write outside the canonical paths.

---

## Gate: Phase 2 Complete When

- [ ] JIRA story read once; all Sitecore endpoints, BFF operationIds, and Figma URLs (with labels) extracted.
- [ ] Sitecore task completed or recorded as Not Found / Fetch Failed.
- [ ] BFF task completed or recorded as API NOT FOUND / Fetch Failed.
- [ ] Every Figma URL written to `figma_design_{nodeId}_{viewport}-context.json` or recorded as failed.
- [ ] Every Figma file has `screenMetadata` (viewport, prefix, width, label, URL) and `layoutGrid` (column count, gutter, margins) populated.
- [ ] Figma names and nodeIds recorded exactly; no Sitecore/BFF/code names in Figma output.
- [ ] No invented or synthesised artefacts, viewports, fields, or components.
- [ ] A failure in one task or URL did not stop the others.
- [ ] CONTEXT_MANIFEST reported with accurate counts and VIEWPORT COVERAGE for mobile, tablet and desktop.
