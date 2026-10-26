# File structure

Every prompt file has the same layout: frontmatter, then one H1 title, then an optional H1 body, then sections. This chapter gives the exact rules the parser enforces on each of those parts, from the delimiter lines at the top of the file down to the headings that name each section. Knowing them lets you write a file that parses the first time, and read any parse error as a pointer to the one rule your file broke.

## The shape of a prompt file

After this section you can lay out a complete prompt file and name each of its regions.

````markdown
---
name: h1_only
description: Returns one word from the H1 body.
runvil: 0
---

# H1 Only

```lua
return "hello"
```
````

Running this prompt ends with this run result:

````text
hello
````

This prompt runs with only a title and one `lua` fence (see [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks)) under it. The YAML between the two `---` lines is the frontmatter (see [A Markdown file that is a program](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program)), and it holds the three keys a prompt needs to run: `name`, `description`, and `runvil: 0`. The line `# H1 Only` is the H1 title. The `lua` fence under it makes up the H1 body, and here the H1 body does all the work. The file has no `##` headings, so the prompt parses with zero sections, which is allowed: `##` sections are optional.

### The four regions in order

The next prompt adds prose to the H1 body and adds one section:

````markdown
---
name: layout
description: Shows every region of a prompt file.
runvil: 0
---

# Layout

This prompt returns one fixed line.

## Answer

Answer with the fixed line.

```lua
return 'laid out'
```
````

````text
laid out
````

A prompt file is one Markdown file, and its parts always come in this fixed order:

1. Frontmatter: YAML between two `---` lines at the very top of the file.
2. The H1 title: exactly one `#` heading, here `# Layout`.
3. The H1 body: optional content under the title, up to the first section heading. Here it is the line `This prompt returns one fixed line.`
4. The sections: the headings after the H1 at level `##` or deeper, each with the content under it. Here there is one, `## Answer`, which holds a prose line and a `lua` fence.

Prose directly under the title is read as the H1 body, and the first `##` heading after it starts the first section. A section such as `## Answer` can hold prose and `lua` fences side by side, and [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks) teaches what each one does. Running this prompt returns `laid out`.

A section can hold more than two blocks. A `## Transform` section can hold a `lua` fence, then the prose line `Ask about {{ var.before }}.`, then a second `lua` fence, and the file parses with all three in that order.

### Sections in file order

A prompt body has one H1 title followed by `##` section headings, and each section can hold its own `lua` fence:

````markdown
---
name: steps
description: d
runvil: 0
---

# Title

## First

```lua
local step = 'first'
```

## Second

```lua
return 'second'
```
````

````text
second
````

The walk visits sections in file order (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). `## First` runs, returns nothing, and falls through, so `## Second` starts after it, and its `return` gives the run result. A prompt can have as many sections as it needs: a prompt titled `# Tasks` can hold `## Spawner`, `## Sibling`, and `## Child`, where `## Sibling` starts after `## Spawner` falls through, and a prompt titled `# ModelTasks` can hold `## Only`, `## Child`, and `## Leaf`, each with its own `lua` fence.

After the frontmatter, every prompt has one H1 title line, such as `# Calls`, `# Limits`, `# Greeter`, `# Test prompt`, or `# Title`, followed by section headings such as `## Main`, `## First`, `## Second`, `## Greet`, or `## Only`.

Sections can also nest deeper than `##`. In a prompt titled `# Fanout`, a `### Worker` heading under `## Parent` starts a section inside `Parent`, and that layout parses and runs with only `name`, `description`, and `runvil: 0` in the frontmatter. [Section headings and nesting](#section-headings-and-nesting) gives the nesting rules.

### What the frontmatter holds

You write the frontmatter in YAML at the very top of the file. The first line of the file must be `---`, and spaces before or after the dashes on that line are ignored. A file that starts any other way is rejected with the message `file must begin with a --- frontmatter delimiter`, which [Frontmatter delimiters, byte-order mark, and line endings](#frontmatter-delimiters-byte-order-mark-and-line-endings) covers with the other delimiter rules.

The frontmatter declares the prompt's files with the optional `input` and `output` keys, its Packs with the `uses` list, its arguments with `args`, and its model roles with `models`. None of these is needed for a prompt to parse and run. A frontmatter of only `name`, `description`, and `runvil: 0` between the `---` lines is complete:

````markdown
---
name: frontmatter-only
description: d
runvil: 0
---

# Frontmatter Only
````

This prompt leaves out `models`, `args`, `uses`, `input`, and `output`, and it still parses and runs. [Prompts without sections](#prompts-without-sections) explains how a prompt with nothing under its title ends. The `description` value can be a single character such as `d`, and `runvil: 0` is the version value. A frontmatter of `name: greeter`, `description: says hi`, and `runvil: 0` is just as complete.

## Frontmatter delimiters, byte-order mark, and line endings

After this section you can open and close the frontmatter so the parser finds it, whichever editor saved the file.

````markdown
---
name: minimal
description: minimum valid
runvil: 0
---

# Minimal
````

A frontmatter delimiter is a line of three dashes, `---`, that opens or closes the frontmatter. You write the frontmatter as a YAML block between two delimiter lines at the very top of the prompt file, and the Markdown body is everything after the closing `---` line. In the file above, line 1 opens the frontmatter, lines 2 to 4 are YAML, line 5 closes it, and the body starts on line 6.

### The opening line

The opening `---` must be the first line of the file, so nothing can come before it, not even a blank line or a comment. The opening line still counts as the delimiter when it has whitespace before or after the dashes.

Because the opener is file line 1, the YAML keys start on file line 2, and a frontmatter YAML error reports its line in the file. When the fourth line of the file is bad YAML, the error reports line 4, as [When the YAML does not decode](#when-the-yaml-does-not-decode) shows.

### The closing line

The closing `---` must start at column 0. Whitespace after the dashes is allowed, but an indented `---` never closes the frontmatter. The first column-0 `---` line after the opening one ends the frontmatter, so the YAML itself never contains a `---` line at column 0, such as a second YAML document marker.

An indented `---` line inside a YAML block scalar stays part of the value. This frontmatter has a multi-line `description` whose middle line is `---`:

````markdown
---
name: notes
description: |
  first
  ---
  last
runvil: 0
---

# Notes
````

The indented `---` is text inside the `description` value, which reads back as the three lines `first`, `---`, and `last` with a final newline. The frontmatter ends at the column-0 `---` after `runvil: 0`.

### The byte-order mark

A byte-order mark is an invisible character, U+FEFF, that can sit at the very start of a UTF-8 file. You can save a prompt file with or without one: the byte-order mark is dropped before the check for the opening `---` line, so the opener still counts as line 1.

### Line endings

A prompt file can use LF (Unix) or CRLF (Windows) line endings throughout, in the frontmatter and the body alike, because CRLF is read as LF. That holds for every line the parser looks at:

- The delimiter lines match exactly as in an LF file.
- `lua` fence openings and closings are recognized exactly as in an LF file, and the Lua and prose taken from the file carry no stray carriage returns.
- A fence-label error, which has parse error kind `Fence`, names the label as you wrote it, such as `lua extra`, without a carriage return on the end.

### Delimiter errors

A file whose first line is not the `---` delimiter fails to parse with parse error kind `Frontmatter`. That includes an empty file and a file with no frontmatter at all:

````markdown
# T

## S

p
````

````text
file must begin with a --- frontmatter delimiter
````

A frontmatter block with no closing `---` line also fails with parse error kind `Frontmatter`:

````markdown
---
name: unclosed
description: d
runvil: 0

# Title
````

````text
frontmatter was not closed with ---
````

Neither delimiter error names a prompt, because the prompt's name is not known yet when the error is raised.

## Writing YAML in the frontmatter

After this section you can write frontmatter values in either YAML style and read the error the parser gives when the YAML does not decode.

````markdown
---
name: analyst_example
description: Demonstrate frontmatter model roles and models.use for careful model selection.
runvil: 0
uses:
  - web
models:
  analyst:
    min_context: 40000
    description: A model suited for careful analysis
---

# Analyst Example
````

You write the frontmatter as YAML, and the parser decodes that YAML into the set of frontmatter keys a prompt accepts. This prompt writes its nested values in YAML block style: the key sits alone on its line, and more-indented lines under it hold `key: value` map entries or `- ` list items. Here `models:` holds a map two levels deep, with `analyst:` under it and `min_context: 40000` and `description: A model suited for careful analysis` under that, and `uses:` holds the single list item `- web`.

### Flow style

YAML flow style writes a map or a list on one line, with `{ }` for maps and `[ ]` for lists, nested as deep as you need. This prompt declares the same keys and values in flow style:

````markdown
---
name: analyst_example
description: Demonstrate frontmatter model roles and models.use for careful model selection.
runvil: 0
uses: [web]
models: { analyst: { min_context: 40000, description: A model suited for careful analysis } }
---

# Analyst Example
````

Both files decode to the same frontmatter. Flow style suits short declarations, such as `uses: [web]`, `models: { writer: { keywords: [fast] } }`, or `args: { times: { type: integer, optional: true, default: 1 }, name: { type: string } }`.

The two styles mix, so a block-style `models:` key can hold a flow-style entry on its indented line:

````markdown
---
name: hello
description: Say hello
runvil: 0
models:
  writer: {}
---

# Hello World
````

### Plain values need no quotes

You write string values as plain, unquoted YAML. Free text with spaces, hyphens, commas, periods, and dotted names such as `models.use` needs no quotes, both in `description:` and in nested values. The values in this section's examples are all unquoted, and so is `description: Research a person from the open web and return a concise, factual summary.` Every shipped prompt writes its values this way and parses.

You can also put inline code spans written with backticks partway through an unquoted `description` value, such as `` `lua` `` or `` `~~~lua` ``, and the frontmatter still decodes.

### Multi-line values

A `description` value can be a YAML literal block scalar, written with `|` after the key and the text on more-indented lines below it, as in the `notes` prompt under [The closing line](#the-closing-line). A literal block scalar keeps one line per source line, removes the indentation the lines share, and ends with a trailing newline. So `description: |` over the indented lines `first`, `---`, and `last` reads back as those three lines plus a final newline, with the indented `---` kept as part of the value.

### When the YAML does not decode

The frontmatter must be valid YAML for the prompt to parse. In this file, the fourth line is not a YAML entry:

````markdown
---
name: p
description: d
: : :
---

# T

## S

hi
````

The parse fails with parse error kind `Frontmatter` and a message of this form, where `{message}` is the YAML reader's own message:

````text
invalid frontmatter: {message}
````

The Engine reports a decode failure in the same `invalid frontmatter: {message}` form. The error also gives the file line and column of the offending spot, counting the opening `---` as line 1, so this one reports line 4 and a column. In the same way, a bad list item under `uses:` on file line 5, written after `  - ` so that its text starts in column 5, reports line 5, column 5.

The same message and parse error kind cover every way the frontmatter can fail to decode:

- a YAML syntax slip, as above
- a value of the wrong type
- a missing required `name` or `description` key (see [The required `name` and `description` keys](#the-required-name-and-description-keys))
- an unknown key, whose message names the key (see [Optional keys and unknown keys](#optional-keys-and-unknown-keys))
- a `max_tool_iterations` value outside 1 to 1000

## The required `name` and `description` keys

After this section you can name and describe a prompt with the two keys every prompt file must carry.

````markdown
---
name: greeter
description: says hi
runvil: 0
---

# Greeter

```lua
return "hi"
```
````

````text
hi
````

The `name` key is the prompt's identifier, and the `description` key is a free-text summary of what the prompt does. They are the only required frontmatter keys: neither has a default, and a frontmatter missing either one fails to parse.

### The `name` key

Every prompt names itself with the `name:` key. Its value is a string, and it becomes the prompt's name exactly as written, so `name: greeter` gives the name `greeter`, and `name: x` reads back as `x`. The shipped prompts use names such as `name: echo` in the file `echo.md`, `name: greet`, and `name: hello`.

By convention, prompts write `name` as a lowercase snake_case identifier, with words joined by underscores and digits allowed after the first letter, such as `phase_boundaries`, `require_library`, `store_triad`, `analyst_example`, or `research_person`. The parser does not enforce this format. A name can also be kebab-case, such as `live-h1-models-infer`, `needs-web`, `h1-decision`, or `empty-h1`, or a single letter such as `t`.

The `name` value does not have to match the file name. The file `research-person.md` declares `name: research_person`, `analyst-example.md` declares `name: analyst_example`, and `prologue-prose-epilog.md` declares `name: phase_boundaries`.

Every parse error found past the frontmatter reports the prompt's `name`, beside its line and column when the error has them, so you can tell which prompt failed. A duplicate `## S` section heading in a prompt with `name: dup` fails with parse error kind `Structure` and names `dup` (see [Sibling name uniqueness](#sibling-name-uniqueness)). A frontmatter failure, such as malformed YAML, fails with parse error kind `Frontmatter` and reports no name.

### The `description` key

Every prompt declares a `description:` string that summarizes what the prompt does, such as `description: Greet the named input using a Lua-computed value` or `description: Writes a note to the store and reads it back.` The value becomes the prompt's description as free text, kept exactly as written, down to its closing period. Even a one-letter value such as `description: d` is accepted.

By convention, the description fits on one line. The parser also accepts a several-line `description: |` block value (see [Multi-line values](#multi-line-values)), so one line is a convention, not a rule.

The Workshop's Run window shows the description as a read-only `description` row under the prompt's `name` row whenever the description is not empty.

### What a file needs to parse and to run

A file needs only `name` and `description` to parse. A frontmatter holding just those two keys, with no `runvil` key or any other key, passes the frontmatter parse, and the parser moves on to check the body:

````markdown
---
name: x
description: d
---

# T

## S

p
````

Every other key, including `runvil`, `max_tool_iterations`, `input`, `output`, `uses`, `args`, and `models`, takes a default when you leave it out.

A runnable prompt's frontmatter carries three keys: `name`, `description`, and `runvil: 0`. A run proceeds only when `runvil` is `0`. A prompt without `name` or `description` fails at parse time, while a prompt without `runvil` parses but stops at run time with `not a runvil prompt: no runvil version` (see [The `runvil` version key](#the-runvil-version-key)). You add optional keys next to the three when a prompt needs them, such as `uses:`:

````markdown
---
name: needs-web
description: d
runvil: 0
uses:
  - web
---
````

This is the frontmatter of a prompt; its H1 title and the rest of its body follow the closing `---`.

### Missing `name` or `description`

Leaving out `name:` or `description:` fails the frontmatter parse with a missing-field error from the YAML reader, because these are the only two keys with no default:

````markdown
---
name: greeter
runvil: 0
---

# Greeter
````

The parse error kind is `Frontmatter`, and the message has the `invalid frontmatter: {message}` form, where `{message}` names the missing field, here ``missing field `description` ``. A frontmatter without `name` gives ``missing field `name` `` the same way.

## The `runvil` version key

After this section you can declare the language version so the Engine runs your prompt, and recognize the two failures a missing or unsupported version gives.

````markdown
---
name: version-zero
description: d
runvil: 0
---

# Version Zero

## Only

```lua
return "ran"
```
````

````text
ran
````

The `runvil:` key declares which major version of the Engine the file targets. Its value is a non-negative integer, and you write it as a bare integer, `runvil: 0`. A prompt that carries it parses and runs, as this one does.

### What the key marks

The `runvil:` key is what marks a Markdown file as a Runvil prompt, and a file without it is not treated as one. That is a run-time check, not a parse-time one. A file needs only the `name` and `description` keys to parse, but it needs `runvil: 0` to run, so a file without the `runvil:` key parses but never runs, and every prompt that runs declares `runvil: 0`. A prompt that leaves out `runvil:` still parses, and it then has no declared major version.

### The active Engine major

The active Engine major is the one major version this build of the Engine runs, and it is `0`. The parser accepts `runvil: 0`, and the parsed prompt's frontmatter reports the declared major, `0`. A prompt that declares `runvil: 0` passes the version check and runs normally, and it can combine lazy `prose`, `models.infer`, `models.loop` with tool dispatch over tools offered through `tools.offer`, `messages` builders, and synchronous `call` in one run (see [Starting a conversation](11-conversations.md#starting-a-conversation)).

The `runvil:` value must be a non-negative integer that fits in 32 bits, `0` through `4294967295`. Any other value makes the frontmatter fail to parse with parse error kind `Frontmatter`.

### When the version is checked

A file parses whether or not it has a `runvil:` key, and the run checks the declared version when it starts, before any section runs. The run checks the version before it checks the run's store, so a prompt with a missing or unsupported version reports the version failure even when the run also has no usable store.

A prompt whose `runvil` version is missing or unsupported fails before any section runs. None of its sections run, nothing is reported beyond that one failure, and its own `return` never runs.

### A missing version

A file with no `runvil:` key still parses, but it is not treated as a Runvil prompt. Here is the `version-zero` prompt without its `runvil: 0` line:

````markdown
---
name: version-zero
description: d
---

# Version Zero

## Only

```lua
return "ran"
```
````

The run ends on its first step with run error kind `Parse`, as a `Structure` parse error with this message:

````text
not a runvil prompt: no runvil version
````

The error's location names the prompt by its frontmatter `name`, here `version-zero`.

### An unsupported version

A file that declares a major other than `0` still parses, but this build does not support that major, so the run ends on its first step with run error kind `Version`. The version check never falls back to major 0. A prompt runs only when it declares `runvil: 0`, and any other declared major fails the run with an error that names that major:

````markdown
---
name: version-zero
description: d
runvil: 1
---

# Version Zero

## Only

```lua
return "ran"
```
````

````text
unsupported runvil version: 1 (this build supports major 0)
````

The message always has the form `unsupported runvil version: {n} (this build supports major 0)`, where `{n}` is the declared number, so `runvil: 2` or `runvil: 7` gives the same message with its own number. A failure caused by an unsupported `runvil` version is not retryable.

## Optional keys and unknown keys

After this section you can add the optional frontmatter keys a prompt needs, and you know why a misspelled key fails the whole file.

````markdown
---
name: x
description: d
---

# T

## S

p
````

The frontmatter recognizes exactly nine top-level keys, and only `name` and `description` are required, so this file parses. You leave out the other seven when a prompt does not need them, and you add them alongside `name`, `description`, and `runvil: 0` when it does.

### The nine keys

This table lists the nine keys in the order this chapter teaches them, with what each one means when you leave it out:

| Key | Required | When you leave it out |
|---|---|---|
| `name` | yes | the parse fails |
| `description` | yes | the parse fails |
| `runvil` | no, but needed to run | no declared version: the file parses, and running it fails with `not a runvil prompt: no runvil version` |
| `max_tool_iterations` | no | the built-in round cap stays in force |
| `input` | no | no declared input file |
| `output` | no | no declared output file |
| `uses` | no | no Packs |
| `args` | no | the default declaration of one optional `string` argument named `prose` |
| `models` | no | no model roles |

Because every key besides `name` and `description` has a default, a frontmatter of just those two parses, and so does a prompt with none of the contract keys. `max_tool_iterations` accepts a value from 1 to 1000, and [Tool rounds and the round cap](11-conversations.md#tool-rounds-and-the-round-cap) covers what it controls. [Input and output file declarations](#input-and-output-file-declarations) teaches `input` and `output`.

### Adding keys a prompt needs

The `hello` prompt under [Flow style](#flow-style) shows the pattern. The shipped `hello` prompt's frontmatter holds only `name`, `description`, `runvil`, and `models`, and leaves out `max_tool_iterations`, `input`, `output`, `uses`, and `args` because it does not need them. Its `models:` key declares a `writer: {}` role next to the usual three keys, and its H1 body then calls `models.default("writer")`.

The optional `input:` and `output:` declarations sit in the same frontmatter block as the other keys, as in a prompt whose frontmatter reads `name: files`, `description: d`, `runvil: 0`, then an `input:` map and an `output:` map.

### The contract keys

The contract keys are `uses`, `args`, and `models` (see [Putting the parts together](01-what-a-prompt-is.md#putting-the-parts-together)). Together they declare what a prompt needs from its environment:

- `uses` lists the Packs the prompt requires, and each one listed is required.
- `args` types the arguments the prompt takes.
- `models` declares the model roles (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)) that concrete models are bound to when the run is prepared.

Checking these declarations against the caller's environment happens when the run is prepared, not at parse time. For example, a `checker` role that asks for the `no-thinking` keyword is reported as an unmet requirement when the supplied model falls short of it.

All three contract keys can sit together in one frontmatter block alongside `name`, `description`, and `runvil: 0`, next to `input:` and `output:`, and each reads back as declared:

````markdown
---
name: greeter
description: says hi
runvil: 0
input: { path: names.md, description: The names to greet }
output: { path: note.md, description: The note the greeter leaves }
uses: [web]
args: { times: { type: integer, optional: true, default: 1 }, name: { type: string } }
models: { writer: { keywords: [fast] } }
---

# Greeter
````

A frontmatter with `uses: [web, user-input]`, an `args:` map holding `use_mcp`, and a `models:` map holding `analyst` and `triage` parses and reads back the same way.

### Unknown keys

You spell frontmatter keys exactly as the language defines them. An unknown frontmatter key is any top-level key other than the nine, and it is never ignored: it fails the whole file with parse error kind `Frontmatter`, because any other key is treated as a prompt authoring error. The `greeter` prompt above parses with `uses: [web]`, but the same file with that key misspelled fails:

````markdown
---
name: greeter
description: says hi
runvil: 0
usse: [web]
---

# Greeter
````

The message has the `invalid frontmatter: {message}` form, and `{message}` names the offending key, here `usse`.

Unknown keys are rejected inside nested declarations too. An `input:` or `output:` declaration accepts only `path` and `description`, an arg entry and a model role each accept only their own fixed set of keys, and any other key inside them fails the parse with parse error kind `Frontmatter`.

## Input and output file declarations

After this section you can declare the file a prompt reads when it starts and the file it leaves behind when it finishes.

````markdown
---
name: files
description: d
runvil: 0
input: { path: paper.md, description: The input paper }
output: { path: report.md, description: The output report }
---

# Files

## Main

```lua
store.write('report.md', 'seen: ' .. store.read('paper.md'))
```
````

Launched with the input text `# Paper`, this prompt leaves `report.md` in the store holding:

````text
seen: # Paper
````

A prompt can declare an input file and an output file. The Harness puts the declared input file in place in the store before the run, and it reads the declared output file from the store after the run (see [Writing and reading files](09-the-store.md#writing-and-reading-files)). Here the launch text `# Paper` becomes the file `paper.md`, the section reads it and writes `report.md`, and the caller gets `seen: # Paper` as the output.

### File declarations

You declare the two files in the frontmatter under the `input:` and `output:` keys. Each one holds a file declaration, which is a map with exactly two required strings:

- `path` is the file's store path, such as `paper.md` or `report.md`, which the prompt reads or writes.
- `description` documents the file's purpose in plain words, and the Workshop's Run window shows it beside the path.

A file declaration accepts no keys besides `path` and `description`, and any other key fails the frontmatter parse. Both keys are required, so leaving out either one fails the frontmatter parse until you add the missing key. The parser keeps both values exactly as written: `paper.md` reads back as `paper.md`, and `The input paper` reads back as `The input paper`, inner spaces included.

`input:` and `output:` are both optional, and each holds a single file declaration rather than a list. A prompt whose frontmatter holds only `name`, `description`, and `runvil: 0` parses with neither declared, so both read as absent.

You can write a declaration as a one-line YAML flow map in `{ key: value }` form with unquoted values, as in `input: { path: paper.md, description: The input paper }`, or in block style:

````markdown
---
name: files
description: d
runvil: 0
input:
  path: paper.md
  description: The paper
output:
  path: report.md
  description: The report
---
````

This is the frontmatter of the `files` prompt in block style; the rest of the file is unchanged.

### The input file

The input file is the one store file a prompt expects to find when it starts, declared with `input:`. You point the prompt at it by giving `input:` a `path`, and you explain it in plain words with a `description`, which you can write unquoted with spaces inside a flow map. So `input: { path: names.md, description: The names to greet }` reads back with the path `names.md` and the description `The names to greet`. A prompt without `input:` has no input file.

The input file is an ordinary store file that is in place before the first section runs, so a section reads it with `store.read` at the declared path, as in `local paper = store.read('paper.md')`. The declared path is a logical store path: when the Host mounts the store at a directory such as `/store`, the staged input appears where `store.read('paper.md')` looks, and nothing is written at `/paper.md`.

The input file can reach the store in two ways:

- When the run is launched with input text, the Harness writes that text byte for byte into the store as the file at the declared `input:` path before the run starts. The text overwrites any file the store already holds at that path.
- When the launch supplies no text, the prompt still prepares and runs if the store already holds a file at the declared path, because that file satisfies the declaration. A store seeded with `paper.md` holding `the paper body` lets `store.read('paper.md')` read that text.

A prompt that declares neither `input:` nor `output:` prepares normally when no input text is supplied.

### The output file

The output file is the one store file a prompt leaves behind when it finishes, declared with `output:`. You give it a store `path` and a `description` that says what the file is, as in `output: { path: report.md, description: The output report }` or `output: { path: note.md, description: The note the greeter leaves }`. A prompt without `output:` has no output file, so the caller gets no output path and has no output file to read.

A prompt produces its declared output file by writing it with `store.write` at the declared path, because nothing else writes it. With `paper.md` holding `the paper body`, the call `store.write('report.md', 'report on: ' .. paper)` after `local paper = store.read('paper.md')` leaves `report.md` holding `report on: the paper body`. The Harness collects the file from the store after the run ends, and whatever the prompt wrote at the declared path is the file the caller reads as the run's output once the run completes.

The Harness's run report carries the output file's text only for a completed run. The report for a failed or cancelled run skips the output file.

### When the files cannot be put in place

The parser accepts a declared `path` as written. The store's path rules (see [Store paths](09-the-store.md#store-paths)) apply only when the Harness stages the input or reads the output, so a path the store refuses shows up when the run starts, not at parse time.

Each input problem refuses the run before any section runs, and the run ends as failed with kind `Input`. Every such refusal carries the message `the prompt's declared input file cannot be put in place`, and its cause tells the problems apart:

| Situation | What the cause says |
|---|---|
| The launch supplies input text, but the prompt declares no `input:` file for it to go to | that the text has no declared file to go to |
| The prompt declares an `input:` file that is neither supplied at launch nor already in the store | the missing path, such as `paper.md` |
| The store refuses the write that stages the input, for example because the store requires approval for the write | the declared path, with the store's own error (see [When a store call fails](09-the-store.md#when-a-store-call-fails)) |
| The store refuses the declared `input:` path under its path rules | the declared path, with the store's own error |

A missing output file is not a run failure. A prompt that declares an `output:` file but never writes it still runs to success, and the run report then carries this error in place of the output text, naming the declared path:

````text
the run completed without writing its declared output file `{path}`
````

## The name grammar for role labels, arg names, and adhoc tool names

After this section you can choose names for role labels, arg names, and adhoc tool names that both the parser and the Engine accept.

````markdown
---
name: contract_names
description: d
runvil: 0
args: { use_mcp: { type: boolean }, max_items: { type: integer, optional: true, default: 1 } }
models: { analyst: {}, triage: {}, fast-model: {} }
---

# Contract Names
````

This frontmatter parses. The keys under `models:` are role labels, and the keys under `args:` are arg names. A role label is the key that names a model role, such as `analyst`. An arg name is the key that names one argument, and the argument reaches Lua as a field of `argv`. Both follow the same name grammar as an adhoc tool name, which is the `name` Lua code gives an adhoc tool (see [Effects and Harness work](01-what-a-prompt-is.md#effects-and-harness-work)) in the attrs table of `tools.adhoc` (see [Adhoc tools with `tools.adhoc`](12-tools.md#adhoc-tools-with-toolsadhoc)).

### The prompt-local alias grammar

Every role label, arg name, and adhoc tool name follows the prompt-local alias grammar:

````text
[A-Za-z][A-Za-z0-9_-]{0,63}
````

In words, a name is 1 to 64 ASCII characters: a letter first, then letters, digits, `_`, or `-`. Names such as `search`, `fetch`, `writer`, `analyst`, `use_mcp`, `limit`, `query`, `fast-model`, and `max_items` are all valid. Rule by rule:

- Length: a name is 1 to 64 characters long, so it always has at least one character. A name of exactly 64 characters is valid, and 64 is the limit.
- First character: an ASCII letter of either case, `A` to `Z` or `a` to `z`.
- Later characters: ASCII letters, ASCII digits, `_`, and `-` only. Every character of a name is ASCII, so its letters are `A` to `Z` and `a` to `z` in every position.
- Digits, underscores, and hyphens: these appear only after the first letter, as in `search1`, `web-fetch2`, `web_search`, or `web-search`.
- Joining words: you join words with `_` or `-`, as in `web_fetch` or `web-fetch`, because spaces and dots are not name characters.
- No slash: a name is a single plain name with no `/`, which keeps an adhoc tool name distinct from a tool id such as `web/fetch`, and from the adhoc tool's own id, `adhoc/<name lowercased>` (see [Pack ids, tool ids, and wire names](12-tools.md#pack-ids-tool-ids-and-wire-names)).

Within those rules, any style works. Names in uppercase, lowercase, mixed case, snake_case, or kebab-case, with trailing digits, or of a single letter are all valid, such as `a`, `A`, `search`, `web_fetch`, `web-fetch2`, and a name of 64 `z` characters. A valid name keeps its exact text as written, including letter case.

Put the other way, a name fails the grammar when it is empty or longer than 64 characters, when it starts with anything other than an ASCII letter, or when it contains a space, a dot, a slash, or a non-ASCII character.

### Where the grammar applies

At parse time, the grammar applies to every key under `models:` and `args:`, such as the role labels `analyst` and `triage` and the arg name `use_mcp` above. Role labels and arg names such as `writer`, `checker`, `times`, and `name` are valid too, with `times` and `name` under `args:` and `writer` and `checker` under `models:`.

Shipped prompts name each model role with a plain lowercase label, such as `writer` or `researcher`, and Lua code refers to the role by that same string, as in `models.default("writer")` and `models.default("researcher")`.

At run time, the Engine enforces the same grammar, with the same length, first-character, and later-character rules. Role labels and adhoc tool names share that one rule, so a name that is valid as one is valid as the other, and `models.use` checks its label with the same rule that `tools.adhoc` applies to its `name`. The grammar applies when Lua passes a name to these calls (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)):

- `tools.adhoc`, for its `name`
- `models.use`, for its label
- `models.default`, for its label
- `models.get`, only for a name that matches none of the prompt's model roles, since a declared role's handle comes back first

When the Host allows raw model ids, `models.get` skips the grammar for a name that matches no declared role and reads the name as a raw model id instead, the name a gateway model is listed under, because model names can contain `/`, `.`, and `:`. With raw model ids allowed, `models.get('qwen/qwen3-8b')` returns a handle bound to `gateway/qwen/qwen3-8b`, whose `model_id` is `qwen/qwen3-8b`. Without them, `models.get('ghost')` fails because `ghost` is not a bound model role.

### Contract maps and duplicate keys

A contract map is the `models:` map or the `args:` map. The two share one set of key rules, so the grammar check and the uniqueness check apply the same way to both. Each key appears once within its map. Keys are compared exactly, so labels that differ only in case, such as `Fast` and `fast`, are two distinct entries.

A repeated key fails the parse with parse error kind `Frontmatter`, and the message names the key kind and the repeated key, where the key kind `{what}` is `model role label` for `models:` and `arg name` for `args:`:

````text
duplicate {what} `{key}`: contract map keys must be unique
````

The parser checks each key against the grammar first and for uniqueness second, and it checks a key before it reads the key's value. It walks the keys in source order and stops at the first bad one, so:

- A map with several bad keys reports only the first one.
- An entry with both a bad key and a bad value reports the key error.

### Reserved names

A reserved name is a name that is already taken in a section's Lua environment, such as the global `store`, the Lua function `pairs`, or the keyword `end` (see [The sandbox](05-lua-environment.md#the-sandbox)). Role labels and arg names are not checked against reserved names, because neither becomes a Lua global: an arg is a field of `argv`, and Lua code reaches a role through `models.get(label)`. So every reserved name the grammar allows works as a role label, and `prose`, `store`, and `end` work as arg names:

````markdown
---
name: reserved
description: d
runvil: 0
args: { prose: { type: string }, store: { type: string }, end: { type: boolean } }
models: { 'store': {} }
---

# Reserved
````

This frontmatter declares three args, named `prose`, `store`, and `end`, and the prompt holds a role under the name `store`. A reserved name such as `pairs` or `end` works as a role label the same way.

### Parse-time name errors

A key outside the grammar fails the parse with parse error kind `Frontmatter`. The `{message}` of the `invalid frontmatter: {message}` message names the key kind and the offending key, and it shows the grammar:

````text
invalid {what} `{key}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

The key kind `{what}` reads `model role label` for a key under `models:` and `arg name` for a key under `args:`, because `args:` keys go through the same key rules as `models:` keys. So the two details read:

````text
invalid model role label `{key}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
invalid arg name `{key}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

Writing `models:` or `args:` as something other than a map, such as a list or a scalar, fails with parse error kind `Frontmatter` and this detail, where `{found}` is the YAML reader's description of the value you gave and `{what}` is again `model role label` or `arg name`:

````text
invalid type: {found}, expected a map of {what} keys to declarations
````

Each grammar, duplicate, and map-type error from `models:` and `args:` reports the file line and column of the offending entry, such as line 5, column 5.

Every grammar violation, whether an empty name, a name that is too long, a bad first character, or a bad later character, produces the same single message form. The message shows the grammar but not which rule failed, so check the name against every rule in [The prompt-local alias grammar](#the-prompt-local-alias-grammar).

### Run-time name errors

When a role label passed to a Lua call breaks the grammar, the call fails with a `Lua` error. Its message names the rejected alias and the grammar, again in one form for every rule:

````text
invalid alias "{alias}": expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

The rejected alias appears in double quotes with escaping, so a quote, backslash, or control character in the name appears escaped.

When the `name` passed to `tools.adhoc` breaks the grammar, the `tools.adhoc` call fails with a `Lua` error whose message names the rejected name and the grammar, again in one form for every rule:

````text
tools.adhoc: invalid name "{name}": expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

The rejected name appears in double quotes with the same escaping. One way to hit this error is to pass a tool id where a name belongs, since a tool id contains `/`. Lua code can catch the error with `pcall` (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

````lua
local _, err = pcall(tools.adhoc, { name = 'web/search', description = 'x' }, function() end)
````

After this line, `tostring(err)` contains `tools.adhoc: invalid name "web/search"`.

Each adhoc tool name can be defined once in a section. A second `tools.adhoc` call with a name that is already defined fails with this message:

````text
tools.adhoc name "{name}" is already registered
````

So defining the name `gh_search` a second time fails with `tools.adhoc name "gh_search" is already registered`.

An adhoc tool's id is `adhoc/` followed by its name in lowercase, so two names that differ only in letter case would share one id. The second of them fails with this message, where `{id}` is that shared id, `adhoc/` followed by the name in lowercase, and `{other}` is the name that already holds the id:

````text
tools.adhoc name "{name}" has the id {id}, which "{other}" already holds
````

So after `gh_search` is defined, defining `GH_Search` fails with `tools.adhoc name "GH_Search" has the id adhoc/gh_search, which "gh_search" already holds`.

## The H1 title and H1 body

After this section you can write the one H1 title every prompt needs and put prose and Lua directly under it.

````markdown
---
name: title_demo
description: d
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

A prompt has exactly one `#` title line, and the text of that H1 heading becomes the prompt's title (see [A Markdown file that is a program](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program)). Lua in the H1 body runs under the prompt's title, so `sys.section_name` there reads the title, and this prompt returns `Title Demo`.

### The title line

Every prompt file needs an H1 title, written as a single `# Title` line with non-empty text. The title is required, and it comes after the frontmatter and before the sections, usually right after the frontmatter, as in `# Echo`, `# Greet`, `# Hello World`, and `# Analyst Example`. You may leave a blank line between the closing frontmatter `---` and the H1 title.

The title text can be any phrase, including spaces and capitals, such as `# Short Circuit`, `# Only Prose`, or `# Research a Person`. The title keeps its inner spaces and case and loses the whitespace around it, so `# Greeter` gives the title `Greeter`, `# Demo Title` gives `Demo Title`, and `# Phase Boundaries` gives `Phase Boundaries`. Inline code or other markup in the H1 keeps only its text in the title, and the markup delimiters drop out.

The H1 title is free text, separate from the frontmatter `name` key, and it need not match it. `name: hello` can sit above `# Hello World`, and `name: greeter` with `# Greeter` gives the name `greeter` and the title `Greeter`.

A `##` section can have the same text as the H1 title, as in `# Self` with a `## Self` section, and the prompt still parses and runs. When that `## Self` section calls `jump('## Self')`, the call fails with `not found` at heading resolution (see [Resolving heading references](08-jump-and-call.md#resolving-heading-references)), which has nothing to do with the shared text.

A `#` heading inside a block quote or a list item is not an H1. It never becomes the title and does not count toward the one-H1 rule, and quoted or listed headings such as `> ## Quoted` and `- ## Listed` stay in the surrounding prose.

Plain or introductory prose never stands in for the H1 title, so a file with prose but no H1 is still invalid, and every section has an H1 above it. However many keys the frontmatter declares, the file still needs its H1 title, and a file with no H1 fails with `prompt requires an H1 title` whatever the frontmatter holds. The prompt's work happens in the H1 body or the sections, never in the frontmatter.

### The H1 body

The H1 body is everything between the H1 heading and the first section heading. It is optional, so the first `##` section can start right after the H1, and the title needs no Lua under it.

The H1 body holds any sequence of prose and `lua` fences, kept in source order, or nothing at all, and an empty H1 body has no blocks. You can write prose directly under the title, before the first `##` section, such as `The helper library is available to each section that requires it.` under `# Require Library`.

Prose under the H1 is an H1 prose block, and a later `lua` fence in the H1 body can read it as `prose`, as in `return models.infer(prose)` (see [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks)). H1 prose that no Lua reads has no effect:

````markdown
---
name: empty-h1
description: d
runvil: 0
---

# Empty H1

```lua
var.omit = ''
```

{{ var.omit }}

## Result

```lua
return 'ok'
```
````

````text
ok
````

The H1 fence runs before the prose, and no Lua after it reads the H1 prose `{{ var.omit }}`, so the prose stays inert, and the run returns `ok` without needing a model.

Nothing in the H1 body becomes a section, because the H1 body ends where the first section heading begins, and sections come only from headings at level 2 or deeper after the H1.

### Prompts without sections

A prompt needs no `##` sections. An H1 title with only `lua` blocks under it parses and runs, and the H1 body does all the work, as in the `h1_only` prompt at the start of this chapter, which runs to the result `hello`. A prompt has no minimum number of sections: a prompt with an H1 title and no `##` sections is valid, parses with an empty section list, and runs with or without a Lua `return`.

- When its H1 Lua returns a value, that value is the run result, such as `hello`.
- When its H1 Lua returns nothing, the run ends with the generic result `done`.
- When it has no H1 blocks at all, as in the `frontmatter-only` prompt under [What the frontmatter holds](#what-the-frontmatter-holds), the run ends with the same generic completion.

A prompt with an H1 title and prose but no `##` sections is valid too. A file of just `# Only a title` and `Text.` under its frontmatter is a complete prompt:

````markdown
---
name: only-a-title
description: d
runvil: 0
---

# Only a title

Text.
````

### The preface

The preface is everything between the frontmatter and the H1. It has no meaning to the prompt: free-note prose or fenced code blocks placed there are ignored, so editing notes above the title change nothing:

````markdown
---
name: with-notes
description: d
runvil: 0
---

Editing notes for this prompt live here and change nothing.

# With Notes

```lua
return 'ok'
```
````

````text
ok
````

A heading placed in the preface never becomes a section, because sections come only from headings at level 2 or deeper that follow the H1. Two rules still reach into the preface:

- An H1 in the preface counts toward the one-H1 rule, so a file with an H1 in its notes above the title fails with `prompt must contain exactly one H1 title`.
- The strict `lua` fence rule applies there too, so a fence in the preface whose label starts like `lua` must be an exact `lua` fence, or the parse fails with parse error kind `Fence`.

A `lua` fence shown inside a longer fence stays prose, so the H1 body can show a Lua fence as an example. A four-backtick `markdown` fence in the H1 body can contain a three-backtick `lua extra` opener, and that inner line is never read as a fence of its own.

### H1 errors

A prompt with no H1 title, or with more than one H1 title, fails to parse with parse error kind `Structure`, and so does a prompt whose H1 title is empty:

| Problem | Message |
|---|---|
| No H1 | `prompt requires an H1 title` |
| More than one H1 | `prompt must contain exactly one H1 title, found {count} at lines {lines}` |
| An empty or whitespace-only H1 | `prompt H1 title must not be empty` |

These three errors name no section and give no line or column. Only the more-than-one message lists line numbers, inside its text.

This file goes straight from the frontmatter to prose and a section, with no H1:

````markdown
---
name: missing_h1
description: Invalid because no H1 title exists
runvil: 0
---

Introductory prose cannot replace the required title.

## Run
````

````text
prompt requires an H1 title
````

A file with more than one H1 anywhere in the body, even in the preface above the title, fails with a message that gives the H1 count and the file line of every H1, in the form `found {count} at lines {lines}`, where `{lines}` lists the lines separated by commas:

````markdown
---
name: t
description: d
runvil: 0
---
# T

## S

p

# U

q

# V
````

````text
prompt must contain exactly one H1 title, found 3 at lines 6, 12, 16
````

The H1 title must have visible text. An H1 that is empty or only whitespace fails, and so does a bare `#` line with no title text:

````markdown
---
name: t
description: d
runvil: 0
---

#

## S
````

````text
prompt H1 title must not be empty
````

## Section headings and nesting

After this section you can nest sections with heading levels H2 through H6 and tell which section owns each line of the file.

````markdown
---
name: nesting
description: d
runvil: 0
---

# Nesting

## Prepare

Gather the subject.

### Author note

Keep the subject short.

## Finish
````

This file has two top-level sections, `Prepare` and `Finish`. `### Author note` is a child of `Prepare`, not a third top-level section, so it belongs to `Prepare`. `Prepare`'s own prose is only `Gather the subject.`, and the line under `### Author note` belongs to the child.

### Heading levels

A heading level is the number of `#` markers that start a heading. You mark sections with Markdown headings from H2 (`##`) through H6 (`######`), the deepest Markdown heading level, and each heading's level decides where its section nests. The H1 (`#`) title is never a section. Headings after the H1 title at level 2 or deeper become the prompt's sections, and each section heading, together with the prose and Lua under it, forms one section.

Each section has a name taken from its heading, a level, its own prose and Lua, and its child sections.

The top-level sections are the H2 headings, kept in file order. A `##` heading under the H1 title starts a top-level section at level 2, and the text of the heading is the section's name, as with `## Main` one level below `# Echo`, or `## Analyze`, `## Greet`, and `## Research` in the shipped prompts. So `## First` followed by `## Second` gives two top-level sections, First then Second.

No particular section name is required. The first H2 is the entry section whatever its name, so `## Zebra` followed by `## Main` makes Zebra the entry (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)).

### Child sections

Sections nest by heading level, so a deeper heading belongs to the section above it rather than starting another top-level section. A child section is a section whose heading sits one level below its parent's heading: an H3 after an H2 becomes that H2's child, and an H4 after that H3 becomes the H3's child. A `###` under a `##` is a level-3 child rather than a new top-level section, a `####` under that `###` is a level-4 grandchild, and each one has its own level and prose and can hold its own `lua` block. A child is named by the text of its heading without the `#` markers, so `### Helpers` after `## A` is A's first child, named Helpers. Sibling sections are sections that share a parent.

````markdown
---
name: tree
description: d
runvil: 0
---

# Tree

## A

### X

#### P

#### Q

### Y

## B
````

Here `## A` holds `### X` and `### Y`, `### X` holds `#### P` and `#### Q`, and `## B` is a sibling of `## A`. Lua names section B with the heading reference `'## B'` (see [Heading references](#heading-references)).

The rule behind this shape is that a heading deeper than the current section becomes its child, and a heading at the same or a shallower level ends that section and belongs to an ancestor. So each H2 after the H1 starts a new top-level section, including an H2 that follows a nested H3 child. Top-level sections keep their file order without counting nested `###` sections, so `## First`, `### Child`, `## Second` gives two top-level sections, First and Second.

You can place several H3 child sections side by side under one H2, as `## Research` holds `### Worker` and `### Topics`.

Sections nest five levels deep under the `#` title, from `##` down to `######`, and each heading's parent is the nearest heading above it that is exactly one level shallower:

````markdown
---
name: deep
description: d
runvil: 0
---

# Deep

## Grand

### Parent

#### Self

##### Child

###### Grandchild
````

### Section names

A section name is the text of a section's heading without the leading `#` markers. It may contain several words separated by spaces, so `## Gather Background Research Notes` is named `Gather Background Research Notes`. A section name can be any nonblank text, including spaces and punctuation, and no other naming rule applies. A single lowercase word such as `helpers` works, and so do several words with spaces and any letter case, such as `Author note`, `Section Two`, or `Stop Early`.

The name keeps only the visible text of the heading:

- Whitespace around the heading's text is dropped from the name.
- Inline code adds its text without the backticks, so the heading ``## Run `fetch` `` is named `Run fetch`.
- Emphasis and link markup add only their visible text, so a link's URL never enters the name.

Headings are read as plain CommonMark with no Markdown extensions, so extension syntax such as a trailing `{#id}` stays part of the section name.

A section's name is how the section is found at run time: jumps, lookups, and fanout find the section by its heading level together with that name (see [Resolving heading references](08-jump-and-call.md#resolving-heading-references)).

### What each section owns

A section's own content is the prose and Lua that follow its heading, up to the next heading of any level. So a parent's own prose and Lua are only the lines before its first child heading. In the `nesting` prompt, `Prepare`'s own prose is only `Gather the subject.`, and `Keep the subject short.` belongs to `### Author note`.

A parent section's content excludes its child sections, and each child section keeps its own prose, separate from its parent's. Text written after a subheading belongs to that subheading, so a parent section's content never resumes after one of its child sections.

Blank lines between a heading and its first content line, and blank lines before the next heading, are not part of the section's content, while indentation on the first content line is kept. A section whose heading is followed only by blank lines has empty content. A section with an empty body is accepted and can still hold child sections, like `## A` and `### X` in the `tree` prompt.

### What counts as a heading

The prompt body is read as CommonMark, so Markdown's own heading rules decide what counts as a heading. Two cases follow from that:

- A line starting with `#` inside a fenced or indented code block, `lua` fences included, is code and never a heading or a section.
- A heading inside a block quote or a list item does not start a section or a child section. Its text stays in the enclosing section's content exactly as written, with its `>` or `-` marker.

````markdown
---
name: containers
description: d
runvil: 0
---

# Containers

## S

> ## Quoted
> kept

- ## Listed

tail

## Next

next
````

This file has two sections, `S` and `Next`, and `S` has no children. The prose of `S` holds `> ## Quoted`, `- ## Listed`, and `tail` as written. An ordinary heading that follows a block quote or a list, like `## Next` here, still starts a new section as usual.

### Orphan and empty headings

Every heading must be exactly one level deeper than its parent, so heading levels step down one at a time and cannot be skipped, and the first section heading is an H2. An orphan heading is a section heading whose parent level is missing: a heading more than one level deeper than the section it sits under, or a first section heading deeper than H2. An orphan is never reattached to a shallower heading. It fails the parse with parse error kind `Structure`:

````markdown
---
name: orphan
description: d
runvil: 0
---

# Orphan

## A

a

#### D

d
````

````text
section `D` is an orphan H4 heading with no parent H3
````

The message names the section, its level, and the missing parent level, which is one level deeper than the heading the orphan sits under. Its form is ``section `{name}` is an orphan H{level} heading with no parent H{n}``, where `{n}` is the missing parent level. The error reports the orphan heading's line in the file at column 1, here line 13, column 1.

A first section heading written as `### A` or `#### A`, with no H2 above it, is an orphan the same way: it fails with a `Structure` parse error whose message contains `orphan` and names H2 as the missing parent.

A section heading with no name text fails with parse error kind `Structure` too, because every section heading needs text after trimming, and that text is the name that finds the section at run time. The message names the heading level, in the form `an H{level} section heading must not be empty`, so a bare `##` line reports:

````text
an H2 section heading must not be empty
````

## Sibling name uniqueness

After this section you can name sibling sections so that every heading reference finds exactly one target.

````markdown
---
name: siblings
description: d
runvil: 0
---

# Siblings

## A

### S

## B

### S
````

This file parses. Sibling sections under the same parent must have distinct names, but the same name can repeat under different parents. Here the two `### S` sections have different parents, `A` and `B`, so they are not siblings.

Jumps and lookups find sections by name, so a repeated sibling name would make the section an ambiguous target. Each nesting level checks a heading's name only against its own siblings, so sections under different parents can share a name, such as `### Helpers` under both `## A` and `## B`, or under both `## Top` and `## Grand`.

A child section can share its parent's name, such as `### Search` under `## Search`, because a parent and its child are not siblings, and the child's level starts its own name check.

### A duplicate sibling name

Two sibling sections with the same name fail the parse with parse error kind `Structure`:

````markdown
---
name: dup
description: d
---

# T

## S

p

## S

q
````

````text
duplicate sibling section name `S`: first declared at line 8, again at line 12; sibling section names must be unique
````

The message names the section and the lines of both headings. The line numbers count from the top of the prompt file, including the frontmatter, so with this four-line frontmatter the two `## S` headings are at lines 8 and 12. The failure also reports the prompt's frontmatter `name`, here `dup`, and the line and column of the second same-named heading, line 12, column 1. It also carries a non-empty byte span over the second heading, with offsets measured in the prompt body, after the frontmatter and any leading byte-order mark.

This file has no `runvil:` key, which is fine for showing a parse error, because the version is checked only when a run starts.

### How sibling names are compared

Sibling names are compared exactly as written, so names that differ only in letter case or inner spacing count as different sections. Within one set of siblings, each section's heading level and name together are unique, so a heading names at most one section among its siblings.

The comparison uses section names, not raw heading lines. Because emphasis, link, and code-span markers add nothing to a name (see [Section names](#section-names)), sibling headings that differ only in Markdown markup share one name. So ``## Run `fetch` `` and `## Run fetch` under the same parent fail as duplicate siblings.

### The sibling rules together

Sibling names are unique, and a section's children sit exactly one heading level deeper than it:

- A heading at or above the parent's level ends that parent's children.
- A heading more than one level deeper than its parent is rejected as an orphan.
- A repeated sibling name is rejected.

When a section has several problems, the parse reports the first one it meets. For each section it checks the heading level, then the name, then the section's blocks, then its child sections, then its list items, and only then a duplicate sibling name. A `lua` fence rule break anywhere in the body fails with parse error kind `Fence` before any of these section checks run.

## Heading references

After this section you can name any section from Lua with a heading reference string.

````markdown
---
name: require_library
description: Exercise a required library section and nested author prose
runvil: 0
---

# Require Library

The helper library is available to each section that requires it.

## Prepare

```lua
require '## helpers'
```

## helpers

```lua
local greeting = 'hello'
```
````

In this file, the string `'## helpers'` names the H2 section whose name is `helpers`. A heading reference is a section heading's `#` markers, a space, and the section name, written as a Lua string. So `'## loader'` names the H2 section named `loader`, and `'### Worker'` names an H3 section named `Worker`.

### Writing a heading reference

You write one `#` marker per heading level, then the section name, such as `'### Worker'` for an H3 or `'## Arm'` for an H2. The number of markers is the target's heading level at whatever depth the section sits, so `'## Help'` names an H2 and `'##### Child'` names an H5. A section name can contain an underscore, such as `house_style`, and still be reached by a heading reference like `'## house_style'`.

You use this same form with every call that names a section, whether the Lua sits in a section or in the H1 body:

| Call | Example |
|---|---|
| `tasks.fanout` | `tasks.fanout('### Worker', topics)` |
| `tasks.spawn` | `tasks.spawn('## Arm', { item = 'a' })` |
| `call` | `call('## Sub')` |
| `jump` | `jump('## C')` |
| `require` | `require '## helpers'` |
| `prompt.list` | `prompt.list('### Items')` |
| `prompt.section` | `prompt.section('## house_style')` |

[Resolving heading references](08-jump-and-call.md#resolving-heading-references) explains which sections each call can reach.

### Exact matching

A heading reference matches its target by exact level and name together, case-sensitively. So `'## Worker'` never finds the section `### Worker`, because the levels differ.

A heading reference is one or more `#` markers, then whitespace, then the section's non-empty name. Within that shape, whitespace works like this:

- Whitespace around the whole heading reference is ignored, so `'  ## Rules  '` names `## Rules`.
- Any whitespace, such as several spaces or a tab, can separate the markers from the name, so `'###    Worker'` names `### Worker`.
- Whitespace inside the name part is kept and must match the section's name exactly.

### Malformed heading references

A malformed heading reference is never silently reinterpreted. It raises a Lua error instead (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)), and each message ends with the reference trimmed of its surrounding whitespace, shown as `{stripped}` here:

| Problem | Message |
|---|---|
| No `#` markers | `section heading must include ### markers, got bare name: {stripped}` |
| No whitespace after the markers | `section heading must have whitespace after the {markers} markers: {stripped}` |
| Markers only, with or without whitespace after them | `section heading has no name: {stripped}` |

In the second message, `{markers}` is the run of `#` marks you wrote. So the bare name `'Nums'` reports `got bare name: Nums`, the reference `'##Nums'` reports `must have whitespace after the ## markers`, and `'###'` or `'### '` reports `section heading has no name`.
