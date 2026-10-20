# Models

A prompt never names a concrete model. Its frontmatter declares model roles instead, each one a model the prompt needs under a label of its own, and before the run starts, prepare binds every role to the Host's current model. Your Lua then runs model rounds on those roles, through `models.infer` or through a model handle. Because a prompt asks for roles rather than models, the same file runs on whatever model the Host selects, and the requirements you write into a role are checked before any of your Lua runs.

## Model roles instead of models

After this section you can declare a model role, send prose to the model bound to it, and tell when a prompt needs a role at all.

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
models.default('writer')
```

## Greet

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

The frontmatter declares one model role, `writer`, under the `models:` key. The `lua` fence in the H1 body selects that role for the whole prompt with `models.default('writer')`, and the `## Greet` section sends its prose to the model with `return models.infer(prose)`, so the model's reply to `Say "Hello, world!"` becomes the run result. Every model round you send this way takes the same three steps: declare a role under `models:`, select it, and call `models.infer(prose)` in a section. The bundled `hello.md` prompt is built exactly this way, with `writer: {}` in its frontmatter, `models.default` in its H1 body, and `return models.infer(prose)` at the end of its `## Greet` section.

### What a model role is

A model role is a need a prompt declares under `models:` in place of a concrete model. Each role stands for one model the prompt needs, and [prepare](04-how-a-prompt-runs.md#preparing-the-run) binds every role to the [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)'s current model before the run starts. The prompt never says which model that is, so the same prompt runs on whatever model the Host selects. With one current model, two roles such as `scout` and `sprinter` both bind to that same model.

You declare each model the prompt needs as a role label of its own, such as `writer`, under `models:`. A role label is the key that names a model role, and it follows the prompt-local name grammar (see [The name grammar for role labels, arg names, and local tool aliases](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-local-tool-aliases)). A role declaration accepts only three fields, `keywords`, `min_context`, and `description`, and none of them names a concrete model. [Declaring roles under `models:`](#declaring-roles-under-models) covers each field.

Prepare never reads the Host's model catalog. It hands every role the Host's one current model, so a prompt's roles never draw on the rest of the Host's models, and the Host's list of models stays with the Host. [How prepare binds and checks roles](#how-prepare-binds-and-checks-roles) explains the model catalog and the current model.

### The `models` global

The `models` global is the table your Lua uses to work with the run's model roles. Every section has it, the H1 section included, and it selects among the run's roles by label. It is reserved as an [Engine global](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals), and you can reach it from the Lua of every section, from the blocks of the H1 body, and from a [library](03-blocks-and-prose.md#sharing-code-with-require).

The table holds five functions. `models.use` and `models.default` select a role, `models.get` returns a model handle, `models.infer` runs one model round, and `models.loop` is taught with conversations:

| Function | Where it is taught |
|---|---|
| `models.use` | [Selecting a role per section with `models.use`](#selecting-a-role-per-section-with-modelsuse) |
| `models.default` | [The prompt-wide default with `models.default`](#the-prompt-wide-default-with-modelsdefault) |
| `models.get` | [Getting handles with `models.get`](#getting-handles-with-modelsget) |
| `models.infer` | [Running a round with `models.infer`](#running-a-round-with-modelsinfer) |
| `models.loop` | [Starting a conversation](11-conversations.md#starting-a-conversation) |

The `models` table is in place with the other Engine globals before any block runs, so any block, its top-level statements included, calls it directly with no `require` and no setup line. Both blocks in `hello` use it that way.

Roles come only from the `models:` frontmatter, and Lua never creates or binds one, so the `models` table holds no function for binding a role. `models.use` and `models.default` only choose among the declared roles by label, and when no declared role matches the label you pass, they fail with an error saying the label `is not a bound model role`. Prepare binds the run's roles before any Lua runs, and every section, the H1 pass included, reads one shared set of those roles rather than its own copy, so the same roles work in every section.

### When a prompt needs a role

A round that names no model, such as `models.infer(prose)`, needs a selected role. The role comes either from `models.use` in the section or from `models.default` for the whole prompt. This prompt declares `writer` but selects it neither way:

````markdown
---
name: unselected
description: Sends prose to a model with no role selected.
runvil: 0
models:
  writer: {}
---

# Unselected

## Only

Say "Hello, world!"

```lua
return models.infer(prose)
```
````

The run fails with this message, which names the section whose round had no role:

````text
model binding required for section Only
````

This is the missing-model error, covered in [Infer errors and the missing-model error](#infer-errors-and-the-missing-model-error). Adding `models.default('writer')` in a `lua` fence of the H1 body, as `hello` does, gives the round a role.

A prompt that runs no model round needs no `models:` declaration at all. The bundled `echo.md` prompt has no `models:` key in its frontmatter and never touches `models` in its body. Prose alone never needs a model either, because only an explicit `models.infer` of it does. A prompt whose blocks run no model round runs to completion even when the Host provides no model at all:

````markdown
---
name: word
description: Returns its own prose without a model.
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

The block renders its prose and returns it, and no model takes part, so this run gives the same result on a Host with no model.

## Declaring roles under `models:`

After this section you can declare any number of roles and give each one keywords, a context minimum, and a description.

````markdown
---
name: research
description: Declares an analyst role and a triage role.
runvil: 0
models:
  analyst:
    keywords: [frontier, thinking]
    min_context: 200000
    description: deep reasoning
  triage:
    keywords: [fast, small]
    description: quick triage of search results
---

# Research

## Main

```lua
return 'done'
```
````

On a Host whose current model can think and has a context window of at least 200000 tokens, the run returns:

````text
done
````

The `models:` key holds a map from each role label, which is local to the prompt, to that role's declaration, with one entry per role. Here it declares two roles, `analyst` and `triage`. A role accepts exactly three keys, `keywords`, `min_context`, and `description`, and each one is optional. One role can combine all three, as `analyst` does, and one `models:` map can declare any number of roles, so `triage` sits beside it with only `keywords` and `description`.

| Key | What it takes | When you leave it out |
|---|---|---|
| `keywords` | a list drawn from the seven keywords, kept in the order written | no keywords |
| `min_context` | a whole number of tokens from 1 to 4294967295 | no context minimum |
| `description` | free prose, kept exactly as written | the bound model's own description |

### Empty roles and no roles

A role written as an empty map, such as `writer: {}` or `researcher: {}`, is valid. It declares no keywords, no context minimum, and no description, so it places no requirement on the model bound to it, and it still counts as a declared role. `models:` with `writer: {}` beneath it, as in `hello`, declares one model role with the label `writer`.

The `models` key itself is optional. Leaving `models:` out of the frontmatter declares no model roles, and the prompt's role declaration is empty.

### Role labels

Each key under `models:` is a role label. A label starts with an ASCII letter followed by up to 63 ASCII letters, digits, `_`, or `-`, the grammar `[A-Za-z][A-Za-z0-9_-]{0,63}` that a local tool alias also follows, such as `analyst`. Lua code refers to the role by that exact label, as in `models.use('analyst')`.

A label is local to the prompt that declares it. The same label in another prompt names an unrelated role, and a run binds models only for the roles that the prompt being run declares.

A label is never a Lua global, so any label the grammar allows works, including a reserved name such as `models`, `string`, or `end`. The only global installed for roles is `models`, and Lua code looks a label up through `models.use`, `models.default`, and `models.get`. [Role labels are never globals](#role-labels-are-never-globals) shows what this means for a label that shares a global's name.

Two label mistakes fail the parse with a `Frontmatter` parse error whose message names the label. A label outside the grammar, such as one starting with a digit, gives this message, which also names the grammar:

````text
invalid model role label `{label}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

Each label appears once under `models:`, and a label written twice gives:

````text
duplicate model role label `{label}`: contract map keys must be unique
````

### The `keywords` key

`keywords` takes a YAML list of keywords from a fixed vocabulary, each describing the model the role wants, for example `keywords: [frontier, thinking]`, `keywords: [fast]`, or `keywords: [no-thinking]`. [Keywords and thinking](#keywords-and-thinking) lists all seven and what each one asks of the model.

`keywords` is always a YAML list, even for a single keyword, as in `keywords: [thinking]`, and any other shape in its place fails the parse. Leaving out `keywords` gives the role an empty keyword list, so no keyword requirement applies to it.

The keywords are kept in the order you write them, so `keywords: [frontier, thinking]` reads back as `frontier` then `thinking`, and the model bound to the role lists its capabilities in that same order.

### The `min_context` key

`min_context` sets the smallest context window, in tokens, that the model bound to the role must provide, such as `min_context: 200000` or `min_context: 40000`. A model with a smaller window is refused, so a 32000-token model against `min_context: 200000` is refused with a line saying the role `requires a context of at least 200000 tokens`.

`min_context` takes a whole number of tokens from 1 to 4294967295, and any other value fails the parse with a `Frontmatter` error. The smallest minimum you can write is 1, so a role that wants no minimum leaves the key out. Leaving out `min_context`, or setting it to null, declares no context minimum, and the role sets no floor on the model's context window.

Parsing a `models` declaration checks only its shape. The checks against a real model happen later, when the run is prepared and each role is bound to a concrete model, and only then is the role's fit against its hard keywords and context minimum checked. So a prompt declaring `analyst` with `min_context: 200000` parses cleanly, and only a run under a 32000-token model refuses it, with `role 'analyst': requires a context of at least 200000 tokens` (see [The context check](#the-context-check)).

### The `description` key

`description` takes free prose that records what the role is for, such as `Deep analysis` or `A model suited for careful analysis`, and the text is kept exactly as written. A role's `description` is separate from the prompt's top-level `description:` key. A role that leaves it out, or sets it to null, takes the bound model's own catalog description instead.

### Flow style

A role declaration can use YAML flow style (see [Writing YAML in the frontmatter](02-file-structure.md#writing-yaml-in-the-frontmatter)) to fit on one line, and a whole `models:` map with several roles can be written the same way. This `models:` line, part of a prompt's frontmatter, declares an `analyst` role with all three keys and an empty `spare` role:

````markdown
models: { analyst: { keywords: [no-thinking, creative, chat], min_context: 32000, description: deep reasoning }, spare: {} }
````

### Mistakes inside `models:`

A role accepts no keys besides `keywords`, `min_context`, and `description`. Any other key, including a misspelling of one of the three, fails the parse with a `Frontmatter` error carrying a message from the YAML reader, so a typo is reported instead of being silently ignored.

A mistake inside `models:` fails the parse with a `Frontmatter` parse error that reports the line and column of the mistake in the whole file, counting the opening `---` line. So a rejection inside a contract key can report line 5, column 5, a position in the file you wrote rather than in the YAML alone. [Where an error points](16-limits-and-errors.md#where-an-error-points) covers error locations.

## Keywords and thinking

After this section you can choose keywords for a role, tell which ones prepare checks, and predict whether the role's rounds ask the model to think.

````markdown
---
name: boiling-point
description: Answers one question with thinking turned off.
runvil: 0
models:
  responder:
    keywords: [no-thinking, fast]
---

# Boiling point

```lua
models.default('responder')
```

## Ask

At what temperature does water boil at sea level?

```lua
return models.infer(prose)
```
````

The `responder` role lists two keywords. `no-thinking` is a hard keyword, so prepare checks that the current model can run with thinking turned off, and every round under the role asks for thinking off, including the round in `## Ask`, which reaches `responder` only through the prompt-wide default. `fast` is a soft keyword, so it records that you want a fast model and changes nothing about the round.

### The seven keywords

A role's keywords come from a closed, fixed vocabulary of seven: `thinking`, `no-thinking`, `frontier`, `fast`, `small`, `creative`, and `chat`. They split into two hard keywords, `thinking` and `no-thinking`, and five soft keywords, `frontier`, `fast`, `small`, `creative`, and `chat`. A hard keyword is a requirement that prepare checks against the bound model, and a soft keyword records what you intend and is never checked.

| Keyword | Kind | What it asks of the model |
|---|---|---|
| `thinking` | hard | supports extended thinking, and thinking is turned on |
| `no-thinking` | hard | can run with thinking off, and thinking is turned off |
| `frontier` | soft | a frontier-capability model |
| `fast` | soft | a fast model |
| `small` | soft | a small model |
| `creative` | soft | a creative model |
| `chat` | soft | a chat-tuned model |

No keyword is set by default, so a role has a keyword only when its `keywords` list names it, such as `keywords: [no-thinking, creative, chat]`, and a role such as `spare: {}` has none. [The `keywords` key](#the-keywords-key) covers the list's shape and order.

Each keyword has exactly one spelling, in lowercase kebab-case, with no aliases, so the thinking-off keyword is `no-thinking` with a hyphen, and the requirements notice spells it back the same way. Prompt parsing checks every entry under `models.{label}.keywords` against the seven, so a misspelled or invented keyword, such as `multimodal`, fails the parse with a `Frontmatter` parse error instead of being silently ignored. Its message comes from the YAML reader and carries the YAML line and column.

Parse does not reject repeated or conflicting keywords, such as `thinking` together with `no-thinking`, because parse checks only the shape of the frontmatter. [Both hard keywords on one role](#both-hard-keywords-on-one-role) explains what such a role does.

### Hard keywords

The `thinking` keyword is a hard requirement that the model bound to the role supports extended thinking, so only a model whose thinking mode is `never` fails it. You list it in a role's `keywords` to turn thinking on, for example `keywords: [frontier, thinking]`.

The `no-thinking` keyword is a hard requirement that the model bound to the role can run with thinking turned off, so only a model whose thinking mode is `always` fails it. You list it to turn thinking off, for example first in `keywords: [no-thinking, creative, chat]`.

A role with `keywords: [frontier, thinking]` declares one hard keyword, so prepare checks the model bound to it only for its context window and its thinking mode.

### Soft keywords

Soft keywords only record what you intend, each marking the kind of model the table above names. Nothing checks them against the model before the run, so they never steer which model a role gets, never make a role unmet, and never cause prepare to refuse a run, even on a model whose thinking mode is `never`. None of them changes anything about a round, so in `boiling-point` above, `fast` leaves the round exactly as `no-thinking` sets it.

You can combine soft keywords with each other and with a hard keyword, as in `keywords: [frontier, thinking]`, `triage: { keywords: [fast, small] }`, and `keywords: [no-thinking, creative, chat]`. So `scout: { keywords: [frontier, fast] }` and `sprinter: { keywords: [small, creative, chat] }` both bind to the same single current model and both pass prepare on any current model, a role such as `writer: { keywords: [fast] }` never appears as unmet in the requirements notice, and an `analyst` role with `[frontier, thinking]` on a model that never thinks reports only `thinking` as unmet.

### Thinking modes

Every model has exactly one of three thinking modes: it never thinks, it always thinks, or it can switch thinking on and off per request. The mode belongs to the model, separate from the role's thinking switch described below. A model's thinking mode is spelled in lowercase as `never`, `always`, or `switchable`, and the requirements notice names the two fixed modes `Never` and `Always`.

| Thinking mode | What the model does | `thinking` role | `no-thinking` role |
|---|---|---|---|
| `never` | always runs with thinking off | refused | met |
| `always` | always emits thinking tokens | met | refused |
| `switchable` | lets each request turn thinking on or off | met | met, with thinking off |

A model in thinking mode `never` always runs with thinking off, so prepare refuses a `thinking` role on it with the message `requires 'thinking'; the current model's thinking capability is Never`. A model in thinking mode `always` always emits thinking tokens, so prepare refuses a `no-thinking` role on it as an unmet requirement. A model in thinking mode `switchable` lets each request turn thinking on or off, so a `no-thinking` role on it prepares, runs, and sends its requests with thinking off.

### The thinking switch

A role's hard keyword sets its thinking switch, the thinking setting that every request under the role sends. A `thinking` role asks for thinking on in every request, and a `no-thinking` role asks for thinking off. A role with neither leaves the switch unset, so its requests send no thinking setting at all, and the model's own thinking default stays in place.

The switch is fixed once per run from the role's keywords, when the run builds its set of roles from prepare's bindings. Every round under the role sends that same switch, even when `models.use` sets sampling options, which change only `temperature` and `max_tokens` (see [Sampling options on `models.use`](#sampling-options-on-modelsuse)).

The switch takes effect on a model that can switch thinking per request. A fixed-mode model only ever runs with a matching switch, because prepare refuses a `thinking` role on a `never` model and a `no-thinking` role on an `always` model.

A `no-thinking` role sends every request with thinking off, whether a section picks the role with `models.use` or runs on the prompt-wide default, and including when the model bound to the role is `switchable`. In `boiling-point` above, `models.default('responder')` makes `responder` the default, and the round in `## Ask` reaches it only through that default, yet it still asks for thinking off.

### Both hard keywords on one role

When a role lists both hard keywords, the later one in the list sets the thinking switch. This `models:` entry, part of a prompt's frontmatter, lists `no-thinking` last:

````markdown
models:
  drafter:
    keywords: [thinking, no-thinking]
````

A `switchable` model satisfies both hard keywords, so `drafter` still passes prepare on it, and its rounds follow the later keyword, `no-thinking`, and send thinking off.

## How prepare binds and checks roles

After this section you can tell which model each role is bound to, read a model id, and predict which roles prepare reports as unmet.

````markdown
---
name: two-roles
description: Declares two roles that share one model.
runvil: 0
models:
  scout:
    keywords: [frontier, fast]
  sprinter:
    keywords: [small, creative, chat]
---

# Two roles

## Main

```lua
return 'done'
```
````

````text
done
````

Prepare binds `scout` and `sprinter` to the Host's current model, one binding each. Neither role has a hard keyword or a `min_context`, so this run returns `done` on any current model, even a 32000-token model that never thinks.

### The model catalog and the current model

The models that a prompt's roles bind to come from the Host, not from the prompt file. The Host supplies a model catalog, its list of available models, in which each model is described by its model info and named by a model id, and it sets one of them as the run's current model before prepare. The current model is the one model that every role in the run is bound to, and the Host chooses it.

Each model's info carries a description, a context window, a thinking mode, and an optional provider id alongside its model id. These show up on a model handle as `description`, `context`, `thinking`, and `provider` (see [Model handles](#model-handles)):

- The description is prose that describes the model.
- The context window is a size in tokens. It is always at least one token and at most 4294967295 tokens, the largest unsigned 32-bit value, with windows such as 131072 and 8192 tokens.
- The thinking mode is `never`, `always`, or `switchable`, as [Thinking modes](#thinking-modes) explains.
- The provider id is present only when the catalog names one, and is absent otherwise.

A catalog keeps its models in the order the Host gives them. Each model id in a catalog names exactly one model, so no two available models share an id.

### Model ids

A model id is a model's stable name, made of a server namespace and a model name, and a model handle's `model_id` field comes from it. It is written `server/name`, the server namespace plus the model name the caller sees, such as `gateway/fast` or `gateway/deep`.

Each part of a model id can be any non-empty text without control characters, so a model name may contain `/`, `.`, `:`, and non-ASCII letters, as in `qwen/qwen3-8b`. Names that mix non-ASCII letters, hyphens, and digits, such as `café-模型`, `valid-alias`, and `claude-sonnet-4-6`, are all valid.

Two rules apply to each part, and a part that breaks one is refused with a reason:

- Each part holds at least one character, and an empty part gives the reason `must not be empty`.
- No part contains a control character, judged per Unicode character rather than per byte. Newline, NUL, U+001E, U+007F, and U+0085 are each refused with the reason `must not contain a control character`, while `café-模型` is accepted.

A model served by the gateway has the server namespace `gateway`. Its name is the name the gateway lists it under, which is the `name` key of its entry in the gateway configuration and the `id` field in the OpenAI-compatible `/models` list.

### Binding every role to the current model

At prepare, the Harness binds every role declared under `models:` to the Host's one current model. All of a prompt's roles share that model, two roles can never get different models, and roles differ only in their settings. Prepare binds exactly one model to each declared role, so a prompt that declares two roles gets two bindings, and `scout` and `sprinter` above both bind to the current model. The `analyst` role in the shipped `analyst-example.md` prompt is bound from the current model the same way.

When the Host has a current model, every declared role is bound to it and checked against the role's requirements. An empty role declaration such as `writer: {}` is valid, because all three keys are optional, so prepare binds it to the current model and reports the declaration satisfied. Several roles can be bound to the same model, and each still counts as its own bound role, so `writer` and `checker` both bound to `gateway/fast` count as two bound roles.

Binding to the current model is the only way roles are bound, and with no current model every role stays unbound. Prepare never picks a different model for each role and never searches for another model to satisfy one. To serve a role with another model, the Host sets that model as current and prepares again. A shortfall, such as a window that is too small, stays, with the role still bound to the same current model, until the Host supplies a better model, such as one with a larger window, and prepares again.

A bound role is served by the current model, so a request made through that role goes to the current model. Each bound role records the model id it is bound to, which is the current model's full model id, and that id leads back to the same model: a declared role's label leads to a model id, and the id leads to one of the models the run may use. The role's model handle reads back only the name part of that id as `model_id`, such as `fast` for `gateway/fast`.

A model request names the model by the name part of its id only. A request through a `writer` role bound to `gateway/m` carries `"model": "m"`, not `writer` or `gateway/m`, and a model bound as `gateway/fast` is requested as `fast`.

A label that the frontmatter does not declare as a role, such as `undeclared`, has no model binding.

### What prepare checks

A role's hard keywords and its `min_context` are requirements, and before the run starts the Harness checks them against the model bound to the role. These two checks are the only role requirements that can refuse the run, and prepare makes them only when the Host provides a current model. Soft keywords are never checked, so `frontier`, `fast`, `small`, `creative`, and `chat` never produce a role line in the notice, even against a 32000-token model that never thinks. A role that declares no hard keyword and no `min_context`, such as one with only soft keywords, is satisfied by any current model, so prepare reports nothing for it and it never blocks the run.

An unmet requirement is one shortfall that prepare records before the run starts (see [Model roles at prepare](04-how-a-prompt-runs.md#model-roles-at-prepare)). Prepare checks a role's keywords and its `min_context` as two separate checks, so a role can fail only its keyword, and a role that fails both gets one notice line for its context and another for its keyword:

````markdown
---
name: analysis
description: Needs a large context window and a thinking model.
runvil: 0
models:
  analyst:
    keywords: [frontier, thinking]
    min_context: 200000
---

# Analysis

## Main

```lua
return 'done'
```
````

On a Host whose current model has a 32000-token context window and never thinks, the run fails with run error kind `RequirementsUnmet` before any of the prompt's blocks run, and its message is the requirements notice:

````text
the environment cannot satisfy this prompt:
- role 'analyst': requires a context of at least 200000 tokens; the current model provides 32000
- role 'analyst': requires 'thinking'; the current model's thinking capability is Never
````

The notice opens with the line `the environment cannot satisfy this prompt:`, followed by one `- ` line per unmet check (see [The requirements notice](04-how-a-prompt-runs.md#the-requirements-notice)). `frontier` adds no line, because it is a soft keyword. Each unmet line names the role by the label the prompt declares under `models:`, shown as `role '{role}'` in both line forms.

### The context check

At prepare, a role's `min_context` is read as a token count and compared with the context window of the model bound to the role. The minimum is inclusive: a model whose window is at least the minimum, including exactly equal to it, meets the requirement. So `min_context: 200000` passes on a 200000-token model, `min_context: 100000` passes on a 200000-token model, and only a smaller window leaves the requirement unmet.

When a role's `min_context` is above the current model's context window, prepare records an unmet context-minimum requirement that names the role and gives the required and actual sizes, and the run is refused before it starts. The line takes this form:

````text
- role '{role}': requires a context of at least {required} tokens; the current model provides {actual}
````

So `analyst` with `min_context: 200000` on a 32000-token model is refused with required 200000 and actual 32000. A role `reader` that needs 200000 tokens on that model makes the whole message read `the environment cannot satisfy this prompt:` followed by `- role 'reader': requires a context of at least 200000 tokens; the current model provides 32000`.

### The thinking check

The `thinking` and `no-thinking` keywords are hard checks: when the bound model's thinking mode does not match what the role asks for, the role is unmet and prepare refuses the run. Both lines take this form, where `{required}` is the keyword and `{actual}` is `Never` or `Always`, the notice's spelling of the modes `never` and `always`:

````text
- role '{role}': requires '{required}'; the current model's thinking capability is {actual}
````

A `thinking` role is unmet only on a model whose thinking mode is `never`, which refuses the run with the line `- role '{role}': requires 'thinking'; the current model's thinking capability is Never`. An `always` or `switchable` model satisfies it, so a role with `keywords: [frontier, thinking]` runs on an `always` model.

A `no-thinking` role is unmet only on a model whose thinking mode is `always`, which refuses the run with the line `- role '{role}': requires 'no-thinking'; the current model's thinking capability is Always`. So `models: { triage: { keywords: [no-thinking] } }` on an `always` model reports role `triage` unmet, with required `no-thinking` and actual `Always`.

A `switchable` model, whose thinking can be turned on or off per request, satisfies both a `thinking` role and a `no-thinking` role, and it is asked to turn thinking off for a `no-thinking` role.

### When the Host sets no current model

````markdown
---
name: unread
description: Declares a role but never asks a model for a reply.
runvil: 0
models:
  writer: {}
---

# Unread

This prose is never sent to a model.

## Main

```lua
return 'ok'
```
````

On a Host with no current model, the run returns:

````text
ok
````

When the Host sets no current model, prepare binds no role and checks no requirement, so even a role with `min_context` does not stop the run from starting, and the run starts with no model. This prompt declares `writer`, but no section asks a model for a reply and its H1 prose is never read by a model, so the run still succeeds.

Prepare refuses nothing in this case, and the declared roles stay unbound. Selecting one with `models.use` or `models.default` then raises an error that names the label, the first message below from `models.use` and the second from `models.default`:

````text
models.use label "{label}" is not a bound model role
models.default label "{label}" is not a bound model role
````

So prepare reports success for a prompt that declares roles, and the run fails later, when a section selects a role. A clean prepare report does not prove that the roles are bound.

## The prompt-wide default with `models.default`

After this section you can choose, once, the role that every section's rounds run on unless a section picks its own.

````markdown
---
name: haiku
description: Drafts a haiku and then polishes it.
runvil: 0
models:
  writer: {}
---

# Haiku

```lua
models.default('writer')
```

## Draft

Write a haiku about autumn.

```lua
var.draft = models.infer(prose)
```

## Polish

Improve this haiku and reply with the haiku only: {{ var.draft }}

```lua
return models.infer(prose)
```
````

The H1 body makes `writer` the default and ignores the handle that `models.default` returns. Both `models.infer(prose)` calls, in `## Draft` and in `## Polish`, pass no model and run on `writer`, and the polished haiku becomes the run result.

The prompt-wide default is the role that a round naming no model runs on, in every section that has not picked its own role. `models.default(label)` takes the label of a role declared under `models:` and bound at prepare, makes that role the prompt-wide default for the whole run rather than just the section that calls it, and returns that role's model handle, as in `local writer = models.default('writer')`.

Once `models.default` names a role, a bare `models.infer(text)` or `models.loop(msgs)` that passes no model handle runs on that role in every section that has not picked its own role with `models.use`, even when the code ignores the handle that `models.default` returns. Model requests made under the default role use that role's bound model id and its thinking setting, so a default role declared with `keywords: [no-thinking]` sends its requests with thinking turned off.

Declaring roles under `models:` sets no default, so a run has no default role until Lua calls `models.default`.

### Where to call it

A default set with `models.default` in the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass) stays in effect in later sections, so `models.infer` and `models.loop` in those sections need no model argument. The H1 can also set the default by loading a library that calls `models.default`, with [`require`](03-blocks-and-prose.md#sharing-code-with-require), as in `require '## setup'` in an H1 block. A default set in the H1 serves both the later sections and any code those sections load with `require`.

`models.default` is conventionally called from the H1 body, but it works from any section, because the H1 body and every section share one set of roles and one default instead of each getting its own copy.

### Calling it again

The prompt-wide default is set once per run. Calling `models.default` again with the label that is already the default does nothing, and it still returns that role's handle. That makes it safe for a library that every section loads with `require` to name the default, and `local writer = models.default('writer')` works in every section that runs it.

### The handle it returns

`models.default` returns the role's model handle, which you can keep to inspect the role or to run rounds on it:

````markdown
---
name: capital
description: Runs one round on the default role's handle.
runvil: 0
models:
  writer: {}
---

# Capital

```lua
local writer = models.default('writer')
var.capital = writer:infer('Reply with one word: the capital of Peru.')
```

## Main

```lua
return var.capital
```
````

The H1 block keeps the handle in `writer` and runs one round on it with `writer:infer`, and `## Main` returns the model's one-word reply as the run result. [Model handles](#model-handles) covers the handle's methods.

`models.default` returns the same kind of model handle as `models.use`, with the same fields and methods. It takes only a label and no options table, so its handle reports `temperature` and `max_tokens` as nil, and rounds that run on the default send neither option. [Sampling options on `models.use`](#sampling-options-on-modelsuse) covers those two options.

### When `models.default` fails

Each failure raises an [error value](05-lua-environment.md#error-values) of kind `lua`. Calling `models.default` with a different label after a default is already set fails with a message that names the current default:

````text
models.default is already "{existing}": the prompt-wide default cannot change mid-run
````

So after the H1 makes `writer` the default, `models.default('critic')` in a later section fails with `models.default is already "writer": the prompt-wide default cannot change mid-run`.

Calling `models.default` with a label that names no bound role fails with a message that names the label, and the default stays unchanged:

````text
models.default label "{label}" is not a bound model role
````

So `models.default('ghost')`, in a prompt that declares no `ghost` role, fails with `models.default label "ghost" is not a bound model role`.

A `models.default` label that breaks the role label grammar fails with the grammar error whose message starts `invalid alias`, the same one `models.use` raises (see [Rejected calls](#rejected-calls)).

## Selecting a role per section with `models.use`

After this section you can run one section, or one part of a section, on a different role from the rest of the prompt.

````markdown
---
name: tagline
description: Drafts a tagline and has a second role critique it.
runvil: 0
models:
  writer: {}
  critic: {}
---

# Tagline

```lua
models.default('writer')
```

## Draft

Write a tagline for a bicycle shop.

```lua
var.tagline = models.infer(prose)
```

## Critique

```lua
models.use('critic')
```

Critique this tagline in one sentence: {{ var.tagline }}

```lua
return models.infer(prose)
```
````

`## Draft` makes no selection of its own, so its round runs on the prompt-wide default, `writer`. `## Critique` calls `models.use('critic')` in its first block, so the round in its last block runs on `critic`, and the critique becomes the run result.

`models.use(label)` selects a bound model role for the current section, naming the role by its label as a string, and returns a model handle for that role. The role a section has chosen this way, together with any options passed with it, is the section's `models.use` selection. After `models.use(label)`, the rounds that follow in the section, such as `models.infer`, run on the selected role's model. So with `writer` as the prompt-wide default, `models.use('analyst')` sends the next `models.infer('ping')` to the model bound to `analyst`.

`models.use` takes effect as a plain statement, so you can write it alone in its own `lua` block before the prose and the round it governs, without keeping its return value, as `## Critique` does. When you keep the handle, its `label` and `name` fields describe the role:

````markdown
---
name: label-check
description: Reads a selected role's label and name.
runvil: 0
models:
  writer: {}
---

# Label check

## Main

```lua
local h = models.use('writer')
return h.label .. '|' .. h.name
```
````

````text
writer|writer
````

Roles are bound before a section's Lua runs, and `models.use` and `models.default` choose among those already-bound bindings by label.

### Changing the selection within a section

Each section keeps its own `models.use` selection. Calling `models.use` again partway through the section succeeds and replaces the earlier label and options, because the selection is read when each round starts. In `tagline`, this one section could replace `## Draft` and `## Critique`:

````markdown
## Review

```lua
models.use('writer')
var.tagline = models.infer('Write a tagline for a bicycle shop.')
models.use('critic')
return models.infer('Critique this tagline in one sentence: ' .. var.tagline)
```
````

The first round runs on `writer` and the second on `critic`. `models.use` switches the model that later [`models.loop`](11-conversations.md#starting-a-conversation) calls send to in the same way, so selecting `other` and then `writer` sends the following `models.loop` requests first to the model bound to `other` and then back to the model bound to `writer`.

Selecting the same label again is allowed, and a plain re-selection clears the earlier options. So `models.use('writer', { temperature = 0.5, max_tokens = 64 })` followed by `models.use('writer')` leaves the section with no `temperature` or `max_tokens` override.

### Which model a round uses

A round that names no model runs on the section's model, picked in a fixed order:

1. The section's latest `models.use` selection.
2. Otherwise, the prompt-wide default set with `models.default`.
3. Otherwise, no model at all.

So a section that never calls `models.use` runs on the prompt-wide default role when one is set, and after `models.default('writer')` the section's model is the `writer` role. Binding a role does not select it, so a section that never calls `models.use`, in a prompt with no `models.default`, has no model for its rounds even when roles are bound. [When a prompt needs a role](#when-a-prompt-needs-a-role) shows that case.

A `models.use` selection applies only to the section whose Lua makes the call, and it overrides the prompt-wide default there. Every section starts with no selection, so a selection never reaches past its own section, and a later section on the prompt-wide default sends none of the options, such as `temperature` and `max_tokens`, that an earlier section passed.

The H1 body runs as a pass of its own with its own section state, so a `models.use` call in the H1 body selects a role only for rounds in the H1 pass, and the first section after the H1 still starts with no selection. To choose a role for every section, use `models.default`.

### The effective model

When a section has a `models.use` selection, its effective model is the selected role's binding with the `models.use` options applied on top. The options shape only that section's rounds, and the role itself stays unchanged, so after `models.use('writer', { temperature = 0.5, max_tokens = 64 })`, `models.get('writer')` still reads `temperature` and `max_tokens` as `nil`.

`models.use` takes at most two arguments: a label and an optional options table such as `{ temperature = 0, max_tokens = 256 }`. Leaving the table out, or passing `nil`, gives a selection with no sampling overrides. [Sampling options on `models.use`](#sampling-options-on-modelsuse) covers each option.

### Rejected calls

A label passed to `models.use` or `models.default`, or to `models.get` outside its model id fallback, follows the role label grammar: 1 to 64 bytes in all, starting with an ASCII letter and continuing with ASCII letters, digits, `_`, or `-`. `models.get` applies this check only after its bound-role lookup and its model id fallback (see [`models.get` errors and check order](#modelsget-errors-and-check-order)).

Every rejection from `models.use`, `models.default`, and `models.get` is raised at the call site as an error value of kind `lua` whose message is the rejection text, so [`pcall`](05-lua-environment.md#raising-and-catching-errors) can catch it and `tostring(err)` gives that text:

````markdown
---
name: ghost
description: Catches the selection of an undeclared role.
runvil: 0
models:
  writer: {}
---

# Ghost

## Main

```lua
local ok, err = pcall(function() return models.use('ghost') end)
return err.kind .. '|' .. tostring(err)
```
````

````text
lua|models.use label "ghost" is not a bound model role
````

`models.use` rejects a call in three ways, each with an error value of kind `lua`:

| Mistake | Message | What the message names |
|---|---|---|
| a label outside the label grammar | `invalid alias "{name}": expected [A-Za-z][A-Za-z0-9_-]{0,63}` | the rejected name and the grammar |
| a label that names no bound role | `models.use label "{label}" is not a bound model role` | the label |
| more than two arguments | `models.use takes at most 2 arguments, got {n}` | the argument count |

The grammar error comes from `models.use`, `models.default`, and `models.get` alike, and for a name that starts with a digit the message quotes that name. In `models.use` the grammar check runs before any role lookup. A call with a third argument after the options table gives `models.use takes at most 2 arguments, got 3`.

`models.use` runs its label and option checks before it records the selection, checking the label first, so an unknown label is reported even when the options are also bad, and a rejected call leaves the section's earlier selection unchanged.

An uncaught `models.use` or `models.default` rejection ends the run with run error kind `Lua`, like any other Lua failure, except in a `lua` fence of the H1 body, where an uncaught Lua failure becomes a `RequirementsUnmet` refusal that keeps the rejection as its cause (see [The H1 hard gate](04-how-a-prompt-runs.md#the-h1-hard-gate)).

## Sampling options on `models.use`

You can set the sampling temperature and the reply token cap for the model rounds a section makes.

````markdown
---
name: slogan
description: Writes a bakery slogan at a low temperature
runvil: 0
models:
  writer: {}
---

# Slogan

## Write

```lua
models.use('writer', { temperature = 0.3, max_tokens = 256 })
```

Write a slogan for a bakery.

```lua
return models.infer(prose)
```
````

The prologue selects the `writer` role for `## Write` and passes `models.use` a second argument, an options table. The epilog's model round then asks the model for a temperature of 0.3 and caps its reply at 256 generated tokens, and the reply becomes the run result.

The `temperature` and `max_tokens` keys are the sampling options: the two settings a `models.use` call can attach to its selection. They come only from `models.use`, because a model role declared under `models:` has no sampling fields of its own (see [Declaring roles under `models:`](#declaring-roles-under-models)). A model round that has neither option sends no temperature and no token cap.

### Reading sampling options back

Replace the `## Write` section of `slogan` with this one:

````markdown
## Write

```lua
local h = models.use('writer', { temperature = 0.3, max_tokens = 256 })
return tostring(h.temperature) .. '|' .. tostring(h.max_tokens)
```
````

````text
0.3|256
````

`models.use` still selects `writer`, and the model handle it returns reads the options back as `h.temperature` and `h.max_tokens` (see [Model handles](#model-handles)). The options apply both to model rounds on the selection and to rounds on that handle.

An option you leave out stays unset, and the handle reads it as `nil`:

````markdown
## Write

```lua
local h = models.use('writer', { temperature = 0 })
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
local careful = models.use('writer', { temperature = 0 })
models.use('writer')
return careful:infer('Write a slogan for a bakery.')
```
````

The second `models.use` call selects `writer` again with no options, so the section's selection now has none. The round on `careful` still asks for a temperature of 0, because a handle keeps the options it was made with.

Sampling options follow these rules:

- The options stay on the section's selection. They apply to every following model round in the section that names no model, such as `models.infer(prose)` and each round of a `models.loop` conversation (see [Starting a conversation](11-conversations.md#starting-a-conversation)), until another `models.use` call replaces the selection.
- A later plain `models.use(label)` with no options clears the options an earlier call set. The handle from `models.use('writer')` reads `nil` for both `temperature` and `max_tokens`, and the section's following rounds send neither.
- The handle a `models.use` call returns keeps that call's options for every round on it, `h:infer` and each round of `h:loop`, even after a later `models.use` changes the section's selection.
- The options never apply to rounds on the prompt-wide default or on a handle from `models.get`, because they shape only the section's effective model (see [The effective model](#the-effective-model)). A round on the prompt-wide default sends no temperature and no token cap, only the role's thinking switch (see [Keywords and thinking](#keywords-and-thinking)).

### Sampling option values

The options table names only `temperature` and `max_tokens`, written as string keys. The second argument of `models.use` is either that table or `nil`, and leaving it out or passing `nil` means no options.

| Option | Accepted values | Sent as | Read back as |
| --- | --- | --- | --- |
| `temperature` | A finite number from 0.0 to 2.0 inclusive, such as `0`, `0.7`, or `2.0` | The sampling temperature | `h.temperature`, a number |
| `max_tokens` | A whole number from 1 to 4294967295, written as a Lua integer or an integral float such as `1024.0` | The response token limit | `h.max_tokens`, a positive integer |

The `temperature` value goes out as the sampling temperature on the section's model requests, so `temperature = 0` asks the model for a temperature of 0. The `max_tokens` value goes out as the response token limit, so `max_tokens = 256` caps the reply at 256 generated tokens.

### Sampling option errors

````markdown
## Write

```lua
local ok, err = pcall(models.use, 'writer', var.options)
if not ok then
  return err.kind .. ': ' .. tostring(err)
end
return models.infer('Write a slogan for a bakery.')
```
````

When `var.options` holds a valid options table or `nil`, the block makes its round. When it holds a string, `pcall` catches an error value whose `kind` is `lua` and whose `tostring` is the message, and the block returns:

````text
lua: models.use options must be a table, got string
````

Every sampling option error is an error value of kind `lua` raised by `models.use`, which `pcall` catches the same way (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

| Problem | Message |
| --- | --- |
| The second argument is neither a table nor `nil` | `models.use options must be a table, got {type}` |
| A key is not a string, including a positional entry | `models.use option names must be strings, got {type}` |
| A string key other than `temperature` or `max_tokens` | `models.use option "{name}" is unknown: expected temperature or max_tokens` |
| `temperature` is not a number | `models.use option temperature must be a number, got {type}` |
| `temperature` is NaN or infinite | `models.use option temperature must be finite, got {value}` |
| `temperature` is below 0.0 or above 2.0 | `models.use option temperature {value} is outside the supported range [0.0, 2.0]` |
| `max_tokens` is not a whole number from 1 to 4294967295 | `models.use option max_tokens must be an integer in [1, 4294967295], got {actual}` |

In these messages, `{type}` is the Lua type name of the value given, such as `string` or `integer`, and `{name}` is the unknown option name. In the finite message, `{value}` is the value given, such as `NaN`. Finiteness is checked before the range, so an infinite temperature gets the finite message rather than the range message. In the range message, `{value}` is the number as written, so an integer temperature appears without a decimal point. In the `max_tokens` message, `{actual}` is the number given, such as `0`, `-1`, or `1.5`, or the type name of a value that is not a number, such as `string`.

When an options table holds more than one bad entry, `models.use` reports the same one on every run, whatever order Lua's `pairs` visits the keys in. It checks keys that are not strings first, sorted by type name, so a `boolean` key is reported before an `integer` key. It then checks string names in bytewise order, so a bad `max_tokens` is reported before a bad `temperature`, and any other name is reported at its own place in that order.

## Model handles

You can inspect the role a model handle is bound to and run model rounds with it.

````markdown
---
name: inspect
description: Reads a model handle's fields
runvil: 0
models:
  analyst:
    keywords: [frontier, thinking]
    description: Finds the weakest claim in a draft
---

# Inspect

## Look

```lua
local h = models.get('analyst')
return h.label .. '|' .. h.model_id .. '|' .. table.concat(h.capabilities, ',')
```
````

When the Host's current model is `claude-sonnet-4-6`, the run result is:

````text
analyst|claude-sonnet-4-6|frontier,thinking
````

`models.get('analyst')` returns the `analyst` role's handle (see [Getting handles with `models.get`](#getting-handles-with-modelsget)), and the block reads three of its fields to inspect the role without running a model round.

A model handle is the value that `models.use`, `models.default`, and `models.get` all return. It is frozen Lua userdata whose read-only fields describe the role it is bound to, and it has two methods, `infer` and `loop`, that run model rounds on that role. All three functions return the same kind of handle, with the same fields and methods.

### The ten handle fields

A model handle has exactly ten data fields, plus the `infer` and `loop` methods:

| Field | What it reads |
| --- | --- |
| `name` | The role label, as a string |
| `label` | The role label, the same string as `name` |
| `model_id` | The name part of the bound model's model id, such as `claude-sonnet-4-6` |
| `description` | The role's `description:`, or the bound model's catalog description, which can be empty |
| `context` | The bound model's context window in tokens, a positive integer such as `8192` |
| `capabilities` | The role's keywords, a sequence of kebab-case strings in declaration order |
| `provider` | The provider id the model catalog records for the bound model, or `nil` |
| `thinking` | `true` for a `thinking` role, `false` for a `no-thinking` role, `nil` for neither |
| `temperature` | The `temperature` option from `models.use`, a number, or `nil` |
| `max_tokens` | The `max_tokens` option from `models.use`, a positive integer, or `nil` |

`name` and `label` always hold the same string: the role label as you wrote it in the frontmatter `models:` map, the one the handle was obtained by. A handle from `models.get('writer')` or `models.use('writer')` reads `writer` for both. The role label stays the role label whatever model the role is bound to, so `models.get('analyst').name` reads `analyst`. On a handle for a raw model id, both fields hold that id (see [Raw model ids and the Host-state snapshot](#raw-model-ids-and-the-host-state-snapshot)).

`model_id` holds only the name part of the bound model's model id, such as `claude-sonnet-4-6` (see [Binding every role to the current model](#binding-every-role-to-the-current-model)), and it is the model's id, distinct from the role label. Every declared role binds to the Host's one current model (see [How prepare binds and checks roles](#how-prepare-binds-and-checks-roles)), so `model_id` reads the same on every declared role's handle. The `description`, `capabilities`, and `thinking` fields show how roles differ.

`description` reads the role's frontmatter `description:` as a string, so the `analyst` handle above reads `Finds the weakest claim in a draft`. When a role declares no `description:`, the field reads the bound model's own catalog description from the Host, which can be empty.

`context` reads the bound model's catalog context window in tokens, always a positive integer of at least 1, such as `8192`. It is the model's own window, not the role's `min_context`, so it reports what the model offers rather than what the role asked for.

`capabilities` reads the role's full declared keyword set from `keywords:` as a Lua sequence of strings, so `table.concat(h.capabilities, ',')` gives `frontier,thinking` for a role with `keywords: [frontier, thinking]`. An omitted `keywords:` is an empty list, so a role with no keywords gives an empty sequence, and `#h.capabilities` is `0`.

`provider` reads the provider id the model catalog records for the bound model, as a string. With a model whose catalog entry records the provider `xai`, `tostring(h.provider)` reads `xai`. With a model whose entry records none, it reads `nil`.

`thinking` reads the frozen thinking switch as a boolean, or `nil` when the role declared neither `thinking` nor `no-thinking` (see [Keywords and thinking](#keywords-and-thinking)).

`temperature` and `max_tokens` read `nil` on every handle except one that `models.use` returned with that option set, such as `0.3` and `256` after `local h = models.use('writer', { temperature = 0.3, max_tokens = 256 })`. A role has only `keywords`, `min_context`, and `description`, and declares no sampling settings of its own.

Each read of `capabilities` builds a fresh table. Replace the `## Look` section of `inspect` with this one:

````markdown
## Look

```lua
local h = models.get('analyst')
local keywords = h.capabilities
keywords[1] = 'fast'
return h.capabilities[1] .. '|' .. tostring(h.thinking)
```
````

````text
frontier|true
````

Changing the table one read returned leaves the handle unchanged, and the next read of `h.capabilities` returns the role's keywords again.

### Handle methods `infer` and `loop`

````markdown
## Look

```lua
local h = models.get('analyst')
return type(h) .. '|' .. type(h.infer) .. '|' .. type(h.loop)
```
````

````text
userdata|function|function
````

Every model handle has `infer` and `loop` fields, and reading either one gives a function value. You call them with a colon, which passes the handle as the first argument, as in `return analyst:infer(prose)`, `return models.get('analyst'):infer('ping')`, or `other:loop(msgs)`. `h:infer(prompt)` runs one model round on the handle's model and returns the reply (see [Running a round with `models.infer`](#running-a-round-with-modelsinfer)), and `h:loop` starts a conversation on it (see [Starting a conversation](11-conversations.md#starting-a-conversation)).

Calling `h:infer` or `h:loop` can pause the running Lua code while the model call is in progress. Reading one of the handle's data fields never does.

Model handles and message lists (see [Building a message list](11-conversations.md#building-a-message-list)) are the only Engine values with colon-call methods.

### A handle's frozen binding

````markdown
---
name: second-opinion
description: Asks the analyst role while the writer role is the default
runvil: 0
models:
  writer: {}
  analyst: {}
---

# Second opinion

```lua
models.default('writer')
```

## Check

```lua
return models.get('analyst'):infer('ping')
```
````

`writer` is the prompt-wide default, so a round in `## Check` that names no model would run on `writer`. The round on the `analyst` handle still sends its request to the model bound to `analyst`.

A model handle holds a frozen binding: the role binding fixed when the handle was made. Its fields report the values fixed then and read the same every time during the run. Its requests go to the concrete model its role is bound to, so a `writer` handle bound to `claude-sonnet-4-6` sends its requests to `claude-sonnet-4-6`. Later changes to the section's model never change which model a handle uses, and repeated rounds on one handle use the same model, as when two `other:loop(msgs)` calls both send their requests to the model bound to `other`.

### Read-only handle fields

Model handle fields can only be read. Assigning to any of them raises a Lua runtime error and leaves the handle unchanged. Reading any key other than the ten fields and the `infer` and `loop` methods raises a Lua error too, so `pcall` around such a read returns `false`.

A model handle is Lua userdata, not a table, so `type(h)` returns `'userdata'`. Only the userdata that `models.use`, `models.default`, or `models.get` returns counts as a model handle. A plain table that copies a handle's fields is not one, and the handle `infer` method raises an error when its first argument is anything other than a real handle (see [Infer errors and the missing-model error](#infer-errors-and-the-missing-model-error)).

## Getting handles with `models.get`

You can get the model handle for any declared role, or with a Host-state snapshot for any model id, without changing the section's selection.

````markdown
---
name: review
description: Asks the analyst role with no model selected
runvil: 0
models:
  analyst: {}
---

# Review

## Ask

```lua
return models.get('analyst'):infer('ping')
```
````

`models.get(label)` returns the model handle for a role declared under `models:`, so you reach the model bound to a role by passing its role label. Here the handle runs one round on the `analyst` role's model, and the reply is the run result. The call is chained without storing the handle, a common way to run one round on a role.

`## Ask` has no `models.use` call and the prompt has no `models.default`, so the section has no model for its rounds. The round still works, because a handle's round runs on its own frozen binding rather than the section's model.

### `models.get` keeps the selection

````markdown
---
name: bread-review
description: Drafts with the writer role and checks with the analyst role
runvil: 0
models:
  writer: {}
  analyst: {}
---

# Review

```lua
models.default('writer')
```

## Draft

Summarize why bread rises.

```lua
local analyst = models.get('analyst')
local summary = models.infer(prose)
return analyst:infer('Name the weakest claim in this summary: ' .. summary)
```
````

`models.get` returns a role's handle without selecting it. It changes neither the section's selection nor the prompt-wide default, so `models.infer(prose)` after it still goes to the section's model, here `writer`, and only the round on `analyst` uses the `analyst` role.

The handle describes the role's own settings, whatever the section has selected, so it never carries the sampling options of a `models.use` call (see [The effective model](#the-effective-model)).

`models.get` returns the same kind of model handle as `models.use` and `models.default`, a Lua userdata, so `type(models.get('writer'))` is `'userdata'`. You can store it and run several rounds on it, as in `local other = models.get('other')` followed by `other:loop(msgs)`. Model roles are reachable through `models.get` from a walked section, from a section run through `call` (see [Calling a section](08-jump-and-call.md#calling-a-section)), and from fanout arms (see [What an arm sees](14-fanout.md#what-an-arm-sees)) alike.

### Catching an unknown role label

````markdown
---
name: ask-role
description: Asks whichever role the argument string names
runvil: 0
models:
  writer: {}
  analyst: {}
---

# Ask role

## Ask

Say hello.

```lua
local ok, h = pcall(models.get, args)
if not ok then
  return 'unknown role: ' .. tostring(h)
end
return h:infer(prose)
```
````

The role label arrives as the run's argument string in `args` (see [The argument string in `args`](06-arguments.md#the-argument-string-in-args)). With the argument string `analyst`, the block runs one round on the `analyst` role. In a run without a Host-state snapshot, the argument string `missing` names no declared role, `pcall` catches the failure, and the block falls back instead of failing:

````text
unknown role: models.get alias "missing" is not a bound model role
````

A failing `models.get` call caught by `pcall` or `xpcall` yields an error value whose `kind` is `lua`, whose `message` is the error message, and whose `tostring` is that message. An `xpcall` handler sees the same error value.

### Raw model ids and the Host-state snapshot

````markdown
---
name: raw-id
description: Reads a handle for a raw model id
runvil: 0
---

# Raw id

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

A Host-state snapshot is the application state a Host can hand a run, the value that `ui()` returns (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)). With one, `models.get` treats a name that no role declares as a raw model id: a model id passed in place of a role label. It returns a handle for that model.

A prompt can then declare no model role at all, with no `models:` key in its frontmatter, and still reach a model by its model id, usually the one the Host reports in `ui().selected_model`:

````markdown
## Reply

Say hello.

```lua
return models.get(ui().selected_model):infer(prose)
```
````

Raw model ids follow these rules:

- A declared role whose role label matches the name passed to `models.get` wins over the raw model id.
- A raw model id skips the role label grammar, so it may contain `/`, `.`, and `:`, and only the model id's own rules apply (see [Model ids](#model-ids)). The id must be non-empty and free of control characters, or `models.get` fails with `models.get model id "{id}" is invalid: invalid model id: name {reason}`, where `{reason}` is `must not be empty` or `must not contain a control character`.
- A handle for a raw model id has `name`, `label`, `model_id`, and `description` all equal to the id, an empty `capabilities`, and `nil` for `provider`, `temperature`, `max_tokens`, and `thinking`. Its `context` is a fixed 8192 tokens, because the run has no catalog details for the id.
- A raw-id handle's `infer` and `loop` methods work like any other handle's, running on that model.
- A run binds every declared role to the Host's one current model, so a raw model id through `models.get` is the only way a prompt reaches a model other than the Host's current model.
- Only `models.get` has the raw model id fallback. `models.use` and `models.default` always need a bound role, and `models.use` fails with `models.use label "{label}" is not a bound model role` when no role matches.
- Without a Host-state snapshot there is no `ui` global, and `models.get` accepts only the role labels the prompt declares, so an undeclared name is an error rather than a raw model id.

### `models.get` errors and check order

Without a Host-state snapshot, a name that matches no bound role fails `models.get` with an error value of kind `lua` and the message `models.get alias "{alias}" is not a bound model role`, where `{alias}` is the name passed. The message reads the same wherever the call runs: on the main walk, inside a fanout arm, or in a run with no current model. A name outside the role label grammar `[A-Za-z][A-Za-z0-9_-]{0,63}` (see [The name grammar for role labels, arg names, and local tool aliases](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-local-tool-aliases)) fails with `invalid alias "{alias}": expected [A-Za-z][A-Za-z0-9_-]{0,63}` instead.

The three functions check a name in different orders:

1. `models.use` and `models.default` check the role label grammar, then look for the bound role.
2. `models.get` looks for a declared role first. When the Host gave a Host-state snapshot, it then treats the name as a raw model id. Otherwise it checks the role label grammar and reports the missing role.

### Role labels are never globals

````markdown
---
name: labels
description: Shows that role labels never become globals
runvil: 0
models:
  writer: {}
  tools: {}
---

# Labels

## Check

```lua
return type(writer) .. '|' .. type(tools)
```
````

````text
nil|table
````

Because a role label never becomes a Lua global (see [Role labels](#role-labels)), declaring a role `writer` leaves `type(writer)` as `nil`. For the same reason, a reserved name such as `store`, `models`, or `tools` can be a role label, and the Engine global with that name keeps its own value, so with a role `tools` declared, `type(tools)` is still `table`.

A global that shares a role label's name, such as one a Pack prelude sets, sits side by side with the role, and the role stays reachable through `models.get`. With a prelude that sets `writer = 'mine'` and a role `writer`, `writer .. '|' .. models.get('writer').name` gives `mine|writer`.

## Running a round with `models.infer`

You can send one prompt to a model in a single round and use the reply as a Lua string.

````markdown
---
name: bakery
description: Asks the writer role for a bakery slogan
runvil: 0
models:
  writer: {}
---

# Bakery

```lua
models.default('writer')
```

## Write

Write a slogan for a bakery.

```lua
return models.infer(prose)
```
````

`return models.infer(prose)` sends the section's prose, the rendered Markdown above the block (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)), to the model in one round and makes the reply the section's result. In this one-section prompt, the run result is whatever the model replies.

Each `models.infer` or `h:infer` call is one infer round: a model round on a fresh conversation whose only message is the prompt, sent as a single message with the `user` message role. Nothing from earlier rounds or from the rest of the section reaches the model.

### Which model an infer round uses

`models.infer(prompt)` runs on the section's model, which it looks up when the call runs, in the order [Which model a round uses](#which-model-a-round-uses) gives. In `bakery`, that is `writer`, the prompt-wide default.

`h:infer(prompt)` sends the prompt to the handle's own model instead, using the frozen binding the handle got when you obtained it, so the request has that handle's role label and model:

````markdown
## Write

```lua
return models.get('writer'):infer('say hello')
```
````

The handle form returns the reply as a plain Lua string, just as `models.infer` does. It runs on the handle's frozen binding even when the section has selected another role, has no model of its own, or the prompt has no prompt-wide default, so it never hits the missing-model error (see [When a section has no model](#when-a-section-has-no-model)).

### Infer prompts and replies

`models.infer(prompt)` takes exactly one required argument, the prompt string, and that is its whole signature. A handle's `h:infer(prompt)` takes the same prompt text. The prompt can be any Lua string: a literal such as `models.infer('hi')`, text read from the store such as `models.infer(store.read('note.md'))` (see [Writing and reading files](09-the-store.md#writing-and-reading-files)), or a string built from earlier results:

````lua
local inner = call('## Inner')
return models.infer('outer saw: ' .. inner)
````

Here `inner` holds the text that `call('## Inner')` returned, and the model receives `outer saw: ` followed by it.

The reply comes back as a plain Lua string, which the block can concatenate with `..`, compare, store in `var`, pass to `tostring`, or `return` as the section's or the run's result. Replace the `## Write` section of `bakery` with this one:

````markdown
## Write

Write a slogan for a bakery.

```lua
var.draft = models.infer(prose)
return models.infer('Shorten this slogan: ' .. var.draft)
```
````

You can call `models.infer` more than once in a section. Each call is its own model round, issued only after the previous one is answered, and each `models.infer` or `h:infer` call makes exactly one completion request to the model, so two calls make two requests, whether they sit in one section or in two.

`models.infer` is a suspending call, a Lua function on the section's `models` table. The block pauses while the round runs and resumes at its next statement once the reply arrives, so the section goes on only after the round is answered. Every model handle has an `infer` method, `h.infer`, defined alongside `models.infer` as one of the suspending calls.

`models.infer` works inside `pcall`. The block still pauses for the round, and when the round succeeds, `pcall` returns `true` and the reply text:

````lua
local ok, text = pcall(function() return models.infer('hi') end)
````

### What an infer round leaves out

An infer round sends the prompt and returns the reply, and nothing more:

- `models.infer` and `h:infer` each send a single round that offers the model no tools, even when the prompt declares tools, so no tool call can happen during the round, and a reply that comes back with tool calls anyway is refused (see [Errors from the model backend](#errors-from-the-model-backend)). When a Lua block needs a model round that can use tools, it runs a section with `call` instead (see [Calling a section](08-jump-and-call.md#calling-a-section)).
- The reply comes back whole as the call's return value, and none of it appears in the run's live output while the model is still writing.
- Any reasoning text the model backend supplies never becomes part of the returned value, which holds only the reply text.
- A round cut off at the generation cap, such as one limited by a `max_tokens` option, still returns the text the model produced up to that point.
- The round sets no global and leaves `sys` untouched, so `sys` reads the same before and after it (see [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui)).

### Infer rounds in tasks and arms

A section running as a spawned task (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)) can call `models.infer` to make its own model request, such as `models.infer('child work')` in a `### Child` section, and return the reply as the task's result. A fanout arm (see [What an arm sees](14-fanout.md#what-an-arm-sees)) can call `models.infer` too, such as `models.infer(prose)`, and each call makes exactly one model round whose reply the arm can return.

## Infer errors and the missing-model error

You can catch a failed infer call, branch on its error kind, and recognize the missing-model error.

````markdown
---
name: careful
description: Falls back when a model round fails
runvil: 0
models:
  writer: {}
---

# Careful

```lua
models.default('writer')
```

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

A failed `models.infer` or `h:infer` call raises an error value that you catch with `pcall` and branch on by its `kind` (see [Error values](05-lua-environment.md#error-values)). The errors fall into three groups: call-shape errors, errors from the model backend, and the missing-model error.

### Infer call-shape errors

Each infer form checks its arguments before any model request is made, and a bad argument raises an error value of kind `lua` at the call site, where `pcall` catches it. The checks run in this order: the first argument, then the argument count, where a trailing `nil` counts as an argument, then the prompt.

| Condition | Message |
| --- | --- |
| A model handle is the first argument to `models.infer` | `models.infer takes (prompt); call handle:infer(prompt) to run on a model handle` |
| `models.infer` gets any argument after the prompt, a trailing `nil` included | `models.infer takes (prompt)` |
| A handle's `infer` is called with a dot instead of a colon, or taken off the handle and called on its own, so its first argument is not the handle | `call infer on a model handle with a colon: handle:infer(prompt)` |
| `handle:infer` gets any argument after the prompt, a trailing `nil` included | `handle:infer takes (prompt)` |
| The prompt is missing or is not a string | `prompt must be a string, got {type}` |
| The prompt is not valid UTF-8 | `prompt must be a valid UTF-8 string` |

In the prompt message, `{type}` is the Lua type name of the value given, so a missing prompt gives `prompt must be a string, got nil` and a number gives `got integer`.

`models.infer` takes no handle and always runs on the section's model, so to run on a model handle you call `handle:infer(prompt)`. It also takes the prompt alone, so sampling options for a round go on `models.use` (see [Sampling options on `models.use`](#sampling-options-on-modelsuse)). Any argument after the prompt raises `models.infer takes (prompt)` at the call site rather than being ignored.

`models.loop` and `handle:loop` check their arguments the same way (see [Starting a conversation](11-conversations.md#starting-a-conversation)). A trailing `nil` counts there too, so an extra `nil` after the compactor raises `models.loop takes (messages, compactor?)` or `handle:loop takes (messages, compactor?)`. A model handle as the first argument to `models.loop` raises `models.loop takes (messages, compactor?); call handle:loop(messages, compactor?) to run on a model handle`, which names the method to call instead, just as `models.infer` does.

### Errors from the model backend

An infer call offers the model no tools, so it never dispatches tools. A model that replies to an infer round with tool calls fails the call with an error value of kind `internal` and this message:

````text
the model backend sent a reply that could not be understood: the round sent tool calls, but no tools were advertised
````

The text is the same for `models.infer` and `handle:infer`, and it never names `models.infer`. Left uncaught, the refusal fails the run with run error kind `Completion` (see [The run result and outcomes](04-how-a-prompt-runs.md#the-run-result-and-outcomes)), and the run error reports itself retryable, since a reply that broke protocol may come back well formed (see [Retryable failures](16-limits-and-errors.md#retryable-failures)).

An infer round that fails at the model, such as a transport or backend failure, raises the round's own error at the call site. `pcall` sees it with kind `internal`, or with kind `empty_model_reply` for an empty reply. Left uncaught, it fails the run with run error kind `Completion`.

### When a section has no model

The missing-model error is the error a model round raises when its section has no model, meaning neither a `models.use` selection nor a prompt-wide `models.default` is in effect (see [Which model a round uses](#which-model-a-round-uses)). Declaring a role under `models:` without selecting it or making it the default is not enough, which is why the `unselected` prompt in [When a prompt needs a role](#when-a-prompt-needs-a-role) fails with `model binding required for section Only`. The message is `model binding required for section {name}`, where `{name}` is the section name, such as `Only` or `Worker`, or the prompt's H1 title for a round in the H1 pass. Left uncaught, it fails the run with run error kind `Binding`. Raised in a fanout arm, it fails both the fanout and the run.

The missing-model error reaches the call as its answer, so `pcall` catches it as an error value of kind `internal` and a block can fall back instead of failing. Remove the H1 body from `careful`, and its block returns:

````text
skipped (internal)
````

`models.loop` in a section with neither `models.use` nor `models.default` in effect fails with the same missing-model error naming the section, which `pcall` also catches as kind `internal`.

Only an explicit model call such as `models.infer(prose)` can raise the missing-model error, so prose that no block sends to a model needs no `models.use` or `models.default`, in the H1 body or in a section, just as it needs no model at all (see [When a prompt needs a role](#when-a-prompt-needs-a-role)).

`handle:infer(prompt)` runs on the handle's own model, so it works in a section with no `models.use` selection and no `models.default`, where `models.infer` would fail with the missing-model error. In the `review` prompt that declares only `analyst` and sets no default, `models.get('analyst'):infer('ping')` succeeds.

## What an infer round reports

You can follow the events a run reports for each infer round, in order, and know what each one holds.

````markdown
---
name: greet-events
description: Makes one infer round on the writer role
runvil: 0
models:
  writer: {}
---

# Greet events

## Greet

```lua
return models.get('writer'):infer('say hello')
```
````

The block in `## Greet` reports these events, in this order:

1. `lua_chunk_started` for the block.
2. `model_turn_completed` when the round's model call finishes.
3. `assistant_reply` with the reply text, `origin` set to `infer`, and `turn` set to `1`, since this is the run's only model round.
4. `lua_chunk_succeeded` for the block.

Each infer call reports a `model_turn_completed` event inside the Lua block that made the call, between that block's `lua_chunk_started` and `lua_chunk_succeeded` events (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)). Each `handle:infer` call shows up in the run's events as exactly one completed model round, because an infer call drives exactly one model round trip.

### Infer event order

A successful infer round reports the same round events, in the same order, as any other model round in the calling section, except that its reply is tagged as an infer reply instead of a chat reply. In full, one infer round reports:

1. `model_turn_completed`.
2. One `model_metadata_degraded` event per problem, only when the backend's response has a malformed `usage`, `timings`, or `metrics` section, or names no model. Each event holds the round's `turn` and a message naming the problem, and the round itself still succeeds.
3. `thinking`, only when the backend supplies reasoning.
4. `model_turn_truncated`, only when the reply stops at the length limit with finish reason `length`. The reply text still comes back to Lua.
5. `assistant_reply`, the last event of the round.

When the backend's response names no model, the `model_metadata_degraded` message reads ``completion response named no string `model`; labeled with the requested model``, and the round is labeled with the requested model.

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
| `finish_reason` | The provider's finish reason, present only when the provider sent one |
| `metrics` | The call's metrics, present only when any were reported |

The Engine adds no metrics of its own, so `metrics` holds only what the provider reported.

Each infer round counts as one model round on the run's round counter, the number its events report in `turn`. Inside a spawned task (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)), it counts on the task's own counter instead, so a task's infer rounds count against the task's own cap rather than the run's.

An infer round's `thinking` event, `assistant_reply` event, and answer record name the model the round was actually sent to. That can be a substitute the Host picked in place of the model the prompt bound. The name comes from the model the request named, whatever name the backend's response gave.

### Infer failures and cancellation

When the model request for an infer round fails, the run reports a `model_turn_failed` event for the calling section, and the infer call fails with that error (see [Errors from the model backend](#errors-from-the-model-backend)).

When the run is cancelled while an infer round is still waiting for its answer, the run ends cancelled and the round's answer is dropped. No `model_turn_failed` event is reported, because cancellation never reports a model failure.
