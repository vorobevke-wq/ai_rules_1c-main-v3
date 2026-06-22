# OpenSpec bundle

Static snapshot of OpenSpec commands and skills used by `install.ps1`.

Important for Kilo Code: the bundle intentionally uses `.kilo/commands/` and `.kilo/skills/`. When checking or copying files from this directory, include dot-prefixed directories (`Get-ChildItem -Force`, `rg --hidden --files`, etc.); otherwise the Kilo bundle may look empty even though the files are present.
