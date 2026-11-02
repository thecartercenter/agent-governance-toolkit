# Claude Code governance baseline

Source of truth for The Carter Center's Claude Code **server-managed settings**.
Applies to every Claude Code user in the organization (terminal, Desktop Code tab, IDE extensions).

## How it is applied

1. Change `managed-settings.json` via pull request. Security review is required before merge.
2. After merge, an Owner pastes the file's contents into
   **claude.ai → Admin Settings → Claude Code → Managed settings** and saves (just-in-time Owner elevation).
3. Record the change in the changelog below. Clients pick it up at their next Claude Code launch.

> The console does not sync from this repo. If the console and this file differ, this file is the intended state.

## Design

| Tier | Effect | Used for |
|---|---|---|
| `deny` | Blocked; no user can approve or override | `sudo`, recursive deletes, force-push / hard reset, secret-file reads, environment dumps, persistence (scheduled tasks, services, Run keys, PowerShell profiles, Startup folder), execution-policy changes, edits to Claude's own config |
| `ask` | User approves each time | Package installs (Python: pip, pip3, `python -m pip`, `py -m pip`, pipx, uv, uvx, poetry, pdm, conda; plus npm, yarn, pnpm, bun, cargo, go, gem, dotnet, winget, choco, scoop), downloads, `WebFetch`, `git push`, ordinary deletes |

Rules cover both the **Bash** and **PowerShell** tools, because on Windows Claude falls back to PowerShell when Bash is denied.
`disableBypassPermissionsMode` prevents users from switching to the mode that skips approval prompts.

## Known limits

- Command rules match **prefixes**: reordered commands (e.g. `git push origin main --force`) are not caught.
- `Read` rules govern Claude's file-reading tool; shell commands such as `cat` rely on the `ask` tier and model judgment.
- Rules govern commands **Claude** runs, not commands a user types into a terminal.
- `**/.env.*` also blocks `.env.example`.
- **PowerShell is case-insensitive; rule matching may not be.** Pending test: ask Claude to run `remove-item -recurse .\testdir`. If it is not denied, lowercase duplicates of the PowerShell rules are needed.
- Versioned Python launchers (e.g. `py -3.12 -m pip install`) and indirect installs (`python -c "import subprocess..."`, scripts that call pip) are not caught.
- `ask` on `git push` and `WebFetch` exists to close data-exfiltration paths used by prompt-injection attacks; injection itself is not detected.
- Do **not** add a `hooks` or similar block requiring user approval: in testing, it prevented the whole settings payload from reaching the Desktop app.

This is a guardrail against accidents, not containment against a determined user.

## Verification (run in a new Code session, via the chat box)

| Ask Claude to… | Expected |
|---|---|
| Run `sudo dir` | Denied |
| Run `Remove-Item -Recurse .\testdir` | Denied |
| Read `~/.ssh/config` | Denied |
| Run `npm install left-pad` | Approval prompt |
| Run `pip install requests` | Approval prompt |
| Run `python -m pip install requests` | Approval prompt |
| Run `Get-ChildItem env:` | Denied |
| Run `dir` | Runs normally |

## Changelog

| Date | Change | Approved by |
|---|---|---|
| 2026-10-22 | Initial baseline | |
| 2026-11-01 | Update 2: env-dump and persistence denies; `ask` for Python and other package managers, `WebFetch`, `git push` | |
