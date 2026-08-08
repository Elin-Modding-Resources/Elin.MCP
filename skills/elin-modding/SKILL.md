---
name: elin-modding
description: Build and debug mods for the game Elin using the Elin MCP server (mcp.elin-modding.net) — source index, modding wiki, source-sheet schemas, game ids, and project scaffolding. Use whenever the user mentions Elin modding, BepInEx/Harmony patches for Elin, Elin source sheets (Thing/Chara/Race xlsx), Elin package.xml, or asks how something in Elin is implemented. Also covers decompiling the game locally with ILSpy.
---

# Elin Modding

Elin is a Unity (Mono, .NET Framework 4.7.2) roguelike whose modding surface is
undocumented in-code: the packaging rules, sheet formats and loader conventions
live in a community wiki, and the game logic lives in a decompilable assembly.

The **Elin MCP server** puts both behind twelve tools. No install, no game files,
no API key.

---

## 1. Setup

Endpoint: `https://mcp.elin-modding.net` — stateless Streamable HTTP, no auth.
Both `/` and `/mcp` work.

**Claude Code**

```bash
claude mcp add --transport http elin https://mcp.elin-modding.net
```

**Codex** — `~/.codex/config.toml`

```toml
[mcp_servers.elin]
url = "https://mcp.elin-modding.net"
```

**OpenCode** — `opencode.json`

```json
{
  "mcp": {
    "elin": { "type": "remote", "url": "https://mcp.elin-modding.net", "enabled": true }
  }
}
```

**Cursor** — `.cursor/mcp.json` · **Zed**, **Windsurf**, most others

```json
{
  "mcpServers": {
    "elin": { "type": "http", "url": "https://mcp.elin-modding.net" }
  }
}
```

**VS Code / Copilot** — `.vscode/mcp.json` uses `servers` instead of `mcpServers`:

```json
{ "servers": { "elin": { "type": "http", "url": "https://mcp.elin-modding.net" } } }
```

MCP client config formats drift between releases. The shape is always the same:
a name, an HTTP/remote transport, and the URL. If your client only speaks stdio,
bridge it:

```json
{ "mcpServers": { "elin": { "command": "npx", "args": ["-y", "mcp-remote", "https://mcp.elin-modding.net"] } } }
```

**Verify:** call `elin_versions`. It takes no arguments and returns the indexed
game versions. If that works, everything works.

---

## 2. The one rule

**Never read an Elin source file whole.** The tree has several 100 KB+ god
classes:

| File | Size | Cost if read whole |
|---|---|---|
| `Trait.cs` | 311 KB | ~85k tokens |
| `Chara.cs` | 211 KB | ~58k tokens |
| `Act.cs` | 184 KB | ~50k tokens |
| `Card.cs` | 167 KB | ~46k tokens |

`elin_outline` gives you the same class in ~2k tokens. The server has no
whole-file read at all — `elin_read` only accepts names the index returned and
caps one call at 400 lines. If you also have the source on disk (§6), the rule
still holds: locate through MCP, then slice by line.

---

## 3. Tools

Prose comes back in `en` (default), `zh` or `ja` — pass `lang`.
Every source tool takes an optional `ver`; omit it to get the newest index.

### Knowledge — start here for "how do I make a mod"

| Tool | Args | What it gives you |
|---|---|---|
| `elin_guide` | `task?`, `lang` | **The entry point.** Turns an intent into an ordered route: which wiki articles, which sheets, which types. No arg = list all routes. |
| `elin_wiki` | `q`, `lang`, `limit` | Full-text search over the modding wiki — ~103 articles, each in EN/ZH/JA. Returns paths. |
| `elin_doc` | `path`, `lang`, `section?` | Read an article. Long ones return a section outline first — then re-call with `section`. |
| `elin_sheet` | `sheet`, `ver?` | Column definitions for a source sheet (`thing`, `chara`, `race`, …), read from the game's own `Source*.Row` classes. |
| `elin_ids` | `category`, `q?`, `limit` | Which ids actually exist. Searchable by EN/JP/CN name. |
| `elin_scaffold` | `kind`, `name`, `author` | Generate a mod skeleton. `kind: "script"` = compilable BepInEx/Harmony project; `"sheet"` = data-only layout. |

Routes available from `elin_guide`:
`first-mod` · `sheet-mod` · `script-mod` · `harmony` · `transpiler` · `cwl` ·
`texture` · `sound` · `i18n` · `elin-api`

### Source — for "how is this implemented"

| Tool | Args | What it gives you |
|---|---|---|
| `elin_find` | `q`, `kind`, `limit` | Locate a type or member. Accepts `Chara.Die`, bare `Die`, or a fragment. `kind`: `any`\|`type`\|`method`\|`prop`\|`field`\|`event`\|`enumval`. |
| `elin_outline` | `type`, `kind` | Every member of a type with signatures, no bodies. |
| `elin_hierarchy` | `type`, `direction` | `down` = who inherits it, `up` = what it inherits. |
| `elin_callers` | `symbol`, `direction`, `limit` | `callers` = where a method is called from; `callees` = what a member calls. |
| `elin_read` | `fqn` | One member's implementation, sliced by line. Max 400 lines. |
| `elin_versions` | — | Which game versions are indexed. |

---

## 4. Working order

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

**Understanding or patching game behaviour** — narrow before reading.

```
elin_find q="Chara.Die"          # confirm the exact signature
elin_hierarchy type="Chara"      # base class or concrete subclass?
elin_callers symbol="Die"        # when does it actually run?
elin_read fqn="Chara.Die"        # only now, read the body
```

Skipping `elin_callers` is the single most common cause of a Harmony patch that
compiles, loads, and silently never fires.

---

## 5. Gotchas the code will not tell you

- **The main assembly is `Elin.dll`**, not Unity's default
  `Assembly-CSharp.dll`. Reference the wrong one and nothing resolves.
- **Target `net472`.** The game runs on Mono. Any .NET Core / .NET 5+ target
  fails to load, usually with no useful error.
- **Every game DLL reference must be `Private=false`.** Otherwise it is copied
  into the mod folder and the runtime ends up holding two distinct instances of
  the same types — casts then fail for no visible reason.
- **`elin_callers` matches by method name only.** The index is syntax-only, so
  there is no overload resolution and no receiver type. Same-named methods on
  different types share results — read the caller column to separate them.
  Reflection and delegate invocations are invisible to it entirely.
- **`package.xml` must reach the output folder** or the game does not recognise
  the build as a mod.
- **Never reuse an existing id.** For any cross-reference field (race, job,
  trait, category…), check with `elin_ids` first — pointing at an id that does
  not exist is the most common source-sheet failure.
- **MCP cannot emit `.xlsx`** (it is binary). For sheet mods, copy the original
  sheet out of the game's own package, keep the header row, delete the data
  rows, add yours.

---

## 6. Decompiling the game locally with ILSpy

The MCP server already serves the decompiled source — indexed, sliced, and
searchable. You do **not** need a local copy for ordinary work.

Get one when you need something the index cannot give you:

- regex sweeps over the whole tree, or across several versions
- go-to-definition and rename-preview inside an IDE
- your installed game is **newer than the indexed version** — check with
  `elin_versions` before assuming the index is current
- offline work

### Option A — use an existing dump

A public decompile of the game is maintained at
[`Elin-Modding-Resources/Elin-Decompiled`](https://github.com/Elin-Modding-Resources/Elin-Decompiled).
Cloning it is
faster than decompiling, and it is the same tree the MCP index is built from, so
paths and line numbers line up with what the tools return.

### Option B — decompile it yourself with `ilspycmd`

Needs the .NET SDK (8.0 or newer).

```bash
dotnet tool install --global ilspycmd
```

Elin's managed assemblies live under the install directory — on Steam that is
`steamapps/common/Elin/Elin_Data/Managed/`:

```bash
GAME="/c/Program Files (x86)/Steam/steamapps/common/Elin"

ilspycmd -p -o ./Elin-Decompiled \
  -r "$GAME/Elin_Data/Managed" \
  "$GAME/Elin_Data/Managed/Elin.dll"
```

- `-p` — emit a compilable project layout (`.csproj` + folders), not one giant file
- `-o` — output directory
- `-r` — where to resolve references; point it at `Managed/` or half the types
  decompile as `object`

Elin is a **Mono** build, so this is all you need. There is no IL2CPP metadata
step and no Cpp2IL — if a guide tells you to dump `global-metadata.dat`, it is
about a different game.

Worth decompiling alongside `Elin.dll`:

| Assembly | Contents |
|---|---|
| `Elin.dll` | **All game logic.** This is the Harmony target. |
| `ClassLibrary2.dll` | Shared helpers the game code calls into |
| `Plugins/*.dll` | Third-party libraries, rarely worth patching |

### Option C — ILSpy GUI

Download from [icsharpcode/ILSpy](https://github.com/icsharpcode/ILSpy) (Windows;
ILSpy AvaloniaUI for macOS/Linux). Open `Elin.dll`, then
**File → Save Code As…** and pick "Project" to export the same tree the CLI
produces. The GUI is better for one-off browsing — the "Analyze" panel finds
callers and implementors interactively.

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

Consequently: **use MCP for names, your local dump for text.** Unless you cloned
Option A, the line numbers in the index will not match your own output — a
different ILSpy version or a different `-lv` language-version setting reshapes
the file. Resolve a symbol with `elin_find`, then search your dump for that
*name*.

Decompiling a game you own for interoperability and modding is what these tools
are for. Publishing the decompiled output as your own work is not — ship your
mod, not the game's code.

---

## 7. Reporting an index gap

If `elin_versions` lags your installed game, or a symbol you can see in your own
dump is missing from `elin_find`, the index needs a rebuild — say so rather than
silently working around it. The server is `https://mcp.elin-modding.net`, and it
is maintained alongside the other tooling under
[Elin-Modding-Resources](https://github.com/Elin-Modding-Resources).
