# lc-claude-mobile-test-cases-handler

A Cowork plugin that generates formatted QA test case spreadsheets from Jira tickets, saving them directly to a Google Drive folder of your choice.

Built for mobile app QA workflows, but adaptable to any Jira project and Drive folder.

## What It Does

Given a Jira ticket ID or URL, the plugin:

1. **Configures itself on first use** — asks for your Jira board and Drive folder (once only)
2. **Fetches the ticket** from Jira via the Atlassian MCP
3. **Classifies the ticket type** (Functional, Integration, Component, or Bug)
4. **Generates test cases** organized into thematic Blocks, from a functional tester's perspective
5. **Exports a formatted `.xlsx`** locally — color-coded blocks, Gherkin descriptions, dropdowns, multi-sheet layout
6. **Uploads to your Google Drive folder** as an XLSX file via the Drive MCP

## First-Time Setup

On the first run, the plugin will ask you two questions:

1. **Your Jira board URL** — e.g. `https://yourorg.atlassian.net/jira/software/c/projects/MYPROJECT/boards/123`
2. **Your Google Drive output folder** — the URL or folder ID where spreadsheets should be saved

Your answers are saved as a file called **`lc-mobile-qa-settings.json`** in the root of your Google Drive (via the Drive MCP). At the start of every future session the plugin searches Drive for this file automatically — no re-configuration needed, regardless of whether you're using Claude Code local or Cowork. See [Configuration File](#configuration-file) for the file format.

## How to Use

Just mention a ticket ID or URL in chat:

- "Generate test cases for PROJ-1234"
- "Create the QA sheet for https://yourorg.atlassian.net/browse/PROJ-5678"
- "I want to test ticket PROJ-9999"

## Ticket Types Supported

| Type | Description |
|------|-------------|
| **Functional** | Complete feature with API integration — focus on screen states and user interaction |
| **Integration** | Endpoint consumption: GraphQL, Segment analytics, Firebase, RevenueCat |
| **Component** | React Native component — visual variants, interactive states, props |
| **Bug** | Mapped error — reproduction, fix verification, regression |

## Spreadsheet Format

- 8 columns: Passed ✓ / ID / Block / Scenario Title / Description (Given / When / Then) / Notes / Status / Automatable?
- Color-coded thematic blocks — no emoji prefixes; color applied via block header background and row tinting
- Column A: Python boolean `False` — Google Sheets converts to a native checkbox in one click (select column → Insert → Checkbox)
- Column G: Status dropdown — Not tested / Passed / Failed / Blocked / N/A
- Column H: Automatable? dropdown — Yes / No / -
- Block Summary tab with TC count per block and color legend
- Mock checklist block at the end (for dev confirmation)
- AC coverage summary section (tracks which Acceptance Criteria are covered)
- **All spreadsheet content is in English** — structural elements and TC content alike

## Drive Output Format

The plugin uploads the XLSX via the Drive MCP. The file lands in the configured folder in XLSX format — the Drive MCP doesn't auto-convert XLSX to Google Sheets. The Step 7 confirmation includes a one-line instruction to convert manually: open the file → **File → Save as Google Sheets** → move the converted copy back to the target folder.

### Stale-duplicate scan

On every successful upload, the plugin scans your Drive for stray copies of the same ticket's test cases (older versions of this plugin sometimes left a duplicate Google Sheet at the Drive root). Any matches are listed with their URLs so you can clean up manually — the Drive MCP doesn't expose a delete tool, so removal is a manual UI action.

## Quality Approach (Shift Left)

The plugin surfaces quality signals before generating TCs:

- Flags missing or vague Acceptance Criteria
- Flags UI tickets with no linked design
- Tracks AC coverage — reports any ACs without a corresponding TC
- Marks each TC as automatable (Yes / No / -) to support CI pipeline prioritization

## Requirements

- **Claude Code** with **Cowork** support — available on **Team** and **Enterprise** plans only
- **Atlassian MCP** connected and authenticated (for Jira access)
- **Google Drive MCP** connected and authenticated (for the upload + verification + stale-duplicate scan)
- Python 3 with `pip` available in the execution environment (`openpyxl` is installed automatically if needed)

> **Note:** This plugin will not work on Free or Pro plans. It depends on the Cowork plugin system and persistent session storage, which are exclusive to Team and Enterprise plans.

## Configuration File

After first-time setup, your settings are stored as **`lc-mobile-qa-settings.json`** in the root of your Google Drive. The plugin finds it at the start of every session via a Drive MCP search — no local filesystem paths involved, so it works identically in Claude Code local and Cowork.

To change your Jira board or Drive output folder, edit the file directly in Drive or delete it and let the plugin re-run first-time setup on the next session. See `config/settings.json.example` for the expected format.

## Releasing a New Version

Releases are created automatically by GitHub Actions whenever a `v*` tag is pushed.

**Steps to release:**

1. Bump `version` in `plugins/mobile-tc/.claude-plugin/plugin.json`
2. Bump `metadata.version` in `plugins/mobile-tc/skills/generate-test-cases/SKILL.md`
3. Add an entry to `CHANGELOG.md`
4. Commit, tag, and push:

```bash
git add -p
git commit -m "chore: bump version to 0.6.2"
git tag v0.6.2
git push && git push --tags
```

The Actions workflow (`.github/workflows/release.yml`) picks up the tag, extracts the matching section from `CHANGELOG.md`, and publishes the GitHub Release automatically. No manual steps needed after the push.

> GitHub Actions is enabled by default on all GitHub repos (public and private). No additional setup is required — the workflow file committed to `.github/workflows/` is all it takes. The `GITHUB_TOKEN` used to create the release is injected automatically by GitHub; no secrets need to be configured. See [GitHub Actions quickstart](https://docs.github.com/en/actions/writing-workflows/quickstart) if you're unfamiliar with the system.

## Author

Lucas Carvalho Silva — Telus Digital's Test Engineer
