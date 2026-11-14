# Tasks

A task is a section that `tasks.spawn` starts as its own [chain](04-how-a-prompt-runs.md#the-section-walk), running beside the chain that spawned it rather than in its place. The waits, `tasks.join_any` and `tasks.join`, collect a task's result when your code is ready for it, and `tasks.cancel` and `tasks.pending` stop and list the tasks you started. `tasks.send` and `tasks.reply` pass messages between a task and its owner, so one task can answer, wait, and take a follow-up.

## Starting a task with `tasks.spawn`

After this section you can start a section as a task with `tasks.spawn`, seed what it starts with, and collect its result with `tasks.join`.

````markdown
---
name: one-task
description: Starts one task and waits for its result.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local results = tasks.join({ t })
return 'got: ' .. results[1].result
```

## Child

```lua
return 'child result'
```
````

````text
got: child result
````

The [section walk](04-how-a-prompt-runs.md#the-section-walk) enters `Main`, whose first line starts `Child` as a task and gets a task handle back at once in `t`. The second line passes that handle to `tasks.join`, which parks `Main` until the task has run and ended. `Child` hands back `child result` with an ordinary `return`, and that text arrives in the `result` field of the first entry `tasks.join` returns. `Main` then returns the run result, and because it returns, the walk never reaches `Child` as a walked section, so `Child` runs only as the task.

### What a task is

A task runs one section as its own chain beside the chain that spawned it. That chain is the task's owner: the task keeps running while its owner waits on Harness work such as a `store.write`, and the owner collects the task's result when it needs it. [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment) covers what an owner must settle before it ends.

A task runs beside the section that spawned it instead of in its place, and that is what sets it apart from [`call`](08-jump-and-call.md#calling-a-section), which runs a section and hands its value back to the calling block. While the owner of a `Child` task is parked on a `store.write`, the task can run all the way to its `return 'child result'`, and the owner then resumes and returns its own result, such as `done`, as the run result.

Reach for tasks when the work is not one collection run through one worker section. That shape is a [fanout](14-fanout.md#fanout-in-one-call), which runs every member of one collection through one worker section. Tasks fit the rest, such as starting two different sections, `Alpha` and `Beta`, and waiting on both.

Tasks need nothing in the frontmatter and have no command-line flags. The frontmatter keys are `name`, `description`, `runvil`, `max_tool_iterations`, `output`, `uses`, `args`, and `models`, and none of them is for tasks, so every task feature is a Lua call, an options table, a returned value, or text the model reads.

### The `tasks` namespace

Every section's Lua has the `tasks` table as a global, and your code reaches every task operation through it. You use it without declaring anything:

````markdown
---
name: tasks-global
description: Shows that the tasks global needs no declaration.
runvil: 0
---

## Main

```lua
return type(tasks) .. ' ' .. type(tasks.spawn)
```
````

````text
table function
````

The `tasks` global is a table in the Lua of every section, including the [preamble](02-file-structure.md#the-preamble) before the first heading, the sections that tasks run, and fanout [arms](14-fanout.md#what-an-arm-sees). The Engine installs it alongside the other [suspending calls](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), such as `call`, `models.infer`, `tools.call`, and `tasks.fanout`.

The table holds twelve task functions, `spawn`, `join_any`, `join`, `concurrency`, `ready`, `status`, `pending`, `note`, `receive`, `send`, `reply`, and `cancel`, plus `fanout`, which the Engine adds to the same table. A Lua block spawns tasks with `tasks.spawn`, waits on them with `tasks.join_any` and `tasks.join`, checks on them with `tasks.ready`, `tasks.status`, and `tasks.pending`, notes a task's progress with `tasks.note`, passes messages with `tasks.send`, `tasks.receive`, and `tasks.reply`, and cancels tasks with `tasks.cancel`. `tasks.concurrency` gives the effective [concurrency limit](14-fanout.md#concurrency-and-the-admission-window), and `tasks.fanout` is the fanout call itself. Their call forms include `tasks.spawn(target, opts?)`, `tasks.join_any(set, opts?)`, `tasks.join(set, opts?)`, `tasks.concurrency(n?)`, `tasks.pending()`, `tasks.receive()`, `tasks.send(task, text)`, and `tasks.reply(value?)`, where `?` marks an optional argument.

Each task function returns these values:

| Function | Returns |
|---|---|
| `tasks.spawn` | A task handle |
| `tasks.join_any` | The member's task handle, its `ok` flag, and its result text or error value, or `nil` when a timeout ends the wait first |
| `tasks.join` | A results sequence, then a `timed_out` flag |
| `tasks.concurrency` | The effective limit |
| `tasks.ready` | A boolean |
| `tasks.status` | A status table |
| `tasks.pending` | A sequence of task handles |
| `tasks.note` | Nothing |
| `tasks.receive` | The text of the chain's next inbox message |
| `tasks.send` | A boolean |
| `tasks.reply` | The text of the owner's next message |
| `tasks.cancel` | Nothing |

This section covers `tasks.spawn`. The waits are in [Waiting with `tasks.join_any` and `tasks.join`](#waiting-with-tasksjoin_any-and-tasksjoin), canceling and listing are in [Canceling and listing tasks](#canceling-and-listing-tasks), messages are in [Messages between tasks](#messages-between-tasks), the checks and the note are in [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote), wait timeouts are in [Time limits on waits](#time-limits-on-waits), and the limit is in [Concurrency, nesting, and scheduling](#concurrency-nesting-and-scheduling).

The helpers behind the `tasks` functions are never globals, so your code cannot call them directly, and the `coroutine` global reads `nil`. `tasks` holds task functions only and has no entry per section, so indexing it with a section name reads `nil`; you name a section by passing its section path to a function such as `tasks.spawn`.

### The task handle

`tasks.spawn(target, opts?)` starts a task over the target section and returns a task handle at once, without waiting for the task to run. The handle is a plain table, `{ task = id }`, with no metatable, no methods, and no other fields. Its single `task` field holds the new task's id as a string, such as `0.0` or `0.3`, and [Task ids and handles](#task-ids-and-handles) explains the id values.

````lua
local t = tasks.spawn('../Child')
log('spawned ' .. t.task)
````

Placed at the start of the opening prompt's `Main`, these two lines log `spawned 0.0`.

You can keep a handle in a local, such as `local a = tasks.spawn('Caller')`, collect handles into a Lua list, and pass them later to the other `tasks` functions. `local s = tasks.spawn('../Sibling')` followed by `local results = tasks.join({ s })` waits on a task, and `local t = tasks.spawn('../Child')` followed by `tasks.cancel(t)` cancels one. Wherever a `tasks` function takes a task, you can pass the handle or the bare id string, and any table with a string `task` field counts as a handle.

### When a task first runs

A spawned task does not run the moment you spawn it. This prompt logs on both sides of the spawn:

````markdown
---
name: spawn-order
description: Shows that a task first runs when its owner parks.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
log('spawned ' .. t.task)
store.write('park', 'x')
local results = tasks.join({ t })
return results[1].result
```

## Child

```lua
log('child ran')
return 'child result'
```
````

````text
child result
````

The log shows `spawned 0.0` before `child ran`. `tasks.spawn` hands the new task's id back to the spawning block before the task runs any of its code, so the spawning section is not suspended and keeps running to its next statement. A `log` line written right after the spawn therefore comes before anything the task logs, and at that moment the task has not even entered its section.

The task first runs once its owner parks on a call that waits for the Harness, and a slot is free under the concurrency limit (see [When a spawned task is admitted](#when-a-spawned-task-is-admitted)). Such a call is a store call like `store.write` or `store.exists`, a model round, or a wait on tasks. Here the `store.write('park', 'x')` gives the task its first chance to run. In a prompt whose `Child` notes its progress as `working` and then waits on a model round, a status read with [`tasks.status`](#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote), as `tasks.status(t)`, before that `store.write` shows `blocked=queued` with no section, and one after it shows `state=running section=Child blocked=chat note=working`. In the same way, an owner's `store.exists` call just before a `tasks.cancel` is what lets the task run before the cancel lands.

Because a task runs only once its owner suspends, a section that returns or fails without suspending leaves its tasks unstarted. A main walk that spawns twice and then does `return 'done'` runs straight through without suspending, and the run fails with a live-tasks error naming `0.0` and `0.1`. Both tasks are [abandoned](#abandoned-tasks), and neither one ever reports a start.

Once it starts, a task runs concurrently with its owner. The owner keeps working after `tasks.spawn` returns, for example running `store.exists` calls and `tasks.cancel` while the task is still waiting on a model reply from [`models.infer`](10-models.md#running-a-round-with-modelsinfer). You can spawn several tasks in a row, such as `local a = tasks.spawn('../A')` and `local b = tasks.spawn('../B')`, and each one runs its own [`models.loop`](11-conversations.md#starting-a-conversation) at the same time as the others and the owner.

### Choosing the section a task runs

`tasks.spawn` names the section the task runs by a relative [section path](02-file-structure.md#section-paths): the bare name of a child of the spawning section, such as `'Worker'`, or `../` and the name of a sibling, such as `'../Child'` or `'../Alpha'`. It resolves that path exactly as `call` resolves its target, over the sections visible from the spawning section (see [Resolving section paths](08-jump-and-call.md#resolving-section-paths)). It checks the path with the same rules `call` applies to its target, and it shares `call`'s [recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap). So a path that names no visible section fails `tasks.spawn` with the same error text `call` gives for that path, and `pcall` catches it the same way.

The target can be a child of the spawning section, such as `'Worker'` or `'Caller'` spawned from the block under `Main`, or a sibling of it, such as `'../Child'` spawned from `Main`. The task starts at the target section's own position in the document, the same way a `jump` and a `call` start theirs. Any section in the visible set can be a target, including one with no Lua of its own.

The spawning section's own child sections are visible from it and make good targets:

````markdown
---
name: pros-and-cons
description: Spawns two child sections and joins them.
runvil: 0
---

## Weigh

```lua
local pros = tasks.spawn('Pros')
local cons = tasks.spawn('Cons')
local results = tasks.join({ pros, cons })
return results[1].result .. '\n' .. results[2].result
```

### Pros

```lua
return 'Pros: fast to build'
```

### Cons

```lua
return 'Cons: hard to maintain'
```
````

````text
Pros: fast to build
Cons: hard to maintain
````

`Weigh` starts `Pros` and `Cons` as two tasks, joins both, and returns their results in the order it listed them.

You can spawn the same section more than once. Each spawn runs as its own task, with its own id, its own handle, its own lifecycle, and its own result: two `tasks.spawn('../Parked')` calls give two handles, and `tasks.pending()` lists both.

### What a task section returns

A task's target section runs ordinary Lua as the task body. It can call a tool with [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls), and the task stays live while that call is outstanding, so a wait on the task lasts until the call returns and the section finishes. It can [offer](12-tools.md#offering-tools-and-packs-with-toolsoffer) its own tools and run its own model loop, as a block does that runs `tools.offer('web/search')` in a prompt that declares the `web` Pack, adds `messages:user('go')` to its own `messages` list, and calls `models.loop()`. It also sees every [Pack prelude](12-tools.md#pack-preludes) global, so in a prompt that declares the made-up `acme/issue-tracker` Pack from that section, a `Child` task can call `tickets.open('Follow up')`. A task cannot wait for the operator, whose messages reach only the main chain: a `tasks.receive()` in a task waits on the task's own inbox, which its owner fills with `tasks.send` (see [Messages between tasks](#messages-between-tasks)).

The section hands back its result with an ordinary `return`, such as `return 'child result'`. A task can return the text of `models.infer`: a task section doing `return models.infer('slow please')`, answered with the reply `slow answer`, gives its owner's wait `ok` and the result `slow answer`. A task that runs off its last section without returning gives the empty string as its result, the same as a fanout arm, and so does a task that ends at a `return` that gives [no value](04-how-a-prompt-runs.md#returning-no-value).

The [scalar return rule](04-how-a-prompt-runs.md#returning-values) applies as in any block, but a return ends the run only from a section on the main walk. A `return` from a section inside a task becomes the task's result for its owner, such as `return 'alpha text'` making `alpha text` the result, and it never ends the run or becomes the run result. The spawning section's own return still ends the run, so when an owner parks on a `store.write` while its task returns `'child result'`, and then returns `'done'`, the run result is `done`.

### Keeping task sections off the walk

A section that runs as a task is still an ordinary section. If the main walk reaches it, the walk runs it again as a walked section, and a spawn leaves the spawning section's own place in the walk untouched. No syntax takes a section out of the walk, so you keep task sections off the walk's path in one of two ways:

- Spawn child sections, which the walk never falls into.
- End the spawning section with `return`, as the opening prompt's `Main` does with its result, so that `Child` runs only as the task.

A child section runs only when another section starts it, because the walk never enters a child section by falling through. A `Child` under `A` that holds `error('a child must not run by fall-through')` never fires when `A` falls through to `B`, and the run gives `A` then `B`. For the same reason, child sections spawned as tasks, such as `Pros` and `Cons` above, run only as tasks, even when the spawning section falls through to a later sibling instead of returning.

### How far a task's chain runs

A task starts at its target and, like a called chain, falls through to the target's following sibling sections until a section returns or the last sibling ends. A target without a `return` therefore also runs the sections after it inside the task:

````markdown
---
name: task-fall-through
description: Spawns a section that falls through to its next sibling.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('Sub')
local results = tasks.join({ t })
return results[1].result
```

### Sub

```lua
log('Sub')
```

### After

```lua
return 'after'
```
````

````text
after
````

The task enters `Sub`, which logs and has no `return`, then falls through to `After`, whose `return 'after'` becomes the task's result.

### Seeding a task with spawn options

`tasks.spawn` accepts an options table as its second argument, with three optional keys, `args`, `item`, and `index`. These are the spawn seeds, and they give the task its own starting values:

````markdown
---
name: seeded-task
description: Starts a task with its own args, item, and index.
runvil: 0
---

## Main

```lua
var.k = 1
local t = tasks.spawn('../Child', { args = 'child args', item = { name = 'alpha' }, index = 7 })
local results = tasks.join({ t })
return results[1].result
```

## Child

```lua
return 'index=' .. tostring(sys.index) .. ' item=' .. item.name .. ' args=' .. args.text .. ' k=' .. tostring(var.k)
```
````

````text
index=7 item=alpha args=child args k=1
````

`Child` reads all three seeds, and it also reads `var.k`, which it gets from the `var` snapshot described below. The options table itself is optional: you can call `tasks.spawn` with a section path alone, such as `tasks.spawn('Gather')`, and a missing `opts` counts as an empty table. Leaving `args`, `item`, or `index` out starts the task without that seed, so `tasks.spawn(target)` alone gives the task no `args` of its own, no `item`, and no `sys.index`, and it still copies `var`. The target, the `args`, `item`, and `index` seeds, the `var` snapshot, and the model the task starts on are all fixed on the task by the `tasks.spawn` call, before the task first runs.

The options argument is a table. Any other value raises an error of kind `lua` whose message names the type it got, `tasks.spawn opts must be a table, got {type}`, such as `tasks.spawn opts must be a table, got integer`. The error is raised at the call site, so `pcall` around the call catches it and gives `false` as its first value (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). The table has no key for a model, and [The model a task starts on](#the-model-a-task-starts-on) says which model a task uses.

### Giving a task its own `args`

`args` takes a table of named fields, which becomes the task's [`args`](06-arguments.md#reading-arguments-from-args) global, or a string, which the task reads as `args.text`: `args = 'child args'` makes `' args=' .. args.text` give `args=child args` above. A `call` takes the same two forms as its second argument, with the same meaning (see [`args` in a called chain](08-jump-and-call.md#args-in-a-called-chain)). A task given its own `args` reads them in every section its chain runs.

Without `args`, the task reads its owner's `args`. This prompt gives one task a table and spawns another with no `args` option:

````markdown
---
name: task-args
description: Spawns one task with its own args and one that reads its owner's.
runvil: 0
args:
  type: object
  properties:
    topic: { type: string }
    depth: { type: string, default: quick }
  required: [topic]
---

## Main

```lua
local own = tasks.spawn('../Child', { args = { topic = 'reefs' } })
local same = tasks.spawn('../Child')
local results = tasks.join({ own, same })
return results[1].result .. '\n' .. results[2].result
```

## Child

```lua
return args.topic .. ' ' .. tostring(args.depth)
```
````

Run it with the arguments `{"topic":"tides"}`:

````text
reefs nil
tides quick
````

The first task reads the table its spawn gave it, so its `args.topic` is `reefs`. The Engine does not check a spawn's arguments against the prompt's [`args:` declaration](06-arguments.md#declaring-arguments-in-frontmatter), and it fills no defaults in them, so that task's `args.depth` is nil even though the declaration gives `depth` a default. A spawn can also give a field the declaration does not list, as in `{ args = { n = 5 } }`, or leave out `topic`, which the declaration requires, and the Engine starts the task with that table as its `args`. The declaration describes what a caller passes the prompt, not what one section passes another. The second task reads its owner's `args`, which here are the run's arguments, so it reads the `topic` the run started with and the default of `depth`.

`{ args = args }` passes the owner's arguments on, and a table can hold parts of them, as in `{ args = { topic = args.topic } }`. A task's `args` is read-only, whether the spawn gave it or the task reads its owner's, as `args` is in every section (see [`args` is read-only](06-arguments.md#args-is-read-only)).

A table that is a list fails with `args must be a table of named fields or a string, got a list`, which covers `{ 1, 2 }`, `json.array()`, and a list from the owner's `args` such as `args.files`. To pass a list, put it in a named field, as in `{ args = { files = args.files } }`. An empty table `{}` is the empty object, so `args = {}` gives the task an empty `args` instead of its owner's. A value of any other type fails with the same sentence ending in its type name, such as `got integer` for `5` or `got boolean` for `true`. A table that holds a value with no JSON form fails as in `args: value.f must be JSON data, got function`, for a function in its field `f`, and a string that is not valid UTF-8 fails with `args must be a valid UTF-8 string`.

### Seeding `item`

`item` takes any JSON data value, a string, number, boolean, or table such as a table of named fields, and that value becomes the task's [`item`](14-fanout.md#what-an-arm-sees) global. A table value keeps its fields, so `item = { name = 'alpha' }` makes `item.name` read `alpha` above. An empty table is sent as the empty object `{}`, so seed an empty list with `json.array()`, as in `item = { rows = json.array() }`, to have it arrive as `[]` (see [Encoding and decoding JSON with `json`](05-lua-environment.md#encoding-and-decoding-json-with-json)). The value is converted to plain data when `tasks.spawn` runs, so the task receives a converted copy instead of a live reference to your Lua value. `item` is also the source of the `{{ item }}` placeholder inside the task, where a table renders as compact JSON (see [Inserting the fanout or task `item`](07-substitution.md#inserting-the-fanout-or-task-item)). Without `item`, the task has no `item` global.

Spawning the same section several times with different `item` values gives each spawn its own task:

````markdown
---
name: item-seeds
description: Spawns one section twice with different item seeds.
runvil: 0
---

## Main

```lua
local one = tasks.spawn('Sub', { item = 'one' })
local two = tasks.spawn('Sub', { item = 'two' })
tasks.join({ one, two })
return store.read('sub-one.md') .. ', ' .. store.read('sub-two.md')
```

### Sub

```lua
store.write('sub-' .. item .. '.md', 'sub ' .. item)
return 'written'
```
````

````text
sub one, sub two
````

Each `Sub` task writes its own store path, `sub-one.md` holding `sub one` and `sub-two.md` holding `sub two`, and `Main` reads both once the join has delivered the tasks.

An `item` whose Lua type is `function`, `userdata`, or `thread` fails at the call site with a `lua` error naming the Lua type, `item must be JSON data, got {type}`, such as `item must be JSON data, got function`. That check runs first, on the value itself. `json.null` is the one userdata it accepts, as `item` itself or inside a table, and it is sent as JSON null, so `item = { a = json.null }` reaches the task as `{"a":null}`. Any other value that cannot convert, such as a table holding a function deeper inside, fails at the call site with a `lua` error whose refusal starts `item must be a JSON-representable value: ` and ends with the conversion failure.

### Seeding `sys.index` with `index`

`index` is a Lua integer of 0 or more, and it becomes [`sys.index`](14-fanout.md#what-an-arm-sees) inside the task: `index = 7` makes `sys.index` give `index=7` above. Spawning `Sub` with `{ index = i }` for `i` from 1 to 3, where `Sub` runs `models.infer('S' .. sys.index)`, sends the prompts `S1`, `S2`, and `S3`. Without `index`, `sys.index` is absent in the task, as it is outside a fanout. With `item` and `index` together, a task gets its own `item` and `sys.index`, as a fanout arm has.

A negative integer fails at the call site with a `lua` error that names the value, `index must be a non-negative integer, got {index}`, such as `index must be a non-negative integer, got -1`. A value of any other type, a float such as `1.0` or `2.0` included, fails with the same sentence ending in its Lua type name, such as `got number` or `got string`.

The `item` and `sys.index` seeds reach only the first section a task enters. A later section the task falls through to or jumps to has no `item` global and no `sys.index`, while `var` carries across.

### The `var` snapshot at spawn time

`tasks.spawn` seeds the task with a deep copy of the caller's [`var`](05-lua-environment.md#keeping-state-in-var), taken at the moment of the call and carried as plain JSON data, the same kind of [`var` snapshot](08-jump-and-call.md#call-args-and-var) a `call` takes. `var` is not an option you pass. The task and its owner never share one `var` table, and the task's changes to `var` are discarded when the task ends:

````markdown
---
name: var-snapshot
description: Shows that each task starts from a copy of var taken at the spawn.
runvil: 0
---

## Main

```lua
var.who = 'a'
local a = tasks.spawn('Caller')
var.who = 'b'
local b = tasks.spawn('Caller')
local results = tasks.join({ a, b })
return results[1].result .. '|' .. results[2].result .. '|' .. var.who
```

### Caller

```lua
local seen = var.who
var.who = 'changed'
return 'saw ' .. seen
```
````

````text
saw a|saw b|b
````

Task `a` starts with the `var` values as they stood when it was spawned, so it reads `'a'` even though `Main` set `var.who = 'b'` before spawning `b`. Later changes the owner makes to its `var` never reach a task, and each task's `var.who = 'changed'` stays inside that task, so `Main` still reads `b`. In the same way, a section that sets `var.k = 1` before the spawn sees its task log `k=1`.

### The model a task starts on

`tasks.spawn` has no model argument or option of its own. A task starts on a copy of its spawner's current model, taken when `tasks.spawn` runs and not when the scheduler admits the task (see [Children and fanout arms](10-models.md#children-and-fanout-arms)). A task that is still queued is not changed by a later `models.use` in its spawner:

````markdown
---
name: task-model
description: Starts two tasks on strong, then switches the spawner to fast.
runvil: 0
---

## Main

```lua
models.use('strong')
local a = tasks.spawn('../Child')
local b = tasks.spawn('../Child')
models.use('fast')
local results = tasks.join({ a, b })
return results[1].result .. '|' .. results[2].result .. '|' .. models.current().label
```

## Child

```lua
return models.current().label
```
````

````text
strong|strong|fast
````

Both tasks are spawned while `Main` is on `strong`, and neither runs until `Main` parks on `tasks.join`, which comes after the switch to `fast`. Each task still reads `strong`, because its model was copied at the spawn, and `Main` reads `fast` afterward. The `label` field is the role name, so the result holds on any Host that supplies a model. A `models.use` inside a task stays in that task, so it never reaches the spawner or a sibling. To run a task on another model, switch the spawner with `models.use` before the spawn, or have the task call `models.use` itself.

A task that is not spawned from the main walk follows the same rule. A task spawned inside a `call` chain copies that chain's current model, and the tasks a task spawns copy the model of the task that spawns them. A [fanout](14-fanout.md#running-arms-on-a-model) arm is a task, so it starts the same way, and `tasks.fanout(model, worker, collection)` is the way to give every arm a model of its own.

### The freeform pattern

In the freeform pattern you spawn a set of worker tasks, join them all, and merge their work by index. You prepare each worker task's input before you spawn it, and you read each worker task's output after you join it:

````markdown
---
name: freeform
description: Spawns one worker task per topic and merges their files by index.
runvil: 0
---

## Parent

```lua
local topics = { 'alpha', 'beta' }
local set = {}
for i, topic in ipairs(topics) do
  set[i] = tasks.spawn('Worker', { item = topic, index = i })
end
tasks.join(set)
local parts = {}
for i = 1, #set do
  parts[i] = store.read('research/' .. i .. '.md')
end
return parts[1] .. '\n' .. parts[2]
```

### Worker

```lua
store.write('research/' .. sys.index .. '.md', item)
return 'saved'
```
````

````text
alpha
beta
````

Each worker task writes its own partition, a store path named from `sys.index`, and `Parent` merges the partitions by index after the join with `parts[i] = store.read('research/' .. i .. '.md')`. Because the merge follows the index and not the order the worker tasks finish, the result is the same on every run. Spawning each worker task with `tasks.spawn` and joining the whole set with `tasks.join` gives the same merged result as `tasks.fanout` over the same members, in every interleaving. [Sharing files across sections, calls, and tasks](09-the-store.md#sharing-files-across-sections-calls-and-tasks) covers how tasks share the store.

## Waiting with `tasks.join_any` and `tasks.join`

After this section you can wait for your tasks with `tasks.join_any` and `tasks.join` and read the result each wait hands back.

````markdown
---
name: two-tasks
description: Starts two tasks and joins them.
runvil: 0
---

## Main

```lua
local a = tasks.spawn('../Alpha')
local b = tasks.spawn('../Beta')
local results = tasks.join({ a, b })
return results[1].result .. ' + ' .. results[2].result
```

## Alpha

```lua
return 'alpha text'
```

## Beta

```lua
return 'beta text'
```
````

````text
alpha text + beta text
````

`tasks.join` waits for both tasks and returns a results sequence with one entry per task, in the order the set lists them, so `results[1]` belongs to `Alpha` and `results[2]` to `Beta`. Each entry's `result` field holds the text that task's section returned.

`tasks.join_any` takes one task at a time instead. Replace the block in `Main` with this one, and the prompt gives the same run result:

````lua
local a = tasks.spawn('../Alpha')
local b = tasks.spawn('../Beta')
local first, ok, result = tasks.join_any({ a, b })
assert(ok, tostring(result))
local other = a
if first.task == a.task then
  other = b
end
local _, ok2, result2 = tasks.join_any({ other })
assert(ok2, tostring(result2))
if first.task == a.task then
  return result .. ' + ' .. result2
end
return result2 .. ' + ' .. result
````

````text
alpha text + beta text
````

The first `tasks.join_any` parks `Main` until one of the two tasks ends and returns three values: that task's handle, an `ok` flag saying whether it succeeded, and its result. Which task ends first is not fixed, so the block branches on the returned handle instead of assuming an order. The task the first wait did not deliver keeps running, so the block waits on it with a second `tasks.join_any` over a set holding only that task.

### Waits and task results

A wait parks its owner until tasks end and hands the owner their results. A task's result is its final text when it returns, or its failure when it fails, and a finished task's result is held until its owner takes it with a wait. Waiting on a finished task delivers its result exactly once: the wait takes the result, and the task then counts as delivered. A task can also hand back a result and stay open, with [`tasks.reply`](#answering-with-tasksreply), and a wait takes each reply the same way, exactly once.

When the delivered task succeeded, `ok` is `true` and the result is the task's final text, the value its section's Lua returned. When it failed, `ok` is `false` and the result is the task's [error value](05-lua-environment.md#error-values), handed back without raising, so the wait itself succeeds and your code decides what to do with the failure. A common guard is `assert(ok, tostring(result))`, which puts a failed task's error value in the error message.

A task that raises an error fails as a task, not as a run, so its owner keeps running and the run still ends with the owner's result. An error left uncaught inside a task, whatever its kind, becomes that task's failure, and the owner's wait receives it as `ok = false` with the same error value, its `kind` included:

````markdown
---
name: checked-join
description: Joins two tasks when one of them fails.
runvil: 0
---

## Main

```lua
local a = tasks.spawn('../Alpha')
local b = tasks.spawn('../Beta')
local results = tasks.join({ a, b })
local lines = {}
for i, entry in ipairs(results) do
  if entry.ok then
    lines[i] = entry.task .. ' ok: ' .. entry.result
  else
    lines[i] = entry.task .. ' failed: ' .. entry.result.kind
  end
end
return lines[1] .. '\n' .. lines[2]
```

## Alpha

```lua
return 'alpha text'
```

## Beta

```lua
error('beta boom')
```
````

````text
0.0 ok: alpha text
0.1 failed: lua
````

`Beta`'s uncaught `error('beta boom')` fails that task, and the run still ends with `Main`'s result. `tasks.join` never raises because a member failed: it collects an entry for every member, the failed member's entry holds `ok == false` and the error value in `result`, and your code decides what to do with each outcome. The error value has the same shape as an error the Engine raises, with `kind`, `code`, `message`, and the kind's own fields, so `entry.result.kind` reads `lua` here and `tostring(entry.result)` reads `checked-join.md:33: beta boom. The Lua code failed at runtime. Fix the code at this location.`, keeping the line in `Beta` where the task failed (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). A member that ran out of its log event quota arrives the same way, with `err.kind` reading `lua` and `tostring(err)` reading `{file}:{line}: lua log event quota exceeded. The section logged more than its budget allows. Log less.`, where `{line}` is the member's `log` call that ran out.

### Wait sets

Both waits take a set, a Lua sequence of task handles or task id strings, and one set can mix both forms. `tasks.spawn` returns a handle you can collect straight into such a list, as the freeform pattern does with `set[i] = tasks.spawn(...)`. Only the sequence part of a set counts, because the waits read members with `ipairs`, so entries after a `nil` hole or under non-integer keys are ignored.

The waits check their set before any wait starts, so a bad set raises at the call, and `pcall` catches it. Each message names the call that raised it, `tasks.join_any` or `tasks.join`, in place of `{call}`:

| Set | Error kind | Message |
|---|---|---|
| A value that is not a table | `lua` | `{call} expects a set of tasks, got {type}` |
| An empty table | `lua` | `{call} requires at least one task` |
| A string member that is not a valid task id | `lua` | `` `{text}` is not a task id such as `0.1`: {error} `` |

The first two messages are the error value's whole message. The third is a refusal the Engine makes, so the error value's message puts the wait's `{file}:{line}: ` in front of it and the Lua runtime prose after it (see [Valid task ids and id errors](#valid-task-ids-and-id-errors)). Every member must be a task handle or a valid task id string, and one string that is not a valid id fails the whole call.

### How `tasks.join_any` picks a member

`tasks.join_any(set, opts?)` waits on a set of tasks the caller owns. It returns at once, without parking, when a member has already ended or holds a reply no wait has taken, and otherwise it parks the caller until a member ends or replies and delivers the first one to do so. When several members already have a result to deliver, it returns the first of them in the order you listed them, not the order they finished. Before returning anything, it checks every member, so one member the caller does not own, or one already delivered, fails the whole wait even when another member has already ended.

The handle `tasks.join_any` returns is a new table, so you learn which member it delivered by comparing its `task` field with the `task` field of a handle you kept, as in `first.task == a.task`, rather than comparing the tables with `==`. Which member it returns when several are pending is not fixed, so a prompt branches on the returned handle instead of assuming an order.

A set can have a single member, such as `tasks.join_any({ b })`, which waits for that specific task, so you can join your tasks one at a time in an order you choose. When a wait returns one member, the other members keep running and stay available to later waits, and each of them still needs another wait or a cancel before the owner ends.

### Delivering a result once with `task_consumed`

Each task's result goes to one wait only, so waiting on a task whose result an earlier wait already delivered raises an error of kind `task_consumed` naming that task:

````markdown
---
name: wait-once
description: Shows that a task's result goes to one wait only.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local _, ok, result = tasks.join_any({ t })
local ok2, err = pcall(tasks.join_any, { t })
return result .. '|' .. tostring(ok2) .. '|' .. err.kind .. '|' .. err.task
```

## Child

```lua
return 'once'
```
````

````text
once|false|task_consumed|0.0
````

The first wait takes `once`, and the second one raises. The `task_consumed` error value's `task` field holds the task id, and its message points at the second wait and names the task:

````text
wait-once.md:12: task `0.0` was already delivered: a task's result is taken by one wait. The task's result was already delivered. Wait on each task only once.
````

The refusal in the middle always reads `` task `{task}` was already delivered: a task's result is taken by one wait ``. A `task_consumed` left uncaught ends the run with that error, under its own [error code](16-limits-and-errors.md#run-error-codes) `task_consumed`, and a message that is that refusal followed by ` (at {file}:{line})`.

An idle task gives the same error to a wait that comes after a wait took its latest reply, until its owner sends it a message (see [Each reply goes to one wait](#each-reply-goes-to-one-wait)).

### Waiting in finish order

A loop that waits on several tasks one at a time and drops each one from its set once delivered handles the tasks in finish order and leaves nothing live. In a prompt with sections `A`, `B`, and `C`, this block returns the ids in the order the tasks finish, such as `0.1 0.0 0.2`:

````lua
local left = { tasks.spawn('../A'), tasks.spawn('../B'), tasks.spawn('../C') }
local order = ''
while #left > 0 do
  local done = tasks.join_any(left)
  if order ~= '' then
    order = order .. ' '
  end
  order = order .. done.task
  for i, t in ipairs(left) do
    if t.task == done.task then
      table.remove(left, i)
      break
    end
  end
end
return order
````

### How `tasks.join` fills its results

`tasks.join(set, opts?)` takes a list of task handles and waits until every listed task finishes or replies, and the section continues once all of them have. It returns two values: a results sequence, then a `timed_out` boolean, which [Time limits on waits](#time-limits-on-waits) covers. Each results entry is a table with the fields `task`, `ok`, and `result`, placed at the same position as its member in the set:

- When the member succeeded, its entry has `ok == true`, and `result` holds the value its section returned, or the reply it handed back with `tasks.reply`.
- When the member raised, its entry has `ok == false`, and `result` holds the error value, whose `kind` and `tostring` describe the failure.

Entries follow the order the set lists the handles, not the order the tasks finish, so `results[1]` always belongs to the first handle: `tasks.join({ b, a })` puts `b`'s entry first. Each entry, `{ task, ok, result }`, is itself a task handle whose `task` field holds the id string, so you can pass it to any other `tasks` function. A join that did not time out returns results with no holes, so `#results` equals the length of the set and `ipairs` walks every position. [Collecting the members a timeout left running](#collecting-the-members-a-timeout-left-running) covers the holes a timeout leaves.

A task named more than once in a set is waited on once and fills every position it was named at, with no `task_consumed` error. Each position is still its own entry table: for `tasks.join({ a, b, a })`, `#results` is 3, and `results[1]` and `results[3]` are separate tables for the same task.

You can also call `tasks.join` purely to wait, as a bare statement such as `tasks.join({ a, b })`, discarding its return values and the tasks' results.

`tasks.join` is built in Lua on top of `tasks.join_any`, the one built-in wait, so both apply the same checks and give the same messages, each naming its own call. When any of the waits `tasks.join` makes underneath is refused, `tasks.join` raises that error and returns none of the entries it already collected.

### Store writes after a wait

Delivery joins the task, so everything the task did, its store writes included, is visible to the owner from the very next operation. `tasks.join_any` joins the task it returns, so you can read that task's store writes right away, and after `tasks.join` returns, every delivered member's store writes are visible to you.

Read a task's files only after a wait delivers it. Reading the store files of a task that no wait has delivered yet, such as a member `tasks.join_any` did not return or a member a timed-out wait left running, is a claims conflict: it ends the run on the spot with error code `determinism`, and no `pcall` catches it, even when the task has already written the file (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)).

A task cannot join its sibling, so you build a pipeline by having the owner spawn a stage, join it, and then spawn the next stage, which reads the earlier stage's store writes:

````markdown
---
name: pipeline
description: Runs two stages in order, the second reading the first one's file.
runvil: 0
---

## Main

```lua
local g = tasks.spawn('Gather')
tasks.join({ g })
local s = tasks.spawn('Summarize')
local results = tasks.join({ s })
return results[1].result
```

### Gather

```lua
store.write('gathered.md', 'gathered content')
return 'gathered'
```

### Summarize

```lua
return 'summary of: ' .. store.read('gathered.md')
```
````

````text
summary of: gathered content
````

The join on `Gather` makes its write to `gathered.md` part of `Main`'s view of the store before `Summarize` is spawned, so `Summarize` reads it, and the run gives the same result in every interleaving.

## Canceling and listing tasks

After this section you can stop a task with `tasks.cancel`, list the tasks you still have running with `tasks.pending`, and read what a wait gives back for a canceled task.

````markdown
---
name: cancel-one
description: Cancels a task and then waits on it.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
tasks.cancel(t)
local _, ok, err = tasks.join_any({ t })
return tostring(ok) .. ' ' .. err.kind .. ' ' .. tostring(#tasks.pending())
```

## Child

```lua
return 'never'
```
````

````text
false canceled 0
````

`Main` has not suspended yet when it cancels, so the task is still queued, and `tasks.cancel(t)` ends it before it ever runs; `Child` never reaches its `return 'never'`. The wait that follows does not raise: it delivers the task with `ok` set to `false` and an error value of kind `canceled`. After that, `tasks.pending()` returns an empty list, because the one task `Main` spawned is canceled and delivered.

### What `tasks.cancel` does

`tasks.cancel(task)` ends a task the caller owns, works only on tasks the caller owns, and returns nothing, so its result reads as `nil`. You pass it a single task handle or the task's bare id string. A cancel moves a live task to state `canceled` with `ok` false and stops its chain, together with every task that chain owns, and those owned tasks end as [abandoned](#abandoned-tasks).

After a cancel, the task's status and listing change at once:

- `tasks.status(t)` reads `state = 'canceled'` and `ok = false`.
- `tasks.ready(t)` is `true`.
- `tasks.pending()` no longer lists the task.
- The task's section never reaches its `return`.

`tasks.cancel` is idempotent. Canceling a task that has already ended succeeds, leaves the task as it is, reports nothing, and raises nothing, so calling it more than once is safe. It is answered at once, without a round trip to the Host, so the section continues straight to its next statement, the same as `tasks.spawn`, `tasks.ready`, `tasks.status`, `tasks.pending`, `tasks.note`, and `tasks.send`. You can cancel several tasks one after another in any order, such as `tasks.cancel(b)` before `tasks.cancel(a)`.

A cancel stops a task wherever it is:

- A task still queued for a concurrency slot never starts its section.
- A task parked waiting on an effect, such as a `store.write`, never resumes past that call.
- A task waiting on a model reply ends, and so does a task parked on a batch of in-flight tool calls inside `models.loop`.
- A task waiting on its inbox ends, whether it waits in `tasks.receive()` or is [idle](#idle-tasks) after `tasks.reply`. An idle task has not finished, so a reply no wait has taken is dropped with it, and the messages left in a canceled task's inbox are dropped too.

A cancel withdraws every effect the task still has unanswered: a model round, a render, a store call, a timed wait's timer, and every tool call, even a call to a tool marked to survive a Host stop. The Harness stops the work already running and drops the rest unstarted, so the canceled task never sees an answer, and the cancel does not fail the run. A store call the task made in the cancel's own step is dropped unperformed, and a write the Harness already performed stays.

A canceled task does not fail its owner, so the owner's return value becomes the run result. A canceled task is not a leaked task either, so canceling your own spawned task with `tasks.cancel(t)` before returning lets the section complete cleanly without [`tasks_live`](#when-a-chain-ends-with-live-tasks).

`tasks.cancel` raises in two cases, both at the call site, where `pcall` catches them:

| Argument | Error kind | Message |
|---|---|---|
| Not a task handle or a string, such as `nil` or a number | `lua` | `tasks.cancel expects a Task handle or task id, got {type}` |
| A string that is not a valid task id | `lua` | `` `{text}` is not a task id such as `0.1`: {error} `` |

The first message is the error value's whole message, while the second is a refusal the Engine makes, which the error value's message holds between the call's location and the Lua runtime prose. A task handle here is any table with a string `task` field. Canceling a task the chain does not own, or an id that names no task, makes the Engine refuse, and `tasks.cancel` raises a [`task_not_owned`](#only-the-owner-may-touch-a-task) error value with the refusal `` task `{task}` is not a task this chain owns `` in its message.

### What a wait gives back for a canceled task

A wait on a canceled task does not raise. It delivers the task with `ok = false` and an error value of kind `canceled` whose `task` field names the task, with this message, which points at the wait's line:

````text
{file}:{line}: The task was canceled before it finished, so it has no result. Handle the cancellation where the task is waited on.
````

The message does not name the task, which `err.task` holds. The run's message does when a re-raised one goes uncaught: `` task `{task}` was canceled (at {file}:{line}) ``.

In the opening prompt, `err.kind` is `canceled` and `err.task` equals `t.task`. The value holds the task id as a string in `err.task` and leaves `err.reason` `nil`. Being canceled is the only ending other than a finished result that a wait delivers; an abandoned task is never delivered, because its owner has ended.

A wait never raises a canceled task's error value. It hands the error value back unraised, and only re-raising it, for example with `error(err)`, makes it a failure. Re-raising it and leaving it uncaught ends the section with that `canceled` error, its `task` field and its code `task_canceled` kept, as a failure rather than the `canceled` code a run cancel gives.

A canceled task holds no result to consume and is never consumed, so waiting on it again returns the same `canceled` error value instead of raising `task_consumed`. A canceled task's completed store writes are kept, and a wait that delivers it makes them readable to the owner from its next operation, however far the task got. Canceling a task that has already finished, but whose result no wait has taken yet, leaves that result in place for a later wait.

A canceled task's error value shares kind `canceled` with a canceled run or a stopped call, and only the canceled task's value carries a `task` field and the `code` `task_canceled` rather than `canceled`, which tells them apart. The other `canceled` values come from a run being canceled or a call being stopped:

- A canceled run, such as a Host cancel that reaches running Lua or a waiting call, ends that Lua with an error value of kind `canceled`.
- A model round, tool call, or other call stopped by cancellation raises an error value of kind `canceled` with the message `{file}:{line}: The operation was canceled.`, pointing at the stopped call.
- When the Host drops a model round, a [render](01-what-a-prompt-is.md#the-five-kinds-of-harness-work), a tool call, or a timer instead of answering it, the Lua waiting on it resumes with the same `canceled` error a run cancel raises. `pcall` catches it and the run continues, and an uncaught one ends the run with code `canceled`.
- When the Host drops a tool call the model issued, `models.loop` raises an error of kind `canceled` at the loop call site, and the section can catch it with `pcall` and finish with its own return value.

### Listing live tasks with `tasks.pending`

`tasks.pending()` returns the caller's own live tasks in spawn order, as a 1-based sequence of task handles, each shaped `{ task = id }`. When nothing is live, it returns an empty sequence, never `nil`, so `#` and `ipairs` need no `nil` check.

Only live tasks are listed, running or [idle](#idle-tasks), so ended tasks drop out whether or not a wait took their results. After you spawn `a`, `b`, and `c` and wait on `b`, `tasks.pending()` holds two handles, for `a` and then for `c`. A task that a [timed-out wait](#time-limits-on-waits) left running is still listed, and so is an idle task after a wait took its reply, since it has not ended. A task that has ended drops out, so `tasks.pending()` returns an empty list once every task the caller spawned has ended. The list holds only tasks the calling chain spawned directly, not tasks those tasks spawned.

Each handle `tasks.pending` returns carries the task id in its `task` field and works with every `tasks` function, so it passes straight to `tasks.cancel`.

`tasks.pending` takes no arguments. Any argument other than `nil` raises an error of kind `lua` at the call site, where `pcall` catches it, and the message names the argument's Lua type, such as `got table`:

````text
tasks.pending takes no arguments, got {type}
````

### Racing two approaches

You can race two approaches by spawning both, taking the first to finish with `tasks.join_any`, and canceling every task left in `tasks.pending()`:

````markdown
---
name: race
description: Races two sections and cancels whichever loses.
runvil: 0
---

## Main

```lua
local quick = tasks.spawn('../Quick')
local careful = tasks.spawn('../Careful')
local first, ok, result = tasks.join_any({ quick, careful })
assert(ok, tostring(result))
for _, t in ipairs(tasks.pending()) do
  tasks.cancel(t)
end
return result
```

## Quick

```lua
return 'quick answer'
```

## Careful

```lua
return 'careful answer'
```
````

The run result is the text of whichever task ended first, `quick answer` or `careful answer`, since which one wins is not fixed. Either way the section ends with nothing left running: a task still running is canceled, and a task that already ended has dropped out of `tasks.pending()`.

## Messages between tasks

After this section you can send a task you started a message with `tasks.send`, read a task's messages with `tasks.receive()`, and keep a task open for follow-ups with `tasks.reply`.

````markdown
---
name: draft-loop
description: Asks a task for a draft and sends it two follow-ups.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Worker')
local _, _, first = tasks.join_any({ t })
tasks.send(t, 'shorten it')
local _, _, second = tasks.join_any({ t })
tasks.send(t, 'done')
local _, _, last = tasks.join_any({ t })
return first .. '|' .. second .. '|' .. last
```

## Worker

```lua
local text = tasks.reply('long draft')
text = tasks.reply('short draft after ' .. text)
return 'returned after ' .. text
```
````

````text
long draft|short draft after shorten it|returned after done
````

`Worker` hands back `long draft` with `tasks.reply` instead of `return`, so the task stays open: the first `tasks.join_any` takes the reply, and the task waits for its owner's next message. `tasks.send(t, 'shorten it')` puts that text in the task's inbox, the waiting `tasks.reply` returns it, and the task runs again and replies a second time, which the second wait takes. The message `done` resumes the second reply, the task returns, and the last wait takes its final text, as it would from any task that returns.

### Each task's inbox

Every task has an inbox, a queue of text messages, oldest first, like the main chain's inbox that holds the operator's messages (see [Messages that arrive first](05-lua-environment.md#messages-that-arrive-first)). Only the task's owner fills it, with `tasks.send`, and only the task reads it, with `tasks.receive()` or `tasks.reply`, and a running `models.loop` also takes the task's queued messages. A section the task runs with `call` has no inbox of its own and reads the task's (see [How a called chain runs](08-jump-and-call.md#how-a-called-chain-runs)). No chain can send to the main chain, whose inbox only the Host fills.

A message stays in the inbox until the task reads it, and a stop leaves it there (see [What a stop leaves running](16-limits-and-errors.md#what-a-stop-leaves-running)). A task that ends with messages still in its inbox drops them, and its owner sees how the task ended through a wait or `tasks.status`.

### Sending with `tasks.send`

`tasks.send(task, text)` puts `text` last in the inbox of a task the caller owns and returns `true`. It takes a task handle or a task id string, like every `tasks` function that names a task, and it answers at once without parking the caller. The task reads the text exactly as sent. A message reaches only the task it names, and what it does there depends on what the task is doing:

- A task waiting in `tasks.receive()` or `tasks.reply` resumes, and that call returns the text.
- A task inside `models.loop` whose model request has no answer yet has that request withdrawn, and the loop sends it again with the message appended (see [Messages that arrive while the loop runs](11-conversations.md#messages-that-arrive-while-the-loop-runs)).
- Otherwise the message waits in the inbox. The task's next `tasks.receive()` or `tasks.reply` returns the oldest message at once. A running `models.loop` appends every queued message before its next model request, so a task whose loop is running tool calls reads the message once they finish, and the loop takes the queue once more before it returns, unless its round cap is spent.

`tasks.send` returns `false` once the task has ended, whether it returned, failed, was canceled, or was closed by the [idle cap](#the-idle-cap), and the text is dropped: it enters no inbox and reports no event. With this block as the Lua block of `Main` in `one-task`, the run returns `false`:

````lua
local t = tasks.spawn('../Child')
tasks.join({ t })
return tostring(tasks.send(t, 'too late'))
````

Only the owner may send to a task, so `tasks.send` raises a [`task_not_owned`](#only-the-owner-may-touch-a-task) error value for a task the caller does not own and for an id that names no task. That covers a task sending to a sibling, a task sending to itself with `tasks.send(sys.taskid, text)`, since no chain owns itself, and the main chain sending to its own `sys.taskid`, which is `0`. A `text` that is not a string raises a `lua` error, `tasks.send text must be a string, got {type}`, before anything is sent (see [Argument errors from the `tasks` functions](#argument-errors-from-the-tasks-functions)).

### Reading a task's inbox with `tasks.receive()`

In a task, `tasks.receive()` reads the task's own inbox, the way it reads the main chain's (see [What `tasks.receive()` takes](05-lua-environment.md#what-tasksreceive-takes)). It returns the oldest queued message at once, or parks the task until its owner sends one, and it returns the text exactly as sent. With these two sections in place of those in `one-task`, the run returns `got: also cover the tests`:

````markdown
## Main

```lua
local t = tasks.spawn('../Child')
tasks.send(t, 'also cover the tests')
local results = tasks.join({ t })
return 'got: ' .. results[1].result
```

## Child

```lua
return tasks.receive()
```
````

The send comes before the task first runs, so the message waits in the inbox and the task's `tasks.receive()` returns it at once. Sent while the task waits, the same message resumes it.

While a task waits on its inbox, `tasks.status(t).blocked` reads `inbox`, and the task gives back its concurrency slot, as a task waiting on other tasks does (see [Waiting gives a slot back](#waiting-gives-a-slot-back)). The wait issues no effect, so only the owner's `tasks.send` ends it, unless a cancel or the owner's end stops the task first.

### Answering with `tasks.reply`

`tasks.reply(value)` hands `value` to the task's owner as the task's latest result, then waits on the task's inbox and returns the owner's next message, as `tasks.receive()` does. The owner takes the reply the way it takes a finished task's result, with `tasks.join_any` or `tasks.join`, so `ok` is `true` and the result is the reply's text. The task keeps its section entry while it waits, its locals and its `messages` list included, so a task that holds a conversation keeps it across follow-ups.

A task can loop on its conversation and answer each message with the model's reply. Spawned with its first question as its `args`, a string the task reads as `args.text`, a task that runs this block holds one conversation across every follow-up:

````lua
messages:user(args.text)
while true do
  models.loop()
  messages:user(tasks.reply(messages[#messages].content))
end
````

Each `models.loop()` runs to the model's reply, and `tasks.reply` hands the reply's text to the owner and returns the owner's next message, which becomes the next `user` record. The task never returns: it ends when its owner cancels it, when its owner ends while it is idle, or when the [idle cap](#the-idle-cap) closes it (see [Idle tasks](#idle-tasks)).

`tasks.reply` turns `value` into text the way a section's `return` does. A string stays as it is and a number or a boolean becomes its text, so `tasks.reply(42)` hands over `42`, and `tasks.reply()` or `tasks.reply(nil)` hands over the empty string, the result of a task whose walk returns nothing. A table raises a `lua` error whose message includes `cannot return a table as a result`, as a section's table return does, and so does any other value with no text form, such as a function.

When a message already waits in the inbox, `tasks.reply` publishes the reply and returns the oldest message at once, so the task runs on without waiting. A reply that no wait has taken yet is replaced by the task's next reply, so a wait always takes the latest one. With these two sections in place of those in `one-task`, the run returns `first|got early`:

````markdown
## Main

```lua
local t = tasks.spawn('../Child')
tasks.send(t, 'early')
local _, _, first = tasks.join_any({ t })
local _, _, second = tasks.join_any({ t })
return first .. '|' .. second
```

## Child

```lua
return 'got ' .. tasks.reply('first')
```
````

The first wait takes the reply `first`. The message `early` was already queued, so the reply returns it at once, the task returns `got early`, and the second wait takes that final text.

`tasks.reply` works only inside a task. On the main chain, which has no owner, it raises a `lua` error with the refusal `tasks.reply has no owner to reply to on the main chain`, and so does a section the main chain runs with `call`. A section a task runs with `call` replies for that task.

### Idle tasks

A task that has replied and waits on its empty inbox is idle. An idle task is still live, so the `tasks` functions read it like a task waiting in `tasks.receive()`:

- `tasks.status(t)` reads `state` `running` and `blocked` `inbox`, with no `ok`, and its `section` names the section the task replied from, the called section when the reply came from a `call`.
- `tasks.ready(t)` is `true` while the task's latest reply waits for a wait to take it, and `false` once a wait has taken it.
- `tasks.pending()` lists the task, since it has not ended.
- The task gives back its concurrency slot while it is idle, so another task can run in it.

An idle task runs again only when its owner sends it a message, which its `tasks.reply` returns. It ends by returning or failing after that, or when its owner cancels it, when its owner ends, or when the [idle cap](#the-idle-cap) closes it. Canceling an idle task ends it `canceled` and drops a reply no wait has taken, so a later wait gives back the `canceled` error value (see [What a wait gives back for a canceled task](#what-a-wait-gives-back-for-a-canceled-task)). An owner may end while it has idle tasks: each is abandoned with it, but an idle task already handed its owner a reply, so unlike a running task it is no leak, and the owner ends without `tasks_live` (see [When a chain ends with live tasks](#when-a-chain-ends-with-live-tasks)).

### Each reply goes to one wait

A wait takes each reply once, as it takes a finished task's result once. A wait on an idle task whose latest reply a wait has already taken, with no message sent to it since, raises [`task_consumed`](#delivering-a-result-once-with-task_consumed), because no other result can come until the owner sends one. With this block as the Lua block of `Main` in `draft-loop`, the run returns `running false task_consumed`:

````lua
local t = tasks.spawn('../Worker')
tasks.join_any({ t })
local state = tasks.status(t).state
local ok, err = pcall(tasks.join_any, { t })
return state .. ' ' .. tostring(ok) .. ' ' .. err.kind
````

Once the owner sends a message, the task runs again, and the next wait waits for its next reply or its end, which is how `draft-loop` waits on one task three times. `tasks.join` takes a reply the same way, so its entry's `result` holds the reply text. A wait that takes a reply joins the task, as every delivery does, so the store writes the task made before it replied are visible to the owner from its next operation (see [Store writes after a wait](#store-writes-after-a-wait)).

### The idle cap

An idle task keeps its Lua VM, so a run holds at most a set number of idle tasks, the idle cap: 8 by default, which the Harness sets as `max_idle_tasks` (see [Run limits and their defaults](16-limits-and-errors.md#run-limits-and-their-defaults)). The cap counts idle tasks across the whole run, whoever owns them. When a reply would put the run past the cap, another idle task closes: of the idle tasks other than the one replying, the one idle longest that owns no live task, or the one idle longest when each of them owns one. Tasks spawned by a section the task is running with `call` count as its own. The closed task's chain ends, and its latest reply stays its result, as if it had returned it.

A task the idle cap closed reads like a task that returned:

- `tasks.status` reads `state` `done` and `ok` `true`, and the run reports a `TaskSucceeded` event for it.
- `tasks.send` to it returns `false`.
- A wait on it takes its latest reply when no wait has, and otherwise raises `task_consumed`.

The cap never abandons the task it closes, because that task's owner is still live and can still take the reply. The closed task's own live tasks end as its return would end them: each is abandoned with reason `owner_returned` (see [Abandon reasons](#abandon-reasons)), and a running one among them is no `tasks_live` failure, because the closed task's result stays its reply. When the closed task owns the task that is replying, the replier is abandoned too, since the cap's bound comes first. Nothing in the run's record says that the cap closed the task.

### Inbox events

Two run events report inboxes, each under the provenance of the chain whose inbox it is, so a task's inbox events report under its own task id (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)):

- `inbox_message` reports each message that enters an inbox, with the message in its `text` field. For a `tasks.send`, the step that applies the message reports it under the receiving task, in the section the task is in. A message sent to a task that has not started yet is reported right after the task's [`TaskStarted`](#the-taskstarted-event) event, in the section the task starts at, one event per message, oldest first, and not at all when the task is canceled or abandoned before it starts. A message the Host sends to the main chain is reported when the Host hands it over, ahead of the step that applies it, and the record holds the events of the answers that landed before the message ahead of its `inbox_message`, so a model reply that came first is recorded first.
- `inbox_waiting` reports a chain parking in `tasks.receive()` or `tasks.reply` with its inbox empty, in the step where it parks. A read that finds a message waiting reports nothing.

A `tasks.send` that returns `false` reports neither event. On the main chain, task `0`, a Host reads `inbox_waiting` as the run waiting for the operator, and `inbox_message` as the operator's message reaching it.

### When nothing can move

A task waiting on its inbox can be woken only by its owner, so a run can reach a point where no chain can move: the owner waits on the task, and the task waits for a message only that owner could send. When no chain can move and the main chain does not wait on its own inbox, the run ends with code `stalled`:

````markdown
---
name: stalled-wait
description: Waits on a task that waits for a message nobody sends.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Worker')
tasks.join({ t })
return 'unreachable'
```

## Worker

```lua
return tasks.receive()
```
````

The run fails with code `stalled` and this message:

````text
the run stalled: no chain can move, and these tasks wait on their inboxes: 0.0
````

The message names the tasks that wait on their inboxes, in id order, and the run error's facts hold their ids. A stall ends the run outside any Lua, so no `pcall` catches it, and it points at no line (see [Run codes](16-limits-and-errors.md#run-codes)). Send a task its message before you wait on it, or do not wait on a task that waits for one. The main chain is the exception: the Host can send it a message, so a main chain that waits in `tasks.receive()` with nothing else to run leaves the run waiting for the operator, not stalled (see [While the section waits](05-lua-environment.md#while-the-section-waits)).

## Ownership, leaked tasks, and abandonment

After this section you can end every chain cleanly, read the `task_not_owned` and `tasks_live` errors, and tell an abandoned task from a canceled one.

````markdown
---
name: clean-end
description: Cancels every live task before the walk returns.
runvil: 0
---

## Main

```lua
tasks.spawn('../Child')
tasks.spawn('../Child')
for _, t in ipairs(tasks.pending()) do
  tasks.cancel(t)
end
return 'done'
```

## Child

```lua
return 'child result'
```
````

````text
done
````

A chain ends cleanly when no task it spawned is still running: each one has ended, been canceled, or been left [idle](#idle-tasks) after a reply. `Main` cancels every handle left in `tasks.pending()` before it returns, so the run ends with text `done` and leaks nothing. Leave the loop out, and the same block ends with two live tasks:

````lua
tasks.spawn('../Child')
tasks.spawn('../Child')
return 'done'
````

Now the run fails with `tasks_live`, and its message names both leaked tasks and ends with the line of the first one's `tasks.spawn`:

````text
chain ended with tasks still live: 0.0, 0.1; wait on or cancel every task a chain spawns before it ends (at {file}:{line})
````

### The owner of a task

Every task runs beside the chain that started it, and that chain is the task's owner, whether the chain was begun by the run, by `call`, by a fanout arm, or by a spawn. The owner keeps running while the task works, and it collects the task's result when it needs it.

Tasks belong to the whole chain that spawned them, the walk, not to the section. A task spawned in one section stays owned and live after the walk falls through or jumps to another section, a later section on the same walk can wait on it, and it is settled only when the chain ends. Fall-through and a [`jump`](08-jump-and-call.md#jumping-to-a-section), such as `jump('../Sibling')`, both move the walk within the same chain, so tasks survive both:

````markdown
---
name: jump-keeps-tasks
description: Waits in a later section on a task spawned before a jump.
runvil: 0
---

## Main

```lua
var.t = tasks.spawn('Worker')
jump('../Next')
```

### Worker

```lua
return 'work'
```

## Next

```lua
local _, ok, result = tasks.join_any({ var.t })
return tostring(ok) .. ' ' .. result
```
````

````text
true work
````

The task spawned in `Main` keeps running, or stays queued, after the walk jumps to `Next`, and `Next` joins it with `ok == true`.

A task spawned in the [preamble](02-file-structure.md#the-preamble) belongs to the main walk in the same way, because the preamble and the sections after it are entries of one chain, the root chain (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). A later section can wait on, inspect, or cancel such a task, and the walk's end is where it leaks.

### Only the owner may touch a task

Only the chain that spawned a task may wait on it, check it, list it, send it a message, or cancel it. The one exception is that code running inside a task may read that task's own status with `tasks.status(sys.taskid)`. Any other task operation, whether a wait, a status read, a send, or a cancel, that names a task the caller does not own raises an error of kind `task_not_owned`, and `err.task` holds the id. That includes a task naming itself with any operation other than a status read:

````markdown
---
name: not-owned
description: Shows that a task cannot cancel itself.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local results = tasks.join({ t })
return results[1].result
```

## Child

```lua
local me = sys.taskid
local ok, err = pcall(tasks.cancel, me)
return tostring(ok) .. ' ' .. err.kind .. ' ' .. err.task
```
````

````text
false task_not_owned 0.0
````

Inside `Child`, `tasks.status(me)` would work, but `pcall(tasks.cancel, me)` is refused, and so are `tasks.ready`, `tasks.send`, and a wait such as `pcall(tasks.join_any, { me })` on its own id. A task that names its owner's task is refused too, such as `'0'` from a task the main walk spawned, since the main walk is task `0` and a caller may name only tasks it owns or its own task. An id that names no task at all is refused the same way, so a caller learns nothing about tasks it never started: `pcall(tasks.cancel, '9.9')` gives `err.kind == 'task_not_owned'` and `err.task == '9.9'`.

The error's message points at the refused call and names the task the caller reached for, such as `0.0`, and the error sets the field `task`. In `not-owned` it reads:

````text
not-owned.md:19: task `0.0` is not a task this chain owns. The task is not one this chain owns. Wait on, inspect, or cancel only the tasks this chain spawned.
````

The refusal in the middle always reads `` task `{task}` is not a task this chain owns ``.

`task_not_owned` is a task error whose value has a `task` string field holding the offending task's id (see [Error values](05-lua-environment.md#error-values) for the groups of error kinds). A wait checks every member it names, so a wait fails outright with this error, rather than delivering a finished member, when the caller does not own any one of them, and `tasks.join` raises the same way because it runs the same wait. An uncaught `task_not_owned` keeps its kind, its code, and its `task` field rather than being flattened to a plain Lua error, so a run it ends fails with error code `task_not_owned`.

### When a chain ends with live tasks

A chain must wait on or cancel every task it spawned before it ends. When a section or other chain ends normally, by a return or by running out of sections, while a task it spawned is still running and not idle, its outcome becomes an error value of kind `tasks_live` in place of the returned value, and the owner chain fails. A leaked task is treated as a bug in your prompt, and only a normal ending with at least one leaked task produces `tasks_live`. Canceled tasks leave nothing live at chain end, so a run that cancels its tasks ends normally, with text such as `done`, and without `tasks_live`.

An [idle](#idle-tasks) task is no leak. It already handed its owner a reply, so an owner may end while it has idle tasks: they are abandoned with it, and its outcome stays its own. An owner that ends with an idle task and a running one fails with `tasks_live` naming only the running one.

The rule applies to every owner, whether the root walk, a `call` chain, or another task. Because tasks belong to the chain and not to the section, a task spawned in a section that falls through is still owned by the walk, and the next section's return leaks it:

````markdown
---
name: leaky-walk
description: Leaves a task live when a later section returns.
runvil: 0
---

## Launch

```lua
tasks.spawn('../Child')
```

## Sibling

```lua
return 'done'
```

## Child

```lua
return 'child result'
```
````

````text
chain ended with tasks still live: 0.0; wait on or cancel every task a chain spawns before it ends (at leaky-walk.md:10)
````

`Launch` spawns `Child` and falls through to `Sibling`, whose `return 'done'` ends the walk with the task still live, so the run fails with `tasks_live` naming `0.0`. The failure points at the `tasks.spawn` of the first task left running, line 10 in `Launch`, not at the `return` that ended the walk. A `jump` settles nothing about owned tasks either, so ending `Launch` with `jump('../Sibling')` gives the same failure. Waiting on the task in `Sibling`, or canceling it there, settles it. In the same way, when the walk returns while a task spawned in the preamble is still live, the run fails with `tasks_live` naming that task, such as `0.0`, and the task is abandoned.

At the main walk, `tasks_live` is the run's failure. Inside a called chain, `tasks_live` is the `call`'s error instead, so the caller can catch it with `pcall`, keep running, and return normally:

````markdown
---
name: leak-check
description: Catches the leak of a called section with pcall.
runvil: 0
---

## Main

```lua
local ok, err = pcall(call, '../Leaky')
return err.kind .. '|' .. err.tasks .. '|' .. tostring(err)
```

## Leaky

```lua
tasks.spawn('../Child')
return 'leaked'
```

## Child

```lua
return 'child result'
```
````

````text
tasks_live|0.0.0|leak-check.md:17: chain ended with tasks still live: 0.0.0; wait on or cancel every task a chain spawns before it ends. The chain ended with tasks still running. Wait on or cancel every task the chain spawns before it ends.
````

`Leaky` spawns a task and returns `'leaked'`, so its caller receives kind `tasks_live` as the call's error, and the run continues with `Main`'s result. The error keeps the line of `Leaky`'s `tasks.spawn`, line 17, rather than the `call` line in `Main`. Tasks spawned inside a `call` chain, including ones it leaks, are abandoned when that chain ends and returns to its caller (see [Catching call failures](08-jump-and-call.md#catching-call-failures)).

A `tasks_live` error value caught in Lua has `kind` equal to `'tasks_live'`. The kind carries one field of its own, `tasks`, and belongs to the cancellation and task lifetimes group of error kinds. `err.tasks` is one string of the leaked task ids in the order they were spawned, joined with `, `, such as `0.0, 0.1`, not a table, so it concatenates directly with `..`. The message names the same ids, and `tostring` on the error value gives that message, where `{line}` is the spawn line of the first task left running:

````text
{file}:{line}: chain ended with tasks still live: {ids}; wait on or cancel every task a chain spawns before it ends. The chain ended with tasks still running. Wait on or cancel every task the chain spawns before it ends.
````

A `tasks_live` left uncaught ends the run with that error, under its own error code `tasks_live`, and a message that ends with ` (at {file}:{line})` for that spawn line, as `leaky-walk` shows.

A chain that fails with its own error while it still owns live tasks keeps that error rather than becoming `tasks_live`, because the leak is the lesser fault. A section that runs `tasks.spawn('../Child')` and then `error('spawner boom')` fails with that error, and its live tasks are still abandoned along with their failed owner, with reason `owner_failed`. An owner that is itself being aborted produces no `tasks_live` for tasks it leaks, because no one receives its outcome, and a run that is ending reports no `tasks_live` for the tasks left live, for the same reason.

Only a task still live at its owner's end is abandoned, and only a running one counts toward `tasks_live`, since an idle task is abandoned without counting. A task that has already finished, whether or not a wait took its result, is left as it is, so a section that spawns and never waits still ends the run normally when its tasks finish first. That makes timing matter: whether a run that leaves spawned tasks running ends `done` or `tasks_live` can depend on the order its pending store writes are answered, since that decides whether the spawning section resumes before or after its task ends. A run that ends normally has no live tasks left, because each chain settled its own.

### Abandoned tasks

A task ends with its owner chain. When a task's owner ends while the task is still live, running or idle, the Engine ends the task, which then ends in state `abandoned` with `ok` false, so a task never keeps running after its owner ends. Every task the ending owner still has live is abandoned and its chain stopped from outside, along with every task that chain owns in turn, so an owner's end reaches every live chain nested below it. In the clean-end prompt without its cancel loop, both leaked tasks, `0.0` and `0.1`, end `abandoned` when `Main` returns, not `done` or `canceled`.

`abandoned` is a terminal state of its own, distinct from `canceled`, because losing an owner and being stopped on purpose are different facts for a log and a UI. Canceled means the owner stopped the task on purpose, and abandoned means the owner ended while the task was live. The status `state` and the task's trace event both keep them apart.

An abandoned task never reaches a wait, because only its owner may wait on it and its owner has already ended. It ends with `state` `abandoned` and `ok` false, and it is never also recorded as succeeded or failed. A task that had started reports exactly one abandoned terminal event, with its reason (see [Task events and task errors](#task-events-and-task-errors)), and a task abandoned while it is still queued for admission never runs its section's code and reports no start and no terminal event, though it still ends `abandoned`. One owner's live tasks are abandoned, and report their abandonment, in the order they were spawned.

Abandonment reaches nested work:

- Abandoning a task aborts the work behind it together with everything that work owns, so tasks nested under an abandoned task end too.
- Tasks spawned inside another task, including inside a fanout arm, are abandoned when that task or arm ends.
- Aborting a chain abandons every task that chain owns, so a fanout arm's spawned task `0.0.0` ends `abandoned` when the arm is aborted.
- When a fanout arm is canceled because a sibling arm failed (see [When an arm fails](14-fanout.md#when-an-arm-fails)), a task that arm spawned is abandoned, and a task that had not started yet never runs its section's code.
- A task owned by a canceled chain ends `abandoned`, not `canceled`, while the owner itself is the one reported as canceled: a canceled arm `0.0` is reported canceled once, and its spawned task `0.0.0` ends `abandoned` with no cancel report.

When the run is canceled, every spawned task that is still live, running or idle, is abandoned. When the run itself ends with tasks live, each one is abandoned exactly once, with `run_terminated` for a task the run's end stranded directly and `owner_aborted` for a task nested under one, so no task is recorded twice. Fanout arms stranded by the run's end report no cancel or failure of their own, and the run's end settles each one with exactly one `abandoned` terminal event that names the run's end.

### Abandon reasons

An abandoned task carries the reason it was abandoned, which says how its owner chain ended while the task was still live, and its abandonment event records that reason. An idle task abandoned with its owner gets the same reason a running one would, and a task whose owner the [idle cap](#the-idle-cap) closed gets `owner_returned`, as if that owner had returned, so neither adds a reason. Each reason has a phrase, which is what the `TaskAbandoned` trace line renders for it. The five reasons are written in snake_case:

| Reason | Phrase | When it applies |
|---|---|---|
| `owner_returned` | `the section ended` | The owner ended normally, by a return or by running out of sections, or the idle cap closed it, while the task was still live, without canceling it |
| `owner_failed` | `the owner failed` | The owner failed with its own error while the task was live |
| `tool_loop_exhausted` | `the tool loop was exhausted` | The owner's `models.loop` ran past its round cap while the task was live |
| `owner_aborted` | `the owner was aborted` | The owner was stopped from outside, by fail-fast after a sibling arm failed fatally or because the owner's own owner ended first |
| `run_terminated` | `the run ended` | The run itself ended while the task was live, for example because the Host canceled the run or an answer was fatal |

A tool loop that runs past its [round cap](11-conversations.md#tool-rounds-and-the-round-cap), set by `max_tool_iterations`, counts as a failure of the owner. `tool_loop_exhausted` is kept apart from `owner_failed` so a report can name the round cap as the cause, and a tool-loop overrun is checked before any other owner failure. Tasks abandoned because their owner failed with any other error carry `owner_failed`, decided from the owner's own outcome before any `tasks_live` is substituted. Tasks owned by a task that is itself being abandoned end with `owner_aborted`, and so does the work a canceled fanout arm had started.

## Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`

After this section you can check on a task without waiting for it, read every field of its status, and publish progress notes from inside a task.

````markdown
---
name: task-status
description: Spawns a task, checks on it, and reads its note.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local fresh = tasks.status(t)
local early = tasks.ready(t)
local _, ok, result = tasks.join_any({ t })
local s = tasks.status(t)
return fresh.state .. ' ' .. fresh.blocked .. ' ' .. tostring(early) .. ' | '
  .. s.state .. ' ' .. tostring(s.ok) .. ' ' .. s.note .. ' '
  .. tostring(tasks.ready(t)) .. ' | ' .. result
```

## Child

```lua
tasks.note('working')
return 'child done'
```
````

````text
running queued false | done true working true | child done
````

`Main` spawns `Child` and reads its status straight away. `tasks.status(t)` returns the task's status as a plain Lua table whose fields the Engine fills in, and the table is readable right after `tasks.spawn`, as `local fresh = tasks.status(t)` shows. At that moment the task has not run yet, so `fresh.state` reads `running` and `fresh.blocked` reads `queued`. `tasks.ready(t)` checks without waiting whether the task has ended, and it returns `false` because the task has not started. The wait `tasks.join_any({ t })` then lets the task run (see [Waiting with `tasks.join_any` and `tasks.join`](#waiting-with-tasksjoin_any-and-tasksjoin)). Inside the task, `tasks.note('working')` publishes a progress note, which the owner reads back as the `note` field. After the wait, the status reads `state` `done` and `ok` `true`, and `tasks.ready(t)` returns `true`.

A task's status lasts for the whole run. After a wait has taken a task's result, `tasks.status` still reports how that task ended, including whether it ended well, which is why `Main` can read `s.state` and `s.ok` after `tasks.join_any` returns.

### Reading a status with `tasks.status`

`tasks.status(task)` reads one task's status per call, so you pass it a single task. Like every `tasks` function that takes a task, it accepts the Task handle that `tasks.spawn` returned or the handle's bare id string (see [Task ids and handles](#task-ids-and-handles)).

You can read the status of any task the calling chain owns (see [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment)), and also the status of the task your code runs inside, by passing `sys.taskid` (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)):

````markdown
---
name: self-note
description: A task publishes a note and reads it back from its own status.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local _, ok, result = tasks.join_any({ t })
return result
```

## Child

```lua
tasks.note('hello from ' .. sys.taskid)
return tasks.status(sys.taskid).note
```
````

````text
hello from 0.0
````

Inside `Child`, `sys.taskid` holds the task's own id, `0.0`, so `tasks.status(sys.taskid)` succeeds and reads back the note the task just set. This is the self exception to ownership: a task can read and annotate itself even though it does not own itself. It also works from a chain reached by [`call`](08-jump-and-call.md#calling-a-section) running inside the task, because that chain shares the task's `sys.taskid`.

Those are the only tasks `tasks.status` reads. Calling it on a task the caller neither owns nor runs inside raises an error value of kind `task_not_owned`. On the main walk, `tasks.status(sys.taskid)` raises `task_not_owned` too, because the main walk is task `0` and has no status of its own.

### The status table

You read the fields of the table `tasks.status(handle)` returns like those of any Lua table, as in `local s = tasks.status(t)` followed by `s.state`, `s.blocked`, and `#s.tasks`. The table has these fields:

| Field | What it holds | Present |
|---|---|---|
| `target` | The section the task's chain started at | Always |
| `state` | `running`, `done`, `canceled`, or `abandoned` | Always |
| `ok` | Whether the task ended well | Once the task ends |
| `section` | The section the live task is in now | While the task is live and inside a section |
| `blocked` | What the live task is waiting on | While the task's chain waits |
| `turns` | The model rounds the task has finished | Always |
| `tasks` | Id strings of the live tasks the task started | Always, empty when none are live |
| `depth` | The task chain's `call` depth | Always |
| `note` | The task's latest progress note | Once a note is set |

Optional fields are nil when absent, so you test them for presence. An optional field with no value is missing from the table entirely, so it reads as nil and `pairs` does not visit it, and this applies to `ok`, `section`, `blocked`, and `note`. Because `section`, `blocked`, and `note` are each a string whenever they are present, a plain truth test such as `if s.blocked then` shows whether one is set.

The fields also split by when they mean something. `section`, `blocked`, and `tasks` describe a live task, while `turns`, `depth`, and `note` stay readable after the task ends.

This prompt reads every field twice, once while its task waits on a [model round](10-models.md#running-a-round-with-modelsinfer) and once after the task ends. It uses a wait with a 50 millisecond time limit to get control back while the round is still in flight (see [Time limits on waits](#time-limits-on-waits)):

````markdown
---
name: status-snapshot
description: Reads a task's status during and after its model round.
runvil: 0
---

## Main

```lua
local function show(s)
  return table.concat({
    s.target, s.state, tostring(s.ok), tostring(s.section),
    tostring(s.blocked), s.turns, #s.tasks, s.depth, tostring(s.note),
  }, ' ')
end
local t = tasks.spawn('../Child')
local first = tasks.join_any({ t }, { timeout = 0.05 })
local during = show(tasks.status(t))
if first == nil then
  tasks.join_any({ t })
end
return during .. ' | ' .. show(tasks.status(t))
```

## Child

```lua
tasks.note('working')
return models.infer('slow please')
```
````

When the model takes longer than 50 milliseconds to reply, the run returns:

````text
Child running nil Child chat 0 0 1 working | Child done true nil nil 1 0 1 working
````

While the task is in its first model round, its status holds `target` `Child`, `state` `running`, `ok` nil, `section` `Child`, `blocked` `chat`, `turns` 0, an empty `tasks`, `depth` 1, and `note` `working`. After the task's section returns, the status holds `state` `done`, `ok` `true`, `section` and `blocked` nil, `turns` 1, and the last note, still `working`. The rest of this section takes the fields one group at a time.

### What `target` says

`target` is always present and names what the task runs: the section the task's chain started at, by its section name, so a task started with `tasks.spawn('../Child')` has `target` `Child`.

### The `state` field

`state` is always present as a string, and it holds one of four values:

| `state` | The task |
|---|---|
| `running` | Is still in flight, with its chain or request still running, or is idle after a reply |
| `done` | Has ended with a result, whether or not a wait has delivered it, the idle cap's close included |
| `canceled` | Was canceled by its owner |
| `abandoned` | Was still live when its owner ended |

`state` reads `running` while the task is in flight, and once the task ends it reads `done`, `canceled`, or `abandoned`. A queued task reads `state == 'running'` as well, so being queued is not a separate task state, and the `blocked` field is what shows a task is queued. A task whose result a wait has already taken still reads `state = "done"`, because delivery is the owner's bookkeeping rather than a change in the task's lifecycle. An [idle](#idle-tasks) task reads `running`, because it is still live and its owner can send it a message, and a task the [idle cap](#the-idle-cap) closed reads `done`, as if it had returned its latest reply. [Canceling and listing tasks](#canceling-and-listing-tasks) covers `canceled`, and [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment) covers `abandoned`.

### The `ok` field

`ok` is the task's success flag, taken from its state. It is a boolean once the Engine reports one and nil otherwise:

- While the task runs, `ok` is nil, because a running task has no `ok` value until it ends. An idle task has none either, even after a wait took its reply.
- A task that finishes moves to state `done`, with `ok` `true` when its section returned and `false` when it failed, for example by raising an error such as `error('beta boom')`. A task the idle cap closed reads `ok` `true`.
- A canceled or abandoned task has `ok` `false`, so a canceled task reads `state` `canceled` with `ok` `false`.

A task whose result a wait has taken keeps its flag. Because `ok` can be `false`, compare it with `nil` to tell a running task from a failed one, since a running task has no `ok` at all while a finished one has a boolean:

````lua
local s = tasks.status(t)
if s.ok == nil then
  return 'still running'
elseif s.ok then
  return 'ended well'
else
  return 'ended badly: ' .. s.state
end
````

### What a live task is doing: `section` and `blocked`

`section` names the section the task's chain is in now. It is set while the task is live and inside a section, and nil otherwise: nil before the task is admitted and enters a section, and nil after it ends. A queued task reads `section == nil`, because it has not entered any section yet. The name updates each time the task's chain enters a section. In `status-snapshot`, the task reads `section` nil when fresh, `Child` while it waits on its model round, and nil when done.

`blocked` is present only while the task's chain is waiting on something, and it holds a string naming what that is. It is nil before the task starts, while it is running Lua, and after it ends. While the task waits, it reads one of these values:

| `blocked` | The task is waiting on |
|---|---|
| `chat` | A model round, such as one from `models.infer` |
| `tool_call` | A tool call |
| `store` | A store operation |
| `tasks` | Other tasks, in a wait with or without a time limit |
| `inbox` | A message to its own inbox, in `tasks.receive()` or idle after `tasks.reply` |
| `call` | A chain it started with `call` |
| `queued` | A concurrency slot, before admission or to retake one after a wait |

A task waiting on a model reply, such as a task running `return models.infer('slow please')`, reports `blocked = 'chat'`, and the field reads nil once the task finishes. A wait for a message is never a tool call: a task waiting in `tasks.receive()` reports `blocked = 'inbox'`, and gives back its concurrency slot while it waits. An [idle](#idle-tasks) task reports `blocked = 'inbox'` too, with `section` naming the section it replied from. Only the main chain reads the operator's messages (see [Asking the operator](05-lua-environment.md#asking-the-operator)), and a task's inbox holds only what its owner sends (see [Messages between tasks](#messages-between-tasks)). A task waiting in `tasks.join_any` reports `blocked = 'tasks'` whether the wait is timed or not, so a task inside `tasks.join_any({ g }, { timeout = 30 })` reads `tasks`.

`queued` means the task is waiting for a slot under the concurrency limit while its `state` is still `running` (see [Concurrency, nesting, and scheduling](#concurrency-nesting-and-scheduling)). A spawned task that cannot run yet waits for an admission slot, and while it waits, `tasks.status` reads `state` `running` and `blocked` `queued` until it is admitted and first runs. So a status read right after `tasks.spawn` shows `state = 'running'`, `section = nil`, and `blocked = 'queued'`, as `fresh` did in `task-status`. When the run's concurrency limit is 1, a second task reads `state` `running` and `blocked` `queued` while the first one runs. A task waiting to retake its slot after a wait answers also reads `queued`, the same as a fresh spawn waiting for its first slot. Once a task is admitted, or retakes its slot after a wait, its status no longer reads `queued`.

### Counting with `turns`, `depth`, and `tasks`

`turns` is always present and counts the model rounds the task has finished, reading `0` while its first `models.infer` round is in flight and `1` after that round returns. It keeps its value after the task ends. A task counts its rounds separately from its owner, and `turns` shows that count. The [round cap](11-conversations.md#tool-rounds-and-the-round-cap) applies to each loop call, not to the task.

`depth` is always present and gives the task chain's `call` depth, its nesting level. It reads `1` for a task spawned from the main walk, such as a task spawned from `Main`, and it keeps its value after the task ends.

`tasks` is always present as a sequence of the id strings of the live tasks the task started itself, in spawn order. It is empty when none are live, including for a task that spawned nothing or has ended, so `#s.tasks` reads `0` for a task that spawned nothing. To see it filled, replace `Child` in `task-status` with this version and add a `Leaf` section after it:

````markdown
## Child

```lua
local a = tasks.spawn('../Leaf')
local b = tasks.spawn('../Leaf')
local mine = tasks.status(sys.taskid).tasks
local listed = tostring(mine[1]) .. ' ' .. tostring(mine[2])
tasks.join({ a, b })
tasks.note('working')
return listed .. ' then ' .. #tasks.status(sys.taskid).tasks
```

## Leaf

```lua
return 'leaf'
```
````

````text
running queued false | done true working true | 0.0.0 0.0.1 then 0
````

Entries in the list print with `tostring` as dotted task ids, one segment deeper per generation of spawning, so the first task spawned by task `0.0` reads `0.0.0`, while the main walk's next spawn after `0.0` would be `0.1`. Once both `Leaf` tasks have ended, the list is empty again. The list never includes the timer behind a timed wait, which reports no task events, so a task timed-waiting with `{ timeout = 30 }` on its one task reads `blocked` `tasks` and lists only that task, such as `0.0.0` (see [The timer behind a time limit](#the-timer-behind-a-time-limit)).

A status table's `tasks` holds id strings, while [`tasks.pending()`](#canceling-and-listing-tasks) returns handles, and both work as arguments to every `tasks` function. So you can pass handles from `tasks.pending()` straight to `tasks.cancel(t)`, just as you can pass an id string.

### Publishing progress with `tasks.note`

`tasks.note(text)` publishes the latest progress note on the task the caller runs inside. It takes only the text, with no task argument, and returns no values, and the owner reads the note back through the `note` field of `tasks.status`. For example, a `Child` section can call `tasks.note('worked')` after `models.infer` to report that its round is over.

Each `tasks.note` call replaces the previous note rather than adding to it, because the status shows only the latest one. With this version of `Child` in `task-status`, the run returns the same text as before, because the owner sees only `working`:

````markdown
## Child

```lua
tasks.note('starting')
tasks.note('working')
return 'child done'
```
````

The `note` field holds the task's note once one is set, and it is nil when the task has published none. `tasks.status` shows the note while the task runs and keeps it after the task finishes, so a task that runs `tasks.note('working')` reads `note` `working` both while it waits and when it is done.

A `tasks.note` from a section reached by `call` inside a task records on the task itself, so the called section's note is the task's note. On the main walk, and in a section reached by `call` from it, `tasks.note` succeeds and records nothing.

### Checking without waiting with `tasks.ready`

`tasks.ready(task)` takes exactly one task to check, and that task argument is required. It returns `true` once the task has ended, or while a reply the task handed back with `tasks.reply` waits for a wait to take it, so that a wait would return at once, and `false` otherwise. It never waits. It reports `true` for an owned task once the task has ended in any way, whether or not a wait has taken its result, so a task already collected by `tasks.join_any({ t })` and a task stopped by `tasks.cancel(t)` both read `true`. An [idle](#idle-tasks) task whose reply a wait has taken reads `false`, because it has not ended and holds no result to take. With this block as the body of `Main` in `task-status`, the run returns `true`:

````lua
local t = tasks.spawn('../Child')
tasks.cancel(t)
return tostring(tasks.ready(t))
````

`tasks.ready` checks only tasks the calling chain owns. On a task the chain does not own, or on an unknown id, it raises `task_not_owned`. It has no self exception, so inside a task `tasks.ready(sys.taskid)` raises `task_not_owned` even though `tasks.status(sys.taskid)` works.

`tasks.ready` answers right away and never parks the caller, so a loop that does nothing but call `tasks.ready` never lets the task run. Check between calls that wait, such as a store call or a model round, or use a wait instead.

### Status, ready, and note errors

Bad arguments to `tasks.status`, `tasks.ready`, and `tasks.note` raise at the call site, where `pcall` catches them (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). Replace `Main` in `task-status` with:

````markdown
## Main

```lua
local _, e1 = pcall(tasks.status, 'not-an-id')
local _, e2 = pcall(tasks.ready, 42)
local _, e3 = pcall(tasks.note, 7)
local _, e4 = pcall(tasks.status, sys.taskid)
return e1.kind .. ' ' .. e2.kind .. ' ' .. e3.kind .. ' ' .. e4.kind
```
````

````text
lua lua lua task_not_owned
````

Each call fails for a different reason:

| Call | Kind | Message |
|---|---|---|
| `tasks.status` or `tasks.ready` given a value that is neither a Task handle nor a string | `lua` | `{call} expects a Task handle or task id, got {type}`, such as `tasks.ready expects a Task handle or task id, got integer` |
| `tasks.status` or `tasks.ready` given a string that is not a task id | `lua` | `` `{text}` is not a task id such as `0.1`: {error} `` |
| `tasks.status` or `tasks.ready` on a task the caller may not read | `task_not_owned` | See [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment) |
| `tasks.note` given text that is not a string | `lua` | `tasks.note text must be a string, got {type}` |
| `tasks.note` given text that is not valid UTF-8 | `lua` | `text must be a valid UTF-8 string` |

The message for a string that is not a task id names the bad value, so `pcall(tasks.status, 'not-an-id')` gives a message whose refusal, after the call's location, starts with `` `not-an-id` is not a task id such as `0.1`: ``. [Valid task ids and id errors](#valid-task-ids-and-id-errors) gives the rest of that message.

## Time limits on waits

After this section you can put a time limit on `tasks.join_any` and `tasks.join`, tell a wait that ran out of time from one that finished, and collect the tasks a timeout left running.

````markdown
---
name: timed-wait
description: Waits on a task with a time limit.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local first, ok, result = tasks.join_any({ t }, { timeout = 30 })
return tostring(first.task == t.task) .. ' ' .. tostring(ok) .. ' '
  .. result .. ' ' .. #tasks.pending()
```

## Child

```lua
return 'quick'
```
````

````text
true true quick 0
````

`tasks.join_any` and `tasks.join` take an optional second argument, an `opts` table `{ timeout = seconds }`, that limits how long the wait lasts. Here the task ends well inside its 30 seconds, so the timed `tasks.join_any` returns the task's handle, `ok`, and result exactly as an untimed wait would. The wait then drops its internal timer, so nothing is left in `tasks.pending()`, and the run does not wait out the rest of the 30 seconds.

### The `timeout` value

Both waits treat the `opts` table as optional and read its `timeout` field as a limit in seconds. When either the table or the field is left out, the wait has no time limit. `timeout` is the only field a wait reads from `opts`, so other keys are ignored, and a misspelled `timeout` key leaves the wait with no time limit.

`opts.timeout` is a number of seconds after which the wait stops waiting. It accepts an integer or a float, and it can be whole, fractional, or zero:

| `timeout` | Limit |
|---|---|
| `30` | 30 seconds |
| `5` | 5 seconds |
| `1.5` | 1500 milliseconds |
| `0.05` | 50 milliseconds |
| `0` | A valid zero-length limit |

A wait timeout only ends the wait and never cancels a member. The value must be zero or more, finite, and small enough for a duration to hold, and a seconds value too large for a duration is refused (see [Wait timeout errors](#wait-timeout-errors)). `timeout` has no value meaning forever, so a wait with no time limit leaves `timeout` out.

### When `tasks.join_any` runs out of time

A timed `tasks.join_any` returns a single `nil` when no member ends in time, so `local first, ok, result = ...` reads three nils. The members keep running, so a later wait can join them:

````markdown
---
name: still-working
description: Gives a task five seconds, then waits for it with no limit.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Child')
local first, ok, result = tasks.join_any({ t }, { timeout = 5 })
if first == nil then
  log('still working after 5 seconds')
  first, ok, result = tasks.join_any({ t })
end
return result
```

## Child

```lua
return models.infer('Write a long poem about the sea.')
```
````

The run result is the model's poem either way. If the model has not replied within 5 seconds, the timed wait returns `nil`, the block logs `still working after 5 seconds`, and the second `tasks.join_any({ t })`, called again on the same handle without a timeout, collects the task and delivers its handle, `ok` flag, and result. The same holds with fresh names, as in `local second, ok2, result2 = tasks.join_any({ t })`.

A member of a timed-out `tasks.join_any` keeps running, and `tasks.status(t).state` reads `running` right after the wait returns. The member also stays in `tasks.pending()`, which lists that member and never the wait's timer. A timed-out wait leaves nothing behind on its own, so once the later wait collects the member, the run ends normally with no `tasks_live` error.

### When `tasks.join` runs out of time

`tasks.join` takes the same optional second argument, and its `timeout` limits how long it waits for all the members together, as in `local results, timed_out = tasks.join({ q, s }, { timeout = 0.05 })`. The second return value is `true` when the timeout expired first and `false` otherwise:

````markdown
---
name: timed-join
description: Joins two tasks with a time limit, then collects the late one.
runvil: 0
---

## Main

```lua
local q = tasks.spawn('../Quick')
local s = tasks.spawn('../Slow')
local results, timed_out = tasks.join({ q, s }, { timeout = 0.05 })
local late = results[2] == nil
local report = tostring(timed_out) .. ' ' .. results[1].result .. ' ' .. tostring(late)
if late then
  report = report .. ' ' .. tasks.status(s).state
  local rest, timed_out2 = tasks.join({ s })
  report = report .. ' ' .. tostring(timed_out2)
end
return report
```

## Quick

```lua
return 'quick'
```

## Slow

```lua
return models.infer('slow please')
```
````

When the model takes longer than 50 milliseconds, the run returns:

````text
true quick true running false
````

A timed `tasks.join` that runs out of time returns normally instead of raising an error, and the section continues with the next statement. `timed_out` is `true`, finished members keep their entries, and the unfinished members' entries are `nil` while those tasks keep running, with `tasks.status(s).state` reading `running`. Calling `tasks.join` again on just the handles the timed-out wait left behind collects those members, as `local rest, timed_out2 = tasks.join({ s })` does here, and that call reports `timed_out2` as `false`. If the model replies within the 50 milliseconds instead, both members finish first and the run returns `false quick false`.

`tasks.join` always returns `timed_out` as its second value. It is `false` whenever no timeout fired: when every member finishes before the timeout, and when no timeout was given, including calls with no `opts` at all.

One timer covers a whole timed `tasks.join`, rather than restarting for each member. When every member finishes before the timeout, `timed_out` is `false`, every member has an entry in the order of the handles passed, and the wait cancels its timer, so `tasks.pending()` is empty and the run ends without waiting out the timeout.

### Collecting the members a timeout left running

After a timed-out `tasks.join`, the unfinished entries are holes that can sit at any position, the first included. Holes make `#results` and `ipairs` unreliable, so walk the set's positions with `for i = 1, #set do` and test each `results[i]` for `nil`:

````lua
local results, timed_out = tasks.join(set, { timeout = 5 })
if timed_out then
  local late = {}
  for i = 1, #set do
    if results[i] == nil then
      late[#late + 1] = set[i]
    end
  end
  tasks.join(late)
end
````

Members that missed a time limit keep running, and a later wait gathers them. They still need a wait or a cancel before their owner ends, because tasks still running and not idle when their owner ends are reported in the owner's `tasks_live` outcome (see [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment)).

### Late members and the store

A timed join that delivers some members leaves the rest for a later wait, and a late member's store writes become readable once that wait delivers it. Reading them between the timed-out wait and that delivery ends the run, as [Store writes after a wait](#store-writes-after-a-wait) explains.

When an earlier timed join may already have delivered the task, wrap the follow-up wait in `pcall`, because a wait naming a task whose result was already taken raises `task_consumed` (see [Waiting with `tasks.join_any` and `tasks.join`](#waiting-with-tasksjoin_any-and-tasksjoin)):

````markdown
---
name: late-write
description: Reads a file that a task wrote, after a timed join and a follow-up wait.
runvil: 0
---

## Main

```lua
local a = tasks.spawn('../Slow')
tasks.join({ a }, { timeout = 0.05 })
local ok2 = pcall(tasks.join_any, { a })
return store.read('a.txt')
```

## Slow

```lua
store.write('a.txt', 'slow task')
return 'written'
```
````

````text
slow task
````

Whichever wait delivers `Slow`, the timed `tasks.join` or the follow-up `tasks.join_any`, the task's store write is readable by the time `store.read` runs, and the run result never changes.

### Waits a stop cuts short

When a stop from the Host drops the timer of a timed `tasks.join` or `tasks.join_any` while the wait is waiting, the wait raises an error value of kind `canceled`, which `pcall` can catch. The tasks the wait was waiting on are not canceled, and a later wait joins each one the wait the stop cut short had not yet delivered. Left uncaught, the error ends the run with code `canceled`.

A timed `tasks.join` cut short this way loses the results of the members it delivered before the stop, so a later wait naming one of those members raises `task_consumed`.

### Timed waits inside tasks

A spawned task can itself use a timed wait on its own tasks. Replace `Child` in `timed-wait` with these two sections:

````markdown
## Child

```lua
local g = tasks.spawn('../Grandchild')
local first, ok, result = tasks.join_any({ g }, { timeout = 30 })
return result
```

## Grandchild

```lua
return 'grandchild done'
```
````

````text
true true grandchild done 0
````

While `Child` sits in its timed `tasks.join_any`, its status reads `blocked` as `tasks`, exactly as it does in an untimed wait, and its `tasks` list shows only the real member, `0.0.0`.

### The timer behind a time limit

A wait with a time limit starts an internal timer, which the wait keeps so it can wait on it and cancel it. The timer is never a task you can see:

- `tasks.pending()` and a status table's `tasks` list never show it.
- It produces no task events, so you observe events only for the real members (see [Task events and task errors](#task-events-and-task-errors)).
- When a member ends before the timeout, the wait cancels the timer silently as its own bookkeeping, and no cancel for it appears anywhere you can see.
- The timer ends with its owner like a task, but it emits no event and never counts as a task left live in `tasks_live` when its section ends.
- When the Engine refuses a timed wait, for example with `task_not_owned`, the wait stops its timer before raising, so no timer outlives its wait.

When a member ends at the same moment the timer fires, the finished member wins and the timeout never appears among the results. That is because the timer sits after all the members in the wait's set, and the wait delivers the first ended entry in that order.

The timer is [Harness work](01-what-a-prompt-is.md#effects-and-harness-work): the Harness performs it as a sleep for the timeout's duration.

The one place the timer shows is in task ids. Each wait with a valid timeout takes one index from the waiting chain's child counter for its timer (see [One child counter per chain](#one-child-counter-per-chain)). Replace `Main` in `timed-wait` with:

````markdown
## Main

```lua
local a = tasks.spawn('../Child')
tasks.join_any({ a }, { timeout = 30 })
local b = tasks.spawn('../Child')
tasks.join_any({ b })
return a.task .. ' ' .. b.task
```
````

````text
0.0 0.2
````

After task `0.0` and one timed wait, the next spawn is `0.2`, because the timer took `0.1`. A wait without a timeout takes no child index, and neither does a wait whose `opts` were refused.

### Wait timeout errors

`opts` and its `timeout` are checked at the call, before the wait starts, and each broken rule fails with an error value of kind `lua`. The error comes back as the wait call's own error rather than ending the run, so it raises at the call site, where `pcall` catches it and the error value's `kind` field reads `lua`. Because the check runs before any timer starts, no time passes before the error. Replace `Main` in `timed-wait` with:

````markdown
## Main

```lua
local t = tasks.spawn('../Child')
local _, e1 = pcall(tasks.join_any, { t }, { timeout = 'soon' })
local _, e2 = pcall(tasks.join, { t }, { timeout = -1 })
local _, e3 = pcall(tasks.join_any, { t }, 'opts')
local _, ok, result = tasks.join_any({ t })
return e1.kind .. ' ' .. e2.kind .. ' ' .. e3.kind .. ' ' .. result
```
````

````text
lua lua lua quick
````

A refused wait starts nothing. No timeout is set, no timer starts, no child index is used up, and the members are untouched, so after a `pcall` the owner can still wait on them, and the following valid wait still delivers `quick`.

The rules, with the message each broken rule raises:

- `opts`, when given, must be a table. Any other value fails with `{call} opts must be a table, got {type}`, a message that starts with the call's name, such as `tasks.join opts must be a table, got {type}`.
- `timeout` must be a Lua number, written as `5` rather than `'5'`, because the wait never converts a string. Any non-number fails with `{call} timeout must be a number, got {type}`, which names the call and the value's Lua type, such as `tasks.join_any timeout must be a number, got string`.
- `timeout` must be zero or more, finite, and small enough for a duration to hold. `tasks.join` and `tasks.join_any` both refuse any other number before the wait starts and before any timer exists, with the refusal `timeout must be a non-negative finite number of seconds, got {seconds}`, which the Engine makes, so the error value's message puts the wait's `{file}:{line}: ` in front of it and the Lua runtime prose after it, while the two messages above have no location. The refusal names `timeout` and prints the value: a negative value prints as written, such as `-1`, a NaN value such as `0/0` prints as `NaN`, an infinite value prints as `inf`, and a value beyond the range a duration can hold prints in full, such as `100000000000000000000` for `1e20`.

## Task ids and handles

After this section you can predict the id of every task a prompt starts and build and pass Task handles.

````markdown
---
name: task-ids
description: Shows the ids of two tasks spawned from the main walk.
runvil: 0
---

## Main

```lua
local a = tasks.spawn('../Child')
local b = tasks.spawn('../Child')
local results = tasks.join({ a, b })
return sys.taskid .. ' ' .. a.task .. ' ' .. b.task .. ' '
  .. results[1].result .. ' ' .. results[2].result
```

## Child

```lua
return sys.taskid
```
````

````text
0 0.0 0.1 0.0 0.1
````

Every task a run starts has a task id, which you see inside the Task handle `{ task = id }`. `tasks.spawn` hands the id back inside the handle, so `a.task` reads `0.0`, and status tables and pending lists show ids the same way, as dotted path strings. The main walk is task `0`, and the two tasks it spawns here are `0.0` and `0.1`, numbered in spawn order. Inside each task, `sys.taskid` holds that task's own id as a string.

### How task ids are built

A task id reads as plain decimal numbers joined by dots, with no prefix, suffix, or padding, such as `0`, `0.2`, or `0.2.0`, so the id spells out the hierarchy it names in a log or a UI.

Every chain in a run other than the main walk, whether a `call` chain or a spawned task, has a hierarchical id made by extending its parent chain's id with one child index. The main walk is the root chain, with the id `0`, and every task id path starts from it. A chain's id is its parent's id followed by a dot and the chain's own child index, counted from zero, so:

- The first task spawned from the main walk gets the id `0.0`, and the tasks spawned after it get `0.1`, `0.2`, and so on in spawn order. Two `tasks.spawn('../Child')` calls in one section produce `0.0` and then `0.1`.
- A task started inside another task gets that task's id extended by one more number, the spawner's next child index, so a `Leaf` task spawned by task `0.0` is `0.0.0`, and child 1 of task `0.2` is `0.2.1`.

A task's id is the id of the chain that runs it, so both read as the same dotted path. `sys.taskid` holds, as a string, the id of the task the code runs inside: `'0'` on the main walk, and the task's own id inside a spawned task or a fanout arm, so code in the first task the main walk spawns reads `'0.0'`. A `call` chain reports its caller's task.

A task's own sections read `sys.id` values that extend the task id with a section entry index (see [Section entry ids](05-lua-environment.md#section-entry-ids)), so the first section of task `0.0` reads `sys.id` `0.0.0`. Section entries and children count separately, so the same text can name a section entry and a task: `0.0` can be both the `sys.id` of the first section the main walk enters and the first task's id.

### One child counter per chain

A chain has one child counter shared by the chains it calls, the tasks it spawns, the fanout arms it starts, and the internal timer of each timed wait (see [The timer behind a time limit](#the-timer-behind-a-time-limit)). So a task's id is its spawner's next child index, and a `call` uses up an index the next spawn would otherwise get:

````markdown
---
name: shared-counter
description: Shows a call and a spawn sharing one child counter.
runvil: 0
---

## Main

```lua
local inner = call('../Helper')
local t = tasks.spawn('../Child')
tasks.join_any({ t })
return inner .. ' ' .. t.task
```

## Helper

```lua
local leaf = tasks.spawn('../Child')
tasks.join_any({ leaf })
return leaf.task
```

## Child

```lua
return 'child done'
```
````

````text
0.0.0 0.1
````

The `call('../Helper')` is the main walk's first child, so the called chain is `0.0`, and the spawn inside it is that chain's first child, `0.0.0`. The spawn after the call is the main walk's second child, so it gets `0.1`. In general, when a chain's first two children are a `call` and then a spawn, they get `.0` and `.1` under it.

The same numbering holds everywhere a chain starts children:

- The first fanout arm started from the main walk has task id `0.0`, and a spawn inside that arm gets `0.0.0`, the same id as a spawn inside the main walk's first called chain, because both extend a child chain `0.0` by its first child index (see [Arms as tasks, ids, and events](14-fanout.md#arms-as-tasks-ids-and-events)).
- The main walk has one child counter for all of its sections, the [preamble](02-file-structure.md#the-preamble) included, so a task spawned in the preamble is `0.0`, the next spawn in a later section is `0.1`, and each task keeps its id for the rest of the run.
- Each wait with a valid timeout takes one index for its timer (see [The timer behind a time limit](#the-timer-behind-a-time-limit)).

Ids depend only on the spawner's own order of calls, not on how the scheduler interleaves work. Two runs of the same prompt with the same inputs give the same tasks the same ids, however their chains interleave. Likewise, the `sys.id` of the section entry that owns a set of tasks does not change with the order in which those tasks finish.

### Spawn order and sorting ids

Every list of tasks the Engine gives you, such as the handles `tasks.pending` returns and the id strings in a status table's `tasks` field, comes in spawn order. Spawn order compares ids number by number, so `0.9` comes before `0.10`. Sorting id strings with `table.sort` does not give spawn order, because Lua's string comparison puts `'0.10'` before `'0.9'`.

### Handles as plain data

A Task handle is the plain table `{ task = id }` that [The task handle](#the-task-handle) describes, with no Engine object behind it, so every operation on it is a `tasks` namespace function that takes the handle, and you compare `task` fields, as in `results[1].task == b.task`, to tell tasks apart.

Each member of a wait set takes the same two forms as any other task argument, so `tasks.status(sys.taskid)` and `tasks.join_any({ sys.taskid })` both pass their task in an accepted form. Because any table with a string `task` field works as a Task handle, you can build one by hand from an id, such as `{ task = sys.taskid }`, and each entry `tasks.join` returns, `{ task = id, ok = ok, result = result }`, works as one too. A task id string reads back as the identical task id, so the id text the Engine hands you names the same task when you pass it back to a `tasks` function.

Because the handle is plain data, a Task handle stored in [`var`](05-lua-environment.md#keeping-state-in-var) keeps its shape and still works in a later section of the same chain:

````markdown
---
name: early-start
description: Starts a task in one section and waits for it in a later one.
runvil: 0
---

## Start

```lua
var.job = tasks.spawn('Job')
```

### Job

```lua
return 'job done'
```

## Finish

```lua
local _, ok, result = tasks.join_any({ var.job })
return result
```
````

````text
job done
````

`Start` spawns its child section `Job` and keeps the handle in `var.job`. The walk falls through to `Finish`, which waits on `var.job`. The same works for a handle saved in `var` in the preamble and waited on from a later section, and for a handle joined from a later section reached by [`jump`](08-jump-and-call.md#jumping-to-a-section).

### Valid task ids and id errors

A valid task id is one or more numbers joined by single dots, where each number is one or more ASCII decimal digits with no sign or space and a value from 0 to 4294967295, such as `0`, `0.2`, or `0.12.0`.

Passing a string that is not a valid task id, such as `0.x`, to `tasks.ready`, `tasks.status`, `tasks.cancel`, `tasks.join_any`, or `tasks.join` fails at the call site with an error value of kind `lua` whose message is the call's location, this refusal, and the Lua runtime prose:

````text
{file}:{line}: `{text}` is not a task id such as `0.1`: invalid chain id `{text}`: required a dot-separated path of decimal components. The Lua code failed at runtime. Fix the code at this location.
````

In a wait set, one malformed member fails the whole call.

A value that is neither a Task handle nor a string, such as `42` passed to `tasks.cancel`, fails with an error value of kind `lua` whose message is `{call} expects a Task handle or task id, got {type}`, with no location, naming the call and the value's type. `tasks.ready`, `tasks.status`, `tasks.cancel`, and each member of a wait set apply this check.

## Concurrency, nesting, and scheduling

After this section you can predict how many tasks run at once, nest tasks inside tasks, and tell when a spawned task actually starts running.

````markdown
---
name: nested-tasks
description: A task spawns two tasks of its own and joins them.
runvil: 0
---

## Main

```lua
local outer = tasks.spawn('Outer')
local _, ok, result = tasks.join_any({ outer })
return result
```

### Outer

```lua
local results = tasks.join({ tasks.spawn('Inner'), tasks.spawn('Inner') })
return results[1].result .. '/' .. results[2].result
```

#### Inner

```lua
return 'inner ' .. sys.taskid
```
````

````text
inner 0.0.0/inner 0.0.1
````

A spawned task can itself spawn and join tasks, so tasks nest. Here `Main` spawns `Outer`, which spawns two `Inner` tasks from its own child section and joins them before `Main` joins `Outer`. Each `Inner` task is a task of `Outer`, so its id extends `0.0`.

### When a spawned task is admitted

Each [suspending call](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), meaning `models.infer`, `models.loop`, `tools.call`, `call`, and every `store` operation, pauses only the current block and hands control to the scheduler, so other chains, such as spawned tasks and `tasks.fanout` arms, run while it waits.

`tasks.spawn`, `tasks.ready`, `tasks.status`, `tasks.pending`, `tasks.note`, `tasks.send`, `tasks.concurrency`, and `tasks.cancel` answer right away and never park the caller, and so do `tasks.receive()` and `tasks.reply` when a message already waits in the inbox. A queued task is admitted only when no chain is ready to run, so a loop that does nothing but call these functions never lets a queued task start. A chain that sends a message and then waits, as `Main` in `draft-loop` does, lets the task it sent to run.

After `tasks.spawn` returns, the new task waits in the admission queue, with its status reading `blocked` as `queued` (see [What a live task is doing: `section` and `blocked`](#what-a-live-task-is-doing-section-and-blocked)). It is admitted only once its spawner has parked or ended and a slot is free under every limit it counts against:

- When the spawner parks on a model round right after spawning, as in `tasks.spawn('../Child')` followed by `models.infer('wait')`, the task is admitted and starts its own model round, so both rounds are in flight together and go out together.
- While the spawning section is parked on a store operation, a spawned task can run all the way to completion, and the spawner resumes only after the store operation is answered.

A queued task holds no Lua VM yet, but its model is already fixed. It starts on the copy of the spawner's current model that `tasks.spawn` took, so a `models.use` the spawner makes while the task waits for a slot does not change it (see [The model a task starts on](#the-model-a-task-starts-on)).

Tasks admitted together first run in spawn order, so their first model requests go out in spawn order. In `nested-tasks`, `Main` parks on its `tasks.join_any`, which admits `Outer`, and `Outer` parks on its `tasks.join`, which admits both `Inner` tasks.

### Concurrency slots

A chain's concurrency limit, the most of its tasks admitted at once, gates when its queued spawned tasks start exactly as it gates fanout arms, so with a limit of two, two spawned tasks start together and their work is in flight at the same time. [Concurrency and the admission window](14-fanout.md#concurrency-and-the-admission-window) covers the run's default limit and how `tasks.concurrency` sets and reads a chain's limit, inside a task included.

A concurrency slot is one place under a chain's concurrency limit. Every spawned task takes one concurrency slot at the chain that owns it and at every enclosing ancestor chain, and it holds that slot until the task ends. So concurrency limits nest: in `nested-tasks`, each `Inner` task takes a slot under the limit of `Outer` and another under the main walk's. A task spawned inside a `call` counts against the call chain's limit and then against the limits of the calling chain and every chain above it.

A task spawned while every slot it needs is taken is queued, and it does not start until a slot frees. A queued task starts only when its owner and every enclosing ancestor each have a free slot under their own limit. The scheduler admits tasks by these rules:

- Tasks waiting for their first slot start in the order they were spawned.
- A queued task whose owner or ancestor is full does not hold back a later-spawned task whose chains have free slots, because each admission pass skips tasks that cannot fit and still admits later ones that can.
- Tasks resuming from a wait retake their slots ahead of tasks that have not started yet, so a resumed task never starves behind a long queue of fresh spawns.
- Tasks waiting to resume retake their slots in the order their waits answered.
- Abandoning tasks frees the concurrency slots they held, so queued tasks that now fit are admitted.

A queued task whose owner is aborted before the task is admitted, for example because the owner was canceled, is abandoned without ever running its block, and it reports no start or end event (see [Task events and task errors](#task-events-and-task-errors)).

### Waiting gives a slot back

A task parked on a wait for other tasks, such as a join, gives its concurrency slot back for the length of the wait, so its own tasks can be admitted and run under its limit. A task waiting on its inbox gives its slot back the same way, in `tasks.receive()` or [idle](#idle-tasks) after `tasks.reply`, so a task waiting for its owner's next message never holds a slot another task could use. That is why nested spawn-and-join does not deadlock, even when the run admits only one task at a time. In `nested-tasks` with a run concurrency limit of 1, `Outer` gives its slot back while it joins, so the two `Inner` tasks can take it one after the other, and the run still returns `inner 0.0.0/inner 0.0.1`.

These rules decide who gives a slot back and how a waiting chain continues:

- A `call` chain running inside a task that parks on a task wait gives back the slot of the task it runs inside, so the tasks under that task can run during the wait.
- The main walk holds no slot, so a wait in the main walk, or in a `call` chain under the main walk, gives no slot back.
- When a wait that gave back a slot answers, a task wait or a message that ends an inbox wait, the waiting task or `call` chain queues again and must retake that slot before it continues, reading `blocked` `queued` while it waits.
- The main walk resumes right away when the wait answers, without queueing for a slot.

### The depth cap for spawned tasks

Each spawned task runs one depth level deeper than the chain that spawned it, so a task spawned from the main walk runs at depth 1, and its `tasks.status` reads `depth` as 1. `tasks.spawn` shares the depth cap of 8 with `call` and `tasks.fanout` (see [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap)), so chains at depths 1 through 8 start, and the spawn that would start depth 9 is refused:

````markdown
---
name: deep-spawn
description: Spawns one task inside another until the depth cap refuses a spawn.
runvil: 0
---

## Main

```lua
local t = tasks.spawn('../Odd')
local _, ok, result = tasks.join_any({ t })
return result
```

## Odd

```lua
local ok, t = pcall(tasks.spawn, '../Even')
if not ok then
  return tostring(t)
end
local _, joined, result = tasks.join_any({ t })
return result
```

## Even

```lua
local ok, t = pcall(tasks.spawn, '../Odd')
if not ok then
  return tostring(t)
end
local _, joined, result = tasks.join_any({ t })
return result
```
````

````text
deep-spawn.md:29: recursion exceeded cap of 8. Calls nest deeper than the cap allows. Nest `call`, `tasks.spawn`, and `tasks.fanout` less deeply.
````

`Odd` and `Even` spawn each other, because a section cannot spawn itself, and each task runs one level deeper than the one that spawned it. The `Even` task at depth 8 tries to spawn depth 9, and that `tasks.spawn` is refused with `recursion exceeded cap of 8`, the same words a `call` or a `tasks.fanout` past the cap gives. The refusal raises at the call site, on the line of the `tasks.spawn` that went too deep, and `pcall` catches it, as in `local ok, err = pcall(tasks.spawn, '../Odd')`. Its message adds the depth cap's prose after the refusal (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). Each task returns the result it joined, so the message travels back up to `Main` and becomes the run result.

Inside Lua the depth-cap refusal is an error value of kind `lua` whose `code` is `call_depth_exceeded`, and left uncaught it ends the run with that [error code](16-limits-and-errors.md#run-error-codes) and the message `recursion exceeded cap of 8 (at {file}:{line})`.

### Calls, tools, and model rounds inside tasks

A task runs the same Lua as any chain, and these rules cover what happens when it calls sections, runs tools, and makes model rounds:

- A section reached by `call` inside a task shares that task's `sys.taskid`, so it can read the task's status too.
- When two tasks each run a `call`, the called chains can finish in either order, one finishing while the other is still running, and each `call` still returns its own result to its own task.
- A section reached by `call` inside a task starts on a copy of the task's current model, taken when the `call` is made, and a `models.use` inside the called chain does not reach the task.
- A `models.infer` reply made inside a `call` chain of a task flows back through `call` to the task's result and then to the spawner's `tasks.join`. For example, `Main` joins `Caller`, which runs `return call('Leaf')`, and `Leaf` runs `return models.infer(var.who)`, so the reply reaches `Main` as the result of `Caller`.
- An adhoc tool handler runs inside the calling chain, so it can start and wait on tasks just as block code can (see [How adhoc tool handlers run](12-tools.md#how-adhoc-tool-handlers-run)).
- Each task keeps its own round count starting at zero, so its rounds are numbered on its own counter and never add to its spawner's count, and its `tasks.status` reads `turns` as 0 while its first `models.infer` round is in flight and 1 after it returns.
- A task spawned after its spawner writes a store path reads the spawner's content, while a task spawned before that write always meets a claims conflict when it reads the path. That conflict ends the run on the spot with error code `determinism` and a message naming the path, and no `pcall` catches it (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)).

## Task events and task errors

After this section you can read the events a task reports in the run log and recognize every error a `tasks` call raises, with the error code each one ends the run with when nothing catches it.

````markdown
---
name: task-events
description: Spawns one task and collects its result.
runvil: 0
---

## Spawner

```lua
local t = tasks.spawn('../Child')
local results = tasks.join({ t })
return results[1].result
```

## Child

```lua
return 'child result'
```
````

The run result is:

````text
child result
````

Task events are the run events that record a task's life (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)). Every task that starts reports a `TaskStarted` event followed by exactly one terminal task event, which is `TaskSucceeded`, `TaskFailed`, `TaskCanceled`, or `TaskAbandoned`. This run reports a `TaskStarted` event under `Spawner` for task `0.0`, with the target `Child`, and then a `TaskSucceeded` event under `Child` for the same task. A task canceled or abandoned while still queued for a concurrency slot never starts, so it reports neither.

### The events of one task

Every started task records exactly one start event and exactly one terminal event.

- A task's events report under the task's own id, so the log shows which task produced them.
- The same task id marks everything a task reports while it runs, so a handle's `task` field matches those reports to the task (see [Task ids and handles](#task-ids-and-handles)).
- A spawned task's section events and logs appear under the target section's name, not the spawner's.
- Between its start and its end, a task that reads its inbox reports `inbox_waiting` each time it parks on its empty inbox, and each message its owner sends it reports `inbox_message` under the task (see [Inbox events](#inbox-events)). A reply is not a terminal event: the task's one terminal event comes when it ends.

### The `TaskStarted` event

Spawning a section reports a `TaskStarted` event under the spawning section that names the task id, the target section's name, and the spawn seeds (see [Starting a task with `tasks.spawn`](#starting-a-task-with-tasksspawn)). With no options, `args`, `item`, and `index` are `null`, while `var` is still a copy of the spawner's `var`.

The event includes the execution, the section, the provenance, the task id, the target, the spawn `args`, the `item`, the `index`, and a copy of `var`. Those last four are the seeds the task sees: the `args` object, the `item` value, the `index`, and a snapshot of `var`. The `args` field holds the JSON object the spawn gave the task: a table of named fields as that object, a string as `{"text":"..."}`, and `null` when the spawn gave none and the task reads its owner's `args`.

A task's start event fires when the task gets a concurrency slot and first runs, not when it is spawned:

- A task still waiting for a slot reports no start event, so under a concurrency limit of 1 the second task's start event comes after the first task's `TaskSucceeded` event (see [Concurrency, nesting, and scheduling](#concurrency-nesting-and-scheduling)).
- Each spawned task emits exactly one start event, even when it waits in the queue before starting, and a task that resumes after a wait does not emit a second one.
- Fanout arms still queued for a slot when they are canceled never report a start, while each arm that gets a slot within the concurrency window reports a start event when it begins (see [Arms as tasks, ids, and events](14-fanout.md#arms-as-tasks-ids-and-events)).
- A task canceled before it ever starts reports neither a start event nor a terminal event, though its state still becomes canceled.

The start event, reported when the task first runs, names the spawn's `args` and the section that called `tasks.spawn`, captured at spawn time. A task spawned in `Main` still reports `Main` even when the spawner calls `jump('../Next')` before the task first runs (see [Jumping to a section](08-jump-and-call.md#jumping-to-a-section)). The event appears in the spawner's own event sequence and names the section the task was spawned from, even when the spawner has moved on to another section by the time the task starts.

### Terminal task events

A finished task fires a `TaskSucceeded` or `TaskFailed` event under its target section, naming the task and stamped with the task's own provenance:

- A task whose section returns normally reports a single `TaskSucceeded` terminal event under its section, after its `TaskStarted`.
- A task whose section raises an error reports `TaskFailed` under its section and never reports `TaskSucceeded`.
- `TaskSucceeded` events appear in the order the tasks actually finish, which can differ from the order you collect them.
- A task the [idle cap](#the-idle-cap) closes reports `TaskSucceeded` under its target section, stamped with its own provenance, as if it had returned its latest reply, after the events of the live tasks it owned, and the event does not say that the cap closed it.

In a fanout, every started arm produces exactly one terminal task event: `TaskSucceeded` for an arm that returns, `TaskFailed` for an arm that raises, and `TaskCanceled` for a sibling canceled by fail-fast (see [When an arm fails](14-fanout.md#when-an-arm-fails)) and for an arm that replied with `tasks.reply`, which ends canceled once the call takes its reply. A failing arm reports a `TaskFailed` event, and when the fanout fails no arm reports `TaskSucceeded`.

Canceling a started task reports a `TaskCanceled` event exactly once, attributed to the canceled task's section (see [Canceling and listing tasks](#canceling-and-listing-tasks)):

- Canceling a started task that is still live, running or idle, emits one `TaskCanceled` event naming the task, after the events of everything the task owned.
- The event carries the canceled task's own provenance, not that of the chain that canceled it.
- A canceled task reports no other terminal event, so no `TaskSucceeded`, `TaskFailed`, or `TaskAbandoned` appears for it.
- Canceling a task that was still queued for a concurrency slot emits no `TaskCanceled` event, since it never started.

An abandoned task fires a `TaskAbandoned` event under its target section, stamped with the task's own provenance and carrying one of the five [abandon reasons](#abandon-reasons). The reason names how the owner ended, and for an owner that returned normally it is `owner_returned`, whose phrase is `the section ended`. When the owner's walk ends with a task still live, the task is abandoned with reason `owner_returned`, and its `TaskAbandoned` event reports under the task's own section. Once an owner is gone, no wait can reach its abandoned tasks, so each task's `TaskAbandoned` event is the only record of how it ended.

- A task still queued for a concurrency slot when it is abandoned emits no terminal event, because it never emitted a start event. A task abandoned before it ever ran, such as a queued task whose owner fails first, emits no `TaskStarted` event and no terminal event, not even `TaskAbandoned`, though its state still reaches abandoned.
- Tasks nested under an abandoned task report their own endings before the abandoned task's terminal event, which is always the last word on that task. During a run-wide shutdown no task reports its terminal event twice, because nested tasks have already ended by the time their owner is reached.
- When the whole run ends through a cancel or a fatal answer, every started task still live receives exactly one terminal event before the run's end boundary.

The run log tells a canceled task, stopped on purpose, apart from an abandoned one, which lost its owner. Either way, the section a stopped task was running does not finish, records no completion, and reports no `SectionFinished` event, and the task is recorded as canceled when a cancel stopped it and as abandoned when its owner's end stopped it.

### Argument errors from the `tasks` functions

The `tasks.*` functions check their arguments at the call site and raise an error of kind `lua` in the calling section, which `pcall` can catch (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). The arguments are validated before the operation runs, so after a bad argument no task operation runs. Every mistake in a `tasks` call raises an error value (see [Error values](05-lua-environment.md#error-values)) carrying a `kind`, a `task` field naming the task where there is one, and a message you read with `tostring(err)`. Argument errors are structured error values with kind `lua` and a `message` field, raised at your own call site.

Argument checks made by the `tasks` functions themselves give messages that start with the call's name, such as `tasks.spawn opts must be a table, got integer`, with no location, while other checks, which the Engine makes, name only the field or value at fault, and their error value's message puts the call's `{file}:{line}: ` in front of that refusal and the Lua runtime prose after it (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). In the checks the `tasks` functions make themselves, the type named at the end is `integer` for a Lua integer such as `5`, `number` for a float such as `1.5`, and the Lua `type()` name for every other value, such as `string` or `table`, except that the `tasks.fanout` first-argument check reads any number as `number`. The checks the Engine makes use the same names, except that they name `json.null` `lightuserdata`.

What each function takes, and the `lua` error message it raises for anything else:

- The `tasks.spawn` target must be a string, the same rule `call` applies to its target (see [Calling a section](08-jump-and-call.md#calling-a-section)), and any other type raises the byte-identical error `call` raises, `section target must be a string, got {type}`, such as `section target must be a string, got integer`.
- `tasks.spawn` takes a table of named fields or a string in `opts.args`, JSON data in `opts.item`, and a Lua integer of 0 or more in `opts.index`. Otherwise the call raises an error naming the field: `args must be a table of named fields or a string, got {type}`, `item must be JSON data, got {type}`, or `index must be a non-negative integer, got {index}` for an integer below 0, while an `index` of another type, floats included, ends in its type name, such as `got number`. The `args` message ends in `got a list` for a table that is a list, such as `{ 1, 2 }`, and in the type name for a number or a boolean, as in `got integer` for `5`, `got number` for `1.5`, and `got boolean` for `true`. A table in `args` that holds a value with no JSON form raises an error that names the value's place, such as `args: value.f must be JSON data, got function` for a function in the field `f`.
- Every call that names a task, `tasks.join_any`, `tasks.join`, `tasks.ready`, `tasks.status`, `tasks.send`, and `tasks.cancel`, takes a task handle or a task id string such as `0.1`. Any other value raises `{call} expects a Task handle or task id, got {type}`, and a string that is not a dot-separated id raises an error whose refusal starts `` `{text}` is not a task id such as `0.1`: ``.
- `tasks.join_any` and `tasks.join` take a non-empty table of tasks and an optional `opts` table whose `timeout` is a non-negative finite number of seconds. Otherwise they raise one of these before any wait starts: `{call} expects a set of tasks, got {type}`, `{call} requires at least one task`, `{call} opts must be a table, got {type}`, `{call} timeout must be a number, got {type}`, or `timeout must be a non-negative finite number of seconds, got {seconds}`.
- `tasks.pending` takes no arguments, and any argument other than `nil` raises `tasks.pending takes no arguments, got {type}`, such as `tasks.pending takes no arguments, got table`.
- `tasks.note` takes a string, and otherwise raises `tasks.note text must be a string, got {type}`, such as `tasks.note text must be a string, got integer`.
- `tasks.receive` takes no arguments, and any argument, `nil` included, raises `tasks.receive takes no arguments, got {type}`, such as `tasks.receive takes no arguments, got string`. A refused call takes no message from the inbox.
- `tasks.send` takes a string `text` after its task, and otherwise raises `tasks.send text must be a string, got {type}`, such as `tasks.send text must be a string, got integer`. A `text` that is not valid UTF-8 raises an error whose refusal reads `text must be a valid UTF-8 string`.
- `tasks.reply` takes one value a section could return, and a table, or any other value with no text form, raises an error whose refusal reads `cannot return a {type} as a result`, such as `cannot return a table as a result`.
- `tasks.concurrency` takes a positive whole number, and [Concurrency and the admission window](14-fanout.md#concurrency-and-the-admission-window) gives the messages it raises for anything else.

### Spawn failures as error values

Put this block in the `Spawner` section of `task-events` in place of the first one:

````lua
local ok, err = pcall(tasks.spawn, '../Missing')
return tostring(ok) .. '|' .. err.kind .. '|' .. err.message:match('^(.- not found)')
````

The prompt has no `Missing` section, so the block returns the message up to `not found`, which starts with the location of the `tasks.spawn` call:

````text
false|lua|{file}:{line}: section `../Missing` not found
````

Every spawn failure, including the depth cap, an unresolved target, and a setup failure, is raised as an error of that `tasks.spawn` call in the spawning code, so `pcall(tasks.spawn, ...)` catches it and the run continues. Bad arguments to `tasks.spawn` raise at the call site the same way. When the Engine rejects a spawn, `tasks.spawn` raises the Engine's error value and returns no handle.

- When the target path names no section, `tasks.spawn` raises an error value whose `kind` is `lua`, and `pcall(tasks.spawn, '../Missing')` returns `false`. The refusal is exactly the one `call` gives for the same path, ``section `{path}` not found: {where}``, naming the path and then the section that was searched. The message puts the `tasks.spawn` line in front of it and the Lua runtime prose after it, so `tostring(err)` matches what `pcall(call, '../Missing')` gives on the same line (see [The not-found error](08-jump-and-call.md#the-not-found-error)).
- The depth cap lets depths 1 through 8 start and refuses depth 9 with the refusal `recursion exceeded cap of 8`, on the line of that `tasks.spawn` (see [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap)). A depth-cap refusal left uncaught ends the run with error code `call_depth_exceeded`.
- Spawn failures are error values of kind `lua`, except a store refusal during task setup, which is kind `store`: its message is the `tasks.spawn` line and then the fixed prose for its store reason, as for any caught store failure, and left uncaught, the run's message starts `store operation failed: ` followed by the store's own failure text (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). When a Host-supplied store refuses to open store access for a new task, `tasks.spawn` fails with that error value, `pcall` can catch it, and no task is left started. The run's own in-memory store never refuses (see [Host stores, policies, and modes](09-the-store.md#host-stores-policies-and-modes)).

### Refusals the Engine raises

When the Engine refuses a `tasks` operation, the call raises the Engine's error value instead of returning it. That holds for a refused `tasks.spawn`, `tasks.join_any`, `tasks.join`, `tasks.ready`, `tasks.status`, `tasks.pending`, `tasks.note`, `tasks.receive`, `tasks.send`, `tasks.reply`, `tasks.cancel`, or `tasks.concurrency`, including a wait whose `timeout` the Engine rejects before any wait starts.

| Error kind | Raised when | Field to read |
| --- | --- | --- |
| `lua` | an argument is wrong, a target does not resolve, the depth cap is crossed, or `tasks.reply` runs on the main chain | `message` |
| `task_not_owned` | code reaches for a task it does not own, a `tasks.send` included | `task` |
| `task_consumed` | a wait asks for a task result that was already delivered, an idle task's taken reply included | `task` |
| `store` | a Host-supplied store refuses during task setup | `message` |

Ownership and delivery refusals keep their own kinds, `task_not_owned` and `task_consumed`, rather than `lua` (see [Ownership, leaked tasks, and abandonment](#ownership-leaked-tasks-and-abandonment) and [Waiting with `tasks.join_any` and `tasks.join`](#waiting-with-tasksjoin_any-and-tasksjoin)). You read `err.task` on a `task_not_owned` error, a `task_consumed` error, or a canceled task's error value to get the task id. Put this block in the `Spawner` section of `task-events` in place of the first one:

````lua
local ok, err = pcall(tasks.cancel, '9.9')
return err.kind .. '|' .. err.task
````

No task `9.9` belongs to `Spawner`, so the block returns:

````text
task_not_owned|9.9
````

### Task error codes

Task lifetime and ownership faults count as your program failing, so when no `pcall` catches them they end the run with their own error (see [The run result](04-how-a-prompt-runs.md#the-run-result)). Each keeps its own code, one of the task codes, and a depth-cap refusal keeps `call_depth_exceeded`, one of the Lua codes (see [Run error codes](16-limits-and-errors.md#run-error-codes)):

| Left uncaught | Error code |
| --- | --- |
| A leaked task, one you leave still running and not idle | `tasks_live` |
| Reaching for a task from code that does not own it | `task_not_owned` |
| Waiting on the same task result twice | `task_consumed` |
| A canceled task's error value, raised again after its wait | `task_canceled` |
| A depth-cap refusal from `tasks.spawn` | `call_depth_exceeded` |

A wait hands a canceled task's error value back without raising it, so a task cancellation goes uncaught only when your code raises that value again, for example with `error(err)` right after the wait. Left uncaught, it ends the run with code `task_canceled`, while an uncaught Host cancel ends the run with code `canceled`.

### What can differ between runs

Given the same model and tool answers, only three outcomes can differ between interleavings of a run's tasks: which task `tasks.join_any` returns, how far a canceled task got before it stopped, and where a message sent with `tasks.send` lands in its task's work. A task's record does not depend on how fast or in what order model answers arrive unless its owner sends it a message, because a `tasks.send` acts on what the task is doing when the message arrives (see [Sending with `tasks.send`](#sending-with-taskssend)): answer order can decide whether the message withdraws a model request, joins the next one, or waits for the task's next read. Without a message, answering requests one at a time, all at once in issue order, or all at once in reverse gives the same result text and Harness work for each task, with only the interleaving across tasks differing, and the run result is the same too. Concurrent tasks finish correctly whatever order their model replies arrive in.
