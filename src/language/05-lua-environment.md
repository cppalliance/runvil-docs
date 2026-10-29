# The Lua environment

Every Lua block in a prompt runs in a sandboxed section VM that already holds the Engine globals, so the Harness's work, the run's facts, and the operator are each one plain function call away. This chapter covers that environment: what a block starts with, what the sandbox keeps, how tables iterate, the run state in `var`, `sys`, and `ui`, logging, asking the operator, and how errors are raised, caught, and located. Knowing it tells you exactly which Lua a block can run and what each call does to your run.

## Lua blocks, section VMs, and Engine globals

After this section you can write Lua blocks that use the Engine globals, and you know which state lasts from one block or section entry to the next.

````markdown
---
name: shout
description: Shouts the argument string.
runvil: 0
---

# Shout

```lua
return string.upper(args)
```
````

Run with the argument string `hi`, the run result is:

````text
HI
````

The block calls `string.upper`, a function from Lua's standard `string` library, on `args`, a global the Engine installs. Neither one needs an import. Every `lua` fence in a prompt runs real PUC Lua 5.5, with the Harness's work, the run's metadata, and the operator one plain function call away.

### Real Lua 5.5 in every block

You write every Lua region of a prompt in Lua 5.5 syntax. The standard functions and operators work inside a block as in any Lua 5.5 program, including `assert`, `error`, `tostring`, `type`, `string.upper`, `string.find`, `table.concat`, the length operator `#`, and the concatenation operator `..`. Ordinary Lua syntax works too: `local` variables, table constructors, list indexing, numeric `for` loops, `while` loops, arithmetic, and line comments. This prompt uses each of them:

````markdown
---
name: counting
description: Counts with ordinary Lua.
runvil: 0
---

# Counting

```lua
-- add up 1 to 4 with a numeric for loop
local n = 0
for i = 1, 4 do
  n = n + i
end

-- grow a table by integer index with a while loop
local t = {}
local i = 1
while i <= 3 do
  t[i] = "item " .. i
  i = i + 1
end

return 'sum ' .. n .. ': ' .. table.concat(t, ", ")
```
````

The run result is:

````text
sum 10: item 1, item 2, item 3
````

The block declares its block-local variables with `local`, adds up a number in a numeric `for` loop, and grows the table `t` by integer-index assignment inside a `while` loop, with `t[i] = ...` and `i = i + 1`. The line comments sit between statements, the way they can sit between any two calls in a block, such as a [`store.glob`](09-the-store.md#finding-files-with-storeglob) and a [`store.write`](09-the-store.md#writing-and-reading-files). String literals take double quotes or single quotes, so `return "hello"` and `return 'early'` both return their text.

The same syntax does real work. `local last = messages[#messages]` declares a block-local variable holding the last record of the section's [message list](11-conversations.md#building-a-message-list). A table constructor such as `{ a, b }`, list indexing such as `results[1]`, and `..` concatenation combine in a block that waits on two tasks with [`tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin):

````lua
local results = tasks.join({ a, b })
return results[1].result .. '|' .. results[2].result
````

Statements in a block run top to bottom. In a block that runs a `store.write`, then switches model role with [`models.use`](10-models.md#selecting-a-role-per-section-entry-with-modelsuse), then calls [`models.infer`](10-models.md#running-a-round-with-modelsinfer), the three happen in that order, and a `store.read` after the write sees what was written (see [Writing and reading files](09-the-store.md#writing-and-reading-files)).

A block can also loop without end, as in `local n = 0` followed by `while true do n = n + 1 end`. Such a block still compiles and loads normally, because nothing rejects an unbounded loop before it runs.

The parser compiles each Lua region when the prompt file is parsed, and compiling never runs the code. Each Lua block becomes bytecode at parse time, so a parsed prompt already holds loadable code for every block.

### A fresh section VM for every section entry

Each section entry's Lua runs in a section VM, the fresh Lua instance the entry's blocks share (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)). This prompt shows what lasts inside one section VM and what ends with it:

````markdown
---
name: lifetimes
description: Shows which globals last between blocks and sections.
runvil: 0
---

# Lifetimes

## First

```lua
note = 'set in the first block'
function decorate(value) return '<' .. value .. '>' end
saved_log = log
```

This prose sits between the two blocks of the section.

```lua
assert(note == 'set in the first block')
assert(decorate('input') == '<input>')
saved_log('still here')
```

## Second

```lua
assert(note == nil, 'note ended with the first section VM')
assert(decorate == nil)
return 'fresh VM'
```
````

The run result is:

````text
fresh VM
````

Inside one section entry, every `lua` block runs in the same Lua environment, so a global set in an earlier block is readable in every later block of that entry. Here `note`, set in the first block of `## First`, is still there in the second block, even with a prose block between the two. A function defined in one block, such as `decorate`, can be called from any later block in that entry. The same holds for fields set on `var` (see [Keeping state in `var`](#keeping-state-in-var)), such as `var.decorated`, and for saved references to Engine globals.

That one Lua environment serves every block in the section entry, and the Engine globals in it stay live for the entry's whole lifecycle, so a later block sees the same `log` and `store` as an earlier one. A first block can run `saved_log = log`, and a later block of the same entry can call `saved_log('still here')`.

Every section entry gets a brand-new section VM, created for that section by name, and the H1 pass gets one too (see [The H1 pass](04-how-a-prompt-runs.md#the-h1-pass)). That holds whether the walk reaches the section by fall-through, by [`jump`](08-jump-and-call.md#jumping-to-a-section), or by [`call`](08-jump-and-call.md#calling-a-section). The section VM is torn down when the section entry ends.

So a plain global assigned in one section entry reads as nil when the next entry runs, because globals live in that entry's own Lua VM. After `note` is set in `## First`, `assert(note == nil)` passes in `## Second`, and the same goes for any plain global, such as `reply` after `reply = 'custom'`. Each section entry gets its own fresh Lua environment, so globals created or changed in one entry never appear in another. When a `counter` global in one section VM has counted up to `2`, a `counter` global in a second section VM starts over at `1`.

Functions follow the same rule. A function such as `greet` that one section defines through [`require`](03-blocks-and-prose.md#sharing-code-with-require) reads as nil in a later section, where `tostring(greet)` is `nil`, because a library's globals stay in each VM that ran it.

Each fresh section VM is also restricted. In every one, `io`, `os`, `load`, and `package` read nil, as [The sandbox](#the-sandbox) explains.

### Globals are read when the code runs

A block reads its globals each time it runs. The same block `return greeting .. ' world'` returns `hello world` when `greeting` is `hello` and `goodbye world` when `greeting` is `goodbye`.

A function body works the same way. It looks up globals such as `args`, `var`, `store`, and `tools` at the moment it is called, not when it is defined, so calling it from a later block reads and changes the section's current state. In a section of a prompt that declares the `web` Pack (see [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key)), a first block can define this function:

````lua
function scope_and_store(id)
  tools.offer(id)
  var.scoped = id
  return id
end
````

A later block of the same section then runs `return scope_and_store('web/search')`. The call offers the tool with [`tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer), sets `var.scoped`, and returns `web/search`.

### The Engine globals

An Engine global is a global the runtime installs in every section VM before any block runs, so it is always present and you use it with nothing to import. Top-level code in the first block of a section can already read the Engine globals, so a first block can start like this:

````lua
first_saw_args = args
first_saw_store = store.read('seed.txt')
````

Every section VM, in the H1 pass and in walked sections alike, starts with these Engine globals:

- `args`, `argv`, `sys`, `var`, and `prose`
- `log`, `store`, `untrusted`, and `json`
- `models`, `tools`, `messages`, and `compactors`
- `call`, `jump`, `prompt`, and `require`
- `tasks`

Two more Engine globals are installed only in some section VMs. `ui` is installed when the run has an application-state snapshot (see [Run facts in `sys` and `ui`](#run-facts-in-sys-and-ui)), and `item` is installed in a spawned chain, such as a fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)). That makes 20 Engine globals in all.

Together they give every section's Lua logging through `log` (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)), tool scoping through `tools`, store operations through `store`, and the `var` and `argv` globals, all inside the sandboxed section VM. `prose` is the global that reads pending prose (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), `prompt` holds the section reads (see [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist)), and `models` holds [`models.infer`](10-models.md#running-a-round-with-modelsinfer). The rest are taught where they are used: `args` in [The argument string in `args`](06-arguments.md#the-argument-string-in-args), `argv` in [Reading parsed input from `argv`](06-arguments.md#reading-parsed-input-from-argv), `store` in [What the store is](09-the-store.md#what-the-store-is), `untrusted` in [Handing store text to a model with `untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted), `json` in [Encoding and decoding JSON with `json`](#encoding-and-decoding-json-with-json), `messages`, the section entry's own conversation that each new section VM starts empty, in [`messages` is the section entry's conversation](11-conversations.md#messages-is-the-section-entrys-conversation), `compactors` in [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion), `tools` in [The members of `tools`](12-tools.md#the-members-of-tools), and `tasks` in [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn).

The Engine installs no global beyond these 20. Fanout is `tasks.fanout`, a function on the `tasks` table (see [Fanout in one call](14-fanout.md#fanout-in-one-call)), and reading a list section goes through `prompt.list`.

The H1 pass and every walked section see the same set of globals. The one difference is `argv`: it is writable in the H1 pass, so H1 code can assign it (see [Checking and repairing `argv` in the H1 pass](06-arguments.md#checking-and-repairing-argv-in-the-h1-pass)), and frozen in every other section.

The role label of a declared [model role](10-models.md#model-roles-instead-of-models) and an offered tool never become globals of their own. With the role `writer` bound, `type(writer)` is `nil`. You read a role's model handle with [`models.get(label)`](10-models.md#getting-handles-with-modelsget), so `models.get('writer').name` reads `writer`, and you read a tool with [`tools.get(id)`](12-tools.md#tool-objects-pack-objects-and-toolspacks), so in a prompt that declares `web`, `tools.get('web/search').id` reads `web/search`.

Every section VM is set up in one fixed order. The same order applies to every section entry, the H1 pass, every fanout arm, and every spawned task chain:

1. `untrusted`, when the VM is built
2. `args`, `argv`, `sys`, `var`, `tools`, `models`, `messages`, and `compactors`
3. `log`, `store`, and `json`
4. `ui`
5. `item`
6. `jump` and the `prompt` reads
7. the suspending calls: `call`, the `tasks` table, `tools.call`, `models.infer`, `models.loop`, and `models.compact`
8. Pack preludes
9. `require`
10. the store's suspending calls

### Reserved names

The Engine reserves the name of every global it installs in a section VM, and its reserved-name list matches that set exactly. The list holds 62 names: the 20 Engine globals, the 19 Lua globals the sandbox keeps (listed in [The sandbox](#the-sandbox)), and the 23 Lua 5.5 keywords. It classes each name as an Engine global, a Lua global, or a Lua keyword.

The 23 Lua 5.5 keywords in prompt Lua are `and`, `break`, `do`, `else`, `elseif`, `end`, `false`, `for`, `function`, `global`, `goto`, `if`, `in`, `local`, `nil`, `not`, `or`, `repeat`, `return`, `then`, `true`, `until`, and `while`. A Lua keyword never reads as a global. Every Engine global and Lua global on the list reads non-nil in a section entry once its conditional bindings apply, such as `ui` with a snapshot and `item` in a spawned chain.

Apart from what a Pack prelude adds, every global a section entry's `_G` holds is either an Engine global or a Lua global on the reserved-name list. With no prelude, `_G` holds exactly what the Engine installs.

`global` is a contextual keyword. `global declared` compiles as a global declaration, while `local global = 1` still compiles. The prompt Lua lexer reads `global` as a contextual keyword rather than a reserved word, because the built-in Lua 5.5 uses its compatibility build for `global`, though the language reserves the name all the same. The other 22 keywords cannot be local names, as in any Lua. Reserved global names can: `store` and `pairs` compile as ordinary local names, as in `local store = 1`.

### Suspending calls

A suspending call is an Engine call that waits on the Harness: the section pauses at the call and resumes with the result. You write each one as an ordinary Lua call in straight-line code:

````markdown
---
name: notes
description: Writes a note and reads it back.
runvil: 0
---

# Notes

## Keep

```lua
store.write('note.md', 'hello')
return store.read('note.md')
```
````

The run result is:

````text
hello
````

Both store calls suspend. The block pauses at `store.write` while the Harness writes `note.md`, then pauses at `store.read` and resumes with the text, which `return` makes the run result. Engine calls look synchronous to you: the section's Lua suspends at the call and resumes with the answer, and a successful Engine call returns its result directly to the calling code.

These are the suspending calls:

- `models.infer` and the `infer` method of a [model handle](10-models.md#model-handles)
- `models.loop` (see [Starting a conversation](11-conversations.md#starting-a-conversation))
- `tools.call` (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls))
- `call`
- `input.ask`, which is itself a `tools.call` to the `user-input` Pack's `ask` tool (see [Asking the operator](#asking-the-operator))
- the `tasks` functions, [`tasks.fanout`](14-fanout.md#fanout-in-one-call) and [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) included
- the `store` calls (see [Writing and reading files](09-the-store.md#writing-and-reading-files))

Each one resumes with its result, and `tasks.spawn` resumes with the task table. `jump`, `prompt.section`, and `prompt.list` return without suspending.

A model call reads the same way. `models.infer("hi")` suspends the section's Lua and resumes with the model's answer, so `local reply = models.infer(prose)` holds the reply on the next line. A write followed by a model round over what was written is two lines:

````lua
store.write('note.md', 'hello')
return models.infer(store.read('note.md'))
````

A suspending call works inside `pcall` too (see [Raising and catching errors](#raising-and-catching-errors)). In this block, the section suspends at `models.infer`, and `pcall` returns the result once the block resumes, so when the model answers `answer`, the block returns `pcall:answer`:

````lua
local ok, result = pcall(function() return models.infer("hi") end)
return "pcall:" .. result
````

A block pauses at a suspending call and resumes with the result, so you never write callbacks or manage coroutines, and `coroutine` reads nil in a block. A block pauses only at one of the Engine's suspending calls and never yields on its own, because the `coroutine` library is not available to it.

While a suspending call waits, only the calling [chain](04-how-a-prompt-runs.md#the-section-walk) pauses, and other chains keep running. Two tasks started with `tasks.spawn` that each run `call('#### Leaf')` both complete, in whichever order their answers arrive.

The suspending calls work in every section's Lua, the H1 pass and fanout arms included, so a fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)) can pass its `item` straight to `tools.call`.

A block has no direct route to I/O or a clock, because it never has the `io`, `os`, `package`, `coroutine`, or `debug` library. Work outside the block happens only through Engine calls.

### Pack preludes

A Pack prelude is Lua source that a Pack the prompt declares contributes, and every section VM installs it after the Engine globals and before any block runs. A prelude is how a declared Pack defines Lua globals of its own:

````markdown
---
name: ask-check
description: Reads the globals a Pack prelude adds.
runvil: 0
uses:
  - user-input
---

# Ask check

```lua
return type(input.ask) .. ',' .. type(store.read) .. ',' .. type(tools.call)
```
````

Run on a Host that has the `user-input` Pack and an operator to ask (see [Packs at prepare](04-how-a-prompt-runs.md#packs-at-prepare)), the run result is:

````text
function,function,function
````

The `user-input` prelude defines the global `input`, so `type(input.ask)` reads `function`. A prelude global sits beside the Engine's globals without replacing them, so `type(store.read)` and `type(tools.call)` still give `function` when `user-input` is declared. Your code still sees the `ui`, `argv`, `sys`, and `models` globals when Pack preludes are installed.

A prelude can add any globals to a section entry's `_G` beyond the ones the Engine installs. A prelude that runs `answer = 42`, `label = 'kit'`, and `function shout(text) ... end` makes `answer`, `label`, and `shout` readable from your blocks.

A prelude can also define a table global holding fields and functions. `input` is one: the `user-input` prelude creates the `input` table and defines `input.ask` as a function on it, so a block calls `input.ask()` like any other global function.

Preludes install when each section VM is set up, before that VM runs its first block, the H1 block included. Every Lua global a prelude defines is therefore available to every block of every section entry in the run, fanout arms and spawned task chains included, and to every library a block loads with `require`. In a prompt that declares `user-input`, a required library, a fanout arm, and a spawned task can each call `input.ask()`.

Preludes install in the order the prompt's `uses:` list declares their Packs.

A prelude function called from a block can use `store.write` and `store.read`, and each store call inside it is a store effect issued in call order. A prelude function that writes a file and then reads it back issues the write first and the read second, just as the same two calls in a block would.

#### Prelude tables are read-only

A table global that a Pack's prelude defines is read-only at its top level. You read and call its fields as usual, so `input.ask()` works, but assigning a field, as in `input.extra = 1`, raises this error:

````text
input is read-only: Pack `user-input` defines it; cannot set 'extra'
````

The message has the form `` {name} is read-only: Pack `{pack}` defines it; cannot set '{field}' ``, naming the table, the Pack that defines it, and the field. The table is sealed: `getmetatable(input)` returns `input is sealed`, and `pairs(input)` visits nothing.

#### Which Packs add globals

A prompt that does not declare a Pack gets none of that Pack's globals. Without `user-input` declared, there is no `input` global, and `input.ask()` raises this error:

````text
attempt to index a nil value (global 'input')
````

A prelude global never takes a reserved name, so no Pack global replaces an Engine global, a kept Lua standard-library global, or a Lua keyword.

#### When a prelude fails

All preludes in a run share one global namespace, and a prelude that cannot install fails the run with [run error kind](04-how-a-prompt-runs.md#the-run-result-and-outcomes) `Lua`. Every such message starts with `` Pack `{pack}`: its prelude ... ``, naming the Pack that contributed the prelude. A prelude fails in these cases:

- It defines a global an earlier prelude already defines. The message has the form `` Pack `{pack}`: its prelude defines the global `{name}`, which Pack `{owner}`'s prelude already defines ``, naming the later Pack, the global, and the Pack that defined it first.
- It defines a global that an Engine global or another reserved name already holds. A prelude defining `ui` or `item` fails the run with a message saying the name `is reserved as an Engine global`.
- It fails to load. The message says `its prelude failed to load`.

A prelude failure happens when the first section VM is set up, before the run issues any effect. No store write, tool call, or model request from the prompt happens, and that includes an H1 block such as `store.write('h1.txt', 'x')`, which would otherwise be the first thing the prompt does.

## The sandbox

By the end of this section you know every library and base function a block can call, and which familiar Lua names read nil in a block.

````markdown
---
name: shout
description: Shouts a fixed word.
runvil: 0
---

# Shout

```lua
return string.upper('hello')
```
````

The run result is:

````text
HELLO
````

This is the smallest sandbox block: it returns the result of one library call as the run result. Every Lua block runs in the sandbox, a restricted global environment that offers no direct printing, the safe standard library without the dangerous globals, a `setmetatable` that refuses finalizers and weak globals but still allows `_G` handlers, and the `untrusted` global.

### The whole toolkit

Each section's Lua runs in a sandbox whose only standard libraries are `string`, `table`, `math`, and `utf8`. Beyond those four libraries, the sandbox keeps only these base functions and values:

| Group | Names |
| --- | --- |
| Checking and errors | `assert`, `error`, `pcall`, `xpcall` |
| Iteration | `ipairs`, `pairs`, `next`, `select` |
| Types and conversion | `tonumber`, `tostring`, `type` |
| Metatables | `getmetatable`, `setmetatable` |
| Values | `_G`, `_VERSION` |

Those 15 names and the four libraries are the 19 Lua globals on the reserved-name list (see [Reserved names](#reserved-names)), and that list is the whole toolkit. Every section VM runs Lua 5.5 with these libraries and base functions. The safe base functions a block calls are exactly the kept base globals other than the `math`, `string`, `table`, and `utf8` libraries. File access, the operating system, loading code from files or search paths, and loading code from strings are all outside the toolkit. `_G` is the global table, and `_VERSION` holds the Lua version string.

Six of the base functions behave in a Runvil way:

- `pairs` and `next` visit keys in a fixed order (see [Deterministic table iteration](#deterministic-table-iteration)).
- `pcall` and `xpcall` deliver a failure raised by an Engine function as an [error value](#error-values), while passing your own raised value through unchanged, whether it is a string, a table of your own, or an error value already built (see [Raising and catching errors](#raising-and-catching-errors)).
- `setmetatable` and `getmetatable` give `_G` an author metatable that never replaces the guard on `argv`, `prose`, and `messages` (see [Giving `_G` a metatable](#giving-_g-a-metatable)).

On values other than `_G`, `getmetatable` and `setmetatable` behave as in stock Lua, so `setmetatable(t, mt) == t` and `getmetatable(t) == mt` both hold (see [`setmetatable` and `getmetatable` on other values](#setmetatable-and-getmetatable-on-other-values)). You can call `getmetatable` on a task handle from `tasks.spawn` too, as in `getmetatable(t) == nil`.

The other base functions work as in standard Lua 5.5. You call `ipairs(t)`, `select(n, ...)`, and `tonumber(v)` as in any Lua program, and `select('#', ...)` and `select(2, pcall(...))` give the same results as in stock Lua. `tonumber` turns a numeric string read from the store into a number, as in `tonumber(ok and v or '0') + 1`. `assert(cond, message)` checks a condition with a message. `ipairs` walks an array table, such as a table literal, in order, so `for _, text in ipairs({ 'first question', 'second question' }) do` handles `first question` before `second question`.

The standard functions `string.rep`, `select`, `table.concat`, `ipairs`, `type`, `tostring`, `getmetatable`, and `assert` work around Engine calls too, `models.loop` included, as in `select('#', models.loop())` and `getmetatable(err) == nil`. Section Lua combines them freely, as in `assert(not pcall(require, '## broken'), 'the broken library raises')`, which checks that requiring a broken library raises.

The kept globals are ordinary globals, so you can reassign one in prompt code. `error = function() end` runs without complaint.

### Checking types

`type(v)` returns a value's Lua type name, such as `'string'`, `'number'`, `'boolean'`, `'table'`, `'function'`, or `'userdata'`, so you can test what a value is:

````markdown
---
name: types
description: Reports the type of several values.
runvil: 0
---

# Types

```lua
return type('x') .. ',' .. type(1) .. ',' .. type(true) .. ',' .. type({}) .. ',' .. type(type) .. ',' .. type(never_set)
```
````

The run result is:

````text
string,number,boolean,table,function,nil
````

`type` returns `'nil'` for a global that was never set, such as `never_set` here. Engine values have types too: an error value reports `'table'`, `tasks` reports `'table'`, `tasks.spawn` reports `'function'`, and a tool object reports `'userdata'`. Together with `assert(condition, message)`, `type` makes a check with a message, as in `assert(type(fetch) == 'userdata', 'tools.get reads a tool object')` after `local fetch = tools.get('web/fetch')` in a prompt that declares `web`.

### Turning values into text

You build strings with the `..` concatenation operator, joining string literals with other values, as in `"Hello, " .. args .. "!"`. `..` joins strings and numbers, and an integer renders without a decimal point, so you can render a number such as the one [`tasks.concurrency`](14-fanout.md#concurrency-and-the-admission-window) returns: with `n` equal to `4`, `'clamped:' .. n` reads `clamped:4`, and a join beginning `#history .. '|' .. history[2].content ..` produces `4|first reply|second reply`. `..` also works on a value an Engine call returns, so `return "pcall:" .. result` gives `pcall:answer` when `result` is the reply `answer`.

`tostring(v)` turns other values into text for joining or storing:

- A number becomes its digits, for a return value or an assert message, as in `'expected 1 call, got ' .. tostring(count)`, or for a list item, as in `items[i] = tostring(i)` or `tostring(#files)`.
- A boolean, such as the status `pcall` returns, becomes `true` or `false`, so `var.answer .. ':' .. tostring(var.sys_untouched)` gives `h1 answer:true`, and `tostring(ok)` stores the status as text.
- `nil` becomes the string `nil`, so `tostring(print) .. ':' .. tostring(warn)` returns `nil:nil`.
- An error value becomes its message, so `tostring(err)` gives the error message.
- A tool result or a model reply becomes its text, so `tostring(reply)` gives `pong` when the model replies `pong`.

### The `string` library

The sandbox keeps the `string` library, and you use it as in standard Lua 5.5, including method calls on strings:

````markdown
---
name: strings
description: Uses the string library.
runvil: 0
---

# Strings

```lua
local s = 'Task id=t7'
local id = s:match('^Task id=(%S+)')
local at = s:find('id=', 1, true)
return id .. '|' .. at .. '|' .. string.sub(s, -2) .. '|' .. string.upper(s) .. '|' .. string.rep('ab', 3)
```
````

The run result is:

````text
t7|6|t7|TASK ID=T7|ababab
````

Strings share a metatable whose `__index` is the `string` library, so `getmetatable('').__index == string` holds, and every `string` function works with colon syntax on a string value. `s:match(pattern)` is `string.match(s, pattern)`, and `s:find(text, 1, true)` is `string.find(s, text, 1, true)`. The functions you reach for most:

- `string.upper` and `string.lower` change case, so `return string.upper(args)` with the argument string `hi` returns `HI`, and `return string.lower(value)` lowers a value.
- `string.match` matches a Lua pattern and returns its captures, so `string.match(s, '^(%d+):([%d%.]+)$')` returns two captures. Anchored patterns work on message content too, so `messages[2].content:match('^Answer: (.+)$')` returns the captured text after the prefix. Pattern escapes work as in standard Lua, as `%-` does in `tostring(err):match('read%-only')`.
- `string.find` finds a pattern, as in `string.find(wrapped, "untrusted_input_")`. With its plain-find flag it searches for literal text: `string.find(m.content, 'Answer:', 1, true) == 1` tests for a prefix, and `err:find(']:2: undefined global undefined_here', 1, true)` finds that exact text. A plain search returns `nil` when the text is absent, so `tostring(err):find('include more surrounding text', 1, true) ~= nil` tests whether a message contains that text.
- `string.sub` takes negative indexes that count from the end, inside sections and fanout arms alike, as in `tostring(sys.index) == string.sub(item, -1)`. Its positions count bytes, not characters, so a cut can split a multibyte character, and [the `utf8` library](#the-utf8-library) finds the character boundaries to cut on.
- `string.rep(s, n)` repeats `s` `n` times, so `messages:user(string.rep('x', 100000))` builds a 100000-character user message.
- `string.char` builds a string from raw byte values, which can produce bytes that are not valid UTF-8. `log` rejects the string from `string.char(255)` as not UTF-8 (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)).

### The `table` library

The sandbox keeps the `table` library, and a block calls its functions as `table.{name}(...)`, as in standard Lua 5.5. The `#` length operator works on tables, so you append with `answers[#answers + 1] = value` and join with `table.concat`:

````markdown
---
name: joined
description: Joins strings, numbers, and a value with __tostring.
runvil: 0
---

# Joined

```lua
local point = setmetatable({ x = 1, y = 2 }, {
  __tostring = function(p) return '(' .. p.x .. ',' .. p.y .. ')' end,
})
local answers = {}
answers[#answers + 1] = 'at'
answers[#answers + 1] = point
answers[#answers + 1] = 3
return table.concat(answers, ' ')
```
````

The run result is:

````text
at (1,2) 3
````

`table.concat(list, sep, i, j)` joins `list[i]` through `list[j]`, with `i` defaulting to `1`, `j` defaulting to `#list`, and `sep` defaulting to the empty string. So `table.concat(list)` joins the whole list with nothing between the elements, and `table.concat(items, ', ')` over `alpha` and `beta` gives `alpha, beta`. When the range is empty, such as an empty list or an `i` greater than `j`, `table.concat` returns an empty string. It works in sections and fanout arms alike, and `table.concat(r, '|')` over ten results can produce `1|2|3|4|5|6|7|8|9|10`.

String and number elements join as they are, as in stock Lua. `table.concat` also joins elements that render through `tostring`, converting each one with `tostring` first:

- Every userdata element, such as an Engine handle with a `__tostring` metamethod. `table.concat` decides by type alone here, because `getmetatable` cannot read an Engine handle's metatable, so a userdata element never raises an error. This covers the Engine's own values, such as model handles.
- Every table element whose `getmetatable` result is a table with a `__tostring` field. This covers fanout result entries and your own objects made with `setmetatable(t, { __tostring = f })`, such as `point` above, where `table.concat({ 'at', point }, ' ')` returns `at (1,2)`.

So `table.concat` over a [`tasks.fanout`](14-fanout.md#fanout-in-one-call) result list joins each arm's text with the separator: `table.concat(r, ',')` over three arms returns `a:x1+y2+z3,b:x1+y2+z3,c:x1+y2+z3`.

`table.pack` returns its arguments in a table with an `n` field holding the count, as in `local results = table.pack(pcall(f))` followed by `for i = 1, results.n do`. `select('#', ...)` also counts its arguments.

#### When `table.concat` fails

`table.concat` raises a Lua runtime error when an element in the range cannot join. A boolean, a function, a thread, or a table without a `__tostring` metamethod gives a message naming the type and the index. A `true` at index 3 gives:

````text
invalid value (boolean) at index 3 in table for 'concat'
````

An index in the range that holds nil, such as a hole in the list or a `j` past `#list`, gives `invalid value (nil) at index {k} in table for 'concat'`, where `{k}` is the index inside `i..j`. A `j` past the end therefore reports index `#list + 1`.

A table element whose metatable holds a non-table `__metatable` field, such as `'sealed'` or `false`, fails with `invalid value (table) at index {k} in table for 'concat'` even when that metatable has `__tostring`. `table.concat` reads the metatable through `getmetatable`, which returns the `__metatable` value instead.

A list argument that is not indexable, such as nil, fails with a Lua runtime error from taking `#list` or reading `list[k]`, rather than with stock Lua's bad-argument message. When Lua adds a position prefix to an element error, it points into the sandbox's own `table.concat` code rather than at your calling line (see [Error line numbers and locations](#error-line-numbers-and-locations)).

Each of these is an ordinary Lua runtime error. `pcall` catches it, and left uncaught it ends the run with run error kind `Lua`, or with `RequirementsUnmet` in the H1 pass, where an uncaught `assert(false, 'the gate cannot hold')` also ends the run as `RequirementsUnmet` (see [What an uncaught error ends](#what-an-uncaught-error-ends)).

### The `math` library

The sandbox keeps the `math` library. A block calls it as `math.{name}(...)`, and it works as in standard Lua 5.5.

### The `utf8` library

The sandbox keeps the `utf8` library, and you use it as in standard Lua 5.5 to count and cut text by character instead of by byte:

````markdown
---
name: characters
description: Counts and cuts a string with the utf8 library.
runvil: 0
---

# Characters

```lua
local s = 'café'
local cut = s:sub(1, utf8.offset(s, 0, 5) - 1)
return #s .. '|' .. utf8.len(s) .. '|' .. cut
```
````

The run result is:

````text
5|4|caf
````

`é` takes two bytes in UTF-8, so `#s` counts `café` as 5 bytes while `utf8.len(s)` counts 4 characters. Positions in every `utf8` function are byte positions, as in the `string` library. `utf8.offset(s, 0, 5)` finds the character that holds byte 5, which is `é` starting at byte 4, so the cut ends at byte 3 and keeps `caf` without splitting `é`. In general, when `#text` is greater than `n`, `text:sub(1, utf8.offset(text, 0, n + 1) - 1)` cuts `text` to at most `n` bytes on a character boundary, so valid UTF-8 stays valid for `untrusted` and `log`.

Lua 5.5's `utf8.offset` returns two values, the start and the end byte of the character, where Lua 5.4 returned only the start. A call used as the last argument of another call passes both values, so `s:sub(utf8.offset(s, 2))` reads `a`, byte 2 through byte 2, rather than `afé`. Adjust the result to one value when you want only the start, as the arithmetic in `utf8.offset(s, 0, i) - 1` does.

`utf8.len` checks its text as it counts. On invalid UTF-8 it returns fail, which is `nil`, and the position of the first invalid byte, so `utf8.len(string.char(255))` returns `nil` and `1`.

The library has six members:

- `utf8.char(...)` builds a string from code points, so `utf8.char(233)` returns `é`.
- `utf8.charpattern` is a Lua pattern that matches one UTF-8 character in valid text, so `s:gmatch(utf8.charpattern)` visits `c`, `a`, `f`, and `é` in turn.
- `utf8.codes(s)` iterates over the characters, giving each one's byte position and code point, as in `for at, code in utf8.codes(s) do`, and raises an error on invalid UTF-8.
- `utf8.codepoint(s, i, j)` returns the code points of the characters that start between byte positions `i` and `j`, with `i` defaulting to 1 and `j` to `i`, so `utf8.codepoint('é')` returns `233`.
- `utf8.len(s, i, j)` counts the characters that start between byte positions `i` and `j`, the whole string by default.
- `utf8.offset(s, n, i)` returns the start and the end byte of the `n`-th character counting from byte position `i`, and with `n` equal to 0 it finds the character that holds byte `i`.

### Names that read nil

Everything outside the toolkit reads nil in a block. This prompt checks the two direct printing functions:

````markdown
---
name: no-print
description: Shows that print and warn read nil.
runvil: 0
---

# No print

```lua
return tostring(print) .. ':' .. tostring(warn)
```
````

The run result is:

````text
nil:nil
````

A section VM has no `print` or `warn` global, so a block cannot write to standard output or emit Lua warnings directly. `log(message)` is the output path instead (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)). `print` and `warn` are absent in every kind of Lua VM a prompt runs in, including library code and a VM with tools bound, not only in ordinary section blocks.

These names all read nil from `_G` in a section VM:

| Kind | Names that read nil |
| --- | --- |
| Direct printing | `print`, `warn` |
| Libraries | `io`, `os`, `package`, `debug`, `coroutine` |
| Code loading | `load`, `loadstring`, `loadfile`, `dofile` |
| Raw access | `rawequal`, `rawget`, `rawset`, `rawlen` |
| Function environments | `getfenv`, `setfenv` |
| Garbage collection | `collectgarbage` |

Each absent name closes off one kind of direct work:

- The `io`, `os`, `package`, and `debug` libraries are never loaded, so a block has no Lua file or stream I/O.
- With no `io` library, a block cannot open files or read standard input.
- With no `os` library, `os.time`, `os.clock`, `os.getenv`, and `os.execute` are unavailable.
- With no `package` library, `package.path`, `package.loaded`, and search paths are unavailable.
- With no `load`, a block cannot compile a string into a function at run time. With no `loadfile` or `dofile`, it cannot load or run a Lua file from disk.
- With no `debug` library, a block cannot inspect stack frames, locals, or hooks.
- With no `collectgarbage`, prompt Lua cannot query or drive the garbage collector.

Your code has no `coroutine` global either. The Engine loads the coroutine library only for its own suspending calls and removes the global before section setup ends, so a block cannot create coroutines or yield on its own, and every yield in a run comes from an Engine function (see [Suspending calls](#suspending-calls)).

The raw-access functions are stripped after the Engine guard on `_G` captures `rawequal`, `rawget`, and `rawset` for its own use. With no `debug` library, no `rawset`, and no code loading, your code in a section VM has no route to the Engine guard on `_G`, to a raw `prose` global, or to a raw `argv` global outside the H1 pass (see [Giving `_G` a metatable](#giving-_g-a-metatable)).

Calling any of these absent names, for example `print("hi")`, fails with a Lua runtime error because the name holds nil. The message comes either from the Engine guard on `_G` or from stock Lua's error for calling a nil value.

#### Runvil's own `require`

A prompt's Lua code gets neither the stock Lua `require` loader nor the `package` library. The `require` a block does see is Runvil's own section-loading function (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)):

````markdown
---
name: own-require
description: Shows that require loads sections, not stock Lua libraries.
runvil: 0
---

# Own require

```lua
local ok = pcall(require, 'os')
return type(require) .. '|' .. tostring(ok) .. '|' .. tostring(os) .. '|' .. tostring(package)
```
````

The run result is:

````text
function|false|nil|nil
````

`type(require)` is `'function'`, while `pcall(require, 'os')` returns `false`, loads nothing, and leaves `os` and `package` nil.

## Deterministic table iteration

After this section you can predict the exact order in which `pairs` and `next` visit a table's keys, and change a table safely while you walk it.

````markdown
---
name: order
description: Walks a table with pairs.
runvil: 0
---

# Order

```lua
local seen = {}
for k in pairs({ zeta = 1, alpha = 2, mid = 3, beta = 4 }) do
  seen[#seen + 1] = k
end
return table.concat(seen, ', ')
```
````

The run result is the same on every run:

````text
alpha, beta, mid, zeta
````

`pairs` visits a table's string, number, and boolean keys in a fixed sorted order instead of Lua's hash order, so the same table iterates the same way on every run. This fixed order is the deterministic key order. Write the same table as `{ beta = 4, mid = 3, alpha = 2, zeta = 1 }` and the walk is still `alpha, beta, mid, zeta`, because the order depends only on the keys and not on the table's hash state or on the order you inserted them.

The order comes from sorting each key on its type and then on its value. A table with every kind of scalar key shows the whole order:

````markdown
---
name: mixed-order
description: Walks a table that mixes key types.
runvil: 0
---

# Mixed order

```lua
local seen = {}
for k in pairs({ 'a', 'b', z = 1, [true] = 2, [10] = 3 }) do
  seen[#seen + 1] = tostring(k)
end
return table.concat(seen, ' ')
```
````

The run result is always:

````text
1 2 true 10 z
````

Table iteration visits the array part first, keys `1` through `#t` in index order. Then it visits the rest of the table's keys, its hash part: boolean keys with `false` before `true`, then number keys by value, then string keys bytewise. The sections below give each rule.

### The array part comes first

The array part of a table is its keys `1` through `n` in ascending order, where `n` is `#t`, and `pairs` visits the array part before any other key. `{10, 20, 30, extra = 'x', another = 'y'}` visits `1, 2, 3, another, extra`.

Every array-part key comes before every hash-part key, booleans included, even though booleans rank first inside the hash part. `{[true] = 1, "x"}` visits `1, true`, and `{10, 20, [false] = 'f', [true] = 't'}` visits `1, 2, false, true`.

The array part ends at the table's raw length border, ignoring any `__len` metamethod, so which integer keys count as array keys follows Lua's `#t` border for that table. An integer key inside `1..#t` appears once, in the array part, and is not repeated among the hash-part keys. A nil slot inside `1..#t` is skipped rather than visited, so iterating a table with holes never yields a nil value. In a table with holes, Lua may report any border, so integer keys past a hole may walk either in the array part or among the number keys after the booleans.

Integer keys outside `1..#t`, such as `0`, negative integers, or integers past the border, are ordered as number keys in the hash part, after the array part and after any boolean keys, and they sort among the other numbers by value. So array-part integers come before booleans, while integers that are not part of the array sequence come after them. `{"a", [10] = "c", [true] = "x", flag = 1}` walks `1`, `true`, `10`, `flag`, and `{ [false] = 'b', [3] = 'c', [7] = 'y' }`, which has no array part, walks `false, 3, 7`.

### Then booleans, numbers, and strings

After the array part, the walk visits the boolean, number, and string keys sorted by type: booleans first, then numbers, then strings. Keys of different types are never compared by value. Every boolean key precedes every number key, and every number key precedes every string key, so a table that mixes the three types still gets one order and iterates without error:

````lua
local t = { [true] = 't', [7] = 'seven', b = 'bee', [false] = 'f', [2.5] = 'half', a = 'ay' }
````

`pairs(t)` visits `false, true, 2.5, 7, a, b`. Among boolean keys, `false` comes before `true`, whatever order they appear in the table constructor, and here `[true]` is written first. Keys of the same type come in one shared sorted order, so their sequence never depends on insertion order or hashing.

Boolean keys, integer keys, finite float keys, and valid UTF-8 string keys each get a sortable position. Their iteration order depends only on which keys a table holds, never on the order they were assigned or inserted. Two tables holding the same such keys iterate in the same order, as long as any positive integer keys run from `1` with no holes.

#### Number keys

Number keys iterate in ascending numeric value, with integer and float keys sorted together in one sequence by exact numeric value. In the table above, `[7]` is written before `[2.5]`, yet the walk visits `2.5` before `7`. Integer and float keys interleave strictly by value rather than grouping by subtype, negative keys included:

- `1`, `1.5`, and `2` iterate in that order, and so do `2`, `2.5`, and `3`.
- `-1.5` comes before `-1`, `-1` comes before `-0.5`, and `-0.5` comes before `0`.

Comparing an integer key with a float key never rounds the integer. `math.maxinteger`, which is `9223372036854775807`, sorts before the float key `2^63`, even though converting the integer to a float would round it to that same value. Very large integer keys past 2^53 keep their integer identity, so two different integers keep distinct, exact positions and never tie, however large they are.

Float keys outside the 64-bit integer range sort beyond every integer key. A float key at or above 2^63, such as `1e19` or `1e300`, sorts after every integer key including `math.maxinteger`, and a float key below -2^63, such as `-1e300`, sorts before every integer key.

#### String keys

String keys come after boolean and number keys, and a `for k in pairs(t)` loop visits them in sorted bytewise order of their raw bytes, with no locale or numeric awareness. So a table with only string keys visits them sorted, and:

- `"Z"` comes before `"a"`, and for ASCII `"B"` comes before `"a"`.
- `"10"` comes before `"9"`.
- `"a"` comes before `"ab"`, and `"ab"` comes before `"b"`.

`{ zeta = 1, alpha = 2, mid = 3, beta = 4, omega = 5 }` visits `alpha, beta, mid, omega, zeta`.

### Keys with no fixed order

Keys that are not strings, numbers, or booleans, such as tables, functions, userdata, and threads, are still visited, after every scalar key. Their order among themselves is not fixed: it is whatever Lua's own traversal produced, so you cannot rely on which of two table-valued keys comes first. A table holding the key `a` and two table keys walks `a` first, then both table keys in either order.

Two kinds of scalar key go in that same trailing group with no fixed order: string keys whose bytes are not valid UTF-8, such as `"\255"`, and infinite number keys, `math.huge` and `-math.huge`. They are visited without error. Only boolean keys, finite number keys, and valid UTF-8 string keys have an order that is guaranteed to match across runs.

### The same order everywhere

`pairs` and `next` visit a table's keys in the same order on every run, in every section's fresh Lua state including the H1 pass, and in libraries loaded with `require`, instead of stock Lua's unspecified hash order. Two sections that walk the same table see the same key sequence. The same deterministic key order applies wherever Runvil reads a table's keys in order: [`tasks.fanout`](14-fanout.md#fanout-in-one-call) uses it when it enumerates a table's members, array part first and then the sorted keys.

### How `pairs` walks a table

`pairs` is a global in every section, Runvil's deterministic version of stock Lua's `pairs`. `pairs(t)` returns three values, `f, s, init`: the iterator function, the table itself, and nil, so code that reads the second result sees the table, as in stock Lua.

`pairs` fixes the key order once, when it is called, and the loop then advances through that captured order by position. The iterator function keeps its own position and ignores its arguments, so each call to it advances one key, and calls after the last key keep returning `nil, nil`.

`pairs` and `next` read raw values without consulting `__index`, and they visit only keys actually stored in the table. Because the walk reads the table's raw contents, `__index` and `__len` metamethods do not change which keys are visited, where the array part ends, or the values returned.

### Changing a table during a `pairs` loop

Clearing keys with `t[k] = nil` while a `pairs` loop runs is safe:

````markdown
---
name: clear-ahead
description: Clears a key before the loop reaches it.
runvil: 0
---

# Clear ahead

```lua
local t = { a = 1, b = 2, c = 3 }
local seen = {}
for k in pairs(t) do
  seen[#seen + 1] = k
  if k == 'a' then t.b = nil end
end
return table.concat(seen, ', ')
```
````

The run result is:

````text
a, c
````

Setting a key's value to nil during a `pairs` loop skips that key if the loop has not reached it yet, whatever the key's type, so the loop never visits it with a nil value. Here clearing `b` when `k == 'a'` visits `a` and `c`.

Clearing the current key is allowed too, and the walk moves on to the next live key without an error. Clearing each key of `{a = 1, b = 2, c = 3}` as it is visited still visits `a`, `b`, and `c`. The loop runs to the end whatever the cleared key's type: clearing the boolean key `false` inside a loop over `{10, 20, [false] = 'f'}` still visits each key exactly once, `1, 2, false`, without revisiting the array part. Clearing the current table-valued key does not end the walk either, so the remaining table-valued keys are still visited before the loop ends.

Keys added to a table during a `pairs` loop are not visited by that loop, because the loop sees only the keys captured when it started. Values are different: the loop reads each value when it reaches the key, so a new value assigned to an existing key the loop has not reached yet is read live.

### Tables with a `__pairs` metamethod

A `__pairs` function in a table's metatable, set with `setmetatable`, takes over `pairs`. `pairs(t)` calls it with the table and returns all its results in place of the ordered walk, even when the table itself is empty:

````markdown
---
name: custom-pairs
description: Walks an empty table through its __pairs metamethod.
runvil: 0
---

# Custom pairs

```lua
local t = setmetatable({}, {
  __pairs = function()
    local i = 0
    return function()
      i = i + 1
      if i <= 2 then return i, i * 10 end
    end
  end,
})
local out = {}
for k, v in pairs(t) do
  out[#out + 1] = k .. ':' .. v
end
return table.concat(out, ' ')
```
````

The run result is:

````text
1:10 2:20
````

This `__pairs` returns just an iterator function, with no state or control value, and `pairs` drives the loop with it. A table without `__pairs` gets the ordered walk.

`pairs` looks up `__pairs` on a table's metatable directly, without going through that metatable's own `__index`. A `__pairs` reachable only through the metatable's `__index` is not found, and the default ordered walk runs.

A userdata whose metatable holds `__pairs` can be passed to `pairs` as well, which iterates the pairs that metamethod yields and returns its results. The section's [message list](11-conversations.md#building-a-message-list) `messages`, and a record read from it, are userdata of this kind.

#### When `pairs` fails

`pairs` raises a Lua runtime error in two cases. When `__pairs` holds a value that is neither nil nor a function, a callable table included, `pairs` raises `attempt to call a {type} value (metamethod '__pairs')`, naming the type it found. A table there gives:

````text
attempt to call a table value (metamethod '__pairs')
````

When `pairs` gets a value that is not a table and has no `__pairs` metamethod, such as a string, a number, nil, or a userdata without `__pairs`, it raises `bad argument #1 to 'pairs' (table expected, got {type})`, naming the argument position, the expected type, and the type it got. A nil argument gives:

````text
bad argument #1 to 'pairs' (table expected, got nil)
````

Both are Lua runtime errors. `pcall` catches them, and left uncaught they end the run with run error kind `Lua`, or with `RequirementsUnmet` in the H1 pass.

### Stepping with `next`

`next` is a global in every section, Runvil's deterministic version of stock Lua's `next`, and it steps through a table's keys in the same order `pairs` uses:

````markdown
---
name: step
description: Walks a table with next.
runvil: 0
---

# Step

```lua
local t = { 10, 20, [false] = 'f', [true] = 't' }
local seen = {}
for k in next, t do
  seen[#seen + 1] = tostring(k)
end
local first, value = next({ only = 1 })
return table.concat(seen, ', ') .. '|' .. first .. '=' .. value .. '|' .. tostring(next({}) == nil)
```
````

The run result is:

````text
1, 2, false, true|only=1|true
````

A stateless generic `for k in next, t do ... end` loop walks keys in the same deterministic order as `pairs`, array part first and then boolean keys, so `{10, 20, [false] = 'f', [true] = 't'}` visits `1, 2, false, true`.

`next(t)` or `next(t, nil)` returns the first key and its value in the order `pairs` uses, so `next({ only = 1 })` returns `only, 1`. For an empty table it returns `nil`, so `next(t) == nil` tests whether a table is empty. `next(t, k)` returns the key strictly after `k` in that order, along with its value, and when no key follows `k` it returns two values, `nil, nil`.

`next` keeps no hidden state between calls. It rebuilds the ordered key list from the live table on each call, so each step costs `O(n log n)` in the table's size. On large tables, prefer `pairs`, which builds the ordered list once when the loop starts.

`next` works only on a table's own contents and never calls a `__pairs` metamethod. It accepts only a table and an optional key, so calling `next` with a first argument that is not a table raises a Lua argument error, whose wording comes from the underlying Lua binding rather than from Runvil's own text.

### Changing a table during a `next` loop

Passing `next` a key that is not in the table, whether it was cleared mid-loop or was never present, does not raise an error. The walk resumes at the first key that sorts after that key's position. So during a `next` loop, a scalar key cleared to nil is skipped, and the walk resumes at the first key after the cleared key's sort position.

Clearing keys while stepping with `next`, in `for k in next, t` or in a manual `next(t, k)` loop, never loses a key. Clearing array or boolean keys visits each key exactly once:

- Clearing the current boolean key with `t[k] = nil` still visits every key exactly once and ends the walk normally, without replaying the array part. Clearing each boolean key of `{10, 20, [false] = 'f', [true] = 't'}` as it is visited gives `1, 2, false, true`.
- Clearing array elements does not stop the walk early, so later boolean keys are still visited. Clearing every key of `{10, 20, [false] = 'f'}` as it is visited gives `1, 2, false`, and of `{10, [false] = 'f'}` gives `1, false`.

A cleared integer key of `1` or more resumes as if it were in the array part, so where the walk continues does not depend on the current `#t`. The same rule means that clearing an integer key that was in the hash part makes the walk replay the scalar keys that sort after the array part. Over `{ [false] = 'b', [3] = 'c', [7] = 'y' }`, clearing `t[7]` when `k == 7` visits `false, 3, 7, false, 3`, so booleans and smaller numbers can be visited more than once. No key is lost, and a `pairs` loop never does this.

Clearing a non-scalar key, such as a table key, makes the walk resume at the start of the trailing group of keys with no fixed order, without ending, so the remaining table-valued keys are still visited before the walk ends. When the cleared key is not first in that group, the walk restarts at the group's first remaining key, so an earlier table key the loop left in place is visited again, and the loop still ends.

For a loop that clears or edits keys while it walks, use `pairs`. `pairs` advances a captured snapshot, so you get the exact captured order, while a raw `next` loop re-reads the live table and can replay keys after a hash-part integer key is cleared.

## Keeping state in `var`

After this section you can keep your own values in `var`, read them in later blocks and later sections, and predict exactly what `var` accepts.

````markdown
---
name: greeting
description: Keeps a value in var and reads it back.
runvil: 0
---

# Greeting

## Main

```lua
var.greeting = 'hi there'
return var.greeting
```
````

````text
hi there
````

`var` is an [Engine global](#lua-blocks-section-vms-and-engine-globals): a writable table where your Lua blocks keep their own values. You assign a field, as in `var.greeting = 'hi there'` or `var.count = 1`, and read it back as `var.greeting` or `var.count` later in the same block and in the block's return value. Strings, numbers, and booleans all work as field values.

### From one section to the next

A plain global lasts only as long as its section entry, and `var` is the only thing that passes your values from one section entry to the next:

````markdown
---
name: trail
description: Passes a value to the next section through var.
runvil: 0
---

# Trail

## First

```lua
note = 'kept only here'
var.trail = 'ab'
```

## Second

```lua
assert(note == nil)
return var.trail .. 'c'
```
````

````text
abc
````

`## First` sets a plain global, `note`, and a `var` field, `var.trail`. Each section entry gets a fresh section VM (see [A fresh section VM for every section entry](#a-fresh-section-vm-for-every-section-entry)), so `note` reads as `nil` in `## Second`, while `var.trail` still holds `ab`. A section's final `var` becomes the starting `var` of the next section, so a value one section stores in `var` is readable in later sections of the same walk, both across fall-through and after a [`jump`](08-jump-and-call.md#jumping-to-a-section).

A value you assign to a `var` field stays in `var` after the block that set it ends. Whenever the run needs that state, it reads `var` back as JSON data, as [When the run reads `var` back](#when-the-run-reads-var-back) explains.

`var` is present in the H1 pass and in every section. You can read and write it in every walked section, in every [arm](14-fanout.md#what-an-arm-sees) of [`tasks.fanout`](14-fanout.md#fanout-in-one-call), and in every chain that [`call`](08-jump-and-call.md#calling-a-section) runs. A chain is a walk with its own entries (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). How a call, a fanout arm, or a task starts from its own copy of `var` is covered in [Call input and `var`](08-jump-and-call.md#call-input-and-var).

### Starting values and computed keys

Reading a `var` key that was never set returns `nil` with no error, so `var.key or default` supplies a starting value:

````lua
var.n = (var.n or 0) + 1
````

The first time this line runs, `var.n` is `nil`, so `var.n` becomes `1`, and each later run of the line adds one.

You can also read and write a `var` field with a computed string key in brackets, as in `var[name] = true` or `var[name] == nil`. A bracketed string key reaches the same field as the dotted form, so with `local name = 'greeting'`, `var[name]` and `var.greeting` are one field.

### Between the blocks of one section

A `var` field set in one Lua block of a section is still set in every later Lua block of that section, including from the prologue to the epilog:

````markdown
---
name: phases
description: Reads a var field set in the prologue from the epilog.
runvil: 0
---

# Phases

## Main

```lua
var.phase = 'prologue'
```

Prose between the two blocks.

```lua
return var.phase .. '-epilog'
```
````

````text
prologue-epilog
````

The blocks of one section entry share one section VM, as [A fresh section VM for every section entry](#a-fresh-section-vm-for-every-section-entry) shows. The prose between the two blocks is never read, so it is dropped.

### `var` in the H1 pass

The [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) starts `var` and hands it to the walk:

````markdown
---
name: executions
description: Counts in two H1 blocks and reads the count in a section.
runvil: 0
---

# Executions

```lua
var.executions = (var.executions or 0) + 1
```

Count a second time.

```lua
var.executions = var.executions + 1
```

## Result

```lua
return 'count:' .. var.executions
```
````

````text
count:2
````

The H1 pass starts from an empty `var` table, since it runs first and is never entered again, so the first H1 block finds `var.executions` unset and stores `1`. `var` writes accumulate across the Lua blocks of the H1 pass, so the counter bumped in two H1 blocks reads `2`. Whatever `var` holds when the H1 pass ends is the `var` the section walk starts with, so a value an H1 Lua block stores in `var` is readable in the first `##` section and in every later walked section. A prompt without H1 blocks starts its first top-level walked section with `var` set to an empty table.

`## Result` builds its return value from a `var` value with Lua's `..` operator. A number operand turns into text in the result, which is why `'count:' .. var.executions` gives `count:2`.

### Showing a `var` value in prose

You store a value on `var` in one Lua block and read it in later prose of the same run through a [placeholder](07-substitution.md#placeholders-in-prose):

````markdown
---
name: greet
description: Shows a var value in prose.
runvil: 0
---

# Greet

## Main

```lua
var.greeting = 'hi there'
```

Say {{ var.greeting }} to the reader.

```lua
return prose
```
````

````text
Say hi there to the reader.
````

### What `var` can hold

JSON data is the set of values JSON can express: numbers, strings, booleans, and tables of arrays and objects built from them, nested to any depth. A `var` field holds only JSON data. With this block as the Lua block of `## Main` in the `greeting` prompt, the run succeeds:

````lua
var.n = 1
var.s = 'x'
var.t = { a = { 1, 2 } }
var.b = true
````

The run reads this `var` back as the JSON data:

````text
{ "n": 1, "s": "x", "t": { "a": [1, 2] }, "b": true }
````

Lua integers, strings, and booleans stored in `var` read back as the matching JSON values, a Lua sequence such as `{ 1, 2 }` reads back as a JSON array, and a table with named keys such as `{ a = ... }` reads back as a JSON object. In the other direction, a JSON array held in `var` reads as a Lua sequence indexed from 1.

An empty list reads back as `[]` when it is a marked list, one made with [`json.array`](#keeping-empty-lists-with-jsonarray), decoded by `json.decode`, or taken from a tool result, and as `{}` when it is a plain empty table. So `var.items = json.array()` reads back as `{ "items": [] }`, while `var.items = {}` reads back as `{ "items": {} }`. A [`json.null`](#null-values-and-jsonnull) stored in `var` reads back as `null`.

### Building nested tables

Every table inside `var` is guarded the same way as `var` itself, so you can build nested `var` tables step by step:

````lua
var.t = {}
var.t.kept = 'yes'
return var.t.kept
````

As the Lua block of `## Main` in the `greeting` prompt, this returns `yes`, and `var` reads back as `{ "t": { "kept": "yes" } }`. Every nested write passes the same JSON data check as a top-level write.

### Assigning a table stores a copy

Assigning a table to a `var` field stores a fresh copy, so later edits to the original Lua table do not show up in `var`:

````lua
local list = { 'one', 'two' }
var.list = list
list[2] = 'changed'
return var.list[2]
````

As the Lua block of `## Main` in the `greeting` prompt, this returns `two`. To change the stored value, write through `var` itself, as in `var.list[2] = 'three'`, after which `var.list[2]` reads `three`.

### Removing a key

Assigning `nil` to a `var` field passes the guard and removes that key:

````lua
var.k = 1
var.kept = 'yes'
var.k = nil
return tostring(var.k == nil)
````

As the Lua block of `## Main` in the `greeting` prompt, this returns `true`, and `var` reads back as `{ "kept": "yes" }`. The removed key reads as `nil` and is gone from the read-back `var` data, and removing one key leaves the other keys `var` already held untouched.

### Reading arrays and other tables from `var`

You read `var` and every nested `var` table by key or with `ipairs`:

````lua
var.items = { 'a', 'b', 'c' }
local out = {}
for _, v in ipairs(var.items) do
  out[#out + 1] = v
end
return table.concat(out) .. ' ' .. #var.items .. ' ' .. table.concat(var.items, '', 1, 3)
````

As the Lua block of `## Main` in the `greeting` prompt, this returns `abc 0 abc`. The values you store through `var` live in hidden storage behind a guard, so the `var` table itself stays empty, and so does every table you read from it. Your code cannot reach that hidden storage, because it lives in the Lua registry and the [sandbox](#the-sandbox) has no `debug` library.

Because a table read from `var` holds no entries of its own, a few Lua operations see it as empty:

- The length operator `#` on it returns `0`, even when it holds array entries, which is the `0` in the result above.
- `table.concat` on it returns an empty string unless you pass the end index, as in `table.concat(var.items, '', 1, 3)`, because its default end index is `#`, which is `0` on a `var` table.
- `pairs` over a `var` table yields nothing.
- `next(var)`, or `next` on any nested `var` table, returns `nil` even when the table holds entries, so `next` cannot test whether a `var` table is empty.

`json.encode` does not see a `var` table as empty. It reads through it to the stored data, so in the block above, `json.encode(var.items)` gives `["a","b","c"]`, and `json.encode(var)` gives everything `var` holds as one JSON object (see [Encoding values with `json.encode`](#encoding-values-with-jsonencode)).

`getmetatable` on `var`, or on any table nested inside it, returns the string `var is guarded`, and your code cannot replace that guard, `setmetatable` included: `setmetatable(var, {})` raises the Lua error `cannot change a protected metatable` at your line.

### When the run reads `var` back

The run reads `var` back as JSON data at fixed points:

- when a section ends by falling through, jumping, or returning a value
- when the H1 pass ends
- when prose renders
- when `call`, `tasks.fanout`, or a task start takes its copy of `var`

Nothing reads `var` back between the blocks of one section, because they share one Lua state. A task start is a [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) call.

### When a value is not JSON data

Only JSON data can be assigned to a `var` field. The guard checks each write at the assigning statement, so you can catch a rejected write and keep going:

````markdown
---
name: guarded
description: Catches a rejected var write and keeps going.
runvil: 0
---

# Guarded

## Main

```lua
local ok, err = pcall(function() var.handler = log end)
assert(not ok and tostring(err):match('must be JSON data'))
var.kept = 'yes'
return var.kept .. ' ' .. err.kind
```
````

````text
yes lua
````

`log` is a function, and a function is not JSON data. Assigning a function or userdata to a `var` field raises a Lua error at the assigning line, and the message names the full target path under `var` and the rejected Lua type:

````text
{target} must be JSON data, got {type}
````

Here the write raises `var.handler must be JSON data, got function`. The path is the field's full path, dotted for string keys of object fields and bracketed for any other key, with array elements counted from 1, as in `var.t.f` or `var.list[2]`. The `messages` list or a message record such as `messages[1]` (see [Building a message list](11-conversations.md#building-a-message-list)) is userdata, so assigning one to a `var` field raises the same error ending in `got userdata`. [`json.null`](#null-values-and-jsonnull) is the one userdata the guard accepts, and it stores JSON null: after `var.x = json.null`, `var` reads back as `{ "x": null }`, and `var.x == json.null` holds in Lua.

`var` checks an assigned table deeply, so a table holding a function at any depth, such as `{ f = function() end }`, fails at the assigning line too. That error comes from the conversion to JSON, and its message names the offending type, such as `function`.

A table that is shared at many levels can also be too large to store, because the conversion to JSON copies it once for every place it appears. Before it converts an assigned table, `var` spends the same value and byte budgets that [`json.encode`](#size-and-nesting-limits) does, and a table that runs the budget out raises this error at the assigning line, with the full target path in place of `{target}`:

````text
{target} is too large to store
````

After `local t = {} for i = 1, 40 do t = { t, t } end`, the write `var.x = t` raises `var.x is too large to store`. Like any rejected write it stores nothing. A table that contains itself is not a size problem, so the conversion to JSON refuses it with its own message.

The guard raises its error at the assigning statement itself, not later when `var` is saved. That is why a `pcall` around that one write catches it, the block keeps running, and later `var` writes still work, as `var.kept = 'yes'` does in the `guarded` prompt. A rejected write stores nothing, so `var` holds only the writes that passed the guard: after a rejected `var.t.f = function() end` following `var.t.kept = 'yes'`, `var` reads back as exactly `{ "t": { "kept": "yes" } }`.

`pcall` returns `false` and an [error value](#error-values) whose `tostring` text contains the guard's message, so `tostring(err):match('must be JSON data')` finds it. In a section, the caught error value has error kind `lua`, its `message` field is the guard's text, and `tostring(err)` returns that message. [Raising and catching errors](#raising-and-catching-errors) covers `pcall` in full.

Left uncaught, assigning a non-JSON value such as a function to a `var` field fails the run at the assigning line with [run error kind](04-how-a-prompt-runs.md#the-run-result-and-outcomes) `Lua`.

### Keeping the `var` global in place

Change `var` only through its fields, and keep the `var` global itself in place. Reassigning the `var` global itself makes the next read-back of `var` fail the run with this Lua error, rather than quietly keeping the data `var` held before the reassignment:

````text
the `var` global was reassigned; write `var.<field>` instead
````

The run error kind is `Lua`. When a section entry has reassigned the `var` global, the read-back at the entry's end or at its `jump` fails the run, and the entry does not count as completed. In the H1 pass the run error kind stays `Lua` rather than `RequirementsUnmet` (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)), even when the H1 pass returns a scalar, because the read-back still runs before the scalar becomes the run result.

## Encoding and decoding JSON with `json`

After this section you can turn JSON text into Lua values and back, write and read JSON Lines, keep an empty list as a list, and dump any table to look at it.

````markdown
---
name: json-round-trip
description: Decodes JSON text, edits it, and encodes it again.
runvil: 0
---

# JSON round trip

```lua
local data = json.decode('{"name":"tides","tags":["sea"],"note":null}')
table.insert(data.tags, 'moon')
data.count = #data.tags
return json.encode(data)
```
````

The run result is:

````text
{"count":2,"name":"tides","note":null,"tags":["sea","moon"]}
````

`json.decode` turns the JSON text into a Lua table you can edit, and `json.encode` turns the edited table back into JSON text. The JSON `null` survives the trip, and the object keys come out sorted, whatever order the text or your code put them in.

`json` is an Engine global, a plain table like `string`, with six members:

- `json.decode` and `json.decode_lines` turn JSON text into Lua values.
- `json.encode` and `json.encode_lines` turn Lua values into JSON text.
- `json.null` stands for JSON null.
- `json.array` marks a table as a list, so it stays a list when it is empty.

`json` is in every section VM, the H1 pass, fanout arms, spawned tasks, and libraries loaded with `require` included, and a Pack prelude's functions can call it too, so you never import it. Encoding is strict: a value that is not JSON data raises an error that names exactly where the value sits, unless you ask for a [debug dump](#dumping-any-value-in-debug-mode). Output and error text are identical on every run, because object keys are written sorted and table keys are always checked in one fixed order. Nulls and lists are recognized by the values themselves, never by looking anything up in the `json` table, so reassigning one of its fields changes nothing about how a value encodes.

Every error a `json` function raises is an [error value](#error-values) of kind `lua`, so `pcall` catches it, and `tostring(err)` gives its message with no prefix. Left uncaught, it ends the block like any other `lua` failure (see [What an uncaught error ends](#what-an-uncaught-error-ends)).

### Decoding JSON text with `json.decode`

`json.decode(text)` returns the Lua value of the one JSON document in `text`:

````markdown
---
name: decoded-types
description: Shows the Lua values JSON text decodes to.
runvil: 0
---

# Decoded types

```lua
local v = json.decode('{"n":2,"x":1.5,"big":9223372036854775808,"list":[1,null,3]}')
return math.type(v.n) .. ' ' .. math.type(v.x) .. ' ' .. math.type(v.big) .. ' ' .. #v.list .. ' ' .. tostring(v.list[2] == json.null)
```
````

The run result is:

````text
integer float float 3 true
````

Each JSON value becomes the matching Lua value:

- An object becomes a plain table whose fields you index directly.
- An array becomes a marked list (see [Keeping empty lists with `json.array`](#keeping-empty-lists-with-jsonarray)), a table indexed from 1 that `#`, `ipairs`, `table.insert`, and `table.remove` work on. A `null` item keeps its place, which is why `#v.list` is `3`.
- `null` becomes [`json.null`](#null-values-and-jsonnull), never `nil`.
- A string becomes a Lua string, and `true` and `false` become booleans.
- An integer in the 64-bit integer range becomes a Lua integer. Every other number becomes a float, including `1.0` and an integer past that range, such as `9223372036854775808`.

Every call builds fresh tables, and you can write to them freely: add fields to a decoded object, or append to and remove from a decoded list.

`json.decode` takes a string of at most 8,388,608 bytes (see [Size and nesting limits](#size-and-nesting-limits)) and checks its argument before it parses anything. It raises three errors:

| When | Message |
| --- | --- |
| The argument is not a string | `json.decode expects a string, got {type}` |
| The string is longer than the cap | `json.decode: input exceeds the 8388608-byte limit` |
| The text is not one valid JSON document | `json.decode: {error}` |

`{error}` is the parser's message, which says what went wrong and where, by line and column. `json.decode('x')` raises `json.decode: expected value at line 1 column 1`, and `json.decode('1 2')`, which holds two documents, raises `json.decode: trailing characters at line 1 column 3`. To decode text that may be malformed, call `json.decode` under `pcall`:

````lua
local ok, data = pcall(json.decode, text)
if not ok then
  return 'bad JSON: ' .. tostring(data)
end
````

With `text` holding `x`, this block returns `bad JSON: json.decode: expected value at line 1 column 1`.

You decode only JSON that arrives as text, such as a plain tool's output, a store file, the argument string, or a model reply. A [structured tool](12-tools.md#structured-output) already hands your Lua a parsed table built the same way, with marked lists and `json.null`. A web tool's text arrives inside the untrusted envelope, so you cut the JSON out with `string.match` before decoding it, as [Decoding a fetched JSON body](13-web-fetch-and-search.md#decoding-a-fetched-json-body) shows.

Decoded strings are plain Lua strings, never wrapped in the run's untrusted envelope, even when the JSON came from the web or a model. Before such text reaches a model, wrap it with [`untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted).

### Null values and `json.null`

`json.null` is the Lua value for JSON null:

````markdown
---
name: null-check
description: Tells a JSON null apart from a missing field.
runvil: 0
---

# Null check

```lua
local row = json.decode('{"id":7,"note":null}')
local seen = {}
if row.note then seen[#seen + 1] = 'truthy' end
if row.note == json.null then seen[#seen + 1] = 'null' end
if row.missing == nil then seen[#seen + 1] = 'absent' end
seen[#seen + 1] = json.encode({ id = row.id, note = json.null, gone = nil })
return table.concat(seen, ' ')
```
````

The run result is:

````text
truthy null absent {"id":7,"note":null}
````

Every JSON `null` decodes to `json.null`, and `json.null` is what you put in a table to encode `null`. A field set to `nil` is no field at all, so `gone` is left out of the encoded object, while `note = json.null` keeps its key. In a list, `json.null` holds its position, so `json.encode({ json.null, 1 })` gives `[null,1]`.

`json.null` is a light userdata, so `type(json.null)` is `userdata`, and error messages name its type `lightuserdata`. Like every userdata it is truthy, so `if row.note then` passes for a null, as the `truthy` in the result shows. Test for null with `v == json.null`, and for an absent field with `v == nil`.

Outside `json.encode`, `json.null` passes the Engine's other conversions to JSON too. The `var` guard accepts it and stores JSON null (see [When a value is not JSON data](#when-a-value-is-not-json-data)), a `tools.call` argument table or a task's `item` can hold it, and a fanout member that is `json.null` runs its arm with `item == json.null`. A JSON null nested in the argument string reads as `json.null` in `argv` too, as [Missing fields and null values](06-arguments.md#missing-fields-and-null-values) shows. A prose placeholder still refuses it: naming a bare global that holds `json.null`, as in `{{ nothing }}` after `nothing = json.null`, fails with `missing {{ nothing }}` and substitution kind `NullValue`, the same failure a `var` field holding null gives (see [Substitution error kinds](07-substitution.md#substitution-error-kinds)).

### Keeping empty lists with `json.array`

An empty Lua table could be an empty list or an empty object, and `json.array` says which one you mean:

````markdown
---
name: empty-list
description: Keeps a filtered list as a list when nothing passes the filter.
runvil: 0
---

# Empty list

```lua
local long, plain = json.array(), {}
for _, word in ipairs({ 'alpha', 'beta', 'gamma' }) do
  if #word > 5 then
    long[#long + 1] = word
    plain[#plain + 1] = word
  end
end
return json.encode({ long = long, plain = plain })
```
````

The run result is:

````text
{"long":[],"plain":{}}
````

No word is longer than five letters, so both tables stay empty, yet they encode differently. An unmarked empty table encodes as an object, `{}`, because nothing in it says it is a list. `long` is a marked list, so it encodes as an array, `[]`, however many items it ends up with. Wrap any list that may end up empty with `json.array` before it leaves your code. There is no `json.object`, because a plain `{}` already encodes as an object.

`json.array()` returns a new empty marked list, and so does `json.array(nil)`. `json.array(t)` marks the table `t` as a list in place and returns `t` itself, so `json.array(t) == t` holds. A table that is already a marked list, such as a decoded list, comes back unchanged.

A marked list carries a metatable whose only field is `__metatable = false`. So `getmetatable` on a marked list returns `false`, and `setmetatable` on one raises `cannot change a protected metatable` (see [`setmetatable` and `getmetatable` on other values](#setmetatable-and-getmetatable-on-other-values)). Every list that `json.decode` returns is marked, and so is every list in a [structured tool result](12-tools.md#structured-output). Otherwise a marked list is an ordinary writable table: `#`, `ipairs`, `table.insert`, and `table.remove` work on it, and a decoded list emptied with `table.remove` still encodes as `[]`.

`json.array` raises two errors:

- `json.array expects a table or no argument, got {type}`, for any value other than a table or nil, so `json.array(5)` reports `got integer`.
- `json.array: the table already has a metatable`, for a table whose metatable is anything other than the list marker. That covers `var` and every table read from it, frozen `argv`, sealed Pack globals, fanout results, `_G`, and any table you gave a metatable yourself.

The marker works beyond `json.encode`. Every place the Engine turns a Lua table into JSON honors it, so a marked list goes out as `[]` in a `tools.call` argument table, a task's `item`, a message record, and `var`, where a plain empty table goes out as `{}`. An empty list stored in `var` or in frozen `argv` keeps its marker, so after `var.items = json.array()`, `var` reads back as `{ "items": [] }`, and a `{{ var.items }}` placeholder renders `[]` (see [When the run reads `var` back](#when-the-run-reads-var-back)). `var.items = {}` still reads back as `{ "items": {} }`.

### Encoding values with `json.encode`

`json.encode(value [, options])` returns `value` as compact JSON text, with no spaces and with object keys sorted bytewise:

````markdown
---
name: encode-var
description: Encodes what var holds.
runvil: 0
---

# Encode var

## Main

```lua
var.cfg = { mode = 'fast', sizes = { 2.0, 1.5 } }
var.empty = json.array()
return json.encode(var) .. ' ' .. #var.cfg.sizes .. ' ' .. json.encode(var.cfg.sizes)
```
````

The run result is:

````text
{"cfg":{"mode":"fast","sizes":[2,1.5]},"empty":[]} 0 [2,1.5]
````

`json.encode` reads `var` and its nested tables through to the data they store, even though `#var.cfg.sizes` sees an empty table, and the float `2.0` comes out as `2`. The only option is `debug`, which [Dumping any value in debug mode](#dumping-any-value-in-debug-mode) covers.

Scalars encode this way:

- `json.null` encodes as `null`, and so does `nil` passed as the whole value, so `json.encode()` gives `null`. Inside a table, a `nil` value is simply no key.
- Booleans encode as `true` and `false`.
- Integers encode as integers. A float with no fractional part, from -2^63 up to but not including 2^63, encodes as an integer too, so `2.0` gives `2`, `-0.0` gives `0`, and `-2^63` gives `-9223372036854775808`, while `2^63` stays a float.
- Every other finite float is written as the shortest text that reads back as exactly the same number, so `1.5` gives `1.5`, and `1e300` gives `1e+300`.
- NaN and the infinities, such as `0/0` and `math.huge`, raise an error.
- Strings must be valid UTF-8. A newline inside a string is written as `\n`, so every encoded value is one line of text.

Tables encode this way:

- A marked list encodes as an array, and its keys must be exactly 1 to n, where n may be 0.
- An unmarked table with no keys encodes as `{}`.
- An unmarked table whose keys are exactly 1 to n encodes as an array.
- An unmarked table whose keys are all strings encodes as an object, and every key must be valid UTF-8.
- Any other table raises an error, such as a table that mixes list items and named keys, a list with a gap, or a table with a boolean or float key.
- A table's keys are checked before any of its values are encoded.
- A table that contains itself raises an error, while the same table in two places is fine: after `local s = { 1 }`, `json.encode({ a = s, b = { s } })` gives `{"a":[1],"b":[[1]]}`.
- A table nested more than 127 levels deep raises an error (see [Size and nesting limits](#size-and-nesting-limits)).
- A value whose walk spends more than the size budget raises an error, as a table shared at every level of a deep nest does (see [Size and nesting limits](#size-and-nesting-limits)).
- Any metatable other than the list marker or a sealed view's is ignored, and the table's own entries are encoded, so a `__pairs` or `__tostring` metamethod never changes the output.

Functions, threads, and every userdata other than `json.null` are not JSON data, so they raise an error. That includes Engine objects such as tool objects, the `messages` list, and the record views that `messages[i]` returns. To encode a message record, encode `messages[i].content` or a plain table you build from its fields.

A sealed view is a table that holds no entries of its own and whose metatable has both `__index` and `__metatable`. `var` and every table read from it, frozen `argv`, sealed Pack globals, and [fanout results](14-fanout.md#reading-the-results) are sealed views. When a view's `__index` is a table, `json.encode` encodes that table in the view's place, so `json.encode(var)` gives everything `var` holds. For a fanout arm over the member `alpha` whose worker returns `item .. '-' .. sys.index`, `json.encode(r[1])` gives `{"exhausted":false,"item":"alpha","ok":true,"text":"alpha-1"}`. When a view's `__index` is a function, as in `sys`, the view cannot be read, and `json.encode(sys)` raises this error:

````text
json.encode: value cannot be read (sys is sealed); copy the fields you need into a plain table
````

The parenthesis holds the view's `__metatable` text and is left out when that is not a string. Inside a Pack prelude's function, `var` is a read-only view of this kind, so `json.encode(var)` there raises the same error with `(var is read-only)`. A table whose `__index` is neither a table nor a function is not a sealed view, and it encodes as its own entries.

### When `json.encode` refuses a value

A strict encode raises at the first value that is not JSON data:

````markdown
---
name: encode-refused
description: Catches a value that json.encode refuses.
runvil: 0
---

# Encode refused

```lua
local ok, err = pcall(json.encode, { items = { 'a', function() end } })
return err.kind .. '|' .. tostring(err)
```
````

The run result is:

````text
lua|json.encode: value.items[2] must be JSON data, got function
````

Every message names the failing value by its path. The path starts at `value`, the value you passed, and adds `.name` for each string key and `[k]` for any other key, so `value.items[2]` is the second item of the `items` field. A string key that is not valid UTF-8 appears in the path with replacement characters. The size refusal is the one message with no path, because the walk stops wherever the budget runs out. These are the messages:

| Problem | Message |
| --- | --- |
| A function, thread, or userdata other than `json.null` | `json.encode: {path} must be JSON data, got {kind}` |
| NaN or an infinity | `json.encode: {path} is not a finite number` |
| A string or object key that is not valid UTF-8 | `json.encode: {path} is not valid UTF-8` |
| An unmarked table with both integer and string keys | `json.encode: {path} mixes list items and named keys` |
| Integer keys that are not exactly 1 to n | `json.encode: {path} is missing list item [{item}]` |
| An unmarked table with a key that is neither an integer nor a string | `json.encode: {path} has a {kind} key; object keys must be strings` |
| A marked list with a key that is not an integer | `json.encode: {path} is marked as a list but has a {kind} key` |
| A table that contains itself | `json.encode: {path} contains itself` |
| A table more than 127 levels deep | `json.encode: {path} is nested more than 127 levels deep` |
| A walk that spends more than the size budget | `json.encode: value is too large to encode` |
| A sealed view whose `__index` is a function | `json.encode: {path} cannot be read ({label}); copy the fields you need into a plain table` |

`{type}`, like `{kind}` in the table above, is the type name every Engine argument error uses, which is not always what Lua's `type` returns: an integer reads `integer`, a float reads `number`, and `json.null` or any other light userdata reads `lightuserdata`. So `json.decode(5)` raises `json.decode expects a string, got integer`, `json.decode(1.5)` raises `json.decode expects a string, got number`, and `json.array(json.null)` raises `json.array expects a table or no argument, got lightuserdata`.

A table that breaks more than one rule reports the same error on every run. An unmarked table reports a key that is neither an integer nor a string first, then a mix of integer and string keys, then a missing list item. Because keys are checked before values, `{ [1] = function() end, a = 1 }` reports `mixes list items and named keys`, not the function. For a missing item, `{item}` is the first of 1 to n that is absent, where n is the table's count of integer keys: `{ [1] = 'a', [3] = 'c' }` reports `[2]`, `{ [0] = 'a' }` reports `[1]`, and `{ [-1] = 'a', [1] = 'b' }` reports `[2]`.

Within a table, `json.encode` visits keys in one fixed order: booleans, then numbers by value, then strings bytewise, as [Then booleans, numbers, and strings](#then-booleans-numbers-and-strings) describes. Unlike `pairs`, it also gives a fixed place to the two kinds of scalar key that [Keys with no fixed order](#keys-with-no-fixed-order) leaves unordered: a string key that is not valid UTF-8 sorts bytewise among the strings, and an infinite number key sorts by value among the numbers, so `-math.huge` comes first and `math.huge` last. Table, function, userdata, and thread keys come last, ranked by type name, so a table with both a table key and a function key always reports `has a function key`.

The options argument is checked before the value, so `json.encode(function() end, 5)` raises the options error, not the function's. Options must be nil or a table whose only key is `debug`:

- A value that is not a table raises `json.encode: options must be a table, got {type}`.
- Any other key raises `json.encode: unknown option {key}`, as in `json.encode: unknown option pretty`. A key that is neither a string nor an integer is named by its type, as in `unknown option boolean`, so a float key reads `unknown option number`.
- A `debug` value that is not a boolean raises `json.encode: option debug must be a boolean, got {type}`.

In every `json` function, a Lua memory-limit error raised while a result is being built passes through unchanged, with no `json.` prefix, like any other memory refusal.

### Dumping any value in debug mode

`log` takes at most 256 characters on one line (see [Message rules](#message-rules)), so it cannot show you a whole table. To look at a table while you work on a prompt, encode it with `json.encode(t, { debug = true })` and write the text to the store or return it:

````markdown
---
name: debug-dump
description: Dumps a table that is not JSON data into the store.
runvil: 0
---

# Debug dump

```lua
local state = { step = 2, retry = function() end, ratio = 0/0 }
state.self = state
store.write('dump.json', json.encode(state, { debug = true }))
return store.read('dump.json')
```
````

The run result is:

````text
{"ratio":"NaN","retry":"[function]","self":"[cycle]","step":2}
````

Debug mode never fails on a value that stays within the [size budget](#size-and-nesting-limits). It follows the same rules as a strict encode, but writes a string in place of each failure:

- A function becomes `"[function]"`, a thread becomes `"[thread]"`, and every userdata other than `json.null` becomes `"[userdata]"`, whether or not it defines `__tostring`.
- A table that contains itself becomes `"[cycle]"`, and a table more than 127 levels deep becomes `"[too deep]"`.
- NaN becomes `"NaN"`, infinity becomes `"Infinity"`, and negative infinity becomes `"-Infinity"`.
- Bytes that are not valid UTF-8, in strings and in keys, are replaced with replacement characters.
- A sealed view that cannot be read becomes `"[cannot read: {label}]"`, as in `"[cannot read: sys is sealed]"`, or `"[cannot read]"` when its `__metatable` is not a string.
- A table that breaks the key rules, marked or not, becomes an object, so `{ [1] = 'a', [3] = 'c' }` becomes `{"1":"a","3":"c"}`. A number key becomes the text it would encode as, such as `2`, `2.5`, or `1e+300`, an infinite key becomes `Infinity` or `-Infinity`, and a boolean key becomes `true` or `false`. When two keys give the same text, the later one in visiting order wins, so with both `t[1]` and `t['1']` set, the string key's value is kept. Table, function, userdata, and thread keys are left out, because their order is not fixed between runs.

Option errors still raise in debug mode, and so does the size refusal, `json.encode: value is too large to encode`, because a placeholder for every remaining value would not stop the walk. The output never contains a memory address. Use debug mode only to look at values. A placeholder such as `"[function]"` is ordinary text to `json.decode`, a tool, or a model, so encode data you send anywhere without `debug`, where a value that is not JSON data fails at the line that encodes it.

### JSON Lines with `json.encode_lines` and `json.decode_lines`

JSON Lines is text with one JSON value per line, which suits a store file you add records to over time:

````markdown
---
name: rows
description: Appends records as JSON Lines and reads them back.
runvil: 0
---

# Rows

## First

```lua
store.append('rows.jsonl', json.encode_lines({ { n = 1, tag = 'a' } }))
```

## Second

```lua
store.append('rows.jsonl', json.encode_lines({ { n = 2, tag = 'b' }, { n = 3, tag = json.null } }))
local rows = json.decode_lines(store.read('rows.jsonl'))
return json.encode(rows)
```
````

When the store holds no `rows.jsonl` yet, the run result is:

````text
[{"n":1,"tag":"a"},{"n":2,"tag":"b"},{"n":3,"tag":null}]
````

`json.encode_lines(list [, options])` encodes each item of `list` by the `json.encode` rules and ends each one with a newline, so an empty list gives `""`. Because the last record ends in a newline too, the result can go straight onto the end of a file that already ends in a record. [`store.append`](09-the-store.md#appending-to-a-file) adds no separator of its own, so `store.append('rows.jsonl', json.encode_lines(rows))` appends whole records. The list can be a marked list, an empty table, a table with keys 1 to n, or a sealed view of one, such as `var.rows`, and `json.encode_lines` takes the same `debug` option as `json.encode`.

`json.decode_lines(text)` returns a marked list of every JSON value in `text`, in order. Any whitespace separates values, so blank lines and `\r\n` line endings are fine, several values can share a line, and one value can span several lines. Empty or blank text gives an empty marked list. It takes at most 8,388,608 bytes, like `json.decode`.

`json.decode_lines` raises the same three errors as `json.decode`, with `json.decode_lines` in place of `json.decode`. A parser message counts lines and columns across the whole text, so a bad third record reports `at line 3`, as in `json.decode_lines: expected value at line 3 column 6`.

`json.encode_lines` checks its list argument before it encodes any item, and it raises these errors:

- `json.encode_lines expects a list, got {type}`, for a value that is not a table.
- `json.encode_lines expects a list, got a table with a {type} key`, for an unmarked table that has keys but no list items, naming the first key in visiting order. An object such as `{ a = 1 }` reports `got a table with a string key`.
- The table rules of `json.encode` with the `json.encode_lines:` prefix, for a table that has list items but is still not a list, as in `json.encode_lines: value mixes list items and named keys` or `json.encode_lines: value is missing list item [2]`.
- `json.encode_lines: value cannot be read (sys is sealed); copy the fields you need into a plain table`, for a sealed view that cannot be read, such as `sys`.
- The three options errors, with the `json.encode_lines:` prefix, as in `json.encode_lines: unknown option pretty`.
- `json.encode_lines: value is too large to encode`, when the items of one call together spend more than the size budget. The items share one budget, so the refusal can come from any item or from the list as a whole.

An error inside an item names the item by its path under the list, as in `json.encode_lines: value[2].name must be JSON data, got function`. The list stays on the cycle path while its items are encoded, so a list that contains itself raises `json.encode_lines: value[1] contains itself`. In debug mode the argument errors above and the size refusal still raise, and only the other failures inside items become placeholders.

### Size and nesting limits

`json.decode` and `json.decode_lines` each take at most 8,388,608 bytes of text, the same cap [`web/fetch`](13-web-fetch-and-search.md#fetching-a-page) applies to a JSON body. A longer string raises `json.decode: input exceeds the 8388608-byte limit`, or the same message with the `json.decode_lines:` prefix, before any parsing starts.

Nesting is limited to 127 levels in both directions, so `json.encode` never writes text that `json.decode` would refuse:

- `json.decode` accepts 127 arrays and objects nested inside each other and rejects the 128th with the parser's own message, as in `json.decode: recursion limit exceeded at line 1 column 128` for text that opens with 128 brackets.
- `json.encode` counts the value you pass as level 1 and raises `json.encode: {path} is nested more than 127 levels deep` for a table at level 128. The path names that table, such as `value` followed by 127 steps of `[1]` for lists nested in lists. In debug mode the table becomes `"[too deep]"` instead.

Reading through a sealed view to its data adds no level, so a `var` value nested 127 levels deep still encodes. At each table, `json.encode` checks for a cycle before it checks the depth, so a loop of 127 tables or fewer reports `contains itself`, even when the repeated table falls at level 128. A longer loop reaches the depth limit before it repeats, so it reports the depth.

`json.encode` and `json.encode_lines` also bound the work of one call, because a table can be reachable from many places while the Lua heap holds it once. After `local t = {} for i = 1, 40 do t = { t, t } end`, the heap holds 41 small tables, but `json.encode(t)` would write about 2^41 values. Each call therefore spends a budget while it walks, and a call that runs the budget out stops and raises this error, or the same message with the `json.encode_lines:` prefix, with no path:

````text
json.encode: value is too large to encode
````

The budget comes from the Lua memory limit of the section VM, `lua_memory_bytes`, which is 64 MiB by default (see [Lua memory and log quotas](16-limits-and-errors.md#lua-memory-and-log-quotas)), so a value that fits the heap once is never refused for its size alone. The walk spends it in three ways:

- One value for each value the walk visits, up to one eighth of the memory limit, which is 8,388,608 values at the default. A table counts once for every place it appears, and each sealed view the walk reads through counts too. At the default limit this is enough for any text that `json.decode` accepts, so a value decoded from text within the 8,388,608-byte cap encodes again.
- One byte for each byte of every string value and every string key the walk visits, up to the memory limit itself, which is 67,108,864 bytes at the default. A string that appears in many places counts at each place.
- At most 100,000 keys that are not numbers or booleans on one path, counting the keys of the table being encoded and of every table above it on the way down. Integer keys cost nothing here, so a list of a million strings still encodes, while an object with more than 100,000 string keys on one path raises the same error.

Debug mode raises this error too. A `var` write spends the same value and byte budgets before it converts the value, as [When a value is not JSON data](#when-a-value-is-not-json-data) describes.

## Run facts in `sys` and `ui`

After this section you can read the facts the run keeps about itself from `sys`, read the Host's state from `ui()`, and know when each value is fixed.

````markdown
---
name: where-am-i
description: Reads run facts from sys.
runvil: 0
---

# Where am I

## Only

```lua
return sys.section_name .. ' ' .. sys.id .. ' of ' .. sys.section_count
```
````

````text
Only 0.1 of 1
````

`sys` is an Engine global that the Engine installs in every section entry and in the H1 pass: a table of runtime values the run supplies, which a Lua block reads run metadata from. `sys` is a sealed, read-only table, and you read a field by name with dot or bracket syntax, as in `sys.id` or `sys['when']`.

### The `sys` fields

Every section entry, each fanout arm's included, gets six `sys` fields: `sys.when`, `sys.id`, `sys.taskid`, `sys.section_name`, `sys.execution`, and `sys.section_count`. Two more fields, `sys.model` and `sys.index`, appear only in some section entries:

| Field | Type | Example value | Present |
|---|---|---|---|
| `sys.when` | string | `2000-02-29T00:00:00.123Z` | always |
| `sys.id` | string | `0.1` | always |
| `sys.taskid` | string | `0` | always |
| `sys.section_name` | string | `Alpha` | always |
| `sys.execution` | string | `execute-test` | always |
| `sys.section_count` | number | `2` | always |
| `sys.model` | string | `claude-sonnet-4-6` | after the section entry's first tool call |
| `sys.index` | number | `7` | on a task's first section entry, when its spawn supplied one |

A present `sys` field reads as a plain Lua value converted from its JSON value. `when`, `id`, `taskid`, `section_name`, `execution`, and `model` arrive as Lua strings, and `section_count` and `index` arrive as Lua numbers, so `sys.section_name == 'Alpha'` compares as a string and `sys.section_count == 2` as a number. The rest of this section takes the fields one at a time, and [Section entry ids](#section-entry-ids) covers `sys.id` in full.

### Section name, section count, and run name

````markdown
---
name: two-sections
description: Reads the section name, section count, and run name.
runvil: 0
---

# Two sections

## Alpha

```lua
assert(sys.section_name == 'Alpha')
assert(sys.section_count == 2)
```

## Beta

```lua
assert(sys.section_name == 'Beta')
assert(sys.section_count == 2)
return sys.execution
```
````

In a run named `execute-test`, every assertion holds and the run result is:

````text
execute-test
````

`sys.section_name` is a string holding the section name of the section whose Lua is running, so it reads `'Alpha'` in section Alpha and `'Beta'` in section Beta. `sys.section_count` is a number holding the count of top-level sections in the prompt. `sys.execution` is a string holding the run's name, which the Harness assigns. Both read the same in every section, the H1 pass included.

In the H1 pass, Lua runs under the title, so `sys.section_name` reads the prompt's title:

````markdown
---
name: title-demo
description: Reads the title from sys in the H1 pass.
runvil: 0
---

# Title Demo

```lua
return sys.section_name
```
````

````text
Title Demo
````

Fanout arms, and chains that an arm starts with `call`, see the same `sys.section_count` as the run, its top-level section count, and a child worker section does not add to it. With `## Parent` as the only top-level section, a `### Worker` arm under it reads `sys.section_count == 1`.

### The task id in `sys.taskid`

````markdown
---
name: arm-task
description: Reads the task id on the main walk and inside a fanout arm.
runvil: 0
---

# Arm task

## Parent

```lua
assert(sys.taskid == '0')
local results = tasks.fanout('### Worker', {'a'})
return results[1].text
```

### Worker

```lua
return sys.taskid
```
````

````text
0.0
````

`sys.taskid` is a string holding the id of the nearest enclosing task, the task the code runs inside (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)). It reads `'0'` on the main walk and in the H1 pass, which is why the assertion in `## Parent` holds. Every section a chain enters reads the same `sys.taskid`, the chain's task id.

Inside a spawned task or a fanout arm, `sys.taskid` reads that task's own id, the same value as the `task` field of the handle `tasks.spawn` returns, so the first task spawned from the main walk reads `0.0`, as the arm above does. Inside a called chain, `sys.taskid` reads the caller's task, because a `call` starts no task, while inside a fanout arm or a `tasks.spawn` chain it reads that task's own id.

### The spawn index in `sys.index`

A task's first section entry also has `sys.index` when its spawn supplied one. A fanout arm reads its member's 1-based position (see [What an arm sees](14-fanout.md#what-an-arm-sees)), and a task started by `tasks.spawn` with `index = 7` among its options reads `7`. Every other section entry, ordinary walked sections and a task's later entries included, has no `sys.index` field, and reading it there fails with `unknown sys field 'index'`.

### Reading a field `sys` does not have

Reading a field that `sys` does not have raises an error instead of returning a silent `nil`, and the message names the field:

````text
unknown sys field '{field}'
````

So reading `sys.index` in an ordinary walked section raises `unknown sys field 'index'`. The error is a runtime error with error kind `lua`. Left uncaught, it ends the run with run error kind `Lua`, or with `RequirementsUnmet` in the H1 pass. A prose placeholder for an absent field fails with its own message, which repeats the placeholder after `missing`, so `{{ sys.index }}` in an ordinary walked section's prose fails with `missing {{ sys.index }}`.

To read a `sys` field that may be absent, such as `sys.model` before any tool call, wrap the read in `pcall`. As the Lua block of `## Only` in the `where-am-i` prompt, this returns `no tool call yet`:

````lua
local ok, model = pcall(function() return sys.model end)
return ok and model or 'no tool call yet'
````

The run goes on whether or not the field is there. A caught unknown field error is an error value with error kind `lua`, and `tostring` on it reads `runtime error: unknown sys field '{field}'` (see [Reading `kind` and `message`](#reading-kind-and-message)), so here, where the read fails, `model` holds that error value and `tostring(model)` reads `runtime error: unknown sys field 'model'`.

### `sys` is sealed and read-only

The `sys` table itself holds no entries, so its values are reachable only by reading a named field, and iterating `sys` with `pairs` or `next` yields no entries. `sys` takes string keys only, so you read a field by its name, as in `sys.when` or `sys['when']`, and a read with any other key type raises `sys fields must be accessed by string key`.

`sys` is read-only. Assigning any `sys` field, existing or new, raises a Lua error that names the key:

````text
sys is read-only; cannot set '{field}'
````

An assignment to `sys.when` gives `sys is read-only; cannot set 'when'`, and one to a new field `sys.extra` gives `sys is read-only; cannot set 'extra'`. The `sys` read and write errors, like the `var` JSON check errors, are ordinary Lua runtime errors raised at the failing line, so `pcall` can catch them, as in `local ok, err = pcall(function() sys.extra = 1 end)`.

`getmetatable(sys)` returns the string `sys is sealed` instead of the real metatable, and your code cannot replace the seal, so `setmetatable(sys, {})` fails with `cannot change a protected metatable`.

### The start time in `sys.when`

````markdown
---
name: started
description: Reads the run's start instant in the H1 pass and in a section.
runvil: 0
---

# Started

```lua
var.h1_when = sys.when
```

## Main

```lua
assert(sys.when == var.h1_when)
return sys.when
```
````

For a run that started at 00:00:00.123 UTC on 29 February 2000, the run result is:

````text
2000-02-29T00:00:00.123Z
````

`sys.when` is a string holding the instant the run started, rendered as RFC 3339 in UTC and ending in `Z`. It is one fixed instant stamped at run start, so every section of the run, the H1 pass included, reads the same value, and it keeps that value across the hand-off from the H1 pass to the section walk, which is why the assertion holds. `sys.when` is the only time field in `sys`, so a prompt reads the run's time through `sys.when`, and `sys` has no field that reads the current time.

`sys.when` is the start instant the Harness supplies when it creates the run, not a live clock, so it never advances while the run executes and reads as the identical string on every read. It does not depend on the run [seed](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host): runs with different seeds but the same start instant read the same `sys.when`, even though the Harness supplies the seed and the start instant together. A replay that passes back the run's recorded start instant reproduces the same `sys.when`.

### How `sys.when` is written

The `sys.when` text has the UTC shape `YYYY-MM-DDTHH:MM:SS[.fff]Z`: every field is zero-padded (a four-digit year and a two-digit month, day, hour, minute, and second), an uppercase `T` sits between the date and the time, the fraction is optional, and a trailing `Z` stands in place of any offset. The fraction follows the start instant's milliseconds:

| Start instant | `sys.when` |
|---|---|
| the Unix epoch | `1970-01-01T00:00:00Z` |
| a whole second | `2024-02-29T12:34:56Z` |
| 789 ms past the second | `2024-02-29T12:34:56.789Z` |
| 780 ms past the second | `2024-02-29T12:34:56.78Z` |
| 700 ms past the second | `2024-02-29T12:34:56.7Z` |
| 50 ms past the second | `2024-02-29T12:34:56.05Z` |
| 5 ms past the second | `2024-02-29T12:34:56.005Z` |
| one millisecond before the epoch | `1969-12-31T23:59:59.999Z` |

When the run starts on a whole second, `sys.when` has no fractional part. Otherwise it includes a millisecond fraction with trailing zeros dropped, giving one, two, or three digits, and never more than three. Leading zeros inside the fraction stay, so 5 ms past the second renders as `.005` and 50 ms as `.05`.

The rest of the format is just as fixed:

- The time of day is UTC on a 24-hour clock, with hours `00` through `23`.
- The text always ends with `Z` and never has a numeric offset such as `+00:00` or a local time zone.
- Dates follow the proleptic Gregorian calendar in UTC with correct leap years, including the century rules, so leap days such as `2024-02-29` appear correctly, 1900 and 2100 are not leap years, and 2000 is.
- A start instant before 1970 reads as an ordinary earlier calendar date, as the last row of the table shows.
- The year is four zero-padded digits for start instants from year 0000 through 9999, the last being `9999-12-31T23:59:59.999Z`.
- The text matches a standard RFC 3339 rendering byte for byte, so you can hand it to any standard RFC 3339 parser.

`sys.when` is a string, not a millisecond count, so when you need date parts or date arithmetic you parse the text. Because the layout is fixed, `string.sub(sys.when, 1, 10)` gives the date, `2000-02-29` in the `started` run.

Comparing two `sys.when` strings as text does not reliably order them in time. The fraction varies in length and is omitted on whole seconds, while `.` and digits sort before `Z`, so `12:34:56Z` sorts after the later `12:34:56.5Z`, and `.1Z` (100 ms) sorts after the later `.15Z` (150 ms).

### The bound model in `sys.model`

````markdown
---
name: model-name
description: Reads the bound model's name before and after the first tool call.
runvil: 0
uses:
  - user-input
models:
  writer: {}
---

# Model name

```lua
models.default('writer')
```

## Main

```lua
local ok, model = pcall(function() return sys.model end)
var.before = ok and model or 'no tool call yet'
tools.call('user-input/ask')
```

Model name is {{ sys.model }}.

```lua
return var.before .. ' / ' .. prose
```
````

The H1 block makes `writer` the prompt-wide default (see [The prompt-wide default with `models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault)), so `## Main` has a model. In a run where prepare binds `writer` to the model `gateway/claude-sonnet-4-6`, once the operator answers the ask, the run result is:

````text
no tool call yet / Model name is claude-sonnet-4-6.
````

`sys.model` is a string holding the name part of the bound model's model id, without the server namespace, so it reads `fast` for a model whose id is `gateway/fast`, and `claude-sonnet-4-6` for `writer` above, bound to `gateway/claude-sonnet-4-6`. It is the same name that role's model handle reads as `model_id` (see [Model handles](10-models.md#model-handles)), never the alias or role label the prompt uses for that model. It becomes readable only after the section entry's first tool call, and only when the entry has a model. Here the first tool call is the [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls) to the ask tool from [Asking the operator](#asking-the-operator), so the `pcall` before it finds no field and the placeholder after it renders the name.

At the section entry's first tool call, `sys.model` takes the entry's model, the model the entry selected for itself or else the prompt-wide default, and stays fixed for the rest of the section entry. A section entered again with `jump`, a fanout arm, a task, and a section run with `call` each start without the field and fill it at their own first tool call. From that call on, later `{{ sys.model }}` prose placeholders and later Lua blocks alike read that same name, so a later `return sys.model` returns `claude-sonnet-4-6` too.

`sys.model` appears at the section entry's first call to a catalog tool or an adhoc tool, whether a Lua block makes it with `tools.call` or the model makes it during a conversation, such as a tool call inside [`models.loop`](11-conversations.md#starting-a-conversation). A model round, one request to a model and its reply, never sets `sys.model` by itself, so a [`models.infer`](10-models.md#running-a-round-with-modelsinfer) or `models.loop` round with no tool call leaves it unreadable. A section entry that makes no tool call, or has no model when its first tool call runs, never gets the field.

Reading `sys.model` before the section entry's first tool call raises an error whose message reads `unknown sys field 'model'`, even when the H1 pass set a prompt-wide default such as `models.default('writer')`. That holds whether the read is in the section's opening Lua block or inside a function from a library the section loads with [`require`](03-blocks-and-prose.md#sharing-code-with-require), as in `require '## sys_reader'` followed by `return read_sys_model()`.

Inside a fanout arm, once the arm's first tool call has been dispatched, `sys.model` reads the name part of the selected model's id in the same way, so an arm for the member `a` that returns `sys.model .. ':' .. item` gives `claude-sonnet-4-6:a`.

### The Host's state in `ui()`

````markdown
---
name: host-model
description: Reads the Host-state snapshot when the run has one.
runvil: 0
---

# Host model

## Main

```lua
if ui then
  local state = ui()
  return state.selected_model .. '/' .. tostring(state.workspace_root)
end
return 'no host state'
```
````

Started with the snapshot `{ "selected_model": "m-1", "workspace_root": null }`, the run result is `m-1/nil`. Started without a snapshot, it is `no host state`.

The `ui` snapshot is the application state the Host hands to the run at run start, when the run has one. You call `ui()` as a global function with no arguments to get it as a Lua table: `ui` is called, not indexed as a table. When the Host supplies a snapshot, every section in the run gets a `ui()` global that returns that same snapshot.

In a run started without a snapshot, the `ui` global is absent from every section rather than stubbed. `type(ui)` is `"nil"`, an unguarded `ui()` call fails as a call to a nil value, and so a block tests for `ui` before calling it, as in `if ui then` above or `ui and ui().selected_model`.

`ui()` returns the snapshot as a Lua table whose fields are the snapshot's JSON fields as Lua values, so for the snapshot above, `ui().selected_model` reads `m-1`. A JSON null field reads as Lua `nil`, the same as an absent field, and never as a NULL sentinel userdata, which is why `ui().selected_model .. '/' .. tostring(ui().workspace_root)` gives `m-1/nil`. `json.decode` and structured tool results do give that sentinel, [`json.null`](#null-values-and-jsonnull), so their null fields are tested with `v == json.null` rather than `v == nil`.

Each `ui()` call builds a fresh table, so changes you make to one returned table never show up in the next call. The snapshot is taken once at run start and stays fixed for the run, so every call in every section returns the same Host state, and a change to application state shows up in `ui()` only on the next run. Which fields the snapshot holds is up to the Host. `ui` is installed before any library loads, so library code can call it too.

## Section entry ids

After this section you can predict the `sys.id` of any section entry, on the main walk, inside a `call`, and inside a task, and use it as a unique key.

````markdown
---
name: numbered
description: Reads section entry ids along the main walk.
runvil: 0
---

# Numbered

```lua
assert(sys.id == '0.0')
```

## First

```lua
assert(sys.id == '0.1')
```

## Second

```lua
return sys.id
```
````

````text
0.2
````

A section entry id names one entry into a section. `sys.id` is a string that holds the id of the current section entry, written as a dotted path such as `0.1`, `0.0.0`, or `0.3.0`. Each section entry reads its own `sys.id`, made of the id of the chain that entered it, a dot, and that chain's zero-based entry counter.

The H1 pass and the walk share one root chain, `0`. The H1 Lua reads `sys.id` as `'0.0'`, the walk continues the same entry counter, and the first walked section reads `'0.1'`. Entry `0.0` stays reserved for the H1 pass even when the prompt has no H1 blocks, so sections on the main walk always read `sys.id` as `0.1`, `0.2`, and so on. Without its H1 block, the `numbered` prompt still returns `0.2`.

`sys.id` compares equal to a plain Lua string, so you test it with `==` against a string literal such as `'0.1'`, as the assertions above do. `tostring(sys.id)` gives the same text as `sys.id`, because the id is already a string, so a block can return or store either form, and `return tostring(sys.id)` in the second walked section gives `0.2` too.

### The id format

A section entry id is its chain's id followed by a dot and the entry's index, so entries 0 and 7 of chain `0.3` read `0.3.0` and `0.3.7`, and the first section entered by chain `0.3` reads `0.3.0`. Section entries on the main walk read `0.N` because the main walk is chain `0`.

Every section entry gets a fresh `sys.id`, so a section entered more than once in the same chain, including through `jump`, reads a new id each time: the id counts entries, not sections. `sys.id` counts up by one for each section the walk chain enters, and a `jump` into a child level stays in the same chain, so the count continues rather than restarting. Across `## A`, `### X`, `### Y`, and `## B`, entered in that order, the ids are `0.1`, `0.2`, `0.3`, and `0.4`.

### Ids inside a `call`

````markdown
---
name: called-ids
description: Reads section entry ids inside called chains.
runvil: 0
---

# Called ids

## Main

```lua
local a = call('### Sub')
local b = call('### Sub')
var.ids = table.concat({ sys.id, a, b }, ',')
```

### Sub

```lua
return sys.id
```

## Last

```lua
return var.ids .. ',' .. sys.id
```
````

````text
0.1,0.0.0,0.1.0,0.2
````

Each `call` starts a contained child chain nested under the caller's chain. The first `call` from `## Main` is the root chain's first child, so it opens chain `0.0`, and its section entries read `0.0.N`, starting with `0.0.0` for the called section. A section run by `call` or as a fanout arm is the first entry of its new chain, so its `sys.id` ends in `.0`.

Each `call` starts a new child chain numbered in call order, so entering the same section through two calls gives two distinct `sys.id` values, `0.0.0` and `0.1.0`. After `call` returns, the caller's `sys.id` is unchanged, so `## Main` still reads `0.1` after both calls, and the walk goes on to `## Last` as `0.2`.

Call chains are numbered as children of the calling chain, not of the calling section's entry. The two calls from `## Main`, itself entry `0.1`, start chains `0.0` and `0.1`, so the second chain's section reads `0.1.0` even though the calling section is also `0.1`. Child chains are numbered by the calling chain's own child counter in the order they start, and calls, tasks, fanout arms, and each [timed wait](15-tasks.md#the-timer-behind-a-time-limit)'s internal timer all share that counter no matter which section made the call, so each new call or arm takes the next child chain number.

A chain started by `call` numbers its own section entries. With this `## Main` and its children in place of the ones in the `called-ids` prompt, the run result is `0.1,0.0.1,0.2`:

````markdown
## Main

```lua
var.ids = sys.id .. ',' .. call('### Sub')
```

### Sub

```lua
assert(sys.id == '0.0.0')
```

### Tail

```lua
return sys.id
```
````

The called section reads `0.0.0`, the fall-through inside that chain takes the next entry, `0.0.1`, and after the chain ends the outer walk resumes its own sequence, so the next walked section reads `0.2` no matter how many entries the chain used.

A `call` from the H1 pass takes the root chain's first child slot, so the called section runs as `0.0.0`. A later `call` from the first walked section then runs as `0.1.0`, and that calling section still reads `sys.id == '0.1'`.

### Ids inside fanout arms

````markdown
---
name: mixed-ids
description: Reads section entry ids from calls and fanout arms.
runvil: 0
---

# Mixed ids

## Main

```lua
local a = call('### Sub')
local r = tasks.fanout('### Worker', {'x', 'y', 'z'})
local b = call('### Sub')
var.ids = table.concat({ sys.id, a, r[1].text, r[2].text, r[3].text, b }, ',')
```

### Sub

```lua
return sys.id
```

### Worker

```lua
return sys.id
```

## Last

```lua
return var.ids .. ',' .. sys.id
```
````

````text
0.1,0.0.0,0.1.0,0.2.0,0.3.0,0.4.0,0.2
````

Each fanout arm is a new child chain of the caller's chain, numbered by the caller's next child slot, so its worker section reads `sys.id` as that chain id followed by `.0`. Here the first `call` took `0.0`, so the three arms read `0.1.0`, `0.2.0`, and `0.3.0`, and the second `call` takes the next slot, `0.4.0`. With no earlier children, three arms from the main walk read `0.0.0`, `0.1.0`, and `0.2.0`. Fanout arms take their ids in collection order when the fanout dispatches, so the ids never depend on which arm finishes first.

`tasks.fanout`, alone or followed by a `call`, leaves the calling section entry's `sys.id` unchanged, so `## Main` still reads `0.1` after its arms return. Dispatching calls and fanout arms does not advance the walk's own entry count either, so after `## Main` makes two calls and a three-arm fanout, `## Last` still reads `0.2`.

Ids nest the same way at every depth:

| Where the section entry is | `sys.id` |
|---|---|
| the first section a `jump` reaches after the worker, in the first arm | `0.0.1` |
| the first section of a `call` made inside arm chain `0.0` | `0.0.0.0` |
| the two arms of `tasks.fanout('## Worker', {'a', 'b'})` inside a called `## Sub` (chain `0.0`) | `0.0.0.0` and `0.0.1.0` |
| the inner arms of a fanout inside outer arm `0.0`, whose worker reads `0.0.0` | `0.0.0.0` and `0.0.1.0` |

Sections that a `jump` reaches from a fanout arm continue the arm chain's `sys.id` sequence. A `call` made inside an arm starts a contained chain that is the arm's child, so with the arm at chain `0.0` the called chain is `0.0.0`. A fanout inside a called section nests its arm ids under the called section's chain, not under the root chain, and a fanout inside a fanout arm nests the inner arms under the outer arm's chain.

### Ids inside a task

````markdown
---
name: task-ids
description: Reads the task id, section entry id, and index inside each arm.
runvil: 0
---

# Task ids

## Main

```lua
local r = tasks.fanout('### Worker', {'x', 'y'})
return r[1].text .. ' | ' .. r[2].text .. ' | ' .. sys.id
```

### Worker

```lua
return sys.taskid .. ' ' .. sys.id .. ' ' .. sys.index
```
````

````text
0.0 0.0.0 1 | 0.1 0.1.0 2 | 0.1
````

Each arm is a task. Inside a spawned task, `sys.id` is a section entry id that extends the task id, while `sys.taskid` is the task id itself, so the first section of task `0.0` reads `sys.taskid` as `0.0` and `sys.id` as `0.0.0`. A task started with [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) follows the same rule.

A `sys.id` can have the same text as a task id, because a chain counts its section entries and its children separately. Here `## Main` reads `sys.id` as `0.1` while its second arm is task `0.1`, and `0.0` can be both the H1 body's `sys.id` and the first task's id. Task ids are reproducible in the same way as `sys.id`, because they come from the same chain-local child counters, so the arm tasks `0.0` and `0.1` keep their ids whichever arm finishes first.

### Using `sys.id` as a key

Every `sys.id` in a run is distinct. Entries in one chain differ in their counter, and entries in different chains differ in their chain path, as with `0.3` against `0.3.0`. `sys.id` values are also the same on every run of the same prompt with the same inputs, however chains interleave or tasks finish, because each id is a path of chain-local counters and no counter spans the whole run.

That makes `sys.id` a unique key per entry, for example as part of a name you build. `sys.id` is a plain string, so you join it into a name with `..`, and this line gives each section entry its own store path (see [Writing and reading files](09-the-store.md#writing-and-reading-files)):

````lua
store.write('park-' .. sys.id, 'x')
````

## Logging and lifecycle reports

After this section you can record progress checkpoints with `log`, keep within its message rules and quotas, and read the lifecycle reports a run emits for your Lua.

````markdown
---
name: progress
description: Records checkpoints while it works.
runvil: 0
---

# Progress

## Prepare

```lua
log('prepare started')
var.subject = 'tides'
log('prepare finished')
```

## Finish

```lua
log('finish started')
return var.subject
```
````

The run result is `tides`, and the run records three checkpoints, in this order:

````text
Lua: prepare started
Lua: prepare finished
Lua: finish started
````

A log checkpoint is a short progress message your Lua records by calling `log(message)`. It appears in the run's event stream as a `lua` checkpoint event, and `log` itself returns nothing. Each checkpoint renders as the text `Lua: ` followed by the message and holds the message text exactly as you passed it to `log`. The first two checkpoints above are attributed to `Prepare` and the third to `Finish`.

`log` accepts exactly one argument: a single-line UTF-8 string of at most 256 Unicode characters with no newline or control characters. [Message rules](#message-rules) gives the details.

### Where you can call `log`

`log` is an Engine global, always present in every section VM, the H1 VM included. You can call `log(message)` to record checkpoints from any Lua block: in any section, in a fanout arm, in the H1 pass, and in prologues and epilogs alike.

`log` stays defined for the section entry's whole lifecycle, so every block in the entry can call it. It is an ordinary function value that you can assign to a variable and call through that variable, as in `saved_log = log`, and a saved reference keeps working in later blocks of the same entry (see [A fresh section VM for every section entry](#a-fresh-section-vm-for-every-section-entry)). A library section's top-level code can call `log` while [`require`](03-blocks-and-prose.md#sharing-code-with-require) loads it, because section setup installs `log` before it installs `require`.

No section VM has `print` or `warn` (see [Names that read nil](#names-that-read-nil)), whether in a library section's code, a prologue, an epilog, or a lone block, so `assert(print == nil)` and `assert(warn == nil)` pass everywhere, and `log` is the only way your Lua reports progress.

### Which section a checkpoint names

````markdown
---
name: log-checkpoints
description: Records checkpoints across the H1 pass, a prologue, and an epilog.
runvil: 0
---

# Log Checkpoints

```lua
log('H1 ran')
```

## Prepare

```lua
log('prepare started')
```

Gather the notes.

```lua
log('prepare finished')
```

## Finish

```lua
log('finish started')
return 'finished'
```

This prose is never read.

```lua
log('unreachable epilog')
```
````

The run result is `finished`, and the checkpoints come out in this order:

````text
Lua: H1 ran
Lua: prepare started
Lua: prepare finished
Lua: finish started
````

Each `log` checkpoint is tagged with, and attributed to, the section name of the section whose Lua called `log`, and it is also tagged with the run's execution id. A `log` call in the H1 body records its checkpoint under the prompt's title, ahead of every section's checkpoints, so `H1 ran` is recorded under `Log Checkpoints`. The next two are recorded under `Prepare`, and `finish started` under `Finish`.

A checkpoint is attributed to the section whose code called `log`, including a section reached through `call`: when `## Outer` runs `call('## Inner')`, `log('inner ran')` in `## Inner` is attributed to `Inner`. Calling `log` inside a task's section records the message as a checkpoint under that section's name, and `log` records each message under the name of the section that logs it, in order, including inside a spawned child. When several runs execute at the same time, each run's checkpoints keep that run's execution id and appear in that run's call order.

### Checkpoint order

The run records each `log` message verbatim, attributed to the section that called it, in call order across blocks. Several `log` calls in one section appear in the trace in the order the Lua made them, and checkpoints from separate Lua blocks in the same section appear in the order the blocks run, as `prepare started` and `prepare finished` do above.

Checkpoints come out in a deterministic order across the H1 pass, prologues, and epilogs, and code that never runs logs nothing. In `## Finish`, the prologue returns, so the epilog never runs and `unreachable epilog` is never recorded. Checkpoints logged before an early return, such as `finish started`, still appear in the trace.

`log` emits its checkpoint at the moment of the call, through the same channel the runtime's own reports use, so checkpoints sit in call order among chunk, store, and teardown reports. `log` checkpoints, message text included, interleave with Engine operation reports such as [store](09-the-store.md#writing-and-reading-files) reports in program order:

````lua
log('before write')
store.write('state.txt', 'value')
log('after write')
````

This block records the `before write` checkpoint, then the write's store report, then the `after write` checkpoint, so the two `log` calls bracket the write's report.

Logging never changes a prompt's results. A block's return value, the `var` contents, and the store's contents are the same whether or not anything observes the checkpoints, so you can add `log` calls freely.

A `log` message is recorded verbatim with no redaction, so keep it to your own status wording, and leave out arguments, replies, tool data, credentials, paths, and store contents.

### Message rules

A `log` message is one string argument that holds valid UTF-8, has at most 256 characters, and fits on a single line. `log` does not convert numbers to text, so pass `tostring(n)` for a number:

````lua
local n = 3
log('items: ' .. tostring(n))
````

A call that breaks a message rule raises a Lua error in the block naming the broken rule, and the call itself records no checkpoint:

| Rule | Message when the rule is broken |
|---|---|
| exactly one argument | `log expects exactly one argument` |
| a string of valid UTF-8 | `log message must be a UTF-8 string` |
| at most 256 characters | `log message must be at most 256 characters` |
| one line, no control characters | `log message must not contain newline or control characters` |

The rules in detail:

- Calling `log` with no arguments, or with two or more, raises `log expects exactly one argument`.
- Passing a non-string value, such as a number, boolean, table, or `nil`, raises `log message must be a UTF-8 string`. A Lua string that holds invalid UTF-8 bytes, such as `string.char(255)`, raises the same error that a number does.
- A message holds up to 256 characters, counted as Unicode characters rather than bytes, so multibyte text up to 256 characters is accepted: 256 copies of `é`, which take 512 bytes, pass. A message at the 256-character maximum is recorded in full, without truncation, and a longer one, such as 257 copies of `é`, raises `log message must be at most 256 characters`.
- A message is a single line of text with no control characters, so it holds no newline, carriage return, or tab, and neither U+2028 LINE SEPARATOR nor U+2029 PARAGRAPH SEPARATOR. A message containing any of them raises `log message must not contain newline or control characters`.

When one `log` call breaks several rules, you see the first failing check in this order: argument count, event quota, string type, UTF-8 validity, the 256-character limit, control characters, then the byte quota. The two quota checks are the subject of [Log quotas](#log-quotas). A call without exactly one argument fails with `log expects exactly one argument` and spends no quota, while every one-argument call spends one unit of the log event quota before its type, length, and line-break rules are checked, so a one-argument call that breaks a rule has still spent its unit.

### Log quotas

The log quotas are the two limits each section VM puts on `log`: the log event quota, which counts calls, and the log byte quota, which counts the UTF-8 bytes of the recorded messages.

Each `log` call spends one event from the log event quota, so with a quota of 1, `log('one')` followed by `log('two')` fails on the second call. The log event quota caps the `log` calls in each Lua VM separately, at 1024 by default, and the Harness can set another count.

All `log` calls in a section VM share a cumulative log byte quota of 256 bytes per allowed checkpoint, so the byte quota follows the event quota: 262,144 bytes by default, and 1024 bytes with a 4-event quota. Each message that passes the other checks spends its length in UTF-8 bytes from that shared quota, so multibyte text uses it up faster, and a 200-character message of two-byte characters such as `é` costs 400 bytes. That way many small messages cannot emit unbounded total log volume.

Both quotas span every block in the section entry rather than resetting per block. Every `log` call counts against the section VM's log quotas, including calls a library section makes while `require` loads it, because a library loads into the requiring section entry's one environment. The next section entry starts with full `log` quotas.

A call past either quota raises one of these messages:

````text
lua log event budget exceeded
lua log cumulative byte budget exceeded
````

The first comes from a call past the log event quota and the second from a call past the log byte quota. The byte quota can refuse a call even while events remain in the event quota. A message whose UTF-8 byte length would overflow the byte quota is refused whole, and messages logged before it stay recorded. A refused call leaves the remaining byte quota unchanged, so a later shorter message that fits can still be logged, though the refused call has already spent its event unit.

With a log event quota of 4 set by the Harness, the log byte quota is 1024 bytes, and this block fails on its third call:

````lua
log(string.rep('é', 200))
log(string.rep('é', 200))
log(string.rep('é', 200))
return 'unreached'
````

Each call costs 400 bytes, so exactly two checkpoints are recorded. A `log` call refused by the byte quota stops the block, so the `return` never runs, and the uncaught refusal ends the run with run error kind `Quota`. That holds for both quotas: an uncaught `log` quota refusal, event or byte, ends the run as `Quota`.

`pcall` catches either refusal as an error value with error kind `lua` whose `message` is the refusal text, such as `lua log cumulative byte budget exceeded`. Only the Engine's own refusal counts as a quota refusal, so a block that raises the same text itself, as in `error('lua log event budget exceeded')`, fails as an ordinary Lua runtime error.

### Lifecycle reports

A lifecycle report is a run report that shows one stage of a section entry's Lua lifecycle. Lifecycle reports hold the section name and never any values, such as a shared global's value or the text the run was given.

For one section with two blocks, the lifecycle reports come in this order:

1. chunk started, for the first block
2. chunk succeeded, for the first block
3. chunk started, for the second block
4. chunk succeeded, for the second block
5. teardown started
6. teardown succeeded

Each Lua block's `log` checkpoints sit between that block's chunk started and chunk succeeded reports, and so do its store reports: a block reports chunk started before its store reports, and chunk succeeded after them when it finishes cleanly. In the `log-checkpoints` prompt, `prepare started` falls inside the first block's pair in `Prepare` and `prepare finished` inside the second.

When a section entry's Lua VM shuts down, the event stream shows teardown started followed by teardown succeeded, tagged with that section. A block that raises reports chunk started and then chunk failed, and the entry's teardown still reports started and succeeded. When a section VM's setup fails after the VM is built, for example when a [Pack prelude](#lua-blocks-section-vms-and-engine-globals) fails to load, the VM is torn down, reporting teardown started and teardown succeeded, before the error is reported.

Two more kinds of lifecycle report come from compiling and loading:

- When a prompt is parsed, each Lua block's compilation reports a started report and then exactly one succeeded or failed report, each tagged with the block's section name and holding no source text or [location label](#error-line-numbers-and-locations).
- The first `require` of a library section reports a library load started report and then a succeeded or failed one, naming the library's heading under the requiring section. A repeat `require` of a loaded library reports nothing, and each retry of a failed load reports a fresh pair (see [Library errors, cycles, and load events](03-blocks-and-prose.md#library-errors-cycles-and-load-events)).

## Asking the operator

After this section you can pause a section until the operator answers, let a model ask the operator instead, and handle an ask that fails.

````markdown
---
name: ask-once
description: Waits for the operator's next message and returns it.
runvil: 0
uses:
  - user-input
---

# Ask once

## Main

```lua
local text = input.ask()
return text
```
````

When the operator sends `kept`, the run result is:

````text
kept
````

The operator is the person who answers during a run. The `runvil/user-input` Pack lets a prompt ask the operator: a prompt that declares it in its frontmatter runs the Pack's Lua prelude, which defines an `input` global table in every section entry, so any section can call `input.ask()` to wait for the operator's next message. The declaration also brings the ask tool, `user-input/ask`.

`input.ask()` takes no arguments, as in `local text = input.ask()`. It pauses the section until the operator's next message and returns that message's text as a string, byte for byte as the operator sent it. Leading and trailing spaces, newlines, and tabs are kept, with no trimming or re-encoding, so a message with two leading spaces, a newline and a tab inside it, and two trailing spaces comes back exactly as typed. The operator's text is trusted, so it never arrives wrapped in the untrusted `<untrusted_input_...>` markers.

### Declaring the Pack

You get operator input in a prompt by declaring the Pack under the frontmatter [`uses:`](12-tools.md#tools-packs-and-the-uses-key) key by its Pack id, `user-input`. Write it as the line `  - user-input` after `uses:`, as the `ask-once` prompt does, or in flow style as `uses: [user-input]`. A bare entry always means the `runvil` Pack of that id, so `uses: [runvil/user-input]` declares the same Pack.

The Pack's fully qualified name is `runvil/user-input`, so its Pack id is the second segment, `user-input`.

Declaring `user-input` gives your Lua the `input` table, one of the globals that [Pack preludes](#pack-preludes) define. Its `ask` field is a function, so `type(input.ask)` returns `function`. The `input` table is plain Lua built by the Pack's prelude, and `ask` is its only function.

The `input` table is sealed like every prelude table (see [Pack preludes](#pack-preludes)). A section can read and call `input.ask`, but writing any field of `input` raises ``input is read-only: Pack `user-input` defines it; cannot set '{field}'``, with the field's name in place of `{field}`, and `pairs(input)` lists nothing.

A prompt that does not declare `user-input` has no `input` global, even when the Host has the Pack, so calling `input.ask()` there fails with Lua's own error `attempt to index a nil value (global 'input')` (see [Pack preludes](#pack-preludes)).

Your own Lua can call `input.ask()` in a section that never calls `tools.offer`, because offering controls only what the model sees, and your code is not the model.

### The input broker

An input broker is the per-run service that delivers a question to the operator, and the Pack names it as the service `runvil/input-broker`. A run can have an optional input broker, and the run lends it to every tool call, so an ask tool can reach the operator.

Every Pack a prompt declares is required, so a prompt that declares `user-input` needs a Host that can ask an operator. The Pack needs the environment to provide an input broker, so on a Host with nobody to ask, the prompt is refused before it starts, with run error kind `RequirementsUnmet`. The [requirements notice](04-how-a-prompt-runs.md#the-requirements-notice) names the missing service by its id:

````text
the environment cannot satisfy this prompt:
- user-input needs runvil/input-broker, and the environment provides none
````

The ask tool is visible only in a run that has an operator input broker, because the Pack's declared needs keep a run without one from seeing the tool, and a Pack missing a service has its tools left out of the tool catalog. The refusal also gives you a guarantee: a section of a prompt that declares `user-input` always has an operator to ask, so it needs no connection check before calling `input.ask()`.

`input.ask()` names no recipient, run, or section. The ask tool reads the input broker of the run that makes the call, and the Host ties that broker to whoever launched the run, so the Host decides where an `input.ask()` question goes. One Pack object is shared by every run, while the input broker arrives with each run, so each ask reads the next message from the operator of the run that makes the call, and reads the reply from the run it is called in.

### What `input.ask()` takes

`input.ask()` takes no arguments. Passing any argument raises a plain Lua error with the message `input.ask takes no arguments`, which `pcall` catches, so a prompt that passes a question fails loudly instead of losing it. Passing an explicit `nil` still counts as an argument and raises the same error. The error is raised at level 2, so it is reported against your own line that called `input.ask` rather than against the prelude.

### While the section waits

The section entry's Lua VM state survives an `input.ask()` wait, however long the operator takes. With this block as the Lua block of `## Main` in the `ask-once` prompt, an operator who sends `kept` gets the run result `41 kept`:

````lua
local before = 41
local text = input.ask()
return tostring(before) .. ' ' .. text
````

An `input.ask()` wait pauses only the calling chain, and the rest of the run keeps going while it waits, because `input.ask()` is a [suspending call](#suspending-calls). Every tool call is Harness work, a wait for operator input included, because `input.ask()` asks the operator for text through a tool call and its chain waits for the answer.

Cancelling the run while a section waits in `input.ask()` drops the pending wait, closes any prompt it opened for the operator, and stops the run promptly with the cancelled run outcome, so the code after the call never runs.

A stop of the run is different. The ask tool is declared to survive a stop, so a pending `input.ask()`, and the ask tool a model calls, stay open through a stop and keep waiting for the operator's answer, and a person in the middle of typing never loses the question. A stop leaves the operator's open question in place instead of dropping it. Every other occasion that ends a pending tool call drops the open question, including a cancel, the end of the run, and the early end of the task that asked.

### The ask tool

The Pack provides exactly one tool, the ask tool, whose tool name is `ask`. Its tool id is the Pack id, then a slash, then `ask`, so it is `user-input/ask`. `tools.get('user-input/ask').id` reads the id back as `user-input/ask`.

Each `input.ask()` is an ordinary tool call to the ask tool, made through [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls) with the tool's full id, so calling the tool yourself asks the operator the same way. With this block as the Lua block of `## Main` in the `ask-once` prompt, the run result is again the operator's text:

````lua
return tools.call('user-input/ask')
````

The tool asks the operator when called and returns the text they type, so `tools.call(tools.get('user-input/ask'))` returns the operator's text as well. The prelude receives the Pack id and builds the tool id from it, so `input.ask()` reaches `user-input/ask` without the prelude spelling the Pack id out.

Each `input.ask()` is one tool call from Lua, reported like any tool call a Lua block makes. The run records a `ToolCall` for the tool `user-input/ask` with the script as the caller and the tool's id, `user-input/ask`, as the alias, never `user-input_ask`, the name a model sees for it (see [How tool calls are recorded](12-tools.md#how-tool-calls-are-recorded)). The result reports as trusted with the operator's text as its content.

The ask tool takes no arguments. Its parameters are described as an object with no properties, `{ "type": "object", "properties": {} }`, and any arguments passed to it are ignored. It returns plain text rather than structured output. Its description, shown to any model it is offered to and wherever Lua reads the tool, reads `Wait for the operator's next message and return its text.`

### Letting the model ask

Declaring `user-input` offers nothing to the model, because a [`models.loop`](11-conversations.md#starting-a-conversation) conversation offers the model only tools from the Packs and adhoc tools the section offers, so unless the prompt opts in, only Lua asks the operator. Neither the Host having the Pack nor the prompt declaring it offers the ask tool to the model, and the model can ask the operator only when the prompt's Lua opts in by offering it.

To let the model ask, offer the ask tool in the section that runs the model, either by its id or as the whole Pack (see [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer)):

````lua
tools.offer('user-input/ask')
````

````lua
tools.offer('user-input')
````

The model sees the offered ask tool under the name `user-input_ask`, not under its id `user-input/ask`, because the `user-input` Pack neither names the tool nor renders it, so its model-facing name is its id with `/` and `.` replaced by `_`. It sees the description `Wait for the operator's next message and return its text.` The model calls the tool with an empty arguments object, `{}`, the call waits until the operator sends their next message, and the tool returns that message's text as its result, in plain text. The operator's answer comes back as trusted tool output, so the text is never wrapped or marked untrusted.

### When an ask fails

````markdown
---
name: ask-safely
description: Catches a failed ask and finishes the section.
runvil: 0
uses:
  - user-input
---

# Ask safely

## Main

```lua
local ok, result = pcall(input.ask)
if ok then
  return result
end
return result.kind .. ': ' .. tostring(result)
```
````

When the Host's input broker fails the wait with the message `the operator's window closed`, the run result is:

````text
tool: tool call failure: the operator's window closed
````

When the input broker fails a wait, `input.ask()` raises at its call site an [error value](#error-values) of error kind `tool` whose `tostring` reads `tool call failure: {message}`, with the broker's failure text in place of `{message}`. So `local ok, result = pcall(input.ask)` returns `false` and the error value instead of the run ending, and the section keeps running to completion. The error value's `message` field holds the same text that `tostring` renders: `tool call failure: ` followed by the broker's message. Whatever kind of failure the broker reports, a failed ask reaches Lua as an error value of error kind `tool`.

The broker's failure message reaches the prompt: a caught error value holds it, and when the model made the call, the model reads it as the tool's result, wrapped as untrusted text. A failed ask's text holds the broker's own failure message, such as `the operator's window closed`, while the broker's underlying cause stays behind it and is never added to the text, so `tostring` on the error value includes the operator-side failure message but never that hidden cause.

Left uncaught, a failed `input.ask()` ends the run as failed with run error kind `Tool`, while caught with `pcall` the same failure is an error value of error kind `tool`.

## Error values

After this section you can read a caught failure's kind and message and branch on them.

````markdown
---
name: read-error
description: Catches an error value and reports its kind and message.
runvil: 0
---

# Read error

## Probe

```lua
local ok, err = pcall(function() return sys.nothing end)
return type(err) .. '|' .. err.kind .. '|' .. tostring(err)
```
````

The run result is:

````text
table|lua|runtime error: unknown sys field 'nothing'
````

[`sys`](#run-facts-in-sys-and-ui) has no field named `nothing`, so the read fails, and `pcall` catches the failure instead of letting it end the block (see [Raising and catching errors](#raising-and-catching-errors)). What comes back is an error value: a Lua table holding a `kind` string, a `message` string, and any extra fields its kind defines. Every failure that an Engine call or an Engine function raises reaches your code as one, so `type(err)` is `table` and never a plain string, whether the failure came from a model round, a store operation, or a guard like the one on `sys`.

The error kind is the value's `kind` field, a lowercase snake_case string such as `lua`, `out_of_scope_tool`, or `empty_model_reply`. It names which failure occurred, so you read it at any call site and branch on it.

An Engine function can fail in its own Lua code or inside the Engine itself, and both give the same error value, with the same shape and the same metatable, so your code never needs to tell the two apart.

### Reading `kind` and `message`

Read both fields with ordinary table access:

````lua
local ok, err = pcall(function() return sys.nothing end)
if not ok and err.kind == 'lua' then
  log('lookup failed: ' .. err.message)
end
````

`err.message` is always a string and holds the failure's display text. When whatever raised the error gave no message, the message is the `kind` string itself, so `tostring(err)` shows the kind.

`tostring(err)` returns exactly the message, with no traceback and no position prefix, so you can print it, join it into other text, or search it with string functions, as in `tostring(err):find('not found', 1, true)`. The `..` operator gives the same text with the error value on either side, so `'context: ' .. err` and `err .. ' suffix'` each join the message with the other operand. `tostring` gives the message in the same way for the value that a failed `require` or a refused `setmetatable` call hands back.

The message is the root cause's own text. It names the error the Engine function itself returned, not a callback frame or added context wrapped around it, and it never carries the stack traceback that the Lua runtime would otherwise add to an error from an Engine function. The Engine raises its errors at level 0 at the call site, the way `error(result, 0)` does, so no `file:line:` position prefix is added and the first line is exactly the Engine's text: a model round that fails with `model is down` gives a message whose first line is `model is down`.

One family of messages starts with a prefix. A refusal that a Runvil guard raises as a plain runtime error, such as reading a `sys` field that does not exist or assigning to `sys`, is caught as an error value of kind `lua` whose message starts with `runtime error: ` and then gives the refusal. Reading `sys.nothing` gives `runtime error: unknown sys field 'nothing'`, and the refusal of `sys.extra = 1` reads `sys is read-only; cannot set 'extra'` after the same prefix. A failure from an Engine call, such as [`models.infer`](10-models.md#running-a-round-with-modelsinfer), has no such prefix.

Messages you meet through `tostring(err)` include `models.infer takes (prompt)`, `handle:infer takes (prompt)`, `lua log event budget exceeded`, `the model replied with no text and no tool calls`, `empty model reply`, ``section heading `## Missing` not found``, `call recursion exceeded cap of 8`, and `argv is frozen outside H1: assign it in H1 only`.

Because `kind` is a string and `tostring` gives the message, one expression can report a failure in full:

- `err.kind .. '|' .. err.name .. '|' .. tostring(err)`, after a model calls a tool that is not offered in its section entry, gives text that starts `out_of_scope_tool|rogue|tool "rogue" is not in this section's scope;`.
- `err.kind .. '|' .. tostring(err)`, after a tool-call loop that never settles, gives `tool_loop_exhausted|tool-call loop did not converge`.
- `err.kind .. '|' .. tostring(err.finish_reason) .. '|' .. tostring(err)` reports an `empty_model_reply` failure with its finish reason.
- `err.kind .. '|' .. err.tasks .. '|' .. tostring(err)` reports a `tasks_live` failure with its `tasks` field.
- An argument rejected for not being a positive whole number gives kind `lua` and a message containing `positive whole number`.

### The fifteen error kinds

An error value's `kind` is always one of exactly fifteen strings. The set is closed, so every failure falls into one of them, and each string names exactly one kind, so you match a kind by comparing `err.kind` against its string, as in `err.kind == 'store'`. The table lists each kind with the fields it adds beside `kind` and `message`:

| Kind | Fields beside `kind` and `message` |
| --- | --- |
| `lua` | none |
| `internal` | none |
| `cancelled` | `task`, when it reports a cancelled task |
| `context_exhausted` | `reason` |
| `empty_model_reply` | `finish_reason` |
| `model_failure` | none |
| `model_required` | none |
| `tool_loop_exhausted` | none |
| `tasks_live` | `tasks` |
| `task_not_owned` | `task` |
| `task_consumed` | `task` |
| `out_of_scope_tool` | `name` |
| `unbound_tool` | `name` |
| `store` | the store fields, such as `path` and `reason` |
| `tool` | none |

You read a kind's fields from the same error value after branching on `kind`, with ordinary table access such as `err.reason`. They are plain fields with no metamethod behind them, so `err.kind`, `err.reason`, and `err.finish_reason` on an error from [`models.loop`](11-conversations.md#starting-a-conversation) read like any other table field. A kind with no fields gives a value holding only `kind` and `message`.

Every field is a string, except two `store` fields that are integers: `count`, on an anchor failure, and `valid_up_to`, on content that is not valid UTF-8 (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). `type` returns `'number'` for these and `%` works on them, so `type(err.count) == 'number' and err.count % 1 == 0` holds.

Each kind comes from the feature that raises it:

- `context_exhausted` comes from a model round that runs out of context, through `models.infer` or `models.loop`.
- `empty_model_reply`, `tool_loop_exhausted`, `out_of_scope_tool`, `unbound_tool`, and `cancelled` can each come from `models.loop`. `out_of_scope_tool` names a tool that is not offered in the section entry (see [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer)).
- `tool` reports a tool failure from [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls).
- `tasks_live`, `task_not_owned`, and `task_consumed` come from the task functions (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) and [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin)).
- `store` comes from store operations, and only from failures the store itself reports.
- `cancelled` reports a cancellation, and carries `task` when what was cancelled is a task.
- `model_failure` comes from a model call's transport, backend, or decode failure, or a disabled gateway, through `models.infer` or `models.loop`.
- `model_required` comes from a model call with no model selected, the [missing-model error](10-models.md#infer-errors-and-the-missing-model-error).
- `internal` comes from a failure that has no kind of its own: a section entry's failed tool freeze (see [`Binding`, `Completion`, and `Tool`](16-limits-and-errors.md#binding-completion-and-tool)), an Engine fault, or an Engine function failure caused by the Lua runtime's own machinery rather than by an authoring or argument problem.
- `lua` covers every failure on the Lua side, as shown next.

### The `lua` kind

Kind `lua` covers every failure on the Lua side whenever it reaches your code as an error value, your own `error` and `assert` calls included. When a spawned task's section runs `error('beta boom')`, the section that joins it finds `results[1].result.kind == 'lua'` in the `tasks.join` result, and `tostring` of that result contains `beta boom`.

A `lua` error value marks an authoring or runtime failure, and its message is the error's own text, such as the unknown field or the store path. These failures are all kind `lua`:

- a compile error (see [Compile errors](#compile-errors))
- a runtime error in your own code, or a syntax error raised from inside an Engine function call
- an argument or misuse error from an Engine function, including a value it cannot convert to the type it expects and a value it fails to serialize or deserialize
- an exhausted log quota, as with a refused `log` call (see [Logging and lifecycle reports](#logging-and-lifecycle-reports))
- a placeholder or substitution in prose that fails to render (see [Placeholders in prose](07-substitution.md#placeholders-in-prose))
- a `call` or `tasks.fanout` recursion past its depth cap
- running out of Lua memory
- an Engine function failure caused by an outside error that is not one of the Engine's own errors

None of these has fields of its own, so a `lua` error value holds only `kind` and `message`. Its message text is the only thing that tells runtime errors, argument errors, memory refusals, refused `log` calls, and failed substitutions apart.

Recursing past the depth cap of [`call`](08-jump-and-call.md#calling-a-section) or [`tasks.fanout`](14-fanout.md#fanout-in-one-call) raises a `lua` error value whose message names the cap: `call recursion exceeded cap of 8` or `tasks.fanout recursion exceeded cap of 8`. A `pcall(tasks.spawn, ...)` in the deepest spawner catches it, with `tostring(err)` reading `call recursion exceeded cap of 8`.

Left uncaught, a `lua` failure ends the run with one of two run error kinds (see [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes)). A runtime, compile, or argument failure ends it as `Lua`. An exhausted Engine quota, such as a refused `log` call, or a recursion-cap refusal ends it as `Quota`, so an uncaught call-depth refusal ends the run as `Quota` with the message `call recursion exceeded cap of 8`.

### Argument errors

An argument error from an Engine function is an authoring error, so its kind is `lua`. The Engine raises it at the call itself, before anything is sent, so a malformed argument to a model round, a task operation, or a tool call never becomes a request:

````lua
local ok, err = pcall(models.infer, 'yo', { temperature = 0 })
return err.kind .. '|' .. tostring(err)
````

`models.infer` takes only the prompt, so this block returns `lua|models.infer takes (prompt)` and finishes without making a model request.

A caught argument error has a message of fixed text that names the argument. Argument errors from `tools.call` are worded and shaped exactly like the others, failures from the `tasks` functions, `tasks.fanout` included, arrive in the same shape as every other Engine failure, and so do the errors `models.loop` builds itself. Rendered as `err.kind .. ':' .. tostring(err)`, every argument error from `models.loop` starts with `lua:`, as in `lua:models.loop takes (compactor?)` (see [Argument errors](11-conversations.md#argument-errors)), and the errors the loop raises about the compactor passed as its argument are kind `lua` too.

Argument type errors on every Engine function, the `store` functions included, are kind `lua`. Only failures the store itself reports are kind `store`, so `pcall(function() store.write(123, 'x') end)` catches `err.kind == 'lua'`.

A string argument has three messages:

| When | Message |
| --- | --- |
| A required string argument is missing | `{name} must be a string, got nil` |
| A string argument has another type | `{name} must be a string, got {type}` |
| A string argument holds bytes that are not valid UTF-8 | `{name} must be a valid UTF-8 string` |

`{name}` is the argument's name, such as `prompt`, `input`, `path`, `contents`, `old`, `new`, `pattern`, or `text` for a task's note. `{type}` tells integers from floats, so `3` reports `got integer` and `2.5` reports `got number`, and a `prompt` given as `42` reports `prompt must be a string, got integer`. The wording is the same whichever part of the Engine raises the error. An optional string argument accepts a string, `nil`, or nothing at all, and any other type fails with `{name} must be a string, got {type}`.

### Engine errors keep their own kind

An Engine call that succeeds returns its values directly, and one the Engine rejects raises an error instead of returning a failure flag. When an Engine function raises one of the Engine's own errors, the caught value keeps that error's kind, message, and fields, and is never reported under the generic `lua` or `internal` kinds:

````lua
local ok, err = pcall(store.read, 'missing.txt')
return err.kind .. '|' .. err.reason .. '|' .. err.path
````

When the store holds no `missing.txt`, this block returns `store|not_found|missing.txt`.

A model round fails the same way. A `pcall(models.infer, 'hi')` that runs out of context returns `false` and an error value of kind `context_exhausted` with `reason` set to `provider`, and the value keeps that `reason` wherever you pass it. Every failure the Engine answers to your Lua has a kind, a spent limit included, such as kind `lua` with the message `lua log event budget exceeded`.

The usual way to handle a failure from Harness work, such as a model round, is to branch on `kind`, read the kind's own fields, and pass on anything you do not handle:

````lua
local ok, result = pcall(models.infer, prose)
if not ok then
  if result.kind == 'context_exhausted' then
    log('context ran out: ' .. result.reason)
  end
  error(result, 0)
end
return result
````

`tostring(result)` gives the Harness's message unchanged, and `error(result, 0)` raises the same error value again, as [Raising and catching errors](#raising-and-catching-errors) explains.

The Engine recognizes its own error values by a metatable they share, not by their fields. All error values in one section VM share a single metatable, so `getmetatable` returns the same table for any two caught errors. A table you build yourself therefore never becomes an Engine failure of the kind it names: `error({ kind = 'tool_loop_exhausted', message = 'x' }, 0)`, left uncaught, ends the block as an ordinary Lua runtime error, because only a table with the shared metatable is an error value.

## Raising and catching errors

After this section you can catch an Engine failure with `pcall`, handle the kinds you expect, and raise the rest again.

````markdown
---
name: careful-read
description: Reads a note, or reports that there is none yet.
runvil: 0
---

# Careful read

## Keep

```lua
local ok, result = pcall(store.read, 'note.md')
if not ok then
  if result.kind == 'store' and result.reason == 'not_found' then
    return 'no note yet'
  end
  error(result)
end
return result
```
````

When the store holds no `note.md`, the run result is:

````text
no note yet
````

`pcall` runs [`store.read`](09-the-store.md#writing-and-reading-files) in protected mode. The read fails, so instead of the failure ending the block, `pcall` returns `false` and the error value. The block handles the one failure it expects, a missing note, and raises every other error value again unchanged with `error(result)`, which fails the run exactly as if the block had never caught it. When `note.md` exists, `pcall` returns `true` and the text, and the block returns the note. The same shape works for any kind, as in a block that catches `pcall(models.infer, prose)`, returns `'model unavailable: ' .. err.message` when `err.kind == 'model_failure'`, and otherwise raises the value again with `error(err)`.

`pcall(f, ...)` calls `f` in protected mode, passing on any arguments after it, and returns a success flag first: `true` followed by every value `f` returned, or `false` followed by an error value or the value your code raised. A re-raise is raising a caught error value again unchanged, with `error(err)` or `error(err, 0)`.

### What `pcall` returns

The usual form is `local ok, err = pcall(f, ...)`, after which `err.kind` gives the error kind and `err.message` the message string. Outside cancellation, a caught failure always comes back as `false` followed by the error value (see [Cancellation passes through `pcall`](#cancellation-passes-through-pcall)).

- A successful `pcall` returns `true` followed by every return value of the protected function, nils included, so `select('#', pcall(function() return 1, nil, 3 end))` gives `4`.
- Extra arguments go on to the function, so `pcall(error, 'boom')` calls `error('boom')`.
- `pcall` returns `false` when the wrapped function raises, so `not pcall(f)` is `true` when `f` fails.
- A failure caught by `pcall` or `xpcall` does not fail the block. The block keeps running and can return normally, and when it does, the run ends normally with the returned value.

`pcall` is available in [the sandbox](#the-sandbox) and can call an anonymous function that closes over a local of the enclosing block:

````lua
local path = 'note.md'
local ok, text = pcall(function() return store.read(path) end)
````

A [suspending call](#suspending-calls) such as `models.infer`, `tools.call`, or a `store` operation works inside `pcall`. The block suspends and resumes there, because the wait crosses the `pcall` boundary, and when the call succeeds `pcall` returns `true` and the result:

````lua
local ok, reply = pcall(function() return models.infer(prose) end)
assert(ok, reply)
return reply
````

### Passing Engine functions to `pcall`

A failed Engine call raises the Engine's error value right at the call site, so you can pass the function straight to `pcall` with its arguments, get `false` and that value back, branch on it, and keep going:

- `pcall(models.infer, prose)` returns a status and the round's error value instead of ending the block.
- `pcall(models.loop)` catches an error the loop raises, and `err.kind` names it, such as `empty_model_reply`, `cancelled`, `out_of_scope_tool`, or `unbound_tool`. With a compactor, as in `pcall(models.loop, f)`, it catches the loop's failures the same way, and the model rounds inside still suspend and resume normally.
- `pcall(call, '## Heading')` catches a failed call, such as one to a missing or non-visible section, and returns `false` and the error value instead of failing the run. A target that is not a string, as in `pcall(call, 42)`, comes back with a message containing `section target must be a string`.
- `pcall(jump, '## Other')` catches a failed [`jump`](08-jump-and-call.md#jumping-to-a-section).
- `pcall(tasks.fanout, '### Worker', 5)` returns `false` and an error value whose message is about the collection, and `pcall(tasks.spawn, ...)` catches a failed spawn.
- `pcall(tools.call, 'web/fetch', { url = url })` returns `false` and the raised value when the tool call fails.
- `pcall(store.append, 'notes.md', line)` takes a store function directly.
- `pcall(tasks.concurrency, value)` passes its argument on to the Engine function, and returns `false` and the error value when the call fails.
- `pcall(require, '## broken_style')` returns `false` when a library raises, and `pcall(require, 'os')` lets the block recover from a sandbox refusal and keep running.
- [`input.ask`](#asking-the-operator) raises its failures at the call site the same way.

Reading or assigning `prose` can fail with the read-only, recursive, and missing-key failures (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), and `pcall(function() return prose end)` catches them as an error value whose `tostring` carries the message. The refusals of assignments to `argv` and `prose` are caught the same way, with the refusal on the first line of `tostring` of the caught error. A failed read through `prompt.section` or `prompt.list` (see [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist)) is catchable with `pcall` too, and `tostring` on it gives text containing the message.

### Which failures become error values

Every failure the Engine raises into Lua reaches `pcall` as an error value, Engine-raised kinds such as `tool_loop_exhausted` included, so `type(err)` is `'table'` and you decide what to do by branching on `err.kind`. Three routes lead there:

- An argument error from an Engine call, such as a badly shaped argument to `models.infer`, `models.loop`, `call`, `tasks.fanout`, `tasks.spawn`, `tools.call`, or a `store` function, is raised at the calling line as a `lua` error value rather than failing the whole block, so a `pcall` around the call catches it.
- A failed Harness request, such as a model round, arrives as an error value whose `kind` names the failure, such as `context_exhausted`, with the kind's own fields beside it and `tostring` giving the error's display text unchanged. An interrupted chat call wrapped in `pcall` likewise comes back as `false` and an error value carrying both `kind` and `message`.
- An Engine function that fails on the spot, such as a failing [`tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer), a read of an unknown `sys` field such as `sys.nothing`, or the `var` guard, gives the same indexable error value. `err.kind` and `err.message` work on it at the call site, and `tostring(err)` is the message with no traceback appended. A refused `var` write gives a message containing `var is read-only` (see [Keeping state in `var`](#keeping-state-in-var)), and a runtime error raised inside a built-in call is kind `lua`.

The `pcall` and `xpcall` globals you call are the Engine's own versions, which make this conversion, and they replace the standard library's.

`pcall` rewrites only a failure raised directly by an Engine function. Any other raised value comes back unchanged as its second result: a string, a number, a boolean, `nil`, your own table, or an error value already built. A string or table you raise with `error`, as in `error('plain', 0)` or `error(tbl)`, comes back from `pcall` or `xpcall` exactly as raised, and a table keeps its identity, so `err == tbl`. A runtime error from your own Lua code, such as calling a nil value, reaches `pcall` as Lua's plain string error, not as an error value. In every case, `tostring(err)` gives the message text.

### Raising errors with `error` and `assert`

Raise an error yourself to fail a block on purpose:

````markdown
---
name: checked
description: Fails on purpose when the answer is wrong.
runvil: 0
---

# Checked

## Verify

```lua
local answer = 'no'
assert(answer == 'yes', 'the answer must be yes')
return 'checked'
```
````

The run fails with run error kind `Lua`, the run error's text contains `the answer must be yes`, and `return 'checked'` never runs.

`assert(condition, message)` raises an error carrying `message` when `condition` is false, and returns normally and silently when it is true, so the block goes on to its next statement. `assert` works everywhere in prompt Lua, in sections, in fanout arms (see [What an arm sees](14-fanout.md#what-an-arm-sees)), and in [adhoc tool](01-what-a-prompt-is.md#effects-and-harness-work) handlers, for checking a condition mid-run with a message that says what was expected. Called with no message, as in `assert(false)`, it raises a Lua runtime error whose text is `assertion failed!`, after a location prefix such as ``section `Web Search` epilog:51:`` (see [Error line numbers and locations](#error-line-numbers-and-locations)).

`error(message)` raises `message` and stops the current block, so nothing after it runs, including the prose after the fence. A string raised this way, as in `error('boom')`, fails the block with an ordinary Lua runtime error whose message includes your text. Called in a block, its top-level code included, `error(message)` fails that block and with it the whole run, and the run error's text contains your message.

`error` fails the run only when it executes. That lets you place `error(message)` in a section as a guard that a `jump`, a chain's jump, or a fall-through (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)) should skip. In a `## Skipped` section that the walk is meant to jump past, a block holding only `error('the jump target must skip this section')` fails the run if the jump ever misses, and when the section is skipped, the `error` never fires and the run succeeds.

`error(message, 0)` raises at level 0, so the raised value is the bare message string with no position prefix, which suits code whose text must arrive exactly as written, as in `error('compacted', 0)`. Passing `error` itself as the protected function, as in `pcall(error, 'boom')`, gives the string back as the error value with no position prefix.

`error` also accepts a table as the raised value, such as a caught error value or one you build, as in `error({ kind = 'custom', message = 'handler exploded' })`. `pcall` returns your table unchanged, and left uncaught it fails the block as an ordinary Lua runtime error, because only the Engine's own error values carry a kind into the run (see [Re-raising a caught error value](#re-raising-a-caught-error-value)).

A spawned section that fails with `error('message')` reports its failure to its caller, such as in a `tasks.join` result, as an error value of kind `lua` whose `tostring` contains the message text.

An uncaught raise inside library code, during its load or from a library function called later, fails the block that made the call (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)). A `require` failure that the section does not catch ends the run with run error kind `Lua`, and the run error's text carries the raised message. Caught, the error from a failed `require` converts with `tostring` to a message that contains the text the library block raised, though not necessarily at the start, so a library that raises `fault: bad text` is found with an unanchored pattern such as `tostring(err):match('fault: .*$')`.

Rebinding `error` or `math` in your code does not change how Engine functions raise errors or name value types.

### What an uncaught error ends

An uncaught Lua runtime error ends the block with an error instead of a return value. An uncaught Lua failure, a runtime error in your code or an error you raise on purpose, then ends the run with run error kind `Lua`, holding the failure's message, wherever it happens: a walked section, a `call` [chain](04-how-a-prompt-runs.md#the-section-walk), a task, a fanout arm, or a library's load.

In the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass), an uncaught failure such as `assert(false, message)` or `error(message)` is a failed gate instead. The run ends before the walk with run error kind `RequirementsUnmet` instead of `Lua`, the Lua failure's message is kept as the error's cause text, and no later block runs (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)).

When an Engine operation fails and the block does not catch the failure, the block fails with the error the operation returned. The run ends with that call's own specific error, such as a `context_exhausted` failure from `models.infer`, rather than a plain string or a generic Lua runtime error. An uncaught error from a built-in Lua function ends the run with run error kind `Lua`, and its message names the calling function, as in `tools.offer: ...`. The exception is a `context_exhausted` raise: an uncaught `compactors.fail` call ends the run with run error kind `ContextExhausted`. An uncaught `models.loop` error ends the run according to its error kind: `tool_loop_exhausted` with run error kind `Tool`, `context_exhausted` with `ContextExhausted`, and `empty_model_reply` with `Completion`. A `lua`-kind loop error, such as a call with the wrong arguments or a loop started while another runs, fails the run as a Lua runtime error with run error kind `Lua`, and its message keeps the loop's own text.

An uncaught Lua failure leaves its block in one of two forms: a Lua error, which holds a message, or a Lua runtime error, which holds a message plus further details. A Lua runtime error's message already holds the Lua error's text, so a printout of the error and its causes shows that text once rather than repeating it. A store failure you do not catch keeps its own error instead and ends the run with run error kind `Vfs`, in the H1 pass too (see [When a store call fails](09-the-store.md#when-a-store-call-fails)).

### Re-raising a caught error value

A re-raise passes a failure on as if you had never caught it:

````lua
local ok, reply = pcall(models.infer, prose)
if not ok then
  error(reply, 0)
end
return reply
````

Raised again unchanged, with `error(err)` or `error(err, 0)`, before any other suspending call, a caught error value ends the block and fails the run with the original error, its kind and fields included, and the same run error kind, exactly as if it had never been caught. Raising the value with `error(err)` and catching it again yields the same error value, not a new one wrapped around it.

Only the exact error value you caught keeps the original error. Catching an error and raising anything else, even a value with the same message, makes the failure the block's own error, and the run reports the new error instead of the original. A string built from the value, such as `error(tostring(err), 0)` or `error(err.message)`, fails as an ordinary Lua runtime error even when the text matches, and that runtime error uses the string as its message, so the original text, such as `model is down`, still appears in it.

The Engine reads a raised value back as a structured error only when it is one of its own error values. Your own table, even one with `kind` and `message` fields such as `{ kind = 'tool_loop_exhausted', message = 'x' }`, fails the block as an ordinary Lua runtime error when it goes uncaught.

A `lua` error value that leaves a block, whether raised again or uncaught, ends the block as an ordinary Lua runtime error that keeps the same message, names the source block, and gives the absolute prompt line, as in ``section `Test` prologue:41:``.

### Raising an error value after another suspending call

An error value kept in a local survives a suspending call such as `call`, so you can still raise it afterward with `error`. This fragment comes from a section with a `### Sibling` child:

````lua
local ok, err = pcall(store.read, 'missing.txt')
call('### Sibling', '')
error(err)
````

Once another suspending call has run, the run ends according to the error value's kind:

| Error value raised after another suspending call | How the run ends |
| --- | --- |
| `context_exhausted` with its `reason`, `tool_loop_exhausted`, `empty_model_reply`, or `tool` | with that error's own run error kind |
| `store` | with run error kind `Vfs` and the original message |
| `cancelled` | with the cancelled run outcome |
| `task_not_owned` or `task_consumed` that still has its `task` field | with run error kind `Lua`, in the H1 pass too |
| any other kind, or a value missing its fields | with run error kind `Lua`, or `RequirementsUnmet` in the H1 pass |

An error value of any kind except `lua` keeps its kind and fields in the run failure this way instead of being flattened to its message string, so a `context_exhausted` value keeps its `reason`, such as `precheck`. The fragment above ends the run with run error kind `Vfs` (see [When a store call fails](09-the-store.md#when-a-store-call-fails)), and a cancelled outcome is one of the run outcomes in [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes).

### Message handlers with `xpcall`

Call `xpcall(f, handler, ...)` like `pcall`, with a message handler that sees the failure first:

````lua
local ok, handled = xpcall(error, function(e) return 'handled ' .. tostring(e) end, 'boom')
return tostring(ok) .. '|' .. tostring(handled)
````

This block returns `false|handled boom`. Arguments after the handler go on to the function, so this `xpcall` calls `error('boom')`. Outside cancellation, the handler receives the same error value `pcall` would return, so a failing built-in or Engine call reaches it as an error value whose `e.kind` and other fields are readable, and the handler's return value becomes the second result of `xpcall` after `false`. The handler is a function, as in standard Lua, and a handler of any other type gets standard Lua's own `xpcall` argument error unchanged.

### Cancellation passes through `pcall`

Your `pcall` cannot swallow cancellation. While the run is being cancelled, `pcall` raises the failure again instead of returning it, so even a retry loop around `pcall` unwinds and the run ends with the cancelled outcome. `xpcall` cannot swallow it either: under cancellation its message handler does not run and the failure is raised again, so even a handler that loops forever cannot stall the abort.

## Error line numbers and locations

After this section you can go from an error message straight to the failing line of your prompt file.

````markdown
---
name: verify-answer
description: Fails on purpose to show where an error points.
runvil: 0
---

# Verify answer

## Check

```lua
local answer = 'no'
if answer ~= 'yes' then
  error('the answer must be yes')
end
```
````

The run fails with run error kind `Lua`, and the first line of its message is:

````text
section `Check` prologue:14: runtime error: section `Check` prologue:14: the answer must be yes
````

``section `Check` prologue`` is the block's location label, and `14` is the absolute prompt line of the `error` call: line 14 of the file, counting the frontmatter. The label and line appear twice, once as a prefix at the head of the message and once where Lua's own position marker stood, with `runtime error:` between them, and a stack traceback follows on the next lines.

An absolute prompt line is a line number counted from the top of the prompt file, with the frontmatter's opening `---` as line 1. A location label names the Lua block an error came from, by its section and its place in the section.

### Absolute prompt lines

A Lua runtime error reports the failing line as an absolute prompt line, not a line counted from the top of the block, because each block keeps its starting line in the file. The absolute line is the block's first Lua line plus the line inside the block, minus one.

A block's first Lua line is the line right after its ```` ```lua ```` opener. In the prompt above, the fence opens on line 11, so the Lua starts on line 12, and the `error` call on the block's third line reports 12 + 3 - 1 = 14. Every line of the file counts: the frontmatter lines (`---`, `name:`, `description:`, `runvil:`, `---`), the H1 title, blank lines, headings, prose, and fence lines. Each block's first line is its true line in the `.md` file, in a library too and in files with CRLF line endings.

An error on a block's first line reports the line where the block's Lua starts, and an error on a later line reports that exact line, not the block's first line. A block whose Lua starts on line 40 reports line 40 for a failure on its only line, line 41 for a failure on its second line, and line 43 for one on its fourth. The rule holds in every kind of block:

- a section's prologue: a `## Work` prologue whose fence opens on line 10 reports an `assert(false)` on line 11 as line 11
- a section's epilog: an `assert(false)` on line 14 of a `## Check` epilog reports line 14
- a block between prose blocks: the third of four Lua blocks in `## S`, starting on line 23, reports an `assert(false)` on line 24 as line 24
- a block under the H1 title: one whose code starts on line 9 reports its `assert(false)` as line 9
- a block deep in a long prompt: one starting on line 112 reports an `assert(false)` on its second line as line 113

The line always names the place in the prompt file where the error was raised, never a line inside the Engine's wrapper around the block, and it sits between colons, in the form `:14:`.

### Location labels

An error message names the block by its location label, followed by a colon and the absolute prompt line, in the form `{location}:{absolute}: {result}`. The label names the section by its section name, without the `#` markers, in backticks, followed by the block's role:

| Block | Location label |
| --- | --- |
| A Lua block that opens the section, before any prose, also when it is the only block | ``section `{name}` prologue`` |
| A Lua block that ends a section that has prose before it | ``section `{name}` epilog`` |
| Any other Lua block of a section | ``section `{name}` lua {ordinal}`` |
| A Lua block under the H1 title | ``H1 `{title}` lua`` |

`{ordinal}` is the block's 1-based place among the section's Lua blocks. The role follows from the block's position, the section's block count, whether the section has any prose, and the block's place among the section's Lua blocks, so it matches the prologue and epilog of [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog) for an ordinary section. One section's two blocks read ``section `Private section` prologue`` and ``section `Private section` epilog``, a failing block under the heading `## lib` reports ``section `lib` prologue:19: load fault``, and the second block of `## lib` is ``section `lib` lua 2``.

Every Lua block is its own unit with its own location, so four Lua blocks in `## S` have four distinct locations. A long section name appears in full, so `## Gather Background Research Notes` gives ``section `Gather Background Research Notes` prologue``. A location of any length still maps, so a block labeled ``section `Shared Research Helper Functions` (line 1200) prologue`` still reports `{location}:40:`.

### Sections that share a section name

When another section heading anywhere after the H1, at any level, has the same section name, the location adds the heading's line as `(line {heading_abs_line})`, so you can tell the two sections' errors apart. With `### Helpers` on line 10 under `## A` and another `### Helpers` on line 18 under `## B`, the first section's prologue is ``section `Helpers` (line 10) prologue`` and the second's is ``section `Helpers` (line 18) prologue``.

The number is the line of the heading itself, not of the fence or the code, so a `### Helpers` heading on line 10 whose fence opens on line 12, with code from line 13, gives `(line 10)`. A heading line of three or more digits appears in full, as in ``section `Research Worker Notes` (line 109) prologue``. A section whose name no other section uses gets a location without a line number, such as ``section `C` prologue``, even when other sections in the file share names.

Errors from two sections with the same name stay separate: each reports its own block's file line, and one `Helpers` section never maps the other's error. Location labels need not be unique for this to work, because the label is a display name only. Lua knows each block by a name built from its first prompt line, `line {source_line}`, and that name decides which block an error belongs to, so two H1 blocks that both carry the label ``H1 `T` lua`` still map only their own errors.

### Compile errors

A compile error is a Lua block that does not compile as Lua 5.5, such as one with a syntax error. Every Lua block is compiled, without being run, when the prompt file is parsed, so a compile error stops the prompt before the run begins and before any of its code runs:

````markdown
---
name: broken-check
description: Holds an epilog that does not compile.
runvil: 0
---

# Broken check

## Check

Ask the model.

```lua
local secret =
return secret
```
````

The parse fails with parse error kind `Lua` (see [Parsing comes first](04-how-a-prompt-runs.md#parsing-comes-first)), and the message starts with ``lua compilation error at section `Check` epilog (line 14): `` followed by the Lua compiler's diagnostic. A compile error has this form:

````text
lua compilation error at {location} (line {source_line}): {message}
````

- `{location}` is the block's location label, so the message tells you which block the problem is in, such as ``section `Transform` epilog``.
- `(line {source_line})` is the block's first Lua line, the line after its opening fence, counted from the top of the file with the frontmatter included. It is not the line of the fault, so an epilog whose fence opens on line 12 reports ``lua compilation error at section `Check` epilog (line 13): {message}``.
- `{message}` is the Lua 5.5 compiler's own diagnostic, unchanged. It names the block `[string "line {source_line}"]`, where `{source_line}` is the block's first prompt line, such as `[string "line 1"]` for a block that starts on line 1.

The compiler's diagnostic is never rewritten to file lines, so the `{k}` in its `[string "line {source_line}"]:{k}:` marker counts from the top of the block, and the faulting file line is `{source_line}` + `{k}` - 1. In the prompt above, the diagnostic names `[string "line 14"]`, so its `{k}` added to 14, less one, is the file line of the fault.

Every block compiles this way, whether it sits in a library, under the H1 title, or in any section, the last block of the last section included: an unclosed `return (` in a final epilog fails the parse as ``section `Transform` epilog``. A syntax error in a section's prologue fence is located as ``section `{name}` prologue``, such as ``section `Private section` prologue``, and one in its epilog fence as ``section `{name}` epilog``. The compile error also carries the full Lua source of the block that failed, such as `private_payload =`.

The compile error's text is the parse error's whole message, with nothing added in front. The message states the position itself, and the parse error carries no separate span, name, line, or column. A compile error in a prompt's own blocks always fails the parse, so no run starts. A compile error during a run, such as one in a [Pack prelude](12-tools.md#pack-preludes), ends the run with run error kind `Lua`.

### Runtime error messages and tracebacks

Compile and runtime errors both name a block by its location label: a compile error as `at {location}`, and a runtime error as `{location}:{absolute}:`, the form Lua uses for a file. Before mapping, Lua's own position marker for a block reads `[string "line {source_line}"]:{k}:`, where `{source_line}` is the block's first prompt line and `{k}` is the line inside the block, so a block at prompt line 120 renders as `[string "line 120"]`. When a runtime error fails a block, the Engine rewrites every marker that belongs to the failing block or to a library it loaded with `require`, so no `[string "line {source_line}"]` from those blocks survives in the message. Lua's raw `[string "line 50"]:2: assertion failed!` becomes text that starts ``section `Web Search` epilog:51: ``.

The mapped message opens with one `{location}:{absolute}: ` prefix, taken from the first marker it rewrote, so its head shows the location and absolute line twice: once as that prefix and once in place of Lua's own marker. The run reports this mapped diagnostic as the entire message of its Lua runtime error, with no extra type label in front.

Below its first line, an uncaught Lua runtime error carries a `stack traceback:` whose frames each start with a location label and an absolute prompt line, as in ``section `lib` prologue:51: in`` and ``section `Main` prologue:31: in``, and go on with Lua's description of the frame, such as `function 'boom'` or `main chunk`. The traceback is taken where the error was raised, with the failing frames still on the stack, so it shows your own frames mapped to prompt lines rather than the Engine's wrapper around the block. It never shows `?:` placeholders, because blocks compile with their debug information kept.

Error text that names no Lua block reaches you unchanged, with no location or line added, and a runtime error message that carries no line from the failing block passes through unchanged too.

### Errors that pass through several blocks

An error from one Lua block is never mapped onto the lines of another block in the same section, and an error from one H1 block is never mapped onto the lines of another H1 block. When an error passes through several blocks, each frame maps using the starting line of the block it came from, so the calling section's frame reports its own prompt line, as when a `boom()` call on prompt line 13 appears as ``section `Main` prologue:13:``.

An error that surfaces through a parent block keeps the child's already absolute line, while the parent's own frames map to the parent's lines, so a message can start with ``section `Main` prologue:24: `` and still hold the child's line 51 intact. An error raised in a fanout arm or a called section traces back through its caller the same way, with each frame's line pointing at its own block's prompt line, so a child's ``section `lib` prologue:51: deep fault`` raised again in the parent stays at line 51.

In a traceback that passes through an Engine function, your own frames still map to absolute prompt lines, as in ``section `Test` prologue:41: in``. The frames of Engine functions written in Lua, such as `tasks.fanout`, the other `tasks` functions, the `store` functions, and the Lua code behind other Engine calls, appear as bare `file:line:` references to the built-in helper's own file. Those frames are never rewritten.

### Errors inside libraries

A Lua runtime error raised inside code loaded with `require` maps to a prompt line through the library block it came from:

````markdown
---
name: lib-fault
description: Shows where an error inside a library points.
runvil: 0
---

# Lib fault

## Main

```lua
require '## lib'
boom()
```

## lib

```lua
function boom()
  error('deep fault')
end
```
````

The run fails with run error kind `Lua`, and the first line of its message is:

````text
section `lib` prologue:20: runtime error: section `lib` prologue:20: deep fault
````

The first line of an uncaught library error carries exactly one leading prefix naming the raise point, then `runtime error:`, then the mapped message. The prefix names the library line where the raise happened, line 20 inside `boom`, even though the call came from ``section `Main` prologue``. The traceback below it shows each frame mapped once to its own prompt line, covering both the library frame, ``section `lib` prologue:20: in``, and the calling section's frame, ``section `Main` prologue:13: in``.

- A raise during a library's load reports the library block's location and the line of the raise, not the requiring block's line, so a library block whose Lua starts on line 40 and raises on its fourth line reports ``section `lib` prologue:43: load fault``.
- A raise from a library function called after the load reports the line inside the function body where the raise happened, so `error('later fault')` on the library block's second line reports ``section `lib` prologue:41: later fault``.
- In a library with several blocks, a raise in a later block maps to that block's own location and line rather than the first block's, as in ``section `lib` lua 3:21: second fault``.
- A message whose lines are already mapped keeps them when it passes back through the same block's mapping, such as when raised again with `error(msg, 0)` in a section that ran `require '## lib'`. For a library block whose Lua starts on line 50, a mapped line 51 stays 51 and never shifts a second time to 100.

## Giving `_G` a metatable

After this section you can put your own metatable on `_G` to give missing globals a default, make undefined globals an error, or hook new global writes, and to know what the Engine guard on `_G` still blocks.

````markdown
---
name: tone-default
description: Gives a missing global a default.
runvil: 0
---

# Tone default

## Reply

Tone: {{ tone }}.

```lua
require '## defaults'
return prose
```

## defaults

```lua
defaults = { tone = 'friendly' }
setmetatable(_G, { __index = defaults })
```
````

The run result is:

````text
Tone: friendly.
````

Your code reaches the global table as `_G`, a standard Lua global that [the sandbox](#the-sandbox) keeps. An author `_G` metatable is a metatable you give `_G` with `setmetatable(_G, mt)`, either in a block or in a library the block loads with `require`, to give missing globals a default, to raise an error for an undefined global, or to hook the writes of new globals. Here the `defaults` library sets `__index` to a table, so a read of a global that `_G` does not hold, such as `tone`, looks the key up in `defaults`. A `{{ name }}` placeholder (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)) reads its global through your metatable, so the default renders in the prose too.

Your metatable sits behind the Engine guard on `_G`, a metatable the Engine puts on `_G` before any of your code runs. `setmetatable(_G, mt)` records `mt` as your metatable behind that guard instead of replacing the guard, and reads of missing globals and writes of new globals reach your metatable as in standard Lua, apart from the two names the guard handles itself (see [What the Engine guard on `_G` still blocks](#what-the-engine-guard-on-_g-still-blocks)).

### Defaults with `__index`

A read of a global that `_G` does not hold goes to the `__index` of your metatable, and an assignment of a new global goes to its `__newindex`, as in standard Lua. With no metatable, or one without `__index`, a global that was never defined reads `nil`. An `__index` table answers by looking the key up in that table, and a global missing from both `_G` and the table reads `nil`. An `__index` function is called with `_G` and the key, and the read evaluates to what it returns:

````lua
setmetatable(_G, { __index = function(_, key) return 'fallback ' .. key end })
return missing_name
````

This block returns `fallback missing_name`.

`__index` and `__newindex` are read from your metatable at every lookup, so a change takes effect at the next global access, and your handlers run once per lookup however many blocks have run. An edit through `getmetatable(_G)` therefore works at once, so after the `defaults` library has run, this block returns `live tone`:

````lua
require '## defaults'
getmetatable(_G).__index = function(_, key) return 'live ' .. key end
return tone
````

Your metatable applies during the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) too, so H1 code reads its defaults and gets its strict errors, and it stays in place across later fences of the same section, so a default still reads in the second fence.

### Strict globals

An `__index` function that raises makes every read of an undefined global your own error:

````lua
setmetatable(_G, {
  __index = function(_, name)
    error('undefined global ' .. name, 2)
  end,
})
````

With this metatable in place, the block fails at the line that reads an undefined global, with your message. Raising with `error(msg, 2)` from an `__index` or `__newindex` function attributes the error to the block line that read or wrote the global, because the Engine calls your function as a tail call.

An error your handler raises reaches `pcall` exactly as raised, a plain string rather than an error value, and `pcall(function() ... end)` catches both the Engine guard's errors and your handlers' errors, returning `false` plus the error. With the strict metatable, `local ok, err = pcall(function() return undefined_here end)` sets `ok` to `false` and `err` to a string containing `undefined global undefined_here`.

### Hooking new globals with `__newindex`

A `__newindex` function receives assignments to ordinary globals that do not exist yet, and it is called with `_G`, the key, and the value:

````lua
local captured = {}
setmetatable(_G, { __newindex = function(_, k, v) captured[k] = v end })
plain = 'y'
return captured.plain
````

This block returns `y`. The hook intercepts the assignment, so the value never lands in `_G`, and a later edit to `__newindex` on your metatable takes effect immediately. A `__newindex` table receives new globals instead, so the value lands in that table rather than in `_G`. With no metatable, or one without `__newindex`, assigning a global that does not exist yet stores it in `_G`.

The Engine sets its own globals, such as `args`, before any of your code runs, so a `__newindex` hook never captures them and sees only your own new globals, while `args` still reads normally. A library loaded with `require` can install the hook with `setmetatable(_G, { __newindex = ... })`, and the hook keeps capturing ordinary global assignments in the section entry that required it.

### Other metamethods on `_G`

`setmetatable(_G, mt)` copies every field of `mt` except `__index`, `__newindex`, `__metatable`, and `__mode` onto `_G`'s guard, so the other metamethods apply to `_G` itself:

````lua
setmetatable(_G, {
  __call = function(_, name) return 'called ' .. name end,
  __tostring = function() return 'globals' end,
})
return _G('twice') .. '|' .. tostring(_G)
````

This block returns `called twice|globals`. `__call` makes `_G` callable, `__tostring` sets what `tostring(_G)` returns, and `__len` or `__concat` apply the same way.

These fields are copied at each `setmetatable(_G, mt)` call, so a later edit to one of them takes effect only at your next `setmetatable(_G, mt)`, while `__index` and `__newindex` are always read live. A second `setmetatable(_G, mt2)` replaces the first metatable completely, so metamethods found only in the first one stop applying to `_G`. `setmetatable(_G, mt)` writes nothing into your table `mt`.

### Reading and clearing the metatable

`setmetatable(_G, mt)` returns `_G`, just as `setmetatable(t, mt)` returns `t` for any other table, and after it `getmetatable(_G)` returns your own table `mt`. `getmetatable(_G)` never returns the Engine guard: it gives `nil` while you have set no metatable, even though the guard is in place, and it gives a required library's table after that library calls `setmetatable(_G, mt)`.

`setmetatable(_G, nil)` removes your metatable. Afterward `getmetatable(_G)` is `nil` and plain global assignments land in `_G` directly. Every metamethod copied from your metatable stops applying too, so a copied `__tostring` and `__call` are dropped: `tostring(_G)` reads `table: ...` and calling `_G` fails.

This handling of `setmetatable` and `getmetatable` applies only to the global table itself, compared by raw identity.

### Protected metatables and refused fields

A `__metatable` field in your metatable protects it as it would any table. `getmetatable(_G)` then returns that field's value instead of the metatable, and the protected metatable keeps working, so its `__index` table still supplies globals:

````lua
setmetatable(_G, { __metatable = 'locked', __index = { tone = 'friendly' } })
return getmetatable(_G) .. '|' .. tone
````

This block returns `locked|friendly`. Once your metatable has a `__metatable` field, any later `setmetatable(_G, ...)`, `setmetatable(_G, nil)` included, raises `cannot change a protected metatable`, prefixed with the position of the calling line.

The second argument of `setmetatable(_G, mt)` must be present and be a table or `nil`. Anything else raises Lua's standard `bad argument #2 to 'setmetatable' (nil or table expected, got {type})`, where `{type}` reads `no value` when the argument is missing.

`_G` never becomes a weak table, so its metatable holds no `__mode` field. A `mt` with a `__mode` field makes `setmetatable(_G, mt)` raise a Lua error whose message ends with `setmetatable: weak tables (__mode) are not available for _G`, while every other table can still be made weak with a `__mode` field. Finalizers are not available in the sandbox at all: `setmetatable(t, mt)` with a `__gc` field in `mt` raises a Lua error with the message `setmetatable: finalizers (__gc) are not available in the sandbox`, whatever `t` is. A rejected `setmetatable(_G, ...)` call records nothing, so `_G` keeps the author metatable it had, or none.

### `setmetatable` and `getmetatable` on other values

On every value other than `_G`, `setmetatable` and `getmetatable` behave exactly as the standard Lua 5.5 base functions, including their return values, their argument errors, and `__metatable` protection:

- `setmetatable(t, mt)` returns `t`, and `getmetatable(t)` then returns `mt`. An `__index` function on an ordinary table supplies values for missing keys, so `setmetatable({}, { __index = function(_, k) return k .. '!' end }).x` is `x!`.
- `setmetatable` takes a table and a metatable that is a table or `nil`, and other arguments raise Lua's stock bad-argument error. Argument errors from either function name it, as in `bad argument #1 to 'setmetatable'` or `bad argument #1 to 'getmetatable'`, and carry the position of your calling line, as the standard functions' own errors do.
- `setmetatable` returns exactly one value, and `getmetatable` on a table without a metatable returns exactly one value, `nil`. `getmetatable` on a number returns `nil`.
- A metatable with a `__metatable` field is protected: `getmetatable` on its table returns that field's value instead of the metatable, even when the value is `false`, and `setmetatable` on that table raises the Lua error `cannot change a protected metatable`, whether the new metatable is a table or `nil`. Lists from `json.decode`, `json.array`, and structured tool results are a case of this: their metatable's `__metatable` field is `false`, so `getmetatable` on one returns `false` (see [Keeping empty lists with `json.array`](#keeping-empty-lists-with-jsonarray)).
- `getmetatable(var)` gives the guard string `var is guarded`, and `getmetatable(sys)` gives `sys is sealed`.

`getmetatable` in a prompt is the sandbox's replacement for the base function, and `table.concat` uses that same replacement when it checks an element for `__tostring`.

### Listing globals

A block can list the globals with `for name in next, _G do` and sort or search the names with `table.sort` and plain `string.find`:

````lua
local names = {}
for name in next, _G do
  names[#names + 1] = name
end
table.sort(names)
return table.concat(names, ',')
````

### What the Engine guard on `_G` still blocks

The Engine guard on `_G` serves three names that your metatable never touches: `prose`, `messages`, and `argv` outside the H1 pass. Your code reads them as globals, even though a raw read of any of them from `_G` gives `nil`. The guard handles `argv`, `prose`, and `messages` before your metatable, so reads and assignments of those names never reach your `__index` or `__newindex`, an `argv` assignment in the H1 pass included, whatever metatable you set, clear, or change. An assignment to `messages` always raises the refusal that [`messages` is the section entry's conversation](11-conversations.md#messages-is-the-section-entrys-conversation) shows.

In an epilog that follows the prose `Hi.`, this block returns `Hi.` even though its `__index` raises for every undefined global and its `__newindex` records every new one:

````lua
local seen = {}
setmetatable(_G, {
  __index = function(_, name) error('undefined global ' .. name, 2) end,
  __newindex = function(_, k, v) seen[k] = v end,
})
return prose
````

Because `setmetatable(_G, mt)` records `mt` behind the guard instead of replacing it, no `setmetatable(_G, ...)` call, whether it clears the metatable with `nil`, swaps in `{}`, or installs hooks, changes what `argv`, `prose`, and `messages` read or lets an assignment to any of them succeed where the Engine refuses it. Editing the table that `getmetatable(_G)` returns cannot free them either, whether you replace `__index`, add or replace `__newindex`, or clear `__metatable`.
