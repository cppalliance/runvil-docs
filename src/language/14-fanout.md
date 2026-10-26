# Fanout

`tasks.fanout` runs one worker section once for each member of a collection, and each of those runs is an arm. The arms run concurrently, and the call hands back their results in collection order, whatever order the arms finish in. This chapter shows how to name the worker section, build the collection, read the results, and keep arms from colliding in the store, so you can spread one piece of work over many inputs and still get the same result on every run.

## Fanout in one call

In this section you run one worker section over a three-member list with a single call and read back one result per member.

````markdown
---
name: fanout-basic
description: Runs one worker section for each of three members.
runvil: 0
---

# Fanout basic

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta', 'gamma'})
return r[1].text .. ',' .. r[2].text .. ',' .. r[3].text
```

### Worker

```lua
return item .. '-done'
```
````

````text
alpha-done,beta-done,gamma-done
````

The [section walk](04-how-a-prompt-runs.md#the-section-walk) enters `## Parent`, whose block makes one fanout: a single `tasks.fanout` call together with the runs it starts. `tasks.fanout(worker, collection)` runs the worker section, the section a fanout runs, once for each member of the collection, the Lua table of values you hand it. Here the worker section is `### Worker` and the collection is `{'alpha', 'beta', 'gamma'}`, so `### Worker` runs three times. Inside each run, the global `item` holds that run's member, so the three runs return `alpha-done`, `beta-done`, and `gamma-done`. `tasks.fanout` returns a list with one result per member, indexed from 1, and each result's `.text` holds the text that member's run returned. `## Parent` joins the three texts and returns them as the run result.

### Writing the call

`fanout` lives in the `tasks` table and is not a global of its own, so you always write `tasks.fanout`. The first argument is the worker section's heading reference as a string, such as `'### Worker'`, with its `#` markers included (see [Heading references](02-file-structure.md#heading-references)). The second argument is the collection. [Choosing the worker section](#choosing-the-worker-section) covers which sections the first argument can name, and [Collections and member order](#collections-and-member-order) covers what the second argument can hold.

### Arms and collection order

Each run of the worker section, one per member, is an arm of the fanout. A two-member collection starts exactly two arms. A one-member collection such as `{'only'}` runs a single arm and returns a one-entry result list, so with a worker section that returns `item`, `r[1].text` is `only`. Inside each arm, the worker section's Lua reads its own member through the global `item`, and [What an arm sees](#what-an-arm-sees) lists everything else an arm starts with.

The arms run concurrently with each other, not one after another. In this version each arm sends two prompts with [`models.infer`](10-models.md#running-a-round-with-modelsinfer), so each arm runs two model rounds:

````markdown
---
name: fanout-rounds
description: Sends two prompts from each of two arms.
runvil: 0
models:
  writer: {}
---

# Fanout rounds

```lua
models.default('writer')
```

## Parent

```lua
local r = tasks.fanout('### Worker', {'one', 'two'})
return r[1].text .. '|' .. r[2].text
```

### Worker

```lua
local first = models.infer(item .. ':1')
local second = models.infer(item .. ':2')
return first .. ' ' .. second
```
````

The arm for `one` sends `one:1` and the arm for `two` sends `two:1` before either arm sends its second prompt. The result list still follows collection order, the fixed order of the collection's members, which for a list like this one is position order. It never follows the order the arms finish in, so `r[1]` always belongs to `one`, even when the arm for `two` finishes first, and the run text always starts with the replies the `one` arm got. [Concurrency and the admission window](#concurrency-and-the-admission-window) explains how the arms share the run.

### The call waits for every arm

`tasks.fanout` is a [suspending call](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), one that waits on the Harness while other chains run, so the calling section pauses inside the call while the arms run. The call returns only after every arm has finished, so the line after it sees every arm's result, including the result of the arm that finished last.

Because the call waits for every arm first, your code after `tasks.fanout` also sees everything the arms did, including the store files they wrote. A `store.glob` right after the call finds every file the arms wrote, as [The store across arms](#the-store-across-arms) shows.

No arm keeps running after `tasks.fanout` returns or raises. When an arm fails, the call cancels every arm still running, arms that have not started never start, and the failing arm's error is raised from the `tasks.fanout` call. [When an arm fails](#when-an-arm-fails) covers that case in full.

### Where you can call `tasks.fanout`

You can call `tasks.fanout` from a section's prologue, the `lua` block before its prose, as `## Parent` does above, or from its epilog, the `lua` block after its prose (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)):

````markdown
---
name: fanout-epilog
description: Calls tasks.fanout from the epilog of a section.
runvil: 0
---

# Fanout epilog

## Parent

The arms below each label one member.

```lua
local r = tasks.fanout('### Worker', {'x', 'y'})
return r[1].text .. ',' .. r[2].text
```

### Worker

```lua
return item .. '-done'
```
````

````text
x-done,y-done
````

You can also call `tasks.fanout` as a bare statement that throws away the result list, as in `tasks.fanout('### Worker', {'alpha'})` on a line of its own. The arms still run, so a failing arm still raises its error from the bare call.

A block can call `tasks.fanout` more than once in sequence, and each call returns its own result list. After `local a = tasks.fanout('### Worker', {'one'})` and then `local b = tasks.fanout('### Worker', {'two'})`, `b[1].text` comes from the arm for `two`.

A section run by [`call`](08-jump-and-call.md#calling-a-section) can itself call `tasks.fanout`, and so can the H1 body's Lua. [Fanout from the H1 body and from called sections](#fanout-from-the-h1-body-and-from-called-sections) shows both.

## Choosing the worker section

In this section you name the worker section and learn which sections that name can reach.

````markdown
---
name: sibling-worker
description: Fans out to a sibling section.
runvil: 0
---

# Sibling worker

## Main

```lua
local r = tasks.fanout('## Worker', {'x'})
return r[1].text
```

## Worker

```lua
return item .. '-done'
```
````

````text
x-done
````

### Matching the heading reference

You name the worker section in the first argument of `tasks.fanout` by its heading reference, `#` markers included, the same form `call` and `jump` take. The heading reference must match a visible section's level and name exactly: one or more `#` markers equal to the section's level, then whitespace, then the section's non-empty name. A section matches only when both its level and its name are equal, so a `### Worker` child is named `'### Worker'`, and in `sibling-worker` the string `'## Worker'` names the top-level `## Worker`.

### The caller's visible set

Any section in the caller's visible set can be the worker section. The visible set is the set of sections a heading reference can name from the current section (see [The visible set](08-jump-and-call.md#the-visible-set)): the other sections at the caller's level under the same parent, plus the caller's direct children. So the worker section can be a `###` child of the calling section, as `### Worker` is under `## Parent` in [Fanout in one call](#fanout-in-one-call), or a `##` sibling of it, as `## Worker` is to `## Main` above.

The visible set never includes the calling section itself, its parent, its parent's siblings, its siblings' children, or its grandchildren. Here `### Niece` sits under `## Other`, so it is a child of a sibling and outside the visible set of `## Main`:

````markdown
---
name: niece-worker
description: Names a section outside the visible set.
runvil: 0
---

# Niece worker

## Main

```lua
local r = tasks.fanout('### Niece', {'x'})
return r[1].text
```

## Other

Other work lives here.

### Niece

```lua
return item
```
````

The call raises an error value of kind `lua` whose message has the form ``section heading `{heading}` not found; available sections: {list}``, naming the heading reference you passed and listing the sections the caller can name. Left uncaught, it ends the run.

### Child worker sections and top-level worker sections

You usually place the worker section as a `###` child under the calling `##` section. The walk never runs a child section by falling through to it (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)), so a child worker section runs only as a fanout arm and never on its own.

A top-level worker section is also a section of the main walk, which reaches it in order unless an earlier section returns or jumps. So you place a top-level worker section after a caller that returns. In `sibling-worker`, `## Main` returns, which ends the run before the walk reaches `## Worker`.

One worker section can serve several callers, each fanning out to it on its own:

````markdown
---
name: shared-worker
description: Two sections fan out to the same worker section.
runvil: 0
---

# Shared worker

## A

```lua
var.first = tasks.fanout('## Worker', {'a'})[1].text
```

## B

```lua
local second = tasks.fanout('## Worker', {'b'})[1].text
return var.first .. ',' .. second
```

## Worker

```lua
return item .. '-done'
```
````

````text
a-done,b-done
````

`## A` keeps its arm's text in [`var`](05-lua-environment.md#keeping-state-in-var) and falls through to `## B`, which fans out to the same `## Worker` and returns before the walk reaches it.

### Running a section as a worker section or a plain section

A child worker section runs once per member under `tasks.fanout`, each run with its member as `item`. A `call` or `jump` that names the same section runs it as a plain section instead, with `item` set to nil.

A worker section can hold anything an ordinary section holds. A `---` thematic break in it only resets its pending prose, as in any section (see [Thematic breaks and setext headings](03-blocks-and-prose.md#thematic-breaks-and-setext-headings)), so the `lua` fence below the break still runs in every arm:

````markdown
---
name: break-worker
description: A worker section with a thematic break above its block.
runvil: 0
---

# Break worker

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
return r[1].text .. ',' .. r[2].text
```

### Worker

---

```lua
return item .. '-done'
```
````

````text
alpha-done,beta-done
````

A section whose body is only list items, with no `lua` block, is a list section (see [List sections](03-blocks-and-prose.md#list-sections)), and a list section cannot be the worker section. Naming one raises an error value of kind `lua` with the message ``section `{name}` is a list section, not a worker template``, where `{name}` is the section name without its `#` markers. So `tasks.fanout('### Items', {'x'})`, aimed at a `### Items` section holding `- a` and `- b`, fails with ``section `Items` is a list section, not a worker template``. To run a fanout over a list section's entries, pass the list section as the collection instead, as [List collections](#list-collections) shows.

### Fanout from the H1 body and from called sections

`tasks.fanout` works in the H1 body's Lua the same way it works in any section (see [The H1 pass](04-how-a-prompt-runs.md#the-h1-pass)). From the H1 body, the worker section's heading reference resolves against all of the prompt's top-level sections, and the arms still join in collection order:

````markdown
---
name: h1-fanout
description: Fans out from the H1 body.
runvil: 0
---

# H1 fanout

```lua
local r = tasks.fanout('## Worker', {'a', 'b'})
return r[1].text .. '|' .. r[2].text
```

## Worker

```lua
return 'item:' .. item
```
````

````text
item:a|item:b
````

The top-level `## Worker` runs once for `a` and once for `b`. The H1 block's return ends the run, so the walk never reaches `## Worker` on its own. When the H1 body names a heading reference that matches no top-level section, such as `tasks.fanout('## Nope', {'a'})`, the run fails with a resolution error whose message names the missing heading `## Nope`.

A section running in a chain that `call` started can name one of its own children as the worker section:

````markdown
---
name: called-fanout
description: A called section fans out over its own child.
runvil: 0
---

# Called fanout

## Parent

```lua
return call('### Sub')
```

### Sub

```lua
local leaves = tasks.fanout('#### Leaf', {'x', 'y'})
return leaves[1].text .. leaves[2].text
```

#### Leaf

```lua
return item .. '!'
```
````

````text
x!y!
````

`## Parent` calls `### Sub`, and `### Sub` runs `#### Leaf` once for `x` and once for `y`, then returns the joined texts to `## Parent` as the value of `call`.

## Collections and member order

In this section you build a list collection and a keyed collection and predict the order their arms take.

````markdown
---
name: list-collection
description: Fans out over the entries of a list section.
runvil: 0
---

# List collection

## Parent

```lua
local r = tasks.fanout('### Worker', prompt.list('### Items'))
assert(r[1].item == 'alpha' and r[2].item == 'beta')
return r[1].text .. ',' .. r[2].text
```

### Items

- alpha
- beta

### Worker

```lua
return item .. '-done'
```
````

````text
alpha-done,beta-done
````

The collection is `tasks.fanout`'s second argument, and it must be a Lua table with at least one member, where a member is one entry of the collection. Each member gets one arm. Here the collection comes from the list section `### Items`, and `r[1].item` and `r[2].item` hand back the member each arm ran on, as [Reading the results](#reading-the-results) explains.

### List collections

To fan out over a list section, you pass what `prompt.list(heading)` returns as the collection, as in `tasks.fanout('### Worker', prompt.list('### Items'))` (see [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist)). Each list item becomes one member and one arm whose `item` is the bullet's text, so the bullets `- alpha` and `- beta` give `r[1].item == 'alpha'` and `r[2].item == 'beta'`.

A literal table works the same way. A table of strings such as `{'alpha', 'beta'}` gives one arm per string, and you can also build the collection while the block runs, for example by filling a sequence table in a loop such as `for i = 1, 10 do items[i] = tostring(i) end` and passing that table to `tasks.fanout`. A collection built this way can be large, and every member runs as its own arm:

````markdown
---
name: many-members
description: Fans out over 1025 members built in a loop.
runvil: 0
---

# Many members

## Parent

```lua
local members = {}
for i = 1, 1025 do members[i] = 'm' .. i end
local r = tasks.fanout('### Worker', members)
assert(#r == 1025)
return r[1025].text
```

### Worker

```lua
return item
```
````

````text
m1025
````

### What a member can be

A member can be a string, a number, a boolean, or a nested table. One collection can mix all four in the same list, and each array-part member reaches its arm as `item` with the same value and its own type:

````markdown
---
name: mixed-members
description: Fans out over members of four types.
runvil: 0
---

# Mixed members

## Parent

```lua
local r = tasks.fanout('### Worker', {'b', 2, true, {nested = 'x'}})
assert(#r == 4)
return r[1].text .. ',' .. r[2].text .. ',' .. r[3].text .. ',' .. r[4].text
```

### Worker

```lua
if type(item) == 'table' then return 'table:' .. item.nested end
return type(item) .. ':' .. tostring(item)
```
````

````text
string:b,number:2,boolean:true,table:x
````

`tasks.fanout` accepts a Lua table literal as its collection, and each member, including a nested table, becomes one arm's `item`. So `tasks.fanout('### Worker', {{7, 'x'}})` runs one arm, whose `item` is the table `{7, 'x'}` and whose `{{ item }}` placeholder renders as `[7,"x"]` (see [The item placeholder in worker section prose](#the-item-placeholder-in-worker-section-prose)). [The collection rule](#the-collection-rule) sums up which members and keys a collection can hold.

### Keyed members

A keyed collection stores its members under keys in the table's hash part, and each of those is a keyed member. A keyed collection runs one arm per key and value pair, and each arm's `item` is a pair table that you read as `item.key` and `item.value`:

````markdown
---
name: keyed-collection
description: Fans out over a keyed table.
runvil: 0
---

# Keyed collection

## Parent

```lua
local r = tasks.fanout('### Worker', {zeta = 1, alpha = 2, mid = 3})
return r[1].text .. ',' .. r[2].text .. ',' .. r[3].text
```

### Worker

```lua
return item.key .. '=' .. item.value .. '@' .. sys.index
```
````

````text
alpha=2@1,mid=3@2,zeta=1@3
````

`sys.index` is the arm's position in collection order (see [The `sys.index` field](#the-sysindex-field)). The table literal lists `zeta` first, yet the `alpha` arm lands in `r[1]` with `sys.index` 1, because keyed members run in sorted key order. The run returns `alpha=2@1,mid=3@2,zeta=1@3` on every run.

A worker section that returns `item.key .. '=' .. tostring(item.value)` over `{alpha = 1, beta = 'two'}` yields `alpha=1` and `beta=two`, and the result list has `#r == 2`.

### The order of members

Member order is fixed and never depends on Lua's string hash seed. The [array part](05-lua-environment.md#deterministic-table-iteration), positions 1 through `#t`, comes first in index order, and then every hash-part member follows as a `{ key, value }` pair, sorted by key. So `{'a', 'b', extra = 'c'}` fans out as `'a'`, `'b'`, and then `{ key = 'extra', value = 'c' }`, and the arm order is the same on every run. The number of arms equals the length of the array part plus the number of remaining keys in the table.

Only integer keys from 1 through `#t` count as array-part members. An integer key outside the array part, such as `{[5] = 'five'}`, is a keyed member, so its arm gets the pair table `{ key = 5, value = 'five' }` as `item`. A float key such as `2.5` is a hash-part member too and arrives as a pair table, so `[2.5] = 'half'` gives an `item` of `{ key = 2.5, value = 'half' }`.

An arm's position follows member order, with array-part members at positions 1 to `#t` and the sorted keyed members after them. That position is the arm's `sys.index` and its place in the result list, as the `keyed-collection` prompt shows.

`tasks.fanout` reads the collection's length and the members in its array part raw, so `__len` and `__index` metamethods do not change which members it sees.

### How keys sort

Keyed members sort by key type first and then by value within each type:

| Key type | Order within the type |
|---|---|
| boolean | `false` before `true` |
| number | exact numeric order |
| string | byte order |

Booleans come before numbers, and numbers come before strings. So `{[true] = 't', [7] = 'seven', b = 'bee', [false] = 'f', [2.5] = 'half', a = 'ay'}` orders its keys `false`, `true`, `2.5`, `7`, `'a'`, `'b'`.

String keys are sorted whatever order the table literal uses, so `{zeta = 1, alpha = 'two', mid = true, beta = 4, omega = 5}` runs arms for `alpha`, `beta`, `mid`, `omega`, and `zeta`, in that order, on every run.

Number keys sort exactly. Integers compare as integers, so large integer keys that round to the same float, such as `9007199254740992` and `9007199254740993`, stay distinct and in order. A float key sorts exactly among the integers, and a float beyond the 64-bit integer range sorts past every integer. Together these rules give the key order `-1e300`, `2`, `2.5`, `3`, `9007199254740992`, `9007199254740993`, `1e300`.

This is the same shared order in which the deterministic `pairs` and `next` visit a table that has no `__pairs` metamethod (see [Deterministic table iteration](05-lua-environment.md#deterministic-table-iteration)), so a keyed table that `tasks.fanout` accepts fans out in the order `pairs` visits its keys. A `__pairs` metamethod on a table's metatable still takes over what `pairs` visits.

### The collection rule

The collection must be a Lua table with at least one member, so an empty table raises an error at the call instead of returning an empty result list. Every member, whether it sits in the array part or under a key, must be data: a string, a number, a boolean, or a table whose contents convert to JSON, so a function, a thread, or any userdata other than `json.null` is never a member. `json.null` is data, so it is a member, and its arm reads `item == json.null` (see [Encoding and decoding JSON with `json`](05-lua-environment.md#encoding-and-decoding-json-with-json)). Every key outside the array part must be a string of valid UTF-8, a finite number, or a boolean. A collection that breaks this rule raises a collection error, and [Collection and argument errors](#collection-and-argument-errors) gives the message for each part of the rule.

## What an arm sees

In this section you learn what an arm starts with, which is `item`, `sys.index`, and a snapshot of `var`, and what stays out of its reach.

````markdown
---
name: arm-view
description: Shows the values each arm starts with.
runvil: 0
---

# Arm view

```lua
var.from_h1 = 'seeded'
```

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
assert(var.alpha == nil and var.beta == nil)
return r[1].text .. ',' .. r[2].text
```

### Worker

```lua
assert(var.from_h1 == 'seeded')
var[item] = true
local sibling = item == 'alpha' and 'beta' or 'alpha'
assert(var[sibling] == nil)
return item .. '-' .. sys.index
```
````

````text
alpha-1,beta-2
````

`tasks.fanout` starts one arm per member, first member first, and seeds each arm with its member as `item`, its position as `sys.index`, and a snapshot of the caller's `var`. So the first arm started is the one for the first member, with `sys.index` 1. Each arm here reads the `var.from_h1` value the H1 body set, writes its own entry into its copy of `var`, and never sees its sibling's entry, and once `tasks.fanout` returns, the caller's `var` holds neither entry.

### The `item` global

Inside a fanout arm, the global `item` holds the member the arm runs for. It is an ordinary Lua value, usable in expressions, strings, conditions, and tool arguments:

- `'ping about ' .. item` reaches the model as `ping about alpha` in the arm for `alpha`.
- `models.infer(item .. ':1')` sends `a:1`, `b:1`, and `c:1` from the arms for `a`, `b`, and `c`.
- `if item == 'a' then ... end` takes a different path for one member.
- `tools.call('web/search', { query = item })` passes the member to a tool as its search query.
- `require('## helpers').label(item)` passes it to a function from a library.
- Over `{'alpha', 'beta'}`, a worker section returning `item .. '-' .. sys.index` gives `alpha-1` and `beta-2`.

Each array-part member reaches its arm as `item` with the same value and keeps its type: a string stays a string, a number a number, a boolean a boolean, and a table a table. A worker section that returns `type(item) .. ':' .. tostring(item)` yields `string:b`, `number:2`, and `boolean:true`, as the `mixed-members` prompt in [What a member can be](#what-a-member-can-be) shows. A whole-number member arrives as an integer, so `tostring(item)` on the member `2` prints `2` rather than `2.0`, and the arm text reads `number:2`.

A keyed member reaches its arm as a pair table, with the key in `item.key` and the value in `item.value`, as [Keyed members](#keyed-members) shows. A nested table member arrives as a table whose fields the worker section reads directly, so the member `{nested = 'x'}` with `if type(item) == 'table' then return 'table:' .. item.nested end` gives `table:x`.

The `item` global in an arm is a data copy of the member, made when the arm starts, and not a shared reference to your table.

### The `sys.index` field

Inside a fanout arm, `sys.index` holds the arm's 1-based position in its own fanout's collection order, as a number. Each arm receives its position as its index, so the first member's arm reads 1 and the second reads 2. Over `{'x', 'y', 'z'}`, `return item .. sys.index` yields `x1`, `y2`, and `z3`, and ten arms returning `tostring(sys.index)` give `1` through `10`.

`sys.index` concatenates into strings and store paths, so `store.write('arm-' .. sys.index .. '.md', item)` gives each arm its own store file, and `item .. '-' .. sys.index` builds a per-arm label. An arm running `store.write('research/' .. sys.index .. '.md', 'part ' .. item)` over seven members `a` through `g` leaves `research/1.md` through `research/7.md` holding `part a` through `part g`, in collection order. [The store across arms](#the-store-across-arms) explains why each arm writes its own path.

### The item placeholder in worker section prose

In the worker section's prose, the `{{ item }}` placeholder (see [Inserting the fanout or task item](07-substitution.md#inserting-the-fanout-or-task-item)) substitutes the arm's member into the text the worker section reads as `prose`:

````markdown
---
name: replies
description: Asks the model about each member.
runvil: 0
models:
  writer: {}
---

# Replies

```lua
models.default('writer')
```

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
return r[1].text .. '\n\n' .. r[2].text
```

### Worker

Reply about {{ item }}.

```lua
return models.infer(prose)
```
````

In the arm for `alpha`, the prose `Reply about {{ item }}.` becomes `Reply about alpha.`, and `return models.infer(prose)` sends that text to the model. A member renders the way any placeholder value does (see [How values render](07-substitution.md#how-values-render)), so a table member `{7, 'x'}` turns `Item: {{ item }}.` into `Item: [7,"x"].`

`models.infer` works inside an arm and returns the reply as a string, whether the prompt is the arm's substituted `prose` or a string built in Lua. You can build an arm's prompt from `item` directly, so `local first = models.infer(item .. ':1')` sends `a:1` from the arm for `a`, and arms over `a`, `b`, and `c` send `a:1`, `b:1`, and `c:1`. Returning the reply makes it the arm's `.text`, so after an arm runs `return models.infer(prose)`, the caller reads the reply as `r[1].text` and can join all the replies with `table.concat(r, '\n\n')`, as [Collection order and joining results](#collection-order-and-joining-results) explains.

`{{ item }}` in the prose of a section that has no `item`, such as a walked section, fails when the prose is read, with the error `{{ item }} is nil (not inside a fanout arm)`. So a walked section whose prose is `Ask about {{ item }}.` fails when its block runs `return prose`.

### Where `item` and `sys.index` exist

The `item` global exists only in the first section an arm or a spawned task enters with an item. For an arm, that is its worker section. For a task, it is the target of a [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) given an `item` option, so after `tasks.spawn('## Child', { input = 'child args', item = { name = 'alpha' }, index = 7 })` the child reads `item.name` as `alpha`. After a `jump` from the worker section, the next section, such as `#### Child`, passes `assert(item == nil)`.

A section the normal walk reaches has no `item` global, so `item` there is nil and `assert(item == nil)` passes. The H1 pass, the main walk, and a chain started with `call` have neither `item` nor `sys.index`, even when the `call` is made from inside an arm, so a `### Sub` that an arm reaches with `call('### Sub')` passes `assert(item == nil)`.

`sys.index` follows the same rule. It exists only in the first section an arm or a spawned task enters with an index, which is the arm's worker section or the target of a `tasks.spawn` given an `index` option. There it holds that option's value as given, so the child spawned above with `index = 7` reads `sys.index` as 7 and logs `index=7`.

Reading `sys.index` in an ordinary walked section, outside any fanout arm, raises a Lua error that names the field, `unknown sys field 'index'`. So a walked `## Main` running `return tostring(sys.index)` fails with that message. A block can use this to tell a fanout arm from an ordinary section, by catching the error with [`pcall`](05-lua-environment.md#raising-and-catching-errors), checking that the caught error value's `kind` is `lua`, and raising any other error again with `error(result, 0)`:

````markdown
---
name: careful
description: Tells a fanout arm from an ordinary section.
runvil: 0
---

# Careful

## Main

```lua
local ok, result = pcall(function() return sys.index end)
if ok then return 'arm ' .. result end
if result.kind ~= 'lua' then error(result, 0) end
return 'not an arm: ' .. tostring(result)
```
````

````text
not an arm: runtime error: unknown sys field 'index'
````

`## Main` runs on the main walk, so outside an arm the caught text is `runtime error: unknown sys field 'index'`.

### A snapshot of `var`

Each arm starts from its own fresh copy of the caller's `var` as it stood when `tasks.fanout` was called, so it reads every entry the caller had, including values set in the H1 body. In `arm-view`, the H1 body sets `var.from_h1 = 'seeded'`, and every arm passes `assert(var.from_h1 == 'seeded')`.

The snapshot is copied out as plain data, the same way a `call` gets a copy of the caller's `var` (see [Call input and `var`](08-jump-and-call.md#call-input-and-var)), so an arm whose caller has an empty `var` starts with an empty `var`. One snapshot, taken when `tasks.fanout` is called, serves every arm, so all arms of one fanout start from the same `var`. The caller cannot change `var` in between, because it is paused inside the `tasks.fanout` call.

An arm can write to its own copy of `var` without error, so `var[item] = true` runs in each arm of `arm-view` and the arm still returns its text. The copy is deep, so an arm's writes never reach its sibling arms or the caller, and the arm's changes are discarded when the arm ends. Each arm passes `assert(var[sibling] == nil)`, the caller still passes `assert(var.alpha == nil and var.beta == nil)` once `tasks.fanout` returns, and `tasks.fanout` hands back only the result list. To get data out of an arm, return it, as [Returning, jumping, and nesting inside an arm](#returning-jumping-and-nesting-inside-an-arm) shows.

### `args`, `argv`, and call input in an arm

A fanout arm starts with no call input of its own (see [Call input and `var`](08-jump-and-call.md#call-input-and-var)). Every arm reads [`args`](06-arguments.md#the-argument-string-in-args) too, because it runs under its caller's `args`. Every arm also inherits its caller's frozen [`argv`](06-arguments.md#reading-parsed-input-from-argv), the same way a `call` without an argument string does.

### Models, tools, and libraries in an arm

An arm inherits its caller's context whole, so the run's models and tools work in it as in any section. A worker section can run `tools.call('web/fetch', { url = item })` over a collection of URLs, or `return models.infer(prose)`, and reach the tool or the model. A fanout arm gets the same section Lua environment setup as a walked section, including the [sandbox](05-lua-environment.md#the-sandbox), and only its seeded `item` and `sys.index` differ.

A [`models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault) call in the H1 body sets the prompt-wide default role, and that default holds in every arm. With `writer: {}` declared under `models:` in the frontmatter and `models.default('writer')` in the H1 body, as in the `replies` prompt, an arm's `return models.infer(prose)` reaches the model.

Worker section Lua can also call the model directly through a [model handle](10-models.md#model-handles) with `handle:infer(...)`. Each arm runs in its own section VM with its own empty [`messages`](11-conversations.md#messages-is-the-sections-conversation) list, which it fills and drives with [`models.loop`](11-conversations.md#starting-a-conversation), so the arms' conversations stay separate and each arm opens its own first round.

A fanout arm can call the run's bound tools with [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls), such as `tools.call('web/search', { query = item })` in a prompt that declares `uses: [web]`. Inside an arm, [`sys.model`](05-lua-environment.md#run-facts-in-sys-and-ui) reads the name part of the selected model's id, without the server namespace, once the arm's first `tools.call` has run, as in any section, so a worker section that runs `tools.call('web/search', { query = item })` and then `return sys.model .. ':' .. item` gives a text such as `claude-sonnet-4-6:a` when that model's id is `gateway/claude-sonnet-4-6`.

Inside an arm, and in a chain the arm starts with `call`, `sys.section_count` is the run's top-level section count, so a child worker section does not add to it. With `## Parent` as the only top-level section and the worker section under it, the arm and a `### Sub` it calls both read `sys.section_count == 1`.

A library loaded with [`require`](03-blocks-and-prose.md#sharing-code-with-require) inside a fanout arm runs in the arm's environment, so its top-level code reads the arm's `item`. [`call` and `require` inside an arm](#call-and-require-inside-an-arm) shows an example.

A fanout arm sees every [Pack prelude](12-tools.md#pack-preludes) global, so where `greeter` is a prelude global, an arm block `return greeter.greet(item)` yields `hello arm`. Each arm runs every prelude afresh in its own Lua environment, so a prelude's install-time side effects happen once per arm. With `tasks.fanout('### Worker', { 'alpha', 'beta' })`, a prelude whose install-time code writes a store file writes it in both arms, so the second arm's write is a [store conflict](#store-conflicts) that ends the run with `Determinism`, even when the prelude wraps the write in `pcall`.

## Returning, jumping, and nesting inside an arm

In this section you return a value from an arm, then use `jump`, `call`, and a nested fanout inside one.

````markdown
---
name: arm-return
description: Each arm returns from its prologue.
runvil: 0
---

# Arm return

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
assert(r[1].text == 'alpha-1' and r[2].text == 'beta-2')
return 'ok'
```

### Worker

```lua
return item .. '-' .. sys.index
```

Do work.
````

````text
ok
````

Each arm's Lua return value becomes that arm's entry in the fanout result. Here the worker section's prologue returns `alpha-1` in the first arm and `beta-2` in the second, and `## Parent` checks both before it returns `ok`.

### What an arm returns

A `return value` in the worker section's prologue ends the arm with that value as its result. The worker section's prose after the fence is never sent to a model, and such an arm needs no model round, so in `arm-return` the prose `Do work.` goes unused. A worker section that is pure Lua and returns right away runs without any model connection, so that kind of fanout needs no model and stays fast. The 1025-arm `many-members` prompt runs with no model configured and returns `m1025`.

A worker section's return hands its value back to `tasks.fanout` as the arm's result and never ends the run, because a scalar return ends the run only from the H1 pass or the main walk (see [Returning values](04-how-a-prompt-runs.md#returning-values)). The calling section, such as `## Parent`, goes on to check the results, and its own return, such as `'ok'`, becomes the run's text.

A section can call `tasks.fanout` from its epilog block, and the results still come back in collection order. The `fanout-epilog` prompt in [Where you can call `tasks.fanout`](#where-you-can-call-tasksfanout) returns `x-done,y-done` with both arms succeeding.

The arm's result is whatever the worker section returns, not whatever its model rounds return. A worker section that runs `models.infer(item)` and then `return item` gets a reply every time, yet over the members `'1'` through `'1025'`, `r[1].text` is `1` and `r[1025].text` is `1025`.

A fanout arm that runs off its last section without returning gives the empty string as its result, so `r[1].ok` holds and `r[1].text == ''`, the same as a task that runs off its last section.

### The worker section's visible set

Inside an arm, heading references resolve over the worker section's own visible set: the sections at its level under the same parent, the worker section itself excluded, plus its own children. A worker section can read its sibling list section with [`prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist), which returns the list's entries as a Lua list of strings:

````markdown
---
name: arm-reach
description: A worker section reads its sibling list section.
runvil: 0
---

# Arm reach

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha'})
return r[1].text
```

### Items

- x
- y

### Worker

```lua
local items = prompt.list('### Items')
return item .. ':' .. table.concat(items, ',')
```
````

````text
alpha:x,y
````

When the worker section is a child of the calling section, the calling section and its siblings are not found from inside the arm. `call` on a heading outside the worker section's visible set raises the not-found error where it is called, so `pcall(call, heading)` returns `false` and the arm goes on, while `prompt.list` on such a heading returns `nil`. Replace the worker section's block in `arm-reach` with this one:

````lua
local ok = pcall(call, '## Parent')
if not ok and prompt.list('## Parent') == nil then
  return item .. ':rejected'
end
return item .. ':reached'
````

The run now returns `alpha:rejected`, because `## Parent`, the section that fanned out, is outside the worker section's visible set: `pcall(call, '## Parent')` gives `false` with `not found` in its error, and `prompt.list('## Parent')` reads `nil`.

### `jump` inside an arm

`jump(heading)` inside an arm skips the rest of the arm's code and walks from the target through its following siblings, and that walk's result becomes the arm's `.text`:

````markdown
---
name: arm-jump
description: An arm jumps to a sibling of its worker section.
runvil: 0
---

# Arm jump

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha'})
return r[1].text
```

### Worker

```lua
jump('### Target')
error('never runs')
```

### Target

```lua
assert(item == nil)
assert(sys.id == '0.0.1')
store.append('order.txt', 'Target\n')
```

### Tail

```lua
store.append('order.txt', 'Tail\n')
return 'tail-reply'
```
````

````text
tail-reply
````

After `jump('### Target')`, the `error('never runs')` line never runs. `### Target` appends its line and falls through to `### Tail`, which returns `'tail-reply'`, so the arm's `.text` is `tail-reply`, and `order.txt` holds `Target` and `Tail` on two lines.

A `jump` target inside an arm runs as the arm chain's next section and takes its next `sys.id` (see [Section entry ids](05-lua-environment.md#section-entry-ids)), so in the first arm `### Target` sees `sys.id == '0.0.1'`. In a spawned chain, meaning a fanout arm or a task spawned with `opts.item`, `item` and `sys.index` are set only in the first section the chain enters. For an arm that is its worker section, where `sys.index` holds the arm's position, and later sections it reaches, for example by `jump`, have neither, which is why `### Target` passes `assert(item == nil)`. For a spawned task, `opts.item` becomes its `item` global and `opts.index` its `sys.index`.

A `jump` from an arm into one of the worker section's own children starts a [child walk](08-jump-and-call.md#child-walks). The target runs as a plain section with no `item`, falls through to its following child siblings, and takes the arm chain's next `sys.id`:

````markdown
---
name: arm-child-jump
description: An arm jumps into a child of its worker section.
runvil: 0
---

# Arm child jump

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha'})
return r[1].text
```

### Worker

```lua
jump('#### Child')
```

#### Child

```lua
assert(item == nil)
assert(sys.id == '0.0.1')
```

#### ChildTail

```lua
return 'child-tail-reply'
```
````

````text
child-tail-reply
````

The child walk runs `#### Child`, which takes the id `0.0.1` and has no `item`, and falls through to `#### ChildTail`, whose return becomes the arm's `.text`.

When the walk from an arm's `jump` target ends without a result, the arm reports `.text == ''`. If `### Target` only appends to `order.txt` and no section follows it, the arm's text is empty.

A `jump` from an arm to a heading outside the worker section's visible set fails the arm, and the fanout with it. The error's message contains `not found` and lists the worker section's visible sections, such as `### Items`, and left uncaught it ends the run with run error kind `Lua`. In `arm-reach`, a worker section running `jump('## Parent')` fails this way. A `pcall` inside the arm cannot catch this failure, because the target resolves after the block ends.

### `call` and `require` inside an arm

`call` inside an arm runs a called chain that starts at the target, falls through the target's following siblings as plain sections with no `item` global, and returns its result to the arm:

````markdown
---
name: arm-call
description: An arm calls a sibling of its worker section.
runvil: 0
---

# Arm call

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha'})
return r[1].text
```

### Worker

```lua
local got = call('### Sub')
return 'worker:' .. got .. ':' .. item
```

### Sub

```lua
assert(item == nil)
```

### Tail

```lua
return 'tail-reply'
```
````

````text
worker:tail-reply:alpha
````

`### Sub` returns nothing and falls through to `### Tail`, whose `'tail-reply'` comes back to the arm as the value of `call`. The arm's own `item` is still `alpha` after the call returns.

Code the worker section loads with `require` behaves differently from a called section. A library the worker section loads runs in the arm's environment, so its top-level code reads the arm's `item`. A worker section that runs `require '## capture'`, where the `## capture` library's top-level code sets `captured_by_library = item`, sees `captured_by_library` set to its own member, and in the arm for `alpha` the run text comes out as `alpha|alpha`.

### Nested fanouts

An arm's section can call `tasks.fanout` again over a collection it builds, and that inner call is a nested fanout:

````markdown
---
name: nested-fanout
description: Each outer arm fans out over two inner members.
runvil: 0
---

# Nested fanout

## Parent

```lua
local r = tasks.fanout('### Outer', {'a', 'b'})
return r[1].text .. ';' .. r[2].text
```

### Outer

```lua
local inner = tasks.fanout('### Inner', {item .. '1', item .. '2'})
return item .. ':' .. inner[1].text .. ',' .. inner[2].text
```

### Inner

```lua
assert(tostring(sys.index) == string.sub(item, -1))
return item .. '!'
```
````

````text
a:a1!,a2!;b:b1!,b2!
````

A nested fanout's worker section resolves over the outer worker section's visible set, so the inner worker section can be a sibling or a child of the outer one. Here `### Inner` is a sibling of `### Outer`. In each inner arm, `item` and `sys.index` refer to the inner collection, while the outer arm's `item` stays its own member, so the outer arm for `a` returns a text that starts with `a:`.

`sys.index` is an arm's 1-based position in its own fanout's collection, so a nested fanout restarts it at 1 instead of counting across outer arms. With the inner members `a1`, `a2`, `b1`, and `b2`, `assert(tostring(sys.index) == string.sub(item, -1))` holds in every inner arm, because `a1` and `b1` are each at position 1 of their own fanout.

In a nested fanout, results place by collection index at every level, regardless of which arm finishes first. A nested fanout also finishes when the concurrency limit is 1, set with [`tasks.concurrency`](#setting-the-limit-with-tasksconcurrency), because an arm, or a chain it started with `call`, gives its slot back while it waits on its own fanout's arms. This version runs three outer arms that each fan out over three members of their own, with the inner worker section as a child of the outer one:

````markdown
---
name: nested-one-wide
description: Runs a nested fanout under a concurrency limit of 1.
runvil: 0
---

# Nested one wide

## Parent

```lua
tasks.concurrency(1)
local r = tasks.fanout('### Arm', {'a', 'b', 'c'})
return r[1].text .. ',' .. r[2].text .. ',' .. r[3].text
```

### Arm

```lua
local inner = tasks.fanout('#### Sub', {'x', 'y', 'z'})
return item .. ':' .. inner[1].text .. '+' .. inner[2].text .. '+' .. inner[3].text
```

#### Sub

```lua
return item .. sys.index
```
````

````text
a:x1+y2+z3,b:x1+y2+z3,c:x1+y2+z3
````

The outer results come back in the order `a`, `b`, `c`, and each outer arm's inner results come back as `x1+y2+z3`.

### The depth cap for arms

A fanout arm runs one level deeper than the section that fanned out, so arms count toward the [recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap) of 8. A `call` inside an arm that is already at depth 8 fails with `call recursion exceeded cap of 8`. A `tasks.fanout` requested from a chain already at depth 8 fails the run with exactly `tasks.fanout recursion exceeded cap of 8`, naming `tasks.fanout` rather than `call`, with no prefix or traceback.

A `tasks.fanout` arm that would nest past the cap ends the run with run error kind `Quota` (see [Run error kinds](16-limits-and-errors.md#run-error-kinds)). The cap applies to each arm, the same as it does to a `call` or to a `tasks.spawn` task.

## Reading the results

In this section you read each fanout result's `.text`, `.item`, `.ok`, and `.exhausted`, and learn why the results are read-only.

````markdown
---
name: read-results
description: Reads every field of the fanout results.
runvil: 0
---

# Read results

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
assert(#r == 2)
assert(r[1].text == 'alpha-1' and r[1].ok == true)
assert(r[1].item == 'alpha' and r[1].exhausted == false)
assert(tostring(r[2]) == r[2].text)
return table.concat(r, ',')
```

### Worker

```lua
return item .. '-' .. sys.index
```
````

````text
alpha-1,beta-2
````

`tasks.fanout` returns a list of fanout results in collection order, so `r[1]` is the result for the first member, `r[2]` for the second, and so on. A fanout result is the object that reports one arm's outcome, and `table.concat` over the list joins their texts.

### The four result fields

Each fanout result has four fields, and `#r` counts the results, one per collection member, so a 1025-member collection gives `#r == 1025`:

| Field | What it holds |
|---|---|
| `.text` | the text the arm returned, always a string |
| `.item` | the member the arm ran on, with its own Lua type |
| `.ok` | `true` when the arm completed normally |
| `.exhausted` | `true` only when the arm's `models.loop` hit the round cap |

`.text` holds the value the arm's worker section returned, or, when the worker section ends with `return models.infer(...)`, the model's reply, which the caller reads as `r[1].text`. A result's `.text` is always a string: the text the arm returned, or the empty string `''` when the arm ended without returning anything. An arm whose worker section ends without returning anything still reports `.ok == true` with `.text == ''`, and not the run-level `done` fallback (see [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes)).

`.ok` is `true` when the arm completed normally and `false` when the arm's [`models.loop`](11-conversations.md#starting-a-conversation) hit the [round cap](11-conversations.md#tool-rounds-and-the-round-cap). `.exhausted` is the reverse of `.ok`: it is `true` only when the arm's `models.loop` hit the round cap, and `false` when the arm completed normally. [Exhausted arms and the exhausted stub](#exhausted-arms-and-the-exhausted-stub) shows what such a result holds.

A result has no methods, and reading any field other than `text`, `ok`, `item`, or `exhausted` gives `nil`.

### Collection order and joining results

The result list follows collection order, never finish order, so `r[i]` belongs to the i-th member even when a later arm finishes first, and a join or merge built from the results comes out the same on every run. A numeric `for i = 1, #r do ... end` loop visits every arm's result in collection order:

````lua
local r = tasks.fanout('### Worker', {'x', 'y', 'z'})
local parts = {}
for i = 1, #r do parts[i] = r[i].text end
return table.concat(parts, '+')
````

With the worker section from `read-results`, this returns `x-1+y-2+z-3`.

You can also join the results directly with `table.concat(r, sep)`, which gives all arm texts in collection order with `sep` between them, as `read-results` does. That works because `tostring(r[i])` gives the result's `.text`, and the sandbox's `table.concat` converts any value that has a `__tostring` through `tostring`.

To build a string with `..`, you use `r[i].text` or `tostring(r[i])`. A result defines no `__concat`, so applying `..` to the result itself raises a Lua error.

### `.item` and your own values

Each result's `.item` holds the collection member that arm ran on, keeping its own Lua type: a number stays a number, a string a string, and a table a table. So you can pair results with inputs without counting positions. Over `{1, 'two', {n = 3}}` with a worker section that returns `'done'`, `r[1].item == 1`, `r[2].item == 'two'`, and `r[3].item.n == 3` all hold.

A result's `.item` is your own member value, the very table the collection held rather than a copy, while the `item` the arm sees is a copy. For a keyed member, a result's `.item` is the `{ key, value }` pair table, with both `key` and `value` set, so in the `keyed-collection` prompt `r[1].item.key == 'alpha'`.

### Results are read-only

Fanout results are read-only: assigning any field, existing or new, raises an error. The list that holds them is an ordinary table:

````markdown
---
name: sealed-results
description: Shows that a fanout result refuses writes.
runvil: 0
---

# Sealed results

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha'})
local ok, err = pcall(function() r[1].text = 'forged' end)
assert(not ok)
assert(tostring(err):find('tasks.fanout results are read-only', 1, true))
local set_ok, set_err = pcall(setmetatable, r[1], nil)
assert(not set_ok)
assert(tostring(set_err):find('protected metatable', 1, true))
assert(r[1].text == 'alpha-1' and tostring(r[1]) == 'alpha-1')
r[#r + 1] = 'extra'
return 'sealed:' .. r[1].text .. ':' .. #r
```

### Worker

```lua
return item .. '-' .. sys.index
```
````

````text
sealed:alpha-1:2
````

Assigning `r[1].text = 'forged'` raises a Lua error reported at the assigning line, with a message containing `tasks.fanout results are read-only`, and `pcall` catches it. A result cannot be written through raw table access either, because the sandbox removes `rawset` from the globals before your code runs.

`getmetatable` on a result returns a decoy table that holds only a `__tostring` function and no `__index` or `__newindex`, so the result's hidden fields cannot be reached or changed through it. `setmetatable` on a result raises an error instead of replacing its protection: `setmetatable(r[1], nil)` fails with a message containing `protected metatable`, and `setmetatable(r[1], {})` fails too. A result keeps its original fields and its `tostring` text after refused writes and refused metatable changes, and rebinding base globals such as `setmetatable` in your own code does not change how `tasks.fanout` builds its results.

Iterating a result with `pairs` visits no fields, because the result is an empty table whose fields sit in a hidden `__index` table, so you read result fields by name.

The list `tasks.fanout` returns is an ordinary Lua table, so you can sort it, append to it, or replace its entries, while the result objects inside it stay read-only. In `sealed-results`, `r[#r + 1] = 'extra'` appends a string to the list, so `#r` becomes 2.

## Concurrency and the admission window

In this section you see how arms interleave, how many run at once, and how to change that number with `tasks.concurrency`.

````markdown
---
name: one-at-a-time
description: Runs two arms one at a time.
runvil: 0
models:
  writer: {}
---

# One at a time

```lua
models.default('writer')
```

## Parent

```lua
local limit = tasks.concurrency(1)
local r = tasks.fanout('### Worker', {'one', 'two'})
return limit .. ':' .. r[1].text .. '|' .. r[2].text
```

### Worker

```lua
local first = models.infer(item .. ':1')
local second = models.infer(item .. ':2')
return first .. ' ' .. second
```
````

This is the `fanout-rounds` prompt from [Fanout in one call](#fanout-in-one-call) with a `tasks.concurrency(1)` call added. That call sets the concurrency limit to 1 for the arms `## Parent` starts, and returns 1, which the block puts at the front of the run text. In `fanout-rounds`, the prompts go out as `one:1` and `two:1` before either second prompt. Here only one arm runs at a time, so the arm for `one` runs to its end before the arm for `two` is admitted, and the prompts go out as `one:1`, `one:2`, `two:1`, `two:2`. The run text still starts with `1:` and then the `one` arm's replies, because the results follow collection order either way.

### Interleaving

Fanout arms run concurrently by interleaving: they alternate at the points where they wait, such as a `models.infer` round or a store call, rather than running their Lua in parallel. Only one arm's Lua runs at any moment. An arm pauses while its request is in flight, and another arm runs until it reaches its own wait. When all the arms fit under the concurrency limit, every arm sends its first model request before any arm resumes.

Arms that each run [`models.loop`](11-conversations.md#starting-a-conversation) drive their loops concurrently in the same way. When all the arms fit under the concurrency limit, every arm's first model round is in flight before any arm's second round, instead of one loop running after another.

Model replies for arms admitted at the same time can land on either arm, so do not rely on the order of concurrent model requests. Arms also alternate at store calls, so the order of the arms' side effects, such as which arm's store write lands first, is not fixed. Only the order of the returned results is.

### The concurrency limit

The Harness sets one concurrency limit for the whole run: the most tasks admitted to run at once. Every task counts against it, including fanout arms, tasks started with [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), and the tasks those start in turn. The run's concurrency limit defaults to 8 tasks at once.

Because every task counts against the one limit, a fanout nested inside an arm shares the budget with that arm and every ancestor instead of multiplying it, and the whole run never exceeds the limit. A spawned task's concurrency limit starts out equal to its spawner's, so the tasks it starts run within the concurrency limit of their parent task and of every ancestor task. The main walk never waits for a slot.

### How arms are admitted

`tasks.fanout` spawns every arm up front, before it waits on any of them and before any arm is admitted to run, so no arm is ordered after a sibling arm. The arms start in collection order, beginning with the first member, and `tasks.fanout` admits them in collection order as slots free up.

Only as many arms run at once as the concurrency limit allows. The rest wait in a queue without sending any model request until a running arm ends, so with the limit at 1 each arm runs to its end before the next one is admitted, as in `one-at-a-time`. An arm waiting on a model round keeps its slot, so a queued arm is admitted only when a running one ends, and under a limit of 1 the next arm stays queued until the first one ends.

Waiting arms are admitted in spawn order as slots free up, so a queued arm is admitted as soon as any running arm finishes, even while an earlier, lower-index arm is still running. A task that was waiting on other tasks and wakes because one of them ended is admitted ahead of tasks that have not started yet. Together these rules make the admission window: at most the limit's number of arms run at once, and the rest move in, in collection order, each time one of the running arms ends.

A collection far larger than the concurrency limit still runs in full, with waiting arms admitted as slots free up:

````markdown
---
name: large-fanout
description: Fans out over far more members than the concurrency limit.
runvil: 0
---

# Large fanout

## Parent

```lua
local items = {}
for i = 1, 1025 do items[i] = tostring(i) end
local r = tasks.fanout('### Worker', items)
return #r .. ':' .. r[1].text .. ':' .. r[1025].text
```

### Worker

```lua
return item
```
````

````text
1025:1:1025
````

An admitted arm holds its own section VM from admission until it ends, so at most the limit's number of arms plus the parent hold a section VM at once, while queued arms hold none. That keeps a fanout over a large collection within memory bounds set by the concurrency limit.

A task started with `tasks.spawn` that is still waiting for a slot reads `state` `running`, `blocked` `queued`, and no `section` in [`tasks.status`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote), and it first runs only once it is admitted.

### Waiting gives the slot back

A task waiting on other tasks (see [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin)) gives its slot back while it waits and takes it back once the wait returns, so a fanout whose arms each fan out again cannot deadlock. An arm of `tasks.fanout` can itself call `tasks.fanout`, and the nested fanout completes even under a concurrency limit of 1, because an arm waiting on its inner fanout gives its slot to its inner arms, as `nested-one-wide` in [Nested fanouts](#nested-fanouts) shows.

The same holds for a `call` inside an arm. When the called section waits on its own fanout, it gives back the slot the arm holds while it waits, so that nested fanout also finishes under a concurrency limit of 1 instead of hanging.

### Setting the limit with `tasks.concurrency`

You call `tasks.concurrency(n)` to set the calling chain's concurrency limit, and the call returns the effective limit as a plain integer:

````markdown
---
name: set-limit
description: Sets and reads the concurrency limit.
runvil: 0
---

# Set limit

## Parent

```lua
local a = tasks.concurrency(16)
local b = tasks.concurrency(2)
local c = tasks.concurrency()
local d = tasks.concurrency(4)
return a .. '|' .. b .. '|' .. c .. '|' .. d
```
````

Under the default run limit of 8, this returns:

````text
8|2|2|4
````

The limit governs the tasks the chain spawns from then on, so calling `tasks.concurrency` before a fanout lowers the limit for that fanout's arms. You pass a positive whole number, such as `1` or `16`. A whole-number float such as `2.0` or `8.0` works too: `tasks.concurrency` treats it as the integer `2` or `8`, sets that limit, and returns it exactly as an integer argument would, so `local set = tasks.concurrency(2.0)` passes `assert(set == 2)`. `tasks.concurrency()` with no argument returns the chain's current effective limit without changing it, which is why `c` above reads 2.

A limit set with `tasks.concurrency` is clamped to the parent chain's current limit, or, in the main walk, to the run's concurrency limit that the Harness sets. `tasks.concurrency(n)` sets the limit to the smaller of `n` and the parent's limit, so asking for more than that is never an error: the call clamps silently and returns the clamped value. That is why `a` above reads 8, and with the Harness's limit at 4 the same prompt returns `4|2|2|4`. A prompt that requests a high limit therefore stays portable to Hosts with a tighter limit, and never runs wider than whatever runs it.

After lowering the limit, you can raise it again with a later `tasks.concurrency(n)` call, up to the parent's limit, as `d` does. Lowering the limit never stops tasks already running, and it only gates tasks admitted afterward.

Inside the H1 pass, the limit starts at the run's concurrency limit, and lowering it with `tasks.concurrency` in the H1 pass stays in effect for the section walk that follows. `tasks.concurrency(n)` called inside a spawned task sets how many of that task's own tasks may run at once, so a fanout it runs admits at most `n` arms at once.

### When `tasks.concurrency` refuses a limit

An argument that is not a positive whole number fails at the call site with an error value of kind `lua` whose message, `tasks.concurrency limit must be a positive whole number, got {type}`, names only the value's type, not the value. This matters when the limit comes from data, such as a number parsed from the argument string. Passing `nil` is the same as passing no argument and only reads the limit, so this block turns an argument string that is not a number into `0`, which is refused:

````lua
local ok, err = pcall(tasks.concurrency, tonumber(args) or 0)
if not ok then
  assert(err.kind == 'lua')
  assert(tostring(err):find('positive whole number', 1, true))
  tasks.concurrency(1)
end
````

`tasks.concurrency` accepts whole numbers below 2^64. A whole number at or above 2^64 fails at the call site with an error value of kind `lua` whose message, `concurrency limit must be a positive whole number in 64-bit range, got {limit}`, shows the value, unlike the type-only message for other refused arguments. So `1e20` reads `got 100000000000000000000`. In both cases the message contains `positive whole number`, and `pcall` catches the error at the call site, as the block above does.

## Arms as tasks, ids, and events

In this section you see that each arm is a task with its own arm id, and which events arms report.

````markdown
---
name: arm-ids
description: Shows the task id and section entry id of each arm.
runvil: 0
---

# Arm ids

## Parent

```lua
assert(sys.taskid == '0')
local r = tasks.fanout('### Worker', {'a', 'b', 'c'})
return r[1].text .. '|' .. r[2].text .. '|' .. r[3].text
```

### Worker

```lua
return sys.taskid .. '@' .. sys.id
```
````

````text
0.0@0.0.0|0.1@0.1.0|0.2@0.2.0
````

The main walk reads [`sys.taskid`](05-lua-environment.md#run-facts-in-sys-and-ui) as `0`. Each arm reads its own task id, `0.0`, `0.1`, and `0.2`, and its worker section reads `sys.id` as `0.0.0`, `0.1.0`, and `0.2.0`.

### Each arm is a task

Each fanout arm is an ordinary task (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)), because `tasks.fanout` is built on the same task machinery as `tasks.spawn` and starts one task per collection member. Each arm starts as its own task with the worker section as its target, through the same start request that `tasks.spawn` uses, rather than running as an inline function call inside the calling section.

Because each arm starts the way a `tasks.spawn` task starts, it gets `call`'s target resolution and depth cap, and the same refusal of a list section as the target, so `tasks.fanout('### Items', {'x'})` over a list section fails with ``section `{name}` is a list section, not a worker template``. A fanout whose arms would pass the depth cap fails with `tasks.fanout recursion exceeded cap of 8`, naming `tasks.fanout`, where a `tasks.spawn` task past the cap names `call` instead. The difference comes from `tasks.fanout` marking each arm's start as a fanout arm, while a plain `tasks.spawn` carries no such mark.

A task that an arm starts with `tasks.spawn` is a child of the arm's chain, so the first spawn from arm `0.0` is task `0.0.0`. In `pcall(tasks.fanout, '## Worker', {'spawner', 'boom'})`, the first arm runs `tasks.spawn('## Child')`, and that child is task `0.0.0`.

### Arm ids

Each arm runs as a new child chain of the calling chain and gets an arm id: a dotted id under the calling chain's id, one per collection position, fixed by that position rather than by the order the arms finish. A first fanout from the main walk over `a`, `b`, `c` gives its arms the arm ids `0.0`, `0.1`, and `0.2`, as in `arm-ids`, and the three arms start in collection order as those three tasks.

An arm's arm id is also its task id, the id its task events carry (see [Task events and task errors](15-tasks.md#task-events-and-task-errors)). Inside a fanout arm, `sys.taskid` reads that id, such as `0.0` for the first arm of a fanout from a fresh main walk, while the main walk reads `0`.

Each arm is a child chain of the calling chain, not of the calling section. The arms are numbered in collection order after any child chains the calling chain already started, such as called chains or earlier arms, because `call` children and spawned tasks share one child counter per chain:

````markdown
---
name: shared-counter
description: Numbers two arms and a call from one counter.
runvil: 0
---

# Shared counter

## Parent

```lua
local r = tasks.fanout('### Worker', {'a', 'b'})
local sub = call('### Sub')
return r[1].text .. '|' .. r[2].text .. '|' .. sub .. '|' .. sys.id
```

### Worker

```lua
return sys.id .. '=' .. item
```

### Sub

```lua
return sys.id
```
````

````text
0.0.0=a|0.1.0=b|0.2.0|0.1
````

The arms are the caller's children `0.0` and `0.1`, the called chain is child `0.2`, and the caller is still at `0.1`. Arm task ids and `sys.id` values are the same on every run, whatever order the arms finish in, because ids are given out when the arms start rather than when they complete. Two different orders of serving the arms' model requests produce the same ids.

### `sys.id` inside arms

Inside an arm, [`sys.id`](05-lua-environment.md#section-entry-ids) is the id of the section the arm is running, and the worker section is the first section of the arm's chain. So for a fanout from the main walk, the worker section of arm K, counting arms from 0, reads `sys.id` as `0.K.0`, and a worker section logging `sys.id .. ':' .. sys.index` writes `0.0.0:1` in the first arm and `0.1.0:2` in the second. The calling section keeps its own `sys.id` before and after `tasks.fanout`, so `## Parent` reads `0.1` both times.

Nested arms are children of the outer arm's chain, so inner arm J of outer arm K has task id `0.K.J`, and its worker section reads `sys.id` as `0.K.J.0`:

````markdown
---
name: tree
description: Shows section entry ids in a nested fanout.
runvil: 0
---

# Tree

## Parent

```lua
local r = tasks.fanout('### Outer', {'a', 'b'})
return r[1].text .. '|' .. r[2].text
```

### Outer

```lua
local inner = tasks.fanout('#### Inner', {'x', 'y'})
return sys.id .. '(' .. inner[1].text .. ',' .. inner[2].text .. ')'
```

#### Inner

```lua
return sys.id .. ':' .. item .. sys.index
```
````

````text
0.0.0(0.0.0.0:x1,0.0.1.0:y2)|0.1.0(0.1.0.0:x1,0.1.1.0:y2)
````

The inner tasks are `0.0.0`, `0.0.1`, `0.1.0`, and `0.1.1`, with `sys.id` values `0.0.0.0`, `0.0.1.0`, `0.1.0.0`, and `0.1.1.0`.

A fanout inside a `call` child nests its arm ids under the `call` child's chain. When `## Main` at `0.1` calls `### Sub`, which runs as the called chain `0.0` with `sys.id` `0.0.0`, the arms' worker sections run at `0.0.0.0` and `0.0.1.0`, and `## Main` reads `0.1` again afterward. Inside an arm, the first section of a chain that `call` starts in arm 0 reads `0.0.0.0`, so a `### Sub` called from the only arm's worker section passes `assert(sys.id == '0.0.0.0')`.

Arm `sys.id` values are given out in collection order, which is sorted key order for a keyed collection. `tasks.fanout('### Worker', { zeta = 1, alpha = 2, mid = 3 })` with a worker section returning `item.key .. '=' .. item.value .. '@' .. sys.index .. ':' .. sys.id` joins to `alpha=2@1:0.0.0,mid=3@2:0.1.0,zeta=1@3:0.2.0`, with ids, `sys.index`, and places in the result list all in key order.

This table gathers the `sys.id` values above for a fanout from a top-level section with no earlier child chain:

| Section | `sys.id` |
|---|---|
| the calling section, before and after the fanout | unchanged, such as `0.1` |
| the worker section in arm K | `0.K.0` |
| a `jump` target in arm 0 | `0.0.1` |
| the first section of a chain that `call` starts in arm 0 | `0.0.0.0` |
| the worker section in inner arm J of a nested fanout inside arm K | `0.K.J.0` |

### Task events for arms

Each fanout arm runs as a task, so the run's event stream shows a `Task started` event for each arm when it begins running, reported under the calling section, and the arm's final event, such as `Task succeeded`, reported under the worker section (see [Task events and task errors](15-tasks.md#task-events-and-task-errors)). When a `## Research` section fans out two arms to `### Worker`, the stream holds two `Task started` events under `Research` and two `Task succeeded` events under `Worker`.

- A successful arm emits exactly one `Task started` event and one `Task succeeded` event, and no failed, cancelled, or abandoned event.
- An arm's `Task started` event reports under the caller section, the section whose Lua called `tasks.fanout`, so two arms fanned out from `Research` both start under `Research`.
- Every arm of one fanout runs the same worker section, and each arm's final task event reports under that worker section's name rather than the caller's, so two arms both succeed under `Worker`.
- An arm's `Task started` event carries the arm's task id, the worker section as its target, the arm's `item`, and its `sys.index` as `index`, so the stream shows which arm each start belongs to. The arm's final event carries the same task id.
- An arm waiting for a slot under the concurrency limit reports `Task started` only when it is admitted, so an arm cancelled while still queued reports no start and no final event. A fanout with a concurrency limit of 2 over `{'boom', 'slow', 'queued'}`, where the `boom` arm fails, ends with only `0.0` failed and `0.1` cancelled, and the queued arm never starts.
- An exhausted arm still reports a failed task event while its successful sibling reports a `Task succeeded` event, because the exhausted stub is the fanout's recovery, not the arm's (see [Exhausted arms and the exhausted stub](#exhausted-arms-and-the-exhausted-stub)).

In a fanout arm, each model round produces debug captures for the run, tagged with the worker section's name, so an arm's captures under `Worker` include a `Request` and a `Response`.

## The store across arms

In this section you see a store conflict between two arms, then the safe store pattern that avoids it.

````markdown
---
name: shared-path
description: Two arms write the same store path.
runvil: 0
---

# Shared path

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
return r[1].text .. ',' .. r[2].text
```

### Worker

```lua
store.write('shared.txt', item)
return item
```
````

Both arms write `shared.txt`, so the run never returns `alpha,beta`. It ends with run error kind [`Determinism`](09-the-store.md#races-claims-and-determinism), and the error message begins `store determinism violation:`, then names the contested path `shared.txt`, the words `conflicts with`, and both arms.

### One store for every arm

The arms of a fanout and the section that called `tasks.fanout` all use the run's one store (see [Writing and reading files](09-the-store.md#writing-and-reading-files)). The store is the exception to arm isolation, and whatever each arm writes stays in the store after the run, so when two arms write `arm-1.md` and `arm-2.md`, those files read `alpha` and `beta` afterward.

An arm can read the files the caller wrote before it called `tasks.fanout`, because starting an arm orders everything the caller did in the store before that point ahead of the arm's first step. It is the same rule that lets a `tasks.spawn` task build on the files its owner wrote. Any number of arms can read such a file, because two reads never conflict.

`tasks.fanout` joins every arm before it returns, so afterward the caller can read, glob, and merge every arm's store writes freely, including a file each arm wrote to its own path. While the arms run, though, they never see each other's store writes, whatever order they are admitted in, because `tasks.fanout` starts every arm before it waits on or collects any of them, so no arm is ordered after a sibling.

### Store conflicts

Two accesses make a store conflict when they touch the same store path, at least one of them writes, they come from different arms, and nothing orders one arm before the other. So two unordered arms that touch one path with a write never silently race, while any number of arms may read the same path. Two arms writing the same store path, as in `shared-path`, end the whole run with run error kind `Determinism`.

A cross-arm conflict is unconditional and never depends on timing. Sibling arms never join each other, and a claim on a path stands for the rest of the run, even after the arm that made it ends (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)). So whichever of two conflicting accesses runs second always meets the first one's claim, in every interleaving.

Of two conflicting arm accesses, the second never reaches the store, so exactly one arm's change lands and can be read from the store after the failed run. Which arm wins is not fixed: when two arms each add their member and a newline to `notes.md`, the file ends as `alpha` or as `beta`, followed by a newline, never both.

A store conflict ends the run at once. The losing arm never resumes, no arm counts as failed, an arm still waiting when the conflict lands reports cancelled, and at most the winning arm reports success. So two conflicting arms both report `Task started`, neither reports a failed task event, and at most one reports `Task succeeded`.

A cross-arm store conflict cannot be caught with `pcall`, not even a `pcall` wrapped around the store call inside the arm, because the conflict never resumes into the arm. When the arms run `pcall(store.append, 'notes.md', item .. '\n')`, the run still ends with `Determinism` instead of returning `alpha,beta`.

### Which store calls can conflict

Every store call can take part in a conflict, not just writes:

| Store call | What it claims |
|---|---|
| `store.write`, `store.append`, `store.str_replace`, `store.delete` | its path, for writing |
| `store.read`, `store.read_numbered`, `store.exists` | its path, for reading |
| `store.glob` | its whole pattern |

A sibling's write to any path a `store.glob` pattern matches conflicts with the glob. Any of these calls that touches a path a sibling arm writes ends the run with `Determinism`, whichever arm runs first.

`store.append` claims its path for writing just like `store.write`, so two arms appending to one file conflict even though neither overwrites the other. When both arms run `store.append('log.txt', item .. ';')`, the run ends with `Determinism` instead of returning the joined results, and the error names `log.txt`.

An arm whose `store.read`, `store.exists`, or `store.glob` touches a path a sibling arm is writing ends the run with the same `Determinism` error in every interleaving. So arms coordinate only through the results `tasks.fanout` returns, never through store files a sibling is writing, and they cannot meet by polling each other's marker files while the sibling is still running.

### The safe store pattern

The safe store pattern for a fanout is that each arm writes only its own path, built from `sys.index`, so no two arms ever claim one path, and the caller merges the per-arm files by index after `tasks.fanout` returns:

````markdown
---
name: safe-store
description: Each arm writes its own file and the caller merges them.
runvil: 0
---

# Safe store

## Parent

```lua
local r = tasks.fanout('### Worker', {'alpha', 'beta'})
local files = store.glob('arm-*.md')
local parts = {}
for i = 1, #r do
  parts[i] = store.read('arm-' .. i .. '.md')
end
store.write('merged.md', table.concat(parts, ','))
return #files .. ':' .. table.concat(parts, ',')
```

### Worker

```lua
store.write('arm-' .. sys.index .. '.md', item)
return item
```
````

````text
2:alpha,beta
````

The arms write `arm-1.md` and `arm-2.md`, the caller's `store.glob('arm-*.md')` after the call finds both, and afterward `merged.md` holds `alpha,beta`. Because the index is also the order of the results, the merge is the same on every run and in every interleaving. The pattern works for appends too: `store.append('ids-' .. sys.index .. '.txt', ...)` builds separate `ids-1.txt` and `ids-2.txt` files with no conflict.

After `tasks.fanout` returns, the caller can also write one store file straight from the returned results, and that file is the same on every run because the results arrive in collection order. With the worker section above, `store.write('merged.md', table.concat(r, ','))` leaves `merged.md` reading `alpha,beta`.

One arm may write the same store path more than once, because a repeat write by the same arm is a rewrite, not a race, and the last write wins. When a single arm over `{'only'}` runs `store.write('own.txt', 'first')` and then `store.write('own.txt', 'second')` and returns `item`, the run returns `only`, and `own.txt` reads `second`.

One block may call `tasks.fanout` more than once, and a later fanout's arms may read, append to, or overwrite paths an earlier fanout's arms wrote with no conflict. The earlier fanout joins every arm before it returns, and so before the later fanout starts any. When two fanouts in a row run `store.write('seq.txt', item)` over `{'one'}` and then `{'two'}`, and the block returns the second fanout's text, the run returns `two`, and `seq.txt` reads `two`.

## When an arm fails

In this section you see what the caller gets when an arm fails, how to catch the failure, and the one kind of arm failure that does not fail the fanout.

````markdown
---
name: caught-arm
description: Catches a failing arm and carries on.
runvil: 0
---

# Caught arm

## Parent

```lua
local ok, err = pcall(tasks.fanout, '### Worker', {'boom', 'fine'})
assert(not ok)
assert(tostring(err):find('fatal arm error', 1, true))
return 'caught:' .. err.kind
```

### Worker

```lua
if item == 'boom' then error('fatal arm error') end
return item
```
````

````text
caught:lua
````

The arm for `boom` raises `fatal arm error`. The fanout fails with that arm's own error value, `pcall` hands it to `## Parent` as `err`, and `## Parent` goes on to return `caught:lua`.

### Fail-fast

`tasks.fanout` is fail-fast: when any arm fails with an error other than hitting the round cap, the whole fanout fails. Every sibling arm still running is cancelled without waiting on its pending model rounds, members whose arms have not started never run, and the failure is raised from the `tasks.fanout` call.

The error `tasks.fanout` raises is the failing arm's own error value, with its original kind and message. It is not wrapped and it is not a cancellation error, so an arm that called `error('message')` produces an error of kind `lua`. Any other arm error fails the fanout the same way, for example a failed model round, or a `models.get` call inside the arm that names a role label with no bound model role, such as `models.get('ghost'):infer('ping')`.

A fanout failure is raised as an ordinary Lua error at the `tasks.fanout` call, inside the section that made the call. If nothing catches it, that section's Lua block fails, and the whole run ends with the arm's error. An arm that called `error('arm deliberately failed')` ends the run with run error kind `Lua` and a run error message that keeps the arm's original message text, and any other uncaught arm error ends the run with that error's message.

When a fanout raises, you get no partial results, not even from arms that had already finished. When several arms have already ended by the time the fanout checks on them, the arm earliest in collection order is handled first, so when failures happen at the same time, the earliest arm's failure is the one raised.

### Catching a fanout failure

You catch a fanout failure with `pcall(tasks.fanout, heading, collection)`, which returns `false` and the error value, as `caught-arm` does. The block then keeps going, further `models.infer` calls included, and the run can complete normally. The caught error converts with `tostring` to text that carries the failing arm's error message.

A late model reply meant for a cancelled sibling arm is thrown away instead of failing the run, so a block that caught the fanout failure carries on. Here the `boom` arm fails while its `slow` sibling may be waiting on a model round:

````markdown
---
name: carry-on
description: Catches a fanout failure and runs another model round.
runvil: 0
models:
  writer: {}
---

# Carry on

```lua
models.default('writer')
```

## Parent

```lua
local ok, err = pcall(tasks.fanout, '### Worker', {'boom', 'slow'})
assert(not ok)
local a = models.infer('after')
return 'caught:' .. a
```

### Worker

```lua
if item == 'boom' then error('fatal arm error') end
return models.infer('slow:1')
```
````

The `slow` arm is cancelled, any reply to its round is discarded, and `## Parent` runs its own model round and returns `caught:` followed by the reply to `after`. A fanout failure never leaves arms running behind it, so it does not trip a `tasks_live` error when the calling section's chain ends (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)).

### What happens to the other arms

When an arm raises an error, its queued siblings that the concurrency limit has not yet let in are cancelled before they are admitted: they never start, never run their block, and never send a model request. When an arm fails while a sibling is still waiting on a model round, the fanout cancels that sibling before it resumes and before the error reaches the caller, and the run never waits for the cancelled sibling's model reply.

When an arm is cancelled because a sibling failed, everything under it stops too, all the way down: a chain it started with `call`, the tasks it spawned, and the arms of a nested fanout. Everything it started is abandoned with the reason `owner_aborted` (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)), and anything that had not yet run its block never runs it.

The events follow the same pattern (see [Task events and task errors](15-tasks.md#task-events-and-task-errors)). The failing arm reports one `Task failed` event, each running sibling the fanout cancels reports exactly one `Task cancelled` event under its arm id, and a queued arm that never started reports no task event. The cancelled sibling's `Task cancelled` event fires before the calling section's Lua block fails with the fanout error.

Store writes an arm made before it failed stay in the store and are not rolled back. An arm that appends `boom` and a newline to `log.txt` and then raises leaves `log.txt` holding that line after the run fails.

### Exhausted arms and the exhausted stub

An arm that fails with error kind [`tool_loop_exhausted`](11-conversations.md#tool-rounds-and-the-round-cap), because its `models.loop` hit the round cap, does not fail the fanout and cancels no sibling arm. The other arms keep running and their results land beside it, and that arm's place in the result list holds an incomplete stand-in result with `ok == false`, `exhausted == true`, and its original `item`. Over `{'loop', 'plain'}`, where only the `loop` arm hits the round cap, you can check each slot like this:

````lua
local r = tasks.fanout('### Worker', {'loop', 'plain'})
assert(r[1].ok == false and r[1].exhausted == true)
assert(r[1].item == 'loop')
assert(r[2].ok == true and r[2].exhausted == false)
````

In a list that `tasks.fanout` returns, every result whose `ok` is `false` has `exhausted` equal to `true`, because every other kind of arm failure raises instead of returning.

An exhausted arm's `.text` is a fixed stub, the exhausted stub: a level-2 heading holding the member rendered exactly as `{{ item }}` renders that member, then `UNKNOWN`, then the note `(section incomplete: tool loop exhausted)`, with a blank line between each part. For the member `loop`, `r[1].text` is:

````text
## loop

UNKNOWN

(section incomplete: tool loop exhausted)
````

The stub renders a string member such as `alpha` as `alpha`, and a pair-table member as compact JSON, such as `{"key":"alpha","value":1}`.

An arm whose `models.loop` hits the round cap still reports a `Task failed` event while its sibling reports `Task succeeded`, because the stub is the fanout's recovery, not the arm's.

### When the run is cancelled

When the run is already cancelled as a fanout starts, the fanout runs none of its arms, and the run ends with a cancelled outcome (see [Cancellation and stops](16-limits-and-errors.md#cancellation-and-stops)).

If the Host cancels the run while arms are waiting on model replies, the run ends at once with the cancelled outcome, not a failure. The waiting arms' pending model requests are dropped without waiting for their replies, and no arm outlives the run. Each started arm reports one `Task abandoned` event naming the run's end, and no `Task cancelled` or `Task failed` event.

## Collection and argument errors

In this section you meet each error `tasks.fanout` raises for a broken collection or argument, with a call that causes it.

````markdown
---
name: empty-collection
description: Catches the error for an empty collection.
runvil: 0
---

# Empty collection

## Parent

```lua
local ok, err = pcall(tasks.fanout, '### Worker', {})
assert(not ok and err.kind == 'lua')
assert(not store.exists('ran.txt'))
return tostring(err)
```

### Worker

```lua
store.write('ran.txt', 'yes')
return item
```
````

````text
tasks.fanout over an empty collection: no work is likely a bug
````

The empty collection raises a collection error, the ordinary error value of kind `lua` that `tasks.fanout` raises when its collection breaks a rule, and its `message` is that rule's message. The worker section never runs, so `ran.txt` is never written, and `## Parent` keeps running after the caught error.

### When the checks run

`tasks.fanout` checks its collection at the call. It checks every rule except whether values nested inside a member can be represented as JSON before it spawns any arm, so a broken collection never runs the worker section.

The checks run in a fixed order, and the first broken rule is the one reported: first the check that the collection is a table, then each array-part member in position order, then each keyed pair with its key checked before its member, then the empty check. Only then does `tasks.fanout` spawn arms.

The empty-collection error fires at the call site before any arm is spawned. The worker section never runs, its store writes never land, no arm start is reported, and the call never hands back a zero-result list. An empty collection means no work, which is likely a bug, such as a list section that parsed empty or a wrong variable.

### Each collection error

Every broken collection rule raises an error value of kind `lua` whose `message` is that rule's message. This table gives each rule, a call that breaks it, and the message it raises:

| Rule | A call that breaks it | Message |
|---|---|---|
| The collection is a table | `tasks.fanout('### Worker', '### Items')` | `tasks.fanout's second parameter is a collection; for a list section use prompt.list(heading), which returns nil when no visible section has that heading` |
| Every member is data | `tasks.fanout('### Worker', {'a', function() end})` | `fanout collection member at index 2 is a function; members must be data` |
| Every key is a string, number, or boolean | `tasks.fanout('### Worker', t)` after `t[{}] = 'x'` | `fanout collection key must be a string, number, or boolean, got table` |
| Every number key is finite | `tasks.fanout('### Worker', {[1/0] = 'x'})` | `fanout collection key is not a finite number` |
| Every string key is valid UTF-8 | `tasks.fanout('### Worker', {['\255'] = 'x'})` | the Lua runtime's own string conversion message |
| The collection has at least one member | `tasks.fanout('### Worker', {})` | `tasks.fanout over an empty collection: no work is likely a bug` |
| Nested values can be represented as JSON | `tasks.fanout('### Worker', {{ f = function() end }})` | `item must be a JSON-representable value: {error}` |

The second argument to `tasks.fanout` must be a Lua table, and any other value, such as the heading string `'### Items'`, the number `5`, or `true`, raises that message. To fan out over a list section, pass `prompt.list(heading)` as the collection, as in `local replies = tasks.fanout('### Worker', prompt.list('### Topics'))`. Passing the heading string itself fails with the message that points to `prompt.list(heading)`, and so does passing the `nil` that `prompt.list` returns when no visible section has the heading.

Members must be data, so a member that is a function, userdata, or thread raises `fanout collection member at index {index} is a {type}; members must be data`. For an array-part member, `{index}` is its position, as in the table above. For a keyed member, `{index}` is the key as plain text: a string key's own text, `true` or `false`, an integer's decimal digits, or a float in its usual form such as `2.5`. So a function stored under the key `cb`, as in `{cb = function() end}`, raises `fanout collection member at index cb is a function; members must be data`. A `json.null` member passes this check, so `tasks.fanout('### Worker', { json.null })` runs one arm, and in that arm `item == json.null` is true and `item == nil` is false.

Every collection key must be a string, number, or boolean, and a key of any other type raises `fanout collection key must be a string, number, or boolean, got {type_name}`, naming the key's Lua type. Every number key must be finite, since a key that is not finite has no ordered position. Every string key must be valid UTF-8, and a key whose bytes are not valid UTF-8 raises a `lua` error that carries the Lua runtime's own string conversion message, with no fanout-specific wording.

The call-time member check looks only at each member's own type and not inside it. A table member that holds a function, such as `{ f = function() end }`, passes the call-time check but fails when `tasks.fanout` spawns its arm, with `item must be a JSON-representable value: {error}`. The call then cancels the arms it already spawned and raises that `lua` error.

### Catching a collection error

Wrapping `tasks.fanout` in `pcall` catches a collection error, as `empty-collection` does. `local ok, err = pcall(tasks.fanout, '### Worker', {})` gives `ok` as `false`, `err.kind` as `lua`, and `tostring(err)` as the rule's message, and the section keeps running after the caught error, for example on to `return 'ok'`.

Left uncaught, a collection error fails the run with run error kind [`Lua`](16-limits-and-errors.md#run-error-kinds) and the same message. When the call runs in the H1 pass, the run fails with run error kind `RequirementsUnmet` instead, with the H1 gate notice and the collection error as its cause (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)).
