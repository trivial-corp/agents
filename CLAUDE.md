# CLAUDE.md

## Releasing

After merging a PR that changes user-facing plugin content (MCP config, skill content, plugin manifest), open a follow-up PR that bumps `version` in both:

- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json` (the `metadata.version` field)

Keep the two numbers in sync. Merging the bump triggers `release.yml`, which tags `vX.Y.Z` and publishes `agents.zip` plus per-skill zips to a GitHub Release. Without the bump, merged changes sit on `main` but never reach marketplace installs or the downloadable zip.

Prefer a dedicated bump PR (easier to revert) over including the version change in the content PR.
