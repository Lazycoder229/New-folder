# Trip Language — Desktop Application Roadmap

Since Trip runs on **Node.js inside Electron**, you have full access to the
OS, file system, native processes, and all Node built-ins. The goal below
is to turn Trip into a language that feels natural for writing **desktop app
logic** — scripting windows, reading/writing files, spawning processes,
reacting to events, and talking to the OS.

---

## What You Already Have ✅

- Variables (`let`, `const`), functions, classes, inheritance
- Control flow (`if/elif/else`, `while`, `for`, `match`)
- Error handling (`try/catch/finally`, `throw`)
- f-strings, ternary, null coalescing (`??`), `in`, `is`
- Generators (`yield`), decorators (`@`), async/await (stub)
- stdlib: `file`, `datetime`, `math`, `string`, `array`
- Import system (`import "stdlib/array"`)

---

## Phase 1 — Electron / OS Bridge 🖥️ *(Do this first)*

These let Trip scripts actually talk to the desktop. Without this, Trip is
just a general scripting language with no desktop identity.

### 1. `os` built-in module

Expose Node's `os` module as a Trip dict:

```trip
let info = os.platform()   # "win32" / "darwin" / "linux"
let home = os.homedir()
let cpu  = os.cpus()
let mem  = os.freemem()
```

**Add to `interpreter.js` → `buildStdLib`:**
```js
const os = require('os')
const osObj = new Map()
osObj.set('platform',  () => os.platform())
osObj.set('homedir',   () => os.homedir())
osObj.set('tmpdir',    () => os.tmpdir())
osObj.set('hostname',  () => os.hostname())
osObj.set('freemem',   () => os.freemem())
osObj.set('totalmem',  () => os.totalmem())
osObj.set('cpus',      () => os.cpus().length)
env.define('os', osObj)
```

---

### 2. `path` built-in module

Desktop scripts constantly join and split paths:

```trip
let full = path.join(os.homedir(), "Documents", "notes.txt")
let ext  = path.ext("report.pdf")       # ".pdf"
let base = path.basename("/a/b/c.tp")   # "c.tp"
let dir  = path.dirname("/a/b/c.tp")    # "/a/b"
```

```js
const nodePath = require('path')
const pathObj = new Map()
pathObj.set('join',     (...parts) => nodePath.join(...parts))
pathObj.set('resolve',  (...parts) => nodePath.resolve(...parts))
pathObj.set('basename', (p, ext)   => nodePath.basename(p, ext))
pathObj.set('dirname',  (p)        => nodePath.dirname(p))
pathObj.set('ext',      (p)        => nodePath.extname(p))
pathObj.set('sep',      nodePath.sep)
env.define('path', pathObj)
```

---

### 3. Expand `file` module

Current `file` only has read/write/append/exists/delete/lines.
Desktop scripts need directory operations:

```trip
file.mkdir("output/reports")
let items = file.readdir("/home/user/docs")   # returns array of names
file.copy("a.txt", "b.txt")
file.move("old.txt", "new.txt")
file.stat("notes.txt")   # { size, modified, isDir }
```

```js
fileObj.set('mkdir',   (p)     => { fs.mkdirSync(p, { recursive: true }); return null })
fileObj.set('readdir', (p)     => fs.readdirSync(p))
fileObj.set('copy',    (src, dst) => { fs.copyFileSync(src, dst); return null })
fileObj.set('move',    (src, dst) => { fs.renameSync(src, dst); return null })
fileObj.set('stat',    (p)     => {
  const s = fs.statSync(p)
  const m = new Map()
  m.set('size', s.size); m.set('isDir', s.isDirectory())
  m.set('modified', s.mtimeMs)
  return m
})
```

---

### 4. `proc` built-in — run shell commands

```trip
let result = proc.run("git status")
let lines  = proc.lines("ls -la")
proc.spawn("code", ["."])       # fire and forget
```

```js
const { execSync, spawnSync, spawn } = require('child_process')
const procObj = new Map()
procObj.set('run',   (cmd)        => execSync(cmd, { encoding: 'utf8' }))
procObj.set('lines', (cmd)        => execSync(cmd, { encoding: 'utf8' }).split('\n'))
procObj.set('spawn', (cmd, args)  => { spawn(cmd, args ?? [], { detached: true, stdio: 'ignore' }).unref(); return null })
env.define('proc', procObj)
```

---

### 5. `env` built-in — environment variables

```trip
let home  = env.get("HOME")
let token = env.get("API_KEY") ?? "none"
env.set("DEBUG", "true")
```

```js
const envObj = new Map()
envObj.set('get', (k)    => process.env[k] ?? null)
envObj.set('set', (k, v) => { process.env[k] = v; return null })
envObj.set('all', ()     => {
  const m = new Map()
  for (const [k, v] of Object.entries(process.env)) m.set(k, v)
  return m
})
env.define('env', envObj)
```

---

## Phase 2 — Real Async I/O 🔄

Right now `async/await` in Trip is a stub — it executes synchronously.
For desktop apps (reading large files, calling APIs, waiting for processes),
you need real async.

### Plan: Make the interpreter async-capable

This is the biggest change. Two options:

**Option A (Simpler) — Async stdlib only**
Keep the interpreter sync but add special async builtins that use
`execSync` / sync Node APIs under the hood. Good enough for 90% of
desktop scripting.

**Option B (Full async) — Async interpreter**
Change `execBlock` / `eval` to return Promises, use `await` properly.
This is a larger refactor but makes Trip feel modern.

**Recommended: Start with Option A**, then migrate to B later.

For now, expose these sync-friendly async-looking APIs:

```trip
async fn readLargeFile(p):
    let data = await file.read(p)   # already sync under hood
    return data
```

---

## Phase 3 — Desktop stdlib Modules 📦

Add these as importable stdlib files (`.tp` files, like your existing `array.tp`):

### `stdlib/fs` — filesystem helpers

```trip
import "stdlib/fs"

let files = fs.glob("src/**/*.tp")
let json  = fs.readJson("config.json")
fs.writeJson("output.json", data)
let exists = fs.exists("notes.txt")
```

Implement as a `.tp` file wrapping the `file` builtin, plus a JSON helper
backed by a new `json` builtin in the interpreter:

```js
// In buildStdLib:
const jsonObj = new Map()
jsonObj.set('parse',     (s) => jsonToTrip(JSON.parse(s)))
jsonObj.set('stringify', (v) => JSON.stringify(tripToJs(v), null, 2))
env.define('json', jsonObj)
```

---

### `stdlib/net` — HTTP requests

Desktop apps often call REST APIs. Use Node's built-in `https`:

```trip
import "stdlib/net"

let resp = net.get("https://api.github.com/users/octocat")
let data = json.parse(resp)
out(data["name"])
```

```js
// In buildStdLib (sync HTTP using execSync trick or node-fetch sync wrapper):
const netObj = new Map()
netObj.set('get', (url) => {
  // Use child_process to call curl synchronously as a bridge
  const { execSync } = require('child_process')
  return execSync(`curl -s "${url}"`, { encoding: 'utf8' })
})
env.define('net', netObj)
```

---

### `stdlib/config` — INI / JSON config files

Very common pattern in desktop tools:

```trip
import "stdlib/config"

let cfg = config.load("settings.json")
cfg["theme"] = "dark"
config.save("settings.json", cfg)
```

---

## Phase 4 — Language Features for Desktop Patterns 🧠

These language additions make desktop scripting more expressive.

### 1. Multi-line strings (heredoc)

```trip
let sql = """
  SELECT *
  FROM users
  WHERE active = true
"""
```

Add to lexer: when `"""` is seen, collect until next `"""`.

---

### 2. Pipe operator `|>`

Very natural for file/data processing chains:

```trip
let result = file.read("data.csv")
    |> split("\n")
    |> filter(fn(l): return len(l) > 0)
    |> map(fn(l): return split(l, ","))
```

Add `PipeExpr` AST node in parser: `a |> b` → `b(a)`.

---

### 3. Pattern destructuring

```trip
let [first, ...rest] = items
let { name, age }    = person
```

---

### 4. `repeat` / `every` / `after` — timers

```trip
after(1000, fn():
    out("1 second passed")
)

every(5000, fn():
    out("tick")
)
```

```js
env.define('after', (ms, fn) => { setTimeout(() => interp.callFunction(fn, []), ms); return null })
env.define('every', (ms, fn) => { setInterval(() => interp.callFunction(fn, []), ms); return null })
```

---

## Phase 5 — Electron-Specific Bridge 🔌

Since Trip runs inside Electron, expose the Electron APIs to Trip scripts:

### `app` built-in (main process only)

```trip
let ver  = app.version()
let name = app.name()
app.quit()
app.relaunch()
```

### `dialog` built-in

```trip
let file = await dialog.openFile({ filters: ["*.txt", "*.tp"] })
let dir  = await dialog.openDir()
dialog.showMessage("Done!", "Files saved successfully.")
```

### `shell` built-in (open URLs, files in native apps)

```trip
shell.open("https://example.com")
shell.openFile("report.pdf")          # opens in default PDF viewer
shell.showInExplorer("output/")       # reveal in Finder / Explorer
```

```js
const { shell, app, dialog } = require('electron')
// Expose as Trip builtins in your Electron main process bootstrap
```

---

## Suggested Implementation Order

| Priority | Feature | Effort | Impact |
|----------|---------|--------|--------|
| 1 | `os` builtin | 30 min | High |
| 2 | `path` builtin | 30 min | High |
| 3 | Expand `file` (mkdir, readdir, copy, move, stat) | 1 hr | High |
| 4 | `proc` builtin | 1 hr | High |
| 5 | `env` builtin | 20 min | Medium |
| 6 | `json` builtin | 30 min | High |
| 7 | `net.get` via curl | 1 hr | Medium |
| 8 | `shell` / `dialog` / `app` Electron bridge | 2 hrs | High |
| 9 | Multi-line strings `"""` | 1 hr | Medium |
| 10 | Pipe operator `\|>` | 2 hrs | Medium |
| 11 | `after` / `every` timers | 30 min | Medium |
| 12 | Full async interpreter | 1–2 days | High (long term) |

---

## Example: What Trip Desktop Code Should Look Like

```trip
import "stdlib/fs"

# Find all .log files in a directory and summarize them
fn summarizeLogs(dir):
    let logs = file.readdir(dir).filter(fn(f): return f.endsWith(".log"))
    let total = 0
    for name in logs:
        let content = file.read(path.join(dir, name))
        let lineCount = len(split(content, "\n"))
        out(f"{name}: {lineCount} lines")
        total = total + lineCount
    out(f"Total: {total} lines across {len(logs)} files")

let logsDir = path.join(os.homedir(), "logs")
summarizeLogs(logsDir)
```

This is the kind of script that should feel natural in Trip — terse,
readable, desktop-native.