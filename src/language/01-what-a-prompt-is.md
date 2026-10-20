# What a prompt is

A Runvil prompt is one Markdown file whose prose is meant for a model and whose Lua decides what happens to that prose. The Harness runs the file for a Host, the application that starts each run. This chapter names every part of a prompt file, shows the smallest prompts that run, and explains who does what during a run, so you know where each line you write belongs and what it can reach.

## A Markdown file that is a program

After this section you can name every part of a prompt file and say what Runvil does with each one.

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

## Say hi

Say hello.

```lua
return prose
```
````

Running this prompt ends with this run result (see [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes)):

````text
Say hello.
````

This file is a prompt file, which is a Markdown file with the `.md` extension. You write a Runvil prompt as that single Markdown file, and the file is also a program: the prose meant for a model sits right next to the Lua that decides what happens to it. Prose is the Markdown text you write for a model to read, such as `Say hello.` above. A Lua block is the code inside a fence that opens with a `lua` label, and it runs as one unit, such as `return prose` above.

### The parts of a complete prompt

A complete prompt is one `.md` file that parses and runs. The prompt above is complete and combines every part a prompt can have: YAML frontmatter, an H1 title, a `##` section heading, a Lua block, and prose. In it:

- The frontmatter is the YAML between the two `---` lines.
- The H1 title is `# Greeter`, the prompt's one level-1 heading.
- `## Say hi` is the heading of a section, which is a heading plus everything under it up to the next heading at the same or a higher level.
- `Say hello.` is prose.
- `return prose` is a Lua block.

The shipped `analyst-example.md` prompt has the same parts with more in its section: frontmatter, the H1 title `# Analyst Example`, an `## Analyze` section, and prose between two Lua blocks.

### Three parts in a fixed order

A prompt file has three parts, always in this order, which is also the order Runvil reads them:

1. Frontmatter, the YAML block at the top of the file.
2. Exactly one H1 title with non-empty text, followed by an optional H1 body of prose and Lua blocks (see [The H1 title and H1 body](02-file-structure.md#the-h1-title-and-h1-body)).
3. Any number of `##` sections, including none.

Because sections are optional, a prompt whose H1 body does all the work is complete and runs. [The smallest complete prompt](#the-smallest-complete-prompt) shows one.

### Frontmatter comes first

A prompt file opens with its frontmatter. The very first line of the file is `---`, YAML follows, and the frontmatter ends at the next `---` line. This is the frontmatter of the prompt above on its own:

````markdown
---
name: greeter
description: says hi
runvil: 0
---
````

The Host can read all of the frontmatter, such as `name` and `description`, as soon as the file is parsed and before any run starts.

Two mistakes stop parsing at this point. A file whose first line is something other than `---` fails with `file must begin with a --- frontmatter delimiter`, and frontmatter with no closing `---` line fails with `frontmatter was not closed with ---`.

### Lua decides what happens to prose

You put logic in Lua blocks and text meant for a model in prose. A Lua block below the prose reads that text as `prose` (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)). The block can hand the text back as the run result with `return prose`, as the first example does, or send it to a model with `models.infer(prose)` (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)).

Prose alone never reaches a model, and nothing reaches a model until Lua sends it with `models.infer`. Prose that no Lua reads, whether in the H1 body or in a section that holds only prose, is discarded at the end of that part without needing a model, and the run continues:

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

This line sits in the H1 body, and no Lua reads it.

## Notes

This section holds only prose.

## Say hi

```lua
return 'ok'
```
````

````text
ok
````

The line in the H1 body and the whole `## Notes` section are discarded unread, no model is needed, and the run goes on to `## Say hi`, whose block returns `ok` as the run result.

### Runs and run events

A run is one execution of one prompt, from its start to its run result. A run reports a run event for each thing that happens during it, so the Host can log them.

Runvil parses a prompt file into a parsed prompt before anything runs. A parsed prompt is the prompt file read into its frontmatter, H1 title, and sections, and each run executes exactly one parsed prompt.

### The runvil version

Every prompt that runs declares `runvil: 0` in its frontmatter. This line is the runvil version, and it marks the file as a Runvil prompt. `0` is the only version Runvil runs.

A file that leaves the key out still parses, but its run stops before any section runs. The error has the parse error kind `Structure` (see [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds)), names the prompt's `name`, and carries the message `not a runvil prompt: no runvil version`. [Leaving out the runvil version](#leaving-out-the-runvil-version) shows this case.

## The smallest complete prompt

Here you build the smallest prompt that runs, then grow it one part at a time.

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter
````

````text
done
````

This is the smallest prompt that runs: frontmatter with `name`, `description`, and `runvil: 0`, plus one H1 title with nothing under it. It parses with no sections, and the run ends with `done`, the default run result when nothing returns a value. A line of prose under the title gives the same result, because nothing reads that prose.

### Lua in the H1 body

Add a Lua block under the title:

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

```lua
return "hello"
```
````

````text
hello
````

A complete prompt needs no `##` section. Frontmatter, one H1 title, and a Lua block in the H1 body run on their own, and the block's return value is the run result. With no return, the run ends with `done`. Replace the block's code with a line that returns nothing:

````lua
local x = 1
````

````text
done
````

### A section holding prose

Now move the work into a section. This prompt has one `##` section holding only a line of prose:

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

## Say hi

Say hello.
````

````text
done
````

Running it makes no model round, which is one request to a model and its reply. The unread prose `Say hello.` is dropped without error, and the run ends with the default run result `done`.

### A section holding Lua

Replace the prose with a Lua block:

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

## Say hi

```lua
return 'hello'
```
````

````text
hello
````

The block's return value is the run result. This prompt declares no model roles (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)), no tools, and no Packs, so it runs on Lua alone: the block's `return` is the whole result, and no model round happens.

### Leaving out the runvil version

Frontmatter with only `name` and `description` still parses, but the run refuses the prompt:

````markdown
---
name: greeter
description: says hi
---

# Greeter

## Say hi

```lua
return 'hello'
```
````

````text
not a runvil prompt: no runvil version
````

The run stops before `## Say hi` or any other section runs. The error has the parse error kind `Structure`, names the prompt `greeter`, and carries the message above, so every prompt that runs declares `runvil: 0`.

## Putting the parts together

Here you walk one complete prompt from its frontmatter to its run result and see how prose and Lua in one section work together.

````markdown
---
name: greet
description: Has the model repeat a greeting.
runvil: 0
models:
  writer: {}
---

# Greet

```lua
models.default("writer")
```

## Main

```lua
var.greeting = "Hello, " .. args .. "!"
```

Repeat exactly, with no extra words: {{ var.greeting }}

```lua
return models.infer(prose)
```
````

Run with the argument `Ada`, the model receives this text, and its reply is the run result:

````text
Repeat exactly, with no extra words: Hello, Ada!
````

### The frontmatter and the contract keys

The frontmatter holds `name`, `description`, and `runvil: 0`, plus a `models:` key. `uses:`, `models:`, and `args:` are the contract keys, three optional keys that state what the prompt needs from its run:

- `uses:` names the Packs the run requires (see [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key)).
- `models:` declares model roles, such as the `writer` role here.
- `args:` types the prompt's arguments (see [Declaring arguments in frontmatter](06-arguments.md#declaring-arguments-in-frontmatter)).

The parser checks only the shape of these keys. Prepare (see [Preparing the run](04-how-a-prompt-runs.md#preparing-the-run)) satisfies them against the run's environment.

### Setup Lua in the H1 body

Under `# Greet`, a Lua block calls `models.default("writer")` (see [The prompt-wide default with `models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault)), which makes the `writer` role the prompt-wide default.

### Prose and Lua in one section

`## Main` holds a Lua block, a line of prose, and a second Lua block. The first block computes a value and stores it in `var` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)) with `var.greeting = "Hello, " .. args .. "!"`, where `args` holds the argument string (see [The argument string in `args`](06-arguments.md#the-argument-string-in-args)).

The prose then uses the placeholder `{{ var.greeting }}` (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)), which renders the stored value into the text. The last block passes that prose to `models.infer(prose)`, and `return` makes the model's reply the run result. Data moves from Lua, through prose, to the model.

The shipped `greet.md` prompt has this shape, with setup Lua under `# Greet` and a `## Main` section that ends in `return models.infer(prose)`.

### Calling a tool from a section

A prompt with frontmatter, an H1 title, and one `##` section holding a Lua block is complete, and that block can call a tool with `tools.call` (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls)). The tool can be one the section defines itself with `tools.offer_local` (see [Defining local tools with `tools.offer_local`](12-tools.md#defining-local-tools-with-toolsoffer_local)). A greeter prompt with a `# Greeter` title and a `## Greet` section ends its block with these lines, where `reply` holds a model's reply:

````lua
tools.offer_local('shout', 'Returns the text in capital letters.', { text = 'string' }, function(args)
  return string.upper(args.text)
end)
return tools.call('shout', { text = reply })
````

The first call defines a tool named `shout` that answers with its `text` argument in capital letters, and the last line passes the model's reply to it. When the model replies `hi there`, the run ends with:

````text
HI THERE
````

The tool's answer is the block's return value, so it becomes the run result.

### Declaring a Pack

A prompt that uses a Pack lists it under `uses:`. This frontmatter declares the `web` Pack:

````markdown
---
name: researcher
description: Researches a topic on the web.
runvil: 0
uses:
  - web
---
````

The rest of a complete Pack-using prompt is the same shape as before: an H1 title and one `##` section holding a Lua block.

### A prompt that sends prose to a model

A prompt that sends prose to a model needs one model role in its frontmatter, a default set in its H1 body, and a section that infers from its prose:

````markdown
---
name: hello
description: Says hello.
runvil: 0
models:
  writer: {}
---

# Hello World

```lua
models.default("writer")
```

## Greet

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

The frontmatter declares one model role, `writer: {}`, under `models:`. The H1 body makes `writer` the default with `models.default("writer")`, and the `## Greet` block sends the section's prose to the model with `return models.infer(prose)`, so the model's reply to `Say "Hello, world!"` is the run result. The shipped `hello.md` prompt, which describes itself as `A minimal test prompt.`, has this shape.

### A prompt with no contract keys

A prompt that declares none of the contract keys passes prepare and runs on its Lua alone, with no model and no tool:

````markdown
---
name: greeter
description: Writes a note to the store and reads it back.
runvil: 0
---

# Greeter

## Greet

```lua
store.write('note.md', 'hello')
return store.read('note.md')
```
````

````text
hello
````

Its frontmatter holds only `name`, `description`, and `runvil: 0`. The one `## Greet` block writes a note to the store (see [What the store is](09-the-store.md#what-the-store-is)) and reads it back, and the run result is `hello`.

### Store, model, and tool in one block

One block can combine store calls, a model round, and a tool call to produce its result. This is the whole `## Greet` block of the greeter prompt from [Calling a tool from a section](#calling-a-tool-from-a-section):

````lua
store.write('note.md', 'hello')
models.use('writer')
local reply = models.infer(store.read('note.md'))
tools.offer_local('shout', 'Returns the text in capital letters.', { text = 'string' }, function(args)
  return string.upper(args.text)
end)
return tools.call('shout', { text = reply })
````

When the model replies `hi there`, the run ends with:

````text
HI THERE
````

The block writes `hello` to `note.md` in the store, selects the `writer` role with `models.use` (see [Selecting a role per section with `models.use`](10-models.md#selecting-a-role-per-section-with-modelsuse)), sends the note's text to the model with `models.infer`, defines the `shout` tool, and passes the reply to it. The tool's answer, the model's reply in capital letters, is the run result.

### A helper section and a nested section

One prompt can combine a section that other sections load, prose in the H1 body, several `##` sections, and a nested section. In a prompt titled `# Require Library`, the H1 body holds prose and a `## helpers` section defines a function named `normalize`. The `## Prepare` and `## Finish` sections each load it with `require '## helpers'` (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)) and call `normalize`. A `### Author note` section of prose nests under `## Prepare`, so the prompt has three `##` sections and `## Prepare` holds one nested section.

### Sections that run in sequence

A prompt can run a fixed sequence of sections, each one handing its result to the next until the run ends. A drafting prompt can write a draft with prose and `models.infer` in `## Draft` and keep it in `var.draft`, refine it in `## Refine` with a `models.loop` that uses tools (see [Starting a conversation](11-conversations.md#starting-a-conversation)), and hand the refined reply to `## Deliver` with `call` (see [Calling a section](08-jump-and-call.md#calling-a-section)). `var` and the value `call` passes carry results from one section to the next, and the run ends with `delivered: refined text`.

## The Engine, the Harness, and the Host

Here you learn which of three pieces does each job in a run, and which run inputs a prompt can read.

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

## Say hi

```lua
return 'hello'
```
````

````text
hello
````

When a Host starts a run of this prompt, the Harness hands the file's text to the Engine, the Engine parses it, and the Harness runs it. Every prompt follows that path: it is one Markdown (`.md`) file that you write, the Engine parses, and the Harness runs.

### What each piece does

- The Engine parses the prompt and carries out its run.
- The Harness runs the prompt and does the run's outside work. It prepares each run and then drives it: it steps the Engine's run, does each piece of outside work the run asks for, and resumes the run with the answer.
- The Host is the application that starts each run. Workshop is one Host.

### Run policy and cancelling

The Host launches runs and sets run policy, the settings that govern a run, such as which model the run uses and whether anyone can answer `input.ask()` (see [Asking the operator](05-lua-environment.md#asking-the-operator)). The Host can also cancel a run.

### Packs and the tool catalog

A Pack is a bundle of tools, such as the web tools, that can also bring a Lua prelude. A Plugin is an entry point that adds zero or more Packs. The Host calls its Plugins once, when it sets up, and every Pack they add is built then and shared across runs. The Harness takes each run's tool catalog, the set of tools that run can use, from a snapshot of the run's Packs: every `runvil` Pack, plus each third-party Pack the prompt declares. The prompt itself installs no Packs, and it names the ones it needs under `uses:`.

### Seed, start time, and repeatable runs

The Host names each run. The Harness draws the run's seed, a random value, from a secure random source, and takes the run's start time from the clock. A run given the same seed, the same start time, and the same outside answers repeats exactly.

### Run inputs a prompt can read

A prompt can observe three run inputs, the values from outside that shape a run. This prompt reads two of them:

````markdown
---
name: run-inputs
description: Reads the start time and an untrusted envelope.
runvil: 0
---

# Run inputs

## Only

```lua
return sys.when .. '|' .. untrusted('data')
```
````

The run result is the run's start time in RFC 3339 form, a `|`, and the envelope that `untrusted('data')` returns, which carries a nonce of 32 hex digits drawn from the seed. The three run inputs are:

- The seed, which a prompt sees through the nonce in each `untrusted()` envelope (see [Handing store text to a model with `untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted)).
- The start time, read as `sys.when` (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)).
- The Host's UI snapshot, served by `ui()`.

When the run has no UI snapshot, `ui` is `nil`. In that run, this block returns `nil`:

````lua
return type(ui)
````

````text
nil
````

### Stopping a model round

Besides cancelling a run, the Host can stop the model round in flight, which is called stopping a round. The dropped round raises an error value whose error kind is `cancelled` (see [Error values](05-lua-environment.md#error-values)), and `pcall` can catch it so the run goes on (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

````lua
local ok, err = pcall(writer.infer, writer, 'Take your time.')
if not ok then
  return err.kind
end
````

Here `writer` is a model handle (see [Model handles](10-models.md#model-handles)). When the Host stops the round while it is in flight, the block catches the error and the run completes with:

````text
cancelled
````

An uncaught `cancelled` error ends the run as cancelled.

## Effects and Harness work

Here you learn which lines of a prompt wait on the outside world and which run as plain Lua.

````markdown
---
name: greeter
description: Writes a note to the store and reads it back.
runvil: 0
---

# Greeter

## Greet

```lua
store.write('note.md', 'hello')
return store.read('note.md')
```
````

````text
hello
````

Each store call in this block is an effect. Every model round, every call to a Pack's tool, and every store read or write that a prompt makes is an effect, and the Harness performs every effect outside the run.

### How an effect is answered

A prompt never does its own outside work. The block that asks for an effect waits while the Harness performs it, then resumes with the Harness's answer. In the prompt above, the block waits on `store.write`, resumes, waits again on `store.read`, and resumes with `hello`, which it returns.

The Engine parses the prompt, steps the run, and emits an effect whenever the run needs something from outside, such as a model reply or a tool result. The Harness performs each effect, hands the answer back to the Engine, and records the run through the Host's recorder, which keeps the Host's record of each run.

### Harness work is the only way out

A prompt reaches the outside world only through Harness work, meaning requests that the Harness performs for the run and answers. The set of effect kinds is closed, so a prompt has no other path outside.

The language itself performs no I/O and reads no clock. Inside a block, `io` and `os` are `nil`:

````markdown
---
name: no-io
description: Shows that io and os are absent.
runvil: 0
---

# No io

## Only

```lua
return tostring(io) .. ',' .. tostring(os)
```
````

````text
nil,nil
````

A block's Lua has no `io` or `os` library, so it reaches the outside world only through Harness work, and it reads the time only as the start time in `sys.when`.

### The four kinds of Harness work

There are exactly four kinds of Harness work:

- A model round.
- A call to a Pack's tool.
- A store operation.
- The timer behind a timed wait on tasks (see [Time limits on waits](15-tasks.md#time-limits-on-waits)).

A model round is one request to a model and its reply. The Harness answers it with the model's reply or with the reason the round failed.

Asking the operator with `input.ask()` is a call to a Pack's tool. The operator is the person who answers `input.ask` during a run. The user-input Pack defines `input.ask` to end in `return tools.call(pack .. "/ask")`, so every ask reaches the Harness as an ordinary tool call, and the wait for the operator's answer is Harness work.

Every store call, even a read, is a store operation that the Harness performs. That covers all eight `store.*` calls a prompt can make (see [Writing and reading files](09-the-store.md#writing-and-reading-files)), and the block that made the call waits for its answer.

### Every effect gets one answer

The Harness answers every effect exactly once, either with its result or with its failure, such as the reason a model round, a tool call, or a store operation failed. A round the Host stops is answered with its failure, as [Stopping a model round](#stopping-a-model-round) shows.

### Work that needs no Harness

Everything else a block does, such as computing and returning a value, is plain Lua that needs no Harness work. The greeter prompt from [A section holding Lua](#a-section-holding-lua), whose `## Say hi` block is `return 'hello'`, ends with `hello` without issuing any effect.

A call to a local tool needs no Harness work either. A local tool is a Lua function that a section defines with `tools.offer_local` (see [Defining local tools with `tools.offer_local`](12-tools.md#defining-local-tools-with-toolsoffer_local)). The Engine runs its handler inside the calling block, so in a prompt whose local tool `grab` answers `got ` followed by its `value` argument, the call `tools.call('grab', { value = 'hi' })` returns `got hi` and the run issues zero effects.

## Shipped example prompts

Here you find the example prompts that ship with Runvil, so you can open a working file for each idea this chapter introduces.

The smallest of them, `echo.md`, does all its work in one Lua block under `## Main`:

````lua
return args
````

Run with the argument `hello`, it ends with:

````text
hello
````

Every prompt shipped in the repository's `prompts` directory, including the prompts in its subdirectories, parses as written, so you can open any of them as a working example. The directory ships these small examples:

- `echo.md` returns its argument unchanged with `return args` and makes no model round; [The argument string in `args`](06-arguments.md#the-argument-string-in-args) teaches `args`.
- `hello.md` has the model say hello through the prose `Say "Hello, world!"` and `return models.infer(prose)`; [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer) teaches `models.infer`.
- `greet.md` builds a greeting from its argument with `var.greeting = "Hello, " .. args .. "!"` and has the model repeat it through `{{ var.greeting }}` in its prose; [Placeholders in prose](07-substitution.md#placeholders-in-prose) teaches placeholders.
- `analyst-example.md` declares an `analyst` model role under `models:` and calls `models.use("analyst")` before `return models.infer(prose)`; [Model roles instead of models](10-models.md#model-roles-instead-of-models) teaches model roles.

Runvil also ships `research-person.md`, which researches a person. Its `## Research` section calls `tools.offer({"web/search", "web/fetch"})` (see [Offering tools with `tools.offer` and `packs.offer`](12-tools.md#offering-tools-with-toolsoffer-and-packsoffer)), and the prompt runs end to end to a factual summary. [What the web Pack gives a prompt](13-web-fetch-and-search.md#what-the-web-pack-gives-a-prompt) teaches `web/search` and `web/fetch`.

The research prompt takes the person to research as its argument. Its prose puts `{{ args }}` after the line `Your input is a request about a person:`, so a run given the argument `Ada Lovelace` places that name into the prose right after that line.
