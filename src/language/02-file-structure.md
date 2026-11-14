# File structure

Every prompt file has the same layout: frontmatter, then an optional preamble, then sections. This chapter gives the exact rules the parser enforces on each of those parts, from the delimiter lines at the top of the file down to the headings that name each section. Knowing them lets you write a file that parses the first time, and read any parse error as a pointer to the one rule your file broke.

## The shape of a prompt file

After this section you can lay out a complete prompt file and name each of its regions.

````markdown
---
name: one_block
description: Returns one word from a single Lua block.
runvil: 0
---

```lua
return "hello"
```
````

Running this prompt ends with this run result:

````text
hello
````

This prompt runs with only its frontmatter and one `lua` fence (see [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks)) after it. The YAML between the two `---` lines is the frontmatter (see [A Markdown file that is a program](01-what-a-prompt-is.md#a-markdown-file-that-is-a-program)), and it holds the three keys a prompt needs to run: `name`, `description`, and `runvil: 0`. The `lua` fence after the frontmatter is the preamble (see [The preamble](#the-preamble)), and here the preamble does all the work. The file has no headings, which is allowed: headings are optional.

### The three regions in order

The next prompt writes a line of notes as its preamble and adds one section:

````markdown
---
name: layout
description: Shows every region of a prompt file.
runvil: 0
---

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
2. The preamble: optional content between the frontmatter and the first heading. Here it is the line `This prompt returns one fixed line.`
3. The sections: the headings, each with the content under it. Here there is one, `## Answer`, which holds a prose line and a `lua` fence.

Text directly after the frontmatter is read as the preamble, and the first heading after it starts the first section. This preamble holds no `lua` fence, so it is commentary with no effect on the run, and [The preamble](#the-preamble) covers what a preamble can hold. A section such as `## Answer` can hold prose and `lua` fences side by side, and [Lua fences and prose blocks](03-blocks-and-prose.md#lua-fences-and-prose-blocks) teaches what each one does. Running this prompt returns `laid out`.

A section can hold more than two blocks. A `## Transform` section can hold a `lua` fence, then the prose line `Ask about {{ var.before }}.`, then a second `lua` fence, and the file parses with all three in that order.

### Sections in file order

A prompt body can hold several `##` section headings one after another, and each section can hold its own `lua` fence:

````markdown
---
name: steps
description: d
runvil: 0
---

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

The walk visits sections in file order (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). `## First` runs, has no `return`, and falls through, so `## Second` starts after it, and its `return` gives the run result. A prompt can have as many sections as it needs: one prompt can hold `## Spawner`, `## Sibling`, and `## Child`, where `## Sibling` starts after `## Spawner` falls through, and another can hold `## Only`, `## Child`, and `## Leaf`, each with its own `lua` fence.

After the frontmatter and the preamble, when the prompt has one, come the section headings, such as `## Main`, `## First`, `## Second`, `## Greet`, or `## Only`.

Sections can also nest deeper than `##`. A `### Worker` heading under `## Parent` starts a section inside `Parent`, and that layout parses and runs with only `name`, `description`, and `runvil: 0` in the frontmatter. [Section headings and nesting](#section-headings-and-nesting) gives the nesting rules.

### What the frontmatter holds

You write the frontmatter in YAML at the very top of the file. The first line of the file must be `---`, and spaces before or after the dashes on that line are ignored. A file that starts any other way is rejected with the message `file must begin with a --- frontmatter delimiter`, which [Frontmatter delimiters, byte-order mark, and line endings](#frontmatter-delimiters-byte-order-mark-and-line-endings) covers with the other delimiter rules.

The frontmatter declares the prompt's Packs with the `uses` list, its arguments with `args`, and requirements on its model roles with `models`. None of these is needed for a prompt to parse and run. A frontmatter of only `name`, `description`, and `runvil: 0` between the `---` lines is complete:

````markdown
---
name: frontmatter-only
description: d
runvil: 0
---
````

This prompt leaves out `models`, `args`, and `uses`, and it still parses and runs. [Prompts without sections](#prompts-without-sections) explains how a prompt with nothing after its frontmatter ends. The `description` value can be a single character such as `d`, and `runvil: 0` is the version value. A frontmatter of `name: greeter`, `description: says hi`, and `runvil: 0` is just as complete.

## Frontmatter delimiters, byte-order mark, and line endings

After this section you can open and close the frontmatter so the parser finds it, whichever editor saved the file.

````markdown
---
name: minimal
description: minimum valid
runvil: 0
---
````

A frontmatter delimiter is a line of three dashes, `---`, that opens or closes the frontmatter. You write the frontmatter as a YAML block between two delimiter lines at the very top of the prompt file, and the Markdown body is everything after the closing `---` line. In the file above, line 1 opens the frontmatter, lines 2 to 4 are YAML, and line 5 closes it. A body would start on line 6, and this file has none.

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

## Main
````

````text
frontmatter was not closed with ---
````

Both errors point at line 1, where the frontmatter opens or should open, and neither names a prompt, because the prompt's name is not known yet when the error is raised.

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
  strong:
    min_context: 40000
---
````

You write the frontmatter as YAML, and the parser decodes that YAML into the set of frontmatter keys a prompt accepts. This prompt writes its nested values in YAML block style: the key sits alone on its line, and more-indented lines under it hold `key: value` map entries or `- ` list items. Here `models:` holds a map two levels deep, with `strong:` under it and `min_context: 40000` under that, and `uses:` holds the single list item `- web`.

### Flow style

YAML flow style writes a map or a list on one line, with `{ }` for maps and `[ ]` for lists, nested as deep as you need. This prompt declares the same keys and values in flow style:

````markdown
---
name: analyst_example
description: Demonstrate frontmatter model roles and models.use for careful model selection.
runvil: 0
uses: [web]
models: { strong: { min_context: 40000 } }
---
````

Both files decode to the same frontmatter. Flow style suits short declarations, such as `uses: [web]`, `models: { strong: { thinking: true } }`, or `args: { type: object, properties: { times: { type: integer, default: 1 }, name: { type: string } } }`.

The two styles mix, so a block-style `models:` key can hold a flow-style entry on its indented line:

````markdown
---
name: hello
description: Say hello
runvil: 0
models:
  fast: {}
---
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

By convention, prompts write `name` as a lowercase snake_case identifier, with words joined by underscores and digits allowed after the first letter, such as `phase_boundaries`, `require_library`, `store_triad`, `analyst_example`, or `research_person`. The parser does not enforce this format. A name can also be kebab-case, such as `needs-web`, `version-zero`, `frontmatter-only`, or `with-notes`, or a single letter such as `t`.

The `name` value does not have to match the file name. The file `research-person.md` declares `name: research_person`, `analyst-example.md` declares `name: analyst_example`, and `prologue-prose-epilog.md` declares `name: phase_boundaries`.

Every parse error found past the frontmatter, a `Lua` compile failure included, reports the prompt's `name`, beside its line and column when the error has them, so you can tell which prompt failed; only an internal fault reports none. A duplicate `## S` section heading in a prompt with `name: dup` fails with parse error kind `Structure` and names `dup` (see [Sibling name uniqueness](#sibling-name-uniqueness)). A frontmatter failure, such as malformed YAML, fails with parse error kind `Frontmatter` and reports no name.

### The `description` key

Every prompt declares a `description:` string that summarizes what the prompt does, such as `description: Greet a name using a Lua-computed value` or `description: Writes a note to the store and reads it back.` The value becomes the prompt's description as free text, kept exactly as written, down to its closing period. Even a one-letter value such as `description: d` is accepted.

By convention, the description fits on one line. The parser also accepts a several-line `description: |` block value (see [Multi-line values](#multi-line-values)), so one line is a convention, not a rule.

The Workshop's Run window shows the description as a read-only `description` row under the prompt's `name` row whenever the description is not empty.

### What a file needs to parse and to run

A file needs only `name` and `description` to parse. A frontmatter holding just those two keys, with no `runvil` key or any other key, passes the frontmatter parse, and the parser moves on to check the body:

````markdown
---
name: x
description: d
---

## S

p
````

Every other key, including `runvil`, `max_tool_iterations`, `uses`, `args`, and `models`, takes a default when you leave it out.

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

This is the frontmatter of a prompt; its body follows the closing `---`.

### Missing `name` or `description`

Leaving out `name:` or `description:` fails the frontmatter parse with a missing-field error from the YAML reader, because these are the only two keys with no default:

````markdown
---
name: greeter
runvil: 0
---
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

A file parses whether or not it has a `runvil:` key, and the run checks the declared version on its first step, before any section runs. The run checks the version first, then its arguments, then its requirements, so a prompt with a missing or unsupported version reports the version failure even when the run also has arguments that do not match the prompt's `args:` declaration or an environment that falls short of the prompt.

A prompt whose `runvil` version is missing or unsupported fails before any section runs. None of its sections run, nothing is reported beyond that one failure, and its own `return` never runs.

### A missing version

A file with no `runvil:` key still parses, but it is not treated as a Runvil prompt. Here is the `version-zero` prompt without its `runvil: 0` line:

````markdown
---
name: version-zero
description: d
---

## Only

```lua
return "ran"
```
````

The run ends on its first step with error code `structure_invalid`, as a `Structure` parse error with this message:

````text
not a runvil prompt: no runvil version (prompt version-zero)
````

The message ends by naming the prompt by its frontmatter `name`, here `version-zero`.

### An unsupported version

A file that declares a major other than `0` still parses, but this build does not support that major, so the run ends on its first step with error code `unsupported_version`. The version check never falls back to major 0. A prompt runs only when it declares `runvil: 0`, and any other declared major fails the run with an error that names that major:

````markdown
---
name: version-zero
description: d
runvil: 1
---

## Only

```lua
return "ran"
```
````

````text
unsupported runvil version: 1 (this build supports major 0) (at version-zero.md:4)
````

The error's message always has the form `unsupported runvil version: {n} (this build supports major 0)`, where `{n}` is the declared number, so `runvil: 2` or `runvil: 7` gives the same message with its own number. The run's message then ends with ` (at {file}:{line})`, the line of the `runvil:` key, here line 4 (see [Where a run error's message points](16-limits-and-errors.md#where-a-run-errors-message-points)). A failure caused by an unsupported `runvil` version is not retryable.

## Optional keys and unknown keys

After this section you can add the optional frontmatter keys a prompt needs, and you know why a misspelled key fails the whole file.

````markdown
---
name: x
description: d
---

## S

p
````

The frontmatter recognizes exactly seven top-level keys, and only `name` and `description` are required, so this file parses. You leave out the other five when a prompt does not need them, and you add them alongside `name`, `description`, and `runvil: 0` when it does.

### The seven keys

This table lists the seven keys in the order this chapter teaches them, with what each one means when you leave it out:

| Key | Required | When you leave it out |
|---|---|---|
| `name` | yes | the parse fails |
| `description` | yes | the parse fails |
| `runvil` | no, but needed to run | no declared version: the file parses, and running it fails with `not a runvil prompt: no runvil version (prompt {name})` |
| `max_tool_iterations` | no | the built-in round cap stays in force |
| `uses` | no | no Packs |
| `args` | no | the default declaration of one optional `string` argument named `text` |
| `models` | no | no requirements on any model role |

Because every key besides `name` and `description` has a default, a frontmatter of just those two parses, and so does a prompt with none of the contract keys. `max_tool_iterations` accepts a value from 1 to 1000, and [Tool rounds and the round cap](11-conversations.md#tool-rounds-and-the-round-cap) covers what it controls. [Prompts with no `args:` key](06-arguments.md#prompts-with-no-args-key) covers the default declaration of `args`.

### Adding keys a prompt needs

The `hello` prompt under [Flow style](#flow-style) shows the pattern. Its frontmatter holds only `name`, `description`, `runvil`, and `models`, and leaves out `max_tool_iterations`, `uses`, and `args` because it does not need them. Its `models:` key names a `fast: {}` role next to the usual three keys. The shipped `hello.md` prompt declares no model requirements, so it leaves out `models` too, and its frontmatter holds only `name`, `description`, and `runvil`.

### The contract keys

The contract keys are `uses`, `args`, and `models` (see [Putting the parts together](01-what-a-prompt-is.md#putting-the-parts-together)). Together they declare what a prompt needs from its environment:

- `uses` lists the Packs the prompt requires, and each one listed is required.
- `args` declares the prompt's arguments. The arguments of a run are one JSON object, and the value of `args` is a declaration of that object's shape, written in a subset of JSON Schema (see [Declaring arguments in frontmatter](06-arguments.md#declaring-arguments-in-frontmatter)).
- `models` attaches requirements to the five built-in model roles (see [Model roles instead of models](10-models.md#model-roles-instead-of-models)): a minimum context and a thinking switch for each role it names.

Checking these declarations against the caller's environment and the run's arguments happens on the run's first step, not at parse time. For example, a `small` role that sets `thinking: false` is reported as an unmet requirement when the model filling that role always thinks, and arguments that do not match the `args:` declaration end the run there, before any section runs (see [Argument errors](06-arguments.md#argument-errors)).

All three contract keys can sit together in one frontmatter block alongside `name`, `description`, and `runvil: 0`, and each reads back as declared:

````markdown
---
name: greeter
description: says hi
runvil: 0
uses: [web]
args: { type: object, properties: { times: { type: integer, default: 1 }, name: { type: string } }, required: [name] }
models: { strong: { min_context: 40000 } }
---
````

Here the `args:` value declares an object with two fields: an optional integer `times`, which is 1 when the caller leaves it out, and a required string `name`.

A frontmatter with `uses: [web, user]`, an `args:` declaration that holds the arg `use_mcp`, and a `models:` map holding `strong` and `fast` parses and reads back the same way.

### Unknown keys

You spell frontmatter keys exactly as the language defines them. An unknown frontmatter key is any top-level key other than the seven, and it is never ignored: it fails the whole file with parse error kind `Frontmatter`, because any other key is treated as a prompt authoring error. The `greeter` prompt above parses with `uses: [web]`, but the same file with that key misspelled fails:

````markdown
---
name: greeter
description: says hi
runvil: 0
usse: [web]
---
````

````text
invalid frontmatter: unknown field `usse`, expected one of `name`, `description`, `runvil`, `max_tool_iterations`, `uses`, `args`, `models` at line 4 column 1
````

The message has the `invalid frontmatter: {message}` form, and `{message}` names the offending key, here `usse`, and lists the seven keys. It ends with the YAML reader's own position, which counts lines from the first line after the opening `---`, so its line 4 is line 5 of the file.

Unknown keys are rejected inside nested declarations too. An `args:` declaration accepts only the keywords of its subset of JSON Schema, both at its top level and in each arg it declares, a model role takes only `min_context` and `thinking`, and any other key inside them fails the parse with parse error kind `Frontmatter`.

## The name grammar for arg names and adhoc tool names

After this section you can choose names for args and adhoc tools that both the parser and the Engine accept, and you know why the keys under `models:` are outside that grammar.

````markdown
---
name: contract_names
description: d
runvil: 0
args:
  type: object
  properties:
    use_mcp: { type: boolean }
    max_items: { type: integer, default: 1 }
models: { strong: {}, fast: {} }
---
````

This frontmatter parses. The keys under `properties:` are arg names, and the keys under `models:` are role names. An arg name is a field name under `properties`, at any depth of the `args:` declaration. At the top level it is the key that names one argument, and the argument reaches Lua as a field of `args`. Deeper, under the `properties` of an object inside the declaration, it names one field of that object. An arg name follows the same name grammar as an adhoc tool name, which is the `name` Lua code gives an adhoc tool (see [Effects and Harness work](01-what-a-prompt-is.md#effects-and-harness-work)) in the attrs table of `tools.adhoc` (see [Adhoc tools with `tools.adhoc`](12-tools.md#adhoc-tools-with-toolsadhoc)). A role name is one of the five built-in model roles, `default`, `strong`, `small`, `fast`, and `creative`, so the grammar does not apply to it.

### The prompt-local alias grammar

Every arg name and adhoc tool name follows the prompt-local alias grammar:

````text
[A-Za-z][A-Za-z0-9_-]{0,63}
````

In words, a name is 1 to 64 ASCII characters: a letter first, then letters, digits, `_`, or `-`. Names such as `search`, `fetch`, `use_mcp`, `limit`, `query`, `fast-model`, and `max_items` are all valid. Rule by rule:

- Length: a name is 1 to 64 characters long, so it always has at least one character. A name of exactly 64 characters is valid, and 64 is the limit.
- First character: an ASCII letter of either case, `A` to `Z` or `a` to `z`.
- Later characters: ASCII letters, ASCII digits, `_`, and `-` only. Every character of a name is ASCII, so its letters are `A` to `Z` and `a` to `z` in every position.
- Digits, underscores, and hyphens: these appear only after the first letter, as in `search1`, `web-fetch2`, `web_search`, or `web-search`.
- Joining words: you join words with `_` or `-`, as in `web_fetch` or `web-fetch`, because spaces and dots are not name characters.
- No slash: a name is a single plain name with no `/`, which keeps an adhoc tool name distinct from a tool id such as `web/fetch`, and from the adhoc tool's own id, `adhoc/<name lowercased>` (see [Pack ids, tool ids, and wire names](12-tools.md#pack-ids-tool-ids-and-wire-names)).

Within those rules, any style works. Names in uppercase, lowercase, mixed case, snake_case, or kebab-case, with trailing digits, or of a single letter are all valid, such as `a`, `A`, `search`, `web_fetch`, `web-fetch2`, and a name of 64 `z` characters. A valid name keeps its exact text as written, including letter case.

Put the other way, a name fails the grammar when it is empty or longer than 64 characters, when it starts with anything other than an ASCII letter, or when it contains a space, a dot, a slash, or a non-ASCII character.

### Where the grammar applies

At parse time, the grammar applies to every key of every `properties` map in the `args:` declaration, at any depth, such as the arg name `use_mcp` above. Arg names such as `times` and `name` are valid too. A key under `models:` is not checked against the grammar. It must be one of the five role names, spelled exactly, so a key such as `reviewer` or `Fast` is rejected even though both fit the grammar. Lua code refers to a role by its name as a string, as in `models.use("strong")`.

At run time, the Engine enforces the same grammar for the `name` Lua passes to `tools.adhoc`, with the same length, first-character, and later-character rules.

`models.use` and `models.get` apply no grammar. `models.use` compares the name with the five role names, so a name outside the grammar and a well-formed name that is not a role fail with the same message, which [Run-time name errors](#run-time-name-errors) gives. `models.get` does the same unless the Host allows raw model ids. With raw model ids allowed, `models.get` reads a name outside the five as a raw model id instead, the name a gateway model is listed under, because model names can contain `/`, `.`, and `:`, so `models.get('qwen/qwen3-8b')` returns a handle bound to `gateway/qwen/qwen3-8b`, whose `model_id` is `qwen/qwen3-8b`. A role name always resolves as the role, with or without raw model ids.

### Contract maps and duplicate keys

A contract map is the `models:` map or a `properties` map of the `args:` declaration. They share the uniqueness check: each key appears once within its map. Keys are compared exactly, so arg names that differ only in case, such as `Times` and `times`, are two distinct entries.

A repeated key fails the parse with parse error kind `Frontmatter`, and the message names the key kind and the repeated key, where the key kind `{what}` is `model role` for `models:` and `argument name` for a `properties` map:

````text
duplicate {what} `{key}`: contract map keys must be unique
````

The parser checks each key first, against the grammar for a `properties` map and against the five role names for `models:`, and for uniqueness second, and it checks a key before it reads the key's value. It walks the keys in source order and stops at the first bad one, so:

- A map with several bad keys reports only the first one.
- An entry with both a bad key and a bad value reports the key error.

### Reserved names

A reserved name is a name that is already taken in a section entry's Lua environment, such as the global `store`, the Lua function `pairs`, or the keyword `end` (see [The sandbox](05-lua-environment.md#the-sandbox)). Arg names are not checked against reserved names, because an arg does not become a Lua global: it is a field of `args`. So every reserved name the grammar allows works as an arg name, and `prose`, `store`, and `end` are all accepted:

````markdown
---
name: reserved
description: d
runvil: 0
args:
  type: object
  properties:
    prose: { type: string }
    store: { type: string }
    end: { type: boolean }
---
````

This frontmatter declares three args, named `prose`, `store`, and `end`. A reserved name such as `pairs` works as an arg name the same way.

### Parse-time name errors

A key under a `properties` map that is outside the grammar fails the parse with parse error kind `Frontmatter`. The `{message}` of the `invalid frontmatter: {message}` message names the offending key and shows the grammar:

````text
invalid argument name `{key}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

A key under `models:` that is not one of the five role names fails the parse with parse error kind `Frontmatter` too. The message contains the YAML reader's own text for an unknown name, which names the key and lists the five roles:

````text
unknown variant `{key}`, expected one of `default`, `strong`, `small`, `fast`, `creative`
````

Inside a role, a field other than `min_context` and `thinking` fails the same way, with a message that contains this text, where `{field}` is the rejected field:

````text
unknown field `{field}`, expected `min_context` or `thinking`
````

So a role that has a `description:` field is rejected, even though the prompt itself takes one.

Writing `models:` or a `properties` map as something other than a map, such as a list or a scalar, fails with parse error kind `Frontmatter` and this detail, where `{found}` is the YAML reader's description of the value you gave and `{what}` is `model role` or `argument name`:

````text
invalid type: {found}, expected a map of {what} keys to declarations
````

Writing `args:` itself as a list or a scalar fails the same way, with this detail:

````text
invalid type: {found}, expected a map of schema keywords
````

Each grammar, duplicate, and map-type error from `models:` and `args:` puts the YAML path of the map at fault in front of its detail: `models` for the roles, `args` for the declaration itself, and `args.properties` for the top-level arg names. A deeper `properties` map has a longer path, such as `args.properties.opts.properties` for the fields of an object argument named `opts`, so the path shows where in the declaration a bad name sits.

Each of these errors also reports a file line and column. A key under `models:` that is not a role name is reported at the key itself, and a map-type error at the value. A grammar or duplicate error from a `properties` map, like a duplicate role, is reported where its map starts, which in block style is the map's first key: line 8, column 5 in the `contract_names` prompt above.

Every arg name violation, whether an empty name, a name that is too long, a bad first character, or a bad later character, produces the same single message form. The message shows the grammar but not which rule failed, so check the name against every rule in [The prompt-local alias grammar](#the-prompt-local-alias-grammar).

### Run-time name errors

When the name passed to `models.use` is not one of the five role names, the call fails with a `lua_runtime` error whose text names the rejected label and lists the five roles, in one form for every rejected name:

````text
models.use label "{label}" is not a model role: expected default, strong, small, fast or creative
````

`models.get` fails the same way when the name is not a role and the Host has not allowed raw model ids, with `models.get` in place of `models.use`:

````text
models.get label "{label}" is not a model role: expected default, strong, small, fast or creative
````

The rejected label appears in double quotes with escaping, so a quote, backslash, or control character in the name appears escaped. For example, `models.use('reviewer')` fails with `models.use label "reviewer" is not a model role: expected default, strong, small, fast or creative`. Caught with `pcall`, the error value's message puts the file and line of the call in front of that text and the default text for a Lua failure after it (see [Where a caught error's message points](16-limits-and-errors.md#where-a-caught-errors-message-points)), and the same holds for every error in this section.

When the `name` passed to `tools.adhoc` breaks the grammar, the `tools.adhoc` call fails with a `lua_runtime` error whose text names the rejected name and the grammar, again in one form for every rule:

````text
tools.adhoc: invalid name "{name}": expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

The rejected name appears in double quotes with the same escaping. One way to hit this error is to pass a tool id where a name belongs, since a tool id contains `/`. Lua code can catch the error with `pcall` (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

````lua
local _, err = pcall(tools.adhoc, { name = 'web/search', description = 'x' }, function() end)
````

After this line, `tostring(err)` contains `tools.adhoc: invalid name "web/search"`.

Each adhoc tool name can be defined once in a section entry. A second `tools.adhoc` call with a name that is already defined fails with this text:

````text
tools.adhoc name "{name}" is already registered
````

So defining the name `gh_search` a second time fails with `tools.adhoc name "gh_search" is already registered`.

An adhoc tool's id is `adhoc/` followed by its name in lowercase, so two names that differ only in letter case would share one id. The second of them fails with this text, where `{id}` is that shared id, `adhoc/` followed by the name in lowercase, and `{other}` is the name that already holds the id:

````text
tools.adhoc name "{name}" has the id {id}, which "{other}" already holds
````

So after `gh_search` is defined, defining `GH_Search` fails with `tools.adhoc name "GH_Search" has the id adhoc/gh_search, which "gh_search" already holds`.

## The preamble

After this section you can put Lua and notes above a prompt's first heading and tell what each does there.

````markdown
---
name: preamble_demo
description: d
runvil: 0
---

```lua
var.greeting = 'hello'
```

## Answer

```lua
return var.greeting .. ' from ' .. sys.section_name
```
````

````text
hello from Answer
````

The preamble is the text between the frontmatter and the first heading, and in a file with no heading it is the whole body. Here it is the first `lua` fence. A preamble that holds a `lua` fence is a section, the first top-level section, so the run starts there: this preamble sets `var.greeting` and falls through, and `Answer` reads the value and returns `hello from Answer` (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)). The preamble is optional, so the first heading can come right after the frontmatter.

### A preamble with Lua

A preamble that holds at least one `lua` fence is a top-level section that has no heading. So it has no name, and `sys.section_name` there reads the empty string:

````markdown
---
name: preamble_name
description: d
runvil: 0
---

```lua
return 'section name: "' .. sys.section_name .. '"'
```
````

````text
section name: ""
````

Such a preamble holds any sequence of prose and `lua` fences, kept in source order, as the content under a heading does. Prose there is the pending prose of the next `lua` fence, which reads it as `prose` (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)):

````markdown
---
name: preamble_prose
description: d
runvil: 0
---

Keep the answer short.

```lua
return 'read: ' .. prose
```
````

````text
read: Keep the answer short.
````

Preamble prose that no Lua reads has no effect:

````markdown
---
name: unread-prose
description: d
runvil: 0
---

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

The preamble's fence runs before the prose, and no Lua after it reads the prose `{{ var.omit }}`, so the prose stays inert, and the run returns `ok` without needing a model.

The preamble ends where the first heading begins, so it has no child sections. No section path names the preamble, because every step of a path is a section name and the preamble has none. Its own Lua names the other sections by path as any section's Lua does, and [`jump` and `call` from the preamble](08-jump-and-call.md#jump-and-call-from-the-preamble) shows the paths it writes.

### A preamble without Lua

A preamble that holds no `lua` fence is commentary. It is not a section, nothing reads it, and it has no effect, so editing notes above the first heading change nothing:

````markdown
---
name: with-notes
description: d
runvil: 0
---

Editing notes for this prompt live here and change nothing.

## Answer

```lua
return 'ok'
```
````

````text
ok
````

The run starts at `Answer`, which is the first top-level section of this prompt.

The strict `lua` fence rule applies in the preamble as it does everywhere in the body, whether or not the preamble holds Lua. A fence there whose label starts like `lua` must be an exact `lua` fence, or the parse fails with parse error kind `Fence` (see [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors)).

A `lua` fence shown inside a longer fence stays prose, so the preamble can show a Lua fence as an example. A four-backtick `markdown` fence in the preamble can contain a three-backtick `lua extra` opener, and that inner line is never read as a fence of its own, so a preamble of notes that shows Lua this way is still commentary.

### Prompts without sections

A prompt needs no heading at all. Frontmatter and one Lua block is a complete prompt: the block is the preamble, and the preamble does all the work, as in the `one_block` prompt at the start of this chapter, which runs to the result `hello`. Such a prompt runs with or without a Lua `return`:

- When the preamble's Lua returns a value, that value is the run result, such as `hello`.
- When the preamble's Lua has no `return`, or returns no value, the run ends with the generic result `done`.

A prompt has no minimum number of sections either. A file with no heading and no `lua` fence has no section at all, and it is still a valid prompt, whose run ends with the same generic result `done`. The `frontmatter-only` prompt under [What the frontmatter holds](#what-the-frontmatter-holds) is one, and so is a file of frontmatter and prose:

````markdown
---
name: only-prose
description: d
runvil: 0
---

Text.
````

````text
done
````

However many keys the frontmatter declares, the prompt's work happens in the preamble and the sections, never in the frontmatter.

## Section headings and nesting

After this section you can nest sections by heading level and tell which section owns each line of the file.

````markdown
---
name: nesting
description: d
runvil: 0
---

## Prepare

Gather the subject.

### Author note

Keep the subject short.

## Finish
````

This file has two top-level sections, `Prepare` and `Finish`. `### Author note` is a child of `Prepare`, not a third top-level section, so it belongs to `Prepare`. `Prepare`'s own prose is only `Gather the subject.`, and the line under `### Author note` belongs to the child.

### Heading levels

A heading level is the number of `#` markers that start a heading. You mark sections with Markdown headings from H1 (`#`) through H6 (`######`), the deepest Markdown heading level, and each heading's level decides where its section nests. Every heading starts a section, and each heading, together with the prose and Lua under it, forms one section.

Each section has a name taken from its heading, a level, its own prose and Lua, and its child sections.

The top level is the shallowest heading level in the file, and the sections at that level are the top-level sections, kept in file order. The examples in this guide write top-level sections as `##` headings, and so do the shipped prompts. Such a `##` heading starts a top-level section at level 2, and the text of the heading is the section's name, as with `## Main`, `## Analyze`, `## Greet`, and `## Research` in the shipped prompts. So `## First` followed by `## Second` gives two top-level sections, First then Second. A preamble that holds a `lua` fence is a top-level section too, ahead of every heading (see [The preamble](#the-preamble)).

No particular section name is required. The first top-level section is the entry section whatever its name, so `## Zebra` followed by `## Main` makes Zebra the entry, and a preamble that holds Lua is the entry of its prompt (see [The section walk](04-how-a-prompt-runs.md#the-section-walk)).

The walk runs the top-level sections one after another, and a deeper section runs only when Lua names it. So sections that are meant to run one after another all sit at the top level. A heading above them at a shallower level makes them its children:

````markdown
---
name: nested_steps
description: d
runvil: 0
---

# Steps

## First

```lua
return 'first'
```

## Second

```lua
return 'second'
```
````

````text
done
````

`# Steps` is the only top-level section of this prompt, and `First` and `Second` are its children. The run enters `Steps`, which has no Lua of its own, and ends there with the generic result `done`, so neither child runs. The children run when the Lua in `Steps` names them:

````markdown
---
name: called_steps
description: d
runvil: 0
---

# Steps

```lua
return call('First') .. ', then ' .. call('Second')
```

## First

```lua
return 'first'
```

## Second

```lua
return 'second'
```
````

````text
first, then second
````

Each `call` runs one child of `Steps` and gives back the value that child returns (see [Calling a section](08-jump-and-call.md#calling-a-section)).

### Child sections

Sections nest by heading level, so a deeper heading belongs to the section above it rather than starting another top-level section. A child section is a section whose heading sits one level below its parent's heading: an H3 after an H2 becomes that H2's child, and an H4 after that H3 becomes the H3's child. A `###` under a `##` is a level-3 child rather than a new top-level section, a `####` under that `###` is a level-4 grandchild, and each one has its own level and prose and can hold its own `lua` block. A child is named by the text of its heading without the `#` markers, so `### Helpers` after `## A` is A's first child, named Helpers. Sibling sections are sections that share a parent.

````markdown
---
name: tree
description: d
runvil: 0
---

## A

### X

#### P

#### Q

### Y

## B
````

Here `## A` holds `### X` and `### Y`, `### X` holds `#### P` and `#### Q`, and `## B` is a sibling of `## A`. Lua written in section A names section B with the section path `'../B'`, and it names A's child X with `'X'` (see [Section paths](#section-paths)).

The rule behind this shape is that a heading deeper than the current section becomes its child, and a heading at the same or a shallower level ends that section and belongs to an ancestor. So where the top level is H2, each H2 starts a new top-level section, including an H2 that follows a nested H3 child. Top-level sections keep their file order without counting nested `###` sections, so `## First`, `### Child`, `## Second` gives two top-level sections, First and Second.

You can place several H3 child sections side by side under one H2, as `## Research` holds `### Worker` and `### Topics`.

Sections nest from the top level down to `######`, which is five levels deep where the top level is `##`, as in the next prompt, and six where it is `#`. Each heading's parent is the nearest heading above it that is exactly one level shallower:

````markdown
---
name: deep
description: d
runvil: 0
---

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

A section's name is how the section is found at run time: jumps, calls, reads, and fanout name a section by a section path, which is made of section names (see [Section paths](#section-paths)).

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

Every heading must be exactly one level deeper than its parent, so heading levels step down one at a time and cannot be skipped. An orphan heading is a section heading whose parent level is missing: a heading more than one level deeper than the section it sits under, or a first heading that is deeper than a heading after it. An orphan is never reattached to a shallower heading. It fails the parse with parse error kind `Structure`:

````markdown
---
name: orphan
description: d
runvil: 0
---

## A

a

#### D

d
````

````text
section `D` is an orphan H4 heading with no parent H3
````

The message names the section, its level, and the missing parent level, which is one level deeper than the heading the orphan sits under. Its form is ``section `{name}` is an orphan H{level} heading with no parent H{n}``, where `{n}` is the missing parent level. The error reports the orphan heading's line in the file at column 1, here line 11, column 1.

The first heading of a file can be at any level, because the top level is the shallowest heading level in the whole file. A file whose only headings are `### A` and `#### B` parses, with `A` as a top-level section and `B` as its child. A later heading that is shallower than the first one puts the top level above the first heading, which leaves the first heading an orphan:

````markdown
---
name: late
description: d
runvil: 0
---

## Run

# Late
````

````text
section `Run` is an orphan H2 heading with no parent H1
````

Here `# Late` makes H1 the top level, and `## Run` has no H1 above it. The error points at `## Run`, on line 7, column 1, and the missing parent level it names is the top level.

A heading with no name text fails with parse error kind `Structure` too, at every level from `#` to `######`, because every heading needs text after trimming, and that text is the name that finds the section at run time. The message names the heading level, in the form `an H{level} section heading must not be empty`, so a bare `##` line reports:

````text
an H2 section heading must not be empty
````

A bare `#` line reports `an H1 section heading must not be empty` in the same way.

## Sibling name uniqueness

After this section you can name sibling sections so that every section path finds exactly one target.

````markdown
---
name: siblings
description: d
runvil: 0
---

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

## S

p

## S

q
````

````text
duplicate sibling section name `S`: first declared at line 6, again at line 10; sibling section names must be unique
````

The message names the section and the lines of both headings. The line numbers count from the top of the prompt file, including the frontmatter, so with this four-line frontmatter the two `## S` headings are at lines 6 and 10. The failure also reports the prompt's frontmatter `name`, here `dup`, and the line and column of the second same-named heading, line 10, column 1. Its location covers that whole heading line and ends at line 11, column 1 (see [Location fields](16-limits-and-errors.md#location-fields)).

This file has no `runvil:` key, which is fine for showing a parse error, because the version is checked only when a run starts.

### How sibling names are compared

Sibling names are compared exactly as written, so names that differ only in letter case or inner spacing count as different sections. Within one set of siblings each section name is unique, so a section path names at most one section.

The comparison uses section names, not raw heading lines. Because emphasis, link, and code-span markers add nothing to a name (see [Section names](#section-names)), sibling headings that differ only in Markdown markup share one name. So ``## Run `fetch` `` and `## Run fetch` under the same parent fail as duplicate siblings.

### The sibling rules together

Sibling names are unique, and a section's children sit exactly one heading level deeper than it:

- A heading at or above the parent's level ends that parent's children.
- A heading more than one level deeper than its parent is rejected as an orphan.
- A repeated sibling name is rejected.

When a section has several problems, the parse reports the first one it meets. For each section it checks the heading level, then the name, then the section's blocks, then its child sections, and only then a duplicate sibling name. A `lua` fence rule break anywhere in the body fails with parse error kind `Fence` before any of these section checks run.

## Section paths

After this section you can name any section from Lua with a section path.

````markdown
---
name: paths
description: Names a child, a sibling, and a section from the top.
runvil: 0
---

## Prepare

```lua
local shout = require '/helpers'
var.note = shout(prompt.section('Author note'))
jump('../Finish')
```

### Author note

Keep the subject short.

## Skipped

```lua
return 'skipped'
```

## Finish

```lua
return var.note
```

## helpers

```lua
return function(text) return text:upper() end
```
````

````text
KEEP THE SUBJECT SHORT.
````

The block in `Prepare` names three sections, each with a section path. `'/helpers'` starts at the top of the prompt and names the top-level section `helpers`. `'Author note'` names `Prepare`'s own child, and `'../Finish'` names its sibling `Finish`. A section path is one or more section names separated by `/`, written as a Lua string.

### Writing a section path

A section path has three forms:

- `name` names the direct child with that section name.
- `../name` names the sibling with that section name, which is another section under the same parent. A top-level section's siblings are the other top-level sections. From the preamble the top-level sections are siblings too, so `call('../Prepare')` written there runs `Prepare` (see [`jump` and `call` from the preamble](08-jump-and-call.md#jump-and-call-from-the-preamble)).
- `/a/b` starts at the top-level sections and names one section per level, so `'/Prepare/Author note'` names the section `Author note` under the top-level section `Prepare`. A path from the top names the same section wherever it is written.

The first two forms are relative paths. A relative path starts at the section its code is written in, and it names a neighbor of that section. The heading level plays no part in any path. A section is found by where it sits, so moving a section and its neighbors to another depth changes none of the paths between them.

Which forms a call takes depends on what the call does:

| Call | Forms | Example |
|---|---|---|
| `call` | relative | `call('Sub')` |
| `jump` | relative | `jump('../C')` |
| `tasks.spawn` | relative | `tasks.spawn('Arm', { item = 'a' })` |
| `tasks.fanout` | relative | `tasks.fanout('Worker', topics)` |
| `prompt.section`, `prompt.section_or` | relative, or from the top | `prompt.section('/house_style')` |
| `require` | from the top | `require '/helpers'` |

The four calls that run a section take only the relative forms, so each of them reaches only a neighbor of the section it starts from (see [The visible set](08-jump-and-call.md#the-visible-set)). The prompt reads and `require` leave the walk where it is, and they can name every section that has a heading, wherever it sits in the prompt. [Resolving section paths](08-jump-and-call.md#resolving-section-paths) explains where each call starts and what it does when a path names no section.

### Exact matching

Each name in a path matches a section name exactly and case-sensitively, so `'worker'` never finds the section `Worker`. A section name can contain spaces, underscores, and punctuation, such as `Author note` or `house_style`, and the path writes it as it is.

Whitespace works like this:

- Whitespace around the whole path and around each name is ignored, so `'  ../Rules  '` and `' .. / Rules '` both name the sibling `Rules`.
- Whitespace inside a name is kept and must match the section name exactly.

Three characters need a backslash in front of them when they are part of a name:

- A slash is written `\/`, because a bare slash separates two names.
- A backslash is written `\\`.
- The bracket of a name that ends in a bracketed whole number, such as `Appendix [2]`, is written `\[`, because a path over parsed text reads that ending as an index (see [Parsing Markdown with `markdown`](05-lua-environment.md#parsing-markdown-with-markdown)).

A quoted Lua string doubles each backslash, so the section named `Input/Output` is `'Input\\/Output'`. A long-bracket string needs no doubling, as in `[[Input\/Output]]`.

A section whose name is exactly `..` cannot be named by a path.

### Malformed section paths

A malformed section path is never silently reinterpreted. It raises a Lua error instead (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)), and the message quotes the path, shown as `{path}` here:

| Problem | Message |
|---|---|
| The path is not a string | `section target must be a string, got {type}` |
| An empty name, as in `''`, `'Kid/'`, or `'../'` | `section path "{path}" has an empty step` |
| A relative path with more than one name, as in `'Kid/Grand'` or `'../../Top'` | `section path "{path}" has more than one name; a relative path is a child's name, or "../" and a sibling's name` |
| `'..'` with no name after it | `section path "{path}" needs a name after ".."` |
| `..` in a path from the top, as in `'/Main/../Nums'` | `section path "{path}" uses ".."; a path from the top takes none` |
| A name that ends in an index, as in `'Kid[1]'` | `section path "{path}" has an index; a path in a prompt takes none` |
| A backslash before any other character, as in `[[a\b]]` | `section path "{path}" has an unknown escape "\b"; the escapes are "\/", "\\", and "\["` |
| A backslash at the end of the path | `section path "{path}" ends in a backslash` |
| A path from the top given to `call`, `jump`, `tasks.spawn`, or `tasks.fanout` | `section path "{path}" starts with "/"; only require, prompt.section, and prompt.section_or take a path from the top` |
| A relative path given to `require` | `section path "{path}" does not start with "/"; require takes a path from the top, such as "/helpers"` |

`require` has its own message for a path that is not a string: `require takes a section path such as '/helpers', got {type}`.

A well-formed path that names no section is not malformed. It is a miss, and [The not-found error](08-jump-and-call.md#the-not-found-error) covers what each call does with one.
