# FilmCraft skill for Claude

FilmCraft (open source, pure Rust, storytold/filmcraft) is a non-linear video editor modelled on Adobe Premiere Pro: same panels, workspaces, tools and shortcuts. It is **alpha** software (Windows and Linux are less tested than macOS).

This repository packages a **skill** that teaches Claude to drive FilmCraft (part of the [ArtCraft suite](https://github.com/storytold/filmcraft) by storytold) and that **knows how to create the MCP server and guide you through it**, on **Windows and macOS**. It is unofficial and not affiliated with storytold.

> Part of the [ArtCraft Claude skills hub](https://github.com/Vinciux99/artcraft-claude-skills): all 13 skills, a picker and a one-command installer.

## What it does
- Operating manual for FilmCraft: connection modes, ground rules, every MCP tool, recipes, CLI, control channel, command families and a safe workflow (work on copies, verify, save only with your OK).
- Built-in MCP setup assistant: Claude checks whether the server is connected, then walks you through installing it (automatic script or manual config), the launcher, authentication and verification.
- Troubleshooting table for the usual problems (server "disconnected", ports, macOS quarantine, Windows Store config location...).

## What you can ask Claude
- "Import these three clips, put them on V1 back to back and cut the first one at 3.5 s."
- "Add a lower-third title 'Jane Doe — Director' on V2 for five seconds starting at 00:12."
- "Import this .srt as a caption track and burn the captions into the export."
- "Apply a Gaussian Blur to clip 87 and bend the luma curve for more contrast."
- "Export the sequence as H.264 to my Exports folder and tell me if any media is offline."
- "Which of my project media files are missing? Relink them from this folder."
- "Remove filler words and long pauses using the transcript, if this build supports it."
- "Save a copy as `edit-v2.fcproj` and render a frame at 4 s so I can check the grade."

## Highlights
- Headless vs live (bridge) MCP, with all 17 tools and when each works.
- Tick-based time model, explicit-target rules (no reliance on selection), offline-media handling.
- Recipes for import, place/razor/move, effects, Lumetri, titles, captions, export and interchange.
- CLI and JSONL script reference plus the full control-channel protocol.
- Built-in guidance for creating and troubleshooting the MCP server on Windows and macOS.

## App-specific notes
- Kit port **9876** (same as upstream docs). Live MCP: `mcp --bridge 127.0.0.1:9876`; headless: `mcp`.
- The control channel has no authentication and listens on loopback only; no read/write roots exist.
- Alpha software: H.265 export needs a hardware encoder; no AV1; Premiere `.prproj` cannot be opened directly.

## Get the skill

| Where | How |
|---|---|
| **Claude app** (claude.ai / Desktop) | Download [`filmcraft-skill.zip](https://github.com/Vinciux99/filmcraft-claude-skill/releases/latest/download/filmcraft-skill.zip)`, then Customize > Skills > "+" > Create skill > Upload a skill. Needs Settings > Capabilities > "Code execution and file creation" on. |
| **Claude Code, plugin** | `/plugin marketplace add Vinciux99/artcraft-claude-skills` then `/plugin install filmcraft@artcraft` |
| **Claude Code, manual** | copy `skills/filmcraft/` to `~/.claude/skills/filmcraft/` (the setup script below does it) |

## Create the MCP server

Prerequisite: install FilmCraft from [upstream releases](https://github.com/storytold/filmcraft/releases) (Windows `.msi`, macOS `.dmg`) and Claude Desktop and/or Claude Code.

**One command (recommended)**

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps filmcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps filmcraft
```
Add `-DryRun` (Windows) or `--dry-run` (macOS) to preview, and `-Mode live|headless|both` / `--mode ...` to choose the mode.

**Or from a clone:** double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time right-click > Open).

**Or just ask Claude:** once the skill is installed, say "set up the FilmCraft MCP server" and Claude will guide you step by step.

After setup: fully quit and reopen Claude, open FilmCraft with the **"FilmCraft (Claude)"** launcher, then ask Claude to use it. If the server shows "disconnected", the app was not running when Claude started: open it with the launcher and re-enable the server, or use headless mode.

What the setup changes (all backed up or reversible): Claude Desktop config (`mcpServers.filmcraft`, backup `<file>.backup-<timestamp>`), the launcher, Claude Code's user-scope MCP list and `~/.claude/skills/filmcraft/`.

## Repository layout
```
skills/filmcraft/SKILL.md      the skill
app.json                   machine-readable settings (ports, paths, MCP args) used by the scripts
scripts/                   artcraft-setup.ps1 (Windows), artcraft-setup.sh (macOS), json-tool.js
setup-windows.cmd          double-click wrapper
setup-macos.command        double-click wrapper
.claude-plugin/plugin.json Claude Code plugin manifest
.github/workflows/ci.yml   validation, setup tests on Windows and macOS, release zip
```

## CI and releases
Every push validates `app.json`, `SKILL.md` and the scripts, then runs the setup scripts against a fake install on **Windows** and **macOS** runners. Pushing a tag `v*` builds `filmcraft-skill.zip` and publishes a GitHub release.

## Status and caveats
- Windows paths and arguments come from a working install. The macOS paths (`/Applications/FilmCraft.app`, CLI location and release asset name) follow the upstream packaging but have **not** been verified on a Mac; the skill marks unverified items and the setup script reports what it could not find.
- FilmCraft is a young open-source project; behaviour can change between releases. `list_commands` / `filmcraft-cli commands` is always the source of truth.
- Unofficial community packaging, MIT licensed (see `LICENSE`).
