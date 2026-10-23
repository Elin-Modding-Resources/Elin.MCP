# Elin MCP setup

> Connects an AI coding agent to the tooling for modding the game Elin. It has two MCP servers, and you set
> them up together. **`elin`** is a remote server that indexes the game's source and the modding wiki.
> **`elin-runtime`** is a local server that comes with the Elin Runtime MCP mod. It evaluates C#, runs
> console commands, takes screenshots, searches the log and hot-loads Harmony patches inside the running
> game. Two agent skills explain how to use both.

This page is for the agent that does the setup. Do every step that applies, in order, and skip the ones that
are already done. "Set up Elin MCP" means both servers. Set up only one if the user asks for only one.

The same text is served at <https://mcp.elin-modding.net/llms.txt> and by the `elin_setup` tool. Its source
is `skills/elin-modding/setup.md` in <https://github.com/Elin-Modding-Resources/Elin.MCP>.

## 0. Check what is connected

| If your tool list has | Then |
|---|---|
| `elin_guide`, `elin_find`, `elin_versions`, … | `elin` is connected. Skip step 1. |
| `elin_instances`, `elin_eval`, `elin_hotload`, … | `elin-runtime` is connected. Skip steps 2 and 3. |

Only tools you can call count. If a config entry exists but you do not see its tools, the entry is stale or
broken. Fix it instead of adding a second one.

You cannot add a server to the session you are running in. Each step below edits configuration that takes
effect when the client restarts (step 5).

## 1. Knowledge server `elin`

`https://mcp.elin-modding.net` uses Streamable HTTP. It needs no auth and nothing installed. Add it at user
level, so that it works in every mod project:

| Client | Add |
|---|---|
| Claude Code | `claude mcp add --scope user --transport http elin https://mcp.elin-modding.net` |
| Codex | `~/.codex/config.toml`: `[mcp_servers.elin]` with `url = "https://mcp.elin-modding.net"` |
| Cursor | `~/.cursor/mcp.json`: `{ "mcpServers": { "elin": { "type": "http", "url": "https://mcp.elin-modding.net" } } }` |
| Gemini CLI | `~/.gemini/settings.json`: `{ "mcpServers": { "elin": { "httpUrl": "https://mcp.elin-modding.net" } } }` |
| VS Code / Copilot | `.vscode/mcp.json`: `{ "servers": { "elin": { "type": "http", "url": "https://mcp.elin-modding.net" } } }` |
| OpenCode | `~/.config/opencode/opencode.json`: `{ "mcp": { "elin": { "type": "remote", "url": "https://mcp.elin-modding.net", "enabled": true } } }` |

Merge the entry into the existing file and keep the other servers in it. For other clients, the entry is
always a name, an HTTP transport and this URL. The URL field is called `url`, `httpUrl` or `serverUrl`,
depending on the client. A client that only speaks stdio can reach the server through `mcp-remote`:
`"command": "npx", "args": ["-y", "mcp-remote", "https://mcp.elin-modding.net"]`. On Windows, use
`"command": "cmd", "args": ["/c", "npx", ...]`, because `npx` is a `.cmd` file there.

## 2. Install the Elin Runtime MCP mod

### Find the game

`<game>` is the folder that holds `Elin_Data/` and `version.json`. Look in these places, in order:

1. The `ElinGamePath` environment variable. Script mod projects use it too.
2. `C:\Program Files (x86)\Steam\steamapps\common\Elin`.
3. `steamapps/common/Elin` under each `"path"` listed in `<Steam>/steamapps/libraryfolders.vdf`. `<Steam>` is
   the `SteamPath` value in the registry key `HKCU\Software\Valve\Steam`.
4. If none of these exists, ask the user.

### Check for an existing install

The mod folder is the one that holds `bridge/elin-mcp.mjs`. Look for it in:

- `<game>/Package/Mod_ElinRuntimeMcp/`, for a GitHub install or a build from source
- the folder under `<Steam library>/steamapps/workshop/content/2135150/` whose `package.xml` contains
  `<id>dk.elinplugins.runtimemcp</id>`, for a Steam Workshop subscription

If you find it, that folder is `<mod>` and you go on to step 3. Do not install a second copy next to it. If
`<mod>/skills/` is missing, the copy is older than 1.0.0. Update it the way it was installed: for a Workshop
copy, Steam does it; for a GitHub copy, repeat the download below.

### Install from GitHub

Releases have tags of the form `RMCP-<version>` on <https://github.com/gottyduke/Elin.Plugins/releases>, and
the first one is `RMCP-1.0.0`. Use the newest release build. 

The zip holds the contents of the mod folder: `package.xml`, `ElinRuntimeMcp.dll`, `bridge/`, `skills/` and
`README.md`. Extract it into `<game>/Package/Mod_ElinRuntimeMcp/`. To update, extract over the old copy.

```pwsh
$Game = if ($env:ElinGamePath) { $env:ElinGamePath } else { 'C:\Program Files (x86)\Steam\steamapps\common\Elin' }
$Meta = Join-Path $Game 'version.json'
$Branch = if ((Test-Path $Meta) -and (Get-Content $Meta -Raw | ConvertFrom-Json).nightly) { 'nightly' } else { 'stable' }
$Release = (Invoke-RestMethod 'https://api.github.com/repos/gottyduke/Elin.Plugins/releases?per_page=100') |
    Where-Object tag_name -Like 'RMCP-*' | Select-Object -First 1
$Asset = $Release.assets | Where-Object name -Like "rmcp-ci-builds-$Branch-*-release.zip" | Select-Object -First 1
$Zip = Join-Path ([IO.Path]::GetTempPath()) $Asset.name
Invoke-WebRequest $Asset.browser_download_url -OutFile $Zip
if ($Asset.digest -and "sha256:$((Get-FileHash $Zip -Algorithm SHA256).Hash.ToLower())" -ne $Asset.digest) { throw "checksum mismatch: $Zip" }
$Mod = Join-Path $Game 'Package\Mod_ElinRuntimeMcp'
Expand-Archive $Zip -DestinationPath $Mod -Force
Remove-Item $Zip
"$($Release.tag_name) ($Branch) -> $Mod"
```

You can do the same from another shell. The GitHub API returns the asset list as JSON, `digest` holds the
asset's sha256, and `unzip -o <zip> -d "<game>/Package/Mod_ElinRuntimeMcp"` extracts it.

After the install, tell the user one thing in a single line: the mod listens on 127.0.0.1, ports 27551–27560,
and runs any C# that a local program sends it. It is a development tool. For normal play the user should
disable it in the mod list, or set `Enabled = false` under `[Listener]` in
`<game>/BepInEx/config/dk.elinplugins.runtimemcp.cfg`.

The game loads the mod the next time it starts, so a running game has to be restarted. New mods start
enabled. If the user turned this one off at some point, it has to be turned back on in the in-game mod list.

## 3. Connect `elin-runtime`

The server needs Node.js 20 or newer on the PATH that the client starts servers with. Check with
`node --version`. If Node is missing or older, ask the user to install the LTS release (on Windows:
`winget install OpenJS.NodeJS.LTS`), then continue in a new terminal.

This command prints the config entry for every common client, with the absolute path of this install filled
in:

```
node "<mod>/bridge/elin-mcp.mjs" --config
```

Apply the block for your client, merged into the existing config. For Claude Code, the command should
carry `--scope user`. Older bridges print it without, so add it if it is missing:

```
claude mcp add --scope user elin-runtime -- node "<mod>/bridge/elin-mcp.mjs"
```

The output also includes the `elin` entry. Skip that one if step 1 is done. To check that the bridge runs,
without restarting anything, use `node "<mod>/bridge/elin-mcp.mjs" --version`.

`elin_build` can get a default project from the env variable `ELIN_BUILD_PROJECT` (OpenCode calls the key
`environment`). Its value is the mod's `.csproj`, relative to the project root. Set it only in a
project-level entry, because the path is relative.

## 4. Install the skills

Skip this step for a client that does not support skills. The skills folder is `~/.claude/skills/` for
Claude Code, and `~/.agents/skills/` for Codex, Gemini CLI and GitHub Copilot CLI.

| Skill | Covers | Install |
|---|---|---|
| `elin-modding` | The knowledge server, mod packaging, source sheets, decompiling | Download `SKILL.md` and `setup.md` from `https://mcp.elin-modding.net/skills/elin-modding/` into `<skills>/elin-modding/` |
| `elin-runtime` | Testing and debugging in the running game | Link `<mod>/skills/elin-runtime` to `<skills>/elin-runtime`, so that mod updates reach it |

```pwsh
$Skills = "$HOME\.claude\skills"   # Codex, Gemini CLI, Copilot CLI: "$HOME\.agents\skills"
New-Item -ItemType Directory -Force "$Skills\elin-modding" | Out-Null
foreach ($f in 'SKILL.md', 'setup.md') {
    Invoke-WebRequest "https://mcp.elin-modding.net/skills/elin-modding/$f" -OutFile "$Skills\elin-modding\$f"
}
# A junction needs no admin rights. $Mod is the mod folder from step 2.
New-Item -ItemType Junction -Path "$Skills\elin-runtime" -Target "$Mod\skills\elin-runtime"
```

Outside PowerShell, use `curl -fsSL -o` for the downloads and
`ln -s "<mod>/skills/elin-runtime" ~/.claude/skills/elin-runtime` for the link.

## 5. Restart and verify

Clients read their server list only at startup, so ask the user to restart the client. In Claude Code,
`claude --continue` resumes this conversation with the new tools.

After the restart:

1. `elin_versions` should return the game version behind `stable` and `nightly`.
2. `elin_instances` should answer. With the game closed, "No game instance is listening" is the correct
   answer. `elin_launch` starts the game through Steam and waits for it.
3. Compare the game version from `elin_instances` with `elin_versions`. The middle number is the one that
   matters. If the installed game is newer than both channels, the index lags, and the `elin-modding` skill
   covers what to do.

Report everything in one message: what you installed and where, what the user still has to do (restart the
client, install Node.js, enable the mod) and the security line from step 2.

## When it does not work

| Symptom | Cause and fix |
|---|---|
| No `elin-runtime` tools after a restart | `node` is not on the client's PATH. This happens to GUI clients started from the desktop. Put the absolute path of `node` in `command`. Running `node "<mod>/bridge/elin-mcp.mjs" --help` in a terminal shows whether the bridge itself starts. |
| `elin_instances` finds nothing while the game runs | The game started before the mod was installed, the mod is off in the mod list, or `Enabled = false` is set under `[Listener]`. When the listener is up, the game log says `listening on 127.0.0.1:<port>`. |
| Every instance shows `BUSY` when the game window loses focus | The game's "run in background" option is off. |
| `no free port in 27551-27560` in the game log | Ten instances are already running, or other programs hold those ports. |
| Anything else about the runtime | `<mod>/README.md` is the full reference. |
