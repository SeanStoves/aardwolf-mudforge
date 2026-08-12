# Writing MudForge plugins — what the docs don't tell you

> **[docs/MUDFORGE-API-GUIDE.md](MUDFORGE-API-GUIDE.md) is the reference —
> check it before this file and before writing anything.** It comes from
> MudForge's own authors and they keep it current, so it outranks everything
> here on any point the two disagree on. Re-copy it from upstream rather than
> editing it; local edits would be lost and would make it untrustworthy for
> exactly the questions it's the answer to.
>
> What follows is the delta: what this codebase learned the hard way, kept
> because most of it is still not in there — and because several entries were
> measured against a running client rather than inferred, which is the only
> reason they're trustworthy at all.
>
> Confirmed by the official guide, so no longer guesses:
> `onLine(sessionId, rawLine, cleanLine)`; `addSpecialExit(from, command, to)`
> (command in the middle); mapper lists being 0-indexed; `addTimer` returning
> `""` while disconnected; `saveTable`/`savePluginFile` with scope `"global"`
> being shared app-wide **across every plugin**, keyed only by name — prefix
> yours or you will clobber someone.
>
> Corrected by it: trigger callbacks are `(captures, line, wildcards, rawLine)`
> — captures FIRST. Code here that sniffs the arguments predates knowing that.

MudForge parses Lua with `luaparse` and **transpiles it to JavaScript**.
There is no Lua VM. Most of the time that's invisible; these are the places
it isn't. Every one below cost a debugging session, and `luac -p` passes
cleanly on all of them.

Sanitiser notes are equally hard-won: widget HTML goes through DOMPurify with
default config, and several obvious approaches silently do nothing.

---

## The Lua-to-JavaScript boundary

### 1. Varargs don't survive

```lua
local function get(...)
    for _, name in ipairs({...}) do   -- loop runs, every name is undefined
```

The loop executes and the values arrive `undefined`. Symptom was
`getGMCPData(): e.split is not a function`, twelve times per refresh. **Take
an explicit table.**

### 2. Arrays from JavaScript are 0-indexed

GMCP payloads, `getLoadedPlugins()` results, anything originating on the JS
side. Lua expects 1. Read from 0 upward and fall back to `ipairs`:

```lua
local i = 0
while list[i] ~= nil do ... i = i + 1 end
if #out == 0 then for _, v in ipairs(list) do ... end end
```

### 3. Lua arrays sent *out* become objects

Passing a Lua table to `sendGMCP` produced `{"1":"Core 1","2":"Char 1"}` — a
JSON *object*, because Lua arrays are 1-indexed. Aardwolf rejects that
outright and the subscription silently never takes. **Build the JSON text
yourself** and pass a string; `sendGMCP` forwards a string untouched.

### 4. Command arguments are a string *or* a table

`registerCommand` may hand over the raw argument string. Indexing a string
yields characters: `args[1]` off `"sex male"` is `"e"`, which then got
validated as a URL. Handle both shapes.

### 5. `undefined` is truthy

It is neither `nil` nor `false`, so Lua truthiness treats it as a value:

```lua
if cfg.size then                    -- passes when cfg.size is undefined
    resizeWidget(w, cfg.size.width) -- throws
```

**Use `type(x) == "table"` and `x == true`, never bare truthiness**, on
anything from GMCP, `loadTable` or a widget event. Related: `tostring(nil)`
prints `null`, and JS `undefined` reaching a string field arrives as the
literal text `"undefined"` — so `x or ""` never fires and a panel cheerfully
renders `SPRITE UNDEFINED`.

### 5b. A trigger created mid-packet never sees that packet

Register every trigger at `init` and gate it on state. Do NOT create one from
inside another trigger's callback to catch the lines that follow.

The identify box arrives whole — `Keywords`, `Name`, `Type`, all in one
packet. Arming the row trigger from the `Keywords` callback meant `Keywords`
was captured (it IS the opening line) and every row after it was missed, so
every record stored under an undefined name. The same defect silently broke
the `{roomchars}` player capture and Core's tag-block gag, where the whole
block arrives at once and went ungagged.

**The MUSHclient set had this right and the Mudlet port lost it.** MUSHclient
declares every trigger up front with `enabled="n"` and flips the flag;
Mudlet's `tempRegexTrigger` works there because it feeds triggers line by
line. Porting the Mudlet shape to this client reintroduces the bug — when the
two disagree, MUSHclient's is the one that survives a packet-at-a-time
runtime.

```lua
-- wrong: the rest of the box is already in this packet
addTrigger("^\\| Keywords", function()
    rowTrig = addTrigger("^[|+](.*)$", read_row, { type = "regex" })
end, { type = "regex" })

-- right: it exists before the packet does
addTrigger("^[|+](.*)$", function(c, l, w)
    if not inBox then return end
    read_row(c, l, w)
end, { type = "regex" })
```

### 6. Nested tables from `loadTable` aren't safely indexable

Even behind a `type()` guard. `cfg.size = { width, height }` kept throwing on
load. **Store scalars** — `cfg.sizeW`, `cfg.sizeH`.

### 7. Lua patterns run on a JavaScript regex engine

`%x`, `%d`, `%a`, `%s` mean nothing to it. `string.match(hex,
"^#(%x%x%x%x%x%x)$")` never matched, so every colour silently fell back to
white. Use `[0-9a-f]`, `\\d`, `[a-z]`, `\\s` — or plain `string.find(s, c, 1,
true)` and a manual loop.

### 8. `next` is not in the sandbox

`next(t)` fails the whole plugin load with `_G.next is not a function`. Use
`for _ in pairs(t) do return false end`.

### 9. No duplicate `local` in one scope

Lua allows shadowing; the transpiler emits `let` and rejects it with
`Cannot declare a let variable twice`, failing the **entire plugin load**.
Nested closures are fine — same block is not.

### 10. `onLine` takes three arguments, and the first is not the line

```lua
function onLine(sessionId, rawLine, cleanLine)
```

Written as `onLine(line)` every pattern matches against
`session-1785816768709-k8wpur` — 50 lines seen, 0 matches, and no error
anywhere to say why. `cleanLine` arrives with colour already stripped, which
matters for anything anchored at `^`: who output is heavily coloured and an
escape sequence in front of the text defeats the anchor.

Returning `false` discards the line. Returning nothing keeps it.

### 11. `string.find`'s `init` argument does not advance the search

```lua
-- hangs: q keeps returning the same position while from grows
local at, from = nil, 1
while true do
    local q = string.find(body, needle, from, true)
    if not q then break end
    at, from = q, q + 1
end
```

This locked the client hard enough to need a force quit. Every other
`string.find` in this codebase passes `init = 1`, so it was the first use of
the argument and nothing catches it by reading.

Walk by slicing instead — the haystack is strictly shorter each pass, so it
terminates whatever `find` does with its arguments:

```lua
local at, offset = nil, 0
while true do
    local q = string.find(string.sub(body, offset + 1), needle, 1, true)
    if not q then break end
    at = offset + q
    offset = at
end
```

**Put an iteration cap on any `while` driven by string length.** A loop that
stops shrinking its input takes the whole client with it, and there is no
error, no log line and no way to tell it from a slow machine.

### 12. Don't assume the whole stdlib

`table.concat(t, sep, i, j)`'s start-index form and similar corners are worth
avoiding. `io` and `os` exist but are restricted implementations.

### 13. `registerCommand` shadows the MUD's command of the same name

The client matches the first word of the input against registered commands
before it decides whether to send anything, and a match means it never sends.
So a plugin registering `search` eats Aardwolf's own `search all`, and one
registering `chat` eats the chat channel — the player types the command they
have used for years and gets plugin help back.

Nothing warns about this and nothing shows it in a plugin list; it only turns
up when someone uses the MUD command. Prefix anything that collides with a
real MUD command: `awsearch`, `awchat`. Check `help <word>` on the MUD before
picking a name, and remember Aardwolf abbreviates, so a short name can collide
with the start of a longer one.

### 9b. A `local` that shadows a file-scope name

The narrow, lethal case of #9. It does not throw where you wrote it — it takes
the **whole plugin down at load**:

```
[Plugin Error] loadPlugin(): Can't find variable: act
```

A **ReferenceError**, not a syntax error, which rules out anything to do with
comments or quoting and points straight at a real identifier.

```lua
local function act() ... end            -- file scope

function init()
    registerWidgetEvent(widget, "action", function(data)
        local act = tostring(data.action or "")   -- shadows it
    end)

    addTimer(2000, act, true)           -- ReferenceError, plugin never loads
end
```

The transpiler hoists that nested declaration to `init`'s function scope, so
`init`'s own reference — textually *after* the handler — resolves to a `let`
that has not been initialised yet.

**Ordinary duplicate locals across unrelated functions are fine and are
everywhere** — aw-snd declares `n` thirteen times, aw-loot twelve. What breaks
is reusing a name that already exists at **file scope**.

The first version of this note said "inside `init()`", because that is where
the `act` collision was. That was too narrow and it cost a second outage:
`left` broke the same way from inside `pots_load()`, a plain top-level
function that `init` happens to call —

```
[Plugin Error] timer in "Aardwolf Spellups": Can't find variable: left
```

Do not try to model which callers are safe. **Never reuse a file-scope name as
a local.** `tools/check-undefined.py` fails the build on any of them, and
turning that check on found four more sitting in aw-core, aw-snd and aw-who,
none of which had gone off yet.

### 13b. Every return path of a function must be the same width

Multiple assignment compiles to **destructuring**, so a function that returns
two values on one path and one on another breaks both ways at once:

```lua
local function can_cast(num)
    if not skills[num] then return false, "not in your skill table" end
    return true                     -- one value
end

local ok, why = can_cast(id)        -- "true is not iterable"
local ok      = can_cast(id)        -- ok is the PAIR, which is truthy
```

The first is loud — `action handler for widget "…": true is not iterable`.
The second is silent and much worse: the whole `[false, "reason"]` pair is
collected into `ok`, and a pair is an object, so it is **truthy**. Every spell
in the Spellups panel read `cast`, including ones the character could not
possibly cast, and the entire potion path was unreachable. One cause, one
visible symptom, one invisible one.

Pad the short path — `return true, ""` — rather than fixing it at the call
site, because the next call site will get it wrong again.

### 13c. A field that `pairs` can see may still be unreachable by name

Measured against the mapper. `findPath` returns the documented table and
printing it whole shows every field:

```
raw: { found=false  distance=0  error=Start room NaN not found  directions={} }
```

...and `res.found` on that same table reads **undefined**. `pairs()` sees the
keys, dot access does not. The reliable route is to walk the pairs and compare
the key as text:

```lua
local function field(t, name)
    if type(t) ~= "table" then return nil end
    if t[name] ~= nil then return t[name] end
    for k, v in pairs(t) do
        if tostring(k) == name then return v end
    end
    return nil
end
```

Related, and the reason positional reads need the same care: **keys arrive as
strings**, so `v[0]` and `v["0"]` are not interchangeable. Test a key with
`tonumber(k) ~= nil`, never `type(k) == "number"` — a check written the second
way silently never fires.

### 13d. Mapper calls that return one value arrive plain; more than one arrives as an array

`getPlayerRoom()` and `getAreaList()` come back as a number and a map.
`getMapRoom(vnum)` comes back as `{ 0 = <the room>, 1 = null }` — the extra
element is the second return value.

**`searchRooms(q)` is NOT one of these.** It returns a single value that
happens to be a list: `{ 0=room, 1=room, 2=room }`. An unwrap rule of "all
positional keys and a table at 0" collapses that to the first room, and the
print that follows then enumerates that room's eleven FIELDS as if they were
the results — three hits reported as one, convincingly. Unwrap only when
element **1 is not a table**; `at(v,1) ~= nil` will not do, because null is not
nil in this runtime and `getMapRoom`'s tuple would stop unwrapping.

The cheap check: `shape(raw)` before `shape(val)`. A count that shrinks between
them is the bug.

### 13e. Read the API doc before designing against probe output

`searchRooms(query, opts?)` is documented as returning an array of room tables,
with `opts = { exact = false, caseSensitive = false }`. Half a day went into
inferring that shape from print output and building a whole `getAreaRooms` +
`getMapRoom` name index to work around a limitation that did not exist. The
doc is `docs/MUDFORGE-API-GUIDE.md` §11.5, locally, and mudforge.org 403s a
fetch — so read the local copy.

Two things the doc settles that guessing got wrong:

- `exact = true` is still **case-insensitive** unless `caseSensitive = true`.
  Dortmund's "The common room" matches a query for "The Common Room".
- `searchRooms` is map-wide with no area option, so a room name resolves to
  several areas. Filter on the room's own `.area`; do not infer the area from
  the name.

### 13f. Custom exits: `;;` stacks commands, `wait(N)` is the client's

`mapper help cexits`:

    ;;        stack multiple commands     (open door;;north)
    wait(N)   pause N seconds mid-move    (push lever;;wait(2.5);;go)

`findPath().directions` is an array of **movement commands**, so a step through
a custom exit arrives as the whole string. Aardwolf has no client-side
separator — send that verbatim and the semicolons go on the wire as text, and
`wait(2)` is a word the MUD does not know.

`findPath` costs a compass exit and a custom exit the same, so where a room has
both (47195 has `w: 49260` **and** `west;;say hi: 49260`) the route comes back
as the bare compass move. That is backwards: a custom exit exists because
someone recorded what actually works. Substitute it back in using the path's
own `vnums` plus `getSpecialExits(room)` → `{ [command] = destVnum }`.

Better still, hand a route containing one to `walkTo(vnum, cb)` — the client's
walker already knows the separator and honours the pauses. It needs the map
widget mounted, and answers `{ success = false, error = ... }` after ~2s when
it is not, so keep a hand-sent fallback.

### 13g. `ipairs` does NOT walk a mapper array — the keys are strings

The guide says the 0-indexed lists "iterate with `ipairs(list)` (which converts
correctly for you)". In this build it does not. The keys arrive as **strings**,
so `ipairs` looks up the NUMBER `1`, finds nothing, and stops **before the
first element**. It returns an empty walk, which reads exactly like "nothing
matched" — no error, no warning.

Measured: `searchRooms("The Common Room")` returns three rooms, and an `ipairs`
loop over that same table yields none. The same probe resolved the room
correctly one version earlier using a `pairs` walk.

This is the same reason `at(v, i)` has to try `v[i]` and then `v[tostring(i)]`.

Read them positionally, tolerating either key type, and keep the order:

```lua
local function mapl(v)
    local out = {}
    if type(v) ~= "table" then return out end

    local i = 0
    while true do
        local e = v[i]
        if e == nil then e = v[tostring(i)] end
        if e == nil then break end
        table.insert(out, e)
        i = i + 1
    end
    if #out > 0 then return out end
    -- then try 1-based, then pairs as a last resort
end
```

`pairs` must stay the LAST resort: its order is undefined, and `findPath`'s
directions scramble into a route that walks somewhere else entirely.

**A Lua-table fixture hides this.** `{ "n", "n", "e" }` is 1-indexed with
number keys, so `ipairs` walks it happily in the sim and fails in the client.
Build mapper fixtures the way the bridge really hands them over —
`{ ["0"] = "n", ["1"] = "n" }` — or the test passes on code that cannot work.

### 13h. What the mapper calls actually cost

Timed by `private/aw-probe.lua` (`probe cost`) against a 23,879-room map.
Two runs, so the spread is real rather than a single sample:

| call | per call |
|---|---|
| `getMapRoom(vnum)` | 0.004 – 0.010 ms |
| `getSpecialExits(vnum)` | 0.006 – 0.018 ms |
| `getPlayerRoom()` | 0.008 – 0.018 ms |
| `findPath(a, b)` | 0.020 ms |
| `getDistance(a, b)` | 0.040 – 0.060 ms |
| `getAreaList()` | 0.140 – 0.210 ms |
| `searchRooms(q, {exact=true})` | **1.10 – 1.20 ms** |
| `searchRooms(q)` | **1.85 – 1.95 ms** |

`searchRooms` is 50–450× everything else and is the only one worth caching or
gating. Room reads are effectively free — a per-step `getSpecialExits` over a
200-step path costs ~1.2 ms, less than one `searchRooms`.

Note `findPath` is CHEAPER than `getDistance`, which is worth knowing before
reaching for the "cheap" one.

### 13i. An unassigned `local` is `undefined`, and `undefined == nil` is FALSE

```lua
local best, bestDist                        -- transpiles to a bare `let`
...
if d and (bestDist == nil or d < bestDist) then best, bestDist = vn, d end
return best
```

`bestDist` is `undefined`, so `bestDist == nil` is **false**, `d < undefined`
is false, and `best` is never assigned. The function then returns `undefined`
— which is **truthy** (note 5), so the caller does not take its not-found
branch either and prints the word "undefined" where a room number should be.

That is exactly what `aw-probe` did: it resolved rooms correctly on 1.4.2
because that version returned early on a single hit and never reached the
comparison, and broke the moment the early return went away.

Three things all agree it is fine: `luac -p` parses it, the Lua sim passes it
(real Lua nil-initialises locals), and the shape is idiomatic Lua. Only the
client disagrees.

**Assign a sentinel and compare against a number:**

```lua
local best     = nil
local bestDist = 1e9
if d and d >= 0 and d < bestDist then best, bestDist = vn, d end
```

Where the accumulator is not numeric, carry its measure alongside
(`bestLen = -1`) rather than re-deriving it from the accumulator and testing
that for nil. `check-undefined.py` reports declaration-with-no-value followed
by a `== nil` test.

### 5b. ...and an unset TABLE FIELD is undefined too

Note 5 says undefined is truthy. The version of it that actually ships bugs is
the table field nobody declared:

```lua
local eng = { q = {}, saw = {} }          -- checking not declared

eng.check = function() eng.checking = true end

-- ...in a trigger, long before eng.check has ever run:
if eng.checking then eng.saw[id] = true end
```

Real Lua nil-inits the field and the guard behaves. The client makes it
`undefined`, which is truthy, so the branch is taken from the first line the
MUD prints. In aw-spellup that was four `[Trigger Error]` lines on a plain
`spellup`.

**Declare every field in the constructor, false ones included.** `false`, not
`0` and not `nil`: `nil` in a table constructor declares nothing at all, and 0
is truthy in Lua and falsy in JS, which is not a difference to build on.

The same bug was live in aw-portrait: `local ch = {}` and `if ch.level then
"Lv. " .. ch.level` printed **"Lv. undefined"** instead of "Lv. --" any time
`char.base` had not arrived.

**A forward declaration is the same bug wearing a hat.** `local render` at
column 0 with no value is fine; a function defined ABOVE that line and calling
`render()` is not, because it binds to the global. aw-spellup had
`eng.check_done` calling `render()` two hundred lines above `local render`, and
it stayed invisible until a check finally took rows off the panel and something
had to repaint. Put forward declarations near the top, above everything that
uses them.

`tools/check-undefined.py` checks this now. It reports truthiness tests only —
`t.x == nil` and `t.x == false` say what they mean — and it brace-matches the
constructor rather than searching for the closing `\n}`, because a one-line
table closes on its own line and the first version of the check silently
treated the rest of the file as the table body. It reports forward
declarations used above their `local` line as well — but only bare ones, since
a general version of that check needs real scope analysis and a first attempt
flagged thirty function-locals that merely shared a name.

### 13b-bis. A helper whose paths return different COUNTS

Note 13b says return one value. The way it actually ships is a helper that
returns two on one path and one on another, read with multiple assignment:

```lua
local function as_place(s)
    if KNOWN[k] then return { name = k, ... } end       -- one
    if #hits > 1 then return nil, table.concat(hits) end -- two
    return nil                                          -- one
end

local hit, ambiguous = as_place(whole)   -- destructuring
```

Multiple assignment transpiles to destructuring, so every path that returns
ONE value hands a plain object to something expecting an iterable:

    Plugin command error: TypeError: {} is not iterable

The cruelty is which path breaks. In aw-coords the two-value path was the rare
one — an ambiguous name — so `/coord fury` and `/coord 11,8`, the cases that
work, were the cases that crashed. aw-loot's `merge_dump` returned `nil` only
when the JSON would not decode, so a malformed paste took the import down
instead of reporting a bad file. aw-healme's `pct()` returned `nil` when
char.vitals had not arrived, which its own comment says happens early in every
session, so the healer crashed exactly when its guards were doing their job.

**A sim cannot catch this.** Real Lua pads a short return with nil and never
complains; only the transpiler minds. Reverting the aw-coords fix leaves
`tools/sim-coords.lua` passing. `tools/check-undefined.py` is the only guard,
and it reports any function destructured at a call site whose paths disagree.

Return a table with the extra information in a field. Nothing to destructure,
nothing to get wrong at the call site.

### 17. Lua's 200 file-scope local limit is a real ceiling

`aw-spellup.lua` is AT it. Adding one more `local` at file scope fails with

    too many local variables (limit is 200) in main function

This is real Lua, not the transpiler — the client compiles to JS and would not
care — but `luac -p` is the syntax gate and `tools/harness.lua` runs real Lua,
so it stops the build either way.

New machinery in a file near the limit goes in ONE table, not six locals.
`aw-spellup` 1.13 added a whole engine — a queue, a coverage map, a send-time
map, a retry counter and six functions — inside a single `local eng`:

```lua
local eng = { q = {}, cov = {}, last = {}, tries = {}, at = 0 }

eng.why_not = function(num) ... end
eng.fire    = function(num) ... end
eng.drain   = function() ... end
```

`eng.fire = function()` is an assignment, not a declaration, so it costs
nothing. `local function eng_fire()` would have cost a slot each.

Two things that are easy to get wrong doing this:

**Define the methods after everything they call.** A closure created before
`local held` is declared does not see it — it reads the global, which is nil,
and the transpiler's `let` makes it a TDZ error instead. The engine block sits
directly below `held()` for exactly that reason, not next to the table.

**Freeing a slot is usually a constant.** One file-scope constant used by one
function becomes a literal inside it. `POT_AFF_WAIT = 5000` paid for `eng`.

And check the name is free first. `cast` is already taken in aw-spellup — a
duplicate file-scope local is note 9b, and it fails the whole plugin load.

### 14. An optional capture group must be last, or not exist

Every plugin here reads captures through the same helper, which works out
whether they arrived 0- or 1-based by asking whether the first one is a
string:

```lua
local base = (type(c[0]) == "string") and 0 or 1
local v = c[n - 1 + base]
```

That inference is sound right up until a group is allowed not to participate.
A non-participating group is not a string, so a pattern whose **first** group
is optional reads as 1-based and every lookup shifts by one — asking for
capture 1 hands back capture 2. In a container listing that is the item's name
where its quantity should be, silently, with no error.

```
^(?:\((\d+)\)\s+)?(.+)$      -- (  3) a potion   fine
                             -- a potion         quantity reads "a potion"
```

Two patterns cost nothing and have no inference in them:

```
^\(\s*(\d+)\)\s+(.+)$        -- counted
^(?!\()\s*(\S.*)$            -- not counted
```

A **trailing** optional group is safe, because nothing after it moves —
`(\d+):(\d+)(?::(\d+))?` for `MM:SS` and `HH:MM:SS` is fine.

Related, and the reason a missing group is worse here than in real Lua: when
one does arrive as `""`, **`tonumber("")` is `nil` in Lua and `0` in
JavaScript**. So the empty capture doesn't fail a numeric guard, it passes one
with a confident zero. Test the string before converting it.

### 11b. When the client spins, `sample` the WebContent process — don't theorise

The client locked up mid-combat and the first theory was disk: its IndexedDB
store had grown to 5.6 GB and it snapshots the world every five seconds. That
theory was wrong, and an hour went into it before anything was measured.

The WebContent process is a separate PID from the Tauri host, and it is the
one to look at:

```
ps aux | grep WebKit.WebContent      # find the PID at ~100% CPU
sample <pid> 6 -file /tmp/hang.txt
```

The host process sat at 0.0% CPU the whole time. WebContent was at 95.2%,
which already rules out waiting on I/O — a stall on disk is a process asleep,
not a process spinning.

What came back:

```
operationValueAdd → jsAddSlowCase → JSObject::toPrimitive
                  → JSArray::fastToString → JSStringJoiner::joinImpl
operationValueSub → JSString::toNumber → fast_float::from_chars
```

Both shapes are the transpiler boundary showing through. `JSArray::fastToString`
means an **array** was an operand of `+` — and a Lua function returning two
values (`string.find` returns start AND end) arrives here as an array, so
`string.find(...) + 1` is `array + number`, which JavaScript resolves by
joining the array into a string. `JSString::toNumber` on the other branch is
that string being coerced back for arithmetic. Neither throws. Both are silent
in real Lua, where the extra return is simply dropped.

So a loop written as

```lua
while string.sub(n, 1, 1) == "(" do
    local close = string.find(n, ")", 1, true)
    if not close then break end
    n = trim(string.sub(n, close + 1))       -- close is not a plain number
end
```

never shortens `n`, and there is no exit. This is NOTES #11 with a different
verb, and it shipped anyway because the code reads exactly like Lua.

**The rule that actually holds**: a loop must not depend on a boundary value
being the type it looks like. Make the termination structural — every pass
strictly shortens the string, or the loop stops:

```lua
local rest = trim(string.sub(n, close + 1))
if string.len(rest) >= string.len(n) then break end
n = rest
```

Nothing `string.find` or `string.sub` do at the boundary can defeat that.

### 14b. The trigger callback's second argument is the LINE, not the captures

```lua
addTrigger(pattern, function(captures, line, wildcards, rawLine) … end, opts)
```

Two of aw-snd's handlers were written `function(a, b)` and then read
`as_text(a, b)`, which is the helper that returns a whole line when it is
handed one. It is right for the handlers that want the line. It is wrong for
a handler whose own pattern has just carved the mob out of that line, and both
of those had one:

```lua
"^\\s*(?:\\((?:[^)]*)\\)\\s*)*(.+?)\\s+is (?:standing|sitting|…|here)"
```

The capture is `a lizardman citizen`. What went into the database was

```
(red aura) a lizardman citizen is here complaining about the catfolk.
your slice ***** pulverizes ***** a complaining lizardman citizen! [104]
```

632 rows of 2236 before it was found, and nothing errors — a line is a
perfectly good string. It only shows up when you read the data. Use `cap3`
(the 0-/1-base inference of NOTES #14) when you want a group; `as_text` only
when you want the line.

### 14c. Triggers are case-INSENSITIVE by default

`addTrigger`'s `caseSensitive` defaults to `false`, so a lowercase alternation
matches shouting:

```lua
"\\b(?:destroys|…|pulverizes)\\b…"   -- fires on '***** PULVERIZES *****'
```

`tools/check-regex.py` was compiling every pattern case-sensitively, which
made it *stricter* than the client: patterns passed the checker while matching
lines in play that the checker said they could not. It compiles with
`re.IGNORECASE` now. The whole existing corpus still passes under it, which is
the only reason the change was safe to make in one go.

A fixed-width column is worth the same warning. `where` prints the mob in a
thirty-character field with **one** space after it, so a name that fills the
field leaves a single space and a `\s{2,}` pattern never fires at all:

```
a complaining lizardman citize Inside the Temple      <- 30 chars, one space
a lizardman hunter             Jungles of Verume      <- 18 chars, padded
```

### 16. A `hyperlink` command reaches a PLUGIN command, and keeps the prompt

Measured, not assumed. The guide says a link's action is "sent to the MUD",
which reads like it bypasses the client's own command matching. It does not:

```lua
hyperlink("$G[keep it]$w", 'awbuff add "potion eyes wolf"')
```

...runs the plugin's registered `awbuff` command. It is not echoed to
Aardwolf, so there is no `Huh?` to design around.

**This is the only clickable control that does not steal the prompt.** A
`data-mud-action` div in a widget moves focus into that widget's iframe, and
there is nothing a plugin can do about it — `/awcore api focus` and
`/awcore api input` both return **0 functions**, so no focus API exists to
call, and DOMPurify strips any script that would move focus back. Terminal
links live in the terminal, so typing carries on straight afterwards.

Worth reaching for when an action is something you fire and then keep playing:
S&D's GO, a "keep it" on an identify, anything mid-combat. A panel button is
still right for settings, where losing the prompt for a moment costs nothing.

---

## Widget HTML and the sanitiser

Content set with `setWidgetProperty(id, "content", html)` is run through
DOMPurify with **default config**, inside an iframe.

### What does not work

| approach | result |
|---|---|
| `style="background: rgb(…)"` | colour stripped |
| `style="background-image: url(…)"` | stripped |
| a **second** `<style>` element | ignored entirely — only the first applies |
| `onclick=` and friends | stripped |

Colour and `url()` in an inline `style` attribute are dropped while
*geometry* in the same attribute survives — `left: 40%` positions fine. That
asymmetry is confusing and cost three attempts to pin down.

### What does work

- **One `<style>` element**, containing everything. A dynamic `url()` inside
  it is fine; that's how portraits and frames load.
- **Class names on markup.** Enumerate a palette as static rules and have
  Lua pick a class. This is the only reliable way to vary colour.
- **`data-mud-action` / `data-mud-data`** for clicks, delivered via
  `registerWidgetEvent(id, "action", fn)`.
- **`<form>` submits**, which carry every named field:
  `registerWidgetEvent(id, "submit", fn)` → `data.formData`. This is how to
  get typed text without JavaScript.
- **`<img src="data:image/png;base64,…">`** — DOMPurify's data-URI tag
  allowlist is `audio, video, img, source, image, track`.
- **`data-mud-bind="key"` with `setBoundValue(id, key, value)`** writes into
  a marked element without rebuilding the widget. Use it for anything that
  updates faster than the user interacts — a one-second countdown re-rendering
  the whole panel is sixty rebuilds a minute, and it resets scroll each time.

### `<a href>` renders, and clicking it crashes the client

Not stripped — that was a guess and it was wrong. An anchor survives the
sanitiser intact, keeps its `class`, and styles correctly. What it does on
click is navigate the **widget's own iframe** to the URL, replacing the panel
with the web page, and MudForge goes down with it. `target="_blank"` does not
help: the frame has no permission to open a window, so it navigates in place
instead.

Tested four ways in one panel — bare anchor, classed anchor, anchor with
`target`, and a `data-mud-action` span. All four render and all three anchors
take the client out.

**Never emit `href` in widget HTML.** To reach a browser, use a
`data-mud-action` span and have the handler call `hyperlink(url, url)`, which
prints a terminal link; an `http(s)` action there opens the browser properly.
That is two clicks and it is the only route that exists.

The same applies to the `prompt:` scheme, which was tried in the shop panel and
did nothing at all — the anchor renders and the scheme is inert.

### Prefer CSS backgrounds to `<img>` for anything that might 404

A failed `<img>` draws the browser's broken-image placeholder. A failed
background layer paints nothing, so a fallback underneath shows through:

```css
background-image: url("remote.png"), url("data:image/png;base64,…");
```

---

## Widgets

- **`name` is required** if a plugin creates more than one. The widget id is
  `pluginId + name`, defaulting to `"widget"` — two unnamed widgets resolve
  to the same id and the second silently returns the first.
- **Saved geometry wins.** MudForge restores per-widget position and size, so
  changing a default only affects a fresh profile. A widget can be restored
  underneath another panel and behind it in z-order: visible by every
  measure, and impossible to see.
- **Re-rendering resets scroll.** Rebuilding the whole widget on every click
  throws a long list back to the top. Use form controls the browser toggles
  locally and apply on submit.

---

## Plugin loading

- **MudForge caches plugin source at install.** Updating the file on disk is
  not enough; remove and re-add to load new code. The file watcher only
  hot-reloads plugins it *already* has loaded.
- **Managed copies are named by display name** — `Aardwolf Core.lua`, not
  `aw-core.lua`. Dropping a source-named file alongside loads the plugin
  twice.
- **Version-stamp your output.** Chasing a bug that was already fixed on disk
  costs rounds you cannot get back:
  `local TAG = "$Y[Name v" .. plugin.version .. "]$w "`.

---

## The API surface is much bigger than the docs

`/awcore api` enumerates `_G` and prints every bound function. On 1.2.2011
that's **326 functions**, most of them undocumented. Before concluding a
plugin can't do something, run it — reasoning from the docs, or from the
shape of the IndexedDB stores, gets it wrong.

Two compatibility layers are in there alongside the native API:

| layer | examples |
|---|---|
| MUSHclient | `Send`, `Note`, `ColourNote`, `Execute`, `GetVariable`, `SetVariable`, `EnableTrigger`, `DoAfterSpecial`, `SaveState` |
| Mudlet | `addSpecialExit`, `getExitStubs1`, `setCustomEnvColor`, `tempRegexTrigger`, `tempTrigger`, `getRoomUserData`, `echo`, `cecho` |

**The mapper is fully writable**, which matters because the map's own
IndexedDB `connections` store is only `fromRoom / toRoom / direction / level /
weight` — no command field. Reading that schema suggests custom exits are
impossible. They aren't; they're just held somewhere else:

```
addSpecialExit  removeSpecialExit  clearSpecialExits  getSpecialExits
setDoor  getDoors  lockExit  hasExitLock  setExitStub  connectExitStub
addRoom  deleteRoom  setExit  setMapExit  setRoomName  setRoomArea
setRoomCoordinates  setRoomUserData  setRoomColor  setCustomEnvColor
createMapperWidget  importMapJson  exportMapJson  refreshMap
gotoRoom  getPath  findPath  setWalkDelay  setFastWalk  stopWalk
onMapReady  onMapUpdate  onMapRoomClick  onRoomChange
```

**`addSpecialExit(from, command, to)`** — command in the MIDDLE. Not Mudlet's
`(from, to, direction)`, despite the name coming from that compatibility layer.
The wrong order returns `false` and leaves `getSpecialExits` empty; it does not
throw, so a `pcall` around it reports success and the exits are silently
dropped. Treat a `false` return from any mapper writer as a failure.

So an Aardwolf mapper is a plugin driving the built-in engine, not a
replacement for it. `/awcore map` prints what each reader returns for the
room you're standing in — names alone don't give away argument order or
return shape.

---

## GMCP

### `onGMCPUpdate` gives you the packet. `getGMCPData` gives you the store.

They are not the same thing, and the difference only shows up on packages that
are **event-shaped** rather than **state-shaped**.

`getGMCPData(name)` returns what that package has accumulated — the last
message, with fields from earlier messages still sitting in it where the newer
one didn't overwrite them. For `char.vitals` that's exactly right: every packet
carries the full set, so the store *is* the current state and reading it on
load is how a panel fills itself in.

For `comm.quest` it is a trap. The packets are events and each carries only
what that event needs:

```
{ action = "start", targ = "a chilling silence", area = "...", timer = 30 }
{ action = "comp",  wait = 30 }          -- no targ; the old one survives
```

So an hour after turning the quest in, `getGMCPData("comm.quest")` still hands
back `targ = "a chilling silence"`. Seeding a panel from it resurrects quests
that are long dead, and it looks convincing because the rest of the record is
real.

**Ask instead of reading.** `sendGMCP("request quest")` comes back as a fresh
`status` packet through `onGMCPUpdate`, which is the only version that's true.
Same for anything else event-shaped — `comm.channel` has the same problem.

Time the request: Core defers its supports rebuild ~700ms after a plugin
enables, and the package has to be subscribed before a reply can arrive.
1500ms has been reliable.

### A reply that answers "no" still carries fields

`request quest` between quests answers with `action = "status"`, **no target,
and a zero `timer`**, alongside the `wait` that is the number you actually
want. Branch on the *data* — is there a target? — not on the action. Reading
the timer first rendered `0:00 left` on a panel with 25 real minutes on it.

---

## Aardwolf specifics

### noexp is on the prompt, not in GMCP

`char.status` carries `level` and `tnl` and nothing that says whether
experience is being earned, and `Help/Prompt` documents no code for it. The
MUD marks it on the prompt itself:

```
[368/368hp 219/219mn 717/717mv 54qt 1270tnl] >*[NOEXP]*
```

Read state off the prompt rather than off a toggle message. The prompt says
what is true *now*, so a reload, a reconnect, or a session that started in
noexp all settle on the first one — and the marker being absent is exactly as
meaningful as it being present, which gives both directions from one signal.
A toggle message only tells you about the moment it changed, and only if you
were listening.

- `Core.Supports.Set` **replaces**, never merges. See [CORE.md](CORE.md).
- `char.vitals` carries current values only; the maxima are in
  `char.maxstats`. Percentages need both.
- The group package is literally `group`, not `Group.Members`, and is **off**
  until `group on`.
- `room.info.exits` is an object keyed by direction, not an array.
- `char.base.class` is sometimes absent. Subclass maps to exactly one primary
  class, so derive it.
- `comm.quest.wait` is **minutes** — the stock client multiplies it by 60.
  `comm.quest.timer` is undocumented there (its branch is an empty comment).
  Quests are granted in the 30-60 minute range and cap well under two hours, so
  a value past 180 can't be minutes and has to be seconds.
- `remorts` counts from **1** for an unremorted character, not 0.
- Alignment runs −2500..2500: Good ≥ 875, Evil ≤ −875.
- `gmcpchannels on` means channels go over GMCP **only** — a server-side gag
  on every channel at once.
- `omitFromOutput` hides a line's text but **keeps its row**, leaving a blank
  gap. Returning `false` from `onLine` removes the line properly. Note
  `onLine` never sees blank lines at all — the client filters them before
  line handlers run.
