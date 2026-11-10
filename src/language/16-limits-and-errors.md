# Limits and errors

This chapter gathers every run limit, default, failure kind, and error code in one place, and shows how to read an error to find what went wrong and where. It covers what each limit allows and who can change it, the kinds a failure has in the parser and in Lua, the code a failed run reports, and how a cancel or a stop ends a run. With this vocabulary you can keep a prompt inside its budgets, catch the failures it can recover from, and go straight to the line that broke.

## Limits and failures at a glance

After this section you can name the run limits and the failure families, and you know which part of a failure to branch on.

````markdown
---
name: steady
description: Finishes well inside every run limit.
runvil: 0
---

# Steady

## Finish

```lua
return 'finished'
```
````

The run result is:

````text
finished
````

Even this small run observes a full set of run limits. The run limits are the bounds every run works within: how many model rounds one loop call may take, how many tasks run at once, how many tasks wait idle after a reply, and how much memory and logging each section's Lua may use. A model round is one request to a model and its reply. One set of run limits holds across the whole run, and those limits are the built-in defaults unless the [Harness](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) that runs the prompt sets its own values for that run. How large a model reply may be and how long a wait for a model reply may last are not run limits but request limits, which the gateway client applies to each model request.

### The run limits in brief

Each run limit has its own default and its own owner, and [Run limits and their defaults](#run-limits-and-their-defaults) gives the numbers:

- The [round cap](11-conversations.md#tool-rounds-and-the-round-cap), the most model rounds one `models.loop` call takes.
- The [concurrency limit](14-fanout.md#concurrency-and-the-admission-window), the most tasks that run at once.
- The [idle cap](15-tasks.md#the-idle-cap), the most tasks the run holds idle after a `tasks.reply`.
- The memory ceiling, the most Lua heap each [section VM](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals) may use.
- The log event quota, the most `log` calls each section VM records, together with the log byte quota that follows from it.

The same section gives the defaults of the two request limits the gateway client applies to each model request:

- The response cap, the largest model reply the client accepts.
- The receive timeout, the longest wait for the next part of a model reply.

### The failure families

A failure belongs to one of four failure families, and each family has its own vocabulary:

- A parse failure is found before anything runs, and it has a parse error kind: `Frontmatter`, `Structure`, `Fence`, `List`, or `Lua`, plus `Internal` for a fault in the parser itself. See [Parse errors and their kinds](#parse-errors-and-their-kinds).
- A failure inside Lua is an [error value](05-lua-environment.md#error-values), the table an Engine failure raises in Lua, whose `kind` is one of sixteen lowercase error kinds, such as `lua`, `tool_loop_exhausted`, or `model_failure`, and whose `code` names the exact failure. See [Error kinds in Lua and the run error they become](#error-kinds-in-lua-and-the-run-error-they-become).
- A failed run ends with an error whose [error code](04-how-a-prompt-runs.md#the-run-result) is the stable name of what failed, such as `lua_runtime` or `not_found`. See [Run error codes](#run-error-codes).
- A Host cancel ends the run with the canceled error, whose code is `canceled`, and Workshop's run log records it as `canceled`, not `failed`. See [Cancellation and stops](#cancellation-and-stops).

A run ends with its result: the final text when it completes, as `steady` does, or the error that ended it. A cancel is the error whose code is `canceled`, and a run that ends with any other error has failed.

### What a run error carries

A run that fails ends with a run error, the error that ended it. A run error is the same universal error that a caught error value stands for in Lua, and it has five parts: a code, a message, prose, an optional location, and facts. The code names what failed, and every Engine failure has exactly one code, such as `lua_runtime` or `not_found`. The message is the human-readable account of what went wrong, the failing error's own text with the text of its causes. The prose is the text a model reads for the failure, the location is where the failure happened when that is known, and the facts are the values the failure names, such as a path or a task id. Together the code and the message form the typed contract of a failed run: the code is exact, and each message always contains a fixed fragment that names the failure.

A run error carries nothing else. Whether a failure is retryable, meaning it may be transient so that running the prompt again could succeed, follows from its code: only the model codes for transient transport or backend problems, such as `overloaded` and `timeout`, are retryable. [Retryable failures](#retryable-failures) lists which failures count.

A failed run's message is its error's located message, so where a failure points at a place, its message says so. A `structure_invalid`, `fence_invalid`, or `list_empty_item` failure ends its message with ` (prompt {name}, line {line}, column {column})`, a `frontmatter_invalid` failure ends it with ` (line {line}, column {column})`, and every other located failure ends it with ` (at {file}:{line})`, the prompt file and line, or for an `internal` fault the Engine source file and line. [Where an error points](#where-an-error-points) covers each form, the location a caught error value's message starts with, and the location fields a parse error itself reports.

### Branching on a failure

Each part of a run error has its own job. Use the code to branch on what failed, and the message to show a reader what happened and where. A block that catches the failure sees the same code as `err.code`.

The [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) decides how it shows the code and the message to the person running the prompt, and how it shows a parse error's location fields. Workshop's own prompt check, which parses a prompt outside any run, reports a parse failure under a code named by its parse error kind, one of `parse_frontmatter`, `parse_structure`, `parse_fence`, `parse_list`, `parse_lua`, or `parse_internal`, and it prefixes `line {line}: ` to the message of every located parse failure except an `Internal` one. The Harness hands the Host each run's result, and Workshop's run log stores each run's status, `completed`, `failed`, or `canceled`, with the code tag and the located message of the error that ended it.

## Run limits and their defaults

After this section you can give each run limit its default, set the one limit a prompt controls, and tell which limits only the Harness changes.

````markdown
---
name: capped
description: Searches with a tight round cap.
runvil: 0
max_tool_iterations: 3
uses:
  - web
---

# Capped

## Ask

```lua
tools.offer('web/search')
messages:user('Find the tallest building in Paris, then answer in one sentence.')
local ok, err = pcall(models.loop)
if not ok then
  return err.kind .. ': ' .. err.message
end
return messages[#messages].content
```
````

When the model keeps calling the search tool through all three rounds, the run result is:

````text
tool_loop_exhausted: capped.md:17: The tool-call limit was reached before a final reply. Stop calling tools and reply with what you have.
````

When the model answers with text within its three rounds, the run result is that reply.

The `max_tool_iterations: 3` line sets this prompt's round cap to 3, so the [`models.loop`](11-conversations.md#starting-a-conversation) call in `## Ask` takes at most three model rounds. Reaching the round cap fails the loop call with an error value whose `kind` is `tool_loop_exhausted`, and the block catches it with [`pcall`](05-lua-environment.md#raising-and-catching-errors) and returns its kind and its message. The message starts with `capped.md:17: `, the line of the loop call, followed by the failure's prose (see [Where an error points](#where-an-error-points)).

### The five run limits

A run observes five limits, each with a default:

| Limit | Default | Applies to | Who changes it |
|---|---|---|---|
| Round cap | 24 model rounds | each `models.loop` call | the prompt's `max_tool_iterations`, or the Harness |
| Concurrency limit | 8 tasks at once | every task the run admits, fanout arms included | the Harness, and `tasks.concurrency` lowers it for a chain's own spawns |
| Idle cap | 8 idle tasks | every task idle after a `tasks.reply`, across the whole run | the Harness, as `max_idle_tasks` |
| Memory ceiling | 64 MiB | the Lua heap of each section VM | the Harness |
| Log event quota | 1024 `log` events | each section VM | the Harness |

These built-in values are the default limits, the set every run starts from. The Engine accepts other limits from the code that starts a run, and the Harness passes it the limits set for the run, which are these defaults unless the Host sets others. Workshop sets none, so every Workshop run observes these defaults. One set of limits applies to the whole run, the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) and every section included.

Every count and size limit of a run is at least 1, so the round cap, the concurrency limit, the idle cap, the memory ceiling, and the log event quota can never be zero.

The gateway client also applies two request limits to each model request, each with a default:

| Limit | Default | Applies to | Who changes it |
|---|---|---|---|
| Response cap | 16 MiB | each model reply | the Harness |
| Receive timeout | 120 seconds | each wait for the next part of a model reply | the Harness |

The response cap is at least 1 byte, so it can never be zero.

### The one limit a prompt sets

`max_tool_iterations` is the only run limit a prompt's frontmatter can change (see [Optional keys and unknown keys](02-file-structure.md#optional-keys-and-unknown-keys)), and the other four run limits come from the Harness that runs the prompt. A prompt's own `max_tool_iterations` value replaces the run's round cap. Without the key, the run's cap applies, which is 24 unless the Harness sets another.

The round cap allows 24 model rounds by default in each `models.loop` call. Each call starts its own count at zero, so two loop calls in one section can each use up to the full cap.

Reaching the round cap fails the loop call with an error value whose `kind` is `tool_loop_exhausted`, as `capped` shows. Without a `max_tool_iterations` key, this happens after exactly the run's default number of model rounds. Left uncaught, that error ends the run with code `tool_loop_exhausted` and the message `tool-call loop did not converge (at {file}:{line})`, which names the line of the loop call.

### Limits the Harness sets

The concurrency limit caps how many tasks run at once across the whole run, 8 by default. The Harness sets it, and no frontmatter key changes it. By default a run admits at most 8 tasks at once, fanout arms included; the Harness can set a different limit, and `tasks.concurrency` lowers it for a chain's own spawns.

The idle cap bounds how many tasks the whole run holds idle after a `tasks.reply`, 8 by default. Each idle task keeps its Lua VM, so the cap bounds the memory idle tasks hold. The Harness sets it as `max_idle_tasks`, and no frontmatter key changes it. One more reply past the cap closes another idle task, the one idle longest that owns no live task, or the one idle longest when each owns one. The closed task ends as if it had returned its latest reply, and its own live tasks are abandoned with reason `owner_returned` (see [The idle cap](15-tasks.md#the-idle-cap)).

The response cap and the receive timeout are request limits, not run limits: the gateway client applies them to each model request. Each model reply may be at most 16 MiB by default, and each wait for the next piece of a model reply times out after 120 seconds by default. No frontmatter key changes either. [Model request limits and failures](#model-request-limits-and-failures) shows how both bound a model round.

The memory ceiling and the log event quota bound each section entry's Lua, and [Lua memory and log quotas](#lua-memory-and-log-quotas) covers them.

### The call depth cap is not a run limit

The call depth cap of 8 is a fixed Engine constant, not one of the five run limits, so neither the Harness nor a prompt's frontmatter can change it. [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap) covers how that cap works.

## Lua memory and log quotas

After this section you can predict when a block runs out of memory or log quota, read the error it raises, and rely on long loops running to the end.

````markdown
---
name: chatty
description: Logs more checkpoints than the quota allows.
runvil: 0
---

# Chatty

## Talk

```lua
local ok, err = pcall(function()
  for i = 1, 2000 do log('step ' .. i) end
end)
if not ok then
  return err.kind .. ': ' .. err.message
end
return 'every step logged'
```
````

The run result is:

````text
lua: chatty.md:13: lua log event quota exceeded. The section logged more than its budget allows. Log less.
````

Each [`log`](05-lua-environment.md#logging-and-lifecycle-reports) call records one checkpoint, and each section VM records up to 1024 of them by default. The loop asks for 2000, so the 1025th call is refused. The refusal raises an error value of kind `lua`, `pcall` catches it, and the block returns `err.kind .. ': ' .. err.message`. The message names the line of the refused `log` call, `chatty.md:13`, then what failed, `lua log event quota exceeded`, then the default text for an exhausted quota. A section VM is the fresh Lua state that one section entry's blocks run in, and every limit in this section is counted per section VM.

### The memory ceiling

The memory ceiling is the most Lua heap a section VM may use. A block that allocates past it fails, so with a Harness ceiling of 4 MiB, a block that keeps allocating hits the ceiling and fails. A section VM runs its Lua under a default memory limit of 64 MiB, applied when the VM is built.

Each section VM may use at most 64 MiB (67,108,864 bytes) of Lua heap by default, and only the Harness sets this ceiling. Running out of it is an ordinary Lua error, not a quota error, because only the two log quotas raise quota errors. The ceiling also sets the size budget that `json.encode`, `json.encode_lines`, and a `var` write spend, as [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) describes.

### The log event quota and the log byte quota

The log event quota is the most `log` events a section VM's Lua can emit, 1024 by default. Each section VM records up to 1024 `log` checkpoints by default, every one-argument `log` call spends one, and only the Harness can change the count.

The log byte quota bounds the total size of those messages. Each section VM has 256 bytes of log byte quota for each `log` call the log event quota allows, so 262,144 bytes by default, counted in UTF-8 bytes across all of that VM's `log` calls. The Harness has no separate setting for the log byte quota, and changes it only by changing the log event quota.

### Each section VM gets its own limits

The memory ceiling and the log quotas apply to each section VM, not to the whole run, because every section entry builds a fresh VM and applies the limits to it. A section entry that spends most of its log events leaves the next entry's allowance untouched.

### No instruction limit

````markdown
---
name: counter
description: Counts to eight million.
runvil: 0
---

# Counter

## Count

```lua
local n = 0
for i = 1, 8000000 do n = n + 1 end
return n
```
````

The run result is:

````text
8000000
````

A Lua block has no instruction-count limit, so it runs until it finishes, runs past the memory ceiling, runs out a log quota, or the run is canceled. The eight-million-iteration loop in `counter` runs to the end and returns its count, and a long-running block like it completes instead of failing with a quota error, because no instruction ceiling applies.

No limit counts Lua instructions. The instruction count has no cap, and it is fixed rather than set by the Harness. Only the cancel check counts instructions, and it runs every 10,000 of them.

A runaway loop such as `while true do end` is legal too. It keeps running until a Host cancel stops it, within about 10,000 instructions, and in a run that is already canceled the same loop stops at the first check. A Host cancel reaches every Lua block, a busy loop included, because each block gets its own cancel check, which fires every 10,000 instructions. So a block can run a long or infinite loop without hitting any instruction limit, and the run's cancel is the kill switch for a runaway loop (see [Cancellation and stops](#cancellation-and-stops)).

### Which budgets raise quota errors

Only the two log quotas raise quota errors. Running past the memory ceiling raises an ordinary Lua error, the same as any other failure, and the instruction count has no budget.

Caught as an error value, an exhausted Engine quota on Lua has kind `lua` and code `lua_quota`. Left uncaught, running out of the log event quota or the log byte quota ends the run with code `lua_quota`, with the message `lua log event quota exceeded` or `lua log byte quota exceeded`, followed by ` (at {file}:{line})`. Among the Lua budgets, only the two log quotas lead to `lua_quota`: running past the memory ceiling leads to `lua_runtime`, and no instruction count can run out.

### Setting the memory ceiling and the log event quota

The memory ceiling and the log event quota are separate Harness settings, each with its own default, so setting one leaves the other at its default. A Harness can set a log event quota of 1 and keep 64 MiB of memory, or set a 4 MiB ceiling and keep 1024 log events.

The memory ceiling is set in bytes, and it applies to each Lua VM separately rather than as one pool shared across the run. Each section VM starts with its own full allowance of memory, log events, and log bytes.

The memory ceiling and both log quotas apply right after each section VM is built, before any setup step, so they cover every block from its first instruction, [library](03-blocks-and-prose.md#sharing-code-with-require) code included. The H1 pass's VM gets the same limits right after it is built.

The Harness sets the log event quota alongside the Lua memory ceiling, for example a 4-event quota with the default memory. The configured log event quota governs each section VM from its first block, so `log` calls in the first block already count against the configured limit rather than the defaults.

### Running past the memory ceiling

````markdown
---
name: hoarder
description: Allocates until the memory ceiling stops it.
runvil: 0
---

# Hoarder

## Grow

```lua
local t, i = {}, 1
while true do t[i] = string.rep('x', 16384) i = i + 1 end
```
````

The loop never ends on its own, yet it does not hang the run, because the memory ceiling stops it with an error. The run fails with code `lua_runtime`, and its message contains the word `memory`.

An allocation past the memory ceiling is refused as an ordinary Lua error whose message mentions memory; it is not a quota error. The section VM is still torn down normally after a memory-ceiling error.

A memory-ceiling refusal raised inside an Engine function reaches `pcall` as an error value of kind `lua`, so `pcall` returns `false` and that error value. A refusal raised by Lua's own code, such as the `string.rep` calls in `hoarder`, is not turned into an error value, so a block that catches it should not count on reading `err.kind`. Left uncaught in a section, a memory-ceiling error ends the run with code `lua_runtime`. Left uncaught in the H1 pass, it ends the run with code `requirements_unmet` instead (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)), and the message is the H1 gate notice followed by the Lua error text and then ` (at {file}:{line})`, the location of the failed gate. The notice reads:

````text
the environment cannot satisfy this prompt:
- the H1 gate failed
````

### Running out a log quota

Each `log` call that passes the argument-count check spends one log event before any other check, so a call later rejected for its type, length, or characters still uses up an event. Once the log event quota is spent, every further one-argument `log` call raises a log event quota error and records nothing.

A `log` message whose UTF-8 byte length exceeds what remains of the log byte quota raises a log byte quota error and is not recorded. Such a message is refused whole, while the messages logged before it stay recorded. With a log event quota of 4, and so a log byte quota of 1024 bytes, these three calls of 400 bytes each fail on the third and leave exactly two checkpoints:

````lua
log(string.rep('é', 200))
log(string.rep('é', 200))
log(string.rep('é', 200))
````

A `log` call past either quota raises an error instead of recording a checkpoint. The error names what failed in one of two texts:

| Quota | What failed |
|---|---|
| Log event quota | `lua log event quota exceeded` |
| Log byte quota | `lua log byte quota exceeded` |

Caught with `pcall`, the error value has kind `lua` and the message `{file}:{line}: {text}. The section logged more than its budget allows. Log less.`, where `{text}` is one of the two and the line is that of the refused `log` call. Left uncaught, the run ends with code `lua_quota` and the message `{text} (at {file}:{line})`, which names the same line.

With a Harness log event quota of 1, a library that calls `log('one')` and then `log('two')` fails on the second call, because the quota applies before the library code runs, and left uncaught the run ends with `lua log event quota exceeded (at {file}:{line})`, at the library's `log('two')` line. The `chatty` prompt at the start of this section shows the caught form: under the default quota of 1024, its 1025th call is refused, and returning `err.kind .. ': ' .. err.message` makes the run result `lua: chatty.md:13: lua log event quota exceeded. The section logged more than its budget allows. Log less.`

### Reading a quota error

Left uncaught, an exhausted log event quota ends the run with code `lua_quota`, and the remedy is to raise the matching ceiling or change the prompt so it needs less. A log quota failure in the H1 pass still ends the run with `lua_quota`, unlike a Lua runtime error there, which becomes `requirements_unmet`. The uncaught message ends with ` (at {file}:{line})`, the line of the refused `log` call.

Left uncaught, every Lua quota error ends the run with code `lua_quota`, and the only quotas a block can run out are the log event quota and the log byte quota. Caught as an error value, a Lua quota error has kind `lua`, the same kind as an ordinary Lua runtime error, rather than a kind of its own, and its `code` is `lua_quota` rather than `lua_runtime` (see [Error values](05-lua-environment.md#error-values)). The error value has no field naming which quota ran out; only its message tells the two apart, by `lua log event quota exceeded` or `lua log byte quota exceeded`.

A Lua quota failure does record which quota ran out, and its message names it as `log event` or `log byte` in the form `lua {resource} quota exceeded`, caught or not. A quota refusal that ends a block always surfaces as the quota error itself, ahead of any error value the block kept or raised.

A `log` call that exceeds the log event quota fails the block with a log event quota error, and the Engine's refusal reads `lua log event quota exceeded`. Exceeding the log byte quota fails the block with a log byte quota error, a stable quota failure rather than a plain Lua error string. Raising the exact text of a refusal yourself, as in `error('lua log event quota exceeded')`, still produces an ordinary runtime error, because only the Engine's own refusal counts as a quota error.

## Parse errors and their kinds

After this section you can tell the parse error kinds apart, cause each one with a small file, and read the message it gives.

````markdown
---
name: transform
description: Loads text, then rewrites it.
runvil: 0
---

# Transform

## Load

```lua
var.text = 'hello'
```

## Transform

Rewrite the text in capitals.

```lua
return string.upper(var.text
```
````

Nothing in this prompt runs. Its parse fails with parse error kind `Lua` and this message:

````text
lua compilation error at section `Transform` epilog (line 20): {message}
````

Here line 20 is the file line where the block's Lua starts, and `{message}` is `syntax error: ` followed by the Lua compiler's own text. The parse error also points at line 20, the line the compiler's diagnostic names, and names the prompt `transform` (see [Lua compile failure positions](#lua-compile-failure-positions)).

The whole prompt file is checked before anything runs. The parser validates the frontmatter, the headings, the fences, and the sections, and it compiles every Lua block in the H1 body and in every section, so a structural error or a Lua syntax error anywhere in the file, even in the last section, stops the prompt before any section runs. A parse error rejects the whole prompt file, even when every other section in it is valid, as `## Load` is here.

A prompt file that fails to parse is rejected with a parse error, and that parse error carries a parse error kind, the stable name of the class of rule the file broke. A parse failure shows up when the prompt file is parsed, before any part of it runs, as one error with a stable kind and optional location fields, such as the line and column of the problem.

### The six parse error kinds

Every parse failure has exactly one of six parse error kinds:

| Parse error kind | What it means |
|---|---|
| `Frontmatter` | the frontmatter block is missing, unclosed, or not valid YAML, or the prompt contract rejects a key or value in it |
| `Structure` | the H1 title or the heading tree is invalid |
| `Fence` | a `lua` fence is left open, or a fence or line breaks the strict Lua fence rule |
| `List` | a list section has an empty item |
| `Lua` | a Lua block does not compile |
| `Internal` | the parser fails through no fault of the prompt |

Common mistakes land in predictable parse error kinds. Malformed YAML and a `max_tool_iterations` value outside 1 to 1000 give `Frontmatter`, two sibling headings with the same name give `Structure`, an empty list item gives `List`, and a Lua syntax error gives `Lua`. Every parse error carries a kind, and the kinds you run into while writing include `Structure`, `Fence`, `Lua`, and `List`.

The set of parse error kinds is open, so a Host program that branches on a parse error's kind needs a catch-all case for kinds it does not know.

A parse error's message names the specific rule the file broke, so you see which rule failed. Each kind of mistake keeps a stable key phrase in its message, such as `requires an H1`, `unrecognized fence label`, or ``section `Transform` epilog``.

### What each kind covers

Every frontmatter failure has parse error kind `Frontmatter`. The block is missing, left unclosed, or not valid YAML, or the prompt contract rejects a key or value in it, such as an unknown or misspelled key, an out-of-range `max_tool_iterations`, a `uses:` entry that is neither a Pack id nor a `vendor/name`, or a Pack id declared twice. [Optional keys and unknown keys](02-file-structure.md#optional-keys-and-unknown-keys) lists the keys the contract accepts.

Parse error kind `Structure` means the [H1 title](02-file-structure.md#the-h1-title-and-h1-body) or the [heading tree](02-file-structure.md#section-headings-and-nesting) is invalid: the H1 is missing, repeated, or empty, a section heading skips a level (such as an H4 directly under an H2) or is empty, or two sibling sections share a name.

Parse error kind `Fence` means a `lua` fence is left open, or a fence or line anywhere in the body breaks the strict Lua fence rule (see [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors)) through a bad label, a bad opening line, a bad closing line, or a lookalike line, and every such break reports `Fence`. A `lua` fence that never gets a closing line, or whose closing line is not exactly three backticks, fails the parse with `Fence`.

Parse error kind `List` means a [list section](03-blocks-and-prose.md#list-sections) has an empty item: a marker such as `-` or `1.` with no text after it.

Parse error kind `Lua` means a Lua block does not compile because of a syntax error, whether in the H1 body, a section block such as an epilog, or a `lua` fence between prose passages. It fails the whole parse before the prompt runs, with this message:

````text
lua compilation error at {location} (line {source_line}): {message}
````

`{location}` names the block, `{source_line}` is the file line where the block's Lua starts, and `{message}` is `syntax error: ` followed by the Lua compiler's own text. A section block's location takes one of the first six forms below, and a block in the H1 body takes the last form, with the H1 title. The `lua {ordinal}` forms name a numbered block between the prologue and the epilog, and the three forms with `(line {heading_abs_line})` name a block of a section whose section name another section heading also has, by the file line of the section's own heading (see [Sections that share a section name](05-lua-environment.md#sections-that-share-a-section-name)):

````text
section `{name}` prologue
section `{name}` epilog
section `{name}` lua {ordinal}
section `{name}` (line {heading_abs_line}) prologue
section `{name}` (line {heading_abs_line}) epilog
section `{name}` (line {heading_abs_line}) lua {ordinal}
H1 `{title}` lua
````

### Which problem the parser reports

The frontmatter is decoded before the body is checked, so a frontmatter failure is reported ahead of any body failure. When a prompt has several problems, the parse reports only the first one it meets, in this order:

1. The frontmatter delimiters.
2. YAML decoding and the contract checks inside it.
3. Duplicate Pack ids.
4. Reserved Pack ids.
5. The H1's presence and count.
6. An empty H1 title.
7. The fence rule.
8. The H1's blocks and their Lua.
9. Each section, in document order.

Only the first fence error in a file is reported, so a later offending fence is not reported.

A parse error's message already contains the underlying cause, such as the YAML or Lua error text, and each part appears only once, so the cause text never shows twice, even when a printer walks the error's chain of causes.

### Frontmatter messages

A file whose first line is not `---` fails with `Frontmatter` and the first message below, and a frontmatter block with no closing `---` line fails with `Frontmatter` and the second:

````text
file must begin with a --- frontmatter delimiter
frontmatter was not closed with ---
````

This file never closes its frontmatter, so it gives the second message:

````markdown
---
name: unclosed
description: Never closes its frontmatter.
runvil: 0

# Unclosed
````

Frontmatter that is not valid YAML fails with `Frontmatter` and the message `invalid frontmatter: {message}`. Here `{message}` is the YAML decoder's own readable diagnostic, never a dump of the raw source, given with its line and column when the decoder reports them.

Once the frontmatter block is present and closed, every frontmatter failure, whether the YAML decoder, a contract check, or the duplicate Pack id check rejects it, has a message that starts with `invalid frontmatter:` followed by the specific problem. Declaring the same Pack id twice under `uses:` shows the pattern:

````markdown
---
name: twice
description: Declares one Pack two times.
runvil: 0
uses:
  - web
  - web
---

# Twice
````

````text
invalid frontmatter: Pack id web is declared more than once under uses
````

A frontmatter key or value the prompt contract rejects fails with `Frontmatter` and that key's own message, with the line and column. That covers an unknown entry field, a bad arg name, type, or default, a malformed `uses:` entry, a `models:` key that is not one of the five [model roles](10-models.md#declaring-roles-under-models), an unknown field in a role declaration, a non-boolean `thinking`, a zero `min_context`, and an out-of-range `max_tool_iterations`. A role name outside the five gives a message that contains ``expected one of `default`, `strong`, `small`, `fast`, `creative` ``, and an unknown field in a role declaration, such as `tier`, gives one that contains ``unknown field `tier`, expected `min_context` or `thinking` ``.

A file whose frontmatter has no `runvil:` key still parses, but running it fails before any section runs, with code `structure_invalid` and a message that ends with the prompt's name:

````text
not a runvil prompt: no runvil version (prompt {name})
````

### Heading messages

Each `Structure` failure has its own message:

````text
prompt requires an H1 title
prompt must contain exactly one H1 title, found {count} at lines {lines}
prompt H1 title must not be empty
section `{name}` is an orphan H{level} heading with no parent H{n}
an H{level} section heading must not be empty
duplicate sibling section name `{name}`: first declared at line {first_line}, again at line {heading_abs_line}; sibling section names must be unique
````

A file with no H1 title gives `prompt requires an H1 title`:

````markdown
---
name: untitled
description: Has no H1 title.
runvil: 0
---

## Step

Do the work.
````

A file with more than one H1 title gets a message that lists the line of every H1, such as `prompt must contain exactly one H1 title, found 3 at lines 6, 12, 16`. An empty H1 title gives `prompt H1 title must not be empty`.

A heading with no parent one level up gives the orphan message, which names the section and the heading level, so an H4 named `Detail` directly under an H2 gives ``section `Detail` is an orphan H4 heading with no parent H3``. An empty section heading gives the empty-heading message with its level. Two sibling sections with the same name give the duplicate message, which names both lines, and [Where an error points](#where-an-error-points) walks through one.

A parse failure's kind is stored with the error, never worked out from its message text, so the kind stays stable even when the message wording changes.

### Fence messages

An unclosed `lua` fence fails with `Fence` and the message `{label} fence is not closed`:

````markdown
---
name: open
description: Leaves a fence open.
runvil: 0
---

# Open

## S

```lua
return 1
````

````text
section `S` prologue `lua` fence is not closed
````

The `{label}` depends on where the fence sits. It is the first form below for a fence that opens its section and the second for any other, and `{section}` is the H1 title for a fence in the H1 body:

````text
section `{section}` prologue `lua`
section `{section}` epilog `lua`
````

A line of four spaces followed by three backticks, or three backticks followed by ` x`, does not close the fence. It leaves the fence open, so it also gives ``section `S` prologue `lua` fence is not closed``.

Each break of the strict Lua fence rule has its own message, reported at the offending line:

````text
unrecognized fence label `{label}`; the only Lua fence label is `lua`
`lua` fence opening line must be exactly ```lua
`lua` fence closing line must be exactly ```
````

A bad label gives the first message, a bad opening line or a lookalike line gives the second, and a bad closing line gives the third. When a file has several breaks, the earliest one wins.

Lines inside the body of another fenced code block are never checked against the Lua fence rule. A longer fence, such as one opened with four backticks, can therefore show a `lua` fence as an example without a `Fence` error, and the inner fence stays prose.

### List messages

A list section with an empty item, one that has no text, fails with parse error kind `List` and a message that names the section:

````markdown
---
name: steps
description: Lists three steps, one of them empty.
runvil: 0
---

# Steps

## Plan

- alpha
-
- gamma
````

````text
empty bullet item in list section `Plan`
````

The `List` kind is reported the same way whatever the section is named, including sections named `frontmatter` or `fence`. A section counts as a list section only when it has no Lua and every nonblank line is a list marker, so a section that mixes prose with bullet lines is ordinary prose and raises no `List` error.

### Internal parser faults

Parse error kind `Internal` means the parser fails through no fault of the prompt: one of its own internal checks breaks, or the Lua compiler cannot start. When the compiler cannot start, compiling a block fails with `Internal` instead of `Lua`.

## Where an error points

After this section you can find the prompt line an error points at, whether a block caught it, it ended the run, or the parser found it, and read the form each location takes.

````markdown
---
name: pointer
description: Reads a sys field that does not exist.
runvil: 0
---

# Pointer

## Read

```lua
local ok, err = pcall(function()
  return sys.nothing
end)
return tostring(err)
```
````

The run result is:

````text
pointer.md:13: unknown sys field 'nothing'. The Lua code failed at runtime. Fix the code at this location.
````

The read of `sys.nothing` on line 13 fails, and `pcall` hands back an error value whose message starts with the file and line of that read, `pointer.md:13`. After the location comes the error's prose: what failed, `unknown sys field 'nothing'`, then the default text for a failed Lua block. Replace the block with one that runs `return sys.nothing` on line 12 and nothing else, and the run fails with code `lua_runtime` and a message that ends with ` (at pointer.md:12)`.

An error points at a place in one of three forms:

- A caught error value's message starts with `{file}:{line}: `, as `pointer` shows. [Where a caught error's message points](#where-a-caught-errors-message-points) gives the rules.
- A failed run's message ends with the location in parentheses, such as ` (at {file}:{line})`. [Where a run error's message points](#where-a-run-errors-message-points) gives each form.
- A parse error has location fields beside its message. [Location fields](#location-fields) lists them.

`{file}` is the prompt's file as the Host names it, or `prompt` when the Host names none. In this guide's examples, a prompt's file is named after its frontmatter `name`, so the `pointer` prompt is `pointer.md`. `{line}` counts from the top of that file, the frontmatter included, so the opening `---` is line 1 (see [Counting lines from the top of the file](#counting-lines-from-the-top-of-the-file)). An example that shows a whole prompt gives the real line, and one that shows only part of a prompt writes `{file}:{line}`.

### Where a caught error's message points

An error value built from an Engine failure has a `message`, and `tostring` returns that message. The message is `{file}:{line}: {prose}` when the failure is located, and the prose alone otherwise. The line is the author's line that made the failing call:

- For a call that waits on the Harness, such as `models.infer`, `store.read`, `call`, `tasks.spawn`, or `tasks.fanout`, it is the line that made the call.
- For an Engine function that fails without waiting, such as `jump`, it is the line running when it failed.
- When `pcall` or `xpcall` catches the failure, it is the line of the failing call inside the protected function, not the line of the `pcall`, so `pointer` reports line 13 rather than line 12.
- A main chunk's `return f()` keeps its own line, while a tail call inside a function you defined reports the line that called that function.

A failure handed back from a chain that `call` or a spawned task ran keeps the line where it happened in that chain.

After the location comes the prose, the text a model reads when the failure reaches one. A failure that the prompt's own Lua caused has author prose, `{what}. {default prose}`: what failed, then the fixed text for that kind of failure. That covers Lua errors, an exhausted log quota, the call depth cap, a failed `{{ }}` substitution, a task used by a chain that does not own it or waited on twice, and a chain that ends with tasks still running and not idle. The prose itself never holds the file or the line.

Every other failure, such as a tool, store, model, or context failure, keeps its own prose, and it gains the `{file}:{line}: ` prefix in Lua when it is located. A model never sees that prefix: when a tool fails during `models.loop`, the failure reaches the model as its prose alone.

Some error values point at no line. An Engine fault, which reaches Lua as kind `internal`, never shows a location in Lua, because its location is a line of Engine source. An argument error that a Lua shim raises itself, such as `models.infer takes (prompt) or (model, prompt)` for a call with a third argument, comes back as written, with no location and no added prose. A string you raise with `error` is not an error value at all and keeps Lua's own form (see [Where a library error points](03-blocks-and-prose.md#where-a-library-error-points)).

### Location fields

When the parser can point at the problem, a parse error has location fields, and a Host can report them beside the message. This prompt repeats a section name:

````markdown
---
name: dup
description: Repeats a section name.
runvil: 0
---

# Dup

## S

First.

## S

Second.
````

The first `## S` is on line 9 and the second is on line 13, so the parse fails with parse error kind `Structure` and this message:

````text
duplicate sibling section name `S`: first declared at line 9, again at line 13; sibling section names must be unique
````

The parse error has these location fields:

| Field | Value |
|---|---|
| Name | `dup`, the prompt's frontmatter `name` |
| File | `dup.md`, the prompt's file |
| Line | 13 |
| Column | 1 |
| End line | 14 |
| End column | 1 |

The run fails with code `structure_invalid` before it starts, because the Harness parses the prompt first, and its message is the parse error's own text followed by its location form, ` (prompt dup, line 13, column 1)`. The location fields belong to the parse error itself, which a Host that parses the prompt, as Workshop does, can read beside the message. Workshop's own prompt check reports this failure as `line 13: duplicate sibling section name ...` under the code `parse_structure`.

A parse error's location fields are the prompt's name, the file, the line and column where the offending region starts, and the line and column just past where it ends, each when known. A failure that covers a whole line, such as the second `## S` heading, ends at column 1 of the next line, and a failure that the YAML decoder places at one spot starts and ends at that spot.

A parse error, whether it comes from the frontmatter or from the body, points into the prompt source, except for an `Internal` fault, whose location, when it has one, is the Engine source line of the fault. A parse error's line is the line in the prompt file as written, and you use it together with the column to point at the mistake.

Error locations report 1-based line numbers and 1-based column numbers. A failure's line and column are 1-based when known, and an internal fault's message names a line but never a column.

### Counting lines from the top of the file

Every location counts 1-based lines from the top of the prompt file, the frontmatter included, so the opening `---` frontmatter delimiter is line 1. That holds for a caught error value's message, a failed run's message, and a parse error's fields alike. A bad `uses:` entry on the fifth line of the file is reported at line 5, and with frontmatter on lines 1 to 4 a `## S` heading on file line 8 is reported at line 8. Line numbers in body parse errors count from the top of the file, with the frontmatter lines included, not from the start of the body.

### The prompt name

A parse error names the prompt by its frontmatter `name` when parsing gets that far, so you can tell which prompt failed. When the name is not known, the parse error has no name, and the Host can use its own label for the file.

Every failure found after the frontmatter, other than an internal fault, reports the prompt's frontmatter `name`, a `Lua` compile failure included, so a prompt with `name: dup` reports `dup`. A frontmatter failure and an internal fault report no name.

The `dup` prompt shows the whole case: parse error kind `Structure`, name `dup`, line 13, column 1, an end at line 14, column 1, and a failed run with code `structure_invalid` whose message ends with ` (prompt dup, line 13, column 1)`.

### Where a run error's message points

A run error has an optional location of its own, and a failed run's message is the error's located message: the failure's own text with the text of its causes, then where the failure happened:

- A `structure_invalid`, `fence_invalid`, or `list_empty_item` failure, the code of a `Structure`, `Fence`, or `List` parse failure, ends with ` (prompt {name}, line {line}, column {column})`, each part only when it is known.
- A `frontmatter_invalid` failure, the code of a `Frontmatter` parse failure, ends with ` (line {line}, column {column})`, with no prompt name.
- Every other located failure ends with ` (at {file}:{line})`: the prompt file and line, or for an `internal` fault the Engine source file and line of the broken invariant.

A failure with no location ends with its own text. A Lua runtime failure's text already names its place: it starts with a `{block}:{line}:` tag, where `{block}` is the block's location label, such as ``section `S` prologue`` (see [How errors name each block](03-blocks-and-prose.md#how-errors-name-each-block)), and a stack traceback follows it. The ` (at {file}:{line})` suffix comes after the traceback.

These forms hold for every failed run, a parse failure that the Harness finds before the run starts included, so that failure ends with its location form just like a parse failure that the run's first step finds. A Lua compile failure, code `lua_compile`, ends with ` (at {file}:{line})` on the line its diagnostic names, and the `dup` prompt's repeated heading ends with ` (prompt dup, line 13, column 1)`. The parse error's own location fields stay beside the message as well, for a Host such as Workshop to read (see [Location fields](#location-fields)). A model lookup failure, an input staging failure, or a `stalled` run has no location, so its message is its own text.

The run's own first step finds a few failures of its own. A missing `runvil:` key fails with `structure_invalid`, with the prompt's name but no line, so the `version-zero` prompt without its `runvil:` line fails with `not a runvil prompt: no runvil version (prompt version-zero)` (see [A missing version](02-file-structure.md#a-missing-version)). An unsupported `runvil:` version fails with `unsupported_version` and ends with ` (at {file}:{line})` on the `runvil:` line, as `future` shows in [Run error codes](#run-error-codes), and a requirements refusal ends the same way when it is located (see [The requirements notice](04-how-a-prompt-runs.md#the-requirements-notice)).

Each located failure points at one line:

- A failure that reaches Lua, caught or not: the line that [Where a caught error's message points](#where-a-caught-errors-message-points) describes.
- A chain that ends with tasks still running and not idle: the spawn line of the first task left running.
- A `jump` to a missing heading: the line of the `jump`.
- A failed [H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate): the line of the failure that tripped it. A failed `{{ }}` substitution in the H1 body does not fail the gate: it ends the run with `substitution`, at its placeholder's line.
- A claims conflict, a store call that cannot open its store view, and a malformed request that a block yields to the Engine: the line of the call.
- A failed `var` or `argv` read-back: the first Lua line of the last Lua block at or before the one running.
- A failed `{{ }}` substitution: its placeholder's line, in a section's prose or a library's.
- The call depth cap: the line of the `call`, `tasks.spawn`, or `tasks.fanout` that went too deep.
- An unsupported `runvil:` version: the `runvil:` line.
- A requirements refusal: the `uses:` or `models:` line of the first gap the prompt caused. A refusal whose every gap is the caller's has no location.
- An `internal` fault: the Engine source line of the fault.

A `stalled` run points at no line, because no one line caused it: its message names the tasks that wait on their inboxes instead (see [Run codes](#run-codes)).

A parse failure that happens before the prompt's `name` is known has no name.

### Frontmatter failure locations

A frontmatter failure carries no prompt name, because it happens before the prompt's `name` is read from the frontmatter itself. A missing delimiter, a YAML decode failure, and an error in the `uses:` list all report none, even one found after the YAML decodes, and the Host can use its own label for the prompt instead. A frontmatter syntax error reports no prompt name for the same reason. A refused `uses:` list holds the refused Pack id in its name field, never the prompt's name.

````markdown
---
name: broken
description: Has a YAML slip on line 4.
: : :
runvil: 0
---

# Broken
````

This file fails with parse error kind `Frontmatter` and `invalid frontmatter: {message}`, at line 4 with a column and no name.

A frontmatter YAML failure points at one spot rather than a region: its location starts and ends at the line and column the YAML error gives, or covers that whole line when the YAML error gives a line but no column. A frontmatter failure's line and column come from the YAML error. An invalid-YAML failure reports the 1-based file line that the YAML parser gave, and reports no line when the YAML error has no location.

The YAML decoder counts lines from the start of the frontmatter, so the reported line is the decoder's line plus one, to account for the opening `---`. The column passes through unchanged, and each is absent when the decoder gives no position or gives zero. The column counts characters, as the YAML parser does, unlike a body failure's byte column. That is why a YAML slip such as `: : :` on the fourth line of the file is reported at line 4, which is 1-based and counts the opening `---` as line 1. The 1-based file column of the YAML error comes alongside the line, when the column is known.

A frontmatter decode failure, whether invalid YAML or a value the contract rejects, gives a 1-based line and a 1-based column when the YAML decoder reports a position. For a rejected value, that position is where the value starts, so a rejection inside a contract key points at the line and column of the offending value instead of giving a bare message:

````markdown
---
name: helper
description: Declares an entry that is not a valid uses entry.
uses:
  - not a Pack id
runvil: 0
---

# Helper
````

The `uses:` entry fails on its own value, so the parse error's line points at that entry's own line in the prompt file, line 5, and its column is 5, where the value starts after the list marker. A rejected frontmatter value's column is 1-based and points at the value's first character, the `n` of `not`, not at the `-` marker and the spaces before it.

A Pack id declared twice under `uses:`, as in `twice`, or a Pack id the Engine reserves, points at the `uses:` line itself, at column 1, because that check compares the entries after the YAML decodes, when the only position left is the line that declares `uses:`. So `twice` reports line 5, and Workshop shows it as `line 5: invalid frontmatter: Pack id web is declared more than once under uses`. A missing opening `---` or an unclosed frontmatter block points at line 1, where the frontmatter opens or should open, at column 1, and it has no name; the messages read `file must begin with a --- frontmatter delimiter` and `frontmatter was not closed with ---`.

### Body failure locations

Any other parse failure's location comes from the offending region in the body: its line and column are where the region starts, and its end line and end column are just past where the region ends. A structural failure's line is the 1-based line in the whole file where the region starts, counting the frontmatter lines above the body.

A duplicate sibling heading error reports the 1-based line of the offending heading, counted from the top of the file including the frontmatter, so with frontmatter on lines 1 to 4 and the second `## S` on line 12, the error is at line 12. Its column is the 1-based column of the offending heading, which is column 1. Its location covers that whole heading line and ends at column 1 of the next line, as `dup` shows.

Line numbers inside heading error messages also count from the first line of the file, frontmatter included. With frontmatter on lines 1 to 4, H1 titles on lines 6, 12, and 16 give the message `prompt must contain exactly one H1 title, found 3 at lines 6, 12, 16`, and a repeated `## S` whose first copy is on line 8 gives a message containing `first declared at line 8` and `again at line 12`.

Orphan and empty heading errors report the heading's file-absolute line, column 1, and cover the heading line, the same as a duplicate sibling heading. Every heading `Structure` error, for an orphan, empty, or duplicate sibling heading, points at the heading itself. A repeated H1 title points at the line of the second H1, so the file with H1 titles on lines 6, 12, and 16 reports line 12, and an empty H1 title points at its own line, each at column 1.

A `Fence` error for a bad fence label reports the line where the offending fence opens, at column 1, so a fence whose opening line carries extra text after `lua` on line 12 is reported at line 12, column 1. A `Fence` error for a broken fence rule covers the text of the offending line, from column 1. A `lua` fence left open, which fails with the not-closed message, points at the line where the fence opens, at column 1, so the `open` prompt reports line 11.

A bad closing line error reports the line of the offending closing line, at column 1. In this file the `lua` fence opens on line 10 and a fence line indented two spaces closes it on line 13:

````markdown
---
name: closer
description: Closes a fence with an indented line.
runvil: 0
---

# Closer

## S
```lua
var.a = 1
var.b = 2
  ```
````

The error is at line 13, column 1, with the closing-line message:

````text
`lua` fence closing line must be exactly ```
````

A lookalike error reports the line of the lookalike line, at column 1. In this file the section body starts on line 10 with `Prose.` and a blank line, and a `lua` opener indented four spaces sits on line 12:

````markdown
---
name: lookalike
description: Indents a fence opener.
runvil: 0
---

# Lookalike

## S
Prose.

    ```lua
    return 1
    ```
````

The error is at line 12, column 1, with the opening-line message:

````text
`lua` fence opening line must be exactly ```lua
````

An empty list item points at its marker, so the bare `-` in the `steps` prompt is reported at line 12, column 1, and an indented marker's column is that of the marker itself, past its indent. The `List` message names the section as well (see [List messages](#list-messages)).

A Lua compile failure points at the line its compiler diagnostic names, as [Lua compile failure positions](#lua-compile-failure-positions) shows.

### Byte columns

A structural failure's column is a 1-based byte column, so each multibyte UTF-8 character before the error on that line counts as more than one column. A body failure's column counts bytes, so a multibyte UTF-8 character earlier on the line pushes it past the character count.

### Failures with no line or column

Few parse failures have no line or column. A file with no H1 title has no line to point at. Neither does a YAML error that the decoder gives no position for, nor a `uses:` refusal in frontmatter that declares `uses:` on no line of its own, such as one flow mapping that holds the whole frontmatter. A missing `runvil:` key, which the run's first step finds rather than the parser, has no line either, because the key is not in the file. A Host therefore shows the line and column when present, and otherwise the message.

An `Internal` parse failure, a fault in the parser itself, reports no name, and its location, when it has one, is the Engine source file and line of the fault, so it gives no prompt line or column. Workshop shows its message without a `line {line}: ` prefix.

### Lua compile failure positions

A Lua compile failure gives its position both in its message and in its location fields. The message has the form `lua compilation error at {location} (line {source_line}): {message}`, where `{source_line}` is the file line where the block's Lua starts and `{message}` is `syntax error: ` followed by the compiler's diagnostic word for word, which names the line within the block. The location fields name the prompt and point at the file line that the diagnostic names, at column 1, so `transform` reports line 20, and a syntax error on the third line of a block whose Lua starts on line 30 reports line 32. A body failure's message, by contrast, is the plain diagnostic with no added prefix, and the name, line, and column are separate details beside it.

### Where body positions are measured

The parser measures a body failure's region within the body after the frontmatter and any leading [byte-order mark](02-file-structure.md#frontmatter-delimiters-byte-order-mark-and-line-endings), with CRLF read as LF, and reports it as the file lines and byte columns where the region starts and ends. Use the line and column to find the failure in the original file.

Body line numbers continue past the frontmatter: the first line after the closing `---` is numbered one more than the frontmatter block's line count, both `---` delimiters included. With a frontmatter block on lines 1 to 4, the body's first line is line 5.

## Run error codes

After this section you can name the error code a failed run reports and the cause behind each one.

````markdown
---
name: future
description: Greets on a supported major version.
runvil: 0
---

# Future

## Greet

```lua
return 'hello'
```
````

The run result is:

````text
hello
````

Change the version line to `runvil: 1` and run the prompt again. The run fails with code `unsupported_version` and this message, which ends with the file and line of the `runvil:` key:

````text
unsupported runvil version: 1 (this build supports major 0) (at future.md:4)
````

An error code, as [The run result](04-how-a-prompt-runs.md#the-run-result) introduces it, is the stable name of what failed. The error that ends a failed run has exactly one, which classifies the failure by what failed, and a code goes by its tag, its name in snake_case, such as `unsupported_version` or `lua_runtime`. A block that catches the same failure reads that tag as `err.code`.

### The eight code families

Every code also has a number, and the number's hundreds place the code in one of eight code families:

| Code family | Codes | What causes them |
|---|---|---|
| Parse codes (1xx) | `frontmatter_invalid`, `structure_invalid`, `fence_invalid`, `list_empty_item`, `unsupported_version`, `invalid_value` | the prompt fails to parse, the `runvil:` key is missing or declares a major version other than `0`, or a value is missing or not valid |
| Lua codes (2xx) | `lua_runtime`, `lua_compile`, `lua_quota`, `call_depth_exceeded`, `substitution` | a section's Lua fails at run time or returns an unusable value, a Pack prelude cannot install, a Lua block does not compile, a log quota runs out, a call nests past the call depth cap, or a `{{ }}` substitution fails |
| Task codes (3xx) | `tasks_live`, `task_not_owned`, `task_consumed`, `task_canceled` | a chain ends with tasks still running and not idle, reaches for a task it does not own, or waits on a result twice, or a canceled task's error value is raised again |
| Model codes (4xx) | `context_overflow`, `rate_limited`, `quota_exhausted`, `overloaded`, `refused`, `timeout`, `transport`, `server_error`, `rejected`, `malformed_response`, `empty_reply`, `unavailable`, `invalid_parts`, `context_exhausted`, `model_required` | a model call fails or gets an empty reply, the Harness cannot fetch the Host's list of models, a request overflows the model's context window and compaction does not make it fit, or a model round runs when the Host gave no model |
| Tool codes (5xx) | `tool_invalid_arguments`, `tool_backend`, `tool_transport`, `tool_other`, `out_of_scope_tool`, `unbound_tool`, `model_only_tool`, `tool_loop_exhausted`, `tool_schema_invalid`, `tool_sets_invalid` | a tool fails, is out of scope or unknown, a script calls a model-only tool, the tool-call loop reaches its round cap, or a section entry's first loop round cannot freeze its tool sets |
| Store codes (6xx) | `not_found`, `already_exists`, `not_a_directory`, `is_a_directory`, `directory_not_empty`, `not_utf8`, `invalid_path`, `invalid_range`, `anchor`, `permission_denied`, `approval_required`, `scope_ended`, `read_only`, `outside_root`, `root_protected`, `no_store`, `cross_mount`, `io`, `unsupported`, `conflict`, `backend`, `determinism` | a `store` operation fails uncaught, the run declares no store, the store backend fails as the run starts, the store refuses to stage the input file, or a store call meets a claims conflict |
| Run codes (7xx) | `requirements_unmet`, `canceled`, `internal`, `stalled` | the environment falls short of what the prompt needs, the run is canceled, an Engine invariant breaks, or no chain can move |
| Caller codes (8xx) | `recorder_failed`, `plugin_load_failed`, `no_output` | the code around a run fails, such as a recorder that refuses a write; a caller code never ends a run |

A failed `{{ }}` substitution is an ordinary Lua error that `pcall` can catch, and it ends the run with `substitution` when uncaught. A cancel ends the run with `canceled`, a run code, yet a canceled run is not a failed run, and Workshop's run log records it as `canceled`, not `failed`.

### The failed run's message

A failed run's message is its error's located message: the underlying error's own text followed by the text of each cause in its chain, joined by `: `, and any cause whose text already appears in the message is left out so no text repeats. Then comes where the failure happened, when it is located, as [Where a run error's message points](#where-a-run-errors-message-points) describes. The code never replaces the message, and the message stands on its own.

### Parse codes

A prompt that fails to parse fails before any run starts, so before any section or code runs, before input staging, and before the wait for Packs, and its parse error kind decides the code. `Frontmatter` is `frontmatter_invalid`, `Structure` is `structure_invalid`, `Fence` is `fence_invalid`, and `List` is `list_empty_item`. A `Lua` parse failure, a block that does not compile, is `lua_compile`, a Lua code (see [Lua codes](#lua-codes)), and an `Internal` parser fault is `internal`, a run code. The failure message is the parse error followed by its chain of causes, then its location form, as [Where a run error's message points](#where-a-run-errors-message-points) describes.

A file with no `runvil:` key also fails with `structure_invalid`, but its prompt parses: the version check is the first start check on the run's first step, after input staging and the wait for Packs and before the store and requirements checks. So an input refusal comes first when one applies, and either way the run's record holds no events. The message is `not a runvil prompt: no runvil version (prompt {name})`, which ends with the prompt's frontmatter `name`.

A run fails with `unsupported_version` when the prompt's [`runvil:` key](02-file-structure.md#the-runvil-version-key) declares a major version other than `0`, and the message is `unsupported runvil version: {n} (this build supports major 0)` followed by ` (at {file}:{line})`, the line of the `runvil:` key, as `future` shows.

The family's last code, `invalid_value`, marks a value that is missing or fails to parse or validate, such as an id, a tag, or a setting.

### Lua codes

A run fails with `lua_runtime` when a section's Lua fails at run time or returns an unusable value, and the message is the Lua error's own text, followed by ` (at {file}:{line})` when the failure is located (see [Where a run error's message points](#where-a-run-errors-message-points)). It covers an uncaught Lua failure in a walked section, a `call` chain, a task, a `tasks.fanout` arm, or a library loaded with `require`, from any of these causes:

- an uncaught runtime error
- running past the memory ceiling
- a block that returns a table, whose message includes `cannot return a table as a result`

A [Pack prelude](12-tools.md#pack-preludes) that cannot install also fails the run with `lua_runtime`, before any section code runs.

A run fails with `lua_compile` when a Lua block does not compile, a parse failure that ends the run before any section or code runs (see [Parse codes](#parse-codes)), and with `substitution` when a failed `{{ }}` substitution goes uncaught (see [Placeholders in prose](07-substitution.md#placeholders-in-prose)).

A run fails with `lua_quota` when a section entry's Lua runs out its `log` event quota or its `log` byte quota, and with `call_depth_exceeded` when a `call`, `tasks.spawn`, or `tasks.fanout` arm would nest past the call depth cap. No instruction-count quota exists. The message is one of these, followed by ` (at {file}:{line})`, the line of the refused `log` call or of the `call`, `tasks.spawn`, or `tasks.fanout` that went too deep:

````text
lua log event quota exceeded
lua log byte quota exceeded
recursion exceeded cap of {cap}
````

The call depth message reads the same whichever of `call`, `tasks.spawn`, or `tasks.fanout` went too deep, so with a cap of 8 it reads `recursion exceeded cap of 8`. Caught with `pcall`, the same failure is an error value of kind `lua` and code `call_depth_exceeded` whose message reads like this:

````text
{file}:{line}: recursion exceeded cap of 8. Calls nest deeper than the cap allows. Nest `call`, `tasks.spawn`, and `tasks.fanout` less deeply.
````

### Task codes

A run fails with a task code when a task error goes uncaught in a walked section, a `call` chain, a task, or a `tasks.fanout` arm, and in the H1 pass too:

- `tasks_live`, a chain that ends with tasks still running and not idle, a leaked task
- `task_not_owned`, a task reached by a chain that does not own it
- `task_consumed`, a result waited on twice
- `task_canceled`, a canceled task's error value raised again after a wait delivers it

The message is the task error's own text, followed by ` (at {file}:{line})`, so an uncaught `task_canceled` reads `` task `{task}` was canceled (at {file}:{line}) `` (see [What a wait gives back for a canceled task](15-tasks.md#what-a-wait-gives-back-for-a-canceled-task)).

### Model codes

A run fails with a model code when a model call fails at the transport, backend, or decode layer and the failure goes uncaught, and the message is the model call's own error text. Each completion error kind has the model code of the same name in snake_case, such as `overloaded` for an overloaded backend or `timeout` for a receive timeout, and an uncaught `empty_model_reply` error ends the run with `empty_reply`. [Model request limits and failures](#model-request-limits-and-failures) lists those messages. The Harness also ends a run with a model code before it prepares the run when it cannot fetch the Host's list of models, such as `transport` when the connection to the model backend fails, and the message then starts with `the model catalog could not be fetched`.

A run fails with `context_exhausted` when a request overflows the model's context window, either on the check before sending or at the provider, and the selected [compactor](11-conversations.md#compactors-and-context-exhaustion) does not make it fit, or when a `compactors.fail(reason)` call goes uncaught, whether a compactor made it inside a loop with a reason of your own or your Lua made it directly outside any loop, with any reason. The message is `context exhausted: {detail}`, followed by ` (at {file}:{line})` when the failure is located, where the detail is the overflow reason in words for `precheck` and `provider`, and the reason itself otherwise.

A run fails with `model_required` when a model round has no model to run on because the Host gave none: an uncaught [missing-model error](10-models.md#infer-errors-and-the-missing-model-error) from [`models.infer`](10-models.md#running-a-round-with-modelsinfer) or [`models.loop`](11-conversations.md#how-the-loop-runs) fails the run with the message `model binding required for section {section}`, followed by ` (at {file}:{line})`, the line of the call. Prose by itself never requires a model; only an explicit `models.infer` of it does.

### Tool codes

A run fails with one of two tool codes when a section entry's first loop round cannot freeze its tool sets (see [When an offer takes effect](12-tools.md#when-an-offer-takes-effect) and [What a Pack's render can change](12-tools.md#what-a-packs-render-can-change)): `tool_schema_invalid` when a tool's model-facing schema does not build, and `tool_sets_invalid` when the sets, or the render answer they are merged from, break a rule of the merge that builds them. The run reports `tool_scope_validation_failed`, the round sends no request and freezes nothing, so offers still work after it, and the `models.loop` call raises an error value of kind `internal` that `pcall` catches. Left uncaught, it ends the run with its code, and the run's message begins with the error's own message (see [The failed run's message](#the-failed-runs-message)).

A schema that does not build gives one of two `tool_schema_invalid` messages, with the tool's wire name as `{name}`: a tool whose wire name breaks the wire-name rule gives `model-facing schema build failure for tool alias "{name}": invalid tool wire name "{name}": expected [A-Za-z][A-Za-z0-9_-]{0,63}`, and a tool whose parameter schema is not a JSON object gives `model-facing schema build failure for tool alias "{name}": tool "{name}" parameters schema must be a JSON object`.

Each broken merge rule has its own `tool_sets_invalid` message, where `{tool}` reads `{id} of the {pack} Pack in the {set}` and `{set}` is `Static Set` or `Dynamic Set`:

- Two tools that share a wire name give `the tools {tool} and {tool} share the wire name "{name}", and a wire name names one tool across a section's Static Set and Dynamic Set; offer only one of them`.
- A Pack's render that returns a tool under another Pack's id gives `the {pack} Pack's render returned {tool}; a Pack renders only tools whose id starts with its own Pack id`.
- A Pack's render that returns a run catalog tool the section did not offer in that set gives `the render returned {tool}, a run catalog tool the section did not offer in that set; a render returns a catalog tool only into a set the section offered it in`.
- A render answer with tools for a set the section did not offer the Pack into gives `the {pack} Pack's render returned {tool}, a set the section did not offer the {pack} Pack into; a render returns tools only for the sets the section offered its Pack into`.
- A render answer that names other Packs, or the offered Packs in another order, gives `the render answered the Packs {answered} where the section offered the Packs {asked}; a render answers each offered Pack once, in offer order`, with each list of Pack ids in brackets, such as `[web, gh]`.

A run fails with one of the other tool codes when a tool call goes wrong and the failure goes uncaught (see [Tool errors](12-tools.md#tool-errors)):

- `tool_invalid_arguments`, `tool_backend`, `tool_transport`, or `tool_other`: a dispatched tool fails, because it rejects its arguments, its backend fails or refuses the request, the call fails at the transport layer, or for any other reason
- `out_of_scope_tool`: the model calls a tool outside the section entry's [tool scope](12-tools.md#tool-scope)
- `unbound_tool`: a `tools.call` names an id that is neither a defined adhoc tool nor a tool in the run's catalog
- `model_only_tool`: a `tools.call` names a catalog tool marked `model_only_call`
- `tool_loop_exhausted`: the tool-call loop reaches its round cap before the model gives a final reply

A tool marked `model_only_call` is one only a model may call (see [Model-only tools](12-tools.md#model-only-tools)), so a script's `tools.call` of it is refused, whether the script names it by id or passes its Tool object. The tool stays in the catalog, a section can still offer it with `tools.offer`, and the model's own call of it goes through. The refusal raises an error value of kind `model_only_tool` whose `name` field holds the name the script passed, and `pcall` catches it like any other error value. Left uncaught, it ends the run with code `model_only_tool` and this message, where `{name}` is the id the script passed, `{id}` is the catalog tool's own id, and the location is the line of the `tools.call`:

````text
tool "{name}" ({id}) is model-only: a render shapes its arguments for the model that receives them, so a script cannot call it; offer it with tools.offer and let the model call it (at {file}:{line})
````

### Store codes

A run fails with a store code when a `store` operation fails and goes uncaught, and the code names how it failed, such as `not_found` or `read_only` (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). A run that declares no store to operate on fails with `no_store`. A store backend that fails as the run starts also ends the run with its store code, even when the prompt never calls `store` (see [Host stores, policies, and modes](09-the-store.md#host-stores-policies-and-modes)), and so does a store that refuses to stage the declared input file before the run starts. An uncaught `store.read('missing.txt')` fails the run with `not_found` and a message that includes `file not found in store: missing.txt`, and a store operation in a run that declares no store fails with `no_store` and a message that includes `this run declares no store, so no store operation can run`.

A store call that meets a claims conflict fails the run with `determinism` (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)), and its message starts with `store determinism violation: `. Unlike other store failures, any claims conflict ends the run on the spot, even in the H1 pass, so no `pcall` can catch it, not even a `pcall` a library runs while it loads, and no task or arm counts as failed.

### Run codes

A run fails with `requirements_unmet` when the environment falls short of what the prompt needs: a required Pack is missing or unavailable, the model that fills a role fails a [requirement the prompt declared](10-models.md#what-the-run-checks) for it, a `min_context` or a `thinking` setting, the Host gave a role map that holds models but has no `default` entry, a tool in the run's catalog or a Pack in its roster has a Pack id the Engine reserves, `adhoc` or `dynamic`, or an H1 block fails the prompt's [hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate). Before the run starts, the Harness also ends a run with `requirements_unmet` when the launch input does not match the prompt's `input:` declaration, or when the Host's selected model is missing from its list of models.

The message of a refused run starts with `the environment cannot satisfy this prompt:` and adds one line for each gap. Pack lines come first, then the model lines in role order (`default`, `strong`, `small`, `fast`, `creative`), and for one role the context line comes before the thinking line. When a gap is one the prompt caused, the message ends with ` (at {file}:{line})` after its last line, naming the `uses:` line when a declared Pack is missing or unavailable, and otherwise the `models:` line of a declared role requirement that falls short. A model requirement that the filling model fails reads like one of these, and the thinking line names the required setting, `thinking: true` or `thinking: false`, and the model's thinking capability, `Never` or `Always`:

````text
- role 'strong': requires a context of at least 200000 tokens; the model filling it provides 32000
- role 'strong': requires 'thinking: true'; the thinking capability of the model filling it is Never
- role 'small': requires 'thinking: false'; the thinking capability of the model filling it is Always
````

A role map that holds models but has no `default` entry gives one model line and no other, because no declared requirement is checked in that case:

````text
- role 'default': requires a model; the caller's role map has no 'default' entry
````

That gap is the caller's, as a reserved Pack id is, so a refusal made only of the caller's gaps points at no line.

[How the Host fills and checks roles](10-models.md#how-the-host-fills-and-checks-roles) covers each check. The run is refused at its first step, before any section runs.

A run ends with `canceled` when the Host cancels it, or when the canceled error from a stop goes uncaught. A canceled run is not a failed run, and Workshop reports it as `the agent run was canceled` (see [Canceled runs and runs that end before they start](#canceled-runs-and-runs-that-end-before-they-start) and [Cancellation and stops](#cancellation-and-stops)).

A run fails with `internal` when an Engine invariant breaks, a fault in the Engine rather than in the prompt. That includes Harness mistakes such as an answer of the wrong kind, an answer for an id never issued, or a second answer to one effect. The message starts with `internal invariant violated: {message}` and ends with ` (at {file}:{line})`, which names an Engine source file and line rather than a place in the prompt. The Harness also ends a run with `internal` when the run is still pending, its main chain does not wait for input, and the Harness has nothing to wait for, which means it lost track of an effect, and the message is then `the effect loop has nothing to await for a pending run`.

A run fails with `stalled` when no chain can move and the main chain does not wait on its own inbox, the one inbox the Host can fill. Every chain left waits on something only another waiting chain could end, such as an owner waiting on a task that waits for a message only that owner could send (see [When nothing can move](15-tasks.md#when-nothing-can-move)). The message names, in id order, the tasks that wait on their inboxes, and the error's facts hold their ids. A stall has no location, so the message is its own text:

````text
the run stalled: no chain can move, and these tasks wait on their inboxes: {ids}
````

A stall with no task waiting on its inbox ends the run with `internal` instead, because only an Engine fault can leave every chain stuck that way, and its message starts `internal invariant violated: the scheduler stalled with no ready chain, no in-flight request, and no queued chain that can be admitted`. A main chain that waits in `tasks.receive()` with nothing else to run never stalls: the run waits for the Host's next message or a cancel (see [While the section waits](05-lua-environment.md#while-the-section-waits)).

### Caller codes

The caller codes name failures in the code around a run rather than in the run itself: `recorder_failed` when the Host's recorder refuses a write, `plugin_load_failed` when a Plugin's entry point does not open or load, and `no_output` when a run report has no output text, because the run did not complete or the prompt declares no `output:` file.

A caller code never ends a run. `Harness::run` hands the Host the run's result, and it fails only when the Host's recorder refuses a write, with `recorder_failed`. A prompt that raises an error value whose `code` names a caller code ends the run with `internal`, keeping the message.

### Canceled runs and runs that end before they start

A run the Host cancels ends with the canceled error, whose code is `canceled`, rather than as a failed run, and Workshop's run log records its status as `canceled`, not `failed`. That holds even for a cancel that comes before the run starts or while the Harness looks up the model.

A frontmatter parse failure ends the run with `frontmatter_invalid`, and a parse failure in the prompt body ends it with its own code, such as `structure_invalid` or `fence_invalid`. A run that ends before it starts (see [Preparing the run](04-how-a-prompt-runs.md#preparing-the-run)) ends with a model code when the Harness cannot fetch the Host's list of models, `requirements_unmet` when that list does not hold the selected model, the code of its parse failure, such as `structure_invalid` or `lua_compile`, when the prompt does not parse, `requirements_unmet` when the launch input does not match the prompt's `input:` declaration, or a store code when the store refuses to stage the input. Its message is the full chain of causes, followed by its location form when it has one: a parse failure ends with its location form, while a model lookup failure or an input staging failure has no location, so its message is its own text.

Each of these codes belongs to one of the eight code families, and the Harness adds no code of its own.

### Codes in the H1 pass and through calls

The Lua codes hold alike in a walked section, a `call` chain, a task, and a `tasks.fanout` arm. An ordinary Lua runtime error in the H1 pass, which would be `lua_runtime` in a section, ends the run with `requirements_unmet` instead, as a failed hard gate, while a failed `{{ }}` substitution and any other failure in the H1 pass keep their own code.

Passing the call depth cap through `call`, `tasks.spawn`, or `tasks.fanout` ends the run with `call_depth_exceeded`, not `lua_runtime`, even though a `pcall` that catches the same error reads its kind as `lua`. When a `call` or a task wait fails and the prompt does not catch it, the run ends with that failure's own code rather than as a generic `lua_runtime` failure, such as `call_depth_exceeded` for the call depth cap, because the error passes up through every parent without being flattened.

## Error kinds in Lua and the run error they become

After this section you can read the error kind and code of a caught failure and predict the code the same failure ends the run with when no `pcall` catches it.

````markdown
---
name: summary
description: Turns a failed model call into the section's result.
runvil: 0
---

# Summary

## Draft

Draft the summary.

```lua
local ok, reply = pcall(models.infer, prose)
if ok then
  return reply
end
if reply.kind == 'model_failure' then
  return 'skipped: ' .. reply.message
end
error(reply)
```
````

When the model call succeeds, the run result is the model's reply. When it fails outside the prompt, for example because the backend cannot answer, the run result is `skipped: ` followed by the failure's message. Any other failure is raised again unchanged, so the run ends exactly as it would have without the `pcall`.

### Sixteen error kinds

Every Engine failure that reaches Lua carries exactly one of sixteen error kinds in its error value's `kind` field: `lua`, `context_exhausted`, `empty_model_reply`, `model_failure`, `model_required`, `canceled`, `tool_loop_exhausted`, `tasks_live`, `task_not_owned`, `task_consumed`, `out_of_scope_tool`, `unbound_tool`, `model_only_tool`, `tool`, `store`, and `internal`.

Error kinds and codes are separate vocabularies. An error kind is the broad class Lua sees in an error value's `kind`, and a code is the exact failure, which Lua sees in its `code` and which the run's error has when the failure ends the run. One error kind covers several codes, as `internal` does for a failed tool freeze, `tool_schema_invalid` or `tool_sets_invalid`, and for an Engine fault, `internal`.

### The `model_failure` and `model_required` error kinds

The `model_failure` error kind marks a model call that failed outside the prompt: a transport, backend, or decode failure, or a disabled gateway. An empty reply is the one model call failure with a kind of its own, `empty_model_reply`. The `model_required` error kind marks the [missing-model error](10-models.md#infer-errors-and-the-missing-model-error), a `models.infer` or `models.loop` round that finds the Host gave no model. [`models.use`, `models.get`, and `models.current`](10-models.md#when-the-host-gives-no-model) raise an ordinary error of kind `lua` instead when the Host gave no model for the role.

Neither value has extra fields. Left uncaught, a `model_failure` value ends the run with its own model code, such as `overloaded` or `timeout`, and a `model_required` value ends it with `model_required`.

### The `internal` error kind

The `internal` error kind marks a failure that has no kind of its own. A violated internal invariant and a section entry's failed tool freeze reach Lua as kind `internal`, while a store failure has its own kind `store` and a failed model call its own kind `model_failure`.

An `internal` error value has no extra fields. Left uncaught, it ends the run with `tool_schema_invalid` or `tool_sets_invalid` for a failed tool freeze (see [Tool codes](#tool-codes)), or `internal` for an Engine fault.

When an Engine function fails and `pcall` catches it, the error kind is `lua` and the code `lua_runtime` for an authoring or argument problem, such as a runtime error, a syntax error, a memory error, a bad argument, or a value that cannot be converted or serialized. The kind and the code are both `internal` when the Lua runtime's own machinery fails.

### Uncaught failures keep their code

An uncaught failure ends the run with its own error, its code, message, and facts intact, never flattened to a message string, so the run ends with the same code the error value's `code` field shows.

### Raising a caught error value again

A caught error value raised again with `error(err)` before the block makes another [suspending call](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), a call that waits on the Harness, ends the run exactly as if it had never been caught, with the same code (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). Raised again after another suspending call, a caught error value of any kind but `lua` still ends the run with its original error, while a `lua` value ends it as an ordinary Lua runtime error, `lua_runtime`.

That is what `summary` relies on. A section can turn a caught `model_failure`, a model call the Harness cannot complete, into its result, and raise every other kind again with `error(reply)` before any other suspending call, so the run ends exactly as it would have without the `pcall`.

A model call's transport, backend, or decode failure, or a disabled gateway, raises a `model_failure` error value, and an empty model reply raises `empty_model_reply` instead. The missing-model error raises `model_required`, a script's `tools.call` of a model-only tool raises `model_only_tool`, and a failure with no kind of its own, such as a failed tool freeze or an Engine fault, raises an `internal` error value. This holds inside a `tasks.fanout` arm too, where an arm's `models.infer` with no model access configured surfaces the disabled-gateway failure as `model_failure`.

Left uncaught in a walked section, a [`models.loop`](11-conversations.md#loop-arguments-and-errors) failure ends the run like this:

| Error kind from `models.loop` | Code when uncaught |
|---|---|
| `lua` | a Lua code, such as `lua_runtime` |
| `empty_model_reply` | `empty_reply` |
| `context_exhausted` | `context_exhausted` |
| `tool_loop_exhausted`, `out_of_scope_tool` | the code with the same tag as the kind |
| `model_failure` | its own model code, such as `overloaded` or `timeout` |
| `model_required` | `model_required` |
| `internal` | `tool_schema_invalid` or `tool_sets_invalid` for a failed tool freeze |

Raised again later, after another suspending call, each error kind ends the run like this:

| Error kind raised again later | Code that ends the run |
|---|---|
| `lua` | `lua_runtime`, or `requirements_unmet` in the H1 pass |
| `empty_model_reply` | `empty_reply` |
| `model_failure` | its own model code, such as `overloaded` or `timeout` |
| `context_exhausted` | `context_exhausted` |
| `tool_loop_exhausted`, `out_of_scope_tool`, `unbound_tool`, `model_only_tool` | the code with the same tag as the kind |
| `tool` | its own tool code: `tool_invalid_arguments`, `tool_backend`, `tool_transport`, or `tool_other` |
| `store` | its own store code, such as `not_found` |
| `model_required` | `model_required` |
| `tasks_live`, `task_not_owned`, `task_consumed` | the code with the same tag as the kind |
| a canceled task's `canceled` value | `task_canceled` |
| `internal` | `tool_schema_invalid` or `tool_sets_invalid` for a failed tool freeze, or `internal` for an Engine fault |

Every kind but `lua` raised again later keeps its whole original error, its code, message, fields, and line included, so it ends the run as it would have uncaught, in the H1 pass too. A `lua` value raised again later is rebuilt as a plain Lua runtime error, code `lua_runtime`, that keeps its message and points at the `error` line. A `canceled` value from a canceled run or a stopped call, raised again later, ends the run with the canceled error, as [Cancellation and stops](#cancellation-and-stops) explains.

### Task and heading errors

When a task error ends the run uncaught, in any chain and in the H1 pass too, the run ends with its task code: `tasks_live` for a leaked task, `task_not_owned` for a task reached for by a chain that does not own it, `task_consumed` for a result waited on twice, and `task_canceled` for a canceled task's `canceled` value re-raised right after the wait (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)).

Left uncaught, a heading-not-found error or a malformed heading reference error ends the run with `lua_runtime`, or with `requirements_unmet` when it reaches the H1 body's own Lua (see [Resolving heading references](08-jump-and-call.md#resolving-heading-references)). The not-found error names the heading in this text, and the run's message ends with ` (at {file}:{line})`, the line of the `jump`, `call`, or `require` that named the heading:

````text
section heading `{heading}` not found; available sections: {list}
````

Only the H1 body's own Lua code remaps this way. The machinery around it, namely section setup, the final `var` read-back, and jump-target resolution, keeps its own code.

## Model request limits and failures

After this section you can catch a failed model call, read its completion error kind and message, and know how the response cap and the receive timeout bound each round.

````markdown
---
name: summary
description: Turns a failed model call into the section's result.
runvil: 0
---

# Summary

## Draft

Draft the summary.

```lua
local ok, reply = pcall(models.infer, prose)
if ok then
  return reply
end
if reply.kind == 'model_failure' then
  log('model call failed')
  return 'skipped: ' .. reply.message
end
error(reply)
```
````

This is `summary` from the previous section with one `log` line added. When the backend answers with status 503, the run result is:

````text
skipped: summary.md:14: the model backend is overloaded (status 503)
````

The block catches the model call failure, checks for kind `model_failure`, logs a checkpoint, and returns `skipped: ` plus the error's message, which starts with the file and line of the `models.infer` call.

### Completion errors and how Lua sees them

A model call that fails surfaces as a completion error, and every completion error carries a completion error kind that names what went wrong, such as `Overloaded` or `Timeout`. When the model request for a `models.infer` round fails, the `models.infer` call raises that failure as its error.

When the prompt catches a model call failure, the error value has kind `model_failure`, whether the cause is a connection failure, a receive timeout, a backend error status, a malformed or oversized reply, or a disabled gateway. An empty reply is the one case that differs, with kind `empty_model_reply` instead. A model call failure that the prompt does not catch ends the run with its model code, such as `overloaded`, and the run's message starts with the failure's own text, such as `the model backend is overloaded (status 503)`, and ends with ` (at {file}:{line})`, the line of the call.

A model failure caused by a backend HTTP status puts ` (status {status})` after the kind's fixed phrase, where `{status}` is the status code. That is how `summary` returns `skipped: summary.md:14: the model backend is overloaded (status 503)` when the backend answers with status 503.

Each completion error kind has one cause and a fixed answer to whether it is retryable:

| Completion error kind | What causes it | Retryable |
|---|---|---|
| `Transport` | a failed connection, a broken stream, or an error reply whose body cannot be read | yes |
| `Timeout` | the headers or the next chunk miss the receive timeout | yes |
| `MalformedResponse` | a reply that cannot be understood, an oversized one included | yes |
| `RateLimited` | backend status 429 | yes |
| `QuotaExhausted` | status 429 whose body names a quota, billing, or credit | no |
| `Overloaded` | status 503 or 529, or a 5xx whose body names `overloaded` | yes |
| `ServerError` | any other 5xx status | yes |
| `Refused` | status 400 whose body names a content policy term and no context limit | no |
| `ContextOverflow` | status 400 or 413 whose body names a context limit | no |
| `Unavailable` | model access turned off or not configured, or status 401 or 403 | no |
| `Rejected` | any other non-success status | no |
| `EmptyReply` | a reply with no tool calls and no text other than whitespace | no |
| `InvalidParts` | a caller built a tool call or completion from parts the validating constructors refuse | no |

### The response cap and the receive timeout

The response cap is the largest model reply a run accepts, 16 MiB by default, and the receive timeout is the longest wait for the next part of a model reply, 120 seconds by default. Both apply to every model round in the run, both `models.infer` rounds, nested ones included, and `models.loop` rounds, so an oversized or stalled reply ends that round with an error instead of hanging the run.

The receive timeout bounds each wait for the next part of a model reply, first the headers and then each body chunk. Every chunk that arrives restarts the wait, and no limit covers the whole request, so a reply that keeps streaming is never cut off.

### What a model failure message holds

A caught model call failure carries no fields beyond `kind`, `code`, and `message`, and its `code` is the completion error kind in snake_case, such as `overloaded` or `rate_limited`. Its message is the line of the call followed by the failure's prose, which never includes the backend's reply body, so a hostile or private payload cannot leak into Lua or forge a log line there. When the failure goes uncaught, the failed run's message can hold the body after the prose.

### Connection, timeout, and malformed-reply failures

A failed connection, a broken stream, or an error reply whose body cannot be read fails the round with completion error kind `Transport`, and the failure is retryable. A model reply whose headers or next chunk do not arrive within the receive timeout fails the round with completion error kind `Timeout`, and the failure is retryable. Their messages are:

````text
the connection to the model backend failed
the model backend did not answer in time
````

A reply that cannot be understood fails the round with completion error kind `MalformedResponse`, and the failure is retryable. The message is a fixed phrase followed by what is wrong, as in the second line here:

````text
the model backend sent a reply that could not be understood: {specific}
the model backend sent a reply that could not be understood: completion stream ended without the [DONE] sentinel
````

When a `length` or `content_filter` [finish reason](11-conversations.md#empty-replies-and-finish-reasons) cuts off a batch of tool calls, the round fails as `MalformedResponse` with the fixed phrase followed by `: tool-call batch truncated by finish_reason "length": partial arguments must not execute`, or `"content_filter"` in place of `"length"`, so no partial tool arguments run.

A model reply that would pass the response cap is refused as its bytes arrive, before any of it is decoded, as `MalformedResponse`, and the cap covers error replies as well as successful ones. The fixed phrase ends one of three ways:

````text
the model backend sent a reply that could not be understood: response stream exceeds the {max_bytes}-byte limit
the model backend sent a reply that could not be understood: response body exceeds the {cap}-byte limit
the model backend sent a reply that could not be understood: response body of {len} bytes exceeds the {cap}-byte limit
````

The first is a successful reply over the cap, the second an error reply over the cap, and the third an error reply whose advertised length is already over the cap. A response-cap or receive-timeout failure is kind `model_failure` when caught, with code `malformed_response` or `timeout`, and ends the run with that code when uncaught, and it counts as transient, so running the prompt again may succeed.

### Failures from a backend status

A backend status 429 fails the round as `RateLimited`, and it is retryable. A 429 whose body names a quota, billing, or credit fails as `QuotaExhausted` instead, and it is not retryable. Their messages are:

````text
the model backend is limiting the request rate (status 429)
the model backend says the usage quota is spent (status 429)
````

A backend status 503 or 529, or any 5xx status whose body names `overloaded`, fails the round as `Overloaded`, and any other 5xx status fails as `ServerError`. Both are retryable, and their messages are:

````text
the model backend is overloaded (status {status})
the model backend reported a fault of its own (status {status})
````

A `models.loop` round answered with status 500 raises `the model backend reported a fault of its own (status 500)` out of the call, so the code after the call does not run. A 502 whose body holds forged log text still shows only `the model backend reported a fault of its own (status 502)`.

A backend status 400 whose body names a content policy term and no context limit fails the round as `Refused`, and a status 400 or 413 whose body names a context limit fails as `ContextOverflow`. Neither is retryable, and their messages are:

````text
the model backend refused the request on content policy grounds (status 400)
the request is larger than the model's context window (status {status})
````

A `models.infer` call raises a `ContextOverflow` failure as kind `model_failure`, while a `models.loop` round hands it to the compactor instead (see [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion)).

When model access is turned off or not configured, every model round fails as `Unavailable`, nothing is sent to the network, and the failure is not retryable. A backend status 401 or 403 also fails the round as `Unavailable`, and it is not retryable. Their messages are:

````text
model access is turned off or not configured
the model backend did not accept the credentials (status {status})
````

Any other non-success backend status fails the round as `Rejected`, and it is not retryable:

````text
the model backend rejected the request (status {status})
````

A backend status 400 whose body names no context limit, such as one that reports an unknown field, propagates out of `models.loop` as a rejection with the message `the model backend rejected the request (status 400)`, and no compactor runs.

### Errors inside a reply stream

When the backend reports an error inside a reply stream that has already begun, the round fails by the error's text alone, with the kind's fixed phrase and no status. Text naming a context limit is `ContextOverflow`, a quota or billing word is `QuotaExhausted`, `overloaded` is `Overloaded`, a content policy term is `Refused`, and anything else is `Transport`.

### Empty replies and unexpected tool calls

A reply with no tool calls and no text other than whitespace fails as `EmptyReply`, and it is not retryable:

````text
the model replied with no text and no tool calls
````

An uncaught `empty_model_reply`, including one the prompt caught and raised again, ends the run with code `empty_reply` and keeps its message.

When the model answers a `models.infer` round with tool calls, the call fails as `MalformedResponse`, and it is retryable:

````text
the model backend sent a reply that could not be understood: the round sent tool calls, but no tools were advertised
````

### Seeing raw model traffic in debug mode

In Host debug mode, the run adds `Request` and `Response` events that hold the raw model request and response bodies.

## Retryable failures

After this section you can tell from a failed run whether running it again may help.

````markdown
---
name: brief
description: Sends one model round and returns the reply.
runvil: 0
---

# Brief

## Write

Write a one-line brief.

```lua
return models.infer(prose)
```
````

When the backend answers with status 503, the run fails with code `overloaded` and a message that starts with this text and ends with ` (at brief.md:14)`, the line of the `models.infer` call, and the failure is retryable:

````text
the model backend is overloaded (status 503)
````

When the backend answers with status 429 and a body that names a spent quota, such as `insufficient_quota`, the run fails with code `quota_exhausted` and a message that starts with this text and ends the same way, and the failure is not retryable:

````text
the model backend says the usage quota is spent (status 429)
````

Both runs fail with a model code, but only the first, `overloaded`, may succeed when you run the prompt again.

### What makes a failure retryable

A failed run is retryable only when a model call fails with a transient transport or backend problem, and that is what a retryable failure means: running the prompt again may succeed. A model call failure is retryable when it comes from a transport failure (a receive timeout included), a malformed or oversized reply, a rate limit (backend status 429), an overloaded backend (status 503 or 529), or any backend status from 500 through 599, so the retryable codes are `transport`, `timeout`, `malformed_response`, `rate_limited`, `overloaded`, and `server_error`. The failure reports the backend status when there is one, and nothing reruns the failed run automatically.

### Failures that are never retryable

Every failure other than a transient model call failure is never retryable, and neither is a canceled run, which is not a failure:

- frontmatter and body parse failures
- Lua errors, runtime and compile errors included
- schema binding failures
- substitution failures
- an exhausted tool loop
- task failures: tasks still running and not idle when their owner ends, a task the prompt does not own, a task already consumed, or a canceled task
- a tool call outside the prompt's scope, to an unbound tool, or from a script to a model-only tool
- a required model that is missing
- an unsupported version
- unmet requirements
- tool failures
- internal failures
- a spent Lua quota
- an exceeded call depth
- an exhausted context
- store failures, claims conflicts included
- a stalled run, where no chain can move

### The completion error kind decides

Only model failures of completion error kind `RateLimited`, `Overloaded`, `Timeout`, `Transport`, `ServerError`, and `MalformedResponse` are retryable, every other kind is not, and the kind alone fixes the answer.

### Nothing retries automatically

Nothing reruns a failed run or resends a failed model round automatically. Each model round is sent once, and a failed round is reported as failed and ends the call with that failure. The run hands its failure back as a result value instead of crashing, so running again after a retryable failure is up to the Host, and a run that fails on a transient model call problem may succeed when run again.

### Transient model failures in detail

Connection failures and receive timeouts, an error reply whose body cannot be read included, count as transient. A receive timeout fails as kind `Timeout`, the others fail as kind `Transport`, and both are retryable.

A malformed or oversized model reply counts as transient: it fails as kind `MalformedResponse` and is retryable, and an oversized body fails with the detail `response body of {len} bytes exceeds the {cap}-byte limit`. A reply stream that ends before its `[DONE]` sentinel is cut off, so it fails as a malformed reply (kind `MalformedResponse`) with the detail `completion stream ended without the [DONE] sentinel`, and it counts as transient.

A rate limit, meaning a backend status 429 whose body does not name a spent quota, and an overloaded backend, status 503 or 529, count as transient. So a 429 with the body `slow down` is retryable, while a 429 whose body names a quota, such as `insufficient_quota`, fails as `QuotaExhausted` and is not retryable.

Model backend statuses from 500 through 599 count as transient: 503, 529, and any 5xx whose body contains `overloaded` fail as `Overloaded`, and every other 5xx fails as `ServerError`.

A context overflow, a spent quota (`QuotaExhausted`), a request refused on content policy grounds (`Refused`) or otherwise rejected (`Rejected`), bad credentials (status 401 or 403, kind `Unavailable`), an empty reply, or a disabled model gateway (kind `Unavailable`) is not transient, so none of them is retryable.

When the backend reports an error partway through a streamed reply, the Harness bounds the error text, with a bound of 2000, and escapes its control characters, then classifies the failure by that text alone. Text naming an overload, such as `overloaded_error: try again`, fails as `Overloaded`, and text naming no known cause fails as `Transport`, both retryable. Text naming a context limit, a spent quota, or a content policy fails as a context overflow, `QuotaExhausted`, or `Refused`, and is not transient.

When the model answers a `models.infer` round with tool calls although none are advertised, the call fails as a malformed reply (kind `MalformedResponse`) with the detail `the round sent tool calls, but no tools were advertised`. The failure is retryable, because a reply that breaks protocol may come back well formed on another try. When that failure ends the run, the run fails with code `malformed_response`, and it still counts as a retryable failure.

## Cancellation and stops

After this section you can predict how a run ends when the Host cancels it or stops the round in flight, and write a block that carries on after a stop.

````markdown
---
name: spinner
description: Spins until the Host cancels the run.
runvil: 0
---

# Spinner

## Spin

```lua
local n = 0
while true do n = n + 1 end
```
````

Nothing in `spinner` ends the loop, and no limit would. When the Host cancels the run, the section entry's Lua instruction hook sees the cancel as the code runs, the loop is aborted, and the run ends with the canceled error.

### Canceling a run

The Host can cancel a run from any thread by setting the cancel flag it gave the run when it built the run. It can cancel at any time, even while Lua is busy in a loop or waiting on a model reply, and the run's next step tears it down wherever it is.

A run the Host cancels, before it starts or while it runs, ends with the canceled error, the error whose code is `canceled`, and its message is `canceled: the run was canceled or this call was stopped`. A canceled run is not a failed run: Workshop records its status as `canceled` and reports it as `the agent run was canceled` rather than with a failure message. Inside the run, a cancel surfaces as the canceled error, whose kind and code in Lua are both `canceled`, and that error ends the run.

Once a run is canceled, it issues no further model rounds or tool calls. Its result is decided at the next step and reported once the Host answers every request still out, whether a model reply, a [render](01-what-a-prompt-is.md#the-five-kinds-of-harness-work), a tool call, or a timer, as dropped.

### The cancel flag

The cancel flag is the one signal a cancel sets. One cancel flag is shared by the Lua instruction hook in every section VM, by every activated [Pack](12-tools.md#tools-packs-and-the-uses-key), and by the run itself, so one cancel reaches them all. A cancel stops a run even in the middle of a running Lua block, because the Engine checks the cancel flag between chain steps and from the Lua instruction hook.

A Host cancel stops a running Lua block promptly, even a tight endless loop such as `while true do n = n + 1 end` that nothing in the prompt and no limit would end, because the section entry's Lua instruction hook checks for cancellation as the code runs, and the run ends with the canceled error, as `spinner` shows. Canceling a run aborts a Lua block that is still running, even a tight `while true do end` loop, and even when the run is canceled before the block starts; the block ends with the canceled error.

### What a cancel does to waiting calls

A cancel never resumes a call that is waiting on the Harness, such as `models.infer`, a `models.loop` round, `tools.call`, or a timed wait. The next step tears the waiting block down, the Harness answers each call still out as dropped, and the run ends with the canceled error, so a `pcall` around the call never returns.

### The canceled error

The canceled error is the error value a cancel or a stop raises in Lua. It has kind `canceled`, the `code` `canceled`, the same message whether a cancel or a stop causes it, and no fields beyond `kind`, `code`, and `message`. The message names the line of the call that was waiting, then the canceled error's prose:

````text
{file}:{line}: The operation was canceled.
````

Your code cannot swallow a cancel. Every protected call raises the instruction hook's abort again, so a `pcall` or `xpcall` loop still unwinds to the block's guard, and an `xpcall` message handler does not run.

### Stopping the round in flight

The Host can [stop the round](01-what-a-prompt-is.md#effects-and-harness-work) in flight without canceling the run. Each `models.infer`, `models.loop` round, `tools.call`, tool call the model makes, and timed `tasks.join` or `tasks.join_any` that is waiting when the stop lands resumes with the canceled error. A wait on an inbox is none of these, so a stop leaves it waiting (see [What a stop leaves running](#what-a-stop-leaves-running)).

````markdown
---
name: patient
description: Returns a marker when the Host stops its model round.
runvil: 0
---

# Patient

## Draft

```lua
local ok, out = pcall(models.infer, 'Draft the summary.')
if not ok and out.kind == 'canceled' then
  return 'stopped'
end
if not ok then
  error(out)
end
return out
```
````

When the Host stops the round while `models.infer` waits, the run completes with this result:

````text
stopped
````

Left uncaught, the canceled error from a stop ends the run with code `canceled`, the same as a cancel, even though nothing canceled the run. A stop leaves the run's cancel flag clear, so running Lua goes on, and a block that catches the stopped call can return a result while the run completes, as `patient` does. A stop drops every tool call in flight except calls to tools that survive a stop, and the run goes on after the stop.

### The cancel check inside Lua

Running Lua checks the run's cancel flag every 10,000 instructions, a fixed interval, in each section VM's main code and in every block coroutine, so the check reaches every block of every section, the H1 pass included. The Engine also checks the flag between steps. A cancel takes effect at the next instruction check inside a running block, not only between blocks.

Cancellation reaches every block in a run, not only the first, because every block coroutine gets the same cancellation hook. Even pure Lua computation that calls nothing outside Lua is aborted and never returns, such as this block after a cancel:

````lua
for i = 1, 100000 do end
return "done"
````

A block aborted by a cancel ends with the canceled error, never an ordinary Lua runtime error, and the canceled error wins over any error value the block raises.

### Telling cancellations apart

The canceled error's `canceled` value has no `task` field, so the presence of `task` tells a [task cancellation](15-tasks.md#canceling-and-listing-tasks) apart from a stop or a run cancel, and so does `code`, which is `task_canceled` for a task cancellation and `canceled` otherwise.

A cancel has a code of its own, `canceled`: a run the Host cancels ends with the canceled error rather than as a failed run.

Whatever ends a run first is the result it reports, so a run that finishes or fails before a cancel lands keeps that result.

A `canceled` error value with no `task` field that escapes a block, including a caught one raised again later after another suspending call, maps back to the canceled error and ends the run with code `canceled` rather than as a failed run.

### Catching a stopped call

`pcall` catches the canceled error from a stopped call like any other error value. A stopped `models.infer` or `tools.call` gives `ok == false` and `err.kind == 'canceled'`, and `tostring(err)` gives the error's message. That is how `patient` returns `stopped` under a stop.

Under a cancel, the `pcall` around a waiting call never returns, so a block written to catch a stop and `return 'stopped'` still ends the run with the canceled error. A prompt therefore need not tell a stop from a cancel.

### Canceling model calls, loops, tools, and fanout

On cancellation, the model reply in flight is dropped and never returns to Lua, and a canceled `models.infer` does not report a model-round failure among the run's events. Canceling a run while a `models.infer` waits on a model reply ends the run with the canceled error, whether the call is in a section or in the H1 pass.

A running `models.loop` stays cancellable, because the loop runs as Lua inside the block and the instruction check keeps running while it does. A `models.loop` round cut short by a stop raises an error value of kind `canceled` and appends nothing to the message list, and a later `models.loop` call runs as usual.

A tool call in flight during a cancel is not waited out: its answer is dropped and the run ends promptly with the canceled error, even when the tool itself would never finish, both for a `tools.call` from Lua and for a call the model makes in a loop. Canceling a run aborts an [adhoc tool](12-tools.md#adhoc-tools-with-toolsadhoc) handler even when it is stuck in an endless Lua loop such as `while true do end`, and the run ends promptly with the canceled error. Canceling a run while a compactor is stuck in an endless Lua loop also ends the run promptly with the canceled error.

A run canceled before a fanout (`tasks.fanout`) begins ends with the canceled error without running any arms. Canceling a run while [fanout arms](14-fanout.md#what-an-arm-sees) are suspended on model calls ends the run with the canceled error.

### What a stop leaves running

A stop reaches only the calls waiting when it lands. A stop that lands while no call is waiting changes nothing, so the next call runs as usual.

A stop drops a tool's calls unless the tool's Pack marks it as surviving a stop; by default a tool does not survive a stop. A tool call that survives a stop still ends when the run ends.

A stop leaves every inbox alone, the main chain's and each task's. A [`tasks.receive()`](05-lua-environment.md#asking-the-operator) keeps waiting through the stop, and so does a task idle after [`tasks.reply`](15-tasks.md#answering-with-tasksreply), so the operator's next message, or the owner's, still reaches the prompt, and a stop never drops a message queued in any inbox. A message is not a stop either: it never raises `canceled`, and a message that arrives while a `models.loop` round waits steers the loop instead (see [Messages that arrive while the loop runs](11-conversations.md#messages-that-arrive-while-the-loop-runs)). When the Host stops the round and then sends a message, the stop comes first: the stopped round raises `canceled`, and the message stays queued for the chain's next read. A stop that cuts short a timed `tasks.join` or `tasks.join_any` does not cancel the tasks it waits on, so a later wait still joins them (see [Time limits on waits](15-tasks.md#time-limits-on-waits)).

## Gateway environment variables

After this section you can set the gateway environment variables a Host uses to reach models, and read the setup error a bad value gives.

````text
RUNVIL_GATEWAY_URL=http://127.0.0.1:8081/v1
````

With only this variable set, the Host builds its model connection with no API key, because `127.0.0.1` is a loopback host, and each model call goes to `http://127.0.0.1:8081/v1/chat/completions`.

The gateway environment variables are the two settings the Host can build the Harness's model connection from. A prompt never reads them, and a missing or invalid value is a setup error the Host sees before any run starts, so no prompt can catch it.

### The gateway URL

`RUNVIL_GATEWAY_URL` is always required. It holds the gateway API root, the base URL every model call is sent under, such as `http://127.0.0.1:8081/v1`. The value must be an absolute `http` or `https` URL with a host and no embedded credentials, query, or fragment. A trailing slash is dropped, so `http://host:8080/v1/` becomes `http://host:8080/v1`, and each model call goes to `{url}/chat/completions`.

### The gateway API key

`RUNVIL_GATEWAY_API_KEY` is required unless the host in `RUNVIL_GATEWAY_URL` is a loopback host. A loopback host is exactly `localhost` in any letter case, any address in `127.0.0.0/8` such as `127.0.0.1`, or `::1`. So `http://127.0.0.2:8081/v1`, `http://[::1]:8081/v1`, and `http://LOCALHOST:8081/v1` need no key, while `http://localhost.evil.com:8081/v1` does. An empty key counts as unset, and a key that is set is still sent to a loopback URL.

### Setup errors

When a required gateway environment variable is unset, or a required key is empty, the setup fails with `missing environment variable: {name}`, such as `missing environment variable: RUNVIL_GATEWAY_URL`. When a gateway environment variable is set to a value that is not valid Unicode, the setup fails with `environment variable is set but not valid Unicode: {name}`.

When the `RUNVIL_GATEWAY_URL` value breaks the URL rule, the setup fails with a message that starts `gateway URL` and names the broken rule, such as `gateway URL must use the http or https scheme`. The message never repeats the URL.
