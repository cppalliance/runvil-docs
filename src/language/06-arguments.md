# Arguments

Every run of a prompt starts with one argument string, the text the Host passes when it starts the run, and your Lua can read that string exactly as it was sent. A prompt can also declare typed arguments under the `args:` frontmatter key, and the argument string then arrives parsed as JSON in the `argv` global. This chapter covers declaring arguments, reading them through `args` and `argv`, and checking and repairing them in the H1 pass before any section runs, so you know exactly what your Lua sees for any argument string a Host sends.

## Arguments at a glance

You can declare an argument in the frontmatter and read it from `argv` in any section, as this prompt does with one argument named `query`.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## Search

```lua
return 'Results for ' .. argv.query
```
````

Run it with the argument string `{"query":"papers"}`:

````text
Results for papers
````

The `args:` key declares one argument, `query`, of type `string`. The argument string `{"query":"papers"}` is parsed as JSON, and the `## Search` section reads its `query` field as `argv.query`. The rest of this section names each piece involved, and the later sections give the full rules for each one.

### The argument string

Every run starts with exactly one argument string from the [Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host), the application that runs prompts. When the Host passes nothing, as it does for a prompt that takes no arguments, the argument string is empty. A prompt with no `args:` key started that way sees `argv.prose == ''`, as [Prompts with no `args:` key](#prompts-with-no-args-key) explains.

### Declaring arguments under `args:`

A prompt declares its arguments in its [frontmatter](02-file-structure.md#the-shape-of-a-prompt-file), and every arg declaration names an argument type, one of four words for the kind of JSON value the arg expects. The `args:` declaration is the value of the `args:` frontmatter key: a map from each arg name to that arg's declaration. This `args:` key, written in flow style, declares an optional `integer` argument named `times` and a `string` argument named `name`:

````markdown
args: { times: { type: integer, optional: true, default: 1 }, name: { type: string } }
````

The same kind of map can be written in block style, one arg per entry. This `args:` key declares `use_mcp`, `limit`, and `query`:

````markdown
args:
  use_mcp:
    type: boolean
    default: true
    description: Search private sources
  limit:
    type: integer
    optional: true
  query:
    type: string
````

An arg declaration is the map under one arg name. It has a required `type` key and the optional keys `optional`, `default`, and `description`: `optional` says whether the argument string may leave the arg out, `default` gives the arg's default value, and `description` describes the arg. So `times: { type: integer, optional: true, default: 1 }` and `name: { type: string }` are both arg declarations. [Declaring arguments in frontmatter](#declaring-arguments-in-frontmatter) gives every rule on their shape.

The `args:` declaration advertises and documents the prompt's arguments to whoever runs it. A Host reads each declared arg's name, `type`, `optional` flag, `default`, and `description` from the parsed prompt, so a prompt that declares `name` and `times` lists exactly those two args.

### Declarations are never enforced

Declaration enforcement, checking a run's argument string against the `args:` declaration, is never done by Runvil at run time. A wrongly typed value or a missing required arg reaches the prompt unchanged, and the run does not fail. With `query` declared `type: string` and the argument string `{"query":5}`, the run succeeds and `tostring(argv.query)` returns `5`. Enforcement is the job of the Lua in the prompt's [H1 body](02-file-structure.md#the-h1-title-and-h1-body), as [Checking and repairing `argv` in the H1 pass](#checking-and-repairing-argv-in-the-h1-pass) shows.

### The implicit default declaration

A prompt whose frontmatter has no `args:` key gets the implicit default declaration: one optional `string` arg named `prose`, with no default and the description `Freeform input for this prompt`. Every prompt therefore has an args declaration, and there are no freeform prompts.

### The `args` and `argv` globals

Your Lua reads the run's raw argument string through the `args` global, with nothing to declare first. This prompt has no `args:` key and greets whatever it is given:

````markdown
---
name: greeter
description: Greets whoever the argument string names
runvil: 0
---

# Greeter

## Greet

```lua
return "Hello, " .. args .. "!"
```
````

Run it with the argument string `world`:

````text
Hello, world!
````

A block that does `return args` echoes the argument string back unchanged.

Every section's Lua also reads the `argv` global, the parsed form of the argument string, side by side with the raw `args`. Under the search prompt's declaration, the argument string `{"query":"papers"}` gives `argv.query == 'papers'`. In the greeter prompt, which has no `args:` key, the argument string `hello there` gives both `argv.prose == 'hello there'` and `args == 'hello there'` in the same section, so this block passes both checks and returns `hello there` when it replaces the block in `## Greet`:

````lua
assert(argv.prose == 'hello there')
assert(args == 'hello there')
return args
````

### Where `argv` can change

`argv` is writable only in the [H1 pass](04-how-a-prompt-runs.md#the-h1-pass), the run of the Lua in the H1 body under the `#` title before the [walk](04-how-a-prompt-runs.md#the-section-walk) starts. Everywhere else it is read-only at every nesting depth, so the H1 body is the only place a prompt can change `argv`, and its value cannot change after the H1 pass. An H1 body that runs `argv = { query = 'repaired' }` hands that table to every later section. In an H2 section, `argv = ...` and `argv.query = ...` both fail with an error naming `argv is frozen`, and so does a nested write such as `argv.nested.hits = 3`. [Freezing `argv` after the H1 pass](#freezing-argv-after-the-h1-pass) covers the freeze in full.

## Declaring arguments in frontmatter

You can declare each argument as a named entry under the `args:` key, starting with a single required argument like the one in the search prompt.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## Search

```lua
return 'Results for ' .. argv.query
```
````

Run it with the argument string `{"query":"papers"}`:

````text
Results for papers
````

### The smallest declaration

The smallest valid arg declaration is an arg name under `args:` with only a `type:` key, as `query:` with `type: string` under it is here. `type:` is the only key a declaration must have, and `optional:`, `default:`, and `description:` can all be left out.

A required argument is an arg whose declaration has no `optional:` key, because `optional:` reads `false` when it is absent. So `query` is a required string argument, one the declaration tells Hosts the argument string must carry. Like the rest of the declaration, this is advertised and never enforced at run time.

### Arg names

The `args:` key holds a set of named arg declarations, one entry per arg name. Each [arg name](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-local-tool-aliases) follows the frontmatter name grammar `[A-Za-z][A-Za-z0-9_-]{0,63}`: a letter, then up to 63 letters, digits, `_`, or `-`, so a name is 1 to 64 characters long. `query`, `use_mcp`, and `limit` all fit. A name outside the grammar, such as `'1bad'` in `args: { '1bad': { type: string } }`, fails to parse with parse error kind `Frontmatter` and a message containing this text, with the name in place of `{key}`:

````text
invalid arg name `{key}`: expected [A-Za-z][A-Za-z0-9_-]{0,63}
````

### The arg declaration map

Each `args.{name}` entry is an arg declaration map, a map of named keys. It must have `type:`, whose value is one of `string`, `boolean`, `integer`, or `number`, and it can add `optional:`, `default:`, and `description:`. Those four are its only keys, and a declaration with any other key fails to parse.

| Key | Value | When left out |
|---|---|---|
| `type` | `string`, `boolean`, `integer`, or `number` | required |
| `optional` | a YAML boolean | `false`, so the arg is required |
| `default` | a value matching `type` | no default |
| `description` | a YAML string | no description |

[Argument types](#argument-types) covers `type`, [Optional arguments, defaults, and descriptions](#optional-arguments-defaults-and-descriptions) covers the other three keys, and [Declaration errors](#declaration-errors) gives the message for each mistake.

### Flow style

The `args:` declaration can be written as a YAML flow map, following the rules in [Writing YAML in the frontmatter](02-file-structure.md#writing-yaml-in-the-frontmatter). This `args:` key declares the same single string argument named `query` as the block form in the search prompt:

````markdown
args: { query: { type: string } }
````

### The order of declared args

The declared args are kept in arg-name order, not in the order you write them. A prompt that writes `times` before `name` has its args ordered `name`, then `times`.

### All four keys together

One `args:` declaration can use all four keys across its args. This one gives `query` a description, makes `limit` optional with a default of `3`, and gives `use_mcp` a default and a description:

````markdown
args:
  query:
    type: string
    description: What to search for
  limit:
    type: integer
    optional: true
    default: 3
  use_mcp:
    type: boolean
    default: true
    description: Search MCP-connected private sources
````

An arg that leaves out `description:`, such as `limit` here or `limit: { type: integer, optional: true }`, has no description.

### Reserved names as arg names

An arg name can be a reserved name, such as the Engine globals `prose` and `store` or the Lua keyword `end`, as long as it fits the name grammar. Arg names become fields of `argv` and never globals, so this `args:` key parses with three args:

````markdown
args: { prose: { type: string }, store: { type: string }, end: { type: boolean } }
````

## Argument types

You can give each argument one of four argument types, each naming the kind of JSON value the argument string is expected to carry for that arg.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
  use_mcp:
    type: boolean
    default: true
    description: Search MCP-connected private sources
  limit:
    type: integer
    optional: true
  weight:
    type: number
    optional: true
---

# Paper search

## Search

```lua
return 'Results for ' .. argv.query
```
````

Run it with the argument string `{"query":"papers","use_mcp":false,"limit":5,"weight":0.5}`:

````text
Results for papers
````

Every arg you declare under `args:` must have a `type` key, written `args.{name}.type`, set to `string`, `boolean`, `integer`, or `number`, such as `type: boolean`. Unlike `optional`, `default`, and `description`, the `type` key has no default and cannot be left out.

The `type` key accepts exactly these four lowercase words, and each names the kind of JSON value the Host is expected to send for that arg in the argument string. Any other word makes the prompt fail to parse with a `Frontmatter` parse error, as [Declaration errors](#declaration-errors) shows.

| `type:` | Example declaration | Expected JSON value | Example argument string |
|---|---|---|---|
| `string` | `query: { type: string }` | a JSON string | `{"query":"papers"}` |
| `boolean` | `use_mcp: { type: boolean, default: true }` | JSON `true` or `false` | `{"use_mcp":false}` |
| `integer` | `limit: { type: integer, optional: true }` | a whole JSON number | `{"limit":5}` |
| `number` | `weight: { type: number, optional: true }` | any JSON number | `{"weight":0.5}` |

The type is an expectation the declaration advertises. The argument string is parsed as JSON the same way whatever the declared types are, and a value of another kind still reaches `argv` unchanged. [Reading parsed input from `argv`](#reading-parsed-input-from-argv) shows how each JSON value reads in Lua.

### String arguments

You write `type: string`, as in `query: { type: string }` or `name: { type: string }`, to declare an arg the Host is expected to send as a JSON string.

### Boolean arguments

You write `type: boolean`, as in `use_mcp: { type: boolean, default: true, description: Search MCP-connected private sources }`, to declare an arg the Host is expected to send as JSON `true` or `false`.

### Integer arguments

You write `type: integer`, as in `limit: { type: integer, optional: true }` or `times: { type: integer, optional: true, default: 1 }`, to declare an arg the Host is expected to send as a whole JSON number, with no fractional part.

### Number arguments

You write `type: number` to declare an arg the Host is expected to send as any JSON number, fractional values included. Its default can be any YAML number, such as `3` or `1.5`.

## Optional arguments, defaults, and descriptions

You can mark an argument optional, advertise a default for it, and describe it, and this section shows what each of those keys does at run time and what it does not.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
    description: What to search for
  limit:
    type: integer
    optional: true
    default: 3
    description: Most results to return
---

# Paper search

## Search

```lua
if argv.limit == nil then
  return 'Results for ' .. argv.query .. ', no limit sent'
end
return 'Results for ' .. argv.query .. ', limit ' .. argv.limit
```
````

Run it with the argument string `{"query":"papers","limit":5}`:

````text
Results for papers, limit 5
````

Run it with `{"query":"papers"}`, which leaves `limit` out:

````text
Results for papers, no limit sent
````

The second run shows that the declared `default: 3` is advertised to Hosts but never reaches `argv`.

### Marking an argument optional

An optional argument is one the argument string may leave out entirely. You mark an arg optional with the boolean key `args.{name}.optional`, written `optional: true`. With `args: { prose: { type: string, optional: true } }`, a run with the argument string `{}` still runs, and `optional` defaults to `false`.

The `optional:` key takes a YAML boolean and is `false` when left out, so an arg is required unless it is marked optional.

### Giving an argument a default

An arg default is the value a declaration advertises for an arg under `default:`. You set it with the `default:` key, written `args.{name}.default`, such as `default: 3` on an `integer` arg or `default: true` on a `boolean` arg, and the prompt keeps that value. An arg that leaves `default:` out has no default.

A `default:` value must match the arg's declared `type:`. A prompt whose default does not match fails to parse with a `Frontmatter` parse error, as with a quoted `'true'` on a `boolean` arg, `1.5` on an `integer` arg, a bare `42` on a `string` arg, or `[1]` on a `number` arg. [Defaults for each type](#defaults-for-each-type) gives the values each type takes, and [Declaration errors](#declaration-errors) gives each message.

You give an optional arg a `default:` in its declared type, as in `times: { type: integer, optional: true, default: 1 }`.

YAML quoting decides a default's type. A quoted value is always a string, so you quote a default only for a `string` arg and write `true`, `false`, and numbers bare. The quoted `'true'` is a string and fails on a `boolean` arg, and the bare `42` is an integer and fails on a `string` arg.

The run never fills in a declared default. The default is advertised to Hosts but never applied to `argv`, which is built only from the args the argument string carries, so a missing optional arg stays absent unless the argument string sends a value. To use the default in your Lua, set it yourself in the H1 body, as [Checking and repairing `argv` in the H1 pass](#checking-and-repairing-argv-in-the-h1-pass) shows.

### Describing an argument

An arg description is the text a declaration gives under `description:`. You add it with the optional string key `args.{name}.description`, as in `description: Search MCP-connected private sources`, and the text is kept exactly as written. An arg without `description:`, such as `limit: { type: integer, optional: true }`, has no description.

Hosts see each arg's `description` together with its `type`, `optional` flag, and `default`. The Workshop Run window's contract lists every declared arg with all four, as in this entry for a required `string` arg named `topic`:

````text
{ "name": "topic", "type": "string", "optional": false, "default": null, "description": "the topic" }
````

### How `optional` and `default` combine

Giving an arg a `default:` does not make it optional. Without `optional:`, the arg is required even when it has a default, as `use_mcp` is here:

````markdown
args:
  use_mcp:
    type: boolean
    default: true
    description: Search MCP-connected private sources
````

You can combine `optional` and `default` freely. A prompt may give a default to an arg that is not optional, and may mark an arg optional without giving it a default, as the implicit `prose` arg is optional with no default.

An omitted optional arg is absent from the argument string, while an empty string counts as a supplied value. With `prose` declared optional, the argument string `{}` leaves `argv.prose` as `nil`, while `{"prose":""}` gives `argv.prose` the value `''`. So to leave an optional arg unset, omit its field instead of sending `""`.

You write `default: null` to mean no default, and a `null` default is never checked against the arg's `type:`.

### Defaults for each type

Each type takes defaults of one YAML kind:

| `type:` | Valid defaults | Examples |
|---|---|---|
| `string` | a YAML string | `draft`, `'42'` |
| `boolean` | a YAML bool | `true`, `false` |
| `integer` | a whole number in 64-bit integer range | `3`, `-1` |
| `number` | any YAML number, whole or fractional | `3`, `1.5` |

A `string` arg takes a YAML string as its default, such as `draft` or the quoted `'42'`, while a bare `42` on a `string` arg fails with a message saying ``the default has type `integer` ``.

A `boolean` arg takes a YAML bool, `true` or `false`, as its default, as in `default: true`.

An `integer` arg takes a whole-number default in 64-bit integer range, such as `3` or `-1`, while a fractional default like `1.5` on an `integer` arg fails with a message saying ``the default has type `number` ``.

A `number` arg takes any YAML number as its default, whole or fractional, such as `3` or `1.5`.

A default is always one scalar value, a string, boolean, or number, since none of the four arg types takes a list or a map as its default. `default: [1]` on a `number` arg fails with a message saying ``the default has type `sequence` ``.

## Declaration errors

You can recognize each mistake an `args:` declaration can hold by the message it gives, starting from this prompt, which parses and runs cleanly.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  use_mcp:
    type: boolean
---

# Paper search

## Search

```lua
return 'ready'
```
````

Run it with the argument string `{"use_mcp":true}`:

````text
ready
````

Mistype the `type:` key as `tipe:` and the prompt fails to parse:

````markdown
args:
  use_mcp:
    tipe: boolean
````

````text
invalid frontmatter: args.use_mcp: unknown field `tipe`, expected one of `type`, `optional`, `default`, `description` at line 6 column 5
````

### What every declaration error shares

Every mistake in an `args:` declaration is a `Frontmatter` parse error. The prompt fails to parse, Runvil stops before it reads the prompt body, and the prompt never runs, as [Parsing comes first](04-how-a-prompt-runs.md#parsing-comes-first) describes. An unknown key, a default that does not match its type, a bad arg name, and an unknown type all fail this way.

A declaration error message starts with `invalid frontmatter: `, followed by a YAML path such as `args` or `args.use_mcp` and a colon, then the text that names the mistake, and it ends with a position that counts the line after the opening `---` as line 1, such as `at line 6 column 5`. The error also reports that position as a 1-based line and column in the prompt file, counting the opening `---` as line 1, so the `tipe:` mistake above, whose message ends `at line 6 column 5`, is reported as line 7, column 5. The rest of this section quotes the text that names each mistake.

### Errors in the `args:` map

The value of `args:` must be a map from arg names to declarations. Any other value is a `Frontmatter` parse error whose message names what was found in place of `{found}`:

````text
invalid type: {found}, expected a map of arg name keys to declarations
````

Each arg name appears only once in an `args:` map. Declaring the same name twice is a `Frontmatter` parse error, with the repeated name in place of `{name}`:

````text
duplicate arg name `{name}`: contract map keys must be unique
````

Each arg's value must itself be a map of declaration keys, even when its only key is `type:`, as in `use_mcp: { type: boolean }`. A bare scalar or a list in that place is a `Frontmatter` parse error.

### Errors in declaration keys

An arg declaration accepts only the keys `type`, `optional`, `default`, and `description`. Any other key, such as the typo `tipe:` in `args: { use_mcp: { tipe: boolean } }`, is a `Frontmatter` parse error rather than being silently ignored, and the message names the bad key in place of `{key}` and lists the allowed keys:

````text
unknown field `{key}`, expected one of `type`, `optional`, `default`, `description`
````

Every arg declaration must have a `type:` key. It is required and has no fallback, unlike `optional`, `default`, and `description`, which may be left out. Leaving `type:` out is a `Frontmatter` parse error whose message contains ``missing field `type` ``.

`type:` accepts only the four lowercase words `string`, `boolean`, `integer`, and `number`. Any other value, such as a capitalized `String`, an alias like `int`, or an unknown word like `text` in `args: { use_mcp: { type: text } }`, is a `Frontmatter` parse error whose message names the bad value in place of `{value}` and lists the four allowed words:

````text
unknown variant `{value}`, expected one of `string`, `boolean`, `integer`, `number`
````

`optional:` accepts only a bare YAML `true` or `false`, and may be left out. A quoted value such as `'true'` is always a YAML string, so `optional: 'true'` is a `Frontmatter` parse error.

### Default type mismatches

A default type mismatch is a `default:` whose YAML type differs from the arg's declared `type:`. A declared default must match the declared type, so a mismatch is a `Frontmatter` parse error whose message names both types in this text:

````text
the default has type `{yaml type}`, which does not match the declared type `{declared type}`
````

The message names the default's YAML type as one of `boolean`, `integer`, `number`, `string`, `sequence`, `mapping`, or `tagged value`, and names the declared type in lowercase as `string`, `boolean`, `integer`, or `number`. A YAML null `default:` counts as no default, so the message never names `null`.

An unquoted whole number such as `42` is a YAML `integer` and is not converted to text, so this declaration fails:

````markdown
args:
  query:
    type: string
    default: 42
````

````text
the default has type `integer`, which does not match the declared type `string`
````

Quoting it as `default: '42'` makes it a valid string default.

A default for an `integer`, `number`, or `boolean` arg is written bare, because a quoted value such as `'42'` or `'true'` is a YAML string, and only a `string` arg accepts a string default:

````markdown
args:
  use_mcp:
    type: boolean
    default: 'true'
````

````text
the default has type `string`, which does not match the declared type `boolean`
````

An `integer` arg's default is a whole number written without a decimal point, because a fractional value such as `1.5`, or even `3.0`, counts as a `number`:

````markdown
args:
  limit:
    type: integer
    default: 1.5
````

````text
the default has type `number`, which does not match the declared type `integer`
````

A `boolean` arg's default is a bare `true` or `false`. A number default on a `boolean` arg fails with the message naming `integer` or `number`, and a bare `true` or `false` default on a `string`, `integer`, or `number` arg fails with the message naming `boolean`.

A default is a single YAML scalar, so a list or map such as `[1]`, or a value with a custom YAML tag such as `!custom 3`, is rejected for every arg type, with the message naming the default's type as `sequence`, `mapping`, or `tagged value`:

````markdown
args:
  weight:
    type: number
    default: [1]
````

````text
the default has type `sequence`, which does not match the declared type `number`
````

## The argument string in `args`

You can read the exact argument string from the `args` global in any section, and this section shows when to reach for it instead of `argv`.

````markdown
---
name: echo
description: Returns its argument string unchanged
runvil: 0
---

# Echo

## Echo back

```lua
return args
```
````

Run it with the argument string `hello`:

````text
hello
````

### Where `args` is available

You can read the `args` global in the Lua of every section, including the H1 body at the top of the prompt, because the Engine installs `args` in each of them once section setup ends.

### The exact text, never parsed

The `args` global always holds the exact argument string, never parsed as JSON and never trimmed. Run with the argument string `  spaced { not json `, this check passes, leading spaces and the stray brace included:

````lua
assert(args == '  spaced { not json ', 'args is the exact passed string')
````

To parse the argument string yourself, call [`json.decode`](05-lua-environment.md#encoding-and-decoding-json-with-json) on it, as in `json.decode(args)`, inside [`pcall`](05-lua-environment.md#raising-and-catching-errors) when it may not be valid JSON, because malformed text raises an error. A JSON `null` decodes to `json.null`, not nil.

### `args` is an ordinary Lua string

`args` is an ordinary Lua string, so you can return it, join it with other text using `..`, pass it to a function, and keep it in a [`var`](05-lua-environment.md#keeping-state-in-var) field. The echo prompt above returns it, and this prompt does the other three:

````markdown
---
name: greeter
description: Greets whoever the argument string names
runvil: 0
---

# Greeter

## Greet

```lua
local function decorate(value)
  return '<' .. value .. '>'
end
var.greeting = 'hi ' .. args
var.answer = args
return decorate(args) .. ' ' .. var.greeting
```
````

Run it with the argument string `bob`. `var.greeting` holds `hi bob`, `var.answer` holds `bob`, and the run ends with:

````text
<bob> hi bob
````

### Ending a run with `args`

A block that does `return args` gets back the argument string exactly as it was passed, so the run ends with that text unchanged as its [run result](04-how-a-prompt-runs.md#the-run-result-and-outcomes). The echo prompt above, run with `hello`, returns `hello`, and the shipped `echo` prompt likewise ends with whatever text it is given. A [prologue](03-blocks-and-prose.md#prologue-prose-and-epilog) can return it before the section's prose:

````markdown
---
name: early
description: Stops in the first prologue
runvil: 0
---

# Early stop

## Stop Early

```lua
return args
```

This prose follows the prologue.
````

Run it with the argument string `early result`:

````text
early result
````

### Reading `args` from a library

A library section loaded with [`require`](03-blocks-and-prose.md#sharing-code-with-require) can read `args` both while it loads and later from a library function a section calls. A library loads into the requiring section's environment with every Engine global installed, so its top-level code may use `args` at load, and so may the functions it defines:

````markdown
---
name: reader
description: Reads the argument string through a library
runvil: 0
---

# Library reader

## Result

```lua
require '## helpers'
return read_args()
```

## helpers

```lua
function read_args()
  return args
end
```
````

Run it with the argument string `later canned value`:

````text
later canned value
````

### When to read `args` instead of `argv`

Read `args` when you need the argument string exactly as the Host sent it: to echo it, to build text from it, or to rebuild `argv` in the H1 body when the argument string is not valid JSON, as [Checking and repairing `argv` in the H1 pass](#checking-and-repairing-argv-in-the-h1-pass) shows. Read `argv` when you want the parsed fields of a declared argument. In a prompt with no `args:` key, `args` and `argv.prose` hold the same string, as [Prompts with no `args:` key](#prompts-with-no-args-key) explains.

## Reading parsed input from `argv`

You can read each field of a JSON argument string from `argv`, then handle malformed input, scalar values, and missing fields.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
  limit:
    type: integer
    optional: true
---

# Paper search

## Search

```lua
return argv.query
```
````

Run it with the argument string `{"query": "papers", "limit": 5}`:

````text
papers
````

### Structured declarations

A structured declaration is any explicit `args:` key in the frontmatter. Under a structured declaration, the argument string is parsed as JSON and each field arrives in `argv`, so a prompt run with `{ "name": "world" }` reads `argv.name` as `world`.

### Objects and fields

A JSON object in the argument string becomes an `argv` table with string keys named after the object's fields, so you read a field with dot access. The argument string `{"query": "papers", "limit": 5}` gives `argv.query` the value `papers` and `argv.limit` the value `5`.

Reading a field the argument string does not carry, such as `argv.absent`, returns nil instead of raising an error.

Outside the H1 body, you read nested fields of `argv` with ordinary dotted access at any depth, because a nested JSON object arrives as a nested table. The argument string `{ "nested": { "hits": 2 } }` gives `argv.nested.hits` the value `2`.

### Arrays

A JSON array in the argument string becomes an `argv` table with 1-based indexes, so the first element of a top-level array is `argv[1]`, and the first element of an `items` field is `argv.items[1]`.

You read an `argv` array with `ipairs` or numeric indexing. `ipairs` is the standard Lua 5.5 one and reads through to the stored elements, so this prompt joins the `items` field:

````markdown
---
name: listing
description: Lists the items found for a query
runvil: 0
args:
  query:
    type: string
---

# Item listing

## List

```lua
local names = {}
for i, name in ipairs(argv.items) do
  names[i] = name
end
return argv.query .. ': ' .. table.concat(names, ', ')
```
````

Run it with the argument string `{"query":"papers","items":["a","b"]}`:

````text
papers: a, b
````

Outside the H1 body, the length operator `#`, `pairs`, and `next` see an empty table on any `argv` table, because each frozen table is an empty stand-in that forwards reads to the data. So `#argv.items` is `0` in the `## List` section above, and `ipairs` or numeric indexing is how you read an array there. Inside the H1 body, `argv` is a plain table, and all three work normally.

### Strings, numbers, and booleans

JSON strings, numbers, and booleans in the argument string reach you as plain Lua strings, numbers, and booleans, outside the H1 body as well as inside it.

### Malformed input

Malformed input is an argument string that is not valid JSON, or the JSON `null`, under an explicit `args:` declaration. It makes `argv` nil in the H1 body and in every section, and the run goes on because nothing refuses it, so the prompt detects malformed input by testing `if argv then` or `argv == nil`:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## Search

```lua
if argv == nil then
  return 'no parsed arguments'
end
return 'Results for ' .. argv.query
```
````

Run it with the argument string `not json`:

````text
no parsed arguments
````

Under a structured declaration, the argument string is parsed as JSON into `argv` and never wrapped into `argv.prose`, even when the `args:` key declares the same single `prose` arg the implicit default declaration would give. So plain text that a prompt with no `args:` key would wrap, such as `hello there`, leaves `argv` nil.

Under an explicit `args:` declaration, a run with no arguments gets a nil `argv`, because the empty string is not JSON. A prompt with no `args:` key gets `argv.prose == ''` instead.

### Scalar `argv`

A scalar `argv` is an `argv` that holds a plain Lua value instead of a table. A bare JSON number, boolean, or string as the whole argument string makes `argv` that value, read directly. The argument string `5` gives `argv == 5`, and `tostring(argv)` returns `"5"`. The argument string `42` gives `tostring(argv)` the value `"42"`, and the argument string `true` makes `argv` the boolean `true`, which `tostring(argv)` renders as `"true"`.

A JSON integer in the argument string reads as a Lua integer, so `tostring` renders it with no decimal part: as `42` for the argument string `42`, and as `5` for `tostring(argv.query)` under the argument string `{"query":5}`.

### Missing fields and null values

An omitted optional arg reads as nil in `argv`, while one passed as an empty string reads as `""`, and Lua tells the two apart:

````markdown
---
name: notes
description: Reports whether prose was sent
runvil: 0
args:
  prose:
    type: string
    optional: true
---

# Prose check

## Check

```lua
if argv.prose == nil then return 'absent' end
assert(argv.prose == '')
return 'empty'
```
````

Run it with the argument string `{}`:

````text
absent
````

Run it with the argument string `{"prose":""}`:

````text
empty
````

A JSON `null` nested inside the argument string reads as [`json.null`](05-lua-environment.md#null-values-and-jsonnull) in every section, the H1 body included, and a `null` element inside a JSON array keeps its index. So for the argument string `{"note":null,"ids":[1,null,3]}`, both `argv.note == json.null` and `argv.ids[2] == json.null` are true, `json.encode(argv)` gives back `{"ids":[1,null,3],"note":null}`, and a field the string leaves out still reads nil. `json.null` is a userdata and so it is truthy, which means `if argv.note then` passes for a null. Test for a null with `argv.note == json.null` and for an absent field with `argv.note == nil`. A whole argument string that is the JSON `null` is different: it makes `argv` itself nil, as [Malformed input](#malformed-input) describes.

## Prompts with no `args:` key

You can leave the `args:` key out, and the prompt then takes its whole argument string as one optional `prose` argument that you read as `argv.prose`.

````markdown
---
name: echo
description: Returns its argument string unchanged
runvil: 0
---

# Echo

## Echo back

```lua
return argv.prose
```
````

Run it with the argument string `hello there`:

````text
hello there
````

### The default declaration in full

A prompt whose frontmatter has no `args:` key gets the implicit default declaration, which is one optional `string` arg named `prose`, with no `default:` value and the description `Freeform input for this prompt`. There are no freeform prompts: leaving out the optional `args:` key simply means "the default argument declaration".

| Part | Value |
|---|---|
| arg name | `prose` |
| `type` | `string` |
| `optional` | `true` |
| `default` | none |
| `description` | `Freeform input for this prompt` |

### How `argv.prose` reads

In a prompt with no `args:` key, the argument string arrives wrapped as `argv = { prose = "<text>" }`, so Lua reads it as `argv.prose`. Run with `hello there`, this check passes:

````lua
assert(argv and argv.prose == 'hello there', 'prose wraps into argv.prose')
````

Under the default declaration, `argv` is `{ prose = args }`, so `argv.prose` holds the whole argument string unchanged and never parsed as JSON. You can hand it to any function, including one from a library:

````markdown
---
name: research
description: Searches every source for one topic
runvil: 0
---

# Research

## Search

```lua
require '## helpers'
return search_all(argv.prose)
```

## helpers

```lua
function search_all(topic)
  return 'searched ' .. topic .. ' everywhere'
end
```
````

Run it with the argument string `rust`:

````text
searched rust everywhere
````

### `args` beside `argv.prose`

With no `args:` key, the `args` global still holds the run's exact argument string alongside `argv.prose`, and you use it directly. Run with `hello there`, `assert(args == 'hello there', 'args holds the exact passed string')` passes, `return args` returns the argument string verbatim, and `"Hello, " .. args .. "!"` builds a greeting from it. A helper can take it too:

````markdown
---
name: greeter
description: Greets whoever the argument string names
runvil: 0
---

# Greeter

## Greet

```lua
local function normalize(text)
  return string.lower(text)
end
var.subject = normalize(args)
return "Hello, " .. var.subject .. "!"
```
````

Run it with the argument string `World`:

````text
Hello, world!
````

### The argument string in prose

You can reference `{{ args }}` in prose in a prompt whose frontmatter has no `args:` key, whatever other keys that frontmatter sets. A [placeholder](07-substitution.md#placeholders-in-prose) can also read the same string as `{{ argv.prose }}`, which renders the same string as `{{ args }}`, because a dotted `{{ argv.<field> }}` placeholder reads that field of the run's `argv`, the same way `{{ argv.query }}` in `Field: {{ argv.query }}` renders `papers` when `argv.query` is `papers`.

````markdown
---
name: brief
description: Restates its topic
runvil: 0
---

# Brief

## Topic

Topic: {{ args }}. Same topic: {{ argv.prose }}.

```lua
return prose
```
````

Run it with the argument string `rust`:

````text
Topic: rust. Same topic: rust.
````

### Empty argument strings

Under the default declaration, an empty argument string arrives as a present empty string, so `argv` is not nil, `argv.prose == ''` rather than nil, and `args` is the empty string. A run with no arguments passes all three of these checks, which tell an empty argument string apart from an absent field:

````lua
assert(argv, 'empty prose still wraps')
assert(argv.prose ~= nil, 'the field is present')
assert(argv.prose == '', 'the field is the empty string')
````

A prompt with no `args:` key never gets a nil `argv` from its argument string, because the default declaration always wraps the string into `argv.prose`.

### Explicit `args:` keys never wrap

Any explicit `args:` key is a structured declaration that turns wrapping off and never puts the argument string into `argv.prose`, even one that copies the default exactly, because a declared `args:` is never the default. This prompt declares the same `prose` arg the default would give:

````markdown
---
name: echo
description: Returns its argument string unchanged
runvil: 0
args:
  prose:
    type: string
    optional: true
    description: Freeform input for this prompt
---

# Echo

## Echo back

```lua
if argv == nil then
  return 'no parsed arguments'
end
return argv.prose
```
````

Run it with the argument string `hello there`, which is not JSON:

````text
no parsed arguments
````

An explicit empty map, `args: {}`, declares zero arguments and is still a structured declaration, not the `prose` default, because the default always holds `prose`. Under `args: {}`, the argument string is parsed as JSON like under any other structured declaration, so an argument string that fails to parse, or the JSON `null`, gives a nil `argv`.

### Argument strings for calls and tasks

The same `argv.prose` wrapping applies to the explicit argument string given to a [`call`](08-jump-and-call.md#calling-a-section) or to a task started with [`tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn), because that chain's `argv` is derived again from its own argument string under the default declaration. A fanout [arm](14-fanout.md#what-an-arm-sees) gets no argument string of its own and reads its caller's `args` and `argv` unchanged, as [Freezing `argv` after the H1 pass](#freezing-argv-after-the-h1-pass) explains.

## Checking and repairing `argv` in the H1 pass

You can check `argv` in the H1 body and repair or complete it before the walk starts, so every section sees arguments in the shape it expects.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

```lua
assert(argv and argv.query, 'query is required')
```

## Search

```lua
return 'Results for ' .. argv.query
```
````

Run it with the argument string `{"query":"papers"}`:

````text
Results for papers
````

Run it with the argument string `{}`, and the check in the H1 body fails the run with an error carrying the message `query is required`, so `## Search` never runs.

### Nothing checks values against the declaration

Nothing checks passed values against the `args:` declaration. A value whose shape disagrees with its declared type, such as `{"query":5}` for a `query` arg declared `type: string`, or `"five"` for an `integer` arg, does not fail the run, and `argv` holds the value exactly as passed. Under `{"query":5}`, `return tostring(argv.query)` gives `5`, and the prompt decides what a bad value means.

### Checking the arguments yourself

Because the declaration advertises and documents the arguments but never enforces them, you check their shape and types in the prompt itself, usually in the Lua of the H1 body under the `#` title. A check such as `assert(argv and argv.query, 'query is required')` fails the run when the argument string is `{}`.

The H1 pass runs before the walk, so a failing check stops the run with an error before any `##` section runs. In the prompt above, run with `{}`, the `## Search` section never runs.

### `argv` is writable in the H1 body

In the H1 pass, `argv` is an ordinary writable global with no guard in the way, so you can assign `argv` itself and its fields without error. It is also a plain table there, so `#`, `pairs`, `next`, and `ipairs` work on it normally.

### Repairing malformed input

When the argument string is malformed input, such as `broken json`, `argv` reads nil in the H1 body, so you detect the problem there and supply a repaired value, as in `if argv == nil then argv = { query = 'repaired' } end`.

The `argv` repair is replacing the parsed arguments in the H1 body by assigning a whole new table to the `argv` global, such as `argv = { query = 'repaired' }`. After it, `argv.query` reads `repaired`, and the arguments the H1 pass leaves behind are `{ "query": "repaired" }`.

### Editing fields in place

In the H1 body you can also edit `argv` in place, overwriting a field such as `argv.query = 'fixed'` or adding one such as `argv.extra = 1`. This works whether `argv` holds the parsed arguments or a table you just assigned, and the edits stay in the value the H1 pass leaves behind. The argument string `{ "query": "broken" }` with `argv.query = 'fixed'` reads back as `{ "query": "fixed" }`, and `argv.extra = 1` after `argv = { query = 'repaired' }` reads back as `{ "query": "repaired", "extra": 1 }`.

### Repairing from the raw argument string

The raw argument string is already in the `args` global beside the writable `argv`, so a repair can read it and assign `argv` a fixed-up value. This prompt turns an argument string that does not parse into a query:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

```lua
if not argv then
  argv = { query = args }
end
```

## Search

Query: {{ argv.query }}

```lua
return prose
```
````

Run it with the argument string `broken json`:

````text
Query: broken json
````

### Filling in a declared default

A declared `default:` never reaches `argv` when the argument string leaves the arg out, so you fill it in in the H1 body. For a `limit` arg declared `default: 3`, this H1 body first makes sure `argv` is a table, then supplies the default:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
  limit:
    type: integer
    optional: true
    default: 3
---

# Paper search

```lua
if type(argv) ~= 'table' then
  argv = { query = args }
end
if argv.limit == nil then
  argv.limit = 3
end
```

## Search

```lua
return 'Results for ' .. argv.query .. ', limit ' .. argv.limit
```
````

Run it with the argument string `{"query":"papers"}`:

````text
Results for papers, limit 3
````

### Clearing `argv`

In the H1 body, you clear the arguments with `argv = nil`, after which `argv == nil` is true.

### Keeping `argv` as JSON data

In the H1 body, keep `argv` as [JSON data](05-lua-environment.md#keeping-state-in-var) or nil, meaning strings, numbers, booleans, and tables of them. The assignment itself accepts any value, so `argv = function() end` raises no error at that line, but a value that is not JSON data, such as a function, userdata other than `json.null`, or thread, fails the run once the H1 body finishes, with this message naming its Lua type in place of `{type}`:

````text
argv must be JSON data, got {type}
````

The one userdata the freeze accepts is `json.null`, which freezes as JSON null, and later sections read it as `json.null` again, like any other JSON `null` in the arguments. So after an H1 body runs `argv = { a = json.null }`, a later section reads `argv.a` as `json.null`, and `argv.a == json.null` is true.

### Your own `_G` metatable in the H1 pass

In the H1 pass, a metatable you set on `_G`, as [Giving `_G` a metatable](05-lua-environment.md#giving-_g-a-metatable) describes, never sees `argv`. A nil `argv` reads as nil even under a strict `__index` that errors on undefined globals, and a repair such as `argv = repaired` lands in `_G` without reaching a `__newindex` hook, so the walk still reads the repaired value.

### Fields the declaration does not list

Fields beyond the declared args, including arrays and nested objects, arrive in `argv` as passed, because the declaration is never enforced. With `query` as the only declared arg, the argument string `{"query":"papers","n":2}` renders through `{{ argv }}` as `{"n":2,"query":"papers"}`, and the `listing` prompt in [Reading parsed input from `argv`](#reading-parsed-input-from-argv) declares only `query` yet reads `argv.items` with `ipairs`.

## Freezing `argv` after the H1 pass

You can rely on `argv` holding the same value in every section after the H1 pass, because the Engine freezes it when the H1 pass ends.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

```lua
if argv == nil then
  argv = { query = 'repaired' }
end
```

## First

```lua
assert(argv.query == 'repaired', 'the repair reaches the first section')
```

## Second

```lua
assert(argv.query == 'repaired', 'the repair reaches every downstream section')
return argv.query
```
````

Run it with the argument string `broken json`:

````text
repaired
````

### What the freeze reads back

The `argv` freeze happens when the H1 pass completes, before the walk starts: the Engine reads the value the H1 body left in `argv` back as JSON and freezes it for the rest of the run. So a repair the H1 body assigns to `argv` becomes the `argv` later sections see.

Whatever `argv` holds when the H1 pass finishes, the parsed arguments or a repair, is the `argv` every walked section reads, both in Lua and in `{{ argv }}` and `{{ argv.field }}` placeholders. In the prompt above, `## First` and every section after it read `argv.query == 'repaired'`.

When the H1 body leaves `argv` untouched, later sections see the `argv` parsed from the argument string. An H1 body that only runs `require '## strict'` and never assigns `argv`, run with the argument string `{"query":"x"}`, leaves a later section reading `argv.query == 'x'`.

Outside the H1 pass, `argv` is never a raw global, and reading it returns its frozen value, which on the walk is the value the H1 body left behind. `argv` is frozen in every walked section after the H1 pass ends.

### Writes after the freeze

Outside the H1 pass, assigning `argv` raises an error. This block in a walked section catches it and returns the error text:

````lua
local ok, err = pcall(function() argv = 'hijacked' end)
return tostring(err)
````

The error contains this message:

````text
argv is frozen outside H1: assign it in H1 only
````

Writable `argv` is the H1 body's one privilege. Every other section gets `argv` read-only, including every section on the walk, in a called chain, in a task started with `tasks.spawn`, and in a fanout arm. In a called section, `local wrote = pcall(function() argv.query = 'x' end)` leaves `wrote` false.

### Non-JSON values at the freeze

Leave `argv` as JSON data, meaning strings, numbers, booleans, and tables of them, or as nil, when the H1 body finishes, because a function, userdata other than `json.null`, or thread is refused at the freeze. The run fails with a `Lua` error when the H1 pass leaves `argv` holding data that is not JSON.

Leaving `argv` as a function, userdata other than `json.null`, or thread when the H1 pass ends, as after `argv = function() end`, fails the freeze read-back with a run error of [run error kind](16-limits-and-errors.md#run-error-kinds) `Lua`, whose message names the Lua type `function`, `userdata`, or `thread` in place of `{type}`:

````text
argv must be JSON data, got {type}
````

The kind is `Lua` even though the failure happens at the end of the H1 pass. Leaving `argv` as data that cannot be represented as JSON, such as a table holding a value that cannot be JSON, also fails the read-back with a `Lua` error.

### When the H1 pass leaves `argv` nil

Leaving `argv` nil at the end of the H1 pass means the run has no `argv` value. Outside the H1 pass, `argv` then reads nil, so `assert(argv == nil, 'no argv reads nil')` holds there.

NilArgv is the substitution kind a placeholder rooted at `argv` fails with when the run has no `argv` value. Placeholders in walked sections read the run's frozen `argv`, so when the H1 pass leaves `argv` nil, `{{ argv }}` and `{{ argv.field }}` in later sections fail with `NilArgv` when a block reads them through `prose`. This prompt has no H1 repair:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## Show

Value: {{ argv }}

```lua
return prose
```
````

Run with the argument string `not json`, it fails the run with an error naming `argv`. [Substitution error kinds](07-substitution.md#substitution-error-kinds) lists the other substitution kinds.

### Placeholders in the H1 body

Inside the H1 body, `{{ argv }}` and `{{ argv.field }}` placeholders render the `argv` parsed from the argument string, never a repair the H1 Lua assigned. So when the argument string does not parse, they fail with `NilArgv` even after the repair, and the repair reaches placeholders from the first walked section on.

### `argv` in calls, tasks, and arms

A `call` or `tasks.spawn` given its own argument string gets a fresh `argv` parsed from that string and frozen in its chain, while one given no argument string inherits the caller's `args` and frozen `argv` whole, an H1 repair included:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

```lua
if argv == nil then
  argv = { query = 'repaired' }
end
```

## Main

```lua
local own = call('## Sub', '{"query":"chain"}')
local inherited = call('## Sub')
return own .. ' ' .. inherited
```

## Sub

```lua
return argv.query
```
````

Run it with the argument string `broken json`. The first `call` gives `## Sub` the `argv.query` value `chain`, and the second gives it the repaired `argv.query` value `repaired`:

````text
chain repaired
````

In `## Sub`, a write to a field of `argv` still fails under either `call`, because the chain's `argv` is frozen.

Every arm reads its caller's `args` and frozen `argv` unchanged, including an `argv` repaired in the H1 pass, because `tasks.fanout` passes no argument string of its own to the arms it spawns.

## Freeze errors and guard mechanics

You can tell each freeze error apart by its message, catch it with `pcall`, and know which operations on a frozen `argv` still work.

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## Search

```lua
local wrote = pcall(function() argv.mode = 'x' end)
if wrote then return 'written' end
return 'refused'
```
````

Run it with the argument string `{"query":"papers"}`:

````text
refused
````

A freeze error is the Lua error a write to a frozen `argv` raises. Each one is listed below, and the rest of this section explains them:

| Write outside the H1 pass | Message |
|---|---|
| `argv = {}`, `argv = nil`, `argv = 'hijacked'`, `argv = 6` | `argv is frozen outside H1: assign it in H1 only` |
| `argv.query = 1` | `argv is frozen outside H1: cannot set field 'query'` |
| `argv[1] = 1` | `argv is frozen outside H1: cannot set field 1` |
| `argv[2.5] = 1` | `argv is frozen outside H1: cannot set field number` |
| `argv.a.b = 1` | `argv is frozen outside H1: cannot set field 'b'` |
| `setmetatable` on any `argv` table | `cannot change a protected metatable`, after the calling position |

### `argv` in every section

Every section and the H1 pass has an `argv` global that the Engine installs, guarded rather than a plain global outside the H1 pass. `argv` is there even when the run has no parsed arguments, in which case it holds nil.

Every section's Lua gets `argv` when the Engine sets up the section, in one of two modes depending on whether the section is the H1 pass: writable in the H1 pass and frozen in every other section.

### The guard proxy

Outside the H1 pass, `argv` is not a plain entry of `_G`. The frozen value sits behind the guard proxy on `_G`, and every read or write of the name `argv` goes through that guard.

Outside the H1 pass, each `argv` table is a read-only proxy whose reads pass straight through to the parsed data. So `argv.query == 'papers'` and `argv.nested.hits == 2` read the data as given, and an absent field such as `argv.absent` reads nil.

### Writing to a frozen `argv`

Outside the H1 pass, such as in an H2 section, writing a top-level field of `argv`, as in `argv.query = 'hijacked'`, raises an error whose message names the freeze with `argv is frozen`.

Assigning the `argv` global itself in any section other than the H1 pass, with any value, as in `argv = {}`, `argv = nil`, `argv = 'hijacked'`, or `argv = { query = 'hijacked' }`, raises a Lua runtime error with the Engine's refusal message:

````text
argv is frozen outside H1: assign it in H1 only
````

The refusal comes before the value is looked at, and the same refusal happens in a later section and in a later fence. It happens even when `argv` is nil, because a nil `argv` is frozen too, so `argv = {}` on a nil `argv` fails with `argv is frozen outside H1: assign it in H1 only`. Reassigning an `argv` that holds a scalar, as in `argv = 6`, raises the freeze error `argv is frozen` as well.

### Field write messages

Writing a string-keyed field of `argv` outside the H1 pass raises a Lua error with the message `argv is frozen outside H1: cannot set field 'name'`, with the field name in single quotes. So `argv.query = 1` gives `argv is frozen outside H1: cannot set field 'query'`, and `argv.mode = "x"` gives `argv is frozen outside H1: cannot set field 'mode'`.

Writing an integer key of `argv` outside the H1 pass, such as an array index, raises the same field write error with the integer shown without quotes, so `argv[1] = 1` gives `argv is frozen outside H1: cannot set field 1`.

A write with a key that is neither a string nor an integer, such as a fractional number or a boolean, raises a refusal whose first line names the key's Lua type instead of its value, so `argv[2.5] = 1` gives `argv is frozen outside H1: cannot set field number`.

Adding a new field to an `argv` table raises the same field write error as changing an existing field. On an `argv` holding only `query`, the new keys `argv[1]` and `argv[2.5]` get the same `argv is frozen outside H1: cannot set field ...` refusal as `argv.query`.

### Nested writes

The freeze is deep: nested tables inside `argv` are frozen too, so a write at any depth, such as `argv.a.b = 1` or `argv.nested.hits = 3`, raises the freeze error. A nested field write names only the innermost key in its message, so `argv.a.b = 1` reports `cannot set field 'b'`, and `argv.opts.depth = 2` names `'depth'`.

Writing a field of `argv` at any depth outside the H1 body, including a new field, is a Lua runtime error whose message names the field written in place of `{field}`:

````text
argv is frozen outside H1: cannot set field {field}
````

### Uncaught freeze errors

An uncaught freeze error ends the run with the [run error kind](16-limits-and-errors.md#run-error-kinds) `Lua`, the same kind as any other runtime error in a section's Lua.

### Catching freeze errors with `pcall`

The frozen-`argv` refusal is an ordinary Lua error, so you catch it with [`pcall`](05-lua-environment.md#raising-and-catching-errors). `tostring(err)` on the caught value gives text that starts with `runtime error: ` and has the refusal on its first line, so this block returns the refusal text:

````lua
local ok, err = pcall(function() argv.query = 1 end)
return tostring(err)
````

The prompt at the top of this section uses the same pattern with a flag instead: `local wrote = pcall(function() argv.mode = 'x' end)` followed by `if wrote then return 'written' end return 'refused'` returns `refused`.

The `argv` of a called section is frozen, so writing one of its fields fails and `pcall` catches the failure:

````lua
local wrote = pcall(function() argv.query = 'x' end)
assert(not wrote, 'the chain argv is frozen')
````

### Your own `_G` metatable outside the H1 pass

Outside the H1 pass, a metatable you set on `_G` cannot lift the freeze and never sees an `argv` read or assignment, because the guard on `_G` handles that name first. After `setmetatable(_G, nil)`, `setmetatable(_G, {})`, `setmetatable(_G, mt)`, an `__newindex` of yours that captures writes, an `__index` that returns `'author'`, or any change to the table `getmetatable(_G)` returns, `argv.query` still reads the frozen value, such as `papers`, and assigning `argv` still raises the freeze error.

### Metatables, length, and iteration on frozen tables

Outside the H1 pass, each frozen `argv` table carries the protection text `argv is frozen`, separate from the field write message `argv is frozen outside H1: cannot set field {field}`. So `getmetatable(argv)`, or `getmetatable` on any `argv` table, returns the string `"argv is frozen"` instead of its real metatable.

Outside the H1 body, the length operator `#`, `pairs`, and `next` treat an `argv` table as empty, because each frozen table is an empty stand-in that forwards reads to the data.

Outside the H1 body, `setmetatable` on any `argv` table raises `cannot change a protected metatable`, prefixed with the calling position, because the table's metatable is protected. It is the same error `setmetatable` gives on the protected `sys` and `var` tables.

### Everything else stays writable

The `argv` freeze guards only the name `argv`. In any section, including outside the H1 pass, you still define, read, and assign every other global normally, so `scratch = 42` followed by `assert(scratch == 42, 'a bare global still assigns')` works while `argv` is frozen.

You keep using the [`prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global), the rendered Markdown above a fence, normally beside the frozen `argv`. In this prompt, `prose` reads as the Engine set it in the same fence where assigning `argv` is refused:

````markdown
---
name: search
description: Searches for papers on one query
runvil: 0
args:
  query:
    type: string
---

# Paper search

## First

Tone: friendly.

```lua
assert(prose == 'Tone: friendly.')
local ok = pcall(function() argv = 'hijacked' end)
assert(not ok)
return argv.query .. ' / ' .. prose
```
````

Run it with the argument string `{"query":"papers"}`:

````text
papers / Tone: friendly.
````
