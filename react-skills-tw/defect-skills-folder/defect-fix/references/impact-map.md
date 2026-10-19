# Impact map template
File: `src/.SS_WF/Agent/TEST_RUNS/<ticket_id>-<ISSUE-n>-impact.md`

```markdown
# <ticket_id> <ISSUE-n> impact map
Root cause: <file → symbol>: <mechanism>
Planned change: <one line>

| ID | Group | Scenario | Expected (source) | Status before | Evidence after fix |
| --- | --- | --- | --- | --- | --- |
| A1 | Reported | <ticket trigger> [tablet] | <expected> (ticket) | broken-reported | new assertion: <file :: it> |
| B1 | Same behaviour | <other code / empty data / retry> | <expected> (AC-n / plan / existing) | correct | existing test: <file :: it> |
| B2 | Same behaviour | <same UI> [mobile] | <expected> (reconciliation) | correct | new assertion: ... |
| B3 | Same behaviour | <same UI> [desktop] | <expected> (reconciliation) | correct | new assertion: ... |
| C1 | Related AC | <AC-n threshold/counter> | <AC wording> (AC-n) | correct | existing test / new assertion |
| D1 | Consumer | <importer file> | unchanged behaviour (existing) | correct | related run: <label> |
| E1 | Introduced by fix | <base class change reaches md:/lg: / timer not cleared / type change> | <expected> | n/a | new assertion: ... |
```

Put the viewport in square brackets in the Scenario column for UI rows: `[mobile]`, `[tablet]`, `[desktop]`.
For a UI defect, each of the three viewports appears in an A, B or E row.

Allowed "Status before": `correct` · `broken-reported` · `broken-found` · `n/a` (E rows).
Allowed "Evidence after fix" prefixes: `new assertion:` · `existing test:` · `related run:` · `found, not fixed:` · `outside FE:` · `AC conflict:`.
Never leave `unknown` or `unverified`.
