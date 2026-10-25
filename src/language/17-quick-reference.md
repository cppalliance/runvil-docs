# Quick reference

This page lists every name a prompt author can write, grouped by kind, with a link to the section that teaches each one. Rows are alphabetical, except the top-level frontmatter keys, which follow the order the file structure chapter teaches them.

## Frontmatter keys

Frontmatter is YAML between two `---` delimiter lines at the very top of the file, a leading byte-order mark is dropped, and any key not listed here fails the parse with parse error kind `Frontmatter`.

| Key | Value | Required | Default | Taught in |
|---|---|---|---|---|
| `name` | string naming the prompt | yes | none | [The required `name` and `description` keys](02-file-structure.md#the-required-name-and-description-keys) |
| `description` | string summarizing what the prompt does | yes | none | [The required `name` and `description` keys](02-file-structure.md#the-required-name-and-description-keys) |
| `runvil` | Engine major, a whole number from 0 to 4294967295, and only `0` runs | no | none | [The `runvil` version key](02-file-structure.md#the-runvil-version-key) |
| `max_tool_iterations` | round cap, a whole number from 1 to 1000 | no | 24 unless the Host or Harness sets another | [Tool rounds and the round cap](11-conversations.md#tool-rounds-and-the-round-cap) |
| `input` | file declaration map of `path` and `description` | no | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `input.description` | string giving the input file's purpose | yes | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `input.path` | store path of the input file, such as `paper.md` | yes | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `output` | file declaration map of `path` and `description` | no | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `output.description` | string giving the output file's purpose | yes | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `output.path` | store path of the output file, such as `report.md` | yes | none | [Input and output file declarations](02-file-structure.md#input-and-output-file-declarations) |
| `uses` | list of Pack ids or `vendor/name` entries, such as `[web, acme/poop]`, each Pack id named once | no | none | [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key) |
| `args` | map of arg name to arg declaration, each name once and matching `[A-Za-z][A-Za-z0-9_-]{0,63}` | no | one optional `string` arg named `prose` | [The name grammar for role labels, arg names, and adhoc tool names](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-adhoc-tool-names) |
| `args.<name>.default` | value whose YAML kind matches `type`, or `null` for none | no | none | [Optional arguments, defaults, and descriptions](06-arguments.md#optional-arguments-defaults-and-descriptions) |
| `args.<name>.description` | string, kept as written | no | none | [Optional arguments, defaults, and descriptions](06-arguments.md#optional-arguments-defaults-and-descriptions) |
| `args.<name>.optional` | boolean | no | `false` | [Optional arguments, defaults, and descriptions](06-arguments.md#optional-arguments-defaults-and-descriptions) |
| `args.<name>.type` | `string`, `boolean`, `integer`, or `number` | yes | none | [Argument types](06-arguments.md#argument-types) |
| `models` | map of role label to role declaration, each label once and matching `[A-Za-z][A-Za-z0-9_-]{0,63}` | no | none | [The name grammar for role labels, arg names, and adhoc tool names](02-file-structure.md#the-name-grammar-for-role-labels-arg-names-and-adhoc-tool-names) |
| `models.<label>.description` | free prose, kept as written | no | the bound model's own description | [Declaring roles under `models:`](10-models.md#declaring-roles-under-models) |
| `models.<label>.keywords` | list from hard `thinking`, `no-thinking` and soft `frontier`, `fast`, `small`, `creative`, `chat` | no | empty list | [Declaring roles under `models:`](10-models.md#declaring-roles-under-models) |
| `models.<label>.min_context` | whole number of tokens from 1 to 4294967295 | no | none | [Declaring roles under `models:`](10-models.md#declaring-roles-under-models) |

## Lua globals and members

Every global, function, field, and error kind a Lua block can use, plus the standard names the sandbox removes or never loads.

| Name | Kind | Summary | Taught in |
| --- | --- | --- | --- |
| `_G` | global | The global table | [The sandbox](05-lua-environment.md#the-sandbox) |
| `_VERSION` | global | Holds the Lua version string | [The sandbox](05-lua-environment.md#the-sandbox) |
| `args` | global | Holds the run's exact argument string, never parsed or trimmed | [The argument string in `args`](06-arguments.md#the-argument-string-in-args) |
| `argv` | global | Parsed argument string; writable only in the H1 pass, frozen elsewhere | [Arguments at a glance](06-arguments.md#arguments-at-a-glance) |
| `assert` | function | Checks a condition, raising its message when false | [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors) |
| `call` | function | Runs a contained chain at `target`; optional `input` string replaces `args` | [Jump and call at a glance](08-jump-and-call.md#jump-and-call-at-a-glance) |
| `collectgarbage` | removed | Reads nil; no garbage collector control | [The sandbox](05-lua-environment.md#the-sandbox) |
| `compactors` | global | Table of shipped compactor policies, holding `fail` and `clear` | [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion) |
| `compactors.clear` | function | Removes every record from `messages`, keeping the Static Set; works only inside a compactor | [`compactors.clear`](11-conversations.md#compactorsclear) |
| `compactors.fail` | function | Default compactor; always raises `context_exhausted` with the reason it is given, any string | [`compactors.fail`](11-conversations.md#compactorsfail) |
| `coroutine` | removed | Reads nil; no creating coroutines or yielding on your own | [The sandbox](05-lua-environment.md#the-sandbox) |
| `debug` | not loaded | Never loaded; no stack frame, local, or hook inspection | [The sandbox](05-lua-environment.md#the-sandbox) |
| `dofile` | removed | Reads nil; no running a Lua file from disk | [The sandbox](05-lua-environment.md#the-sandbox) |
| `err.anchor` | field | Anchor text passed to a failed `store.str_replace` | [Editing, deleting, and checking files](09-the-store.md#editing-deleting-and-checking-files) |
| `err.count` | field | Times the anchor matched: `0` absent, `2` or more ambiguous | [Editing, deleting, and checking files](09-the-store.md#editing-deleting-and-checking-files) |
| `err.finish_reason` | field | Empty round's finish reason, such as `stop` or `length`; `nil` if unreported | [Empty replies and finish reasons](11-conversations.md#empty-replies-and-finish-reasons) |
| `err.kind` | field | Lowercase error kind string, one of thirteen, to branch on | [Error values](05-lua-environment.md#error-values) |
| `err.message` | field | Message text; `tostring(err)` and `..` give the same text | [Error values](05-lua-environment.md#error-values) |
| `err.path` | field | Path or pattern as passed; `not_a_directory` may name the blocking file | [When a store call fails](09-the-store.md#when-a-store-call-fails) |
| `err.reason` | field | Store tag such as `not_found`, or a `context_exhausted` reason such as `precheck` or `provider` | [When a store call fails](09-the-store.md#when-a-store-call-fails) |
| `err.rule` | field | Broken rule's snake_case tag, such as `traversal` or `wildcard` | [Store paths](09-the-store.md#store-paths) |
| `err.task` | field | Task id on `task_not_owned`, `task_consumed`, and a cancelled task's error | [Cancelling and listing tasks](15-tasks.md#cancelling-and-listing-tasks) |
| `err.tasks` | field | On `tasks_live`, leaked task ids in spawn order joined by `, ` | [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment) |
| `err.valid_up_to` | field | Byte offset where valid UTF-8 ends, on a `not_utf8` failure | [Writing and reading files](09-the-store.md#writing-and-reading-files) |
| `error` | function | Raises a string or table and stops the current block | [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors) |
| `getfenv` | removed | Reads nil; no function environments | [The sandbox](05-lua-environment.md#the-sandbox) |
| `getmetatable` | function | Reads a metatable, as in stock Lua on values other than `_G` | [The sandbox](05-lua-environment.md#the-sandbox) |
| `handle.capabilities` | field | Role's keywords, a sequence of kebab-case strings in declaration order | [Model handles](10-models.md#model-handles) |
| `handle.context` | field | Bound model's context window in tokens, a positive integer | [Model handles](10-models.md#model-handles) |
| `handle.description` | field | Role's `description:`, or the bound model's catalog description | [Model handles](10-models.md#model-handles) |
| `handle.label` | field | Role label or raw model id, the same string as `handle.name` | [Model handles](10-models.md#model-handles) |
| `handle.max_tokens` | field | `max_tokens` option from `models.use`, or `nil` | [Model handles](10-models.md#model-handles) |
| `handle.model_id` | field | Name part of the bound model's model id | [Model handles](10-models.md#model-handles) |
| `handle.name` | field | Role label, or the raw model id, as a string | [Model handles](10-models.md#model-handles) |
| `handle.provider` | field | Provider id the model catalog records, or `nil` | [Model handles](10-models.md#model-handles) |
| `handle.temperature` | field | `temperature` option from `models.use`, or `nil` | [Model handles](10-models.md#model-handles) |
| `handle.thinking` | field | `true` for `thinking`, `false` for `no-thinking`, `nil` for neither | [Model handles](10-models.md#model-handles) |
| `handle:infer` | method | Runs one round on the handle's model and returns the reply | [Model handles](10-models.md#model-handles) |
| `handle:loop` | method | Runs the section's `messages` on the handle's model, optional compactor; returns `nil` | [How the loop runs](11-conversations.md#how-the-loop-runs) |
| `input.ask` | function | Returns the operator's next message as typed; needs `user-input` declared | [Asking the operator](05-lua-environment.md#asking-the-operator) |
| `io` | not loaded | Never loaded; no file or stream I/O | [The sandbox](05-lua-environment.md#the-sandbox) |
| `ipairs` | function | Walks an array table in order | [The sandbox](05-lua-environment.md#the-sandbox) |
| `item` | global | The member a fanout arm runs for, or a task's `item` seed | [What an arm sees](14-fanout.md#what-an-arm-sees) |
| `json` | global | Table of JSON helpers: `encode`, `encode_lines`, `decode`, `decode_lines`, `null`, `array` | [Encoding and decoding JSON with `json`](05-lua-environment.md#encoding-and-decoding-json-with-json) |
| `json.array` | function | Marks a table as a list, or returns a new empty one, so an empty list encodes as `[]` | [Keeping empty lists with `json.array`](05-lua-environment.md#keeping-empty-lists-with-jsonarray) |
| `json.decode` | function | Returns the Lua value of one JSON document; lists come back marked, nulls as `json.null` | [Decoding JSON text with `json.decode`](05-lua-environment.md#decoding-json-text-with-jsondecode) |
| `json.decode_lines` | function | Returns a marked list of every JSON value in a text, such as a JSON Lines file | [JSON Lines with `json.encode_lines` and `json.decode_lines`](05-lua-environment.md#json-lines-with-jsonencode_lines-and-jsondecode_lines) |
| `json.encode` | function | Returns compact one-line JSON with sorted keys; `{ debug = true }` writes a placeholder in place of each value that is not JSON data | [Encoding values with `json.encode`](05-lua-environment.md#encoding-values-with-jsonencode) |
| `json.encode_lines` | function | Encodes each list item as one line, each followed by a newline | [JSON Lines with `json.encode_lines` and `json.decode_lines`](05-lua-environment.md#json-lines-with-jsonencode_lines-and-jsondecode_lines) |
| `json.null` | field | Truthy userdata standing for JSON null; test with `v == json.null` | [Null values and `json.null`](05-lua-environment.md#null-values-and-jsonnull) |
| `jump` | function | Ends the block and moves the walk to another section | [Jumping to a section](08-jump-and-call.md#jumping-to-a-section) |
| `kind = "cancelled"` | error kind | Cancelled task delivered by a wait, or a stopped run or call | [Cancelling and listing tasks](15-tasks.md#cancelling-and-listing-tasks) |
| `kind = "context_exhausted"` | error kind | Conversation overflows the context window and compaction does not make it fit, or `compactors.fail` raises | [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion) |
| `kind = "empty_model_reply"` | error kind | No text or tool calls, except an empty `stop` after tool calls | [Empty replies and finish reasons](11-conversations.md#empty-replies-and-finish-reasons) |
| `kind = "internal"` | error kind | A failure the prompt cannot fix, such as a transport failure | [Error kinds in Lua and the run error they become](16-limits-and-errors.md#error-kinds-in-lua-and-the-run-error-they-become) |
| `kind = "lua"` | error kind | An authoring or runtime failure, such as a compile or argument error | [Error values](05-lua-environment.md#error-values) |
| `kind = "out_of_scope_tool"` | error kind | Model calls a tool outside the scope, raised at `models.loop` | [Tool errors](12-tools.md#tool-errors) |
| `kind = "store"` | error kind | Any failure the store itself reports for an operation | [When a store call fails](09-the-store.md#when-a-store-call-fails) |
| `kind = "task_consumed"` | error kind | A wait names a task whose result is already delivered | [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin) |
| `kind = "task_not_owned"` | error kind | A task operation names an unowned task or an unknown id | [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment) |
| `kind = "tasks_live"` | error kind | A chain ends normally while a task it spawned is live | [Ownership, leaked tasks, and abandonment](15-tasks.md#ownership-leaked-tasks-and-abandonment) |
| `kind = "tool"` | error kind | A bound tool fails on a script `tools.call`, invalid structured JSON included | [Tool errors](12-tools.md#tool-errors) |
| `kind = "tool_loop_exhausted"` | error kind | Loop reaches the round cap without a final reply | [Tool rounds and the round cap](11-conversations.md#tool-rounds-and-the-round-cap) |
| `kind = "unbound_tool"` | error kind | A script `tools.call` names an id that is neither a defined adhoc tool nor a catalog tool | [Tool errors](12-tools.md#tool-errors) |
| `load` | removed | Reads nil; no compiling a string into a function | [The sandbox](05-lua-environment.md#the-sandbox) |
| `loadfile` | removed | Reads nil; no loading a Lua file from disk | [The sandbox](05-lua-environment.md#the-sandbox) |
| `loadstring` | removed | Reads nil; no loading code from a string | [The sandbox](05-lua-environment.md#the-sandbox) |
| `log` | function | Records one single-line checkpoint of at most 256 characters | [Logging and lifecycle reports](05-lua-environment.md#logging-and-lifecycle-reports) |
| `math` | global | Standard `math` library, as in Lua 5.5 | [The sandbox](05-lua-environment.md#the-sandbox) |
| `messages` | global | The section's conversation, a message list created empty at each section entry; cannot be reassigned | [`messages` is the section's conversation](11-conversations.md#messages-is-the-sections-conversation) |
| `#messages` | method | Number of records in the message list | [Reading records](11-conversations.md#reading-records) |
| `messages:append` | method | Appends a record table that names its own `role`, with optional `meta`; returns the list | [Building a message list](11-conversations.md#building-a-message-list) |
| `messages:assistant` | method | Appends an `assistant` record, optional `tool_calls` then `meta`; returns the list | [Building a message list](11-conversations.md#building-a-message-list) |
| `messages:replace` | method | Inside a compactor only, replaces records `first` through `last` with new records; returns the list | [Editing a list with `replace`](11-conversations.md#editing-a-list-with-replace) |
| `messages:system` | method | Appends a `system` record, optional `meta`; returns the list | [Building a message list](11-conversations.md#building-a-message-list) |
| `messages:tool` | method | Appends a `tool` record answering a `tool_call_id`, optional `meta`; returns the list | [Building a message list](11-conversations.md#building-a-message-list) |
| `messages:user` | method | Appends a `user` record, optional `meta`; returns the list | [Building a message list](11-conversations.md#building-a-message-list) |
| `messages[i]` | field | Read-only view of record `i`, or `nil` past the end | [Reading records](11-conversations.md#reading-records) |
| `models` | global | Table that selects model roles by label and runs rounds | [Model roles instead of models](10-models.md#model-roles-instead-of-models) |
| `models.compact` | function | Runs `compactor(reason)` once over `messages` outside any loop, checking the structural rules; returns `nil` | [Compacting on demand with `models.compact`](11-conversations.md#compacting-on-demand-with-modelscompact) |
| `models.default` | function | Sets the prompt-wide default role and returns its handle | [The prompt-wide default with `models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault) |
| `models.get` | function | Returns a handle for a role label or raw model id | [Getting handles with `models.get`](10-models.md#getting-handles-with-modelsget) |
| `models.infer` | function | Runs one round on the section's model and returns the reply | [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer) |
| `models.loop` | function | Runs the section's `messages` to a final reply, appending records, optional compactor; its first round fixes the section's tool list; returns `nil` | [Starting a conversation](11-conversations.md#starting-a-conversation) |
| `models.use` | function | Selects a role for the current section and returns its handle | [Selecting a role per section with `models.use`](10-models.md#selecting-a-role-per-section-with-modelsuse) |
| `models.use (max_tokens)` | option | Response token limit, a whole number from 1 to 4294967295 | [Sampling options on `models.use`](10-models.md#sampling-options-on-modelsuse) |
| `models.use (temperature)` | option | Sampling temperature, a finite number from 0.0 to 2.0 | [Sampling options on `models.use`](10-models.md#sampling-options-on-modelsuse) |
| `next` | function | Steps through keys in the same fixed order as `pairs` | [Deterministic table iteration](05-lua-environment.md#deterministic-table-iteration) |
| `os` | not loaded | Never loaded; no `os.time`, `os.clock`, `os.getenv`, or `os.execute` | [The sandbox](05-lua-environment.md#the-sandbox) |
| `package` | not loaded | Never loaded; no `package.path`, `package.loaded`, or search paths | [The sandbox](05-lua-environment.md#the-sandbox) |
| `pairs` | function | Visits keys in a fixed sorted order on every run | [Deterministic table iteration](05-lua-environment.md#deterministic-table-iteration) |
| `pairs(messages)` | method | Visits each record in order with its 1-based position | [Reading records](11-conversations.md#reading-records) |
| `pcall` | function | Catches errors, suspending calls included; Engine failures arrive as error values | [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors) |
| `pack.name` | field | The Pack's Pack id as a string | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `pack.required` | field | `true` when the Pack is declared under `uses:`; always `false` for the adhoc Pack | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `pack.tools` | field | Fresh array of the Pack's Tool objects in tool id order | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `print` | removed | Reads nil; `log` is the output path instead | [The sandbox](05-lua-environment.md#the-sandbox) |
| `prompt` | global | Table holding `prompt.section` and `prompt.list` | [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist) |
| `prompt.list` | function | Returns a visible list section's items as strings, or `nil` | [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist) |
| `prompt.section` | function | Returns a visible section's own raw Markdown body, or `nil` | [Reading sections with `prompt.section` and `prompt.list`](03-blocks-and-prose.md#reading-sections-with-promptsection-and-promptlist) |
| `prose` | global | Read-only pending prose above the block, placeholders rendered | [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global) |
| `rawequal` | removed | Reads nil; no raw table access | [The sandbox](05-lua-environment.md#the-sandbox) |
| `rawget` | removed | Reads nil; no raw table access | [The sandbox](05-lua-environment.md#the-sandbox) |
| `rawlen` | removed | Reads nil; no raw table access | [The sandbox](05-lua-environment.md#the-sandbox) |
| `rawset` | removed | Reads nil; no raw table access | [The sandbox](05-lua-environment.md#the-sandbox) |
| `record.content` | field | Record's text, or a non-empty array of content parts | [Message records](11-conversations.md#message-records) |
| `record.meta` | field | Record's metadata, any JSON data up to 8192 bytes, never sent to the model; `nil` when absent | [Record metadata with `meta`](11-conversations.md#record-metadata-with-meta) |
| `record.role` | field | Message role: `system`, `user`, `assistant`, or `tool` | [Message records](11-conversations.md#message-records) |
| `record.tool_call_id` | field | On a `tool` record, the id of the call it answers | [Message records](11-conversations.md#message-records) |
| `record.tool_calls` | field | Assistant's tool calls, each with `id`, `name`, and `arguments` | [Message records](11-conversations.md#message-records) |
| `require` | function | Loads a section's Lua as a library; replaces stock Lua `require` | [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require) |
| `result.exhausted` | field | `true` only when the arm's `models.loop` hits the round cap | [Reading the results](14-fanout.md#reading-the-results) |
| `result.item` | field | The member the arm runs on, keeping its Lua type | [Reading the results](14-fanout.md#reading-the-results) |
| `result.ok` | field | `true` when the arm completes normally, `false` at the round cap | [Reading the results](14-fanout.md#reading-the-results) |
| `result.text` | field | Arm's text or `''`; a capped arm gets the `UNKNOWN` exhausted stub | [Reading the results](14-fanout.md#reading-the-results) |
| `select` | function | `select(n, ...)` as in standard Lua 5.5 | [The sandbox](05-lua-environment.md#the-sandbox) |
| `setfenv` | removed | Reads nil; no function environments | [The sandbox](05-lua-environment.md#the-sandbox) |
| `setmetatable` | function | Sets a metatable; refuses finalizers and weak globals | [The sandbox](05-lua-environment.md#the-sandbox) |
| `status.blocked` | field | What the task waits on: `chat`, `tool_call`, `store`, `tasks`, `call`, `queued` | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.depth` | field | The task chain's `call` depth, `1` when spawned from the main walk | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.note` | field | The task's latest progress note from `tasks.note` | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.ok` | field | `true` or `false` once the task ends, `nil` while it runs | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.section` | field | The section the live task is in now | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.state` | field | Task state: `running`, `done`, `cancelled`, or `abandoned` | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.target` | field | Section name the task's chain starts at, without `#` marks | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.tasks` | field | Id strings of its own live tasks, in spawn order | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `status.turns` | field | Count of finished model rounds | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `store` | global | Table of store operations, present in every Lua block | [What the store is](09-the-store.md#what-the-store-is) |
| `store.append` | function | Adds text to a file's end, creating it if absent; returns `nil` | [Writing and reading files](09-the-store.md#writing-and-reading-files) |
| `store.delete` | function | Removes a file or empty directory; returns `nil`, even when absent | [Editing, deleting, and checking files](09-the-store.md#editing-deleting-and-checking-files) |
| `store.exists` | function | Returns whether anything exists at a path | [Editing, deleting, and checking files](09-the-store.md#editing-deleting-and-checking-files) |
| `store.glob` | function | Returns a sorted array of store paths matching a wildcard pattern | [Finding files with `store.glob`](09-the-store.md#finding-files-with-storeglob) |
| `store.read` | function | Returns a file's text, or lines `start` through `end` | [Writing and reading files](09-the-store.md#writing-and-reading-files) |
| `store.read_numbered` | function | Returns a file's lines as `N\| text`, optionally a line range | [Reading line ranges and numbered lines](09-the-store.md#reading-line-ranges-and-numbered-lines) |
| `store.str_replace` | function | Replaces the single anchor `old` with `new` and returns `nil` | [Editing, deleting, and checking files](09-the-store.md#editing-deleting-and-checking-files) |
| `store.write` | function | Creates or replaces a file with a text string and returns `nil` | [Writing and reading files](09-the-store.md#writing-and-reading-files) |
| `string` | global | Standard `string` library, including method calls such as `s:match` | [The sandbox](05-lua-environment.md#the-sandbox) |
| `sys` | global | Read-only table of run facts | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.execution` | field | The run's name, which the Harness assigns | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.id` | field | Current section entry id, such as `0.1` | [Section entry ids](05-lua-environment.md#section-entry-ids) |
| `sys.index` | field | An arm's 1-based position in collection order, or a task's `index` seed | [What an arm sees](14-fanout.md#what-an-arm-sees) |
| `sys.model` | field | Bound model's catalog id, readable after the section's first tool call | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.section_count` | field | Number of top-level sections in the prompt | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.section_name` | field | Running section's name; the H1 title in the H1 pass | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.taskid` | field | Nearest enclosing task's id, `'0'` on the main walk | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `sys.when` | field | Run start instant as an RFC 3339 UTC string | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `table` | global | Standard `table` library, as in Lua 5.5 | [The sandbox](05-lua-environment.md#the-sandbox) |
| `table.concat` | function | Joins strings, numbers, and `__tostring` values with a separator | [The sandbox](05-lua-environment.md#the-sandbox) |
| `tasks` | global | Table holding every task function, `fanout` included | [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) |
| `tasks.cancel` | function | Ends a task the caller owns and returns nothing | [Cancelling and listing tasks](15-tasks.md#cancelling-and-listing-tasks) |
| `tasks.concurrency` | function | Sets or reads the chain's concurrency limit, returning the effective limit | [Concurrency and the admission window](14-fanout.md#concurrency-and-the-admission-window) |
| `tasks.fanout` | function | Runs a worker section once per member, results in collection order | [Fanout in one call](14-fanout.md#fanout-in-one-call) |
| `tasks.join` | function | Waits for all tasks; returns `{ task, ok, result }` entries, then `timed_out` | [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin) |
| `tasks.join_any` | function | Waits for the first task to end; returns handle, `ok`, result | [Waiting with `tasks.join_any` and `tasks.join`](15-tasks.md#waiting-with-tasksjoin_any-and-tasksjoin) |
| `tasks.join_any (timeout)` | option | Wait limit in seconds, also on `tasks.join`; never cancels tasks | [Time limits on waits](15-tasks.md#time-limits-on-waits) |
| `tasks.note` | function | Publishes the caller's own task's latest progress note | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `tasks.pending` | function | Lists the caller's live task handles in spawn order; takes no arguments | [Cancelling and listing tasks](15-tasks.md#cancelling-and-listing-tasks) |
| `tasks.ready` | function | Checks without waiting whether a task has ended | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `tasks.spawn` | function | Starts a section as a task, returns handle `{ task = id }` at once | [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) |
| `tasks.spawn (index)` | option | Integer of 0 or more that becomes the task's `sys.index` | [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) |
| `tasks.spawn (input)` | option | String that replaces the task's `args`, re-deriving `argv` | [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) |
| `tasks.spawn (item)` | option | JSON data that becomes the task's `item` and `{{ item }}` seed | [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn) |
| `tasks.status` | function | Returns a task's status as a plain table | [Checking on a task with `tasks.status`, `tasks.ready`, and `tasks.note`](15-tasks.md#checking-on-a-task-with-tasksstatus-tasksready-and-tasksnote) |
| `tonumber` | function | Turns a value such as a numeric string into a number | [The sandbox](05-lua-environment.md#the-sandbox) |
| `tool.description` | field | The tool's description: the catalog text, or an adhoc tool's `description` attr | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `tool.id` | field | The tool's canonical tool id, such as `web/search` or `adhoc/grab` | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `tool.pack` | field | Pack object of the Pack named by the tool id; the adhoc Pack for an adhoc tool | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `tool.parameters` | field | Fresh table holding the tool's parameter schema, converted the way `json.decode` converts | [Tool objects, Pack objects, and `tools.packs`](12-tools.md#tool-objects-pack-objects-and-toolspacks) |
| `tools` | global | Table holding every tool operation | [Tools, Packs, and the `uses:` key](12-tools.md#tools-packs-and-the-uses-key) |
| `tools.adhoc` | function | Defines an adhoc tool from `name`, `description`, and optional `parameters` attrs and a function; returns its Tool object and offers nothing | [Adhoc tools with `tools.adhoc`](12-tools.md#adhoc-tools-with-toolsadhoc) |
| `tools.call` | function | Calls a tool from Lua by tool id or Tool object and returns its output | [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls) |
| `tools.calls` | field | Read-only per-section call counts keyed by tool id | [Calling tools from Lua and counting calls](12-tools.md#calling-tools-from-lua-and-counting-calls) |
| `tools.get` | function | Reads one Tool object by tool id, a catalog tool's or `adhoc/<name>`, or `nil` | [Finding a tool with `tools.get`](12-tools.md#finding-a-tool-with-toolsget) |
| `tools.offer` | function | Offers a tool or Pack by id or object, or an array of them, to the section's model; raises after the first loop round renders the tool list | [Offering tools and Packs with `tools.offer`](12-tools.md#offering-tools-and-packs-with-toolsoffer) |
| `tools.offer_dynamic` | function | Records tools and Packs, in the forms `tools.offer` takes, in the section's Dynamic Set, which the model cannot reach; raises after the first loop round renders the tool list | [The Dynamic Set with `tools.offer_dynamic`](12-tools.md#the-dynamic-set-with-toolsoffer_dynamic) |
| `tools.packs` | field | Read-only list of the section's Pack objects in name order, the adhoc Pack included, with `get` and `extras` | [Walking Packs with `tools.packs`](12-tools.md#walking-packs-with-toolspacks) |
| `tools.packs.extras` | function | Returns a fresh array of the usable Pack objects not declared under `uses:`, in name order, never the adhoc Pack | [Extra Packs with `tools.packs.extras`](12-tools.md#extra-packs-with-toolspacksextras) |
| `tools.packs.get` | function | Reads one Pack object by Pack id, or `nil` when the run has no such Pack | [Finding a Pack with `tools.packs.get`](12-tools.md#finding-a-pack-with-toolspacksget) |
| `tostring` | function | Turns a value such as a number into text | [The sandbox](05-lua-environment.md#the-sandbox) |
| `type` | function | Returns a value's Lua type name, such as `'table'` | [The sandbox](05-lua-environment.md#the-sandbox) |
| `ui` | function | Returns the Host's run-start snapshot table, when the run has one | [Run facts in `sys` and `ui`](05-lua-environment.md#run-facts-in-sys-and-ui) |
| `untrusted` | function | Wraps a string in the run's untrusted envelope and returns it | [Handing store text to a model with `untrusted`](09-the-store.md#handing-store-text-to-a-model-with-untrusted) |
| `utf8` | global | Standard `utf8` library, as in Lua 5.5 | [The sandbox](05-lua-environment.md#the-sandbox) |
| `var` | global | Guarded table of JSON data that passes between sections | [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var) |
| `warn` | removed | Reads nil; no Lua warnings | [The sandbox](05-lua-environment.md#the-sandbox) |
| `xpcall` | function | Like `pcall`, with a message handler that sees the failure first | [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors) |

## Substitution forms

Each form goes in a section's prose and takes effect when a Lua block reads `prose`.

| Form | Inserts | Taught in |
|---|---|---|
| `\\` | A single literal `\` | [Escapes and literal braces](07-substitution.md#escapes-and-literal-braces) |
| `\{{` | A literal `{{`, starting no placeholder | [Escapes and literal braces](07-substitution.md#escapes-and-literal-braces) |
| `\}}` | A literal `}}` | [Escapes and literal braces](07-substitution.md#escapes-and-literal-braces) |
| `{{ args }}` | The raw argument string, exactly as passed | [Inserting arguments with `args` and `argv`](07-substitution.md#inserting-arguments-with-args-and-argv) |
| `{{ argv }}` | The whole parsed arguments, with an object or array as compact JSON | [Inserting arguments with `args` and `argv`](07-substitution.md#inserting-arguments-with-args-and-argv) |
| `{{ argv.key }}` | One field of the parsed arguments, at any depth | [Inserting arguments with `args` and `argv`](07-substitution.md#inserting-arguments-with-args-and-argv) |
| `{{ item }}` | The item seeded into a fanout arm or a task, rendered by its type | [Inserting the fanout or task `item`](07-substitution.md#inserting-the-fanout-or-task-item) |
| `{{ name }}` | The whole value of the section-local Lua global by that name | [Inserting Lua globals by bare name](07-substitution.md#inserting-lua-globals-by-bare-name) |
| `{{ path }}` | A live value from the run, named by the trimmed path | [Placeholders in prose](07-substitution.md#placeholders-in-prose) |
| `{{ prose }}` | Rejected as recursive, so the `prose` read fails with substitution kind `Serialize` | [Substitution error kinds](07-substitution.md#substitution-error-kinds) |
| `{{ sys.key }}` | A field of `sys`, such as `sys.id` or `sys.when` | [Inserting `var` and `sys` values](07-substitution.md#inserting-var-and-sys-values) |
| `{{ var.key }}` | The current value of that field of `var` | [Inserting `var` and `sys` values](07-substitution.md#inserting-var-and-sys-values) |

## Pack tools

Each Pack tool comes from a Pack you declare under `uses:` by its Pack id, `web` or `user-input`, while Provided by gives the fully qualified name of the Pack, which a Host's Plugin adds.

| Tool | Provided by | Arguments | Returns | Taught in |
|---|---|---|---|---|
| `user-input/ask` | `runvil/user-input` | none | The operator's next message as trusted plain text | [Asking the operator](05-lua-environment.md#asking-the-operator) |
| `web/fetch` | `runvil/web` | `url` (required); `max_chars` (default 40,000; 1 to 40,000); `raw` (default `false`) | Untrusted text: a `url:`, `truncated:`, and `extraction:` header, a blank line, then the page | [Fetching a page](13-web-fetch-and-search.md#fetching-a-page) |
| `web/search` | `runvil/web` | `query` (required; 1 to 400 characters, not only whitespace); `count` (search provider default; 1 to 20); `freshness` (`pd`, `pw`, `pm`, or `py`); `country` (not blank, at most 128 characters); `search_lang` (not blank, at most 128 characters); `safesearch` (`off`, `moderate`, or `strict`); `include_domains` (default empty; at most 20 bare hostnames); `exclude_domains` (default empty; at most 20 bare hostnames) | Untrusted compact JSON of `query` and `results`, each with `title`, `url`, and `description` | [Searching the web](13-web-fetch-and-search.md#searching-the-web) |

## Limits and defaults

Each row gives a limit, its default, and who can change it, and the Pack tools table holds the Pack tool limits.

| Limit | Default | Set by | Taught in |
|---|---|---|---|
| `concurrency` | 8 tasks at once | Harness | [Run limits and their defaults](16-limits-and-errors.md#run-limits-and-their-defaults) |
| `json` nesting limit | 127 levels of nested tables, arrays, and objects, on encode and decode | fixed | [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) |
| `json.decode` input cap | 8,388,608 bytes per `json.decode` or `json.decode_lines` call | fixed | [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) |
| `json.encode` key limit | 100,000 keys that are not numbers or booleans on one path, per `json.encode` or `json.encode_lines` call | fixed | [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) |
| `json.encode` size budget | One eighth of `lua_memory_bytes` in values and `lua_memory_bytes` in bytes of text per `json.encode`, `json.encode_lines`, or `var` write, so 8,388,608 values and 67,108,864 bytes by default | Harness, through `lua_memory_bytes` | [Size and nesting limits](05-lua-environment.md#size-and-nesting-limits) |
| log byte quota | 256 bytes per allowed `log` event, 262,144 bytes by default | Harness | [Logging and lifecycle reports](05-lua-environment.md#logging-and-lifecycle-reports) |
| log message limit | 256 Unicode characters | fixed | [Logging and lifecycle reports](05-lua-environment.md#logging-and-lifecycle-reports) |
| `lua_log_events` | 1024 `log` events per section VM | Harness | [Lua memory and log quotas](16-limits-and-errors.md#lua-memory-and-log-quotas) |
| `lua_memory_bytes` | 64 MiB per section VM | Harness | [Lua memory and log quotas](16-limits-and-errors.md#lua-memory-and-log-quotas) |
| `max_response_bytes` | 16 MiB per model reply | Harness | [Run limits and their defaults](16-limits-and-errors.md#run-limits-and-their-defaults) |
| `max_tool_iterations` | 24 model rounds per loop call unless the Harness sets another | frontmatter | [Run limits and their defaults](16-limits-and-errors.md#run-limits-and-their-defaults) |
| output reserve | `max_tokens`, else one eighth of the window up to 8192 tokens, at most half the window | fixed | [The context precheck](11-conversations.md#the-context-precheck) |
| recursion depth cap | 8 nested levels, counting each `call`, task, and fanout arm | fixed | [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap) |
| `request_timeout` | 120 seconds per wait for the next part of a model reply | Harness | [Run limits and their defaults](16-limits-and-errors.md#run-limits-and-their-defaults) |
| `tools.offer` array length | 100,000 entries per `tools.offer` or `tools.offer_dynamic` call | fixed | [Errors from `tools.offer`](12-tools.md#errors-from-toolsoffer) |

## Parse error kinds

Each parse failure has one parse error kind, the four fence rules say what a `Fence` failure can break, and the location fields say which prompt failed and where.

| Name | Type | Meaning | Taught in |
| --- | --- | --- | --- |
| `Closer` | fence rule | a closing line after an exact `lua` opener that is not exactly three backticks | [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors) |
| `column` | location field | the 1-based column of the mistake, read with the line | [Where an error points](16-limits-and-errors.md#where-an-error-points) |
| `Fence` | parse error kind | a `lua` fence left open, or a break of the Label, Opener, Closer, or Lookalike rule | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |
| `Frontmatter` | parse error kind | frontmatter missing, unclosed, or not valid YAML, or a key or value the contract rejects | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |
| `Internal` | parse error kind | the parser fails through no fault of the prompt, such as a Lua compiler that cannot start | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |
| `Label` | fence rule | a fence label that starts with `lua` in any letter case but is not exactly `lua` | [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors) |
| `line` | location field | the 1-based line in the prompt file as written | [Where an error points](16-limits-and-errors.md#where-an-error-points) |
| `List` | parse error kind | a list section item that is a marker with no text after it | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |
| `Lookalike` | fence rule | a line outside every fence that looks like a `lua` opening line | [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors) |
| `Lua` | parse error kind | a Lua block that does not compile because of a syntax error | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |
| `name` | location field | the prompt's frontmatter `name`, on a `Structure`, `Fence`, or `List` failure after the frontmatter | [Where an error points](16-limits-and-errors.md#where-an-error-points) |
| `Opener` | fence rule | a `lua` fence whose opening line is not exactly three backticks then `lua` | [Exact fence rules and fence errors](03-blocks-and-prose.md#exact-fence-rules-and-fence-errors) |
| `span` | location field | the byte range of the offending region in the body after the frontmatter, when the failure can locate it | [Where an error points](16-limits-and-errors.md#where-an-error-points) |
| `Structure` | parse error kind | an invalid H1 title or heading tree, such as two sibling sections with one name | [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds) |

## Run error kinds

A failed run reports exactly one run error kind, and a failed `{{ }}` placeholder names its substitution kind in its message.

| Name | Type | Meaning | Taught in |
| --- | --- | --- | --- |
| `BadPath` | substitution kind | `{{ var }}` or `{{ sys }}` written with no key | [Substitution error kinds](07-substitution.md#substitution-error-kinds) |
| `Binding` | run error kind | `models.infer` or `models.loop` runs with neither `models.use` nor `models.default` in effect, or a section's first loop round fails because the section offers a tool whose parameter schema is not a JSON object, whose wire name is invalid, or whose wire name another offered tool shares, unless an adhoc tool in the same set shadows it | [Infer errors and the missing-model error](10-models.md#infer-errors-and-the-missing-model-error) |
| `Cancelled` | run error kind | never reported; a run the Host cancels ends with the cancelled outcome, not a failure | [Run error kinds](16-limits-and-errors.md#run-error-kinds) |
| `Completion` | run error kind | a model call fails at the transport, backend, or decode layer, or an `empty_model_reply` goes uncaught | [Run error kinds](16-limits-and-errors.md#run-error-kinds) |
| `ContextExhausted` | run error kind | a request overflows the model's context window and the selected compactor leaves it over, or a direct `compactors.fail` call goes uncaught | [Compactors and context exhaustion](11-conversations.md#compactors-and-context-exhaustion) |
| `Determinism` | run error kind | a store call meets a claims conflict, which no `pcall` catches | [Races, claims, and determinism](09-the-store.md#races-claims-and-determinism) |
| `EmptySegment` | substitution kind | an empty or space-padded path segment, as in `{{ var. }}` or `{{ }}` | [Substitution error kinds](07-substitution.md#substitution-error-kinds) |
| `Internal` | run error kind | an Engine invariant breaks, a fault in the Engine rather than the prompt | [Run error kinds](16-limits-and-errors.md#run-error-kinds) |
| `Lua` | run error kind | an uncaught runtime error or `{{ }}` failure, running past the memory ceiling, a block that returns a table, a Pack prelude that cannot install, or a task fault: a leaked task, a task its chain does not own, a result waited on twice, or a cancelled task's error raised again right after its wait | [Lua blocks, section VMs, and Engine globals](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals) |
| `MissingKey` | substitution kind | an absent key under `var`, `sys`, `argv`, or a global, as in `{{ sys.bogus }}`, or a path that continues past a scalar or aims at an array element | [Inserting `var` and `sys` values](07-substitution.md#inserting-var-and-sys-values) |
| `NilArgv` | substitution kind | any `argv` placeholder while `argv` is nil, as when the argument string is not JSON | [Inserting arguments with `args` and `argv`](07-substitution.md#inserting-arguments-with-args-and-argv) |
| `NilItem` | substitution kind | `{{ item }}` where no item is seeded, such as in a walked section | [Inserting the fanout or task `item`](07-substitution.md#inserting-the-fanout-or-task-item) |
| `NotATable` | substitution kind | a dotted path after `args` or `item`, as in `{{ args.x }}` | [Inserting arguments with `args` and `argv`](07-substitution.md#inserting-arguments-with-args-and-argv) |
| `NullValue` | substitution kind | a path that lands on JSON null, under every root except `item` | [Substitution error kinds](07-substitution.md#substitution-error-kinds) |
| `Parse` | run error kind | the frontmatter or the body's structure fails to parse, a Lua block does not compile, or the `runvil:` key is missing | [Run error kinds](16-limits-and-errors.md#run-error-kinds) |
| `Quota` | run error kind | a block runs out its `log` event or byte quota, or a `call`, `tasks.spawn`, or `tasks.fanout` arm nests past the call depth cap | [The recursion depth cap](08-jump-and-call.md#the-recursion-depth-cap) |
| `RequirementsUnmet` | run error kind | the environment falls short of a requirement the prompt states, or an H1 block fails the hard gate | [Preparing the run](04-how-a-prompt-runs.md#preparing-the-run) |
| `Serialize` | substitution kind | a bare global that is not JSON data, and `{{ prose }}` | [Inserting Lua globals by bare name](07-substitution.md#inserting-lua-globals-by-bare-name) |
| `Substitution` | run error kind | never reported; an uncaught substitution failure ends as `Lua`, or as `RequirementsUnmet` in the H1 pass | [Catching and reading substitution errors](07-substitution.md#catching-and-reading-substitution-errors) |
| `Tool` | run error kind | a dispatched tool fails, the model calls a tool outside its round's scope, a `tools.call` names an id that is neither a defined adhoc tool nor a catalog tool, or `models.loop` reaches its round cap | [Tool rounds and the round cap](11-conversations.md#tool-rounds-and-the-round-cap) |
| `Unclosed` | substitution kind | a `{{` with no `}}` anywhere later in the prose | [Substitution error kinds](07-substitution.md#substitution-error-kinds) |
| `UnknownNamespace` | substitution kind | a first segment that is neither a root nor a set global, as in `{{ ghost }}` | [Inserting Lua globals by bare name](07-substitution.md#inserting-lua-globals-by-bare-name) |
| `Version` | run error kind | the `runvil:` key declares a major version other than `0` | [The `runvil` version key](02-file-structure.md#the-runvil-version-key) |
| `Vfs` | run error kind | an uncaught `store` failure, a run that declares no store, or a store backend that fails as the run starts | [When a store call fails](09-the-store.md#when-a-store-call-fails) |
