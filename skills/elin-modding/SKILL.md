---
name: elin-modding
description: Build and debug mods for the game Elin. Covers looking up game code in a local ilspycmd decompile of the installed game (checked first), the Elin knowledge MCP server at mcp.elin-modding.net (modding wiki, source-sheet columns, game ids, project scaffolding, and an online source index as the fallback), and setting up the decompile, the knowledge server and the Elin Runtime MCP mod. Use whenever the user mentions Elin modding, BepInEx/Harmony patches for Elin, Elin source sheets (Thing/Chara/Race xlsx), Elin package.xml, asks how something in Elin is implemented, or asks to set up, install or connect Elin MCP.
---

# Elin Modding

Elin is a Unity game built on Mono (.NET Framework 4.7.2, no IL2CPP), so `ilspycmd` decompiles its DLLs directly.

## Where to look

Look up game code in this order:

1. The local decompile in `~/Elin-Decompiled`. It is built from the installed game, so it matches the version the user runs, and `rg` searches all of it at once.
2. The knowledge server `elin`, when the local copy cannot answer: you cannot build one (no shell, no .NET SDK, or the game is not on this machine), you need the branch the user does not run (`channel`), or you need an inheritance tree several levels deep (`elin_hierarchy`).

The modding wiki, ids, sheet columns and scaffolding come only from the knowledge server. To build or change a mod, start at `elin_guide`, because the code does not show how to package a mod so that the game loads it.

For the running game (testing a patch, reading the game log), use the runtime server `elin-runtime` and the `elin-runtime` skill.

## Setup

Before the first code lookup in a session, check that the local copy matches the installed game:

```bash
G=$(cat ~/Elin-Decompiled/game-path) && [ "$(sha256sum < "$G/Elin_Data/Managed/Elin.dll" | cut -d' ' -f1)" = "$(cat ~/Elin-Decompiled/Elin.dll.sha256)" ] && echo current || echo stale
```

```pwsh
$G = Get-Content ~\Elin-Decompiled\game-path; (Get-FileHash "$G\Elin_Data\Managed\Elin.dll").Hash.ToLower() -eq (Get-Content ~\Elin-Decompiled\Elin.dll.sha256)
```

Anything other than `current` or `True` means the copy is stale or missing: run step 2 of [`setup.md`](setup.md) in this folder. It takes under a minute.

If the tool list lacks `elin_guide` (knowledge server) or `elin_instances` (runtime server), or the user asks to set up Elin MCP, follow all of `setup.md`. The same text is at <https://mcp.elin-modding.net/llms.txt> and comes back from `elin_setup`.

## Never read a source file whole

`Chara.cs` is about 67k tokens, `Card.cs` 45k, `BaseTileMap.cs` 31k, `Zone.cs` 27k. Find the member first, then read only its lines.

## Searching the local copy

Files are laid out as `<assembly>/<namespace>/<Type>.cs`. Types without a namespace, which is most of `Elin`, sit directly under the assembly: `Elin/Chara.cs`, `Plugins.Modding/PackageIterator.cs`. The knowledge server returns the same paths. Its line numbers agree with yours only when both came from the same `ilspycmd` version, so match by name.

```bash
cd ~/Elin-Decompiled
rg -n "^\s+(public|protected|internal|private).*\bDie\("        # where Die is declared, with signatures
rg -n "^\s+(public|protected|internal|private)\b" Elin/Trait.cs # every member of Trait
rg -n "class \w+ : Trait\b"                                     # direct subclasses of Trait
rg -n "\bDie\("                                                 # every declaration of and call to Die
rg -n '"openDoor"'                                              # a string value: sound id, lang key, sheet column, tag
```

Then read about 60 lines from the hit. Without ripgrep, use `grep -rnE` or `Get-ChildItem -Recurse -Filter *.cs | Select-String '<regex>'`. Calls are matched by name only: same-named methods on other types show up too, and calls through reflection or delegates do not.

What lives where:

- `Elin`: all game logic, and what you patch. `Assembly-CSharp.dll` is a near-empty Unity leftover.
- `Plugins.BaseCore`: `Lang`, `EInput`, `BaseModManager` (the mod event bus).
- `Plugins.Modding`: the official modding API, such as `EMod`, `ICustomContent`, `PackageIterator` and `SourceCache`.
- `ElinModdingKit`: the built-in script compiler, `CodeMatchExt`, `AccessorHelper`.
- `ClassLibrary2`: `AK.ExpressionSolver`, which evaluates the formula strings in source sheets.

Decompiled code is rebuilt from compiled IL. It has no comments, lambdas turn into `<>c__DisplayClass` classes, `yield` and `async` methods turn into state machines, and a `switch` can come back as `if` chains. It is for reading and need not compile. Do not publish it or ship any of it with a mod.

## Knowledge server

Prose comes back in `en` (default), `zh` or `ja`; pass `lang`.

| Tool | Use |
|---|---|
| `elin_guide task?` | Turns an intent into an ordered route: which articles, sheets and types. Without `task`, lists the routes. |
| `elin_wiki q`, `elin_doc path section?` | Search and read the modding wiki. A long article returns its outline first; call again with `section`. |
| `elin_sheet sheet` | A source sheet's columns, read from the game's `Source*.Row` classes. |
| `elin_ids category q?` | The ids that exist, searchable by EN/JP/CN name. |
| `elin_scaffold kind name author` | `script`: the template commands plus `package.xml` and a sample patch. `sheet`: a data-only mod layout. |
| `elin_versions` | The game version behind each index channel. |
| `elin_find`, `elin_outline`, `elin_hierarchy`, `elin_callers`, `elin_strings`, `elin_read` | The online source index, for when the local copy cannot answer. |

Using the online index:

- Index tools take `channel`: `stable` (default) or `nightly`, Elin's two branches. The user runs `nightly` when `<game>/version.json` says `"nightly": true`. No older version is kept.
- `elin_find` matches names. Sound ids, lang keys and sheet column names are string values, so look for them with `elin_strings`.
- `elin_read` returns at most 400 lines; a cut-off body tells you which `from` to pass next. When a name is declared more than once, a note lists the others as `relpath:line`; pass one as `relpath`.
- If `elin_versions` is behind the installed game that `elin_instances` reports, the index needs a rebuild. Compare the middle number (`EA 23.349` against `23.348.1`). Tell the user; the server is maintained under [Elin-Modding-Resources](https://github.com/Elin-Modding-Resources).

## Building a mod

A data mod:

```
elin_guide task="add a new item"
  → elin_doc <the articles it names>
  → elin_sheet sheet="thing"        # what the columns mean
  → elin_ids category="category"    # which values are legal
  → elin_scaffold kind="sheet" name="..." author="..."
```

A C# script mod starts from the official `dotnet new` template. `elin_scaffold kind="script"` returns its commands and prerequisites (.NET 10 SDK, `ElinGamePath`), plus the `package.xml` the template leaves out. The game loads mod DLLs only at startup, so restart it after every build.

Before you patch a method, find in the local copy its exact signature (overloads on base classes included), whether to patch the base method or a subclass override, and where it is called from. Skipping the call sites is the most common reason a patch compiles and loads but never runs. Then watch it work in the game; the `elin-runtime` skill covers that loop.

## What the code will not tell you

- Everything the mod ships goes under `package\`: `package.xml`, `preview.jpg`, `LangMod\`, `Texture\`, `Sound\`. Without `package.xml` the loader logs `Not a package:` and ignores the folder. Its `<id>` must match `ModInfo.Guid` in `Plugin.cs`.
- `<version>` in `package.xml` is not your mod's version. If it has three parts and is below the game's minimum (`BaseCore.versionMod`), the loader skips the mod before `Activate()`, logs `Skipped (made for game …)` and shows the mod red in the mod list. Values with fewer than three parts are not checked. The scaffold writes `1.0.0`.
- Never reuse an existing id, and check every cross-reference field (race, job, trait, category…) with `elin_ids`. A reference to an id that does not exist is the most common source-sheet failure.
- You cannot write `.xlsx` as text. Copy the game's own sheet out of its package, keep the header rows and replace the data rows.
- When a mod does not load, `elin_guide task="troubleshoot"` maps each loader log line to its cause. The game's log is `Player.log`, next to the `Save/` folder; BepInEx's `LogOutput.log` lacks the game's own lines. With the runtime server, `elin_logs contains="..."` searches it.
