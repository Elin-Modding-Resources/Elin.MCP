---
name: elin-modding
description: Build, debug and test mods for the game Elin. Covers the Elin knowledge MCP server (mcp.elin-modding.net - game source index, modding wiki, source-sheet schemas, game ids, project scaffolding), the Elin Runtime MCP mod (run C#, console commands, screenshots, logs and hot-loaded Harmony patches inside the running game), and decompiling the game locally when neither is enough. Use whenever the user mentions Elin modding, BepInEx/Harmony patches for Elin, Elin source sheets (Thing/Chara/Race xlsx), Elin package.xml, testing or debugging a mod in the game, asks how something in Elin is implemented, or asks to set up, install or connect Elin MCP.
---

# Elin Modding

Elin is a Unity (Mono, .NET Framework 4.7.2) roguelike whose modding surface is
undocumented in-code: the packaging rules, sheet formats and loader conventions
live in a community wiki, and the game logic lives in a decompilable assembly.

Three sources cover it. Each answers a different question:

| Source | Answers | Needs |
|---|---|---|
| **Knowledge server** `elin` (remote) | How do I make X? Where is X implemented? Which ids and sheet columns exist? | A URL. No install, no game files, no API key |
| **Runtime server** `elin-runtime` (local) | What does the game I have installed actually do? Does my patch work? | The Elin Runtime MCP mod, Node.js 20+, the game |
| **Local decompile** | Full-text search, offline work, a game version the index does not have yet | .NET SDK and `ilspycmd` |

Use the knowledge server for names and conventions, the running game for
ground truth, and the local decompile only when the other two cannot answer.

---

## 1. Setup

Look at your tool list first. The knowledge tools are named `elin_guide`,
`elin_find`, … and the runtime tools `elin_instances`, `elin_eval`, …. If both
sets are there, skip this section.

If either set is missing, or the user asks you to set up Elin MCP, read
[`setup.md`](setup.md) in this folder and follow it. It finds the game,
installs the Elin Runtime MCP mod from its GitHub release for the branch the
game runs on, connects both servers in your client, installs both skills, and
then checks the result. The same procedure is served at
<https://mcp.elin-modding.net/llms.txt> and returned by the `elin_setup` tool.

Setting up Elin MCP means setting up both servers. Only the runtime server can
show what a change does in the game, so do not stop after the knowledge server
unless the user asks you to. Clients read their server list at startup, so a
server added during a session appears only after the client restarts.

**Verify:** `elin_versions` takes no arguments and returns the game version
behind each channel. `elin_instances` answers "No game instance is listening"
while the game is closed, and `elin_launch` starts it. The runtime server sends
its own usage notes when it connects, `README.md` in the mod folder is its full
reference, and §4 lists what you need to plan with.

---

## 2. The one rule

**Never read an Elin source file whole.** The decompiled tree has several
100 KB-class god classes:

| File | Size | Cost if read whole |
|---|---|---|
| `Chara.cs` | 248 KB | ~67k tokens |
| `Card.cs` | 167 KB | ~45k tokens |
| `BaseTileMap.cs` | 116 KB | ~31k tokens |
| `Zone.cs` | 99 KB | ~27k tokens |
| `ActEffect.cs` | 85 KB | ~23k tokens |

`elin_outline` lists a class's members for a fraction of that (about 8k tokens
for `Chara`, 4k for `Trait`). The server has no whole-file read at all — `elin_read` only accepts names the index returned and
caps one call at 400 lines. The rule holds just as much for a local decompile
(§8): locate the member first, then read only its lines.

---

## 3. Knowledge tools

Prose comes back in `en` (default), `zh` or `ja` — pass `lang`.

### Knowledge — start here for "how do I make a mod"

| Tool | Args | What it gives you |
|---|---|---|
| `elin_guide` | `task?`, `lang` | **The entry point.** Turns an intent into an ordered route: which wiki articles, which sheets, which types. No arg = list all routes. |
| `elin_wiki` | `q`, `lang`, `limit` | Full-text search over the modding wiki — 55 base-game articles, each in EN/ZH/JA. Searches all three languages at once and returns results in `lang`. |
| `elin_doc` | `path`, `lang`, `section?` | Read an article. Long ones return a section outline first — then re-call with `section`. |
| `elin_sheet` | `sheet`, `channel` | Column definitions for a source sheet (`thing`, `chara`, `race`, …), read from the game's own `Source*.Row` classes. |
| `elin_ids` | `category`, `q?`, `channel`, `limit` | Which ids actually exist. Searchable by EN/JP/CN name. |
| `elin_scaffold` | `kind`, `name`, `author` | Start a mod. `kind: "script"` = the `dotnet new` commands for the official plugin template, plus the `package.xml` it leaves you to write; `"sheet"` = data-only layout. |
| `elin_setup` | — | The setup procedure for both servers and both skills — the same text as `setup.md` next to this file. |

Routes available from `elin_guide`:
`first-mod` · `sheet-mod` · `script-mod` · `harmony` · `transpiler` · `custom-content` ·
`texture` · `sound` · `i18n` · `elin-api` · `troubleshoot`

(`cwl` still finds `custom-content` — Custom Whatever Loader was merged into the base
game, and the route says so. Pass the id when calling with an exact route in mind.)

### Source — for "how is this implemented"

| Tool | Args | What it gives you |
|---|---|---|
| `elin_find` | `q`, `kind`, `channel`, `limit` | Locate a type or member. Accepts `Chara.Die`, bare `Die`, or a fragment. `kind`: `any`\|`type`\|`method`\|`prop`\|`field`\|`event`\|`enumval`. |
| `elin_outline` | `type`, `kind`, `channel`, `relpath?` | Every member of a type with signatures, no bodies. |
| `elin_hierarchy` | `type`, `direction`, `channel`, `limit` | `down` = who inherits it, `up` = what it inherits. |
| `elin_callers` | `symbol`, `direction`, `channel`, `limit` | `callers` = where a method is called from; `callees` = what a member calls. |
| `elin_strings` | `q?`, `owner?`, `exact`, `channel`, `limit` | String literals hard-coded in the source, with the member each sits in. |
| `elin_read` | `fqn`, `channel`, `relpath?`, `from?` | One member's implementation, sliced by line. Max 400 lines per call. |
| `elin_versions` | — | Which game version each channel is indexed at. |

When a name is declared more than once (another assembly, or another overload in
the same file), `elin_outline` and `elin_read` say so and list each one as
`relpath:line`; pass that string as `relpath` to read that one. A body longer
than 400 lines ends with a truncation note; call again with the `from` it gives.
If the server you are connected to predates these two options, read the line
range `elin_find` reports from a local decompile (§8) instead.

Every tool that reads the index also takes `channel`: `stable` (the default) or
`nightly`. Those are Elin's two branches and the only two indexes that exist —
nightly builds get merged into stable periodically, and no older version is kept
around, so there is nothing to pin to. Pass `nightly` only when the players you
target run the beta branch. Ask for a channel that has no index and you get the
other one, with a warning in the response — never a silent substitution.

`elin_find` searches **names**; `elin_strings` searches **values**. A sound id
like `openDoor` is not the name of anything — it exists only as a string inside
`TraitDoor.TryOpen`, so no amount of `elin_find` will surface it. Same for lang
keys, source-sheet column names (`soundFoot`), custom-content tags, and override
keys (`portrait_override`). Reach for `elin_strings` whenever the thing you are
hunting is a value the game compares against.

---

## 4. Runtime tools

| Tool | What it does |
|---|---|
| `elin_instances` | Running instances: port, pid, scene, game version, plus fields other mods add |
| `elin_eval` | C# script on the Unity main thread, compiled by the game's own Roslyn |
| `elin_wait_eval` | Poll a C# expression until truthy; in-game exceptions count as false |
| `elin_command` | An in-game console command (`mod.elin_dev true` enables debug cheats) |
| `elin_state` | Scene, zone and player as JSON |
| `elin_screenshot` | The current frame as an inline PNG |
| `elin_hotload` / `elin_hotunload` | Compile one `.cs` file in the game and load it as a throwaway assembly |
| `elin_logs` / `elin_wait_log` | The instance's BepInEx and Unity log since process start; wait for a line |
| `elin_build` | `dotnet build`, returning only error and warning lines |
| `elin_launch` / `elin_kill` | Start the game through Steam (optionally in a Sandboxie box) / stop it |

What to plan around:

- **`elin_eval` freezes the game while it runs.** Qualify game statics
  (`EClass.pc`, `EClass._zone`, `EClass.game`, `EClass.core`). Return
  `JsonConvert.SerializeObject(...)` for structure; anything non-primitive
  otherwise comes back as `ToString()`. `Script["key"]` keeps values between
  calls. Never `await`, never loop waiting for game state inside a script — poll
  with `elin_wait_eval`.
- **Most of the game needs a loaded save.** The title screen is enough for
  reflection and console commands; anything touching the player is not. List
  saves with
  `JsonConvert.SerializeObject(GameIO.GetGameList(CorePath.RootSave).Select(g => new { g.id, g.pcName }))`,
  load one with `Game.Load("<id>", false)`, then `elin_wait_eval`
  `EClass.core.IsGameStarted`.
- **A new build needs a restart.** The game does not lock mod DLLs, so
  `elin_build` works while it runs, but the running game keeps the code it
  loaded: `elin_kill`, then `elin_launch`.
- **`elin_hotload` is for quick experiments.** It takes a `.cs` file or the
  source as a string. Call `new Harmony("some.id").PatchAll(...)` from a static
  `EScriptLoad()` and `harmony.UnpatchSelf()` from `EScriptUnload()`. Never call
  the static `Harmony.UnpatchAll()`: it removes every mod's patches. An exception
  in `EScriptLoad()` comes back as an error with its real message. Mono cannot
  unload assemblies, so every load stays until restart and a name can be used
  once per session.
- **A timed-out call still finishes in the game.** Do not blindly retry a script
  with side effects.

---

## 5. Which source to trust

| Question | Go to |
|---|---|
| How do I make X (packaging, sheets, loaders, templates) | `elin_guide`, then the articles it names |
| Where is X implemented; who calls it | `elin_find` → `elin_callers` → `elin_read` |
| Which ids or sheet columns exist | `elin_ids`, `elin_sheet` |
| Exact signatures and overloads in **the installed game** | Reflection in `elin_eval` (below) |
| What happens at runtime; does my patch fire | `elin_eval`, `elin_logs`, `elin_screenshot` |
| Regex over the whole tree, offline work, a type the index lacks | Local decompile (§8) |

Reflection gives the signatures of whatever version the user actually runs,
including overloads declared on base classes and the parameter names Harmony
injects by:

```csharp
string.Join("\n", typeof(Chara).GetMethods(AccessTools.all).Where(m => m.Name == "DamageHP").Select(m => m.DeclaringType.Name + "." + m.Name + "(" + string.Join(", ", m.GetParameters().Select(p => p.ParameterType.Name + " " + p.Name)) + ")"))
```

**Version drift.** Compare `elin_versions` with the game version
`elin_instances` reports. `elin_instances` prints `EA 23.349`, `elin_versions`
prints `23.348.1`; compare the middle number (the batch), the third part is a
hotfix counter. When the installed game is newer than both indexed channels,
trust reflection for signatures and a local decompile for method bodies, and
tell the user the index lags (§9).

---

## 6. Working order

**Building or changing a mod** — always start at `elin_guide`. Source search
alone will not produce a working mod: the packaging, loading and sheet
conventions are not expressed in the code.

```
elin_guide task="add a new item"
  → elin_doc <the articles it names>
  → elin_sheet sheet="thing"        # what the columns mean
  → elin_ids category="category"    # what values are legal
  → elin_scaffold kind="sheet" name="..." author="..."
```

**Starting a C# script mod** — the project comes from the official `dotnet new`
template. It carries the reference set, the build configurations, and a build
step that copies your assets next to the DLL, and it is maintained alongside the
game. The wiki's own "Setup Script Mod Project" article hands this step to the
template rather than describing it.

```pwsh
dotnet new install ElinPluginTemplate      # once; add --force to update later
dotnet new elinplugin -n MyMod --Guid "your.unique.mod.id" --ModName "My Mod"
cd MyMod
dotnet build
```

Prerequisites: **.NET SDK 10.0 or newer**, and an **`ElinGamePath` environment
variable** pointing at the game root — the folder holding `BepInEx\` and
`Elin_Data\`. Game references resolve through it. On Windows the template falls
back to the default Steam folder (`C:\Program Files (x86)\Steam\steamapps\common\Elin`)
when it is unset, so any other install location needs it. Rider and Visual
Studio list the same template in their New Project dialog,
with Guid / ModName / Version as form fields.

`dotnet build` writes straight into `<ElinGamePath>\Package\Mod_<AssemblyName>\`
and copies the whole `package\` tree beside the DLL, so the mod is loadable with
no manual copying. Two things the template leaves to you:

- **`package\package.xml`** — `package\` is generated empty, and a package
  without that file is not a mod to the loader (stable drops it silently,
  nightly logs `Not a package:`). Its `<id>` must match `ModInfo.Guid` in
  `Plugin.cs`.
- **Your patches** — `Plugin.Awake` already calls `Harmony.CreateAndPatchAll`
  over the assembly, so a new `[HarmonyPatch]` class needs no registration.

`elin_scaffold kind="script"` returns these commands with your names filled in,
plus both of those files, with the id already consistent across them.

**Understanding or patching game behaviour** — narrow before reading.

```
elin_find q="Chara.Die"          # confirm the exact signature
elin_hierarchy type="Chara"      # base class or concrete subclass?
elin_callers symbol="Die"        # when does it actually run?
elin_read fqn="Chara.Die"        # only now, read the body
```

Skipping `elin_callers` is the single most common cause of a Harmony patch that
compiles, loads, and silently never fires.

**Proving a patch works** — with the runtime server, do not stop at "it
compiles":

```
elin_hotload file="Probe.cs"                 # the patch, or a logging prefix on the target
elin_eval / elin_command                     # trigger the code path
elin_wait_log pattern="<what the patch logs>"
elin_hotunload name="<assembly name>"
```

For the real mod: `elin_build`, `elin_kill`, `elin_launch`, load a save, then
the same trigger-and-watch loop. `elin_logs level="Warning" group=true` after a
session shows every repeated error at a glance.

**When a mod does not work** — `elin_guide task="troubleshoot"` gives the
decision tree, keyed to what the loader actually logs. The short version: the
game announces most load decisions, so the job is knowing which string to search
for.

| Search for | Meaning |
|---|---|
| `Activating(` | Activation was **attempted**. It is logged before activation runs, so a mod listed here can still have failed. |
| `Failed to activate mod` | Activation threw. Stable: `Failed to activate mod: <title>, reason: <reason>`; nightly: `Failed to activate mod <title>: <reason>`. |
| `Skipped (` | Nightly only: made for an older game version, or a dependency is missing. |
| `Blocked:` | Nightly only: incompatible with another installed mod. |
| `Not a package:` / `No <id> in package.xml` | Nightly only: no `package.xml`, or one without an `<id>`. Stable skips both silently. |
| `does not parse` | Nightly only: broken `package.xml`; the game then refuses to start (`#mod game will not start`). |
| `Source Sheets:`, `Texture:`, `Sound:`, … | Per-content counts after activation: whether your files were recognised at all. |

With the runtime server, `elin_logs contains="<string>"` searches all of that
directly, including the lines from before the runtime mod itself loaded. Without
it, read `Player.log`, which sits next to the `Save/` folder. BepInEx's
`LogOutput.log` does not carry the game's own lines.

---

## 7. Gotchas the code will not tell you

- **All game logic is in `Elin.dll`.** Unity's default `Assembly-CSharp.dll` is a
  6 KB leftover holding a few demo effect scripts, so `Elin.dll` is what you
  decompile and what you go looking through. The game is also split further than
  that: `Lang` and `EInput` are in `Plugins.BaseCore`, the modding API in
  `Plugins.Modding` — neither is in the main assembly.
- **Set `ElinGamePath` unless the game is in the default Steam folder.** It is
  an environment variable pointing at the game root (the folder with `BepInEx\`
  and `Elin_Data\`), and every game reference resolves through it.
- **`elin_callers` matches by method name only.** The index is syntax-only, so
  there is no overload resolution and no receiver type. Same-named methods on
  different types share results — read the caller column to separate them.
  Reflection and delegate invocations are invisible to it entirely.
- **Everything a mod ships goes under `package\`** — `package.xml`,
  `preview.jpg`, `LangMod\`, `Texture\`, `Sound\`. That tree is what gets copied
  next to the DLL, and a package whose `package.xml` never arrives is not
  recognised as a mod at all.
- **`<version>` in `package.xml` is a compatibility floor, not your mod's
  version.** The loader scores it as `major*1000000 + minor*1000 + batch` and
  skips any package below the game's own minimum, before `Activate()`; the mod
  sits in the list, red and unclickable. On stable nothing is logged and a
  version with fewer than three parts (`1.0`) parses as zero, which fails every
  comparison. Nightly accepts short versions and logs `Skipped (made for game
  <version>)`. Write three parts; `1.0.0` is a safe floor on both.
- **Never reuse an existing id.** For any cross-reference field (race, job,
  trait, category…), check with `elin_ids` first — pointing at an id that does
  not exist is the most common source-sheet failure.
- **MCP cannot emit `.xlsx`** (it is binary). For sheet mods, copy the original
  sheet out of the game's own package, keep the header row, delete the data
  rows, add yours.
- **A rebuilt DLL does nothing until the game restarts.** Building over a
  running game succeeds silently; the old code keeps running.

---

## 8. Decompiling the game locally

The knowledge server already serves the decompiled source — indexed, sliced, and
searchable — and the runtime server answers "what does my installed version
have" through reflection. You need a local copy only for:

- regex sweeps over the whole tree
- method **bodies** of a game version newer than both indexed channels
- go-to-definition and rename-preview inside an IDE
- offline work

### Decompile the same assemblies the index covers

Needs the .NET 10 SDK: current `ilspycmd` releases ship for .NET 10 only, and
`-lv CSharp14_0` below needs `ilspycmd` 10 or newer.

```bash
dotnet tool install --global ilspycmd
```

The index is built from these fourteen assemblies, one output folder each, so
the paths the knowledge server returns (`Elin/Chara.cs`,
`Plugins.BaseCore/…`, `ElinModdingKit/EModding/…`) have the same shape in your
copy:

| Assembly | Where | Contents |
|---|---|---|
| `Elin` | `Elin_Data/Managed` | **All game logic.** This is the Harmony target. |
| `Plugins.BaseCore` | `Elin_Data/Managed` | `Lang`, `EInput`, `BaseModManager` (the mod event bus) |
| `Plugins.Modding` | `Elin_Data/Managed` | The official modding API: `EMod`, `ICustomContent`, `PackageIterator`, `SourceCache` |
| `Plugins.UI`, `Plugins.Scripting`, `Plugins.Util`, `Plugins.Calc`, `Plugins.Sound`, `Plugins.ActorSystem`, `Plugins.NameGen`, `Plugins.NoiseSystem`, `Plugins.Tween` | `Elin_Data/Managed` | UI, runtime C#, utilities, formulas, sound, actors, names, noise, tweens |
| `ClassLibrary2` | `Elin_Data/Managed` | `AK.ExpressionSolver`, which evaluates the formula strings in source sheets |
| `ElinModdingKit` | `Package/_ModdingKit` | The built-in modding kit: script compiler, `CodeMatchExt`, `AccessorHelper` |

PowerShell:

```pwsh
$Game = "C:\Program Files (x86)\Steam\steamapps\common\Elin"   # or $env:ElinGamePath
$Out  = ".\Elin-Decompiled"
$Managed = "$Game\Elin_Data\Managed"; $Kit = "$Game\Package\_ModdingKit"; $Core = "$Game\BepInEx\core"
$Asms = 'Elin','Plugins.BaseCore','Plugins.UI','Plugins.Modding','Plugins.Scripting','Plugins.Util','Plugins.Calc',
        'Plugins.Sound','Plugins.ActorSystem','Plugins.NameGen','Plugins.NoiseSystem','Plugins.Tween','ClassLibrary2'
foreach ($a in $Asms + 'ElinModdingKit') {
    $dll = if ($a -eq 'ElinModdingKit') { "$Kit\$a.dll" } else { "$Managed\$a.dll" }
    ilspycmd -p -o "$Out\$a" -r $Managed -r $Kit -r $Core -lv CSharp14_0 --disable-updatecheck $dll
}
```

bash:

```bash
GAME="/c/Program Files (x86)/Steam/steamapps/common/Elin"   # or "$ElinGamePath"
OUT=./Elin-Decompiled
M="$GAME/Elin_Data/Managed"; KIT="$GAME/Package/_ModdingKit"; CORE="$GAME/BepInEx/core"
for a in Elin Plugins.BaseCore Plugins.UI Plugins.Modding Plugins.Scripting Plugins.Util Plugins.Calc \
         Plugins.Sound Plugins.ActorSystem Plugins.NameGen Plugins.NoiseSystem Plugins.Tween ClassLibrary2 ElinModdingKit; do
  dll="$M/$a.dll"; [ "$a" = ElinModdingKit ] && dll="$KIT/$a.dll"
  ilspycmd -p -o "$OUT/$a" -r "$M" -r "$KIT" -r "$CORE" -lv CSharp14_0 --disable-updatecheck "$dll"
done
```

- `-p` — a project layout, one file per type, not one giant file
- `-r` — where references resolve. All three folders matter: the modding kit
  references Roslyn in its own folder and Harmony in `BepInEx/core`; without them
  signatures show bare assembly-qualified names instead of types
- `-lv CSharp14_0` — the language version the index is decompiled with

The whole set takes well under a minute. Elin is a **Mono** build, so this is
all you need. There is no IL2CPP metadata step and no Cpp2IL — if a guide tells
you to dump `global-metadata.dat`, it is about a different game.

Line numbers match the index only when your `ilspycmd` version matches the one
it was built with. Resolve a symbol with `elin_find`, then search your copy for
that **name**, not for a line number.

A public dump exists at
[`Elin-Modding-Resources/Elin-Decompiled`](https://github.com/Elin-Modding-Resources/Elin-Decompiled),
but it only holds `Elin`, `Plugins.BaseCore` and `Plugins.UI`. The modding API in
`Plugins.Modding` is missing, so a lookup like `CustomContent : ICustomContent`
dead-ends there. Decompile yourself when you need more than casual browsing.

The [ILSpy GUI](https://github.com/icsharpcode/ILSpy) is better for one-off
browsing: open `Elin.dll`, and its Analyze panel finds callers and implementors
interactively.

### Searching a local copy

Locate first, then read a slice — the rule from §2 applies here too:

```bash
rg -n "class Chara\b" Elin-Decompiled/Elin            # the declaration and its file
rg -n "\bvoid Die\(" Elin-Decompiled/Elin/Chara.cs    # the member's line
rg -n "\.Die\(" Elin-Decompiled -g "*.cs"             # call sites, when elin_callers is unavailable
```

Then read about 60 lines from the hit (with `grep -n` or `Select-String` if
ripgrep is not installed).

### What decompiled code is and is not

It is faithful to what executes, not to what was written:

- **No comments.** They do not survive compilation.
- **Lambdas and closures become generated classes** with names like
  `<>c__DisplayClass12_0`. `yield return` methods become state machines; `async`
  methods become `MoveNext` bodies.
- **Control flow may be restructured** — `switch` can reappear as `if` chains,
  loops can be rotated.
- **The output does not necessarily compile,** and is not meant to. It is for
  reading.

Decompiling a game you own for interoperability and modding is what these tools
are for. Publishing the decompiled output as your own work is not — ship your
mod, not the game's code.

---

## 9. Reporting an index gap

If your channel in `elin_versions` lags your installed game, or a symbol you can
see through reflection or in your own dump is missing from `elin_find`, the
index needs a rebuild — say so rather than silently working around it. The
server is `https://mcp.elin-modding.net`, and it is maintained alongside the
other tooling under
[Elin-Modding-Resources](https://github.com/Elin-Modding-Resources).
