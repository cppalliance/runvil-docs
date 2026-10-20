# Limits and errors

This chapter gathers every run limit, default, and failure kind in one place, and shows how to read an error to find what went wrong and where. It covers what each limit allows and who can change it, the kinds a failure carries from the parser, from Lua, and from the run, and how a cancel or a stop ends a run. With this vocabulary you can keep a prompt inside its budgets, catch the failures it can recover from, and go straight to the line that broke.

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

Even this small run observes a full set of run limits. The run limits are the bounds every run works within: how many model rounds one loop call may take, how many tasks run at once, how large a model reply may be, how much memory and logging each section's Lua may use, and how long a wait for a model reply may last. A model round is one request to a model and its reply. One set of run limits holds across the whole run, and those limits are the built-in defaults unless the [Harness](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) that runs the prompt sets its own values for that run.

### The run limits in brief

Each run limit has its own default and its own owner, and [Run limits and their defaults](#run-limits-and-their-defaults) gives the numbers:

- The [round cap](11-conversations.md#tool-rounds-and-the-round-cap), the most model rounds one `models.loop` call takes.
- The [concurrency limit](14-fanout.md#concurrency-and-the-admission-window), the most tasks that run at once.
- The response cap, the largest model reply a run accepts.
- The memory ceiling, the most Lua heap each [section VM](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals) may use.
- The log event quota, the most `log` calls each section VM records, together with the log byte quota that follows from it.
- The receive timeout, the longest wait for the next part of a model reply.

### The failure families

A failure belongs to one of four failure families, and each family has its own vocabulary:

- A parse failure is found before anything runs, and it has a parse error kind: `Frontmatter`, `Structure`, `Fence`, `List`, or `Lua`, plus `Internal` for a fault in the parser itself. See [Parse errors and their kinds](#parse-errors-and-their-kinds).
- A failure inside Lua is an [error value](05-lua-environment.md#error-values), the table an Engine failure raises in Lua, whose `kind` is one of thirteen lowercase error kinds, such as `lua`, `tool_loop_exhausted`, or `internal`. See [Error kinds in Lua and the run error they become](#error-kinds-in-lua-and-the-run-error-they-become).
- A failed run has a [run error kind](04-how-a-prompt-runs.md#the-run-result-and-outcomes), the stable name of what failed. See [Run error kinds](#run-error-kinds).
- A Host cancel ends the run with the cancelled outcome, which has no error kind. See [Cancellation and stops](#cancellation-and-stops).

A run ends in one of three run outcomes: it completes with a text result, as `steady` does, it is cancelled, or it fails. Only a failed run carries a run error kind.

### What a run error carries

A run that fails ends with a run error, the record of that failure. A run error carries exactly one run error kind naming what failed, because every Engine failure maps to exactly one run error kind. It also carries a human-readable message, the failing error's own text, that says what went wrong. Together the run error kind and the message form the typed contract of a failed run: the kind is exact, and each message always contains a fixed fragment that names the failure.

A run error also carries a retryable flag, which says whether the failure may be transient, so that running the prompt again could succeed. Only model call failures caused by transient transport or backend problems set the flag, and every other failure reports that a retry will not help. [Retryable failures](#retryable-failures) lists which failures count.

A run error can also carry a location, the place in a file that the failure points at. A parse failure in the frontmatter or in the structure of the prompt's body carries a location in the prompt file, an internal fault carries the Engine source file and line instead, and every other failure carries no location. [Where an error points](#where-an-error-points) covers each location field.

### Branching on a failure

Each part of a run error has its own job. Use the run error kind to branch on what failed, the message to show a reader what happened, and the location to find the fault.

The [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) decides how it shows the run error kind, the message, and, for a parse failure, the location to the person running the prompt. Workshop reports a parse failure under a code named by its parse error kind, one of `parse_frontmatter`, `parse_structure`, `parse_fence`, `parse_list`, or `parse_lua`, with the message prefixed `line {N}: ` when the parser finds the line. The Harness records a failed run as the kind's name plus the message with its chain of causes.

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
models:
  writer: {}
---

# Capped

## Ask

```lua
models.use('writer')
tools.offer('web/search')
local msgs = messages.new()
msgs:user('Find the tallest building in Paris, then answer in one sentence.')
local ok, err = pcall(models.loop, msgs)
if not ok then
  return err.kind .. ': ' .. err.message
end
return msgs[#msgs].content
```
````

When the model keeps calling the search tool through all three rounds, the run result is:

````text
tool_loop_exhausted: tool-call loop did not converge
````

When the model answers with text within its three rounds, the run result is that reply.

The `max_tool_iterations: 3` line sets this prompt's round cap to 3, so the [`models.loop`](11-conversations.md#starting-a-conversation) call in `## Ask` takes at most three model rounds. Reaching the round cap fails the loop call with an error value whose `kind` is `tool_loop_exhausted` and whose message is `tool-call loop did not converge`, and the block catches it with [`pcall`](05-lua-environment.md#raising-and-catching-errors) and returns both.

### The six run limits

A run observes six limits, each with a default:

| Limit | Default | Applies to | Who changes it |
|---|---|---|---|
| Round cap | 24 model rounds | each `models.loop` call or model handle `loop` call | the prompt's `max_tool_iterations`, or the Harness |
| Concurrency limit | 8 tasks at once | every task the run admits, fanout arms included | the Harness, and `tasks.concurrency` lowers it for a chain's own spawns |
| Response cap | 16 MiB | each model reply | the Harness |
| Memory ceiling | 64 MiB | the Lua heap of each section VM | the Harness |
| Log event quota | 1024 `log` events | each section VM | the Harness |
| Receive timeout | 120 seconds | each wait for the next part of a model reply | the Harness |

These built-in values are the default limits, the set every run starts from. The Harness that runs a prompt can replace any of the six defaults, so the limits a given run observes can differ from the ones in the table. One set of limits applies to the whole run, the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) and every section included, and it is the default set unless the Harness sets another.

Every count and size limit of a run is at least 1, so the round cap, the concurrency limit, the response cap, the memory ceiling, and the log event quota can never be zero.

### The one limit a prompt sets

`max_tool_iterations` is the only run limit a prompt's frontmatter can change (see [Optional keys and unknown keys](02-file-structure.md#optional-keys-and-unknown-keys)), and the other five limits come from the Harness that runs the prompt. A prompt's own `max_tool_iterations` value replaces the run's round cap. Without the key, the run's cap applies, which is 24 unless the Harness sets another.

The round cap allows 24 model rounds by default in each `models.loop` call or model handle `loop` call. Each call starts its own count at zero, so two loop calls in one section can each use up to the full cap.

Reaching the round cap fails the loop call with an error value whose `kind` is `tool_loop_exhausted` and whose message is `tool-call loop did not converge`, as `capped` shows. Without a `max_tool_iterations` key, this happens after exactly the run's default number of model rounds. Left uncaught, that error ends the run with run error kind `Tool`.

### Limits the Harness sets

The concurrency limit caps how many tasks run at once across the whole run, 8 by default. The Harness sets it, and no frontmatter key changes it. By default a run admits at most 8 tasks at once, fanout arms included; the Harness can set a different limit, and `tasks.concurrency` lowers it for a chain's own spawns.

Each model reply may be at most 16 MiB by default, and only the Harness can change that cap. Each wait for the next piece of a model reply times out after 120 seconds by default, and only the Harness can change that timeout. [Model request limits and failures](#model-request-limits-and-failures) shows how both bound a model round.

The memory ceiling and the log event quota bound each section's Lua, and [Lua memory and log quotas](#lua-memory-and-log-quotas) covers them.

### The call depth cap is not a run limit

The call depth cap of 8 is a fixed Engine constant, not one of the six run limits, so neither the Harness nor a prompt's frontmatter can change it. [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap) covers how that cap works.

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
lua: lua log event budget exceeded
````

Each [`log`](05-lua-environment.md#logging-and-lifecycle-reports) call records one checkpoint, and each section VM records up to 1024 of them by default. The loop asks for 2000, so the 1025th call is refused. The refusal raises an error value of kind `lua` with the message `lua log event budget exceeded`, `pcall` catches it, and the block returns `err.kind .. ': ' .. err.message`. A section VM is the fresh Lua state that one section's blocks run in, and every limit in this section is counted per section VM.

### The memory ceiling

The memory ceiling is the most Lua heap a section VM may use. A block that allocates past it fails, so with a Harness ceiling of 4 MiB, a block that keeps allocating hits the ceiling and fails. A section VM runs its Lua under a default memory limit of 64 MiB, applied when the VM is built.

Each section VM may use at most 64 MiB (67,108,864 bytes) of Lua heap by default, and only the Harness sets this ceiling. Running out of it is an ordinary Lua error, not a quota error, because only the two log quotas raise quota errors. The ceiling also sets the size budget that `json.encode`, `json.encode_lines`, and a `var` write spend, as [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) describes.

### The log event quota and the log byte quota

The log event quota is the most `log` events a section VM's Lua can emit, 1024 by default. Each section VM records up to 1024 `log` checkpoints by default, every one-argument `log` call spends one, and only the Harness can change the count.

The log byte quota bounds the total size of those messages. Each section VM has 256 bytes of log byte quota for each `log` call the log event quota allows, so 262,144 bytes by default, counted in UTF-8 bytes across all of that VM's `log` calls. The Harness has no separate setting for the log byte quota, and changes it only by changing the log event quota.

### Each section VM gets its own limits

The memory ceiling and the log quotas apply to each section VM, not to the whole run, because every section builds a fresh VM and applies the limits to it. A section that spends most of its log events leaves the next section's allowance untouched.

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

A Lua block has no instruction-count limit, so it runs until it finishes, runs past the memory ceiling, runs out a log quota, or the run is cancelled. The eight-million-iteration loop in `counter` runs to the end and returns its count, and a long-running block like it completes instead of failing with a quota error, because no instruction ceiling applies.

No limit counts Lua instructions. The instruction count has no cap, and it is fixed rather than set by the Harness. Only the cancel check counts instructions, and it runs every 10,000 of them.

A runaway loop such as `while true do end` is legal too. It keeps running until a Host cancel stops it, within about 10,000 instructions, and in a run that is already cancelled the same loop stops at the first check. A Host cancel reaches every Lua block, a busy loop included, because each block gets its own cancel check, which fires every 10,000 instructions. So a block can run a long or infinite loop without hitting any instruction limit, and the run's cancel is the kill switch for a runaway loop (see [Cancellation and stops](#cancellation-and-stops)).

### Which budgets raise quota errors

Only the two log quotas raise quota errors. Running past the memory ceiling raises an ordinary Lua error, the same as any other failure, and the instruction count has no budget.

Caught as an error value, an exhausted Engine quota on Lua has kind `lua`. Left uncaught, running out of the log event quota or the log byte quota ends the run with run error kind `Quota`, with the message `lua log event quota exceeded` or `lua log byte quota exceeded`. Among the Lua budgets, only the two log quotas lead to run error kind `Quota`: running past the memory ceiling leads to `Lua`, and no instruction count can run out.

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

The loop never ends on its own, yet it does not hang the run, because the memory ceiling stops it with an error. The run fails with run error kind `Lua`, and its message contains the word `memory`.

An allocation past the memory ceiling is refused as an ordinary Lua error whose message mentions memory; it is not a quota error. The section VM is still torn down normally after a memory-ceiling error.

A memory-ceiling refusal raised inside an Engine function reaches `pcall` as an error value of kind `lua`, so `pcall` returns `false` and that error value. A refusal raised by Lua's own code, such as the `string.rep` calls in `hoarder`, is not turned into an error value, so a block that catches it should not count on reading `err.kind`. Left uncaught in a section, a memory-ceiling error ends the run with run error kind `Lua`. Left uncaught in the H1 pass, it ends the run with run error kind `RequirementsUnmet` instead (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)), and the message is the H1 gate notice followed by the Lua error text. The notice reads:

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

A `log` call past either quota raises an error instead of recording a checkpoint, and its message depends on whether the block catches it:

| Quota | Caught with `pcall`, kind `lua` | Left uncaught, run error kind `Quota` |
|---|---|---|
| Log event quota | `lua log event budget exceeded` | `lua log event quota exceeded` |
| Log byte quota | `lua log cumulative byte budget exceeded` | `lua log byte quota exceeded` |

With a Harness log event quota of 1, a library that calls `log('one')` and then `log('two')` fails on the second call, because the quota applies before the library code runs, and left uncaught the run ends with `lua log event quota exceeded`. The `chatty` prompt at the start of this section shows the caught form: under the default quota of 1024, its 1025th call is refused, and returning `err.kind .. ': ' .. err.message` makes the run result `lua: lua log event budget exceeded`.

### Reading a quota error

Left uncaught, an exhausted log event quota ends the run with run error kind `Quota`, and the remedy is to raise the matching ceiling or change the prompt so it needs less. A log quota failure in the H1 pass still ends the run as `Quota`, unlike a Lua error there, which becomes `RequirementsUnmet`. The uncaught message names no prompt line, and the run error carries no location.

Left uncaught, every Lua quota error ends the run with run error kind `Quota`, and the only quotas a block can run out are the log event quota and the log byte quota. Caught as an error value, a Lua quota error has kind `lua`, the same kind as an ordinary Lua runtime error, rather than a kind of its own. The error value has no field naming which quota ran out; only its message tells the two apart, `lua log event budget exceeded` or `lua log cumulative byte budget exceeded`.

A Lua quota failure does record which quota ran out, and its uncaught message names it as `log event` or `log byte` in the form `lua {resource} quota exceeded`. A quota refusal that ends a block always surfaces as the quota error itself, ahead of any error value the block kept or raised, and it carries no prompt line.

A `log` call that exceeds the log event quota fails the block with a log event quota error, and the Engine's refusal reads `lua log event budget exceeded`. Exceeding the log byte quota fails the block with a log byte quota error, a stable quota failure rather than a plain Lua error string. Raising the exact text of a refusal yourself, as in `error('lua log event budget exceeded')`, still produces an ordinary runtime error, because only the Engine's own refusal counts as a quota error.

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
lua compilation error at section `Transform` epilog (line {line}): {diagnostic}
````

Here `{line}` is the file line where the block's Lua starts, and `{diagnostic}` is the Lua compiler's own text.

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
lua compilation error at {location} (line {line}): {diagnostic}
````

`{location}` names the block, `{line}` is the file line where the block's Lua starts, and `{diagnostic}` is the Lua compiler's own text. A section block's location takes one of the first three forms below, where the `lua {n}` form names a numbered block between the prologue and the epilog, and a block in the H1 body takes the last form, with the H1 title:

````text
section `{name}` prologue
section `{name}` epilog
section `{name}` lua {n}
H1 `{title}` lua
````

### Which problem the parser reports

The frontmatter is decoded before the body is checked, so a frontmatter failure is reported ahead of any body failure. When a prompt has several problems, the parse reports only the first one it meets, in this order:

1. The frontmatter delimiters.
2. YAML decoding and the contract checks inside it.
3. Duplicate Pack ids.
4. The H1's presence and count.
5. An empty H1 title.
6. The fence rule.
7. The H1's blocks and their Lua.
8. Each section, in document order.

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

A frontmatter key or value the prompt contract rejects fails with `Frontmatter` and that key's own message, with the line and column. That covers an unknown entry field, a bad arg name, type, or default, a malformed `uses:` entry, an unknown model keyword, a role label outside the [prompt-local alias grammar](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-local-tool-aliases), a zero `min_context`, and an out-of-range `max_tool_iterations`.

A file whose frontmatter has no `runvil:` key still parses, but running it fails before any section runs, with run error kind `Parse` and this message:

````text
not a runvil prompt: no runvil version
````

### Heading messages

Each `Structure` failure has its own message:

````text
prompt requires an H1 title
prompt must contain exactly one H1 title, found {count} at lines {lines}
prompt H1 title must not be empty
section `{name}` is an orphan H{level} heading with no parent H{n}
an H{level} section heading must not be empty
duplicate sibling section name `{name}`: first declared at line {first_line}, again at line {line}; sibling section names must be unique
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

The `{label}` depends on where the fence sits. It is the first form below for a fence that opens its section, the second for the last fence, and the third for any other, and `{name}` is the H1 title for a fence in the H1 body:

````text
section `{name}` prologue `lua`
section `{name}` epilog `lua`
section `{name}` `lua`
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

After this section you can read a parse failure's location, with its path, line, column, and byte span, and know which failures carry each field.

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

The run fails with run error kind `Parse`, and the failure carries this location:

| Field | Value |
|---|---|
| Path | `dup`, the prompt's frontmatter `name` |
| Line | 13 |
| Column | 1 |
| Byte span | the range of the second `## S` heading |

### Location fields

When the parser can point at the problem, a parse failure carries a location, and the Host reports it beside the message as a path, a line, a column, and a byte span, each when known. A location is that set of fields, and a byte span is the range of bytes the offending region covers. A failure that can locate the offending region, such as a duplicate sibling section, reports a byte span, and every other failure reports none. The failures that can locate their region are the heading checks and the fence rule checks in the prompt body.

A failure with run error kind `Parse`, whether it comes from the frontmatter or from the body, carries a location in the prompt source. A parse error's line is the line in the prompt file as written, and you use it together with the column, not the byte span, to point at the mistake.

Error locations report 1-based line numbers and 1-based column numbers. A failure's line and column are 1-based when known, and an internal fault never has a column.

### Counting lines from the top of the file

Parse locations use 1-based lines counted from the top of the file, the frontmatter included, so the opening `---` frontmatter delimiter is line 1. A bad `uses:` entry on the fifth line of the file is reported at line 5, and with frontmatter on lines 1 to 4 a `## S` heading on file line 8 is reported at line 8. Line numbers in body parse errors count from the top of the file, with the frontmatter lines included, not from the start of the body.

### The prompt name or `<prompt>`

A parse failure's location names the prompt by its frontmatter `name` when parsing gets that far, and falls back to the `<prompt>` placeholder otherwise, so you can tell which prompt failed. The `<prompt>` placeholder is the path a location shows when the prompt's name is not known, and the Host replaces it with its own label for the file.

Every `Structure`, `Fence`, or `List` failure found after the frontmatter reports the prompt's frontmatter `name`, so a prompt with `name: dup` reports `dup`, plus the line and column of its offending span when it has one. A frontmatter failure, a Lua compile failure, and an internal fault report no name.

A failure's location path is one of three things: the prompt's frontmatter `name`, the placeholder `<prompt>` that the Host replaces with its own label, or, for an internal fault, the path of an Engine source file. A `Structure`, `Fence`, or `List` failure has the prompt's frontmatter `name` as its location path, plus the 1-based line and column where the offending region starts when the parser can locate that region, and neither a line nor a column otherwise. The `dup` prompt shows the whole case: parse error kind `Structure`, location path `dup`, line 13, column 1, a byte span, and run error kind `Parse`.

### Which run errors carry a location

A run error can point back to a source location in the prompt file. A run error carries a source location only for parse and internal failures, and an internal fault points at a line of Engine source code rather than at the prompt. The source location a run error carries has the same location fields a parse error reports.

Only parse failures have a prompt location. An `Internal` failure, a fault in the Engine rather than a mistake in the prompt, names the Engine source file and line of the broken invariant instead, and every other run error kind reports no location.

A parse failure that happens before the prompt's `name` is known shows the placeholder `<prompt>` as its path.

### Frontmatter failure locations

A frontmatter failure carries no prompt name, because it happens before the prompt's `name` is read from the frontmatter itself. A missing delimiter, a YAML decode failure, and an error in the `uses:` list all report none, even one found after the YAML decodes, and the Host can use its own label for the file instead. A frontmatter syntax error reports no prompt name for the same reason.

A frontmatter failure's location path is the placeholder `<prompt>`, which the Host may replace with its own name for the file. A frontmatter YAML failure always shows `<prompt>` as its path, because it happens before the name is read.

````markdown
---
name: broken
description: Has a YAML slip on line 4.
: : :
runvil: 0
---

# Broken
````

This file fails with parse error kind `Frontmatter` and `invalid frontmatter: {message}`, at line 4 with a column, path `<prompt>`, and no byte span.

A frontmatter YAML failure has no byte span, so its only location is its line and column, and it never reports a span. A frontmatter failure's line and column come from the YAML error. An invalid-YAML failure reports the 1-based file line that the YAML parser gave, and reports no line when the YAML error has no location.

The YAML decoder counts lines from the start of the frontmatter, so the reported line is the decoder's line plus one, to account for the opening `---`. The column passes through unchanged, and each is absent when the decoder gives no position or gives zero. That is why a YAML slip such as `: : :` on the fourth line of the file is reported at line 4, which is 1-based and counts the opening `---` as line 1. The 1-based file column of the YAML error comes alongside the line, when the column is known.

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

A Pack id declared twice under `uses:`, as in `twice`, gives neither a line nor a column, because that check compares the entries after the YAML decodes, when no source position is kept. A missing opening `---` or an unclosed frontmatter block also gives no line or column, and its location path is the placeholder `<prompt>`; the messages read `file must begin with a --- frontmatter delimiter` and `frontmatter was not closed with ---`.

### Body failure locations

Any other parse failure's line and column come from the span of the offending region. A structural failure's line is the 1-based line in the whole file where its span starts, counting the frontmatter lines above the body.

A duplicate sibling heading error reports the 1-based line of the offending heading, counted from the top of the file including the frontmatter, so with frontmatter on lines 1 to 4 and the second `## S` on line 12, the error is at line 12. Its column is the 1-based column of the offending heading, which is column 1. The error keeps the byte span of the offending source alongside its line and column, as `dup` shows.

Line numbers inside heading error messages also count from the first line of the file, frontmatter included. With frontmatter on lines 1 to 4, H1 titles on lines 6, 12, and 16 give the message `prompt must contain exactly one H1 title, found 3 at lines 6, 12, 16`, and a repeated `## S` whose first copy is on line 8 gives a message containing `first declared at line 8` and `again at line 12`.

Orphan and empty heading errors report the heading's file-absolute line, column 1, and a byte span. Every heading `Structure` error, for an orphan, empty, or duplicate sibling heading, carries a non-empty byte span that points at the heading itself.

A `Fence` error for a bad fence label reports the line where the offending fence opens, at column 1, so a fence whose opening line carries extra text after `lua` on line 12 is reported at line 12, column 1. A `Fence` error for a broken fence rule carries a byte span, but a `lua` fence left open, which fails with the not-closed message, carries no span, line, or column.

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

### Byte columns

A structural failure's column is a 1-based byte column, so each multibyte UTF-8 character before the error on that line counts as more than one column. A body failure's column counts bytes, so a multibyte UTF-8 character earlier on the line pushes it past the character count.

### Failures with no line or column

Only a parse failure can report a byte span of the offending region. A failure's byte span is set only for structural parse failures in the prompt body, never for a frontmatter failure or an internal fault. A structural failure without a byte span reports no line and no column.

Many parse failures carry no line or column, among them H1 title failures (missing, repeated, or empty), rule failures in otherwise valid frontmatter such as a Pack id declared twice, and Lua compile failures. A Host therefore shows the line and column when present, and otherwise the message.

A `List` failure never gives a line or column, because the list check records no span, so its message names the section instead:

````text
empty bullet item in list section `{name}`
````

An `Internal` parse failure, a fault in the parser itself, reports no span, name, line, or column.

### Lua compile failure positions

A Lua syntax error's position is inside its message, in the form `lua compilation error at {location} (line {source_line}): {message}`, because the Lua compiler's diagnostic passes through word for word and the separate name, line, and column details are empty. A body failure's message, by contrast, is the plain diagnostic with no added prefix, and the name, line, and column are separate details beside it.

### What a byte span counts

A parse error's byte span counts offsets into the body after the frontmatter and any leading [byte-order mark](02-file-structure.md#frontmatter-delimiters-byte-order-mark-and-line-endings), with CRLF read as LF, so it does not index the original text. Use the line and column to find the failure in the original file instead.

Body line numbers continue past the frontmatter: the first line after the closing `---` is numbered one more than the frontmatter block's line count, both `---` delimiters included. With a frontmatter block on lines 1 to 4, the body's first line is line 5.

## Run error kinds

After this section you can name the run error kind a failed run reports and the cause behind each one.

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

Change the version line to `runvil: 1` and run the prompt again. The run fails with run error kind `Version` and this message:

````text
unsupported runvil version: 1 (this build supports major 0)
````

A run error kind, as [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes) introduces it, is the stable name a failed run reports for what failed. Every failed run reports exactly one, which classifies the failure by what failed, and the run outcome shows that kind by its name, such as `Parse` or `Lua`.

### The twelve run error kinds

A failed run reports one run error kind from a fixed set of twelve:

| Run error kind | What causes it |
|---|---|
| `Parse` | the prompt fails to parse, a Lua block does not compile, or the frontmatter has no `runvil:` key |
| `Version` | the `runvil:` key declares a major version other than `0` |
| `Binding` | `models.infer` or `models.loop` runs with neither `models.use` nor `models.default` in effect |
| `Completion` | a model call fails, or an `empty_model_reply` goes uncaught |
| `Tool` | a tool fails, is out of scope or unknown, or the tool-call loop reaches its round cap |
| `Vfs` | a `store` operation fails uncaught, the run declares no store, or the store backend fails as the run starts |
| `Determinism` | a store call meets a claims conflict |
| `Lua` | a section's Lua fails at run time or returns an unusable value, or a Pack prelude cannot install |
| `Quota` | a log quota runs out, or a call nests past the call depth cap |
| `ContextExhausted` | a request overflows the model's context window and the compactor leaves it over |
| `Internal` | an Engine invariant breaks |
| `RequirementsUnmet` | the environment falls short of a requirement the prompt states |

The Engine also defines `Substitution` and `Cancelled`, but no failed run reports either. A failed `{{ }}` substitution is an ordinary Lua error that `pcall` can catch, and it reports `Lua` when uncaught. A cancelled run ends as cancelled, never as a failure of kind `Cancelled`.

### The failed run's message

A failed run's message is the underlying error's own text followed by the text of each cause in its chain, joined by `: `, and any cause whose text already appears in the message is left out so no text repeats. The kind never replaces the message, and the message stands on its own.

### `Parse` and `Version`

A run fails with kind `Parse` when the prompt fails to parse, including when one of its Lua blocks has a syntax error and does not compile. A prompt that fails to parse ends its run as failed with kind `Parse` before any section or code runs, before input staging and before the wait for Packs, and the recorded message is the parse error followed by its chain of causes.

Any parse failure, whatever its parse error kind, ends the run with kind `Parse`, including a file with no `runvil:` key, whose parse error reads `not a runvil prompt: no runvil version`. The run carries the parse failure's own message and its location when known: the path is the frontmatter name or `<prompt>`, with line, column, and span when known. A `runvil:` key with an unsupported version reports `Version` instead.

A run fails with kind `Version` when the prompt's [`runvil:` key](02-file-structure.md#the-runvil-version-key) declares a major version other than `0`, and the message is `unsupported runvil version: {n} (this build supports major 0)`, as `future` shows.

### `Binding`, `Completion`, and `Tool`

A run fails with kind `Binding` when a section calls [`models.infer`](10-models.md#running-a-round-with-modelsinfer) while neither `models.use` nor a prompt-wide `models.default` is in effect, and the message is `model binding required for section {section}`. A [`models.loop`](11-conversations.md#how-the-loop-runs) call in such a section raises the same [missing-model error](10-models.md#infer-errors-and-the-missing-model-error) and fails the run the same way. Prose by itself never requires a model; only an explicit `models.infer` of it does.

A run fails with kind `Completion` when a model call fails at the transport, backend, or decode layer, such as an overloaded backend, or when an `empty_model_reply` error goes uncaught, and the message is the model call's own error text. [Model request limits and failures](#model-request-limits-and-failures) lists those messages.

A run fails with kind `Tool` when a dispatched tool fails, the model calls a tool outside the section's [tool scope](12-tools.md#tool-scope-and-the-task-built-ins), a `tools.call` names no local tool and no tool in the run's catalog, or the tool-call loop reaches its round cap before the model gives a final reply (see [Tool errors](12-tools.md#tool-errors)).

### `Vfs` and `Determinism`

A run fails with kind `Vfs` when a `store` operation fails and goes uncaught, or when the run declares no store to operate on (see [When a store call fails](09-the-store.md#when-a-store-call-fails)). A store backend that fails as the run starts also ends the run with `Vfs`, even when the prompt never calls `store` (see [Host stores, policies, and modes](09-the-store.md#host-stores-policies-and-modes)). The kind is named `Vfs` even though the Lua global is `store`. An uncaught `store.read('missing.txt')` fails the run with `Vfs` and a message that includes `file not found in store: missing.txt`, and a store operation in a run that declares no store fails with `Vfs` and a message that includes `this run declares no store, so no store operation can run`.

A run fails with kind `Determinism` when a store call meets a claims conflict (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)). Any claims conflict ends the run on the spot, even in the H1 pass, so no `pcall` can catch it, not even a `pcall` a library runs while it loads, and no task or arm counts as failed.

### `Lua` and `Quota`

Run error kind `Lua` means a section's Lua fails at run time or returns an unusable value, and the message is the Lua error's own text. It covers an uncaught Lua failure in a walked section, a `call` chain, a task, a `tasks.fanout` arm, or a library loaded with `require`, from any of these causes:

- an uncaught runtime error
- running past the memory ceiling
- a failed `{{ }}` substitution (see [Placeholders in prose](07-substitution.md#placeholders-in-prose))
- an uncaught `tasks_live`, a leaked task
- `task_not_owned`, a task reached by a chain that does not own it
- `task_consumed`, a result waited on twice
- a block that returns a table, whose message includes `cannot return a table as a result`

A [Pack prelude](12-tools.md#pack-preludes) that cannot install also fails the run with `Lua`, before any section code runs.

A run fails with kind `Quota` when a section's Lua runs out its `log` event quota or its `log` byte quota, or when a `call`, `tasks.spawn`, or `tasks.fanout` arm would nest past the call depth cap. No instruction-count quota runs out in practice. A `Quota` failure's message is one of these:

````text
lua log event quota exceeded
lua log byte quota exceeded
call recursion exceeded cap of {cap}
tasks.fanout recursion exceeded cap of {cap}
````

A `tasks.spawn` past the cap reports as `call`, because it shares `call`'s cap. With a cap of 8, the message reads `call recursion exceeded cap of 8`.

### `ContextExhausted`, `Internal`, and `RequirementsUnmet`

A run fails with kind `ContextExhausted` when a request overflows the model's context window, either on the check before sending or at the provider, and the selected [compactor](11-conversations.md#compactors-and-context-exhaustion) leaves it over the limit. The message is `context exhausted: {reason}`.

Run error kind `Internal` means an Engine invariant breaks, a fault in the Engine rather than in the prompt. That includes Harness mistakes such as an answer of the wrong kind, an answer for an id never issued, or a second answer to one effect. The message starts with `internal invariant violated: {message}`, and the location names an Engine source file and line rather than a place in the prompt.

A run fails with kind `RequirementsUnmet` when the environment falls short of a requirement the prompt states: a required Pack is missing, unavailable, or needs a service the run lacks, the bound model fails a declared requirement such as a context minimum or a hard keyword, or an H1 block fails the prompt's [hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate).

### Cancelled runs and preparation failures

A run the Host cancels ends as cancelled rather than failed, so it reports no run error kind.

A frontmatter parse failure ends the run with kind `Parse`, and so does a structural parse failure in the prompt body. A run that fails during [preparation](04-how-a-prompt-runs.md#preparing-the-run) ends as `Parse` when the prompt does not parse, `Input` when its input cannot be staged, or `RequirementsUnmet` when the environment refuses it, and its message is the full chain of causes.

Every input refusal, such as input text given to a prompt that declares no input or a store failure while staging the input, ends the run as failed under the Harness's `Input` kind, which sits outside the twelve run error kinds, and the message is the full chain of causes.

### Kinds in the H1 pass and through calls

The `Lua` kind holds alike in a walked section, a `call` chain, a task, and a `tasks.fanout` arm. An ordinary Lua error in the H1 pass ends the run as `RequirementsUnmet` instead, as a failed hard gate, while any other failure in the H1 pass keeps its own kind.

Passing the call depth cap through `call`, `tasks.spawn`, or `tasks.fanout` ends the run as `Quota`, not `Lua`, even though a `pcall` that catches the same error reads its kind as `lua`. When a `call` or a task wait fails and the prompt does not catch it, the run ends under that failure's own run error kind rather than as a generic `Lua` failure, such as `Quota` for the call depth cap, because the error passes up through every parent without being flattened.

## Error kinds in Lua and the run error they become

After this section you can read the error kind of a caught failure and predict the run error kind the same failure reports when no `pcall` catches it.

````markdown
---
name: summary
description: Turns a failed model call into the section's result.
runvil: 0
models:
  writer: {}
---

# Summary

```lua
models.default('writer')
```

## Draft

Draft the summary.

```lua
local ok, reply = pcall(models.infer, prose)
if ok then
  return reply
end
if reply.kind == 'internal' then
  return 'skipped: ' .. reply.message
end
error(reply)
```
````

When the model call succeeds, the run result is the model's reply. When it fails outside the prompt, for example because the backend cannot answer, the run result is `skipped: ` followed by the failure's message. Any other failure is raised again unchanged, so the run ends exactly as it would have without the `pcall`.

### Thirteen error kinds

Every Engine failure that reaches Lua carries exactly one of thirteen error kinds in its error value's `kind` field: `lua`, `context_exhausted`, `empty_model_reply`, `cancelled`, `tool_loop_exhausted`, `tasks_live`, `task_not_owned`, `task_consumed`, `out_of_scope_tool`, `unbound_tool`, `tool`, `store`, and `internal`.

Error kinds and run error kinds are separate vocabularies. An error kind is what Lua sees at the call, and a run error kind is what the Host reports when a failure ends the run. One error kind can lead to different run error kinds depending on its source, as `internal` does when it ends the run as `Completion`, `Binding`, or `Internal`.

### The `internal` error kind

The `internal` error kind marks a failure the prompt cannot fix. A failure you cannot act on, such as a violated internal invariant or a transport, backend, or configuration failure outside the prompt, reaches Lua as kind `internal`, while a store failure has its own kind `store`.

An `internal` error value has no extra fields. Left uncaught, it ends the run as `Completion` for a model call failure, `Binding` for the [missing-model error](10-models.md#infer-errors-and-the-missing-model-error), or `Internal` for an Engine fault.

When an Engine function fails and `pcall` catches it, the error kind is `lua` for an authoring or argument problem, such as a runtime error, a syntax error, a memory error, a bad argument, or a value that cannot be converted or serialized. It is `internal` when the Lua runtime's own machinery fails.

### Uncaught failures keep their kind

An uncaught failure ends the run with its kind, message, and fields intact, never flattened to a message string, so the run is classified the same as the original error.

### Raising a caught error value again

A caught error value raised again with `error(err)` before the block makes another [suspending call](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), a call that waits on the Harness, ends the run exactly as if it had never been caught, with the same run error kind (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). A caught error value raised again after another suspending call is rebuilt from its `kind`, and that `kind` decides the run error kind.

That is what `summary` relies on. A section can turn a caught `internal` failure, such as a model call the Harness cannot complete, into its result, and raise every other kind again with `error(reply)` before any other suspending call, so the run ends exactly as it would have without the `pcall`.

A failure outside the prompt, such as a model call's transport, backend, or decode failure, a disabled gateway, the missing-model error, or an Engine fault, raises an `internal` error value. An empty model reply is the one case that differs, and it raises `empty_model_reply` instead. This holds inside a `tasks.fanout` arm too, where an arm's `models.infer` with no model access configured surfaces the disabled-gateway failure.

Left uncaught in a walked section, a [`models.loop`](11-conversations.md#loop-arguments-and-errors) failure ends the run like this:

| Error kind from `models.loop` | Run error kind when uncaught |
|---|---|
| `lua` | `Lua` |
| `empty_model_reply` | `Completion` |
| `context_exhausted` | `ContextExhausted` |
| `tool_loop_exhausted`, `out_of_scope_tool` | `Tool` |
| `internal` | `Completion` for a backend or transport failure, `Binding` for the missing-model error |

Raised again later, after another suspending call, each error kind ends the run like this:

| Error kind raised again later | Run error kind |
|---|---|
| `empty_model_reply` | `Completion` |
| `context_exhausted`, with its `reason` field | `ContextExhausted` |
| `tool_loop_exhausted`, `tool` | `Tool` |
| `store` | `Vfs` |
| `task_not_owned`, `task_consumed`, with their `task` field | `Lua`, in the H1 pass too |
| `out_of_scope_tool`, `unbound_tool`, `tasks_live`, `lua`, `internal` | `Lua`, or `RequirementsUnmet` in the H1 pass |

An `empty_model_reply` value raised again later keeps its message. An `out_of_scope_tool`, `unbound_tool`, `tasks_live`, `lua`, or `internal` value raised again later is rebuilt as a plain Lua failure that keeps only its message.

A value missing its kind's fields is rebuilt as a plain Lua failure too. A `context_exhausted` value without a valid `reason`, or a `task_not_owned` or `task_consumed` value without a valid `task`, raised again later ends the run as `Lua`, or as `RequirementsUnmet` in the H1 pass. A `store` value without a recognized `reason` still ends the run as `Vfs`. A `cancelled` value raised again later ends the run with the cancelled outcome, as [Cancellation and stops](#cancellation-and-stops) explains.

### Task, alias, and heading errors

When a task error ends the run uncaught, in any chain and in the H1 pass too, the run error kind is `Lua`. That covers a leaked task, a task reached for by a chain that does not own it, a result waited on twice, and a `cancelled` value re-raised right after the wait (see [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment)).

Left uncaught, an invalid alias error ends the run as `Lua`, or as `RequirementsUnmet` when it reaches the H1 body's own Lua. Its message reads:

````text
invalid alias "<alias>": expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

A valid alias starts with a letter and continues with at most 63 letters, digits, underscores, or hyphens.

Left uncaught, a heading-not-found error or a malformed heading reference error ends the run as `Lua`, or as `RequirementsUnmet` when it reaches the H1 body's own Lua (see [Resolving heading references](08-jump-and-call.md#resolving-heading-references)). The not-found message reads:

````text
section heading `<heading>` not found; available sections: <list>
````

Only the H1 body's own Lua code remaps this way. The machinery around it, namely section setup, the final `var` read-back, and jump-target resolution, keeps its own kind.

## Model request limits and failures

After this section you can catch a failed model call, read its completion error kind and message, and know how the response cap and the receive timeout bound each round.

````markdown
---
name: summary
description: Turns a failed model call into the section's result.
runvil: 0
models:
  writer: {}
---

# Summary

```lua
models.default('writer')
```

## Draft

Draft the summary.

```lua
local ok, reply = pcall(models.infer, prose)
if ok then
  return reply
end
if reply.kind == 'internal' then
  log('model call failed')
  return 'skipped: ' .. reply.message
end
error(reply)
```
````

This is `summary` from the previous section with one `log` line added. When the backend answers with status 503, the run result is:

````text
skipped: the model backend is overloaded (status 503)
````

The block catches the model call failure, checks for kind `internal`, logs a checkpoint, and returns `skipped: ` plus the error's message.

### Completion errors and how Lua sees them

A model call that fails surfaces as a completion error, and every completion error carries a completion error kind that names what went wrong, such as `Overloaded` or `Timeout`. When the model request for a `models.infer` round fails, the `models.infer` call raises that failure as its error.

When the prompt catches a model call failure, the error value has kind `internal`, whether the cause is a connection failure, a receive timeout, a backend error status, a malformed or oversized reply, or a disabled gateway. An empty reply is the one case that differs, with kind `empty_model_reply` instead. A model call failure that the prompt does not catch ends the run with run error kind `Completion`, and the run's message is the failure's own message, such as `the model backend is overloaded (status 503)`.

A model failure caused by a backend HTTP status ends its message with ` (status N)`, where N is the status code, after the kind's fixed phrase. That is how `summary` returns `skipped: the model backend is overloaded (status 503)` when the backend answers with status 503.

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

### The response cap and the receive timeout

The response cap is the largest model reply a run accepts, 16 MiB by default, and the receive timeout is the longest wait for the next part of a model reply, 120 seconds by default. Both apply to every model round in the run, both `models.infer` rounds, nested ones included, and `models.loop` rounds, so an oversized or stalled reply ends that round with an error instead of hanging the run.

The receive timeout bounds each wait for the next part of a model reply, first the headers and then each body chunk. Every chunk that arrives restarts the wait, and no limit covers the whole request, so a reply that keeps streaming is never cut off.

### What a model failure message holds

A caught model call failure carries no fields beyond `kind` and `message`, and its message never includes the backend's reply body, so a hostile or private payload cannot leak into a message or forge a log line.

### Connection, timeout, and malformed-reply failures

A failed connection, a broken stream, or an error reply whose body cannot be read fails the round with completion error kind `Transport`, and the failure is retryable. A model reply whose headers or next chunk do not arrive within the receive timeout fails the round with completion error kind `Timeout`, and the failure is retryable. Their messages are:

````text
the connection to the model backend failed
the model backend did not answer in time
````

A reply that cannot be understood fails the round with completion error kind `MalformedResponse`, and the failure is retryable. The message is a fixed phrase followed by what is wrong, as in the second line here:

````text
the model backend sent a reply that could not be understood: {what was wrong}
the model backend sent a reply that could not be understood: completion stream ended without the [DONE] sentinel
````

When a `length` or `content_filter` [finish reason](11-conversations.md#empty-replies-and-finish-reasons) cuts off a batch of tool calls, the round fails as `MalformedResponse` with the fixed phrase followed by `: tool-call batch truncated by finish_reason "length": partial arguments must not execute`, or `"content_filter"` in place of `"length"`, so no partial tool arguments run.

A model reply that would pass the response cap is refused as its bytes arrive, before any of it is decoded, as `MalformedResponse`, and the cap covers error replies as well as successful ones. The fixed phrase ends one of three ways:

````text
the model backend sent a reply that could not be understood: response stream exceeds the {max_bytes}-byte limit
the model backend sent a reply that could not be understood: response body exceeds the {cap}-byte limit
the model backend sent a reply that could not be understood: response body of {len} bytes exceeds the {cap}-byte limit
````

The first is a successful reply over the cap, the second an error reply over the cap, and the third an error reply whose advertised length is already over the cap. A response-cap or receive-timeout failure is kind `internal` when caught and run error kind `Completion` when uncaught, and it counts as transient, so running the prompt again may succeed.

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

A `models.infer` call raises a `ContextOverflow` failure as kind `internal`, while a `models.loop` round hands it to the compactor instead (see [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion)).

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

An uncaught `empty_model_reply`, including one the prompt caught and raised again, ends the run with run error kind `Completion` and keeps its message.

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
models:
  writer: {}
---

# Brief

```lua
models.default('writer')
```

## Write

Write a one-line brief.

```lua
return models.infer(prose)
```
````

When the backend answers with status 503, the run fails with run error kind `Completion` and this message, and the run error's retryable flag is set:

````text
the model backend is overloaded (status 503)
````

When the backend answers with status 429 and a body that names a spent quota, such as `insufficient_quota`, the run fails as `Completion` with this message, and the retryable flag is clear:

````text
the model backend says the usage quota is spent (status 429)
````

Both runs fail with the same run error kind, but only the first may succeed when you run the prompt again.

### What makes a failure retryable

A failed run is retryable only when a model call fails with a transient transport or backend problem, and that is what a retryable failure means: running the prompt again may succeed. A `Completion` failure is retryable when it comes from a transport failure (a receive timeout included), a malformed or oversized reply, a rate limit (backend status 429), an overloaded backend (status 503 or 529), or any backend status from 500 through 599. The failure reports the backend status when there is one, and nothing reruns the failed run automatically.

### Failures that are never retryable

Every failure other than a transient model call failure is never retryable:

- frontmatter and body parse failures
- an interrupted run
- Lua errors, runtime and compile errors included
- schema binding failures
- substitution failures
- an exhausted tool loop
- task failures: tasks still live, a task the prompt does not own, a task already consumed, or a cancelled task
- a tool call outside the prompt's scope or to an unbound tool
- a required model that is missing
- an unsupported version
- unmet requirements
- tool failures
- internal failures
- a spent Lua quota
- an exceeded call depth
- an exhausted context
- store failures
- determinism failures

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

When the model answers a `models.infer` round with tool calls although none are advertised, the call fails as a malformed reply (kind `MalformedResponse`) with the detail `the round sent tool calls, but no tools were advertised`. The failure is retryable, because a reply that breaks protocol may come back well formed on another try. When that failure ends the run, the run error keeps the retryable flag, so the run outcome reports a retryable failure.

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

Nothing in `spinner` ends the loop, and no limit would. When the Host cancels the run, the section's Lua instruction hook sees the cancel as the code runs, the loop is aborted, and the run ends with the cancelled outcome.

### Cancelling a run

The Host can cancel a run from any thread through the run's cancel handle. It can cancel at any time, even while Lua is busy in a loop or waiting on a model reply, and the run's next step tears it down wherever it is.

A run the Host cancels, before it starts or while it runs, ends with the cancelled outcome, the run outcome of a cancelled run. The cancelled outcome carries no message and no run error kind, and a cancelled run never ends as a failure. Inside the run, a cancel surfaces as the interrupted error, whose kind in Lua is `cancelled`, and that error ends the run with the cancelled outcome.

Once a run is cancelled, it issues no further model rounds or tool calls. Its outcome is decided at the next step and reported once the Host answers every request still out, whether a model reply, a tool call, or a timer, as dropped.

### The cancel flag

The cancel flag is the one signal a cancel sets. One cancel flag is shared by the Lua instruction hook in every section VM, by every activated [Pack](12-tools.md#tools-packs-and-the-uses-key), and by the run itself, so one cancel reaches them all. A cancel stops a run even in the middle of a running Lua block, because the Engine checks the cancel flag between chain steps and from the Lua instruction hook.

A Host cancel stops a running Lua block promptly, even a tight endless loop such as `while true do n = n + 1 end` that nothing in the prompt and no limit would end, because the section's Lua instruction hook checks for cancellation as the code runs, and the run ends with the cancelled outcome, as `spinner` shows. Cancelling a run aborts a Lua block that is still running, even a tight `while true do end` loop, and even when the run is cancelled before the block starts; the block ends with the interrupted error.

### What a cancel does to waiting calls

A cancel never resumes a call that is waiting on the Harness, such as `models.infer`, a `models.loop` round, `tools.call`, or a timed wait. The next step tears the waiting block down, the Harness answers each call still out as dropped, and the run ends with the cancelled outcome, so a `pcall` around the call never returns.

### The interrupted error

The interrupted error is the error value a cancel or a stop raises in Lua. It has kind `cancelled`, the same message whether a cancel or a stop causes it, and no fields beyond `kind` and `message`:

````text
interrupted: the run was cancelled or this call was stopped
````

Your code cannot swallow a cancel. Every protected call raises the instruction hook's abort again, so a `pcall` or `xpcall` loop still unwinds to the block's guard, and an `xpcall` message handler does not run.

### Stopping the round in flight

The Host can [stop the round](01-what-a-prompt-is.md#effects-and-harness-work) in flight without cancelling the run. Each `models.infer`, `models.loop` round, `tools.call`, tool call the model makes, and timed `tasks.join` or `tasks.join_any` that is waiting when the stop lands resumes with the interrupted error.

````markdown
---
name: patient
description: Returns a marker when the Host stops its model round.
runvil: 0
models:
  writer: {}
---

# Patient

```lua
models.default('writer')
```

## Draft

```lua
local ok, out = pcall(models.infer, 'Draft the summary.')
if not ok and out.kind == 'cancelled' then
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

Left uncaught, the interrupted error from a stop ends the run with the cancelled outcome, the same as a cancel, even though nothing cancelled the run. A stop leaves the run's cancel flag clear, so running Lua goes on, and a block that catches the stopped call can return a result while the run completes, as `patient` does. A stop cancels every tool call in flight except calls to tools that survive a stop, and the run goes on after the stop.

### The cancel check inside Lua

Running Lua checks the run's cancel flag every 10,000 instructions, a fixed interval, in each section VM's main code and in every block coroutine, so the check reaches every block of every section, the H1 pass included. The Engine also checks the flag between steps. A cancel takes effect at the next instruction check inside a running block, not only between blocks.

Cancellation reaches every block in a run, not only the first, because every block coroutine gets the same cancellation hook. Even pure Lua computation that calls nothing outside Lua is interrupted and never returns, such as this block after a cancel:

````lua
for i = 1, 100000 do end
return "done"
````

A block aborted by a cancel ends with the interrupted error, never an ordinary Lua runtime error, and the interrupted error wins over any error value the block raises.

### Telling cancellations apart

The interrupted error's `cancelled` value has no `task` field, so the presence of `task` tells a [task cancellation](15-tasks.md#cancelling-and-listing-tasks) apart from a stop or a run cancel.

Run error kind `Cancelled` classifies only errors raised while a cancelled run is still going, such as a Host interrupt seen inside the run. A run the Host cancels ends with the cancelled outcome rather than as a failure of this kind.

The first outcome that ends a run is the one it reports, so a run that finishes or fails before a cancel lands keeps that result.

A `cancelled` error value that escapes a block, including a caught one raised again later after another suspending call, maps back to the interrupted error and ends the run with the cancelled outcome rather than as a failed run.

### Catching a stopped call

`pcall` catches the interrupted error from a stopped call like any other error value. A stopped `models.infer` or `tools.call` gives `ok == false` and `err.kind == 'cancelled'`, and `tostring(err)` gives the error's message. That is how `patient` returns `stopped` under a stop.

Under a cancel, the `pcall` around a waiting call never returns, so a block written to catch a stop and `return 'stopped'` still ends with the cancelled outcome. A prompt therefore need not tell a stop from a cancel.

### Cancelling model calls, loops, tools, and fanout

On cancellation, the model reply in flight is dropped and never returns to Lua, and a cancelled `models.infer` does not report a model-round failure among the run's events. Cancelling a run while a `models.infer` waits on a model reply ends the run with the cancelled outcome, whether the call is in a section or in the H1 pass.

A running `models.loop` stays cancellable, because the loop runs as Lua inside the block and the instruction check keeps running while it does. A `models.loop` round cut short by a stop raises an error value of kind `cancelled` and appends nothing to the message list, and a later `models.loop` call runs as usual.

A tool call in flight during a cancel is not waited out: its answer is dropped and the run ends promptly with the cancelled outcome, even when the tool itself would never finish, both for a `tools.call` from Lua and for a call the model makes in a loop. Cancelling a run interrupts a [local tool](12-tools.md#defining-local-tools-with-toolsoffer_local) handler even when it is stuck in an endless Lua loop such as `while true do end`, and the run ends promptly with the cancelled outcome. Cancelling a run while a compactor is stuck in an endless Lua loop also ends the run promptly with the cancelled outcome.

A run cancelled before a fanout (`tasks.fanout`) begins ends with the cancelled outcome without running any arms. Cancelling a run while [fanout arms](14-fanout.md#what-an-arm-sees) are suspended on model calls ends the run with the cancelled outcome.

### What a stop leaves running

A stop reaches only the calls waiting when it lands. A stop that lands while no call is waiting changes nothing, so the next call runs as usual.

A stop drops a tool's calls unless the tool's Pack marks it as surviving a stop; by default a tool does not survive a stop. A tool call that survives a stop still ends when the run ends.

A stop leaves a question to the operator open: [`input.ask()`](05-lua-environment.md#asking-the-operator) keeps waiting through the stop, and the operator's answer still reaches the prompt. A stop that interrupts a timed `tasks.join` or `tasks.join_any` does not cancel the tasks it waits on, so a later wait still joins them (see [Time limits on waits](15-tasks.md#time-limits-on-waits)).

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
