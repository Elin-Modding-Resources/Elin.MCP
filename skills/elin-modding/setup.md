# Elin MCP setup

> Sets up an AI coding agent for modding the game Elin. It has three parts.
> A local decompile of the installed game in `~/Elin-Decompiled` is the first place to look up game code.
> `elin` is a remote MCP server with the modding wiki, game ids, sheet columns and an online index of the
> game's source. `elin-runtime` is a local MCP server that comes with the Elin Runtime MCP mod. It evaluates
> C#, runs console commands, takes screenshots, searches the log and hot-loads Harmony patches inside the
> running game. Two agent skills explain how to use all of it.

This page is for the agent that does the setup. Do every step that applies, in order, and skip what is already
done. "Set up Elin MCP" means all three parts. Set up only one if the user asks for only one.

The same text is served at <https://mcp.elin-modding.net/llms.txt> and by the `elin_setup` tool. Its source
is `skills/elin-modding/setup.md` in <https://github.com/Elin-Modding-Resources/Elin.MCP>.

## 0. Check what is connected

| If your tool list has | Then |
|---|---|
| `elin_guide`, `elin_find`, `elin_versions`, … | `elin` is connected. Skip step 3. |
| `elin_instances`, `elin_eval`, `elin_hotload`, … | `elin-runtime` is connected. Skip steps 4 and 5. |

Only tools you can call count. If a config entry exists but you do not see its tools, the entry is stale or
broken. Fix it instead of adding a second one.

You cannot add a server to the session you are running in. The client picks up steps 3, 5 and 6 when it
restarts (step 7).

## 1. Find the game

`<game>` is the folder that holds `Elin_Data/` and `version.json`. Look in these places, in order:

1. The `ElinGamePath` environment variable. Script mod projects use it too.
2. `C:\Program Files (x86)\Steam\steamapps\common\Elin`, or `~/.steam/steam/steamapps/common/Elin` on Linux.
3. `steamapps/common/Elin` under each `"path"` listed in `<Steam>/steamapps/libraryfolders.vdf`. `<Steam>` is
   the `SteamPath` value in the registry key `HKCU\Software\Valve\Steam`, or `~/.steam/steam` on Linux.
4. If none of these exists, ask the user.

## 2. Decompile the game

This needs the .NET 10 SDK, which C# script mods need too. Check with `dotnet --list-sdks`; on
Windows, install it with `winget install Microsoft.DotNet.SDK.10`. Then install `ilspycmd` 10 or newer with
`dotnet tool install --global ilspycmd`, or `update` instead of `install` for an older copy.

If you cannot install them, tell the user and go on with step 3. The skill then uses the knowledge server's
online index.

The script decompiles the 14 assemblies that the online index covers into `~/Elin-Decompiled/<assembly>/`.
It builds into `~/Elin-Decompiled.new` and swaps that in only when every assembly succeeded. It records the
game folder in `game-path` and the SHA-256 of `Elin.dll` in `Elin.dll.sha256`, does nothing while the game is
unchanged, and refuses to replace an `~/Elin-Decompiled` it did not create. Run it again after every game
update. A full run takes under a minute.

```pwsh
$Game = '<game>'   # from step 1
$Out = "$HOME\Elin-Decompiled"
$M = "$Game\Elin_Data\Managed"; $Kit = "$Game\Package\_ModdingKit"; $Core = "$Game\BepInEx\core"
$env:PATH += ";$HOME\.dotnet\tools"
if (!(Test-Path "$M\Elin.dll")) { throw "no Elin.dll under $Game" }
$Hash = (Get-FileHash "$M\Elin.dll").Hash.ToLower()
if ((Get-Content "$Out\Elin.dll.sha256" -ErrorAction Ignore) -eq $Hash) { 'up to date'; return }
if ((Test-Path $Out) -and !(Test-Path "$Out\Elin.dll.sha256")) { throw "$Out was not made by this script; move it away first" }
Remove-Item "$Out.new" -Recurse -Force -ErrorAction Ignore
foreach ($a in 'Elin','Plugins.BaseCore','Plugins.UI','Plugins.Modding','Plugins.Scripting','Plugins.Util','Plugins.Calc',
               'Plugins.Sound','Plugins.ActorSystem','Plugins.NameGen','Plugins.NoiseSystem','Plugins.Tween','ClassLibrary2','ElinModdingKit') {
    $Dll = if ($a -eq 'ElinModdingKit') { "$Kit\$a.dll" } else { "$M\$a.dll" }
    ilspycmd -p -o "$Out.new\$a" -r $M -r $Kit -r $Core -lv CSharp14_0 --disable-updatecheck $Dll
    if ($LASTEXITCODE) { throw "ilspycmd failed on $a" }
}
Set-Content "$Out.new\Elin.dll.sha256" $Hash -NoNewline
Set-Content "$Out.new\game-path" $Game -NoNewline
Remove-Item $Out -Recurse -Force -ErrorAction Ignore
Move-Item "$Out.new" $Out
"decompiled to $Out"
```

```bash
(
set -e
GAME="<game>"   # from step 1
OUT=~/Elin-Decompiled
M="$GAME/Elin_Data/Managed"; KIT="$GAME/Package/_ModdingKit"; CORE="$GAME/BepInEx/core"
export PATH="$PATH:$HOME/.dotnet/tools"
[ -f "$M/Elin.dll" ] || { echo "no Elin.dll under $GAME"; exit 1; }
HASH=$(sha256sum < "$M/Elin.dll" | cut -d' ' -f1)
[ "$(cat "$OUT/Elin.dll.sha256" 2>/dev/null)" != "$HASH" ] || { echo "up to date"; exit 0; }
[ ! -e "$OUT" ] || [ -f "$OUT/Elin.dll.sha256" ] || { echo "$OUT was not made by this script; move it away first"; exit 1; }
rm -rf "$OUT.new"
for a in Elin Plugins.BaseCore Plugins.UI Plugins.Modding Plugins.Scripting Plugins.Util Plugins.Calc \
         Plugins.Sound Plugins.ActorSystem Plugins.NameGen Plugins.NoiseSystem Plugins.Tween ClassLibrary2 ElinModdingKit; do
  dll="$M/$a.dll"; [ "$a" != ElinModdingKit ] || dll="$KIT/$a.dll"
  ilspycmd -p -o "$OUT.new/$a" -r "$M" -r "$KIT" -r "$CORE" -lv CSharp14_0 --disable-updatecheck "$dll"
done
echo "$HASH" > "$OUT.new/Elin.dll.sha256"
echo "$GAME" > "$OUT.new/game-path"
rm -rf "$OUT"; mv "$OUT.new" "$OUT"
echo "decompiled to $OUT"
)
```

`-r` lists the folders that references resolve from. The modding kit references Roslyn in its own folder and
Harmony in `BepInEx/core`; without all three, signatures show assembly-qualified names instead of types.
`-lv CSharp14_0` is the language version the online index uses.

## 3. Knowledge server `elin`

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

## 4. Install the Elin Runtime MCP mod

### Check for an existing install

The mod folder is the one that holds `bridge/elin-mcp.mjs`. Look for it in:

- `<game>/Package/Mod_ElinRuntimeMcp/`, for a GitHub install or a build from source
- the folder under `<Steam library>/steamapps/workshop/content/2135150/` whose `package.xml` contains
  `<id>dk.elinplugins.runtimemcp</id>`, for a Steam Workshop subscription

If you find it, that folder is `<mod>` and you go on to step 5. Do not install a second copy next to it. If
`<mod>/skills/` is missing, the copy is older than 1.0.0. Update it the way it was installed: for a Workshop
copy, Steam does it; for a GitHub copy, repeat the download below.

### Install from GitHub

Releases have tags of the form `RMCP-<version>` on <https://github.com/gottyduke/Elin.Plugins/releases>, and
the first one is `RMCP-1.0.0`. Use the newest release build.

The zip holds the contents of the mod folder: `package.xml`, `ElinRuntimeMcp.dll`, `bridge/`, `skills/` and
`README.md`. Extract it into `<game>/Package/Mod_ElinRuntimeMcp/`. To update, extract over the old copy.

```pwsh
$Game = '<game>'   # from step 1
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

After the install, tell the user one thing in a single line: the mod listens on 127.0.0.1, ports 27551 to 27560,
and runs any C# that a local program sends it. It is a development tool. For normal play the user should
disable it in the mod list, or set `Enabled = false` under `[Listener]` in
`<game>/BepInEx/config/dk.elinplugins.runtimemcp.cfg`.

The game loads the mod the next time it starts, so a running game has to be restarted. New mods start
enabled. If the user turned this one off at some point, it has to be turned back on in the in-game mod list.

## 5. Connect `elin-runtime`

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

The output also includes the `elin` entry. Skip that one if step 3 is done. To check that the bridge runs,
without restarting anything, use `node "<mod>/bridge/elin-mcp.mjs" --version`.

`elin_build` can get a default project from the env variable `ELIN_BUILD_PROJECT` (OpenCode calls the key
`environment`). Its value is the mod's `.csproj`, relative to the project root. Set it only in a
project-level entry, because the path is relative.

## 6. Install the skills

Skip this step for a client that does not support skills. The skills folder is `~/.claude/skills/` for
Claude Code, and `~/.agents/skills/` for Codex, Gemini CLI and GitHub Copilot CLI.

| Skill | Covers | Install |
|---|---|---|
| `elin-modding` | Looking up game code, the knowledge server, mod packaging, source sheets | Download `SKILL.md` and `setup.md` from `https://mcp.elin-modding.net/skills/elin-modding/` into `<skills>/elin-modding/` |
| `elin-runtime` | Testing and debugging in the running game | Link `<mod>/skills/elin-runtime` to `<skills>/elin-runtime`, so that mod updates reach it |

```pwsh
$Skills = "$HOME\.claude\skills"   # Codex, Gemini CLI, Copilot CLI: "$HOME\.agents\skills"
New-Item -ItemType Directory -Force "$Skills\elin-modding" | Out-Null
foreach ($f in 'SKILL.md', 'setup.md') {
    Invoke-WebRequest "https://mcp.elin-modding.net/skills/elin-modding/$f" -OutFile "$Skills\elin-modding\$f"
}
# A junction needs no admin rights. $Mod is the mod folder from step 4.
New-Item -ItemType Junction -Path "$Skills\elin-runtime" -Target "$Mod\skills\elin-runtime"
```

Outside PowerShell, use `curl -fsSL -o` for the downloads and
`ln -s "<mod>/skills/elin-runtime" ~/.claude/skills/elin-runtime` for the link.

## 7. Restart and verify

Clients read their server list only at startup, so ask the user to restart the client. In Claude Code,
`claude --continue` resumes this conversation with the new tools.

After the restart:

1. `~/Elin-Decompiled/Elin/Chara.cs` should exist.
2. `elin_versions` should return the game version behind `stable` and `nightly`.
3. `elin_instances` should answer. With the game closed, "No game instance is listening" is the correct
   answer. `elin_launch` starts the game through Steam and waits for it.

Report everything in one message: what you installed and where (the decompile folder included), what the
user still has to do (restart the client, install .NET or Node.js, enable the mod) and the security line from
step 4.

## When it does not work

| Symptom | Cause and fix |
|---|---|
| `dotnet` is not found right after installing the SDK | The shell still has the old PATH. Call it by full path, `C:\Program Files\dotnet\dotnet.exe` on Windows. |
| No `elin-runtime` tools after a restart | `node` is not on the client's PATH. This happens to GUI clients started from the desktop. Put the absolute path of `node` in `command`. Running `node "<mod>/bridge/elin-mcp.mjs" --help` in a terminal shows whether the bridge itself starts. |
| `elin_instances` finds nothing while the game runs | The game started before the mod was installed, the mod is off in the mod list, or `Enabled = false` is set under `[Listener]`. When the listener is up, the game log says `listening on 127.0.0.1:<port>`. |
| Every instance shows `BUSY` when the game window loses focus | The game's "run in background" option is off. |
| `no free port in 27551-27560` in the game log | Ten instances are already running, or other programs hold those ports. |
| Anything else about the runtime | `<mod>/README.md` is the full reference. |
