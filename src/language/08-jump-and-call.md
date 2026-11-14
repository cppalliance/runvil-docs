# Jump and call

`jump` and `call` are the control functions, the two Lua globals that let a block decide which section runs next. `jump` moves the walk to another section and never comes back, while `call` runs a section like a subroutine and hands its value back to the block that called it. Both name their target with a section path such as `'../Help'`, and this chapter covers how that path resolves, which sections it can name, and what each function does in every place a section can run.

## Jump and call at a glance

After this section you can send the walk to another section with `jump` and run a section for its value with `call`.

````markdown
---
name: jump-and-call
description: Calls a child section for a value, then jumps past a sibling.
runvil: 0
---

## Check

```lua
var.answer = call('Answer')
jump('../Help')
error('unreachable')
```

### Answer

```lua
return 'forty-two'
```

## Accept

```lua
return 'accepted'
```

## Help

```lua
return 'helped:' .. var.answer
```
````

````text
helped:forty-two
````

The [section walk](04-how-a-prompt-runs.md#the-section-walk) enters `Check` first. Its `call('Answer')` runs the child section `Answer`, waits for it, and gets back the string `forty-two`, which the block keeps in [`var`](05-lua-environment.md#keeping-state-in-var). The `jump('../Help')` on the next line ends the block on the spot, so `error('unreachable')` never runs. The walk then continues at `Help`, skipping `Accept` and its `return 'accepted'`, and `Help` returns the run result. So `call` comes back to the line after it, and `jump` never comes back at all.

### Two globals in every section

`jump` and `call` are the control functions. The Engine installs both as Lua globals while it sets up every section, so every block can use them, and once setup finishes neither one is ever `nil`.

Each one takes the target section's section path as its first argument, written as a Lua string, as in `call('Sub')` and `jump('../B')`. A section path names a neighbor of the section the code is written in: the bare name of a direct child, or `../` and the name of a sibling (see [Section paths](02-file-structure.md#section-paths)). Either quote style works, as in `call('Inner')` or `call("../Child")`. So `call('Answer')` runs the child named `Answer`, `jump('../Target')` moves to the sibling `Target`, and from `A`, `call('../S1')` runs the sibling `S1`. Both `call('../Sub', { query = 'chain' })` and `call('../Sub')` reach the same `Sub` section, because only the first argument names the target.

### What `jump` does

`jump(path)` transfers control outright: the block ends at the `jump`, and the walk continues at the target section. `jump` returns nothing to the block, because the block is over, and code after it in the same block never runs. In a block holding `jump("../Target")` followed by `error("unreachable")`, the `error` is never reached. In the opening prompt, the line after `jump('../Help')` never runs, `Accept` is skipped, and the run ends with `helped:forty-two`. A path to a child works the same way, as in `jump('S1')`. [Jumping to a section](#jumping-to-a-section) covers sibling moves and backward-jump loops in full.

### What `call` does

`call(target, args?)` runs another section as a contained chain: a chain that starts at the target and [falls through](04-how-a-prompt-runs.md#the-section-walk) the target's following siblings until a section returns or the level runs out. The second argument is optional: a table of named fields, as in `call('../Sub', { query = 'chain' })`, or a string, as in `call('../Sub', 'chain-args')` or `call("../Child", "override")`. Leaving it out, as in `call('../Child')`, gives the chain no arguments of its own. [Call args and `var`](#call-args-and-var) covers what the called sections see in each case.

`call` runs the section like a function and returns the contained chain's final text to the block as a plain Lua string, or `nil` when the chain ends with no text. Here `Main` calls its child `Sub`, which falls through to `After`:

````markdown
---
name: call-child
description: Calls a child section that falls through to its next sibling.
runvil: 0
---

## Main

```lua
local r = call('Sub')
return 'got:' .. r
```

### Sub

```lua
var.trail = 'Sub '
```

### After

```lua
return var.trail .. 'After'
```
````

````text
got:Sub After
````

`call('Sub')` runs `Sub` first. It has no `return`, so the contained chain falls through to the next sibling, `After`, whose return ends the chain. That return value comes back as `r`, and `Main` finishes its block with it. [Calling a section](#calling-a-section) covers contained chains in full.

### Children run only when named

A walk moves only across siblings, so a child section never runs by plain fall-through:

````markdown
---
name: child-stays-put
description: Shows that the walk skips a child section.
runvil: 0
---

## A

```lua
var.trail = 'A'
```

### Child

```lua
error('a child must not run by fall-through')
```

## B

```lua
return var.trail .. ' B'
```
````

````text
A B
````

The walk runs `A`, then moves to its next sibling, `B`, and `Child` with its `error` never runs. A child runs only when something names it by section path. `jump('Child')` from its parent starts a walk over the children, and `call('Child')` runs it, along with the children after it, and comes back. When `A` runs `jump('X')` over children `X` and `Y`, the walk runs `X` and `Y` and then resumes at `B`, in the order A, X, Y, B. Without that `jump`, `X` and `Y` never run and only A and B do. When `Main` runs `call('Sub')`, `Sub` and `After` run, and then `Main` resumes with the reply, as in the previous prompt.

## Resolving section paths

After this section you can tell which section a section path names, where each call starts looking, and which calls take which form of path.

````markdown
---
name: levels
description: Calls a child section by its section path.
runvil: 0
---

## Main

```lua
return call('Worker')
```

### Worker

```lua
return 'worker ran'
```

### Topics

```lua
return 'topics ran'
```
````

````text
worker ran
````

The section path `'Worker'` names the direct child of `Main` with exactly that name, so among the children `Worker` and `Topics` it finds `Worker`. Change the call to `call('../Worker')` and the lookup fails, because `../` asks for a sibling, and no sibling of `Main` has the name `Worker`. The call raises the not-found error, whose text says where the lookup looked:

````text
section `../Worker` not found: `/Main` has no sibling `Worker`
````

### Children, siblings, and paths from the top

A relative path has two forms, and each names one kind of neighbor: `"Name"` is a direct child and `"../Name"` is a sibling, so the two strings name different sections. A path from the top, such as `"/Main/Worker"`, starts at the top-level sections and names one section per level. [Section paths](02-file-structure.md#section-paths) covers how each form is written.

A name in a path is compared with a section name as an exact string, with no prefix, fuzzy, or case-insensitive matching. The heading level plays no part, because a section is found by where it sits in the section tree. Whitespace around the whole path and around each name is ignored. Spaces inside a name are kept, so `"../My Section"` names a multi-word section with its inner spacing intact.

### One lookup for every call that names a section

You name another section from Lua in seven places: `call`, `jump`, [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), `tasks.fanout` (see [Choosing the worker section](14-fanout.md#choosing-the-worker-section)), [`prompt.section`, `prompt.section_or`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptsection_or), and [`require`](03-blocks-and-prose.md#sharing-code-with-require). All seven resolve a section path through one lookup over the section tree, so the same path searched from the same section always names the same section.

The seven differ in which forms they take:

- `call`, `jump`, `tasks.spawn`, and `tasks.fanout` run a section, and they take only the two relative forms. So `jump('../Help')` written in `Check` runs `Check`'s sibling `Help`, and `call('Answer')` runs its child `Answer`.
- `prompt.section` and `prompt.section_or` read a section's text. They take the two relative forms and a path from the top.
- `require` loads a section's Lua as a library. It takes only a path from the top.

A form given to a call that does not take it is an error, covered in [Malformed section paths](#malformed-section-paths).

A path is looked up when the call runs, not when the prompt is parsed. A prompt containing `call('../Missing')` still parses and starts running, and the call fails only when it runs, where [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch the failure.

### Where a relative path starts

A relative path starts at the section its code is written in:

- A section's own blocks start at that section, which is the one that is running. A called chain, a task, and a fanout arm each run a section of their own, and each starts from it.
- A block in the [preamble](02-file-structure.md#the-preamble), the text between the frontmatter and the first heading, starts at the preamble. The other top-level sections are its siblings, and it has no children (see [Jump and call from the preamble](#jump-and-call-from-the-preamble)).
- A library's code starts at the library section, both while the library loads and when its functions run later, whichever section is using the library (see [A library names its own sections](03-blocks-and-prose.md#a-library-names-its-own-sections)).

`jump` is the one exception. It moves the walk of the section that is running, so its path starts at the running section even when the `jump` is written in a library.

A path from the top starts at the top of the prompt, so it names the same section from everywhere.

### Malformed section paths

A malformed section path is a value that is not a string, or a string that breaks the path rules. It is always an error, and it is never read as a different form. Every call that takes a path checks it before any lookup, in this order:

| Order | Check | Message |
| --- | --- | --- |
| 1 | The target is a string | `section target must be a string, got {type}` |
| 2 | The string splits into names | `section path "{path}" has an empty step`, or the message for an unknown escape, a trailing backslash, or an index |
| 3 | The call takes that form | `section path "{path}" starts with "/"; only require, prompt.section, and prompt.section_or take a path from the top`, or for `require`, `section path "{path}" does not start with "/"; require takes a path from the top, such as "/helpers"` |
| 4 | The form is complete | `section path "{path}" has more than one name; a relative path is a child's name, or "../" and a sibling's name`, `section path "{path}" needs a name after ".."`, or `section path "{path}" uses ".."; a path from the top takes none` |
| 5 | A section matches | the not-found error, covered in [The not-found error](#the-not-found-error) |

[Malformed section paths](02-file-structure.md#malformed-section-paths) lists every message with a path that causes it.

- A non-string target raises a Lua runtime error at the call itself. `{type}` names the Lua type passed: `integer` for `42`, `number` for a float, `nil`, or `table`. A `pcall` around the call catches it, so `pcall(call, 42)` returns `false`.
- A relative path names one neighbor, so it has one name, with `../` in front for a sibling. `call('Kid/Grand')` gives the fourth check's first message.
- A path from the top given to a call that runs a section, such as `call('/Main')`, gives the third message. The section may well exist, and the call still refuses the form.

The first four checks are mistakes in the call, and they raise from every call, `prompt.section_or` included. Only the fifth is a miss, and `prompt.section_or` returns `nil` for it.

### Each section path names at most one section

A section path names at most one section. The parser rejects two sibling sections with the same name, with the error ``duplicate sibling section name `{name}` ... sibling section names must be unique`` (see [Sibling name uniqueness](02-file-structure.md#sibling-name-uniqueness)), so a child's name and a sibling's name are each unique, and so is each name in a path from the top. The parser also rejects any heading that skips a level as an orphan heading (see [Section headings and nesting](02-file-structure.md#section-headings-and-nesting)), so a child section sits exactly one level deeper than its parent.

## The visible set

After this section you can tell which sections a given section can run, using its place in the section tree, and read the not-found error when a section path names nothing.

````markdown
---
name: visible-set
description: Probes which sections a relative path can name.
runvil: 0
---

## Main

```lua
local seen = {}
for _, path in ipairs({'../Sibling', '../Other', 'Kid', '../Main', 'Grand', 'Niece'}) do
  seen[#seen + 1] = path .. '=' .. tostring(prompt.section_or(path) ~= nil)
end
return table.concat(seen, ', ')
```

### Kid

Kid notes.

#### Grand

Grand notes.

## Sibling

Sibling notes.

## Other

Other notes.

### Niece

Niece notes.
````

````text
../Sibling=true, ../Other=true, Kid=true, ../Main=false, Grand=false, Niece=false
````

From `Main`, the paths `'../Sibling'`, `'../Other'`, and `'Kid'` each name a section. `'../Main'`, `'Grand'`, and `'Niece'` name nothing, because `Main` is not its own sibling and has no child named `Grand` or `Niece`. The probe uses `prompt.section_or`, which reads `nil` for a path that names no section. `call`, `jump`, `tasks.spawn`, and `tasks.fanout` resolve the same six paths the same way, and they raise the not-found error for the last three.

### The visible set rule

The visible set is the set of sections the two relative forms can name from a section: its sibling sections, not counting itself, plus its own direct children. `call`, `jump`, `tasks.spawn`, and `tasks.fanout` run a section and take only the relative forms, so each of them reaches only the visible set of the section it starts from. None of them has a way to write a parent, a grandchild, a sibling's child, or a cousin. The preamble's visible set is every other top-level section, as its siblings, and no nested section, so from the preamble `call('../Main')` runs the top-level `Main` and `call('../Kid')` fails (see [Jump and call from the preamble](#jump-and-call-from-the-preamble)).

So a top-level `A` can `jump('../B')` to a sibling or `jump('X')` to one of its own children. All four calls resolve a path through the same lookup, so from a given section they accept the same paths, and `pcall(tasks.spawn, '../Missing')` and `pcall(call, '../Missing')` fail with the same error text.

For the prompt above, this is what `Main` can name with a relative path:

| Section | Relation to `Main` | Relative path | In the visible set |
| --- | --- | --- | --- |
| `Sibling`, `Other` | siblings | `'../Sibling'`, `'../Other'` | yes |
| `Kid` | direct child | `'Kid'` | yes |
| `Main` | itself | none | no |
| `Grand` | grandchild | none | no |
| `Niece` | a sibling's child | none | no |

A section's visible set never includes the section itself, so a section cannot run itself through `call`, `jump`, or the `tasks` functions. `Self` running `call('../Self')` or `jump('../Self')` fails the run with an error whose message says `not found`, because a section is not its own sibling.

A section can run a child one level below it, not just its siblings. `Sub` can `jump('S1')` to its child `S1`, and `Parent` can run `tasks.fanout('Worker', ...)` on its child `Worker`.

The prompt reads and `require` are not held to the visible set, because neither one runs a section as part of the walk. A read takes a path from the top beside the relative forms, so `Main` reads its grandchild with `prompt.section('/Main/Kid/Grand')` and its sibling's child with `prompt.section('/Other/Niece')` (see [Which sections a read can name](03-blocks-and-prose.md#which-sections-a-read-can-name)). `require` takes only a path from the top (see [How `require` finds and caches libraries](03-blocks-and-prose.md#how-require-finds-and-caches-libraries)).

### What a nested section can name

A running child section has a visible set of its own: its own siblings plus its own children.

````markdown
---
name: child-reach
description: A child section calls its own child, then jumps to its sibling.
runvil: 0
---

## A

```lua
jump('X')
```

### X

```lua
local r = call('Grand')
var.trail = 'X:' .. r .. ' '
jump('../Y')
```

#### Grand

```lua
return 'grand-ran'
```

### Y

```lua
var.trail = var.trail .. 'Y'
```

## B

```lua
return var.trail
```
````

````text
X:grand-ran Y
````

`A` jumps into its child `X`, which starts a [child walk](#child-walks). `X` then calls its own child `Grand` and jumps to its own sibling `Y`, both inside its own visible set. This is also how you run a deeper section: a section cannot run its grandchildren directly, so it goes through the direct child that contains them, as `A` reaches `Grand` here through `X`.

A section's grandchildren, its parent, and its parent's siblings are not in its visible set, and no relative path names them:

- A nested section cannot run its parent. Written in `Worker` under `Parent`, `'../Parent'` asks for a sibling of `Worker` named `Parent`, and there is none. So `jump('../Parent')` ends the run with [error code](16-limits-and-errors.md#run-error-codes) `lua_runtime` and a message containing `not found`, and `pcall(call, '../Parent')` fails with `not found`.
- A nested section cannot run its parent's siblings, its aunts and uncles. When `X` under `A` runs `jump('../B')` and `B` is a sibling of `A`, the jump fails the run with a `not found` error, because `B` is not a sibling of `X`.
- A section cannot run the children of its siblings, its nieces and nephews. `A` running `jump('Niece')` while `Niece` sits under the sibling `B` fails the run with a `not found` error, and `Main` running `tasks.fanout('Niece', {'x'})` while `Niece` sits under `Other` fails the same way.

A relative path that would need a section outside the visible set behaves exactly like a path to a section that does not exist, because only the visible set is searched. `X` running `jump('../B')` fails with `not found` even though the prompt has a `B`.

### The not-found error

The not-found error is the Lua error a well-formed section path raises when it names no section:

````markdown
---
name: missing-call
description: Calls a sibling that the section does not have.
runvil: 0
---

## Main

```lua
return call('../Missing')
```

### Kid

Kid notes.

## Other

Other notes.
````

The run fails with code `lua_runtime`. Its message is the not-found error's text, followed by the line of the `call`:

````text
section `../Missing` not found: `/Main` has no sibling `Missing` (at missing-call.md:10)
````

The not-found error's text always has this form:

````text
section `{path}` not found: {where}
````

`{path}` is the section path with its surrounding whitespace trimmed. `{where}` names the section that was searched, by that section's path from the top, and says whether a child or a sibling was wanted there. Written in `Main` above, each path reports:

| Path | Message |
| --- | --- |
| `'Missing'` | ``section `Missing` not found: `/Main` has no child `Missing` `` |
| `'../Missing'` | ``section `../Missing` not found: `/Main` has no sibling `Missing` `` |
| `'/Other/Missing'` | ``section `/Other/Missing` not found: `/Other` has no child `Missing` `` |
| `'/Othr/Kid'` | ``section `/Othr/Kid` not found: the top level has no section `Othr` `` |

A path from the top reports the step where it stopped, so the last row names `Othr` and never looks for `Kid`. A relative path written in the preamble is searched from the preamble, which has no name and no path, so the message says `the preamble`: `call('Kid')` written there gives ``section `Kid` not found: the preamble has no child `Kid` `` and `jump('../Missing')` gives ``section `../Missing` not found: the preamble has no sibling `Missing` ``.

The message tells the two relative forms apart, which makes two common mistakes easy to read. A sibling written without `../`, as in `call('Other')` from `Main`, reports that `/Main` has no child `Other`. A child written with `../`, as in `call('../Kid')`, reports that `/Main` has no sibling `Kid`.

A relative path is searched from the section its code is written in, so that is the section the message names. For code written in a library, it is the library (see [A library names its own sections](03-blocks-and-prose.md#a-library-names-its-own-sections)).

The message names only the section it searched. It lists no other section, so a not-found message never shows the rest of the prompt's structure.

Each call treats a path that names no section in one of two ways:

- `call`, `jump`, `tasks.spawn`, `tasks.fanout`, `require`, and `prompt.section` raise the not-found error. For the same path searched from the same section, all of them raise the same text, so the `tasks.spawn` message is exactly the one `call` produces.
- `prompt.section_or` returns `nil` without raising. So in a fanout worker section, `pcall(call, '../Parent')` fails with `not found` while `prompt.section_or('../Parent')` reads `nil`.

`pcall(call, '../Missing')` returns `false` with a message containing ``section `../Missing` not found``. An uncaught not-found error in a section ends the run with code `lua_runtime`, as when an unresolved `jump('../Parent')` ends the run.

## Jumping to a section

After this section you can move the walk to a sibling with `jump`, build a backward-jump loop, and know what happens to the rest of the jumping block.

````markdown
---
name: sibling-move
description: Jumps over a section to a later sibling.
runvil: 0
---

## A

```lua
var.from_a = 'a'
jump('../C')
```

## B

```lua
error('the jump must skip B')
```

## C

```lua
var.from_c = 'c'
```

## D

```lua
return var.from_a .. var.from_c
```
````

````text
ac
````

### Sibling moves

Calling `jump('../Name')` from a `lua` block moves the walk to another section within the same chain. A `jump` to a sibling, another section at the jumping section's own level, is a sibling move. The walk stays within the jumper's own group of siblings, every sibling except the jumper is a valid target, and the target starts as a fresh section entry and falls through from there.

After a sibling move the walk continues at the target, so the sections between the jumper and the target do not run, even one that would `return 'accepted'` or call `error('the jump must skip B')`, as `B` does above. When the target finishes without a `return`, the walk falls through to the section that follows the target, not the one that follows the jumper, which is why `D` runs after `C`. When the target returns a value, that value becomes the run's text: a `jump('../Other')` to an `Other` section that does `return 'jumped'` ends the run with `jumped`.

A `jump` target shares the walk's `var`, so it sees every value written before the jump, including the jumping section's own writes, as `C` and `D` see `var.from_a`. Values the target writes to `var` stay visible to the sections the walk falls through to afterward, as `D` sees `var.from_c`.

### Backward-jump loops

A sibling target can sit before or after the jumping section. A jump to an earlier sibling runs that sibling and every section after it again, including the jumper, so a backward jump makes a backward-jump loop:

````markdown
---
name: redraft
description: Jumps back to redraft until the draft has run three times.
runvil: 0
---

## Draft

```lua
var.trail = (var.trail or '') .. 'D'
```

## Review

```lua
if #var.trail < 3 then jump('../Draft') end
return 'passes: ' .. var.trail
```
````

````text
passes: DDD
````

Each time `Draft` runs, it appends a `D`, and `Review` jumps back while the trail is shorter than three characters. You end a backward-jump loop by guarding the `jump` with a value kept in `var`, as `if #var.trail < 3 then jump('../Draft') end` does here. Each time around starts from the `var` the previous time left, so the guard sees the count grow, and the third time `Review` runs it reaches the `return`.

### What a jump ends

Code after a `jump` call in the same `lua` block never runs, because `jump` does not return to the jumper. A jump also skips every remaining block of the jumping section, so that section's later `lua` blocks never run:

````markdown
---
name: jump-ends
description: Shows that a jump skips the rest of its section.
runvil: 0
---

## Check

```lua
var.checked = 'yes'
jump('../Help')
error('unreachable')
```

This prose is left behind.

```lua
error('the second block never runs')
```

## Help

```lua
return 'checked: ' .. var.checked
```
````

````text
checked: yes
````

`jump` checks that its argument is a string, records the target, and ends the block right there. A non-string argument fails with `section target must be a string, got {type}`. To end the block, `jump` raises its transfer marker: an error that unwinds the block from the point of the call. When the block ends, the recorded target takes precedence over that error. A `jump` is a control transfer, not a failure, so the jumping block reports success, not an error. `jump` is not a [suspending call](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), so nothing waits on the Harness when a block jumps.

### What crosses a jump

A section that jumps counts as completed, exactly as when it falls through, and its final `var` becomes the target section's starting `var`. The jumper reports that its section finished before the target reports that its section started (see [Logging and lifecycle reports](05-lua-environment.md#logging-and-lifecycle-reports)).

Only `var` crosses a jump. Locals, globals, and unread pending prose stay behind, because the target starts in a fresh [section VM](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals) of its own, seeded only with `var`:

````markdown
---
name: what-crosses
description: Shows that only var reaches a jump target.
runvil: 0
---

## A

```lua
greeting = 'hi'
var.note = 'kept'
jump('../B')
```

## B

```lua
return tostring(greeting) .. ' ' .. var.note
```
````

````text
nil kept
````

The global `greeting` stays behind in `A`'s section VM, so it reads `nil` in `B`, while `var.note` arrives intact. Pass anything the target needs through `var`.

## Child walks

After this section you can start a child walk by jumping into a direct child, and predict where the walk goes when that child walk ends.

````markdown
---
name: child-walk
description: Jumps into a child level and returns to the parent level.
runvil: 0
---

## A

```lua
var.trail = 'A'
jump('X')
```

### X

```lua
var.trail = var.trail .. ' X'
```

### Y

```lua
var.trail = var.trail .. ' Y'
```

## B

```lua
return var.trail .. ' B'
```
````

````text
A X Y B
````

### Starting a child walk

A `jump` to one of the running section's direct children, such as `jump('X')` written in `A`, descends into that section's children and starts a child walk at the named child. The child runs as part of that walk. A child walk falls through from the jump target to the target's following siblings at the same heading level, under the same rules as the top-level walk, so `Y` runs after `X` without being named.

A `jump` to a later child starts the child walk at that target, so the target's earlier siblings never run. With children `X`, `Off`, and `Y`, `jump('Off')` from `A` runs `Off` and then `Y`, and `X` never runs.

### Where the walk resumes

When a child level runs out of sections, the parent walk resumes at the section after the jumper, not after the jump target. So after `Y`, the next section to run is `B`, the sibling after `A`, which made the jump.

A child level returns control to its parent's level only by running out of siblings. A running child cannot `jump` up to a section at its parent's level, so `jump('../B')` written in `X` fails with a `not found` error, because the parent's level is not in the running child's visible set.

A running child section can `jump` to its own sibling, and when that child level runs out, the parent walk still resumes after the section that jumped into the child level. So `jump('../Y')` from `X` runs `Y`, and `B` runs next.

A parent section and its child level share `var`. The children start from the jumper's final `var`, and the section the parent walk resumes at starts with the `var` left by the child level's last section, which is why `B` sees every piece of the trail the earlier sections appended.

### Nested child walks

Child walks nest to any depth. A `jump` from an H3 child to its own H4 child starts an H4-level walk, and each level that runs out resumes its parent level after that level's jumper:

````markdown
---
name: nested-walk
description: Jumps two levels down and climbs back up.
runvil: 0
---

## A

```lua
var.trail = 'A'
jump('X')
```

### X

```lua
var.trail = var.trail .. ' X'
jump('P')
```

#### P

```lua
var.trail = var.trail .. ' P'
```

#### Q

```lua
var.trail = var.trail .. ' Q'
```

### Y

```lua
var.trail = var.trail .. ' Y'
```

## B

```lua
return var.trail .. ' B'
```
````

````text
A X P Q Y B
````

After `Q`, the H4 level runs out, so the walk resumes after its jumper, `X`, at `Y`. Then the H3 level runs out, so the walk resumes after `A` at `B`. The run climbs back from H4 to H3 to the top level in order.

### Returning from a child walk

A `return` inside a jump-started child walk ends the whole chain (see [Returning values](04-how-a-prompt-runs.md#returning-values)), so neither the child level's remaining sections nor the parent walk's next sibling run. Change `X` in the first prompt of this section to return a value:

````markdown
### X

```lua
return 'x-value'
```
````

````text
x-value
````

Neither `Y` nor `B` runs, and the run result is `x-value`. A `return` that gives [no value](04-how-a-prompt-runs.md#returning-no-value) ends the chain the same way: with a bare `return` in `X`, neither `Y` nor `B` runs, and the run result is `done`.

## When a jump is refused or fails

After this section you can predict every case where `jump` is refused or fails, and read the message each one gives.

````markdown
---
name: caught-jump
description: Catches a jump with pcall and still lands on the target.
runvil: 0
---

## A

```lua
local ok = pcall(jump, '../C')
var.ok = tostring(ok)
return 'never the result'
```

## B

```lua
error('the jump must skip B')
```

## C

```lua
return 'C saw ok=' .. var.ok
```
````

````text
C saw ok=false
````

`pcall` catches the transfer marker, so the block keeps running, but the jump is already recorded. When the block ends, the jump wins: `return 'never the result'` is dropped, `B` is skipped, and `C` sees the `var` the block left. This table lists every case, and the subsections below explain each one:

| Case | What happens | Message |
| --- | --- | --- |
| `jump` inside `pcall` | the jump still lands | `jump transfer`, returned to `pcall` |
| A non-string target | the `jump` call raises | `section target must be a string, got {type}` |
| A path that names no section | the run fails after the block ends | the not-found error |
| A malformed section path | the run fails after the block ends | the matching message from [Malformed section paths](#malformed-section-paths) |
| Inside an adhoc tool handler | `jump` is refused | the adhoc tool handler refusal |
| While a library loads | `jump` is refused | the message in [Jumps while a library loads](#jumps-while-a-library-loads) |

### Catching a jump with `pcall`

Each Lua block starts with no pending jump, so a `jump` affects only the block that recorded it and never spills over from one block into the next.

`pcall` cannot cancel a jump. A `jump` made inside `pcall`, directly or in a function that `pcall` runs, still records its target. `pcall(jump, path)` returns `false`, the error it returns holds the text `jump transfer`, and the rest of the block keeps running after the `pcall`. How the block ends then makes no difference:

- When the block ends normally, the jump wins and the block's return value is discarded. After `local ok = pcall(jump, '../C')`, a later `return 'never the result'` is dropped and the walk moves to `C`.
- When the block ends with an error, the jump wins and the error is discarded, so the walk moves to the recorded target instead of failing on the block's error.

### Lookup after the block ends

At the call itself, `jump` checks only that its target is a string. The section path is looked up only after the jumping block has ended, and the walk then continues at the section it names. The jumping section closes as completed, with its final `var` saved, before the path is looked up, so it still reports that its section finished even when the target fails to resolve. Because the lookup runs after the jumping section has closed, a bad `jump` target never raises an error inside the jumping block, and `pcall` there cannot catch it.

A `jump` whose path names no section fails the run with the not-found error, ``section `{path}` not found: {where}``, followed by ` (at {file}:{line})` with the line of the `jump` call (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). The message names the section the jump started from, so a failed `jump('../Parent')` in the fanout worker section `Worker` under `Parent` reports that `/Parent/Worker` has no sibling `Parent`.

A `jump` whose section path is malformed, such as a path with an empty name, a relative path with more than one name, or a path from the top, also fails the run after the jumping block has ended, with the matching message from [Malformed section paths](#malformed-section-paths). Every failed `jump` lookup has code `lua_runtime` and points at the line of the `jump` call.

### The adhoc tool handler refusal

An [adhoc tool](12-tools.md#how-adhoc-tool-handlers-run)'s handler cannot jump. Calling `jump` inside an adhoc tool handler raises a Lua error with this message, the adhoc tool handler refusal:

````text
jump is unavailable inside an adhoc tool handler: return a value from the handler and call jump from the block after the tool call returns
````

So the handler returns a value, and the block calls `jump` after the tool call returns. The run fails if nothing catches the refusal. The rules around it:

- `jump` is refused while an adhoc tool's handler runs, whether you called the tool with [`tools.call`](12-tools.md#calling-tools-from-lua-and-counting-calls) or the model called it during [`models.loop`](11-conversations.md#starting-a-conversation), and no jump lands in either case.
- The refusal holds even when the handler calls `jump` through a reference saved before the adhoc tool was defined, such as `local j = jump` followed by `j('../Other')`. The error still holds `jump is unavailable inside an adhoc tool handler`.
- A refusal the handler does not catch fails the handler and is raised where the tool was called. So `pcall(tools.call, ...)` in the block catches it, returns `false` with the refusal text, and the block keeps running.
- `jump` stays refused in an outer handler after a nested handler it called has returned. After the outer handler's `tools.call('adhoc/inner', {})` returns, where `adhoc/inner` is the id of another adhoc tool, named `inner`, `pcall(jump, '../Other')` inside the outer handler still fails with the refusal message.
- `jump` works again in the block once the outermost adhoc tool handler returns or fails, whether the handlers ran under `tools.call` or `models.loop`, including after `pcall(tools.call, ...)` has caught the handler's own error.

Here the block defines an adhoc tool, `grab`, with [`tools.adhoc`](12-tools.md#adhoc-tools-with-toolsadhoc). Its handler calls `jump`, and the block catches the refusal and then jumps on its own:

````markdown
---
name: refused-jump
description: Catches a handler's refused jump, then jumps from the block.
runvil: 0
---

## Main

```lua
local grab = tools.adhoc({ name = 'grab', description = 'Grab a value' }, function() jump('../Other') end)
local ok, err = pcall(tools.call, grab, {})
jump('../Other')
```

## Skipped

```lua
error('the jump must skip this section')
```

## Other

```lua
return 'jumped'
```
````

````text
jumped
````

`ok` is `false` and `err` holds the refusal text. Once the handler has failed, `jump` works again, so the block's own `jump('../Other')` transfers, `Skipped` never runs, and `Other` ends the run with `jumped`.

### Jumps while a library loads

A [library](03-blocks-and-prose.md#sharing-code-with-require) cannot jump while it loads. Calling `jump` while a library loads through `require` raises a Lua error with this message:

````text
jump is unavailable while a library loads: return a value from the library and call jump from the section after require returns
````

So the library returns a value, and the section calls `jump` after `require` returns. Library loads nest, and `jump` stays refused until every load under way has ended, so a library's load-time code still cannot `jump` after a library it required has finished loading. When a library load ends, whether it finishes, raises an error, or is canceled, `jump` works again for the caller. So after `pcall(require, '/a')` fails on the library's refused jump, `jump '../after'` in the same block transfers to `after`.

A function that a library defines can call `jump` once the library has finished loading, and the jump transfers the section that called the function:

````markdown
---
name: library-jump
description: Calls a library function that jumps.
runvil: 0
---

## Main

```lua
require '/a'
go()
```

## a

```lua
function go() jump '../next' end
```

## next

```lua
return 'next ran'
```
````

````text
next ran
````

`require '/a'` loads the library, which only defines `go`. By the time `Main` calls `go()`, the load has ended, so the `jump` inside `go` transfers `Main` to `next`, and the walk skips `a`.

## Calling a section

After this section you can run a section with `call`, use the value it hands back, and place subroutine sections so they run only when called.

````markdown
---
name: research
description: Calls a subroutine section and uses its reply.
runvil: 0
---

## Main

```lua
local found = call('../Research')
return 'found: ' .. found
```

## Research

```lua
return 'research-reply'
```
````

````text
found: research-reply
````

`call(target, args?)` takes a section path for the target section, such as `'../Research'`, plus optional arguments for the called chain, and starts a called chain at that target. It runs the named section, such as a sibling, as a contained chain and returns that chain's final returned value to the calling block, so `local found = call('../Research')` gets `research-reply` because `Research` ends by returning that text. The optional arguments reach the called section as its `args`, and the called section can read values the caller stored in `var` (see [Call args and `var`](#call-args-and-var)).

When the called chain returns, its result becomes the value of `call`, the calling block keeps running, and when that block finishes, the walk resumes at the section after the caller. Here `Main` returns, so the run ends with `found: research-reply`, and the walk never reaches `Research` on its own.

### Contained chains

A chain started by `call` begins at the target and falls through to the target's following siblings under the same rules as the main walk, until a section returns or the level runs out. The returning section's value is what `call` gives back:

````markdown
---
name: call-chain
description: Calls a section whose chain falls through to the next sibling.
runvil: 0
---

## A

```lua
var.result = call('../S1')
```

## B

```lua
return 'B saw: ' .. var.result
```

## S1

```lua
var.trail = 'S1 '
```

## S2

```lua
return var.trail .. 'S2'
```
````

````text
B saw: S1 S2
````

`S1` has no `return`, so the contained chain falls through to `S2`. Within a called chain, `var` rolls forward across the chain's sections as it does on the main walk, so the `var.trail` that `S1` stores is readable in `S2`. `S2`'s return ends the chain, `call` returns `S1 S2` to `A`, and the walk resumes at `B`.

A `return value` inside a called chain ends only that chain, not the run (see [Returning values](04-how-a-prompt-runs.md#returning-values)). The value becomes what `call` returns, the chain's remaining sections are skipped, and the outer walk keeps going. `call` returns the final reply of its contained chain from whichever section ends it with a return, even when that section is a later sibling of the target rather than the section you named, so a call to `Sub` that falls through to a `Tail` returning `'tail-reply'` gives `tail-reply`. A called chain that runs off its last section without returning has no text to give back, so `call` returns `nil`, and so does one that ends at a `return` that gives [no value](04-how-a-prompt-runs.md#returning-no-value), such as a bare `return` or `return nil`. A chain that returns the empty string on purpose, with `return ''`, gives `call` that string. [A call that gets no text](#a-call-that-gets-no-text) shows what the calling block can do with the `nil`.

A section can `call` any other sibling at its level, earlier or later in the file, even one that sits past other sections, such as `A` calling a `Sub` that sits after `B`.

### Using the value `call` returns

After `call` returns, the calling block keeps running its remaining statements, and the block's own `return` becomes its reply. The value `call` returns is a Lua string when the called chain ends with text, so you can use it in several ways:

- Use it directly inside an expression, such as concatenating it with `..` in `return var.first .. ',' .. call('../Answer')`.
- Write `call('../Sub')` as a plain statement that discards the chain's reply. The block's following statements still run.
- Return it directly with `return call(...)`, which passes the called section's return value straight through as the section's own result. That `return` ends the calling section whatever the called chain gave back.

Calls nest, so a called section can `call` another section and hand that result back to its own caller, as with `return call('../Inner')`.

You can `call` the same section more than once, and each call runs the section again as a separate child with its own [`sys.id`](05-lua-environment.md#section-entry-ids), and returns that run's reply:

````markdown
---
name: call-twice
description: Calls the same section twice.
runvil: 0
---

## Main

```lua
local first = call('../Sub')
local second = call('../Sub')
return first .. ',' .. second
```

## Sub

```lua
return sys.id
```
````

````text
0.0.0,0.1.0
````

### A call that gets no text

When the called chain ends with no text, `call` returns `nil` in place of a string:

````markdown
---
name: call-nothing
description: Calls a section that ends with no text.
runvil: 0
---

## Main

```lua
local quiet = call('Quiet')
local loud = call('Loud')
return type(quiet) .. '|' .. (quiet or 'fallback') .. '|' .. loud
```

### Loud

```lua
return 'loud'
```

### Quiet

```lua
var.note = 'ran'
```
````

````text
nil|fallback|loud
````

`Quiet` has no `return` and is the last section of its chain, so the chain ends with no text, and `quiet` is `nil`. `Loud` returns, so `loud` is the string `loud`. A `nil` tells the calling block that the chain gave it nothing, where the empty string would look like a reply:

- Test the value, as in `if quiet then`, which is false for a chain that ended with no text.
- Give it a fallback with `or`, as in `quiet or 'fallback'`.
- Do not concatenate it as it is. `'got: ' .. call('Quiet')` raises the Lua error `attempt to concatenate a nil value` at that line, which fails the run unless something catches it. Write `'got: ' .. (call('Quiet') or '')` where the called chain may end with no text.
- `return call('Quiet')` still ends the calling section, as every `return` does. The calling section then ends with no text itself, so its own caller gets `nil`, and on the main walk the run result is `done`.

Each kind of chain reports an ending with no text in its own way. The main walk gives the run result `done` (see [The run result](04-how-a-prompt-runs.md#the-run-result)), a called chain gives `call` `nil`, and a [task](15-tasks.md#what-a-task-section-returns) or a [fanout arm](14-fanout.md#what-an-arm-returns) gives its owner the empty string.

### Calling a child section

`call` on a direct child of the calling section, such as `call('Sub')` from `Main`, starts a contained chain at that child, which falls through the child's following siblings. `call` resolves its path against the same visible set as `jump`, and a direct child target is found among the caller's own children, so the chain runs at the child level, while any other target runs at its own level.

A `call` to a later child starts at that child, so the caller's earlier children never run, and the chain falls through the following child siblings and returns the reply of the one that returns:

````markdown
---
name: call-later-child
description: Calls the second of three children.
runvil: 0
---

## Main

```lua
local r = call('Sub2')
return 'got:' .. r
```

### Sub1

```lua
error('an earlier child must not run')
```

### Sub2

```lua
var.trail = 'Sub2 '
```

### Sub3

```lua
return var.trail .. 'Sub3'
```
````

````text
got:Sub2 Sub3
````

A running child section can `call` one of its own children and use the value that child returns, such as `X` running `local r = call('Grand')` and getting `grand-ran`.

Fall-through in a `call` chain moves to the next sibling and does not walk into the called section's own child sections. A `call` chain also stays within the target's heading level: when that level runs out, the chain ends and control returns to the caller instead of continuing into the parent level, so a call into an H4 child does not continue into the H3 sibling that follows it.

### Subroutine sections

A subroutine section is a section that runs only when something calls it. Children make natural call targets because the walk never reaches them by fall-through. A section the main walk never reaches, such as one placed after the section whose `return` ends the run, still runs when `call` or `tasks.fanout` names it directly, so you put `call` targets there to have them run only when called. That is how `Research` works in the first prompt of this section.

A called section starts a chain that falls through to the sections after it like any walk. So a call target placed before other sections pulls them into its chain, while the main walk also runs the target on its own. Subroutine sections belong after the section that ends the run.

### Jumps inside a called chain

A `jump` inside a section reached through `call` is contained by the call chain. It is followed rather than rejected, and the chain's final result, such as the jump target's reply, becomes what `call` returns:

````markdown
---
name: jump-in-call
description: A called section jumps to a later sibling.
runvil: 0
---

## Main

```lua
local r = call('../Sub')
return 'main:' .. r
```

## Sub

```lua
jump('../Peer')
```

## Skipped

```lua
error('the chain jump must move past me')
```

## Peer

```lua
return 'peer-ran'
```
````

````text
main:peer-ran
````

When a `jump` inside a call chain moves to a later sibling, the sections between the jumping section and the target do not run, as `Skipped` shows. After the jump, the chain continues from the target under the normal fall-through rules, so a target with no return falls through to its next sibling, and the chain's final reply is the call's return value. A called section can also `jump` to one of its own children, which starts a child walk inside the call that falls through the child's following siblings, and that level's final return becomes the result of `call`: when `Sub` runs `jump('S1')` and `S2` returns `'s2-result'`, the caller gets `s2-result`.

The outer walk never moves while a called chain runs. Wherever the chain ends, even after a `jump` inside it to a section past the caller, the caller resumes and the walk continues at the section after the caller.

### How a called chain runs

A `call` runs its target as a new child chain of the caller, one call level deeper (see [How call depth counts](#how-call-depth-counts)) and with the caller's next child `sys.id`. The Engine drives it the same way it drives the walk.

`call` suspends its caller. The calling block pauses at `call` while the target runs in its own section VM, so the caller and the called chain never interleave, and the called chain's model rounds come before the caller's later ones. A called section can make its own model rounds with [`models.infer`](10-models.md#running-a-round-with-modelsinfer), each a separate model round of the run, and it can return a model's reply, which reaches the caller unchanged. When `Inner` does `return models.infer('inner ask')`, a caller that runs `local inner = call('../Inner')` gets the model's reply in `inner`. The called section starts on a copy of its caller's current model, taken when the `call` is made, so a `models.use` inside it never reaches the caller (see [Children and fanout arms](10-models.md#children-and-fanout-arms)).

A called chain has no inbox of its own: it runs inside its caller's task, so it uses its caller's inbox, through every level of nested calls. A `tasks.receive()` in a section the main chain runs with `call` takes the operator's next message (see [Messages that arrive first](05-lua-environment.md#messages-that-arrive-first)), and one in a section a task runs with `call` takes the next message the task's owner sends, while a `tasks.reply` there replies for that task (see [Messages between tasks](15-tasks.md#messages-between-tasks)). A message sent to the caller's chain also reaches a `models.loop` running inside the call (see [Messages that arrive while the loop runs](11-conversations.md#messages-that-arrive-while-the-loop-runs)).

A function from a `require`d library that runs `call` resolves the path from the library section, whichever section calls the function, so it runs the library's own child. When a library defines `later()` with `call('mine')` inside, and the library has a child `mine` that does `return 'mine'`, the `call` returns `mine` for every section that calls `later()`. The chain still runs as a child chain of the section that called the function. A library that means a section of the one using it writes `_G.call` (see [A library names its own sections](03-blocks-and-prose.md#a-library-names-its-own-sections)).

A called section cannot call itself, and the only Lua values it gets from its caller are the ones [Call args and `var`](#call-args-and-var) describes.

## Call args and `var`

After this section you can pass arguments to a called section and read them there, and know which `var` values and which `args` the called chain starts with.

````markdown
---
name: pipeline
description: Passes one section's text to the next as its arguments.
runvil: 0
---

## Main

```lua
local draft = 'refined text'
return call('../Deliver', draft)
```

## Deliver

```lua
return 'delivered: ' .. args.text
```
````

````text
delivered: refined text
````

### Passing arguments to a call

A call's arguments are what you pass as the second argument to `call`: a table of named fields, as in `call('../Sub', { query = 'chain' })`, or a string, as in `call('../Child', 'override')`. The called chain receives them as its own arguments instead of the ones it would otherwise inherit. In every section the chain runs, the [`args` global](06-arguments.md#reading-arguments-from-args) holds the call's arguments instead of the run's own, and the prose of any section in the chain reads the call's arguments too. A table becomes the chain's `args`, so in a run started with the arguments `{"query":"run"}`, `call('../Sub', { query = 'chain' })` makes `args.query` read `chain` inside `Sub`. A string becomes a table with one field, `text`, that holds the string, so `call('../Child', 'override')` makes `args.text` read `override` inside `Child`, as `Deliver` reads `args.text` above. `text` is the same field a prompt with no `args:` key reads its one free text from (see [Prompts with no `args:` key](06-arguments.md#prompts-with-no-args-key)). [`args` in a called chain](#args-in-a-called-chain) shows what the chain reads in each case.

A [placeholder](07-substitution.md#placeholders-in-prose) such as `{{ args.text }}` inside a called chain renders the call's arguments instead of the run's. In a run started with the arguments `{"text":"run-args"}`, `call('../Sub', 'chain-args')` makes the chain's model round receive `chain-args` and not `run-args`. A `call` with arguments of its own leaves the caller's `args` as they were, so the calling section's `args` still holds its own arguments after the call.

You build a pipeline by passing a string one section produced as the next section's arguments, as `Main` does above with `return call('../Deliver', draft)`. The run's own arguments make no difference: when `Refine` ends with `return call('../Deliver', messages[#messages].content)`, the run's result is `delivered: refined text`, even though the run itself started with the arguments `{"text":"quantum"}`.

### Inheriting the caller's arguments

A `call` with no second argument inherits its caller's arguments: the called chain reads the caller's `args`. From the main walk those are the run's arguments, and inside a called chain that was given arguments of its own they are that chain's arguments, not the run's, because a call's arguments apply to the whole chain:

````markdown
---
name: inherit-args
description: A nested call with no arguments of its own reads the chain's arguments.
runvil: 0
---

## Main

```lua
return call('../Sub', 'chain-args')
```

## Sub

```lua
return call('../Inner')
```

## Inner

```lua
return 'Inner saw ' .. args.text
```
````

````text
Inner saw chain-args
````

A `call` with the second argument left out or passed as `nil` makes the called chain inherit the caller's arguments whole: it reads the caller's [`args`](06-arguments.md#reading-arguments-from-args), which is [read-only](06-arguments.md#args-is-read-only) in the called chain as it is in every section. In a run started with the arguments `{"query":"run"}`, under an `args:` declaration with a `query` field, `args.query` reads `run` in a `Sub` reached by `call('../Sub')`. `call('../Sub', args)` passes the caller's arguments on too, and so does a table built from parts of `args`, such as `call('../Sub', { query = args.query })`. A [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) with no `args` option inherits the same way.

An empty string counts as an argument, so `call(path, '')` gives the called chain an `args.text` of `''` instead of the caller's arguments. Only a `nil` or missing second argument inherits.

### `args` in a called chain

A call given arguments of its own gives the called chain an `args` table of its own, built from what the call passed. That `args` is [read-only](06-arguments.md#args-is-read-only) for the whole chain:

````markdown
---
name: chain-args
description: Gives a called chain its own args.
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
---

## Main

```lua
return call('../Sub', { query = 'chain' })
```

## Sub

```lua
local wrote = pcall(function() args.query = 'x' end)
return args.query .. ':' .. tostring(wrote)
```
````

Run with the arguments `{"query":"run"}`, the prompt returns:

````text
chain:false
````

Inside `Sub`, `args.query` reads `chain` from the call, not `run` from the run's arguments. The chain's `args` is read-only, so a field write such as `args.query = 'x'` fails, and `pcall` catches it, leaving `wrote` false. What the chain's `args` holds depends on the second argument of the `call`:

| Second argument | `args` in the chain |
| --- | --- |
| Left out or `nil` | the caller's `args` |
| A table of named fields | the fields of that table |
| A string | one field, `text`, that holds the string |

The Engine does not check a call's arguments against the prompt's `args:` declaration, and it fills no defaults in them, because the declaration describes what a caller passes the prompt, not what one section passes another. Under the declaration above, `call('../Sub', { query = 5, extra = true })` starts the chain all the same, and there `args.query` is the integer `5` and `args.extra` is `true`. A string arrives as `args.text` under that declaration too, though it declares no `text`. Sent as the arguments of the run, the same fields end the run at its first step (see [Argument errors](06-arguments.md#argument-errors)).

Every section of the chain reads the same `args`: the target, each section the chain falls through to, and the target of a `jump` inside the chain, because a jump stays in its chain. When the chain ends, the caller's `args` are what they were before the call. Here a call gives `Sub` a `query` of its own, `Sub` jumps to `Peer`, and each of the three sections reports the `query` and the `limit` it reads:

````markdown
---
name: chain-reads
description: Shows which arguments each section reads around a call.
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    limit: { type: integer, default: 10 }
  required: [query]
---

## Main

```lua
local seen = call('../Sub', { query = 'chain' })
return seen .. ' | Main:' .. args.query .. ',' .. args.limit
```

## Sub

```lua
var.seen = 'Sub:' .. args.query .. ',' .. tostring(args.limit)
jump('../Peer')
```

## Peer

```lua
return var.seen .. ' | Peer:' .. args.query .. ',' .. tostring(args.limit)
```
````

Run it with the arguments `{"query":"run"}`:

````text
Sub:chain,nil | Peer:chain,nil | Main:run,10
````

`Sub` and `Peer` read the same `chain`, before the jump and after it. Neither reads a `limit`, because the call passed none, and the Engine fills the default of `10` only in the arguments of the run. After the chain ends, `Main` reads its own `run` and `10`.

### Call args errors

The second argument of `call` is a table of named fields, a string, `nil`, or left out. These refusals fail the call at the call site, before the chain starts:

- A list fails with `args must be a table of named fields or a string, got a list`. That covers a list written in place, such as `{ 1, 2 }`, an empty marked list from [`json.array()`](05-lua-environment.md#keeping-empty-lists-with-jsonarray), and a list read from `args`, such as `args.files`. To pass a list, put it in a named field, as in `call('../Sub', { files = args.files })`.
- A value of any other type fails with `args must be a table of named fields or a string, got {type}`, where `{type}` is the Engine's type name, such as `integer` for `5`, `number` for `1.5`, or `boolean` for `true`.
- A table that holds a value with no JSON form fails with a message that names the value by its path. `call('../Sub', { f = function() end })` fails with `args: value.f must be JSON data, got function`.
- A string that is not valid UTF-8 fails with `args must be a valid UTF-8 string`.

An unmarked empty table, a plain `{}`, is the empty object, not a list, so `call('../Sub', {})` starts the chain, and the chain reads an `args` with no fields.

### The `var` snapshot

Every `call` gives the called chain a `var` snapshot: a plain deep copy of the caller's `var` data, taken at the moment of the call. There is nothing to pass, since every `call` takes the copy on its own:

````markdown
---
name: var-snapshot
description: A called section reads the caller's var but cannot write back.
runvil: 0
---

## Main

```lua
var.shared = 'caller'
local r = call('../Sub')
assert(var.child_write == nil)
return r
```

## Sub

```lua
var.child_write = 'sub'
return 'sub saw ' .. var.shared
```
````

````text
sub saw caller
````

A section run through `call` reads the `var` values the caller set before the call, because it receives a copy of the caller's `var`: a caller `var` holding `k = 1` arrives in the chain holding `k = 1`. This holds through nesting too, so a `Leaf` reached through a `call` inside a task reads a `var.who` that only the main section set. A `tasks.fanout` takes the same copy.

Writes the called chain makes to its copy of `var` stay in the chain and are discarded when it ends, so they never reach the caller. That is why `assert(var.child_write == nil)` still passes in `Main` after the call. A child section entered with `call` also starts without the Lua globals its parent defined, so with `greet` defined in `Parent`, `tostring(greet)` in `Child` renders `nil`.

Because a called chain's `var` writes never reach the caller, pass results back as the chain's return value, which `call` hands to the caller, as `local r = call('../Sub')` gets `sub saw caller` from the `return` in `Sub`.

The snapshot needs the `var` global itself to stay in place. After you reassign the `var` global, as in `var = 5`, the next `call` or `tasks.fanout` fails at the call when it takes its copy of `var`, with an [error value](05-lua-environment.md#error-values) of kind `lua` whose message holds this refusal after the file and line of the call:

````text
the `var` global was reassigned; write `var.<field>` instead
````

So write `var` field by field, as `var.<field>`.

## Catching call failures

After this section you can catch a failed `call` with `pcall`, fall back to another value, and read what the caught error value holds.

````markdown
---
name: risky
description: Falls back when a called section fails.
runvil: 0
---

## Main

```lua
local ok, result = pcall(call, '../Risky')
if not ok then
  return 'fallback'
end
return 'risky said ' .. result
```

## Risky

```lua
error('risky failed')
```
````

````text
fallback
````

### What `pcall` catches

You catch a failing `call` by writing `pcall(call, '../Section')`, which returns `false` and the error value instead of ending the run. To fall back after a failed call, check the first value from `pcall(call, ...)`. It is `true` followed by the called chain's result on success, which is `nil` for a chain that ended with no text, and `false` followed by the failure on error, as in `local ok, result = pcall(call, '../Risky')` with a fallback when `ok` is false. If `Risky` returned `'fine'` instead, the run above would end with `risky said fine`.

A `pcall` around a `call` catches each of these, and each comes back to the calling block as the call's ordinary error instead of ending the run:

- A call refused by the recursion depth cap (see [The recursion depth cap](#the-recursion-depth-cap)).
- A call whose path names no section in the visible set, which fails with an error containing `not found`.
- A call whose called chain ends in an error, such as one whose error value has `err.kind == 'tasks_live'` when the called chain ends with a live task (see [Task events and task errors](15-tasks.md#task-events-and-task-errors)).

Path errors from `call` are raised at the call site as the call's own error, so a `pcall` inside the calling block catches them. That covers a non-string target, as when `pcall(call, 42)` fails with `section target must be a string`, a malformed path, and a path that names no section, which fails with a message that contains `not found` and names the path.

### What the caught error value holds

A failed `call` raises the Engine's error value in the calling block, so `pcall(call, ...)` returns `false` and that error value. A failure inside a `call` reaches the caller as the called chain's own error value, with its `kind`, its message, and its kind's fields intact, and it is never rewrapped as a generic `call` failure. Its message keeps the line where the failure happened inside the called chain, not the line of the `call` (see [Where an error points](16-limits-and-errors.md#where-an-error-points)). Here `Leaky` starts a task with `tasks.spawn` and ends while that task is still live:

````lua
local ok, err = pcall(call, '../Leaky')
````

`ok` is `false`, `err.kind` is `'tasks_live'`, `err.tasks` is `'0.0.0'`, and `tostring(err)` gives this message, whose line is that of the `tasks.spawn` in `Leaky` that started the task left running:

````text
{file}:{line}: chain ended with tasks still live: 0.0.0; wait on or cancel every task a chain spawns before it ends. The chain ended with tasks still running. Wait on or cancel every task the chain spawns before it ends.
````

### What `pcall` cannot catch

A determinism violation inside a called chain (see [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism)) is not caught by a `pcall` around the `call`. It ends the run on the spot instead of coming back to the caller as the call's error.

## The recursion depth cap

After this section you can predict how deep calls nest, how call depth counts across `call`, tasks, and fanout arms, and what the `CallDepthExceeded` failure looks like.

````markdown
---
name: ping-pong
description: Two sections call each other until the depth cap stops them.
runvil: 0
---

## Alpha

```lua
return call('../Beta')
```

## Beta

```lua
return call('../Alpha')
```
````

The prompt parses and runs until the recursion depth cap fires, and then the run fails with code `call_depth_exceeded` and this message, which points at the `call` in `Alpha` that would have nested one level too deep:

````text
recursion exceeded cap of 8 (at ping-pong.md:10)
````

### Mutual recursion

A section's visible set excludes the section itself, for `call` just as for `tasks.spawn`, so a section cannot target itself: `return call('../Self')` written in `Self` fails with a `not found` error. Recursion has to go between two different sections that target each other, such as `Alpha` and `Beta` above. That is mutual recursion: two sections call each other back and forth, each nested call adding one level of call depth, until the recursion depth cap stops them. In the prompt above, `Alpha` runs at depths 0, 2, 4, 6, and 8. The pair can be child sections too, such as `X` running `return call('../Y')` and `Y` running `return call('../X')`.

The recursion depth cap is the limit of 8 nested calls along one call chain, so unbounded mutual recursion such as `A` running `return call('../B')` and `B` running `return call('../A')` fails the run when it reaches the cap of 8. Calls nest up to 8 levels deep, counting the first call, and the ninth nested call fails the run with the message `recursion exceeded cap of 8` and the location of that call.

You can catch the refusal with `pcall` at the deepest level:

````markdown
---
name: caught-cap
description: Catches the depth cap refusal and reports its kind.
runvil: 0
---

## Alpha

```lua
local ok, r = pcall(call, '../Beta')
if ok then return r end
return 'caught ' .. r.kind
```

## Beta

```lua
return call('../Alpha')
```
````

````text
caught lua
````

`Alpha` at depth 8 cannot start `Beta` at depth 9, so its `pcall` returns `false` and the depth cap error. It returns `caught lua`, and each level above it hands that reply straight back up to the top.

### How call depth counts

Call depth is how many nested calls deep a section runs. A top-level section runs at call depth 0, which is why `Alpha` first runs at depth 0. Call depth accumulates through nested calls: each `call` runs the called section at its caller's depth plus one, and that called section keeps its depth, while the main walk of top-level sections always starts at depth 0. The [main walk](04-how-a-prompt-runs.md#the-section-walk) runs every top-level section, the preamble included, at the root call depth of 0, so `call` nesting and its cap count from there.

The cap is 8. A section or fanout arm may run at depth 8, but nothing it starts may run at depth 9, so an arm already running at depth 8 cannot `call`. The cap is fixed and cannot be configured.

A `jump` never adds a level of call depth. A descent into a child level is not a call, so the child walk shares the chain's call depth, and jumping into children spends no depth. When `Main` does `jump('X')` and then `X` and `Y` ping-pong with `call`, nine section entries run before the cap trips, in the order `x`, `y`, `x`, `y`, `x`, `y`, `x`, `y`, `x`, where a descent that spent depth would allow only eight.

### Depth across tasks and fanout arms

Each `call`, each task started by `tasks.spawn`, and each [fanout arm](14-fanout.md#what-an-arm-sees) adds one level toward a single recursion depth cap of 8 levels, first call included. Depth adds up across all three, task boundaries included, so sections that call each other in turn add one level per call.

| Started by | Adds a level | Message at the cap |
| --- | --- | --- |
| `call` | yes | `recursion exceeded cap of 8` |
| `tasks.spawn` | yes | `recursion exceeded cap of 8` |
| A `tasks.fanout` arm | yes | `recursion exceeded cap of 8` |
| `jump` | no | none |

Each `tasks.fanout` arm runs one call level deeper than the section that called `tasks.fanout`, so arms count against the same cap that `call` uses. A section at depth 7 can still fan out, but a `call` made inside one of its arms trips the cap: when `B` fans out at depth 7 over `W`, whose body is `return call('../C')`, the arm's `call` fails with `recursion exceeded cap of 8`, and uncaught it fails the run with a `call_depth_exceeded` error.

Call depth accumulates across a fanout boundary, so fanning out does not reset the depth, and a section already at the cap cannot fan out, because its arms would run one level deeper still. In a ping-pong like the one above where `Alpha`, on its fifth run at depth 8, does `return tasks.fanout('../W', {'x'})` instead of calling `Beta`, the run fails with a `call_depth_exceeded` error whose message is `recursion exceeded cap of 8` followed by the location of that `tasks.fanout`. Calling `tasks.fanout('Worker', {'x'})` from any section at depth 8 fails the same way, since its arms would run at depth 9.

A fanout whose arms would pass the cap starts no arm. Every arm would run at the same depth, so the first arm's start is refused, the fanout stops, and it raises the depth cap error, which `pcall` reads as kind `lua`. Its text is `recursion exceeded cap of 8`, the same as for a `call`, and it reaches the run unchanged through every enclosing `call` level, with no Lua runtime-error prefix or traceback. Even after eight nested `call` levels, the run's message is `recursion exceeded cap of 8 (at {file}:{line})`, with the line of the `tasks.fanout` in the chain that went too deep.

Tasks started by `tasks.spawn` run one call level deeper than the section that spawned them, count toward the same cap as `call`, and share `call`'s target resolution too. A task refused at the cap reads `recursion exceeded cap of 8`, the same text as for `call`, whether `tasks.spawn` started it or it is a fanout arm. A `tasks.spawn` refused at the cap is the spawn's own answer, so `pcall(tasks.spawn, ...)` catches it in the spawning block instead of ending the run. With `local ok, err = pcall(tasks.spawn, '../Beta')` followed by `log(tostring(err))`, the deepest section logs this message, whose line is that of the refused `tasks.spawn`:

````text
{file}:{line}: recursion exceeded cap of 8. Calls nest deeper than the cap allows. Nest `call`, `tasks.spawn`, and `tasks.fanout` less deeply.
````

The run then ends with the `done` fallback or a `tasks_live` error rather than with the cap error. When nothing catches the refusal and it ends the run, it is a `call_depth_exceeded` error.

The `depth` field of [`tasks.status`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) reports a task's call depth, the same count the cap checks, so `tasks.status(t).depth` reads 1 for a task spawned from the main walk.

### The `CallDepthExceeded` failure

`CallDepthExceeded` is the dedicated depth cap error. Its text holds the cap and nothing else, in the form `recursion exceeded cap of {cap}`, so it reads `recursion exceeded cap of 8` whether a `call`, a `tasks.spawn`, or a `tasks.fanout` arm went too deep. The error points at the line of the `call`, `tasks.spawn`, or `tasks.fanout` that would have started a chain past the cap.

Going past the cap fails with the text `recursion exceeded cap of 8`, exactly, whichever of the three went too deep.

Caught with `pcall`, the depth cap error is an error value of kind `lua`, the same kind as any Lua-phase failure, with no dedicated kind of its own, so a `pcall` handler that already branches on the `lua` kind keeps working, and its `code` is `call_depth_exceeded`. Its message is the file and line of the call that went too deep, then the text, then the fixed prose for the cap, as the logged message above shows, and the error value has no field for the depth reached or for the cap.

Uncaught, the depth cap error reaches the run's result unchanged through every calling section, so you see the specific depth cap error and its message rather than a generic wrapper, still pointing at the line where it happened. It fails the run with code `call_depth_exceeded`, whichever construct went too deep. So a `call` that would nest past the cap ends the run with code `call_depth_exceeded` and a message that names the cap and the line of that `call`, as `recursion exceeded cap of 8 (at ping-pong.md:10)` does for the prompt at the top of this section.

## Jump and call from the preamble

After this section you can use `jump` and `call` from the preamble, and know which sections they can name there.

````markdown
---
name: answer-once
description: Calls a section once from the preamble and keeps the result.
runvil: 0
---

```lua
var.answer = call('../Answer')
```

## Result

```lua
return var.answer
```

## Answer

```lua
return 'called from the preamble'
```
````

````text
called from the preamble
````

The [preamble](02-file-structure.md#the-preamble) is the text between the frontmatter and the first heading, and when it holds a `lua` fence it is the first top-level section of the prompt. `call`, `jump`, and `tasks.fanout` work in its `lua` blocks just as they do in every other section. Here the preamble calls `Answer` and keeps its reply in `var`, and the [walk](04-how-a-prompt-runs.md#the-section-walk) then falls through to `Result`, which returns it. In the same way, `tasks.fanout('../Worker', {'a', 'b'})` in the preamble runs its arms over the top-level `Worker`.

### The preamble's visible set

The preamble's visible set is every other top-level section, and no nested section. The preamble is a top-level section itself, so a relative path written there starts at the preamble, whose siblings are the sections under the top-level headings, and code there writes `call('../Answer')` and `jump('../Target')` with `../` in front. The preamble has no children, so a bare `'Name'` names nothing there. Code in the preamble can `jump` to or `call` any other top-level section, and only top-level sections are valid `jump` targets from the preamble. The `tasks` functions take the same paths there, as in `tasks.spawn('../Task')`.

The preamble has no name, and every name in a section path has at least one character, so no section path names the preamble. No `jump` or `call` leads back to it, and it runs once, as the first section of the run.

The prompt reads work from the preamble too. `prompt.section('../Main')` reads the top-level `Main`, `prompt.section_or('../Kid')` is `nil` for a nested `Kid`, and a path from the top, such as `prompt.section('/Main/Kid')`, reads any section. With a top-level `Nums` section whose list holds `one`, `two`, and `three`, `table.concat(prompt.section('../Nums', 'list').items, ',')` in the preamble gives `one,two,three`.

A `call` from the preamble resolves its path through the same lookup a `call` from any other section uses, starting at the preamble.

### Jumping from the preamble

A `jump` from the preamble moves the main walk to the named top-level section, so every section between the preamble and the target is skipped:

````markdown
---
name: preamble-jump
description: Jumps from the preamble to a later section.
runvil: 0
---

```lua
jump('../Target')
```

## Skipped

```lua
error('the jump target must skip this section')
```

## Target

```lua
return 'jumped'
```
````

````text
jumped
````

The jump ends the preamble as it ends any section: the rest of the jumping block and the preamble's later blocks do not run, and `Target` starts from the preamble's final `var` (see [What a jump ends](#what-a-jump-ends) and [What crosses a jump](#what-crosses-a-jump)).

### Calling from the preamble

In the preamble, `call('../Name')` runs a top-level section as a contained chain and returns whatever that chain returns, as `var.answer = call('../Answer')` gets `called from the preamble` in the first prompt of this section. A chain that ends with no text gives the preamble's block `nil`, as it does in any block (see [A call that gets no text](#a-call-that-gets-no-text)).

To compute a value once before the sections under the headings run, call a section from the preamble, keep the result in `var`, and place the call target after the section that returns. In that prompt, `Result` returns `var.answer` and sits before `Answer`, so the walk never runs `Answer` on its own.

The preamble and the sections after it are entries of one root chain, so calls made in any of them share a child counter and take consecutive child ids. A `call` made in the preamble is child `0.0`, and the next `call` on the walk is child `0.1`:

````markdown
---
name: preamble-ids
description: Shows the child ids of a call from the preamble and from the next section.
runvil: 0
---

```lua
var.first = call('../Answer')
```

## Result

```lua
return var.first .. ',' .. call('../Answer')
```

## Answer

```lua
return sys.id
```
````

````text
0.0.0,0.1.0
````

### Failures from the preamble

A `jump`, `call`, or `tasks.fanout` fails from the preamble exactly as it fails from any other section. Here the preamble names the top-level `Answer` with a child's path:

````markdown
---
name: preamble-miss
description: Calls a top-level section from the preamble as if it were a child.
runvil: 0
---

```lua
return call('Answer')
```

## Answer

```lua
return 'answer'
```
````

The run fails with code `lua_runtime`. Its message is the not-found error's text, followed by the line of the `call`:

````text
section `Answer` not found: the preamble has no child `Answer` (at preamble-miss.md:8)
````

The preamble has no children, so every child path written there misses with `the preamble has no child`: `jump('Answer')` and `tasks.fanout('Answer', {'a', 'b'})` in that block fail with this same message. The sibling path `'../Answer'` names the section, and with `return call('../Answer')` in the block the run ends with `answer`. A sibling path that names no top-level section misses with the sibling form. Change the block to `jump('../Missing')` and the run fails with:

````text
section `../Missing` not found: the preamble has no sibling `Missing` (at preamble-miss.md:8)
````

`call` and `tasks.fanout` raise at the call site, while `jump` is looked up after the preamble's block has finished:

| From the preamble | Where it fails | A `pcall` in the block |
| --- | --- | --- |
| `call` or `tasks.fanout` with a path that names no section | at the call site | catches it |
| `jump` with a path that names no section | after the preamble's block ends | cannot catch it |
| `jump` with a malformed path | after the preamble's block ends | cannot catch it |

When `call` in the preamble is given a path that names no top-level section, such as `call('../Nope')`, it raises the not-found error, whose text is ``section `../Nope` not found: the preamble has no sibling `Nope` ``. It raises at the call site, so `pcall` catches it: `local ok, err = pcall(call, '../Nope'); return tostring(ok) .. ':' .. tostring(err)` gives a result that starts with `false:` and contains `Nope`.

An uncaught not-found error raised from the preamble ends the whole run, and you see it as the run's failure message. The run ends with code `lua_runtime`, and its message is the not-found error's text, followed by the line of the `call` or `tasks.fanout` that failed.

When a `jump` from the preamble is given a path that names no top-level section, such as `jump('../Nope')`, the run fails with a `lua_runtime` not-found error whose message names the path and points at the line of the `jump`. The path is looked up after the preamble's block has finished, so a `pcall` around the `jump` does not catch the failure (see [Lookup after the block ends](#lookup-after-the-block-ends)). A malformed path given to `jump` in the preamble ends the run with code `lua_runtime` the same way.

## Jump and call in arms, tasks, and tool handlers

After this section you can use `jump` and `call` inside fanout arms, tasks, and adhoc tool handlers, and know which sections each one can name.

````markdown
---
name: arm-jump
description: An arm jumps from its worker section to a sibling.
runvil: 0
---

## Parent

```lua
local r = tasks.fanout('Worker', {'alpha'})
return r[1].text
```

### Worker

```lua
jump('../Target')
error('the arm remaining blocks are skipped')
```

### Target

```lua
var.trail = 'Target '
```

### Tail

```lua
return var.trail .. 'Tail'
```
````

````text
Target Tail
````

`Parent` runs a [fanout](14-fanout.md#what-an-arm-sees) over one member, so one arm runs the worker section `Worker`. Its `jump('../Target')` ends the worker section's code, so the `error` never fires, and the arm's walk runs `Target` and falls through to `Tail`, whose reply becomes the arm's text and then the run result.

The rest of this section uses these headings, from a larger prompt where `Parent` runs `tasks.fanout('Worker', ...)` and each arm runs `Worker` for one member, to show what an arm can name and what `jump` and `call` do inside it:

````markdown
## Parent

### Items

### Worker

#### Child

#### ChildTail

### Sub

#### Leaf

### Target

### Tail
````

### The worker visible set

An arm resolves section paths from the worker section it runs, so the sections it can run, the worker visible set, are the worker section's sibling sections other than the worker section itself, plus the worker section's own children. In the headings above, the worker visible set is `Items`, `Sub`, `Target`, `Tail`, `Child`, and `ChildTail`. An arm can read the list in `Items`, a sibling of `Worker`: with `Items` listing `x` and `y`, an arm for the member `alpha` that returns `item .. ':' .. table.concat(prompt.section('../Items', 'list').items, ',')` gives `alpha:x,y`.

The worker visible set never includes the worker section itself, so an arm cannot `call` or `jump` back into its own worker section. The limits on `jump` and `call` hold wherever a section runs, so a worker section running as an arm cannot run its own parent section either. From `Worker` running as an arm, both `jump('../Parent')` and `pcall(call, '../Parent')` fail with not-found:

````markdown
---
name: arm-reach
description: An arm tries to jump to its parent section.
runvil: 0
---

## Parent

```lua
tasks.fanout('Worker', {'x'})
```

### Items

- x
- y

### Worker

```lua
jump('../Parent')
```
````

A `jump` from an arm cannot reach a section outside the worker visible set, such as the top-level `Parent` that called `tasks.fanout`. No relative path written in `Worker` names `Parent`, so the jump fails the run with code `lua_runtime` and a not-found message that names the worker section it searched:

````text
section `../Parent` not found: `/Parent/Worker` has no sibling `Parent`
````

### Jumping inside an arm

`jump` works in the Lua blocks of any section, including a section running as a fanout arm. Calling `jump(path)` inside an arm ends the worker section's code at that point, so the lines after the `jump` never run, as the `error` in `Worker` in the first prompt of this section shows.

A `jump` from an arm to a sibling of the worker section runs the target section and then falls through to the target's following sibling sections, and the last reply of that walk becomes the arm's text, or the empty string when no section in the walk replies. With the headings above, `Worker` runs `jump('../Target')`, then `Target` and `Tail` run in that order, and when `Tail` returns `'tail-reply'`, the arm's text is `tail-reply`. When `Target` has no `return` and no later section replies, the arm's text is `""`.

An arm can also `jump` into one of the worker section's own children. `jump('Child')` runs that child and then falls through to the worker section's later children, so `Child` and `ChildTail` run in that order, and when `ChildTail` returns `'child-tail-reply'`, the arm's text is `child-tail-reply`.

### Calling inside an arm

`call` inside an arm runs a contained chain over the worker visible set and returns that chain's final reply as the value of the `call`, as in `local got = call('../Sub')`, where `Sub` is a sibling of the worker section. A chain started by `call` from an arm falls through from the called section to its following siblings, and the last section's return value becomes the `call` result. When `Sub` sets `var.leaves` and has no `return`, `Target` has none either, and `Tail` does this:

````lua
return 'tail-reply:' .. var.leaves
````

then `got` holds `tail-reply:` followed by the value `Sub` stored, such as `tail-reply:x!y!`. When the chain ends with no text, `got` is `nil`, as for a `call` in any block (see [A call that gets no text](#a-call-that-gets-no-text)), while the arm's own text is still the empty string when the arm ends with no text.

A `call` from an arm whose path names no section in the worker visible set raises a not-found error that the arm can catch with `pcall`:

````lua
local ok_call, err_call = pcall(call, '../Parent')
````

`ok_call` is false and `tostring(err_call)` contains `not found`, so the arm can go on to return its own text, such as `alpha:rejected`.

### Calls stay inside the calling task

A section run with `call` runs as a chain inside the calling task, not as a new task. It emits no [task events](15-tasks.md#task-events-and-task-errors) of its own, and its effects and events are reported under the task that made the call. The main walk is task `0`, and in a two-arm fanout whose arm makes a `call`, only the two arms, `0.0` and `0.1`, report as succeeded tasks.

A section run with `call` starts with the same [concurrency limit](14-fanout.md#concurrency-and-the-admission-window) as the section that called it, so the tasks it spawns run within the caller's share. It takes no concurrency slot of its own, and while it waits on tasks it spawned, it gives back the slot of the task it runs inside. So a section called from an arm can still fan out after the section that started the fanout set `tasks.concurrency(1)`. When `Parent` runs `tasks.concurrency(1)` and fans out to `Worker`, the worker section runs `call('../Sub')`, and `Sub` fans out over its own child `Leaf`, the run completes, with `worker:tail-reply:x!y!:alpha` as its text.

A `call` made after a fanout takes the caller's next child number after the arms, because called sections, spawned tasks, and each timed wait's internal timer share one child counter. In the main walk after a two-arm fanout, the arms are children 0 and 1, the called section is child 2, and a `Sub` that returns `sys.id` returns `0.2.0`.

### Calling from tasks and adhoc tool handlers

A section running as a spawned task can use `call` to run its own subsection, as when `Caller`, which runs only as a task started with [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), runs `call('Leaf')` on its own `Leaf` child. A section can also pass its own child section to `tasks.spawn`, as when `Main` runs `tasks.spawn('Caller')` on its own `Caller` child, and both spawned tasks complete with `r(a)|r(b)`.

You can use `call` inside an [adhoc tool](12-tools.md#how-adhoc-tool-handlers-run) handler, and it works there exactly as it does in a block. `jump` is the one control function a handler cannot use, as [The adhoc tool handler refusal](#the-adhoc-tool-handler-refusal) describes.
