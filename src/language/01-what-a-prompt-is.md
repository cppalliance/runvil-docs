# Runvil prompts: structure and execution

A Runvil prompt is a single Markdown file (`.md`) that combines instructions for a model with Lua code that controls execution.

This chapter introduces the prompt-file structure, shows how prose and Lua work together, and explains how a prompt interacts with models, tools, the store, and its runtime environment.

## 1. Prompt-file structure

A prompt has three parts, in this order:

1. **Frontmatter**, starting on the first line and enclosed by `---` delimiters, in YAML format.
2. **One H1 title**, with non-empty text. The title may be followed by an H1 body containing prose and Lua blocks.
3. **Optional H2 sections** (`##`), which may contain prose, Lua blocks, or nested subsections.

### A minimal prompt

````markdown
---
name: greeter
description: Says hi.
runvil: 0
---

# Greeter
````

This prompt runs successfully and returns `done`, Runvil's default result when no code returns a value.

Every runnable prompt must declare `runvil: 0`,  the only Runvil version currently accepted.

The `name` and `description` are text that make sense to the user. In addition to these, frontmatter can declare optional contract keys:

- `uses:` lists the Packs the prompt requires.
- `models:` declares model roles.
- `args:` declares the prompt's arguments and their types.

Parsing checks the structure of these keys. During preparation, Runvil checks them against the run environment. See [Declaring arguments in frontmatter](06-arguments.md#declaring-arguments-in-frontmatter), [Model roles instead of models](10-models.md#model-roles-instead-of-models), and [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key).

### Prose and Lua

Write model-facing instructions as Markdown prose and execution logic in fenced Lua blocks. A Lua block begins with a code fence labelled `lua`; each block runs as a unit. Prose is collected as pending `prose`, which a Lua block can return as the run result or pass to a model with `models.infer(prose)`.

For example, this prompt returns its prose directly; it does not call a model:

````markdown
---
name: greeter
description: Says hi.
runvil: 0
---

# Greeter

## Say hi

Say hello.

```lua
return prose
```
````

The result is:

````text
Say hello.
````

**Prose does not reach a model automatically.** Lua must explicitly send it to a model, usually with `models.infer(prose)`. If no Lua block reads the prose, Runvil discards it at the end of that part and continues without making a model request. This applies both to prose in the H1 body and to prose-only sections.

A Lua block can also return an ordinary value:

````markdown
---
name: greeter
description: Says hi.
runvil: 0
---

# Greeter

## Say hi

```lua
return "hello"
```
````

For more on pending prose and Lua blocks, see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global).

## 2. Combining Lua, prose, and a model

The following prompt illustrates the usual flow: Lua prepares a value, prose incorporates it, and another Lua block sends that prose to a model.

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

Run it with the argument `Ada`. The model receives:

````text
Repeat exactly, with no extra words: Hello, Ada!
````

The pieces work together as follows:

1. `models:` declares the `writer` role in frontmatter.
2. `models.default("writer")` sets that role as the prompt-wide default.
3. The first Lua block in `## Main` builds a greeting from `args` and stores it in `var.greeting`.
4. The placeholder `{{ var.greeting }}` inserts the stored value into the section's prose.
5. `models.infer(prose)` sends the resulting prose to the model, and `return` makes the reply the run result.

See [The prompt-wide default with `models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault), [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var), [The argument string in `args`](06-arguments.md#the-argument-string-in-args), and [Placeholders in prose](07-substitution.md#placeholders-in-prose).

### Use the store without a model

A prompt needs no model role or tool if ordinary Lua is sufficient. For example:

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

The run result is `hello`. Store access is an effect but the prompt does not need a model round.

## 3. Sections, state, and tools

Sections let you divide a prompt into steps. Lua can keep values in `var`, call other sections, and load shared code. Sections execute as the run directs them; values in `var` and values passed by `call` can carry results between steps.

A helper section can define a function for other sections to load with `require '## helpers'`. A prompt can also contain nested headings such as `### Author note` beneath an H2 section. See [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require) and [Calling a section](08-jump-and-call.md#calling-a-section).

### Calling a tool

A Lua block can call a tool with `tools.call`. It can also define an ad hoc tool with `tools.adhoc` and call it from Lua:

````lua
local shout = tools.adhoc({
  name = 'shout',
  description = 'Returns the text in capital letters.',
  parameters = { text = 'string' }
}, function(args)
  return string.upper(args.text)
end)

return tools.call(shout, { text = reply })
````

Here `reply` is a value already obtained by the prompt. The `shout` handler is a Lua function. If `reply` is `hi there`, the block returns `HI THERE`.

Tools can be supplied by Packs. A prompt that needs a Pack declares it under `uses:`. For example:

````yaml
---
name: researcher
description: Researches a topic on the web.
runvil: 0
uses:
  - web
---
````

Refer to [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer).

### Combining a store, a model, and a tool

One Lua block can combine several kinds of work:

````lua
-- Assume the writer role declared under models: in the frontmatter
store.write('note.md', 'hello')
models.use('writer')
local reply = models.infer(store.read('note.md'))

local utflen = tools.adhoc({
  name = 'utflen',
  description = 'Returns the length of text in UTF-8 characters',
  parameters = { text = 'string' }
}, function(args)
  return utf8.len(args.text)
end)

return tools.call(utflen, { text = reply })
````

This block writes and reads a note, selects the `writer` role, sends the note to the model, and passes the model's reply to the `adhoc` tool. If the model replies `hi there`, the run result is `8`. The model does not see the `adhoc` tool unless the prompt offers it.

For model conversations and loops, see [Starting a conversation](11-conversations.md#starting-a-conversation). For tool-call details, see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls).

## 4. The Engine, Harness, and Host

These three components have distinct responsibilities:

- **Engine:** parses the prompt and advances its execution.
- **Harness:** prepares each run, performs outside work requested by the Engine, returns the answers, and records run events.
- **Host:** the application that starts the run and sets its policy. Workshop is one example of a Host.

The Host supplies the prompt text to the Harness, which uses the Engine to parse it. The Harness then drives the run: when the Engine requests outside work, the Harness performs it and resumes execution with the result.

### Run policy and inputs

The Host sets run policy, including which model is used and whether someone can answer `input.ask()`. It can also cancel a run or stop a model round that is in progress.

The Harness assigns a secure random seed and takes the run's start time from the clock. A prompt can observe three run inputs:

- **Seed:** exposed indirectly through the nonce in each `untrusted()` envelope.
- **Start time:** available as `sys.when`, in RFC 3339 form.
- **UI snapshot:** available through `ui()`. If no snapshot exists, `ui` is `nil`.

Given the same seed, start time, and outside answers, a run repeats exactly.

For example, `return sys.when .. '|' .. untrusted('data')` returns the start time, a separator, and the envelope returned by `untrusted('data')`. The envelope includes a 32-character hexadecimal nonce derived from the seed. See [Handing store text to a model with `untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted) and [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui).

The Host can stop a model round independently of cancelling the entire run. A stopped round raises an error value whose kind is `cancelled`; Lua can catch it with `pcall`. If the error is not caught, the run ends as cancelled. See [Error values](05-lua-environment.md#error-values) and [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors).

### Packs and the tool catalog

A **Pack** bundles tools and may provide a Lua prelude. A **Plugin** is an entry point that adds zero or more Packs. The Host initializes Plugins during setup; the Packs they add are built then and shared across runs.

For each run, the Harness creates a tool catalog from the run's Packs: the Runvil Packs plus the third-party Packs declared by the prompt. Packs must be declared with `uses:`.

## 5. Effects and Harness work

An **effect** is a request for work that must be performed outside the prompt's ordinary Lua execution. When a block requests an effect, it waits while the Harness performs the work, then resumes with either the result or the failure.

For example, when there is a `store.write`, followed by a `store.read` (see the example in section 2) the block waits for the write to complete, resumes, waits for the read, and then returns the read result.

The Lua environment does not provide general operating-system or file I/O. Inside a block, `io` and `os` are `nil`; the prompt can access time only through run facts such as `sys.when`. Its path to outside work is through the defined effect system.

### The five kinds of Harness work

Runvil defines five kinds of Harness work:

1. **Model round:** a request to a model and its reply, or the reason the request failed.
2. **Tool render:** before the first loop round for a section entry that offers catalog tools, the Harness asks relevant Packs how to present their tools to the model.
3. **Pack tool call:** a call to a tool supplied by a Pack.
4. **Store operation:** any store read or write.
5. **Timed wait:** the timer used for a timed wait on tasks.

A tool render occurs once per relevant section entry, before its first model round. Each Pack may change how its tools are presented: it can rename tools, reshape descriptions or schemas, omit tools, or add tools visible only to the model. A Pack without rendering passes its offered tools through unchanged. See [Tool scope](12-tools.md#tool-scope) and [What a Pack's render can change](12-tools.md#what-a-packs-render-can-change).

`input.ask()` is also a Pack tool call. The user-input Pack routes it through `tools.call(pack .. "/ask")`, so waiting for the operator's answer is Harness work. Every store call - including reads - is a store operation.

The Harness answers every effect exactly once, with either its result or its failure. A model round stopped by the Host is answered with a cancellation failure.

### What does not require Harness work?

Ordinary Lua operations - such as calculating a value or returning a string - run inside the Engine and produce no effect, so do not require the Harness. Calling an ad hoc tool also runs its handler inside the Engine, so it does not require Harness work either. For details and more examples, refer to the [Tools chapter](12-tools.md).


## 6. Shipped example prompts

The repository's `prompts` directory, including its subdirectories, contains working examples:

- **`echo.md`:** returns its argument unchanged with `return args`; it does not make a model round. See [The argument string in `args`](06-arguments.md#the-argument-string-in-args).
- **`hello.md`:** sends `Say "Hello, world!"` to a model and returns the reply. See [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer).
- **`greet.md`:** builds a greeting from the prompt argument, inserts it into prose with `{{ var.greeting }}`, and asks the model to repeat it. See [Placeholders in prose](07-substitution.md#placeholders-in-prose).
- **`analyst-example.md`:** declares an `analyst` model role and selects it with `models.use("analyst")` before calling `models.infer(prose)`. See [Model roles instead of models](10-models.md#model-roles-instead-of-models).
- **`research-person.md`:** researches a person by offering `web/search` and `web/fetch`. Its argument is inserted after `Your input is a request about a person:`. For example, the argument `Ada Lovelace` appears in the prose after that line. See [What the web Pack gives a prompt](13-web-fetch-and-search.md#what-the-web-pack-gives-a-prompt).

Use these files as runnable starting points when exploring a feature in more detail.
