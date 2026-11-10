# How a prompt runs

Every run moves through the same stages in the same order: the parse, prepare, the start checks, the H1 pass, and the section walk, and it ends with its result: its final text, or the error that ended it. This chapter follows that order, one stage at a time. Knowing the order tells you which mistakes stop a run before any of your Lua runs, which sections run and when, and what text the run hands back when it finishes.

## How a run proceeds

After this section you can list the stages of a run in order and tell which calls in your Lua ask the Harness for work.

````markdown
---
name: hello
description: Returns a fixed greeting.
runvil: 0
---

# Hello

## Main

```lua
return 'hello'
```
````

````text
hello
````

This run is as short as a run gets. The start checks find nothing missing, because the frontmatter declares no Packs and no model requirements. The H1 body under `# Hello` is empty, so there is no H1 pass. The walk enters `## Main`, the block returns `hello`, and the run ends in success with `hello` as its run result. A block that only returns a literal value asks the Harness for no work at all, so this run finishes without waiting on anything outside the prompt.

### The stages in order

Every run goes through these stages, in this order:

1. Parse. The whole file is parsed and every Lua block in the H1 body and the sections is compiled, and nothing runs yet. See [Parsing comes first](#parsing-comes-first).
2. Prepare. The Harness stages the declared input file, readies the run's Packs, and builds the run. See [Preparing the run](#preparing-the-run).
3. The start checks. The run's first step checks the `runvil` version, the store, and the requirements, and refuses a run that fails one of them. See [Preparing the run](#preparing-the-run).
4. The H1 pass. The H1 body's blocks run before the first section, but only when the H1 body holds a `lua` block or prose. See [The H1 pass](#the-h1-pass).
5. The section walk. The top-level sections run in file order. See [The section walk](#the-section-walk).
6. The result. The run ends with its text or with the error that ended it. See [The run result](#the-run-result).

The Harness parses the prompt before the run exists, so a file that does not parse starts no run and never reaches the checks of Packs and model roles. Once the prompt parses, the Harness draws the run's seed and start time, begins the run's record, and looks up the Host's current model. A prompt whose H1 body holds no `lua` block and no prose skips the H1 pass and goes straight to its first section, as `hello` does. A run that completes ends with its final text, and any other run ends with the error that ended it. A cancel is one of those errors: a run the Host cancels ends with the error whose code is `canceled`.

### A run can be refused before it starts

The start checks run before any of your Lua, so a prompt the environment cannot satisfy fails before its first section runs. This prompt requires a large context window for its `strong` model role:

````markdown
---
name: needs-context
description: Needs a model with a large context window.
runvil: 0
models:
  strong:
    min_context: 200000
---

# Needs context

## Main

```lua
return 'done'
```
````

Run on a Host whose model for the `strong` role has a 32000-token context window, this prompt never returns `done`. The run fails with error code `requirements_unmet`, and its message is this requirements notice, which ends by pointing at the `models:` line:

````text
the environment cannot satisfy this prompt:
- role 'strong': requires a context of at least 200000 tokens; the model filling it provides 32000 (at needs-context.md:5)
````

The `strong` role is one of the five model roles every prompt has (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)), and `min_context` is one of the two keys a role can declare (see [Declaring roles under `models:`](10-models.md#declaring-roles-under-models)). [Model roles at prepare](#model-roles-at-prepare) shows how the run checks the role, and [The requirements notice](#the-requirements-notice) explains each line of the message.

### What a run starts with

Prepare also sets up the run so your Lua needs no setup of its own. Every run starts with its own store, separate from every other run's store (see [What the store is](09-the-store.md#what-the-store-is)):

````markdown
---
name: handoff
description: Writes a file in one section and reads it in the next.
runvil: 0
---

# Handoff

## First

```lua
store.write('default.txt', 'stock')
```

## Second

```lua
return store.read('default.txt')
```
````

````text
stock
````

Neither section creates or passes a store handle: `store` is ready when the first block runs, and `## Second` reads back what `## First` wrote. Because each run has its own store, two runs that both write `paper.md` each read back their own copy. On a Host that supplies a model, the five model roles are likewise already filled when the first block runs, so `models.use('strong')` reaches the model directly, and a `models.infer` call before any `models.use` runs on the `default` role.

### The work the Harness does

Some calls in your Lua need something outside the prompt, and the Harness performs that work for the run (see [Effects and Harness work](01-what-a-prompt-is.md#effects-and-harness-work)). There are five kinds of Harness work:

- a model round
- a render of the tools a section entry offers, in which each offered Pack decides how its tools reach the model, which comes before the entry's first loop round when its offers name a catalog tool, and shows in the run's record as one more step ahead of that round (see [The five kinds of Harness work](01-what-a-prompt-is.md#the-five-kinds-of-harness-work))
- a tool call
- a store operation
- the timer behind a timed wait on a task started with `tasks.spawn` (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn))

While one chain waits for Harness work, only that chain waits, and every other chain keeps running. A chain is a walk with its own entries, defined in [The section walk](#the-section-walk). So the three arms of one `tasks.fanout` call (see [Fanout in one call](14-fanout.md#fanout-in-one-call)) all send their model rounds together, without any arm waiting for another.

A wait for the operator is not Harness work. The main chain reads the operator's messages with `tasks.receive()`, which waits on the chain's inbox and asks the Harness for nothing, and while it waits, every other chain keeps running too (see [Asking the operator](05-lua-environment.md#asking-the-operator)).

A call that needs Harness work looks synchronous to you: the block pauses at the call and continues with the call's answer, as if the call had returned it directly. Each Lua block runs as its own Lua coroutine, which is what lets it pause at the call and resume later. A block that asks for no Harness work, such as `return 'hello'`, never pauses.

## Parsing comes first

After this section you can tell what parsing checks, in what order it compiles your Lua, and what a parse failure stops.

````markdown
---
name: t
description: Holds one Lua block for the parse to compile.
runvil: 0
---

# T

```lua
local secret = 42
```

## S

p
````

Parsing this prompt compiles one Lua block, the H1 block's `local secret = 42`, and runs none of it. Parsing builds the prompt's tree of H1, sections, and blocks without running any of its Lua or prose. Section `S` holds only prose, so it gives the parse nothing to compile.

### The whole file parses before anything runs

The whole prompt file is parsed, and every Lua block in its H1 body and sections compiled, before any part of the prompt runs. A prompt with a parse error never starts. An invalid prompt file is rejected as a whole, never accepted in part, so none of its sections run. Parsing needs no network access and no connection to a model, so you can check a prompt's structure offline.

Parsing checks only the shape of the frontmatter's contract keys and what the file's text shows: keys, names, duplicates, ranges, and default types. A prompt whose contract has a valid shape parses even when the environment that runs it cannot satisfy it. Matching the Packs under `uses:` and the requirements under `models:` against the environment happens on the run's first step, and the tool catalog and the model reach the prompt only when the Harness prepares the run. The tool catalog holds the tools of the run's Packs that can serve it, every `runvil` Pack and each third-party Pack the prompt declares (see [The Engine, the Harness, and the Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)). A prompt that declares Pack `web` in an environment without it parses first, and then the requirements notice reports the gap:

````text
the environment cannot satisfy this prompt:
- missing required Pack: web
````

Each check stops the prompt before the next one starts, so the order of failures is fixed. A parse failure is reported ahead of any failed model lookup, a failed model lookup ahead of any input failure, and an input failure ahead of the start checks, which check the version, then the store, then the requirements.

### Lua blocks compile in source order

Compiling is part of parsing: the parse compiles every Lua block, without running it, in source order. The H1 body's blocks come first, then each section's blocks in the order the file holds them. A section with a prologue and an epilog (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)) holds two blocks, and its prologue compiles first. Replace `## S` in the prompt above with this section:

````markdown
## S

```lua
local before = 1
```

p

```lua
local after = 2
```
````

The parse now compiles three blocks: the H1 block, then the prologue of `S`, then its epilog. A prose-only section, like the first version of `S`, has no block to compile.

A Lua block that fails to compile fails the whole parse with the code `lua_compile`. The parse stops at that block: the blocks before it compiled, and the blocks after it are never compiled. The compile error names the failing block by its section and its place in it, such as ``section `Private section` prologue``, or the epilog when the epilog fails.

A prompt that breaks a Lua fence rule (see [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors)) fails its parse before any Lua compiles, even when the H1 body holds Lua.

### A prompt that does not parse starts no run

Parsing happens before the run exists. A prompt that does not parse starts no run: none of its Lua or prose runs, no model is asked for anything, and no run record is made. The Host gets the parse error and reports it. Its code is a parse code such as `fence_invalid`, or `lua_compile` when a Lua block does not compile. Its message is the parse error's full cause chain, followed by where the parse failed (see [Where a run error's message points](16-limits-and-errors.md#where-a-run-errors-message-points)).

Parsing reports no run events. A run's events begin with `run_started`, which the run reports once it passes its start checks (see [Run events and records](#run-events-and-records)). A run that stops before then reports no events, so its record holds none, only the run's own values and the error that ended it. That is the case when a cancel comes before the run or during the model lookup, when the model lookup fails, when the input is refused, and when a start check fails on the run's first step.

## Preparing the run

After this section you can say what prepare sets up and what the start checks look at before any Lua runs, and why a run is refused with `requirements_unmet` when the environment falls short.

````markdown
---
name: no-requirements
description: Declares no Packs and no model requirements.
runvil: 0
---

# No requirements

## Main

```lua
return 'no Packs'
```
````

````text
no Packs
````

Prepare is the stage before a run starts in which the Harness sets up everything the frontmatter asks for and builds the run. The run's first step then makes the start checks, which make sure the environment provides what the prompt needs. This prompt declares no Packs under `uses:` and has no `models:` key, so the requirements check finds no gaps and never refuses it. A prompt like this one, which also calls no model, runs to completion on a Host with no Packs and no model configured.

### What the start checks look at

The run's first step makes three start checks before any section runs, in this order: the `runvil` version, the store, which the run must declare and whose backend must pass its probe, and the requirements. The requirements check compares the Packs the prompt declares with the Packs the Harness took for the run, and compares the requirements the prompt declares for its model roles with the models that fill those roles. A start check that fails ends the run on that first step, and the run reports no run events.

A prompt the environment cannot satisfy is refused before any of its blocks run. `requirements_unmet` is the error code of that refusal: the run fails with that error, and its message is the requirements notice, which lists what is missing, one line per gap, and which ends by pointing at a frontmatter line when a gap is one the prompt caused. The requirements check refuses the run with `requirements_unmet` when any of these holds:

- the Host has no Pack, from any vendor, for a bare entry the prompt declares
- a declared Pack is unavailable, because the Host has it but it cannot serve, for example because it failed to build or because it is the `user` Pack in a run with no user, or because the Host lacks a declared `vendor/name` entry, or has a bare entry's Pack id only from other vendors
- a tool in the run's catalog or a Pack in its roster has a Pack id the Engine reserves, `adhoc` or `dynamic`
- the model that fills a role falls short of that role's declared context minimum or `thinking` requirement
- the run's role map has entries but none for the `default` role

An unavailable Pack is one the Host has that cannot serve the run, a declared `vendor/name` entry the Host lacks, or a bare entry whose reason tells you which vendor's Pack to write instead. The run proceeds only when every model role requirement is met, the Host has every required Pack, each one can serve, and nothing in the run takes a reserved Pack id. [Packs at prepare](#packs-at-prepare) shows each Pack case.

The refusal comes on the run's first step, after the version and store checks pass. So when the environment cannot provide what the prompt requires, such as a model with a large enough context window for one of its roles, no section runs, no Lua runs, and nothing in the prompt can catch the refusal. The refusal's message is the notice text, line for line, followed by ` (at {file}:{line})` when a gap is one the prompt caused (see [The requirements notice](#the-requirements-notice)). The notice opens with the line `the environment cannot satisfy this prompt:` and then gives one `- ` line per gap, in this order, with each `{...}` standing for the value the line names:

````text
the environment cannot satisfy this prompt:
- missing required Pack: {id}
- {pack} is unavailable: {reason}
- the run's tool catalog or Pack roster names the Pack id {id}, which the Engine reserves
- role '{role}': requires a context of at least {required} tokens; the model filling it provides {actual}
- role '{role}': requires '{required}'; the thinking capability of the model filling it is {actual}
- role 'default': requires a model; the caller's role map has no 'default' entry
````

A real notice holds a line only for each gap the run has. [The requirements notice](#the-requirements-notice) explains every line.

### The order before the first section

Before any section runs, the Harness and the run work in a fixed order:

1. The Harness parses the prompt. A prompt that does not parse starts no run.
2. It draws the run's seed and start time and begins the run's record (see [The Engine, the Harness, and the Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)).
3. It looks up the Host's current model.
4. It stages the input file the prompt declares.
5. It picks the run's Packs from the prompt's `uses:` entries.
6. It waits for the run's Packs to be ready.
7. It takes the snapshot of the run's Packs.
8. It builds the run with a role map that holds the Host's current model under `default`, which fills the five model roles.
9. The run's first step checks the `runvil` version, then the store, then the requirements, and refuses the run when one of them fails.

The model lookup lists the Host's models and finds the one the Host selected, or takes the first model listed when the Host selected none. When the list cannot be fetched, the run fails with the model code the listing failed with, such as `transport`, and when the list does not hold the selected model, the run fails with `requirements_unmet`. The message is the lookup error with its chain of causes, such as ``the selected model `{id}` is absent from the fetched catalog``. A cancel that comes before the run or during the lookup ends the run with the canceled error. A run that ends here never started, so its record holds no events.

The version check is the first start check (see [The `runvil` version key](02-file-structure.md#the-runvil-version-key)). Version `0` proceeds, and any other version is refused as unsupported, with a message that ends with ` (at {file}:{line})`, the line of the `runvil:` key. A missing version fails with the message `not a runvil prompt: no runvil version (prompt {name})`, where `{name}` is the prompt's frontmatter `name`. Either way, a prompt with an unsupported or missing version produces no section result. The run then checks its store: a run that declares no store fails with the store code `no_store`, and a run whose store backend fails its probe fails with the store code of that failure. The requirements check comes last. A run that a start check refuses reports no run events, so its record holds no events. When the run's cancel flag is set as a start check fails, the run ends with the canceled error instead of the check's own error.

### Staging the input file

Input staging happens during prepare, before the run starts. The launch's input text is placed at the path the frontmatter's `input:` key names (see [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations)), so the first section's Lua can already read the input file. Prepare fails the run before it starts in three cases:

- The launch supplies input text, but the prompt declares no `input:` file. The error code is `requirements_unmet`, and the message is ``the launch supplied input text, but the prompt declares no `input:` file``.
- The prompt declares an `input:` file that the launch did not supply and the store does not already hold. The error code is `requirements_unmet`, and the message is ``the prompt declares the input file `{path}`, but the launch supplied no input text and the store has no such file``, with the declared path in place of `{path}`.
- The store refuses the write that stages the input, or the check that the file is there. The error code is the store code of the refusal, such as `approval_required`, and the message is ``the input file `{path}` could not be staged``, followed by `: ` and the store's own error.

### Waiting for Packs and taking the snapshot

Before a run starts, the Harness picks the run's Packs: each Pack the prompt declares under `uses:`, plus every `runvil` Pack whose Pack id no declared Pack takes. It then waits until each of those Packs is ready or the run is canceled, so a Pack outside the run, such as an undeclared third-party Pack, never delays it. A Pack whose readiness check fails is unavailable to the run, with its error as the reason, so a prompt that declares it is refused on the run's first step. Canceling a run while Packs are still starting ends the wait, and each Pack is taken as it stands at that moment. A refusal that follows the cancel, such as one for a declared Pack that was still starting, gives way to it: the run ends with the canceled error rather than with `requirements_unmet`.

After the wait, each run takes one snapshot of its Packs. That snapshot supplies the tool catalog the run offers, the object for each Pack in the run's Lua (see [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks)), and the preludes that go to every section entry's Lua (see [Pack preludes](12-tools.md#pack-preludes)).

### Binding model roles

Every prompt has five model roles: `default`, `strong`, `small`, `fast`, and `creative`. When the Harness builds the run in prepare, it gives the run a role map from roles to models, and that map holds the Host's current model under `default`. A role with no entry of its own uses the `default` entry, so that one model fills all five roles. The `models:` key only attaches requirements to the roles, and the run's first step checks each declared context minimum and `thinking` requirement against the model that fills the role. A role requiring `thinking: true` fails on a model whose thinking capability is `Never`, and a role requiring `thinking: false` fails on a model whose thinking capability is `Always` (see [The thinking requirement](10-models.md#the-thinking-requirement)). When the Host has no current model, the map is empty, no role is filled, and nothing is checked. [Model roles at prepare](#model-roles-at-prepare) shows each case.

### An H1 gate failure reads like a refusal

One failure that is not a refusal reports the same error code. An uncaught Lua runtime error, code `lua_runtime`, in the prompt's H1 blocks, such as a failed `assert` or an error raised by a library that an H1 block loads with `require` (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)), is the prompt's failed hard gate. It ends the run before any later H1 block or any section runs, with error code `requirements_unmet` and a message made of the line `the environment cannot satisfy this prompt:`, the line `- the H1 gate failed`, then `: ` and the Lua error's own text, and last ` (at {file}:{line})`, the line of the failure that tripped the gate. [The H1 hard gate](#the-h1-hard-gate) shows an example and every gate case.

## Packs at prepare

After this section you can predict how the run treats each Pack you declare, what an unavailable Pack does to a run, and what the tool catalog holds once the run starts.

````markdown
---
name: needs-web
description: Declares the web Pack.
runvil: 0
uses: [web]
---

# Needs web

## Main

```lua
return 'done'
```
````

On a Host that has `runvil/web` and where it can serve, the start checks pass and the run returns `done`. On a Host with no Packs, the run is refused on its first step, before any section runs, with error code `requirements_unmet` and this notice, which ends by pointing at the `uses:` line:

````text
the environment cannot satisfy this prompt:
- missing required Pack: web (at needs-web.md:5)
````

The refused run's record holds no events, because the run was refused before it started.

### Declared Packs are required Packs

Every Pack a prompt lists under the `uses:` key (see [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key)) is a required Pack: the run does not start unless the Host can provide it. Each one is named by its Pack id and kept in declaration order. Before any Lua runs, the run's first step checks each declared Pack and refuses the run unless the Host has that Pack and it can serve.

A declared Pack counts as present when the Host has it and it can serve, even if it offers no tools, so declaring a Pack that has no tools never refuses the run. A Pack the frontmatter declares, and any Pack listed as able to serve, still has a Pack object in Lua when it offers no tool. The Harness's part is the run's roster, its list of Packs, which marks each Pack as usable or as unavailable with a reason. The Engine checks each declared Pack against that roster, not against the tool catalog, so it finds nothing missing in `uses: [web]` even when the catalog holds no `web` tool.

Each declared Pack's prelude runs in every section entry's VM, in the order the prompt declares the Packs, after the Engine globals are installed and before any block runs. An undeclared Pack's prelude never runs, even when the prompt calls that Pack's tools.

A prompt that declares no Packs has no Pack requirements, so it prepares and runs even on a Host with no Packs, as `no-requirements` does in [Preparing the run](#preparing-the-run).

### The tool catalog

The run's tool catalog holds the tools of each of the run's Packs that can serve it: every `runvil` Pack, whether the prompt declares it or not, and each third-party Pack the prompt declares. A third-party Pack the prompt does not declare adds no tools. A run holds one Pack per Pack id, so a declared third-party Pack replaces the `runvil` Pack of the same Pack id: with `acme/web` declared, the catalog holds the tools of `acme/web` and none of `runvil/web`. A Pack that cannot serve adds no tools. Each usable Pack's tool list is read once, after every Pack is ready, so the catalog and the tools a call can reach stay fixed for the whole run.

A section can call an undeclared `runvil` Pack's tool by its full tool id. On a Host that has `runvil/web`, a prompt with no `uses:` key can run this block:

````lua
local page = tools.call('web/fetch', { url = 'https://example.com' })
````

The call reaches `web/fetch` even though the frontmatter declares no Pack (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls)). The prelude of an undeclared Pack never installs, even when the Pack is in the run, so only a Pack the prompt declares can add globals to its Lua (see [Pack preludes](12-tools.md#pack-preludes)).

In Lua, `tools.packs.get(id).required` is `true` for a Pack the frontmatter declares, and `tools.packs.extras()` lists the Pack objects of every other usable Pack in the run, each listing its Tool objects in `tools` (see [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks)). With `uses: [web]`, in a run with a user, on a Host whose only Pack besides the built-ins is `runvil/web`, `tools.packs.get('web').required` is `true` and that Pack's `tools` holds the Tool objects for `web/fetch` and `web/search`, while `tools.packs.extras()` holds the `subagents` and `user` Pack objects, in that order, and neither object's `tools` holds any Tool object. In a run with no user, `tools.packs.extras()` holds only the `subagents` object, because the `user` Pack is unavailable there and an unavailable Pack has no Pack object.

A section can offer only the tools in the run's catalog and the adhoc tools it defines, and offering an id the run cannot offer is a hard error (see [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer)). Calling `tools.offer` with a tool id the run lacks raises `tools.offer: "{name}" is not a tool in this run`, with the id in place of `{name}`.

### Runs with no user

The built-in Pack `runvil/user`, whose Pack id is `user`, marks a prompt that needs a person to answer it (see [Asking the operator](05-lua-environment.md#asking-the-operator)). Every Host has it, and it offers no tools, but a run holds it only when the run has a user, someone the Host lets answer the run's main chain. The Host decides that for each run it launches.

In a run with no user, the `user` Pack is unavailable, with the reason `this run has no user`. A prompt that declares it is refused on the run's first step with error code `requirements_unmet`, and the notice names the Pack and the reason:

````text
the environment cannot satisfy this prompt:
- user is unavailable: this run has no user
````

That notice does not contain `missing required Pack: user`, because every Host has the Pack. A run refused this way runs none of the prompt's Lua and records zero effects.

Whether a declared `user` is met depends only on the run. The same prompt text runs to completion when the Host launches it with a user and is refused in a run without one. A prompt that leaves `user` undeclared is never refused for it, and it can check `tools.packs.get('user')`, which reads `nil` in a run with no user, before it waits for the operator.

### Unavailable Packs

A declared Pack that the Host has but that cannot serve refuses the run with error code `requirements_unmet` and the notice line `- {pack} is unavailable: {reason}`. A Pack cannot serve in four cases:

- Building it failed when the Host set up. That is not a setup error: the Host stores the Pack as unavailable with the failure as its reason, and every run that needs it is refused naming that reason.
- Its factory refuses or panics when a run makes its own instance of the Pack. Only that run holds the Pack as unavailable, with the factory's refusal as its reason, or `the Pack's factory panicked` after a panic, and the next run calls the factory again.
- Its readiness check fails before the run.
- It is the built-in `user` Pack, and the run has no user (see [Runs with no user](#runs-with-no-user)).

The `web` Pack cannot be built on a Host that supplies no search provider, so a prompt that declares `web` there is refused with the notice line `- web is unavailable: web needs runvil/search-provider, and this host provides none`. A prompt that declares `user` in a run with no user is refused with this notice line:

````text
- user is unavailable: this run has no user
````

Two declared entries the Host has no Pack for are reported as unavailable too, because their reason tells you what to write. A bare Pack id the Host has only from other vendors gets a line such as `- poop is unavailable: poop means runvil/poop, which is not installed; write acme/poop to use that Pack`, which joins every such vendor's fully qualified name with `or`. A `vendor/name` the Host does not have gets a line such as `- poop is unavailable: acme/poop is not installed`.

When one of the Host's Packs cannot serve because building it failed when the Host set up, its factory refused or panicked, or its readiness check failed, the Harness writes a line naming the Pack to the Harness's own log, which is separate from the run's events. A Pack the Host set up as unavailable logs the warning `Pack was inserted unavailable` once, when the Host sets up. A factory that refuses a run logs `Pack factory refused this run; the Pack is unavailable to it` at debug level, a factory that panics logs the error `Pack factory panicked; the Pack is unavailable to this run`, and a readiness failure logs the warning `Pack will not be ready; this run treats it as unavailable`.

### Only declared Packs can refuse a run

The requirements check looks only at the Packs the prompt declares, so only declared Packs can refuse a run. A `runvil` Pack the prompt does not declare that fails to start just has no tools in the run, and the run goes on. The check looks up each Pack the prompt declares, while the run's roster, its list of Packs, holds every Pack in the run, declared or not. A third-party Pack the prompt does not declare is in no run, so it can neither refuse nor delay one. An undeclared `user` Pack in a run with no user joins the roster as unavailable, with the reason `this run has no user`, and the run goes on.

Offering an unavailable Pack with `tools.offer` raises `tools.offer: the {pack} Pack is unavailable in this run: {reason}`. So an undeclared Pack whose readiness check fails offers no tools, and a section that offers it with `tools.offer` fails the run with that text in its message.

## Model roles at prepare

After this section you can predict which model fills each of the five model roles when prepare builds the run, which checks the run makes against the models that fill the roles you declare, and what an unmet requirement looks like.

````markdown
---
name: roles
description: Declares requirements for two model roles.
runvil: 0
models:
  fast: {}
  strong:
    min_context: 200000
---

# Roles

## Main

```lua
return 'done'
```
````

On a Host whose only model has a context window of exactly 200000 tokens, the run returns:

````text
done
````

When prepare builds the run, the Engine fills the five model roles from the role map the run is given, one binding per role, so each role is ready the first time Lua uses it. A role with an entry of its own in the map uses that entry's model, and a role with no entry uses the `default` entry, so a map that holds only a `default` entry fills all five roles with that one model. The Harness gives the run a map with only a `default` entry, the Host's current model, or an empty map when the Host has no model. Every prompt gets all five roles, whether or not it declares them, and `models:` only attaches requirements to them. This prompt attaches requirements to two roles. The `fast` role, declared as `fast: {}`, adds no requirement. The `strong` role needs a window of at least 200000 tokens, and a model whose context window equals a role's `min_context` meets it; only a smaller window falls short. A role with `min_context: 200000` therefore runs on a model whose window is exactly 200000 tokens.

The run checks a role's requirements against the model that fills that role. With a one-model map, every role is filled by that model, so `fast` and `strong` both read the same model here. A map that gives `strong` an entry of its own, which only a caller that drives the Engine directly passes, checks the `strong` requirement against that model and checks the other roles against the `default` entry's model. A name other than the five roles, or a key other than `min_context` and `thinking`, fails the parse (see [Mistakes inside `models:`](10-models.md#mistakes-inside-models)).

A prompt whose only declaration is a role with no requirements, `models: { fast: {} }`, passes the requirements check when the Host supplies a model. A prompt with no `models:` key declares no requirements, so no model requirement can stop it; a prompt whose frontmatter declares only `uses: [web]` starts with no model requirement to meet. When the Host gives no model, the map is empty and no role is filled, so the run checks no role requirement and reports no model requirement. A prompt that never calls a model still runs to completion on such a Host, and a model round fails with `model binding required for section {section}` (see [When the Host gives no model](10-models.md#when-the-host-gives-no-model)).

### The model role check

The model role check is the comparison the run makes when it is built, between each declared role requirement and the model that fills the role, and the run's first step reports any shortfall it finds. It checks the role's `min_context` and its `thinking` requirement, `thinking: true` or `thinking: false`, and it reports the context minimum and the thinking requirement separately.

````markdown
---
name: analysis
description: Needs a large context window and a thinking model.
runvil: 0
models:
  strong:
    min_context: 200000
    thinking: true
---

# Analysis

## Main

```lua
return 'done'
```
````

When the model that fills `strong` has a 32000-token window and never thinks, the role falls short on both checks, and the run is refused on its first step, before any section runs. The run fails with error code `requirements_unmet`, and the notice opens with `the environment cannot satisfy this prompt:`, followed by one line per shortfall, and ends by pointing at the `models:` line:

````text
the environment cannot satisfy this prompt:
- role 'strong': requires a context of at least 200000 tokens; the model filling it provides 32000
- role 'strong': requires 'thinking: true'; the thinking capability of the model filling it is Never (at analysis.md:5)
````

The notice reports each failed check on its own line. A role that meets its context minimum but fails its `thinking` requirement gets exactly one line, and a role that fails both its context minimum and its `thinking` requirement gets two lines, as `strong` does here.

Whether a `thinking` requirement is met depends on the thinking capability of the model that fills the role:

| Requirement | `Never` | `Switchable` | `Always` |
|---|---|---|---|
| `thinking: true` | falls short | met | met |
| `thinking: false` | met | met, with thinking turned off | falls short |

`thinking: true` fails only on a model that never thinks, and `thinking: false` fails only on a model that always thinks, so a model whose thinking is switchable meets both. A `thinking: false` role on a switchable model prepares and runs, with thinking turned off on its requests, and a `thinking: true` role turns thinking on for its requests (see [The thinking switch](10-models.md#the-thinking-switch)).

The run checks only `min_context` and `thinking`, the two keys a role can declare. A role that declares neither, such as `fast: {}`, never causes a shortfall, even on a 32000-token model that never thinks.

### The two role lines

A context window below a role's `min_context` shows as this line, with both numbers as plain token counts:

````text
- role '{role}': requires a context of at least {required} tokens; the model filling it provides {actual}
````

A failed `thinking` requirement shows as this line, where `{required}` is `thinking: true` or `thinking: false`, and `{actual}` is `Never` when `thinking: true` fails and `Always` when `thinking: false` fails:

````text
- role '{role}': requires '{required}'; the thinking capability of the model filling it is {actual}
````

So a `small` role with `thinking: false` on a model that always thinks gives `- role 'small': requires 'thinking: false'; the thinking capability of the model filling it is Always`. Each shortfall line in the notice starts with `- `, so for a role `strong` that needs 200000 tokens on a 32000-token model, the whole notice is the two lines shown in [How a run proceeds](#how-a-run-proceeds).

### A role map without a default entry

A role map with entries but none for `default` leaves the run no model to fall back on. The run is refused on its first step, no declared role requirement is checked, and the notice holds one model line, whatever role requirements the prompt declares:

````text
the environment cannot satisfy this prompt:
- role 'default': requires a model; the caller's role map has no 'default' entry
````

An empty map is not this case. It means the Host has no model, and the run checks nothing. The Harness never passes a map without `default`, because its map holds the Host's one model under `default` or nothing at all, so only a caller that drives the Engine directly meets this refusal.

### Unmet requirements are recorded, not raised

An unmet requirement is one shortfall the model role check finds, recorded when the run is built and reported on its first step rather than raised as an error. Because of that, a model that cannot serve a role never surfaces as an error partway through a run. A role `small` that needs `thinking: false` on a model that always thinks gives exactly one unmet requirement, and the run is refused before any section has run.

The check never repairs. A role that fails a `thinking` or context check stays filled by the same model, and nothing looks for a different model to serve it.

## The requirements notice

After this section you can read any requirements notice line by line and tell what produced each line.

````markdown
---
name: deep-analysis
description: Needs two Packs and a large thinking model.
runvil: 0
uses: [web, user]
models:
  strong:
    min_context: 200000
    thinking: true
---

# Deep analysis

## Main

```lua
return 'done'
```
````

Run this prompt in a run with no user, on a Host that has no `web` Pack from any vendor and whose model for `strong` has a 32000-token window and never thinks. The run fails with error code `requirements_unmet`, and its message is this requirements notice, which ends by pointing at the `uses:` line:

````text
the environment cannot satisfy this prompt:
- missing required Pack: web
- user is unavailable: this run has no user
- role 'strong': requires a context of at least 200000 tokens; the model filling it provides 32000
- role 'strong': requires 'thinking: true'; the thinking capability of the model filling it is Never (at deep-analysis.md:5)
````

The requirements notice is the text the run's first step reports when the requirements check refuses the run. When a prompt's requirements are unmet, the run is refused before any section or Lua runs, so nothing in the prompt can catch the refusal. Any unmet requirement, including an unmet model role requirement, makes the refusal a run error with code `requirements_unmet` whose message is the notice, word for word, followed by ` (at {file}:{line})` when a gap is one the prompt caused. When nothing blocks the run, there is no refusal.

The location names the frontmatter key of the first gap the prompt caused: the `uses:` line when a declared Pack is missing or unavailable, as here, and otherwise the `models:` line when a declared role requirement falls short. A reserved Pack id and a role map with no `default` entry are the caller's gaps, so a refusal made only of those points at no line (see [Where a run error's message points](16-limits-and-errors.md#where-a-run-errors-message-points)).

### The notice's fixed layout

The notice has a fixed layout: the opening line `the environment cannot satisfy this prompt:`, then one line starting with `- ` for each unmet requirement. The check does not stop at the first gap, so when several requirements are unmet you see all of them together in one notice, not only the first. Each line names what is missing or falls short, such as each unavailable Pack and the reason it cannot serve, and states the required value against the actual value where that applies.

The notice is written for a model to read: concise, factual, and self-contained. It names only the gaps that block the run, because the check records only what needs a person's attention. Each kind of line comes from one kind of gap:

| Notice line | What produces it |
|---|---|
| `- missing required Pack: {id}` | a bare entry whose Pack id no Pack of the Host has, from any vendor |
| `- {pack} is unavailable: {reason}` | a declared Pack that the Host has but that cannot serve, a declared `vendor/name` entry the Host lacks, or a bare entry whose Pack id only other vendors' Packs have |
| `- the run's tool catalog or Pack roster names the Pack id {id}, which the Engine reserves` | a catalog tool or a roster Pack under a reserved Pack id, `adhoc` or `dynamic` |
| `- role '{role}': requires a context of at least {required} tokens; the model filling it provides {actual}` | a role whose `min_context` is above the window of the model filling it |
| `- role '{role}': requires '{required}'; the thinking capability of the model filling it is {actual}` | a role whose `thinking` requirement the model filling it fails |
| `- role 'default': requires a model; the caller's role map has no 'default' entry` | a role map that has entries but none for `default` |
| `- the H1 gate failed` | an uncaught `lua_runtime` error in the H1 pass |

### Pack lines in the notice

A bare entry under `uses:` whose Pack id no Pack of the Host has, from any vendor, refuses the run with the line `- missing required Pack: {id}`, such as `- missing required Pack: web`. A `vendor/name` entry never gives this line.

A declared Pack that the Host has but that cannot serve, for example because it failed to build, appears as `- {pack} is unavailable: {reason}`, with the Pack id and the reason the environment gave, word for word. The reason is written for the model that reads the refusal, as in `- web is unavailable: web needs runvil/search-provider, and this host provides none`, or `- user is unavailable: this run has no user` for the `user` Pack declared in a run with no user.

A declared entry the Host has no Pack for appears the same way, unless it is a bare entry that no vendor has. A bare Pack id that only other vendors' Packs have gives `- poop is unavailable: poop means runvil/poop, which is not installed; write acme/poop to use that Pack`, and a `vendor/name` the Host lacks gives `- poop is unavailable: acme/poop is not installed`, whether or not any vendor has that Pack id.

The Engine reserves two Pack ids, `adhoc` for a section entry's own tools (see [Adhoc tools with `tools.adhoc`](12-tools.md#adhoc-tools-with-toolsadhoc)) and `dynamic` for its discovery tools (see [Finding and calling dynamic tools](12-tools.md#finding-and-calling-dynamic-tools)), so a run whose tool catalog holds a tool under a reserved Pack id, or whose roster holds a Pack with one, usable or not, is refused with the line `- the run's tool catalog or Pack roster names the Pack id {id}, which the Engine reserves`, such as `- the run's tool catalog or Pack roster names the Pack id adhoc, which the Engine reserves`. Each reserved Pack id the run names gets one line, however many such tools or Packs the run has, and a run that names both gets the `adhoc` line first.

Each Pack appears under its real cause only. A Pack that the Host has but that fails to build or activate, and the `user` Pack in a run with no user, are reported only as unavailable, with their reasons, never as missing required Packs. So when a declared Pack is unavailable, the notice leaves out its `missing required Pack` line and names only the real cause.

### Role lines in the notice

Each unmet model role requirement gets its own line under the opening line, starting with `- role '{role}':` and naming the role by its name, one of `default`, `strong`, `small`, `fast`, and `creative`. The notice names every failed check across all roles, each on its own line with required against actual, so a role that fails both its context check and its `thinking` check gets two lines, as `strong` does above.

When the context window of the model filling a role is smaller than the role's `min_context`, set at `models.{role}.min_context`, the line is `- role '{role}': requires a context of at least {required} tokens; the model filling it provides {actual}`. The comparison is strict, so a model whose window equals the minimum passes.

An unmet `thinking` requirement gives the line `- role '{role}': requires '{required}'; the thinking capability of the model filling it is {actual}`, where `{required}` is `thinking: true` or `thinking: false`. `{actual}` reads `Never` for an unmet `thinking: true` and `Always` for an unmet `thinking: false`, capitalized as shown, and a `Switchable` model meets both. A role with `thinking: false` on a model that always thinks gives `- role '{role}': requires 'thinking: false'; the thinking capability of the model filling it is Always`.

Only `min_context` and `thinking` can add these role lines, and the one other model line is the `default` line that [A role map without a default entry](#a-role-map-without-a-default-entry) describes. The notice lists only the roles that fall short and leaves out roles the model meets. When the Host gives no model, the notice has no role lines, because the run skips the role checks.

### The H1 gate line

The `- the H1 gate failed` line does not come from a refusal. An uncaught `lua_runtime` error in the H1 pass fails the prompt's hard gate, which also ends the run with error code `requirements_unmet` before any later block runs. Its notice is the same opening line followed by `- the H1 gate failed`, and its message adds `: ` and the Lua error's text, then ` (at {file}:{line})`, the line of the failure that tripped the gate, as [The H1 hard gate](#the-h1-hard-gate) shows.

### The order of the lines

Notice lines appear in a fixed order: missing Packs first, then unavailable Packs, then the lines for reserved Pack ids, in sorted order, then unmet model role requirements. Missing required Packs come before unavailable Packs, no matter which gap was found first. Missing and unavailable Packs each keep the order in which the prompt declares them. Role lines are listed in role order, `default`, `strong`, `small`, `fast`, `creative`, and within a role the context line comes first, then the `thinking` line.

Each gap appears once. The check looks up each declared Pack once in the run's roster, so each declared Pack gets at most one line, and an unavailable Pack is listed with its own reason.

### How the gaps are gathered

The requirements are gathered once, when the run is built, and the run's first step reports them after the version and store checks pass. The check looks at every Pack the prompt declares, every reserved Pack id in the run's catalog and roster, and every requirement a declared model role carries, so one refusal names every gap. Unmet requirements never make prepare fail: the run is built all the same, and its first step fails with the notice. When the notice would have no lines, nothing blocks the run.

### When a cancel comes first

When the run's cancel flag is set by the time a start check fails, for example because a cancel cut short the wait while a declared Pack was still starting, no notice is reported: the cancel decides the run's result, which is the canceled error rather than `requirements_unmet`, as [Waiting for Packs and taking the snapshot](#waiting-for-packs-and-taking-the-snapshot) explains. A failed version or store check that meets a cancel ends the run with the canceled error in the same way.

## The section walk

After this section you can trace which sections a run enters, in what order, and what one section leaves behind for the next.

````markdown
---
name: walk
description: Passes values through three sections.
runvil: 0
---

# Walk

## A

```lua
var.from_a = 'a'
```

## B

```lua
assert(var.from_a == 'a')
var.from_b = 'b'
```

## C

```lua
return var.from_a .. var.from_b
```
````

````text
ab
````

The H1 body under `# Walk` is empty, so the run moves straight into the first `##` section, `## A`. Its block sets `var.from_a` and ends without `return`, so the walk falls through to `## B`. `## B` sees the value `## A` left on `var` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)), sets `var.from_b`, and falls through to `## C`. `## C` returns `ab`, and that return value becomes the run result (see [Returning values](#returning-values)).

### The main walk and the entry section

A Lua block placed under a named `##` section, such as `## Greet`, runs when the section walk reaches that section. Without `jump` or `call`, a prompt's top-level `##` sections run one after another in file order, top to bottom, each finishing before the next one starts. The main walk is this walk over the top-level sections, the walk the run itself begins. It starts at the entry section: the first top-level `##` section in file order, whatever its name.

````markdown
---
name: entry
description: Starts at whichever section comes first.
runvil: 0
---

# Entry

## Zebra

```lua
return 'zebra'
```

## Main

```lua
return 'main'
```
````

````text
zebra
````

You choose the starting section by placing it first. With `## Zebra` written before `## Main`, the walk starts at `Zebra`, and because `## Zebra` returns, `## Main` never runs. Naming a section `Main` gives it no special role, so a `## Main` written after another `##` section is not the entry section.

When the H1 title has no Lua block under it, the run moves into the first `##` section and runs its `lua` block, and that section's `return` value becomes the run result. A prompt can also have no `##` section at all (see [Section headings and nesting](02-file-structure.md#section-headings-and-nesting)): such an H1-only prompt still parses, but it has no entry section.

### Falling through to the next section

Fall-through is what happens when a section's Lua finishes without `return`: the walk or chain the section belongs to continues with the next sibling section in file order. Inside a section, the blocks run top to bottom. A section can hold several prose and Lua block pairs (see [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks)), and a block that ends without `return` lets the section continue to its next prose block and Lua block. After the last block, the section falls through with no result, and the walk moves on to whatever follows. A section that holds only prose and no Lua fence produces no result, and the walk continues to the next section.

Fall-through hands no result value to the next section, and each section entry starts fresh:

````markdown
---
name: fresh
description: Shows that plain globals end with their section.
runvil: 0
---

# Fresh

## First

```lua
reply = 'custom'
```

## Second

```lua
assert(reply == nil)
return 'ok'
```
````

````text
ok
````

Each section entry starts with its own section VM, the fresh Lua instance a section's blocks run in (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)). The move between sections tears down the old section VM and clears its state, so a plain global such as `reply`, or a function such as `greet` defined in a `## helpers` section, is `nil` in a later section. Each section entry runs in its own frame, built when the section is entered and torn down when the entry ends. What a section leaves on `var`, and what it writes with `store`, outlives the move: `## C` in the `walk` prompt reads both values set before it, and `## Second` in the `handoff` prompt reads the file `## First` wrote.

Prose that a section leaves unconsumed at its end is discarded without being evaluated, and it never reaches the next section:

````markdown
---
name: discarded
description: Leaves prose unread at the end of a section.
runvil: 0
---

# Discarded

## First

{{ var.missing }} is never read here.

## Second

```lua
return 'ok'
```
````

````text
ok
````

The `{{ var.missing }}` in `## First` is never evaluated, so it cannot fail the run, and `## First` holds only prose, so the walk continues to `## Second`.

### Child sections stay off the walk

Fall-through moves only to the next sibling and never goes down into child sections (see [Section headings and nesting](02-file-structure.md#section-headings-and-nesting)):

````markdown
---
name: children-wait
description: Shows the walk passing over a child section.
runvil: 0
---

# Children wait

## A

```lua
var.seen = 'A'
```

### Child

```lua
error('a child must not run by fall-through')
```

## B

```lua
return var.seen .. 'B'
```
````

````text
AB
````

The walk goes from `## A` straight to `## B` and passes over `### Child`, so its `error` never runs and the run succeeds. The walk never enters a child section on its own: it skips the subsections nested under a walked section and moves on to the next sibling at the same heading level. A child section runs only when something names it with a heading reference, such as `jump` (see [Jumping to a section](08-jump-and-call.md#jumping-to-a-section)), `call` (see [Calling a section](08-jump-and-call.md#calling-a-section)), or a task started with `tasks.spawn('### Arm')` (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)). A child section that holds only prose, such as `### Author note`, works as a note for people reading the file, because the walk never enters it.

### Every top-level section is on the main walk

Every top-level section is on the main walk, and no syntax takes a section out of it. The walk runs each top-level section it reaches, unless the run has already ended or a `jump` such as `jump('## C')` has moved the walk past it. A top-level section the walk reaches runs under the section rules, even when you meant it only as a library for `require`, so a top-level `## lib` that returns a table and that the walk reaches fails the run with `cannot return a table as a result`.

A section that should run only when something names it, such as a helper library, stays off the walk in either of two placements: nested under the section that uses it, or placed after a section that returns. Here `### lib` is nested under `## Main`:

````markdown
---
name: nested-lib
description: Keeps a library off the walk by nesting it.
runvil: 0
---

# Nested lib

## Main

```lua
var.name = (require '### lib').name
```

### lib

```lua
return { name = 'nested' }
```

## Last

```lua
return var.name
```
````

````text
nested
````

`## Main` loads the library and falls through, and the walk goes from `## Main` straight to `## Last`. It never reaches `### lib`, so its table return never meets the section rules, and the run succeeds with `nested`. Placing a `## lib` after a `## Main` whose block returns keeps it off the walk the same way, because the walk ends before it gets there.

### The main walk as a chain

A chain is a walk with its own entries. The main walk is a chain: a walk over sibling sections that the run itself begins. A `call` begins a chain of its own:

````markdown
---
name: sub-chain
description: Calls a child section that falls through to its sibling.
runvil: 0
---

# Sub chain

## Main

```lua
return call('### Sub')
```

### Sub

```lua
local x = 1
```

### After

```lua
return 'after-reply'
```
````

````text
after-reply
````

`call('### Sub')` walks from `### Sub`, falls through to the next sibling, `### After`, and hands that chain's final reply back to the caller. Fall-through works the same way in every chain, which is why it moves the walk or chain a section belongs to.

When a walk runs past the last section at its level with no outer level waiting to resume, the chain finishes with no result value. A main walk that ends this way gives the run the generic completion text `done`:

````markdown
---
name: fallback
description: Ends without returning a value.
runvil: 0
---

# Fallback

## Only

```lua
local x = 1
```
````

````text
done
````

[The run result](#the-run-result) explains this `done` fallback.

## Returning values

After this section you can end a Lua block with a value and predict what that value does to its section, its chain, and the run.

````markdown
---
name: say_hi
description: Returns a fixed greeting.
runvil: 0
---

# Say hi

## Greet

```lua
return 'hello'
```
````

````text
hello
````

The block ends with `return 'hello'`, so the string `hello` becomes the section's result. Because `## Greet` is on the [main walk](#the-section-walk), that return also ends the run, and `hello` is the run result.

### How a block ends

A Lua block ends in exactly one of three ways: it returns, it jumps, or it fails with an error. When a block calls `jump` and then raises an error, the jump wins:

````lua
jump('## Target')
error('unreachable')
````

This block ends as a jump to `## Target`, not as a failure.

### The scalar return rule

A `return` at the top level of a Lua block ends that block, and the returned value becomes the block's result. This is the scalar return rule: returning a string, an integer, a float, or a boolean makes that value the result, as text, and returning `nil` or ending with no `return` statement gives no result. A block that finishes normally hands back an optional scalar value, so a `return` gives either no value or a value as text.

The value a block returns becomes its section's result, and the `return` ends the section with that value. A block can return early from inside an `if`, and the first `return` it reaches gives the section's result:

````lua
local n = 3
if n > 2 then
  return 'big'
end
return 'small'
````

This block returns `big`, and the last line never runs.

Any expression that produces a scalar can be returned: a literal such as `return 'ok'` or `return 'child'`, a value such as `return var.answer` or `return prose`, or a model reply. The most common last line of a prompt is this one:

````lua
return models.infer(prose)
````

Ending the last block of a section with `return models.infer(prose)` makes the model's reply the run's result through the scalar return rule (see [`models.infer`](10-models.md#running-a-round-with-modelsinfer)). In a prompt with several sections that is usually the last section, because earlier sections that return nothing hand off to the next one. A block can return a tool's reply the same way, as in `return tools.call('web/fetch', { url = 'https://example.com' })` (see [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls)). An epilog can return a value it computes, as in `return normalize(var.subject)`, where `normalize` is a function the prompt defines.

A block's return value survives the block pausing and resuming. A block that waits on a model round inside `pcall` and resumes with the reply still returns the value it builds from that reply, intact.

### How each scalar becomes text

Each scalar type becomes text by a fixed rule, and the table shows one case of each:

| Block ends with | Result |
| --- | --- |
| `return 'text'` | `text` |
| `return 42` | `42` |
| `return 1.5` | `1.5` |
| `return 3.0` | `3` |
| `return 1e20` | `100000000000000000000` |
| `return 0/0` | `NaN` |
| `return true` | `true` |
| `return 'a', 'b'` | `a` |
| `return nil, 'text'` | no result |

A string becomes the result unchanged, except that bytes that are not valid UTF-8 become the Unicode replacement character U+FFFD. An integer becomes its decimal digits as plain text, so `return 42` gives `42` and a counter that reaches eight million gives `8000000`.

A float is formatted by the Engine rather than by Lua's `tostring`. It is written as decimal text in plain display form and never in exponent form, and a whole-number float drops its fraction. So `return 1.5` gives `1.5`, `return 3.0` gives `3` where `tostring(3.0)` gives `3.0`, and `return 1e20` gives `100000000000000000000` where `tostring` gives `1e+20`. An infinity or NaN becomes `inf`, `-inf`, or `NaN`, so `return 0/0` gives `NaN` where `tostring` gives `nan` or `-nan`. A boolean becomes `true` or `false`.

Only the first returned value counts, and the values after it are never checked, so `return 'a', 'b'` gives `a`. Because only the first value counts, a `nil` first value gives no result even when a value follows it: `return nil, 'text'` keeps the run going as if the block returned nothing.

### Returning nothing

A block with no result hands control to whatever follows it:

````markdown
---
name: hand_off
description: Shows a section with no return handing off to the next section.
runvil: 0
---

# Hand off

## First

```lua
var.greeting = 'hello'
```

## Second

```lua
return var.greeting .. ' world'
```
````

````text
hello world
````

`## First` stores a value in [`var`](05-lua-environment.md#keeping-state-in-var) and ends with no `return`, so it produces no result. A block that returns `nil` or nothing lets the section go on to the prose and blocks that follow it. When the section has nothing left, the run keeps going with the next section, where a later `return` can set the result. Here `## Second` returns, and its value is the run result.

Returning an empty string is different from returning nothing. `return ''` sets a result, so it ends the chain the way any other string does, while no result moves on to the next block.

### A return ends the chain

A scalar return ends the [chain](#the-section-walk) it runs in, and the value becomes that chain's result. On the main walk, a return from any section ends the whole run at once, so the rest of that section and every later section never run:

````markdown
---
name: first_wins
description: Shows a return ending the run before later sections.
runvil: 0
---

# First wins

## First

```lua
return 'first'
```

## Second

```lua
return 'unreached'
```
````

````text
first
````

`## First` returns, so the walk never reaches its sibling `## Second`, even though it follows in the file. On the main walk, the run's result is the value returned by the last section that runs.

The same holds for a section the walk reached through [`jump`](08-jump-and-call.md#jumping-to-a-section). Replace the sections of `first_wins` with these three:

````markdown
## A

```lua
jump('## B')
```

## B

```lua
return 'b-ran'
```

## C

```lua
return 'c-ran'
```
````

````text
b-ran
````

`## B` returns, so its value becomes the run result, and the later section `## C` never runs, so its return value never replaces it.

A section reached through [`call`](08-jump-and-call.md#calling-a-section) runs in a chain of its own, so its return ends only that chain and hands the value back to the caller as the value of `call`:

````markdown
---
name: ask_research
description: Shows a called section handing its value back to the caller.
runvil: 0
---

# Ask research

## Main

```lua
local answer = call('## Research')
return 'got: ' .. answer
```

## Research

```lua
return 'research-reply'
```
````

````text
got: research-reply
````

`## Research` runs once, when `## Main` calls it. `## Main` then returns, which ends the run, so the walk never falls through into `## Research` a second time. A section after the one that returns runs only when something targets it: `require` (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)), `call`, and [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) can still reach it, but fall-through never does. This holds for any returning section, such as one that ends with `return 'done'` or one that returns after waiting on the tasks it started.

### Returning from the prologue or the epilog

A section can hold a [prologue and an epilog](03-blocks-and-prose.md#prologue-prose-and-epilog), Lua before its prose and Lua after it, and a `return` in either one ends the section:

````markdown
---
name: stop_early
description: Returns from the prologue so the prose and epilog never run.
runvil: 0
---

# Stop early

## Finish

```lua
return 'early'
```

This prose never reaches a model.

```lua
return 'late'
```
````

````text
early
````

The prologue's `return 'early'` ends the section at once, and `early` becomes the run result. The prose after the prologue is never sent to a model, so this run needs no model at all. The epilog never runs either, so its [effects](01-what-a-prompt-is.md#effects-and-harness-work), such as store writes, never happen, and its `return 'late'` never replaces the earlier value. When the prologue does not return, the epilog runs, and a value it returns becomes the run result the same way.

### Tables and other values that cannot be returned

Only a scalar can be a result. To return data held in a table, build a string from it first, for example with `table.concat`:

````lua
local kinds = { 'alpha', 'beta' }
return table.concat(kinds, ',')
````

This block returns `alpha,beta`. For a result that should be JSON text, build the string with [`json.encode`](05-lua-environment.md#encoding-and-decoding-json-with-json) instead, so `return json.encode(kinds)` returns `["alpha","beta"]`.

Returning any other first value fails the block with `cannot return a {type} as a result`, where `{type}` is the Lua type name: `table` for a table, `function` for a function, and `userdata` for an Engine object. So a table return fails with a message that contains `cannot return a table as a result`. This is an ordinary Lua error, so in a walked section it fails the run with error code `lua_runtime`. In the H1 pass the same error fails the [hard gate](#the-h1-hard-gate) instead.

The easiest way to hit this error is with a library section that ends by returning a table for `require`. When the walk reaches that section on its own, the walk applies the scalar return rule to it, and the run fails with error code `lua_runtime` and `cannot return a table as a result`. Placing library sections after the section that returns keeps the walk from reaching them.

## The H1 pass

After this section you can run Lua before the first section and know what that early pass can change for the rest of the run.

````markdown
---
name: h1_tone
description: Sets a value in the H1 pass that the first section reads.
runvil: 0
---

# H1 tone

```lua
var.tone = 'friendly'
```

## Reply

```lua
return 'tone: ' .. var.tone
```
````

````text
tone: friendly
````

The block directly under `# H1 tone` runs first, before `## Reply` is walked. That is the H1 pass: every Lua block in the [H1 body](02-file-structure.md#the-h1-title-and-h1-body), meaning under the `#` title and before the first `##` heading, runs first as section 0, before any section runs. To run a block ahead of every section, you put it there.

The H1 pass starts with an empty `var`. Its final `var` is read back when it ends, and the first top-level section starts from that `var`, so values the H1 blocks store in `var` become the starting state of the sections walked afterward. That is how `## Reply` reads `friendly`.

### What the H1 body holds

You can put Lua fences and prose directly under the `#` title, before the first `##` heading, and they are live blocks kept in file order. The H1 body can hold several Lua blocks, even with prose between them, and they run in file order as one pass, each exactly once, before any section:

````markdown
---
name: count_blocks
description: Counts the H1 blocks that ran before the first section.
runvil: 0
---

# Count blocks

```lua
var.executions = (var.executions or 0) + 1
```

Ask for one round.

```lua
var.executions = (var.executions or 0) + 1
```

## Result

```lua
return var.executions
```
````

````text
2
````

The H1 counts as section 0 of the prompt, and its blocks run once, together, as a single section entered under the prompt's `#` title. Prose in the H1 body takes part in the run too: the line `Ask for one round.` is both a live prose block and the prompt's description text. The H1 pass reads that prose only through an explicit model round, such as `models.infer(prose)`.

### What the H1 pass can do

Lua in the H1 body can make the same Engine calls a section can, including model rounds, `require`, [`prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist), `call`, `jump`, and [`tasks.fanout`](14-fanout.md#fanout-in-one-call). It can spawn tasks with `tasks.spawn`, and it can call the store functions, such as `store.read` and `store.write`, just as section code does. Because the H1 block runs before any `##` section, a `store.write` there is the run's first effect.

The model an H1 block selects stays in force for the walk, because the main walk starts on the model the H1 pass ended on (see [Where a chain starts](10-models.md#where-a-chain-starts)). That makes the H1 the place to move every section onto another role with [`models.use`](10-models.md#selecting-a-role-with-modelsuse):

````lua
models.use('strong')
````

With this block in the H1 body, every section walked afterward starts on the `strong` [model role](10-models.md#model-roles-instead-of-models) until a section calls `models.use` again. A prompt whose H1 selects nothing starts the walk on the `default` role.

The H1 is also the place to validate and repair the prompt's parsed input in `argv` (see [Reading parsed input from `argv`](06-arguments.md#reading-parsed-input-from-argv)). An `assert` on `argv` checks it, and assigning a repaired `argv` table in the H1 body means every later section sees the repaired value.

A model round in the H1 pass reaches the model exactly once, even though sections run afterward. Model rounds happen in source order, so an H1 pass's `models.infer` call reaches the model before a later `##` section's round.

The H1 pass is its own section entry for offers. A Pack the H1 offers with [`tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer) reaches only the H1 pass's own model rounds, and walked sections start without that offer. Checkpoints that H1 Lua records with [`log`](05-lua-environment.md#logging-and-lifecycle-reports), such as `log("H1 ran")`, are attributed to the H1 title.

### Falling through to the walk

H1 code does not need to return a value. When the H1 pass finishes with neither a scalar `return` nor a `jump`, it falls through, and the main walk starts at the first top-level `##` section on its own. The H1 pass never completes the run this way, because it is never a walked section that can complete it.

`h1_tone` and `count_blocks` both fall through. So does an H1 block that stores a model reply with `var.h1 = models.infer(prose)` instead of returning it: the walk starts at the first section, whose scalar return can become the run result.

### Jumping from the H1 pass

A `jump` in an H1 block ends the H1 pass at once:

````markdown
---
name: h1_jump
description: Jumps from the H1 pass past a section.
runvil: 0
---

# H1 jump

```lua
jump('## Target')
```

## Skipped

```lua
error('this section never runs')
```

## Target

```lua
return 'jumped'
```
````

````text
jumped
````

Later H1 blocks never run, and the main walk starts at the top-level section the jump names, with the H1 pass's `var`, skipping every section before it. `## Skipped` never runs, so its `error` never fires.

### Returning from the H1 pass

A scalar returned from a block in the H1 body ends the whole run at once:

````markdown
---
name: early_exit
description: Returns from the H1 pass so the walk never starts.
runvil: 0
---

# Early exit

```lua
return 'early'
```

## Never

```lua
error('the walk must not start after an H1 return')
```
````

````text
early
````

The returned value becomes the run result, no `##` section runs, and the `error` inside the skipped section `## Never` never fires. Returning `models.infer(prose)` from the H1 pass works the same way: the model's reply becomes the run result, and the walk never starts.

The final `var` is still read back on that early exit, as on every exit from the H1 pass. On the early exit the H1 pass is never reported as a finished section, and the run's section start and finish events never name the H1 title at all (see [Run events and records](#run-events-and-records)). Because of the read-back, an H1 block that replaces `var` itself with another value still fails the run even when it then returns a scalar, as [Failures outside the gate](#failures-outside-the-gate) explains.

### How the H1 pass differs from the walk

The H1 pass follows the walk's rules with exactly three differences:

- It runs first, ahead of the main walk's sections.
- A scalar return from it ends the whole run.
- An uncaught Lua runtime error in it, code `lua_runtime`, such as a failed `assert`, is the prompt's failed [hard gate](#the-h1-hard-gate), which fails the run with error code [`requirements_unmet`](#preparing-the-run) and stops later H1 blocks from running.

So the H1 region, from the `#` title to the first `##` heading, runs as its own live pass that can return early, fall through to the walk, or read its prose through an explicit model round such as `models.infer(prose)`.

### Prompts with no sections

A prompt can consist of only a `#` title with one Lua block and no `##` sections, and that H1 Lua runs when the prompt runs:

````markdown
---
name: h1_only
description: A prompt whose only block sits in the H1 body.
runvil: 0
---

# H1 only

```lua
return 'hello'
```
````

````text
hello
````

A prompt with no `##` section has no [entry section](#the-section-walk), so its H1 pass is the whole run, and a scalar the H1 block returns becomes the run's result text, even when that block is the prompt's only block. With `local x = 1` in place of the return, the H1 pass ends without a result, the end of the H1 pass is the end of the run, and the result text is `done`, the same [`done` fallback](#the-done-fallback) that ends any run that produces nothing.

### When there is an H1 pass

When the H1 body holds a Lua block or prose, it runs once as the H1 pass before the walk starts, and prose alone under the title also starts the pass. The H1 region can hold only prose and a `---` [thematic break](03-blocks-and-prose.md#thematic-breaks-and-setext-headings) with no Lua block, and then the H1 pass runs no author code.

A `#` title with nothing under it is valid. With no Lua block and no prose in the H1 body there is no H1 pass, and the run starts its walk directly at the first top-level `##` section, or, with no sections either, ends at once with the result `done`.

### Section entry ids in the H1 pass

The H1 pass is entry 0 of root chain `0`, and the main walk continues that same chain:

````markdown
---
name: h1_ids
description: Checks the section entry ids of the H1 pass and the first section.
runvil: 0
---

# H1 ids

```lua
assert(sys.id == '0.0')
```

## Result

```lua
assert(sys.id == '0.1')
return 'ok'
```
````

````text
ok
````

The H1 pass's [`sys.id`](05-lua-environment.md#run-facts-in-sys-and-ui) is `0.0`, and the first walked section's is `0.1`. A `call` made from the H1 pass runs as `0.0.0`, and one made from that first section runs as `0.1.0`.

## The H1 hard gate

After this section you can use the H1 body to stop a run before any section is walked, and to tell which failures there count as a failed gate.

````markdown
---
name: gated
description: Fails the hard gate before the walk starts.
runvil: 0
---

# Gated

```lua
assert(false, 'the gate cannot hold')
```

## Work

```lua
return 'never reached'
```
````

````text
the environment cannot satisfy this prompt:
- the H1 gate failed: H1 `Gated` lua:10: runtime error: H1 `Gated` lua:10: the gate cannot hold
````

The run fails with error code `requirements_unmet`, and its message opens with these two lines. After `- the H1 gate failed: ` comes the Lua error's own text. Its first line names the failing block and its line in the prompt file and then gives the `assert` message `the gate cannot hold`, and a Lua stack traceback follows on the next lines. After the traceback, the message ends with ` (at gated.md:10)`, the line of the failed `assert`. `## Work` never runs.

The Lua blocks under a prompt's H1 title are its hard gates. An `assert` there checks the prompt's arguments or prompt-wide settings, and when it fails, the run ends before any section is walked, with error code `requirements_unmet`. Move the same `assert` into the block under `## Work` and the run fails with error code `lua_runtime` instead, because the same uncaught Lua error in a block under a walked section is an ordinary `lua_runtime` failure.

Once the hard gate fails, the later blocks in the H1 body and every section never run, so a later H1 block's effects, such as a `store.write`, never happen. The failure still counts as the failing block's own: a `lua_chunk_failed` event is reported under the H1 title before the run ends.

### Failures that fail the gate

Any ordinary Lua error left uncaught in an H1 block fails the hard gate, and the run's error message holds the Lua error text, then the line of the failure. That covers these cases:

- An `error` or `assert` call.
- A call to a function that does not exist.
- Running out of Lua memory.
- An `error(...)` inside a library that the H1 body loads with `require`. The library's raised message is kept as the cause, so a library section `## broken` that runs `error('library boom')` gives a cause containing `library boom`.
- A `call` that names no visible section.
- A `tasks.fanout` error that would otherwise end the run with `lua_runtime`, such as a bad collection or an arm that failed with a `lua_runtime` error (see [When an arm fails](14-fanout.md#when-an-arm-fails)).

A `{{ }}` substitution that fails in the H1 body does not fail the gate: it keeps its own code and ends the run with `substitution`, at its placeholder's line (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)).

Each case fails the gate only when it is left uncaught. A `call` that names no visible section raises its error at the call site, so you can catch it with `pcall`:

````lua
local ok, err = pcall(call, '## Nope')
return tostring(ok) .. ':' .. tostring(err)
````

As the only block in an H1 body, this block returns text that starts with `false:`, followed by an error naming `## Nope`, and that text is the run result.

Because `tasks.fanout` raises the failing arm's own error, an arm's tool or store failure keeps its own code instead of becoming `requirements_unmet`.

### The failed gate's message

Error code `requirements_unmet` covers two cases inside the run: the [requirements check](#preparing-the-run) refusing the run, and the H1 pass failing its hard gate. The message tells them apart. A refusal's message is the [requirements notice](#the-requirements-notice), with the line of a gap the prompt caused when there is one, while a failed gate's message holds the Lua error text after the notice. A model round that has no model fails with a model code of its own instead, `model_required`, and the message `model binding required for section {section}`. The Harness also ends a run with `requirements_unmet` before it starts, when the launch input does not match the prompt's `input:` declaration or the Host's selected model is missing from its list of models (see [Runs that end before the walk](#runs-that-end-before-the-walk)).

A failed gate's error has a fixed notice, the line `the environment cannot satisfy this prompt:` followed by the line `- the H1 gate failed`, and it keeps the Lua failure as its cause. The message is the notice, then a colon and a space, then the cause's text, with the cause text appearing exactly once. So the assertion's message appears in the cause and not in the notice. The run's message then ends with ` (at {file}:{line})`, the line where the cause failed, as `gated` shows.

### Failures that keep their own code

In the H1 pass only ordinary Lua errors, the ones with code `lua_runtime`, turn into `requirements_unmet`, and every other uncaught failure keeps its own code:

| Failure in the H1 pass | How the run ends |
| --- | --- |
| A `log` quota that runs out, or a `call`, task, or fanout arm past the call depth cap | `lua_quota` or `call_depth_exceeded` |
| A failed `{{ }}` substitution | `substitution` |
| A task error | Its task code, such as `tasks_live` or `task_canceled` |
| A failed model call, a model round with no model, or a context the compactor cannot make fit | Its model code, such as `transport`, `model_required`, or `context_exhausted` |
| A tool failure or a failed tool freeze | Its tool code, such as `tool_backend` or `tool_sets_invalid` |
| A failed store call or a claims conflict | Its store code, such as `not_found`, or `determinism` for a claims conflict |
| A Host cancel | `canceled` |

Task errors keep their task codes rather than becoming `requirements_unmet`. That covers an uncaught `tasks_live`, `task_not_owned`, or `task_consumed` error (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)), a delivered canceled task's `task_canceled` error, and a `task_not_owned` or `task_consumed` error raised again later. A Host cancel during the H1 pass ends the run with the canceled error, as a cancel does anywhere in the run.

### Errors raised again after a catch

An error you catch with `pcall` in the H1 pass and raise again keeps the failure it came from when you raise it before the block makes another call that waits on the Harness, such as a model round, so its own code holds. Raised again after such a call, an error whose `kind` is `lua` is rebuilt as an ordinary Lua error with code `lua_runtime` and fails the gate as `requirements_unmet`, while an error of any other kind still keeps the failure it came from and its own code.

### Failures outside the gate

Two checks run around the H1 blocks rather than inside them. They are not part of the gate, so each fails the run with error code `lua_runtime`:

- When the H1 pass's `var` is read back and `var` turns out to have been replaced with another value, the run fails with an error naming `global was reassigned`, at the first Lua line of the last Lua block that ran. This happens even when the block returned a scalar that would otherwise end the run.
- A `jump` from the H1 pass whose heading matches no top-level section, or more than one, fails the run, because the jump's heading is checked after the H1 block has finished. The failure points at the line of the `jump`.

## The run result

After this section you can say where a run's result text comes from and which error a run ends with when it does not complete.

````markdown
---
name: plain
description: Returns a fixed string as the run result.
runvil: 0
---

# Plain

## Only

```lua
return 'plain'
```
````

````text
plain
````

The run result is what a run ends with: its final text when it completes, or the error that ended it when it does not. A completed run gives back its result as a single text string. Here the section on the main walk returns the scalar `'plain'` from its Lua block, so the run completes with the final text `plain`.

When nothing returns, the run still has a result:

````markdown
---
name: no_return
description: Ends with the fallback result because no block returns.
runvil: 0
---

# No return

## Only

```lua
local x = 1
```
````

````text
done
````

No block returns a scalar, the walk runs past the end of the last top-level section, and the run's result is the fixed fallback text `done`. This is the `done` fallback.

### Where the run result comes from

A completed run's result text comes only from a `return` in the H1 pass, a `return` on the main walk, or the `done` fallback. Any scalar that a section on the main walk returns becomes the run's result text, including these:

- `return 'done'` gives `done`.
- `return store.read('note.md')` gives the note's text.
- `return sys.when` gives a timestamp such as `2023-11-14T22:13:20Z`, and the run result is that string, byte for byte.
- `return tools.call('web/fetch', { url = 'https://example.com' })` gives the tool's reply.
- Values the block builds, such as `return var.answer`, `return r[1].text .. '|' .. r[2].text`, or `return err.kind .. '|' .. tostring(err)`.

The rule holds at every heading level the main walk reaches: when the top-level chain ends with `return 'x-value'` in a section `### X`, the run's final text is `x-value`.

A section's `return` value becomes the run result, not the model's last reply. A section that ends with `return 'ok'` after the model's final reply `done` gives the result `ok`. A section that calls a model without returning the reply adds nothing to the run's result, and only a reply that a block returns, such as `return models.infer(prose)` in the last block, becomes the result. Running [`models.loop`](11-conversations.md#starting-a-conversation) on the section entry's `messages` list and then returning `messages[#messages].content` makes the model's final text reply from the loop the run result, which is how the last block of the [shipped](01-what-a-prompt-is.md#shipped-example-prompts) `research-person` prompt ends.

Returns from other chains go back to the code that started them instead. A value returned inside a section reached through `call('## S1')` goes back to the caller as the value of `call`. A section started with `tasks.spawn` hands its `return` value back as that task's result, and the section that started the tasks reads those values as `results[1].result` and `results[2].result`. When two `## Reply` tasks each return `hi there`, the run result is `hi there hi there` only because `## Greet`, the entry section, returns that text after waiting on both tasks. A called section or task that ends without returning gives back empty text, not `done`.

### The `done` fallback

When no block returns a scalar and nothing else produces a result, the run's result is `done`. A walk whose last section falls through without returning still ends the run as completed, and a prompt with only an H1 title, nothing that runs under it, and no sections finishes at once with the result `done`. A prompt with no Lua at all ends the same way:

````markdown
---
name: prose_only
description: A prompt with one prose section and no Lua.
runvil: 0
---

# Prose only

## Only

This prose is never sent to a model.
````

````text
done
````

This prompt has one prose-only section, no Lua anywhere, and an empty H1 body, so it declares nothing the Harness has to provide. A prose-only section with no Lua fence runs without calling a model and falls through to the next section, and here there is no next section, so the run ends with `done`.

### How a run ends

Every run ends with exactly one result, which is one of two things:

- Its final text, when the run completes. A run that finishes gives back its final text, and that text is recorded as the run's result.
- The error that ended it, when the run does not complete. The error has a code that names the cause, such as `lua_runtime` for an uncaught Lua error, together with a self-contained message for people to read. A cancel is one of these errors: the one whose code is `canceled`.

Whatever ends a run first decides its result, and a later error or cancel never replaces it. A cancel that lands after the run's first step has refused the run leaves the run with error code `requirements_unmet`.

### Canceling a run

When the Host cancels a run, the run ends with the canceled error, whose code is `canceled` and whose message is `canceled: the run was canceled or this call was stopped`. The run ends with that error rather than the failure of whatever call the cancel cut short, even when the cancel lands while a block waits on `return models.infer(prose)`. A cancel stops the run even while its Lua is busy in a loop such as `while true do n = n + 1 end`.

A cancel that lands while every chain is waiting takes effect at the run's next step, which tears down every chain, while Lua code that is already running sees the cancel as it runs. A run canceled while a `tools.call` to a script tool is suspended ends with the canceled error rather than a text result or the tool's own error. A model request cut short by a cancel is not reported as a failed model call.

A start check that fails after the run's cancel fired, whether on the version, the store, or the requirements, ends the run with the canceled error instead of the check's own error, as when a cancel cuts the wait short while a needed Pack is still starting (see [Waiting for Packs and taking the snapshot](#waiting-for-packs-and-taking-the-snapshot)). A cancel that comes before the run, or while the Harness looks up the model, ends the run with the canceled error before the Harness prepares it.

### Failed runs and error codes

A failed run ends with an error, the same error that a caught [error value](05-lua-environment.md#error-values) stands for in Lua. It has a code, a message for people to read, the prose a model reads, an optional location, and the values it names, such as a path. The code names the cause by its tag, in snake_case, such as `lua_runtime`, which a caught error value holds as `err.code`. Each code also has a number whose hundreds give its family, such as 2xx for the Lua codes (see [Run error codes](16-limits-and-errors.md#run-error-codes)). The failed run's message is the error's located message: its text with its causes, then where the failure happened. Running a prompt fails with one of these two codes most often:

- `lua_runtime`, for an uncaught Lua error in a section.
- `requirements_unmet`, when the Harness cannot satisfy the prompt, such as when a required Pack is missing, or when an H1 block fails the prompt's hard gate.

Any uncaught Lua error in a walked section fails the run with error code `lua_runtime`. That includes `assert(false, 'message')`, a call to a function that does not exist, and a walked library section that returns a table, which fails with `cannot return a table as a result`. Calling `error('message')` in a section's Lua block stops the run on purpose, and the rest of the prompt never runs:

````markdown
---
name: second_fails
description: Fails on purpose in the second section.
runvil: 0
---

# Second fails

## First

```lua
local x = 1
```

## Second

```lua
error('expected failure')
```
````

The run fails with error code `lua_runtime` and a message containing `expected failure` that ends with ` (at second_fails.md:18)`, the line of the `error` call. Other failures report their own codes, such as a tool code like `tool_backend` for a tool failure and a store code like `not_found` for a failed store call, and [Failures that keep their own code](#failures-that-keep-their-own-code) lists the ones the H1 pass leaves unchanged.

### Runs that end before the walk

A run that ends before the walk still has a record, because the Harness begins the record before anything else in the run. The run's record ends with the error that ended it, its code and its located message, and the recorded code tells you why a run that never reached its sections failed:

- The model code the listing failed with, such as `transport`, when the Harness cannot fetch the Host's list of models, and `requirements_unmet` when that list does not hold the selected model.
- `structure_invalid` when the prompt's `runvil:` version is missing, and `unsupported_version` when the version is unsupported.
- `requirements_unmet` when the launch input does not match the prompt's [input file](02-file-structure.md#input-and-output-file-declarations) declaration, and the store code of the refusal, such as `approval_required`, when the store refuses to stage the input.
- `no_store` when the run declares no store, and the store code of the failure when its store fails the probe.
- `requirements_unmet` when the environment falls short of the prompt's requirements.

A run whose cancel fires first ends with the canceled error instead, even when the cancel comes before the run starts.

A prompt that does not parse is not one of these runs. It starts no run and has no record, and its parse error goes to the Host (see [A prompt that does not parse starts no run](#a-prompt-that-does-not-parse-starts-no-run)).

## Run events and records

After this section you can read the events a run reports, in order, and know what each run's record holds.

````markdown
---
name: loop
description: Returns a fixed string from one section.
runvil: 0
---

# Loop

## Only

```lua
return 'ok'
```
````

This run records these events, in this order, each with the name it carries:

| Order | Event kind | Name |
|---|---|---|
| 1 | `run_started` | `Loop` |
| 2 | `section_started` | `Only` |
| 3 | `lua_chunk_started` | `Only` |
| 4 | `lua_chunk_succeeded` | `Only` |
| 5 | `lua_teardown_started` | `Only` |
| 6 | `lua_teardown_succeeded` | `Only` |
| 7 | `section_finished` | `Only` |
| 8 | `run_succeeded` | `Loop` |

The run's events open with `run_started` under the H1 title `Loop`. Parsing comes before the run and reports no events (see [Parsing comes first](#parsing-comes-first)). `# Loop` has no H1 body, so the run goes straight from `run_started` to the first section's `section_started`.

### The order of run events

A run reports its progress as ordered [run events](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program). Each event's `kind` field spells its name in snake_case, and a run reports them in this order:

1. `run_started`.
2. When the H1 body has Lua blocks, the H1 pass's block events and teardown events.
3. For each walked section: `section_started`; `lua_chunk_started` and then `lua_chunk_succeeded` around each Lua block it runs, with that block's model and tool events, such as `model_turn_completed`, between them; one `lua_teardown_started` and `lua_teardown_succeeded` pair after its last block; and `section_finished`.
4. `run_succeeded`.

A prompt with no H1 blocks, such as `loop`, goes straight from `run_started` to the first section's `section_started`. A block that is waiting reports neither succeeded nor failed until it finally returns or fails.

A run that started and does not succeed ends its events with `run_failed` under the H1 title in place of `run_succeeded`. The only run-level events are `run_started`, `run_succeeded`, and `run_failed`, and there is no run-level cancel event, so a canceled run also ends with `run_failed`, even though a canceled run is not a failed run.

### Which name an event carries

Every run event, error, and report names the section that produced it by its heading text without the leading hashes, so `# Boundaries` reports as `Boundaries` and `## Only` as `Only`. Run-level events and those from the H1 pass carry the prompt's H1 title, while those from a walked section name that section. That is why `loop` reports `run_started` and `run_succeeded` under `Loop`, around the events named `Only`.

The H1 pass reports no `section_started` or `section_finished` of its own, because it is not a walked section. Only its block and teardown events appear under the H1 title.

### Section boundaries

Each walked section reports `section_started` when the walk enters it and `section_finished` when it completes, both under the section's name. At a fall-through, the section that ends reports `section_finished` before the next section reports `section_started`, so sections `One` and `Two` report One started, One finished, Two started, Two finished.

A section that ends by jumping counts as completed too. It reports `section_finished` before the section the jump names reports `section_started`, the same as at a fall-through, so `## A` running `jump('## B')` reports A started, A finished, B started, B finished. A section whose `jump` names a heading that matches no visible section, or more than one, still reports `section_finished`, because the section completes before the heading is looked up. The run then fails on the unresolved heading, and for `jump('## Missing')` the error contains `not found`.

A section that ends the walk with `return` still finishes normally, as at a fall-through: its final `var` is read back and it reports `section_finished`. So `## Two` doing `return 'two-ran'` gives the result `two-ran` and a final Two finished.

### Events from a failing section

A walked section that raises an uncaught error, such as `error('expected failure')`, reports `section_started` and `lua_chunk_failed` but never `section_finished`. Earlier sections have already finished normally. The failing section entry's VM is still torn down exactly once, with one `lua_teardown_started` and `lua_teardown_succeeded` pair, and then the run reports `run_failed`.

In `second_fails`, that means `First` reports everything through its `section_finished`, then `Second` reports `section_started`, `lua_chunk_started`, `lua_chunk_failed`, and one teardown pair, and the run ends with `run_failed` under `Second fails`.

### Log checkpoints in the record

A `log` checkpoint is recorded under the name of the section whose Lua wrote it, in the order the run reaches it. Checkpoints from the H1 body come under the H1 title ahead of every walked section's, so a `log("shared loaded")` call in the H1 body records its checkpoint ahead of every section's checkpoints.

### Events from a run that ends early

A run that ends before it starts has no events in its record, only the values below and the error that ended it. That holds for a run refused on its input or by a start check, and for one that ended on a failed model lookup or an early cancel.

### What a run's record holds

Each run of a prompt is recorded as its own run with its own run id, even when the prompt text is identical. A run's record holds these values:

| Value | What it holds |
| --- | --- |
| Run id | An id of its own, different for every run |
| `name` | The run's name as the Host session gives it, such as `session-1` |
| `prompt_hash` | A hash of the prompt text, written with the `sha256:` prefix that names its algorithm |
| Seed | A fresh seed drawn for this run |
| `started_at` | The run's start time, in UTC milliseconds since the Unix epoch |

Beside these values, the record holds the run's history: every event the run reports, every piece of [Harness work](01-what-a-prompt-is.md#the-five-kinds-of-harness-work) the run asks for, and the answer to each.

The run's name is the run metadata's `name` and also the `execution` field of every event the run records, from its first event through its last. The same prompt text always yields the same prompt hash across runs.

Each run draws its own fresh [seed](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) from the operating system's cryptographic random source and records it at the start of the run, before anything else, so two runs of the same prompt get different seeds. The run's start time, `started_at`, is stamped once from the system wall clock when the Harness begins the run's record, because the Engine reads no clock of its own, and a later run never starts earlier. The Harness supplies both the seed and the start instant, drawing them when it begins the run's record, and a prompt cannot set either one.

Two runs of the same prompt running at the same time stay separate. Each returns its own result, and each run's records hold only its own events, Harness work and answers, `log` checkpoints, and final `run_succeeded`.

### Repeating a run exactly

Two runs with the same seed, the same start instant, the same Harness answers, and the same messages, each reaching the run at the same point, produce byte-identical text, including the same `sys.when`, the same seeded values, and the same untrusted envelope nonce. A replay runs the same prompt under the same seed, start instant, answers, and messages, so it also issues the same [Harness work](01-what-a-prompt-is.md#effects-and-harness-work) as the original run and reports the same events, in the same order.
