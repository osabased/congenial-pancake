# Roblox table Library — Full Reference

Load this file when the user asks about specific `table.*` functions,
table performance, freezing/cloning, `table.move`, `table.pack`/`unpack`,
sorting comparator rules, or table pool patterns.

---

## Contents

1. [Quick API Reference](#quick-api-reference)
2. [Patterns & Notes](#patterns--notes)
   - [table.create — Pre-allocate for performance](#tablecreate--pre-allocate-for-performance)
   - [table.clear — Reuse tables (pooling)](#tableclear--reuse-tables-pooling)
   - [table.freeze — Immutable constants](#tablefreeze--immutable-constants)
   - [table.find vs key lookup](#tablefind-vs-key-lookup)
   - [table.move — Efficient array manipulation](#tablemove--efficient-array-manipulation)
   - [table.sort — Comparison function rules](#tablesort--comparison-function-rules)
   - [table.pack / table.unpack — Vararg handling](#tablepack--tableunpack--vararg-handling)
---

## Quick API Reference

| Function | Signature | Description |
|---|---|---|
| `table.insert` | `(t, [pos,] value)` | Append to end, or insert at `pos` (shifts elements up) |
| `table.remove` | `(t, [pos]) → value` | Remove at `pos` (default: last). Returns removed value. Shifts elements down. |
| `table.sort` | `(t, [comp])` | In-place sort. `comp(a,b)` must return `true` if `a` comes before `b`. Both `comp(a,b)` and `comp(b,a)` returning `true` throws `"invalid order function for sorting"`. |
| `table.find` | `(haystack, needle, [init]) → number?` | **O(n) linear search.** Returns index of first match starting from `init`, or `nil`. |
| `table.concat` | `(t, [sep], [i], [j]) → string` | Joins string/number array into a string with separator. |
| `table.create` | `(count, [value]) → table` | Pre-allocates array of `count` elements, optionally filled with `value`. **Use to avoid resize cost on large arrays.** |
| `table.clear` | `(t)` | Sets all keys to `nil` (`#t → 0`). **Preserves allocated capacity** — use for table pools/reuse. |
| `table.clone` | `(t) → table` | Shallow copy. Result is **unfrozen** even if source was frozen. |
| `table.move` | `(src, a, b, t, [dst]) → dst` | Copies `src[a..b]` into `dst` starting at index `t`. `dst` defaults to `src`. Ranges may overlap. Returns `dst`. |
| `table.pack` | `(...) → table` | Packs varargs into `{1=v1, 2=v2, ..., n=#args}`. Has `t.n` field for arg count. |
| `table.unpack` | `(list, [i], [j]) → ...` | Returns `list[i..j]` as a tuple. Same as global `unpack()`. Defaults: `i=1`, `j=#list`. |
| `table.freeze` | `(t) → t` | Makes table **read-only (shallow)**. Modifying a frozen table throws an error. Returns the same table. |
| `table.isfrozen` | `(t) → bool` | Returns `true` if the table is frozen. |

### Deprecated (do not use in new code)

| Function | Replacement |
|---|---|
| `table.foreach(t, f)` | `for k, v in t do` |
| `table.foreachi(t, f)` | `for i, v in ipairs(t)` |
| `table.getn(t)` | `#t` |

---

## Patterns & Notes

### table.create — Pre-allocate for performance

When building large arrays incrementally, `table.create` avoids repeated
internal resizing (which is expensive):

```lua
-- BAD: table resizes multiple times as it grows
local results = {}
for i = 1, 10000 do
    table.insert(results, computeValue(i))
end

-- GOOD: pre-allocated, no resize
local results = table.create(10000)
for i = 1, 10000 do
    results[i] = computeValue(i)
end
```

Also useful for filling a table with a default value:

```lua
local flags = table.create(64, false)  -- 64 false values
```

### table.clear — Reuse tables (pooling)

`table.clear` nils all keys but **preserves the array's allocated capacity**.
This makes it ideal for pool patterns where the same table is refilled
repeatedly — no resizing cost on the next fill cycle:

```lua
local pool = {}

local function resetPool()
    table.clear(pool)
    -- pool is empty but memory is reserved — next fill is cheap
end
```

Do not use `table.clear` on tables you intend to discard — just let the GC
collect them.

### table.freeze — Immutable constants

Freeze module-level constant tables so accidental writes throw an error
immediately rather than silently corrupting state:

```lua
local Config = table.freeze({
    MAX_PLAYERS = 50,
    ROUND_DURATION = 300,
    SPAWN_TAGS = table.freeze({ "Spawn", "Team1Spawn", "Team2Spawn" }),
})

-- Config.MAX_PLAYERS = 99  -- throws: "attempt to modify a readonly table"
```

`table.freeze` is **shallow** — nested tables must be frozen separately (as
shown above). Combine with `table.clone` to make a mutable copy of a frozen
table:

```lua
local mutable = table.clone(frozenTable)  -- always returns unfrozen copy
mutable.key = "new value"
```

### table.find vs key lookup

`table.find` is a **linear O(n) search** — avoid in tight loops or hot paths.
Prefer a key-indexed table for O(1) membership checks:

```lua
-- BAD: O(n) on every check
local allowed = {"sword", "bow", "staff"}
if table.find(allowed, weapon) then ... end

-- GOOD: O(1) lookup
local allowed = table.freeze({ sword = true, bow = true, staff = true })
if allowed[weapon] then ... end
```

### table.move — Efficient array manipulation

Faster than a manual copy loop for merging or shifting array segments:

```lua
-- Append array B onto array A
local a = {1, 2, 3}
local b = {4, 5}
table.move(b, 1, #b, #a + 1, a)
-- a is now {1, 2, 3, 4, 5}

-- Shift elements right by 1 (to insert at front)
table.move(t, 1, #t, 2, t)
t[1] = newValue
```

`dst` defaults to `src`, so same-table moves are supported.

### table.sort — Comparison function rules

The comparator **must be a strict weak ordering**: if `comp(a, b)` is `true`,
then `comp(b, a)` must be `false`. Violating this throws `"invalid order
function for sorting"`:

```lua
-- BAD: can return true for both directions when equal
table.sort(t, function(a, b) return a.score >= b.score end)

-- GOOD: strict less-than
table.sort(t, function(a, b) return a.score > b.score end)
```

### table.pack / table.unpack — Vararg handling

```lua
-- Capture varargs for later use
local function capture(...)
    local args = table.pack(...)
    -- args.n is the true argument count (unlike #args, handles nil correctly)
    task.defer(function()
        doThing(table.unpack(args, 1, args.n))
    end)
end

-- Spread a table as function arguments
local values = {10, 20, 30}
print(table.unpack(values))  --> 10  20  30
```


