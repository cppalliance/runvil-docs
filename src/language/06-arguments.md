# Arguments

Every run of a prompt starts with its arguments: one JSON object that the Host passes when it starts the run. A prompt declares the shape of that object under the `args:` frontmatter key, the Engine checks the object against the declaration at the run's first step, and every section's Lua reads the checked object from the read-only `args` global. This chapter covers declaring arguments, the six argument types, required arguments and defaults, the errors a declaration and a run's arguments can give, and reading `args`, so you know exactly what your Lua sees for any arguments a Host sends.

## Arguments at a glance

You can declare a prompt's arguments in the frontmatter and read them from `args` in any section, as this prompt does with three arguments named `query`, `limit`, and `depth`.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query:
      type: string
      description: What to search for
    limit:
      type: integer
      default: 3
    depth:
      type: string
      enum: [quick, full]
      default: quick
  required: [query]
---

## Search

```lua
return 'Results for ' .. args.query .. ', limit ' .. args.limit .. ', depth ' .. args.depth
```
````

Run it with the arguments `{"query":"papers"}`:

````text
Results for papers, limit 3, depth quick
````

The `args:` key declares one object with three fields. `query` is a required string, `limit` is an integer that defaults to `3`, and `depth` is one of the strings `quick` and `full` and defaults to `quick`. The arguments `{"query":"papers"}` match the declaration, the two fields they leave out take their defaults, and the `Search` section reads all three as fields of `args`. The rest of this section names each piece involved, and the later sections give the full rules for each one.

### The arguments are one JSON object

The arguments of a run are one JSON object. The [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host), the application that runs prompts, starts a run with the text of that object, such as `{"query":"papers"}`, and each field of the object is one argument. A Host with nothing to pass sends `{}`, the empty object. This chapter calls whoever supplies the arguments the caller.

### The `args:` declaration

A prompt declares its arguments in its [frontmatter](02-file-structure.md#the-shape-of-a-prompt-file), under the `args:` key. The `args:` declaration is the value of that key: a description of the object in a subset of JSON Schema, written in YAML. It is built from nodes. A node is a map of keywords that describes one value: its `type`, and whatever else the declaration says about it. The value of `args:` is the root node, which describes the whole object and is always `type: object`, and its `properties` map names each argument and gives that argument's node. [Declaring arguments in frontmatter](#declaring-arguments-in-frontmatter) gives every rule on the declaration's shape, [Argument types](#argument-types) covers the six types, and [Required arguments, defaults, and descriptions](#required-arguments-defaults-and-descriptions) covers `required`, `default`, and `description`.

A mistake in the declaration fails the parse, so the prompt never runs. [Declaration errors](#declaration-errors) gives the message for each mistake.

### The check at the run's first step

The Engine checks the arguments against the declaration at the run's first step, before any section runs, and arguments that do not match end the run there. Run the search prompt with the arguments `{"query":"papers","limit":"five"}`, and the run fails with this message:

````text
argument `limit` must be an integer, got string
````

Text that is not JSON, JSON that is not an object, a missing required argument, a value of the wrong type, a value outside its `enum`, and a field the declaration does not list all end the run the same way, with the [error code](16-limits-and-errors.md#run-error-codes) `args_invalid`. So a run's arguments reach its sections only when they match the declaration, and the prompt needs no checks of its own for anything the declaration says. [Argument errors](#argument-errors) gives each message.

### The `args` global

Your Lua reads the checked arguments through the `args` global, a table that is present in every section. Each field of the object is a field of the table, so the search prompt reads `args.query`, `args.limit`, and `args.depth`. A field the caller left out reads as its default, or as nil when it has none. [Reading arguments from `args`](#reading-arguments-from-args) covers fields, nested objects, and lists, and [Missing fields and null values](#missing-fields-and-null-values) covers what reads nil.

### `args` refuses writes

`args` is read-only in every section and at every depth, so a section always reads the arguments as the check left them. `args = {}` and `args.query = 'other'` both raise an error that says `args is read-only`, and so does a write to a nested object or to a list. A prompt that needs a different value builds it in Lua and keeps it in a local or in [`var`](05-lua-environment.md#keeping-state-in-var). [`args` is read-only](#args-is-read-only) covers the rule in full.

### A prompt with no `args:` key

A prompt whose frontmatter has no `args:` key takes one optional string argument named `text`. This prompt declares nothing and greets whoever its text names:

````markdown
---
name: greeter
description: Greets whoever the text names
runvil: 0
---

## Greet

```lua
return 'Hello, ' .. args.text .. '!'
```
````

Run it with the arguments `{"text":"world"}`:

````text
Hello, world!
````

[Prompts with no `args:` key](#prompts-with-no-args-key) gives the default declaration in full.

## Declaring arguments in frontmatter

You can declare each argument as a field of the root object, starting with a single required argument.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query:
      type: string
  required: [query]
---

## Search

```lua
return 'Results for ' .. args.query
```
````

Run it with the arguments `{"query":"papers"}`:

````text
Results for papers
````

### The root object and its `properties`

The value of `args:` is the root node, and it describes the whole object the caller passes. The root is always `type: object`. Its `properties` keyword holds a map from each argument's name to that argument's node, so `query:` with `type: string` under it declares a string argument named `query`. Its `required` keyword lists the names the caller must send. Here it lists `query`, so the arguments `{}` end the run with ``argument `query` is required``.

Every node must have a `type`, and the smallest node is a `type` alone, as the node of `query` is here. Only an `array` node needs a second keyword, its `items`.

### The eight keywords

A declaration is written with eight keywords, and any other keyword fails the parse:

| Keyword | Nodes that take it | What it declares |
|---|---|---|
| `type` | every node | the type of the value: `string`, `boolean`, `integer`, `number`, `array`, or `object` |
| `properties` | an `object` node | a map from each field name to that field's node |
| `required` | an `object` node | a list of the field names the caller must send |
| `additionalProperties` | an `object` node | `true` when the object also takes fields it does not declare |
| `items` | an `array` node | the one node that every item of the list must match |
| `enum` | a `string` or an `integer` node | the list of values the node allows |
| `description` | every node | text for whoever calls the prompt |
| `default` | every node but the root and an `items` node | the value the field takes when the caller leaves it out |

[Argument types](#argument-types) covers `type`, `items`, and `enum`, [Required arguments, defaults, and descriptions](#required-arguments-defaults-and-descriptions) covers `required`, `default`, and `description`, and [Declaration errors](#declaration-errors) gives the message for each mistake.

### Block style and flow style

The declaration is YAML, so you can write it in block style, one keyword per line, as the search prompt does, or as YAML flow maps, following the rules in [Writing YAML in the frontmatter](02-file-structure.md#writing-yaml-in-the-frontmatter). This `args:` key declares the same single required string as the block form above:

````markdown
args: { type: object, properties: { query: { type: string } }, required: [query] }
````

The two styles mix freely. A common layout keeps the root in block style and writes each field's node as a flow map on one line:

````markdown
args:
  type: object
  properties:
    query: { type: string, description: What to search for }
    limit: { type: integer, default: 3 }
  required: [query]
````

### Field names

Each name under `properties` follows the [name grammar](02-file-structure.md#the-name-grammar-for-arg-names-and-adhoc-tool-names) `[A-Za-z][A-Za-z0-9_-]{0,63}`: a letter, then up to 63 letters, digits, `_`, or `-`, so a name is 1 to 64 characters long. `query`, `use_mcp`, and `max-results` all fit. Names are case-sensitive, so `Query` and `query` are two arguments, and one `properties` map declares each name once. The grammar holds at every depth, for the fields of a nested object as for the fields of the root.

A name can be a [reserved name](02-file-structure.md#reserved-names), such as the Engine globals `store` and `var` or the Lua keyword `end`, as long as it fits the grammar. Names become fields of `args` and never globals, so this `args:` key parses with three arguments:

````markdown
args: { type: object, properties: { store: { type: string }, var: { type: string }, end: { type: boolean } } }
````

Lua reads the first two as `args.store` and `args.var`. A name that is a Lua keyword, or that holds a `-`, cannot follow a dot in Lua, so you read it with brackets, as in `args['end']` or `args['max-results']`.

### Nested objects

A field whose node is `type: object` holds an object of its own, declared the same way as the root, with its own `properties` and its own `required` list:

````markdown
args:
  type: object
  properties:
    query: { type: string }
    opts:
      type: object
      properties:
        depth: { type: string, enum: [quick, full] }
        mode: { type: string }
  required: [query]
````

The arguments `{"query":"papers","opts":{"depth":"full"}}` match this declaration, and Lua reads the inner field as `args.opts.depth`. An object can hold objects and lists of its own, and those can hold more. A `required` list names fields of its own object only, so the root's list here can name `query` and `opts`, and it cannot name `depth`.

### Lists

A field whose node is `type: array` holds a list, and its `items` keyword gives the one node that every item must match:

````markdown
args:
  type: object
  properties:
    files:
      type: array
      items: { type: string }
    pages:
      type: array
      items:
        type: object
        properties:
          url: { type: string }
          depth: { type: integer, default: 1 }
        required: [url]
````

`files` is a list of strings, such as `["a.md","b.md"]`, and `pages` is a list of objects, such as `[{"url":"x"},{"url":"y","depth":2}]`. An `array` node must have `items`. The items node can be any node, a list included, so `items: { type: array, items: { type: integer } }` declares a list of lists of integers. [List arguments](#list-arguments) covers what the check does with each item.

### Closed and open objects

An object is closed: it takes only the fields its `properties` declare, and a field the declaration does not list ends the run. With `query` as the only declared argument, the arguments `{"query":"papers","n":2}` fail with:

````text
argument `n` is not declared; the declared arguments are `query`
````

An open object says `additionalProperties: true`, and it also takes fields it does not declare:

````markdown
args:
  type: object
  properties:
    query: { type: string }
  additionalProperties: true
````

Under this declaration the same arguments pass, and `args.n` reads `2`. The Engine checks the declared fields of an open object as usual, and it keeps every other field as the caller sent it, whatever JSON the field holds. The keyword speaks for one object and not for the objects inside it, so a nested object stays closed unless it says `additionalProperties: true` itself. `additionalProperties: false` is the same as leaving the keyword out.

### A prompt that takes no arguments

A root with no `properties` declares an object with no fields, for a prompt that takes no arguments:

````markdown
args: { type: object }
````

The arguments `{}` pass, and `args` is an empty table. Any field ends the run, as `{"n":1}` does:

````text
argument `n` is not declared; the prompt declares no arguments
````

This is not the same as leaving the `args:` key out, which declares one optional `text` argument, as [Prompts with no `args:` key](#prompts-with-no-args-key) explains.

### The order of keywords and fields

The order in which you write the keywords of a node, and the fields of a `properties` map, does not matter. A node can give `required` before `properties` and `type` last. The declared fields are kept in name order, not in the order you write them, so a prompt that writes `zeta` before `alpha` has its arguments ordered `alpha`, then `zeta`: the check visits them in that order, a message that lists the declared arguments lists them in that order, and `pairs(args)` walks them in that order.

### What a Host reads from the declaration

A Host reads the declaration from the parsed prompt as plain JSON Schema, in which each closed object says `additionalProperties: false`. That is the form a tool's input schema takes.

## Argument types

You can give each node one of six types, each naming the kind of JSON value the caller must send for it.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    use_mcp: { type: boolean }
    limit: { type: integer }
    weight: { type: number }
    files: { type: array, items: { type: string } }
    opts:
      type: object
      properties:
        depth: { type: string }
      required: [depth]
  required: [query, use_mcp, limit, weight, files, opts]
---

## Search

```lua
return args.query .. ' / ' .. tostring(args.use_mcp) .. ' / ' .. args.limit .. ' / ' .. args.weight .. ' / ' .. #args.files .. ' files / ' .. args.opts.depth
```
````

Run it with the arguments `{"query":"papers","use_mcp":false,"limit":5,"weight":0.5,"files":["a.md","b.md"],"opts":{"depth":"full"}}`:

````text
papers / false / 5 / 0.5 / 2 files / full
````

Every node must have a `type` keyword, set to one of six lowercase words. Each word names the kind of JSON value the node takes, and it decides what your Lua reads:

| `type:` | JSON the caller sends | What Lua reads |
|---|---|---|
| `string` | a string | a string |
| `boolean` | `true` or `false` | a boolean |
| `integer` | a number with no fractional part that fits in 64 bits | an integer |
| `number` | any number | an integer or a float |
| `array` | an array | a list that counts from 1 |
| `object` | an object | a table keyed by field name |

A value of another type ends the run at its first step. Send `"use_mcp":"no"` in place of `"use_mcp":false` in the arguments above, and the run fails with:

````text
argument `use_mcp` must be a boolean, got string
````

The message names the type the node needs as `a string`, `a boolean`, `an integer`, `a number`, `an array`, or `an object`, and it names what the caller sent as `string`, `boolean`, `integer`, `number`, `array`, `object`, or `null`. A JSON number counts as `integer` when it is written with no decimal point and no exponent, as `5` is, and as `number` otherwise, as `5.0` and `1e3` are.

### String arguments

`type: string` takes a JSON string, and Lua reads a string with the same text. Nothing is trimmed and nothing is converted: `{"query":"  two words  "}` reads with its spaces, `{"query":"5"}` reads the string `'5'`, and the empty string `""` is a value like any other and reads `''`. A JSON number is never turned into text, so `{"query":5}` is refused.

### Boolean arguments

`type: boolean` takes JSON `true` or `false`, and Lua reads a boolean. Nothing else stands in for one: `{"use_mcp":1}` fails with ``argument `use_mcp` must be a boolean, got integer``, and `{"use_mcp":"true"}` fails the same way with `got string`.

### Integer arguments

`type: integer` takes a JSON number that has no fractional part and fits in 64 bits. Lua always reads an integer, a number for which `math.type` returns `integer`, however the caller wrote it:

| The caller sends | `args.limit` reads |
|---|---|
| `5` | `5` |
| `5.0` | `5` |
| `1e3` | `1000` |
| `-7` | `-7` |

So a section never has to turn a value such as `5.0` into an integer itself. A number with a fractional part, such as `2.5`, is refused:

````text
argument `limit` must be an integer, got number
````

A whole number too large for 64 bits, such as `1e300`, is refused with a message of its own:

````text
argument `limit` must be an integer that fits in 64 bits, got 1e+300
````

The largest value that passes is `9223372036854775807`, and the smallest is `-9223372036854775808`. A number written with a decimal point or an exponent is read as a float before the check, and a float cannot hold every whole number above 2^53, so `9223372036854775000.0` reads `9223372036854774784`. Write an integer that large as plain digits.

### Number arguments

`type: number` takes any JSON number. Lua reads an integer when the caller wrote the number with no decimal point and no exponent, and a float otherwise:

| The caller sends | `args.weight` reads | `math.type(args.weight)` |
|---|---|---|
| `5` | `5` | `integer` |
| `5.0` | `5.0` | `float` |
| `0.5` | `0.5` | `float` |
| `1e3` | `1000.0` | `float` |

The two forms compare equal, so `args.weight == 5` holds for both `5` and `5.0`. They differ as text: `'weight ' .. args.weight` gives `weight 5` for the first and `weight 5.0` for the second. Declare the node `integer` when the prompt needs a whole number. A whole number too large for 64 bits, which an `integer` node refuses, reads here as a float.

### Allowed values with `enum`

`enum` lists the values a `string` node or an `integer` node allows, and a value outside the list ends the run:

````markdown
args:
  type: object
  properties:
    depth: { type: string, enum: [quick, full] }
    level: { type: integer, enum: [1, 2, 3] }
````

With this declaration, the arguments `{"depth":"deep"}` fail with:

````text
argument `depth` must be one of "quick", "full", got "deep"
````

The arguments `{"level":4}` fail with:

````text
argument `level` must be one of 1, 2, 3, got 4
````

Strings match exactly, so `"Full"` is not `full`. An integer is compared after the integer rule, so `{"level":2.0}` passes and `args.level` reads `2`. The check tests the type before the list, so `{"depth":5}` fails with ``argument `depth` must be a string, got integer``. The message lists the allowed values in the order the declaration writes them. Of a refused string longer than 40 characters it shows the first 40, followed by `...`.

### List arguments

`type: array` takes a JSON array. Its `items` node is the type of every item, and the check tests each item against that node, in order from the first. Lua reads a list that counts from 1: with `files` declared as `{ type: array, items: { type: string } }` and the arguments `{"files":["a.md","b.md"]}`, `args.files[1]` is `a.md`, `args.files[2]` is `b.md`, and `#args.files` is `2`. An empty list passes, and `#args.files` is then `0`.

An item of the wrong type ends the run. The message names the item by the list's name and the item's index, which counts from 0. The arguments `{"files":["a.md",5]}` fail with:

````text
argument `files[1]` must be a string, got integer
````

Mind the two counts: the item the message calls `files[1]` is the one Lua would read as `args.files[2]`.

A value that is not an array is refused whole, as `{"files":"a.md"}` is with ``argument `files` must be an array, got string``. A caller with one file still sends a list of one.

### Object arguments

`type: object` takes a JSON object, and Lua reads a table whose keys are the field names. The node declares its fields the way the root does, and every rule of the root holds for it: it is closed unless it says `additionalProperties: true`, its `required` list names the fields the caller must send inside it, and its missing fields take their defaults. With `opts` declared as in [Nested objects](#nested-objects), the arguments `{"query":"papers","opts":{"depth":"full"}}` give `args.opts.depth` the value `full`. A JSON array is not an object, so `{"query":"papers","opts":[]}` fails with ``argument `opts` must be an object, got array``.

A mismatch inside a nested object is named by its path from the root, with a dot between the names. Under the same declaration, `{"query":"papers","opts":{"depth":"deep"}}` fails with:

````text
argument `opts.depth` must be one of "quick", "full", got "deep"
````

## Required arguments, defaults, and descriptions

You can require an argument, give one a default, and describe one, and this section shows what each of those keywords does at the run's first step.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query:
      type: string
      description: What to search for
    limit:
      type: integer
      default: 3
      description: Most results to return
    note:
      type: string
  required: [query]
---

## Search

```lua
local line = 'Results for ' .. args.query .. ', limit ' .. args.limit
if args.note == nil then
  return line .. ', no note'
end
return line .. ', note: ' .. args.note
```
````

Run it with the arguments `{"query":"papers"}`:

````text
Results for papers, limit 3, no note
````

Run it with the arguments `{"query":"papers","limit":5,"note":"recent"}`:

````text
Results for papers, limit 5, note: recent
````

The first run leaves out `limit` and `note`. `limit` has a default, so `args.limit` reads `3`. `note` has none and is not required, so it stays absent and `args.note` reads nil.

### Requiring an argument

A required argument is one that the `required` list of its object names. The caller must send it, and a run whose arguments leave it out ends at its first step. The search prompt run with the arguments `{}` fails with:

````text
argument `query` is required
````

A field is optional unless `required` names it, and nothing else makes a field required. Each name in the list must be a field that the same object's `properties` declares, and the list names each field once. `required` belongs to an object and not to a field, so a nested object has a list of its own for its own fields:

````markdown
args:
  type: object
  properties:
    opts:
      type: object
      properties:
        depth: { type: integer }
      required: [depth]
````

Here `opts` itself is optional, and `depth` is required inside it. The arguments `{}` pass, and `{"opts":{}}` fails with ``argument `opts.depth` is required``.

A required argument must hold a value, and JSON `null` is not one: the search prompt run with `{"query":null}` fails with ``argument `query` is required, got null``. An empty string is a value, so `{"query":""}` passes.

### Giving an argument a default

A default is the value a field takes when the caller leaves the field out. You write it with the `default` keyword on the field's node, as `default: 3` on `limit` above. The Engine fills in the default of each missing field before it checks the arguments, so for `{"query":"papers"}` the search prompt reads `args.limit` as `3`, exactly as it would for `{"query":"papers","limit":3}`. Your Lua never applies a default itself.

A node without the keyword has no default, and `default: null` says the same thing in writing. A field with no default stays absent when the caller leaves it out, and it reads nil.

A JSON `null` sent for a field that is not required counts as the field left out, so `{"query":"papers","limit":null}` also gives `args.limit` the value `3`.

Each type takes defaults of its own kind:

| `type:` | Example defaults |
|---|---|
| `string` | `draft`, `'42'` |
| `boolean` | `true`, `false` |
| `integer` | `3`, `-1` |
| `number` | `3`, `1.5` |
| `array` | `[]`, `[draft, open]` |
| `object` | `{}`, `{ depth: 2 }` |

YAML quoting decides a default's type. A quoted value is always a string, so you quote a default only for a `string` node, and you write `true`, `false`, and numbers bare. [A default must match its node](#a-default-must-match-its-node) shows what the parse refuses.

### Defaults at every depth

The Engine fills in defaults at every depth of the arguments: in the root, inside a nested object the caller sent, and inside each object item of a list. This declaration has a default in each of those places:

````markdown
args:
  type: object
  properties:
    limit: { type: integer, default: 3 }
    opts:
      type: object
      properties:
        depth: { type: integer, default: 1 }
    pages:
      type: array
      items:
        type: object
        properties:
          url: { type: string }
          depth: { type: integer, default: 1 }
        required: [url]
````

With the arguments `{"opts":{},"pages":[{"url":"a"},{"url":"b","depth":3}]}`, `json.encode(args)` returns:

````text
{"limit":3,"opts":{"depth":1},"pages":[{"depth":1,"url":"a"},{"depth":3,"url":"b"}]}
````

The root got `limit`, the `opts` object the caller sent got `depth`, and the first item of `pages` got `depth`, while the second item kept the `3` it was sent.

### Defaults inside an object the caller leaves out

An object field with no `default` stays absent when the caller leaves it out, and the defaults inside it are not filled in, because there is no object to hold them. Under the declaration above, the arguments `{}` give:

````text
{"limit":3}
````

So `args.opts` reads nil, and reading `args.opts.depth` raises a Lua error, because it indexes nil. To have the object always present, give it the default `{}`. The Engine then fills in the object's own missing fields, so the default arrives with the defaults of its fields:

````markdown
args:
  type: object
  properties:
    opts:
      type: object
      default: {}
      properties:
        depth: { type: integer, default: 1 }
        mode: { type: string, default: fast }
````

With the arguments `{}`, `json.encode(args)` returns:

````text
{"opts":{"depth":1,"mode":"fast"}}
````

A section then reads `args.opts.depth` without testing `args.opts` first. An object's default can hold fields too. With `default: { depth: 2 }` on `opts`, a caller who leaves `opts` out gets `{"depth":2,"mode":"fast"}`, and a caller who sends `{"opts":{}}` gets `{"depth":1,"mode":"fast"}`, the defaults of the fields themselves, because the object's default stands in only for a missing object.

A list takes a default the same way. With `default: []` on an `array` node, a missing list arrives as an empty list, so `#args.files` is `0` and a loop over `ipairs(args.files)` does nothing, with no test for nil.

### A required argument with a default

A field can be listed under `required` and have a `default` as well. When the caller leaves it out it takes the default, so it is never missing. What `required` still adds is the refusal of `null`:

````markdown
args:
  type: object
  properties:
    limit: { type: integer, default: 3 }
  required: [limit]
````

With this declaration, the arguments `{}` give `args.limit` the value `3`, and `{"limit":null}` fails with:

````text
argument `limit` is required, got null
````

Without the `required` list, `{"limit":null}` counts as the field left out, and `args.limit` reads `3`.

### Describing an argument

A description is text for whoever calls the prompt. You write it with the `description` keyword, which any node takes: the root, a field, a nested object, a list, and an `items` node. It plays no part in the check, and a node without the keyword has no description:

````markdown
args:
  type: object
  description: What the search needs
  properties:
    query: { type: string, description: What to search for }
    files:
      type: array
      description: Files to search in
      items: { type: string, description: One path in the store }
````

### How `required`, `default`, and `description` combine

The three keywords are independent, and one field can use any of them together. `description` never affects the check. `required` and `default` decide between them what a missing field and a `null` do:

| `required` names it | It has a `default` | The caller leaves it out | The caller sends `null` |
|---|---|---|---|
| no | no | it stays absent and reads nil | it stays absent and reads nil |
| no | yes | it takes the default | it takes the default |
| yes | no | the run ends: `is required` | the run ends: `is required, got null` |
| yes | yes | it takes the default | the run ends: `is required, got null` |

### A default must match its node

A default is checked when the prompt is parsed, against the node that declares it, by the rules the run applies to a caller's value. A default that does not match fails the parse, so a prompt never runs with a default that its own declaration would refuse. This declaration gives an `integer` node a string:

````markdown
args:
  type: object
  properties:
    limit: { type: integer, default: three }
````

The parse fails with a message that holds:

````text
args.properties.limit: the default must be an integer, got string
````

The same rules mean that a default follows its node's `enum`, that an object's default holds only declared fields with values of the right types, and that an `integer` node takes the default `3.0` and reads it as `3`. Two nodes take no default at all: the root and an `items` node. [Defaults that do not match](#defaults-that-do-not-match) gives each message.

## Declaration errors

You can recognize each mistake an `args:` declaration can hold by the message it gives, starting from this prompt, which parses and runs cleanly.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
---

## Search

```lua
return 'ready'
```
````

````text
ready
````

Mistype the `type` keyword as `tipe` and the prompt fails to parse:

````markdown
args:
  type: object
  properties:
    query: { tipe: string }
````

````text
invalid frontmatter: args.properties.query: unknown keyword "tipe"; the keywords are `type`, `properties`, `required`, `additionalProperties`, `items`, `enum`, `description`, and `default` at line 7 column 14
````

### What every declaration error shares

Every mistake in an `args:` declaration is a `Frontmatter` parse error, whose [error code](16-limits-and-errors.md#parse-codes) is `frontmatter_invalid`. The prompt fails to parse, the Engine stops before it reads the prompt body, and the prompt never runs, whatever arguments the caller sends, as [Parsing comes first](04-how-a-prompt-runs.md#parsing-comes-first) describes. The parse reports the first mistake it finds, so a declaration with two mistakes reports one of them.

A declaration error message has four parts. It starts with `invalid frontmatter: `. Next comes the YAML path of the node or keyword at fault, followed by a colon: `args` for the root, `args.properties.query` for the node of the `query` argument, and `args.properties.query.type` for that node's `type`. Then comes the reason. The message ends with a position, such as ` at line 7 column 14`, which counts the first line after the opening `---` as line 1. The position is the keyword itself for an unknown keyword, and the start of the node for a mistake that concerns the whole node, such as a missing `type`.

The error's location is the same place counted from the top of the file, where the opening `---` is line 1, so the `tipe` mistake above is located at line 8, column 14 (see [Where an error points](16-limits-and-errors.md#where-an-error-points)).

The rest of this section shows the path and the reason of each message, and leaves off the `invalid frontmatter: ` in front and the position at the end.

### Unknown keywords

A node takes only the [eight keywords](#the-eight-keywords). Any other key is refused where it stands and is never ignored, so a misspelled keyword cannot slip through, as `tipe` shows above. A JSON Schema keyword outside the eight is refused the same way:

````markdown
args:
  type: object
  properties:
    query: { type: string, minLength: 1 }
````

````text
args.properties.query: unknown keyword "minLength"; the keywords are `type`, `properties`, `required`, `additionalProperties`, `items`, `enum`, `description`, and `default`
````

A field name written directly under `args:` is read as a keyword of the root, so it gets the same refusal. The fields of the root go under `properties`:

````markdown
args:
  query:
    type: string
````

````text
args: unknown keyword "query"; the keywords are `type`, `properties`, `required`, `additionalProperties`, `items`, `enum`, `description`, and `default`
````

### A missing or unknown `type`

Every node must have a `type`, and a node without one is refused:

````markdown
args:
  type: object
  properties:
    query: { description: What to search for }
````

````text
args.properties.query: a node needs a `type`: `string`, `boolean`, `integer`, `number`, `array`, or `object`
````

The root is a node too. An `args:` key that gives `properties` and no `type: object` gets the same message with the path `args`, and so does the empty map `args: {}`. A prompt that takes no arguments writes `args: { type: object }`.

`type` takes exactly the six lowercase words. Any other word, such as a capitalized `String`, an alias like `int`, or `text`, is refused, and the path ends in `.type`:

````markdown
args:
  type: object
  properties:
    query: { type: text }
````

````text
args.properties.query.type: unknown variant `text`, expected one of `string`, `boolean`, `integer`, `number`, `array`, `object`
````

`null` is not a type either. A field that may be absent is simply one that no `required` list names.

A node has one type. A list of words, as in `type: [string, integer]`, is refused with:

````text
args.properties.query.type: invalid type: sequence, expected one type word: `string`, `boolean`, `integer`, `number`, `array`, or `object`
````

### A keyword on the wrong type

Five keywords belong to nodes of certain types: `properties`, `required`, and `additionalProperties` to an `object` node, `items` to an `array` node, and `enum` to a `string` or an `integer` node. On a node of another type each one is refused:

````markdown
args:
  type: object
  properties:
    query: { type: string, items: { type: string } }
````

````text
args.properties.query: `items` applies to an `array` node, and this node is `string`
````

The other four have messages of the same form. These are the messages for `properties` on a `string` node named `query`, `required` on an `array` node named `files`, `additionalProperties` on the same `query`, and `enum` on a `number` node named `weight`:

````text
args.properties.query: `properties` applies to an `object` node, and this node is `string`
args.properties.files: `required` applies to an `object` node, and this node is `array`
args.properties.query: `additionalProperties` applies to an `object` node, and this node is `string`
args.properties.weight: `enum` applies to a `string` or an `integer` node, and this node is `number`
````

So `enum` cannot limit a `boolean`, a `number`, a list, or an object.

### An array without `items`

An `array` node must say what its items are:

````markdown
args:
  type: object
  properties:
    files: { type: array }
````

````text
args.properties.files: an `array` node needs `items`
````

The value of `items` is one node, a map of keywords. A bare type word, as in `items: string`, is refused with:

````text
args.properties.files.items: invalid type: string "string", expected a map of schema keywords
````

### The `required` list

`required` names only fields that the `properties` of the same object declare, so a misspelled name fails the parse:

````markdown
args:
  type: object
  properties:
    query: { type: string }
  required: [quer]
````

````text
args: `required` names `quer`, which `properties` does not declare
````

A name listed twice, as in `required: [query, query]`, is refused with:

````text
args: `required` names `query` twice
````

`required` is a list on the object, and it is not a flag on a field. A field's node that says `required: true` is refused:

````markdown
args:
  type: object
  properties:
    query: { type: string, required: true }
````

````text
args.properties.query.required: invalid type: boolean `true`, expected a sequence
````

### The root

The root must be `type: object`, because the arguments are always one object:

````markdown
args:
  type: array
  items: { type: string }
````

````text
args: the root of `args` must be `type: object`, and it is `array`
````

The root takes no `default`. With `default: {}` beside its `type: object`, the parse fails with:

````text
args: the root of `args` takes no `default`
````

The value of `args:` must be a map of keywords. A bare word, as in `args: query`, is refused with:

````text
args: invalid type: string "query", expected a map of schema keywords
````

### Defaults that do not match

A default must match the node that declares it. A quoted number is a string, so this default is refused on an `integer` node:

````markdown
args:
  type: object
  properties:
    limit: { type: integer, default: '3' }
````

````text
args.properties.limit: the default must be an integer, got string
````

Each of these defaults fails the same way, with the reason shown:

| Node | Default | Reason |
|---|---|---|
| `{ type: string }` | `42` | `the default must be a string, got integer` |
| `{ type: boolean }` | `'true'` | `the default must be a boolean, got string` |
| `{ type: integer }` | `1.5` | `the default must be an integer, got number` |
| `{ type: number }` | `[1]` | `the default must be a number, got array` |
| `{ type: string, enum: [quick, full] }` | `deep` | `the default must be one of "quick", "full", got "deep"` |

The default of an object or of a list is checked all the way down, and a value inside it is named by its path from the default:

````markdown
args:
  type: object
  properties:
    page:
      type: object
      properties:
        limit: { type: integer }
      default: { limit: ten }
````

````text
args.properties.page: the default's `limit` must be an integer, got string
````

An object's default holds only fields that its node declares:

````markdown
args:
  type: object
  properties:
    field:
      type: object
      properties:
        size: { type: integer }
      default: { n: 1 }
````

````text
args.properties.field: the default's `n` is not declared; the declared fields of this node are `size`
````

When the node has no `properties`, the message ends with `this node declares no fields`. A default is also refused when it leaves out a field that its own node requires and that has no default of its own: an `opts` node with `required: [depth]` and `default: {}` fails with ``args.properties.opts: the default's `depth` is required``.

An `items` node takes no default, because a default is for a field the caller leaves out, and a list has no missing items:

````markdown
args:
  type: object
  properties:
    files:
      type: array
      items: { type: string, default: a }
````

````text
args.properties.files.items: a list item takes no `default`
````

`default: null` is never a mismatch, because it declares no default.

### Bad `enum` lists

An `enum` lists at least one value, lists no value twice, and holds only values of its node's type. An empty list, `enum: []`, on a node named `depth` is refused with:

````text
args.properties.depth: `enum` lists no value
````

A repeated value, as in `enum: [fast, slow, fast]` on a `string` node named `mode`, is refused with:

````text
args.properties.mode: `enum` lists "fast" twice
````

A value of another type, as in `enum: [1, two]` on an `integer` node named `n`, is refused with:

````text
args.properties.n: the `enum` of an `integer` node lists integers, and it lists a string
````

The same holds for a `string` node: `enum: [fast, 2]` on `mode` is refused with ``the `enum` of a `string` node lists strings, and it lists an integer``. YAML decides what each value is, so quote a word that YAML reads as something other than a string. `enum: [on, off]` lists two strings, while `enum: [true, false]` lists two booleans, which a `string` node refuses. Write that list as `enum: ['true', 'false']`.

### Bad field names

A name under `properties` that does not fit the [name grammar](02-file-structure.md#the-name-grammar-for-arg-names-and-adhoc-tool-names) fails the parse:

````markdown
args:
  type: object
  properties:
    2fast: { type: string }
````

````text
args.properties: invalid argument name `2fast`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

A name declared twice in one `properties` map, here `query`, is refused with:

````text
args.properties: duplicate argument name `query`: contract map keys must be unique
````

The path names the `properties` map that holds the name, so a bad name among the fields of a nested object `opts` has the path `args.properties.opts.properties`.

### A keyword written twice

A node holds each keyword once:

````markdown
args:
  type: object
  type: object
````

````text
args: keyword `type` is written twice
````

### A value of the wrong YAML kind

`properties` takes a map, `required` and `enum` take a list, `additionalProperties` takes a boolean, and `items` takes a node. A list where a map belongs, or a word where a list or a boolean belongs, is refused with a reason that starts `invalid type:` and names what was found and what was expected. YAML reads only `true` and `false` as booleans, so `yes` is a string:

````markdown
args:
  type: object
  additionalProperties: yes
````

````text
args.additionalProperties: invalid type: string "yes", expected a boolean
````

## Argument errors

You can recognize each way a run's arguments can fail the check by the message the run reports, starting from the search prompt of [Arguments at a glance](#arguments-at-a-glance).

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query:
      type: string
      description: What to search for
    limit:
      type: integer
      default: 3
    depth:
      type: string
      enum: [quick, full]
      default: quick
  required: [query]
---

## Search

```lua
return 'Results for ' .. args.query .. ', limit ' .. args.limit .. ', depth ' .. args.depth
```
````

Run it with the arguments `{"limit":"five","n":1}`:

````text
argument `limit` must be an integer, got string
````

These arguments hold three mismatches: `query` is missing, `limit` is not an integer, and `n` is not declared. The run reports one of them and ends at its first step, so `Search` never runs.

### What every argument error shares

Arguments that do not match the declaration end the run with the [error code](16-limits-and-errors.md#run-error-codes) `args_invalid`. The argument check is one of the start checks that the run's first step makes before any section runs, in this order: the `runvil` version, the arguments, the store, and the requirements (see [What the start checks look at](04-how-a-prompt-runs.md#what-the-start-checks-look-at)). Four things follow from that:

- No Lua has run when the check fails, so nothing in the prompt can catch the failure. A `pcall` in the preamble never gets its turn.
- The run reports no [run events](04-how-a-prompt-runs.md#run-events-and-records), not even `run_started`.
- The error has no file and no line, because the arguments come from the caller and no line of the prompt is at fault. Its message is the text shown here, with nothing after it.
- A prompt whose `runvil` version is refused fails with the version error whatever its arguments are, because the version is checked first.

Beside its code and its message, the error holds two facts, named values that a Host can read. `Reason` is one of `not_json`, `not_an_object`, `required`, `type`, `enum`, or `undeclared`, and when one argument is at fault, `Name` is that argument's path. The error's prose, the text a model reads for the failure, is the message, a period, and then `The arguments do not match what the prompt declares. Send one JSON object that matches its declared arguments.` For text that is not JSON, the prose leaves off the JSON parser's reason.

### The order of the check

The check reports the first mismatch only, so a caller who fixes it may then meet the next one. The order is fixed:

1. Text that is not JSON.
2. JSON that is not an object.
3. The declared fields of the root, in name order.
4. The undeclared fields of the root, in name order.

Each declared field is checked for a missing required value, then for its type, then for its `enum`, and then for its own contents: the fields of an object by steps 3 and 4, and the items of a list in index order. The contents of one field are checked before the next field, so a mismatch deep inside one argument is reported ahead of any mismatch in an argument whose name sorts after it.

In name order, the declared fields of the search prompt are `depth`, `limit`, and `query`. So the run at the top of this section reports `limit`, the first declared field at fault. With that fixed, the arguments `{"limit":5,"n":1}` fail with ``argument `query` is required``, and only with `query` sent does the run report the undeclared `n`.

### Text that is not JSON

A Host passes the arguments as text, and text that does not parse as JSON ends the run. The argument text `papers` fails with:

````text
the arguments are not valid JSON: expected value at line 1 column 1
````

The part after the colon is the JSON parser's reason and the place in the argument text where it stopped, and it varies with the text. `{"query":"papers",}` gives `trailing comma at line 1 column 19`, and `{'query':'papers'}`, with single quotes, gives `key must be a string at line 1 column 2`. Empty text is not JSON either and gives `EOF while parsing a value at line 1 column 0`, which is why a Host with nothing to pass sends `{}`.

### JSON that is not an object

Valid JSON of any kind but an object ends the run. The argument text `["papers"]` fails with:

````text
the arguments must be a JSON object, got array
````

The message names what was sent as `array`, `string`, `integer`, `number`, `boolean`, or `null`. A bare JSON string such as `"papers"` is refused with `got string`: text goes in a field of the object, as in `{"query":"papers"}`.

### A missing required argument

A required field that the arguments leave out, and that has no default, ends the run. The arguments `{}` fail with:

````text
argument `query` is required
````

A `null` in a required field ends the run too, whether the field has a default or not. The arguments `{"query":null}` fail with:

````text
argument `query` is required, got null
````

### A value of the wrong type

A value whose JSON type is not the declared type ends the run. The arguments `{"query":5}` fail with:

````text
argument `query` must be a string, got integer
````

An `integer` node has two refusals of its own. A number with a fractional part, as in `{"query":"papers","limit":2.5}`, fails with ``argument `limit` must be an integer, got number``, and a whole number too large for 64 bits, as in `{"query":"papers","limit":1e300}`, fails with ``argument `limit` must be an integer that fits in 64 bits, got 1e+300``. [Argument types](#argument-types) gives the rule for each type.

### A value outside its `enum`

A value of the right type that the node's `enum` does not list ends the run. The arguments `{"query":"papers","depth":"deep"}` fail with:

````text
argument `depth` must be one of "quick", "full", got "deep"
````

### An undeclared argument

A field that a closed object does not declare ends the run, and the message lists the fields the object does declare. The arguments `{"query":"papers","n":2}` fail with:

````text
argument `n` is not declared; the declared arguments are `depth`, `limit`, `query`
````

Under a root with no `properties`, the same field fails with ``argument `n` is not declared; the prompt declares no arguments``. An undeclared field of a nested object is named by its path, and the message names the object that holds it. With `opts` declared as in [Nested objects](#nested-objects), the arguments `{"query":"papers","opts":{"n":1}}` fail with:

````text
argument `opts.n` is not declared; the declared fields of `opts` are `depth`, `mode`
````

When the nested object declares no fields, the message reads ``argument `opts.n` is not declared; `opts` declares no fields``.

### Paths in the messages

A message names the argument at fault by its path from the root. A field of a nested object is joined to the object's name with a dot, as in `opts.depth`, and a list item is its list's name with the item's index in brackets, counted from 0, as in `files[1]`. The two combine. With `pages` declared as in [Lists](#lists), the arguments `{"pages":[{"url":"x"},{"depth":2}]}` leave the `url` out of the second item and fail with:

````text
argument `pages[1].url` is required
````

A `null` list item is a value of the wrong type, so with `files` declared as a list of strings, `{"files":["a.md",null]}` fails with ``argument `files[1]` must be a string, got null``.

## Reading arguments from `args`

You can read every argument as a field of the `args` table, nested objects and lists included.

````markdown
---
name: digest
description: Sums up a set of files for one query
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    files:
      type: array
      items: { type: string }
    opts:
      type: object
      default: {}
      properties:
        depth: { type: integer, default: 1 }
  required: [query, files]
---

## Digest

```lua
return args.query .. ': ' .. table.concat(args.files, ', ') .. ' (' .. #args.files .. ' files), depth ' .. args.opts.depth
```
````

Run it with the arguments `{"query":"papers","files":["a.md","b.md"]}`:

````text
papers: a.md, b.md (2 files), depth 1
````

### What `args` holds

`args` holds the object the check accepted: each field the caller sent a value for, plus the default of each field the caller left out. It is a table in every section, never nil and never a string, and it is an empty table when the arguments are `{}` and no field has a default. Apart from refusing writes, an `args` table behaves like the plain Lua table it stands for: you index it, measure it with `#`, and walk it with `pairs`, `next`, and `ipairs`.

### Reading fields

You read an argument with ordinary field access, `args.query`, or with brackets, `args['query']`, which also take a computed name. A JSON string reads as a Lua string, `true` and `false` read as booleans, and a number reads as an integer or a float, as [Argument types](#argument-types) lists for each type. Reading a field that `args` does not hold returns nil with no error, as [Missing fields and null values](#missing-fields-and-null-values) explains.

### Reading nested objects

A nested object is a table inside `args`, so you read its fields with further dots, as `args.opts.depth` does here, to whatever depth the declaration nests. Each table inside `args` behaves as `args` itself does. When an object field can be absent, test it before you read through it, as in `args.opts and args.opts.depth`, or give the field `default: {}` so that it is always there, as the `digest` prompt does.

### Reading lists

A JSON array reads as a Lua list that counts from 1. `#` gives the number of items, `ipairs` visits them in order, and `table.concat` and `table.unpack` read them as they would the items of any list:

````lua
local names = {}
for i, name in ipairs(args.files) do
  names[i] = string.upper(name)
end
local first, second = table.unpack(args.files)
return #args.files .. ' | ' .. table.concat(args.files, ', ') .. ' | ' .. table.concat(names, ', ') .. ' | ' .. first .. ' ' .. second
````

As the Lua block of `Digest`, with the arguments above, this returns `2 | a.md, b.md | A.MD, B.MD | a.md b.md`. An index past the end of the list reads nil, and so does index `0`.

### Walking `args` with `pairs`

`pairs(args)` visits every field that `args` holds, in sorted key order, like every `pairs` in Runvil (see [Deterministic table iteration](05-lua-environment.md#deterministic-table-iteration)):

````lua
local seen = {}
for name, value in pairs(args) do
  seen[#seen + 1] = name .. '=' .. type(value)
end
return table.concat(seen, ' ')
````

As the Lua block of `Digest`, with the arguments above, this returns `files=table opts=table query=string`. `opts` is there because its default filled it in. `next(args)` returns the first key in the same order, here `files`, and `#args` is `0`, because an object has no list items.

### Encoding `args` as JSON

`json.encode(args)` writes the whole object as JSON text with its keys sorted (see [Encoding and decoding JSON with `json`](05-lua-environment.md#encoding-and-decoding-json-with-json)). In the `digest` prompt, with the arguments above, it returns:

````text
{"files":["a.md","b.md"],"opts":{"depth":1},"query":"papers"}
````

It encodes any table inside `args` the same way, so `json.encode(args.files)` returns `["a.md","b.md"]`. An empty list encodes as `[]`, and an empty object as `{}`.

### `args` in every section

`args` is present in every section of the run: in the [preamble](02-file-structure.md#the-preamble), in every section on the [walk](04-how-a-prompt-runs.md#the-section-walk), and in library code loaded with [`require`](03-blocks-and-prose.md#sharing-code-with-require), both while the library loads and later, inside the functions it defines:

````markdown
---
name: reader
description: Reads its arguments in the preamble, a section, and a library
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
  required: [query]
---

```lua
var.seen = 'preamble: ' .. args.query
```

## Result

```lua
require '/helpers'
return var.seen .. ', library: ' .. loaded_with .. ', function: ' .. read_query()
```

## helpers

```lua
loaded_with = args.query
function read_query()
  return args.query
end
```
````

Run it with the arguments `{"query":"papers"}`:

````text
preamble: papers, library: papers, function: papers
````

The preamble's block, the top-level code of `helpers`, and the `read_query` function all read the same `args.query`.

### `args` in a called chain, a task, and a fanout arm

A chain that [`call`](08-jump-and-call.md#calling-a-section) runs reads its caller's `args` when the call passes no arguments, and it reads the arguments the call gives it when the call passes some. A task started with `tasks.spawn` follows the same rule through its `args` option (see [Tasks](15-tasks.md)), and a fanout arm reads its caller's `args`, with the member it runs for in `item` (see [Fanout](14-fanout.md)):

````markdown
---
name: relay
description: Reads the arguments in a called chain, a task, and a fanout arm
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    files: { type: array, items: { type: string } }
  required: [query, files]
---

## Main

```lua
local inherited = call('../Sub')
local own = call('../Sub', { query = 'chain' })
local task = tasks.join({ tasks.spawn('../Sub') })[1].result
local arms = tasks.fanout('Arm', args.files)
return inherited .. ' ' .. own .. ' ' .. task .. ' ' .. arms[1].text .. ' ' .. arms[2].text
```

### Arm

```lua
return args.query .. ':' .. item
```

## Sub

```lua
return args.query
```
````

Run it with the arguments `{"query":"run","files":["a.md","b.md"]}`:

````text
run chain run run:a.md run:b.md
````

The first `call` and the task read the run's `query`, the second `call` reads the `query` it was given, and each arm reads the run's `query` beside its own `item`. `args` is read-only in every one of them. A `call` or a spawn can also pass a string in place of a table, and the chain then reads that string as `args.text`.

The Engine does not check the arguments a `call` or a spawn passes against the `args:` declaration, and it fills in no defaults for them: the declaration describes what a caller passes the prompt, not what one section passes another. [`args` in a called chain](08-jump-and-call.md#args-in-a-called-chain) covers the rule in full.

### Arguments in prose

A [placeholder](07-substitution.md#placeholders-in-prose) inserts an argument into prose by its path, as `{{ args.query }}` and `{{ args.limit }}` do here:

````markdown
---
name: brief
description: Restates its query in prose
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    limit: { type: integer, default: 3 }
  required: [query]
---

## Topic

Find {{ args.limit }} papers on {{ args.query }}.

```lua
return prose
```
````

Run it with the arguments `{"query":"rust"}`:

````text
Find 3 papers on rust.
````

[Inserting arguments with `args`](07-substitution.md#inserting-arguments-with-args) covers how each kind of value renders and what a missing field does.

### Handing `args` tables to other calls

You can hand an `args` table to `json.encode`, to `tasks.fanout`, and to `var`, and each receives the data the table stands for. `json.encode` writes it, `tasks.fanout` takes a list from `args` as its collection, as the `relay` prompt does with `args.files`, and `var` stores it:

````lua
var.files = args.files
table.insert(var.files, 'c.md')
return #var.files .. ' in var, ' .. #args.files .. ' in args'
````

As the Lua block of `Digest`, with the arguments above, this returns `3 in var, 2 in args`. `var.files = args.files` stores a copy, the copy in `var` is writable, and `args.files` stays as it was.

## Missing fields and null values

You can tell in Lua whether the caller sent an optional field, and you can say what the check does with a JSON `null` in each place one can appear.

````markdown
---
name: notes
description: Reports whether a note was sent
runvil: 0
args:
  type: object
  properties:
    note: { type: string }
---

## Check

```lua
if args.note == nil then return 'absent' end
if args.note == '' then return 'empty' end
return 'note: ' .. args.note
```
````

Run it with the arguments `{}`:

````text
absent
````

Run it with the arguments `{"note":""}`:

````text
empty
````

### A field left out

A field the caller leaves out, with no default, is absent from `args`, and reading it returns nil with no error. That holds at every depth: for `args.note` here, for a missing field of a nested object, and for an index past the end of a list. What does raise an error is reading through a field that is absent. With an optional `opts` object left out, `args.opts` is nil, and `args.opts.depth` raises `attempt to index a nil value (field 'opts')`.

An empty string is not a missing field, as the second run shows. A caller who wants a field unset leaves it out, and does not send `""`.

### `null` for a field that is not required

A JSON `null` sent for a field that is not required counts as the field left out. The field takes its default, or stays absent when it has none, so the `notes` prompt run with the arguments `{"note":null}` returns `absent`. The same holds inside a nested object: when `opts` declares an optional `depth`, the arguments `{"opts":{"depth":null}}` arrive as `{"opts":{}}`. A declared field therefore never holds a null, and `args.note == nil` is the one test for a field that was not sent.

### `null` for a required field

A `null` sent for a field that `required` names ends the run at its first step, whether the field has a default or not, as [A required argument with a default](#a-required-argument-with-a-default) shows. With `query` required, the arguments `{"query":null}` fail with ``argument `query` is required, got null``.

### `null` in a list

A `null` item in a list ends the run, because `null` matches no items node. With `files` declared as a list of strings, the arguments `{"files":["a.md",null]}` fail with:

````text
argument `files[1]` must be a string, got null
````

### `null` in an open object

An object that says `additionalProperties: true` keeps a `null` in a field it does not declare, and Lua reads that value as [`json.null`](05-lua-environment.md#null-values-and-jsonnull):

````markdown
args:
  type: object
  properties:
    query: { type: string }
  additionalProperties: true
````

With the arguments `{"query":"papers","extra":null}`, `args.extra == json.null` holds, and `json.encode(args)` returns:

````text
{"extra":null,"query":"papers"}
````

An undeclared field is kept as the caller sent it, so a null inside it is kept too: with `{"ids":[1,null,3]}`, `args.ids[2] == json.null` holds. This is the only way a null in a run's arguments reaches `args`. A `null` in a declared field of the same object still counts as the field left out.

### Testing for each case in Lua

Each case reads as a different Lua value, so one comparison tells them apart:

| What the caller did | What Lua reads | The test |
|---|---|---|
| left out an optional field that has no default, or sent `null` for it | nil | `args.note == nil` |
| sent an empty string | `''` | `args.note == ''` |
| sent `false` | `false` | `args.use_mcp == false` |
| sent `null` in an undeclared field of an open object | `json.null` | `args.extra == json.null` |

A field that has a default never reads nil, and Lua cannot tell a default the Engine filled in from the same value sent by the caller. `if args.note then` passes for every string, the empty one included, and fails only for an absent field. For a boolean, `if not args.use_mcp then` passes both for `false` and for a field left out, so compare with `nil` or with `false` when the two must differ, or give the field a default. `json.null` is a userdata and so it is truthy, which means `if args.extra then` passes for a null.

## Prompts with no `args:` key

You can leave the `args:` key out, and the prompt then takes one optional string argument named `text`, which you read as `args.text`.

````markdown
---
name: echo
description: Returns the text it is given
runvil: 0
---

## Echo back

```lua
return args.text
```
````

Run it with the arguments `{"text":"hello there"}`:

````text
hello there
````

### The default declaration in full

A prompt whose frontmatter has no `args:` key has the default declaration. It is the same as a prompt that writes this `args:` key:

````markdown
args:
  type: object
  properties:
    text: { type: string, description: Freeform text for this prompt }
````

The default declaration has one field, `text`, a string with the description `Freeform text for this prompt`. No `required` list names it, it has no default, and the root is closed. So every prompt has a declaration, and every run's arguments are checked against one.

### Sending text

A Host with plain text for such a prompt sends the text as the `text` field, as in `{"text":"hello there"}`, and the prompt reads `args.text`. The text alone is not the arguments. The argument text `hello there` fails with `the arguments are not valid JSON: expected value at line 1 column 1`, and `"hello there"`, a JSON string, fails with `the arguments must be a JSON object, got string`.

### Runs with no text

`text` is optional, so the arguments `{}` pass. `args` is then an empty table and `args.text` reads nil, as it also does for `{"text":null}`, while `{"text":""}` gives `args.text` the empty string. A prompt that can run without its text tests for nil:

````markdown
---
name: greeter
description: Greets whoever the text names
runvil: 0
---

## Greet

```lua
return 'Hello, ' .. (args.text or 'stranger') .. '!'
```
````

Run it with the arguments `{}`:

````text
Hello, stranger!
````

Run it with the arguments `{"text":"world"}`:

````text
Hello, world!
````

### Any other field is refused

The root of the default declaration is closed, so a field other than `text` ends the run. The `echo` prompt run with the arguments `{"query":"papers"}` fails with:

````text
argument `query` is not declared; the declared arguments are `text`
````

A `text` that is not a string is refused as well: `{"text":5}` fails with ``argument `text` must be a string, got integer``.

### The text in prose

A [placeholder](07-substitution.md#placeholders-in-prose) inserts the text into prose as `{{ args.text }}`:

````markdown
---
name: brief
description: Restates its topic
runvil: 0
---

## Topic

Topic: {{ args.text }}.

```lua
return prose
```
````

Run it with the arguments `{"text":"rust"}`:

````text
Topic: rust.
````

With the arguments `{}` there is no `text` to insert, so the read of `prose` fails with `missing {{ args.text }} [MissingKey at byte 7]`. [Inserting arguments with `args`](07-substitution.md#inserting-arguments-with-args) covers placeholders in full.

### A prompt that needs its text

A prompt that cannot run without its text declares `text` itself and lists it under `required`:

````markdown
---
name: brief
description: Restates its topic
runvil: 0
args:
  type: object
  properties:
    text: { type: string, description: The topic to restate }
  required: [text]
---

## Topic

Topic: {{ args.text }}.

```lua
return prose
```
````

Run it with the arguments `{}`:

````text
argument `text` is required
````

The run ends at its first step, before any section runs, and the message tells the caller what to send. With the arguments `{"text":"rust"}` the prompt returns `Topic: rust.` as before.

## `args` is read-only

You can rely on `args` staying exactly as the check left it, because `args` is read-only in every section: a write to it is refused, and a read still works.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
    files: { type: array, items: { type: string } }
    opts:
      type: object
      properties:
        depth: { type: integer }
  required: [query]
---

## Search

```lua
local wrote = pcall(function() args.query = 'other' end)
if wrote then return 'written' end
return 'refused, query is still ' .. args.query
```
````

Run it with the arguments `{"query":"papers","files":["b.md","a.md"],"opts":{"depth":2}}`:

````text
refused, query is still papers
````

The write `args.query = 'other'` raises an error that `pcall` catches, and `args.query` still reads `papers`. The rest of this section gives the error each write raises and shows how a prompt keeps a value of its own. Each Lua fragment below runs as the Lua block of `Search`, with these same arguments.

### What a write raises

A write to `args` raises a Lua error. Each one is listed here, and the subsections after the table explain them:

| Write | Message |
|---|---|
| `args = {}`, `args = nil`, `args = 'x'`, `args = 6` | ``args is read-only: assign to `var` or a section global instead`` |
| `args.query = 'x'`, `args.query = nil` | `args is read-only: cannot set field 'query'` |
| `args.mode = 'x'` | `args is read-only: cannot set field 'mode'` |
| `args.opts.depth = 3` | `args is read-only: cannot set field 'depth'` |
| `args.files[1] = 'x'` | `args is read-only: cannot set field 1` |
| `args.files[#args.files + 1] = 'c.md'` | `args is read-only: cannot set field 3` |
| `table.insert(args.files, 'c.md')` | `args is read-only: cannot set field 3` |
| `table.sort(args.files)` | `args is read-only: cannot set field 1` |
| `args[2.5] = 1` | `args is read-only: cannot set field number` |
| `setmetatable(args, {})` | `cannot change a protected metatable` |

### Assigning `args`

Assigning the `args` global itself, with any value, raises a Lua runtime error with the Engine's refusal message:

````text
args is read-only: assign to `var` or a section global instead
````

The refusal comes before the value is looked at, so `args = {}`, `args = nil`, `args = 'x'`, and `args = 6` all raise it, and so does an assignment through `_G`, as in `_G.args = {}`. The message names the two places for a value of your own: [`var`](05-lua-environment.md#keeping-state-in-var), which the sections after this one read too, and a global of the section, which lasts for that section entry.

### Field writes

Writing a field of `args` raises a Lua error with the message `args is read-only: cannot set field 'name'`, with the field's name in single quotes. So `args.query = 'x'` gives `args is read-only: cannot set field 'query'`. The refusal is the same for a field that `args` holds, for a field it does not hold, and for an assignment of nil, so `args.mode = 'x'` and `args.query = nil` are refused too.

`args` is read-only at every depth. A write to a table inside `args` is refused the same way, and its message names only the innermost key, so `args.opts.depth = 3` reports `cannot set field 'depth'`. A local that holds a table from `args` holds that same read-only table: after `local opts = args.opts`, the write `opts.depth = 3` is refused too.

A write to a list names the index without quotes. `args.files[1] = 'x'` gives `args is read-only: cannot set field 1`, and appending with `args.files[#args.files + 1] = 'c.md'` gives `args is read-only: cannot set field 3`, because the list holds two items.

A write with a key that is neither a string nor an integer names the key's Lua type in place of its value, so `args[2.5] = 1` gives `args is read-only: cannot set field number`, and `args[true] = 1` gives `args is read-only: cannot set field boolean`.

### Table functions that write

`table.insert`, `table.remove`, and `table.sort` write into the list they are given, so on a list inside `args` each raises the field refusal for the first index it tries to set. With the two files above, `table.insert(args.files, 'c.md')` raises `args is read-only: cannot set field 3`, `table.remove(args.files)` raises `args is read-only: cannot set field 2`, and `table.sort(args.files)` raises `args is read-only: cannot set field 1`.

`table.sort` writes only when it has to move an item. On a list that is already in order, such as `["a.md","b.md"]`, it has nothing to set and returns without an error, so do not count on it to raise. Sort a copy instead, as [Keeping a value of your own](#keeping-a-value-of-your-own) shows.

### Catching a refused write with `pcall`

A refused write is an ordinary Lua error, so you catch it with [`pcall`](05-lua-environment.md#raising-and-catching-errors). The caught value is an [error value](05-lua-environment.md#error-values) whose `kind` is `lua` and whose `code` is `lua_runtime`, and `tostring` on it gives the file and line of the write, then the refusal, then the fixed prose for a Lua runtime failure:

````lua
local ok, err = pcall(function() args.query = 1 end)
return err.kind .. ' | ' .. err.code .. ' | ' .. tostring(err)
````

This block returns:

````text
lua | lua_runtime | search.md:20: args is read-only: cannot set field 'query'. The Lua code failed at runtime. Fix the code at this location.
````

The prompt at the top of this section uses the same pattern with a flag: `local wrote = pcall(function() args.query = 'other' end)` leaves `wrote` false, so the block goes on to its last line and returns `refused, query is still papers`.

### Uncaught write errors

An uncaught write to `args` ends the run with the [error code](16-limits-and-errors.md#run-error-codes) `lua_runtime`, the same code as any other runtime error in a section's Lua. Its message holds the refusal, as in `runtime error: args is read-only: cannot set field 'query'`, and it ends with the location of the write, such as ` (at search.md:20)` (see [Where an error points](16-limits-and-errors.md#where-an-error-points)).

### Keeping a value of your own

A prompt that needs a different value builds it in Lua and keeps it in a local, or in [`var`](05-lua-environment.md#keeping-state-in-var) when later sections need it too:

````lua
local query = string.upper(args.query)
local sorted = { table.unpack(args.files) }
table.sort(sorted)
var.files = args.files
table.insert(var.files, 'c.md')
return query .. ' | ' .. table.concat(sorted, ', ') .. ' | ' .. table.concat(var.files, ', ') .. ' | ' .. table.concat(args.files, ', ')
````

This block returns:

````text
PAPERS | a.md, b.md | b.md, a.md, c.md | b.md, a.md
````

`{ table.unpack(args.files) }` builds a plain list with the same items, which the block is free to sort. `var.files = args.files` stores a copy of the list, the copy in `var` is writable, and `args.files` stays as it was.

### Metatables on `args` tables

`getmetatable(args)` returns the string `args is read-only` in place of a metatable, and so does `getmetatable` on any table inside `args`.

`setmetatable` on `args`, or on any table inside it, raises `cannot change a protected metatable`, the same error `setmetatable` gives on the protected `sys` and `var` tables. `pcall` catches this refusal as a plain string and not as an error value, so it has no `kind` and no `code`: after `local ok, err = pcall(setmetatable, args, {})`, `err` is the string `cannot change a protected metatable`. When the `setmetatable` call sits inside a function of yours, the string has the position of that call in front of the same text.

### Your own `_G` metatable

A metatable you set on `_G`, as [Giving `_G` a metatable](05-lua-environment.md#giving-_g-a-metatable) describes, cannot free `args` and never sees a read or an assignment of `args`, because the Engine's guard on `_G` handles that name first. After `setmetatable(_G, nil)`, `setmetatable(_G, {})`, or `setmetatable(_G, mt)` with an `__newindex` of yours that captures writes and an `__index` that returns `'author'`, `args.query` still reads the run's value, such as `papers`, and assigning `args` still raises the read-only error. Under a strict `__index` that raises an error for every undefined global, `args` still reads as the arguments.

### Everything else stays writable

The read-only rule guards only `args`. In any section you still define, read, and assign your own globals, so `scratch = 42` followed by `assert(scratch == 42)` works beside the read-only `args`, and `var` takes writes as it always does. A local or a function parameter named `args` is an ordinary Lua variable: it hides the global inside its scope, by Lua's own scoping, and it is yours to write.

You keep reading the [`prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global), the rendered Markdown above a fence, beside `args`. In this prompt, `prose` reads as the Engine set it in the same fence where assigning `args` is refused:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  type: object
  properties:
    query: { type: string }
  required: [query]
---

## First

Tone: friendly.

```lua
assert(prose == 'Tone: friendly.')
local ok = pcall(function() args = 'hijacked' end)
assert(not ok)
return args.query .. ' / ' .. prose
```
````

Run it with the arguments `{"query":"papers"}`:

````text
papers / Tone: friendly.
````
