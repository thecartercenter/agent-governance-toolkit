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

**macOS additions** (Bash): Keychain secret reads (`security find-*-password`, `dump-keychain`, `export`; `~/Library/Keychains`), browser credential/cookie stores under `~/Library`, persistence (LaunchAgents/LaunchDaemons, `launchctl load|bootstrap|submit`, `crontab`, shell profiles), Gatekeeper/SIP/TCC bypass (`xattr -d|-c`, `spctl`, `csrutil`, `tccutil reset`), `diskutil erase`, `osascript … administrator privileges` are denied; `brew install|reinstall|tap` and other `osascript` ask. Windows rules are inert on macOS and vice versa.
`disableBypassPermissionsMode` prevents users from switching to the mode that skips approval prompts.

Other managed controls:

| Setting | Effect |
|---|---|
| `allowManagedPermissionRulesOnly: true` | Only these managed rules apply. Rules in user or project settings (including a cloned repo's `.claude/settings.json`) and "Always allow" choices are ignored. |
| `allowManagedMcpServersOnly: true` + `allowedMcpServers: []` | Blocks MCP servers that **users add themselves**. It does not cover plugin-provided servers, Desktop-provided servers, or connectors in the Desktop Code tab (tested); see Companion controls. |
| `env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB: "1"` | Strips credential environment variables from commands Claude runs, hooks, and local MCP servers. |

## Known limits

- Command rules match the text Claude writes. Common flag orders are covered (e.g. `rm x -rf`, `git push origin main --force`), but other forms are not: full paths (`/bin/rm`), `bash -c '…'`, combined flags such as `rm -fvr`. Anthropic documents this as a guardrail, not a boundary.
- In this syntax `:*` means "space, then anything", so `Bash(rm -r:*)` would miss `rm -rf`. Flag rules here use `*` with no space (e.g. `Bash(rm -r*)`).
- `Read`/`Edit` deny rules cover Claude's file tools and file commands Claude Code recognizes in shells (`cat`, `head`, `tail`, `sed`, `tee`, redirects), but not scripts that open files themselves (e.g. a Python script).
- Rules govern commands **Claude** runs, not commands a user types into a terminal.
- `.env.*` rules also block `.env.example`.
- Rule matching is case-insensitive (verified: `remove-item -recurse` is denied like `Remove-Item -Recurse`).
- Rule syntax (per Anthropic docs): `*` matches any text and can appear anywhere; `:*` is only recognized at the very end. PowerShell aliases (`gci`, `ls`, `dir`, `rm`, `iex`) are canonicalized to the cmdlet, so write rules for the cmdlet name. `Read(//**/x)` matches on any drive; `Read(**/x)` only under the working directory.
- Versioned Python launchers (e.g. `py -3.12 -m pip install`) and indirect installs (`python -c "import subprocess..."`, scripts that call pip) are not caught.
- `ask` on `git push` and `WebFetch` exists to close data-exfiltration paths used by prompt-injection attacks; injection itself is not detected.
- Avoid settings that require a [security approval](https://code.claude.com/docs/en/server-managed-settings#security-approval-dialogs) (hooks, shell-command settings, proxy/endpoint variables). The Desktop Code tab can't show the approval dialog, so it applies them for that run only and doesn't cache them; terminal users must approve them, and rejecting exits Claude Code.
- The OS-level sandbox, the strongest Claude Code control, is not available on native Windows. It **is** available on macOS (not yet enabled; pilot first, and never set `sandbox.failIfUnavailable` org-wide or Windows users can't run commands).
- macOS false positives by design: `crontab -l` and any `xattr -d` are denied.
- macOS rules are validated by simulation only; no live macOS test yet.
- Compensating controls on Windows (no sandbox): standard (non-admin) user accounts, Defender attack surface reduction rules, and an internal package proxy.
- Because "Always allow" is ignored, users see an approval prompt every time for `ask` items.

### Tested gaps in the Claude Desktop Code tab (Windows)

| Control | Result | Compensating control |
|---|---|---|
| `allowManagedMcpServersOnly` + `allowedMcpServers: []` | **Not enforced.** Code sessions loaded connectors, Nonprofits-plugin servers, Claude in Chrome, and scheduled tasks. | Restrict at the source (claude.ai organization settings, below); permission `deny` rules on MCP servers as a backstop |
| `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` | **Not effective** for `CLAUDE_CODE_*` variables | `deny` rules on environment dumps |
| Managed plugin auto-install (`enabledPlugins`) | **Not performed** | n/a (AGT parked) |

Permission rules (`deny`/`ask` for Bash, PowerShell, Read, Edit) and `disableBypassPermissionsMode` **are** enforced in the Desktop Code tab.

## Companion controls (claude.ai organization settings)

Claude Code sessions inherit tools from sources these managed settings do not control. Configure them in claude.ai **Organization settings**:

| Area | Setting | Baseline |
|---|---|---|
| Connectors | Enabled connectors; per-connector action limits | Only approved connectors; read-only where possible. Admin "+" on the Discover page enables a connector **for the whole organization**. |
| Connector requests | Member requests | Off |
| Plugins & skills | Organization plugins (they can bundle their own MCP servers) | Only approved plugins |
| Desktop extensions | Allowlist | On (off by default, which allows any extension) |
| Claude in Chrome | Availability | Disabled unless approved |

This is a guardrail against accidents, not containment against a determined user.

## Pre-deployment checks

`tests/` contains a model of Claude Code's documented matching rules and 179 cases (Windows and macOS) (intended deny / ask / none, including false-positive guards). Run before every change:

```
python tests/run.py managed-settings.json
```

The model is built from Anthropic's permissions documentation, not from the product, so it catches rule-writing mistakes; it can't prove product behavior. Confirm product behavior with the live checks below.

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
| Run `($env:CLAUDE_CODE_MESSAGING_TOKEN).Length` | Known gap in the Desktop Code tab: returns a length (not scrubbed). Prints only a length, never the value. |
| Run `dir` | Runs normally |

## Changelog

| Date | Change | Approved by |
|---|---|---|
| 2026-10-21 | Initial baseline | JH |
| 2026-11-01 | Second update: env-dump and persistence denies; `ask` for Python and other package managers, `WebFetch`, `git push` | JH |
| 2026-11-02 | Third update: more credential paths (GitHub CLI, gcloud, pip, netrc, browser profiles, `.p12`, `.tfstate`), git hook edit deny; env-dump rule syntax fix | JH |
| 2026-11-02 | Fourth update: managed-only permission rules; MCP servers blocked unless allowlisted; subprocess credential scrubbing | JH |
| 2026-11-02 | Fifth update (doc review + simulated testing): cmdlet-canonical PowerShell rules; flag rules rewritten so they match combined flags (`rm -rf`, `--force-with-lease`, `git clean -fd`); credential globs anchored to all drives; `~/.claude` edit rule narrowed to config files (keeps Plan mode and memory working) plus `~/.claude.json`; elevation and `find -delete` denies; `tests/` added |  |
| 2026-11-02 | Sixth update: macOS coverage (Keychain, browser stores, LaunchAgents/cron/shell profiles, Gatekeeper/SIP/TCC bypass, `brew`, `osascript`); 179 simulated cases |  |
