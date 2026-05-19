# Changelog

<!-- CHANGELOG_ENTRIES_START -->

## 2026-05-19 — [PR #3](https://github.com/rahul-aut-ind/ai-hackathon-2026/pull/3)

### Fixed
- Fixed an issue (AH-003) where GitHub Actions workflows failed to push directly to the `main` branch due to repository rulesets. The workflow was updated to create a temporary branch, generate a pull request, and auto-merge to resolve these permission conflicts.

