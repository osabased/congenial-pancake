---
name: roblox-systems
description: >
  Expert guidance for Roblox-specific coding patterns, community libraries, and engine quirks that training data gets wrong or misses. Use for ANY Roblox dev question and programming. Also load roblox-optimization for performance/profiling.
---

# Roblox Systems

For production Roblox development — focuses on things training data gets wrong.

> **See also**: `roblox-optimization` (performance, batching, memory profiling),
> `roblox-async` (task library, coroutines, Promise).

---

## ⚠️ Always Search Before Answering

Roblox APIs and community libraries evolve fast. Before answering about any specific API or library:
- Authority sources: `create.roblox.com/docs`, `devforum.roblox.com`, library GitHub repos
- Prefer sources from the last 12 months
- Do **not** rely on training data alone — especially for Fusion (v0.2 → v0.3 is a near-complete rewrite) and ProfileStore vs ProfileService

---

## 1. Community Libraries

### Signals (same-side communication)

**Never use `BindableEvent`** for module-to-module communication — Instance overhead + memory leaks.

**GoodSignal** (stravant) — community standard:
```lua
local Signal = require(path.to.GoodSignal)
local onDamaged = Signal.new()
local conn = onDamaged:Connect(function(amount, source) end)
onDamaged:Fire(25, "Sword")
conn:Disconnect()
```
Wally: `stravant/goodsignal@^0.2.2` · 🔍 `github.com/stravant/goodsignal`

---

### Networking (typed remotes)

Raw `RemoteEvent` + manual argument parsing is error-prone and wasteful. Use a typed library:

| Library | Best for | Key trait |
|---|---|---|
| **ByteNet** | Default for most games | Typed buffer schemas, minimal bandwidth |
| **Zap** | Bandwidth-critical / large teams | IDL codegen — `.zap` schema compiles to Lua; requires build step |
| **Packet** | Simpler projects | Typed schemas wrapping existing RemoteEvents |

🔍 `ByteNet roblox github`, `Zap roblox networking`

---

### Player Data — ProfileStore

**ProfileStore** (loleris / MadStudioRoblox) is the community standard. Training data often recommends **ProfileService** (same author, older) — **incompatible APIs**. ProfileStore is current.

Key features: session locking, auto-save, `Reconcile` (merges missing keys), GDPR `AddUserId`.

```lua
local ProfileStore = require(path.to.ProfileStore)
local PlayerStore = ProfileStore.New("PlayerData", DEFAULT_DATA)

Players.PlayerAdded:Connect(function(player)
    local profile = PlayerStore:StartSessionAsync(tostring(player.UserId), {
        Cancel = function() return not player:IsDescendantOf(Players) end
    })
    if not profile then player:Kick("Data load failed") return end
    profile:AddUserId(player.UserId)
    profile:Reconcile()
end)

Players.PlayerRemoving:Connect(function(player)
    local profile = Profiles[player]
    if profile then profile:EndSession() end
end)
```

> ⚠️ Always kick the player on profile load failure.

🔍 `github.com/MadStudioRoblox/ProfileStore`

---

### State Replication — ReplicaService / Replica

**ReplicaService** handles selective server→client state replication via `Replica` objects — diffs and replicates automatically.

> ⚠️ **Replica** (no "Service") is the official successor by loleris (late 2024). Check status for new projects; ReplicaService remains stable for existing code.
> 🔍 `Replica roblox loleris devforum 2024`

Natural pairing: ProfileStore (authoritative data) → expose read-only Replica to client.

```lua
-- Server
local replica = ReplicaService.NewReplica({
    ClassToken = ReplicaService.NewClassToken("PlayerData"),
    Data = { Coins = 0, Level = 1 },
    Replication = player,
})
replica:SetValue({"Coins"}, 100)  -- auto-replicated

-- Client
ReplicaController.RequestData()
ReplicaController.ReplicaOfClassCreated("PlayerData", function(replica)
    replica:ListenToChange({"Coins"}, function(newValue) end)
end)
```

🔍 `github.com/MadStudioRoblox/ReplicaService`

---

### Cleanup — Trove

Community standard for managing connections, instances, and threads. (Maid is fine for Nevermore codebases; prefer Trove for new projects.)

```lua
local trove = Trove.new()
trove:Add(part.Touched:Connect(function() end))  -- disconnected on clean
trove:Add(instance)                               -- :Destroy() on clean
trove:AddTask(task.spawn(function() end))         -- thread cancelled on clean
local sub = trove:Extend()                        -- nested sub-trove

trove:Destroy()  -- full cleanup
-- or trove:Clean() to reset and reuse
```

Wally: `sleitnick/trove@^1.0.0`

---

### Observer Pattern

Declarative, auto-cleanup reactions to instance/tag/attribute changes. Eliminates manual `CollectionService` + `AttributeChanged` wiring.

```lua
local stop = Observers.observeTag("Enemy", function(part: BasePart)
    local trove = Trove.new()
    trove:Add(part.Touched:Connect(function() end))
    return function() trove:Destroy() end  -- cleanup when tag removed
end, { workspace })

-- Nested observers
Observers.observeTag("Enemy", function(part)
    local trove = Trove.new()
    trove:Add(Observers.observeAttribute(part, "IsStunned", function(stunned)
        return function() end
    end, function(v) return v == true end))
    return function() trove:Destroy() end
end)
```

Available: `observeTag`, `observeAttribute`, `observeProperty`, `observeCharacter`, `observeChildren`, `observeDescendants`, `observeAllAttributes`

🔍 `Sleitnix observer module roblox`

---

### UI Frameworks

| Library | Model | Best for |
|---|---|---|
| **Fusion** | Reactive state, Roblox-native | Teams comfortable with Roblox idioms |
| **react-lua** | React port — components, hooks, virtual DOM | Web-background teams; large component trees |

#### ⚠️ Fusion v0.2 vs v0.3 — Breaking API Change

Training data predominantly reflects **v0.2**. v0.3 is a near-complete rewrite. Always confirm version.

**v0.2 (legacy)**:
```lua
local coins = Value(0)
local display = Computed(function() return "Coins: " .. coins:get() end)
New "TextLabel" { Text = display }
```

**v0.3 (current)**:
```lua
local scope = scoped(Fusion)
local coins = scope:Value(0)
local display = scope:Computed(function(use) return "Coins: " .. use(coins) end)
scope:New "TextLabel" { Text = coins }
doCleanup(scope)
```

v0.3: everything is **scoped**, `use()` replaces `:get()` inside Computed, no globals.

🔍 `github.com/Elttob/Fusion`, `github.com/Roblox/react-lua`

---

### ECS Frameworks

| Library | Best for |
|---|---|
| **Matter** | Full-featured — loop hooks, hot reloading, debugger. Established ecosystem. |
| **jecs** | Lightweight, high-performance. Newer, smaller ecosystem. |

🔍 `matter-ecs/matter github`, `ukendio/jecs roblox`

---

### Flamework (roblox-ts only)

TypeScript-first DI framework. Training data has very limited knowledge of it.

```ts
@Service()
export class CurrencyService implements OnStart {
    private profileService = Dependency<ProfileService>();
    onStart() { }
}

@Component({ tag: "Coin" })
export class CoinComponent extends BaseComponent<{}, Part> implements OnStart {
    onStart() { this.instance.Touched.Connect((hit) => { }); }
}
```

Key: `@Service`/`@Controller` = auto-initialized singletons, `@Component` = tag-based instance components, `Dependency<T>()` = DI, lifecycle: `OnInit`, `OnStart`, `OnPlayerAdded`, `OnTick`.

🔍 `fireboltofdeath/flamework`

---

### Legacy — Knit

Widely present in existing codebases but superseded. **Do not recommend for new projects.** Steer new projects to the plain service pattern or Flamework.

---

## 2. Luau Strict Mode & OOP

Production code uses `--!strict`. Training data often omits this.

```lua
--!strict
local MyClass = {}
MyClass.__index = MyClass

type MyClass = typeof(setmetatable({} :: {
    health: number,
    name: string,
}, MyClass))

function MyClass.new(name: string): MyClass
    return setmetatable({ health = 100, name = name }, MyClass)
end

function MyClass:takeDamage(amount: number): ()
    self.health -= amount
end
```

**Strict-mode pitfalls:**
- `self` type inferred from `__index` — wrong metatable setup causes cascading errors
- `pcall` second return is `any` — cast explicitly if needed
- Intersection types (`A & B`) degrade inference quickly — prefer explicit casting

---

## 3. Runtime Type Checking — T Library

Manual `type()` checks on `OnServerEvent` miss edge cases. **T library** (osyrisrblx):

```lua
local t = require(path.to.T)
local validateArgs = t.tuple(t.instanceOf("Player"), t.number, t.string)
local ok, err = validateArgs(player, amount, itemName)
if not ok then warn(err) return end
```

🔍 `osyrisrblx/t github`

---

## 4. Engine Gotchas

### `script.RunContext`
Set to `Enum.RunContext.Server`, `Client`, or `Legacy`. Lets you place a Script anywhere in the DataModel. Training data rarely mentions this.

### PlayerCharacterDestroyBehavior
Old characters linger on respawn by default:
```lua
Players.PlayerCharacterDestroyBehavior = Enum.PlayerCharacterDestroyBehavior.Enabled
```

### Network Ownership
Roblox auto-assigns unanchored parts to nearest player.
- Server-authoritative physics: `part:SetNetworkOwner(nil)`
- Mismatched ownership → rubber-banding / desync

### StreamingEnabled
- Clients may not have parts the server knows about — never unconditionally `FindFirstChild`
- `RequestStreamAroundAsync(position)` forces streaming a region
- `StreamingIntegrityMode.PauseOutsideLoadedArea` prevents movement until loaded

### Attribute Replication
Server-set attributes replicate to all clients. Client-set are **local only**.

### `task.spawn` vs `task.defer`
- `task.spawn(f)` — runs immediately in new thread
- `task.defer(f)` — queues after current thread (FIFO)
- Legacy `spawn()` was deferred → use `task.defer` as drop-in, **not** `task.spawn`

### RunService Replacements
| Deprecated | Replacement |
|---|---|
| `RunService.RenderStepped` | `RunService.PreRender` |
| `RunService.Stepped` | `RunService.PreSimulation` |
| `RunService.Heartbeat` | `RunService.PostSimulation` |

### Other Gotchas
- **`Instance.new(class, parent)`** — second arg deprecated; set `.Parent` last
- **`:WaitForChild()` no timeout** — yields forever; always pass timeout in production
- **`game:GetService()`** — always use over direct access (`game.Players` can fail before init)
- **`ReplicatedStorage` sensitive data** — accessible by exploiters; use `ServerStorage`
- **`math.random` global seed** — use `Random.new()` per object
- **`Humanoid:LoadAnimation()`** — deprecated; use `Animator:LoadAnimation()`
- **DataStoreService silent failure in Studio** — enable "Studio Access to API Services"
- **`ModuleScript` caching** — `require()` cached per VM; use factory functions for per-caller state

---

## 5. Security

### Server Must Own All Authoritative Values

```lua
-- BAD: client dictates damage
DealDamage.OnServerEvent:Connect(function(player, target, damage)
    target.Humanoid:TakeDamage(damage)
end)

-- GOOD: server looks up the value
DealDamage.OnServerEvent:Connect(function(player, target)
    local tool = player.Character and player.Character:FindFirstChildOfClass("Tool")
    if not tool then return end
    local damage = tool:GetAttribute("Damage")
    if not validateTarget(player, target) then return end
    target.Humanoid:TakeDamage(damage)
end)
```

### OnServerEvent Validation Checklist
1. Player is alive (`Character` exists, `Humanoid.Health > 0`)
2. Target is valid (exists, correct type, not dead)
3. Proximity — player close enough to act
4. Ownership — player owns what they're modifying
5. Game state — action currently allowed (round active, cooldown elapsed)
6. Value bounds — numbers in realistic range; strings match allowlist

### Rate Limiting
```lua
local cooldowns = {}
local COOLDOWN = 0.5
Remote.OnServerEvent:Connect(function(player)
    local now = tick()
    if cooldowns[player.UserId] and now - cooldowns[player.UserId] < COOLDOWN then return end
    cooldowns[player.UserId] = now
end)
Players.PlayerRemoving:Connect(function(p) cooldowns[p.UserId] = nil end)
```

### `RemoteFunction:InvokeClient()` — Do Not Use for Game Logic
Server thread **yields forever** if client disconnects. Use a `RemoteEvent` pair with a `requestId` correlation ID instead.

---

## 6. Remote Communication — Quick Reference

| Remote | Use case |
|---|---|
| `RemoteEvent` | Fire-and-forget, guaranteed delivery. Default for all game events. |
| `UnreliableRemoteEvent` | High-frequency cosmetic data only (FX, position sync). May drop/reorder. **Never** for inventory or currency. |
| `RemoteFunction` (server→server) | OK. |
| `RemoteFunction:InvokeClient()` | **Avoid** — server yields forever if client disconnects. |
| `BindableEvent` / `BindableFunction` | **Avoid** — use GoodSignal instead. |

---

## 7. Wally — Package Manager

```toml
[dependencies]
GoodSignal = "stravant/goodsignal@^0.2.2"
Trove      = "sleitnick/trove@^1.0.0"
T          = "osyrisrblx/t@^3.1.1"
# ProfileStore: no official Wally package — install from GitHub
```

```bash
wally install  # installs to Packages/
```

Map `Packages/` → `ReplicatedStorage.Packages` in Rojo (split `ServerPackages/` for server-only deps).

---

## 8. System Design Pattern

Split every system into a frozen `SharedService` (both contexts), `ServerService` (authority), and `ClientService` (UI/feedback), with game-specific wiring in a separate `Implementation/` folder.

For full detail — read `references/system-design.md`.

---

## 9. `table` Library — Quick Reference

For full reference — read `references/table-library.md`.

**Load when**: user asks about `table.*` functions, table performance, freezing/cloning, `table.move`, or sorting.

| Function | Key behaviour |
|---|---|
| `table.create(n, v)` | Pre-allocates `n` slots — avoids resize on large arrays |
| `table.clear(t)` | Nils all keys but **preserves capacity** — ideal for pool reuse |
| `table.clone(t)` | Shallow copy, always **unfrozen** even if source was frozen |
| `table.freeze(t)` | Read-only (shallow) — nested tables need separate `freeze` calls |
| `table.move(src,a,b,t,[dst])` | Copy `src[a..b]` into `dst[t..]` |
| `table.pack(...)` | `→ {1=v1,...,n=count}` — use `.n` for true vararg count with nils |
| `table.find(t,v,[init])` | **O(n)** — use key lookup for hot-path membership |
| `table.sort(t,[comp])` | `comp` must be strict weak ordering |

**Deprecated**: `table.foreach`, `table.foreachi`, `table.getn`

---

## Quick Reference: Common Mistakes

| Mistake | Fix |
|---|---|
| `BindableEvent` for same-side signals | GoodSignal |
| Raw `RemoteEvent` for typed networking | ByteNet, Zap, or Packet |
| Rolling a custom DataStore wrapper | ProfileStore (session locking is critical) |
| Confusing ProfileStore with ProfileService | Different APIs — ProfileStore is current |
| No type-checking on `OnServerEvent` args | T library |
| Trusting client-sent values | Server looks up authoritative values itself |
| `InvokeClient` for game logic | Event pair + requestId + timeout |
| Observer connections not cleaned up | Return cleanup fn or `trove:Add()` |
| `PlayerCharacterDestroyBehavior` at default | Set to `Enabled` |
| Physics parts with wrong network owner | `part:SetNetworkOwner(nil)` |
| `StreamingEnabled` + unconditional `FindFirstChild` | Add timeouts; `RequestStreamAroundAsync` |
| `task.spawn` as drop-in for legacy `spawn()` | Use `task.defer` |
| `RunService.RenderStepped` / `.Stepped` | `PreRender` / `PreSimulation` |
| Sensitive data in `ReplicatedStorage` | Move to `ServerStorage` |
| Recommending Knit for new projects | Plain service pattern or Flamework |
| Answering Fusion without checking version | v0.2 vs v0.3 are completely different APIs |
| `Instance.new(class, parent)` two-arg form | Set `.Parent` last |
| `:WaitForChild()` with no timeout | Always pass a timeout |
| `game.Players` direct access | `game:GetService("Players")` |
| `math.random` global seed conflicts | `Random.new()` per object |
| `Humanoid:LoadAnimation()` | `Animator:LoadAnimation()` |
| DataStoreService silent failure in Studio | Enable "Studio Access to API Services" |
| Mutating shared ModuleScript return value | Factory functions or `table.clone()` |
