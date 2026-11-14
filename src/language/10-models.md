# Models

A prompt never names a concrete model. Every prompt has five model roles, `default`, `strong`, `small`, `fast`, and `creative`, and the Host decides which model fills each one. A run starts on the `default` role's model, so your Lua can run a model round with `models.infer` right away, and it switches to another role or to a model handle when a section needs one. Because a prompt asks for roles rather than models, the same file runs on whatever models the Host provides, and the requirements you attach to a role are checked before any of your Lua runs.

## Model roles instead of models

After this section you can send prose to the Host's model, name the five model roles, and tell when a prompt needs to say anything about a role at all.

````markdown
---
name: hello
description: Says hello.
runvil: 0
---

## Greet

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

The frontmatter declares no model role, and the `Greet` section sends its prose to the model with `return models.infer(prose)`, so the model's reply to `Say "Hello, world!"` becomes the run result. A prompt that only wants the Host's model needs nothing else: no `models:` key, no selection, and one `models.infer(prose)` call in a section. The bundled `hello.md` prompt is built this way, with no `models:` key in its frontmatter and `return models.infer(prose)` at the end of its `Greet` section.

### What a model role is

A model role is a name for a kind of model a prompt can ask for. There are exactly five, fixed by the Engine, and every prompt has all of them:

| Role | What the name suggests |
|---|---|
| `default` | the model every other role falls back to, and the model a run starts on |
| `strong` | the most capable model, for hard reasoning |
| `small` | a compact model for light work |
| `fast` | the lowest-latency model |
| `creative` | a model for open-ended writing |

The names are only hints. The Engine does not enforce any of those meanings, and a role is nothing more than a name that the [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host) maps to a model. The prompt never says which model that is, so the same prompt runs on whatever models the Host provides.

The Host passes one map from role to model. A role with no entry of its own uses the `default` entry, so a Host that gives one model, as the Harness does with the model the Host selected, fills all five roles with that model. [How the Host fills and checks roles](#how-the-host-fills-and-checks-roles) covers the map and the checks. Role names are lowercase and exact, so `Strong` is not a role.

You attach requirements to a role in the frontmatter, such as a context minimum, under `models:`. A role declares only two keys, `min_context` and `thinking`, and none of them names a concrete model. [Declaring roles under `models:`](#declaring-roles-under-models) covers both keys.

The Host's list of models stays with the Host. A prompt's roles draw only on the models the Host's map gives them, so a prompt reaches no other model, with one exception: when the Host passes a Host-state snapshot, `models.get` also accepts a raw gateway model id (see [Raw model ids and the Host-state snapshot](#raw-model-ids-and-the-host-state-snapshot)).

### The `models` global

The `models` global is the table your Lua uses to choose and run models. Every section has it, and it is reserved as an [Engine global](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals). You can reach it from the Lua of every section and from a [library](03-blocks-and-prose.md#sharing-code-with-require). That includes the [preamble](02-file-structure.md#the-preamble), the text between the frontmatter and the first heading, which is the first top-level section when it holds Lua.

The table holds six functions. `models.use` selects a role or a handle, `models.get` returns a model handle without selecting it, `models.current` returns a handle for the model in use, `models.infer` runs one model round, and `models.loop` and `models.compact` are taught with conversations:

| Function | Where it is taught |
|---|---|
| `models.use` | [Selecting a role with `models.use`](#selecting-a-role-with-modelsuse) |
| `models.get` | [Getting handles with `models.get`](#getting-handles-with-modelsget) |
| `models.current` | [Reading the current model with `models.current`](#reading-the-current-model-with-modelscurrent) |
| `models.infer` | [Running a round with `models.infer`](#running-a-round-with-modelsinfer) |
| `models.loop` | [Starting a conversation](11-conversations.md#starting-a-conversation) |
| `models.compact` | [Compacting on demand with `models.compact`](11-conversations.md#compacting-on-demand-with-modelscompact) |

The `models` table is in place with the other Engine globals before any block runs, so any block, its top-level statements included, calls it directly with no `require` and no setup line. The block in `hello` uses it that way.

The roles are fixed, and Lua never creates or binds one, so the `models` table holds no function for adding a role. `models.use` and `models.get` take one of the five role names, and any other name is an error that lists the five (see [Rejected calls](#rejected-calls)), except that when the Host passes a Host-state snapshot, `models.get` also accepts a raw gateway model id (see [Raw model ids and the Host-state snapshot](#raw-model-ids-and-the-host-state-snapshot)). The run fills its roles before any Lua runs, and every section, the preamble included, reads that same set, so the same roles work in every section.

### When a prompt needs a role

A prompt needs to say nothing about roles to run a model round. A run starts on the `default` role, so `models.infer(prose)` runs on the Host's `default` model, as `hello` does. A prompt names a role in two cases: it attaches a requirement to a role under `models:`, or it switches to another role with `models.use`.

A section has no model only when the Host gave the run none. This prompt names no role, and on a Host that gave no model its round has nothing to run on:

````markdown
---
name: unselected
description: Sends prose to a model.
runvil: 0
---

## Only

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

On a Host that gave the run no model, the run fails with this message, which names the section whose round had no model and ends with the line of the `models.infer` call (see [Where an error points](16-limits-and-errors.md#where-an-error-points)):

````text
model binding required for section Only (at unselected.md:12)
````

This is the missing-model error, covered in [Infer errors and the missing-model error](#infer-errors-and-the-missing-model-error). On a Host that gives the run a model, the same prompt runs as `hello` does.

A prompt that runs no model round needs no `models:` declaration at all. The bundled `echo.md` prompt has no `models:` key in its frontmatter and never touches `models` in its body. Prose alone never needs a model either, because only an explicit `models.infer` of it does. A prompt whose blocks run no model round runs to completion even when the Host provides no model at all:

````markdown
---
name: word
description: Returns its own prose without a model.
runvil: 0
---

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

The block renders its prose and returns it, and no model takes part, so this run gives the same result on a Host with no model.

## Declaring roles under `models:`

After this section you can attach a context minimum and a thinking requirement to any of the five roles.

````markdown
---
name: research
description: Attaches requirements to the strong and fast roles.
runvil: 0
models:
  strong:
    min_context: 200000
    thinking: true
  fast:
    thinking: false
---

## Main

```lua
return 'done'
```
````

On a Host whose model for `strong` can think and has a context window of at least 200000 tokens, and whose model for `fast` can run with thinking turned off, the run returns:

````text
done
````

The `models:` key is optional. When it is there, it holds a map from role name to that role's declaration, with one entry per role the prompt attaches requirements to. Here it declares two roles, `strong` and `fast`. A role accepts exactly two keys, `min_context` and `thinking`, and each one is optional. One role can use both, as `strong` does, and `fast` sits beside it with only `thinking`.

Declaring a role does not select it. The run still starts on the `default` role, and a section runs on `strong` only after `models.use('strong')`. The requirements are checked at the run's first step whether or not the prompt ever selects the role.

| Key | What it takes | When you leave it out |
|---|---|---|
| `min_context` | a whole number of tokens from 1 to 4294967295 | no context minimum |
| `thinking` | `true` or `false` | no thinking requirement and no thinking switch |

### Empty roles and no roles

A role written as an empty map, such as `strong: {}` or `fast: {}`, is valid. It declares no requirement, so it places none on the model that fills the role, and it changes nothing about the run.

The `models` key itself is optional. Leaving `models:` out of the frontmatter declares no requirements, and all five roles stay available: `hello` has no `models:` key and can still pass any of the five names to `models.use`.

### The five roles

Each key under `models:` must be one of `default`, `strong`, `small`, `fast`, and `creative`, the five roles from [What a model role is](#what-a-model-role-is). Names are lowercase and compared exactly, so `Strong` and `strong ` are not roles. Lua code refers to a role by the same name, written as a string, as in `models.use('strong')`.

A role name means the same thing in every prompt, and the model that fills it is the Host's choice for each run. A requirement under `strong` is checked against whichever model fills `strong` on that run, and a requirement on a role the Host's map gives no entry of its own is checked against the `default` model (see [The role map](#the-role-map)). The set is closed: a prompt cannot add a sixth role.

### The `min_context` key

`min_context` sets the smallest context window, in tokens, that the model filling the role must provide, such as `min_context: 200000` or `min_context: 40000`. A model with a smaller window is refused, so a 32000-token model against `min_context: 200000` is refused with a line saying the role `requires a context of at least 200000 tokens`.

`min_context` takes a whole number of tokens from 1 to 4294967295, and any other value fails the parse with a `Frontmatter` error. The smallest minimum you can write is 1, so a role that wants no minimum leaves the key out. Leaving out `min_context`, or setting it to null, declares no context minimum, and the role sets no floor on the model's context window.

Parsing a `models` declaration checks only its shape. The checks against a real model happen at the run's first step, after the version check and the argument check, once the Host's models fill the roles, and any shortfall refuses the run there. So a prompt declaring `strong` with `min_context: 200000` parses cleanly, and only a run whose model for `strong` has a smaller window refuses it, with `role 'strong': requires a context of at least 200000 tokens` (see [The context check](#the-context-check)).

### The `thinking` key

`thinking` takes `true` or `false`. `thinking: true` requires a model that can think and turns thinking on for the role's requests, and `thinking: false` requires a model that can run with thinking turned off and turns it off for the role's requests. Leaving the key out places no requirement on the model and sets no thinking switch. Any other value, such as a string or a number, fails the parse with a `Frontmatter` error. [The thinking requirement](#the-thinking-requirement) covers which models meet each value and what the switch does.

### Flow style

A role declaration can use YAML flow style (see [Writing YAML in the frontmatter](02-file-structure.md#writing-yaml-in-the-frontmatter)) to fit on one line, and a whole `models:` map with several roles can be written the same way. This `models:` line, part of a prompt's frontmatter, declares a `strong` role with both keys, a `fast` role with a thinking requirement, and an empty `small` role:

````markdown
models: { strong: { min_context: 200000, thinking: true }, fast: { thinking: false }, small: {} }
````

### Mistakes inside `models:`

The blocks below show the part of each message that names the mistake. The full message starts with `invalid frontmatter: ` and contains that text, and the YAML reader can add a path before it and a line and column after it.

A role name outside the five fails the parse with a `Frontmatter` parse error, and the message lists the five names:

````text
expected one of `default`, `strong`, `small`, `fast`, `creative`
````

The message comes from the YAML reader and contains that list, so a line such as `models: { reviewer: {} }` is reported instead of being silently ignored.

A role accepts no keys besides `min_context` and `thinking`. Any other key fails the parse with a `Frontmatter` error whose message names the key and the two that are accepted. A key named `tier` gives:

````text
unknown field `tier`, expected `min_context` or `thinking`
````

`description` and `keywords` are not role keys, so they fail the same way. A value of the wrong kind fails the parse with a `Frontmatter` error too, such as `min_context: 0`, a negative or fractional `min_context`, or a `thinking` that is not `true` or `false`.

A mistake inside `models:` fails the parse with a `Frontmatter` parse error that reports the line and column of the mistake in the whole file, counting the opening `---` line. So a rejection inside a contract key can report line 5, column 5, a position in the file you wrote rather than in the YAML alone. [Where an error points](16-limits-and-errors.md#where-an-error-points) covers error locations.

## The thinking requirement

After this section you can require thinking on or off for a role, tell which models meet the requirement, and predict whether the role's rounds ask the model to think.

````markdown
---
name: boiling-point
description: Answers one question with thinking turned off.
runvil: 0
models:
  fast:
    thinking: false
---

```lua
models.use('fast')
```

## Ask

At what temperature does water boil at sea level?

```lua
return models.infer(prose)
```
````

The `fast` role declares `thinking: false`, so the run checks that the model filling `fast` can run with thinking turned off, and every round on `fast` asks for thinking off, including the round in `Ask`. That round reaches `fast` through the `models.use('fast')` call in the preamble, because the choice lasts for the rest of the chain (see [The current model](#the-current-model)).

### Thinking modes

Every model has exactly one of three thinking modes: it never thinks, it always thinks, or it can switch thinking on and off per request. The mode belongs to the model, separate from the role's thinking switch described below. A model's thinking mode is spelled in lowercase as `never`, `always`, or `switchable`, and the requirements notice names the two fixed modes `Never` and `Always`.

| Thinking mode | What the model does | `thinking: true` | `thinking: false` |
|---|---|---|---|
| `never` | always runs with thinking off | refused | met |
| `always` | always emits thinking tokens | met | refused |
| `switchable` | lets each request turn thinking on or off | met, with thinking on | met, with thinking off |

A model in thinking mode `never` always runs with thinking off, so a role with `thinking: true` on it refuses the run with the message `requires 'thinking: true'; the thinking capability of the model filling it is Never`. A model in thinking mode `always` always emits thinking tokens, so a role with `thinking: false` on it refuses the run with the message `requires 'thinking: false'; the thinking capability of the model filling it is Always`. A model in thinking mode `switchable` lets each request turn thinking on or off, so a role with `thinking: false` on it runs and sends its requests with thinking off.

### The thinking switch

A role's `thinking` value sets its thinking switch, the thinking setting that every request on the role sends. `thinking: true` asks for thinking on in every request, and `thinking: false` asks for thinking off. A role that leaves `thinking` out leaves the switch unset, so its requests send no thinking setting at all, and the model's own thinking default stays in place.

The switch is fixed once per run from the role's declaration, when the run fills its roles. Every round on the role sends that same switch, even when `models.use` sets sampling options, which change only `temperature` and `max_tokens` (see [Sampling options on `models.use`](#sampling-options-on-modelsuse)). A handle for the role reads the switch back as its `thinking` field (see [The nine handle fields](#the-nine-handle-fields)).

The switch takes effect on a model that can switch thinking per request. A fixed-mode model only ever runs with a matching switch, because `thinking: true` on a `never` model and `thinking: false` on an `always` model refuse the run on its first step.

A role with `thinking: false` sends every request with thinking off, whether a section picks the role with `models.use` or the chain reaches it from an earlier section, and including when the model filling the role is `switchable`. In `boiling-point` above, the preamble's `models.use('fast')` puts the chain on `fast`, and the round in `Ask` runs there, yet it still asks for thinking off.

## How the Host fills and checks roles

After this section you can tell which model fills each role, read a model id, and predict which requirements the run reports as unmet.

````markdown
---
name: two-roles
description: Attaches requirements to two roles that one model fills.
runvil: 0
models:
  strong:
    min_context: 16000
  small:
    thinking: false
---

## Main

```lua
return 'done'
```
````

````text
done
````

On a Host whose map has only a `default` entry, for a model with a context window of at least 16000 tokens that can run with thinking turned off, `strong` and `small` both fall back to that entry. Both requirements are met against that one model, so this run returns `done`.

### The model catalog

The models that fill a prompt's roles come from the Host, not from the prompt file. The Host supplies a model catalog, its list of available models, in which each model is described by its model info and named by a model id, and it passes the run a role map that names a model for each role it fills (see [The role map](#the-role-map)).

Each model's info holds a description, a context window, a thinking mode, and an optional provider id alongside its model id. The description, context window, and provider id show up on a model handle as `description`, `context`, and `provider` (see [Model handles](#model-handles)), and the thinking mode is what a role's `thinking` requirement is checked against (see [Thinking modes](#thinking-modes)):

- The description is prose that describes the model.
- The context window is a size in tokens. It is always at least one token and at most 4294967295 tokens, the largest unsigned 32-bit value, with windows such as 131072 and 8192 tokens.
- The thinking mode is `never`, `always`, or `switchable`, as [Thinking modes](#thinking-modes) explains.
- The provider id is present only when the catalog names one, and is absent otherwise.

A catalog keeps its models in the order the Host gives them. Each model id in a catalog names exactly one model, so no two available models share an id.

### Model ids

A model id is a model's stable name, made of a server namespace and a model name, and a model handle's `model_id` field comes from it. In this guide a model id is written `server/name`: the server namespace plus the model name the caller sees, such as `gateway/fast` or `gateway/deep`.

Each part of a model id can be any non-empty text without control characters, so a model name may contain `/`, `.`, `:`, and non-ASCII letters, as in `qwen/qwen3-8b`. Names that mix non-ASCII letters, hyphens, and digits, such as `café-模型`, `valid-alias`, and `claude-sonnet-4-6`, are all valid.

Two rules apply to each part, and a part that breaks one is refused with a reason:

- Each part holds at least one character, and an empty part gives the reason `must not be empty`.
- No part contains a control character, judged per Unicode character rather than per byte. Newline, NUL, U+001E, U+007F, and U+0085 are each refused with the reason `must not contain a control character`, while `café-模型` is accepted.

A model served by the gateway has the server namespace `gateway`. Its name is the name the gateway lists it under, which is the `name` key of its entry in the gateway configuration and the `id` field in the OpenAI-compatible `/models` list.

### The role map

The Host fills the roles by passing one map from role to model when it starts the run. Three rules govern the map:

- A role with no entry of its own uses the `default` entry. A map with only a `default` entry fills all five roles with that model, and a map that also has a `strong` entry gives only `strong` its own model.
- An empty map means the Host has no model. No role is filled and nothing is checked (see [When the Host gives no model](#when-the-host-gives-no-model)).
- A non-empty map with no `default` entry is refused, because the roles without an entry have nothing to fall back to.

The refusal ends the run on its first step with error code `requirements_unmet`. The run checks no declared requirement in that case, so the notice has exactly one model line, in which the caller is the Host:

````text
the environment cannot satisfy this prompt:
- role 'default': requires a model; the caller's role map has no 'default' entry
````

The gap is the caller's, not the prompt's, so the message ends without a location.

Every role gets a binding of its own, named for the role, even when the `default` entry fills it. A handle from `models.get('strong')` reads `strong` for `name` whether `strong` has an entry of its own or falls back to `default`, and its `model_id` is the name part of whichever model fills it.

The Harness passes a map with only a `default` entry, the one model the Host selected, and passes an empty map when the Host selected none. Every role is then filled by that one model, so the roles differ only in their requirements and thinking switch. Where an example in this chapter prints a `model_id`, a one-model map prints the same id for every role, and a map that gives `strong` its own model prints two ids.

A role is served by the model that fills it, so a request made on the role goes to that model. A model request names the model by the name part of its id only. A request on a role filled by `gateway/m` sends `"model": "m"`, not the role name and not `gateway/m`, and a model filling a role as `gateway/fast` is requested as `fast`. The role's model handle reads back the same name part as `model_id`.

The run fills its roles once, when it is built, and keeps that fill for its whole life. The requirements are checked against that fill later, at the run's first step, after the version check and the argument check. To serve a role with another model, or to clear a shortfall such as a window that is too small, the Host passes a different map when it starts the next run.

### What the run checks

A role's `min_context` and `thinking` are requirements, and at the run's first step, before any section runs, the run checks every declared requirement against the model that fills the role. A role's own entry in the map fills it, and the `default` entry fills a role with none, so a requirement on a role with no entry of its own is checked against the `default` model. These two checks are the only role requirements that can refuse the run, and the run makes them only when the Host's map has a `default` entry. A role the prompt does not declare, or declares as `{}`, has no requirement, so nothing is reported for it and it never blocks the run.

An unmet requirement is one shortfall in the refusal that ends the run on its first step with error code `requirements_unmet` and no run events (see [The requirements notice](04-how-a-prompt-runs.md#the-requirements-notice)). The run checks a role's `min_context` and its `thinking` as two separate checks, so a role can fail only one of them, and a role that fails both gets one notice line for its context and another for its thinking:

````markdown
---
name: analysis
description: Needs a large context window and a thinking model.
runvil: 0
models:
  strong:
    min_context: 200000
    thinking: true
---

## Main

```lua
return 'done'
```
````

On a Host whose model for `strong` has a 32000-token context window and never thinks, the run fails with error code `requirements_unmet` at its first step, before any of the prompt's blocks run, and its message is the requirements notice, followed by the location of the prompt's `models:` line:

````text
the environment cannot satisfy this prompt:
- role 'strong': requires a context of at least 200000 tokens; the model filling it provides 32000
- role 'strong': requires 'thinking: true'; the thinking capability of the model filling it is Never (at analysis.md:5)
````

The notice opens with the line `the environment cannot satisfy this prompt:`, followed by one `- ` line per unmet check (see [The requirements notice](04-how-a-prompt-runs.md#the-requirements-notice)). Each unmet line names the role by its role name, shown as `role '{role}'` in every line form. The model lines come after any lines about Packs, in role order, `default`, `strong`, `small`, `fast`, `creative`, and for one role the context line comes before the thinking line.

### The context check

At the run's first step, after the version check and the argument check, a role's `min_context` is read as a token count and compared with the context window of the model that fills the role. The minimum is inclusive: a model whose window is at least the minimum, including exactly equal to it, meets the requirement. So `min_context: 200000` passes on a 200000-token model, `min_context: 100000` passes on a 200000-token model, and only a smaller window leaves the requirement unmet.

When a role's `min_context` is above the filling model's context window, the run records an unmet context-minimum requirement that names the role and gives the required and actual sizes, and the run is refused on its first step. The line takes this form:

````text
- role '{role}': requires a context of at least {required} tokens; the model filling it provides {actual}
````

So `strong` with `min_context: 200000` on a 32000-token model is refused with required 200000 and actual 32000. A role `fast` that needs 200000 tokens on that model makes the whole message read `the environment cannot satisfy this prompt:` followed by `- role 'fast': requires a context of at least 200000 tokens; the model filling it provides 32000` and the location of the prompt's `models:` line, ` (at {file}:{line})`.

### The thinking check

The `thinking` requirement is a check against the filling model's thinking mode: when the mode does not match what the role asks for, the role is unmet and the run is refused on its first step. Both lines take this form, where `{required}` is `thinking: true` or `thinking: false` and `{actual}` is `Never` or `Always`, the notice's spelling of the modes `never` and `always`:

````text
- role '{role}': requires '{required}'; the thinking capability of the model filling it is {actual}
````

A role with `thinking: true` is unmet only on a model whose thinking mode is `never`, which refuses the run with the line `- role '{role}': requires 'thinking: true'; the thinking capability of the model filling it is Never`. An `always` or `switchable` model satisfies it, so a role with `thinking: true` runs on an `always` model.

A role with `thinking: false` is unmet only on a model whose thinking mode is `always`, which refuses the run with the line `- role '{role}': requires 'thinking: false'; the thinking capability of the model filling it is Always`. So `models: { small: { thinking: false } }` on an `always` model reports role `small` unmet, with required `thinking: false` and actual `Always`.

A `switchable` model, whose thinking can be turned on or off per request, satisfies both `thinking: true` and `thinking: false`, and it is asked to turn thinking on or off to match the role.

### When the Host gives no model

````markdown
---
name: unread
description: Declares a requirement but never asks a model for a reply.
runvil: 0
models:
  strong:
    min_context: 200000
---

This prose is never sent to a model.

## Main

```lua
return 'ok'
```
````

On a Host that gives no model, the run returns:

````text
ok
````

When the Host gives no model, its role map is empty, so the run fills no role and checks no requirement, and even a role with `min_context` does not stop the run from starting. This prompt declares a requirement on `strong`, but no section asks a model for a reply, and the line above `Main` sits in a [preamble](02-file-structure.md#the-preamble) with no Lua, so it is commentary that nothing reads. The run therefore still succeeds.

Nothing refuses the run in this case, and every role stays empty. Selecting a role with `models.use` or `models.get` then raises an error that names the role, and `models.current()` raises its own, where the caller in both is the Host:

````text
the caller supplied no model for the strong role
the caller supplied no model
````

`models.use('strong')` and `models.get('strong')` give the first message, and `models.current()` gives the second. `models.infer` and `models.loop` with no model handle raise the missing-model error instead (see [When a section has no model](#when-a-section-has-no-model)). So the requirements check passes for a prompt on a Host with no model, and the run fails later, when a section asks for one.

## The current model

After this section you can say which model a round runs on, read it back, and tell which model a section started by `call`, a spawned task, or a fanout arm starts on.

````markdown
---
name: current
description: Switches model in the preamble and reads it in a section.
runvil: 0
---

```lua
models.use('strong')
```

## Check

```lua
return models.current().name
```
````

````text
strong
````

The current model is the model a round runs on when the call names none, such as `models.infer(prose)`. `models.use` sets it and `models.current()` reads it. It belongs to a chain, not to a section: the preamble's `models.use('strong')` set it, and `Check`, a different section, still sees it.

A chain is a walk with its own entries (see [The main walk as a chain](04-how-a-prompt-runs.md#the-main-walk-as-a-chain)). The main walk is one chain, a section run through `call` begins another, and each spawned task is a chain of its own.

### Where a chain starts

Each chain starts on a model the run decides when it begins:

- The main walk, which is the root chain, starts on the `default` role's model, or on no model when the Host gave none. A `models.use` lasts for the rest of its chain, so one in the preamble chooses the model for every section the walk enters after it, as `current` shows, because the preamble and those sections are entries of the same chain. A `jump` between sections stays in one chain too, so it keeps the current model.
- A section run through `call` starts on a copy of its caller's current model, taken when the `call` is dispatched.
- A spawned task starts on a copy of its spawner's current model, taken when the spawn is dispatched. Every fanout arm is a spawned task, so arms start the same way.

Every section a chain enters runs on whatever model the chain is on, so the first section of a prompt that never calls `models.use` runs on `default`. Because the selection lasts across sections, a `models.use('strong')` in one section leaves every later section of the chain on `strong`. To limit a switch to one section, switch back at the end of the section with a handle you saved from `models.current()`.

### Reading the current model with `models.current`

`models.current()` returns a model handle for the chain's current model: the model the last `models.use` in the chain chose, with the options that call set, or the model the chain started on. It takes no arguments, and each call builds a fresh handle that freezes the model at that moment (see [A handle's frozen binding](#a-handles-frozen-binding)). Saving the handle and passing it to `models.use` later puts the chain back on it, options included:

````markdown
---
name: restore
description: Switches to the creative role and back.
runvil: 0
---

## Main

```lua
local previous = models.current()
models.use('creative')
local inside = models.current().name
models.use(previous)
return previous.name .. '|' .. inside .. '|' .. models.current().name
```
````

````text
default|creative|default
````

The chain starts on `default`, so `previous` reads `default`. After `models.use('creative')` the current model reads `creative`, and `models.use(previous)` puts the chain back on `default`. When the Host gave no model there is no current model, and `models.current()` raises an error (see [When the Host gives no model](#when-the-host-gives-no-model)).

### Children and fanout arms

A section run through `call`, a spawned task, and every fanout arm start on a copy of their parent's current model, taken when the call or the spawn is dispatched. After the copy the two chains are independent: a `models.use` in the child changes only the child and the chains it starts, and it never reaches the parent or a sibling.

````markdown
---
name: children
description: Shows that a child's switch never reaches its parent.
runvil: 0
---

## Main

```lua
models.use('strong')
local report = call('Child')
return models.current().name .. '|' .. report
```

### Child

```lua
local start = models.current().name
models.use('fast')
return start .. '>' .. models.current().name
```
````

````text
strong|strong>fast
````

`Child` starts on `strong`, the model its caller was on, switches to `fast`, and returns. The caller is still on `strong` afterwards.

A task that waits in the queue under a concurrency limit (see [Concurrency slots](15-tasks.md#concurrency-slots)) starts on the model its parent had when the task was spawned, not on the model the parent has when the scheduler admits it. If a parent spawns two tasks on `strong` and then switches to `fast`, the task still queued runs on `strong`. All the arms of one fanout start on the same model as one another.

To run every arm of a fanout on a model you choose, without switching the parent, give `tasks.fanout` the model first. The form is covered with fanout in [the model-first form of `tasks.fanout`](14-fanout.md#running-arms-on-a-model).

## Selecting a role with `models.use`

After this section you can run one part of a prompt on a different role from the rest, switch to a saved handle, and read the error a rejected call raises.

````markdown
---
name: tagline
description: Drafts a tagline on the creative role and critiques it on the strong role.
runvil: 0
---

## Draft

```lua
models.use('creative')
```

Write a tagline for a bicycle shop.

```lua
var.tagline = models.infer(prose)
```

## Critique

```lua
models.use('strong')
```

Critique this tagline in one sentence: {{ var.tagline }}

```lua
return models.infer(prose)
```
````

`Draft` calls `models.use('creative')` in its first block, so the round in its last block runs on `creative`. `Critique` calls `models.use('strong')`, so its round runs on `strong`, and the critique becomes the run result. A section that makes no `models.use` call of its own runs on whatever model its chain is on (see [The current model](#the-current-model)).

`models.use(target, options?)` selects a model for the rest of the chain and returns a model handle for it. The `target` is one of the five role names, written as a string, or a model handle. The selection becomes the chain's current model, together with any options passed with it. After `models.use`, the rounds that follow in the chain that name no model, such as `models.infer(prose)`, run on the selected model, in the same section and in every later section of the chain until the next `models.use`. So after `models.use('strong')`, the next `models.infer('ping')` goes to the model that fills `strong`.

`models.use` takes effect as a plain statement, so you can write it alone in its own `lua` block before the prose and the round it governs, without keeping its return value, as `Draft` and `Critique` do. When you keep the handle, its `label` and `name` fields describe the role:

````markdown
---
name: label-check
description: Reads a selected role's label and name.
runvil: 0
---

## Main

```lua
local h = models.use('strong')
return h.label .. '|' .. h.name
```
````

````text
strong|strong
````

Roles are filled before a section's Lua runs, and `models.use` chooses among those already-filled roles by name.

### Changing the selection within a section

Calling `models.use` again partway through a section succeeds and replaces the earlier selection, because the selection is read when each round starts. In `tagline`, this one section could replace `Draft` and `Critique`:

````markdown
## Review

```lua
models.use('creative')
var.tagline = models.infer('Write a tagline for a bicycle shop.')
models.use('strong')
return models.infer('Critique this tagline in one sentence: ' .. var.tagline)
```
````

The first round runs on `creative` and the second on `strong`. `models.use` switches the model that later [`models.loop`](11-conversations.md#starting-a-conversation) calls send to in the same way, so a `models.loop` that follows `models.use('fast')` sends its requests to the model that fills `fast`.

Selecting the same role again is allowed, and a plain re-selection clears the earlier options. So `models.use('strong', { temperature = 0.5, max_tokens = 64 })` followed by `models.use('strong')` leaves the chain on `strong` with no `temperature` or `max_tokens` override.

### Selecting a handle

`models.use(handle)` selects the model a handle was made for, which is how you return to a model you saved or select one that is not a role. The handle's own frozen binding is used as it is, so `models.use(previous)` puts the chain back on the model and the options `previous` was made with (see [Reading the current model with `models.current`](#reading-the-current-model-with-modelscurrent)). A handle for a raw model id works the same way (see [Raw model ids and the Host-state snapshot](#raw-model-ids-and-the-host-state-snapshot)).

Options passed with a handle go on top of the handle's own settings. Each option that is set replaces the handle's value, and each option that is left out keeps it:

````markdown
## Main

```lua
local base = models.use('strong', { max_tokens = 128 })
local h = models.use(base, { temperature = 0.2 })
return tostring(h.temperature) .. '|' .. tostring(h.max_tokens)
```
````

````text
0.2|128
````

A role name starts from the role's own binding, which has no sampling options, so `models.use('strong')` clears the options an earlier call set. [How long sampling options last](#how-long-sampling-options-last) covers that rule.

### Which model a round uses

A round that names no model runs on the chain's current model, which is picked in a fixed order:

1. The chain's latest `models.use`.
2. Otherwise, the model the chain started on (see [Where a chain starts](#where-a-chain-starts)): the `default` role's model for the main walk, and a copy of the parent's current model for a `call` child, a spawned task, or a fanout arm.
3. Otherwise, no model at all, which happens only when the Host gave the run none.

A round that names a model with a handle, as in `models.infer(handle, prompt)`, runs on that handle and leaves the current model alone.

A `models.use` selection lasts until the next `models.use` in its chain. It reaches the sections that follow, including the options it set, so a later section's rounds send the same `temperature` and `max_tokens` until a plain `models.use(role)` replaces the selection. A `models.use` call in the preamble selects for the rest of the preamble and for every section the walk enters afterward, because the preamble and those sections are entries of the same chain. [When a prompt needs a role](#when-a-prompt-needs-a-role) covers the one case where a round has no model.

### The effective model

The chain's effective model is the selected role's or handle's binding with the `models.use` options applied on top. The options shape the chain's rounds and nothing else, and the role itself stays unchanged, so after `models.use('strong', { temperature = 0.5, max_tokens = 64 })`, `models.get('strong')` still reads `temperature` and `max_tokens` as `nil`.

`models.use` takes at most two arguments: a target and an optional options table such as `{ temperature = 0, max_tokens = 256 }`. Leaving the table out, or passing `nil`, sets no sampling overrides on a role name and keeps the handle's own settings on a handle. [Sampling options on `models.use`](#sampling-options-on-modelsuse) covers each option.

### Rejected calls

Every rejection from `models.use` and `models.get` is raised at the call site as an error value of kind `lua` whose message is the file and line of the call, then the rejection text, then the fixed prose for a Lua runtime failure, so [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch it and `tostring(err)` gives that message:

````markdown
---
name: ghost
description: Catches the selection of a name that is not a role.
runvil: 0
---

## Main

```lua
local ok, err = pcall(function() return models.use('reviewer') end)
return err.kind .. '|' .. tostring(err)
```
````

````text
lua|ghost.md:10: models.use label "reviewer" is not a model role: expected default, strong, small, fast or creative. The Lua code failed at runtime. Fix the code at this location.
````

`models.use` rejects a call in these ways, each with an error value of kind `lua`:

| Mistake | Message | What the message names |
|---|---|---|
| a first argument that is neither a role name nor a model handle | `models.use takes a model role name or a model handle, got {type}` | the Lua type of the value |
| a string that is not one of the five role names | `models.use label "{label}" is not a model role: expected default, strong, small, fast or creative` | the label and the five roles |
| a role the Host gave no model | `the caller supplied no model for the {role} role` | the role |
| more than two arguments | `models.use takes at most 2 arguments, got {n}` | the argument count |

In the first row, `{type}` is the Lua type name of the value given, where integers and floats both read `number`, so `models.use(5)` gives `models.use takes a model role name or a model handle, got number`, a userdata that is not a model handle reads `userdata`, and `json.null` reads `lightuserdata`. A call with a third argument after the options table gives `models.use takes at most 2 arguments, got 3`.

`models.use` checks its target first and its options second, and it records the selection only after every check passes. So an unknown role name is reported even when the options are also bad, and a rejected call leaves the chain's current model unchanged. [Sampling option errors](#sampling-option-errors) lists the option messages.

An uncaught `models.use` rejection ends the run with error code `lua_runtime`, like any other Lua runtime failure.

## Sampling options on `models.use`

You can set the sampling temperature and the reply token cap for the model rounds that follow a `models.use` call.

````markdown
---
name: slogan
description: Writes a bakery slogan at a low temperature
runvil: 0
---

## Write

```lua
models.use('default', { temperature = 0.3, max_tokens = 256 })
```

Write a slogan for a bakery.

```lua
return models.infer(prose)
```
````

The prologue selects the `default` role and passes `models.use` a second argument, an options table. The epilog's model round then asks the model for a temperature of 0.3 and caps its reply at 256 generated tokens, and the reply becomes the run result.

The `temperature` and `max_tokens` keys are the sampling options: the two settings a `models.use` call can attach to its selection. They come only from `models.use`, because a model role declared under `models:` has no sampling fields of its own (see [Declaring roles under `models:`](#declaring-roles-under-models)). A model round that has neither option sends no temperature and no token cap.

### Reading sampling options back

Replace the `Write` section of `slogan` with this one:

````markdown
## Write

```lua
local h = models.use('default', { temperature = 0.3, max_tokens = 256 })
return tostring(h.temperature) .. '|' .. tostring(h.max_tokens)
```
````

````text
0.3|256
````

`models.use` still selects `default`, and the model handle it returns reads the options back as `h.temperature` and `h.max_tokens` (see [Model handles](#model-handles)). The options apply both to model rounds on the selection and to rounds on that handle.

An option you leave out stays unset, and the handle reads it as `nil`:

````markdown
## Write

```lua
local h = models.use('default', { temperature = 0 })
return tostring(h.temperature == 0) .. '|' .. tostring(h.max_tokens)
```
````

````text
true|nil
````

Rounds on this selection ask the model for a temperature of 0 and send no token cap. `0` is an accepted temperature, and an integer works as well as a float, so `temperature = 0`, `temperature = 1`, and `temperature = 0.7` are all valid.

### How long sampling options last

````markdown
## Write

```lua
local careful = models.use('default', { temperature = 0 })
models.use('default')
return models.infer(careful, 'Write a slogan for a bakery.')
```
````

The second `models.use` call selects `default` again with no options, so the chain's selection now has none. The round on `careful` still asks for a temperature of 0, because a handle keeps the options it was made with.

Sampling options follow these rules:

- The options stay on the chain's selection. They apply to every following model round in the chain that names no model, in this section and in the sections after it, such as `models.infer(prose)` and each round of a `models.loop` conversation (see [Starting a conversation](11-conversations.md#starting-a-conversation)), until the next `models.use` call replaces the selection. A `call` child, a spawned task, and a fanout arm start on a copy of the parent's selection with its options, and their own `models.use` calls never reach the parent (see [Children and fanout arms](#children-and-fanout-arms)).
- A later plain `models.use(role)` with no options clears the options an earlier call set. The handle from `models.use('default')` reads `nil` for both `temperature` and `max_tokens`, and the chain's following rounds send neither.
- The handle a `models.use` call returns keeps that call's options for every round on it, `models.infer(h, prompt)` and each round of `models.loop(h)`, even after a later `models.use` changes the chain's selection.
- The options do not apply to a handle from `models.get`, because it describes the role's own settings, and they shape only the chain's effective model (see [The effective model](#the-effective-model)). A round on `models.get('strong')` sends no temperature and no token cap, only the role's thinking switch (see [The thinking switch](#the-thinking-switch)).

### Sampling option values

The options table names only `temperature` and `max_tokens`, written as string keys. The second argument of `models.use` is either that table or `nil`, and leaving it out or passing `nil` means no options.

| Option | Accepted values | Sent as | Read back as |
| --- | --- | --- | --- |
| `temperature` | A finite number from 0.0 to 2.0 inclusive, such as `0`, `0.7`, or `2.0` | The sampling temperature | `h.temperature`, a number |
| `max_tokens` | A whole number from 1 to 4294967295, written as a Lua integer or an integral float such as `1024.0` | The response token limit | `h.max_tokens`, a positive integer |

The `temperature` value goes out as the sampling temperature on the chain's model requests, so `temperature = 0` asks the model for a temperature of 0. The `max_tokens` value goes out as the response token limit, so `max_tokens = 256` caps the reply at 256 generated tokens.

### Sampling option errors

````markdown
## Write

```lua
local ok, err = pcall(models.use, 'default', var.options)
if not ok then
  return err.kind .. ': ' .. tostring(err)
end
return models.infer('Write a slogan for a bakery.')
```
````

When `var.options` holds a valid options table or `nil`, the block makes its round. When it holds a string, `pcall` catches an error value whose `kind` is `lua` and whose `tostring` is the message, and the block returns this, with the line of the `pcall(models.use, ...)` call:

````text
lua: {file}:{line}: models.use options must be a table, got string. The Lua code failed at runtime. Fix the code at this location.
````

Every sampling option error is an error value of kind `lua` raised by `models.use`, which `pcall` catches the same way (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

| Problem | Message |
| --- | --- |
| The second argument is neither a table nor `nil` | `models.use options must be a table, got {type}` |
| A key is not a string, including a positional entry | `models.use option names must be strings, got {type}` |
| A string key other than `temperature` or `max_tokens` | `models.use option "{other}" is unknown: expected temperature or max_tokens` |
| `temperature` is not a number | `models.use option temperature must be a number, got {actual}` |
| `temperature` is NaN or infinite | `models.use option temperature must be finite, got {actual}` |
| `temperature` is below 0.0 or above 2.0 | `models.use option temperature {value} is outside the supported range [0.0, 2.0]` |
| `max_tokens` is not a whole number from 1 to 4294967295 | `models.use option max_tokens must be an integer in [1, 4294967295], got {actual}` |

In these messages, `{type}` is the Lua type name of the value given, such as `string` or `integer`, and `{other}` is the unknown option name. When `temperature` is not a number, `{actual}` is the Lua type name of the value given, and in the finite message, `{actual}` is the value given, such as `NaN`. Finiteness is checked before the range, so an infinite temperature gets the finite message rather than the range message. In the range message, `{value}` prints the way a float prints, so a trailing `.0` is dropped: `temperature = 3` and `temperature = 3.0` both give `3`, and `temperature = 2.5` gives `2.5`. In the `max_tokens` message, `{actual}` is the number given, such as `0`, `-1`, or `1.5`, or the type name of a value that is not a number, such as `string`.

When an options table holds more than one bad entry, `models.use` reports the same one on every run, whatever order Lua's `pairs` visits the keys in. It checks keys that are not strings first, sorted by type name, so a `boolean` key is reported before an `integer` key. It then checks string names in bytewise order, so a bad `max_tokens` is reported before a bad `temperature`, and any other name is reported at its own place in that order.

## Model handles

You can inspect the model a handle stands for and run model rounds on it.

````markdown
---
name: inspect
description: Reads a model handle's fields
runvil: 0
models:
  strong:
    thinking: true
---

## Look

```lua
local h = models.get('strong')
return h.name .. '|' .. h.model_id .. '|' .. tostring(h.thinking)
```
````

On a Host whose map gives `strong` the model `gateway/claude-sonnet-4-6`, and that model can think, the run result is:

````text
strong|claude-sonnet-4-6|true
````

`models.get('strong')` returns the `strong` role's handle (see [Getting handles with `models.get`](#getting-handles-with-modelsget)), and the block reads three of its fields to inspect the role without running a model round.

A model handle is the value that `models.use`, `models.get`, and `models.current` all return. It is frozen Lua userdata whose read-only fields describe the role and model it stands for. It has no methods. You run a model round on a handle by passing it to `models.infer(model, prompt)` (see [Running a round with `models.infer`](#running-a-round-with-modelsinfer)), and you start a conversation on it with `models.loop(model, compactor?)` (see [Starting a conversation](11-conversations.md#starting-a-conversation)).

### The nine handle fields

A model handle has exactly nine fields, all read-only:

| Field | What it reads |
| --- | --- |
| `name` | The role name, as a string |
| `label` | The role name, the same string as `name` |
| `model_id` | The name part of the filling model's model id, such as `claude-sonnet-4-6` |
| `description` | The filling model's catalog description, which can be empty |
| `context` | The filling model's context window in tokens, a positive integer such as `8192` |
| `provider` | The provider id the model catalog records for the filling model, or `nil` |
| `thinking` | `true` for a role declared `thinking: true`, `false` for `thinking: false`, `nil` for neither |
| `temperature` | The `temperature` option from `models.use`, a number, or `nil` |
| `max_tokens` | The `max_tokens` option from `models.use`, a positive integer, or `nil` |

`name` and `label` always hold the same string: the role name the handle was obtained by. A handle from `models.get('strong')` or `models.use('strong')` reads `strong` for both. The role name stays the role name whatever model fills the role, so `models.get('strong').name` reads `strong` even when the `default` entry fills `strong`. On a handle for a raw model id, both fields hold that id (see [Raw model ids and the Host-state snapshot](#raw-model-ids-and-the-host-state-snapshot)).

`model_id` holds only the name part of the filling model's model id, such as `claude-sonnet-4-6` for a role filled by `gateway/claude-sonnet-4-6` (see [Model ids](#model-ids)), never the server namespace and never the role name. When the Host fills several roles with one model, as it does with a map that holds only a `default` entry, every role's handle reads the same `model_id`, and the `name` and `thinking` fields show how the roles differ (see [The role map](#the-role-map)).

`description` reads the filling model's catalog description from the Host, which can be empty. It is Host data, so a prompt cannot set it and its text differs from one Host to the next.

`context` reads the filling model's catalog context window in tokens, always a positive integer of at least 1, such as `8192`. It is the model's own window, not the role's `min_context`, so it reports what the model offers rather than what the role asked for.

`provider` reads the provider id the model catalog records for the filling model, as a string. With a model whose catalog entry records the provider `xai`, `tostring(h.provider)` reads `xai`. With a model whose entry records none, it reads `nil`.

`thinking` reads the role's thinking switch as a boolean, or `nil` when the role declared no `thinking` (see [The thinking switch](#the-thinking-switch)).

`temperature` and `max_tokens` read `nil` on every handle except one that `models.use` returned with that option set, such as `0.3` and `256` after `local h = models.use('default', { temperature = 0.3, max_tokens = 256 })`. A role has only `min_context` and `thinking`, and declares no sampling settings of its own.

### A handle's frozen binding

````markdown
---
name: second-opinion
description: Asks the strong role while the chain stays on the default role
runvil: 0
---

## Check

```lua
return models.infer(models.get('strong'), 'ping')
```
````

The chain starts on `default`, so a round in `Check` that names no model would run on `default`. The round on the `strong` handle still sends its request to the model that fills `strong`.

A model handle holds a frozen binding: the model and settings fixed when the handle was made. Its fields report the values fixed then and read the same every time during the run. Its requests go to the model it was made for, so a handle for a `strong` role filled by `gateway/claude-sonnet-4-6` sends its requests to that model, naming it `claude-sonnet-4-6`. Later changes to the chain's current model never change which model a handle uses, and repeated rounds on one handle use the same model, as when two `models.loop(other)` calls both send their requests to the model `other` was made for.

### Read-only handle fields

Model handle fields can only be read. Reading any name other than the nine fields raises a Lua error, so `pcall` around such a read returns `false`. That includes `infer` and `loop`: a handle has no methods.

A model handle is Lua userdata, not a table, so `type(h)` returns `'userdata'`. Only the userdata that `models.use`, `models.get`, or `models.current` returns counts as a model handle. A plain table that copies a handle's fields is not one, and `models.infer` and `models.use` refuse it (see [Infer call-shape errors](#infer-call-shape-errors) and [Rejected calls](#rejected-calls)).

## Getting handles with `models.get`

You can get the model handle for any of the five roles, or with a Host-state snapshot for a gateway model by its name, without changing the chain's current model.

````markdown
---
name: review
description: Asks the strong role without switching to it
runvil: 0
---

## Ask

```lua
return models.infer(models.get('strong'), 'ping')
```
````

`models.get(name)` returns the model handle for one of the five roles, so you reach the model that fills a role by passing its role name. Here the handle runs one round on the `strong` role's model, and the reply is the run result. The call is nested without storing the handle, a common way to run one round on a role.

`Ask` has no `models.use` call, so the chain stays on `default`. The round still runs on `strong`, because a round given a handle runs on that handle's frozen binding rather than on the chain's current model.

### `models.get` keeps the selection

````markdown
---
name: bread-review
description: Drafts on the default role and checks on the strong role
runvil: 0
---

## Draft

Summarize why bread rises.

```lua
local strong = models.get('strong')
local summary = models.infer(prose)
return models.infer(strong, 'Name the weakest claim in this summary: ' .. summary)
```
````

`models.get` returns a role's handle without selecting it. It does not change the chain's current model, so `models.infer(prose)` after it still goes to the current model, here `default`, and only the round given the `strong` handle uses the `strong` role.

The handle describes the role's own settings, whatever the chain has selected, so it never has the sampling options of a `models.use` call (see [The effective model](#the-effective-model)).

`models.get` returns the same kind of model handle as `models.use` and `models.current`, a Lua userdata, so `type(models.get('fast'))` is `'userdata'`. You can store it and run several rounds on it, as in `local other = models.get('fast')` followed by `models.loop(other)`. Model roles are reachable through `models.get` from a walked section, from a section run through `call` (see [Calling a section](08-jump-and-call.md#calling-a-section)), and from fanout arms (see [What an arm sees](14-fanout.md#what-an-arm-sees)) alike.

### Catching an unknown role label

````markdown
---
name: ask-role
description: Asks whichever role the text argument names
runvil: 0
---

## Ask

Say hello.

```lua
local ok, h = pcall(models.get, args.text)
if not ok then
  return 'unknown role: ' .. tostring(h)
end
return models.infer(h, prose)
```
````

The role name arrives as the run's `text` argument, which the block reads as `args.text` (see [Reading arguments from `args`](06-arguments.md#reading-arguments-from-args)). In a run without a Host-state snapshot, the arguments `{"text":"reviewer"}` name none of the five roles, `pcall` catches the failure, and the block falls back instead of failing:

````text
unknown role: ask-role.md:12: models.get label "reviewer" is not a model role: expected default, strong, small, fast or creative. The Lua code failed at runtime. Fix the code at this location.
````

With the arguments `{"text":"strong"}`, the block runs one round on the `strong` role.

A failing `models.get` call caught by `pcall` or `xpcall` yields an error value whose `kind` is `lua`, whose `message` is the file and line of the call, then the rejection text, then the fixed prose for a Lua runtime failure, and whose `tostring` is that message. An `xpcall` handler sees the same error value.

### Raw model ids and the Host-state snapshot

````markdown
---
name: raw-id
description: Reads a handle for a raw model id
runvil: 0
---

## Look

```lua
local h = models.get('qwen/qwen3-8b')
return h.name .. '|' .. h.model_id .. '|' .. h.context
```
````

When the Host runs this prompt with a Host-state snapshot, the run result is:

````text
qwen/qwen3-8b|qwen/qwen3-8b|8192
````

A Host-state snapshot is the application state a Host can hand a run, the value that `ui()` returns (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)). With one, `models.get` treats a name that is not one of the five roles as a raw model id: the name a gateway model is listed under, passed in place of a role name. It returns a handle for the model id made of the server namespace `gateway` and that name, so the handle above is for `gateway/qwen/qwen3-8b` and its `model_id` reads the name as passed. Pass the name alone, because a full model id such as `gateway/fast` is read as a name too, giving the model id `gateway/gateway/fast`, and the handle's requests then name the model `gateway/fast`.

A prompt can then reach a gateway model by its name with no further setup, usually the one the Host reports in `ui().selected_model`:

````markdown
## Reply

Say hello.

```lua
return models.infer(models.get(ui().selected_model), prose)
```
````

Raw model ids follow these rules:

- One of the five role names always resolves as the role, even when raw model ids are allowed.
- A raw model id is not a role name, so it may contain `/`, `.`, and `:`, and only the model id's own rules apply (see [Model ids](#model-ids)). The id must be non-empty and free of control characters, or `models.get` fails with `models.get model id "{id}" is invalid: invalid model id: name {reason}`, where `{reason}` is `must not be empty` or `must not contain a control character`.
- A handle for a raw model id has `name`, `label`, `model_id`, and `description` all equal to the id, and `nil` for `provider`, `temperature`, `max_tokens`, and `thinking`. Its `context` is a fixed 8192 tokens, because the run has no catalog details for the id.
- A raw-id handle works anywhere a handle does, in `models.use`, `models.infer(model, prompt)`, `models.loop(model, compactor?)`, and the model-first form of `tasks.fanout`, and it works on a Host that gave the run no model, because the handle holds its own binding.
- The run's roles are filled from the Host's role map, so a raw model id through `models.get` is the only way a prompt reaches a model outside that map.
- Only `models.get` has the raw model id fallback. `models.use` takes only the five role names and handles, so `models.use('qwen/qwen3-8b')` fails with `models.use label "qwen/qwen3-8b" is not a model role: expected default, strong, small, fast or creative`. To select a raw model id, pass its handle: `models.use(models.get('qwen/qwen3-8b'))`.
- Without a Host-state snapshot there is no `ui` global, and `models.get` accepts only the five role names, so any other name is an error rather than a raw model id.

### `models.get` errors and check order

Without a Host-state snapshot, a name outside the five roles fails `models.get` with an error value of kind `lua` whose message holds the rejection `models.get label "{label}" is not a model role: expected default, strong, small, fast or creative`, where `{label}` is the name passed. The rejection reads the same wherever the call runs: on the main walk, inside a fanout arm, or in a run with no model. With a snapshot, such a name is tried as a raw model id instead, and fails only when the id is invalid, with the `is invalid` message above. A role the Host gave no model fails with `the caller supplied no model for the {role} role`.

The two functions check a name in different orders:

1. `models.use` checks the type of its target, then whether a string names one of the five roles, then whether the Host gave that role a model, and only then its options.
2. `models.get` looks for one of the five role names first. When the Host gave a Host-state snapshot, it then treats any other name as a raw model id. Otherwise it reports that the name is not a role.

## Running a round with `models.infer`

You can send one prompt to a model in a single round and use the reply as a Lua string.

````markdown
---
name: bakery
description: Asks the model for a bakery slogan
runvil: 0
---

## Write

Write a slogan for a bakery.

```lua
return models.infer(prose)
```
````

`return models.infer(prose)` sends the section's prose, the rendered Markdown above the block (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), to the model in one round and makes the reply the section's result. In this one-section prompt, the run result is whatever the model replies. The call names no model, so it runs on the chain's current model, which here is the `default` role's.

Each `models.infer` call is one infer round: a model round on a fresh conversation whose only message is the prompt, sent as a single message with the `user` message role. Nothing from earlier rounds or from the rest of the section reaches the model. An infer round never reads or changes the section entry's `messages` list (see [Starting a conversation](11-conversations.md#starting-a-conversation)), sends no tools, and works inside a loop's tool handlers and compactors, where a loop cannot start.

### Which model an infer round uses

`models.infer(prompt)` runs on the chain's current model, which it looks up when the call runs, in the order [Which model a round uses](#which-model-a-round-uses) gives. In `bakery`, that is `default`, the model the chain starts on.

`models.infer(model, prompt)` sends the prompt to the model handle's own model instead, using the frozen binding the handle got when you obtained it, so the request has that handle's role name and model. It does not change the chain's current model:

````markdown
## Write

```lua
return models.infer(models.get('fast'), 'say hello')
```
````

The handle form returns the reply as a plain Lua string, just as the one-argument form does. It runs on the handle's frozen binding even when the chain has switched to another role, so it never needs the chain to have a model. A handle for a raw model id therefore runs even on a Host that gave the run no model, and a handle for a role needs the Host to have given that role a model in the first place (see [When the Host gives no model](#when-the-host-gives-no-model)).

### Infer prompts and replies

`models.infer` has two forms. `models.infer(prompt)` takes the prompt string and runs on the chain's current model, and `models.infer(model, prompt)` takes a model handle first and runs on that handle. Any other number of arguments is refused with `models.infer takes (prompt) or (model, prompt)` (see [Infer call-shape errors](#infer-call-shape-errors)). The prompt can be any Lua string: a literal such as `models.infer('hi')`, text read from the store such as `models.infer(store.read('note.md'))` (see [Writing and reading files](09-the-store.md#writing-and-reading-files)), or a string built from earlier results:

````lua
local inner = call('../Inner')
return models.infer('outer saw: ' .. inner)
````

Here `inner` holds the text that `call('../Inner')` returned, and the model receives `outer saw: ` followed by it.

The reply comes back as a plain Lua string, which the block can concatenate with `..`, compare, store in `var`, pass to `tostring`, or `return` as the section's or the run's result. Replace the `Write` section of `bakery` with this one:

````markdown
## Write

Write a slogan for a bakery.

```lua
var.draft = models.infer(prose)
return models.infer('Shorten this slogan: ' .. var.draft)
```
````

You can call `models.infer` more than once in a section. Each call is its own model round, issued only after the previous one is answered, and each `models.infer` call makes exactly one completion request to the model, so two calls make two requests, whether they sit in one section or in two.

`models.infer` is a suspending call, a Lua function on the `models` table. The block pauses while the round runs and resumes at its next statement once the reply arrives, so the section goes on only after the round is answered.

`models.infer` works inside `pcall`. The block still pauses for the round, and when the round succeeds, `pcall` returns `true` and the reply text:

````lua
local ok, text = pcall(function() return models.infer('hi') end)
````

### What an infer round leaves out

An infer round sends the prompt and returns the reply, and nothing more:

- Each `models.infer` call sends a single round that offers the model no tools, even when the prompt declares tools, so no tool call can happen during the round, and a reply that comes back with tool calls anyway is refused (see [Errors from the model backend](#errors-from-the-model-backend)). When a Lua block needs a model round that can use tools, it runs a section with `call` instead (see [Calling a section](08-jump-and-call.md#calling-a-section)).
- The reply comes back whole as the call's return value, and none of it appears in the run's live output while the model is still writing.
- Any reasoning text the model backend supplies never becomes part of the returned value, which holds only the reply text.
- A round cut off at the generation cap, such as one limited by a `max_tokens` option, still returns the text the model produced up to that point.
- The round sets no global and leaves `sys` untouched, so `sys` reads the same before and after it (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)).

### Infer rounds in tasks and arms

A section running as a spawned task (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)) can call `models.infer` to make its own model request, such as `models.infer('child work')` in a `Child` section, and return the reply as the task's result. A fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)) can call `models.infer` too, such as `models.infer(prose)`, and each call makes exactly one model round whose reply the arm can return. A task or an arm starts on a copy of its parent's current model, so a `models.infer(prompt)` call there runs on that model until the task or arm calls `models.use` itself (see [Children and fanout arms](#children-and-fanout-arms)).

## Infer errors and the missing-model error

You can catch a failed infer call, branch on its error kind, and recognize the missing-model error.

````markdown
---
name: careful
description: Falls back when a model round fails
runvil: 0
---

## Write

Write a slogan for a bakery.

```lua
local ok, result = pcall(models.infer, prose)
if not ok then
  return 'skipped (' .. result.kind .. ')'
end
return result
```
````

When the round succeeds, `pcall` returns `true` and the reply, and the reply is the run result. When the round fails, `pcall` returns `false` and an error value, and the block returns `skipped (` followed by the error's kind and `)`.

A failed `models.infer` call raises an error value that you catch with `pcall` and branch on by its `kind` (see [Error values](05-lua-environment.md#error-values)). The errors fall into three groups: call-shape errors, errors from the model backend, and the missing-model error.

### Infer call-shape errors

Each infer form checks its arguments before any model request is made, and a bad argument raises an error value of kind `lua` at the call site, where `pcall` catches it. The checks run in this order: whether the first argument is a model handle, then the argument count for that shape, where a trailing `nil` counts as an argument, then the prompt.

| Condition | Message |
| --- | --- |
| A model handle first and more than two arguments, or anything else first and more than one argument, a trailing `nil` included | `models.infer takes (prompt) or (model, prompt)` |
| The prompt is missing or is not a string | `prompt must be a string, got {type}` |
| The prompt is not valid UTF-8 | `prompt must be a valid UTF-8 string` |

So an extra argument after the prompt, such as an options table, gets the first message, and so does a table that copies a handle's fields standing where the handle goes, because a table is not a model handle. In the prompt message, `{type}` is the Lua type name of the value given, so a missing prompt gives `prompt must be a string, got nil`, an integer gives `got integer`, and a float gives `got number`.

`models.infer` takes the prompt alone, or a model handle and the prompt, so sampling options for a round go on `models.use` (see [Sampling options on `models.use`](#sampling-options-on-modelsuse)). Any other argument raises `models.infer takes (prompt) or (model, prompt)` at the call site rather than being ignored.

`models.loop` takes `(compactor?)` or `(model, compactor?)` and checks its arguments the same way, counting a trailing `nil` (see [Starting a conversation](11-conversations.md#starting-a-conversation)).

### Errors from the model backend

An infer call offers the model no tools, so it never dispatches tools. A model that replies to an infer round with tool calls fails the call with an error value of kind `model_failure` whose message is the file and line of the `models.infer` call, then this text:

````text
the model backend sent a reply that could not be understood: the round sent tool calls, but no tools were advertised
````

The text is the same with or without a model handle, and it never names `models.infer`. Left uncaught, the refusal fails the run with error code `malformed_response` (see [The run result](04-how-a-prompt-runs.md#the-run-result)), and the failure counts as retryable, since a reply that broke protocol may come back well formed (see [Retryable failures](16-limits-and-errors.md#retryable-failures)).

An infer round that fails at the model, such as a transport or backend failure, raises the round's own error at the call site. `pcall` sees it with kind `model_failure`, or with kind `empty_model_reply` for an empty reply. Left uncaught, it fails the run with that error's own code, one of the model codes, such as `transport` for a transport failure or `empty_reply` for an empty reply.

### When a section has no model

The missing-model error is the error a model round raises when the chain it runs in has no model. A chain starts on the `default` role's model, so this happens only when the Host gave the run no model (see [When the Host gives no model](#when-the-host-gives-no-model)), which is why the `unselected` prompt in [When a prompt needs a role](#when-a-prompt-needs-a-role) fails with `model binding required for section Only` on such a Host. Only `models.infer` and `models.loop` raise it, and only when they name no model handle. Its own text is `model binding required for section {section}`, where `{section}` is the section name, such as `Only` or `Worker`. The preamble's name is the empty string, so a round there leaves `{section}` empty. Left uncaught, it fails the run with error code `model_required` and that text as the message, ending with the line of the `models.infer` or `models.loop` call. Raised in a fanout arm, it fails both the fanout and the run.

The missing-model error reaches the call as its answer, so `pcall` catches it as an error value of kind `model_required` and a block can fall back instead of failing. The caught value's message is the call's file and line, then the fixed prose ``The round has no model. Select a model before `models.infer` or `models.loop`, or bind a default model.``, so it does not name the section. On a Host that gave the run no model, `careful` returns:

````text
skipped (model_required)
````

`models.loop` with no model handle on such a Host fails with the same missing-model error naming the section, which `pcall` also catches as kind `model_required`.

Only an explicit model call such as `models.infer(prose)` can raise the missing-model error, so prose that no block sends to a model needs no model at all (see [When a prompt needs a role](#when-a-prompt-needs-a-role)). `models.use`, `models.get`, and `models.current` do not raise it. On a Host with no model they fail with ordinary Lua errors of kind `lua`, as [When the Host gives no model](#when-the-host-gives-no-model) shows.

`models.infer(model, prompt)` runs on the handle's own model, so it works on a Host that gave no model when the handle is a raw-id handle such as `models.get('qwen/qwen3-8b')`, where `models.infer(prompt)` would fail with the missing-model error.

## What an infer round reports

You can follow the events a run reports for each infer round, in order, and know what each one holds.

````markdown
---
name: greet-events
description: Makes one infer round on the default role
runvil: 0
---

## Greet

```lua
return models.infer('say hello')
```
````

The block in `Greet` reports these events, in this order:

1. `lua_chunk_started` for the block.
2. `model_turn_completed` when the round's model call finishes.
3. `assistant_reply` with the reply text, `origin` set to `infer`, and `turn` set to `1`, since this is the run's only model round.
4. `lua_chunk_succeeded` for the block.

Each infer call reports a `model_turn_completed` event inside the Lua block that made the call, between that block's `lua_chunk_started` and `lua_chunk_succeeded` events (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)). Each `models.infer` call shows up in the run's events as exactly one completed model round, because an infer call drives exactly one model round trip.

### Infer event order

A successful infer round reports the same round events, in the same order, as any other model round in the calling section, except that its reply is tagged as an infer reply instead of a chat reply. When the caller turns debug capture on, a `request` event and a `response` event come first, before `model_turn_completed`, and they appear only then. Apart from those two, one infer round reports:

1. `model_turn_completed`.
2. One `model_metadata_degraded` event per problem the Harness finds, only when the backend's response has a malformed `usage`, `timings`, or `metrics` section, or names no model. Each event holds the round's `turn` and the Harness's message naming the problem, and the round itself still succeeds.
3. `thinking`, only when the backend supplies reasoning.
4. `model_turn_truncated`, only when the reply stops at the length limit with finish reason `length`. The reply text still comes back to Lua.
5. `assistant_reply`, the last event of the round.

When the backend's response names no model, the Harness's `model_metadata_degraded` message reads ``completion response named no string `model`; labeled with the requested model``, and the Harness labels the round with the requested model.

When the backend supplies no reasoning, or empty reasoning, an infer round reports no `thinking` event. When it supplies reasoning, the round reports it as one `thinking` event after `model_turn_completed` and before the infer `assistant_reply`, the same way other model rounds report reasoning. The `thinking` event holds the reasoning text in `text`, the round number in `turn`, the round id in `round`, and the model name in `model`.

### The infer `assistant_reply` event

An infer round that returns text reports exactly one `assistant_reply` event, stamped with the round's id, and it never reports a chat-origin `assistant_reply`, whether or not the backend supplied reasoning.

| Field | What it holds |
| --- | --- |
| `text` | The reply text |
| `turn` | The round's number on the run's round counter |
| `round` | The round's id |
| `origin` | `infer` |
| `model` | The name of the model the round was sent to |
| `finish_reason` | The provider's finish reason, or `null` when the provider sent none |
| `metrics` | The call's metrics, or `null` when none were reported |

The Engine adds nothing of its own, so every figure comes from one of four sources: the backend's `usage`, llama.cpp's `timings`, vLLM's `metrics`, and the client's own `client` timing.

Each infer round counts as one model round on the run's round counter, the number its events report in `turn`. Inside a spawned task (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)), it counts on the task's own counter instead, so a task's `turn` numbers, and its `tasks.status` `turns`, count only its own rounds. No cap counts infer rounds.

An infer round's `thinking` event, `assistant_reply` event, and answer record name the model the round was actually sent to. That can be a substitute the Host picked in place of the model that fills the role. The Harness labels each completion with the model the request named, whatever name the backend's response gave, and the Engine reports the name the completion carries.

### Infer failures and cancellation

When the model request for an infer round fails, the run reports a `model_turn_failed` event for the calling section, and the infer call fails with that error (see [Errors from the model backend](#errors-from-the-model-backend)).

When the run is canceled while an infer round is still waiting for its answer, the run ends with code `canceled` and the round's answer is dropped. No `model_turn_failed` event is reported, because cancellation never reports a model failure.
