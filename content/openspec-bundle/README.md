# OpenSpec bundle

Static snapshot of OpenSpec commands and skills used by `install.ps1`.

Cursor, Codex, OpenCode and Kilo Code bundles normalize OpenSpec skills to
`.agents/skills/`; Claude Code keeps `.claude/skills/`. Tool-specific commands
remain under the owning tool directory. When checking or copying files from
this directory, include dot-prefixed directories (`Get-ChildItem -Force`,
`rg --hidden --files`, etc.); otherwise a bundle may look empty even though
the files are present.

DeepSeek Harness has no bundle in this snapshot. Its adapter still installs the
shared `openspec/` workspace and the `sdd-integrations.md` rule; OpenSpec work is
requested through the regular Harness conversation until a native bundle is
added deliberately.
