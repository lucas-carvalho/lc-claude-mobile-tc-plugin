# Changelog

## [0.6.0] — current
- **Fix: Google Sheet now always lands in the configured Drive folder.** Step 6 rewritten to use a single converting `create_file` call with `parentId` and `disableConversionToGoogleType: false` set explicitly. Eliminates the two-call pattern that historically dropped a duplicate Google Sheet at Drive root.
- **New Step 6.5: mandatory post-upload verification.** Validates `mimeType == application/vnd.google-apps.spreadsheet`, `parents` contains the configured folder, name has no `.xlsx` suffix, and (optionally) the `"Test Cases"` / `"Block Summary"` sheet names exist. If anything fails, the skill stops and presents the local `.xlsx` as a fallback instead of falsely reporting success.
- **New Step 6.6: stale-duplicate scan + delete limitation surfaced.** Adds search queries for previously-orphaned XLSX/GSheet files from older runs. Documents that the Drive MCP exposes no `delete_file` / `trash_file` tool, so deletion must be done manually via the Drive UI.

## [0.5.1]
- Initial local snapshot extracted from installed plugin

## How to bump the version
1. Edit `version` in `.claude-plugin/plugin.json`
2. Edit `metadata.version` in `skills/generate-test-cases/SKILL.md`
3. Add an entry here
4. Re-upload via Claude desktop: Settings → Plugins → Mobile TC by LC → Edit
