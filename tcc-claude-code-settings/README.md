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
| `deny` | Blocked; no user can approve or override | `sudo`, recursive deletes, force-push / hard reset, secret-file reads (SSH, cloud CLIs, GitHub CLI, package-registry and browser credential stores), environment dumps, persistence (scheduled tasks, services, Run keys, PowerShell profiles, Startup folder), execution-policy changes, git hook edits, edits to Claude's own config |
| `ask` | User approves each time | Package installs (Python: pip, pip3, `python -m pip`, `py -m pip`, pipx, uv, uvx, poetry, pdm, conda; plus npm, yarn, pnpm, bun, cargo, go, gem, dotnet, winget, choco, scoop), downloads, `WebFetch`, `git push`, ordinary deletes |

Rules cover both the **Bash** and **PowerShell** tools, because on Windows Claude falls back to PowerShell when Bash is denied.
`disableBypassPermissionsMode` prevents users from switching to the mode that skips approval prompts.

Other managed controls:

| Setting | Effect |
|---|---|
| `allowManagedPermissionRulesOnly: true` | Only these managed rules apply. Rules in user or project settings (including a cloned repo's `.claude/settings.json`) and "Always allow" choices are ignored. |
| `allowManagedMcpServersOnly: true` + `allowedMcpServers: []` | No MCP servers load in Claude Code, including user-added servers and claude.ai connectors. Approve a server by adding an entry to `allowedMcpServers`. Chat on claude.ai is unaffected. |
| `env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB: "1"` | Strips credential environment variables from commands Claude runs, hooks, and local MCP servers. |

## Known limits

- Command rules match **prefixes**: reordered commands (e.g. `git push origin main --force`) are not caught.
- `Read` rules govern Claude's file-reading tool; shell commands such as `cat` rely on the `ask` tier and model judgment.
- Rules govern commands **Claude** runs, not commands a user types into a terminal.
- `**/.env.*` also blocks `.env.example`.
- Rule matching is case-insensitive (verified: `remove-item -recurse` is denied like `Remove-Item -Recurse`).
- In rule syntax, a trailing `:*` means "prefix". A command that itself ends in a colon (e.g. `Get-ChildItem env:`) needs both an exact rule (`env:`) and a prefix rule (`env::*`).
- Versioned Python launchers (e.g. `py -3.12 -m pip install`) and indirect installs (`python -c "import subprocess..."`, scripts that call pip) are not caught.
- `ask` on `git push` and `WebFetch` exists to close data-exfiltration paths used by prompt-injection attacks; injection itself is not detected.
- Avoid settings that require a [security approval](https://code.claude.com/docs/en/server-managed-settings#security-approval-dialogs) (hooks, shell-command settings, proxy/endpoint variables). The Desktop Code tab can't show the approval dialog, so it applies them for that run only and doesn't cache them; terminal users must approve them, and rejecting exits Claude Code.
- The OS-level sandbox, the strongest Claude Code control, is not available on native Windows. Compensating controls: standard (non-admin) user accounts, Defender attack surface reduction rules, and an internal package proxy.
- Because "Always allow" is ignored, users see an approval prompt every time for `ask` items.

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
| Run `/permissions` | Shows these rules as managed |
| Run `($env:CLAUDE_CODE_MESSAGING_TOKEN).Length` | `0` or empty (variable scrubbed). Prints only a length, never the value. |
| Run `dir` | Runs normally |

## Changelog

| Date | Change | Approved by |
|---|---|---|
| 2026-10-21 | Initial baseline | JH |
| 2026-11-01 | Second update: env-dump and persistence denies; `ask` for Python and other package managers, `WebFetch`, `git push` | JH |
| 2026-11-02 | Third update: more credential paths (GitHub CLI, gcloud, pip, netrc, browser profiles, `.p12`, `.tfstate`), git hook edit deny; env-dump rule syntax fix | JH |
| 2026-11-02 | Fourth update: managed-only permission rules; MCP servers blocked unless allowlisted; subprocess credential scrubbing | JH |
