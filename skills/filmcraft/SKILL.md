---
name: filmcraft
description: Edit video projects with FilmCraft (Premiere Pro-style, ArtCraft suite) through MCP, the CLI or the control channel, and set up, connect or troubleshoot its MCP server on Windows and macOS.
---

# FilmCraft for Claude

FilmCraft (open source, pure Rust, storytold/filmcraft) is a non-linear video editor modelled on Adobe Premiere Pro: same panels, workspaces, tools and shortcuts. It is **alpha** software (Windows and Linux are less tested than macOS).

**Every action is a command** with a stable id (`timeline.razor`, `file.exportMedia`, ...), typed JSON parameters and an "enabled" state. The UI, CLI, control channel and MCP server all run the same 650+ engine commands (`filmcraft-cli commands` lists them). Projects are `.fcproj` files (UTF-8 JSON with a schema version; media is referenced by path, never embedded).

If something here disagrees with the installed version, trust `filmcraft-cli --help` and `filmcraft-cli describe <id>`.

## Quick check: is the MCP server connected?

1. Look at the tools you have. If tools from the MCP server `filmcraft` are available (for example `list_commands`, `execute` or `inspect_*` tools) the server is connected: skip to "Operating knowledge" below and use them directly.
2. If they are missing, check whether the user installed it under the name `filmcraft` or `filmcraft-headless`. In Claude Desktop the server may show as **disconnected**.
3. **Most common cause of "disconnected":** the live server (`mcp --bridge 127.0.0.1:{port}`) exits when FilmCraft is not running at the moment Claude starts. Fix: open FilmCraft with the **"FilmCraft (Claude)" launcher** (it starts the app with its control channel on), then re-enable the server in Claude's settings or fully quit and reopen Claude.
4. If the user does not need to watch the app live, **headless mode** always starts (no window, works on files). See "Live, headless or both".
5. If nothing is configured yet, follow "Set up the MCP server" below. Ask the user which OS they use if you do not know.

## Set up the MCP server (guide the user)

Walk the user through this step by step. Ask for the OS (Windows or macOS) first. If you have a shell on the user's machine you may run the steps yourself, but **ask before editing any config file and always back it up first**. Never print tokens or keys in chat.

### 1. Prerequisites
- Install FilmCraft from its upstream releases page: https://github.com/storytold/filmcraft/releases (Windows: the `.msi` installer; macOS: the `.dmg`, then drag the app to `/Applications`).
- Install Claude Desktop and/or Claude Code. For Claude Desktop, fully quit and reopen it after any config change.
- The MCP server is the **CLI** that ships with the app: `filmcraft-cli`. Windows installs it next to the app (`C:\Program Files\FilmCraft\filmcraft-cli.exe`, not on `PATH`). On macOS the CLI is a separate release download (asset `filmcraft-cli-<version>-macos-universal.zip`) unless it is bundled in the app (unverified): the setup script below fetches it into `~/.artcraft/bin/` and clears the macOS quarantine flag.
- This kit uses port **9876** (the upstream default), which does not clash with the other apps of the kit.

### 2. Option A: automatic setup (recommended)
The setup script finds the app and CLI, creates the launcher, backs up and updates Claude Desktop's config, registers the server in Claude Code (if the `claude` command exists) and installs this skill. Preview first with a dry run: nothing is changed.

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps filmcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps filmcraft
```
Or clone/download this repository and double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time: right-click, Open). Useful options: `-Mode live|headless|both` / `--mode ...` (default `live`; `both` registers `filmcraft` live and `filmcraft-headless`), `-DryRun` / `--dry-run`, `-NoLauncher`, `-NoClaudeDesktop`, `-NoClaudeCode`, `-NoSkill` (macOS: `--no-launcher`, `--no-claude-desktop`, `--no-claude-code`, `--no-skill`).

What it changes: Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json` plus the Microsoft Store/MSIX copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`; backup saved next to it as `<file>.backup-<timestamp>`), the launcher "FilmCraft (Claude)" (Windows: Desktop shortcut; macOS: `~/Applications/FilmCraft (Claude).app` plus a Desktop alias), Claude Code's user-scope MCP list, and `~/.claude/skills/filmcraft/`.

### 3. Option B: manual setup
**Windows** - edit `%APPDATA%\Claude\claude_desktop_config.json` (if Claude was installed from the Microsoft Store, also the copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`). Keep other servers; add this inside `mcpServers`. Backslashes must be doubled in JSON.
```json
{
  "mcpServers": {
    "filmcraft": {
      "command": "C:\\Program Files\\FilmCraft\\filmcraft-cli.exe",
      "args": [
        "mcp",
        "--bridge",
        "127.0.0.1:9876"
      ]
    },
    "filmcraft-headless": {
      "command": "C:\\Program Files\\FilmCraft\\filmcraft-cli.exe",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**macOS** - edit `~/Library/Application Support/Claude/claude_desktop_config.json`. Use the absolute CLI path (replace `YOUR_NAME`; JSON does not expand `~`). The CLI path is unverified if your install differs: check with `ls ~/.artcraft/bin /Applications/FilmCraft.app/Contents/MacOS`.
```json
{
  "mcpServers": {
    "filmcraft": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/filmcraft-cli",
      "args": [
        "mcp",
        "--bridge",
        "127.0.0.1:9876"
      ]
    },
    "filmcraft-headless": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/filmcraft-cli",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**Launcher** (starts the app with the control channel; needed before the live server can connect):
- Windows: create a Desktop shortcut named "FilmCraft (Claude)" with target `"C:\Program Files\FilmCraft\filmcraft.exe" --control 9876`
- macOS: run `open -n -a "/Applications/FilmCraft.app" --args --control 9876`, or save it as an app with Script Editor/`osacompile` (the setup script does this for you).

**Claude Code** (any OS; run once):
Windows (PowerShell):
```powershell
claude mcp add --scope user filmcraft -- "C:\Program Files\FilmCraft\filmcraft-cli.exe" mcp --bridge 127.0.0.1:9876
claude mcp add --scope user filmcraft-headless -- "C:\Program Files\FilmCraft\filmcraft-cli.exe" mcp
```
macOS (Terminal):
```sh
claude mcp add --scope user filmcraft -- "$HOME/.artcraft/bin/filmcraft-cli" mcp --bridge 127.0.0.1:9876
claude mcp add --scope user filmcraft-headless -- "$HOME/.artcraft/bin/filmcraft-cli" mcp
```
Check with `claude mcp list`.

### 4. Authentication
FilmCraft needs no token: the control channel listens on loopback (`127.0.0.1`) only and has no authentication. Anyone on the machine can drive the app while it is open, so close it when you are done.

### 5. Finish and verify
1. **Fully quit and reopen Claude** (Desktop: quit from the tray/menu bar, not just close the window).
2. Open FilmCraft with the **"FilmCraft (Claude)" launcher** before asking Claude to use it.
3. In Claude, confirm the server `filmcraft` is connected and call its `list_commands` tool (or equivalent). A non-empty command list means it works.

### Live, headless or both
- **Live** (`filmcraft`): drives the open app; the user sees every change and screenshots/UI tools work. Needs the launcher first.
- **Headless** (`filmcraft-headless`): in-process engine, no window; always starts; works on files; UI/screenshot tools are not available.
- **Both**: register both and pick per task. Switch with `-Mode` / `--mode`, or by editing the config entries above.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server shows "disconnected" right after Claude starts | Live server exits when FilmCraft is not running | Open FilmCraft with the "FilmCraft (Claude)" launcher, then re-enable the server or restart Claude; or use headless mode |
| "Port already in use" / app opens but Claude cannot connect | Another app or an old FilmCraft instance holds port 9876 | Quit all FilmCraft windows, check the launcher uses the kit port, make sure no other ArtCraft app uses the same port |
| `command not found` / spawn error | CLI path wrong or the CLI is not installed (Windows: not on `PATH`; macOS: separate download) | Use the full absolute path in the config; macOS: run the setup script (it downloads the CLI to `~/.artcraft/bin/`) |
| macOS: "cannot be opened because the developer cannot be verified" | Gatekeeper/quarantine on the app or CLI | Right-click the app, Open; for the CLI: `xattr -dr com.apple.quarantine ~/.artcraft/bin/filmcraft-cli` |
| Anyone else can drive the app | The control channel has no authentication (loopback only) | Start the app with the control channel only when needed and quit it when done |
| Config edited but nothing changed (Windows) | Claude installed from the Microsoft Store reads the MSIX copy of the config | Edit/update `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json` too |
| JSON config error / server not listed | Unescaped backslashes or trailing comma | Double the backslashes on Windows; validate the JSON; restore the `.backup-*` file if needed |
| App crashes on start or window is blank | Graphics backend problem | Windows: set `WGPU_BACKEND=vulkan` (or `dx12`) for one launch; avoid `gl` on some AMD drivers |
| Changes do not appear in the window | You are on the headless server | Use the live server (launcher first) |
| Quotes disappear in PowerShell JSON arguments | Windows PowerShell 5.1 quoting for `.exe` | Use `--script`/files where the CLI supports them, or single-quote the whole JSON |

## Operating knowledge

## Connection modes

| Mode | Start | What it drives |
|---|---|---|
| MCP headless | `filmcraft-cli mcp` (empty project), `--demo`, or `--project p.fcproj`; optional `--data-dir DIR` | An in-process engine session, no window |
| MCP bridge | `filmcraft-cli mcp --bridge 127.0.0.1:9876` | The desktop app already open with `filmcraft --control 9876` |
| CLI | `filmcraft-cli [options] <subcommand>` (also `--bridge 127.0.0.1:9876` for the live app) | One command per call, JSON output |
| Control channel | `filmcraft --control 9876` (or `FILMCRAFT_CONTROL_PORT=9876`) | JSON lines over TCP `127.0.0.1:9876`: commands, synthetic UI input, screenshots |

- **Port:** 9876 is the port used throughout the upstream docs (there is no implicit default; pass `--control`). Loopback only.
- **Authentication: none.** Anyone on the machine who can open the port controls the app. Start `--control` only when needed and close the app when done.
- **No read/write roots.** The headless MCP server only accepts `--demo`, `--project`, `--data-dir`; any other option (e.g. `--automation-read-root`) is rejected with exit 2. The server reads and writes wherever the user can, so choose paths carefully.
- **Input only from control:** `--control <port> --control-input-only` accepts widget input only from control requests and excludes native keyboard/mouse, file drops and native menus (useful for unattended sessions).
- **Focus:** driving the app does not steal the user's keyboard; only `ui.focus {}` activates the window.

### App flags, data folder, environment
- Desktop flags: `--control <port>`, `--control-input-only`, `--empty` (no demo project), `--recover` / `--no-recover`, `--data-dir <dir>`, `--version`; files as positional args (`filmcraft a.mp4 b.wav` imports; a `.fcproj` opens).
- Data folder (preferences, auto-save without a project, recovery, logs, presets): Windows `%APPDATA%\FilmCraft`; macOS `~/Library/Application Support/FilmCraft` (unverified); override with `--data-dir` or `FILMCRAFT_DATA_DIR`. A `portable.txt` next to the executable keeps data in `data\` beside it (portable zip).
- Log: `<data>/Logs/filmcraft.log` (previous run → `filmcraft.1.log`). Attach it to bug reports; Help ▸ Reveal Log Files opens the folder.

| Variable | Effect |
|---|---|
| `FILMCRAFT_CONTROL_PORT` | Same as `--control <port>` |
| `FILMCRAFT_DATA_DIR` | Same as `--data-dir <dir>` |
| `FILMCRAFT_CPU_COMPOSITE=1` | Disable the GPU compositor |
| `WGPU_BACKEND` | Force the graphics backend (`dx12`, `vulkan`, `metal`); on Windows the default is Vulkan + DX12. Avoid `gl` (can crash AMD drivers) |
| `RUST_LOG` | Log levels (stderr + log file) |

If the app opens and immediately closes on Windows, try `$env:WGPU_BACKEND = "vulkan"; & "C:\Program Files\FilmCraft\filmcraft.exe"` and then `Remove-Item Env:WGPU_BACKEND`.

### Choosing the channel
1. **A FilmCraft MCP server is available** → use it (tools below). Most precise, with progress and cancellation for exports.
2. **A shell on the user's machine** → call `filmcraft-cli` with its full path; for the open app use `--bridge 127.0.0.1:9876 exec ...`.
3. **A cloud-sandbox shell** is a different machine: it cannot run the user's app or reach their localhost ports. It can only prepare files (`.jsonl` scripts for `filmcraft-cli run`, `.srt`/`.vtt` subtitles, media lists).
4. **Last resort: computer use on the GUI.** Non-US keyboard layouts may break shortcuts that use `[` `]`; prefer menus or command ids. In the docs `Cmd` means `Ctrl` on Windows/Linux.

## Ground rules

- **Time:** the internal unit is the tick, **254,016,000,000 ticks per second** (divides every common frame and sample rate exactly). Commands also accept `seconds`, `frame` or `timecode` (converted once). Do edit math in ticks, not float seconds. `sequence_inspect` reports `start`/`end`/`sourceIn`/`sourceOut` in ticks and frames.
- **Frame grid:** in `timeline.place`, clip start and end snap to the nearest frame. `durationFrames` counts a trailing partial frame.
- **References:** timeline clips, project items, effects are **numeric ids** from `project_inspect` / `sequence_inspect`. Tracks: id or name `"V1"`, `"A2"`; caption tracks `"C1"`, `"C2"`. A non-existent track is an "invalid parameters" error, never another track. Omitting `track` uses the command's default track.
- **Selection vs explicit targets:** commands acting on the selection are disabled ("no clips selected") until something is selected, **unless** you pass targets in the documented keys: `clips`/`clip` (timeline clips), `items`/`item` (project items). E.g. `clip.replaceFromBin {"clips":[12],"item":4}` works without a selection and does not change it. `command_list` reports enablement against the real selection.
- **Linked selection:** `timeline.move` also moves linked partners (video+audio); pass `"linked": false` to move only the listed clips.
- **Undo:** every change is one undo step (`edit.undo` / `edit.redo`); interchange imports are a single step.
- **Strict arguments (MCP):** an unknown argument is JSON-RPC error `-32602` listing the accepted ones. `command_run.params` is passed through unchanged. A failing command returns an `isError` result with the engine message; a panic becomes `internal error: ...` and the session stays usable.
- **Batch:** `command_batch {steps:[{id, params}], stop_on_error}` → `{completed, failed, results}`. Step-result references like `$N` are **not** documented: if you need an id created by a step, make separate calls and re-read with `sequence_inspect`.
- **Long jobs:** `file.exportMedia` without `wait` returns `{job, path}` immediately; poll `jobs.list` (has `etaSeconds`), stop with `jobs.cancel`. With `"wait": true` it blocks until the file is written (headless and bridge) and emits `notifications/progress` when the call carries `"_meta": {"progressToken": T}`; `notifications/cancelled` aborts and deletes partial output.
- **Offline media** renders as a "media not found" placeholder; export reports it in `offlineMedia` (`[{item, name, path, status, error}]`) and the CLI exits 1 unless `--allow-offline`. Check first with `media.status` or `media.findMissing`.
- **Transcription** (`transcript.generate`) is optional and often missing ("speech-to-text is not available in this build"); it runs synchronously, no job.
- **Known limits:** Premiere `.prproj` cannot be opened (go through FCP7 XML, AAF or OTIO); no VST3/AU/OpenFX plug-ins; H.265 export needs a hardware encoder (NVENC on Windows); no AV1 export; Lumetri, keying and export still run on CPU.

## MCP tools

17 tools; the first 9 work in both modes, the `ui_*` ones only in bridge mode. Each has a title and read-only/destructive/idempotent/open-world hints.

| Tool | Mode | Arguments / notes |
|---|---|---|
| `command_list` | both | `filter`, `enabled_only`: id, label, menu, shortcut, params, enabled now |
| `command_run` | both | `{id, params}`; changes are undoable |
| `command_batch` | both | `{steps:[{id, params}], stop_on_error}` → `{completed, failed, results}` |
| `doc_inspect` | both | Project tree + active sequence in one call |
| `render_preview` | both | Same as `render_frame` |
| `project_inspect` | both | Bins and items with ids, types, durations; active sequence |
| `sequence_inspect` | both | Tracks, clips (`start`/`end`, `sourceIn`/`sourceOut` in ticks, frames, `speed`, `reverse`, `gainDb`), effects, transitions, markers, playhead, selection |
| `media_import` | both | `paths` (array of **absolute** paths), optional `bin`, `image_sequence` |
| `render_frame` | both | PNG of the program frame at `seconds`; headless renders, bridge screenshots the Program monitor. Read-only |
| `ui_inspect` | bridge | UI state: tool, workspace, panels, zoom, playback, fps |
| `ui_elements` | bridge | Interactive on-screen elements with id, label, rect (`prefix`) |
| `ui_click` | bridge | By `id` or `x`,`y`; `button`, `count`, `modifiers` |
| `ui_drag` | bridge | Press–move–release between points or element ids |
| `ui_key` | bridge | Key or shortcut: `Space`, `Cmd+K`, `Shift+Delete` |
| `ui_type` | bridge | Type text into the focused field |
| `ui_screenshot` | bridge | PNG of the window or a `panel` |
| `ui_control` | bridge | Any control-channel method (`ui.set`, `ui.scroll`, `ui.timeline.locate`, ...); check exact argument names via `tools/list` |

Resources: `filmcraft://document` (like `doc_inspect`) and `filmcraft://commands` (like `command_list`), both `application/json`.

Typical loop: `project_inspect` / `sequence_inspect` → ids → `command_run` → `render_frame` or `ui_screenshot` → look → `edit.undo` if wrong.

## Recipes
IDs and parameters below come from the upstream docs; for others use `command_list {"filter":"..."}` and `describe`.

**Import and assemble**
```jsonc
// 1. import (absolute paths)
{"tool": "media_import", "arguments": {"paths": ["/Users/me/Video/interview.mp4", "/Users/me/Video/music.wav"]}}
// 2. read item ids
{"tool": "project_inspect", "arguments": {}}
// 3. new sequence from an item's settings (also {mix, trackType})
{"tool": "command_run", "arguments": {"id": "file.newSequence", "params": {"fromItem": 48}}}
// 4. place a clip: item 1 on V1 at 0 s; sourceIn/duration in ticks (5 s = 1270080000000)
{"tool": "command_run", "arguments": {"id": "timeline.place", "params":
  {"item": 1, "track": "V1", "seconds": 0, "sourceIn": 284298240000000, "duration": 1270080000000}}}
```
`timeline.place` also takes `audioTrack` and `video` / `audio` flags; an audio `track` places only the sound.

**Cut and move**
```jsonc
{"id": "playhead.set", "params": {"seconds": 2}}
{"id": "sequence.addEdit", "params": {"seconds": 3}}          // → {"cuts": 2}
{"id": "timeline.razor", "params": {"seconds": 3.5}}
{"id": "timeline.move", "params": {"moves": [{"clip": 12, "track": "V2", "time": 0}], "linked": true}}
```
`timeline.move` also accepts `insert` and `copy: true`. Ripple delete, trim, slip/slide exist as keyboard commands and trim-mode commands (`trim.selectEditPoint`, `trim.selectNearest`, `trim.shuttle`, ...): find ids with `command_list {"filter":"ripple"}` or `"trim"`.

**Effects, color, transitions**
```jsonc
{"id": "effects.apply", "params": {"effect": "Gaussian Blur"}}
{"id": "effects.setParam", "params": {"clip": 87, "effect": "lumetri", "param": "curve_luma",
  "value": [[0,0],[0.25,0.21],[0.75,0.82],[1,1]]}}
{"id": "sequence.applyVideoTransition", "params": {"effect": "<id or name>", "params": {}, "reverse": false}}
{"id": "effects.list", "params": {"folder": "Video Transitions/Wipe", "detail": true}}
```

**Titles and captions**
```jsonc
{"id": "graphics.newText", "params": {"text": "My Title", "seconds": 5, "track": "V2", "time": 0}}
{"id": "graphics.list", "params": {"clip": 90}}                                   // layers, names, properties
{"id": "graphics.set", "params": {"clip": 90, "layer": "<from graphics.list>", "props": {"<param or camelCase alias>": "..."}}}
{"id": "captions.newTrack", "params": {"format": "Subtitle", "name": "EN", "language": "en"}}
{"id": "captions.add", "params": {"track": "C1", "text": "Hello", "time": 0, "durationSeconds": 3}}
{"id": "captions.import", "params": {"path": "/Users/me/Video/sub.srt"}}
```
Without `clip`, `graphics.*` commands use the selected graphic clip or the topmost one under the playhead. Caption formats: SRT, WebVTT, SCC (import/export, format from extension).

**Verify and export**
```jsonc
{"tool": "render_frame", "arguments": {"seconds": 4}}
{"id": "file.exportFrame", "params": {"target": "program", "format": "png", "time": 1016064000000, "path": "/Users/me/Export/f.png"}}
{"id": "file.exportMedia", "params": {"format": "h264", "path": "/Users/me/Export/out.mp4", "wait": true}}
{"id": "file.exportMedia", "params": {"path": "/Users/me/Export/out.mp4", "burnCaptions": true, "wait": true}}
{"id": "file.saveAs", "params": {"path": "/Users/me/Projects/edit-v2.fcproj"}}
```
`file.exportMedia {path, preset?, settings?, format?, range?, ...}`; formats: `h264`, `prores`, `dnxhr`, `mjpeg`, `aac`, `mxf-op1a`, `mxf-opatom`, `png`/`tiff`/`bmp`/`jpg`/`tga`/`dpx`/`exr` (sequences), `gif`, `wav`, `aiff`. `sdr: true` exports an HDR sequence as SDR. Alternatives: `export.quick {preset?, path?}` and the queue `export.queue.add` / `export.queue.start` / `export.queue.list`. (On Windows use `C:\\...` JSON-escaped paths.)

## CLI (`filmcraft-cli`)
JSON output; exit 0 = ok, 1 = command failed, 2 = usage error. `help`, `--help`, `-h` print the reference. Options can go anywhere. Subcommands: `commands`, `describe`, `inspect`, `exec`, `run`, `import`, `export`, `render`, `probe`, `mcp`. On Windows use the full path (`& "C:\Program Files\FilmCraft\filmcraft-cli.exe" ...` in PowerShell).
```sh
filmcraft-cli commands razor                     # search ids (--json for machine output)
filmcraft-cli describe timeline.razor            # menu, shortcut, params, enabled now
filmcraft-cli --demo inspect sequence            # project / sequence as JSON
filmcraft-cli --project p.fcproj --save exec timeline.razor seconds=3.5
filmcraft-cli --project p.fcproj exec effects.apply '{"effect":"Gaussian Blur"}'
filmcraft-cli --project p.fcproj exec markers.exportCsv path=review.csv
filmcraft-cli --project p.fcproj --save import a.mov b.wav
filmcraft-cli --project p.fcproj export out.mp4  # format from extension; waits for the job
filmcraft-cli --project p.fcproj export out --preset "YouTube 1080p Full HD" --start 0 --end 10
filmcraft-cli export --list-presets prores       # built-in + user presets (--data-dir for another library)
filmcraft-cli --project p.fcproj exec export.queue.add preset="Apple ProRes 422 HQ" path=renders/ start=true wait=true
echo '{"id":"file.newBin","params":{"name":"Selects"}}' | filmcraft-cli --project p.fcproj --save run -
filmcraft-cli --bridge 127.0.0.1:9876 exec window.workspace.color   # the open app
filmcraft-cli --bridge 127.0.0.1:9876 exec file.exportMedia path=out.mp4 wait=true
filmcraft-cli render --demo --seconds 3 --out frame.png
```
- `key=value`: the value is JSON when possible (`3.5`, `true`, `[1,2]`), else a string; dotted keys nest (`color.r=1`).
- `run script.jsonl` (or `run -` from stdin): one `{"id":"...","params":{...}}` per line; prints one JSON line per command (`{"line","id","ok","result"|"error"}`) and stops at the first error unless `--keep-going`.
- `--save` rewrites `--project`; `--save-as path` writes elsewhere (preferred, leaves the original untouched). Without `--save`, `exec` changes are not saved.
- `export` / `render` with offline media: warning on stderr and exit 1; `--allow-offline` accepts them.
- PowerShell quoting: if `'{"effect":"..."}'` misbehaves, use the `key=value` form or a `run` script.

## Control channel (raw protocol)
- **Transport:** TCP on `127.0.0.1:<port>`, one JSON request per line → one JSON response per line. Each line must be an object with a string `method` (`id` and `params` optional; blank lines ignored).
- **Limits:** lines up to 4 MiB; max 16 connections; an invalid line (not JSON, array, no `method`, invalid UTF-8, too long, an HTTP request) gets an error and **closes the connection** — reconnect. No authentication.
```json
{"id": 1, "method": "engine.execute", "params": {"command": "sequence.addEdit", "params": {"seconds": 3}}}
{"id": 1, "ok": true, "result": {"cuts": 2}}
{"id": 7, "ok": false, "error": "..."}
```
Minimal Python client (runs on the machine where the app runs):
```python
import json, socket
s = socket.create_connection(("127.0.0.1", 9876)); f = s.makefile("rw")
def call(method, **params):
    f.write(json.dumps({"id": 1, "method": method, "params": params}) + "\n"); f.flush()
    return json.loads(f.readline())
print(call("engine.execute", command="sequence.inspect"))
```

| Method | Params | Notes |
|---|---|---|
| `engine.execute` / `ui.menu.invoke` | `{command, params}` | Any engine or UI command (also `tool.razor`, `playback.toggle`, `window.workspace.color`, `window.panel.<name>`) |
| `engine.commands` / `ui.menu.list` | – | Command registry / menu tree |
| `ui.inspect` | – | UI state (tool, workspace, dock, timeline, playback, fps, `keyboardFocus`, `input`, ...) |
| `ui.elements` | `{prefix?}` | Interactive elements: id, label, rect |
| `ui.set` | `{tool, workspace, mode, theme, focused, playbackRes, timeline:{pps,scroll,fit,thumbnails}, program, source, clipDialog, menuDialog, panels, export, keyframeSelection, projectPanel, mediaBrowser}` | Writes UI state (details in the upstream control-protocol doc) |
| `ui.panel.show` / `ui.panel.close` | `{panel}` | |
| `ui.click` / `ui.move` | `{id}` or `{x,y}`, `button`, `count`, `modifiers` | Synthetic pointer |
| `ui.drag` | `{from, to, steps, modifiers}` | |
| `ui.scroll` | `{id\|x,y, dx, dy, modifiers}` | |
| `ui.key` / `ui.type` | `{key}` / `{text}` | `Cmd+K` = `Ctrl+K` outside macOS |
| `ui.timeline.hit` / `ui.timeline.locate` | `{x,y,modifiers?}` / `{clip, edge?}` | Timeline hit-testing |
| `ui.playback` | `{action: play\|stop\|toggle, speed?}` | |
| `ui.screenshot` | `{path?, panel?}` | PNG; fails after 10 s if no frame is presented (hidden window, display asleep) |
| `ui.resize` / `app.quit` | `{width,height}` / `{}` | `app.quit` exits without asking to save (changes stay in the recovery journal) |
| `ui.focus` | `{}` / `{id}` | `{}` activates the window; `{id}` focuses a widget |
| `perf.stats` | – | Cumulative performance counters |

Stable element ids: `timeline.clip.<id>`, `timeline.track.V1.lock`, `tools.Razor`, `project.item.<id>`, `effects.item.gaussian_blur`, `panel.tab.Timeline`, `program.transport.playback.toggle`, `timeline.caption.<id>`, `text.tab.Captions`, `mixer.A1.fader`, `source.drag.video`, `lumetri.switch.<section>`. Ids come from the previous frame; after a layout change the app retries for a few frames.

## Commands by family
Partial list of ids that appear in the docs; discover the rest with `command_list` / `filmcraft-cli commands <filter>` and parameters with `describe <id>`.

- **file.\*:** `file.newProject`, `file.newSequence {fromItem|mix|trackType}`, `file.open {path}`, `file.openDemoProject`, `file.save {path?}`, `file.saveAs {path}`, `file.saveCopy {path}`, `file.revert`, `file.import {replace?}`, `file.newBin {name}`, `file.exportMedia`, `file.exportFrame`, `file.exportInterchange {format: edl|xml|fcpxml|otio|aaf|omf}`, `file.exportEdl`, `file.exportFcpxml`, `file.exportOtio`, `file.exportAaf`, `file.exportOmf`, `file.projectManager`, `file.recover`, `file.recoveryList`, `file.discardRecovery`, `file.autoSaveNow`, `file.autoSaveStatus`, `file.listAutoSaves`, `file.replaceFonts`
- **edit / playhead / state:** `edit.undo`, `edit.redo`, `playhead.set`, `state.inspect`, `events.list`, `events.clear`
- **project / sequence:** `project.inspect`, `project.notes`, `project.setNotes`, `project.view.set`, `project.columns.set`, `project.sort`, `sequence.inspect`, `sequence.settings {mix}`, `sequence.addEdit`, `sequence.applyVideoTransition`, `sequence.setTransition`, `sequence.colorSettings`
- **timeline / clip / trim:** `timeline.place`, `timeline.razor`, `timeline.move`, `clip.replaceFromBin`, `clip.replaceFromSource`, `clip.replaceFromSourceMatchFrame`, `clip.interpretFootage`, `clip.audioGain {mode: set|adjust|normalizeMax|normalizeAll, db}`, `clip.audioPeak`, `clip.synchronize`, `clip.mergeClips`, `clip.createMulticam`, `trim.selectEditPoint`, `trim.selectNearest`, `trim.monitor`, `trim.shuttle`, `trim.tick`, `trim.shuttleStop`, `trim.cancelDynamic`, `trim.playAround`
- **markers:** `markers.exportCsv {path}`, `markers.clearInOut`
- **effects / masks / color:** `effects.apply`, `effects.setParam`, `effects.addKeyframe`, `effects.remove`, `effects.list`, `effects.setDefaultTransition`, `masks.add`, `masks.roundCorners`, `lumetri.setInputLut`, `lumetri.setLook`, `lumetri.setSection`, `lumetri.applyMatch`, `lut.import`, `lut.list`, `lut.remove`, `lut.export`, `color.spaces`, `media.colorInfo`, `scopes.read`
- **graphics (titles):** `graphics.newText`, `graphics.newVerticalText`, `graphics.newRectangle`, `graphics.newEllipse`, `graphics.newPolygon`, `graphics.newShape`, `graphics.newFromFile`, `graphics.setText`, `graphics.set`, `graphics.setCharStyle`, `graphics.list`, `graphics.align`, `graphics.distribute`, `graphics.deleteLayer`, `graphics.arrangeLayer`, `graphics.template.apply`, `graphics.template.export`, `fonts.list`
- **captions:** `captions.newTrack`, `captions.add`, `captions.setText`, `captions.setTimes`, `captions.trim`, `captions.move`, `captions.split`, `captions.merge`, `captions.delete`, `captions.setTrack`, `captions.setStyle`, `captions.deleteTrack`, `captions.import`, `captions.export`, `captions.list`
- **transcript:** `transcript.generate`, `transcript.set`, `transcript.inspect`, `transcript.search`, `transcript.select`, `transcript.extract`, `transcript.lift`, `transcript.removeFillers`, `transcript.removePauses`, `transcript.createCaptions`, `transcript.renameSpeaker`, `transcript.models`, `transcript.downloadModel`
- **audio:** `mixer.inspect`, `mixer.setStrip`, `mixer.setValue`, `mixer.addSubmix`, `mixer.addInsert`, `mixer.addSend`, `mixer.setKeyframe`, `mixer.writeAutomation`, `clipMixer.set`, `essentialSound.setType {type: dialogue|music|sfx|ambience}`, `essentialSound.set`, `essentialSound.applyPreset`, `essentialSound.autoMatch`, `essentialSound.generateDucking`, `tts.create`, `voiceover.recordToggle`
- **media / proxy:** `media.status`, `media.findMissing`, `media.relink`, `media.autoRelink`, `media.replaceFootage {item, path}`, `media.makeOffline`, `media.createProxies`, `media.attachProxies`, `media.toggleProxies`, `project.ingestSettings`
- **export / jobs:** `export.quick`, `export.queue.add`, `export.queue.start`, `export.queue.list`, `jobs.list`, `jobs.cancel`
- **multicam:** `multicam.cut {camera}`, `multicam.switchAngle`, `multicam.recordStart`, `multicam.recordStop`, `multicam.grid`, `multicam.gridLayout`, `multicam.page`
- **preferences / shortcuts:** `prefs.get`, `prefs.set {key, value}`, `prefs.reset`, `shortcuts.list`, `shortcuts.set`, `shortcuts.loadPreset`, `shortcuts.conflicts`
- **UI-only:** `tool.razor`, `playback.toggle`, `window.workspace.color`, `window.panel.<name>`, `window.floatPanel`, `window.dockPanel`, `window.closePanel`, `view.showRulers`, `view.safeMargins`

## Recommended workflow
1. **Orient.** Pick the channel, then `doc_inspect` (or `filmcraft-cli ... inspect sequence`) for the project, active sequence, tracks and ids.
2. **Never destroy.** Work on a copy: `file.open` then straight away `file.saveAs` to a new `.fcproj`, or `--save-as` in the CLI. Never overwrite the project or the original media.
3. **Find commands.** `command_list {"filter":"...", "enabled_only":true}` / `describe <id>`: read the documented `params` instead of guessing; if a command is disabled the message says why.
4. **Edit in small steps** with explicit targets (`clips`, `item`, `track`) instead of relying on selection; re-read with `sequence_inspect` after every step that creates or moves clips.
5. **Verify visually.** `render_frame {seconds}` (or `ui_screenshot` in bridge mode), look at the PNG, `edit.undo` if wrong. Check `media.status` before exporting.
6. **Save/export only with the user's OK**, to a new path (`file.saveAs`, `file.exportMedia {path, wait:true}`), and check `offlineMedia` in the result. When done, remind the user that the control port is unauthenticated and should be closed.
