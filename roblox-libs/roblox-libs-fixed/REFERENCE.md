# Reference — GoodSignal · t · Trove

## GoodSignal (stravant/goodsignal v0.3.1)

A pure-Lua signal with full RBXScriptSignal API parity. No BindableEvent — no memory leaks.

### Luau types

GoodSignal exports two types for use in `--!strict` modules:

```lua
-- Annotate a public signal field:
MyService.OnEvent: GoodSignal.Signal = GoodSignal.new()

-- Use in a function signature:
local function waitForSignal(sig: GoodSignal.Signal, timeout: number) ... end

-- Store a connection with a type:
local conn: GoodSignal.Connection = sig:Connect(function() end)
```

If the installed version does not export these names, derive them via `typeof`:

```lua
type Signal     = typeof(GoodSignal.new())
type Connection = typeof(GoodSignal.new():Connect(function() end))
```

### Signal methods

| Method | Signature | Description |
|---|---|---|
| `Signal.new()` | `() -> Signal` | Create a new signal |
| `:Connect(fn)` | `(fn: (...any) -> ...any) -> Connection` | Connect callback, returns connection |
| `:Once(fn)` | `(fn: (...any) -> ...any) -> Connection` | Auto-disconnect after first fire |
| `:Fire(...)` | `(...any) -> ()` | Fire all connected callbacks |
| `:Wait()` | `() -> ...any` | Yield until next Fire, return fired args |
| `:DisconnectAll()` | `() -> ()` | Disconnect every connection on this signal |

### Connection methods

| Method | Signature | Description |
|---|---|---|
| `:Disconnect()` | `() -> ()` | Disconnect this single connection. Safe to call multiple times — double-disconnect is a no-op. |

### Behavioral notes

- Fire is immediate — callbacks run synchronously during `:Fire()`
- Connections fire in order of connection
- Disconnected callbacks are skipped during Fire
- `:Wait()` uses a coroutine-based implementation — suspends the caller until the next `:Fire()` (no BindableEvent, no polling)
- **`:Wait()` yields indefinitely** if the signal never fires. When the signal is not guaranteed to fire, use `:Connect()` with a `task.delay` timeout instead:

  ```lua
  -- Safe alternative to :Wait() when the signal might never fire.
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

- Even without calling `:Disconnect()`, everything GCs normally
- **Double-disconnect is safe.** Calling `conn:Disconnect()` on an already-disconnected connection (e.g., one from `:Once` that already fired) is a no-op.

---

## Trove (sleitnick/trove v1.8.0)

Object lifecycle tracker — clean up connections, instances, threads, and functions in one call.

### Trackable types and their cleanup

| Type | Cleanup |
|---|---|
| `Instance` | `object:Destroy()` |
| `RBXScriptConnection` | `object:Disconnect()` |
| `function` | `object()` — called directly and synchronously |
| `thread` | `task.cancel(object)` |
| Table with `:Destroy()` | `object:Destroy()` |
| Table with `:Disconnect()` | `object:Disconnect()` — **NOT GoodSignal connections** (see below) |
| Table with `:destroy()` | `object:destroy()` |
| Table with `:disconnect()` | `object:disconnect()` |
| Table + `cleanupMethod` | `object:<cleanupMethod>()` |
| PromiseLike | `object:cancel()` |

> **GoodSignal + Trove — two distinct incompatibilities:**
>
> 1. **`trove:Add(conn)` / `trove:Connect(goodSignal, fn)`** — GoodSignal Connection objects are tables with `:Disconnect()`, but their strict `__index` metamethod **throws** when Trove probes `object["Destroy"]` (which it checks before `Disconnect`). Both crash on cleanup. Use `trove:Add(function() conn:Disconnect() end)` instead.
>
> 2. **`trove:Construct(GoodSignal)`** — The Signal object returned by `GoodSignal.new()` has neither `:Destroy()` nor `:Disconnect()`. Trove cannot determine how to clean it and throws **"Unhandled cleanup type"**. This is a different error than the `__index` throw above. Construct signals manually and add a cleanup function instead.

### Methods

#### `Trove.new() -> Trove`

Constructs a new Trove.

#### `trove:Add(object: any, cleanupMethod: string?) -> object`

Add a trackable object. Returns the object for chaining. Cannot be called while cleaning.

```lua
trove:Add(part)
trove:Add(connection)
trove:Add(function() print("cleanup") end)
trove:Add(someTable, "CustomCleanup")
```

#### `trove:Clone(instance: Instance) -> Instance`

Shorthand for `trove:Add(instance:Clone())`.

#### `trove:Construct(class, ...args) -> T`

Construct via `class.new(...)` (table) or `class(...)` (function), then add to trove.

```lua
-- WRONG: GoodSignal Signal has no :Destroy() or :Disconnect().
-- Trove throws "Unhandled cleanup type" — this is NOT a __index error,
-- it is Trove failing to find any recognized cleanup method on the Signal.
-- local signal = trove:Construct(GoodSignal)

-- CORRECT for GoodSignal: construct manually, wrap cleanup in a function
local signal = GoodSignal.new()
trove:Add(function() signal:DisconnectAll() end)

-- Works for Luau table classes and Roblox Instance:
local part = trove:Construct(Instance, "Part")
local myObj = trove:Construct(MyLuauClass)   -- calls MyLuauClass.new()

-- For simple Instance creation, the explicit form is equally valid and clearer:
local part2 = trove:Add(Instance.new("Part"))
```

#### `trove:Connect(signal, fn) -> Connection`

Shorthand for `trove:Add(signal:Connect(fn))`. **Only works with `RBXScriptSignal`.**

**INCOMPATIBLE with GoodSignal** — GoodSignal connections have a strict `__index` metamethod that throws on unknown keys. Trove checks `object["Destroy"]` before `object["Disconnect"]` during cleanup, triggering the error. Use the workaround instead:

```lua
-- WRONG: crashes on trove:Clean() / trove:Destroy()
trove:Connect(MyService.OnChanged, function(val) end)

-- CORRECT: manually connect, clean up via function
local conn = MyService.OnChanged:Connect(function(val) end)
trove:Add(function() conn:Disconnect() end)
```

#### `trove:Once(signal, fn) -> Connection`

Shorthand for `trove:Add(signal:Once(fn))`. **Auto-pops from the trove when the signal fires** — no manual removal needed after the one-shot fires.

**INCOMPATIBLE with GoodSignal** — same crash as `:Connect`. `signal:Once(fn)` returns a GoodSignal Connection object, which Trove cannot track safely. Use the workaround instead:

```lua
-- WRONG: crashes on trove:Clean() / trove:Destroy()
trove:Once(MyService.OnChanged, function(val) end)

-- CORRECT: manually connect once, clean up via function.
-- If the signal fires before cleanup, conn:Disconnect() is a safe no-op.
local conn = MyService.OnChanged:Once(function(val) end)
trove:Add(function() conn:Disconnect() end)
```

#### `trove:BindToRenderStep(name: string, priority: number, fn: (dt) -> ())`

> **Client only.** `RunService:BindToRenderStep` is not available on the server. Only use this inside LocalScripts or client-side ModuleScripts.

Binds to RenderStep, auto-unbinds on clean.

```lua
trove:BindToRenderStep("MyStep", Enum.RenderPriority.Last.Value, function(dt) end)
```

#### `trove:AddPromise(promise: Promise) -> Promise`

Cancels the promise on clean. Requires a promise object that exposes a lowercase `:cancel()` method — compatible with roblox-lua-promise (evaera/promise). Any promise-like object that uses uppercase `:Cancel()` instead of `:cancel()` is not compatible.

#### `trove:Remove(object: any) -> boolean`

Remove and clean up a single object. Returns `true` if found.

#### `trove:Pop(object: any) -> boolean`

Remove without cleaning. Returns `true` if found.

#### `trove:Extend() -> Trove`

Create a sub-trove tracked by this trove. `:Clean()` on parent cleans sub-trove too.

```lua
local subTrove = trove:Extend()
```

#### `trove:Clean()`

Clean all tracked objects. **Cleanup runs in LIFO order** (last-added, first-cleaned) — Trove removes from the tail of its internal object list, so the most recently added objects are cleaned first. Design dependent-object teardown with this in mind: add objects that must outlive others *first*, so they are cleaned *last*. Trove remains reusable after `:Clean()`. No-op if already cleaning.

#### `trove:Destroy()`

Alias for `:Clean()`. **Not terminal** — unlike most Roblox objects, the trove remains fully usable after this call. You can continue adding objects and calling `:Clean()`/`:Destroy()` again.

#### `trove:WrapClean() -> () -> ()`

Returns a function that calls `:Clean()`. Useful for observer patterns:

```lua
someObserver(function()
    local trove = Trove.new()
    -- setup
    return trove:WrapClean()
end)
```

#### `trove:AttachToInstance(instance: Instance) -> RBXScriptConnection`

Auto-clean the trove when the instance is destroyed. **Inverts ownership** — the instance now controls trove lifetime. **The instance must already be a descendant of `game`**; passing a non-parented instance errors.

```lua
trove:AttachToInstance(npcModel)
-- destroying npcModel now cleans the trove
```

### Per-player pattern

Used in DeathService, AFKService — one trove per player, tracked in a dictionary. The trove's own cleanup function owns the dictionary nil-out; callers don't need to do it manually.

```lua
local playerTroves: {[Player]: Trove.Trove} = {}

local function setupPlayer(player: Player)
    local trove = Trove.new()
    playerTroves[player] = trove

    -- Cleanup owns the dictionary entry.
    -- The `playerTroves[player] == trove` guard handles a rapid rejoin race:
    -- if setupPlayer fires again before this trove's cleanup runs, a new trove
    -- is already in the dictionary; this old cleanup must NOT nil it out.
    trove:Add(function()
        if playerTroves[player] == trove then
            playerTroves[player] = nil
        end
    end)

    -- CharacterAdded is an RBXScriptSignal — trove:Connect is safe here
    trove:Connect(player.CharacterAdded, function(char)
        -- Use a per-character sub-trove so humanoid.Died connections are
        -- removed on each respawn instead of accumulating in the player trove.
        local charTrove = trove:Extend()
        -- Auto-clean when the character model is destroyed (i.e. on death/respawn)
        charTrove:AttachToInstance(char)

        -- WaitForChild yields — the character can be destroyed during this yield.
        -- If that happens, AttachToInstance fires and cleans charTrove before we
        -- resume. Guard on BOTH the player trove (rejoin race) AND char.Parent
        -- (character-destroyed-during-yield race) before adding anything to charTrove.
        local humanoid = char:WaitForChild("Humanoid", 10)
        -- humanoid.Died is also an RBXScriptSignal — charTrove:Connect is safe
        if humanoid and char.Parent ~= nil and playerTroves[player] == trove then
            charTrove:Connect(humanoid.Died, function()
                onDied(player, char)
            end)
        end
    end)
end

local function cleanupPlayer(player: Player)
    local trove = playerTroves[player]
    if trove then trove:Destroy() end
    -- Dictionary entry is already nilled by the trove's own cleanup function above
end
```

---

## t (osyrisrblx/t v3.1.1)

Runtime type checker. All checkers return `(value) -> true | (false, string)`.

### assert usage — do not discard t's error message

`assert(checker(value), "custom msg")` looks safe but silently drops t's detailed
error string. Because `checker(value)` is not in tail-call position, Lua/Luau only
passes its **first** return value (`false`) to `assert`; the second return value
(t's error message) is discarded, and `"custom msg"` is used instead.

```lua
-- BAD: t's detailed error message is silently discarded
assert(isHandler(handler), "Invalid handler")

-- GOOD: t's own error message propagates
assert(isHandler(handler))

-- GOOD: explicit destructure — clear and preserves t's message
local ok, err = isHandler(handler)
assert(ok, err)
```

### Primitives

| Checker | Matches |
|---|---|
| `t.any` | Any non-nil value |
| `t.boolean` | `boolean` |
| `t.buffer` | `buffer` |
| `t.callback` | `function` (also `t["function"]`) |
| `t.none` | `nil` (also `t["nil"]`) |
| `t.number` | `number` (**rejects NaN** — `0/0` fails this check) |
| `t.string` | `string` |
| `t.table` | `table` |
| `t.thread` | `thread` |
| `t.userdata` | `userdata` |

### Roblox types

All Roblox `typeof` types available as `t.<Type>`:

`t.Instance`, `t.CFrame`, `t.Color3`, `t.Vector3`, `t.Vector2`, `t.UDim2`, `t.UDim`, `t.Enum`, `t.EnumItem`, `t.RBXScriptConnection`, `t.RBXScriptSignal`, `t.TweenInfo`, `t.NumberRange`, `t.NumberSequence`, `t.ColorSequence`, `t.Ray`, `t.Rect`, `t.Region3`, `t.PhysicalProperties`, `t.OverlapParams`, `t.RaycastParams`, `t.RaycastResult`, `t.PathWaypoint`, `t.Font`, `t.DateTime`, `t.BrickColor`, `t.Axes`, `t.Faces`, `t.Random`, `t.CatalogSearchParams`, `t.Content`, `t.FloatCurveKey`, `t.NumberSequenceKeypoint`, `t.ColorSequenceKeypoint`, `t.DockWidgetPluginGuiInfo`, `t.Region3int16`, `t.Vector2int16`, `t.Vector3int16`

### Meta type functions

| Function | Description |
|---|---|
| `t.literal(...)` | Matches exact values: `t.literal("a", "b")` |
| `t.keyOf(table)` | Union of table keys as literals |
| `t.valueOf(table)` | Union of table values as literals |
| `t.optional(check)` | Passes if nil or passes `check` |
| `t.tuple(...)` | Validates positional args: `t.tuple(t.string, t.number)` |
| `t.union(...)` | Any one must pass (alias: `t.some`) |
| `t.intersection(...)` | All must pass (alias: `t.every`) |
| `t.keys(check)` | All table keys pass `check` |
| `t.values(check)` | All table values pass `check` |
| `t.map(keyCheck, valCheck)` | Keys + values checked |
| `t.set(check)` | Map where all values are `true` |
| `t.array(check)` | Sequential integer keys, all values pass `check` |
| `t.strictArray(...)` | Fixed-size array with per-index type checks |
| `t.interface(def)` | Table matches field types (extra fields allowed) |
| `t.strictInterface(def)` | Table matches field types (extra fields rejected) |
| `t.instanceOf(className, childTable?)` | Instance with **exact** ClassName match (e.g. `"Part"` rejects `MeshPart`) |
| `t.instanceIsA(className, childTable?)` | Instance passes `:IsA()` — accepts subclasses (e.g. `"BasePart"` accepts `Part`, `MeshPart`, `WedgePart`) |
| `t.children(checkTable)` | Validate named children of an Instance |
| `t.enum(enum)` | EnumItem belongs to the given Enum |
| `t.match(pattern)` | String matches Lua pattern |
| `t.wrap(fn, checkArgs)` | Wraps fn with arg validation — throws on invalid args before calling fn; returns fn's return values on success |
| `t.strict(check)` | Returns a guard function: **errors on failure**, passes the original value(s) through on success. Not a checker — do not compose it with other t functions. |

### Number functions

| Function | Constraint |
|---|---|
| `t.nan` | Value is NaN |
| `t.integer` | Integer number — uses `math.floor(value) == value`, so `2.0` **passes** (integer-valued float) |
| `t.numberPositive` | `value > 0` |
| `t.numberNegative` | `value < 0` |
| `t.numberMin(min)` | `value >= min` |
| `t.numberMax(max)` | `value <= max` |
| `t.numberMinExclusive(min)` | `value > min` |
| `t.numberMaxExclusive(max)` | `value < max` |
| `t.numberConstrained(min, max)` | `min <= value <= max` |
| `t.numberConstrainedExclusive(min, max)` | `min < value < max` |

### Common patterns

**Config validation:**
```lua
local checkConfig = t.strictInterface({
    Name = t.string,
    Rate = t.numberPositive,
    Mode = t.optional(t.enum(Enum.RunMode)),
})
assert(checkConfig(CONFIG))   -- t's own error message propagates on failure

-- Alternative using t.strict — guard function that errors on failure,
-- returns CONFIG on success (not true/nothing).
-- NOTE: do NOT compose t.strict with other t functions (e.g. inside t.interface,
-- t.optional, t.union). It throws on failure instead of returning false, which
-- crashes the parent checker. Use only as a standalone guard.
local assertConfig = t.strict(checkConfig)
assertConfig(CONFIG)

-- The return-through behavior enables validated assignment:
local validated = assertConfig(rawInput) -- validated IS rawInput when valid
```

**Handler shape validation:**
```lua
local isHandler = t.interface({
    Priority = t.number,
    OnPlayerDied = t.callback,
})
function Service:RegisterHandler(handler)
    assert(isHandler(handler))   -- no custom second arg; preserves t's error message
    table.insert(handlers, handler)
end
```

**Remote input validation (server):**
```lua
local isRemoteData = t.strictInterface({
    Action = t.literal("attack", "defend"),
    Target = t.instanceIsA("Player"),   -- instanceIsA: accepts Player and subclasses
    Power = t.numberConstrained(0, 100),
})
-- remote.OnServerEvent is an RBXScriptSignal — trove:Connect is safe here
trove:Connect(remote.OnServerEvent, function(player, data)
    if not isRemoteData(data) then return end
end)
```

**t.wrap — validated function wrapper:**
```lua
-- Wraps a function so arguments are type-checked before the body runs.
-- Throws on invalid args; the inner function only ever sees valid inputs.
-- Returns the inner function's return values on success.
local safeFoo = t.wrap(
    function(name: string, count: number)
        print(name, count)
    end,
    t.tuple(t.string, t.number)
)

safeFoo("hello", 42)   -- ok
safeFoo("hello", "x")  -- throws: argument 2 expected number, got string
```

**Custom type checker:**
```lua
local function isMyClass(value)
    if not t.table(value) then return false, "table expected" end
    local mt = getmetatable(value)
    if not mt or mt.__index ~= MyClass then return false, "not a MyClass" end
    return true
end
```
