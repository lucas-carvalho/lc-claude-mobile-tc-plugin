# Changelog

## [0.6.2] — current
- **Automated GitHub Releases via GitHub Actions.** A `.github/workflows/release.yml` workflow now triggers on every `v*` tag push, extracts the matching CHANGELOG section, and publishes the GitHub Release automatically — no manual steps after `git push --tags`.
- **README: new "Releasing a New Version" section** documenting the tag-push → Actions → Release flow for maintainers.
- **CHANGELOG: release instructions updated** — replaced the stale "re-upload via Claude desktop" step with the new automated flow.

## [0.6.1]
- **Fix: Native Google Sheet now actually lands in the configured Drive folder.** v0.6.0 prescribed a single converting `create_file` call, but the Drive MCP doesn't auto-convert XLSX (only `text/csv` and `text/plain`), so the file ended up as XLSX. v0.6.1 routes the upload through Claude in Chrome and the Drive web UI, leveraging the account-level "Convert uploads" setting that converts XLSX → GSheet server-side on upload. The conversion now happens in-place in the target folder, with no Drive-root duplicate. **Requires the user to enable "Convert uploads to Google Docs editor format" at `drive.google.com/drive/settings`.**
- **New Path B fallback.** When Claude in Chrome is unavailable, the skill falls back to a Drive-MCP-only path: uploads XLSX to the configured folder (no conversion), and surfaces an explicit manual-conversion instruction in Step 7. Honest about the limitation; no false success.
- **Stale-duplicate scan now runs on every successful upload**, not just on user prompt. Finds older-version artifacts (Google Sheets at Drive root for this ticket) and lists them with URLs so the user can clean up manually.
- **Fix: `lc-mobile-qa-settings.json` persistence.** The previous discovery picked the first `*/mnt/.claude` directory without checking writability, silently failing in sessions where that mount is read-only. v0.6.1 iterates writable candidates (`mnt/.claude` → `$HOME/.claude` → ephemeral `mnt/outputs`), reports the chosen path, and warns explicitly if it had to fall back to ephemeral storage.
- **README updated** to reflect the new upload flow, the Chrome in Claude + Drive setting prerequisites, and the multi-location settings persistence.

## [0.6.0]
- **Fix: Google Sheet now always lands in the configured Drive folder.** Step 6 rewritten to use a single converting `create_file` call with `parentId` and `disableConversionToGoogleType: false` set explicitly. Eliminates the two-call pattern that historically dropped a duplicate Google Sheet at Drive root.
- **New Step 6.5: mandatory post-upload verification.** Validates `mimeType == application/vnd.google-apps.spreadsheet`, `parents` contains the configured folder, name has no `.xlsx` suffix, and (optionally) the `"Test Cases"` / `"Block Summary"` sheet names exist. If anything fails, the skill stops and presents the local `.xlsx` as a fallback instead of falsely reporting success.
- **New Step 6.6: stale-duplicate scan + delete limitation surfaced.** Adds search queries for previously-orphaned XLSX/GSheet files from older runs. Documents that the Drive MCP exposes no `delete_file` / `trash_file` tool, so deletion must be done manually via the Drive UI.

## [0.5.1]
- Initial local snapshot extracted from installed plugin

## How to bump the version
1. Edit `version` in `plugins/mobile-tc/.claude-plugin/plugin.json`
2. Edit `metadata.version` in `plugins/mobile-tc/skills/generate-test-cases/SKILL.md`
3. Add an entry here
4. Commit, tag, and push — GitHub Actions creates the release automatically:
   ```bash
   git add -p
   git commit -m "chore: bump version to X.Y.Z"
   git tag vX.Y.Z
   git push && git push --tags
   ```
