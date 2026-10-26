# Substitution

A `{{ }}` placeholder in prose inserts a run value into the text that a Lua block reads through [`prose`](03-blocks-and-prose.md#pending-prose-and-the-prose-global), so the words you write for a model can hold the argument string, values your Lua computed, and facts about the run. This chapter covers every placeholder form, how each kind of value renders, the moment substitution happens, how to write literal braces, and the errors a placeholder can raise. With these rules you can write prose that renders the same way every time and fails loudly when a value is missing.

## Placeholders in prose

After this section you can put a run value into a section's prose with a placeholder and read the rendered text from Lua.

````markdown
---
name: greet
description: Greets the argument string
runvil: 0
---

# Greet

## Main

hi {{ args }}!

```lua
return prose
```
````

Run with the argument string `Acme Corp`, the run result is:

````text
hi Acme Corp!
````

A placeholder is `{{ path }}` written inside a section's prose. It runs from `{{` to the next `}}`, and the path between them is trimmed of surrounding spaces. When a Lua block reads `prose`, the Engine replaces each placeholder with a live value from the run, so the text the block gets already holds those values. A placeholder sits inline and is replaced in place, so the words around it, here `hi ` and `!`, stay exactly as written.

`{{ args }}` inserts the argument string the run was given. Besides inline use like the one above, a prompt can put `{{ args }}` on its own line after an instruction that asks the model to analyze the following text:

````markdown
Analyze the following input:

{{ args }}
````

### The substitution sources

The first segment of a path names where the value comes from. That segment is the substitution source, and there are five built-in sources plus Lua globals:

| Placeholder | Inserts |
|---|---|
| `{{ args }}` | The argument string |
| `{{ argv }}`, `{{ argv.field }}` | The parsed arguments, whole or one field |
| `{{ item }}` | The item seeded into a fanout arm or a task |
| `{{ var.key }}` | A value in `var` |
| `{{ sys.key }}` | A field of `sys` |
| `{{ name }}`, `{{ name.field }}` | A section-local Lua global, whole or one field |

Any first segment other than `args`, `argv`, `item`, `var`, or `sys` is read as the name of a Lua global. Each source has its own H2 later in this chapter: [Inserting `var` and `sys` values](#inserting-var-and-sys-values), [Inserting arguments with `args` and `argv`](#inserting-arguments-with-args-and-argv), [Inserting the fanout or task `item`](#inserting-the-fanout-or-task-item), and [Inserting Lua globals by bare name](#inserting-lua-globals-by-bare-name).

### Prose only, never Lua source

````markdown
---
name: literal
description: Shows that Lua source keeps its braces
runvil: 0
---

# Literal

## Main

```lua
return '{{ args }} stays literal; the input is ' .. args
```
````

Run with the argument string `docs`, the run result is:

````text
{{ args }} stays literal; the input is docs
````

Only a section's prose is substituted. Lua source never is, so `{{ }}` written inside Lua code, such as in a string literal, stays exactly as written. The block reads the argument string through the [`args` global](06-arguments.md#the-argument-string-in-args) instead.

### Setting a value, then reading the rendered prose

````markdown
---
name: subject
description: Writes about the argument string
runvil: 0
---

# Subject

## Main

```lua
var.subject = args
```

Write about {{ var.subject }}.

```lua
return prose
```
````

Run with the argument string `rivers`, the run result is:

````text
Write about rivers.
````

This is the usual shape. A [prologue](03-blocks-and-prose.md#prologue-prose-and-epilog) sets a value in [`var`](05-lua-environment.md#keeping-state-in-var), the prose names it with a placeholder, and the epilog reads `prose`, which already holds the rendered text. The values are the ones in effect at the read, so a block that sets `var.word = 'mutated'` and then reads the prose `The word is {{ var.word }}.` gets `The word is mutated.`

The rendered text can go straight to a model. This section, which needs a model as [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer) explains, sends the prose with the greeting already in it:

````markdown
## Main

```lua
var.greeting = "Hello, " .. args .. "!"
```

Repeat exactly, with no extra words: {{ var.greeting }}

```lua
return models.infer(prose)
```
````

The lead-in words `Repeat exactly, with no extra words:` stay in place, and the placeholder after them is replaced.

### A placeholder that cannot render fails

````markdown
---
name: ghost
description: Reads prose that names an unset global
runvil: 0
---

# Ghost

## Main

{{ ghost }} here.

```lua
return prose
```
````

The run fails, and its error message contains `unknown namespace or global 'ghost'`. A placeholder never silently renders blank. Every substitution failure is a hard error, raised as an ordinary Lua error at the moment a block reads `prose`. The prose `Missing: {{ var.missing }}.` read the same way fails the run with an error containing `missing {{ var.missing }}`. Each failure has a substitution kind, listed in [Substitution error kinds](#substitution-error-kinds), and you can catch one as [Catching and reading substitution errors](#catching-and-reading-substitution-errors) shows.

## Writing a path

After this section you can write paths that reach nested values and know exactly which text is read as the path.

````markdown
---
name: cell
description: Reads a nested value with a dotted path
runvil: 0
---

# Cell

## Main

```lua
var.row = { a = 1 }
```

cell {{ var.row.a }}, again {{var.row.a}}

```lua
return prose
```
````

The run result is:

````text
cell 1, again 1
````

A path is one or more segments joined by dots, as in `var.row.a`. A dotted path walks into nested tables one key per segment: `var` names the source, `row` the table stored there, and `a` the field inside it. The text between `{{` and `}}` is trimmed before it is used as a path, so the spaces just inside the braces are optional, and `{{args}}`, `{{ args }}`, and `{{   args   }}` all mean the same thing.

### Where a placeholder ends

A placeholder ends at the first `}}` after its opening `{{`, so a path never contains `}}`, and the prose picks up again right after that `}}`. A backslash inside a placeholder is plain path text, not an escape, so the placeholder still closes at the first `}}`, even one written right after a backslash. Escapes work only in prose outside placeholders, as [Escapes and literal braces](#escapes-and-literal-braces) shows. Because the closing `}}` can sit anywhere later in the prose, and the trim also removes newlines just inside the braces, a placeholder can span lines:

````markdown
Input: {{
args
}}
````

This means the same as `Input: {{ args }}`.

### The segment rules

Every segment must be nonempty, and no segment may start or end with whitespace. Spaces between other characters are fine, and the segment text is used as the key unchanged, so a key that contains a space is reached by writing it as is:

````markdown
---
name: spaced
description: Reads a key that contains a space
runvil: 0
---

# Spaced

## Main

```lua
var['first name'] = 'Ada'
```

Hello {{ var.first name }}.

```lua
return prose
```
````

The run result is:

````text
Hello Ada.
````

This works the same under `var`, `sys`, and `argv`. A path with an empty or space-padded segment fails with substitution kind `EmptySegment`, which covers a trailing dot, two dots in a row, and a space next to a dot, as in `{{ var. }}`, `{{ var..x }}`, `{{ var. .x }}`, `{{ var.x. }}`, and `{{ var. .x .y }}`.

Every segment matches exactly, case included. Only the lowercase roots `args`, `argv`, `item`, `var`, and `sys` are built in, so any other spelling of the first segment is read as a global name, and each key after it must match the table's spelling exactly. A key that itself contains a dot cannot be reached by a path, because every dot starts a new segment before any lookup, so copy such a value to a plain key in Lua first and reference that key.

### Whole values and keys

`{{ args }}`, `{{ argv }}`, `{{ item }}`, and a bare global name can each render a whole value, and a bare global with no key renders its whole value as JSON. `var` and `sys` always need a key, so `{{ var }}` and `{{ sys }}` each fail with substitution kind `BadPath`.

### Computing values in Lua

A placeholder is a plain lookup with no arithmetic or expressions, so compute a derived value in Lua, keep it in `var` or a global, and reference the result:

````markdown
---
name: full_name
description: Joins two names in Lua before the prose reads them
runvil: 0
---

# Full name

## Main

```lua
var.first = 'Ada'
var.last = 'Lovelace'
var.full = var.first .. ' ' .. var.last
```

Name: {{ var.full }}

```lua
return prose
```
````

The run result is:

````text
Name: Ada Lovelace
````

Operator text inside the braces is read as part of the path, so `{{ var.n + 1 }}` looks up a key named `n + 1` and fails with substitution kind `MissingKey` instead of computing.

Dotted segments match table keys only and never index into an array, so pick the element in Lua first. With this section in place of the `## Main` section of the `full_name` prompt, the run result is `First tag: red`:

````markdown
## Main

```lua
var.tags = { 'red', 'green' }
var.first_tag = var.tags[1]
```

First tag: {{ var.first_tag }}

```lua
return prose
```
````

A path that aims a segment at an array element fails with substitution kind `MissingKey`.

## Inserting `var` and `sys` values

After this section you can insert values your Lua stored in `var` and facts the run keeps in `sys`.

````markdown
---
name: paper
description: Renders values the prologue stored in var
runvil: 0
---

# Paper

## Main

```lua
var.kind = 'library'
var.count = 3
```

a {{ var.kind }} paper, {{ var.count }} copies

```lua
return prose
```
````

The run result is:

````text
a library paper, 3 copies
````

A `var` placeholder, `{{ var.key }}`, inserts the current value the section's Lua wrote to that field of `var`. A string field renders as its raw text with no quotes around it, and a number field in its plain form, so `var.kind = 'library'` renders as `library` and `var.count = 3` renders as `3`.

The placeholder resolves against values set by Lua blocks that already ran. A value the prologue stores, such as `var.question`, is read in the same section's prose with `{{ var.question }}`, because the prose renders after the prologue runs. Prose between two Lua fences renders after the earlier fence runs, so when the first fence runs `var.n = 2`, the prose `Second fence {{ var.n }}.` before the next fence renders `Second fence 2.` The placeholder sees the current contents of `var`, nested tables included, as plain JSON, so the prologue or any earlier block can set the value the prose uses.

### Values from earlier on the walk

A `var` placeholder can insert a value that the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) or an earlier section on the walk wrote, because each section starts with the walk's current `var` and the render reads that table:

````markdown
---
name: plan
description: Renders a value the H1 pass stored
runvil: 0
---

# Plan

```lua
var.topic = args
```

## Draft

Plan for {{ var.topic }}.

```lua
return prose
```
````

Run with the argument string `rivers`, the run result is:

````text
Plan for rivers.
````

### Nested tables in `var`

A dotted path inside a `var` placeholder reaches a nested field of a table stored in `var`, and a placeholder that names a whole table or array renders it as compact JSON, the JSON text with no spaces. With this section in place of the `## Main` section of the `paper` prompt, the run result is `cell 1, row {"a":1}, arr [1,2,3]`:

````markdown
## Main

```lua
var.row = { a = 1 }
var.arr = { 1, 2, 3 }
```

cell {{ var.row.a }}, row {{ var.row }}, arr {{ var.arr }}

```lua
return prose
```
````

[How values render](#how-values-render) gives the full rules for compact JSON.

### Missing keys

A placeholder that names a `var` key that does not exist, such as `{{ var.nope }}`, fails instead of rendering empty text, with substitution kind `MissingKey`.

### Fields of `sys`

A `sys` placeholder, `{{ sys.key }}`, inserts a field of [`sys`](05-lua-environment.md#run-facts-in-sys-and-ui), the facts the run provides, such as `{{ sys.id }}` or `{{ sys.when }}`:

````markdown
id {{ sys.id }} at {{ sys.when }}
````

The run decides which `sys` fields exist. A placeholder that names a field that does not exist, such as `{{ sys.bogus }}`, fails like any missing key, with substitution kind `MissingKey` and a message that repeats the placeholder after `missing`, here `missing {{ sys.bogus }}`.

Once the section has made its first tool call (see [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls)), `{{ sys.model }}` inserts the id of the section's model, such as `claude-sonnet-4-6`. Before that call `sys` has no `model` field, so a Lua block that reads `sys.model` first fails with `unknown sys field 'model'`, and the placeholder fails like any missing `sys` field.

### The moment `var` and `sys` are read

When a block reads `prose`, each `var` placeholder renders from the section's own `var` table as it stands at the first read, so a later change such as `var.word = 'changed'` does not change the text that later reads return. `sys` is read as it stands at that same moment, and the read sees the live `sys`, so a `sys` field the run adds after the Lua block starts but before that first read, such as `sys.model` after a tool call, is visible in the rendered text. [When substitution happens](#when-substitution-happens) covers this timing in full.

### `var` in H1 body prose

Prose in the [H1 body](02-file-structure.md#the-h1-title-and-h1-body) can use `{{ var.key }}` too. When an H1 block reads `prose`, the placeholder renders from the `var` values the H1 blocks have written so far.

## Inserting arguments with `args` and `argv`

After this section you can insert the raw argument string, the whole parsed arguments, or one field of them.

````markdown
---
name: raw
description: Inserts the argument string exactly as passed
runvil: 0
args: { query: { type: string } }
---

# Raw

## Main

Args: {{ args }}

```lua
return prose
```
````

Run with the argument string `  spaced { not json `, which has two leading spaces and one trailing space, the run result keeps every one of them:

````text
Args:   spaced { not json 
````

An `args` placeholder, `{{ args }}`, renders the run's single raw argument string exactly as passed: unparsed, unquoted, and with every character kept, leading spaces and stray braces included. It works whether or not the argument string is JSON, under any [`args:` declaration](06-arguments.md#declaring-arguments-in-frontmatter), as here, or in a prompt with no `args:` key at all, and `return prose` gives back the rendered text, as `hi {{ args }}!` showed at the start of this chapter.

A common pattern puts `{{ args }}` on its own line after an instruction such as `Your input is a request about a person:`, and because `prose` already has the placeholder replaced, `models.infer(prose)` over the prose `Summarize in one word: {{ args }}` in a run with the argument string `quantum` sends `Summarize in one word: quantum` to the model.

### The whole parsed arguments

````markdown
---
name: query
description: Inserts the parsed arguments whole and by field
runvil: 0
args: { query: { type: string } }
---

# Query

## Main

Whole: {{ argv }}; Field: {{ argv.query }}

```lua
return prose
```
````

Run with the argument string `{"query":"papers","n":2}`, the run result is:

````text
Whole: {"n":2,"query":"papers"}; Field: papers
````

`args` and `argv` are the two argument roots a placeholder can start from, and one line of prose can use both `argv` forms, as this one does. An `argv` placeholder, `{{ argv }}`, inserts the whole parsed arguments held in [`argv`](06-arguments.md#reading-parsed-input-from-argv), which under an explicit `args:` declaration is the decoded JSON of the argument string. An object or array renders as compact JSON with object keys in sorted order, not the order the argument string lists them, so the rendered value has no spaces, `n` comes before `query`, and the integer `2` has no decimal part. `got {{ argv }}` with the same argument string renders `got {"n":2,"query":"papers"}`. When the parsed value is a scalar, `{{ argv }}` renders it as its plain value, so the argument string `42` renders `42`.

### Fields at any depth

`{{ argv.key }}` reads one field of the parsed arguments, and a longer dotted path walks nested objects one key at a time, at any depth:

````markdown
q={{ argv.query }} cell={{ argv.row.a }}
````

As the prose of the `query` prompt's `## Main` section, this line run with the argument string `{"query":"papers","row":{"a":1}}` renders `q=papers cell=1`. With the same argument string, `got {{ argv }}` renders `got {"query":"papers","row":{"a":1}}`.

### Prompts with no `args:` key

With no `args:` key, the [implicit default declaration](06-arguments.md#prompts-with-no-args-key) applies, and `argv` holds the argument string wrapped as `{"prose": ...}`, so `{{ argv.prose }}` renders the argument string and `{{ argv }}` renders the whole wrapper:

````markdown
---
name: wrapped
description: Shows argv under the default declaration
runvil: 0
---

# Wrapped

## Main

{{ argv.prose }} / {{ argv }}

```lua
return prose
```
````

Run with the argument string `hello there`, the run result is:

````text
hello there / {"prose":"hello there"}
````

Here `argv.prose` and `args` both equal `'hello there'`.

### Dotted paths that fail

`args` is a plain string, so a dotted path into it, such as `{{ args.x }}`, fails with substitution kind `NotATable` and the message `args is a string, not a table`. A dotted `argv` path that reaches a field `argv` does not have, or that continues into a scalar, fails with substitution kind `MissingKey` and a message of the form `missing {{ {path} }}`, and it never renders an empty string. So `{{ argv.x }}` fails when `argv` is the string `"scalar"`, and `{{ argv.query.x }}` with a string `query` fails with `missing {{ argv.query.x }}`.

### When `argv` is nil

Under an explicit declaration, `argv` is nil for substitution when the argument string does not parse as JSON, when it is the JSON `null`, or when the H1 pass leaves `argv` nil, so the argument strings `not json` and `null` both leave a nil `argv` under a `query` declaration. The default declaration always wraps the string instead. An `argv` placeholder never renders an empty string for a nil `argv`: `{{ argv }}` and every `{{ argv.field }}` fail with substitution kind `NilArgv`, the kind for a nil `argv` (see [Freezing `argv` after the H1 pass](06-arguments.md#freezing-argv-after-the-h1-pass)), with a message of the form `{{ {path} }} is nil (the args string is not JSON)` that uses the placeholder's own path. Check `argv` in Lua first and read `prose` only when the argument string parsed:

````markdown
---
name: guarded
description: Falls back when the argument string does not parse
runvil: 0
args: { query: { type: string } }
---

# Guarded

## Main

Value: {{ argv }}

```lua
if not argv then return 'no usable input' end
return prose
```
````

Run with the argument string `not json`, the run result is:

````text
no usable input
````

Run with the argument string `{"query":"x"}`, the run result is:

````text
Value: {"query":"x"}
````

Without the guard, the read of `prose` under `not json` fails with an error naming `argv`, `{{ argv }} is nil (the args string is not JSON)`. The guard works because prose that a block never reads never renders, so even prose holding an unclosed `{{` or a missing key runs clean when the block never reads `prose`.

### The frozen `argv` and H1 repair

In every section after the H1 pass, both `argv` placeholder forms render the same frozen `argv` that the section's Lua reads. An `argv` repaired in the H1 body (see [Checking and repairing `argv` in the H1 pass](06-arguments.md#checking-and-repairing-argv-in-the-h1-pass)) reaches substitution from the first walked section on:

````markdown
---
name: repaired
description: Repairs argv in the H1 pass before a section renders it
runvil: 0
args: { query: { type: string } }
---

# Repaired

```lua
if not argv then argv = { query = 'repaired' } end
```

## First

Query: {{ argv.query }}

```lua
return prose
```
````

Run with the argument string `broken input`, the run result is:

````text
Query: repaired
````

The section's Lua sees `argv.query == 'repaired'`, and its prose renders the same value. The H1 body's own prose still sees the parse of the run's argument string, because the repaired `argv` takes effect only when the H1 pass ends, and placeholders read `argv` from the [chain](04-how-a-prompt-runs.md#the-section-walk), not from the Lua `argv` global.

### Arguments in called chains and tasks

`{{ args }}` and `{{ argv }}` show the argument string of the chain the section runs in, and its parse. On the main walk that is the run's own argument string. A chain started by `call(target, input)` (see [Calling a section](08-jump-and-call.md#calling-a-section)) sees `input` as its argument string in every section of that chain, with `argv` parsed fresh from it (see [Call input and `var`](08-jump-and-call.md#call-input-and-var)):

````markdown
---
name: chains
description: Shows the argument string a called chain sees
runvil: 0
---

# Chains

## Main

```lua
return call('## Sub', 'chain-args')
```

## Sub

Args: {{ args }}

```lua
return prose
```
````

Run with the argument string `run-args`, the run result is:

````text
Args: chain-args
````

A chain started by `call` with no argument string, such as `call('## Inner')`, keeps the argument string of the chain that started it, and so does a fanout arm. With these two sections in place of `## Sub` in the `chains` prompt, `## Inner` still renders `Args: chain-args`, not the run's `run-args`:

````markdown
## Sub

```lua
return call('## Inner')
```

## Inner

Args: {{ args }}

```lua
return prose
```
````

Such a chain also keeps its starter's frozen `argv`, H1 repair included, so `{{ argv }}` there renders the repaired value rather than a fresh parse of the argument string. With the H1 block setting `argv = { query = 'repaired' }` and `## Main` running `call('## Sub')` under the argument string `broken`, `## Sub` sees `argv.query == 'repaired'`. Every section of a task started by [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) with an `input` option sees that string through `{{ args }}` and `{{ argv }}`, as a `call` given an argument string does, so `tasks.spawn('## Child', ...)` with `input = 'child args'` among its options gives `## Child` the argument string `child args`.

### Placeholders ignore the `args` and `argv` globals

`{{ args }}` and `{{ argv }}` read the chain's argument string and its parse, never the `args` and `argv` Lua globals, so reassigning the `args` global leaves `{{ args }}` unchanged:

````markdown
---
name: original
description: Shows that reassigning args leaves the placeholder alone
runvil: 0
---

# Original

## Main

Input: {{ args }}

```lua
args = 'changed'
return prose
```
````

Run with the argument string `original`, the run result is:

````text
Input: original
````

Only placeholder names that are not built-in sources fall through to Lua globals, as [Inserting Lua globals by bare name](#inserting-lua-globals-by-bare-name) describes.

## Inserting the fanout or task `item`

After this section you can insert the member that a fanout arm or a task was started with.

````markdown
---
name: topics
description: Renders each member into the worker section's prose
runvil: 0
---

# Topics

## Parent

```lua
tasks.fanout('### Worker', {'alpha', 'beta'})
return 'done'
```

### Worker

Plain words for {{ item }}.

```lua
return prose
```
````

The arm for `alpha` reads `Plain words for alpha.` from `prose`, the arm for `beta` reads `Plain words for beta.`, and the run result is:

````text
done
````

In a [fanout arm](14-fanout.md#what-an-arm-sees), an `item` placeholder, `{{ item }}`, inserts the arm's current item into the prose, and a string item appears as plain text, so with the item `the angle`, the prose `topic: {{ item }}` renders `topic: the angle`. A [worker section](14-fanout.md#choosing-the-worker-section) can place the arm's member into the text its `lua` fence reads as `prose` and send it to the model with `return models.infer(prose)`, so arms over `alpha` and `beta` with the prose `Reply about {{ item }}.` send `Reply about alpha.` and `Reply about beta.`, and another worker section can write `Ask the model about {{ item }}.` above its Lua block.

### How an item renders

`{{ item }}` renders the seeded item of a fanout arm or of a task by its type. Every type has a rendering and none of them is an error, so a non-string item renders instead of failing:

| Item | Renders as |
|---|---|
| String `plain` | `plain` |
| Number `7` or `2.5` | `7` or `2.5` |
| Boolean | `true` or `false` |
| Array `[7, "x"]` | `[7,"x"]` |
| Keyed table `{ "k": 1 }` | `{"k":1}` |
| JSON null | `null` |

Strings appear verbatim without quotes, so `abc` renders as `abc`. Numbers render in their natural form, integers and decimals alike, so `3` renders as `3` and `1.5` as `1.5`, and `true` renders as `true`. Arrays and keyed tables render whole as compact JSON with no spaces, so `["a", 1]` renders as `["a",1]`. A table member renders the same way:

````markdown
---
name: pairs
description: Renders a table member as compact JSON
runvil: 0
---

# Pairs

## Parent

```lua
tasks.fanout('### Worker', {{7, 'x'}})
return 'done'
```

### Worker

Item: {{ item }}.

```lua
return prose
```
````

The collection holds one member, the table `{7, 'x'}`, so the run's single arm reads `Item: [7,"x"].` from `prose`, and the run result is `done`. The `{{` in the Lua block is Lua source, so it is a table constructor and never a placeholder.

In a keyed collection (see [Collections and member order](14-fanout.md#collections-and-member-order)), each member reaches its arm as a table with `key` and `value` fields, and `{{ item }}` renders the whole pair, so the member `{key='alpha', value=1}` renders as `{"key":"alpha","value":1}`. An item holding JSON null renders as the text `null`, while every other root fails on a null value with a message of the form `missing {{ {path} }}`. An `item` that is nil because none is seeded still fails, as [No item outside an arm or task](#no-item-outside-an-arm-or-task) shows.

### Where an item is seeded

`{{ item }}` works only where an item is seeded: the first section a fanout arm enters, or the first section of a task started with an `item` option. A spawned chain's first section consumes its seeds, so every later section that chain enters has no item, and neither does any other chain. Each fanout arm is spawned the same way as a task, with the arm's member as its `item`. To seed a task, pass `item` in the options of [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn):

````lua
tasks.spawn('## Child', { item = 'alpha' })
````

In the first section of that task, `{{ item }}` renders `alpha`. The `item` option is converted to JSON and seeded into the chain's first section, where it also becomes the chain's `item` global, and it renders by the same rule as in a fanout arm, because every seeded item uses one rendering rule.

Prose in a library that an arm loads with [`require`](03-blocks-and-prose.md#sharing-code-with-require) can use `{{ item }}` too, and it renders that arm's item, because a section's own prose and the prose of the libraries it loads share one `item`. So when the arm for `'alpha'` runs `require '## house_style'`, whose prose reads `Plain words for {{ item }}.`, the library yields `Plain words for alpha.` A `{{ item }}` placeholder in a subsection's prose, such as `Do work on {{ item }}.` in a `### Worker` body, loads without any parse error.

### Fields of a table item

A dotted path into `item`, `{{ item.key }}`, always fails with substitution kind `NotATable` and the message `item is a string, not a table`, so `{{ item.x }}` with the item `text` fails with a message saying `not a table`. The message says the root is a string whatever type the item really has and whether or not an item is set, because the dotted path is refused before the item's value, or whether one is set, is checked. To show one field of a table member, copy it to a global in the section's Lua and reference that global:

````markdown
### Worker

```lua
title = item.title
```

Title: {{ title }}

```lua
return prose
```
````

The seeded item is the Lua global `item`, and a placeholder rooted at an ordinary global name such as `title` reads that section-local global. As the worker section for a collection of tables with a `title` field, each arm renders `Title: ` followed by its own member's title.

### No item outside an arm or task

````markdown
---
name: no_item
description: Reads an item where none is seeded
runvil: 0
---

# No item

## Main

Ask about {{ item }}.

```lua
return prose
```
````

The run fails with an error containing `{{ item }} is nil`. Reading `{{ item }}` where no item is seeded, such as in a walked section like this one, in a section entered through `call`, or in a later section the same chain enters, fails with substitution kind `NilItem` and the message `{{ item }} is nil (not inside a fanout arm)`. The read of `prose` raises it as an ordinary Lua error, so left uncaught in a walked section it ends the run with [run error kind](04-how-a-prompt-runs.md#the-run-result-and-outcomes) `Lua`.

## Inserting Lua globals by bare name

After this section you can insert a Lua global by writing its name in a placeholder.

````markdown
---
name: globals
description: Renders Lua globals set in an earlier block
runvil: 0
---

# Globals

## Main

```lua
answer = 42
data = { score = 9 }
```

The answer is {{ answer }}; score {{ data.score }}; raw {{ data }}.

```lua
return prose
```
````

The run result is:

````text
The answer is 42; score 9; raw {"score":9}.
````

Any placeholder whose first segment is not one of the built-in sources `args`, `argv`, `item`, `var`, or `sys` reads the section-local Lua global of that name. That is a bare global, a global set without `local`, such as `answer = 42`. `{{ name }}` with no further segments renders the global's whole value, with a table rendered as compact JSON with no spaces, so `{{ data }}` renders `{"score":9}` and the global `row = { a = 1 }` makes `{{ row }}` render `{"a":1}`. This is where bare globals differ from `var` and `sys`, where `{{ var }}` or `{{ sys }}` with no key stays an error.

`{{ name.key }}` indexes into a bare global with the same dotted-path rules `var` and `sys` use, one key per segment, so `{{ data.score }}` inserts one field of the table held in `data`. A longer dotted path walks into nested tables held in the global, so with `row = { a = { b = 2 } }` the prose `cell {{ row.a.b }}` renders `cell 2`. A key that is not there fails with substitution kind `MissingKey`.

### Which values a bare global can hold

A bare global that holds data, meaning a string, number, boolean, or table, is converted to its JSON form when a placeholder names it, and a global holding a function, userdata, or coroutine thread is refused. The one userdata that converts is [`json.null`](05-lua-environment.md#encoding-and-decoding-json-with-json), which becomes JSON null, so after `nothing = json.null` the placeholder `{{ nothing }}` fails with substitution kind `NullValue` and the message `missing {{ nothing }}`, the same failure as a `var` path that lands on JSON null. A `local` variable is not a global, so no placeholder reaches it, and a name held only in a `local` reads as unset. A global that holds nil counts as unset rather than as a value. The five built-in roots `args`, `argv`, `item`, `var`, and `sys` are matched before any global lookup and always win over a Lua global of the same name, so a global with one of those names is never reached by bare name.

### When the global is set

The global a bare name reads can be set in an earlier Lua block of the same section, as in the `globals` prompt, earlier in the block that reads `prose`, or by a library that block loads with `require` before the read. A bare name reads the section's global of that name as it stands at the moment of the `prose` read. The first read of `prose` renders the text, and later reads reuse that rendering.

### Bare names in library prose

A library section that a Lua block loads with `require` renders the bare names in its own prose from the calling section's globals:

````markdown
---
name: style
description: Renders a library's prose with the calling section's globals
runvil: 0
---

# Style

## Main

```lua
style = 'terse'
return require '## house_style'
```

## house_style

Write in a {{ style }} voice for {{ args }}.

```lua
return prose
```
````

Run with the argument string `readers`, the run result is:

````text
Write in a terse voice for readers.
````

`require` hands back the value the library returns, here its rendered prose. A library whose prose reads `Broken {{ style }}.` renders `Broken terse.` for the same calling section.

### Defaults through a `_G` metatable

A bare name with no global of its own resolves through your `_G` metatable (see [Giving `_G` a metatable](05-lua-environment.md#giving-_g-a-metatable)), so an `__index` default renders in prose. When `tone` exists only in the `__index` defaults table, `Tone: {{ tone }}.` renders `Tone: friendly.` The metatable can come from a library: a Lua block that runs `require '## defaults'`, which sets `defaults` and an `__index` metatable on `_G`, and then reads `prose` renders `Tone: {{ tone }}.` as `Tone: friendly.`

### When a bare name fails

A bare name whose global reads as nil at the `prose` read, whether it was never assigned or was assigned nil, fails with substitution kind `UnknownNamespace`, and the message names the global, so the prose `{{ ghost }} here.` read by `return prose` fails with a message containing `unknown namespace or global 'ghost'`.

A bare name whose global cannot become JSON data fails with substitution kind `Serialize` and the message `global '{name}' in {{ {path} }} is not JSON data`. That covers a global holding a function, userdata other than `json.null`, or coroutine thread, and a table that contains one of those or refers to itself:

````markdown
---
name: not_data
description: Names a global that holds a function
runvil: 0
---

# Not data

## Main

```lua
function f() return 1 end
```

Call {{ f }}.

```lua
return prose
```
````

The run fails with an error containing `global 'f' in {{ f }} is not JSON data`. The message does not name the Lua type, so when you see it, check what the global holds. The error text holds only the message, the substitution kind, and the byte offset, in the form `{message} [Serialize at byte {offset}]`.

The section's `messages` list, and each [message record](11-conversations.md#message-records) read from it, are userdata, so a bare name whose global holds one fails under the same rule. After `messages:user('u')`, with `list = messages` and `record = messages[1]`, both `{{ list }}` and `{{ record }}` fail with substitution kind `Serialize`. A global holding `json.null` fails with substitution kind `NullValue` instead, as [Which values a bare global can hold](#which-values-a-bare-global-can-hold) describes. Under a strict `_G` metatable whose `__index` raises an error for unknown names, a bare name with no global fails at the `prose` read with substitution kind `Serialize` and the `is not JSON data` message, not `UnknownNamespace`, because the raised error comes back as a failed lookup.

Both bare-global failures, substitution kinds `UnknownNamespace` and `Serialize`, are raised at the `prose` read inside the section's Lua as ordinary Lua errors that [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch. Uncaught, they fail the run with run error kind `Lua`, as the prose `{{ ghost }} here.` does.

## How values render

After this section you can predict the exact text any resolved value renders as.

````markdown
---
name: render
description: Renders one value of each type
runvil: 0
---

# Render

## Main

```lua
var.kind = 'library'
var.count = 3
var.ratio = 1.5
var.ready = true
var.row = { a = 1 }
var.arr = { 1, 2, 3 }
```

a {{ var.kind }} paper; {{ var.count }}; {{ var.ratio }}; {{ var.ready }}; {{ var.row }}; {{ var.arr }}

```lua
return prose
```
````

The run result is:

````text
a library paper; 3; 1.5; true; {"a":1}; [1,2,3]
````

A placeholder that resolves renders its value by type:

| Value | Renders as | Example |
|---|---|---|
| String | Its text as is, with no quotes | `library` renders `library` |
| Integer | Plain digits | `3` renders `3` |
| Number with a fractional part | Decimal form | `1.5` renders `1.5` |
| Boolean | `true` or `false` | `true` renders `true` |
| Table with keys | Compact JSON, keys sorted | `{ a = 1 }` renders `{"a":1}` |
| Array | Compact JSON | `{ 1, 2, 3 }` renders `[1,2,3]` |
| Empty list | `[]` | `json.array()` renders `[]` |
| Empty table | `{}` | `{}` renders `{}` |

Scalars render as strings, and tables and arrays render as compact JSON: JSON text with no spaces and no pretty printing, with keys in sorted order whatever order the author or the argument string wrote them in. So the argument string `{"query":"papers","n":2}` renders through `{{ argv }}` as `{"n":2,"query":"papers"}`. A placeholder that names a table, an array, or a table-valued global without indexing any further renders the whole value this way, so `{{ var.row }}` renders `{"a":1}` when `var.row = { a = 1 }`, `{{ var.arr }}` renders `[1,2,3]` when `var.arr = { 1, 2, 3 }`, `{{ row }}` renders `{"a":1}` for the global `row = { a = 1 }`, and `raw {{ data }}` renders `raw {"score":9}` for the global `data = { score = 9 }`.

An empty list and an empty table look the same in Lua, so the list marker that [`json.array`](05-lua-environment.md#encoding-and-decoding-json-with-json) adds tells them apart. After `var.items = json.array()`, `{{ var.items }}` renders `[]`, and after `var.items = {}` it renders `{}`. An empty list decoded with `json.decode` carries the same marker, so it renders `[]` too.

An integer renders as its plain digits wherever it comes from, so after `var.n = 2` the rendered prose reads `Second fence 2.`, and the globals `answer = 42` and `score = 9` render `42` and `9`. A number with a fractional part renders in decimal form, and `{{ item }}` uses the same rendering, so an item `1.5` renders `1.5`.

A string renders bare when it is the whole value, but inside a rendered table or array it appears in double quotes as a JSON string. So `var.kind = "library"` makes `a {{ var.kind }} paper` render `a library paper`, while `papers` appears as `"papers"` inside the rendered `{{ argv }}` value `{"n":2,"query":"papers"}`.

Nil never renders as empty text. A missing `var`, `sys`, or `argv` key fails with substitution kind `MissingKey`, a bare global that holds nil fails with substitution kind `UnknownNamespace`, and a path that lands on JSON null fails with substitution kind `NullValue`. Only an `item` holding JSON null renders, as the text `null`.

## When substitution happens

After this section you can control when prose renders and which values it captures.

````markdown
---
name: lazy
description: Sets a value after the prose but before the read
runvil: 0
---

# Lazy

## Main

The word is {{ var.word }}.

```lua
var.word = 'mutated'
return prose
```
````

The run result is:

````text
The word is mutated.
````

Placeholders in the [pending prose](03-blocks-and-prose.md#pending-prose-and-the-prose-global) above a Lua block render lazily, at the first read of `prose`: when the section's Lua first reads the `prose` global, not when the file is loaded and not when the block starts. Here the block sets `var.word` before it reads `prose`, so the placeholder renders `mutated`. At that first read, every placeholder in the prose renders together, in one left-to-right pass, and the block gets the finished string. The render uses the section state as it stands at that moment, so `var` and global assignments made earlier in the same Lua block show up in the rendered text, and `sys` is read as it stands then. A bare global that an earlier Lua block in the same section assigned shows up too: a first block sets `answer = 42`, the prose says `The answer is {{ answer }}`, and a later block's `return prose` yields `The answer is 42`.

### Render once

Every placeholder in a block's `prose` renders at most once, on the first read of `prose` during the run, and the result is kept for every later read in that block. This is called render once. With this section in place of `## Main` in the `lazy` prompt, the assertion passes and the run result is `The word is one.`:

````markdown
## Main

The word is {{ var.word }}.

```lua
var.word = 'one'
local first = prose
var.word = 'two'
assert(prose == first, 'a later read returns the same string')
return prose
```
````

A block can read `prose` any number of times and always gets the same string, rendered against the section's `var`, `sys`, and globals as they stood at the first read. A value assigned only after that read, like `var.word = 'two'` here, never appears in the rendered text.

### Each prose and Lua pair renders on its own

Each prose-and-Lua pair in a section renders once, against its own first read, so a later Lua block's `prose` renders fresh from the prose above that block:

````markdown
---
name: pairs_of_fences
description: Renders two prose blocks against different values
runvil: 0
---

# Pairs of fences

## Main

```lua
var.word = 'one'
```

First: {{ var.word }}.

```lua
var.first = prose
var.word = 'two'
```

Second: {{ var.word }}.

```lua
return var.first .. ' ' .. prose
```
````

The run result is:

````text
First: one. Second: two.
````

The second fence's `prose` is `First: one.`, which `var.first` keeps, and the third fence's `prose` is `Second: two.`

### Errors appear at the read

Because rendering happens at the read, a bad placeholder costs nothing until the `prose` read that renders it. A substitution error is raised where your code touches `prose`, such as at `return prose`, not when the file is parsed and not when the section starts. When `return prose` is a block's only Lua, a placeholder such as `{{ reply }}` with no `reply` global fails right at that read. The prose `Missing: {{ var.missing }}.` with `return prose` fails the run with `missing {{ var.missing }}`, while the same kind of bad placeholder above a block that only does `return 'ok'` yields `ok`. Since the block starts normally, it can also catch the failure: `pcall(function() return prose end)` catches the `{{ var.missing }}` error, and the block can go on to return `caught`.

### Prose that is never read

````markdown
---
name: unread
description: Leaves broken placeholders unread
runvil: 0
---

# Unread

## Main

An unclosed {{ placeholder and a {{ var.missing }} key.

```lua
return 'ok'
```

Trailing {{ var.missing }} commentary.
````

The run result is:

````text
ok
````

Markdown that no Lua block reads through `prose` never renders, so a malformed or unresolvable placeholder in it causes no error, and the prompt still loads and runs. Neither placeholder above the block is evaluated, because the block never reads `prose`. The same holds for prose after a section's last Lua fence, like the trailing line here, and for prose in a section with no Lua fence after it, such as a section `First` that holds only `{{ var.missing }} is never read here.` Unread prose in the H1 body is discarded when the H1 pass ends, even when a placeholder in it would fail or render empty, such as `{{ var.omit }}` after `var.omit = ''`.

### Parsing keeps placeholders

Parsing keeps every placeholder exactly as written, and nothing renders until the run reads `prose`. A section whose own text ends with `{{ x }}` parses with no substitution error for the unknown `x`. Even when a Lua block before the prose sets `var.before`, the parsed prose is still literally `Ask about {{ var.before }}.`, and during the run a placeholder resolves against the values as set just before the block's read. A placeholder in H1 prose stays verbatim in the parsed prompt too, as in `Plan {{ args }}.`, and so does `Write about {{ var.subject }}.` in a section.

### Prose in the H1 body

````markdown
---
name: outline
description: Renders H1 body prose from values the H1 pass set
runvil: 0
---

# Outline

```lua
var.topic = args
```

Outline {{ var.topic }}.

```lua
var.outline = prose
```

## Main

```lua
return var.outline
```
````

Run with the argument string `rivers`, the run result is:

````text
Outline rivers.
````

Prose in the H1 body, the blocks under the prompt's title, is substituted the same way as any section's prose when a block in the H1 body reads `prose`, and its placeholders render from the `var` values the H1 blocks have written so far. An H1 block can also send its own pending prose to a model, as `var.h1 = models.infer(prose)` does.

### Library prose under `require`

A library's prose renders when the library's Lua block reads `prose` while `require` loads it, using the `var` values and globals the calling section set before the `require` call. A Lua block that runs `var.x = 'seven'` and then `local seen = require '## lib'` sees `seven` in the library's rendered prose, and the `style` prompt in [Inserting Lua globals by bare name](#inserting-lua-globals-by-bare-name) works the same way with a global.

A `require` leaves the calling section's `prose` alone. Text the calling block already read stays the same, and prose it has not read yet still renders from its own placeholders at its first read, even after a library that raised an error. A block that reads `prose` as `caller saw one`, requires a library whose prose renders `lib text`, and reads `prose` again joins the three as `caller saw one|lib text|caller saw one`, and after `pcall(require, '## bad')` fails, the block's `prose` still reads `caller text`.

## Escapes and literal braces

After this section you can write literal `{{`, `}}`, and backslashes in prose.

````markdown
---
name: escapes
description: Writes literal braces and a backslash
runvil: 0
---

# Escapes

## Main

literal \{{ args }} here, close \}} brace, back \\ slash

```lua
return prose
```
````

Run with the argument string `docs`, the run result is:

````text
literal {{ args }} here, close }} brace, back \ slash
````

An escape is a backslash that makes the character after it literal, and three escapes cover what prose needs:

| Write | Renders |
|---|---|
| `\{{` | `{{` |
| `\}}` | `}}` |
| `\\` | `\` |

`\{{` emits a literal `{{` without starting a placeholder, `\}}` emits a literal `}}`, and `\\` emits a single backslash. Only the two characters `{{` open a placeholder, so a single `{` passes through as literal text, and so does a `}}` that closes no open placeholder: the prose `Set {x} and close }} here.` comes back exactly as written. That is why the `}}` after `\{{ args` above needs no escape.

A backslash before a single `{` or `}` is consumed too, so `\{` emits `{` and `\}` emits `}`, with no backslash left in the rendered text. A backslash before any other character, or a backslash at the very end of the prose, stays as an ordinary backslash, so Windows paths such as `C:\temp\new`, text such as `\n`, and regex text such as `\d+` need no doubling, and `a\b` renders `a\b`. Where the rendered text needs two backslashes in a row, write `\\\\`, because each `\\` pair becomes one backslash.

### Placeholders inside code samples and comments

Substitution works on the raw Markdown text and does no Markdown parsing, so placeholders render anywhere in the pending prose, including inline code, code blocks that are not `lua` fences, and HTML comments. An unescaped `{{` in a code sample is a live placeholder. Fences such as a `json` fence, or a fence whose info string is `text lua`, stay in the section prose exactly as written, and the comment `<!-- written for {{ args }} -->` run with the argument string `docs` renders `<!-- written for docs -->`.

To show a placeholder literally, even inside inline code, escape the opening as `\{{`. The `}}` after it then closes nothing and passes through with no escape needed:

````markdown
Insert a value with `\{{ var.name }}`.
````

This prose renders as:

````text
Insert a value with `{{ var.name }}`.
````

### Escapes next to live placeholders

Escapes compose, and each one resolves on its own, so an escaped `\{{` can sit directly before a live placeholder that still resolves. With the argument string `Acme Corp`, `\{{x}}{{ args }}` renders `{{x}}Acme Corp`: the first delimiter is escaped, and the second is live and resolved. To put a literal backslash directly before a live placeholder, write `\\` there, as in `C:\\{{ var.dir }}`, which renders `C:\` followed by the value of `var.dir`. A single backslash before `{{` escapes it instead, so that placeholder is not resolved.

### Single pass

Substitution is a single pass: text that a placeholder inserts appears verbatim and is never scanned for placeholders again, even when it looks like `{{ ... }}`, which makes such a value safe to insert:

````markdown
---
name: payload
description: Inserts a value that looks like a placeholder
runvil: 0
---

# Payload

## Main

```lua
var.payload = '{{ args }}'
```

value: {{ var.payload }}

```lua
return prose
```
````

Run with the argument string `SECRET`, the run result is:

````text
value: {{ args }}
````

The inserted text is verbatim and never scanned again, so the result never becomes `value: SECRET`. Prose outside placeholders and escapes is copied unchanged, multibyte UTF-8 characters included, so prose with no placeholders and no escapes, such as `plain text`, comes back from the `prose` read exactly as written.

## Substitution error kinds

After this section you can tell each substitution kind by the placeholder that raises it and the message it gives.

````markdown
---
name: unclosed
description: Leaves a placeholder open
runvil: 0
---

# Unclosed

## Main

open {{ args

```lua
return prose
```
````

The read of `prose` fails with substitution kind `Unclosed`, and the run's error message contains `unclosed '{{' in prose`.

A substitution kind classifies a substitution failure, and every failure has exactly one of ten. The ten form a stable classification:

| Substitution kind | Raised by | Message |
|---|---|---|
| `Unclosed` | A `{{` with no `}}` anywhere later in the prose, as in `open {{ args` | `unclosed '{{' in prose` |
| `EmptySegment` | An empty or space-padded segment, as in `{{ var. }}` or `{{ }}` | `empty or padded path segment in {{ {path} }}` |
| `BadPath` | `{{ var }}` or `{{ sys }}` with no key | `bad path: {{ var }}` or `bad path: {{ sys }}` |
| `UnknownNamespace` | A first segment that is neither a root nor a set global, as in `{{ ghost }}` | `unknown namespace or global '{name}' in {{ {path} }}` |
| `NotATable` | A dotted path after `args` or `item`, as in `{{ args.x }}` | `{namespace} is a string, not a table` |
| `MissingKey` | An absent key under `var`, `sys`, `argv`, or a global, as in `{{ sys.bogus }}` | `missing {{ {path} }}` |
| `NullValue` | A path that lands on JSON null, under every root except `item` | `missing {{ {path} }}` |
| `NilItem` | `{{ item }}` where no item is seeded | `{{ item }} is nil (not inside a fanout arm)` |
| `NilArgv` | Any `argv` placeholder while `argv` is nil | `{{ {path} }} is nil (the args string is not JSON)` |
| `Serialize` | A global that is not JSON data, and `{{ prose }}` | `global '{name}' in {{ {path} }} is not JSON data` |

In the messages, `{path}` stands for the placeholder's path, `{name}` for the global's name, and `{namespace}` for the root you wrote.

### Unclosed and EmptySegment

Substitution kind `Unclosed` comes from a `{{` with no `}}` anywhere later in the prose, and the error points at that opening `{{`. Substitution kind `EmptySegment` comes from a path with a leading or trailing dot, two dots in a row, nothing between the braces, or a space next to a dot, as in `{{ var. }}`, `{{ var..x }}`, `{{ var. .x }}`, `{{ var.x. }}`, and `{{ var. .x .y }}`. Its check runs before any lookup, so a space-padded path such as `{{ var. x }}` fails even when a key with that exact name exists. An empty placeholder, `{{ }}`, fails with the message `empty or padded path segment in {{  }}`, with two spaces between the braces.

### BadPath and UnknownNamespace

Substitution kind `BadPath` comes from `{{ var }}` or `{{ sys }}` with no key, because these two roots always need a key and only bare globals render whole values, and its message names the root you wrote, either `bad path: {{ var }}` or `bad path: {{ sys }}`.

Substitution kind `UnknownNamespace` comes from a path whose first segment is neither a built-in root nor a set Lua global, bare or dotted, and it fails the read of `prose` as a hard error. The message names the missing segment and the path, so the prose `{{ ghost }} here.` fails with `unknown namespace or global 'ghost' in {{ ghost }}`, a placeholder naming an unset global `x` fails with `unknown namespace or global 'x' in {{ x }}`, and the prose `prefix {{ ghost.x }}` fails with a message that names the path `ghost.x`. `{{ reply }}` with no `reply` global fails the read the same way. A misspelled root, a `local` variable, and a nil global all produce it, because any first segment other than `var`, `sys`, `argv`, `args`, or `item` is looked up only among Lua globals, which never include a `local`, and a nil global counts as absent.

### NotATable, MissingKey, and NullValue

Substitution kind `NotATable` comes from a dotted key after `args` or `item`, because both roots are used whole only, so `{{ args.x }}` fails with `args is a string, not a table` and `{{ item.x }}` fails with `item is a string, not a table`.

Substitution kind `MissingKey` comes from a dotted path whose key is absent under `var`, `sys`, `argv`, or a global, such as `{{ sys.bogus }}` with the message `missing {{ sys.bogus }}`. It also comes from a path that continues past a string, number, or boolean, so `{{ var.name.first }}` when `name` is a string fails with `missing {{ var.name.first }}`.

Substitution kind `NullValue` comes from a path under `var`, `sys`, `argv`, or a global that lands on JSON null, such as `{{ var.n }}` when `var` holds `{ "n": null }`, or `{{ argv.note }}` with the argument string `{"note":null}`. Its message reads the same as a missing key, `missing {{ {path} }}`, so only the substitution kind tells the two apart.

### NilItem, NilArgv, and Serialize

Substitution kinds `NilItem` and `NilArgv` come from their roots, as [Inserting the fanout or task `item`](#inserting-the-fanout-or-task-item) and [Inserting arguments with `args` and `argv`](#inserting-arguments-with-args-and-argv) show. Substitution kind `Serialize` comes from a bare global that cannot become JSON data, and also from a `{{ prose }}` placeholder inside prose, which is rejected as recursive:

````markdown
---
name: recursion
description: Names prose inside its own prose
runvil: 0
---

# Recursion

## Main

Recursion: {{ prose }}.

```lua
local ok = pcall(function() return prose end)
return tostring(ok)
```
````

The run result is:

````text
false
````

The read of `prose` fails at the point where Lua reads it, with substitution kind `Serialize` and the message `global 'prose' in {{ prose }} is not JSON data`, because `prose` is not a substitution source. The message names `prose`, so `tostring(err):match('prose')` succeeds on the caught error, and the message does not include the underlying recursion refusal.

## Catching and reading substitution errors

After this section you can catch a substitution failure, read what it says, and know how an uncaught one ends the run.

````markdown
---
name: caught
description: Catches a failed prose read
runvil: 0
---

# Caught

## Main

Missing: {{ var.missing }}.

```lua
local ok, err = pcall(function() return prose end)
if not ok then return 'caught' end
return prose
```
````

The run succeeds, and the run result is:

````text
caught
````

A placeholder that cannot render raises an ordinary Lua error at the moment the section's Lua reads `prose`, and the error's message says what went wrong, most often by quoting the placeholder. Because it is an ordinary Lua error raised at the read, not a separate kind of run failure, you can catch it with [`pcall`](05-lua-environment.md#raising-and-catching-errors). Wrap the read itself: `local ok, err = pcall(function() return prose end)` leaves `ok` false and the [error value](05-lua-environment.md#error-values) in `err`, and the Lua block goes on.

### What the error value holds

With this Lua block in place of the one in the `caught` prompt, the block returns the error value's fields:

````lua
local ok, err = pcall(function() return prose end)
return err.kind .. ' | ' .. err.message
````

The run result is:

````text
lua | missing {{ var.missing }} [MissingKey at byte 9]
````

A caught substitution failure gives an error value whose error kind, its `kind` field, is always `lua`, whatever its substitution kind, so the substitution kind, here `MissingKey`, shows only in the text. The error value's `message` field holds that text, and `tostring(err)` returns the same string, so over the prose `Missing: {{ var.missing }}.`, `ok` is `false`, `err.kind` is `lua`, and `tostring(err)` is `missing {{ var.missing }} [MissingKey at byte 9]`. The error value has no fields beyond `kind` and `message`, so nothing but the text names the failing placeholder, path, or offset.

### The error text

Every substitution failure's text has the form `{message} [{Kind} at byte {offset}]`, such as `missing {{ var.x }} [MissingKey at byte 42]`. That same text is what `tostring(err)` returns under `pcall`, and what the run's failure message contains when the failure goes uncaught.

### The byte offset

The byte offset is the position of the failing placeholder's opening `{{`, counted in bytes from the start of the prose being substituted. In `Missing: {{ var.missing }}.` nine bytes come before the `{{`, so the offset is 9. That prose is the pending prose the Lua block read, trimmed of surrounding blank lines and starting after the nearest preceding heading, `lua` fence, or [thematic break](03-blocks-and-prose.md#thematic-breaks-and-setext-headings), so the offset is not a position in the file:

````markdown
---
name: offset
description: Shows where a byte offset counts from
runvil: 0
---

# Offset

## Main

Draft notes.

---

prefix {{ ghost.x }}

```lua
local ok, err = pcall(function() return prose end)
return tostring(err)
```
````

The run result is:

````text
unknown namespace or global 'ghost' in {{ ghost.x }} [UnknownNamespace at byte 7]
````

The thematic break starts fresh pending prose, so `Draft notes.` is not counted, and the offset is 7, just past `prefix `. A body of `Draft notes.`, a `---` break, and `Ask the question.` before a fence leaves pending prose of exactly `Ask the question.` in the same way.

### How messages quote a path

When a substitution error message quotes the placeholder, it prints the path trimmed and with one space inside each brace pair, whatever spacing you used, so `{{var.x}}` reports as `missing {{ var.x }}`. The message escapes a newline, carriage return, or tab in the path as `\n`, `\r`, and `\t`, and shows any other control character as `\u{xxxx}` in lowercase hex, such as `\u{001b}`. A message shows at most the first 80 characters of the path, followed by `...` when the path is longer. The 80-character cut is taken before escaping, so an escaped preview can run a little longer than 80 characters.

### When a failure goes uncaught

An uncaught substitution failure raised by reading `prose` fails the run with run error kind `Lua`. The run's error message contains the substitution's own text, so it names the failing placeholder, such as `missing {{ var.missing }}` for a Lua block that runs `return prose` over `Missing: {{ var.missing }}.` In the H1 pass, an uncaught substitution failure ends the run with run error kind `RequirementsUnmet` instead of `Lua`:

````markdown
---
name: gate
description: Fails a prose read in the H1 pass
runvil: 0
---

# Gate

Missing: {{ var.missing }}.

```lua
var.text = prose
```

## Main

```lua
return var.text
```
````

The run's message starts with the [requirements notice](04-how-a-prompt-runs.md#the-requirements-notice):

````text
the environment cannot satisfy this prompt:
- the H1 gate failed
````

The substitution failure follows the notice as its cause, after a colon and a space, so the message goes on to contain `missing {{ var.missing }} [MissingKey at byte 9]`. A substitution failure never ends a run under a run error kind of its own: every uncaught one ends as run error kind `Lua`, or as run error kind `RequirementsUnmet` in the H1 pass.
