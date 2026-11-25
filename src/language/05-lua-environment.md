# The Lua environment

Every Lua block in a prompt runs in a sandboxed section VM that already holds the Engine globals, so the Harness's work, the run's facts, and the operator are each one plain function call away. This chapter covers that environment: what a block starts with, what the sandbox keeps, how tables iterate, the run state in `var` and `sys`, logging, asking the operator, and how errors are raised, caught, and located. Knowing it tells you exactly which Lua a block can run and what each call does to your run.

## Lua blocks, section VMs, and Engine globals

After this section you can write Lua blocks that use the Engine globals, and you know which state lasts from one block or section entry to the next.

````markdown
---
name: shout
description: Shouts the text it is given.
runvil: 0
---

```lua
return string.upper(args.text)
```
````

Run with the arguments `{"text":"hi"}`, the run result is:

````text
HI
````

The block calls `string.upper`, a function from Lua's standard `string` library, on `args.text`. `args` is a global the Engine installs: a read-only table that holds the run's checked arguments (see [Reading arguments from `args`](06-arguments.md#reading-arguments-from-args)), and a prompt that leaves the `args:` key out, like this one, takes one optional string named `text` (see [The default `args:` declaration](06-arguments.md#the-default-args-declaration)). Both `string.upper` and `args` are in place when the block starts. Every `lua` fence in a prompt runs real PUC Lua 5.5, with the Harness's work, the run's metadata, and the operator one plain function call away.

### Real Lua 5.5 in every block

You write every Lua region of a prompt in Lua 5.5 syntax. The standard functions and operators work inside a block as in any Lua 5.5 program, including `assert`, `error`, `tostring`, `type`, `string.upper`, `string.find`, `table.concat`, the length operator `#`, and the concatenation operator `..`. Ordinary Lua syntax works too: `local` variables, table constructors, list indexing, numeric `for` loops, `while` loops, arithmetic, and line comments. This prompt uses each of them:

````markdown
---
name: counting
description: Counts with ordinary Lua.
runvil: 0
---

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

Statements in a block run top to bottom. In a block that runs a `store.write`, then switches model role with [`models.use`](10-models.md#selecting-a-role-with-modelsuse), then calls [`models.infer`](10-models.md#running-a-round-with-modelsinfer), the three happen in that order, and a `store.read` after the write sees what was written (see [Writing and reading files](09-the-store.md#writing-and-reading-files)).

A block can also loop without end, as in `local n = 0` followed by `while true do n = n + 1 end`. Such a block still compiles and loads normally, because an unbounded loop is as valid here as in any Lua program.

The parser compiles each Lua region when the prompt file is parsed, and compiling never runs the code. Each Lua block becomes bytecode at parse time, so a parsed prompt already holds loadable code for every block.

### A fresh section VM for every section entry

Each section entry's Lua runs in a section VM, the fresh Lua instance the entry's blocks share (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)). This prompt shows what lasts inside one section VM and what ends with it:

````markdown
---
name: lifetimes
description: Shows which globals last between blocks and sections.
runvil: 0
---

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

Inside one section entry, every `lua` block runs in the same Lua environment, so a global set in an earlier block is readable in every later block of that entry. Here `note`, set in the first block of `First`, is still there in the second block, even with a prose block between the two. A function defined in one block, such as `decorate`, can be called from any later block in that entry. The same holds for fields set on `var` (see [Keeping state in `var`](#keeping-state-in-var)), such as `var.decorated`, and for saved references to Engine globals.

That one Lua environment serves every block in the section entry, and the Engine globals in it stay live for the entry's whole lifecycle, so a later block sees the same `log` and `store` as an earlier one. A first block can run `saved_log = log`, and a later block of the same entry can call `saved_log('still here')`.

Every section entry gets a brand-new section VM, created for that section by name. That holds whether the walk reaches the section by fall-through, by [`jump`](08-jump-and-call.md#jumping-to-a-section), or by [`call`](08-jump-and-call.md#calling-a-section), and it holds for the preamble, the part of a prompt above its first heading, which is a section of its own when it holds Lua (see [The preamble](02-file-structure.md#the-preamble)). The section VM is torn down when the section entry ends.

So a plain global assigned in one section entry reads as nil when the next entry runs, because globals live in that entry's own Lua VM. After `note` is set in `First`, `assert(note == nil)` passes in `Second`, and the same goes for any plain global, such as `reply` after `reply = 'custom'`. Each section entry gets its own fresh Lua environment, so globals created or changed in one entry end with that entry. When a `counter` global in one section VM has counted up to `2`, a `counter` global in a second section VM starts over at `1`.

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

An Engine global is a global the runtime installs in every section VM before any block runs, so it is always present and you use it directly by name. Top-level code in the first block of a section can already read the Engine globals, so a first block can start like this:

````lua
first_saw_args = args
store.write('note.md', 'hello')
first_saw_store = store.read('note.md')
````

Every section VM, the preamble's included, starts with these Engine globals:

- `args`, `sys`, `var`, and `prose`
- `log`, `store`, `untrusted`, `json`, and `markdown`
- `models`, `tools`, `messages`, and `compactors`
- `call`, `execute`, `jump`, `prompt`, and `require`
- `tasks`

One more Engine global is installed only in some section VMs. `item` is installed in a spawned chain, such as a fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)). That makes 20 Engine globals in all.

Together they give every section's Lua logging through `log` (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)), tool scoping through `tools`, store operations through `store`, and the `var` and `args` globals, all inside the sandboxed section VM. `prose` is the global that reads pending prose (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), `prompt` holds the section reads (see [Reading sections with `prompt.section` and `prompt.section_or`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptsection_or)), and `models` holds the model functions: [`models.infer`](10-models.md#running-a-round-with-modelsinfer), `models.loop`, and `models.compact` run model rounds, and `models.use`, `models.get`, and `models.current` choose and read the model (see [The `models` global](10-models.md#the-models-global)). The rest are taught where they are used: `args`, the read-only table that holds the run's checked arguments, in [Reading arguments from `args`](06-arguments.md#reading-arguments-from-args), `store` in [What the store is](09-the-store.md#what-the-store-is), `untrusted` in [Handing store text to a model with `untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted), `json` in [Encoding and decoding JSON with `json`](#encoding-and-decoding-json-with-json), `markdown` in [Parsing Markdown with `markdown`](#parsing-markdown-with-markdown), `messages`, the section entry's own conversation that each new section VM starts empty, in [`messages` is the section entry's conversation](11-conversations.md#messages-is-the-section-entrys-conversation), `compactors` in [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion), `tools` in [The members of `tools`](12-tools.md#the-members-of-tools), `tasks` in [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), and `execute`, the function that runs another prompt with its own arguments and its own `var`, in [Executing another prompt](08-jump-and-call.md#executing-another-prompt).

These 20 are all the globals the Engine installs. Fanout is `tasks.fanout`, a function on the `tasks` table (see [Fanout in one call](14-fanout.md#fanout-in-one-call)), and reading a list kept in a section goes through `prompt.section`.

The preamble and every other section see the same set of globals, and `args` is read-only in all of them (see [`args` is read-only](06-arguments.md#args-is-read-only)).

The name of a [model role](10-models.md#model-roles-instead-of-models) and an offered tool never become globals of their own. `type(strong)` is `'nil'`. You read a role's model handle with [`models.get(role)`](10-models.md#getting-handles-with-modelsget), so `models.get('strong').name` reads `strong`, and you read a tool with [`tools.get(id)`](12-tools.md#tool-objects-pack-objects-and-toolspacks), so in a prompt that declares `web`, `tools.get('web/search').id` reads `web/search`.

Every section VM is set up in one fixed order. The same order applies to every section entry, every fanout arm, and every spawned task chain:

1. `untrusted`, when the VM is built
2. `args`, `sys`, `var`, `tools`, `models`, `messages`, and `compactors`
3. `log`, `json`, and `markdown`
4. `item`
5. `jump` and the `prompt` reads
6. the suspending calls: `call`, `execute`, the `tasks` table, `tools.call`, `models.infer`, `models.loop`, `models.compact`, and the `store` table
7. Pack preludes
8. `require`

### Reserved names

The Engine reserves the name of every global it installs in a section VM, and its reserved-name list matches that set exactly. The list holds 62 names: the 20 Engine globals, the 19 Lua globals the sandbox keeps (listed in [The sandbox](#the-sandbox)), and the 23 Lua 5.5 keywords. It classes each name as an Engine global, a Lua global, or a Lua keyword.

The 23 Lua 5.5 keywords in prompt Lua are `and`, `break`, `do`, `else`, `elseif`, `end`, `false`, `for`, `function`, `global`, `goto`, `if`, `in`, `local`, `nil`, `not`, `or`, `repeat`, `return`, `then`, `true`, `until`, and `while`. A Lua keyword never reads as a global. Every Engine global and Lua global on the list reads non-nil in a section entry once the one conditional binding applies, `item` in a spawned chain.

Apart from what a Pack prelude adds, every global a section entry's `_G` holds is either an Engine global or a Lua global on the reserved-name list. With no prelude, `_G` holds exactly what the Engine installs.

`global` is a contextual keyword. `global declared` compiles as a global declaration, while `local global = 1` still compiles. The prompt Lua lexer reads `global` as a contextual keyword rather than a reserved word, because the built-in Lua 5.5 uses its compatibility build for `global`, though the language reserves the name all the same. The other 22 keywords cannot be local names, as in any Lua. Reserved global names can: `store` and `pairs` compile as ordinary local names, as in `local store = 1`.

### Suspending calls

A suspending call is an Engine call that pauses the section until its answer comes, whether from the Harness, from an inbox, or from other chains, and then resumes the section with the result. You write each one as an ordinary Lua call in straight-line code:

````markdown
---
name: notes
description: Writes a note and reads it back.
runvil: 0
---

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

- `models.infer`, which takes the prompt alone or a [model handle](10-models.md#model-handles) and the prompt
- `models.loop` (see [Starting a conversation](11-conversations.md#starting-a-conversation))
- `tools.call` (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls))
- `call`
- `execute` (see [Executing another prompt](08-jump-and-call.md#executing-another-prompt))
- the `tasks` functions, [`tasks.fanout`](14-fanout.md#fanout-in-one-call), [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), and [`tasks.receive`](#asking-the-operator) included
- the `store` calls (see [Writing and reading files](09-the-store.md#writing-and-reading-files))

Each one resumes with its result, and `tasks.spawn` resumes with the task table. `jump`, `prompt.section`, `prompt.section_or`, and the `markdown` functions return without suspending.

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

A block pauses at a suspending call and resumes with the result, so you never write callbacks or manage coroutines, and `coroutine` reads nil in a block. A block pauses only at one of the Engine's suspending calls, because the Engine keeps the `coroutine` library for its own use.

While a suspending call waits, only the calling [chain](04-how-a-prompt-runs.md#the-section-walk) pauses, and other chains keep running. Two tasks started with `tasks.spawn` that each run `call('Leaf')` both complete, in whichever order their answers arrive.

The suspending calls work in every section's Lua, the preamble and fanout arms included, so a fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)) can pass its `item` straight to `tools.call`.

The sandbox keeps the `io`, `os`, `package`, `coroutine`, and `debug` libraries out of every block, which closes off the direct routes to I/O and a clock. Work outside the block happens only through Engine calls.

### Pack preludes

A Pack prelude is Lua source that a Pack the prompt declares contributes, and every section VM of that prompt installs it after the Engine globals and before any block runs. A prelude is how a declared Pack defines Lua globals of its own. The Packs that ship with Runvil lack a prelude, so this section uses a made-up third-party Pack, `acme/issue-tracker`, whose tool `issue-tracker/open` opens a ticket and whose prelude defines a `tickets` table (see [Pack preludes](12-tools.md#pack-preludes)):

````markdown
---
name: ticket-check
description: Reads the globals a Pack prelude adds.
runvil: 0
uses:
  - acme/issue-tracker
---

```lua
return type(tickets.open) .. ',' .. type(store.read) .. ',' .. type(tools.call)
```
````

Run on a Host that has the `acme/issue-tracker` Pack (see [Packs at prepare](04-how-a-prompt-runs.md#packs-at-prepare)), the run result is:

````text
function,function,function
````

The `issue-tracker` prelude defines the global `tickets`, so `type(tickets.open)` reads `function`. A prelude global sits beside the Engine's globals without replacing them, so `type(store.read)` and `type(tools.call)` still give `function` when `acme/issue-tracker` is declared. Your code still sees the `args`, `sys`, and `models` globals when Pack preludes are installed.

A prelude can add any globals to a section entry's `_G` beyond the ones the Engine installs. A prelude that runs `answer = 42`, `label = 'kit'`, and `function shout(text) ... end` makes `answer`, `label`, and `shout` readable from your blocks.

A prelude can also define a table global holding fields and functions. `tickets` is one: the `issue-tracker` prelude creates the `tickets` table and defines `tickets.open` as a function on it, so a block calls `tickets.open('Fix login')` like any other global function.

Preludes install when each section VM is set up, before that VM runs its first block, in the preamble too. Every Lua global a prelude defines is therefore available to every block of every section entry of the prompt, fanout arms and spawned task chains included, and to every library a block loads with `require`. In a prompt that declares `acme/issue-tracker`, a required library, a fanout arm, and a spawned task can each call `tickets.open`. A prompt run with [`execute`](08-jump-and-call.md#executing-another-prompt) installs the preludes of the Packs that it declares itself and leaves out its caller's, so a prelude global exists only in the prompts that declare its Pack (see [Packs and preludes in an executed prompt](12-tools.md#packs-and-preludes-in-an-executed-prompt)).

Preludes install in the order the prompt's `uses:` list declares their Packs.

A prelude function called from a block can use `store.write` and `store.read`, and each store call inside it is a store effect issued in call order. A prelude function that writes a file and then reads it back issues the write first and the read second, just as the same two calls in a block would.

#### Prelude tables are read-only

A table global that a Pack's prelude defines is read-only at its top level. You read and call its fields as usual, so `tickets.open('Fix login')` works, but assigning a field, as in `tickets.extra = 1`, raises this error:

````text
tickets is read-only: Pack `issue-tracker` defines it; cannot set 'extra'
````

The message has the form `` {name} is read-only: Pack `{pack}` defines it; cannot set '{field}' ``, naming the table, the Pack id of the Pack that defines it, and the field. The table is sealed: `getmetatable(tickets)` returns `tickets is sealed`, while `pairs(tickets)` still visits the fields the prelude gave it.

#### Which Packs add globals

A Pack's globals go only to the prompts that declare it. When `acme/issue-tracker` is undeclared, `tickets` reads nil, and `tickets.open('Fix login')` raises this error:

````text
attempt to index a nil value (global 'tickets')
````

A prelude that defines a reserved name fails to install, so a Pack leaves every Engine global, kept Lua standard-library global, and Lua keyword as it is.

#### When a prelude fails

All the preludes that one prompt installs share one global namespace, and a prelude that cannot install fails the run with [error code](16-limits-and-errors.md#run-error-codes) `lua_runtime`. Every such message starts with `` Pack `{pack}`: its prelude ... ``, naming the Pack that contributed the prelude. A prelude fails in these cases:

- It defines a global an earlier prelude already defines. The message has the form `` Pack `{pack}`: its prelude defines the global `{name}`, which Pack `{owner}`'s prelude already defines ``, naming the later Pack, the global, and the Pack that defined it first.
- It defines a global that an Engine global or another reserved name already holds. A prelude defining `item` fails the run with a message saying the name `is reserved as an Engine global`.
- It fails to load. The message says `its prelude failed to load`.

A prelude failure happens when the first section VM is set up, before the run issues any effect. The failure comes before every store write, tool call, and model request of the prompt, and that includes a preamble block such as `store.write('first.txt', 'x')`, which would otherwise be the first thing the prompt does. In a prompt run with `execute`, the failure happens when that prompt's own first section VM is set up: it is raised at the `execute`, as an error value of kind `lua` that a `pcall` around the `execute` catches, after everything the calling prompt did before that line.

## The sandbox

By the end of this section you know every library and base function a block can call, and which familiar Lua names read nil in a block.

````markdown
---
name: shout
description: Shouts a fixed word.
runvil: 0
---

```lua
return string.upper('hello')
```
````

The run result is:

````text
HELLO
````

This is the smallest sandbox block: it returns the result of one library call as the run result. Every Lua block runs in the sandbox, a restricted global environment that leaves out direct printing and the dangerous globals, and offers the safe standard library, a `setmetatable` that refuses finalizers and weak globals but still allows `_G` handlers, and the `untrusted` global.

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

Seven of the base functions behave in a Runvil way:

- `pairs` and `next` visit keys in a fixed order (see [Deterministic table iteration](#deterministic-table-iteration)).
- `tostring` gives a table, a function, or a userdata as its name alone, without a memory address (see [Turning values into text](#turning-values-into-text)).
- `pcall` and `xpcall` deliver a failure raised by an Engine function as an [error value](#error-values), while passing your own raised value through unchanged, whether it is a string, a table of your own, or an error value already built. The one exception is a string that starts with the position of an Engine file, which loses that start (see [Raising and catching errors](#raising-and-catching-errors)).
- `setmetatable` and `getmetatable` give `_G` an author metatable that sits behind the guard on `args`, `prose`, and `messages` (see [Giving `_G` a metatable](#giving-_g-a-metatable)).

On values other than `_G`, `getmetatable` and `setmetatable` behave as in stock Lua, so `setmetatable(t, mt) == t` and `getmetatable(t) == mt` both hold (see [`setmetatable` and `getmetatable` on other values](#setmetatable-and-getmetatable-on-other-values)). You can call `getmetatable` on a task handle from `tasks.spawn` too, as in `getmetatable(t) == nil`.

The other base functions work as in standard Lua 5.5. Inside the libraries, `string.format`, `table.concat`, `table.sort`, and `math.randomseed` are Runvil's own, and the section on each library says how it differs. You call `ipairs(t)`, `select(n, ...)`, and `tonumber(v)` as in any Lua program, and `select('#', ...)` and `select(2, pcall(...))` give the same results as in stock Lua. `tonumber` turns a numeric string read from the store into a number, as in `tonumber(ok and v or '0') + 1`. `assert(cond, message)` checks a condition with a message. `ipairs` walks an array table, such as a table literal, in order, so `for _, text in ipairs({ 'first question', 'second question' }) do` handles `first question` before `second question`.

The standard functions `string.rep`, `select`, `table.concat`, `ipairs`, `type`, `tostring`, `getmetatable`, and `assert` work around Engine calls too, `models.loop` included, as in `select('#', models.loop())` and `getmetatable(err) == nil`. Section Lua combines them freely, as in `assert(not pcall(require, '/broken'), 'the broken library raises')`, which checks that requiring a broken library raises.

The kept globals are ordinary globals, so you can reassign one in prompt code. `error = function() end` runs without complaint. Only your own code calls the function you put there: the Engine's own Lua code took every global it uses before your code ran, so `tasks.join` reads its set with the stock `ipairs` whatever `ipairs` is by then.

### Checking types

`type(v)` returns a value's Lua type name, such as `'string'`, `'number'`, `'boolean'`, `'table'`, `'function'`, or `'userdata'`, so you can test what a value is:

````markdown
---
name: types
description: Reports the type of several values.
runvil: 0
---

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

You build strings with the `..` concatenation operator, joining string literals with other values, as in `"Hello, " .. args.text .. "!"`. `..` joins strings and numbers, and an integer renders without a decimal point, so you can render a number such as the one [`tasks.concurrency`](14-fanout.md#concurrency-and-the-admission-window) returns: with `n` equal to `4`, `'clamped:' .. n` reads `clamped:4`, and a join beginning `#history .. '|' .. history[2].content ..` produces `4|first reply|second reply`. `..` also works on a value an Engine call returns, so `return "pcall:" .. result` gives `pcall:answer` when `result` is the reply `answer`.

`tostring(v)` turns other values into text for joining or storing:

- A number becomes its digits, for a return value or an assert message, as in `'expected 1 call, got ' .. tostring(count)`, or for a list item, as in `items[i] = tostring(i)` or `tostring(#files)`.
- A boolean, such as the status `pcall` returns, becomes `true` or `false`, so `var.answer .. ':' .. tostring(var.sys_untouched)` gives `first answer:true`, and `tostring(ok)` stores the status as text.
- `nil` becomes the string `nil`, so `tostring(print) .. ':' .. tostring(warn)` returns `nil:nil`.
- An error value becomes its message, so `tostring(err)` gives the error message.
- A tool result or a model reply becomes its text, so `tostring(reply)` gives `pong` when the model replies `pong`.
- Any other table, function, or userdata becomes its name alone: `table`, `function`, `userdata`, or the `__name` of its metatable when that is a string.

````markdown
---
name: names
description: Turns tables and functions into text.
runvil: 0
---

```lua
local point = setmetatable({}, { __name = 'Point' })
return tostring({}) .. '|' .. tostring(type) .. '|' .. tostring(point) .. '|' .. tostring(messages) .. '|' .. tostring({} == {})
```
````

The run result is:

````text
table|function|Point|MessageList|false
````

Stock Lua adds the value's memory address to the name, as in `table: 0x55d0c0a1e2f0`. The address is where the value happens to sit in memory, so it differs from run to run, and Runvil leaves it out so that a text built from such a value is the same on every run. Two different tables therefore turn into the same text, and code that must tell them apart compares the tables themselves with `==`, as `{} == {}` does here. A value with a `__tostring` metamethod turns into what that metamethod returns, as in stock Lua.

`tostring`, `string.format`, `table.concat`, and `table.sort` are Runvil's own functions, and each one runs your code as the stock function does. Your code here is a `__tostring`, `__len`, `__lt`, `__index`, or `__newindex` metamethod, or the comparison function of a sort. That code may make a [suspending call](#suspending-calls), such as `store.read`, where the same call in stock Lua raises `attempt to yield across a C-call boundary`: in a `__tostring` that `tostring` or `string.format` runs, in the `__len` and `__index` of a list that `table.concat` joins, and in the code that a sort runs (see [Sorting with `table.sort`](#sorting-with-tablesort)). An error that code raises names the line that raised it, and it reaches the message handler of an `xpcall` around the call before a `<close>` variable of that code closes. A level counts as under the stock function, which is a C function with no line: `error(message, 2)` in that code adds no position to the message, and `error(message, 3)` adds the line of your call to `tostring`, `string.format`, `table.concat`, or `table.sort`. When that call is the whole of a `return` inside a function, as in `return tostring(value)`, Lua makes it a tail call and drops your function from the levels, so level 3 is the line that called your function, where stock Lua gives the line of the `return`. When Lua cannot run the code, the error has stock Lua's text, as in `attempt to call a number value` for a `__tostring` that holds a number. Calls that nest without end, such as a `__tostring` that passes its own value to `tostring`, fail with Lua's `stack overflow` or `C stack overflow`. The error names your line when the stack runs out in your function, and has no position when it runs out inside one of Runvil's functions. A metamethod that Lua runs for an operator, such as a `__len` that sorts its own table, fails with `C stack overflow` at about 200 levels, as in stock Lua. When that limit is met while one of the four functions handles a userdata such as `messages`, to print it, to join or sort it as a list, or to compare it in a sort, `pcall` returns an [error value](#error-values) whose message contains `C stack overflow`, where stock Lua gives the string. A `stack overflow` that is met at that point comes back as an error value too, with `stack overflow` in its message. The same holds when the limit of about 200 levels is met inside another Runvil function, such as `json.encode`, or while `tostring` prints an error value. With a memory limit below the default of 64 MiB (see [Lua memory and log quotas](16-limits-and-errors.md#lua-memory-and-log-quotas)), such a nest can fill the Lua heap before the stack, and it then fails with `not enough memory`: a `__tostring` that passes its own value to `string.format` does at 48 MiB.

### The `string` library

The sandbox keeps the `string` library, and you use it as in standard Lua 5.5, including method calls on strings:

````markdown
---
name: strings
description: Uses the string library.
runvil: 0
---

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

- `string.upper` and `string.lower` change case, so `return string.upper(args.text)` with the arguments `{"text":"hi"}` returns `HI`, and `return string.lower(value)` lowers a value.
- `string.match` matches a Lua pattern and returns its captures, so `string.match(s, '^(%d+):([%d%.]+)$')` returns two captures. Anchored patterns work on message content too, so `messages[2].content:match('^Answer: (.+)$')` returns the captured text after the prefix. Pattern escapes work as in standard Lua, as `%-` does in `tostring(err):match('read%-only')`.
- `string.find` finds a pattern, as in `string.find(wrapped, "untrusted_input_")`. With its plain-find flag it searches for literal text: `string.find(m.content, 'Answer:', 1, true) == 1` tests for a prefix, and `err:find(']:2: undefined global undefined_here', 1, true)` finds that exact text. A plain search returns `nil` when the text is absent, so `tostring(err):find('appears exactly once', 1, true) ~= nil` tests whether a message contains that text.
- `string.sub` takes negative indexes that count from the end, inside sections and fanout arms alike, as in `tostring(sys.index) == string.sub(item, -1)`. Its positions count bytes, not characters, so a cut can split a multibyte character, and [the `utf8` library](#the-utf8-library) finds the character boundaries to cut on.
- `string.rep(s, n)` repeats `s` `n` times, so `messages:user(string.rep('x', 100000))` builds a 100000-character user message.
- `string.format` builds text from a format string, with two differences from stock Lua. `%s` gives a table, a function, or a userdata the way [`tostring`](#turning-values-into-text) does, as its name alone, so `string.format('%s has %d', {}, 3)` returns `table has 3`. `%p` is not available, because it prints nothing but a memory address: a format that holds it raises `string.format: conversion '%p' is not available, it prints a memory address`.
- `string.char` builds a string from raw byte values, which can produce bytes that are not valid UTF-8. `log` rejects the string from `string.char(255)` as not UTF-8 (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)).

### The `table` library

The sandbox keeps the `table` library, and a block calls its functions as `table.{name}(...)`, as in standard Lua 5.5. The `#` length operator works on tables, so you append with `answers[#answers + 1] = value` and join with `table.concat`:

````markdown
---
name: joined
description: Joins strings, numbers, and a value with __tostring.
runvil: 0
---

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

`table.concat(list, sep, i, j)` joins `list[i]` through `list[j]`, with `i` defaulting to `1`, `j` defaulting to `#list`, and `sep` defaulting to the empty string. So `table.concat(list)` joins the whole list end to end, and `table.concat(items, ', ')` over `alpha` and `beta` gives `alpha, beta`. When the range is empty, such as an empty list or an `i` greater than `j`, `table.concat` returns an empty string. It works in sections and fanout arms alike, and `table.concat(r, '|')` over ten results can produce `1|2|3|4|5|6|7|8|9|10`.

String and number elements join as they are, as in stock Lua. `table.concat` also joins these elements, each as `tostring` gives it:

- Every userdata element. One with a `__tostring` metamethod joins as what that metamethod returns, and one without joins as its name, so a record of the message list joins as `RecordView`. This covers the Engine's own values.
- Every table element whose metatable has a `__tostring` field. This covers fanout result entries and your own objects made with `setmetatable(t, { __tostring = f })`, such as `point` above, where `table.concat({ 'at', point }, ' ')` returns `at (1,2)`. A `__metatable` field in that metatable changes nothing here, because `table.concat` finds `__tostring` as Lua does and not through `getmetatable`.

So `table.concat` over a [`tasks.fanout`](14-fanout.md#fanout-in-one-call) result list joins each arm's text with the separator: `table.concat(r, ',')` over three arms returns `a:x1+y2+z3,b:x1+y2+z3,c:x1+y2+z3`.

`table.pack` returns its arguments in a table with an `n` field holding the count, as in `local results = table.pack(pcall(f))` followed by `for i = 1, results.n do`. `select('#', ...)` also counts its arguments.

#### When `table.concat` fails

`table.concat` raises a Lua runtime error when an element in the range cannot join. A boolean, a function, a thread, or a table without a `__tostring` metamethod gives a message naming the type and the index. A `true` at index 3 gives:

````text
invalid value (boolean) at index 3 in table for 'concat'
````

An index in the range that holds nil, such as a hole in the list or a `j` past `#list`, gives `invalid value (nil) at index {k} in table for 'concat'`, where `{k}` is the index inside `i..j`. A `j` past the end therefore reports index `#list + 1`.

The arguments are checked as stock Lua checks them. The list must be a table, or a value whose metatable has both `__index` and `__len`, such as the message list, so `table.concat(nil)` raises `bad argument #1 to 'concat' (table expected, got nil)`. A bound that is not a whole number, as in `table.concat({ 1, 2 }, '', 1.5)`, raises `bad argument #3 to 'concat' (number has no integer representation)`. The position prefix of each of these errors names your calling line (see [Error line numbers and locations](#error-line-numbers-and-locations)). An argument error names the function as you write the call, as stock Lua does. `pcall(table.concat, nil)`, where no Lua code of yours makes the call, gives `bad argument #1 to 'table.concat' (table expected, got nil)`, and a call through a local variable gives the name of the variable. The same holds for `table.sort`, `string.format`, `tostring`, `math.randomseed`, `pcall`, `xpcall`, `setmetatable`, and `getmetatable`. A method call such as `('%d'):format('x')` leaves the string out of the count and names argument 1. A tail call and a `__call` metamethod read otherwise than in stock Lua. When the call is the whole of a `return` inside a function, as in `return join(nil)` for a local `join` that holds `table.concat`, the error names the function `concat`, where stock Lua says `join`, and a method call written that way counts its value. Runvil calls an `__index` or `__newindex` function that you set on `_G` in the same way, so when that function is one of these, the error names it by its field, as in `format`, where stock Lua says `index` or `newindex`. Runvil calls the message handler of an `xpcall` in the same way too, so a handler that is one of these functions is named by its field, as in `format`, where stock Lua says `string.format`. When the function is the `__call` metamethod of a value you call, that value counts as argument 1, so each number is one higher than in stock Lua.

Each of these is an ordinary Lua runtime error. `pcall` catches it, and left uncaught it ends the run with code `lua_runtime` (see [What an uncaught error ends](#what-an-uncaught-error-ends)).

#### Sorting with `table.sort`

`table.sort(list)` sorts a list in place by `<`, and `table.sort(list, before)` sorts it by your comparison function, which returns true when its first argument must come before its second:

````markdown
---
name: sorted
description: Sorts records by weight and keeps ties in their order.
runvil: 0
---

```lua
local list = { { n = 'pear', w = 2 }, { n = 'fig', w = 1 }, { n = 'plum', w = 2 }, { n = 'kiwi', w = 1 }, { n = 'lime', w = 2 } }
table.sort(list, function(a, b) return a.w < b.w end)
local names = {}
for i, item in ipairs(list) do names[i] = item.n end
return table.concat(names, ' ')
```
````

The run result is the same on every run:

````text
fig kiwi pear plum lime
````

Runvil's `table.sort` is a stable sort: items that compare equal keep the order they had, so `fig` stays ahead of `kiwi`, and `pear`, `plum`, and `lime` stay in the order the list gave them. The result therefore depends only on the list and the comparison. Stock Lua's sort makes no such promise, and it sorts this list to `kiwi fig pear plum lime`.

The sort reads every item once, compares, and then writes every item back once. This gives it four properties that stock Lua's sort lacks:

- A sort that fails leaves the list as it was. With `local t = { 3, 1, 2 }`, a comparison function that raises leaves `t` as `3, 1, 2`, and so does `table.sort({ 3, 'a', 1 })`, which raises `attempt to compare string with number`.
- A sort of two or more items always writes, even when the list is already in order, so a list that refuses writes, such as a list inside `args`, always raises (see [Table functions that write](06-arguments.md#table-functions-that-write)). A list of fewer than two items has nothing to sort, and the call returns without a read or a write.
- A comparison function may make a [suspending call](#suspending-calls), such as `store.exists`, because the sort is Lua code, and so may the `__lt` and `__len` metamethods the sort uses and the `__index` and `__newindex` metamethods of the sorted list. In stock Lua the same call raises `attempt to yield across a C-call boundary`.
- A comparison function that contradicts itself, such as one that always returns true, gives the list some order, the same on every run. It never raises stock Lua's `invalid order function for sorting`.

The list must be a table, or a value whose metatable has `__index`, `__newindex`, and `__len`, as in stock Lua. A record of the message list has no `__len`, so `table.sort(messages[1])` raises `bad argument #1 to 'sort' (table expected, got RecordView)`. The message list has all three, so `table.sort(messages)` is taken and fails at its work once the list holds two records: with `attempt to compare two RecordView values`, or, with a comparison function, with the refusal of the write, `messages cannot be assigned to by index; add records with append(record), or rewrite history with models.compact(compactor, reason)`. A comparison that is not a function raises `bad argument #2 to 'sort' (function expected, got {type})`, where `{type}` is the `__name` of the value's metatable when that is a string, as in stock Lua. The error names the function as you write the call (see [When `table.concat` fails](#when-tableconcat-fails)). The sort measures the list by `#list`, so keep the list without a gap (see [The length of a table with a gap](#the-length-of-a-table-with-a-gap)). Each of these errors is an ordinary Lua runtime error that names your calling line.

### The `math` library

The sandbox keeps the `math` library. A block calls it as `math.{name}(...)`, and it works as in standard Lua 5.5, except for where its random numbers start.

`math.random` draws from a generator that the Engine seeds for each section entry, from the run's [seed](01-what-a-prompt-is.md#seed-start-time-and-repeatable-runs) and the [section entry's id](#section-entry-ids). So the draws differ from one run to the next and from one section entry to the next, and a run repeated with the same seed draws the same numbers again. Stock Lua seeds the generator from the clock and a memory address, which a later run cannot repeat. A library loaded with `require` and a [Pack prelude](#pack-preludes) draw from the generator of the section VM they run in.

`math.randomseed(n)` seeds the generator yourself, as in stock Lua, and the draws after it follow from `n` alone:

````markdown
---
name: reseed
description: Seeds the generator twice with one number.
runvil: 0
---

```lua
math.randomseed(42)
local first = math.random(1, 100)
math.randomseed(42)
return tostring(first == math.random(1, 100))
```
````

The run result is:

````text
true
````

`math.randomseed()` with no argument does not read the clock as stock Lua does. It seeds the generator from the generator's own next two draws, so the draws after it still follow from the run's seed. The generator is Lua's own and is not cryptographic, so a draw is no secret: do not use one as a token or a key.

### The `utf8` library

The sandbox keeps the `utf8` library, and you use it as in standard Lua 5.5 to count and cut text by character instead of by byte:

````markdown
---
name: characters
description: Counts and cuts a string with the utf8 library.
runvil: 0
---

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

```lua
return tostring(print) .. ':' .. tostring(warn)
```
````

The run result is:

````text
nil:nil
````

`print` and `warn` read nil in a section VM, so a block cannot write to standard output or emit Lua warnings directly. `log(message)` is the output path instead (see [Logging and lifecycle reports](#logging-and-lifecycle-reports)). `print` and `warn` read nil in every kind of Lua VM a prompt runs in, library code and a VM with tools bound included.

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

- The `io`, `os`, `package`, and `debug` libraries are never loaded, which closes off Lua file and stream I/O.
- `io` reads nil, so a block cannot open files or read standard input.
- `os` reads nil, so `os.time`, `os.clock`, `os.getenv`, and `os.execute` are unavailable.
- `package` reads nil, so `package.path`, `package.loaded`, and search paths are unavailable.
- `load` reads nil, so a block cannot compile a string into a function at run time. `loadfile` and `dofile` read nil too, so it cannot load or run a Lua file from disk.
- `debug` reads nil, so a block cannot inspect stack frames, locals, or hooks.
- `collectgarbage` reads nil, so prompt Lua cannot query or drive the garbage collector.

The `coroutine` global reads nil in your code as well. The Engine loads the coroutine library only for its own suspending calls and removes the global before section setup ends, so a block cannot create coroutines or yield on its own, and every yield in a run comes from an Engine function (see [Suspending calls](#suspending-calls)).

The raw-access functions are stripped after the Engine guard on `_G` captures `rawequal`, `rawget`, and `rawset` for its own use. A section VM lacks the `debug` library, `rawset`, and code loading, so the Engine guard on `_G`, a raw `prose` global, and a raw `args` global stay out of reach of your code (see [Giving `_G` a metatable](#giving-_g-a-metatable)).

Calling any of these absent names, for example `print("hi")`, fails with a Lua runtime error because the name holds nil. The message is stock Lua's error for calling a nil value, or your own error when an `__index` function you gave `_G` raises at the read (see [Strict globals](#strict-globals)).

#### Runvil's own `require`

The sandbox leaves both the stock Lua `require` loader and the `package` library out of a prompt's Lua code. The `require` a block does see is Runvil's own section-loading function (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)):

````markdown
---
name: own-require
description: Shows that require loads sections, not stock Lua libraries.
runvil: 0
---

```lua
local ok = pcall(require, 'os')
return type(require) .. '|' .. tostring(ok) .. '|' .. tostring(os) .. '|' .. tostring(package)
```
````

The run result is:

````text
function|false|nil|nil
````

`type(require)` is `'function'`, while `pcall(require, 'os')` returns `false`, since the load fails, and leaves `os` and `package` nil.

## Deterministic table iteration

After this section you can predict the exact order in which `pairs` and `next` visit a table's keys, tell which tables they refuse to walk, and change a table safely while you walk it.

````markdown
---
name: order
description: Walks a table with pairs.
runvil: 0
---

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

`pairs` visits a table's string, number, and boolean keys in a fixed order instead of Lua's hash order, so the same table iterates the same way on every run. This fixed order is the deterministic key order. Write the same table as `{ beta = 4, mid = 3, alpha = 2, zeta = 1 }` and the walk is still `alpha, beta, mid, zeta`, because the order depends only on the keys and not on the table's hash state or on the order you inserted them.

The order comes from each key alone, from its type and its value. A table with every kind of key the walk takes shows the whole order:

````markdown
---
name: mixed-order
description: Walks a table that mixes key types.
runvil: 0
---

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
1 2 10 true z
````

Table iteration visits the integer keys from `1` up first, in rising order. Then it visits the rest of the table's keys: boolean keys with `false` before `true`, then the other number keys by value, then string keys bytewise. The sections below give each rule.

### Integer keys from 1 up come first

`pairs` visits every integer key from `1` up before any other key, in rising order. So a list comes out in index order, ahead of every named field: `{10, 20, 30, extra = 'x', another = 'y'}` visits `1, 2, 3, another, extra`.

These keys come before the booleans too, even though booleans come first among the other keys. `{[true] = 1, "x"}` visits `1, true`, and `{10, 20, [false] = 'f', [true] = 't'}` visits `1, 2, false, true`.

A key's place depends on the key alone and never on the table's length, so a gap in the indices changes nothing. `{"a", [10] = "c", [true] = "x", flag = 1}` walks `1`, `10`, `true`, `flag`, and `{ [false] = 'b', [3] = 'c', [7] = 'y' }`, which has no key `1`, walks `3, 7, false`. An index that holds nil is not a key, so iterating a table with gaps never yields a nil value.

The integer keys below `1`, which are `0` and the negative integers, are not in this group. They are ordered with the other number keys, after any boolean keys.

### Then booleans, the other numbers, and strings

After the integer keys from `1` up, the walk visits the boolean keys, the number keys that are left, and the string keys, sorted by type: booleans first, then numbers, then strings. Between keys of different types, the type alone decides the order. Every boolean key precedes every one of those number keys, and every one of those number keys precedes every string key, so a table that mixes the types still gets one order and iterates without error:

````lua
local t = { [true] = 't', [7] = 'seven', b = 'bee', [false] = 'f', [2.5] = 'half', a = 'ay' }
````

`pairs(t)` visits `7, false, true, 2.5, a, b`. The key `7` comes first, as an integer from `1` up. Among boolean keys, `false` comes before `true`, whatever order they appear in the table constructor, and here `[true]` is written first. Keys of the same type come in one shared sorted order, so their sequence never depends on insertion order or hashing.

Every boolean, number, and string key has a place in the order, and the place depends only on the key. So the iteration order depends only on which keys a table holds, never on the order they were assigned or inserted, and two tables holding the same keys iterate in the same order.

#### Number keys

The number keys of this group are `0`, the negative integers, and every float key. They iterate in ascending numeric value, with integer and float keys sorted together in one sequence by exact numeric value:

- `-1.5` comes before `-1`, `-1` comes before `-0.5`, `-0.5` comes before `0`, and `0` comes before `0.5`.
- A float key above `1`, such as `1.5` or `2.5`, is in this group as well, so it comes after every integer key from `1` up and after the booleans. A table with the keys `1`, `1.5`, `2`, and `false` walks `1, 2, false, 1.5`.

A float key with a whole value is the same key as the integer, as in stock Lua: `t[2.0]` and `t[2]` name one entry, and the walk yields the integer `2`.

A float key outside the 64-bit integer range sorts by its value like any other: `1e19` and `1e300` come after every integer key of this group, and `-1e300` comes before `math.mininteger`. The infinite keys have places too: `-math.huge` sorts before every other number key of this group, and `math.huge` after every other one.

Integer keys from `1` up keep their integer identity however large they are. Keys past 2^53, such as `9007199254740992` and `9007199254740993`, keep distinct, exact positions and never tie. `math.maxinteger`, which is `9223372036854775807`, is the last key the first group can hold, so it comes before every boolean and before every float, the float key `2^63` included.

#### String keys

String keys come after boolean and number keys, and a `for k in pairs(t)` loop visits them in sorted bytewise order of their raw bytes, the same in every locale, with digits compared as plain bytes. So a table with only string keys visits them sorted, and:

- `"Z"` comes before `"a"`, and for ASCII `"B"` comes before `"a"`.
- `"10"` comes before `"9"`.
- `"a"` comes before `"ab"`, and `"ab"` comes before `"b"`.

`{ zeta = 1, alpha = 2, mid = 3, beta = 4, omega = 5 }` visits `alpha, beta, mid, omega, zeta`. A string key whose bytes are not valid UTF-8, such as `"\255"`, sorts by its bytes like any other string key.

### Keys that cannot be walked

A key that is a table, a function, or a userdata has nothing to sort by except its memory address, and the address differs from run to run. So `pairs` and `next` raise an error for a table that holds such a key, instead of visiting it in an order that could change:

````markdown
---
name: object-keys
description: Looks up object keys and tries to walk them.
runvil: 0
---

```lua
local a, b = { name = 'a' }, { name = 'b' }
local seen = { [a] = true, [b] = true }
local ok, err = pcall(pairs, seen)
return tostring(seen[a]) .. '|' .. tostring(ok) .. '|' .. err.kind
```
````

The run result is:

````text
true|false|lua
````

The refusal names the function you called and the type of the key:

````text
'pairs' walks boolean, number, and string keys only: the table has a table key, which has no order that is the same in two runs
````

`next` raises the same refusal under its own name. It also raises `'next' walks boolean, number, and string keys only: got a {type} key to go on from` when the key you pass it to go on from is one of these. `{type}` is `table`, `function`, or `userdata`, and it is `lightuserdata` for `json.null`. When a table holds such keys of several types, the refusal names the type that comes first by name, so a table with a function key and a table key reports the function key on every run. Each is a Lua runtime error: `pcall` catches it as an [error value](#error-values) of kind `lua`, and left uncaught it ends the run with code `lua_runtime`.

Such a table still reads and writes by key, so a set of objects works as a lookup, as `seen[a]` does above. A block that must also walk the objects keeps them in a list as well, or gives the table a [`__pairs` metamethod](#tables-with-a-__pairs-metamethod) that walks that list.

### Weak tables

`pairs` and `next` also refuse a weak table, one whose metatable has a `__mode` string with `k` or `v` in it. Lua's garbage collector takes entries out of a weak table at moments a prompt does not choose, so what a walk would find there differs from run to run. The refusal is `'pairs' does not walk a weak table, one whose metatable sets __mode: the entries the collector leaves in it are not the same in two runs`, with `'next'` in place of `'pairs'` when `next` raises it. A weak table still reads and writes by key, and a `__pairs` metamethod on it still takes over `pairs`. `ipairs` and the functions of `table` read a table by its indices, as `t[i]` does, so they take a weak table too.

A weak table is never read as a [sealed view](#encoding-values-with-jsonencode). A sealed view is a table with no entry of its own, and the collector decides whether a weak table has one, so an empty weak table whose metatable has `__index` and `__metatable` raises the refusal. This holds for `pairs` and `next` and for every Engine function that refuses a weak table: `json.encode`, a write to `var`, `models.use`, `tools.adhoc`, `tools.offer`, `tasks.fanout`, the other [`tasks` functions](15-tasks.md#argument-errors-from-the-tasks-functions), the `markdown` functions, and each place where [a table becomes data](#tables-that-become-data).

A sealed view whose `__index` leads to a weak table raises the refusal wherever the Engine reads through a sealed view: in `pairs`, `next`, `json.encode` for its value, a write to `var`, `tasks.fanout`, the other `tasks` functions, the `markdown` functions, the array of a `tools.adhoc` parameter, and each place where a table becomes data. `models.use`, the attributes and the parameters of `tools.adhoc`, and the options of `json.encode` read only the entries that their table itself holds, so to them a sealed view is an empty table, whatever its `__index` leads to. `tools.offer` reads the entries that its array itself holds, or as many as a `__len` gives through the `__index` of the array, and such a read by index takes a weak table, as `t[i]` in your own code does.

A table whose metatable is a weak table is refused as well, wherever the Engine asks a metatable what its table is. The fields of a metatable are its entries, so the collector takes `__index`, `__metatable`, and `__len` out of a weak one, and the table would be a sealed view in one run and an empty table in the next. `pairs`, `next`, `json.encode`, a write to `var`, `tasks.fanout`, the other `tasks` functions, the `markdown` functions, the array of a `tools.adhoc` parameter, `tools.offer`, and each place where a table becomes data put `a table whose metatable is a weak table: the fields the collector leaves in that metatable are not the same in two runs` where the refusal of a weak table has its own words, as in `'pairs' does not walk a table whose metatable is a weak table: the fields the collector leaves in that metatable are not the same in two runs`. Such a table is refused whether it holds entries or not.

The Engine checks a table when it reads it. A table that was weak earlier and no longer is reads as the collector left it. `pairs` checks its table when the loop begins, so a table that the body of the loop makes weak reads as the collector leaves it.

For the check the Engine reads the `__mode` string up to its first `k`, `v`, or zero byte, each time it meets the table. A `__mode` can be as long as the VM's memory limit allows, and any number of tables can share one metatable, so one walk of a value, one call of `pairs`, `next`, `tasks.fanout`, or `tools.offer`, and one check that another `tasks` function makes of a table, may read as many bytes of `__mode` as that limit, 64 MiB by default, and then to the end of the one `__mode` that passes it, which is at most twice the limit. A place where a table becomes data walks a value that it refuses a second time, so it reads at most four times the limit. Past the limit, `pairs`, `next`, `tasks.fanout`, `tools.offer`, and the other `tasks` functions put `a table with {scanned} bytes of __mode to scan: one call scans at most {limit} bytes` where the refusal of a weak table has its own words, with the bytes of `__mode` the call came to and the limit, as in `tasks.join expects a set of tasks, got a table with 68157440 bytes of __mode to scan: one call scans at most 67108864 bytes`. The short form of `tools.adhoc` parameters and each place where a table becomes data raise their error for a value that is too large, whatever else is wrong with the value. `json.encode`, a write to `var`, and the `markdown` functions stop at the first fault of their walk, so they raise their error for a value that is too large when the walk meets no other fault before the limit. Each of these counts the bytes of `__mode` toward the same limit as the bytes of strings. A `__mode` of a few bytes never comes near the limit.

### The same order everywhere

`pairs` and `next` visit a table's keys in the same order on every run, in every section's fresh Lua state, the preamble's included, and in libraries loaded with `require`, instead of stock Lua's unspecified hash order. Two sections that walk the same table see the same key sequence. [`tasks.fanout`](14-fanout.md#fanout-in-one-call) runs the arms of a collection in the order `pairs` visits its keys (see [The order of members](14-fanout.md#the-order-of-members)).

### How `pairs` walks a table

`pairs` is a global in every section, Runvil's deterministic version of stock Lua's `pairs`. `pairs(t)` returns three values, `f, s, init`: the iterator function, the table itself, and nil, so code that reads the second result sees the table, as in stock Lua.

`pairs` fixes the key order once, when it is called, and the loop then advances through that captured order by position. The iterator function keeps its own position and ignores its arguments, so each call to it advances one key, and calls after the last key keep returning `nil, nil`.

`pairs` and `next` read raw values without consulting `__index`, and they visit only keys actually stored in the table. Because the walk reads the table's raw contents and never its length, `__index` and `__len` metamethods do not change which keys are visited, their order, or the values returned.

The exception is a sealed view, an empty table that stands in for a table kept elsewhere, such as a table read from `var` or a table inside `args` (see [Encoding values with `json.encode`](#encoding-values-with-jsonencode)): `pairs` and `next` read through it and visit the entries of that table, in the same fixed order. After `var.t = { zeta = 1, alpha = 2, mid = 3, beta = 4 }`, a `pairs(var.t)` loop visits `alpha, beta, mid, zeta`.

### Changing a table during a `pairs` loop

Clearing keys with `t[k] = nil` while a `pairs` loop runs is safe:

````markdown
---
name: clear-ahead
description: Clears a key before the loop reaches it.
runvil: 0
---

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

Clearing the current key is allowed too, and the walk moves on to the next live key without an error. Clearing each key of `{a = 1, b = 2, c = 3}` as it is visited still visits `a`, `b`, and `c`. The loop runs to the end whatever the cleared key's type: clearing the boolean key `false` inside a loop over `{10, 20, [false] = 'f'}` still visits each key exactly once, `1, 2, false`.

Keys added to a table during a `pairs` loop are not visited by that loop, because the loop sees only the keys captured when it started. For the same reason, a table key added during the loop does not stop it: `pairs` looks for [keys that cannot be walked](#keys-that-cannot-be-walked) once, when it is called. Values are different: the loop reads each value when it reaches the key, so a new value assigned to an existing key the loop has not reached yet is read live.

### Tables with a `__pairs` metamethod

A `__pairs` function in a table's metatable, set with `setmetatable`, takes over `pairs`. `pairs(t)` calls it with the table and returns all its results in place of the ordered walk, even when the table itself is empty:

````markdown
---
name: custom-pairs
description: Walks an empty table through its __pairs metamethod.
runvil: 0
---

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

This `__pairs` returns an iterator function alone, leaving out the state and control values, and `pairs` drives the loop with it. A table without `__pairs` gets the ordered walk.

`pairs` looks up `__pairs` on a table's metatable directly, without going through that metatable's own `__index`. A `__pairs` reachable only through the metatable's `__index` is not found, and the default ordered walk runs.

A userdata whose metatable holds `__pairs` can be passed to `pairs` as well, which iterates the pairs that metamethod yields and returns its results. The section's [message list](11-conversations.md#building-a-message-list) `messages`, and a record read from it, are userdata of this kind.

#### When `pairs` fails

Besides the refusals of [keys that cannot be walked](#keys-that-cannot-be-walked) and of [weak tables](#weak-tables), `pairs` raises a Lua runtime error in two cases. When `__pairs` holds any value other than nil or a function, a callable table included, `pairs` raises `attempt to call a {type} value (metamethod '__pairs')`, naming the type it found. A table there gives:

````text
attempt to call a table value (metamethod '__pairs')
````

When `pairs` gets a non-table value that lacks a `__pairs` metamethod, such as a string, a number, nil, or a userdata without `__pairs`, it raises `bad argument #1 to 'pairs' (table expected, got {type})`, naming the argument position, the expected type, and the type it got. A nil argument gives:

````text
bad argument #1 to 'pairs' (table expected, got nil)
````

Both are Lua runtime errors. `pcall` catches them, and left uncaught they end the run with code `lua_runtime`.

### Stepping with `next`

`next` is a global in every section, Runvil's deterministic version of stock Lua's `next`, and it steps through a table's keys in the same order `pairs` uses:

````markdown
---
name: step
description: Walks a table with next.
runvil: 0
---

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

A stateless generic `for k in next, t do ... end` loop walks keys in the same deterministic order as `pairs`, integer keys from `1` up first and then boolean keys, so `{10, 20, [false] = 'f', [true] = 't'}` visits `1, 2, false, true`.

`next(t)` or `next(t, nil)` returns the first key and its value in the order `pairs` uses, so `next({ only = 1 })` returns `only, 1`. For an empty table it returns `nil`, so `next(t) == nil` tests whether a table is empty. `next(t, k)` returns the key strictly after `k` in that order, along with its value, and when `k` comes last it returns two values, `nil, nil`.

`next` is stateless between calls. It reads every key of the live table on each call to find the one that follows `k`, so a whole loop of `next` over `n` keys costs `n` times `n` reads. On large tables, prefer `pairs`, which sorts the keys once when the loop starts.

`next` works only on a table's own contents, or on the data behind a sealed view, and never calls a `__pairs` metamethod. It accepts only a table and an optional key, so calling `next` with a first argument that is not a table raises a Lua argument error, whose wording comes from the underlying Lua binding rather than from Runvil's own text. It refuses [keys that cannot be walked](#keys-that-cannot-be-walked) and [weak tables](#weak-tables) as `pairs` does.

### Changing a table during a `next` loop

`next` accepts a key that is not in the table, whether it was cleared mid-loop or was never present. The walk resumes at the first key that sorts after that key's place. So during a `next` loop, a key cleared to nil is skipped, and the walk goes on from the cleared key's place.

Clearing keys while stepping with `next`, in `for k in next, t` or in a manual `next(t, k)` loop, never loses a key and never visits one twice, because every key keeps its place whatever happens to the others:

- Clearing each boolean key of `{10, 20, [false] = 'f', [true] = 't'}` as it is visited gives `1, 2, false, true`.
- Clearing every key of `{10, 20, [false] = 'f'}` as it is visited gives `1, 2, false`, and of `{10, [false] = 'f'}` gives `1, false`.
- Over `{ [false] = 'b', [3] = 'c', [7] = 'y' }`, clearing `t[7]` when `k == 7` visits `3, 7, false`.

A key added during a `next` loop is visited when its place lies after the current key and skipped when it lies before, because each step reads the live table.

Both functions are safe for a loop that clears or edits keys while it walks. `pairs` advances a snapshot of the keys taken when the loop starts, so it never visits a key added later, while a raw `next` loop re-reads the live table at each step and costs more on a large table.

### The length of a table with a gap

`#t` is the number of items in a list, a table that holds a value at every index from `1` up to its last. For a table with a gap in those indices, `#t` is some index that holds a value and whose next index holds none, and Lua does not say which. Two runs with the same inputs can get different ones: `#t` of `{ [1] = 1, [2] = 2, [3] = 3, [5] = 5 }` is `3` on one run and `5` on another.

`pairs`, `next`, and `ipairs` do not use the length, so they walk such a table the same way on every run, and `ipairs` stops at the first gap. Everything that measures a table by `#t` follows the length Lua gives: a loop such as `for i = 1, #t do`, and `table.insert`, `table.remove`, `table.concat`, `table.unpack`, and `table.sort`.

A block that needs the same result on every run keeps its lists without a gap. Every other Engine function treats a gap the same way on every run, whatever `#t` says. `json.encode` raises `json.encode: value is missing list item [4]` for the table above (see [When `json.encode` refuses a value](#when-jsonencode-refuses-a-value)). An assignment of it to `var.x` raises `var.x is missing list item [4]` (see [When `var` rejects a write](#when-var-rejects-a-write)). A table that the Engine turns into data, such as the [`item` of a `tasks.spawn`](15-tasks.md#seeding-item), raises an error that names the missing item too (see [Tables that become data](#tables-that-become-data)). [`tasks.fanout`](14-fanout.md#the-order-of-members) ends the list at the first gap, and so does the array of [`tools.offer`](12-tools.md#what-toolsoffer-accepts).

### Tables that become data

The Engine turns a Lua value into JSON data in four places: the `args` of [`tools.call`](12-tools.md#passing-arguments-to-a-tool), the [`item` of a `tasks.spawn`](15-tasks.md#seeding-item) and of each [`tasks.fanout`](14-fanout.md) member, a field of a [message](11-conversations.md), and a global that prose names. A table that holds a value at index `1`, or that `json.array` marked, becomes a list of its items, and what its other keys hold is left out, whether the key is a name, a boolean, a number with a fraction, or a whole number below 1. Any other table becomes an object, whose keys must be strings.

Three kinds of table have no JSON form here, and each raises an error at the call:

- A list with a gap. The reason names the place and the first missing item: `{ rows = { [1] = 'a', [2] = 'b', [3] = 'c', [5] = 'e' } }` gives `value.rows is missing list item [4]`, and `{ [2] = 'b' }` gives `value is missing list item [1]`.
- A weak table. The reason reads `value.w is a weak table, one whose metatable sets __mode: the entries the collector leaves in it are not the same in two runs` (see [Weak tables](#weak-tables)).
- A table nested more than 127 tables deep, the outermost being the first. The reason ends `is nested more than 127 levels deep`. A value 127 deep is the deepest whose JSON text reads back.

A table that contains itself has no JSON form either, and its reason reads `recursive table detected`. That holds when the table contains itself through a sealed table too, such as a view of its own data. A sealed table that only a function can read, such as `sys`, becomes an empty object and counts as one level of the 127.

Each place puts its own words in front of the reason, as `tasks.spawn` does in `item must be a JSON-representable value: deserialize error: value.rows is missing list item [4]`. The three rules apply to the tables that become data, so what another key of a list holds is not checked. A number that is NaN or an infinity becomes `null` with no error, so `{ 0/0, 1/0 }` becomes `[null,null]`, where `json.encode` and a write to `var` refuse such a number.

A [record view](11-conversations.md#what-a-record-view-allows) inside the value becomes the record's data, and counts as that data. Its nesting adds to the tables it sits under, so `{ prev = messages[1] }` is one level deeper than the record is, and a view whose record would end more than 127 levels deep raises the same error as a table there. Its values and the bytes of its strings count toward the two size limits below.

When several entries of one table cannot become data, the error is about the entry whose key comes first: boolean keys, then number keys by value, then string keys in bytewise order. So `{ a = { [1] = 1, [3] = 3 }, b = function() end }` is refused for the gap in the list under `a` on every run, with `value.a is missing list item [2]`. A key that is a table, a function, a userdata, or a thread has no place in that order, so its error names only the key's type and the table that holds it: `{ rows = { [{}] = 1 } }` gives `value.rows has a table key; object keys must be strings`, whatever the key table holds. Of several such keys, the error names the type that comes first by name.

One value may hold as many values as one eighth of the VM's memory limit, 8,388,608 at the default of 64 MiB, and a larger one raises `the value is too large to convert`. A value also has a limit on its text: when its strings, keys included and each counted every time it occurs, hold more bytes than the memory limit, the error is `the value is too large to convert`, in place of any reason above. Both limits count what a list holds under its other keys too, though that is left out of the data. Every string key is the name of a field, whatever its text.

## Keeping state in `var`

After this section you can keep your own values in `var`, read them in later blocks and later sections, and predict exactly what `var` accepts.

````markdown
---
name: greeting
description: Keeps a value in var and reads it back.
runvil: 0
---

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

`First` sets a plain global, `note`, and a `var` field, `var.trail`. Each section entry gets a fresh section VM (see [A fresh section VM for every section entry](#a-fresh-section-vm-for-every-section-entry)), so `note` reads as `nil` in `Second`, while `var.trail` still holds `ab`. A section's final `var` becomes the starting `var` of the next section, so a value one section stores in `var` is readable in later sections of the same walk, both across fall-through and after a [`jump`](08-jump-and-call.md#jumping-to-a-section).

A value you assign to a `var` field stays in `var` after the block that set it ends. Whenever the run needs that state, it reads `var` back as JSON data, as [When the run reads `var` back](#when-the-run-reads-var-back) explains.

`var` is present in every section, the preamble included. You can read and write it in every walked section, in every [arm](14-fanout.md#what-an-arm-sees) of [`tasks.fanout`](14-fanout.md#fanout-in-one-call), and in every chain that [`call`](08-jump-and-call.md#calling-a-section) runs. A chain is a walk with its own entries (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). How a call, a fanout arm, or a task starts from its own copy of `var` is covered in [Call args and `var`](08-jump-and-call.md#call-args-and-var). A prompt run with [`execute`](08-jump-and-call.md#executing-another-prompt) starts with an empty `var` in place of a copy, and its writes end with it (see [What crosses an execute](08-jump-and-call.md#what-crosses-an-execute)).

### Starting values and computed keys

Reading a `var` key that was never set is allowed and returns `nil`, so `var.key or default` supplies a starting value:

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

### `var` in the preamble

A [preamble](02-file-structure.md#the-preamble) that holds Lua starts `var` and hands it to the section after it:

````markdown
---
name: executions
description: Counts in two preamble blocks and reads the count in a section.
runvil: 0
---

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

`var` is an empty table in the first section a chain enters, and here that section is the preamble, so its first block finds `var.executions` unset and stores `1`. `var` writes accumulate across the Lua blocks of the preamble as in any section, so the counter bumped in two blocks reads `2`. The preamble and the sections after it are entries of one chain (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)), so whatever `var` holds when the preamble ends is the `var` that `Result` starts with, and every later section of the walk reads it too. In a prompt with no preamble, the first top-level section is the one that starts with `var` set to an empty table.

`Result` builds its return value from a `var` value with Lua's `..` operator. A number operand turns into text in the result, which is why `'count:' .. var.executions` gives `count:2`.

### Showing a `var` value in prose

You store a value on `var` in one Lua block and read it in later prose of the same walk through a [placeholder](07-substitution.md#placeholders-in-prose):

````markdown
---
name: greet
description: Shows a var value in prose.
runvil: 0
---

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

JSON data is the set of values JSON can express: numbers, strings, booleans, and tables of arrays and objects built from them, nested to any depth. A `var` field holds only JSON data, and `var` as a whole nests at most 127 levels deep (see [When `var` rejects a write](#when-var-rejects-a-write)). With this block as the Lua block of `Main` in the `greeting` prompt, the run succeeds:

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

As the Lua block of `Main` in the `greeting` prompt, this returns `yes`, and `var` reads back as `{ "t": { "kept": "yes" } }`. Every nested write passes the same JSON data check as a top-level write.

### Assigning a table stores a copy

Assigning a table to a `var` field stores a fresh copy, so later edits to the original Lua table leave `var` unchanged:

````lua
local list = { 'one', 'two' }
var.list = list
list[2] = 'changed'
return var.list[2]
````

As the Lua block of `Main` in the `greeting` prompt, this returns `two`. To change the stored value, write through `var` itself, as in `var.list[2] = 'three'`, after which `var.list[2]` reads `three`.

### Removing a key

Assigning `nil` to a `var` field passes the guard and removes that key:

````lua
var.k = 1
var.kept = 'yes'
var.k = nil
return tostring(var.k == nil)
````

As the Lua block of `Main` in the `greeting` prompt, this returns `true`, and `var` reads back as `{ "kept": "yes" }`. The removed key reads as `nil` and is gone from the read-back `var` data, and removing one key leaves the other keys `var` already held untouched.

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

As the Lua block of `Main` in the `greeting` prompt, this returns `abc 3 abc`. The values you store through `var` live in hidden storage behind a guard, so the `var` table itself stays empty, and so does every table you read from it. Each one is a [sealed view](#encoding-values-with-jsonencode): a stand-in that passes every read to the stored data. Your code reaches that hidden storage only through the guard, because it lives in the Lua registry and the [sandbox](#the-sandbox) leaves out the `debug` library.

Because every read passes through, a table read from `var` measures and iterates like the table it stands for. With `var.items` as in the block above:

- The length operator `#` gives the length of the stored list, which is the `3` in the result above.
- `table.concat` joins the stored items with or without an end index, so `table.concat(var.items, ', ')` gives `a, b, c`.
- `ipairs` visits the items in order, as the loop above does, and `table.unpack(var.items)` returns `a`, `b`, and `c`.
- `pairs` visits the stored entries in the [deterministic key order](#deterministic-table-iteration), so `pairs(var.items)` visits the keys `1`, `2`, and `3`, and `pairs(var)` visits the one key `items`.
- `next` steps through the same entries, so `next(var.items)` returns `1` and `a`. On an empty `var` table it returns `nil`, so `next(t) == nil` tests whether a `var` table `t` is empty.

You append to a list in `var` as you append to any list. After `var.items[#var.items + 1] = 'd'` and then `table.insert(var.items, 'e')`, `json.encode(var.items)` gives `["a","b","c","d","e"]`.

`json.encode` reads through a `var` table to the stored data in the same way, so in the block above, `json.encode(var.items)` gives `["a","b","c"]`, and `json.encode(var)` gives everything `var` holds as one JSON object (see [Encoding values with `json.encode`](#encoding-values-with-jsonencode)).

`getmetatable` on `var`, or on any table nested inside it, returns the string `var is guarded`, and your code cannot replace that guard, `setmetatable` included: `setmetatable(var, {})` raises the Lua error `cannot change a protected metatable` at your line.

### When the run reads `var` back

The run reads `var` back as JSON data at fixed points:

- when a section ends by falling through, jumping, or returning
- when prose renders
- when `call`, `tasks.fanout`, or a task start takes its copy of `var`

The run does not read `var` back between the blocks of one section, because they share one Lua state. A task start is a [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) call.

### When `var` rejects a write

Only JSON data can be assigned to a `var` field. The guard checks each write at the assigning statement, so you can catch a rejected write and keep going:

````markdown
---
name: guarded
description: Catches a rejected var write and keeps going.
runvil: 0
---

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

`var` checks an assigned table deeply, by the rules of [`json.encode`](#when-jsonencode-refuses-a-value), so a table that JSON cannot hold fails at the assigning line too. The message names the failing part by its path under `var`:

| Write | Message |
|---|---|
| `var.t = { f = function() end }` | `var.t.f must be JSON data, got function` |
| `var.t = { [true] = 1 }` | `var.t has a boolean key; object keys must be strings` |
| `var.t = { 'a', name = 'x' }` | `var.t mixes list items and named keys` |
| `var.t = { [1] = 'a', [3] = 'c' }` | `var.t is missing list item [2]` |
| `var.n = 0/0` | `var.n is not a finite number` |

A float with no fractional part stays a float in `var`: after `var.x = 2.0`, `math.type(var.x)` is `float`.

The guard checks the key of a write too, because a table of JSON data is an object, whose keys are strings, or a list, whose keys are the indexes 1 to its length. `var` itself is an object, so it takes string keys alone: `var[true] = 1` raises `a key of var must be a string, got boolean`, and `var[1] = 'x'` raises the same message ending in `got integer`. A table inside `var` takes a write that leaves it a list or an object. With `var.t = {}` and `var.list = { 'a' }`:

| Write | Message |
|---|---|
| `var.t[2.5] = 1` | `a key of var.t must be a string or a list index, got number` |
| `var.t[0] = 'x'` | `a list index of var.t must be 1 or more, got 0` |
| `var.list[3] = 'c'` | `var.list[3] is past the end of the list, whose length is 1` |
| `var.list.name = 'x'` | `var.list.name is a named field, and var.list is a list` |
| `var.t.a = 1` and then `var.t[1] = 'x'` | `var.t[1] is a list item, and var.t holds named fields` |

A path in these messages shows a key longer than 64 bytes as its first 64 bytes, without a character that the cut would split, and its length, as in `var.kkk... (1000 bytes)`.

A string key must be valid UTF-8. `var['\xff'] = 1` raises `var.{key} is not valid UTF-8`, where the path shows each byte that is not UTF-8 as the replacement character U+FFFD. A write to a table inside `var` raises it the same way, as `var.t['\xff'] = 1` does.

`var.list[2] = 'b'` appends, and `var.list[1] = 'z'` gives the first item a new value. Assigning `nil` removes a named field, or the last item of a list. The removal of an earlier item would leave a gap, so with two items in `var.list`, `var.list[1] = nil` raises `var.list[1] is not the last item of the list, whose length is 2; remove it with table.remove`, and `table.remove(var.list, 1)` removes the item and moves the rest down.
A table that is shared at many levels can also be too large to store, because the conversion to JSON copies it once for every place it appears. Before it converts an assigned table, `var` spends the same value and byte budgets that [`json.encode`](#size-and-nesting-limits) does, and a table that runs the budget out raises this error at the assigning line, with the full target path in place of `{target}`:

````text
{target} is too large to store
````

After `local t = {} for i = 1, 40 do t = { t, t } end`, the write `var.x = t` raises `var.x is too large to store`. Like any rejected write it is dropped whole. A table that contains itself is refused for another reason, as a cycle: after `local t = {} t.self = t`, the write `var.c = t` raises `var.c.self contains itself`.

`var` as a whole nests at most 127 levels deep, and `var` itself is the first level. So a value in a field of `var` nests at most 126 tables deep, one fewer than [`json.encode`](#size-and-nesting-limits) accepts. A write that would put a table at level 128 raises `{path} is nested more than 127 levels deep` at the assigning line, with the path of that table. The levels count the same whether one write nests the tables or each write adds one: after `local t = var for i = 1, 126 do t.n = {} t = t.n end`, one more `t.n = {}` raises it, and so does `var.n = v` for a `v` that is 127 tables deep. The limit keeps `var` within the nesting that JSON text reads back, because the Engine hands `var` whole to a called chain and to a task.

The guard raises its error at the assigning statement itself, not later when `var` is saved. That is why a `pcall` around that one write catches it, the block keeps running, and later `var` writes still work, as `var.kept = 'yes'` does in the `guarded` prompt. A rejected write is dropped whole, so `var` holds only the writes that passed the guard: after a rejected `var.t.f = function() end` following `var.t.kept = 'yes'`, `var` reads back as exactly `{ "t": { "kept": "yes" } }`.

`pcall` returns `false` and an [error value](#error-values) whose `tostring` text contains the guard's message, so `tostring(err):match('must be JSON data')` finds it. In a section, the caught error value has error kind `lua`, and its `message` field gives the guard's text as [author prose](#reading-kind-and-message) after the assigning line, so in the `guarded` prompt `tostring(err)` returns `guarded.md:10: var.handler must be JSON data, got function. The Lua code failed at runtime. Fix the code at this location.` [Raising and catching errors](#raising-and-catching-errors) covers `pcall` in full.

Left uncaught, assigning a non-JSON value such as a function to a `var` field fails the run at the assigning line with [error code](16-limits-and-errors.md#run-error-codes) `lua_runtime`.

### Keeping the `var` global in place

Change `var` only through its fields, and keep the `var` global itself in place. Reassigning the `var` global itself makes the next read-back of `var` fail the run with this Lua error, rather than quietly keeping the data `var` held before the reassignment:

````text
the `var` global was reassigned; write `var.<field>` instead
````

The run's error code is `lua_runtime`. When a section entry has reassigned the `var` global, the read-back at the entry's end or at its `jump` fails the run, and the entry does not count as completed. The run's message ends with ` (at {file}:{line})` for the first Lua line of the last block that ran, which is the block that called `jump` when the read-back comes at a jump. An entry that returns fails the same way, because the read-back still runs before the `return` takes effect.

## Encoding and decoding JSON with `json`

After this section you can turn JSON text into Lua values and back, write and read JSON Lines, keep an empty list as a list, and dump any table to look at it.

````markdown
---
name: json-round-trip
description: Decodes JSON text, edits it, and encodes it again.
runvil: 0
---

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

`json` is in every section VM, the preamble, fanout arms, spawned tasks, and libraries loaded with `require` included, and a Pack prelude's functions can call it too, so you never import it. Encoding is strict: a value that is not JSON data raises an error that names exactly where the value sits, unless you ask for a [debug dump](#dumping-any-value-in-debug-mode). Output and error text are identical on every run, because object keys are written sorted and table keys are always checked in one fixed order. Nulls and lists are recognized by the values themselves, whatever the `json` table holds, so a value encodes the same way after you reassign one of that table's fields.

Every error a `json` function raises is an [error value](#error-values) of kind `lua`, so `pcall` catches it, and `tostring(err)` gives its message: the error's own text, such as `json.decode: expected value at line 1 column 1`, as [author prose](#reading-kind-and-message) after the line of the failing call. The messages this section quotes are that own text. Left uncaught, it ends the block like any other `lua` failure (see [What an uncaught error ends](#what-an-uncaught-error-ends)).

### Decoding JSON text with `json.decode`

`json.decode(text)` returns the Lua value of the one JSON document in `text`:

````markdown
---
name: decoded-types
description: Shows the Lua values JSON text decodes to.
runvil: 0
---

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

With `text` holding `x`, this block returns `bad JSON: {file}:{line}: json.decode: expected value at line 1 column 1. The Lua code failed at runtime. Fix the code at this location.`, where `{file}:{line}` names the line of `pcall(json.decode, text)`.

You decode only JSON that arrives as text, such as a plain tool's output, a store file, or a model reply. A [structured tool](12-tools.md#structured-output) already hands your Lua a parsed table built the same way, with marked lists and `json.null`. A web tool's text arrives inside the untrusted envelope, so you cut the JSON out with `string.match` before decoding it, as [Decoding a fetched JSON body](13-web-fetch-and-search.md#decoding-a-fetched-json-body) shows.

Decoded strings are plain Lua strings, never wrapped in the run's untrusted envelope, even when the JSON came from the web or a model. Before such text reaches a model, wrap it with [`untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted).

### Null values and `json.null`

`json.null` is the Lua value for JSON null:

````markdown
---
name: null-check
description: Tells a JSON null apart from a missing field.
runvil: 0
---

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

Every JSON `null` decodes to `json.null`, and `json.null` is what you put in a table to encode `null`. A table drops a field set to `nil`, so `gone` is left out of the encoded object, while `note = json.null` keeps its key. In a list, `json.null` holds its position, so `json.encode({ json.null, 1 })` gives `[null,1]`.

`json.null` is a light userdata, so `type(json.null)` is `userdata`, and most error messages name its type `lightuserdata`. Like every userdata it is truthy, so `if row.note then` passes for a null, as the `truthy` in the result shows. Test for null with `v == json.null`, and for an absent field with `v == nil`.

Outside `json.encode`, `json.null` passes the Engine's other conversions to JSON too. The `var` guard accepts it and stores JSON null (see [When `var` rejects a write](#when-var-rejects-a-write)), a `tools.call` argument table or a task's `item` can hold it, and a fanout member that is `json.null` runs its arm with `item == json.null`. A null that the arguments keep, in an undeclared field of an object that says `additionalProperties: true`, reads as `json.null` in `args` too, as [Missing fields and null values](06-arguments.md#missing-fields-and-null-values) shows. A prose placeholder still refuses it: naming a bare global that holds `json.null`, as in `{{ nothing }}` after `nothing = json.null`, fails with `missing {{ nothing }}` and substitution kind `NullValue`, the same failure a `var` field holding null gives (see [Substitution error kinds](07-substitution.md#substitution-error-kinds)).

### Keeping empty lists with `json.array`

An empty Lua table could be an empty list or an empty object, and `json.array` says which one you mean:

````markdown
---
name: empty-list
description: Keeps a filtered list as a list when nothing passes the filter.
runvil: 0
---

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

Every word is at most five letters long, so both tables stay empty, yet they encode differently. An unmarked empty table encodes as an object, `{}`, because the marker is what says an empty table is a list. `long` is a marked list, so it encodes as an array, `[]`, however many items it ends up with. Wrap any list that may end up empty with `json.array` before it leaves your code. `json.object` reads nil, because a plain `{}` already encodes as an object.

`json.array()` returns a new empty marked list, and so does `json.array(nil)`. `json.array(t)` marks the table `t` as a list in place and returns `t` itself, so `json.array(t) == t` holds. A table that is already a marked list, such as a decoded list, comes back unchanged.

A marked list has a metatable whose only field is `__metatable = false`. So `getmetatable` on a marked list returns `false`, and `setmetatable` on one raises `cannot change a protected metatable` (see [`setmetatable` and `getmetatable` on other values](#setmetatable-and-getmetatable-on-other-values)). Every list that `json.decode` returns is marked, and so is every list in a [structured tool result](12-tools.md#structured-output). Otherwise a marked list is an ordinary writable table: `#`, `ipairs`, `table.insert`, and `table.remove` work on it, and a decoded list emptied with `table.remove` still encodes as `[]`.

`json.array` raises two errors:

- `json.array expects a table or no argument, got {type}`, for any value other than a table or nil, so `json.array(5)` reports `got integer`.
- `json.array: the table already has a metatable`, for a table whose metatable is anything other than the list marker. That covers `var` and every table read from it, `args` and every table read from it, sealed Pack globals, fanout results, `_G`, and any table you gave a metatable yourself.

The marker works beyond `json.encode`. Every place the Engine turns a Lua table into JSON honors it, so a marked list goes out as `[]` in a `tools.call` argument table, a task's `item`, a message record, and `var`, where a plain empty table goes out as `{}`. An empty list stored in `var` or held in `args` keeps its marker, so after `var.items = json.array()`, `var` reads back as `{ "items": [] }`, and a `{{ var.items }}` placeholder renders `[]` (see [When the run reads `var` back](#when-the-run-reads-var-back)). `var.items = {}` still reads back as `{ "items": {} }`.

### Encoding values with `json.encode`

`json.encode(value [, options])` returns `value` as compact JSON text, with no spaces and with object keys sorted bytewise:

````markdown
---
name: encode-var
description: Encodes what var holds.
runvil: 0
---

## Main

```lua
var.cfg = { mode = 'fast', sizes = { 2.0, 1.5 } }
var.empty = json.array()
return json.encode(var) .. ' ' .. #var.cfg.sizes .. ' ' .. json.encode(var.cfg.sizes)
```
````

The run result is:

````text
{"cfg":{"mode":"fast","sizes":[2,1.5]},"empty":[]} 2 [2,1.5]
````

`json.encode` reads `var` and its nested tables through to the data they store, `#var.cfg.sizes` counts the two sizes stored there, and the float `2.0` comes out as `2`. The only option is `debug`, which [Dumping any value in debug mode](#dumping-any-value-in-debug-mode) covers.

Scalars encode this way:

- `json.null` encodes as `null`, and so does `nil` passed as the whole value, so `json.encode()` gives `null`. Inside a table, a key set to `nil` is dropped.
- Booleans encode as `true` and `false`.
- Integers encode as integers. A float with no fractional part, from -2^63 up to but not including 2^63, encodes as an integer too, so `2.0` gives `2`, `-0.0` gives `0`, and `-2^63` gives `-9223372036854775808`, while `2^63` stays a float.
- Every other finite float is written as the shortest text that reads back as exactly the same number, so `1.5` gives `1.5`, and `1e300` gives `1e+300`.
- NaN and the infinities, such as `0/0` and `math.huge`, raise an error.
- Strings must be valid UTF-8. A newline inside a string is written as `\n`, so every encoded value is one line of text.

Tables encode this way:

- A marked list encodes as an array, and its keys must be exactly 1 to n, where n may be 0.
- An unmarked empty table encodes as `{}`.
- An unmarked table whose keys are exactly 1 to n encodes as an array.
- An unmarked table whose keys are all strings encodes as an object, and every key must be valid UTF-8.
- Any other table raises an error, such as a table that mixes list items and named keys, a list with a gap, or a table with a boolean or float key.
- A table's keys are checked before any of its values are encoded.
- A table that contains itself raises an error, while the same table in two places is fine: after `local s = { 1 }`, `json.encode({ a = s, b = { s } })` gives `{"a":[1],"b":[[1]]}`.
- A table nested more than 127 levels deep raises an error (see [Size and nesting limits](#size-and-nesting-limits)).
- A value whose walk spends more than the size budget raises an error, as a table shared at every level of a deep nest does (see [Size and nesting limits](#size-and-nesting-limits)).
- Any metatable other than the list marker or a sealed view's is ignored, and the table's own entries are encoded, so a `__pairs` or `__tostring` metamethod never changes the output.

Functions, threads, and every userdata other than `json.null` are not JSON data, so they raise an error. That includes Engine objects such as tool objects, the `messages` list, and the record views that `messages[i]` returns. To encode a message record, encode `messages[i].content` or a plain table you build from its fields.

A sealed view is a table that is itself empty and whose metatable has both `__index` and `__metatable`. `var` and every table read from it, `args` and every table read from it, sealed Pack globals, and [fanout results](14-fanout.md#reading-the-results) are sealed views, and each of them measures and iterates like the data it stands for: `#` gives the data's length, and `pairs` and `next` visit the data's entries. When a view's `__index` is a table, `json.encode` encodes that table in the view's place, so `json.encode(var)` gives everything `var` holds. For a fanout arm over the member `alpha` whose worker returns `item .. '-' .. sys.index`, `json.encode(r[1])` gives `{"exhausted":false,"item":"alpha","ok":true,"text":"alpha-1"}`. When a view's `__index` is a function, as in `sys`, the view cannot be read, and `json.encode(sys)` raises this error:

````text
json.encode: value cannot be read (sys is sealed); copy the fields you need into a plain table
````

The parenthesis holds the view's `__metatable` text and is left out when that is not a string. Inside a Pack prelude's function, `var` is a read-only view of this kind, so `json.encode(var)` there raises the same error with `(var is read-only)`. A table whose `__index` is something other than a table or a function counts as an ordinary table, and it encodes as its own entries.

### When `json.encode` refuses a value

A strict encode raises at the first value that is not JSON data:

````markdown
---
name: encode-refused
description: Catches a value that json.encode refuses.
runvil: 0
---

```lua
local ok, err = pcall(json.encode, { items = { 'a', function() end } })
return err.kind .. '|' .. tostring(err)
```
````

The run result is:

````text
lua|encode-refused.md:8: json.encode: value.items[2] must be JSON data, got function. The Lua code failed at runtime. Fix the code at this location.
````

Every message names the failing value by its path. The path starts at `value`, the value you passed, and adds `.name` for each string key and `[k]` for any other key, so `value.items[2]` is the second item of the `items` field. A string key that is not valid UTF-8 appears in the path with replacement characters, and a key longer than 64 bytes appears as its first 64 bytes, without a character that the cut would split, and its length, as in `value.kkk... (1000 bytes)` with 64 letters `k`. The size refusal is the one message with no path, because the walk stops wherever the budget runs out. These are the messages:

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
| A [weak table](#weak-tables) | `json.encode: {path} is a weak table, one whose metatable sets __mode: the entries the collector leaves in it are not the same in two runs` |
| A table whose metatable is a [weak table](#weak-tables) | `json.encode: {path} is a table whose metatable is a weak table: the fields the collector leaves in that metatable are not the same in two runs` |
| A table more than 127 levels deep | `json.encode: {path} is nested more than 127 levels deep` |
| A walk that spends more than the size budget | `json.encode: value is too large to encode` |
| A sealed view whose `__index` is a function | `json.encode: {path} cannot be read ({label}); copy the fields you need into a plain table` |

`{type}`, like `{kind}` in the table above, is the type name the `json` functions and most Engine argument errors use, which is not always what Lua's `type` returns: an integer reads `integer`, a float reads `number`, and `json.null` or any other light userdata reads `lightuserdata`. So `json.decode(5)` raises `json.decode expects a string, got integer`, `json.decode(1.5)` raises `json.decode expects a string, got number`, and `json.array(json.null)` raises `json.array expects a table or no argument, got lightuserdata`. A few refusals name types their own way: the `models.use` target refusal and the `tasks.fanout` first-argument refusal read any number as `number`, and the `models.loop` and `models.compact` refusals and the argument checks the `tasks` functions make themselves read every userdata, `json.null` included, as `userdata`.

A table that breaks more than one rule reports the same error on every run. An unmarked table reports a key that is neither an integer nor a string first, then a mix of integer and string keys, then a missing list item. Because keys are checked before values, `{ [1] = function() end, a = 1 }` reports `mixes list items and named keys`, not the function. For a missing item, `{item}` is the first of 1 to n that is absent, where n is the table's count of integer keys: `{ [1] = 'a', [3] = 'c' }` reports `[2]`, `{ [0] = 'a' }` reports `[1]`, and `{ [-1] = 'a', [1] = 'b' }` reports `[2]`.

Within a table, `json.encode` visits keys in one fixed order: booleans, then numbers by value, then strings bytewise. A string key that is not valid UTF-8 sorts bytewise among the strings, and an infinite number key sorts by value among the numbers, so `-math.huge` comes first and `math.huge` last. This is the order [Then booleans, the other numbers, and strings](#then-booleans-the-other-numbers-and-strings) describes, with one difference: `json.encode` sorts every number key by value, where `pairs` visits the integer keys from `1` up ahead of the booleans. Table, function, userdata, and thread keys come last, ranked by type name, so a table with both a table key and a function key always reports `has a function key`.

The options argument is checked before the value, so `json.encode(function() end, 5)` raises the options error, not the function's. Options must be nil or a table whose only key is `debug`:

- A value that is not a table raises `json.encode: options must be a table, got {type}`.
- A [weak table](#weak-tables) raises `json.encode: options must not be a weak table, one whose metatable sets __mode: the entries the collector leaves in it are not the same in two runs`, before any of its keys is read.
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

Debug mode encodes every value that stays within the [size budget](#size-and-nesting-limits). It follows the same rules as a strict encode, but writes a string in place of each failure:

- A function becomes `"[function]"`, a thread becomes `"[thread]"`, and every userdata other than `json.null` becomes `"[userdata]"`, whether or not it defines `__tostring`.
- A table that contains itself becomes `"[cycle]"`, a table more than 127 levels deep becomes `"[too deep]"`, a weak table becomes `"[weak table]"`, and a table whose metatable is a weak table becomes `"[weak metatable]"`.
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

The run result is:

````text
[{"n":1,"tag":"a"},{"n":2,"tag":"b"},{"n":3,"tag":null}]
````

`json.encode_lines(list [, options])` encodes each item of `list` by the `json.encode` rules and ends each one with a newline, so an empty list gives `""`. Because the last record ends in a newline too, the result can go straight onto the end of a file that already ends in a record. [`store.append`](09-the-store.md#appending-to-a-file) puts the new text right after the old, so `store.append('rows.jsonl', json.encode_lines(rows))` appends whole records. The list can be a marked list, an empty table, a table with keys 1 to n, or a sealed view of one, such as `var.rows`, and `json.encode_lines` takes the same `debug` option as `json.encode`.

`json.decode_lines(text)` returns a marked list of every JSON value in `text`, in order. Any whitespace separates values, so blank lines and `\r\n` line endings are fine, several values can share a line, and one value can span several lines. Empty or blank text gives an empty marked list. It takes at most 8,388,608 bytes, like `json.decode`.

`json.decode_lines` raises the same three errors as `json.decode`, with `json.decode_lines` in place of `json.decode`. A parser message counts lines and columns across the whole text, so a bad third record reports `at line 3`, as in `json.decode_lines: expected value at line 3 column 6`.

`json.encode_lines` checks its list argument before it encodes any item, and it raises these errors:

- `json.encode_lines expects a list, got {type}`, for a value that is not a table.
- `json.encode_lines expects a list, got a table with a {type} key`, for an unmarked table that has keys but no list items, naming the first key in visiting order. An object such as `{ a = 1 }` reports `got a table with a string key`.
- The table rules of `json.encode` with the `json.encode_lines:` prefix, for a table that has list items but is still not a list, as in `json.encode_lines: value mixes list items and named keys` or `json.encode_lines: value is missing list item [2]`.
- `json.encode_lines: value cannot be read (sys is sealed); copy the fields you need into a plain table`, for a sealed view that cannot be read, such as `sys`.
- The four options errors, with the `json.encode_lines:` prefix, as in `json.encode_lines: unknown option pretty`.
- `json.encode_lines: value is too large to encode`, when the items of one call together spend more than the size budget. The items share one budget, so the refusal can come from any item or from the list as a whole.

An error inside an item names the item by its path under the list, as in `json.encode_lines: value[2].name must be JSON data, got function`. The list stays on the cycle path while its items are encoded, so a list that contains itself raises `json.encode_lines: value[1] contains itself`. In debug mode the argument errors above and the size refusal still raise, and only the other failures inside items become placeholders.

### Size and nesting limits

`json.decode` and `json.decode_lines` each take at most 8,388,608 bytes of text, the same cap [`web/fetch`](13-web-fetch-and-search.md#fetching-a-page) applies to a JSON body. A longer string raises `json.decode: input exceeds the 8388608-byte limit`, or the same message with the `json.decode_lines:` prefix, before any parsing starts. `json.encode` can write a longer text than that, and `json.decode` refuses such a text for its length.

Nesting is limited to 127 levels in both directions, so `json.decode` accepts the nesting of every text that `json.encode` writes:

- `json.decode` accepts 127 arrays and objects nested inside each other and rejects the 128th with the parser's own message, as in `json.decode: recursion limit exceeded at line 1 column 128` for text that opens with 128 brackets.
- `json.encode` counts the value you pass as level 1 and raises `json.encode: {path} is nested more than 127 levels deep` for a table at level 128. The path names that table, such as `value` followed by 127 steps of `[1]` for lists nested in lists. In debug mode the table becomes `"[too deep]"` instead.

A sealed view and the data it stands for count as one level, so `json.encode(var)` writes a `var` that is at its own limit of 127 levels. At each table, `json.encode` checks for a cycle before it checks the depth, so a loop of 127 tables or fewer reports `contains itself`, even when the repeated table falls at level 128. A longer loop reaches the depth limit before it repeats, so it reports the depth.

`json.encode` and `json.encode_lines` also bound the work of one call, because a table can be reachable from many places while the Lua heap holds it once. After `local t = {} for i = 1, 40 do t = { t, t } end`, the heap holds 41 small tables, but `json.encode(t)` would write about 2^41 values. Each call therefore spends a budget while it walks, and a call that runs the budget out stops and raises this error, or the same message with the `json.encode_lines:` prefix, with no path:

````text
json.encode: value is too large to encode
````

The budget comes from the Lua memory limit of the section VM, `lua_memory_bytes`, which is 64 MiB by default (see [Lua memory and log quotas](16-limits-and-errors.md#lua-memory-and-log-quotas)), so a value that fits the heap once is never refused for its size alone. The walk spends it in three ways:

- One value for each value the walk visits, up to one eighth of the memory limit, which is 8,388,608 values at the default. A table counts once for every place it appears, and each sealed view the walk reads through counts too. At the default limit this is enough for any text that `json.decode` accepts, so a value decoded from text within the 8,388,608-byte cap encodes again.
- One byte for each byte of every string value and every string key the walk visits, up to the memory limit itself, which is 67,108,864 bytes at the default. A string that appears in many places counts at each place.
- At most 100,000 keys that are not numbers or booleans on one path, counting the keys of the table being encoded and of every table above it on the way down. Integer keys stay out of this count, so a list of a million strings still encodes, while an object with more than 100,000 string keys on one path raises the same error.

The budget bounds the values and the text of one call. It does not bound the memory that the data takes while the call builds it, which is outside the Lua heap. That memory is up to about 670 bytes for each value, most for a table of one key, so a table that many places share can take up to about 84 times the memory limit, 5.2 GiB at the default, for one `json.encode` call, one `var` write, or one table that becomes data.

Debug mode raises this error too. A `var` write spends the same value and byte budgets before it converts the value, as [When `var` rejects a write](#when-var-rejects-a-write) describes.

## Parsing Markdown with `markdown`

After this section you can parse Markdown text into plain Lua tables, find a section or a single block in it, and write blocks back as Markdown text.

````markdown
---
name: markdown-parse
description: Parses a Markdown document and reads a section, a list, and a fence from it.
runvil: 0
---

## Main

```lua
local md = markdown.parse(prompt.section('Reply', 'fence markdown').text)
local findings = markdown.section(md, '/Report/Findings', 'list').items
local data = json.decode(markdown.section(md, '//Data', 'fence json').text)
return #md .. '|' .. md[1].title .. '|' .. findings[2] .. '|' .. data.count
```

### Reply

```markdown
# Report

Two findings.

## Findings

- tides follow the moon
- storms raise the sea

## Data

~~~json
{ "count": 2 }
~~~
```
````

The run result is:

````text
1|Report|storms raise the sea|2
````

`markdown.parse` turns Markdown text into a list of blocks, here one block, the section `Report` with everything under it. `markdown.section` then finds a section by a path of titles and picks one block in it: the list under `Findings`, and the `json` fence under `Data`, found at any depth by `//Data`. The text here comes from a fence in the prompt, read with [`prompt.section`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptsection_or). A model reply or a store file parses the same way.

`markdown` is an Engine global, a plain table like `json`, with five functions:

- `markdown.parse(text)` parses Markdown text into a list of blocks.
- `markdown.section(blocks, path)` finds one section, and `markdown.section(blocks, path, selector, n?)` finds one block of that section. Both raise when what they name is not there.
- `markdown.section_or` takes the same arguments and returns `nil` when the section or the block is not there.
- `markdown.select(blocks, selector)` collects every block of one kind, at any depth, or every block of any kind.
- `markdown.render(blocks)` writes a list of blocks back as Markdown text. For a parsed document, that is the text it was parsed from.

`markdown` is in every section VM, the preamble, fanout arms, spawned tasks, and libraries loaded with `require` included, so you never import it. It takes text and never a file name: you read a file with [`store.read`](09-the-store.md#writing-and-reading-files) and parse what it returns. Every function is pure. The same input gives the same result, and each of them returns without performing an effect or suspending. All of the text is read as data, code and placeholders included, so a `{{ name }}` placeholder comes back as written.

Every error a `markdown` function raises is an [error value](#error-values) of kind `lua`, so `pcall` catches it, and `tostring(err)` gives its message: the error's own text, such as `markdown.section: path "/Guide/" has an empty step`, as [author prose](#reading-kind-and-message) after the line of the failing call. The messages this section quotes are that own text.

### The blocks `markdown.parse` returns

A parsed document is a list of blocks. A block is a plain table with a `kind`, the `line` it starts on, the `source` it was read from, and the fields of its kind:

| `kind` | Fields | What it is |
| --- | --- | --- |
| `frontmatter` | `data` | The YAML mapping that opens the text, as tables shaped like decoded JSON. It is only ever the first block. |
| `section` | `level`, `title`, `blocks` | A heading and everything under it. `level` is 1 to 6, and `blocks` is a list of blocks, subsections included. |
| `prose` | `text` | A run of Markdown that is no other kind: paragraphs, block quotes, indented code, HTML, and link definitions. |
| `fence` | `lang`, `info`, `text` | A fenced code block. `info` is the whole info string, `lang` is its first word, and `text` is the body. |
| `list` | `ordered`, `items` | A bulleted or numbered list. `items` is a list of strings with the markers removed. |
| `rule` | none | A thematic break such as `---`. |

Take this text, which the rest of this section calls the notes document:

````markdown
---
title: Notes
tags: [a, b]
---

Opening paragraph.

# Guide

Intro text
on two lines.

- alpha
- beta

## Templates

Use these.

```lua
return 1
```

```markdown
# Hello
```

Then the fallback:

```json strict
{ "ok": true }
```

### Extras

---

```lua
return 2
```

## Usage

1. first
2. second
````

`local md = markdown.parse(text)` gives three top-level blocks: the frontmatter, the opening paragraph, and the section `Guide`, which holds the rest. Ordinary indexing reads all of it:

````lua
#md                              --> 3
md[1].kind                       --> 'frontmatter'
md[1].data.title                 --> 'Notes'
md[1].data.tags[2]               --> 'b'
md[2].text                       --> 'Opening paragraph.'
md[2].source                     --> 'Opening paragraph.\n\n'
md[2].line                       --> 6
md[4]                            --> nil

local guide = md[3]
guide.kind                       --> 'section'
guide.level                      --> 1
guide.title                      --> 'Guide'
guide.source                     --> '# Guide\n\n'
#guide.blocks                    --> 4
guide.blocks[1].text             --> 'Intro text\non two lines.'
guide.blocks[2].items[1]         --> 'alpha'
guide.blocks[2].ordered          --> false
guide.blocks[2].source           --> '- alpha\n- beta\n\n'

local templates = guide.blocks[3]
templates.title                  --> 'Templates'
templates.line                   --> 16
templates.blocks[2].lang         --> 'lua'
templates.blocks[2].source       --> '```lua\nreturn 1\n```\n\n'
templates.blocks[5].lang         --> 'json'
templates.blocks[5].info         --> 'json strict'
templates.blocks[5].text         --> '{ "ok": true }'
templates.blocks[6].title        --> 'Extras'
templates.blocks[6].blocks[1]    --> { kind = 'rule', line = 36, source = '---\n\n' }
templates.blocks[2].title        --> nil
````

The result is plain data, with a few fixed shapes:

- Every value is a string, a number, a boolean, or a table, and every table holds data alone. A table is either a list or a record with named fields, never both, which is why a section keeps its members in `blocks`. So a parse result passes [`json.encode`](#encoding-values-with-jsonencode), can be stored in [`var`](#keeping-state-in-var), and can be a [`tasks.fanout` collection](14-fanout.md#collections-and-member-order).
- The document, every `blocks`, and every `items` are marked lists (see [Keeping empty lists with `json.array`](#keeping-empty-lists-with-jsonarray)), so an empty one encodes as `[]`.
- A field that a kind does not have reads as `nil`, as `templates.blocks[2].title` does for a fence.
- `title`, a fence's `text`, and an item can be the empty string. A prose `text` always holds at least one character, and so does the `source` of a parsed block.
- A block does not point at its parent or its neighbors.

`line` is 1-based and counts lines of the text given to `markdown.parse`, frontmatter lines included. It is the heading line for a section, the opening fence line for a fence, the first marker line for a list, and the first non-blank line for prose. It is stored when the text is parsed, and it stays the same through every lookup.

### The `source` of a block

Every block holds `source`, the lines it was read from, exactly as written and each with its line ending. These rules decide which lines a block holds:

- A block's `source` runs from its first line to the first line of the next block, so the blank lines after a block are in its `source`. A blank line holds only spaces and tabs.
- A section's `source` is its heading and the blank lines after it. The blocks under the heading hold their own lines, so each line is in one source.
- Blank lines at the very top of the text are in the first block's `source`, above its own lines. The block's `line` is still the line of its own first line.

Take this text, with a blank line between each two blocks:

````markdown
# Fruit

Some words.

- apples
- pears

## Citrus

### Lemon
````

It parses to one section, and these are the sources of its blocks:

````lua
local fruit = markdown.parse(text)[1]
fruit.source                         --> '# Fruit\n\n'
fruit.blocks[1].source               --> 'Some words.\n\n'
fruit.blocks[2].source               --> '- apples\n- pears\n\n'
fruit.blocks[3].source               --> '## Citrus\n\n'
fruit.blocks[3].blocks[1].source     --> '### Lemon\n'
````

A blank line between two blocks is kept with the block above it, so `Lemon` is the first block of `Citrus`. Every character of the text is in the `source` of exactly one block, so the sources, joined in the order the blocks appear, are the text. That is what [`markdown.render`](#writing-blocks-with-markdownrender) writes. Text that holds only blank lines is the one exception: it parses to an empty list, and its blank lines are left out of the result.

`source` holds what the other fields leave out. A list's `items` leave out the markers and the numbers, a section's `title` leaves out the markup, and a frontmatter's `data` leaves out the comments, and all of those are in `source`:

````lua
markdown.parse('3) x\n4) y\n')[1].items                    --> { 'x', 'y' }
markdown.parse('3) x\n4) y\n')[1].source                   --> '3) x\n4) y\n'
markdown.parse('See [API](x) *now*\n===\n')[1].title       --> 'See API now'
markdown.parse('See [API](x) *now*\n===\n')[1].source      --> 'See [API](x) *now*\n===\n'
````

### How text becomes blocks

`markdown.parse` reads the text as CommonMark with every extension off, the same reading the prompt parser gives a prompt, so a heading or a fence means the same thing in a prompt and in parsed text. Tables, task lists, and footnotes are not parsed as such, and they stay in prose or in an item's string. These rules decide the blocks:

- Only top-level constructs become blocks. A heading, fence, list, or thematic break inside a block quote stays in the surrounding prose, and one inside a list item stays in that item's string.
- Each line is kept. Every non-blank line of the text belongs to exactly one block, counting a heading line as part of its section, and every line, blank or not, is in the `source` of exactly one block.
- Frontmatter is recognized only at the very start: the first line is `---`, a later line is `---`, and the lines between parse as a YAML mapping. In `data`, a mapping is a record, a sequence is a marked list, and a YAML null is [`json.null`](#null-values-and-jsonnull). When any of that fails, those lines are ordinary Markdown, so the first line is a `rule`.
- A heading opens a section, which ends at the next heading of the same or a smaller level. A deeper heading opens a subsection inside it, even when it skips levels, so an H4 directly under an H2 is a member of the H2's `blocks` with `level` 4. `title` is the heading's text with inline markup removed, so ``## The *big* `foo` API`` has the title `The big foo API`.
- Prose is a maximal run. Two paragraphs separated by a blank line are one block, and a second prose block exists only when a fence, list, or rule sits between it and the first. `text` is the source as written, with leading blank lines and trailing whitespace removed.
- Backtick fences and tilde fences both count. `info` is the info string, `lang` is its first word, or `''` when the info string is empty, and `text` is the lines between the two fence lines without the final line ending. A `#` line inside a fence is not a heading. An unclosed fence runs to the end of the text.
- Each list is one block, and a change of marker, such as `-` then `*`, starts a new list. `ordered` is `true` for a numbered list. An item is its source with the marker removed. An item that spans lines or holds nested content keeps it as raw Markdown, such as `'a\n- a1\n- a2'`, so `markdown.parse(item)` recovers its structure. A bare marker is the item `''`. A list ends with its last item, so a link definition right after a list is prose.

Parsing succeeds on any content. Any UTF-8 text within the size limit is a document: skipped levels, repeated titles, and an unclosed fence are all accepted, and empty text gives an empty list, as does text that holds only blank lines. This is looser than the prompt parser on purpose, because `markdown` reads text you do not control, such as a model reply. A byte-order mark at the very start is dropped, and `\r\n` and a lone `\r` are read as `\n`, so every string in the result uses `\n` only, each `source` included.

`markdown.parse` takes a string of at most 8,388,608 bytes, the limit [`json.decode`](#decoding-json-text-with-jsondecode) has, and raises three errors:

| When | Message |
| --- | --- |
| The argument is not a string | `markdown.parse expects a string, got {type}` |
| The string is longer than the cap | `markdown.parse: input exceeds the 8388608-byte limit` |
| The text is not valid UTF-8 | `markdown.parse: text is not valid UTF-8` |

### Finding a section with a path

`markdown.section(blocks, path)` returns the one section that `path` names, starting from the list `blocks`. The path is a [section path](02-file-structure.md#section-paths) from the top of that list: it always starts with `/`, each step is a section's title, and a title matches exactly and case-sensitively. With the notes document:

````lua
markdown.section(md, '/Guide').title                    --> 'Guide'
markdown.section(md, '/Guide/Templates').line           --> 16
markdown.section(md, '/Guide/Templates/Extras').level   --> 3
markdown.section(md, '//Extras').line                   --> 34
markdown.section(md, '/Guide//Extras').line             --> 34
markdown.section(guide.blocks, '/Usage').line           --> 42
markdown.section(md, '/Templates')                      --> raises
markdown.section(md, '/guide')                          --> raises
markdown.section_or(md, '/Templates')                   --> nil
````

A path over parsed text has two forms a path in a prompt does not have:

- `/title` steps to the sections with that title among the direct members of the current list.
- `//title` steps to the sections with that title at any depth in the current list, direct members included.
- `[n]` after a title keeps the `n`th section that step matched, where `[1]` is the first and `[-1]` is the last.

The first step starts from `blocks`, and each later step starts from the `blocks` of every section the step before it kept. After the last step, one section is the result, none is a miss, and more than one raises. Only the last step has to end with one section, so a step in the middle may keep several.

Parsed text may repeat a title among siblings, which a prompt may not, and an index picks one. With `log` parsed from `# Log`, then `## Entry` over the paragraph `first`, then a second `## Entry` over `second`:

````lua
markdown.section(log, '/Log/Entry')                    --> raises
markdown.section(log, '/Log/Entry[1]').line            --> 3
markdown.section(log, '/Log/Entry[-1]').line           --> 7
markdown.section(log, '/Log/Entry[3]')                 --> raises
markdown.section_or(log, '/Log/Entry[3]')              --> nil
markdown.section(log, '/Log/Entry[2]', 'prose').text   --> 'second'
````

The first line raises because the path matches two sections, and the message says so: `markdown.section: path "/Log/Entry" is ambiguous; it matches 2 sections, at lines 3 and 7; add an index such as "/Log/Entry[1]"`. An index counts everything its step matched, in the order the headings appear in the text, so `//Entry[2]` is the second `Entry` anywhere in the document.

A few more rules round out the path:

- The path must start with `/`. The top of the list is the one place a path over parsed text can start from, so `'Guide/Usage'` raises, and so does a path that uses `..`.
- Only a bracketed integer at the very end of a step is an index. `/[Draft] Notes` and `/Notes [old]` are plain titles, and a title that itself ends in a bracketed integer escapes the bracket, as in `/Appendix \[2]`. A slash in a title is written `\/` and a backslash `\\`.
- A section whose title is empty, a bare `##`, cannot be named by a path. Indexing and `markdown.select` still reach it.

The `blocks` argument is a list of blocks: the value `markdown.parse` returned, any section's `blocks`, or a list you built. A lookup sees only the list it is given, so passing `guide.blocks` searches inside `Guide` alone, which is why `'/Usage'` finds a section there that `'/Usage'` over `md` misses. A section is not a list. Passing the section table itself raises, and the message points at its `blocks` field.

A lookup returns the table that is already inside the list it was given, not a copy, so `markdown.section(md, '/Guide') == md[3]` is `true`. An edit made through a lookup's result is therefore an edit of `md`. A later `markdown.render(md)` writes each block's `source`, so it writes an edit of `source` and skips an edit of any other field. The selector forms of `prompt.section` are the opposite: they build a new table on every call.

### Picking a block with a selector

`markdown.section(blocks, path, selector, n?)` finds the section as above, then picks one block among that section's own `blocks`. Blocks inside its subsections are not counted. The selector and the index are the ones [`prompt.section`](03-blocks-and-prose.md#reading-one-block-with-a-selector) takes:

| Selector | Picks |
| --- | --- |
| `'frontmatter'`, `'section'`, `'prose'`, `'fence'`, `'list'`, `'rule'` | a block of that kind |
| `'fence markdown'` | a fence whose `lang` is `markdown`, compared exactly and case-sensitively, and likewise for any other language |
| `'fence ""'` | a fence with no language, the one whose `lang` is `''` |
| `'*'` | a block of any kind |

Without an index, the selector must match exactly one block. With an index, it may match any number, and the index picks among them: `1` is the first match and `-1` is the last.

````lua
markdown.section(md, '/Guide/Templates', 'fence markdown').text   --> '# Hello'
markdown.section(md, '/Guide/Templates', 'fence lua').text        --> 'return 1'
markdown.section(md, '/Guide/Templates', 'section').title         --> 'Extras'
markdown.section(md, '/Guide', 'list').items[2]                   --> 'beta'
markdown.section(md, '/Guide/Templates', 'fence', 3).text         --> '{ "ok": true }'
markdown.section(md, '/Guide/Templates', 'fence', -1).lang        --> 'json'
markdown.section(md, '/Guide/Templates', 'prose', 2).text         --> 'Then the fallback:'
markdown.section(md, '/Guide/Templates', '*', 2).lang             --> 'lua'
markdown.section(md, '/Guide/Usage', '*').ordered                 --> true
markdown.section(md, '/Guide/Templates', 'list')                  --> raises
markdown.section(md, '/Guide/Templates', 'fence')                 --> raises
markdown.section(md, '/Guide/Templates', 'fence', 4)              --> raises
````

`'fence lua'` matches one fence in `Templates`, because the other `lua` fence is in its subsection `Extras`. `'*'` with an index picks a block by its position, whatever its kind, and without one it needs a section with one block, as `Usage` is. The last three lines raise for three different reasons: `Templates` lacks a list, it holds three fences and the call leaves out the index, and the index of the last call runs past its three fences. To take every direct member of one kind, loop over the section's `blocks` with `ipairs` and test `kind`.

`markdown.section` and `markdown.section_or` take the same arguments and find the same thing. They differ only when it is not there:

| Outcome | `markdown.section` | `markdown.section_or` |
| --- | --- | --- |
| The path misses | raises | `nil` |
| The selector misses, or matches fewer blocks than the index asks for | raises | `nil` |
| The path matches several sections | raises | raises |
| The selector matches several blocks and the index is left out | raises | raises |
| The path, selector, or index is malformed, or `blocks` is not a list of blocks | raises | raises |

The first two rows are things that are absent. The rest are mistakes in the call, or a choice that only an index can make, and both functions raise for them. An index that runs past the last match is an absence, so a loop can count upward with `markdown.section_or` until it returns `nil`:

````lua
markdown.section_or(md, '/Guide/Templates', 'list')         --> nil
markdown.section_or(md, '/Guide/Templates', 'fence ""')     --> nil
markdown.section_or(md, '/Guide/Templates', 'fence lua', 2) --> nil
markdown.section_or(md, '/Guide/Nope', 'fence')             --> nil
markdown.section_or(md, '/Guide/Templates', 'fence')        --> raises
````

### Collecting blocks with `markdown.select`

`markdown.select(blocks, selector)` collects every block that matches the selector at any depth, in the order the blocks appear in the text, with a section before its members. It returns a new list that holds the matching block tables themselves, so a selector that misses gives an empty list:

````lua
#markdown.select(md, 'section')                      --> 4
#markdown.select(md, 'fence')                        --> 4
#markdown.select(md, 'fence lua')                    --> 2
markdown.select(md, 'fence lua')[2].text             --> 'return 2'
markdown.select(md, 'list')[2].ordered               --> true
#markdown.select(md, 'fence python')                 --> 0
markdown.select(templates.blocks, 'prose')[1].text   --> 'Use these.'
````

The four sections are `Guide`, `Templates`, `Extras`, and `Usage`. Like the lookups, `markdown.select` searches only the list it is given, so the last line looks inside `Templates` alone. It is the function to reach for when the text's shape is not known ahead of time, as with a model reply: `markdown.select(markdown.parse(reply), 'fence json')[1]` is the first `json` fence wherever the reply put it, or `nil`.

With `'*'` it collects every block, which gives you the document as one flat list:

````lua
local flat = markdown.select(md, '*')
#flat                                     --> 16
flat[3] == md[3]                          --> true
flat[4] == md[3].blocks[1]                --> true
#markdown.select(templates.blocks, '*')   --> 8

local parts = {}
for i, block in ipairs(flat) do
  parts[i] = block.source
end
table.concat(parts) == text               --> true
````

The list is new, and the tables in it are the document's own blocks, `Guide` at position 3 and the first block under it at position 4. Joining their sources gives the text that was parsed. Do not pass a flat list to `markdown.render`: it goes into each section itself, so it would write the blocks under each section twice.

### Writing blocks with `markdown.render`

`markdown.render(blocks)` writes the [`source`](#the-source-of-a-block) of every block in the list, at any depth and in the order the blocks appear, a section's before those of the blocks under it. A block's other fields stay out of the written text, so a parsed document is written back as the text it was parsed from, and an empty list gives the empty string:

````lua
markdown.render(md) == text               --> true
markdown.render(guide.blocks[4].blocks)   --> '1. first\n2. second\n'
markdown.render({ guide.blocks[4] })      --> '## Usage\n\n1. first\n2. second\n'
````

A section's `blocks` render without its heading, and wrapping the section in a list renders the heading too. A block is written with the blank lines that followed it in the text, so `markdown.render(section.blocks)` is the section's body from its first line of content to the line above the next heading. Each block is written as it was read: a list keeps its numbers and its markers, a fence its own characters, a heading the way it was written, and a frontmatter its comments. A byte-order mark is not written back, and every line ending is written as `\n`, because that is how the text was read.

`markdown.render` reads three fields of a block. `source` is what it writes. `kind` and `blocks` tell it what comes next: only a section's `blocks` are written, and a section that lacks a `blocks` field counts as empty. So an edit of `source` changes what is written, and an edit of `title`, `text`, `items`, or any other field does not. To change a document, replace a block, or write the block's `source`:

````lua
local usage = markdown.section(md, '/Guide/Usage')
usage.source = '## How to use it\n\n'
table.insert(usage.blocks, { kind = 'prose', source = '\nDone.\n' })
return markdown.render({ usage })
````

````text
## How to use it

1. first
2. second

Done.
````

The heading's new `source` ends with the blank line the old one had, because blank lines after a heading are part of its `source`. The block added by hand has a `source` and a `kind`, and those two fields are enough. Its `source` starts with a blank line of its own, because the list before it is the last block of the text, and the text ends with the list's last line.

To replace everything under a heading, give the section a new list of blocks. With `log` parsed from `# Log`, then `## Entry` over the paragraph `first`, then a second `## Entry` over `second`:

````lua
local entry = markdown.section(log, '/Log/Entry[1]')
entry.blocks = { { kind = 'prose', source = 'changed\n\n' } }
markdown.render(log)   --> '# Log\n\n## Entry\n\nchanged\n\n## Entry\n\nsecond\n'
````

Every other line of the document is written as it was read.

#### The line endings `markdown.render` adds

Blocks from different texts, or blocks whose `source` you wrote, may not fit together as written. `markdown.render` adds a line ending of its own in two cases, and neither occurs between a block and the block that followed it in the same parse:

- Every block starts on a new line. When the text so far does not end with a line ending, one is written before the next block. The last block of a text lacks a line ending when the text itself ends without one.
- A blank line sits between two prose blocks. When a prose block follows a prose block, and both the last line so far and the block's first line are non-blank, a blank line is written, so the two stay separate paragraphs.

````lua
local one, two = markdown.parse('# A\nlast'), markdown.parse('# B\n\n- x')
markdown.render({ one[1], two[1] })   --> '# A\nlast\n# B\n\n- x'

markdown.render({
  { kind = 'prose', source = 'one\n' },
  { kind = 'prose', source = 'two\n' },
})                                    --> 'one\n\ntwo\n'
````

Without the first, the result would hold the line `last# B`. Without the second, `one` and `two` would be one paragraph.

`markdown.render` joins any other two blocks as they are, because whether they need a blank line depends on their text. A paragraph directly above `- a` is fine as written. Directly above `2. b` it takes the list into itself, and directly above `---` it becomes a heading. So when you write a `source` yourself, keep the blank line a block needs: before a `---` rule, before a list that does not start at 1, and after a list. Two lists with the same marker read as one list even with a blank line between them.

`markdown.render` raises for a value that is not a list of blocks, and for a block with no `source` to write. The message names the block by its place, such as `blocks[2].source`:

| When | Message |
| --- | --- |
| The value is a section, not a list of blocks | `markdown.render: blocks must be a list of blocks, got a section; pass its blocks field` |
| A block lacks a `source` | `markdown.render: blocks[1].source must be a string, got nil` |
| A block under a section lacks one | `markdown.render: blocks[1].blocks[2].source must be a string, got nil` |
| A `source` is not a string | `markdown.render: blocks[1].source must be a string, got integer` |
| A section's `blocks` is not a list | `markdown.render: blocks[1].blocks must be a list of blocks, got integer` |

The other fields are never checked, and `kind` and `line` are optional.

### Keeping parsed blocks

Every section entry gets a fresh section VM, so a parse result does not survive a fall-through or a `jump`. Two things can pass it on:

- `var.doc = md` stores a deep copy as JSON, and reading it back gives [guarded views](#reading-arrays-and-other-tables-from-var). The `markdown` functions read through them, so `markdown.section(var.doc, '/Guide/Usage', 'list').items[1]` is `'first'` in a later section. The copy holds every `source`, so `markdown.render(var.doc)` is still the text that was parsed.
- A fanout collection gives each arm one block as its `item`, as in `tasks.fanout('Worker', markdown.select(md, 'section'))`.

Passing on the text, or the name of a store file, and parsing it where it is needed is usually cheaper than passing on the tables.

`line` always counts lines of the text that was parsed, never lines of a part of it, so `markdown.section(guide.blocks, '/Templates', 'fence', 3).line` is still `30`. A new parse counts from 1: `markdown.parse(templates.blocks[3].text)[1].line` is `1`, and `markdown.parse(prompt.section(path))` counts from the first line of that section's text. The selector forms of `prompt.section` count lines of the prompt file instead.

The lookups read only the fields they need, `kind`, `title`, `blocks`, `lang`, and `line`, and a block whose `kind` they do not know matches the selector `'*'` alone. A list nested more than 127 levels deep, or a list that contains itself, raises. So does a list or a block that is a [weak table](#weak-tables), as in `markdown.render: blocks is a weak table, one whose metatable sets __mode: the entries the collector leaves in it are not the same in two runs`.

### Errors from the lookups

The lookups check the call before they search. Each row raises from `markdown.section_or` too, with `markdown.section_or` at the start of the message, except the three misses at the end, for which it returns `nil`:

| Call | Message |
| --- | --- |
| `markdown.section(guide, '/Usage')` | `markdown.section: blocks must be a list of blocks, got a section; pass its blocks field` |
| `markdown.section('text', '/Usage')` | `markdown.section: blocks must be a list of blocks, got string` |
| `markdown.section(md, 42)` | `markdown.section: path must be a string, got integer` |
| `markdown.section(md, 'Guide/Setup')` | `markdown.section: a path starts with "/", got "Guide/Setup"` |
| `markdown.section(md, '/Guide/../Usage')` | `markdown.section: path "/Guide/../Usage" uses ".."; parsed text has no parent to step to` |
| `markdown.section(md, '/Guide/')` | `markdown.section: path "/Guide/" has an empty step` |
| `markdown.section(md, [[/a\b]])` | `markdown.section: path "/a\b" has an unknown escape "\b"; the escapes are "\/", "\\", and "\["` |
| `markdown.section(md, '/Guide[0]')` | `markdown.section: path "/Guide[0]" has index 0; an index is a nonzero integer` |
| `markdown.section(log, '/Log/Entry')` | `markdown.section: path "/Log/Entry" is ambiguous; it matches 2 sections, at lines 3 and 7; add an index such as "/Log/Entry[1]"` |
| `markdown.section(md, '/Guide', 'fences')` | `markdown.section: unknown kind "fences"; kinds are frontmatter, section, prose, fence, list, rule, and * selects every block` |
| `markdown.section(md, '/Guide', 'list lua')` | `markdown.section: only a fence selector takes a language, got "list lua"` |
| `markdown.section(md, '/Guide', nil, 1)` | `markdown.section: an index needs a selector before it` |
| `markdown.section(md, '/Guide', 'fence', 0)` | `markdown.section: index must be a nonzero integer, got 0` |
| `markdown.section(md, '/Guide/Templates', 'fence')` | `markdown.section: selector "fence" matches 3 blocks, at lines 20, 24, and 30; add an index, such as 1 for the first` |
| `markdown.section(md, '/Guide/Nope')` | `markdown.section: path "/Guide/Nope" matches no section; step 2, "Nope", matches nothing` |
| `markdown.section(md, '/Guide/Templates', 'list')` | `markdown.section: no block in "/Guide/Templates" matches selector "list"` |
| `markdown.section(md, '/Guide/Templates', 'fence', 4)` | `markdown.section: selector "fence" matches 3 blocks in "/Guide/Templates"; there is no block 4` |

`markdown.select` raises the `blocks` and selector messages under its own name, as in `markdown.select: selector must be a string, got integer`.

### Common uses

````lua
-- the JSON a model replied with, wherever the reply put the fence
local reply = models.infer(prose)
local found = markdown.select(markdown.parse(reply), 'fence json')[1]
local data = found and json.decode(found.text)

-- a section a model's reply may have left out, or written twice
local findings = markdown.section_or(markdown.parse(reply), '//Findings[1]')
local summary = findings and markdown.render(findings.blocks) or ''

-- one section of a stored document, as text for a model; it raises when the document has no Abstract
local paper = markdown.parse(store.read('paper.md'))
local abstract = markdown.render(markdown.section(paper, '//Abstract').blocks)

-- that document with the body of one section replaced, and every other line written back as it was
markdown.section(paper, '//Abstract').blocks = { { kind = 'prose', source = 'A new abstract.\n\n' } }
store.write('paper.md', markdown.render(paper))

-- each top-level section of that document
for _, block in ipairs(paper) do
  if block.kind == 'section' then
    log(block.title)
  end
end

-- several blocks of one section of the prompt: parse it once and index the result
local blocks = markdown.parse(prompt.section('Template'))
````

Text from a model, a tool, or the store is still untrusted text after it is parsed. The strings in the blocks are plain Lua strings, so wrap one with [`untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted) before it reaches a model.

## Run facts in `sys`

After this section you can read the facts the run keeps about itself from `sys` and know when each value is fixed.

````markdown
---
name: where-am-i
description: Reads run facts from sys.
runvil: 0
---

## Only

```lua
return sys.section_name .. ' ' .. sys.id .. ' of ' .. sys.section_count
```
````

````text
Only 0.0 of 1
````

`sys` is an Engine global that the Engine installs in every section entry: a table of runtime values the run supplies, which a Lua block reads run metadata from. `sys` is a sealed, read-only table, and you read a field by name with dot or bracket syntax, as in `sys.id` or `sys['when']`.

### The `sys` fields

Every section entry, each fanout arm's included, gets five `sys` fields: `sys.when`, `sys.id`, `sys.taskid`, `sys.section_name`, and `sys.section_count`. Two more fields, `sys.model` and `sys.index`, appear only in some section entries:

| Field | Type | Example value | Present |
|---|---|---|---|
| `sys.when` | string | `2000-02-29T00:00:00.123Z` | always |
| `sys.id` | string | `0.1` | always |
| `sys.taskid` | string | `0` | always |
| `sys.section_name` | string | `Alpha` | always |
| `sys.section_count` | number | `2` | always |
| `sys.model` | string | `claude-sonnet-4-6` | after the section entry's first tool call |
| `sys.index` | number | `7` | on a task's first section entry, when its spawn supplied one |

A present `sys` field reads as a plain Lua value converted from its JSON value. `when`, `id`, `taskid`, `section_name`, and `model` arrive as Lua strings, and `section_count` and `index` arrive as Lua numbers, so `sys.section_name == 'Alpha'` compares as a string and `sys.section_count == 2` as a number. The rest of this section takes the fields one at a time, and [Section entry ids](#section-entry-ids) covers `sys.id` in full.

### Section name and section count

````markdown
---
name: two-sections
description: Reads the section name and the section count.
runvil: 0
---

## Alpha

```lua
assert(sys.section_name == 'Alpha')
assert(sys.section_count == 2)
```

## Beta

```lua
assert(sys.section_name == 'Beta')
assert(sys.section_count == 2)
return sys.section_name .. ' of ' .. sys.section_count
```
````

Every assertion holds, and the run result is:

````text
Beta of 2
````

`sys.section_name` is a string holding the section name of the section whose Lua is running, so it reads `'Alpha'` in section Alpha and `'Beta'` in section Beta. `sys.section_count` is a number holding the count of top-level sections in the prompt that the section belongs to. The count reads the same in every section of that prompt, and a prompt run with [`execute`](08-jump-and-call.md#how-an-executed-prompt-runs) reads its own count, not its caller's.

A [preamble](02-file-structure.md#the-preamble) that holds Lua is a top-level section whose name is empty, so `sys.section_name` reads the empty string there, and `sys.section_count` counts the preamble with the other top-level sections:

````markdown
---
name: preamble-name
description: Reads the section name and the section count in the preamble.
runvil: 0
---

```lua
return '[' .. sys.section_name .. '] ' .. sys.section_count
```
````

````text
[] 1
````

The brackets enclose the empty section name, and the preamble is this prompt's only top-level section, so the count is `1`.

Fanout arms, and chains that an arm starts with `call`, see the same `sys.section_count` as the rest of their prompt, its top-level section count, and a child worker section does not add to it. With `Parent` as the only top-level section, a `Worker` arm under it reads `sys.section_count == 1`.

### The task id in `sys.taskid`

````markdown
---
name: arm-task
description: Reads the task id on the main walk and inside a fanout arm.
runvil: 0
---

## Parent

```lua
assert(sys.taskid == '0')
local results = tasks.fanout('Worker', {'a'})
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

`sys.taskid` is a string holding the id of the nearest enclosing task, the task the code runs inside (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)). It reads `'0'` on the main walk, which is why the assertion in `Parent` holds. Every section a chain enters reads the same `sys.taskid`, the chain's task id.

Inside a spawned task or a fanout arm, `sys.taskid` reads that task's own id, the same value as the `task` field of the handle `tasks.spawn` returns, so the first task spawned from the main walk reads `0.0`, as the arm above does. Inside a called chain, and inside a prompt run with [`execute`](08-jump-and-call.md#how-an-executed-prompt-runs), `sys.taskid` reads the caller's task, because a `call` and an `execute` each start a chain inside the caller's task, while inside a fanout arm or a `tasks.spawn` chain it reads that task's own id.

### The spawn index in `sys.index`

A task's first section entry also has `sys.index` when its spawn supplied one. A fanout arm reads its member's 1-based position (see [What an arm sees](14-fanout.md#what-an-arm-sees)), and a task started by `tasks.spawn` with `index = 7` among its options reads `7`. In every other section entry, ordinary walked sections and a task's later entries included, reading `sys.index` fails with `unknown sys field 'index'`.

### Reading an unknown `sys` field

Reading a field that `sys` does not have raises an error instead of returning a silent `nil`, and the message names the field:

````text
unknown sys field '{field}'
````

So reading `sys.index` in an ordinary walked section raises `unknown sys field 'index'`. The error is a runtime error with error kind `lua`. Left uncaught, it ends the run with code `lua_runtime`. A prose placeholder for an absent field fails with its own message, which repeats the placeholder after `missing`, so `{{ sys.index }}` in an ordinary walked section's prose fails with `missing {{ sys.index }}`.

To read a `sys` field that may be absent, such as `sys.model` before any tool call, wrap the read in `pcall`. As the Lua block of `Only` in the `where-am-i` prompt, this returns `no tool call yet`:

````lua
local ok, model = pcall(function() return sys.model end)
return ok and model or 'no tool call yet'
````

The run goes on whether or not the field is there. A caught unknown field error is an error value with error kind `lua`, and `tostring` on it reads `{file}:{line}: unknown sys field '{field}'. The Lua code failed at runtime. Fix the code at this location.`, with the line of the read (see [Reading `kind` and `message`](#reading-kind-and-message)), so here, where the read fails, `model` holds that error value and `tostring(model)` reads `where-am-i.md:10: unknown sys field 'model'. The Lua code failed at runtime. Fix the code at this location.`

### `sys` is sealed and read-only

The `sys` table itself is empty, and a function answers each read instead of a table of data, so its values are reachable only by reading a named field: iterating `sys` with `pairs` or `next` finds it empty, and `#sys` is `0`. `sys` takes string keys only, so you read a field by its name, as in `sys.when` or `sys['when']`, and a read with any other key type raises `sys fields must be accessed by string key`.

`sys` is read-only. Assigning any `sys` field, existing or new, raises a Lua error that names a string key by its text, an integer key by its digits, and any other key by its type name:

````text
sys is read-only; cannot set '{field}'
````

An assignment to `sys.when` gives `sys is read-only; cannot set 'when'`, and one to a new field `sys.extra` gives `sys is read-only; cannot set 'extra'`. The `sys` read and write errors, like the `var` JSON check errors, are ordinary Lua runtime errors raised at the failing line, so `pcall` can catch them, as in `local ok, err = pcall(function() sys.extra = 1 end)`.

`getmetatable(sys)` returns the string `sys is sealed` instead of the real metatable, and your code cannot replace the seal, so `setmetatable(sys, {})` fails with `cannot change a protected metatable`.

### The start time in `sys.when`

````markdown
---
name: started
description: Reads the run's start instant in the preamble and in a section.
runvil: 0
---

```lua
var.first_when = sys.when
```

## Main

```lua
assert(sys.when == var.first_when)
return sys.when
```
````

For a run that started at 00:00:00.123 UTC on 29 February 2000, the run result is:

````text
2000-02-29T00:00:00.123Z
````

`sys.when` is a string holding the instant the run started, rendered as RFC 3339 in UTC and ending in `Z`. It is one fixed instant stamped at run start, so every section of the run reads the same value. The preamble keeps its reading in `var`, `Main` is the next entry of the same chain and finds it there, and the two readings are equal, which is why the assertion holds. `sys.when` is the only time field in `sys`, so a prompt reads the run's time through `sys.when`, and `sys` leaves the current time out.

`sys.when` is the start instant the Harness supplies when it creates the run, not a live clock, so it stays fixed while the run executes and reads as the identical string on every read. It does not depend on the run [seed](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host): runs with different seeds but the same start instant read the same `sys.when`, even though the Harness supplies the seed and the start instant together. A replay that passes back the run's recorded start instant reproduces the same `sys.when`.

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

When the run starts on a whole second, `sys.when` omits the fraction. Otherwise it includes a millisecond fraction with trailing zeros dropped, giving one, two, or three digits, and never more than three. Leading zeros inside the fraction stay, so 5 ms past the second renders as `.005` and 50 ms as `.05`.

The rest of the format is just as fixed:

- The time of day is UTC on a 24-hour clock, with hours `00` through `23`.
- The text always ends with `Z` and never has a numeric offset such as `+00:00` or a local time zone.
- Dates follow the proleptic Gregorian calendar in UTC with correct leap years, including the century rules, so leap days such as `2024-02-29` appear correctly, 1900 and 2100 are not leap years, and 2000 is.
- A start instant before 1970 reads as an ordinary earlier calendar date, as the last row of the table shows.
- The year is four zero-padded digits for start instants from year 0000 through 9999, the last being `9999-12-31T23:59:59.999Z`.
- The text matches a standard RFC 3339 rendering byte for byte, so you can hand it to any standard RFC 3339 parser.

`sys.when` is a string, not a millisecond count, so when you need date parts or date arithmetic you parse the text. Because the layout is fixed, `string.sub(sys.when, 1, 10)` gives the date, `2000-02-29` in the `started` run.

Comparing two `sys.when` strings as text can put a later instant first. The fraction varies in length and is omitted on whole seconds, while `.` and digits sort before `Z`, so `12:34:56Z` sorts after the later `12:34:56.5Z`, and `.1Z` (100 ms) sorts after the later `.15Z` (150 ms).

### The bound model in `sys.model`

````markdown
---
name: model-name
description: Reads the current model's name before and after the first tool call.
runvil: 0
uses:
  - web
---

```lua
models.use('strong')
```

## Main

```lua
local ok, model = pcall(function() return sys.model end)
var.before = ok and model or 'no tool call yet'
tools.call('web/fetch', { url = 'https://example.com/' })
```

Model name is {{ sys.model }}.

```lua
return var.before .. ' / ' .. prose
```
````

The preamble selects the `strong` role, and `Main` is the next entry of the same chain, so it keeps that model (see [The current model](10-models.md#the-current-model)) and runs on `strong`. On a Host whose role map fills `strong` with the model `gateway/claude-sonnet-4-6` (a one-model map fills every role with that model), once the fetch answers, the run result is:

````text
no tool call yet / Model name is claude-sonnet-4-6.
````

`sys.model` is a string holding the name part of the current model's model id, without the server namespace, so it reads `fast` for a model whose id is `gateway/fast`, and `claude-sonnet-4-6` for `strong` above, filled by `gateway/claude-sonnet-4-6`. It is the same name that role's model handle reads as `model_id` (see [Model handles](10-models.md#model-handles)), never the role name the prompt uses for that model. It becomes readable only after the section entry's first tool call, and only when the entry has a model. Here the first tool call is the [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls) to the `web` Pack's fetch tool (see [Fetching a page](13-web-fetch-and-search.md#fetching-a-page)), so the `pcall` before it catches the unknown field error and the placeholder after it renders the name.

At the section entry's first tool call, `sys.model` takes the chain's current model at that moment, the model the chain last selected with `models.use` or else the model it started on, and stays fixed for the rest of the section entry. A switch made in an earlier section of the same chain shows up: after `models.use('strong')` in one section, `sys.model` at the first tool call of the next section names the `strong` model. A section entered again with `jump`, a fanout arm, a task, and a section run with `call` each start without the field and fill it at their own first tool call. From that call on, later `{{ sys.model }}` prose placeholders and later Lua blocks alike read that same name, so a later `return sys.model` returns `claude-sonnet-4-6` too.

`sys.model` appears at the section entry's first tool call: a call to a catalog tool or an adhoc tool, or the model's call to a discovery tool, whether a Lua block makes it with `tools.call` or the model makes it during a conversation, such as a tool call inside [`models.loop`](11-conversations.md#starting-a-conversation). A discovery call that the Engine answers with refusal text still counts as that first call. A call refused before it is dispatched does not: a `tools.call` that fails with `unbound_tool` or `model_only_tool`, and a model call that fails with `out_of_scope_tool`, leave `sys.model` and `tools.stats` for a later call to create. A model round, one request to a model and its reply, never sets `sys.model` by itself, so a [`models.infer`](10-models.md#running-a-round-with-modelsinfer) or `models.loop` round with no tool call leaves it unreadable. A section entry gets the field only when it makes a tool call and has a model when its first tool call runs.

Reading `sys.model` before the section entry's first tool call fails with `unknown sys field 'model'`, even when an earlier section of the chain, such as the preamble, selected a model with `models.use('strong')`. That holds whether the read is in the section's opening Lua block or inside a function from a library the section loads with [`require`](03-blocks-and-prose.md#sharing-code-with-require), as in `require '/sys_reader'` followed by `return read_sys_model()`.

Inside a fanout arm, once the arm's first tool call has been dispatched, `sys.model` reads the name part of the arm's current model's id in the same way. In the two-argument form `tasks.fanout(worker, collection)`, an arm starts on the model its parent was on when the arm was dispatched, so an arm for the member `a` that returns `sys.model .. ':' .. item`, with the parent on `strong` as above, gives `claude-sonnet-4-6:a`. With `tasks.fanout(model, worker, collection)`, every arm starts on the given handle's model instead.

## Section entry ids

After this section you can predict the `sys.id` of any section entry, on the main walk, inside a `call`, and inside a task, and use it as a unique key.

````markdown
---
name: numbered
description: Reads section entry ids along the main walk.
runvil: 0
---

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

The main walk is one chain, the root chain `0`, and it counts its entries from zero: the first section it enters reads `sys.id` as `'0.0'`, the next reads `'0.1'`, and so on. The `numbered` prompt has a [preamble](02-file-structure.md#the-preamble), the first section the walk enters, so the preamble's Lua reads `'0.0'`, `First` reads `'0.1'`, and `Second` reads `'0.2'`. In a prompt with no preamble, the first heading's section is entry `0.0`, as `Main` is in the prompts below.

`sys.id` compares equal to a plain Lua string, so you test it with `==` against a string literal such as `'0.1'`, as the assertions above do. `tostring(sys.id)` gives the same text as `sys.id`, because the id is already a string, so a block can return or store either form, and `return tostring(sys.id)` in `Second` gives `0.2` too.

### The id format

A section entry id is its chain's id followed by a dot and the entry's index, so entries 0 and 7 of chain `0.3` read `0.3.0` and `0.3.7`, and the first section entered by chain `0.3` reads `0.3.0`. Section entries on the main walk read `0.N` because the main walk is chain `0`.

Every section entry gets a fresh `sys.id`, so a section entered more than once in the same chain, including through `jump`, reads a new id each time: the id counts entries, not sections. `sys.id` counts up by one for each section the walk chain enters, and a `jump` into a child level stays in the same chain, so the count continues rather than restarting. Across `A`, `X`, `Y`, and `B`, entered in that order from the start of the walk, the ids are `0.0`, `0.1`, `0.2`, and `0.3`.

### Ids inside a `call`

````markdown
---
name: called-ids
description: Reads section entry ids inside called chains.
runvil: 0
---

## Main

```lua
local a = call('Sub')
local b = call('Sub')
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
0.0,0.0.0,0.1.0,0.1
````

Each `call` starts a contained child chain nested under the caller's chain. The first `call` from `Main` is the root chain's first child, so it opens chain `0.0`, and its section entries read `0.0.N`, starting with `0.0.0` for the called section. A section run by `call` or as a fanout arm is the first entry of its new chain, so its `sys.id` ends in `.0`.

Each `call` starts a new child chain numbered in call order, so entering the same section through two calls gives two distinct `sys.id` values, `0.0.0` and `0.1.0`. After `call` returns, the caller's `sys.id` is unchanged, so `Main` still reads `0.0` after both calls, and the walk goes on to `Last` as `0.1`.

Call chains are numbered as children of the calling chain, not of the calling section's entry. The two calls from `Main`, itself entry `0.0`, start chains `0.0` and `0.1`, so the first chain's section reads `0.0.0` and the second's reads `0.1.0`, which does not start with the calling entry's id. Child chains are numbered by the calling chain's own child counter in the order they start, and calls, prompts run with [`execute`](#ids-inside-an-executed-prompt), tasks, fanout arms, and each [timed wait](15-tasks.md#the-timer-behind-a-time-limit)'s internal timer all share that counter no matter which section made the call, so each new call or arm takes the next child chain number.

A chain started by `call` numbers its own section entries. With this `Main` and its children in place of the ones in the `called-ids` prompt, the run result is `0.0,0.0.1,0.1`:

````markdown
## Main

```lua
var.ids = sys.id .. ',' .. call('Sub')
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

The called section reads `0.0.0`, the fall-through inside that chain takes the next entry, `0.0.1`, and after the chain ends the outer walk resumes its own sequence, so the next walked section reads `0.1` no matter how many entries the chain used.

A `call` from a preamble takes the root chain's first child slot, so the called section runs as `0.0.0`. A later `call` from the section after the preamble then runs as `0.1.0`, and that calling section itself reads `sys.id == '0.1'`.

### Ids inside an executed prompt

````markdown
---
name: report-ids
description: Returns the section entry ids of its two sections.
runvil: 0
---

## One

```lua
var.one = sys.id
```

## Two

```lua
return var.one .. ' ' .. sys.id
```
````

The next prompt lists `report-ids.md` and runs it twice with `execute`:

````markdown
---
name: executed-ids
description: Reads section entry ids inside an executed prompt.
runvil: 0
prompts:
  - report-ids.md
---

## Main

```lua
local a = execute('report-ids.md')
local b = execute('report-ids.md')
var.ids = table.concat({ sys.id, a, b }, ' | ')
```

## Last

```lua
return var.ids .. ' | ' .. sys.id
```
````

````text
0.0 | 0.0.0 0.0.1 | 0.1.0 0.1.1 | 0.1
````

An executed prompt is a prompt that another prompt, its caller, runs with [`execute`](08-jump-and-call.md#executing-another-prompt). Its sections take ids the way a called chain's sections do: each `execute` starts a child chain of the calling chain, in that chain's next child slot, and the child chain counts the executed prompt's section entries from zero. The first `execute` from `Main` opens chain `0.0`, so `One` reads `0.0.0` and `Two`, the next entry of the same chain, reads `0.0.1`. The second `execute` opens chain `0.1`, where the same two sections read `0.1.0` and `0.1.1`. `Main` still reads `0.0` after both, and the walk goes on to `Last` as `0.1`.

An `execute` takes its slot from the child counter that calls, tasks, and fanout arms share, so after one `call` from `Main`, the first `execute` runs as chain `0.1`. The rule is the same at every depth: the first `execute` that the task `0.2` makes runs the prompt's first section as `0.2.0.0`. An executed prompt is not a task, so `sys.taskid` in it reads the caller's task (see [How an executed prompt runs](08-jump-and-call.md#how-an-executed-prompt-runs)).

### Ids inside fanout arms

````markdown
---
name: mixed-ids
description: Reads section entry ids from calls and fanout arms.
runvil: 0
---

## Main

```lua
local a = call('Sub')
local r = tasks.fanout('Worker', {'x', 'y', 'z'})
local b = call('Sub')
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
0.0,0.0.0,0.1.0,0.2.0,0.3.0,0.4.0,0.1
````

Each fanout arm is a new child chain of the caller's chain, numbered by the caller's next child slot, so its worker section reads `sys.id` as that chain id followed by `.0`. Here the first `call` took `0.0`, so the three arms read `0.1.0`, `0.2.0`, and `0.3.0`, and the second `call` takes the next slot, `0.4.0`. As the first children of the main walk, three arms read `0.0.0`, `0.1.0`, and `0.2.0`. Fanout arms take their ids in collection order when the fanout dispatches, so the ids are the same whichever arm finishes first.

`tasks.fanout`, alone or followed by a `call`, leaves the calling section entry's `sys.id` unchanged, so `Main` still reads `0.0` after its arms return. Dispatching calls and fanout arms does not advance the walk's own entry count either, so after `Main` makes two calls and a three-arm fanout, `Last` still reads `0.1`.

Ids nest the same way at every depth:

| Where the section entry is | `sys.id` |
|---|---|
| the first section a `jump` reaches after the worker, in the first arm | `0.0.1` |
| the first section of a `call` made inside arm chain `0.0` | `0.0.0.0` |
| the two arms of `tasks.fanout('../Worker', {'a', 'b'})` inside a called `Sub` (chain `0.0`) | `0.0.0.0` and `0.0.1.0` |
| the inner arms of a fanout inside outer arm `0.0`, whose worker reads `0.0.0` | `0.0.0.0` and `0.0.1.0` |

Sections that a `jump` reaches from a fanout arm continue the arm chain's `sys.id` sequence. A `call` made inside an arm starts a contained chain that is the arm's child, so with the arm at chain `0.0` the called chain is `0.0.0`. A fanout inside a called section nests its arm ids under the called section's chain, not under the root chain, and a fanout inside a fanout arm nests the inner arms under the outer arm's chain.

### Ids inside a task

````markdown
---
name: task-ids
description: Reads the task id, section entry id, and index inside each arm.
runvil: 0
---

## Main

```lua
local r = tasks.fanout('Worker', {'x', 'y'})
return r[1].text .. ' | ' .. r[2].text .. ' | ' .. sys.id
```

### Worker

```lua
return sys.taskid .. ' ' .. sys.id .. ' ' .. sys.index
```
````

````text
0.0 0.0.0 1 | 0.1 0.1.0 2 | 0.0
````

Each arm is a task. Inside a spawned task, `sys.id` is a section entry id that extends the task id, while `sys.taskid` is the task id itself, so the first section of task `0.0` reads `sys.taskid` as `0.0` and `sys.id` as `0.0.0`. A task started with [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) follows the same rule.

A `sys.id` can have the same text as a task id, because a chain counts its section entries and its children separately. Here `Main` reads `sys.id` as `0.0` while its first arm is task `0.0`. Task ids are reproducible in the same way as `sys.id`, because they come from the same chain-local child counters, so the arm tasks `0.0` and `0.1` keep their ids whichever arm finishes first.

### Using `sys.id` as a key

Every `sys.id` in a run is distinct. Entries in one chain differ in their counter, and entries in different chains differ in their chain path, as with `0.3` against `0.3.0`. `sys.id` values are also the same on every run of the same prompt with the same inputs, however chains interleave or tasks finish, because each id is a path of chain-local counters, and every counter stays inside its own chain.

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

A log checkpoint is a short progress message your Lua records by calling `log(message)`. It appears in the run's event stream as a `lua` checkpoint event, and `log` itself returns zero values. Each checkpoint renders as the text `Lua: ` followed by the message and holds the message text exactly as you passed it to `log`. The first two checkpoints above are attributed to `Prepare` and the third to `Finish`.

`log` accepts exactly one argument: a single-line UTF-8 string of at most 256 Unicode characters with no newline or control characters. [Message rules](#message-rules) gives the details.

### Where you can call `log`

`log` is an Engine global, always present in every section VM. You can call `log(message)` to record checkpoints from any Lua block: in any section, in the preamble, in a fanout arm, and in prologues and epilogs alike.

`log` stays defined for the section entry's whole lifecycle, so every block in the entry can call it. It is an ordinary function value that you can assign to a variable and call through that variable, as in `saved_log = log`, and a saved reference keeps working in later blocks of the same entry (see [A fresh section VM for every section entry](#a-fresh-section-vm-for-every-section-entry)). A library section's top-level code can call `log` while [`require`](03-blocks-and-prose.md#sharing-code-with-require) loads it, because section setup installs `log` before it installs `require`.

`print` and `warn` read nil in every section VM (see [Names that read nil](#names-that-read-nil)), whether in a library section's code, a prologue, an epilog, or a lone block, so `assert(print == nil)` and `assert(warn == nil)` pass everywhere, and `log` is the only way your Lua reports progress.

### Which section a checkpoint names

````markdown
---
name: log-checkpoints
description: Records checkpoints across the preamble, a prologue, and an epilog.
runvil: 0
---

```lua
log('preamble ran')
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
Lua: preamble ran
Lua: prepare started
Lua: prepare finished
Lua: finish started
````

Each `log` checkpoint is tagged with, and attributed to, the section name of the section whose Lua called `log`. The preamble's section name is empty, so a `log` call in it records its checkpoint under the empty section name, and `preamble ran` comes ahead of every later section's checkpoints. The next two are recorded under `Prepare`, and `finish started` under `Finish`.

A checkpoint is attributed to the section whose code called `log`, including a section reached through `call`: when `Outer` runs `call('../Inner')`, `log('inner ran')` in `Inner` is attributed to `Inner`. Calling `log` inside a task's section records the message as a checkpoint under that section's name, and `log` records each message under the name of the section that logs it, in order, including inside a spawned child. When several runs execute at the same time, each run's checkpoints stay in that run's record and appear in that run's call order.

### Checkpoint order

The run records each `log` message verbatim, attributed to the section that called it, in call order across blocks. Several `log` calls in one section appear in the trace in the order the Lua made them, and checkpoints from separate Lua blocks in the same section appear in the order the blocks run, as `prepare started` and `prepare finished` do above.

Checkpoints come out in a deterministic order across the preamble, prologues, and epilogs, and a checkpoint comes only from code that runs. In `Finish`, the prologue returns, so the epilog is skipped and `unreachable epilog` is never recorded. Checkpoints logged before an early return, such as `finish started`, still appear in the trace.

`log` emits its checkpoint at the moment of the call, through the same channel the runtime's own reports use, so checkpoints sit in call order among chunk, store, and teardown reports. `log` checkpoints, message text included, interleave with Engine operation reports such as [store](09-the-store.md#writing-and-reading-files) reports in program order:

````lua
log('before write')
store.write('state.txt', 'value')
log('after write')
````

This block records the `before write` checkpoint, then the write's store report, then the `after write` checkpoint, so the two `log` calls bracket the write's report.

Logging leaves a prompt's results unchanged. A block's return value, the `var` contents, and the store's contents are the same whether or not anything observes the checkpoints, so you can add `log` calls freely.

A `log` message is recorded verbatim with no redaction, so keep it to your own status wording, and leave out arguments, replies, tool data, credentials, paths, and store contents.

### Message rules

A `log` message is one string argument that holds valid UTF-8, has at most 256 characters, and fits on a single line. `log` rejects a number, so pass `tostring(n)` for a number:

````lua
local n = 3
log('items: ' .. tostring(n))
````

A call that breaks a message rule raises a Lua error in the block naming the broken rule, and the call's message is never recorded:

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
- A message is a single line of text with no control characters. A newline, a carriage return, a tab, U+2028 LINE SEPARATOR, and U+2029 PARAGRAPH SEPARATOR each break this rule, and a message containing any of them raises `log message must not contain newline or control characters`.

When one `log` call breaks several rules, you see the first failing check in this order: argument count, event quota, string type, UTF-8 validity, the 256-character limit, control characters, then the byte quota. The two quota checks are the subject of [Log quotas](#log-quotas). A call without exactly one argument fails with `log expects exactly one argument` and leaves both quotas untouched, while every one-argument call spends one unit of the log event quota before its type, length, and line-break rules are checked, so a one-argument call that breaks a rule has still spent its unit.

### Log quotas

The log quotas are the two limits each section VM puts on `log`: the log event quota, which counts calls, and the log byte quota, which counts the UTF-8 bytes of the recorded messages.

Each `log` call spends one event from the log event quota, so with a quota of 1, `log('one')` followed by `log('two')` fails on the second call. The log event quota caps the `log` calls in each Lua VM separately, at 1024 by default, and the Harness can set another count.

All `log` calls in a section VM share a cumulative log byte quota of 256 bytes per allowed checkpoint, so the byte quota follows the event quota: 262,144 bytes by default, and 1024 bytes with a 4-event quota. Each message that passes the other checks spends its length in UTF-8 bytes from that shared quota, so multibyte text uses it up faster, and a 200-character message of two-byte characters such as `é` costs 400 bytes. That way the total log volume of many small messages stays bounded.

Both quotas span every block in the section entry rather than resetting per block. Every `log` call counts against the section VM's log quotas, including calls a library section makes while `require` loads it, because a library loads into the requiring section entry's one environment. The next section entry starts with full `log` quotas.

A call past either quota fails with one of these texts:

````text
lua log event quota exceeded
lua log byte quota exceeded
````

The first comes from a call past the log event quota and the second from a call past the log byte quota. The byte quota can refuse a call even while events remain in the event quota. A message whose UTF-8 byte length would overflow the byte quota is refused whole, and messages logged before it stay recorded. A refused call leaves the remaining byte quota unchanged, so a later shorter message that fits can still be logged, though the refused call has already spent its event unit.

With a log event quota of 4 set by the Harness, the log byte quota is 1024 bytes, and this block fails on its third call:

````lua
log(string.rep('é', 200))
log(string.rep('é', 200))
log(string.rep('é', 200))
return 'unreached'
````

Each call costs 400 bytes, so exactly two checkpoints are recorded. A `log` call refused by the byte quota stops the block, so the `return` never runs, and the uncaught refusal ends the run with code `lua_quota` and the message `lua log byte quota exceeded (at {file}:{line})`, which names the line of the refused call. That holds for both quotas: an uncaught `log` quota refusal, event or byte, ends the run with `lua_quota`.

`pcall` catches either refusal as an error value with error kind `lua` whose `message` gives the refusal text as [author prose](#reading-kind-and-message) after the line of the refused call, such as `{file}:{line}: lua log byte quota exceeded. The section logged more than its budget allows. Log less.` Only the Engine's own refusal counts as a quota refusal, so a block that raises the same text itself, as in `error('lua log event quota exceeded')`, fails as an ordinary Lua runtime error.

### Lifecycle reports

A lifecycle report is a run report that shows one stage of a section entry's Lua lifecycle. Lifecycle reports hold the section name and leave out every value, such as a shared global's value or the run's arguments.

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

- When a prompt is parsed, each Lua block's compilation reports a started report and then exactly one succeeded or failed report, each tagged with the block's section name and leaving out the source text and the [location label](#error-line-numbers-and-locations).
- The first `require` of a library section reports a library load started report and then a succeeded or failed one, naming the library's heading under the requiring section. A repeat `require` of a loaded library is silent, and each retry of a failed load reports a fresh pair (see [Library errors, cycles, and load events](03-blocks-and-prose.md#library-errors-cycles-and-load-events)).

## Asking the operator

After this section you can declare that a prompt needs a person to answer it, wait for the operator's next message, and write a prompt that runs with or without one.

````markdown
---
name: ask-once
description: Waits for the operator's next message and returns it.
runvil: 0
uses:
  - user
---

## Main

```lua
local text = tasks.receive()
return text
```
````

When the operator sends `kept`, the run result is:

````text
kept
````

The operator is the person who answers a run while it runs. The operator's messages go to the run's main chain, the chain of the main walk (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)), and the main chain reads them with `tasks.receive()`. Declaring the `user` Pack, as `ask-once` does, says that the prompt needs an operator, so a run with nobody to answer refuses the prompt at its first step, before any of its Lua runs.

`tasks.receive()` pauses the section until the operator's next message and returns that message's text as a string, exactly as it was sent. Leading and trailing spaces, newlines, and tabs are kept, with no trimming or re-encoding, and the text never arrives wrapped in the untrusted `<untrusted_input_...>` markers.

### Declaring the Pack

You declare that a prompt needs an operator by listing the Pack id `user` under the frontmatter [`uses:`](12-tools.md#tools-packs-and-the-uses-key) key. Write it as the line `  - user` after `uses:`, as the `ask-once` prompt does, or in flow style as `uses: [user]`. A bare entry always means the `runvil` Pack of that id, so `uses: [runvil/user]` declares the same Pack.

The Pack's fully qualified name is `runvil/user`, and every Host has it built in, beside `runvil/subagents`, with no Plugin to add it. Its tool list is empty and it lacks a prelude, so declaring it leaves the prompt's globals and tools as they were. The main chain reads the operator with `tasks.receive()` whether or not the prompt declares `user`, and what the declaration adds is the requirement.

### When a run has a user

A run has a user when the Host that launches it has someone to answer its main chain, and the Host decides that for each run it launches. The `user` Pack is available only in a run that has a user. In any other run the Pack is unavailable, with the reason `this run has no user`, so a prompt that declares it is refused at the run's first step with code `requirements_unmet`, before any of its Lua runs. The [requirements notice](04-how-a-prompt-runs.md#the-requirements-notice) names the Pack and the reason and ends with where the prompt declares the Pack, here the `uses:` line of the `ask-once` prompt:

````text
the environment cannot satisfy this prompt:
- user is unavailable: this run has no user (at ask-once.md:5)
````

The refusal is a guarantee: in a run of a prompt that declares `user`, the main chain always has an operator to answer its `tasks.receive()`, so it can skip any check before it waits.

A prompt that leaves `user` out of its declared Packs stays clear of this refusal, unless a prompt that it lists under [`prompts:`](08-jump-and-call.md#listing-prompts-under-prompts), directly or through another listed prompt, declares it. Its main chain can still call `tasks.receive()` in a run with no user, and the call waits all the same, because a Host can send the run messages from its own code.

### Checking for a user before waiting

A prompt that works either way leaves `user` undeclared and checks for the Pack before it waits. `tools.packs.get('user')` returns the Pack's object in a run with a user and `nil` in a run without one, because only a Pack the run can serve has a Pack object (see [Finding a Pack with `tools.packs.get`](12-tools.md#finding-a-pack-with-toolspacksget)):

````lua
if tools.packs.get('user') then
  return 'you said: ' .. tasks.receive()
end
return 'nobody to ask'
````

With this block as the Lua block of `Main` in a copy of `ask-once` that leaves out `uses:`, a run with a user whose operator sends `hi` returns `you said: hi`, and a run with no user returns `nobody to ask`.

### What `tasks.receive()` takes

`tasks.receive()` takes no arguments. Passing any argument raises a Lua error of kind `lua` whose message names the argument's type, such as `tasks.receive takes no arguments, got string`, so a prompt that passes a question fails loudly instead of losing it. Passing an explicit `nil` still counts as an argument and raises `tasks.receive takes no arguments, got nil`. `pcall` catches the error, and a refused call leaves the inbox as it was, so the next `tasks.receive()` still returns the operator's next message.

`tasks.receive()` reads the inbox of the main chain or, inside a task, the inbox of that task, and a section that a `call` or an `execute` runs reads its caller's inbox. The Host that launched the run decides whose messages reach its main chain.

### Messages that arrive first

The main chain has an inbox, a queue that holds the operator's messages, oldest first. A message that arrives while the main chain is not waiting in `tasks.receive()` stays in the inbox until the main chain reads it, and so does a message the operator sends before the run starts. A `tasks.receive()` with a message already queued returns the oldest queued message at once, without pausing. When the operator sends `one` and then `two` before the main chain reaches this block, the block returns `one|two`:

````lua
local first = tasks.receive()
local second = tasks.receive()
return first .. '|' .. second
````

Every part of the main chain reads that one inbox: the preamble, the sections after it, every section the main chain runs with [`call`](08-jump-and-call.md#calling-a-section), and every section of a prompt it runs with [`execute`](08-jump-and-call.md#executing-another-prompt), so a `tasks.receive()` in a section run with `call` takes the operator's next message. A message one section leaves unread is still queued when the next section begins.

A `models.loop` on the main chain reads that inbox too. It takes every queued message before each model request, and once more before it returns unless its round cap is spent, so a message that arrives while the loop runs goes to the model rather than to the next `tasks.receive()` (see [Messages that arrive while the loop runs](11-conversations.md#messages-that-arrive-while-the-loop-runs)).

The operator's messages reach only the main chain. A task, whether [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) or [`tasks.fanout`](14-fanout.md#fanout-in-one-call) started it, has an inbox of its own, so a `tasks.receive()` in a task waits on that inbox, and the operator's messages stay with the main chain. Only the task's owner fills it (see [Messages between tasks](15-tasks.md#messages-between-tasks)).

### While the section waits

The section entry's Lua VM state survives a `tasks.receive()` wait, however long the operator takes. With this block as the Lua block of `Main` in the `ask-once` prompt, an operator who sends `kept` gets the run result `41 kept`:

````lua
local before = 41
local text = tasks.receive()
return tostring(before) .. ' ' .. text
````

A `tasks.receive()` wait pauses only the main chain, and the rest of the run keeps going while it waits, because `tasks.receive()` is a [suspending call](#suspending-calls). The wait is not Harness work: the Engine holds it, and it waits on the inbox alone. When the main chain waits there and every other chain has run out of work, the run stays open. It waits for the operator's next message, or for a cancel.

Canceling the run while the main chain waits in `tasks.receive()` ends the run promptly with the canceled error, whose code is `canceled`, so the code after the call never runs.

A stop of the run is different. A stop drops the model round and the tool calls in flight (see [Stopping a model round](01-what-a-prompt-is.md#stopping-a-model-round)), and a `tasks.receive()` wait is neither, so a stop leaves the wait open, and the operator's next message still returns from it. A stop leaves the inbox as it is too, so every queued message stays queued.

### The operator and the model

Waiting for the operator is the prompt's own work, beyond the reach of a model's tools. A model's reply without tool calls ends [`models.loop`](11-conversations.md#starting-a-conversation) once the loop finds the inbox empty, or at once when its round cap is spent, so a prompt that holds a conversation with the operator reads each message itself and adds it to the conversation, as `messages:user(tasks.receive())` does before the next `models.loop()`.

## Error values

After this section you can read a caught failure's kind and message and branch on them.

````markdown
---
name: read-error
description: Catches an error value and reports its kind and message.
runvil: 0
---

## Probe

```lua
local ok, err = pcall(function() return sys.nothing end)
return type(err) .. '|' .. err.kind .. '|' .. tostring(err)
```
````

The run result is:

````text
table|lua|read-error.md:10: unknown sys field 'nothing'. The Lua code failed at runtime. Fix the code at this location.
````

`nothing` is an unknown field of [`sys`](#run-facts-in-sys), so the read fails, and `pcall` catches the failure instead of letting it end the block (see [Raising and catching errors](#raising-and-catching-errors)). What comes back is an error value: a Lua table holding a `kind` string, a `code` string, a `message` string, and any extra fields its kind defines. Every failure that an Engine call or an Engine function raises reaches your code as one, so `type(err)` is `table` and never a plain string, whether the failure came from a model round, a store operation, or a guard like the one on `sys`.

The message starts with where the failure happened, `read-error.md:10:`, the prompt file and the line of the `sys.nothing` read, and goes on with what failed and what to do about it, as [Reading `kind` and `message`](#reading-kind-and-message) explains.

The error kind is the value's `kind` field, a lowercase snake_case string such as `lua`, `out_of_scope_tool`, or `empty_model_reply`. It names which failure occurred, so you read it at any call site and branch on it.

The value's `code` field names the exact failure, as the stable tag of the Engine's error code: a lowercase snake_case string such as `lua_runtime`, `lua_quota`, `not_found`, or `overloaded`. One kind can hold several codes, so `code` tells apart failures that share a kind. A `lua` value's `code` is `lua_quota` for a refused `log` call, `call_depth_exceeded` for the call-depth cap, `substitution` for a failed `{{ }}` placeholder, `args_invalid` for the arguments of an [`execute`](08-jump-and-call.md#arguments-for-an-executed-prompt) that the executed prompt's `args:` declaration refuses, and `lua_runtime` for a runtime, argument, or memory failure. A `canceled` value's `code` is `canceled` for a canceled run or a stopped call and `task_canceled` for a canceled task. A `model_failure` value's `code` is its [completion error kind](16-limits-and-errors.md#completion-errors-and-how-lua-sees-them) in snake_case, such as `overloaded` or `rate_limited`, and a `store` value's `code` is the same tag as its `reason`. Every error value has a `code`, including an argument error the Engine's own Lua raises, whose `code` is `lua_runtime`.

An Engine function can fail in its own Lua code or inside the Engine itself, and both give the same error value, with the same shape and the same metatable, so your code handles the two the same way.

### Reading `kind` and `message`

Read both fields with ordinary table access:

````lua
local ok, err = pcall(function() return sys.nothing end)
if not ok and err.kind == 'lua' then
  log('lookup failed: ' .. err.message)
end
````

`err.message` is always a string. For a failure the Engine places on a line of your prompt, it is `{file}:{line}: {prose}`: the prompt file, the line of your code that made the failing call, and the failure's prose. For any other failure it is the prose alone. An Engine fault never shows a location, because the place it broke is in the Engine's own source. When whatever raised the error left the message out, the message is the `kind` string itself, so `tostring(err)` shows the kind. [Where an error points](16-limits-and-errors.md#where-an-error-points) gives the rules for which file and which line.

`tostring(err)` returns exactly the message, with no traceback, so you can print it, join it into other text, or search it with string functions, as in `tostring(err):find('not found', 1, true)`. The `..` operator gives the same text with the error value on either side, so `'context: ' .. err` and `err .. ' suffix'` each join the message with the other operand. `tostring` gives the message in the same way for the value that a failed `require` or a refused `setmetatable` call hands back.

The prose comes from the root cause. It names the error the Engine function itself returned, with any callback frame or added context around it stripped away, and the message leaves out the stack traceback that the Lua runtime would otherwise add to an error from an Engine function. The Engine raises its errors at the call site the way `error(result, 0)` does, so the only position is the `{file}:{line}: ` in front: a model round that fails with `model is down` gives `{file}:{line}: model is down`, with the line of the [`models.infer`](10-models.md#running-a-round-with-modelsinfer) call.

The prose depends on what caused the failure. A failure your prompt's own Lua caused has author prose, `{what}. {default prose}`: what failed, then the default prose for that kind of failure. That covers a Lua error, an exhausted `log` quota, the call-depth cap, a failed substitution, the refused arguments of an `execute`, a misused task, and a `tasks_live` leak, and it covers the refusals of Runvil's guards, so reading `sys.nothing` gives `unknown sys field 'nothing'. The Lua code failed at runtime. Fix the code at this location.` after its location, and the refusal of `sys.extra = 1` gives `sys is read-only; cannot set 'extra'. The Lua code failed at runtime. Fix the code at this location.` A failure from a tool, the store, a model, or the context window keeps its own prose instead, as `model is down` does. The prose never names a file or a line. Where this guide quotes what a failure says, such as `unknown sys field '{field}'` or a `log` rule's refusal, that text is the `{what}` of its author prose.

Messages you meet through `tostring(err)` include these, shown without the `{file}:{line}: ` that a located one starts with:

- `models.infer takes (prompt) or (model, prompt)`, an argument error the Engine's own Lua raises, whose message is this text alone (see [Argument errors](#argument-errors))
- `models.loop takes (compactor?) or (model, compactor?). The Lua code failed at runtime. Fix the code at this location.`
- `lua log event quota exceeded. The section logged more than its budget allows. Log less.`
- `the model replied with no text and no tool calls`
- ``section `../Missing` not found: {where}. The Lua code failed at runtime. Fix the code at this location.``
- ``recursion exceeded cap of 8. Calls nest deeper than the cap allows. Nest `call`, `execute`, `tasks.spawn`, and `tasks.fanout` less deeply.``
- ``args is read-only: assign to `var` or a section global instead. The Lua code failed at runtime. Fix the code at this location.``

Because `kind` is a string and `tostring` gives the message, one expression can report a failure in full:

- `err.kind .. '|' .. err.name .. '|' .. tostring(err)`, after a model calls a tool that is not offered in its section entry, gives text that starts `out_of_scope_tool|rogue|{file}:{line}: tool "rogue" is not in this section's scope;`, with the line of the `models.loop` call.
- `err.kind .. '|' .. tostring(err)`, after a tool-call loop that never settles, gives `tool_loop_exhausted|{file}:{line}: The tool-call limit was reached before a final reply. Stop calling tools and reply with what you have.`
- `err.kind .. '|' .. tostring(err.finish_reason) .. '|' .. tostring(err)` reports an `empty_model_reply` failure with its finish reason.
- `err.kind .. '|' .. err.tasks .. '|' .. tostring(err)` reports a `tasks_live` failure with its `tasks` field.
- An argument rejected for not being a positive whole number gives kind `lua` and a message containing `positive whole number`.

### The sixteen error kinds

An error value's `kind` is always one of exactly sixteen strings. The set is closed, so every failure falls into one of them, and each string names exactly one kind, so you match a kind by comparing `err.kind` against its string, as in `err.kind == 'store'`. The table lists each kind with the fields it adds beside `kind`, `code`, and `message`:

| Kind | Fields beside `kind`, `code`, and `message` |
| --- | --- |
| `lua` | none |
| `internal` | none |
| `canceled` | `task`, when it reports a canceled task |
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
| `model_only_tool` | `name` |
| `store` | the store fields, such as `path` and `reason` |
| `tool` | none |

You read a kind's fields from the same error value after branching on `kind`, with ordinary table access such as `err.reason`. They are plain fields held in the table itself, so `err.kind`, `err.reason`, and `err.finish_reason` on an error from [`models.loop`](11-conversations.md#starting-a-conversation) read like any other table field. A kind with no fields gives a value holding only `kind`, `code`, and `message`.

Every field is a string, except one `store` field that is an integer: `count`, on an anchor failure (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). `type` returns `'number'` for it and `%` works on it, so `type(err.count) == 'number' and err.count % 1 == 0` holds.

Each kind comes from the feature that raises it:

- `context_exhausted` comes from a model round that runs out of context, through `models.infer` or `models.loop`.
- `empty_model_reply`, `tool_loop_exhausted`, `out_of_scope_tool`, `unbound_tool`, and `canceled` can each come from `models.loop`. `out_of_scope_tool` names a tool that is not offered in the section entry (see [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer)).
- `tool` reports a tool failure from [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls).
- `model_only_tool` comes from a script's `tools.call` of a tool that only a model may call, and its `name` field holds the name the script passed (see [Model-only tools](12-tools.md#model-only-tools)).
- `tasks_live`, `task_not_owned`, and `task_consumed` come from the task functions (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) and [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin)).
- `store` comes from store operations, for the failures the store itself reports. A `tasks.spawn` or a tool call that the run's files refuse raises it too (see [Store codes](16-limits-and-errors.md#store-codes)).
- `canceled` reports a cancellation, and has `task` when what was canceled is a task.
- `model_failure` comes from a model call's transport, backend, or decode failure, or a disabled gateway, through `models.infer` or `models.loop`.
- `model_required` comes from a model call on a run whose Host gave it no model, the [missing-model error](10-models.md#infer-errors-and-the-missing-model-error).
- `internal` comes from a failure that the other kinds leave out: a section entry's failed tool freeze (see [Tool codes](16-limits-and-errors.md#tool-codes)), an Engine fault, or an Engine function failure caused by the Lua runtime's own machinery rather than by an authoring or argument problem.
- `lua` covers every failure on the Lua side, as shown next.

### The `lua` kind

Kind `lua` covers every failure on the Lua side whenever it reaches your code as an error value, your own `error` and `assert` calls included. When a spawned task's section runs `error('beta boom')`, the section that joins it finds `results[1].result.kind == 'lua'` in the `tasks.join` result, and `tostring` of that result contains `beta boom`.

A `lua` error value marks an authoring or runtime failure, and its message gives the error's own text, such as the unknown field or the store path, as author prose after the failing line, except for an argument error the Engine's own Lua raises, which is that text alone (see [Argument errors](#argument-errors)). These failures are all kind `lua`:

- a compile error (see [Compile errors](#compile-errors))
- a runtime error in your own code, or a syntax error raised from inside an Engine function call
- an argument or misuse error from an Engine function, including a value it cannot convert to the type it expects and a value it fails to serialize or deserialize
- the arguments of an `execute` that the executed prompt's `args:` declaration refuses (see [Arguments for an executed prompt](08-jump-and-call.md#arguments-for-an-executed-prompt))
- an exhausted log quota, as with a refused `log` call (see [Logging and lifecycle reports](#logging-and-lifecycle-reports))
- a placeholder or substitution in prose that fails to render (see [Placeholders in prose](07-substitution.md#placeholders-in-prose))
- a `call`, `execute`, `tasks.spawn`, or `tasks.fanout` that nests past the call-depth cap
- running out of Lua memory
- an Engine function failure caused by an outside error that is not one of the Engine's own errors

A `lua` error value holds only `kind`, `code`, and `message`, whichever of these failures it reports. Its `code` sets a refused `log` call, the call-depth cap, a failed substitution, and the refused arguments of an `execute` apart, while a runtime error, an argument error, and a memory refusal share the `code` `lua_runtime`, so only the message text tells those three apart.

Nesting a [`call`](08-jump-and-call.md#calling-a-section), an [`execute`](08-jump-and-call.md#executing-another-prompt), a `tasks.spawn`, or a [`tasks.fanout`](14-fanout.md#fanout-in-one-call) arm past the call-depth cap raises a `lua` error value on the line of the call that went too deep, and its message names the cap the same way whichever call it was: ``{file}:{line}: recursion exceeded cap of 8. Calls nest deeper than the cap allows. Nest `call`, `execute`, `tasks.spawn`, and `tasks.fanout` less deeply.`` A `pcall(tasks.spawn, ...)` in the deepest spawner catches it, with `tostring(err)` reading that message for the line of its own `tasks.spawn` call.

Left uncaught, a `lua` failure ends the run with its own code, one of the Lua codes or, for the refused arguments of an `execute`, the run code `args_invalid` (see [Run error codes](16-limits-and-errors.md#run-error-codes)). A runtime, argument, or memory failure ends it with `lua_runtime`, and a failed placeholder with `substitution`. A compile failure in the prompt's own blocks never reaches Lua: it fails the parse with `lua_compile` before the run starts. One in a Pack prelude ends the run with `lua_runtime`. An exhausted log quota, as with a refused `log` call, ends it with `lua_quota`, and a recursion-cap refusal with `call_depth_exceeded`, so an uncaught call-depth refusal ends the run with `call_depth_exceeded` and the message `recursion exceeded cap of 8 (at {file}:{line})`, which names the line of the call that went too deep.

### Argument errors

An argument error from an Engine function is an authoring error, so its kind is `lua`. The Engine raises it at the call itself, before anything is sent, so a malformed argument to a model round, a task operation, or a tool call never becomes a request:

````lua
local ok, err = pcall(models.infer, 'yo', { temperature = 0 })
return err.kind .. '|' .. tostring(err)
````

`models.infer` takes the prompt, or a model handle and the prompt, so this block returns `lua|models.infer takes (prompt) or (model, prompt)` and finishes without making a model request. Sampling options such as `temperature` belong to [`models.use`](10-models.md#sampling-options-on-modelsuse).

A caught argument error names the argument in fixed text. When the Engine's own Lua raises it, as `models.infer` does here, the message is that text alone. When the Engine raises it from inside the call, the message gives that text as author prose after the line of your call. Argument errors from `tools.call` are worded and shaped exactly like the others, failures from the `tasks` functions, `tasks.fanout` included, arrive in the same shape as every other Engine failure, and so do the errors `models.loop` builds itself. Rendered as `err.kind .. ':' .. tostring(err)`, every argument error from `models.loop` starts with `lua:`, as in `lua:{file}:{line}: models.loop takes (compactor?) or (model, compactor?). The Lua code failed at runtime. Fix the code at this location.` (see [Argument errors](11-conversations.md#argument-errors)), and the errors the loop raises about the compactor passed as its argument are kind `lua` too.

Argument type errors on every Engine function, the `store` functions included, are kind `lua`. Of a store call's failures, only the ones the store itself reports are kind `store`, so `pcall(function() store.write(123, 'x') end)` catches `err.kind == 'lua'`.

A string argument has three messages:

| When | Message |
| --- | --- |
| A required string argument is missing | `{name} must be a string, got nil` |
| A string argument has another type | `{name} must be a string, got {type}` |
| A string argument holds bytes that are not valid UTF-8 | `{name} must be a valid UTF-8 string` |

`{name}` is the argument's name, such as `prompt`, `path`, `contents`, `old`, `new`, `pattern`, or `text` for a task's note. `{type}` tells integers from floats, so `3` reports `got integer` and `2.5` reports `got number`, and a `prompt` given as `42` reports `prompt must be a string, got integer`. The wording is the same whichever part of the Engine raises the error. An optional string argument accepts a string or `nil`, or can be left out, and any other type fails with `{name} must be a string, got {type}`.

An [`execute`](08-jump-and-call.md#executing-another-prompt) that the Engine refuses at the call raises a `lua` error value too, and its `code` tells the refusals apart: `lua_runtime` for a name that the calling prompt does not list under `prompts:`, `args_invalid` for arguments that the executed prompt's `args:` declaration refuses, and `call_depth_exceeded` at the call-depth cap. The declaration checks the arguments before the executed prompt starts, so the refusal comes before any of its sections runs, and `tostring(err)` gives the mismatch as author prose after the line of the `execute`, as in ``{file}:{line}: argument `query` must be a string, got integer. The arguments do not match what the prompt declares. Send one JSON object that matches its declared arguments.`` (see [Arguments for an executed prompt](08-jump-and-call.md#arguments-for-an-executed-prompt)).

### Engine errors keep their own kind

An Engine call that succeeds returns its values directly, and one the Engine rejects raises an error instead of returning a failure flag. When an Engine function raises one of the Engine's own errors, the caught value keeps that error's kind, code, message, and fields, and is never reported under the generic `lua` or `internal` kinds:

````lua
local ok, err = pcall(store.read, 'missing.txt')
return err.kind .. '|' .. err.reason .. '|' .. err.path
````

When the store lacks `missing.txt`, this block returns `store|not_found|missing.txt`.

A model round fails the same way. A `pcall(models.infer, 'hi')` that runs out of context returns `false` and an error value of kind `context_exhausted` with `reason` set to `provider`, and the value keeps that `reason` wherever you pass it. Every failure the Engine answers to your Lua has a kind, a spent limit included, such as kind `lua` for an exhausted `log` quota, whose message gives `lua log event quota exceeded` as author prose.

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

`tostring(result)` gives the failure's own prose after the line of the `models.infer` call, and `error(result, 0)` raises the same error value again, as [Raising and catching errors](#raising-and-catching-errors) explains.

The Engine recognizes its own error values by a metatable they share, not by their fields. All error values in one section VM share a single metatable, so `getmetatable` returns the same table for any two caught errors. A table you build yourself therefore stays an ordinary Lua value, whatever kind it names: `error({ kind = 'tool_loop_exhausted', message = 'x' }, 0)`, left uncaught, ends the block as an ordinary Lua runtime error, because only a table with the shared metatable is an error value.

## Raising and catching errors

After this section you can catch an Engine failure with `pcall`, handle the kinds you expect, and raise the rest again.

````markdown
---
name: careful-read
description: Reads a note, or reports that there is none yet.
runvil: 0
---

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

This prompt only reads `note.md`, so the run result is:

````text
no note yet
````

`pcall` runs [`store.read`](09-the-store.md#writing-and-reading-files) in protected mode. The read fails, so instead of the failure ending the block, `pcall` returns `false` and the error value. The block handles the one failure it expects, a missing note, and raises every other error value again unchanged with `error(result)`, which fails the run exactly as if the block had never caught it. In a prompt where an earlier block wrote `note.md`, `pcall` returns `true` and the text, and the block returns the note. The same shape works for any kind, as in a block that catches `pcall(models.infer, prose)`, returns `'model unavailable: ' .. err.message` when `err.kind == 'model_failure'`, and otherwise raises the value again with `error(err)`.

`pcall(f, ...)` calls `f` in protected mode, passing on any arguments after it, and returns a success flag first: `true` followed by every value `f` returned, or `false` followed by an error value or the value your code raised. A re-raise is raising a caught error value again unchanged, with `error(err)` or `error(err, 0)`.

### What `pcall` returns

The usual form is `local ok, err = pcall(f, ...)`, after which `err.kind` gives the error kind and `err.message` the message string. Outside cancellation, a caught failure always comes back as `false` followed by the error value (see [Cancellation passes through `pcall`](#cancellation-passes-through-pcall)).

- A successful `pcall` returns `true` followed by every return value of the protected function, nils included, so `select('#', pcall(function() return 1, nil, 3 end))` gives `4`.
- Extra arguments go on to the function, so `pcall(error, 'boom')` calls `error('boom')`.
- `pcall` returns `false` when the wrapped function raises, so `not pcall(f)` is `true` when `f` fails.
- A failure caught by `pcall` or `xpcall` leaves the block running. The block can return normally, and when it does, the run ends normally with the returned value.

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
- `pcall(models.loop)` catches an error the loop raises, and `err.kind` names it, such as `empty_model_reply`, `canceled`, `out_of_scope_tool`, or `unbound_tool`. With a compactor, as in `pcall(models.loop, f)`, it catches the loop's failures the same way, and the model rounds inside still suspend and resume normally.
- `pcall(call, '../Missing')` catches a failed call, such as one whose path misses, and returns `false` and the error value instead of failing the run. A target that is not a string, as in `pcall(call, 42)`, comes back with a message containing `section target must be a string`.
- `pcall(jump, '../Other')` catches a failed [`jump`](08-jump-and-call.md#jumping-to-a-section).
- `pcall(execute, 'search.md', { query = 5 })` catches a failed [`execute`](08-jump-and-call.md#catching-a-failed-execute), whether the Engine refuses the call or the executed prompt itself fails.
- `pcall(tasks.fanout, 'Worker', 5)` returns `false` and an error value whose message is about the collection, and `pcall(tasks.spawn, ...)` catches a failed spawn.
- `pcall(tools.call, 'web/fetch', { url = url })` returns `false` and the raised value when the tool call fails.
- `pcall(store.append, 'notes.md', line)` takes a store function directly.
- `pcall(tasks.concurrency, value)` passes its argument on to the Engine function, and returns `false` and the error value when the call fails.
- `pcall(require, '/broken_style')` returns `false` when a library raises, and `pcall(require, 'os')` lets the block recover from a sandbox refusal and keep running.

Reading or assigning `prose` can fail with the read-only, recursive, and missing-key failures (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), and `pcall(function() return prose end)` catches them as an error value whose `tostring` contains the message. The refusals of assignments to `args` and `prose` are caught the same way, with the refusal on the first line of `tostring` of the caught error. A failed read through `prompt.section` or `prompt.section_or` (see [Reading sections with `prompt.section` and `prompt.section_or`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptsection_or)) is catchable with `pcall` too, and `tostring` on it gives text containing the message.

### Which failures become error values

Every failure the Engine raises into Lua reaches `pcall` as an error value, Engine-raised kinds such as `tool_loop_exhausted` included, so `type(err)` is `'table'` and you decide what to do by branching on `err.kind`. Three routes lead there:

- An argument error from an Engine call, such as a badly shaped argument to `models.infer`, `models.loop`, `call`, `execute`, `tasks.fanout`, `tasks.spawn`, `tools.call`, or a `store` function, is raised at the calling line as a `lua` error value rather than failing the whole block, so a `pcall` around the call catches it.
- A failed Harness request, such as a model round, arrives as an error value whose `kind` names the failure, such as `context_exhausted`, with the kind's own fields beside it and `tostring` giving the failure's own prose after the line that made the call. A chat call wrapped in `pcall` that a stop drops likewise comes back as `false` and an error value that has both `kind` and `message`.
- An Engine function that fails on the spot, such as a failing [`tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer), a read of an unknown `sys` field such as `sys.nothing`, or the `var` guard, gives the same indexable error value. `err.kind` and `err.message` work on it at the call site, and `tostring(err)` is the message, author prose after the line that was running when it failed, with no traceback appended. A refused `var` write gives a message containing `must be JSON data` (see [When `var` rejects a write](#when-var-rejects-a-write)), and a runtime error raised inside a built-in call is kind `lua`.

Apart from an argument error the Engine's own Lua raises, whose message is its text alone, each of these is placed on your own line: a suspending call on the line that made the call, an Engine function that fails on the spot, such as [`jump`](08-jump-and-call.md#jumping-to-a-section), on the line running when it failed, and a failure inside a function you pass to `pcall` or `xpcall` on the line of the failing call in that function, not the `pcall` line. A `call` or an `execute` can raise a failure placed elsewhere: when the chain or the prompt it ran fails, it raises that failure's own error value, which keeps the line where it happened, in the executed prompt's file for an `execute` (see [Errors that pass through several blocks](#errors-that-pass-through-several-blocks)). [Where an error points](16-limits-and-errors.md#where-an-error-points) gives the full rules, tail calls included.

The `pcall` and `xpcall` globals you call are the Engine's own versions, which make this conversion, and they replace the standard library's.

`pcall` rewrites only a failure raised directly by an Engine function. Any other raised value comes back unchanged as its second result: a string, a number, a boolean, `nil`, your own table, or an error value already built. A string or table you raise with `error`, as in `error('plain', 0)` or `error(tbl)`, comes back from `pcall` or `xpcall` exactly as raised, and a table keeps its identity, so `err == tbl`. The one exception is a string that starts with `?:?: ` or with the position of an Engine file: it loses that start, as it does in the run's failure. A runtime error from your own Lua code, such as calling a nil value, reaches `pcall` as Lua's plain string error, not as an error value. In every case, `tostring(err)` gives the message text.

### Raising errors with `error` and `assert`

Raise an error yourself to fail a block on purpose:

````markdown
---
name: checked
description: Fails on purpose when the answer is wrong.
runvil: 0
---

## Verify

```lua
local answer = 'no'
assert(answer == 'yes', 'the answer must be yes')
return 'checked'
```
````

The run fails with code `lua_runtime`, the run error's text contains `the answer must be yes` and ends with ` (at checked.md:11)`, the line of the `assert`, and the block ends before `return 'checked'` runs.

`assert(condition, message)` raises an error that holds `message` when `condition` is false, and returns normally and silently when it is true, so the block goes on to its next statement. `assert` works everywhere in prompt Lua, in sections, in fanout arms (see [What an arm sees](14-fanout.md#what-an-arm-sees)), and in [adhoc tool](01-what-a-prompt-is.md#effects-and-harness-work) handlers, for checking a condition mid-run with a message that says what was expected. Called with the condition alone, as in `assert(false)`, it raises a Lua runtime error whose text is `assertion failed!`, after a location prefix such as ``section `Web Search` epilog:51:`` (see [Error line numbers and locations](#error-line-numbers-and-locations)).

`error(message)` raises `message` and stops the current block, so everything after it is skipped, the prose after the fence included. A string raised this way, as in `error('boom')`, fails the block with an ordinary Lua runtime error whose message includes your text. Called in a block, its top-level code included, `error(message)` fails that block and with it the whole run, and the run error's text contains your message.

`error` fails the run only when it executes. That lets you place `error(message)` in a section as a guard that a `jump`, a chain's jump, or a fall-through (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)) should skip. In a `Skipped` section that the walk is meant to jump past, a block holding only `error('the jump target must skip this section')` fails the run if the jump ever misses, and when the section is skipped, the `error` never fires and the run succeeds.

`error(message, 0)` raises at level 0, so the raised value is the bare message string with no position prefix, which suits code whose text must arrive exactly as written, as in `error('compacted', 0)`. Passing `error` itself as the protected function, as in `pcall(error, 'boom')`, gives the bare string back as the error value.

`error` also accepts a table as the raised value, such as a caught error value or one you build, as in `error({ kind = 'custom', message = 'handler exploded' })`. `pcall` returns your table unchanged, and left uncaught it fails the block as an ordinary Lua runtime error, because only the Engine's own error values bring a kind into the run (see [Re-raising a caught error value](#re-raising-a-caught-error-value)).

The message of that failure names the raised value without its memory address, so that the failure reads the same on every run. An uncaught `error({ kind = 'custom' })` reads `error object is a table value`. A table whose metatable has a string `__name` puts that name in place of `table`, and a table with a `__tostring` metamethod reads as the text the metamethod returns.

The Engine finds such a value by the text Lua makes of it, a name, then `: `, then an address, and it cannot tell that text from a string of your own that ends the same way. So an uncaught error string that ends in `: ` and a hexadecimal address changes too: on Linux and macOS, `error('size: 0x1f', 0)` reads `error object is a size value`, and `error('bad byte: 0x1f')` ends in `bad byte: (address)`. On Windows an address is 16 uppercase hexadecimal digits with no `0x` in front, so there those two strings stay as written and `error('size: 000000000000001F', 0)` changes. A string with text after the number, such as `error('see 0x1f: fine')`, stays as written everywhere, and a string that `pcall` returns keeps its address. End an error message with something other than a bare hexadecimal number when the number must reach the run's failure.

The Engine also takes the position of its own code off the front of an error string. Lua puts the position of the function that raised an error in front of the message, and for an error that Lua raises inside one of Runvil's functions that position names an Engine file, or reads `?:?:` for a function that has no line. A `stack overflow` under `tostring` is such an error. So an error string that starts with `?:?: `, or with `crates/runvil-internal/lua/src/`, a file name, `:`, a line number, and `: `, loses that start, in the run's failure and in the string that `pcall` returns or an `xpcall` handler receives: `error('?:?: boom', 0)` reads `boom`. The Engine reads the text, so it cannot tell such a start from one that your own string has. Only that start goes, and the same text anywhere else in a string stays as written. A `__close` metamethod of a `<close>` variable runs before `pcall` returns, so the error it receives as its second argument still has that start: for a `table.concat` of a list whose `__index` is a number it reads `?:?: attempt to index a number value`, and the `pcall` around it returns `attempt to index a number value`. Start an error message with something else when the whole of it must reach the run's failure.

A spawned section that fails with `error('message')` reports its failure to its caller, such as in a `tasks.join` result, as an error value of kind `lua` whose `tostring` contains the message text.

An uncaught raise inside library code, during its load or from a library function called later, fails the block that made the call (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)). A `require` failure that the section does not catch ends the run with code `lua_runtime`, and the run error's text contains the raised message. Caught, the error from a failed `require` converts with `tostring` to a message that contains the text the library block raised, though not necessarily at the start, so a library that raises `fault: bad text` is found with an unanchored pattern such as `tostring(err):match('fault: .*$')`.

Engine functions raise errors and name value types the same way after your code rebinds `error`, `type`, `tostring`, or `math`.

### What an uncaught error ends

An uncaught Lua runtime error ends the block with an error instead of a return value. An uncaught Lua failure, a runtime error in your code or an error you raise on purpose, then ends the run with code `lua_runtime`, holding the failure's message, wherever it happens: the preamble, any other walked section, a `call` [chain](04-how-a-prompt-runs.md#the-section-walk), a prompt run with `execute`, a task, a fanout arm, or a library's load. The run's message is the failure's text, its causes included, followed by where it happened: ` (at {file}:{line})` for a failure on a line of the prompt file, after the stack traceback when the failure has one (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). Bytes of the failure's message or of its traceback that are not valid UTF-8 become the Unicode replacement character U+FFFD, as in the name of a function held under a key such as `'\xff'`.

When an Engine operation fails and the block does not catch the failure, the block fails with the error the operation returned. The run ends with that call's own specific error, placed on the line of the call, such as a `context_exhausted` failure from `models.infer`, rather than a plain string or a generic Lua runtime error. An uncaught error from a built-in Lua function ends the run with code `lua_runtime`, and its message names the calling function, as in `tools.offer: ...`. The exception is a `context_exhausted` raise: an uncaught `compactors.fail` call ends the run with code `context_exhausted`. An uncaught `models.loop` error ends the run with its own code: `tool_loop_exhausted`, `context_exhausted`, or, for an `empty_model_reply` error, `empty_reply`. A `lua`-kind loop error, such as a call with the wrong arguments or a loop started while another runs, fails the run as a Lua runtime error with code `lua_runtime`, and its message keeps the loop's own text.

An uncaught Lua failure leaves its block in one of two forms: a Lua error, which holds a message, or a Lua runtime error, which holds a message plus further details. A Lua runtime error's message already holds the Lua error's text, so a printout of the error and its causes shows that text once rather than repeating it. A store failure you do not catch keeps its own error instead and ends the run with its store code, such as `not_found` (see [When a store call fails](09-the-store.md#when-a-store-call-fails)).

### Re-raising a caught error value

A re-raise passes a failure on as if you had never caught it:

````lua
local ok, reply = pcall(models.infer, prose)
if not ok then
  error(reply, 0)
end
return reply
````

Raised again unchanged, with `error(err)` or `error(err, 0)`, before any other suspending call, a caught error value ends the block and fails the run with the original error, its kind, code, fields, and line included, exactly as the uncaught failure would have, so the run's message points at the line of the call that failed, not at the `error` line. Raising the value with `error(err)` and catching it again yields the same error value, not a new one wrapped around it.

Only the exact error value you caught keeps the original error. Catching an error and raising anything else, even a value with the same message, makes the failure the block's own error, and the run reports the new error instead of the original. A string built from the value, such as `error(tostring(err), 0)` or `error(err.message)`, fails as an ordinary Lua runtime error even when the text matches, and that runtime error uses the string as its message, so the original text, such as `{file}:{line}: model is down`, still appears in it, while the run now points at the `error` line.

The Engine reads a raised value back as a structured error only when it is one of its own error values. Your own table, even one with `kind` and `message` fields such as `{ kind = 'tool_loop_exhausted', message = 'x' }`, fails the block as an ordinary Lua runtime error when it goes uncaught.

A `lua` error value raised again after another suspending call ends the block as an ordinary Lua runtime error that keeps the same message, names the source block, and gives the absolute prompt line, as in ``section `Test` prologue:41:``.

### Raising an error value after another suspending call

An error value kept in a local survives a suspending call such as `call`, so you can still raise it afterward with `error`. This fragment comes from a section with a `Sibling` child:

````lua
local ok, err = pcall(store.read, 'missing.txt')
call('Sibling', '')
error(err)
````

Once another suspending call has run, the run ends according to the error value's kind:

| Error value raised after another suspending call | How the run ends |
| --- | --- |
| `lua` | as an ordinary Lua runtime error at the `error` line, with code `lua_runtime` |
| `canceled` from a canceled run or a stopped call, with no `task` field | with the canceled error, whose code is `canceled` |
| any other kind, a canceled task's `canceled` value included | with the original error and its own code, such as a store code like `not_found` for `store`, a model code like `overloaded` for `model_failure`, or `task_canceled` for a canceled task's value |

An error value of any kind except `lua` keeps its kind, code, fields, and line in the error that ends the run this way instead of being flattened to its message string, so a `context_exhausted` value keeps its `reason`, such as `precheck`. The fragment above ends the run with the store's code, `not_found`, and the store's original message, which still points at the line of the failed `store.read` rather than the `call` or the `error` line (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). A run ends with its result, its final text or the error that ended it, a cancel included, as [The run result](04-how-a-prompt-runs.md#the-run-result) explains.

### Message handlers with `xpcall`

Call `xpcall(f, handler, ...)` like `pcall`, with a message handler that sees the failure first:

````lua
local ok, handled = xpcall(error, function(e) return 'handled ' .. tostring(e) end, 'boom')
return tostring(ok) .. '|' .. tostring(handled)
````

This block returns `false|handled boom`. Arguments after the handler go on to the function, so this `xpcall` calls `error('boom')`. Outside cancellation, the handler receives the same error value `pcall` would return, so a failing built-in or Engine call reaches it as an error value whose `e.kind` and other fields are readable, and the handler's return value becomes the second result of `xpcall` after `false`. The handler is a function, as in standard Lua, and a handler of any other type gets standard Lua's own `xpcall` argument error on the line of your call, as in `bad argument #2 to 'xpcall' (function expected, got number)` for `xpcall(f, 5)`. `pcall()` with no argument raises `bad argument #1 to 'pcall' (value expected)` the same way. Each error names the function as you write the call, as standard Lua does, so a call through a local variable gives the name of the variable. When the call is the whole of a `return` inside a function, as in `return xpcall(f, 5)`, Lua makes it a tail call, so the error names the line that called your function, or no line when a `pcall` called it, where standard Lua names the line of the `return`. Such an error names the function `xpcall` or `pcall` whatever variable you call it through.

### Cancellation passes through `pcall`

Your `pcall` cannot swallow cancellation. While the run is being canceled, `pcall` raises the failure again instead of returning it, so even a retry loop around `pcall` unwinds and the run ends with the canceled error, whose code is `canceled`. `xpcall` lets cancellation through as well: under cancellation the Engine skips its message handler and raises the failure again, so the abort goes ahead even with a handler that loops forever.

## Error line numbers and locations

After this section you can go from an error message straight to the failing line of your prompt file.

````markdown
---
name: verify-answer
description: Fails on purpose to show where an error points.
runvil: 0
---

## Check

```lua
local answer = 'no'
if answer ~= 'yes' then
  error('the answer must be yes')
end
```
````

The run fails with code `lua_runtime`, and the first line of its message is:

````text
section `Check` prologue:12: runtime error: section `Check` prologue:12: the answer must be yes
````

``section `Check` prologue`` is the block's location label, and `12` is the absolute prompt line of the `error` call: line 12 of the file, counting the frontmatter. The label and line appear twice, once as a prefix at the head of the message and once where Lua's own position marker stood, with `runtime error:` between them, and a stack traceback follows on the next lines. After the traceback, the message ends with ` (at verify-answer.md:12)`, the prompt file and the same line (see [Where an error points](16-limits-and-errors.md#where-an-error-points)).

An absolute prompt line is a line number counted from the top of the prompt file, with the frontmatter's opening `---` as line 1. A location label names the Lua block an error came from, by its section and its place in the section. A caught [error value](#error-values) names the same line another way, with the prompt file instead of the label, in the `{file}:{line}: ` its message starts with.

### Absolute prompt lines

A Lua runtime error reports the failing line as an absolute prompt line, not a line counted from the top of the block, because each block keeps its starting line in the file. The absolute line is the block's first Lua line plus the line inside the block, minus one.

A block's first Lua line is the line right after its ```` ```lua ```` opener. In the prompt above, the fence opens on line 9, so the Lua starts on line 10, and the `error` call on the block's third line reports 10 + 3 - 1 = 12. Every line of the file counts: the frontmatter lines (`---`, `name:`, `description:`, `runvil:`, `---`), blank lines, headings, prose, and fence lines. Each block's first line is its true line in the `.md` file, in a library too and in files with CRLF line endings.

An error on a block's first line reports the line where the block's Lua starts, and an error on a later line reports that exact line. A block whose Lua starts on line 40 reports line 40 for a failure on its only line, line 41 for a failure on its second line, and line 43 for one on its fourth. The rule holds in every kind of block:

- a section's prologue: a `Work` prologue whose fence opens on line 10 reports an `assert(false)` on line 11 as line 11
- a section's epilog: an `assert(false)` on line 14 of a `Check` epilog reports line 14
- a block between prose blocks: the third of four Lua blocks in `S`, starting on line 23, reports an `assert(false)` on line 24 as line 24
- a preamble block: one whose code starts on line 8 reports its `assert(false)` as line 8
- a block deep in a long prompt: one starting on line 112 reports an `assert(false)` on its second line as line 113

The line always names the place in the prompt file where the error was raised, and it sits between colons, in the form `:12:`.

### Location labels

An error message names the block by its location label, followed by a colon and the absolute prompt line, in the form `{location}:{absolute}: {result}`. The label names the section by its section name, without the `#` markers, in backticks, followed by the block's role:

| Block | Location label |
| --- | --- |
| A Lua block that opens the section, before any prose, also when it is the only block | ``section `{name}` prologue`` |
| A Lua block that ends a section that has prose before it | ``section `{name}` epilog`` |
| Any other Lua block of a section | ``section `{name}` lua {ordinal}`` |

`{ordinal}` is the block's 1-based place among the section's Lua blocks. The role follows from the block's position, the section's block count, whether the section has any prose, and the block's place among the section's Lua blocks, so it matches the prologue and epilog of [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog) for an ordinary section. One section's two blocks read ``section `Private section` prologue`` and ``section `Private section` epilog``, a failing block under the heading `## lib` reports ``section `lib` prologue:19: load fault``, and the second block of `lib` is ``section `lib` lua 2``.

The [preamble](02-file-structure.md#the-preamble)'s section name is empty, so its blocks take the same three roles after the word `preamble`: `preamble prologue`, `preamble epilog`, and `preamble lua {ordinal}`, as in `preamble lua 2`. An `assert(false)` on line 8 of a preamble's first block fails with a message that starts `preamble prologue:8:`.

Every Lua block is its own unit with its own location, so four Lua blocks in `S` have four distinct locations. A long section name appears in full, so `Gather Background Research Notes` gives ``section `Gather Background Research Notes` prologue``. A location of any length still maps, so a block labeled ``section `Shared Research Helper Functions` (line 1200) prologue`` still reports `{location}:40:`.

### Sections that share a section name

When another section heading anywhere in the file, at any level, has the same section name, the location adds the heading's line as `(line {heading_abs_line})`, so you can tell the two sections' errors apart. With `Helpers` on line 10 under `A` and another `Helpers` on line 18 under `B`, the first section's prologue is ``section `Helpers` (line 10) prologue`` and the second's is ``section `Helpers` (line 18) prologue``.

The number is the line of the heading itself, not of the fence or the code, so a `### Helpers` heading on line 10 whose fence opens on line 12, with code from line 13, gives `(line 10)`. A heading line of three or more digits appears in full, as in ``section `Research Worker Notes` (line 109) prologue``. A section whose name is unique in the file gets a location without a line number, such as ``section `C` prologue``, even when other sections in the file share names.

Errors from two sections with the same name stay separate: each reports its own block's file line, and one `Helpers` section never maps the other's error. Location labels need not be unique for this to work, because the label is a display name only. Lua knows each block by a name built from its first prompt line, `line {source_line}`, and that name decides which block an error belongs to.

### Compile errors

A compile error is a Lua block that does not compile as Lua 5.5, such as one with a syntax error. Every Lua block in the prompt, the preamble's included, is compiled, without being run, when the prompt file is parsed, so a compile error stops the prompt before the run begins and before any of its code runs:

````markdown
---
name: broken-check
description: Holds an epilog that does not compile.
runvil: 0
---

## Check

Ask the model.

```lua
local secret =
return secret
```
````

The parse fails with parse error kind `Lua` (see [Parsing comes first](04-how-a-prompt-runs.md#parsing-comes-first)), and the message starts with ``lua compilation error at section `Check` epilog (line 12): `` followed by `syntax error: ` and the Lua compiler's diagnostic. A compile error has this form:

````text
lua compilation error at {location} (line {source_line}): {message}
````

- `{location}` is the block's location label, so the message tells you which block the problem is in, such as ``section `Transform` epilog``.
- `(line {source_line})` is the block's first Lua line, the line after its opening fence, counted from the top of the file with the frontmatter included. It is not the line of the fault, so an epilog whose fence opens on line 12 reports ``lua compilation error at section `Check` epilog (line 13): {message}``.
- `{message}` is the Lua 5.5 compiler's own diagnostic, unchanged, after a `syntax error: ` label. It names the block `[string "line {source_line}"]`, where `{source_line}` is the block's first prompt line, such as `[string "line 1"]` for a block that starts on line 1.

The compiler's diagnostic keeps its own line numbers, so the `{k}` in its `[string "line {source_line}"]:{k}:` marker counts from the top of the block, and the faulting file line is `{source_line}` + `{k}` - 1. In the prompt above, the diagnostic names `[string "line 12"]`, so its `{k}` added to 12, less one, is the file line of the fault.

Every block compiles this way, whether it sits in a library, in the preamble, or in any other section, the last block of the last section included: an unclosed `return (` in a final epilog fails the parse as ``section `Transform` epilog``. A syntax error in a section's prologue fence is located as ``section `{name}` prologue``, such as ``section `Private section` prologue``, and one in its epilog fence as ``section `{name}` epilog``. The compile error does not include the block's source: it holds the block's location label, its first line, and the compiler's diagnostic, which quotes at most the token where compiling stopped, such as `near <eof>`, or an unclosed string up to the end of its line.

The compile error's text is exactly the parse error's whole message. The parse error itself points at the faulting file line, the one the diagnostic's `{k}` names, in its own location fields, which a Host such as Workshop reads. The Harness finds the error before the run starts, and the failed run's code is `lua_compile`. Its message is the located message every failed run has: that text followed by ` (at {file}:{line})` for the faulting line (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). A compile error in a prompt's own blocks, or in the blocks of a prompt it lists under [`prompts:`](08-jump-and-call.md#listing-prompts-under-prompts), always fails the parse, so the prompt stops before the run starts. A compile error during a run, such as one in a [Pack prelude](12-tools.md#pack-preludes), ends the run with code `lua_runtime` (see [When a prelude fails](#when-a-prelude-fails)).

### Runtime error messages and tracebacks

Compile and runtime errors both name a block by its location label: a compile error as `at {location}`, and a runtime error as `{location}:{absolute}:`, the form Lua uses for a file. Before mapping, Lua's own position marker for a block reads `[string "line {source_line}"]:{k}:`, where `{source_line}` is the block's first prompt line and `{k}` is the line inside the block, so a block at prompt line 120 renders as `[string "line 120"]`. When a runtime error fails a block, the Engine rewrites every marker that belongs to the failing block or to a library it loaded with `require`, so the message shows a location label in place of each `[string "line {source_line}"]` from those blocks. Lua's raw `[string "line 50"]:2: assertion failed!` becomes text that starts ``section `Web Search` epilog:51: ``.

The mapped message opens with one `{location}:{absolute}: ` prefix, taken from the first marker it rewrote, so its head shows the location and absolute line twice: once as that prefix and once in place of Lua's own marker. The run reports this mapped diagnostic as the entire message of its Lua runtime error, with no extra type label in front, and the failed run's message adds ` (at {file}:{line})` after the traceback, for the line that prefix names.

Below its first line, an uncaught Lua runtime error has a `stack traceback:` whose frames each start with a location label and an absolute prompt line, as in ``section `lib` prologue:51: in`` and ``section `Main` prologue:31: in``, and go on with Lua's description of the frame, such as `function 'boom'` or `main chunk`. The traceback is taken where the error was raised, with the failing frames still on the stack, so it shows your own frames mapped to prompt lines rather than the Engine's wrapper around the block. It never shows `?:` placeholders, because blocks compile with their debug information kept.

The mapping changes a message only when the message names a line of the failing block or of a library that block loaded. Any other error text passes through exactly as it was, and so does a runtime error message whose lines all lie elsewhere.

### Errors that pass through several blocks

An error from one Lua block maps onto the lines of its own block, whichever other blocks share its section. When an error passes through several blocks, each frame maps using the starting line of the block it came from, so the calling section's frame reports its own prompt line, as when a `boom()` call on prompt line 11 appears as ``section `Main` prologue:11:``.

An error that surfaces through a parent block keeps the child's already absolute line, while the parent's own frames map to the parent's lines, so a message can start with ``section `Main` prologue:24: `` and still hold the child's line 51 intact. An error raised in a fanout arm or a called section traces back through its caller the same way, with each frame's line pointing at its own block's prompt line, so a child's ``section `lib` prologue:51: deep fault`` raised again in the parent stays at line 51. An error value that a called or spawned chain hands back keeps the line where it happened in that chain the same way, so its `{file}:{line}: ` names the child's line, not the parent's `call`, `tasks.fanout`, or wait. A failure inside a prompt run with [`execute`](08-jump-and-call.md#catching-a-failed-execute) keeps its file as well as its line: the message of the error value raised at the `execute` starts with the executed prompt's file and the failing line of that file, and left uncaught, the run's message ends with ` (at {file}:{line})` for that file and line, not for the line of the `execute`. A location label in that message names a block of the executed prompt, and its absolute line counts from the top of the executed prompt's file.

In a traceback that passes through an Engine function, your own frames still map to absolute prompt lines, as in ``section `Test` prologue:41: in``. The frames of Engine functions written in Lua, such as `tasks.fanout`, the other `tasks` functions, the `store` functions, and the Lua code behind other Engine calls, appear as bare `file:line:` references to the built-in helper's own file. Those frames stay as they are.

### Errors inside libraries

A Lua runtime error raised inside code loaded with `require` maps to a prompt line through the library block it came from:

````markdown
---
name: lib-fault
description: Shows where an error inside a library points.
runvil: 0
---

## Main

```lua
require '/lib'
boom()
```

## lib

```lua
function boom()
  error('deep fault')
end
```
````

The run fails with code `lua_runtime`, and the first line of its message is:

````text
section `lib` prologue:18: runtime error: section `lib` prologue:18: deep fault
````

The first line of an uncaught library error has exactly one leading prefix naming the raise point, then `runtime error:`, then the mapped message. The prefix names the library line where the raise happened, line 18 inside `boom`, even though the call came from ``section `Main` prologue``. The traceback below it shows each frame mapped once to its own prompt line, covering both the library frame, ``section `lib` prologue:18: in``, and the calling section's frame, ``section `Main` prologue:11: in``. After the traceback, the message ends with ` (at lib-fault.md:18)`, the same library line.

- A raise during a library's load reports the library block's location and the line of the raise, not the requiring block's line, so a library block whose Lua starts on line 40 and raises on its fourth line reports ``section `lib` prologue:43: load fault``.
- A raise from a library function called after the load reports the line inside the function body where the raise happened, so `error('later fault')` on the library block's second line reports ``section `lib` prologue:41: later fault``.
- In a library with several blocks, a raise in a later block maps to that block's own location and line rather than the first block's, as in ``section `lib` lua 3:21: second fault``.
- A message whose lines are already mapped keeps them when it passes back through the same block's mapping, such as when raised again with `error(msg, 0)` in a section that ran `require '/lib'`. For a library block whose Lua starts on line 50, a mapped line 51 stays 51 and never shifts a second time to 100.

## Giving `_G` a metatable

After this section you can put your own metatable on `_G` to give missing globals a default, make undefined globals an error, or hook new global writes, and to know what the Engine guard on `_G` still blocks.

````markdown
---
name: tone-default
description: Gives a missing global a default.
runvil: 0
---

## Reply

Tone: {{ tone }}.

```lua
require '/defaults'
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

Your metatable sits behind the Engine guard on `_G`, a metatable the Engine puts on `_G` before any of your code runs. `setmetatable(_G, mt)` records `mt` as your metatable behind that guard instead of replacing the guard, and reads of missing globals and writes of new globals reach your metatable as in standard Lua, apart from `args`, `prose`, and `messages`, which the guard handles itself (see [What the Engine guard on `_G` still blocks](#what-the-engine-guard-on-_g-still-blocks)).

### Defaults with `__index`

A read of a global that `_G` does not hold goes to the `__index` of your metatable, and an assignment of a new global goes to its `__newindex`, as in standard Lua. While your metatable is unset or lacks `__index`, an undefined global reads `nil`. An `__index` table answers by looking the key up in that table, and a global missing from both `_G` and the table reads `nil`. An `__index` function is called with `_G` and the key, and the read evaluates to what it returns:

````lua
setmetatable(_G, { __index = function(_, key) return 'fallback ' .. key end })
return missing_name
````

This block returns `fallback missing_name`.

An `__index` or `__newindex` that Lua cannot index, such as a number, fails at the first global access that reaches it, with Lua's own text, as in `attempt to index a number value`, and the run's failure names the line of that access. A `pcall` around the access returns the text with no position in front, where standard Lua puts the line there. The same holds for `error(msg, 2)` in an `__index` function of a table that you give as `__index`: the message gets no position, because the Engine's guard is a function between your line and that table.

`__index` and `__newindex` are read from your metatable at every lookup, so a change takes effect at the next global access, and your handlers run once per lookup however many blocks have run. An edit through `getmetatable(_G)` therefore works at once, so after the `defaults` library has run, this block returns `live tone`:

````lua
require '/defaults'
getmetatable(_G).__index = function(_, key) return 'live ' .. key end
return tone
````

Your metatable works in the [preamble](02-file-structure.md#the-preamble) as in any other section, so preamble code reads its defaults and gets its strict errors, and it stays in place across later fences of the same section, so a default still reads in the second fence.

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

This block returns `y`. The hook intercepts the assignment, so the value never lands in `_G`, and a later edit to `__newindex` on your metatable takes effect immediately. A `__newindex` table receives new globals instead, so the value lands in that table rather than in `_G`. While your metatable is unset or lacks `__newindex`, assigning a new global stores it in `_G`.

The Engine sets its own globals, such as `args`, before any of your code runs, so a `__newindex` hook sees only your own new globals, while `args` still reads normally. A library loaded with `require` can install the hook with `setmetatable(_G, { __newindex = ... })`, and the hook keeps capturing ordinary global assignments in the section entry that required it.

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

These fields are copied at each `setmetatable(_G, mt)` call, so a later edit to one of them takes effect only at your next `setmetatable(_G, mt)`, while `__index` and `__newindex` are always read live. A second `setmetatable(_G, mt2)` replaces the first metatable completely, so metamethods found only in the first one stop applying to `_G`. `setmetatable(_G, mt)` leaves your table `mt` unchanged.

### Reading and clearing the metatable

`setmetatable(_G, mt)` returns `_G`, just as `setmetatable(t, mt)` returns `t` for any other table, and after it `getmetatable(_G)` returns your own table `mt`. `getmetatable(_G)` never returns the Engine guard: it gives `nil` while your metatable is unset, even though the guard is in place, and it gives a required library's table after that library calls `setmetatable(_G, mt)`.

`setmetatable(_G, nil)` removes your metatable. Afterward `getmetatable(_G)` is `nil` and plain global assignments land in `_G` directly. Every metamethod copied from your metatable stops applying too, so a copied `__tostring` and `__call` are dropped: `tostring(_G)` reads `table` and calling `_G` fails.

This handling of `setmetatable` and `getmetatable` applies only to the global table itself, compared by raw identity.

### Protected metatables and refused fields

A `__metatable` field in your metatable protects it as it would any table. `getmetatable(_G)` then returns that field's value instead of the metatable, and the protected metatable keeps working, so its `__index` table still supplies globals:

````lua
setmetatable(_G, { __metatable = 'locked', __index = { tone = 'friendly' } })
return getmetatable(_G) .. '|' .. tone
````

This block returns `locked|friendly`. Once your metatable has a `__metatable` field, any later `setmetatable(_G, ...)`, `setmetatable(_G, nil)` included, raises `cannot change a protected metatable`, prefixed with the position of the calling line.

The second argument of `setmetatable(_G, mt)` must be present and be a table or `nil`. Anything else raises Lua's standard `bad argument #2 to 'setmetatable' (nil or table expected, got {type})`, where `{type}` reads `no value` when the argument is missing.

`_G` never becomes a weak table: the Engine keeps the `__mode` field out of its metatable. A `mt` with a `__mode` field makes `setmetatable(_G, mt)` raise a Lua error whose message ends with `setmetatable: weak tables (__mode) are not available for _G`, while every other table can still be made weak with a `__mode` field. A weak table reads and writes by key, and `pairs` and `next` refuse to walk it (see [Weak tables](#weak-tables)). Finalizers are not available in the sandbox at all: `setmetatable(t, mt)` with a `__gc` field in `mt` raises a Lua error with the message `setmetatable: finalizers (__gc) are not available in the sandbox`, whatever `t` is. A rejected `setmetatable(_G, ...)` call is dropped whole, so `_G` keeps the author metatable it had, if it had one.

### `setmetatable` and `getmetatable` on other values

On every value other than `_G`, `setmetatable` and `getmetatable` behave exactly as the standard Lua 5.5 base functions, including their return values, their argument errors, and `__metatable` protection:

- `setmetatable(t, mt)` returns `t`, and `getmetatable(t)` then returns `mt`. An `__index` function on an ordinary table supplies values for missing keys, so `setmetatable({}, { __index = function(_, k) return k .. '!' end }).x` is `x!`.
- `setmetatable` takes a table and a metatable that is a table or `nil`, and other arguments raise Lua's stock bad-argument error. Argument errors from either function name it as you write the call, as in `bad argument #1 to 'setmetatable'` or `bad argument #1 to 'getmetatable'`, and include the position of your calling line, as the standard functions' own errors do.
- `setmetatable` returns exactly one value, and `getmetatable` on a table without a metatable returns exactly one value, `nil`. `getmetatable` on a number returns `nil`.
- A metatable with a `__metatable` field is protected: `getmetatable` on its table returns that field's value instead of the metatable, even when the value is `false`, and `setmetatable` on that table raises the Lua error `cannot change a protected metatable`, whether the new metatable is a table or `nil`. Lists from `json.decode`, `json.array`, and structured tool results are a case of this: their metatable's `__metatable` field is `false`, so `getmetatable` on one returns `false` (see [Keeping empty lists with `json.array`](#keeping-empty-lists-with-jsonarray)).
- `getmetatable(var)` gives the guard string `var is guarded`, and `getmetatable(sys)` gives `sys is sealed`.

`getmetatable` in a prompt is the sandbox's replacement for the base function. `tostring`, `string.format`, and `table.concat` do not use it: each finds `__tostring` as Lua does, so a `__metatable` field never hides one from them.

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

The Engine guard on `_G` keeps three names to itself: `args`, `prose`, and `messages`. Your code reads them as globals, even though a raw read of any of them from `_G` gives `nil`, which is why a [listing of the globals](#listing-globals) leaves out all three. The guard handles `args`, `prose`, and `messages` before your metatable, so reads and assignments of those names never reach your `__index` or `__newindex`, whatever metatable you set, clear, or change. An assignment to `messages` always raises the refusal that [`messages` is the section entry's conversation](11-conversations.md#messages-is-the-section-entrys-conversation) shows.

In an epilog that follows the prose `Hi.`, this block returns `Hi.` even though its `__index` raises for every undefined global and its `__newindex` records every new one:

````lua
local seen = {}
setmetatable(_G, {
  __index = function(_, name) error('undefined global ' .. name, 2) end,
  __newindex = function(_, k, v) seen[k] = v end,
})
return prose
````

Because `setmetatable(_G, mt)` records `mt` behind the guard instead of replacing it, every `setmetatable(_G, ...)` call, whether it clears the metatable with `nil`, swaps in `{}`, or installs hooks, leaves what `args`, `prose`, and `messages` read as it was, and an assignment to any of them that the Engine refuses stays refused. Editing the table that `getmetatable(_G)` returns leaves them guarded too, whether you replace `__index`, add or replace `__newindex`, or clear `__metatable`.
