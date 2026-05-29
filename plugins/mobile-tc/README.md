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
6. **Uploads to your Google Drive folder** as a **native Google Sheet** when Claude in Chrome is connected (Path A), falling back to an XLSX upload when it isn't (Path B — see [Drive Output Format](#drive-output-format) below)

## First-Time Setup

On the first run, the plugin will ask you two questions:

1. **Your Jira board URL** — e.g. `https://yourorg.atlassian.net/jira/software/c/projects/MYPROJECT/boards/123`
2. **Your Google Drive output folder** — the URL or folder ID where spreadsheets should be saved

Your answers are saved to the first writable persistent location available — the plugin tries, in order:

1. A `mnt/.claude/` directory under the current Cowork session
2. `$HOME/.claude/` (typical for local agent mode)
3. A `mnt/outputs/` directory (ephemeral — cleared between sessions; only used as last resort)

The chosen path is printed back to you after first-run setup so you can see if it landed somewhere persistent or ephemeral. See [Configuration File](#configuration-file) for the file format.

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

The plugin uses two upload paths depending on what's connected:

### Path A — Native Google Sheet (preferred)

When **Claude in Chrome** is connected and you've enabled **"Convert uploads to Google Docs editor format"** in your Drive account settings, the plugin routes the upload through the Drive web UI. Drive auto-converts the XLSX to a native Google Sheet server-side, and it lands directly inside the configured folder. No leftover files in Drive root, no manual conversion needed.

### Path B — XLSX with manual conversion (fallback)

When Claude in Chrome is unavailable (extension not connected, browser logged out, etc.), the plugin falls back to uploading the XLSX via the Drive MCP. The file lands correctly in the configured folder but stays in XLSX format — the Drive MCP doesn't auto-convert XLSX. The Step 7 confirmation will include a one-line instruction to convert manually (Open the file → File → Save as Google Sheets → move to the target folder).

### Stale-duplicate scan

On every successful upload, the plugin scans your Drive for stray copies of the same ticket's test cases (older versions of this plugin sometimes left a duplicate Google Sheet at the Drive root). Any matches are listed with their URLs so you can clean up manually — the Drive MCP doesn't expose a delete tool, so removal is a manual UI action.

## Quality Approach (Shift Left)

The plugin surfaces quality signals before generating TCs:

- Flags missing or vague Acceptance Criteria
- Flags UI tickets with no linked design
- Tracks AC coverage — reports any ACs without a corresponding TC
- Marks each TC as automatable (Yes / No / -) to support CI pipeline prioritization

## Requirements

**Always required:**
- **Claude Code** with **Cowork** support — available on **Team** and **Enterprise** plans only
- **Atlassian MCP** connected and authenticated (for Jira access)
- **Google Drive MCP** connected and authenticated (for the upload + verification + stale-duplicate scan)
- Python 3 with `pip` available in the execution environment (`openpyxl` is installed automatically if needed)

**Required only for Path A (native Google Sheet output):**
- **Claude in Chrome** extension connected, with the user signed in to Google Drive on the active browser
- **Drive setting "Convert uploads to Google Docs editor format" enabled** at `drive.google.com/drive/settings` → Uploads. This is what tells Drive to auto-convert XLSX uploads to native Google Sheets server-side.

Without either of these two, the plugin still works — it just uses Path B (XLSX upload + manual conversion instruction). See [Drive Output Format](#drive-output-format) for details.

> **Note:** This plugin will not work on Free or Pro plans. It depends on the Cowork plugin system and persistent session storage, which are exclusive to Team and Enterprise plans.

## Configuration File

After first-time setup, your settings are stored at `lc-mobile-qa-settings.json` in the first writable persistent directory found at run time. The discovery order is:

1. `<cowork-session>/mnt/.claude/` (typical for Cowork remote sessions where the mount is writable)
2. `$HOME/.claude/` (typical for local agent mode, or when the Cowork `.claude` mount is read-only)
3. `<cowork-session>/mnt/outputs/` (last-resort fallback — ephemeral, cleared between sessions; the plugin will warn explicitly if it lands here)

On every run, the same locations are searched in order until the file is found. The plugin tells you the actual path after first-run setup so you can spot if it landed somewhere ephemeral. You can edit the file directly if you need to change your Jira board or Drive folder. See `config/settings.json.example` for the expected format.

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
