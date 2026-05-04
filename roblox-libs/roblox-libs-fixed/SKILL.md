---
name: roblox-libs
description: Use GoodSignal, t, and Trove packages from ReplicatedStorage.Packages in Roblox Luau. GoodSignal is a leak-free signal replacing BindableEvent. t is a runtime type checker for validating configs, remote inputs, and handler shapes. Trove tracks and cleans up connections, instances, threads, and functions. Always use this skill before writing any Roblox Luau code involving signals, type checking, or object cleanup — even if the user doesn't name the libraries explicitly. Use when working with signals, events, type validation, runtime type checking, cleanup, lifecycles, or service patterns, or when the user mentions GoodSignal, t, Trove, or any of these packages.
---

# Roblox Libs — GoodSignal · t · Trove

```lua
-- Import (standard pattern used across codebase).
-- Direct property access gives a clear "is not a valid member" error if a package is missing.
-- Note: Wally installs "goodsignal" lowercase; "Trove" retains its original casing.
-- LocalScript note: if this runs before server replication completes, replace
--   ReplicatedStorage.Packages with ReplicatedStorage:WaitForChild("Packages").
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Packages   = ReplicatedStorage.Packages
local GoodSignal = require(Packages.goodsignal)
local t          = require(Packages.t)
local Trove      = require(Packages.Trove)
```

## Luau types

GoodSignal exports two types for `--!strict` modules:

```lua
-- Signal     — the signal object itself (what GoodSignal.new() returns).
-- Connection — the object returned by :Connect() / :Once().
type Signal     = GoodSignal.Signal
type Connection = GoodSignal.Connection
```

Use these when declaring public signal fields or function signatures:

```lua
-- Typed public signal field (required in --!strict):
MyService.OnEvent: GoodSignal.Signal = GoodSignal.new()

-- Function that accepts a pre-existing connection:
local function cancelOnTimeout(conn: GoodSignal.Connection, seconds: number) ... end
```

If your installed version of goodsignal does not export these type names, use
`typeof` as a fallback:

```lua
type Signal     = typeof(GoodSignal.new())
type Connection = typeof(GoodSignal.new():Connect(function() end))
```

## Combined service template

**t validates → GoodSignal communicates → Trove cleans up** (pattern from DeathService, AFKService):

```lua
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Packages   = ReplicatedStorage.Packages
local GoodSignal = require(Packages.goodsignal)
local t          = require(Packages.t)
local Trove      = require(Packages.Trove)

local MyService = {}
MyService.OnEvent: GoodSignal.Signal = GoodSignal.new() -- public signal; typed for --!strict

local checkConfig = t.strictInterface({ Interval = t.numberPositive })
local CONFIG = { Interval = 1 }
assert(checkConfig(CONFIG))

local rootTrove: Trove.Trove? = nil

function MyService:Start()
    -- Guard against double-start: clean previous trove before creating a new one.
    if rootTrove then rootTrove:Destroy() end

    local trove = Trove.new()
    rootTrove = trove

    -- Added FIRST so it runs LAST on cleanup (Trove cleans in LIFO order).
    -- This keeps OnEvent alive while other tracked objects (connections, sub-troves)
    -- are being torn down, preventing use-after-disconnect errors during teardown.
    trove:Add(function()
        MyService.OnEvent:DisconnectAll()
        rootTrove = nil
    end)
end

function MyService:Destroy()
    if rootTrove then rootTrove:Destroy() end
end

return MyService
```

## GoodSignal

```lua
local sig = GoodSignal.new()
local conn = sig:Connect(function(p) end) -- returns Connection
sig:Fire(player)                          -- synchronous, immediate
sig:Wait()                                -- yields until next Fire
sig:Once(function(p) end)                 -- auto-disconnect after first fire
conn:Disconnect()
sig:DisconnectAll()
```

- Full RBXScriptSignal API parity (`:Connect`, `:Once`, `:Wait`, `:Fire`)
- Pure Lua — no BindableEvent, no memory leaks even without explicit Disconnect
- In services: expose as public field, clean up via `trove:Add(function() sig:DisconnectAll() end)`
- **`:Wait()` caution:** yields indefinitely if the signal never fires — use `:Connect()` with a `task.delay` timeout instead when the signal might not fire:

  ```lua
  -- Prefer this over :Wait() when the signal is not guaranteed to fire.
  local conn
  local timer = task.delay(5, function()
      if conn then conn:Disconnect() end
      warn("Timed out waiting for signal")
  end)
  conn = sig:Connect(function(value)
      task.cancel(timer)
      conn:Disconnect()
      -- handle value
  end)
  ```

- **Trove incompatibility:** `trove:Connect(goodSignal, fn)` and `trove:Add(conn)` crash — GoodSignal connections error on unknown property access. Use `trove:Add(function() conn:Disconnect() end)` instead.
- **Double-disconnect is safe:** if the connection was already disconnected (e.g., a `:Once` that already fired), calling `conn:Disconnect()` again is a no-op. Safe to wrap all `:Once` connections with a trove cleanup function unconditionally.
- **`trove:Construct(GoodSignal)` is also invalid** — Signal has no `:Destroy()` or `:Disconnect()`, so Trove throws "Unhandled cleanup type". Construct the signal manually and add a cleanup function instead.

## Trove

Core flow: **create → add objects → clean/destroy**.

| Method | Use |
|---|---|
| `:Add(obj, method?)` | Track any trackable (Instance, connection, fn, thread, table). **Returns the tracked object** — use for one-liner chaining: `local part = trove:Add(Instance.new("Part"))` |
| `:Connect(signal, fn)` | Connect + track — **RBXScriptSignal only** (see GoodSignal note) |
| `:Once(signal, fn)` | One-shot connect + track — **RBXScriptSignal only** (same incompatibility as `:Connect` — see GoodSignal note) |
| `:Construct(Class, ...)` | `Class.new(...)` + track result |
| `:Clone(instance)` | `instance:Clone()` + track result |
| `:Extend()` | Create & track a sub-trove |
| `:BindToRenderStep(name, pri, fn)` | Bind + auto-unbind — **client only** |
| `:AttachToInstance(inst)` | Auto-clean when instance destroyed — **instance must be a descendant of `game`** |
| `:Clean()` / `:Destroy()` | Clean all tracked objects in LIFO order (trove remains reusable) |
| `:WrapClean()` | Returns a `function()` that calls `:Clean()` |
| `:Remove(obj)` / `:Pop(obj)` | Remove single object (with/without cleanup) |
| `:AddPromise(promise)` | Cancel promise on clean — requires a lowercase `:cancel()` method on the promise (see REFERENCE.md) |

> **`:Destroy()` is not terminal.** Unlike most Roblox objects, calling `:Destroy()` on a Trove is an alias for `:Clean()` — the trove stays usable and you can keep adding objects afterward.

Per-player pattern: see [REFERENCE.md](REFERENCE.md#per-player-pattern).

## t

Define checkers, assert at boundaries. `strictInterface` = no extra fields (configs). `interface` = extra fields allowed (handlers).

```lua
local checkConfig = t.strictInterface({ Name = t.string, Rate = t.numberPositive })
assert(checkConfig(CONFIG))   -- t's own error message propagates on failure

local isHandler = t.interface({ Priority = t.number, OnFire = t.callback })
-- Correct: no second arg — isHandler(handler) is in tail position, so both return
-- values reach assert, and t's detailed error message propagates on failure.
-- WRONG form: assert(isHandler(handler), "Invalid handler") — adding a custom second
-- arg pushes isHandler(handler) out of tail position; only its first return value
-- (false) reaches assert and "Invalid handler" replaces t's error string entirely.
assert(isHandler(handler))

-- Server-side remote validation (critical for security).
-- remote.OnServerEvent is an RBXScriptSignal — trove:Connect is safe here.
trove:Connect(remote.OnServerEvent, function(player, data)
    if not isRemoteData(data) then return end
end)

-- t.wrap: wraps a function so args are validated before the body runs
local safeFoo = t.wrap(function(a: string, b: number) end, t.tuple(t.string, t.number))
safeFoo("hello", 42)   -- passes validation, calls inner function
safeFoo("hello", "x")  -- throws: argument 2 expected number, got string

-- t.strict: wraps a checker into a guard function that errors on failure
-- and passes the original value(s) through on success (does NOT return true/nothing).
-- IMPORTANT: do NOT compose t.strict with other t functions (e.g. inside t.interface,
-- t.optional, t.union). t.strict throws on failure instead of returning false, which
-- crashes the parent checker. Use it only as a standalone guard function.
local assertConfig = t.strict(checkConfig)
assertConfig(CONFIG) -- throws on invalid; returns CONFIG on success

-- The return-through behavior enables validated assignment:
local validated = assertConfig(rawInput) -- validated IS rawInput when valid
```

- **`assert(checker(value), "custom msg")` drops t's error.** When the checker is not
  in tail-call position, Lua/Luau only passes its first return value (`false`) to
  `assert`; the second return value (t's error string) is discarded and your custom
  message replaces it. To preserve t's message, use `assert(checker(value))` with no
  second argument, or destructure explicitly:
  ```lua
  local ok, err = isHandler(handler)
  assert(ok, err)   -- explicit; preserves t's detailed error message
  ```
- **`t.number` rejects NaN.** A value of `0/0` fails `t.number`, `t.numberPositive`, etc. This is intentional but can surprise — validate inputs before dividing.
- `assert(check(value))` works because checkers return `(true)` on success or `(false, message)` on failure; `assert` receives both return values when the checker is in tail position.
- Use `t.optional(check)` for nullable fields: `t.optional(t.string)` passes both `nil` and `string`.
- **`t.instanceOf` vs `t.instanceIsA`:** `t.instanceOf("Part")` requires an exact `ClassName` match (only `Part`, not `MeshPart`). `t.instanceIsA("BasePart")` uses `:IsA()` and accepts any subclass (`Part`, `MeshPart`, `WedgePart`, etc.). Prefer `instanceIsA` for base-class constraints; use `instanceOf` only when the exact class matters.

## Advanced

Full API reference: see [REFERENCE.md](REFERENCE.md)
