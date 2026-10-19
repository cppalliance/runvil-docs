# How a prompt runs

Every run moves through the same stages in the same order: the parse, prepare, the H1 pass, and the section walk, and it ends in exactly one outcome. This chapter follows that order, one stage at a time. Knowing the order tells you which mistakes stop a run before any of your Lua runs, which sections run and when, and what text the run hands back when it finishes.

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

This run is as short as a run gets. Prepare finds nothing to check, because the frontmatter declares no Plugins and no model roles. The H1 body under `# Hello` is empty, so there is no H1 pass. The walk enters `## Main`, the block returns `hello`, and the run ends in success with `hello` as its run result. A block that only returns a literal value asks the Harness for no work at all, so this run finishes without waiting on anything outside the prompt.

### The stages in order

Every run goes through these stages, in this order:

1. Parse. The whole file is parsed and every Lua block is compiled, and nothing runs yet. See [Parsing comes first](#parsing-comes-first).
2. Prepare. The Harness sets up everything the frontmatter asks for and checks that it can provide it, or refuses the run. See [Preparing the run](#preparing-the-run).
3. The H1 pass. The H1 body's blocks run before the first section, but only when the H1 body holds a `lua` block or prose. See [The H1 pass](#the-h1-pass).
4. The section walk. The top-level sections run in file order. See [The section walk](#the-section-walk).
5. The outcome. The run ends in exactly one of three run outcomes. See [The run result and outcomes](#the-run-result-and-outcomes).

Parsing is the first check prepare makes after it draws the run's seed and start time, so a file that does not parse never reaches the checks of Plugins and model roles. A prompt whose H1 body holds no `lua` block and no prose skips the H1 pass and goes straight to its first section, as `hello` does. The three run outcomes are success, with the text the run returned; cancelled, when the run is interrupted; and failure, with the error that ended the run.

### Prepare can stop a run before it starts

Prepare runs before any of your Lua, so a prompt the Harness cannot satisfy fails before its first section runs. This prompt declares a model role that needs a large context window:

````markdown
---
name: needs-context
description: Needs a model with a large context window.
runvil: 0
models:
  reader:
    min_context: 200000
---

# Needs context

## Main

```lua
return 'done'
```
````

Run on a Host whose current model has a 32000-token context window, this prompt never returns `done`. The run fails with run error kind `RequirementsUnmet`, and its message is this requirements notice:

````text
the environment cannot satisfy this prompt:
- role 'reader': requires a context of at least 200000 tokens; the current model provides 32000
````

The `reader` role is a model role (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)), and `min_context` is one of its keys (see [Declaring roles under `models:`](10-models.md#declaring-roles-under-models)). [Model roles at prepare](#model-roles-at-prepare) shows how prepare checks the role, and [The requirements notice](#the-requirements-notice) explains each line of the message.

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

Neither section creates or passes a store handle: `store` is ready when the first block runs, and `## Second` reads back what `## First` wrote. Because each run has its own store, two runs that both write `paper.md` each read back their own copy. Every model role the frontmatter declares is likewise already bound when the first block runs, so on a declared `writer` role, `models.use('writer')` reaches the model directly.

### The work the Harness does

Some calls in your Lua need something outside the prompt, and the Harness performs that work for the run (see [Effects and Harness work](01-what-a-prompt-is.md#effects-and-harness-work)). There are four kinds of Harness work:

- a model round
- a tool call, including a wait on `input.ask()` (see [Asking the operator](05-lua-environment.md#asking-the-operator))
- a store operation
- the timer behind a timed wait on a task started with `tasks.spawn` (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn))

`input.ask()` is an ordinary tool call to the `user-input` Plugin's ask tool. While one chain waits for its answer, only that chain waits, and every other chain keeps running. A chain is a walk with its own entries, defined in [The section walk](#the-section-walk). So the three arms of one `tasks.fanout` call (see [Fanout in one call](14-fanout.md#fanout-in-one-call)) all send their model rounds together, without any arm waiting for another.

A call that needs Harness work looks synchronous to you: the block pauses at the call and continues with the call's answer, as if the call had returned it directly. Each Lua block runs as its own Lua coroutine, which is what lets it pause at the call and resume later. A block that asks for none of the four kinds of work, such as `return 'hello'`, never pauses.

## Parsing comes first

After this section you can tell what parsing checks, what a parse failure stops, and which parse events open every run's record.

````markdown
---
name: t
description: Shows the events a parse reports.
runvil: 0
---

# T

```lua
local secret = 42
```

## S

p
````

Parsing this prompt reports four parse events, in this order. Each event has a kind and the name of the section it belongs to:

| Order | Event kind | Section name |
|---|---|---|
| 1 | `parse_started` | `Prompt` |
| 2 | `lua_compilation_started` | `T` |
| 3 | `lua_compilation_succeeded` | `T` |
| 4 | `parse_succeeded` | `Prompt` |

The H1 block's `local secret = 42` compiles during the parse, but none of it runs then. Parsing builds the prompt's tree of H1, sections, and blocks without running any of its Lua or prose. Section `S` holds only prose, so it adds no compilation events.

### The whole file parses before anything runs

The whole prompt file is parsed, and every Lua block in it compiled, before any part of the prompt runs. A prompt with a parse error never starts. An invalid prompt file is rejected as a whole, never accepted in part, so none of its sections run. Parsing needs no network access and no connection to a model, so you can check a prompt's structure offline.

Parsing checks only the shape of the frontmatter's contract keys and what the file's text shows: keys, names, duplicates, ranges, and default types. A prompt whose contract has a valid shape parses even when the environment that runs it cannot satisfy it. Matching the Plugins under `plugins:` and the roles under `models:` against the environment is prepare's job, and the tool catalog and the model reach the prompt only at prepare. The tool catalog holds the tools of every Plugin the Host can serve for a run (see [The Engine, the Harness, and the Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)). A prompt that declares Plugin `web` in an environment without it parses first, and then the requirements notice reports the gap:

````text
the environment cannot satisfy this prompt:
- missing required Plugin: web
````

Each check ends the run before the next one starts, so the order of failures is fixed. A parse failure is reported ahead of any input failure, and an input failure is reported ahead of any requirements refusal.

### The six parse event kinds

A parse event is an event the run reports while the prompt is parsed (see [A Markdown file that is a program](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program) for run events). Parsing a prompt reports `parse_started` under the section name `Prompt`, then `parse_succeeded` or `parse_failed`, also under `Prompt`. Compiling each Lua block reports `lua_compilation_started` under the name of the section that holds the block, or the H1 title for an H1 block, then `lua_compilation_succeeded` or `lua_compilation_failed`. Every lifecycle event's `kind` is its event name in snake case.

A parse reports only these six event kinds, and no other events:

| Event kind | Reported when |
|---|---|
| `parse_started` | parsing begins |
| `parse_succeeded` | the whole file parsed |
| `parse_failed` | the parse failed |
| `lua_compilation_started` | a Lua block starts to compile |
| `lua_compilation_succeeded` | that Lua block compiled |
| `lua_compilation_failed` | that Lua block failed to compile |

A successful parse reports `parse_started`, then one `lua_compilation_started` and `lua_compilation_succeeded` pair for each Lua block in source order, then `parse_succeeded`. A section with a prologue and an epilog (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)) adds two pairs. Replace `## S` in the prompt above with this section:

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

The parse now reports two `lua_compilation_started` and `lua_compilation_succeeded` pairs named `S`, after the pair named `T` and before `parse_succeeded`. A prose-only section, like the first version of `S`, adds no compilation events.

A Lua block that fails to compile reports `lua_compilation_started` and then `lua_compilation_failed`, and the whole parse fails. That parse reports exactly four events: `parse_started`, those two, and `parse_failed`. The compile error names the failing block by its section and its place in it, such as ``section `Private section` prologue``, or the epilog when the epilog fails.

A prompt that breaks a Lua fence rule (see [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors)) fails its parse before any Lua compiles. Its parse reports only `parse_started` and `parse_failed`, both named `Prompt` rather than the H1 title, even when the H1 body holds Lua.

Every parse event is reported under task `0`, because parsing happens before any run starts. Every parse event also holds the execution id the parse ran under, so each one reads back as execution, section, and kind. Parse and compilation events hold only the execution id, the section name, and their task and sequence number. They never include any of the prompt's Lua source, not even from a Lua block that fails to compile.

### Parse events open the run's record

A prompt's parse events are the first records of its run (see [Effects and Harness work](01-what-a-prompt-is.md#effects-and-harness-work) for the recorder), and they are recorded whether or not the parse succeeds. When a run stops before any section runs, its first and last events tell you where it stopped:

| How the run stops | First event | Last event |
|---|---|---|
| the prompt does not parse | `parse_started` | `parse_failed` |
| the input is refused | `parse_started` | `parse_succeeded` |
| the requirements are unmet | `parse_started` | `parse_succeeded` |

A prompt that does not parse ends its run as failed with the kind `Parse`, and the failure message is the parse error's full cause chain. Requirements are checked only after the prompt parses, so the record of a run refused for unmet requirements ends with `parse_succeeded`. The prompt is also parsed before its input is checked, so an input refusal always follows a successful parse, and the refused run's last event is `parse_succeeded`. A refused or unparseable prompt runs no section code, and its record holds only parse events.

## Preparing the run

After this section you can say what prepare checks and binds before any Lua runs, and why it refuses a run with `RequirementsUnmet` when it cannot.

````markdown
---
name: no-requirements
description: Declares nothing for prepare to check.
runvil: 0
---

# No requirements

## Main

```lua
return 'no Plugins'
```
````

````text
no Plugins
````

Prepare is the stage before a run starts in which the Harness sets up everything the frontmatter asks for and checks that it can provide it. This prompt declares no Plugins under `plugins:` and no model roles under `models:`, so prepare has nothing to check, its requirements report has no gaps, and prepare never refuses it. A prompt like this one, which also calls no model, runs to completion on a Host with no Plugins installed and no model configured.

### What prepare checks

Before a run starts, the Harness checks the Plugins the prompt declares against the Plugins installed on the Host and the services the run provides, and checks the model roles the prompt declares against the Host's current model. Prepare can refuse to start the run, and a refused run ends before it is built.

A prompt the environment cannot satisfy is refused before any of its blocks run. `RequirementsUnmet` is the run error kind of that refusal: the run ends as failed, and its message is the requirements notice, which lists what is missing, one line per gap. Prepare refuses the run with `RequirementsUnmet` when any of these holds:

- a Plugin the prompt declares is not installed
- a declared Plugin is installed but unavailable because it cannot serve, for example because it failed to build
- a declared Plugin needs a service the run lacks
- the current model falls short of a declared role's context minimum or one of its hard keywords

An unavailable Plugin is one the Host installed that cannot serve the run. The run proceeds only when every model role requirement is met and every required Plugin is installed, can serve, and has the services it needs. [Plugins at prepare](#plugins-at-prepare) shows each Plugin case.

The refusal comes after the Plugin report and the model report are merged into one, and before the run is built. So when the Harness cannot provide what the prompt requires, such as a model with a large enough context window for one of its roles, no section runs, no Lua runs, and nothing in the prompt can catch the refusal. The refusal's message is the notice text, line for line. The notice opens with the line `the environment cannot satisfy this prompt:` and then gives one `- ` line per gap, in this order, with each `{...}` standing for the value the line names:

````text
the environment cannot satisfy this prompt:
- missing required Plugin: {name}
- {plugin} needs {service}, and the environment provides none
- {plugin} is unavailable: {reason}
- role '{role}': requires a context of at least {n} tokens; the current model provides {m}
- role '{role}': requires '{keyword}'; the current model's thinking capability is {mode}
````

A real notice holds a line only for each gap the run has. [The requirements notice](#the-requirements-notice) explains every line.

### The order prepare works in

Prepare runs in a fixed order:

1. It draws the run's seed and start time (see [The Engine, the Harness, and the Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)).
2. It parses the prompt.
3. It stages the input file the prompt declares.
4. It waits for Plugins to be ready.
5. It takes the snapshot of the Host's Plugins.
6. It prepares the run context, which binds the declared model roles.
7. It refuses a prompt the environment cannot satisfy.
8. It builds the run.

When the run is built, after preparation and before any section runs, the `runvil` version in the frontmatter is checked (see [The `runvil` version key](02-file-structure.md#the-runvil-version-key)). Version `0` proceeds, and any other version is refused as unsupported. A missing version fails with the message `not a runvil prompt: no runvil version`. Either way, a prompt with an unsupported or missing version produces no section result.

### Staging the input file

Input staging happens during prepare, before the run starts. The launch's input text is placed at the path the frontmatter's `input:` key names (see [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations)), so the first section's Lua can already read the input file. Prepare fails the run before it starts, with the failure kind `Input`, in two cases:

- The launch supplies input text, but the prompt declares no `input:` file. The message is ``the launch supplied input text, but the prompt declares no `input:` file``.
- The prompt declares an `input:` file that the launch did not supply and the store does not already hold. The message is ``the prompt declares the input file `{path}`, but the launch supplied no input text and the store has no such file``, with the declared path in place of `{path}`.

### Waiting for Plugins and taking the snapshot

Before a run starts, the Harness waits until every built Plugin is ready or the run is cancelled. A Plugin whose readiness check fails is unavailable to the run, with its error as the reason, so a prompt that declares it is refused. Cancelling a run while Plugins are still starting ends the wait, and each Plugin is taken as it stands at that moment. A refusal that follows the cancel, such as one for a declared Plugin that was still starting, ends the run as cancelled rather than failed.

After the wait, each run takes one snapshot of the Host's Plugins. That snapshot supplies the tool catalog the run offers, the object for each Plugin in the run's Lua (see [Tool objects and Plugin objects](12-tools.md#tool-objects-and-plugin-objects)), and the preludes that go to every section's Lua (see [Plugin preludes](12-tools.md#plugin-preludes)).

### Binding model roles

When it prepares the run context, prepare binds every model role the prompt declares to the Host's current model, and checks each role's context minimum and hard keywords against that model. A role requiring `thinking` fails on a model whose thinking capability is `Never`, and a role requiring `no-thinking` fails on a model whose thinking capability is `Always`. Soft keywords only record your intent and are not checked (see [Keywords and thinking](10-models.md#keywords-and-thinking)). When the Host has no current model, no role is bound and nothing is checked. [Model roles at prepare](#model-roles-at-prepare) shows each case.

### An H1 gate failure reads like a refusal

One failure that is not a refusal reports the same run error kind. An uncaught Lua error in the prompt's H1 blocks, such as a failed `assert` or an error raised by a library that an H1 block loads with `require` (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)), is the prompt's failed hard gate. It ends the run before any later H1 block or any section runs, with run error kind `RequirementsUnmet` and a message made of the line `the environment cannot satisfy this prompt:`, the line `- the H1 gate failed`, then `: ` and the Lua error's own text. [The H1 hard gate](#the-h1-hard-gate) shows an example and every gate case.

## Plugins at prepare

After this section you can predict how prepare treats each Plugin you declare, what an unavailable Plugin does to a run, and what the tool catalog holds once the run starts.

````markdown
---
name: needs-web
description: Declares the web Plugin.
runvil: 0
plugins: [web]
---

# Needs web

## Main

```lua
return 'done'
```
````

On a Host that has installed `web` and where it can serve, prepare passes and the run returns `done`. On a Host with no Plugins, the run is refused at prepare, before any section runs, with run error kind `RequirementsUnmet` and this notice:

````text
the environment cannot satisfy this prompt:
- missing required Plugin: web
````

The refused run's record ends with the `parse_succeeded` event, because the prompt parsed before the environment refused it.

### Declared Plugins are required Plugins

Every Plugin a prompt lists under the `plugins:` key (see [Tools, Plugins, and the `plugins:` key](12-tools.md#tools-plugins-and-the-plugins-key)) is a required Plugin: the run does not start unless the Host can provide it. Each one is named by its Plugin id and kept in declaration order. Before any Lua runs, prepare checks each declared Plugin against the Host and refuses the run unless that Plugin is installed, can serve, and has every service it needs.

A declared Plugin counts as present when the Host installed it and it can serve, even if it offers no tools, so declaring a Plugin that has no tools never refuses the run. A Plugin the frontmatter declares, and any Plugin listed as able to serve, still has a Plugin object in Lua when it offers no tool. The Plugin check is the Harness's part of prepare. The Engine's own part checks model roles only, so it finds nothing missing in `plugins: [web]` even when the catalog holds no `web` tool.

Each declared Plugin's prelude runs in every section's section VM, in the order the prompt declares the Plugins, after the Engine globals are installed and before any block runs. An undeclared Plugin's prelude never runs, even when the prompt calls that Plugin's tools.

A prompt that declares no Plugins has no Plugin requirements, so it prepares and runs even on a Host with no Plugins installed, as `no-requirements` does in [Preparing the run](#preparing-the-run).

### The tool catalog

The run's tool catalog holds the tools of every Plugin the Host can serve for that run, whether the prompt declares it or not. A Plugin that cannot serve, or that lacks a service it needs, adds no tools. Each usable Plugin's tool list is read once, after every Plugin is ready, so the catalog and the tools a call can reach stay fixed for the whole run.

A section can call an undeclared Plugin's tool by its full tool id. On a Host that installs `web`, a prompt with no `plugins:` key can run this block:

````lua
local page = tools.call('web/fetch', { url = 'https://example.com' })
local kind = type(input)
````

The call reaches `web/fetch` even though the frontmatter declares no Plugin (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls)). The prelude of an undeclared Plugin is never installed, even on a Host that installs that Plugin. The `user-input` Plugin's prelude is what defines `input.ask()`, so in this prompt, which leaves `user-input` undeclared, `type(input)` is `nil`.

In Lua, `tools.required()` lists the Tool objects of the Plugins the frontmatter declares, and `tools.extras()` lists the Tool objects of every other Plugin in the catalog (see [Tool objects and Plugin objects](12-tools.md#tool-objects-and-plugin-objects)). With `plugins: [web]` on a Host that installs and can serve both `web` and `user-input`, `tools.required()` holds the Tool objects for `web/fetch` and `web/search`, and `tools.extras()` holds the one for `user-input/ask`.

A section can offer only tools in the run's catalog, and offering an id the run cannot offer is a hard error (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)). Calling `tools.offer` with an id the catalog lacks raises `tools.offer: "{id}" is not a catalog tool in this run`, with the id in place of `{id}`.

### Host services

A host service is a named service the Host supplies for each run, such as an input broker. A Plugin can need host services, and prepare checks every installed Plugin's needs against the run's services. A Plugin is usable only when the run provides every service it needs, and a Plugin missing any of its services has no tools in the run.

A declared Plugin that lacks a service it needs refuses the run at prepare with run error kind `RequirementsUnmet`. The notice gets one line per missing service, in the form `- {plugin} needs {service}, and the environment provides none`, which names both the Plugin and the service instead of calling the Plugin missing. A run refused this way never calls the Plugin, so no tool effect happens, and the refused run records zero effects.

The `user-input` Plugin is the common case. Its fully qualified name is `runvil/user-input`, so a Host that picks no other name installs it with the Plugin id `user-input`. It needs the run's input broker, the service `runvil/input-broker`, so a run without a broker cannot use it. A prompt that declares it is refused before the run starts on a Host with no operator to ask, and the notice names both the Plugin and the service:

````text
the environment cannot satisfy this prompt:
- user-input needs runvil/input-broker, and the environment provides none
````

That notice does not contain `missing required Plugin: user-input`, because the Plugin is installed.

Whether a declared Plugin's service needs are met depends only on the Host running the prompt. The same prompt text runs to completion on a Host that supplies the service and is refused on one that does not, and the two runs differ only in the services they provide. When the run supplies the service a declared Plugin needs, the run prepares and that Plugin's tool calls are served through the service the run supplies, so a declared `user-input` on a Host with an operator to ask runs and its ask tool reaches that operator. A Plugin that needs no service prepares and runs normally on a Host that has no operator to ask, so a run without an input broker can still use it.

### Unavailable Plugins

A declared Plugin that is installed but cannot serve refuses the run with run error kind `RequirementsUnmet` and the notice line `- {plugin} is unavailable: {reason}`. A Plugin cannot serve in two cases:

- Building it failed at install. That is not an install error: the Host stores the Plugin as unavailable with the failure as its reason, and every run that needs it is refused naming that reason.
- Its readiness check fails before the run.

The `web` Plugin cannot be built on a Host that supplies no search provider, so a prompt that declares `web` there is refused with the notice line `- web is unavailable: web needs runvil/search-provider, and this host provides none`.

When an installed Plugin fails to build or fails its readiness check, the Harness writes a warning naming the Plugin to the Harness's own log, which is separate from the run's events. A build failure logs `Plugin construct failed; it is installed as unavailable`, and a readiness failure logs `Plugin will not be ready; this run treats it as unavailable`.

### Only declared Plugins can refuse a run

The requirements check looks only at the Plugins the prompt declares, so only declared Plugins can refuse a run. An installed Plugin the prompt does not declare that fails to start or lacks a service just has no tools in the run, and the run goes on. The requirements report lists each Plugin the prompt declares, while the run's roster, its list of Plugins, holds every Plugin, declared or not. An undeclared Plugin with unmet needs joins the roster as unavailable, with the reason `the run lacks the services it needs: {services}`.

Offering an unavailable Plugin with `plugins.offer` raises `plugins.offer: the {name} Plugin is unavailable in this run: {reason}`. So an undeclared Plugin whose readiness check fails offers no tools, and a section that offers it with `plugins.offer` fails the run with that message.

## Model roles at prepare

After this section you can predict how prepare binds each model role you declare, which checks it runs against the Host's current model, and what an unmet requirement looks like.

````markdown
---
name: roles
description: Declares two model roles.
runvil: 0
models:
  writer: {}
  reader:
    min_context: 200000
---

# Roles

## Main

```lua
return 'done'
```
````

On a Host whose current model has a context window of exactly 200000 tokens, the run returns:

````text
done
````

Before the run starts, prepare binds every model role declared under `models:` to the Host's current model, one binding per role, so each role is ready the first time Lua uses it. This prompt declares two roles and gets two bindings. The `writer` role, declared as `writer: {}`, is usable through `models.use('writer')`. The `reader` role needs a window of at least 200000 tokens, and a model whose context window equals a role's `min_context` meets it; only a smaller window falls short. A role with `min_context: 200000` therefore runs on a model whose window is exactly 200000 tokens.

Every declared role binds to the same current model, whatever keywords it has, so roles `scout` and `sprinter` with different keywords both bind to that one model. A label the prompt does not declare gets no binding.

A prompt whose only declaration is a role with no keys, `models: { writer: {} }`, passes prepare with its requirements satisfied when the Host supplies a model. A prompt with no `models:` key gives prepare no roles to bind or check, so no model requirement can stop it; a prompt whose frontmatter declares only `plugins: [web]` prepares with no model requirement to meet. When the Host sets no current model, prepare skips the role binding and the keyword and context checks, so no role is bound and no model requirement is reported.

### The model role check

The model role check is the comparison prepare makes, before the run starts, between each declared model role and the Host's current model. It checks the role's `min_context` and its hard keywords, `thinking` and `no-thinking`, and it reports the context minimum and each hard keyword separately.

````markdown
---
name: analysis
description: Needs a large context window and a thinking model.
runvil: 0
models:
  analyst:
    keywords: [thinking]
    min_context: 200000
---

# Analysis

## Main

```lua
return 'done'
```
````

On a model with a 32000-token window that never thinks, `analyst` falls short on both checks, and the Host refuses the run before any section runs. The run fails with run error kind `RequirementsUnmet`, and the notice opens with `the environment cannot satisfy this prompt:`, followed by one line per shortfall:

````text
the environment cannot satisfy this prompt:
- role 'analyst': requires a context of at least 200000 tokens; the current model provides 32000
- role 'analyst': requires 'thinking'; the current model's thinking capability is Never
````

Prepare reports each failed check on its own line. A role that meets its context minimum but fails a keyword gets exactly one line, and a role that fails both its context minimum and a keyword gets two lines, as `analyst` does here.

Whether a hard keyword is met depends on the current model's thinking capability:

| Keyword | `Never` | `Switchable` | `Always` |
|---|---|---|---|
| `thinking` | falls short | met | met |
| `no-thinking` | met | met, with thinking turned off | falls short |

`thinking` fails only on a model that never thinks, and `no-thinking` fails only on a model that always thinks, so a model whose thinking is switchable meets both hard keywords. A `no-thinking` role on a switchable model prepares and runs, with thinking turned off on its requests.

Prepare checks only the hard keywords `thinking` and `no-thinking`. Any other keyword on a role, such as `frontier`, `fast`, `small`, `creative`, or `chat`, records your intent and never causes a shortfall, even on a 32000-token model that never thinks.

### The two role lines

A context window below a role's `min_context` shows as this line, with both numbers as plain token counts:

````text
- role '{label}': requires a context of at least {min_context} tokens; the current model provides {window}
````

A failed hard keyword shows as this line, where `{mode}` is `Never` when `thinking` fails and `Always` when `no-thinking` fails:

````text
- role '{label}': requires '{keyword}'; the current model's thinking capability is {mode}
````

So a `writer` role with `keywords: [no-thinking]` on a model that always thinks gives `- role 'writer': requires 'no-thinking'; the current model's thinking capability is Always`. Each shortfall line in the notice starts with `- `, so for a role `reader` that needs 200000 tokens on a 32000-token model, the whole notice is the two lines shown in [How a run proceeds](#how-a-run-proceeds).

### Unmet requirements are recorded, not raised

An unmet requirement is one shortfall the model role check finds, recorded in the requirements report before the prompt starts rather than raised as an error. Because of that, a model that cannot serve a role never surfaces as an error partway through a run. A role `checker` that needs `no-thinking` on a model that always thinks gives exactly one unmet requirement, and the run is refused before any section has run.

Prepare checks and never repairs. A role that fails a keyword or context check stays bound to the current model, and prepare never looks for a different model to serve it.

## The requirements notice

After this section you can read any requirements notice line by line and tell what produced each line.

````markdown
---
name: deep-analysis
description: Needs two Plugins and a large thinking model.
runvil: 0
plugins: [web, user-input]
models:
  analyst:
    keywords: [frontier, thinking]
    min_context: 200000
---

# Deep analysis

## Main

```lua
return 'done'
```
````

Run this prompt on a Host that has not installed `web`, that has installed `user-input` but has no operator to ask, and whose current model has a 32000-token window and never thinks. The run fails with run error kind `RequirementsUnmet`, and its message is this requirements notice:

````text
the environment cannot satisfy this prompt:
- missing required Plugin: web
- user-input needs runvil/input-broker, and the environment provides none
- role 'analyst': requires a context of at least 200000 tokens; the current model provides 32000
- role 'analyst': requires 'thinking'; the current model's thinking capability is Never
````

The requirements notice is the text prepare writes when it refuses to start a run. When a prompt's requirements are unmet, the Harness refuses the run before any section or Lua runs, so nothing in the prompt can catch the refusal. Any unmet requirement, including an unmet model role requirement, makes the refusal a run error of kind `RequirementsUnmet` whose message is the notice, word for word. When nothing blocks the run, there is no refusal. The `frontier` keyword adds no line, because it is a soft keyword.

### The notice's fixed layout

The notice has a fixed layout: the opening line `the environment cannot satisfy this prompt:`, then one line starting with `- ` for each unmet requirement. Prepare does not stop at the first gap, so when several requirements are unmet you see all of them together in one notice, not only the first. Each line names what is missing or falls short, such as each missing service and each unavailable Plugin, and states the required value against the actual value where that applies.

The notice is written for a model to read: concise, factual, and self-contained. It names only the gaps that block the run, because the report behind it lists only what needs a person's attention. Each kind of line comes from one kind of gap:

| Notice line | What produces it |
|---|---|
| `- missing required Plugin: {name}` | a declared Plugin the Host has not installed |
| `- {plugin} needs {service}, and the environment provides none` | a declared Plugin whose service the run lacks |
| `- {plugin} is unavailable: {reason}` | a declared Plugin that is installed but cannot serve |
| `- role '{label}': requires a context of at least {min} tokens; the current model provides {actual}` | a role whose `min_context` is above the current model's window |
| `- role '{label}': requires '{keyword}'; the current model's thinking capability is {mode}` | a role whose hard keyword the current model fails |
| `- the H1 gate failed` | an uncaught Lua error in the H1 pass |

### Plugin lines in the notice

A Plugin declared under `plugins:` that the Harness does not have refuses the run with the line `- missing required Plugin: {name}`, such as `- missing required Plugin: web`.

A declared Plugin that is installed but needs a host service the Host lacks appears as `- {plugin} needs {service}, and the environment provides none`, naming both the Plugin and the missing service id. A Plugin missing several services gets one line for each missing service. The service name in the line is shown exactly as the Host names it, so service names can differ from one Host to another.

A declared Plugin that is installed but cannot serve, for example because it failed to build, appears as `- {plugin} is unavailable: {reason}`, with the Plugin id and the reason the environment gave, word for word. The reason is written for the model that reads the refusal, as in `- web is unavailable: web needs runvil/search-provider, and this host provides none`.

Each Plugin appears under its real cause only. A Plugin that lacks a host service is reported only under that service gap, never also as a missing required Plugin, whichever check finds it first. A Plugin that is installed but fails to build or activate is reported only as unavailable, with its reason, never as a missing required Plugin. So when a declared Plugin lacks a service or is unavailable, the notice leaves out its `missing required Plugin` line and names only the real cause.

### Role lines in the notice

Each unmet model role requirement gets its own line under the opening line, starting with `- role '{label}':` and naming the role by its label. The notice names every failed check across all roles, each on its own line with required against actual, so a role that fails both its context check and a keyword check gets two lines, as `analyst` does above.

When the bound model's context window is smaller than a role's `min_context`, set at `models.{label}.min_context`, the line is `- role '{label}': requires a context of at least {min} tokens; the current model provides {actual}`. The comparison is strict, so a model whose window equals the minimum passes.

An unmet hard keyword gives the line `- role '{label}': requires '{keyword}'; the current model's thinking capability is {capability}`. The capability reads `Never` for an unmet `thinking` and `Always` for an unmet `no-thinking`, capitalized as shown, and a `Switchable` model meets both keywords. A role with the `no-thinking` keyword on a model that always thinks gives `- role '{label}': requires 'no-thinking'; the current model's thinking capability is Always`.

Only `min_context` and the hard keywords `thinking` and `no-thinking` can add role lines. Soft keywords such as `frontier` or `fast` are never checked and never refuse a run. The notice lists only the roles that fall short and leaves out roles the model meets. When the Host sets no current model, the notice has no role lines, because prepare skips the role checks.

### The H1 gate line

The `- the H1 gate failed` line does not come from a refusal. An uncaught Lua error in the H1 pass fails the prompt's hard gate, which also ends the run with run error kind `RequirementsUnmet` before any later block runs. Its notice is the same opening line followed by `- the H1 gate failed`, and its message adds `: ` and the Lua error's text, as [The H1 hard gate](#the-h1-hard-gate) shows.

### The order of the lines

Notice lines appear in a fixed order: missing Plugins first, then missing services, then unavailable Plugins, then unmet model role requirements. Missing required Plugins come before Plugins that lack a host service, no matter which gap was found first. Within each kind of gap, the notice keeps the order in which the gaps were found. Role lines are listed in sorted label order, and within a role the context line comes first, then the keyword lines in the order the keywords are declared.

Each gap appears once. A Plugin gap that more than one merged report names appears only once in the notice. Each Plugin and missing service pair appears once even when it is reported twice, and every distinct pair is listed. Each unavailable Plugin appears once even when more than one check reports it, and every distinct unavailable Plugin is listed with its own reason.

### The requirements report

The requirements report is the preflight check behind the notice, made before the prompt starts. It combines the Harness's check that each declared Plugin can serve, made when the Harness takes its snapshot of installed Plugins, with the model role checks made while the run context is prepared, so one refusal names every gap. It names missing required Plugins, declared Plugins that lack a host service, declared Plugins that are unavailable, and unmet requirements.

Unmet requirements do not make prepare itself fail. Prepare still produces a requirements report, marked as not satisfied, and when prepare refuses to start the run, its refusal text is the notice built from that report. The report keeps one list per kind of gap:

- missing required Plugins
- missing services, one entry for each Plugin and missing service
- unavailable Plugins, each with its reason
- unmet requirements, one entry for each failed model role check

The report keeps unmet role checks apart from its list of missing required Plugins, which stays empty when only a role check fails. Each unmet role entry records:

- the label of the role that falls short, such as `checker`
- which check the role failed: `ContextMinimum` for the context minimum, or `HardKeyword` for a hard keyword
- the requirement the prompt declared, such as `no-thinking` from `keywords: [no-thinking]` or the `min_context` value `200000`
- the model's actual value, such as `32000`

### When a cancel comes first

When the run is cancelled before the Harness refuses it, for example while a declared Plugin is still starting, no notice is reported: the run ends as cancelled rather than failing with `RequirementsUnmet`, as [Waiting for Plugins and taking the snapshot](#waiting-for-plugins-and-taking-the-snapshot) explains.

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

Each section entry starts with its own section VM, the fresh Lua instance a section's blocks run in (see [Prologue, prose, and epilog](03-blocks-and-prose.md#prologue-prose-and-epilog)). The move between sections tears down the old section VM and clears its state, so a plain global such as `reply`, or a function such as `greet` defined in a `## helpers` section, is `nil` in a later section. Each section runs in its own frame, built when the section is entered and torn down when the section ends. What a section leaves on `var`, and what it writes with `store`, outlives the move: `## C` in the `walk` prompt reads both values set before it, and `## Second` in the `handoff` prompt reads the file `## First` wrote.

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

[The run result and outcomes](#the-run-result-and-outcomes) explains this `done` fallback.

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

Returning any other first value fails the block with `cannot return a {type} as a result`, where `{type}` is the Lua type name: `table` for a table, `function` for a function, and `userdata` for an Engine object. So a table return fails with a message that contains `cannot return a table as a result`. This is an ordinary Lua error, so in a walked section it fails the run with run error kind `Lua`. In the H1 pass the same error fails the [hard gate](#the-h1-hard-gate) instead.

The easiest way to hit this error is with a library section that ends by returning a table for `require`. When the walk reaches that section on its own, the walk applies the scalar return rule to it, and the run fails with run error kind `Lua` and `cannot return a table as a result`. Placing library sections after the section that returns keeps the walk from reaching them.

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

Prompt-wide settings made during the H1 pass stay in force for every later section, because the whole run shares them. That makes the H1 the place to choose the prompt-wide default with [`models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault):

````lua
models.default('writer')
````

With this block in the H1 body, the default stays in force for every section walked afterward, where `writer` is a [model role](10-models.md#model-roles-instead-of-models) the prompt declares. A default that a library sets while the H1 loads it with `require` stays visible to the walk the same way.

The H1 is also the place to validate and repair the prompt's parsed input in `argv` (see [Reading parsed input from `argv`](06-arguments.md#reading-parsed-input-from-argv)). An `assert` on `argv` checks it, and assigning a repaired `argv` table in the H1 body means every later section sees the repaired value.

A model round in the H1 pass reaches the model exactly once, even though sections run afterward. Model rounds happen in source order, so an H1 pass's `models.infer` call reaches the model before a later `##` section's round.

The H1 pass is its own section entry for offers. A Plugin the H1 offers with [`plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer) reaches only the H1 pass's own model rounds, and walked sections start without that offer. Checkpoints that H1 Lua records with [`log`](05-lua-environment.md#logging-and-lifecycle-reports), such as `log("H1 ran")`, are attributed to the H1 title.

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
- An uncaught Lua error in it, such as a failed `assert`, is the prompt's failed [hard gate](#the-h1-hard-gate), which fails the run with run error kind [`RequirementsUnmet`](#preparing-the-run) and stops later H1 blocks from running.

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

The run fails with run error kind `RequirementsUnmet`, and its message opens with these two lines. After `- the H1 gate failed: ` comes the Lua error's own text. Its first line names the failing block and its line in the prompt file and then gives the `assert` message `the gate cannot hold`, and a Lua stack traceback follows on the next lines. `## Work` never runs.

The Lua blocks under a prompt's H1 title are its hard gates. An `assert` there checks the prompt's arguments or prompt-wide settings, and when it fails, the run ends before any section is walked, with run error kind `RequirementsUnmet`. Move the same `assert` into the block under `## Work` and the run fails with run error kind `Lua` instead, because the same uncaught Lua error in a block under a walked section is an ordinary `Lua` failure.

Once the hard gate fails, the later blocks in the H1 body and every section never run, so a later H1 block's effects, such as a `store.write`, never happen. The failure still counts as the failing block's own: a `lua_chunk_failed` event is reported under the H1 title before the run ends.

### Failures that fail the gate

Any ordinary Lua error left uncaught in an H1 block fails the hard gate, and the run's error message ends with the Lua error text. That covers these cases:

- An `error` or `assert` call.
- A call to a function that does not exist.
- Running out of Lua memory.
- A `{{ }}` substitution that fails (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)).
- An `error(...)` inside a library that the H1 body loads with `require`. The library's raised message is kept as the cause, so a library section `## broken` that runs `error('library boom')` gives a cause containing `library boom`.
- A `call` that names no visible section.
- A `tasks.fanout` error that would otherwise end the run as `Lua`, such as a bad collection or an arm that failed with a `lua`-kind error (see [When an arm fails](14-fanout.md#when-an-arm-fails)).

Each case fails the gate only when it is left uncaught. A `call` that names no visible section raises its error at the call site, so you can catch it with `pcall`:

````lua
local ok, err = pcall(call, '## Nope')
return tostring(ok) .. ':' .. tostring(err)
````

As the only block in an H1 body, this block returns text that starts with `false:`, followed by an error naming `## Nope`, and that text is the run result.

Because `tasks.fanout` raises the failing arm's own error, an arm's tool or store failure keeps its own run error kind instead of becoming `RequirementsUnmet`.

### The failed gate's message

Run error kind `RequirementsUnmet` covers two cases: [prepare](#preparing-the-run) refusing the run, and the H1 pass failing its hard gate. The message tells them apart. A prepare refusal's message is only the [requirements notice](#the-requirements-notice), while a failed gate's message ends with the Lua error text.

A failed gate's error has a fixed notice, the line `the environment cannot satisfy this prompt:` followed by the line `- the H1 gate failed`, and it keeps the Lua failure as its cause. The message is the notice, then a colon and a space, then the cause's text, with the cause text appearing exactly once. So the assertion's message appears in the cause and not in the notice.

### Failures that keep their own kind

In the H1 pass only ordinary Lua errors turn into `RequirementsUnmet`, and every other uncaught failure keeps its own run error kind:

| Failure in the H1 pass | How the run ends |
| --- | --- |
| A tool failure | `Tool` |
| A `Quota`, `Completion`, `Binding`, or `ContextExhausted` failure | That same kind |
| A failed store call | `Vfs` |
| A claims conflict | `Determinism` |
| A task error | `Lua` |
| A Host cancel | The cancelled outcome |

Task errors stay `Lua` rather than becoming `RequirementsUnmet`. That covers an uncaught `tasks_live`, `task_not_owned`, or `task_consumed` error (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)), a delivered cancelled task's error, and a `task_not_owned` or `task_consumed` error raised again later with its `task` field present. A Host cancel during the H1 pass is not a failure at all, and the run ends with its cancelled outcome.

### Errors raised again after a catch

An error you catch with `pcall` in the H1 pass and raise again keeps the failure it came from when you raise it before the block makes another call that waits on the Harness, such as a model round, so its own run error kind holds. Raised again after such a call, some errors are rebuilt as an ordinary Lua error and fail the gate as `RequirementsUnmet`. The rebuilt errors are any error whose `kind` is `lua`, `internal`, `out_of_scope_tool`, `unbound_tool`, or `tasks_live`, and any `task_not_owned` or `task_consumed` error missing its `task` field.

### Failures outside the gate

Two checks run around the H1 blocks rather than inside them. They are not part of the gate, so each fails the run with run error kind `Lua`:

- When the H1 pass's `var` is read back and `var` turns out to have been replaced with another value, the run fails with an error naming `global was reassigned`. This happens even when the block returned a scalar that would otherwise end the run.
- A `jump` from the H1 pass whose heading matches no top-level section, or more than one, fails the run, because the jump's heading is checked after the H1 block has finished.

## The run result and outcomes

After this section you can say where a run's result text comes from and how a run ends when it does not complete.

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

The run result is the text a run produces, and a completed run gives back its result as a single text string. Here the section on the main walk returns the scalar `'plain'` from its Lua block, so the run completes with the final text `plain`.

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

The run's result comes only from a `return` in the H1 pass, a `return` on the main walk, or the `done` fallback. Any scalar that a section on the main walk returns becomes the run's result text, including these:

- `return 'done'` gives `done`.
- `return store.read('note.md')` gives the note's text.
- `return sys.when` gives a timestamp such as `2023-11-14T22:13:20Z`, and the run result is that string, byte for byte.
- `return tools.call('web/fetch', { url = 'https://example.com' })` gives the tool's reply.
- Values the block builds, such as `return var.answer`, `return r[1].text .. '|' .. r[2].text`, or `return err.kind .. '|' .. tostring(err)`.

The rule holds at every heading level the main walk reaches: when the top-level chain ends with `return 'x-value'` in a section `### X`, the run's final text is `x-value`.

A section's `return` value becomes the run result, not the model's last reply. A section that ends with `return 'ok'` after the model's final reply `done` gives the result `ok`. A section that calls a model without returning the reply adds nothing to the run's result, and only a reply that a block returns, such as `return models.infer(prose)` in the last block, becomes the result. Running [`models.loop`](11-conversations.md#starting-a-conversation) on a message list `msgs` and then returning `msgs[#msgs].content` makes the model's final text reply from the loop the run result, which is how the last block of the [shipped](01-what-a-prompt-is.md#shipped-example-prompts) `research-person` prompt ends.

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

### The three run outcomes

Every run ends in exactly one of three run outcomes:

- Completed, with its result text. A run that finishes ends with a completed outcome carrying its final text, and that completed outcome is recorded as the run's outcome.
- Cancelled, when the Host cancels the run. A cancelled run carries no result text and no error.
- Failed, with the error it failed with: a run error kind that names the cause, such as an uncaught Lua error, together with a self-contained message for people to read.

The first outcome that ends a run is the one reported, and a later error or cancel never replaces it. A cancel that lands while a requirements refusal is being recorded leaves the run failed with run error kind `RequirementsUnmet`.

### Cancelling a run

When the Host cancels a run, the run ends as cancelled. A cancel is reported as its own outcome and never as a failure, even when the cancel lands while a block waits on `return models.infer(prose)`. A cancel stops the run even while its Lua is busy in a loop such as `while true do n = n + 1 end`.

A cancel that lands while every chain is waiting takes effect at the run's next step, which tears down every chain, while Lua code that is already running sees the cancel as it runs. A run cancelled while a `tools.call` to a script tool is suspended ends with the cancelled outcome rather than a text result or an error. A model request cut short by a cancel is not reported as a failed model call.

A requirements refusal that happens after the run's cancel fired ends the run as cancelled instead of failed, as when a cancel cuts the wait short while a needed Plugin is still starting (see [Waiting for Plugins and taking the snapshot](#waiting-for-plugins-and-taking-the-snapshot)).

### Failed runs and run error kinds

A run error kind is the name a failed run reports for its cause, and a failed run reports it together with a message. Running a prompt fails with one of these two kinds most often:

- `Lua`, for an uncaught Lua error in a section.
- `RequirementsUnmet`, when the Harness cannot satisfy the prompt, such as when a required Plugin is missing, or when an H1 block fails the prompt's hard gate.

Any uncaught Lua error in a walked section fails the run with run error kind `Lua`. That includes `assert(false, 'message')`, a call to a function that does not exist, and a walked library section that returns a table, which fails with `cannot return a table as a result`. Calling `error('message')` in a section's Lua block stops the run on purpose, and the rest of the prompt never runs:

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

The run fails with run error kind `Lua` and a message containing `expected failure`. Other failures report their own run error kinds, such as `Tool` for a tool failure and `Vfs` for a failed store call, and [Failures that keep their own kind](#failures-that-keep-their-own-kind) lists the ones the H1 pass leaves unchanged.

### Runs that end before the walk

A run that ends during preparation still has a record. When the Harness refuses its requirements, the prompt fails to parse, or an [input file](02-file-structure.md#input-and-output-file-declarations) cannot be put in place, the run's record ends as failed with the cause as its message. The recorded kind is `RequirementsUnmet` for a refusal and `Parse` for a prompt that does not parse, so you can see why a run that never reached its sections failed.

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
| 1 | `parse_started` | `Prompt` |
| 2 | `lua_compilation_started` | `Only` |
| 3 | `lua_compilation_succeeded` | `Only` |
| 4 | `parse_succeeded` | `Prompt` |
| 5 | `run_started` | `Loop` |
| 6 | `section_started` | `Only` |
| 7 | `lua_chunk_started` | `Only` |
| 8 | `lua_chunk_succeeded` | `Only` |
| 9 | `lua_teardown_started` | `Only` |
| 10 | `lua_teardown_succeeded` | `Only` |
| 11 | `section_finished` | `Only` |
| 12 | `run_succeeded` | `Loop` |

The parse events come first, whether or not the parse succeeds (see [Parsing comes first](#parsing-comes-first)), and the run's own events then open with `run_started` under the H1 title `Loop`. `# Loop` has no H1 body, so the run goes straight from `run_started` to the first section's `section_started`.

### The order of run events

A run reports its progress as ordered [run events](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program). Each event's `kind` field spells its name in snake_case, and a run reports them in this order:

1. `run_started`.
2. When the H1 body has Lua blocks, the H1 pass's block events and teardown events.
3. For each walked section: `section_started`; `lua_chunk_started` and then `lua_chunk_succeeded` around each Lua block it runs, with that block's model and tool events, such as `model_turn_completed`, between them; one `lua_teardown_started` and `lua_teardown_succeeded` pair after its last block; and `section_finished`.
4. `run_succeeded`.

A prompt with no H1 blocks, such as `loop`, goes straight from `run_started` to the first section's `section_started`. A block that is waiting reports neither succeeded nor failed until it finally returns or fails.

A run that does not succeed ends its events with `run_failed` under the H1 title in place of `run_succeeded`. The only run-level events are `run_started`, `run_succeeded`, and `run_failed`, and there is no run-level cancel event, so a cancelled run also ends with `run_failed`, even though its outcome is cancelled.

### Which name an event carries

Every run event, error, and report names the section that produced it by its heading text without the leading hashes, so `# Boundaries` reports as `Boundaries` and `## Only` as `Only`. Run-level events and those from the H1 pass carry the prompt's H1 title, while those from a walked section name that section. That is why `loop` reports `run_started` and `run_succeeded` under `Loop`, around the events named `Only`.

The H1 pass reports no `section_started` or `section_finished` of its own, because it is not a walked section. Only its block and teardown events appear under the H1 title.

### Section boundaries

Each walked section reports `section_started` when the walk enters it and `section_finished` when it completes, both under the section's name. At a fall-through, the section that ends reports `section_finished` before the next section reports `section_started`, so sections `One` and `Two` report One started, One finished, Two started, Two finished.

A section that ends by jumping counts as completed too. It reports `section_finished` before the section the jump names reports `section_started`, the same as at a fall-through, so `## A` running `jump('## B')` reports A started, A finished, B started, B finished. A section whose `jump` names a heading that matches no visible section, or more than one, still reports `section_finished`, because the section completes before the heading is looked up. The run then fails on the unresolved heading, and for `jump('## Missing')` the error contains `not found`.

A section that ends the walk with `return` still finishes normally, as at a fall-through: its final `var` is read back and it reports `section_finished`. So `## Two` doing `return 'two-ran'` gives the result `two-ran` and a final Two finished.

### Events from a failing section

A walked section that raises an uncaught error, such as `error('expected failure')`, reports `section_started` and `lua_chunk_failed` but never `section_finished`. Earlier sections have already finished normally. The failing section's VM is still torn down exactly once, with one `lua_teardown_started` and `lua_teardown_succeeded` pair, and then the run reports `run_failed`.

In `second_fails`, that means `First` reports everything through its `section_finished`, then `Second` reports `section_started`, `lua_chunk_started`, `lua_chunk_failed`, and one teardown pair, and the run ends with `run_failed` under `Second fails`.

### Log checkpoints in the record

A `log` checkpoint is recorded under the name of the section whose Lua wrote it, in the order the run reaches it. Checkpoints from the H1 body come under the H1 title ahead of every walked section's, so a `log("shared loaded")` call in the H1 body records its checkpoint ahead of every section's checkpoints.

### Events from a run that ends early

When a run ends during preparation, the events recorded before it ended stay in its record and can still be read. A prompt that parsed and was then refused by the environment still has its parse events, ending with the parse-succeeded event.

### What a run's record holds

Each run of a prompt is recorded as its own run with its own run id, even when the prompt text is identical. A run's record holds these values:

| Value | What it holds |
| --- | --- |
| Run id | An id of its own, different for every run |
| `name` | The run's name as the Host session gives it, such as `session-1` |
| `prompt_hash` | A hash of the prompt text, written with the `sha256:` prefix that names its algorithm |
| Seed | A fresh seed drawn for this run |
| `started_at` | The run's start time, in UTC milliseconds since the Unix epoch |

The run's name is the run metadata's `name` and also the `execution` field of every event the run records, from its parse events through its last event. The same prompt text always yields the same prompt hash across runs.

Each run draws its own fresh [seed](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) from the operating system's cryptographic random source and records it at the start of the run, before anything else, so two runs of the same prompt get different seeds. The run's start time, `started_at`, is stamped once from the system wall clock when the run is prepared, because the Engine reads no clock of its own, and a later run never starts earlier. The Harness supplies both the seed and the start instant, drawing them before the prompt is parsed, and a prompt cannot set either one.

Two runs of the same prompt running at the same time stay separate. Each returns its own result, and each run's records hold only its own events, `log` checkpoints, and final `run_succeeded`.

### Repeating a run exactly

Two runs with the same seed, the same start instant, and the same Harness answers produce byte-identical text, including the same `sys.when`, the same seeded values, and the same untrusted envelope nonce. A replay runs the same prompt under the same seed, start instant, and answers, so it also issues the same [Harness work](01-what-a-prompt-is.md#effects-and-harness-work) as the original run and reports the same events, in the same order.
