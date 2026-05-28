# Test Case Generation Methodology — Mobile App QA (Shift Left)

## Core Principle

Think like a **functional tester**, not a developer. Think **earlier**, not later.

- ❌ "Test when `isCompleted=true` and `hasAttempted=false`" — isolated variable perspective (wrong)
- ✅ "Test an in-progress course state where the exam is now available" — user scenario perspective (correct)

Test cases must reflect real situations a user would encounter in the app, grouped into **Blocks** (thematic groups). Every TC should be traceable to at least one Acceptance Criterion. Quality concerns — gaps in ACs, missing mocks, ambiguous requirements — should be surfaced before testing begins, not discovered during it.

---

## TC Count

**There is no fixed number.** The count depends entirely on ticket complexity:

- A simple component ticket: ~5–10 TCs
- A functional ticket with multiple screen states: ~20–35 TCs
- An integration ticket with multiple Segment events: ~15–25 TCs
- A simple bug: ~3–8 TCs

Generate all relevant scenarios. Do not invent unnecessary cases, but do not omit anything that is testable and relevant to the ticket.

---

## TC Description Format — Gherkin (Default)

All TC descriptions must use **Gherkin format** as the default. This aligns ACs with tests and makes future automation easier.

Standard keywords:

```
Given [context / mock state or screen precondition]
When  [user action or event that triggers the behavior]
Then  [expected outcome, visible to the user]
```

Use `And` to chain multiple conditions or outcomes:

```
Given the course is in progress and lesson 1 has been completed
When the user views the Course Overview screen
Then lesson 1 displays the "REVIEW COURSE" button with teal outline
And  lesson 2 displays the "CONTINUE COURSE" button filled in teal
And  lesson 3 displays the "START COURSE" button in disabled gray
```

**Rules:**
- `Given` sets the mock state or precondition — be specific (e.g. `courseCompleted=false`, `hasAttempted=true`)
- `When` is always a user action or a screen event, not a variable change
- `Then` always describes what the user *sees*, not what the code *does*
- If a TC has no user action (pure display state), use: `When the user opens / views the screen`
- Keep each clause on its own line for readability in the cell (wrap text is enabled)

---

## AC Coverage Tracking

Before generating TCs, extract all Acceptance Criteria from the ticket. Number them (AC-1, AC-2...).

As TCs are generated, tag each one internally with the AC it covers. After generation, produce a summary:

```
AC Coverage:
  AC-1 → TC-01, TC-03
  AC-2 → TC-05
  AC-3 → ⚠️ Not covered — no TC generated for this AC
```

Any uncovered AC must be flagged to the user. Do not silently skip ACs.

If the ticket has no formal ACs, note which TCs are based on description assumptions vs. explicit requirements.

---

## Pre-Quality Flags (Shift Left Signals)

Raise these flags *before* generating TCs. They are opportunities to fix quality issues upstream:

| Condition | Type | Action |
|-----------|------|--------|
| No ACs found | Warning | Notify user; proceed only with confirmation; mark TCs as assumption-based |
| Vague/untestable ACs | Warning | List specific ACs that need refinement |
| No design link on a UI ticket | Info | Suggest adding Figma link to ticket |
| Very short description | Info | Flag potential coverage gaps |

These are not blockers by default — the user decides whether to proceed or update the ticket first.

---

## Block Structure

Each Block must have:
- A **color** that visually categorizes the block (see Color Conventions below — no emoji prefixes)
- A sequential number (`BLOCK 1`, `BLOCK 2`...)
- A descriptive name for the scenario set

Example block names (color applied via spreadsheet formatting, not text prefix):
- `BLOCK 1 — Access and Loading`
- `BLOCK 2 — State: Not Started`
- `BLOCK 3 — State: In Progress`
- `BLOCK 4 — State: Completed`
- `BLOCK 5 — Card Types and Labels`
- `BLOCK 6 — Edge Cases`
- `BLOCK 7 — Non-Functional Quality (Mobile)`
- `BLOCK 8 — Mock Checklist`

## Color Conventions

Colors are applied to the block header row and its TC rows. The header uses a saturated tone; TC rows use the matching light tint. Choose the color that best fits the block's semantic meaning:

| Color Name | Header (hex) | Row Tint (hex) | Semantic Meaning |
|------------|-------------|----------------|-----------------|
| Blue | `1A56A0` | `DDEEFF` | Access, loading, initial navigation |
| Amber | `92400E` | `FEF3C7` | Ordering, grouping, screen structure |
| Purple | `7B2D8B` | `F0E0FF` | Not Started — primary entry point |
| Dark Green | `166534` | `D1FAE5` | Not Started — with pretest / alternative flow |
| Teal | `065666` | `CCFBF1` | Not Started — reduced scope (e.g. no final exam) |
| Violet | `3D1A78` | `EDE9FE` | In Progress — partial progress |
| Navy | `1E3A5F` | `DBEAFE` | In Progress — exam or next step available |
| Forest | `14532D` | `DCFCE7` | In Progress — resumed or retried state |
| Red-Orange | `7C2D12` | `FFEDD5` | Completed states / destructive edge cases |
| Slate | `374151` | `F3F4F6` | Card types, labels, visual-only checks |
| Royal Blue | `1E3A8A` | `EFF6FF` | Edge cases, known gaps, null-safety |
| Deep Purple | `4C1D95` | `EDE9FE` | RevenueCat / paywall / subscriptions |
| Forest Green | `1E4D2B` | `D1FAE5` | Non-functional quality (mobile layer) |
| Gray | `374151` | `F9FAFB` | Mock checklist, coverage summary |

---

## Test Case Fields

Each TC must have 6 fields:

| Field | Description |
|-------|-------------|
| **ID** | `TC-01`, `TC-02`... (sequential, zero-padded to 2 digits) |
| **Block** | Full block name (e.g. `BLOCK 1 — Access and Loading`) |
| **Scenario Title** | Short phrase (3–8 words) identifying the scenario |
| **Description** | Gherkin format: `Given / When / Then [/ And]` |
| **Notes / Mock** | Mock data, preconditions, AC reference (e.g. `Ref: AC-2`), known gaps. Use `—` if none. |
| **Automatable?** | `Yes` — deterministic, good automation candidate; `No` — requires visual/exploratory judgment; `-` — not yet assessed |

### Status Dropdown Values

Column G uses: `Not tested` / `Passed` / `Failed` / `Blocked` / `N/A`

Default value on generation: `Not tested`

### Automatable? Dropdown Values

Column H uses: `Yes` / `No` / `-`

Default value on generation: `-` (leave unassessed unless clearly classifiable)

**`Yes`** (good automation candidates):
- Deterministic inputs and outputs
- State-driven behavior (e.g. button X appears when field Y is true)
- API response mapping to UI elements
- Navigation flows with predictable destinations

**`No`** (manual-only):
- Visual consistency checks (colors, alignment, spacing)
- Exploratory edge cases
- Cross-device subjective UX perception
- Cases requiring human judgment on quality

---

## Non-Functional Quality Block — Mobile Layer

Include a `BLOCK N — Non-Functional Quality (Mobile)` block (forest green color) when the ticket involves any of the following. This block covers concerns in the **mobile client layer only** — not server-side performance, API security, or contract testing.

Include when relevant; omit when the ticket is purely data/logic with no UX surface.

**In scope:**

| Concern | Example TC |
|---------|-----------|
| UI performance perception | Given the screen has 20+ cards / When the user scrolls / Then scrolling is smooth without visible jank |
| Loading state duration | Given the network is slow / When the screen is opened / Then the loading spinner is shown for the full duration |
| Graceful degradation (offline/slow network) | Given the device has no connection / When the screen is opened / Then an appropriate error message is shown (no crash) |
| Sensitive data visibility on screen | Given the user lacks permission for X / When they attempt to access it / Then restricted content is not displayed or exposed in the UI |
| State persistence on navigation | Given the user navigates away and returns / When they view the screen again / Then the previous state is correctly preserved |

**Out of scope** (not covered by this plugin):
- Server-side security or data encryption
- API performance or contract testing
- Accessibility

---

## Mock Checklist Block (Final Block)

Always add a final block (`BLOCK N — Mock Checklist`) listing the mock data sets the developer must implement. Each item should include:
- A descriptive mock name
- The key variables required (e.g. `isPreTest=true, orderId=0`)

---

## AC Coverage Summary (After Mock Checklist)

After the mock checklist, add a coverage summary section in the spreadsheet:

```
AC Coverage:
  AC-1 — Covered by TC-01, TC-03  ✅
  AC-2 — Covered by TC-05         ✅
  AC-3 — ⚠️ Not covered by any TC — verify with the team
```

If the ticket had no ACs, write: "No ACs defined — TCs are based on ticket description and context."

---

## What NOT to Generate

- Internal function logic tests (unit tests)
- Individual prop tests outside of context
- Redundant TCs that cover the exact same scenario with different names
- Implementation tests (e.g. "verify the reducer updates the state") — test visible behavior only
- Server-side performance, API security, or contract testing scenarios
