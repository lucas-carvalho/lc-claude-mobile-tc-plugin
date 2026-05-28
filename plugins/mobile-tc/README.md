# lc-claude-mobile-test-cases-handler

A Cowork plugin that generates formatted QA test case spreadsheets from Jira tickets, saving them directly to a Google Drive folder of your choice.

Built for mobile app QA workflows, but adaptable to any Jira project and Drive folder.

## What It Does

Given a Jira ticket ID or URL, the plugin:

1. **Configures itself on first use** — asks for your Jira board and Drive folder (once only)
2. **Fetches the ticket** from Jira via the Atlassian MCP
3. **Classifies the ticket type** (Functional, Integration, Component, or Bug)
4. **Generates test cases** organized into thematic Blocks, from a functional tester's perspective
5. **Exports a formatted `.xlsx`** with color-coded blocks, Gherkin-style descriptions, and status dropdowns
6. **Saves it to your Google Drive folder** via the Drive MCP

## First-Time Setup

On the first run, the plugin will ask you two questions:

1. **Your Jira board URL** — e.g. `https://yourorg.atlassian.net/jira/software/c/projects/MYPROJECT/boards/123`
2. **Your Google Drive output folder** — the URL or folder ID where spreadsheets should be saved

Your answers are saved to the persistent Cowork directory (`mnt/.claude/lc-mobile-qa-settings.json`) and reused automatically on every subsequent run — including across new sessions.

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

- 8 columns: ✓ Passed / ID / Block / Scenario Title / Description (Given / When / Then) / Notes / Status / Automatable?
- Color-coded thematic blocks — no emoji prefixes; color applied via block header and row tinting
- Column A: `☐` symbol (Passed?) — Google Sheets compatible (no TRUE/FALSE dropdown)
- Column G: Status dropdown — Not tested / Passed / Failed / Blocked / N/A
- Column H: Automatable? dropdown — Yes / No / -
- Block Summary tab with TC count per block
- Mock checklist block at the end (for dev confirmation)
- AC coverage summary section (tracks which Acceptance Criteria are covered)
- **All spreadsheet content is in English** — structural elements and TC content alike

## Quality Approach (Shift Left)

The plugin surfaces quality signals before generating TCs:

- Flags missing or vague Acceptance Criteria
- Flags UI tickets with no linked design
- Tracks AC coverage — reports any ACs without a corresponding TC
- Marks each TC as automatable (Yes / No / -) to support CI pipeline prioritization

## Requirements

- **Claude Code** with **Cowork** support — available on **Team** and **Enterprise** plans only
- **Atlassian MCP** connected and authenticated (for Jira access)
- **Google Drive MCP** connected and authenticated (for saving the spreadsheet)
- Python 3 with `pip` available in the execution environment (`openpyxl` is installed automatically if needed)

> **Note:** This plugin will not work on Free or Pro plans. It depends on the Cowork plugin system and persistent session storage, which are exclusive to Team and Enterprise plans.

## Configuration File

After first-time setup, your settings are stored at:
`mnt/.claude/lc-mobile-qa-settings.json`

This file lives in the persistent Cowork directory and survives across sessions — the plugin locates it dynamically regardless of the session name. You can edit it directly if you need to change your Jira board or Drive folder. See `config/settings.json.example` for the expected format.

## Author

Lucas Carvalho Silva — Telus Digital's Test Engineer
