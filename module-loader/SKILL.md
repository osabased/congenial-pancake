---
name: module-loader
description: >
  Use this skill when working in a Roblox project that uses the ActualFire-Games/crusherfire
  module-loader — any time the user wants to add, edit, or debug a module or service.
  Always trigger for requests like "add a service", "write a module", "create a system", or anything
  involving Start.server.lua, Start.client.lua, ParallelModuleLoader, or RelocatedTemplate.
---

# ActualFire-Games Module Loader — Working Within an Existing Project

The module loader is already set up and running. Your job is to help the user add, edit, and maintain modules that plug into this system. Don't assume any specific folder layout — follow wherever the existing modules already live in the project.

---

## Module Lifecycle

Every module the loader finds goes through three phases in order:

1. **Require** — `require()` is called on the module, sequentially
2. **Init** — `:Init()` is called on every module, sequentially
3. **Start** — `:Start()` is called on every module, sequentially

All three phases are sequential — each module completes its phase before the next module begins. Both `:Init()` and `:Start()` are optional; if a module doesn't define one, that phase is skipped for it.

---

## Writing a New Module

```lua
--!strict
local MyService = {}

-- Init: runs before Start, sequentially across all modules.
-- Use for: initializing state, setting up data structures.
-- Avoid yielding here — it blocks all other modules from initing.
function MyService:Init()

end

-- Start: runs after all modules have Init'd, also sequentially.
-- Use for: connecting RemoteEvents, starting loops, game logic.
function MyService:Start()

end

return MyService
```

**Rules:**
- Avoid yielding in `:Init()` — it is sequential and blocks everything else.
- By the time `:Start()` runs, all modules have completed `:Init()`, so it's safe to call other modules' public methods there.
- No base class or inheritance required — a plain table is fine.
- Use `--!strict` at the top (conventional in this codebase).

---

## Module Attributes

Set these directly on a `ModuleScript` instance to control how the loader handles it:

| Attribute | Type | Effect |
|---|---|---|
| `LoaderPriority` | number | Higher number = loaded earlier. Default priority is 0. |
| `IgnoreLoader` | boolean | Excludes this module from being loaded entirely. |
| `ServerOnly` | boolean | Module is skipped on the client. |
| `ClientOnly` | boolean | Module is skipped on the server. |
| `Parallel` | boolean | Module is required inside its own Actor for parallel execution. |
| `RelocateToServerScriptService` | boolean | Automatically moves the module to `ServerScriptService` at runtime and leaves a proxy in its place, hiding server code from clients. |

---

## Parallel Modules

To run a module in parallel, set the `Parallel` boolean attribute on the `ModuleScript`. The loader handles Actor creation automatically — no manual Actor setup is needed.

The Actor folder structure (with `init.meta.json`, `Script.server.lua`, and `ParallelModuleLoader.lua`) is the internal template the loader clones. You don't create these yourself; the loader manages them.

**Key caveat:** Each Actor runs in its own Luau VM. Module-level variables are not shared between parallel modules — each gets its own isolated copy of anything it requires.

---

## Server-Only Module Relocation

Setting `RelocateToServerScriptService = true` on a module tells the loader to move it into `ServerScriptService` at runtime and replace it with a proxy (`RelocatedTemplate`). This hides server-only code from clients while letting you organize modules freely in Studio.

- Only works for server-only modules not already in `ServerScriptService`.
- The real module is moved to `ServerScriptService/RELOCATED_MODULES/` at runtime.
- The proxy is automatically named and parented to replace the original.
- Avoid yielding during `require()` on a server module with this attribute — the loader warns that yielding delays relocation and may expose server code to clients.

---

## Common Patterns

**Cross-module communication:** Require other modules at the top of your file the same way the project's existing modules do. Call their methods in `:Start()` rather than `:Init()` — by Start time, all modules have completed Init.

```lua
local OtherService = require(...)  -- follow the project's path convention

function MyService:Start()
	OtherService:DoSomething() -- safe, OtherService:Init() has already run
end
```

**Controlling load order:** If a module must initialize before others, set a higher `LoaderPriority` number attribute on it.

**Excluding a module temporarily:** Set `IgnoreLoader = true` on the ModuleScript to have the loader skip it without deleting it.

**Checking load state from other code:**
```lua
ModuleLoader.IsServerLoaded()              -- returns boolean
ModuleLoader.IsClientLoaded(player)        -- returns boolean
ModuleLoader.WaitForLoadedClient(player)   -- yields until client finishes loading
```

**Debugging slow modules:** The loader warns when a module takes too long during any phase (require, Init, or Start). If you see a yield warning, check for `task.wait()`, `:WaitForChild()`, or event `.Wait()` calls in the flagged phase.

**Custom module filtering:** Use `StartCustom(predicate, ...containers)` instead of `Start()` if you need custom logic for which modules get loaded.
