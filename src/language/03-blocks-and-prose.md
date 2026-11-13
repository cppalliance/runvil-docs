# Blocks and prose

The H1 body and every section body of a prompt are made of two kinds of blocks that alternate in file order: Lua blocks, which hold the prompt's logic, and prose blocks, which hold Markdown text. This chapter shows how you write each kind, which prose a Lua block can read, how [`prompt.section`](#reading-sections-with-promptsection-and-promptsection_or) reads another section's text, a list, or a fenced template, and how a library shares its code through [`require`](#sharing-code-with-require). Knowing where each block starts and ends tells you exactly which Lua runs and which text can reach a model.

## Lua fences and prose blocks

You can write `lua` fences and prose in the H1 body and in any section, and tell which blocks they become.

````markdown
---
name: fences
description: One Lua block between two prose blocks
runvil: 0
---

# Fences

## Main

Before.

```lua
return 'hello'
```

After.
````

````text
hello
````

The section `Main` holds three blocks in file order: a prose block `Before.`, a Lua block, and a prose block `After.`. The Lua block returns `hello`, which becomes the run result (see [Returning values](04-how-a-prompt-runs.md#returning-values)), and neither prose block runs anything or calls a model.

A Lua fence is a fenced block that opens with a line of three backticks followed by `lua` and closes with a line of three backticks, and you usually place one under a section heading. A Lua block is the code one Lua fence holds, run as one unit (see [A Markdown file that is a program](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program)). A prose block is a run of Markdown text outside the fences: before the first fence, between two fences, or after the last one.

### Where Lua and prose go

Lua lives only inside `lua` fences. Everything outside them, in the H1 body or in a section, is Markdown prose, so a fence with any other label, such as `python`, stays part of the prose. The exact shape a Lua fence must have is in [Exact fence rules and fence errors](#exact-fence-rules-and-fence-errors).

Each section holds Markdown prose, `lua` fences with the prompt's logic, or both. The H1 body and each section body can alternate fences and prose freely and in any number: each fence becomes a Lua block, and each run of text between fences becomes a prose block. Blocks keep their file order, so prose blocks and Lua blocks alternate exactly as you write them.

Blocks fill two places in a prompt file: the H1 body, which is everything between the H1 title and the first `##` heading (see [The H1 title and H1 body](02-file-structure.md#the-h1-title-and-h1-body)), and every section. You fill both in the same way:

````markdown
---
name: plan
description: Lua blocks and prose in the H1 body
runvil: 0
---

# Plan

```lua
local first = 1
```

Plan the work in two steps.

```lua
local second = 2
```

## Main

```lua
return 'planned'
```
````

````text
planned
````

An H1 body holds any mix of plain `lua` fences and prose paragraphs before the first `##` section, and each becomes its own block in file order, so this H1 body holds a Lua block, a prose block, and a Lua block. You can place a fence directly under the H1 title, as here, or inside a section such as a `##` section, and a Lua block is one `lua` fence in either place.

You typically write an explanatory paragraph under the H1 title, before the first section, to describe what the prompt does, such as `Returns the single input argument verbatim, entirely in Lua, with no model call.` under `# Echo`. Prose written under the `#` title is kept as an H1 prose block. An H1 title with no content under it, or only whitespace, has no blocks at all, and an H1 body holding only one `lua` fence is valid and has a single Lua block.

### Writing Lua inside a fence

Inside a fence you write ordinary Lua statements, such as the `local` declaration `local first = 1` or a function definition. This fragment is one section of a prompt:

````markdown
## Setup

```lua
function normalize(value)
    return string.lower(value)
end
```
````

A Lua block's source is exactly the text between its opening and closing fence lines. The fence lines are left out, multiple lines and inner indentation are kept as written, and trailing line breaks are dropped, so the source of this block is the three lines from `function normalize(value)` to `end`, with the four-space indent of the middle line kept.

### Prose blocks

You write a section's instruction as plain prose under its heading, such as `Say "Hello, world!"` under `Greet`. Double quotes, commas, and exclamation marks go in verbatim, without escaping, and a paragraph may wrap across several source lines.

Which text becomes a prose block follows a few rules:

- When anything other than blank lines comes before the first `lua` fence in the H1 body or a section, that text becomes its own prose block ahead of the first Lua block. Blank or whitespace-only lines alone before the fence produce no block.
- Prose after the last `lua` fence becomes a trailing prose block only when it is not empty after trimming. Markdown after the last fence is never an error and stays an ordinary prose block.
- Every prose block is trimmed of leading and trailing whitespace.
- Apart from that trimming, a prose block holds the prompt file's Markdown as written, so text like `Plan {{ args }}.` keeps its `{{ args }}` placeholder (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)) exactly as written when the prompt loads.
- A section with no `lua` fence is a single prose block.

A section made only of prose is valid, and the prompt still loads. It can sit beside sections that use fences:

````markdown
---
name: greeting
description: A prose-only section beside a section with a fence
runvil: 0
---

# Greeting

## Fallback

This section has prose only.

## Greet

Say "Hello, world!"

```lua
return 'greeted'
```
````

This prompt loads cleanly. When the run reaches `Fallback`, the section runs no Lua and makes no model call.

### When Lua blocks compile and run

Every Lua block, in the H1 body or in a section, is compiled as its own Lua program when the prompt loads, before any block runs. Inside a section, every fence labeled exactly `lua` is compiled wherever it sits around the section's prose: a fence before the prose becomes the section's prologue and a fence after it becomes the epilog, as [Prologue, prose, and epilog](#prologue-prose-and-epilog) explains, so a section with a prologue and an epilog compiles two programs.

When the run reaches a section (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)), its blocks run in file order. Each Lua block runs when its place in that order comes, and a prose block runs nothing and calls no model on its own.

Because compiling happens at load, a Lua syntax error in any block, in the H1 body or a section, fails the prompt load with parse error kind `Lua` before any block runs:

````markdown
---
name: transform
description: An epilog with a Lua syntax error
runvil: 0
---

# Draft

## Transform

Ask the model for a response.

```lua
local reply =
```
````

The message starts with `lua compilation error at {location} (line {source_line}): ` and ends with the Lua compiler's own diagnostic. `{location}` is the block's location label, such as ``H1 `{title}` lua`` for a block in the H1 body, and here a label containing ``section `Transform` epilog``. `{source_line}` is the line of the prompt file where the block's Lua source starts, here line 14. [How errors name each block](#how-errors-name-each-block) lists every location label. The parse error also points at line 14, the line the compiler's diagnostic names, and names the prompt `transform` (see [Lua compile failure positions](16-limits-and-errors.md#lua-compile-failure-positions)).

## Prologue, prose, and epilog

You can put Lua before and after a section's prose, and know what each position can do and when it runs.

````markdown
---
name: tides
description: A prologue, prose, and an epilog in one section
runvil: 0
---

# Tides

## Explain

```lua
var.topic = 'tides'
```

Explain {{ var.topic }} in one sentence.

```lua
return models.infer(prose)
```
````

In `Explain`, the first fence is the section's prologue, the prose after it is the section prose, and the second fence is the epilog, so the block sequence is exactly Lua, prose, Lua. The prompt has no `models:` block, so the epilog's `models.infer` runs on the model of the `default` role, which is the model the Host supplies (see [The current model](10-models.md#the-current-model)). The model is asked `Explain tides in one sentence.`, and its reply is the run result.

This is the common section shape. The prologue runs before the prose and stores a value in `var` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)). The prose uses that value through a `{{ var.topic }}` placeholder. The epilog runs after the prose, reads the pending prose through `prose`, and sends the finished prose to a model with `return models.infer(prose)` (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)).

A prologue is a `lua` fence that opens a section, before any prose, and it runs before the prose. Blank lines before it are fine: blank lines and lines holding only spaces or tabs between the section heading and the first fence are discarded and never stop that fence from being the prologue. The prose after a prologue stays a separate prose block, and if no fence follows that prose, the section has no epilog.

An epilog is a `lua` fence after a section's prose, the trailing Lua block that ends the section, and it runs after the prose. When no fence comes before the prose, the section has no prologue. A section can have a prologue alone, an epilog alone, or both, and when it has both, the prologue runs first.

### Section shapes

Where the fences sit decides which blocks a section has and which of them are the prologue and the epilog:

| Section content | Blocks | Prologue | Epilog |
| --- | --- | --- | --- |
| Prose only | one prose block | none | none |
| One fence only | one Lua block | the fence | none |
| A fence, then prose | Lua, prose | the fence | none |
| Prose, then a fence | prose, Lua | none | the fence |
| A fence, prose, a fence | Lua, prose, Lua | first fence | second fence |
| Two back-to-back fences | Lua, empty prose, Lua | first fence | second fence |
| Prose, a fence, prose | prose, Lua, prose | none | none |
| Fences with prose between them | Lua, prose, Lua, and so on | first fence | last fence |

A section with no `lua` fence is a single prose block with no prologue and no epilog. A section whose only content is one `lua` fence, including a library section (see [Sharing code with `require`](#sharing-code-with-require)), treats that fence as its prologue and has no epilog, because an epilog needs prose before it, even empty prose. A `lua` fence with prose on both sides becomes its own Lua block in the middle of the section and is neither the prologue nor the epilog. Blank lines before a leading fence and empty text after the last fence produce no block.

### Back-to-back and empty fences

Text between two `lua` fences becomes a prose block, and two back-to-back fences with no prose between them still get an empty prose block. Both fences run, as the prologue and the epilog around that empty prose, in order and with no model call:

````markdown
---
name: phases
description: A prologue and an epilog around empty prose
runvil: 0
---

# Phases

## Main

```lua
var.phase = 'prologue'
```

```lua
return var.phase .. '-epilog'
```
````

````text
prologue-epilog
````

The same holds when only blank lines or lines of spaces and tabs sit between the two fences. Prose made only of spaces, tabs, and blank lines is trimmed to empty, so the next fence reads `prose` as an empty string.

A prologue does not have to do anything. An empty `lua` fence, an opener followed directly by its closer, is valid: it still produces a Lua block, with empty source, that does nothing, and it can stand as a section's prologue. A fence that holds only a Lua comment, such as `-- prologue`, is a valid prologue too:

````markdown
---
name: quiet
description: A comment-only prologue before an epilog
runvil: 0
---

# Quiet

## Main

```lua
-- prologue
```

```lua
return 'ok'
```
````

````text
ok
````

You can leave the prologue fence empty and do the section's work in the epilog, which then runs with empty prose, for example an epilog that calls `tasks.fanout` to run a worker section (see [Choosing the worker section](14-fanout.md#choosing-the-worker-section)).

### Several fences in one section

You can place several `lua` fences in one section, interleaved with prose, to split the section's work. They run top to bottom in file order as separate blocks of that one section. In a section that opens and closes with a fence, the first fence is the prologue, the last fence is the epilog, and every fence between them is neither:

````markdown
---
name: tour
description: Two questions in one section, the second using the first reply
runvil: 0
---

# Tour

## Landmark

```lua
var.city = 'Lisbon'
```

Name one landmark in {{ var.city }}.

```lua
var.landmark = models.infer(prose)
```

Write one sentence about {{ var.landmark }}.

```lua
return models.infer(prose)
```
````

The prologue sets `var.city`, and the middle fence sends `Name one landmark in Lisbon.` to the model and stores the reply in `var.landmark`. The prose after it reads `{{ var.landmark }}`, so the second question uses the first reply, and the epilog sends that second question and returns the reply. The section makes two model rounds, one for each `models.infer` call.

The H1 body works the same way: it can hold several Lua blocks interleaved with prose blocks, and they run in file order as one H1 pass (see [The H1 pass](04-how-a-prompt-runs.md#the-h1-pass)).

### One section VM per section entry

Each section entry, the H1 body's blocks, and every fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)) run in a section VM, a fresh Lua instance of its own. Several Lua blocks in one section entry run one after another on the same section VM, so a bare global that one Lua block assigns, called a section global, stays visible to later Lua blocks of the same section entry:

````markdown
---
name: greeting
description: A section global shared by two fences
runvil: 0
---

# Greeting

## Main

```lua
greeting = 'hello'
```

world

```lua
return greeting .. ', ' .. prose
```
````

````text
hello, world
````

Other state persists from one fence to the next in the same section entry too, such as the tools an earlier fence offers and the call counts in `tools.stats` (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls)).

### What a prologue and an epilog can do

A prologue prepares values for the prose, and an epilog acts on the finished prose. One section can hold all three together: the prologue runs before the prose, and the epilog runs after it and reads the pending prose through `prose`. An epilog can read `prose`, call the model, and write to the store (see [Writing and reading files](09-the-store.md#writing-and-reading-files)):

````markdown
---
name: save
description: An epilog that saves the model reply to the store
runvil: 0
---

# Save

## Ask

Name one ocean.

```lua
local reply = models.infer(prose)
store.write('reply.txt', reply)
return 'saved'
```
````

````text
saved
````

The epilog sends `Name one ocean.` to the model, writes the reply to `reply.txt` in the store, and returns `saved`. A Lua block can also call a tool with `tools.call` and end with no `return`, and the section then continues with the prose that follows.

The prologue and the epilog run as two separate Lua blocks, each with its own `lua_chunk_started` and `lua_chunk_succeeded` events, and a `models.infer` call in the epilog reports its model round between the epilog's own pair of events.

### How errors name each block

Error messages name each Lua block with a location label that says where the block sits:

| Block | Location label |
| --- | --- |
| A section's first block, when it is a Lua block | ``section `{name}` prologue`` |
| A section's last block, when it is a Lua block and the section also has prose | ``section `{name}` epilog`` |
| Any other Lua block in a section | ``section `{name}` lua {ordinal}`` |
| Any of those three section blocks, when another section heading has the same section name | ``section `{name}` (line {heading_abs_line}) prologue``, ``section `{name}` (line {heading_abs_line}) epilog``, or ``section `{name}` (line {heading_abs_line}) lua {ordinal}`` |
| Any block in the H1 body | ``H1 `{title}` lua`` |

`{name}` is the section name, the heading text without the `#` marks, `{heading_abs_line}` is the file line of the section's own heading (see [Sections that share a section name](05-lua-environment.md#sections-that-share-a-section-name)), and `{title}` is the H1 title. `{ordinal}` is the block's 1-based place among the section's Lua blocks, counting the prologue as 1, so the third Lua block of `lib` is ``section `lib` lua 3``, as in the library error in [Where a library error points](#where-a-library-error-points). In a section of just two back-to-back fences, the second fence is labeled `epilog`, because the empty prose between them counts as prose. A malformed epilog makes the prompt fail to parse with parse error kind `Lua` and a message that names the section and its epilog, as in ``section `Transform` epilog`` in [When Lua blocks compile and run](#when-lua-blocks-compile-and-run).

## Pending prose and the `prose` global

You can read the prose above a Lua block, know what it holds and when it renders, and handle the errors that reading or assigning `prose` raises.

````markdown
---
name: reader
description: Return the pending prose as the run result
runvil: 0
---

# Reader

## Main

Say hello to the reader.

```lua
return prose
```
````

````text
Say hello to the reader.
````

Prose written since the heading or the previous `lua` fence is pending prose, and it builds up for the next Lua block. A Lua block reads the pending prose written just above its fence in the same section through the read-only `prose` global. Inside the block, `prose` holds that pending prose as a plain Lua string with every `{{ }}` placeholder already filled in. It needs no assignment and is ready to pass to a model or to return.

A block that returns `prose` makes its rendered pending prose, the substituted text a model would get, the result, so `return prose` from a section makes the rendered prose the run's result text, as here.

### What a block's `prose` holds

Markdown written after a section heading accumulates as pending prose that the next ordinary `lua` fence consumes, so `Gather the facts.` followed by a fence holding `return 1` gives that fence the prose `Gather the facts.`. Prose written before a section's first fence is part of that fence's `prose`, as in the example above. Each fence consumes the pending prose, so every Lua block reads exactly the Markdown written since the previous fence in that section:

````markdown
---
name: steps
description: Each fence reads the prose written since the previous fence
runvil: 0
---

# Steps

## Main

First.

```lua
var.first = prose
```

Second.

```lua
return var.first .. ' ' .. prose
```
````

````text
First. Second.
````

The first block gets `First.` and the second gets `Second.`. Markdown written after an ordinary `lua` fence starts a fresh pending prose buffer, so a fence splits prose, and the text before and after it are separate prose blocks.

The string a block reads follows these rules:

- Pending prose is the Markdown body text only, trimmed of the blank lines around it and never including the heading, so `## Run`, a blank line, and `Done.` give the prose `Done.`.
- `prose` holds the prose just above the fence with its placeholders already substituted, without the surrounding blank lines or a trailing newline.
- Reading `prose` returns the section's prose text with its placeholders filled and without the code of its `lua` fences.
- Reading `prose` always gives a Lua string, and a Lua block with no pending Markdown before it reads `prose` as the empty string `''`.

Prose can hold any Markdown you mean as instruction text for the model, such as numbered lists and inline code spans: for example a `Do this:` line followed by items `1.` through `4.` that name `web_search` and `web_fetch`. Prose becomes useful when a `lua` fence below it in the same section reads it.

### Prose never calls a model on its own

You send prose to a model by passing `prose` to `models.infer` in a Lua block:

````markdown
---
name: hello
description: Ask the model to say hello
runvil: 0
---

# Hello

## Greet

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

`models.infer(prose)` sends the pending prose above the current Lua block to the model, so this epilog sends `Say "Hello, world!"` as the message and returns the reply (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)). The epilog runs after the section's prose and can call `models.infer(prose)` to send that prose to the model, and any other Lua block can do the same, as in `var.first = models.infer(prose)` in a middle fence.

Prose never calls a model on its own and only feeds the next Lua block. Nothing reaches a model until Lua passes the prose to an explicit `models.infer`, and only that call needs a configured model. Prose written above a fence waits in the pending buffer, so you use it only by passing `prose` to an explicit infer such as `var.draft = models.infer(prose)`, and the prose itself adds no model request beyond that call. A section with two prose paragraphs and two `models.infer(prose)` calls makes exactly two model rounds, and the prose adds none.

The H1 body and each section collect their own pending prose, so each part's prose reaches the model only through its own `models.infer(prose)` call, and those calls run in source order:

````markdown
---
name: twoquestions
description: H1 prose and section prose each feed their own model round
runvil: 0
---

# Two questions

Name one planet.

```lua
var.planet = models.infer(prose)
```

## Answer

Name one ocean.

```lua
return models.infer(prose)
```
````

The prompt asks `Name one planet.` first, during the H1 pass, and then `Name one ocean.` from `Answer`, and the result is the ocean reply. The H1 pass and each section keep separate pending prose buffers, so each reads only its own prose through `prose`, and prose written between `lua` fences in the H1 body collects into a buffer that `prose` reads the same way. H1 prose never drives inference on its own: on a Host that supplies no model, a prompt whose H1 prose stays unread still runs, while an H1 block running `return models.infer(prose)` fails there because the chain has no model (see [When a section has no model](10-models.md#when-a-section-has-no-model)).

### What resets pending prose

Pending prose resets at three points: the nearest preceding heading, `lua` fence, or thematic break (`---`). Text above a `---`, such as `Draft notes.`, never reaches the next fence's `prose`, and the `---` itself is not part of it either. [Thematic breaks and setext headings](#thematic-breaks-and-setext-headings) covers breaks in full.

A heading ends the pending prose, and each section entry starts with empty pending prose, so prose one section leaves unread never reaches the next section's first fence:

````markdown
---
name: boundary
description: Pending prose stops at a section boundary
runvil: 0
---

# Boundary

## A

Prose for A.

## B

```lua
return '[' .. prose .. ']'
```
````

````text
[]
````

Pending prose resets at each section entry, so a section's unconsumed prose never reaches the next section. Because pending prose never moves data from one section to the next, data a later section needs travels through an explicit channel such as the store (see [Writing and reading files](09-the-store.md#writing-and-reading-files)).

### Commentary

Prose that no Lua block reads is commentary. It is dropped unread at the end of its section, without being rendered or sent anywhere, so you can keep notes for human readers in it:

````markdown
---
name: notes
description: Commentary that is never rendered
runvil: 0
---

# Notes

Notes for human readers can mention {{ var.omit }} safely.

## First

{{ var.missing }} is never read here.

## Second

An unclosed {{ placeholder and a {{ var.missing }} key.

```lua
return 'ok'
```

Trailing {{ var.missing }} commentary.
````

````text
ok
````

Every piece of prose in this prompt is commentary:

- Pending prose in the H1 body that no Lua block reads is discarded when the H1 pass ends, without error, even when it holds a placeholder like `{{ var.omit }}`. Unread H1 prose sends nothing to a model, so it needs no model, and H1 prose meant only for human readers can stay unread.
- Prose at the end of a section with no Lua fence after it is discarded when the section ends, without being rendered, so a placeholder naming a missing value, such as `{{ var.missing }}`, cannot fail the run.
- A block that never reads `prose` is untouched even by an unclosed `{{` in the prose above it.
- Markdown left after the final `lua` fence of a body is inert trailing commentary: no fence reads it, its placeholders are never filled, and it never raises an error, so `Trailing {{ var.missing }} commentary.` leaves the result `ok`. Prose left pending after a section's last fence is not a parse error either.

### When `prose` renders

`prose` renders lazily: its `{{ }}` placeholders render at the block's first read of `prose`, not when the block starts, so `var` and global changes made earlier in the same block show up in the text:

````markdown
---
name: word
description: prose renders at its first read
runvil: 0
---

# Word

## Main

The word is {{ var.word }}.

```lua
var.word = 'mutated'
return prose
```
````

````text
The word is mutated.
````

Reading the `prose` global returns the current block's prose text, rendered when the first read happens. A `prose` the block has not read yet renders with the `var` values current at the moment of that read, not the values the block started with or held at any earlier point. With `Second {{ var.word }}.` above a fence whose block starts with `var.word` set to `'changed'`, sets it to `'two'` while a library loads through `require` (see [The caller's `prose` after a `require`](#the-callers-prose-after-a-require)), and sets it to `'three'` just before the read, `prose` gives `Second three.`. A closing Lua block that returns `prose` returns the pending prose with its placeholders already substituted in this way.

`prose` renders at most once per block. After the first successful read, every later read in the same block returns the same string, even after `var` or a global changes:

````markdown
---
name: once
description: prose renders at most once per block
runvil: 0
---

# Once

## Main

The word is {{ var.word }}.

```lua
var.word = 'one'
local first = prose
var.word = 'two'
assert(prose == first)
return prose
```
````

````text
The word is one.
````

Each `lua` fence gets a fresh `prose` before its Lua block starts, replacing the previous block's. The new `prose` holds only the Markdown written since the previous fence, heading, or thematic break, and it renders against the state at its own first read. A string saved from an earlier block's `prose` in a local or a `var` key keeps its earlier rendering after a later block gets and renders its own `prose`:

````markdown
---
name: pairs
description: Each fence gets a fresh prose
runvil: 0
---

# Pairs

## Main

First: {{ var.word }}.

```lua
var.word = 'one'
var.first = prose
```

Second: {{ var.word }}.

```lua
var.word = 'two'
return var.first .. ' ' .. prose
```
````

````text
First: one. Second: two.
````

### When reading `prose` fails

A placeholder that fails to render makes the read of `prose` raise the substitution error at that read (see [Substitution error kinds](07-substitution.md#substitution-error-kinds)), and `pcall` catches it (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). A failed render is not remembered, so reading `prose` again after you fix the cause, for example by setting the missing `var` key, renders again:

````markdown
---
name: retry
description: Catch a failed render and read prose again
runvil: 0
---

# Retry

## Main

Missing: {{ var.missing }}.

```lua
local ok = pcall(function() return prose end)
if not ok then
  var.missing = 'found'
end
return prose
```
````

````text
Missing: found.
````

Left uncaught, as in a bare `return prose` over `Missing: {{ var.missing }}.`, the substitution error fails the run with an error containing `missing {{ var.missing }}`, and the run's message ends with ` (at {file}:{line})`, the line of the placeholder.

Prose cannot refer to itself. A `{{ prose }}` placeholder inside prose makes the read of `prose` fail with an error naming `prose`, and that failure is catchable with `pcall` too.

### Keeping and reshaping the text

`prose` is read-only, so to keep or reshape the rendered text you copy it into a local, a `var` key, or a section global, as in `local before = prose` or `var.first = prose`, and work on the copy. Assigning to `prose` always raises a Lua runtime error, catchable with `pcall`:

````markdown
---
name: keep
description: Assigning to prose is refused
runvil: 0
---

# Keep

## Main

Keep this line.

```lua
local ok = pcall(function() prose = 'overwritten' end)
if ok then
  return 'changed'
end
return 'still: ' .. prose
```
````

````text
still: Keep this line.
````

The refused assignment raises an error with this text:

````text
prose is read-only: assign to `var` or a section global instead
````

Reading `prose` gives the block's rendered prose, and every write is refused, whatever the value, including `prose = nil`. The refusal applies at any time: before or after the first read, during the H1 pass, and in a library too. During the H1 pass, `argv` is writable (see [Reading parsed input from `argv`](06-arguments.md#reading-parsed-input-from-argv)), but `prose` stays read-only with the same refusal message. The refusal is a separate runtime error that has no substitution kind and no `[{Kind} at byte {offset}]` suffix. Caught with `pcall`, as in `keep`, its message holds the text above after `keep.md:14: `, the line of the assignment, and before the default text for a Lua failure (see [Where a caught error's message points](16-limits-and-errors.md#where-a-caught-errors-message-points)).

`prose` is a reserved global in every section and in the H1 pass, and the Lua environment has no `rawget`, `rawset`, `rawequal`, `debug`, `load`, or other raw or code-loading route around it, so every read and write of `prose` gets the read-only, lazy behavior. A metatable you set on `_G`, including one with its own `__index` or `__newindex`, never sees or changes how `prose` reads or refuses assignment, because `prose` is served and refused before any metatable of yours runs. With a metatable on `_G`, including one a library sets, `prose` stays read-only and still renders at its first read, and the next fence still gets its fresh `prose` even when the metatable has a `__newindex` write hook.

`prose` reads `nil` in code that runs before any block has started, such as a [Pack prelude](12-tools.md#pack-preludes) while it installs.

## Thematic breaks and setext headings

You can drop working notes from pending prose with a thematic break, and write breaks that never make a line of prose into a heading.

````markdown
---
name: instructions
description: A thematic break drops the notes above it
runvil: 0
---

# Instructions

## Main

Working notes the model should never see.

---

Write the actual instructions here.

```lua
return prose
```
````

````text
Write the actual instructions here.
````

A thematic break is a line such as `---` on its own, with a blank line before it. It resets pending prose: the Markdown above it is dropped, only the Markdown below the last break before the next `lua` fence is captured as `prose`, and the break line itself is never part of `prose`. Headings, `lua` fences, and thematic breaks are all prose boundaries, and the text after a break starts a new prose block.

Breaks follow CommonMark, so any genuine CommonMark thematic break line works as the reset, such as `---`, `***`, or `___` on a line of its own after a blank line.

### Breaks between fences

Between two fences, a break drops the notes about the previous step that you write above it:

````markdown
---
name: stepnotes
description: A break between two fences
runvil: 0
---

# Step notes

## Main

```lua
var.a = 1
```

Notes on step one.

---

Pending for step two.

```lua
return prose
```
````

````text
Pending for step two.
````

A section can contain any number of thematic breaks, and only the Markdown below the last break stays pending. A break only affects the prose segment it sits in, so a break before one `lua` fence does not trim the prose after that fence. Each prose block is the pending Markdown for the fence that follows it and keeps only the text below the last break in its segment, so commentary written above a `---` is dropped from the block. The reset also applies to the trailing prose after the last `lua` fence and to a section with no `lua` fence at all. A `lua` fence that sits directly below a break, with no Markdown between them, receives empty pending prose.

### A break never ends a section

Resetting pending prose is all a break does. It has no control-flow meaning and never ends a section's or the H1 body's content, so `lua` fences, prose, and headings below it parse and run as usual:

````markdown
---
name: belowbreak
description: A section that opens with a break
runvil: 0
---

# Below

## Main

---

```lua
var.x = 1
```

Below the break.

```lua
return prose
```
````

````text
Below the break.
````

A `lua` fence below a break is still part of its section's content and stays live, because a break resets prose capture only and makes nothing reader-only, and prose below a break is ordinary pending Markdown. You can put a `---` break as a section's first content, as here, and the section runs like any other. A section body that holds a break followed by a `lua` fence still runs that fence when the section is called with `call` (see [Calling a section](08-jump-and-call.md#calling-a-section)), and a break inside a fanout worker section (see [Choosing the worker section](14-fanout.md#choosing-the-worker-section)) only resets its pending prose, as in any section. A heading below a break still starts a new section.

A break stays in its section's raw Markdown and never ends the section, so [`prompt.section`](#reading-sections-with-promptsection-and-promptsection_or) returns the `---` line and the text on both sides of it verbatim, even though prose capture drops what sits above it. A [selector read](#reading-one-block-with-a-selector) sees the break as a block of its own, of kind `rule`.

### A break below the H1 title

You can set the prose under the H1 title off from the title with a `---` break on its own line, with a blank line on each side:

````markdown
---
name: summary
description: A break between the H1 title and its description
runvil: 0
---

# Summary

---

Returns a fixed greeting, entirely in Lua, with no model call.

## Main

```lua
return 'hello'
```
````

````text
hello
````

Plain prose under the H1 title describes what the prompt does, and because nothing sits above the break, the break drops nothing. A `---` line in the body with blank lines around it is a thematic break rather than a frontmatter delimiter or a setext heading underline. The H1 body splits into prose and `lua` blocks the same way a section does, so when Markdown does sit above the last break in the H1 body, it is left out of the H1 body's prose, and a following H1 `lua` block reads only the text below the break as `prose`.

### Breaks inside fenced code blocks

A `---` line inside a fenced code block is code, not a thematic break, and resets nothing. The same holds inside a `lua` fence, where a `---` line is a Lua comment, and it still holds below a leading break. A `text` fence stays whole inside the prose even when it holds a `---` line:

````markdown
---
name: livetext
description: A dash line inside a text fence is code
runvil: 0
---

# Live text

## Main

Live prose.

```text
---
```

Also live.

```lua
return prose
```
````

````text
Live prose.

```text
---
```

Also live.
````

### The setext heading trap

Headings follow CommonMark, so setext headings count too. A setext heading is a line of text underlined with `---` or `===`. A prose line directly followed by `---`, with no blank line between them, is a setext H2 underline: it makes the line a new section heading, and the Markdown after it belongs to the new section. This fragment is part of a prompt:

````markdown
## S

Some prose
---

More prose
````

The fragment holds two sections: `S`, and an H2 named `Some prose` whose prose is `More prose`. A `---` line directly under a text line is a setext heading underline rather than a thematic break, so it is never a reset point, and the text line above it becomes a heading instead.

A break needs a blank line before it. With a blank line between `Some prose` and the `---`, the `---` is an ordinary break:

````markdown
## S

Some prose

---

More prose
````

Now `S` is one section, and its pending prose is `More prose`.

A line of text underlined with `===` is a second H1, and a prompt must contain exactly one H1 title (see [The H1 title and H1 body](02-file-structure.md#the-h1-title-and-h1-body)), so it fails the parse with parse error kind `Structure`:

````markdown
---
name: twotitles
description: A setext underline that makes a second H1
runvil: 0
---

# Title

## S

Some prose
===
````

````text
prompt must contain exactly one H1 title, found 2 at lines 7, 11
````

The `{}` slots hold how many H1 titles the parser found and the lines they are on.

## Exact fence rules and fence errors

You can write `lua` fences the parser always accepts, and read the `Fence` errors it raises when a line comes close to a `lua` fence but misses.

````markdown
---
name: exact
description: An exact lua fence
runvil: 0
---

# Exact

## Main

```lua
return 'exact'
```
````

````text
exact
````

A `lua` block opens with a line of exactly three backticks followed by lowercase `lua`, starting at the first column, with no indent, no extra backticks, no space or tab before `lua`, and nothing after the label. It closes at the first later line that is exactly three backticks at column 1, unindented and with nothing else on the line. Opener and closer lines can end in CRLF as well as LF, so a prompt file with Windows line endings parses the same way.

A Lua block's source is every line between its opener and the first line that is exactly three backticks, so Lua source, including a multi-line string, cannot contain that line.

### Fences with other labels

Only a fence labeled exactly `lua` becomes an executable Lua block, and `lua` is the only Lua fence label: everything else in the H1 body and section content is prose. A fenced code block with no label, or with a label that does not start with `lua` in any letter case, such as `text`, `json`, `text lua`, or `moonlua`, is ordinary Markdown. It stays in the section's prose verbatim, fence lines and contents included, compiles no Lua, and creates no prologue or epilog, and the next Lua block reads it through `prose`:

````markdown
---
name: explain
description: A python fence stays in the prose
runvil: 0
---

# Explain

## Main

Explain what this code prints:

```python
print(1)
```

```lua
return prose
```
````

````text
Explain what this code prints:

```python
print(1)
```
````

### Showing a `lua` fence as an example

Lines inside the body of any other fenced code block are never checked against the Lua fence rules. To show a `lua` fence to a reader or a model as an example, even a near miss such as `` ```lua extra ``, nest it inside a longer fence, such as a four-backtick `markdown` fence or a tilde fence. It stays prose, with no Lua compiled and no `Fence` error, and near-miss lines inside the outer fence stay prose without errors too. A section that opens with such an outer fence has no Lua prologue, and the whole fence stays in the section's prose:

````markdown
---
name: teach
description: Show a lua fence to the model as an example
runvil: 0
---

# Teach

## Main

~~~markdown
```lua
return 1
```
~~~

```lua
return prose
```
````

````text
~~~markdown
```lua
return 1
```
~~~
````

### The four fence rules

The parser checks every fence against four rules, and breaking any of them fails the parse with parse error kind `Fence`:

| Rule | Broken by | Reported at |
| --- | --- | --- |
| Label | a fence label that starts with `lua` in any letter case but is not exactly `lua` | the opening line |
| Opener | a fence labeled exactly `lua` whose opening line is not exactly three backticks then `lua` | the opening line |
| Closer | a fence with an exact opening line whose closing line is not exactly three backticks | the closing line |
| Lookalike | a line outside every fence that looks like a `lua` opening line | that line |

Each rule below comes with one failing example.

### The Label rule

A fence whose label starts with `lua` in any letter case without being exactly `lua`, such as `Lua`, `LUA`, `luajit`, `luau`, or `lua extra`, breaks the Label rule:

````markdown
---
name: label
description: A fence label that is not exactly lua
runvil: 0
---

# Label

## S

Prose.

```lua extra
local a = 1
```
````

````text
unrecognized fence label `lua extra`; the only Lua fence label is `lua`
````

The parse fails with parse error kind `Fence`, and the message `` unrecognized fence label `{label}`; the only Lua fence label is `lua` `` names the offending label as written. The label is the whole info string the Markdown parser reports after the backticks, so a label that starts with `lua` but adds more, such as `luau`, `lua extra`, or an attribute after `lua`, breaks the Label rule rather than being read as plain code. The `lua` label is case-sensitive, so a fence labeled `Lua` or `LUA`, in the H1 body or a section, fails the same way.

Fence detection follows CommonMark as the Markdown parser reads it, so any line the parser treats as a fence opener is checked against the Label rule. When the parser reads a line as a fence opener, a near-miss label gets the label error whatever the opener's shape, rather than the opener error: a four-backtick fence with a near-miss label, closed by four backticks, fails with the unrecognized-label error, and so does a fence indented by one space. A near-miss label is reported even when the fence's closing line does not match its opener. The same line indented four spaces is no fence at all, and it gets the opener message as a lookalike, as [The Lookalike rule](#the-lookalike-rule) describes.

The label check covers every part of the body: the preface, which is the prose between the frontmatter and the H1 title; the H1 region between the `# Title` line and the first section, including a fence that follows a whitespace-only line; and fences inside a section body after the section's prose. A near-miss label fails the same way whether its fence opens the H1 body or follows an intro paragraph.

### The Opener rule

A fence labeled exactly `lua` whose opening line is not exactly `` ```lua `` breaks the Opener rule:

````markdown
---
name: tilde
description: A tilde fence around Lua
runvil: 0
---

# Tilde

## Main

~~~lua
local greeting = "hello"
~~~
````

````text
`lua` fence opening line must be exactly ```lua
````

The parse fails with parse error kind `Fence` at the opening line, so the prompt is rejected as invalid, and `lua` code needs backtick fences. Every opener failure has this one fixed message, which contains `fence opening line must be exactly`. Each of these opening lines breaks the rule:

- a trailing space or a trailing tab after `lua`
- a space between the backticks and `lua`
- an indent of one to three spaces, even three, which CommonMark accepts
- a tilde fence, `~~~lua`
- four or more backticks, so a `lua` fence uses exactly three
- a fence inside a block quote or a list item, because its opening line begins with the container marker, such as `>` or `-`

A fence inside a block quote or list item with a `lua`-like label, rather than exactly `lua`, breaks the Label rule instead. The Opener rule applies in the preface before the H1 title, in the H1 region, whether the fence opens it or follows an intro paragraph, and inside a section body, and a breaking fence fails whether it starts the section or follows intro prose. Every non-exact opener is rejected even when its fence closes with a line that matches its own opener, such as `~~~` after `~~~lua` or four backticks after a four-backtick opener.

### The Closer rule

A fence with an exact `` ```lua `` opener whose closing line is not exactly `` ``` `` breaks the Closer rule:

````markdown
---
name: closer
description: An indented closing line
runvil: 0
---

# Closer

## Main

```lua
return 'closed'
  ```
````

````text
`lua` fence closing line must be exactly ```
````

The parse fails with parse error kind `Fence` at that closing line, and the prompt is rejected as invalid. The message contains `fence closing line must be exactly`. A closing line breaks the rule when it has trailing spaces or tabs after the three backticks, an indent of one to three spaces, or four backticks. A bad closing line fails the parse no matter what follows it, whether another `lua` fence or prose with a `python` block.

### The Lookalike rule

Any line outside every fence that, after leading spaces or tabs, has three or more backticks or tildes, optional spaces or tabs, and then `lua` in any letter case looks like a `lua` opening line and breaks the Lookalike rule:

````markdown
---
name: lookalike
description: An indented line that looks like a lua opener
runvil: 0
---

# Lookalike

## Main

Intro.

    ```lua
    return 1
    ```
````

````text
`lua` fence opening line must be exactly ```lua
````

The parse fails with parse error kind `Fence`, and you see the lookalike reported with the Opener message. Instead of becoming an indented code block in the prose, the `` ```lua `` line indented by four spaces fails. The rule catches such lines inside indented code blocks and raw HTML blocks, so these lines fail the same way:

- an indented `~~~lua` tilde line
- indented fence lines with a space or a tab before `lua`, or with uppercase `LUA`
- a `` ```lua `` fence inside an HTML block such as `<div>` and `</div>`, which fails rather than passing through as HTML

Lines inside a `lua` fence's own body are never checked against the Lookalike rule, so Lua source can contain a line such as `~~~lua` or `` ```lua `` inside a multi-line string.

### Which error you see

For each fence, the Label, Opener, and Closer rules are checked in that order, and the parse reports only the first violation in source order. It points at the opening line for a Label or Opener break, at the closing line for a Closer break, and at the line itself for a Lookalike line. When several fences have bad labels, the parse reports only the first one in document order, and when a section has both a lookalike line and a bad closing line, it reports whichever comes first in the file.

A fence rule error points at the offending line: its line number is that line's 1-based number in the whole file, counting the frontmatter lines, and its column is 1. In the Label rule example, the error points at line 13, column 1, the `` ```lua extra `` line. Fence error messages name the label or the rule but never include the code inside the fence.

The Lua fence rules are checked across the whole body after the frontmatter, including the preface before the H1, before any content is split into blocks. A fence problem anywhere in the file fails the parse before the run starts and before any Lua compiles, so a bad label or a non-exact opener in a later section stops the parse even when the H1 body's Lua is valid. A fence error also wins over a Lua syntax error inside the same fence, so you see parse error kind `Fence` rather than a Lua compile error.

### Unclosed fences

A `lua` fence left open with no closing line fails the parse with parse error kind `Fence`:

````markdown
---
name: unclosed
description: A lua fence with no closing line
runvil: 0
---

# Unclosed

## S

```lua
return 1
````

````text
section `S` prologue `lua` fence is not closed
````

An unclosed `lua` fence that is the only fence and sits at the top of its content, left open to the end of the section, gets this prologue message. In the not-closed messages, the section name is the section's heading text without the `#` marks, such as `S` for `## S`. An unclosed fence that follows prose or another fence, left open to the end of the section, gets the epilog wording instead:

````text
section `{section}` epilog `lua` fence is not closed
````

An unclosed `lua` fence under the H1 uses the same section wording with the H1 title as `{section}`, so under the H1 `# Open` it reads ``section `Open` prologue `lua` fence is not closed``. A backtick line indented by four spaces, or three backticks followed by other text, cannot close a `lua` fence, so the fence stays open and the parse fails with the not-closed message. An unclosed fence error names its section and phase in the message, and like a rule error it points at a line, at column 1: the line where the fence opens, so `unclosed` reports line 11.

## Reading sections with `prompt.section` and `prompt.section_or`

You can read another section's raw Markdown, or one block of it such as a list or a fenced template, and know what each read does when the section or the block isn't there.

````markdown
---
name: prompt_reads
description: Reads a child's text, a fenced template, and a list.
runvil: 0
---

# Prompt reads

## Write

```lua
local style = prompt.section('house_style')
local template = prompt.section('Template', 'fence markdown').text
local topics = prompt.section('Topics', 'list').items
return style .. '|' .. template .. '|' .. table.concat(topics, ',')
```

### house_style

Short sentences. No hype.

### Template

Fill in this outline.

```markdown
# Title
```

### Topics

- allocators
- coroutines
````

````text
Short sentences. No hype.|# Title|allocators,coroutines
````

You read other sections through the `prompt` global, a table that holds two functions, `prompt.section` and `prompt.section_or`. The Engine installs `prompt` in every section's Lua and in the H1's Lua. Each read names its target with a [section path](02-file-structure.md#section-paths): a relative path for a neighbor, such as `'Topics'` for a child or `'../Review'` for a sibling, or a path from the top for any section, such as `'/Write/Template'`.

Each function has two forms:

| Call | Returns |
|---|---|
| `prompt.section(path)` | the section's own raw Markdown, as a string |
| `prompt.section(path, selector, n?)` | one block of the section, as a table |
| `prompt.section_or(path)` | the same string, or `nil` when no section has the path |
| `prompt.section_or(path, selector, n?)` | the same block, or `nil` when the section or the block is not there |

The two functions take the same arguments and find the same thing. They differ only when it is not there: `prompt.section` raises, and `prompt.section_or` returns `nil`.

### Reading raw Markdown with `prompt.section`

`prompt.section(path)` returns a section's own raw Markdown body as a string. Replace the block of `Write` in the previous prompt with this one:

````markdown
## Write

```lua
return prompt.section('Template')
```
````

````text
Fill in this outline.

```markdown
# Title
```
````

A section's own raw Markdown runs from just after its heading line to the next heading of any level. It leaves out the section's own heading line and its child sections, headings and content alike, and it trims leading blank lines and trailing whitespace. A child section is a section of its own, read by its own path, so `prompt.section('/Write')` returns only the `lua` fence of `Write`, with nothing from `house_style`, `Template`, or `Topics`. A `#` line inside a fence is not a heading, so the `# Title` line stays part of `Template`.

Apart from that trimming, the body comes back byte for byte, and nothing in it runs or renders:

````markdown
---
name: raw_rules
description: Returns a section's raw Markdown unchanged.
runvil: 0
---

# Raw rules

## Main

```lua
return prompt.section('../Rules')
```

## Rules

  Indented first line.

- **Bold** and `code` stay raw.

```lua
var.never = 'run'
```

---

Closing line with {{ args }}.
````

````text
  Indented first line.

- **Bold** and `code` stay raw.

```lua
var.never = 'run'
```

---

Closing line with {{ args }}.
````

The result keeps the first line's leading indentation and the blank lines inside the body, and drops only the blank lines that separate the body from its heading and from the next heading. Inline Markdown such as `**bold**` and backtick code comes back raw, not rendered. The `lua` fence comes back as text, and the read does not run it. The `{{ args }}` [placeholder](07-substitution.md#placeholders-in-prose) comes back literally, without substitution. The thematic break stays in the Markdown too, even though the break resets the section's prose, as [A break never ends a section](#a-break-never-ends-a-section) explains.

The result is a Lua string that string methods work on, such as `text:find('Mine text.', 1, true)`. A section with nothing under its heading reads as the empty string `''`.

A read performs no effect and moves nothing. It does not consume the pending prose of the section it reads, and the walk continues where it was.

### Reading one block with a selector

`prompt.section(path, selector)` reads one block of a section instead of its whole text. A block is one top-level piece of the section's own Markdown, and a selector is a string that names a kind of block:

| Selector | Picks | Fields of the block |
|---|---|---|
| `'prose'` | a run of ordinary Markdown: paragraphs, block quotes, and anything that is no other kind | `text` |
| `'fence'` | a fenced code block | `lang`, `info`, `text` |
| `'fence markdown'` | a fence whose language is `markdown`, and likewise for any other language | `lang`, `info`, `text` |
| `'fence ""'` | a fence with no language | `lang`, `info`, `text` |
| `'list'` | a bulleted or numbered list | `ordered`, `items` |
| `'rule'` | a thematic break | none |
| `'*'` | a block of any kind | the fields of its kind |

The block comes back as a plain Lua table built for the call, and you read its fields by name, as `prompt.section('Template', 'fence markdown').text` does in the first prompt. Every block also has `kind`, the kind's name as a string, `line`, the line of the prompt file the block starts on, and `source`, the block's lines exactly as the section holds them, with the blank lines after them. These are the blocks [`markdown.parse`](05-lua-environment.md#parsing-markdown-with-markdown) produces, and that section gives the full rules for each kind and for [`source`](05-lua-environment.md#the-source-of-a-block).

A fence's `text` is the lines between its two fence lines, without the final line ending. Its `info` is the text after the opening fence, and `lang` is the first word of `info`, or `''` when the fence has none. The language is compared exactly and case-sensitively. A `lua` fence is a fence like any other: its `lang` is `lua` and its `text` is the source, which the read does not run.

With no index, the selector must match exactly one of the section's own blocks. When it matches several, pass an index as the third argument, where `1` is the first match and `-1` is the last:

````markdown
---
name: selectors
description: Picks blocks of a section by kind, language, and index.
runvil: 0
---

# Selectors

## Main

```lua
local first = prompt.section('Samples', 'fence', 1)
local last = prompt.section('Samples', 'fence', -1)
local data = prompt.section('Samples', 'fence json')
local plain = prompt.section('Samples', 'fence ""')
return first.lang .. '|' .. last.lang .. '|' .. data.text .. '|' .. plain.text .. '|' .. data.line
```

### Samples

Three samples.

```json
{ "ok": true }
```

```
plain text
```

```yaml
ok: true
```
````

````text
json|yaml|{ "ok": true }|plain text|23
````

`Samples` holds one prose block and three fences. `'fence json'` and `'fence ""'` each match one fence, so they need no index, and `data.line` is 23, the line of the prompt file where the `json` fence opens. `'fence'` alone matches all three, so it needs the index. Without one, the read raises and says how many blocks matched and where:

````text
prompt.section: selector "fence" matches 3 blocks, at lines 23, 27, and 31; add an index, such as 1 for the first
````

`'*'` matches a block of any kind, so with an index it picks a block by its position: `prompt.section('Samples', '*', 1)` is the prose block, and `prompt.section('Samples', '*', 2).lang` is `'json'`. The sources of a section's blocks, joined in order, are the text `prompt.section('Samples')` returns.

A selector looks only at the section's own content. A block inside a child section belongs to the child, and you read it by the child's path. The kinds `section` and `frontmatter` exist for `markdown.parse` and are refused here, because a section's own content never holds either.

### Reading a list

A list is a `list` block, wherever it sits in a section, and the language has no other notion of a list. `prompt.section(path, 'list').items` gives the items as a Lua sequence of strings in document order, as `prompt.section('Topics', 'list').items` gives `{'allocators', 'coroutines'}` in the first prompt.

````markdown
---
name: list_items
description: Reads the items of two lists in one section.
runvil: 0
---

# List items

## Main

```lua
local bullets = prompt.section('Notes', 'list', 1)
local steps = prompt.section('Notes', 'list', 2)
return json.encode(bullets.items) .. '|' .. tostring(bullets.ordered) .. '|'
  .. json.encode(steps.items) .. '|' .. tostring(steps.ordered)
```

### Notes

Keep these in mind.

- first
  - nested
-
- last

1. draft
2. review
````

````text
["first\n- nested","","last"]|false|["draft","review"]|true
````

Each part of that result is one rule:

- The marker is removed from each item. A bulleted list and a numbered list read the same way, and the block's `ordered` field is `true` for a numbered one.
- A nested bullet stays in its parent's item. An item that spans lines or holds nested content keeps it as raw Markdown in one string, so `- first` with `- nested` indented under it is the single item `'first\n- nested'`.
- A bullet with no content is the item `''`.
- A change of marker starts a new list, so the bullets and the numbered steps are two lists, and a section with two lists needs an index.
- The list sits beside the prose line `Keep these in mind.` and is still found, because the selector picks the list and not the section.

The `items` table is a plain Lua table, so `#`, indexing, and `table.concat` work on it. It is an ordinary sequence of strings, so `tasks.fanout` accepts it as its [collection](14-fanout.md#collections-and-member-order), as in `tasks.fanout('Worker', prompt.section('Topics', 'list').items)`.

A section that holds only a list is an ordinary section. The [section walk](04-how-a-prompt-runs.md#the-section-walk) reaches it like any other section, and with no Lua in it, nothing runs there. It never has to run for another section to read its list: in the first prompt, `Write` returns before the walk reaches any other section, yet it reads both topics.

### Which sections a read can name

A read can name any section in the prompt. A relative path names a neighbor of the section the read is written in, and a path from the top names any section, the same one from everywhere:

````markdown
---
name: read_paths
description: Shows which paths name a section from Main.
runvil: 0
---

# Read paths

## Main

```lua
local seen = {}
for _, path in ipairs({ 'Kid', '../Nums', 'Grand', '/Main/Kid/Grand', '/Other/Niece', '/Main', '../Main' }) do
  seen[#seen + 1] = path .. ': ' .. (prompt.section_or(path) and 'found' or 'nil')
end
return table.concat(seen, '\n')
```

### Kid

- kid-one
- kid-two

#### Grand

- grand-item

## Nums

1. one
2. two
3. three

## Other

Other prose.

### Niece

- niece-item
````

````text
Kid: found
../Nums: found
Grand: nil
/Main/Kid/Grand: found
/Other/Niece: found
/Main: found
../Main: nil
````

From `Main`:

- `Kid` is a direct child and `Nums` is a sibling, so each relative path finds its section.
- `Grand` is a grandchild. `Main` has no child of that name, so the relative path `'Grand'` names nothing, and the path from the top `'/Main/Kid/Grand'` reads it.
- `Niece` is a child of the sibling `Other`, and `'/Other/Niece'` reads it.
- `'/Main'` is the running section itself. A section can read its own text, and its ancestors', by a path from the top. `'../Main'` names nothing, because a section is not its own sibling.

A read is not held to the [visible set](08-jump-and-call.md#the-visible-set) that `call` and `jump` are, because a read runs nothing and leaves the walk where it is. A shared section therefore has one path at every depth: a worker nested anywhere reads a top-level `Templates` section as `prompt.section('/Templates', 'fence markdown')`. `prompt` has no read for the H1 body or the frontmatter.

### Reads from the H1 and from libraries

Both reads work in the H1 body too. There a relative path names a top-level section, since those are the H1's children, and a path from the top names any section:

````markdown
---
name: h1_reads
description: Reads sections from the H1 body.
runvil: 0
---

# H1 reads

```lua
var.nums = table.concat(prompt.section('Nums', 'list').items, ',')
var.main = tostring(prompt.section_or('Main') ~= nil)
var.kid = tostring(prompt.section_or('Kid'))
var.deep = prompt.section('/Main/Kid')
```

## Main

```lua
return var.nums .. '|' .. var.main .. '|' .. var.kid .. '|' .. var.deep
```

### Kid

Kid notes.

## Nums

1. one
2. two
3. three
````

````text
one,two,three|true|nil|Kid notes.
````

From the H1, `prompt.section('Nums', 'list')` reads the list in the top-level section `Nums`, and `prompt.section_or('Main')` finds a top-level section. `prompt.section_or('Kid')` returns `nil`, because `Kid` is not a child of the H1, and the path from the top `'/Main/Kid'` reads it. The H1 keeps what it found in [`var`](05-lua-environment.md#keeping-state-in-var), and `Main` reads it from there.

A `prompt.section` read in the H1 body doubles as a check that the prompt has the sections it needs, made before any section runs: an uncaught miss there fails the [H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate).

A read written in a [library](#sharing-code-with-require) section starts at the library, not at the section that is using it, so a library reads its own children and siblings by relative paths. [A library names its own sections](#a-library-names-its-own-sections) covers the rule.

### What a read does on a miss

When its path names no section, `prompt.section` raises the [not-found error](08-jump-and-call.md#the-not-found-error), the same error `call` raises for the same path. `prompt.section_or` returns `nil` instead:

````markdown
---
name: read_misses
description: Shows what each read does when no section has the path.
runvil: 0
---

# Read misses

## Main

```lua
local strict_ok = pcall(prompt.section, '../Missing')
local lenient_ok, value = pcall(prompt.section_or, '../Missing')
local call_ok = pcall(call, '../Missing')
local style = prompt.section_or('/house_style') or 'Plain sentences.'
return tostring(strict_ok) .. '|' .. tostring(lenient_ok) .. ':' .. tostring(value) .. '|' .. tostring(call_ok) .. '|' .. style
```
````

````text
false|true:nil|false|Plain sentences.
````

`prompt.section` and `prompt.section_or` are plain functions, so you can pass them to `pcall` with their arguments, as this block does. The strict read fails at the line of the read, with a message that says what is missing, here ``section `../Missing` not found: `/Main` has no sibling `Missing` ``. That is the text `call('../Missing')` raises from the same section. Reach for `prompt.section_or` when a section is optional, and give it a default with `or`, as the `style` line does.

The selector forms treat a missing block the same way:

| Outcome | `prompt.section` | `prompt.section_or` |
|---|---|---|
| The path names no section | raises | `nil` |
| The selector matches no block, or fewer blocks than the index asks for | raises | `nil` |
| The selector matches several blocks and no index is given | raises | raises |
| The path, selector, or index is malformed | raises | raises |

The first two rows are things that are absent. The last two are a choice that only an index can make and a mistake in the call, and both functions raise for them. So an optional block reads as `local fence = prompt.section_or('Extras', 'fence json')`, which is `nil` when `Extras` is missing and when it has no `json` fence. An index that runs past the last match is an absence too, so a loop can count upward with `prompt.section_or(path, 'fence', n)` until it returns `nil`.

A name built at run time can act as a key: `prompt.section_or('/Vendors/' .. vendor)` is `nil` for a vendor with no section. The name is a path step like any other, so a `/` or a `\` in it must be escaped, and an empty name makes the path malformed, which raises.

### Errors a read raises

An empty section is not a miss. `prompt.section` on it returns the empty string `''`, and a selector finds no block in it.

Each row below raises a Lua error that [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch. The last two are misses, which only `prompt.section` raises. Every other row raises from `prompt.section_or` too, with `prompt.section_or` in place of `prompt.section` at the start of the message:

| Call | Message |
|---|---|
| `prompt.section(42)` | `section target must be a string, got integer` |
| `prompt.section('Kid/Grand')` | `section path "Kid/Grand" has more than one name; a relative path is a child's name, or "../" and a sibling's name`, and the other messages in [Malformed section paths](02-file-structure.md#malformed-section-paths) |
| `prompt.section('Samples', 42)` | `prompt.section: selector must be a string, got integer` |
| `prompt.section('Samples', 'fences')` | `prompt.section: unknown kind "fences"; kinds are frontmatter, section, prose, fence, list, rule, and * selects every block` |
| `prompt.section('Samples', 'list lua')` | `prompt.section: only a fence selector takes a language, got "list lua"` |
| `prompt.section('Samples', 'fence a b')` | `prompt.section: a fence selector takes one language, got "fence a b"` |
| `prompt.section('Samples', 'section')` | `prompt.section: a section's own content has no subsections; name a subsection by its own path` |
| `prompt.section('Samples', 'frontmatter')` | `prompt.section: a section's own content has no frontmatter` |
| `prompt.section('Samples', 'fence')`, with three fences | `prompt.section: selector "fence" matches 3 blocks, at lines 23, 27, and 31; add an index, such as 1 for the first` |
| `prompt.section('Samples', 'fence', 0)` | `prompt.section: index must be a nonzero integer, got 0` |
| `prompt.section('Samples', nil, 1)` | `prompt.section: an index needs a selector before it` |
| `prompt.section('Samples', 'list')`, with no list | ``prompt.section: no block in section `Samples` matches selector "list"`` |
| `prompt.section('Samples', 'fence', 4)`, with three fences | ``prompt.section: selector "fence" matches 3 blocks in section `Samples`; there is no block 4`` |

The selector and the index are checked before the path is looked up, so a malformed one raises whether or not the section is there. A message names the section by the path you wrote, with its surrounding whitespace trimmed.

`prompt` holds these two functions and nothing else. The read returns text or one block and never runs anything: reading a `lua` fence gives its source as a string, and the [sandbox](05-lua-environment.md#the-sandbox) has no way to load code from a string. To run a section's Lua, use [`call`](08-jump-and-call.md#calling-a-section) or [`require`](#sharing-code-with-require).

## Sharing code with `require`

You can put Lua in one section and use it from any section that needs it.

````markdown
---
name: greet_library
description: Loads a library section and calls the function it defines.
runvil: 0
---

# Greet library

## Main

```lua
require '/helpers'
return greet('main')
```

## helpers

```lua
function greet(name)
  return 'hello ' .. name
end
```
````

````text
hello main
````

`helpers` is a library: an ordinary section that other sections pull in with `require`. Nothing in the file marks it as a library. `require '/helpers'` runs the named section's Lua in the current block, and the globals it defines, such as the function `greet`, are callable in the requiring section right after the call.

A typical library section holds only a `lua` fence of global function definitions and sits after the section that returns the run result, as `helpers` does here. A library section can sit later in the file than the H1 or the sections that require it. When the [section walk](04-how-a-prompt-runs.md#the-section-walk) does reach a library in document order, it runs it under the normal section rules, as [Libraries on the section walk](#libraries-on-the-section-walk) shows.

### Calling `require`

`require` names its library by a [section path](02-file-structure.md#section-paths) from the top of the prompt, as in `require '/helpers'` for a top-level section or `require '/Main/helpers'` for a section nested under `Main`. Lua's string-call form without parentheses works, and so do `require('/helpers')` and `pcall(require, '/a')`. You can call `require` from the H1 body's block and from any section's block.

Any section with a Lua block can serve as a library, whether it holds only Lua or prose followed by a Lua block. `require` loads every `lua` fence of the named section and runs them one after another in document order.

`require` takes the path of a section in the prompt. It loads sections only, never a file from disk or native code, and it reads its argument as a section path every time, so Lua's `os` and `package` tables stay `nil` however you call it.

When a library section returns no value, you call `require` as a bare statement purely for its effects. A `setup` library that only runs [`models.use('strong')`](10-models.md#selecting-a-role-with-modelsuse) works this way. The choice belongs to the chain, so the block that called `require` and every later section of that chain make their model rounds on the `strong` role until the next `models.use`.

### What `require` returns

A library shares code by returning a table of functions or values, as in `return M`. Every caller of `require` receives that table and can call its functions or index it directly:

````markdown
---
name: search_library
description: Loads a library that returns a table of functions.
runvil: 0
---

# Search library

## Main

```lua
local search = require '/search_tools'
return search.search_all(argv.prose)
```

## search_tools

```lua
local M = {}
function M.search_all(topic)
  return 'searched ' .. topic .. ' everywhere'
end
return M
```
````

Run it with the argument string `rust`, which arrives as [`argv.prose`](06-arguments.md#reading-parsed-input-from-argv):

````text
searched rust everywhere
````

`require` returns the value the library's code returns, whatever its type: a table, a function, a string, or `false`. A section can require a library that returns a scalar, such as a string, without failing. You can index the result in place, as in `(require '/lib').name`, and `return require '/loader'` makes the library's return value the caller's return value. Only the first value a library block returns becomes the library's value, and any extra return values are dropped.

A library's top-level statements run once, at the point it is required, and functions it defines can return values computed at that moment. A library table can mix a value computed at load time with a function that computes its result when called later, as in `lib.helped .. '|' .. lib.later()`.

A library with several Lua blocks runs them in order, and the first block that executes a `return` ends the library:

````markdown
---
name: block_returns
description: Runs a library's blocks until one returns.
runvil: 0
---

# Block returns

## Main

```lua
return require '/later'
```

## later

```lua
first = 'ran'
```

```lua
return first .. ' then second'
```

```lua
return 'never reached'
```
````

````text
ran then second
````

A block that ends with no `return` lets the next block run, and the blocks after the one that returns do not run. The rule is the one a section follows (see [Returning values](04-how-a-prompt-runs.md#returning-values)): a `return` ends the library whatever it returns. A block that returns `false` ends the library, and `require` returns `false`. A block that returns `nil`, or returns with no value, ends the library too, and `require` then returns `true`, as Lua's own `require` does for a module that returns no value. A library whose blocks all end with no `return` gives `true` to `require` as well. So a `silent` section with the blocks `quiet = 1` and `return nil` makes `tostring(require '/silent')` read `true`, and a third block written after them never runs.

A library loads successfully whether its code ends in a `return` value or is plain code with no `return`, such as `loaded = true`. Either way the run records `Lua library load succeeded: /lib` for a top-level library named `lib`.

### Library code runs in the caller's environment

Library blocks run in the requiring section entry's one Lua environment, its [section VM](#one-section-vm-per-section-entry). Globals the library sets are visible to the caller afterward, while the library's `local` variables stay inside the library:

````markdown
---
name: library_environment
description: Shows what a library shares with the section that requires it.
runvil: 0
---

# Library environment

## Main

```lua
style = 'terse'
require '/lib'
return exported .. '|' .. tostring(hidden) .. '|' .. seen_style .. '|' .. var.note
```

## lib

```lua
local hidden = 'local'
exported = 'global'
seen_style = style
var.note = 'library ran'
```
````

````text
global|nil|terse|library ran
````

Everything in that result follows from the shared environment:

- Globals a library defines at load, values and functions alike, including any function defined without `local`, are visible to and callable by the requiring code after `require` returns. The library's `local hidden` reads as `nil` to the caller.
- A library's code runs at the point of the `require` call and reads globals that the requiring block set before the call, as `seen_style` picks up `style`.
- Library code reads and writes the caller's [`var`](05-lua-environment.md#keeping-state-in-var) table, and its writes are visible to the caller after `require` returns.
- A library's `lua` blocks run one after another in file order when it loads, so a later block reads globals an earlier block set.
- A section reached only through `require` can set the metatable of `_G` for the code that required it, so one library sets up global defaults or strict checks once. A `defaults` library that runs `setmetatable(_G, { __index = { tone = 'plain' } })` gives every section that requires it the same defaults.

Global functions defined by a required section stay callable for the rest of the requiring section, in its prologue and in its epilog alike:

````markdown
---
name: prologue_epilog_library
description: Calls a library function in the prologue and in the epilog.
runvil: 0
---

# Prologue and epilog

## Main

```lua
require '/helpers'
first = decorate('prologue')
```

Some prose for Main.

```lua
return first .. decorate('epilog')
```

## helpers

```lua
function decorate(value)
  return '<' .. value .. '>'
end
```
````

````text
<prologue><epilog>
````

### Effects a library makes belong to the caller

Code in a section loaded with `require` runs as part of the calling section entry, so every store, model, and tool call it makes while it loads is issued on the caller's behalf. A [`store.write`](09-the-store.md#writing-and-reading-files) lands exactly as if the caller had made it, and a [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls) returns the tool's result.

A library's top-level code runs at load time with every global the Engine provides already installed, so `args`, `store`, `log`, and the `tools` and `models` tables all work while it loads. A library loaded with `require` also sees the globals a [Pack prelude](12-tools.md#pack-preludes) installs, including while its top-level code runs at load time.

Library blocks run inside the calling block, so a library block can make every call the caller can, including calls that pause the block, such as `tools.call` and [`models.infer`](10-models.md#running-a-round-with-modelsinfer). Each one pauses the calling block like an ordinary call made in the caller:

- A library that calls [`call`](08-jump-and-call.md#calling-a-section) during its load pauses the caller's block until the called section returns, and the library receives the call's result.
- Library code that calls `models.infer` while it loads runs one model round under the model the caller is using.
- An H1 that calls `require` lets the library set state that every later section sees, such as the current model through [`models.use`](10-models.md#selecting-a-role-with-modelsuse). The choice belongs to the chain, and the section walk starts on the model the H1 pass ended on.

### Library functions resolve names when called

Library functions look up globals when they are called, not when they are defined. A library function called from a section reads that run's current values, such as `args`, and the store writes it makes take effect for the caller:

````markdown
---
name: read_args_library
description: Calls a library function that reads args when it runs.
runvil: 0
---

# Read args

## Result

```lua
require '/helpers'
return read_args()
```

## helpers

```lua
function read_args()
  return args
end
```
````

Run it with the argument string `later canned value`:

````text
later canned value
````

A library function that calls [`jump`](08-jump-and-call.md#jumping-to-a-section), when the caller calls it after the load finishes, transfers control for the caller's section. A library that defines `function go() jump '../next' end` lets a caller that runs `require '/a'` and then `go()` move its walk to `next`.

The other functions that take a section path do not follow this rule. They resolve from the library, as the next section explains.

### A library names its own sections

A relative [section path](02-file-structure.md#section-paths) written in a library starts at the library section, not at the section that is using the library. That holds while the library loads and when its functions run later, so each line of a library means the same thing for every section that requires it. A library can therefore keep its own templates and workers as child sections and name them by their bare names:

````markdown
---
name: library_sections
description: A library reads its own template and runs its own worker.
runvil: 0
---

# Library sections

## Research

```lua
local kit = require '/kit'
var.research = kit.template .. '|' .. kit.shout('research') .. '|' .. kit.topics()
```

### Topics

- allocators
- coroutines

## Review

```lua
local kit = require '/kit'
return var.research .. '\n' .. kit.template .. '|' .. kit.shout('review') .. '|' .. kit.topics()
```

## kit

```lua
return {
  template = prompt.section('Template', 'fence markdown').text,
  shout = function(text)
    return call('Shout', text)
  end,
  topics = function()
    local list = _G.prompt.section_or('Topics', 'list')
    return list and table.concat(list.items, ',') or 'none'
  end,
}
```

### Template

```markdown
# Title
```

### Shout

```lua
return args:upper()
```
````

````text
# Title|RESEARCH|allocators,coroutines
# Title|REVIEW|none
````

`kit` reads `Template` while it loads and calls `Shout` from a function that runs later, and both paths name `kit`'s own children whether `Research` or `Review` is using the library. Neither of those sections has a `Template` or a `Shout`, and neither needs one.

The rule covers every function that takes a relative path, with one exception:

- `call`, `tasks.spawn`, `tasks.fanout`, `prompt.section`, and `prompt.section_or` written in a library start at the library section.
- `jump` starts at the section that is running. A library has no walk of its own, and `jump` moves the walk of the running section, so its path names a neighbor of that section.
- `require` takes only a path from the top, which names the same section from everywhere.

A name is resolved where it is used, not where it came from. A section that passes the string `'Worker'` to a function of a library gets the library's `Worker`, and when the library has none, the [not-found error](08-jump-and-call.md#the-not-found-error) names the library: ``section `Worker` not found: `/kit` has no child `Worker` ``. A function keeps the place it was written in, though. A function that a section passes to a library still names the section's own neighbors, and an [adhoc tool](12-tools.md#how-adhoc-tool-handlers-run) handler written in a library names the library's.

Inside a library, the `_G` table still holds the running section's own functions. `_G.prompt`, `_G.call`, and `_G.tasks` resolve from whichever section is using the library, which is how `kit.topics` reads the `Topics` list of `Research` and finds none under `Review`. The `_G` at the call shows a reader that the line depends on the user. A library that does this relies on its user to have that section, so it reads with `_G.prompt.section_or` when the section is optional. `_G.prompt.section('Topics')` in its place raises and names the user: ``section `Topics` not found: `/Review` has no child `Topics` ``. In a section's own blocks, a plain name and its `_G` name are the same function.

The chain a library's `call` starts, and the tasks its `tasks.spawn` and `tasks.fanout` start, still belong to the section that is running, as [Effects a library makes belong to the caller](#effects-a-library-makes-belong-to-the-caller) describes. Only the place a path starts from is the library's.

### Prose inside a library

A library section can carry prose alongside its Lua blocks. To share a piece of prose between sections, write a library section with a prose block and a Lua block that does `return prose`, then call `require` to get the rendered text:

````markdown
---
name: house_style
description: Shares a piece of prose through a library section.
runvil: 0
---

# House style

## Main

First one.

```lua
style = 'terse'
local text = require '/house_style'
return text .. '|' .. prose
```

## house_style

Write in a {{ style }} voice.

```lua
return prose
```
````

````text
Write in a terse voice.|First one.
````

While a library block runs, the [`prose` global](#pending-prose-and-the-prose-global) reads that library block's own prose, not the caller's. The caller's `prose` is back in place once the library block finishes, which is why `Main` still reads `First one.` after the `require`.

Each Lua block in a library section pairs with the prose block right before it, and that Markdown is what the block's `prose` renders from:

- A Lua block with no prose block right before it, including a `lua` fence that directly follows another `lua` fence, pairs with empty prose.
- Prose after the last Lua block of a library section pairs with nothing and is left out of the loaded library.

A library block's `prose` is lazy. It renders from that block's own prose when the block first reads it, at most once, against the live `sys` values at that moment, and prose with no placeholders reads back verbatim. Its [placeholders](07-substitution.md#placeholders-in-prose) render from the calling section's state, meaning its `var`, its live `sys` values, and its globals, never from a separate library state:

- `{{ style }}` in the example renders the global `style` that `Main` set before calling `require`.
- `{{ var.x }}` in a library's prose renders the caller's `var.x`.
- When a fanout [arm](14-fanout.md#what-an-arm-sees) requires a library, the library's prose renders with that arm's `item`, so each arm gets its own text.

A library block can also put its prose to other uses:

- A library section's Lua block can read the section's preceding prose through `prose` and keep it in a closure for later calls.
- A library block can read its own `prose` while raising, so its error text can include the library's rendered prose, as in `error('fault: ' .. prose)`.
- Assigning to `prose` inside a library block raises the same `prose is read-only` error as in any block (see [Keeping and reshaping the text](#keeping-and-reshaping-the-text)), `pcall` catches it, and `prose` keeps its rendered value.

A library block's own `prose` is in place only while that block runs. A library function that reads `prose` when the caller calls it after `require` returns gets the caller's `prose`, so a library that needs its prose later copies it into a local first, as in `local seen = prose`:

````markdown
---
name: library_closure
description: Keeps a library's prose in a local for later calls.
runvil: 0
---

# Library closure

## Main

Caller prose.

```lua
local lib = require '/labeler'
return lib.label('alpha') .. '|' .. lib.late()
```

## labeler

Library prose.

```lua
local seen = prose
return {
  label = function(value) return value .. ': ' .. seen end,
  late = function() return prose end,
}
```
````

````text
alpha: Library prose.|Caller prose.
````

`label` uses the copy it took while the library loaded, while `late` reads `prose` when `Main` calls it and gets `Main`'s own prose.

### The caller's `prose` after a `require`

A `require` never disturbs the caller's own pending prose:

````markdown
---
name: caller_prose
description: Shows that a require leaves the caller's prose alone.
runvil: 0
---

# Caller prose

## Main

Mood is {{ var.mood }}.

```lua
var.mood = 'calm'
local before = prose
require '/bump'
return before .. '|' .. prose
```

## bump

```lua
var.mood = 'bright'
```
````

````text
Mood is calm.|Mood is calm.
````

The caller's `prose` is lazy and renders once, on its first read. `Main` read it before the `require`, so the read after the `require` gets back the text it already rendered, without rendering again, even though the library changed `var.mood`. Now move the first read after the `require` by replacing the block of `Main` with this one:

````markdown
## Main

Mood is {{ var.mood }}.

```lua
var.mood = 'calm'
require '/bump'
return prose
```
````

````text
Mood is bright.
````

When the caller has not read `prose` before calling `require`, its `prose` stays unrendered and renders at its first read after `require` returns, so it sees any `var` or global changes the library made.

The caller's `prose` comes back the same way however the load ends. After a `require` that finishes, even one whose library read and returned its own `prose`, and after a `require` that raises and is caught with `pcall`, the caller's `prose` is still its own pending prose. It keeps any value it already rendered, whether or not the caller read it before the `require`, and reads exactly as it did before the load. When the caller has not read `prose` before a failed `require`, reading it afterward renders the caller's own prose block, not the library's.

### `require` belongs to the Engine

`require` is Runvil's own section loader, not the Lua base library's `require`. The Engine installs it as a global for each section entry's whole lifetime, so it is available in every section's Lua and any block can call it, and `type(require)` is `function`. The Engine finds the named library section and renders its prose for it. Author code that rebinds `error`, `setmetatable`, or `table.concat` does not change how `require` behaves.

Because `require` is reserved as one of the Engine's own globals, a [Pack prelude](12-tools.md#pack-preludes) cannot define its own `require`, and trying fails with an error containing `which is reserved as an Engine global`. A Pack prelude cannot call `require` while it installs either, because the Engine installs `require` after the preludes. That call fails the run with a message that starts `` Pack `{pack}`: its prelude failed to load: `` and contains `attempt to call a nil value (global 'require')`, where `{pack}` is the Pack id.

## How `require` finds and caches libraries

You can predict which section a `require` names from any place in the prompt, and when a library's code runs again.

````markdown
---
name: require_reach
description: A called child section requires the library its parent uses.
runvil: 0
---

# Require reach

## Parent

```lua
require '/lib'
return call('Child')
```

### Child

```lua
local before = tostring(greet)
require '/lib'
return before .. '|' .. greet('child')
```

## lib

```lua
function greet(name)
  return 'hello ' .. name
end
```
````

````text
nil|hello child
````

`Child` names the library by the same path its parent uses, `'/lib'`, because a path from the top names the same section from every depth. `Child` also starts without the `greet` its parent loaded, so it calls `require` itself. What follows explains both: which section a `require` names, and which `require` calls share a saved library.

### One path for each library

`require` takes only a [section path](02-file-structure.md#section-paths) from the top of the prompt: `'/lib'` for a top-level section, and one more name per level for a nested one, such as `'/Main/helpers'`. Each name in the path matches a section name exactly. The H1 title is not part of the path.

A library therefore has one spelling in the H1 body, in a top-level section, and in a nested worker section. The same `require` names the same library on every line of the prompt, inside another library included, so nesting a section deeper, or moving its code to another section, changes none of its `require` calls. The [library-load events](#library-load-events) and the [require cycle error](#the-require-cycle-error) print a library by the same path, so one search finds every place that requires it.

A relative path given to `require` raises, and the message says what `require` takes. `require 'lib'` fails with this message, and `require '../lib'` fails the same way with its own path in the quotes:

````text
section path "lib" does not start with "/"; require takes a path from the top, such as "/helpers"
````

A path names at most one section, because [sibling section names must be unique](02-file-structure.md#sibling-name-uniqueness). A prompt with two same-named siblings fails to parse with a message that starts `` duplicate sibling section name `{name}` `` and says that sibling section names must be unique.

### Which sections `require` can name

`require` can name any section in the prompt. Take this section tree:

````text
## Top
## Grand
### Uncle
#### Cousin
### Parent
#### Sibling
##### Niece
#### Self
##### Child
###### Grandchild
## Last
````

From `Self`, and from everywhere else, these paths name these sections:

| Path | Section | Relationship to `Self` |
|---|---|---|
| `/Top`, `/Last` | `Top`, `Last` | top-level sections |
| `/Grand/Uncle` | `Uncle` | a sibling of the parent |
| `/Grand/Uncle/Cousin` | `Cousin` | a child of an ancestor's sibling |
| `/Grand/Parent/Sibling` | `Sibling` | a sibling |
| `/Grand/Parent/Sibling/Niece` | `Niece` | a sibling's child |
| `/Grand/Parent/Self/Child` | `Child` | a direct child |
| `/Grand/Parent/Self/Child/Grandchild` | `Grandchild` | a grandchild |
| `/Grand/Parent`, `/Grand` | `Parent`, `Grand` | ancestors |
| `/Grand/Parent/Self` | `Self` | the section itself |

`require` is not held to the [visible set](08-jump-and-call.md#the-visible-set) that `call` and `jump` are, because it loads a section's Lua into the running section entry and leaves the walk where it is. A `Worker` nested under `Research`, including one running as a fanout arm, requires a top-level `helpers` as `require '/helpers'`, the same line the H1 body writes.

Nothing is excluded, and nothing guards against an odd `require`. A section may require its own ancestor, whose blocks then run as any library's do. A section that requires itself, and two sections that require each other, meet [the require cycle error](#the-require-cycle-error).

A library kept under the section that uses it is named through that section, as in `require '/Main/helpers'`. Renaming or moving `Main` breaks that path, and a copy of `Main` keeps requiring the original's library until its path is edited. A library that several sections share is simplest as a top-level section.

### Requires made while a library loads

A `require` written in a library means what it means everywhere else, because its path starts at the top of the prompt. A library that keeps a helper library under itself names the helper through its own path. Two libraries that each keep a nested `helpers` each get their own:

````markdown
---
name: own_helpers
description: Two libraries each load their own nested helper.
runvil: 0
---

# Own helpers

## Main

```lua
return require('/a') .. '|' .. require('/b')
```

## a

```lua
return require '/a/helpers'
```

### helpers

```lua
return 'helper of a'
```

## b

```lua
return require '/b/helpers'
```

### helpers

```lua
return 'helper of b'
```
````

````text
helper of a|helper of b
````

`require '/a/helpers'` is the one line in `a` that names `a`. Its helper loads the same way whichever section requires `a`, and the inner library finishes loading before the outer one does.

The other functions that take a section path start at the library when they are written in one, so a library names its own child sections by their bare names in `call` and the prompt reads (see [A library names its own sections](#a-library-names-its-own-sections)).

### When `require` cannot find a section

A `require` path that is malformed or names no section does not fail parsing. The prompt parses and runs, and `require` raises the error as a Lua error when the call runs, which [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch:

````markdown
---
name: require_miss
description: Catches a require whose path names no section.
runvil: 0
---

# Require miss

## Main

```lua
local ok = pcall(require, '/deep')
return tostring(ok)
```

## other

Some prose.

### deep

```lua
return 'deep'
```

## helpers

```lua
function greet(name)
  return 'hello ' .. name
end
```
````

````text
false
````

`deep` sits under `other`, so its path is `/other/deep`, and no top-level section is named `deep`. A well-formed `require` path that names no section fails with the [not-found error](08-jump-and-call.md#the-not-found-error), which names the path and the step where the lookup stopped. Here the error's text is:

````text
section `/deep` not found: the top level has no section `deep`
````

Caught as it is here, the error value's message holds that text after `require_miss.md:12: `, the line of the `pcall`, and before the default text for a Lua failure (see [Where a caught error's message points](16-limits-and-errors.md#where-a-caught-errors-message-points)).

A miss below the top level names the section that was searched. From anywhere in the tree above, `require '/Grand/Missing'` gives `` section `/Grand/Missing` not found: `/Grand` has no child `Missing` ``. The message lists no other section.

Passing anything other than a string to `require` raises a Lua runtime error that names the argument's Lua type, and no section lookup happens. For a number, the error's text is:

````text
require takes a section path such as '/helpers', got integer
````

### The library cache

A second `require` of a library that already loaded returns the saved value without running the library's blocks again:

````markdown
---
name: library_cache
description: Requires the same library twice in one section.
runvil: 0
---

# Library cache

## Main

```lua
local first = require '/counter'
local second = require '/counter'
return tostring(first == second) .. ':' .. loads
```

## counter

```lua
loads = (loads or 0) + 1
return { count = loads }
```
````

````text
true:1
````

Every `require` of one library within a section entry gets the same saved value, so a table the library returns is the same table each time, and a change one caller in that entry makes to it is seen by later callers in the same entry. A library that returns `false` is saved like any other value, so a second `require` returns `false` again from the cache without rerunning the library. A `require` of a library that already loaded successfully reuses the saved value, runs nothing, and emits no library-load report.

A library's saved value lasts as long as the [section VM](#one-section-vm-per-section-entry) of the section entry that loaded it, so every `require` in that entry shares one cache. That includes the requires a library makes while it loads, so a library that the entry or another library already loaded hands back its saved value without rerunning or reporting, whichever of the two loaded it first. A library's own code can call `require` to load another library, and the inner library finishes loading before the outer one does.

`require` keys each library by its section's position in the document. Two sections with the same name under different parents are two libraries, and each one runs once and keeps its own saved value, as the two `helpers` sections in the earlier example do.

### Each section loads its own libraries

A library saved in one section entry is not reused by a later entry, the same section entered again included. Each section entry that requires the library runs it again on its first `require`, repeating its top-level calls and recording a new library-load start and success. Several places in one run, such as the H1, a parent section and its child, a section entered with `call`, a fanout arm's section, or a later section, can each `require` the same library section without error, and each section that uses the library calls `require` for it itself.

Global functions a library defines exist only in the sections that ran it, so a later section that never required the library sees them as `nil`. A child section entered with [`call`](08-jump-and-call.md#calling-a-section) starts without its parent's libraries and loads them itself with its own `require`. That is why `Child` in the `require_reach` prompt reads `greet` as `nil` until its own `require '/lib'` runs.

Each fanout [arm](14-fanout.md#what-an-arm-sees) loads a required library in its own section VM, so the library load runs once per arm instead of being shared across arms, and each arm records its own load entry, such as `Lua library load succeeded: /helpers`. A library's top-level code runs under each requiring arm's own identity, so its load-time effects count as that arm's. A library that writes the same store file at load time with `store.write`, required by two arms, fails the run because two arms wrote the same store file, even when the library wraps the write in `pcall`.

### Libraries on the section walk

When the section walk reaches a required library in its normal place, the walk runs it as a section under the same rules as any other section. A library that a section requires and that also sits on the walk runs twice, once inside the requiring section and once as its own section when the walk reaches it. A library that only defines globals runs harmlessly, and the walk then falls through to the next section:

````markdown
---
name: library_walk
description: Lets the walk reach a library that only defines globals.
runvil: 0
---

# Library walk

## Main

```lua
require '/helpers'
var.greeting = greet('main')
```

## helpers

```lua
function greet(name)
  return 'hello ' .. name
end
```

## Last

```lua
return var.greeting .. '|' .. tostring(greet)
```
````

````text
hello main|nil
````

`Main` requires `helpers` and falls through, the walk then runs `helpers` as a section, and `Last` runs in a section VM that never required the library, so `greet` is `nil` there.

A library that returns something follows the same rules as any section that returns, which [Returning values](04-how-a-prompt-runs.md#returning-values) covers:

- When the walk reaches a library that returns a scalar, that scalar becomes the run result, and later sections never run.
- When the walk reaches a library that returns a table, the run fails with a Lua error whose message contains `cannot return a table as a result`.

Requiring a table-returning library from inside a section succeeds, because that return rule applies only when the walk runs the library as a section. You keep a returning library off the walk in one of two ways. You can place it after a section that returns, since the walk ends at that return before it reaches the library, as `search_tools` sits after `Main` in [What `require` returns](#what-require-returns). Or you can nest it as a subsection under the section that requires it:

````markdown
---
name: nested_library
description: Keeps a table-returning library off the walk by nesting it.
runvil: 0
---

# Nested library

## Main

```lua
var.name = (require '/Main/lib').name
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

The same section can be used with both `require` and `call`, and each returns that section's block value. They differ for a section whose blocks return no value: `require` returns `true`, and `call` returns `nil`.

## Library errors, cycles, and load events

You can catch a failing `require`, read where a library error points, and recognize the require cycle error and the library-load events.

````markdown
---
name: flaky_library
description: Catches a failing library load and retries it.
runvil: 0
---

# Flaky library

## Main

```lua
var.attempts = 0
local first_ok = pcall(require, '/flaky')
local second = require '/flaky'
return tostring(first_ok) .. '|' .. second .. '|' .. var.attempts
```

## flaky

```lua
var.attempts = var.attempts + 1
if var.attempts == 1 then
  error('the first load fails')
end
return 'loaded'
```
````

````text
false|loaded|2
````

### Ways a `require` fails

A `require` fails in one of five ways, and each one raises a Lua error in the requiring block:

- The argument is not a string. [When `require` cannot find a section](#when-require-cannot-find-a-section) gives the message.
- The path is malformed, is not a path from the top, or names no section. The same place gives the not-found message, and [One path for each library](#one-path-for-each-library) gives the message for a relative path.
- The library's code raises an error while it loads.
- The library's code calls `jump` while it loads.
- The library is already loading, which makes a require cycle.

### Catching a failed load

An error that library code raises while loading travels out of `require` into the block that required it. You catch it with `pcall(require, path)`, which returns `false` along with the error, and the section keeps running and can still return normally, such as with `return 'done'`. The error reaches the caller as the same value, unchanged, whether the library raised a string or a table.

A library load that raises is not saved, so every later `require` of that library runs its code again from the top and raises again if the fault remains. Once the fault is gone, the retry returns the library's value, and the library's code has run twice. In the example, `flaky` fails only while `var.attempts` is `1`, so the caught first load and the successful retry give `false|loaded|2`.

What a failed load already did stays done:

- [`var`](05-lua-environment.md#keeping-state-in-var) writes a library makes before it raises stay in place after the error reaches the caller. That is why `var.attempts` is still `1` when the retry starts. Likewise, a library that sets `var.broken_saw = prose` before `error('the style broke')` still leaves `var.broken_saw` readable by the caller.
- Globals a library set before its load failed stay set in the caller's environment, so `partial = 'kept'` written before `error('library fault')` still reads `kept` in the caller.

A required library runs inside the Lua block that requires it, so an uncaught raise in library code is that block's own failure. A failing library required in the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) fails the H1 pass, so the run stops before the walk starts, with an error whose cause contains the library's message, such as `library boom`.

### Where a library error points

A caught library error and an uncaught one point at the fault in different forms. This prompt catches one:

````markdown
---
name: library_fault
description: Shows where a caught library error points.
runvil: 0
---

# Library fault

## Main

```lua
local ok, err = pcall(require, '/lib')
return tostring(err)
```

## lib

```lua
error('load fault')
```
````

````text
[string "line 19"]:1: load fault
````

A library error caught with `pcall(require, path)` comes back in standard Lua form, `[string "line {source_line}"]:{k}: {message}`. `{source_line}` is the prompt line of the library block's first Lua line, here line 19, and `{k}` is the line within that block, here its first. For an `error` call on the third line of a block that starts at prompt line 100, the caught error reads `[string "line 100"]:3: library fault`.

Only an uncaught failure is rewritten to the library's location and prompt line. Replace the block of `Main` with one that calls `require` without `pcall`, keeping the same number of lines:

````markdown
## Main

```lua
require '/lib'
return 'done'
```
````

The run fails, and its error message contains this text and ends with ` (at library_fault.md:19)`, the library's line:

````text
section `lib` prologue:19: load fault
````

An uncaught Lua error raised while a library loads is reported as `` section `{name}` {block}:{absolute}: {result} ``. It names the library section by its bare name and gives the failing statement's prompt-file line, at the library block's location rather than at the `require` call site. The location names the full section title even when the title is long, such as `` section `Research Helper Functions For Every Worker` prologue:19: load fault ``. In a library with more than one Lua block, an uncaught error maps to the block that raised it and that block's own prompt lines, using the labels in [How errors name each block](#how-errors-name-each-block), such as `` section `lib` lua 3:21: second fault `` for a raise on the second line of the section's third Lua block, which starts at prompt line 20.

An uncaught error from a function the library defined points at the library too, even when the caller calls that function after the load:

````markdown
---
name: later_fault
description: A library function fails after the load.
runvil: 0
---

# Later fault

## Main

```lua
require '/lib'
boom()
```

## lib

```lua
function boom()
  error('later fault')
end
```
````

The run fails, and its error message contains both of these locations and ends with ` (at later_fault.md:20)`:

````text
section `lib` prologue:20: later fault
section `Main` prologue:13:
````

The library's line 20 is where `error` runs, and it is the line the message ends with. The caller's own frames in the traceback map to the caller's location and prompt lines as well, each frame mapped once with no raw `[string "..."]` marker left, so the message also shows line 13 of `Main`, the caller's `boom()` line.

### `jump` during a library load

Load code cannot transfer control. Calling [`jump`](08-jump-and-call.md#jumping-to-a-section) from library code while the library loads through `require` raises a Lua error with this text:

````text
jump is unavailable while a library loads: return a value from the library and call jump from the section after require returns
````

The refusal ends with the load, so a caller that catches the failed load with `pcall` can still `jump` afterward, as in `pcall(require, '/a')` followed by `jump '../after'`. A library function that calls `jump` after the load has finished transfers control normally, as [Library functions resolve names when called](#library-functions-resolve-names-when-called) shows.

### The require cycle error

A `require` of a library that is still loading raises a Lua error before anything runs again:

````markdown
---
name: require_cycle
description: Catches a require cycle between two libraries.
runvil: 0
---

# Require cycle

## Main

```lua
local ok, err = pcall(require, '/a')
return err.kind .. '|' .. tostring(err)
```

## a

```lua
require '/b'
```

## b

```lua
require '/a'
```
````

````text
lua|require cycle: /a -> /b -> /a
````

The cycle can come from the library's own blocks or through other libraries it requires, as here, where `a` requires `b` and `b` requires `a`. The message starts with `require cycle:` and a space, followed by each library's path from the top in load order, joined by `->` with a space on each side and ending with the path required again. It starts at the repeated library's own load and lists only loads from there inward, leaving out outer libraries that started loading earlier. A section that requires itself is the shortest cycle: `require '/Main'` written in `Main` gives `require cycle: /Main -> /Main`.

The require cycle error is a structured [error value](05-lua-environment.md#raising-and-catching-errors) with kind `lua` and a `message` field holding the cycle text, which `tostring(err)` also returns.

### Library-load events

The first `require` of a library emits a `lua_library_load_started` report as the load begins, before the library's code runs, and a `lua_library_load_succeeded` report when that code finishes without error, in that order, each naming the library loaded. In readable form they appear as:

````text
Lua library load started: /helpers
Lua library load succeeded: /helpers
````

Each phase of a library load is its own event, of kind `lua_library_load_started`, `lua_library_load_succeeded`, or `lua_library_load_failed`. Each event carries the requiring section in `section` and the library in `library`:

- `section` is the name of the section running the load, such as `Worker`.
- `library` is the library section's path from the top of the prompt, such as `/helpers`. It is rebuilt from the section rather than copied from the string you passed to `require`.

A library whose code raises during its load emits `lua_library_load_started` and then `lua_library_load_failed`, even when the caller catches the error with `pcall`, and every retry of a failing library emits a fresh pair. A load counts as succeeded only when all its blocks finish. The `flaky_library` prompt reports these four events, each with `section` set to `Main` and `library` set to `flaky`:

1. `lua_library_load_started`
2. `lua_library_load_failed`
3. `lua_library_load_started`
4. `lua_library_load_succeeded`

Because the `section` field names the section running the load, the same library loaded by two sections reports once under each, such as once under `First` and once under `Second`. A child section that requires a library its parent already loaded runs its own load, with its own started and succeeded events under the child's section name, so `Parent` events for `lib` are followed by `Child` events. A `require` answered from the cache reports nothing.
