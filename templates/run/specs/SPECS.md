# Ticket specs: {{RUN_NAME}}

<!--
One `## <TICKET>: <title>` section per ticket; lanes find theirs with `grep -n '^## TICKET-1:' specs/SPECS.md`.
Replace the example tickets below and delete this comment.
-->

**Precedence.** For a ticket, its section here is the spec. Where it differs from the issue tracker, this file wins. A section's **Conductor notes** override everything else, including the rest of the section, the lane file, and the workflow defaults in `LANE_RULES.md`. They never relax its Hard limits (local only, secrets, test data, file ownership, Herdr hygiene). Lanes may read the issue tracker for context but never write to it.

Defaults for every ticket: base `{{BASE_REF}}`, gates `$GATES` from `run.env`, all work local.

---

## TICKET-1: <short title>

- **Tracker:** <https://tracker.example.com/TICKET-1>
- **Lane:** L1
- **Branch:** `<feat/TICKET-1-short-slug>`
- **Base:** `{{BASE_REF}}`
- **Screenshots required:** yes (before and after of <screen/state>)

### Behavior and acceptance criteria
- <Observable behavior 1: what the user, admin, or API caller sees or gets.>
- <Observable behavior 2, including edge cases (empty state, missing value, permissions).>
- <What must stay unchanged.>

### Test expectations
- <Which regressions the tests must catch, e.g. "a record with no value shows the default label".>
- <Where tests live, following the repo's conventions.>

### Out of scope
- <Related changes this ticket must not make.>

### Conductor notes (override)
- <Decisions, clarifications, or corrections that win over everything above. Leave empty if none.>

---

## TICKET-2: <short title>

- **Tracker:** <https://tracker.example.com/TICKET-2>
- **Lane:** L1
- **Branch:** `<fix/TICKET-2-short-slug>`
- **Base:** `<feat/TICKET-1-short-slug>` (stacked on TICKET-1)
- **Screenshots required:** no (not user-visible)

### Behavior and acceptance criteria
- <…>

### Test expectations
- <…>

### Out of scope
- <…>

### Conductor notes (override)
- <…>
