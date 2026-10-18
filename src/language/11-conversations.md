# Conversations

A conversation is a message list that you run with `models.loop`, which keeps running model rounds, each one a request to a model and its reply, and dispatching any tool calls the model makes until it can return a reply. This chapter shows you how to build the list, what each record in it holds, and how to read records back, so you can send a model instructions, earlier messages, and tool results together instead of a single piece of prose. It then follows the loop round by round and covers every error it raises.

## Starting a conversation

Here you build a message list, add one `user` record, run the list with `models.loop`, and read the reply the loop leaves at its end.

````markdown
---
name: first-chat
description: Says hello to a model and returns its reply.
runvil: 0
models:
  writer: {}
---

# First chat

## Chat

```lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
models.loop(msgs)
return msgs[#msgs].content
```
````

The run result is the model's reply to `hello`. The block does four things:

1. `models.use('writer')` selects the `writer` model role for the section (see [Selecting a role per section with `models.use`](10-models.md#selecting-a-role-per-section-with-modelsuse)).
2. `messages.new()` creates an empty message list. A message list is the ordered list of message records a conversation sends to the model, and a message record is one message in it, written as a Lua table holding a `role` and a `content` (see [Message records](#message-records)).
3. `msgs:user('hello')` appends the user record `{ role = 'user', content = 'hello' }` at the end of the list, so `#msgs` is 1 and `msgs[1].role` is `'user'`. When `models.loop` runs, the first message the model receives has the `user` message role and `hello` as its content.
4. `models.loop(msgs)` sends the list to the model and appends the model's text reply to the same list as a new `assistant` record. That record is the terminal record, the model's final reply, which the loop always leaves last in the list, so `return msgs[#msgs].content` reads its text.

This is the shortest loop pattern: create a list, add a user message, run the loop, and return the last record's content.

### Reading the list by position

`#msgs` gives a list's length, and `msgs[i]` reads the record at 1-based position `i`. You read a record's fields, such as `msgs[i].role` and `msgs[i].content`, directly, so `msgs[#msgs]` is the last record and `msgs[#msgs].content` is its text. A key that is neither a whole-number position nor a method name reads `nil`. [Reading records](#reading-records) covers every way to read a list.

### The `messages` global

`messages` is an Engine global, so a prompt's Lua uses it with no import or declaration (see [Lua blocks, section VMs, and Engine globals](05-lua-environment.md#lua-blocks-section-vms-and-engine-globals)). Its only member is `new`, the function that builds message lists, so the whole namespace is the one form `messages.new()`.

The global exists in every Lua block of the prompt: in every section, in the H1 body's blocks during [the H1 pass](04-how-a-prompt-runs.md#the-h1-pass), and in a library loaded with `require` (see [Sharing code with `require`](03-blocks-and-prose.md#sharing-code-with-require)), because the Engine installs it before a library's blocks run.

`messages.new()` takes no arguments and returns a new, empty message list on every call. Before anything is added, `#msgs` is 0 and `msgs[1]` is `nil`. You fill the list with builder methods, the list methods that append records, such as `msgs:system(...)` and `msgs:user(...)`, which is why a conversation block starts with `local msgs = messages.new()`.

````markdown
---
name: two-lists
description: Shows that each new message list is separate.
runvil: 0
---

# Two lists

## Build

```lua
local a, b = messages.new(), messages.new()
a:user('x')
return type(a) .. '|' .. #a .. '|' .. #b
```
````

````text
userdata|1|0
````

Each call returns a separate list, so adding a record to `a` leaves `b` unchanged. A list is a userdata value, so `type(a)` reports `'userdata'` rather than `'table'`.

### A list belongs to the Engine

A message list is a value the Engine owns, not a plain Lua table, and its methods are the only way to change it. Assigning into a list, whether at a record position or under a new name, raises this error, and the records stay as they were:

````text
a messages.new() list cannot be assigned to; add records with append(record) and change them with replace(first, last, records...)
````

[Building a message list](#building-a-message-list) covers `append`, and [Editing a list with `replace`](#editing-a-list-with-replace) covers `replace`.

Assigning a list to another variable, or passing it along, shares the one list rather than copying it. Put this block in the `## Build` section of `two-lists` in place of the first one:

````lua
local msgs = messages.new():system('be terse'):user('hi')
local same = msgs
same:user('more')
return #msgs
````

````text
3
````

An edit made through either name shows up through both. A list handed to a model request is the same list too, so a record you add afterward with `msgs:user('more')` shows up in that request's list, which then holds 3 records.

Only a value made by `messages.new()` counts as a message list. `models.loop` refuses a plain Lua table, even one holding the same system and user records, and it refuses an empty table, with the list error that [Argument errors](#argument-errors) shows.

Lists and their records work inside [the sandbox](05-lua-environment.md#the-sandbox) that section Lua runs in, so a chained `messages.new():system('s'):user('u')` builds a list there, and a block can return `{ msgs[1], msgs[2] }`, a table of its records, without error.

### Running the list with `models.loop`

`models.loop(messages, compactor?)` is a plain function that takes the message list first and an optional compactor second. `models.loop(msgs)` passes only the list, which selects the default compactor, `compactors.fail`. [Choosing a compactor for a call](#choosing-a-compactor-for-a-call) covers the second argument, and [Compactors and context exhaustion](#compactors-and-context-exhaustion) explains what a compactor does.

`models.loop(msgs)` returns `nil`. What it produces is the records it appends to your list. This version of the `## Chat` block in `first-chat` checks both:

````markdown
---
name: first-chat
description: Says hello to a model and returns its reply.
runvil: 0
models:
  writer: {}
---

# First chat

## Chat

```lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
local result = models.loop(msgs)
return tostring(result) .. '|' .. #msgs .. '|' .. msgs[2].role
```
````

When the model replies with text, the run result is:

````text
nil|2|assistant
````

The one-record list grows to two records, `msgs[2].role` is `'assistant'`, and `msgs[2].content` holds the reply.

To continue the conversation, add to the same list and run the loop again. With this block in the `## Chat` section of `first-chat`, the model answers twice:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('first question')
models.loop(msgs)
msgs:user('second question')
models.loop(msgs)
return #msgs
````

````text
4
````

Two user records with a loop after each leave `#msgs` at 4. The second call continues the conversation, because its request carries the earlier reply ahead of the new user record: when the first reply was `ok`, the second request holds `{ "role": "assistant", "content": "ok" }` before `second question`.

When the model asks for tools, one `models.loop` call runs a whole multi-round conversation. It sends the list round after round, runs the tools the model asks for (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)), and appends every record back to the list until the model replies with text. A round that ends with the model asking for tools is a tool round, which [Tool rounds and the round cap](#tool-rounds-and-the-round-cap) covers. After two tool rounds and a final text reply of `both done`, one `models.loop(msgs)` call leaves `#msgs` at 6, counting the user record, the records the two tool rounds added, and the terminal record, with `msgs[6].content` equal to `'both done'`. The model received 3 requests, and the last one carried both tool results.

When `models.loop` returns, the terminal record is the last record in the list, after tool rounds as well. `msgs[#msgs].role` is `'assistant'`, and you read the final answer with `msgs[#msgs].content`, which after a tool round and a text reply of `final answer` returns `final answer`.

### Common conversation shapes

The single-shot research shape builds a list from the section's prose, runs it, and returns the last record's content:

````markdown
---
name: summarize
description: Summarizes a person in one conversation.
runvil: 0
models:
  writer: {}
---

# Summarize

## Summary

Summarize the person.

```lua
models.use('writer')
local msgs = messages.new()
msgs:user(prose)
models.loop(msgs)
return msgs[#msgs].content
```
````

`msgs:user(prose)` adds the section prose above the fence (see [Pending prose and the `prose` global](03-blocks-and-prose.md#pending-prose-and-the-prose-global)) as the user record, so the request carries `Summarize the person.` in its first message, and the run result is the model's summary. The shipped `research-person` prompt (see [Shipped example prompts](01-what-a-prompt-is.md#shipped-example-prompts)) builds its list in this shape.

One section can make a single model round while a later section holds a conversation:

````markdown
---
name: draft-and-refine
description: Drafts with one round, then refines the draft in a conversation.
runvil: 0
models:
  writer: {}
---

# Draft and refine

## Draft

Write a short paragraph about the sea.

```lua
models.use('writer')
var.draft = models.infer(prose)
```

## Refine

```lua
models.use('writer')
local msgs = messages.new()
msgs:system('You refine drafts.')
msgs:user(var.draft)
models.loop(msgs)
return msgs[#msgs].content
```
````

`## Draft` runs one round with `models.infer` (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)) and keeps the reply in `var` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)). `## Refine` builds a list from that draft and runs it, so the loop sends the system record first and the draft as the user message after it. The run result is the refined paragraph.

A chat keeps one list for the whole conversation and adds to it before each loop. With this block in the `## Chat` section of `first-chat`, two user messages from a Lua array each get a reply, and each loop call is guarded with `pcall` (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)):

````lua
models.use('writer')
local history = messages.new()
for _, text in ipairs({ 'first question', 'second question' }) do
  history:user(text)
  local ok, err = pcall(function() return models.loop(history) end)
  if not ok then return 'failed: ' .. tostring(err) end
end
return #history .. '|' .. history[2].content .. '|' .. history[4].content
````

When the model answers `first reply` and then `second reply`, the run result is:

````text
4|first reply|second reply
````

Each request carries every record the earlier loops left in the list, so the second request's messages are `first question`, `first reply`, and `second question`, and the two loops leave 4 records. Wrapping a call in `pcall` runs the loop normally and still appends the reply, so the chat can guard each call against errors. The chat prompt built into Workshop has this shape: it keeps one `local history = messages.new()` list and, inside `while true do`, calls `history:user(text)` with each chat turn's text and then `pcall(function() return models.loop(history) end)`.

## Message records

Here you write a message record for each message role by hand, then fill records with content parts, tool calls, and tool results.

````markdown
---
name: records
description: Builds one message record for each message role.
runvil: 0
---

# Records

## Build

```lua
local msgs = messages.new()
msgs:append({ role = 'system', content = 'be terse' })
msgs:append({ role = 'user', content = 'say hi' })
msgs:append({
  role = 'assistant',
  content = 'working',
  tool_calls = { { id = 'call_1', name = 'web_search', arguments = { query = 'hi' } } },
})
msgs:append({ role = 'tool', content = 'three results for hi', tool_call_id = 'call_1' })
return #msgs .. '|' .. msgs[1].role .. '|' .. msgs[4].tool_call_id
```
````

````text
4|system|call_1
````

You write a message record as a Lua table holding a `role` and a `content`, plus `tool_calls` or `tool_call_id` on the records that carry them, and you add it to a list with `msgs:append(record)`. One list can mix all four message roles, as this one does, with its system record first.

### The four message roles

A record's `role` is one of the exact lowercase strings `system`, `user`, `assistant`, or `tool`. The record is sent to the model with its message role in that same lowercase form, such as `"role": "assistant"`. Each message role holds one kind of message and carries its own fields:

| Message role | Holds | Fields |
|---|---|---|
| `system` | A system instruction | `role`, `content` |
| `user` | The user side of the conversation | `role`, `content` |
| `assistant` | The model's side of the conversation | `role`, `content`, and `tool_calls` when it holds calls |
| `tool` | A tool result | `role`, `content`, `tool_call_id` |

After a model call, the model's reply is an `assistant` record at the end of the list, so a reply such as `pong` reads as `msgs[#msgs].content`. A `tool` record answers one tool call, so a record where `role == 'tool'` and `tool_call_id == 'call_1'` is the result for the call `call_1`.

A record whose `role` is missing or not a string is refused with `messages[N] role must be a string, one of: system, user, assistant, tool`. A `role` string other than the four is refused with a message that quotes it and lists `system, user, assistant, tool` as the known roles. [Record error messages](#record-error-messages) gives every record error in full, along with what `N` counts.

### Content

Every message record has a `content` field, whatever its message role, and `content` is always sent to the model beside `role`. It holds the message's visible content, its text and images, in one of two forms: a plain text string, or a non-empty array of content parts.

When the content is plain text, the `content` field is that string itself, and a record gives you its text through that field, so `msgs[#msgs].content` reads the text of the last record, such as the latest model reply `done`. Any string is valid content, the empty string included, because strings get no length check, and the non-empty rule applies only to a parts array. That is how an `assistant` record that only makes tool calls fills its content, with `content = ""` beside its `tool_calls`. Any other value is refused with `messages[N] content must be a string or a non-empty array of content parts`.

A plain message has only a message role and its content, as in `{ "role": "system", "content": "be terse" }`. A plain assistant text record has no `tool_calls` field, so reading its `tool_calls` gives `nil`.

### Content parts

A content part is one piece of a record's content when the content is an array instead of a string. One record can mix text parts and image parts. Put this block in the `## Build` section of `records` in place of the first one:

````lua
local msgs = messages.new()
msgs:user({
  { type = 'text', text = 'look' },
  { type = 'image_url', image_url = { url = 'data:image/png;base64,AA==' } },
})
local content = msgs[1].content
return type(content) .. '|' .. #content .. '|' .. content[1].text
````

````text
table|2|look
````

The `user` record's `content` is an array of two typed parts, which appends without error. Reading `content` back from the list gives that parts array as a Lua table. Each part is a table whose string `type` field names its variant, and the chat protocol accepts exactly two types:

- A text part is `{ type = 'text', text = '...' }`, carrying its text in a string `text` field, as in `{ type = 'text', text = 'look' }`. It is sent to the model as `{ "type": "text", "text": ... }`.
- An image part is `{ type = 'image_url', image_url = { url = '...' } }`. The URL can be a base64 `data:` URL such as `data:image/png;base64,AA==`, and the part is sent to the model as `{ "type": "image_url", "image_url": { "url": ... } }`. The check requires a string `url` inside the `image_url` table and ignores any other keys there.

The parts above are sent as `"content": [ { "type": "text", "text": "look" }, ... ]`. A part keeps only its payload, its `text` or its `image_url.url`, and any other field on the part is dropped.

A parts `content` is an array of part tables even when it holds one part, as in `content = { { type = 'text', text = 'hi' } }`. A single part table placed directly in `content` is not an array, so it fails with `messages[N] content must be a string or a non-empty array of content parts`, and so does an empty table. The parts themselves follow three rules, each with its own message:

- A part that is not a table with a string `type` field is refused with a message ending `must be a table with a string type field`.
- A part whose `type` is neither `text` nor `image_url` is refused with a message that quotes the type and lists `known types: text, image_url`.
- An image part without an `image_url` table holding a string `url` is refused with a message ending `must set an image_url table with a string url field`.

### Tool calls

A tool call is one request from the model to run a tool, and an `assistant` record holds its calls in a `tool_calls` array next to its string `content`. Put this block in the `## Build` section of `records`:

````lua
local msgs = messages.new()
msgs:append({ role = 'user', content = 'What is the weather in Paris, and the time?' })
msgs:append({
  role = 'assistant',
  content = 'Let me check both.',
  tool_calls = {
    { id = 'call_1', name = 'weather', arguments = { city = 'Paris' } },
    { id = 'call_2', name = 'clock' },
  },
})
local calls = msgs[2].tool_calls
return #calls .. '|' .. calls[1].id .. '|' .. calls[2].name .. '|' .. type(calls[2].arguments)
````

````text
2|call_1|clock|table
````

Each entry in `tool_calls` is a table with exactly three fields:

- `id` is a string holding the call id the model assigned, such as `'call_1'`.
- `name` is a string holding the name the model called the tool by, such as `'web_search'`, which is how the model calls the tool you offer as `web/search` (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)).
- `arguments` is a table holding the model's JSON arguments, already parsed into a Lua table, so a model that sends `{"query":"hi"}` gives `call.arguments.query == 'hi'`.

`arguments` may be left out, set to `nil`, or set to a table, and it can be empty or richer, such as a table holding `path` set to `"notes.txt"` and `limit` set to `3`. A call whose `arguments` is left out or `nil` gets the empty object `{}` and is sent that way, which is why the `clock` call above reads `type(calls[2].arguments)` as `table`. A Lua `arguments` table is stored as a JSON object with the same keys and values, so `arguments = { query = 'hi' }` is stored as `{ "query": "hi" }`, and [Tool exchanges in a request](#tool-exchanges-in-a-request) shows how it reaches the model. It must be a Lua table, not JSON text, or adding the record fails with `messages[N] tool_calls[K] arguments must be a table`.

`tool_calls` is an array of call tables even for one call, as in `tool_calls = { { id = 'call_1', name = 'web_search' } }`. A single call table placed directly in `tool_calls`, or a string there, is not an array and fails with `messages[N] tool_calls must be an array`. A call with a bad `id`, `name`, or `arguments` is refused with a message naming that field.

One `assistant` record can carry visible reply text in `content` together with several tool calls, as `Let me check both.` does beside `call_1` and `call_2`, and the text and every call are sent to the model together. The calls keep the order they were written in, so `call_1` stays first. A record with `tool_calls` can also have the empty string as its content, as in `{ role = 'assistant', content = '', tool_calls = { ... } }`, and such a record followed by its tool result is accepted. A tool call keeps only `id`, `name`, and `arguments`, and any other field on it is dropped.

`tool_calls` is optional, and leaving it out means the record carries no tool calls. A record the loop writes holds calls only when the model asked for tools in that reply. After such a reply, `msgs[2].tool_calls[1]` is the first call the model made, so `msgs[2].tool_calls[1].id` reads its id, such as `'call_1'`, and `msgs[2].tool_calls[1].name` reads the tool it called, such as `'web_search'`. Reading one entry with `local call = msgs[2].tool_calls[1]` gives you `call.id`, `call.name`, and `call.arguments`, the call id, the name the model called, and the parsed arguments table. A model that sends `{"query":"one"}` gives `msgs[2].tool_calls[1].arguments.query == 'one'`.

Put `tool_calls` only on `assistant` records. Adding a record of another message role with `tool_calls` succeeds, because adding does not check the message role against `tool_calls`, but the next model call over the list fails with `messages[N] sets tool_calls but is not an assistant message`, such as `messages[1] sets tool_calls but is not an assistant message` for a user record at position 1 (see [Cross-record errors](#cross-record-errors)).

### Tool results

A `tool` record answers a call, with the result text in `content` and the call's `id` in `tool_call_id`, and several tool records can follow one `assistant` record, each answering its own call. To answer both calls above, put these lines at the end of that block in place of its `return`:

````lua
msgs:append({ role = 'tool', content = 'Sunny, 24 C', tool_call_id = 'call_1' })
msgs:append({ role = 'tool', content = '14:05', tool_call_id = 'call_2' })
return #msgs .. '|' .. msgs[3].role .. '|' .. msgs[4].tool_call_id
````

````text
4|tool|call_2
````

A tool record is sent to the model as `{ "role": "tool", "content": "...", "tool_call_id": "..." }`. Its `tool_call_id` is a string holding the id of the one call it answers, so a call's `id` is the identifier that the answering record's `tool_call_id` matches. A tool result record reads `role` as `'tool'`, as in `msgs[3].role == 'tool' and msgs[3].tool_call_id == 'call_x'`, and the tool result records the loop writes hold their `content` as a string.

Every tool record sets `tool_call_id`, and `tool_call_id` belongs on tool records only:

- A `tool` record without a string `tool_call_id` is refused with `messages[N] is a tool message and must set a string tool_call_id`.
- A `system`, `user`, or `assistant` record that sets `tool_call_id` is refused with `messages[N] sets a tool_call_id but is not a tool message`.

So `system`, `user`, and `assistant` records leave the field out, and it is never sent for them. The pairing of calls with their results is checked later, when the model call runs: a tool record whose `tool_call_id` is not among the pending call ids of the `assistant` record before it makes the model call fail with a message that calls it an orphan tool record, in the words `is an orphan tool record`. [What the model receives](#what-the-model-receives) covers these cross-record rules.

### The leading system block

System records appear only in the leading system block, the unbroken run of system records at the start of the list. Adding a system record after any other record fails, and the list stays unchanged, as [System records lead the list](#system-records-lead-the-list) shows with the full message.

### Fields a record keeps

A message record has no fields other than `role`, `content`, `tool_calls`, and `tool_call_id`, with `role` and `content` always present and the other two only when they apply. Any other field you set is silently dropped when the record is added. Put this block in the `## Build` section of `records`:

````lua
local msgs = messages.new()
msgs:user('u1'):append({ role = 'user', content = 'u2', extra = 1 })
return tostring(msgs[2].extra) .. '|' .. msgs[2].content
````

````text
nil|u2
````

The record reads back as only `role` and `content`. Only those four fields ever reach the model, so other fields such as `name = "bob"`, `weight = 3`, or `extra = { nested = true }` are dropped, and a credential copied into an extra field, such as `api_key` or `authorization`, never reaches the provider. A record's `name` field is not sent either, so labeling a speaker with `name`, as in `{ role = "user", content = "hi", name = "bob" }`, has no effect on the request.

A record without tool calls or a tool call id has no `tool_calls` or `tool_call_id` field at all, rather than an empty one, so a system record's form is `{ "role": "system", "content": "be terse" }`, with only two keys.

The builders `msgs:system`, `msgs:user`, `msgs:assistant`, and `msgs:tool` make the same validated record that `msgs:append` makes from the equivalent table, because each builder runs the same checks that `append` runs.

## Building a message list

Here you add records with each builder method, chain the calls into one expression, and learn the rules every edit checks.

````markdown
---
name: builders
description: Builds a message list with every builder method.
runvil: 0
---

# Builders

## Build

```lua
local msgs = messages.new()
local same = msgs:system('be terse')
  :user('hi')
  :assistant('working', { { id = 'call_1', name = 'web_search', arguments = { query = 'hi' } } })
  :tool('three results for hi', 'call_1')
  :assistant('done')
  :append({ role = 'user', content = 'thanks' })
return tostring(same == msgs) .. '|' .. #msgs .. '|' .. msgs[6].content
```
````

````text
true|6|thanks
````

One chain of six calls stores six records in call order, a system record, a user record, an assistant record with one tool call, the tool record answering it, a plain assistant record, and a user record, and it hands back the list it was called on.

### The builder methods

A builder method is a list method that appends one record at the end of the list and returns that same list. Every list that `messages.new()` returns has five of them, `system`, `user`, `assistant`, `tool`, and `append`, plus `replace`, which [Editing a list with `replace`](#editing-a-list-with-replace) covers:

| Call | Appends |
|---|---|
| `list:system(content)` | `{ role = "system", content = content }` |
| `list:user(content)` | `{ role = "user", content = content }` |
| `list:assistant(content, tool_calls?)` | `{ role = "assistant", content = content }`, with `tool_calls` set only when it is passed |
| `list:tool(content, tool_call_id)` | `{ role = "tool", content = content, tool_call_id = tool_call_id }` |
| `list:append(record)` | `record`, checked, with only its `role`, `content`, `tool_calls`, and `tool_call_id` kept |

The list methods are colon methods, so the list itself is the first argument: you write `list:user("hi")`, and the colon passes `list` in. Each of the four builders named after a message role sets the new record's `role` from the method you called, `system`, `user`, `assistant`, or `tool`, and leaves out any field you pass as `nil`.

Message lists and model handles are the only Engine values with colon-call methods. A model handle's methods run rounds on its model, as in `h:infer(prompt)` and `h:loop(messages)` (see [Model handles](10-models.md#model-handles), and [How the loop runs](#how-the-loop-runs) for `h:loop`), and no other Engine handle has methods.

### Adding user and system records

`msgs:user(text)` appends one user record holding that text at the end of the list, so on an empty list that record becomes the first message. `list:user(content)` builds `{ role = "user", content = content }` and returns the same list, and the model receives the record with the `user` message role and the same text.

`list:user` accepts any Lua string value, such as a loop variable, not only `prose`. `msgs:user(prose)` appends a user record holding the section prose above the fence, and that text reaches the model as a user record. A list can start with `list:user(text)` and no system message, and it still works as a request to the model.

A builder passes its content on as given and never converts a number to text, so for a number `n` you write `list:user(tostring(n))`. A number as content raises `messages[N] content must be a string or a non-empty array of content parts`.

`msgs:system(text)` appends a system message. `list:system(content)` builds `{ role = "system", content = content }` and returns the same list, so a chain such as `messages.new():system('Write for a reader who has never programmed.'):user(prose)` puts the system record ahead of the user record.

### Adding assistant and tool records

`list:assistant(content, tool_calls?)` appends an assistant record whose `content` is the given text, with an optional `tool_calls` value, and returns the same list. With no second argument, as in `list:assistant('done')`, it appends a plain assistant record with no `tool_calls` field, since `tool_calls` is set only when it is passed. The optional `tool_calls` argument is an array of tool calls, each a table `{ id = ..., name = ..., arguments = {...} }` with a string `id`, a string `name`, and an optional `arguments` table that becomes `{}` when left out or `nil`.

`list:tool(content, tool_call_id)` appends the tool record `{ role = "tool", content = content, tool_call_id = tool_call_id }`, taking the result text first and the id of the earlier tool call it answers second, and returns the same list.

Together they write an earlier tool call and its result into a list: `:assistant` carrying the call, followed by `:tool` with the matching id. Put this block in the `## Build` section of `builders` in place of the first one:

````lua
local msgs = messages.new()
  :user('What is the weather in Paris?')
  :assistant('', { { id = 'call_1', name = 'weather', arguments = { city = 'Paris' } } })
  :tool('Sunny, 24 C', 'call_1')
return #msgs .. '|' .. msgs[3].tool_call_id
````

````text
3|call_1
````

### Adding whole records with `append`

`list:append(record)` adds one record table to the list and returns the list, and the list keeps records in the order they were appended. The table names its own `role` and `content`, such as `{ role = 'system', content = 'be terse' }`, because unlike the other builders, `append` does not fill in a message role for you. You can build a whole conversation this way, by appending Lua record tables with `role` and `content` fields to a `messages.new()` list. Records you wrote as a plain Lua array go in one `append` at a time:

````lua
local records = {
  { role = 'system', content = 'be terse' },
  { role = 'user', content = 'hi' },
}
local msgs = messages.new()
for _, record in ipairs(records) do msgs:append(record) end
msgs:user('more')
return #msgs .. '|' .. msgs[1].role
````

````text
3|system
````

Builder calls and hand-written records passed to `append` mix freely in one list, as `msgs:user('more')` does here, but the list itself always comes from `messages.new()`.

Each `append` call checks its one record against the per-record rules before the record joins the list. It keeps only the record's `role`, `content`, `tool_calls`, and `tool_call_id`, dropping any other keys, and it accepts records that carry extra fields, nested tables included, without error. Every edit does the same inside a record, so a text part keeps only its `text`, an image part only its `url`, and a tool call only its `id`, `name`, and `arguments`, and a dropped field cannot be read back from the list and never reaches the model. Records that pass the rules append normally, so a list built by successive `append` calls keeps every valid record, and only the call with a bad record fails.

### Chaining builder calls

Every builder method, `append` included, changes the list and returns that same list, and `replace` returns the list too, so builder calls chain to any length and can start straight off `messages.new()`, as in `messages.new():system("rules"):user("question")`. A chain such as `msgs:user('u1'):assistant('a1'):user('u2')` adds three messages in one expression. Each call appends one record at the end of the list, so records sit in the list in the order your calls added them.

You keep adding records to a list after building it, and each call grows that same list. Because each builder call changes the list itself, you can add records in separate statements or inside a Lua loop, as in `for _, review in ipairs(reviews) do msgs:user(review) end`.

Chaining also puts worked examples, such as a user record followed by an assistant record, ahead of the real request:

````markdown
---
name: translate
description: Translates a phrase after one worked example.
runvil: 0
models:
  writer: {}
---

# Translate

## Phrase

good night

```lua
models.use('writer')
local msgs = messages.new()
  :system('Translate each phrase into French.')
  :user('good morning')
  :assistant('bonjour')
  :user(prose)
models.loop(msgs)
return msgs[#msgs].content
```
````

The model sees one worked pair, `good morning` and `bonjour`, before the section prose, and the run result is its translation of `good night`.

### System records lead the list

System records, when present, open the list and belong only in its leading block. You may stack several of them at the front, as `msgs:system('s1'):system('s2'):user('u')` does, and user records can be added at any point. `system`, or `append` with a system record, works only while the list holds no record of another message role. This block catches the error a later system record raises:

````markdown
---
name: system-first
description: Shows that system records lead the list.
runvil: 0
---

# System first

## Build

```lua
local msgs = messages.new():system('s1'):system('s2'):user('u')
local ok, err = pcall(function() msgs:system('late') end)
return tostring(ok) .. '|' .. err.kind .. '|' .. #msgs
```
````

````text
false|lua|3
````

The error value (see [Error values](05-lua-environment.md#error-values)) has kind `lua`, and the list still holds its three records. Its message names the position the record would take, `messages[4]`:

````text
messages[4] is a system message after a non-system message; system messages lead the list, so change the leading block with replace(first, last, records...)
````

`list:append(record)` with a system record after a non-system record raises the same error. Any builder, `append`, or `replace` call that would leave a system record after a record of another message role fails, names the position of the first such record, and leaves the list unchanged. To change the leading block itself, use `replace` (see [Editing a list with `replace`](#editing-a-list-with-replace)).

### When records are checked

Each builder checks the record it adds, including the record's own fields, their types, and system placement, so a record that breaks a rule raises an error at the builder call that adds it and never enters the list. A bad record therefore fails at the `:user`, `:append`, or `:replace` call that adds it rather than at `models.loop`. Every edit checks each record it adds as the record at its target position, and a record that fails raises an error carrying the reason it failed.

A list checks each record's own fields, and system placement, once, when you add the record. It leaves the rules about how records relate to each other, such as unique tool call ids and pairing each call with its result, until the model call (see [What the model receives](#what-the-model-receives)).

Every record a list edit refuses raises a `lua`-kind error value at your call, which `pcall` catches (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)), and the list stays exactly as it was before the call. A value refused for JSON conversion leaves the list unchanged too. The error's message is the refusal text itself, exactly as [Record error messages](#record-error-messages) shows it, with no prefix, and only the first broken rule is reported.

### Positions in a record error

Every record error a list raises names the offending record by its 1-based Lua index, written `messages[N]`. N is the position the record takes in the list as it would stand after the edit, so a bad first record on an empty list is `messages[1]`, a bad record after one valid record is `messages[2]`, and appending to a three-record list names `messages[4]`. The first bad record is the one reported, in argument order for a `replace` that adds several records, and in list order at the `models.loop` call.

Errors inside a record add a second position:

- A content-part error names the part by its 1-based position within the record's content array, written `messages[N] content part J`, so the second part is reported as `content part 2`. The first bad part is the one reported.
- A tool-call error names the call by its 1-based position within the record's `tool_calls` array, written `messages[N] tool_calls[K]`, so a bad first call on the first record is `messages[1] tool_calls[1]`.

### The order of the record checks

A record's rules run in a fixed order, so you see the first rule the record breaks in that order:

1. JSON form. Every value in a record, including values in extra fields that get dropped, must be JSON-representable: strings, numbers, booleans, and nested tables. This check runs first, so a record holding a function anywhere fails with it, whatever else is wrong. A builder converts each non-nil argument from Lua to JSON first, so a field value can be any JSON-representable Lua value, which the record check then accepts or refuses.
2. Table shape. The record must be a table.
3. Role.
4. Content, including each content part.
5. The type of `tool_call_id`. This check is reported ahead of the check that only tool records carry a `tool_call_id`, so a record with a non-string `tool_call_id` reports the type message, even on a `user` record, and a tool record with a non-string `tool_call_id` reports the type message too.
6. Whether the message role and `tool_call_id` go together.
7. `tool_calls`, and within each call, `id` before `name`, so a call missing both reports the `id` message.

A record that passes these rules then joins the list only if it keeps the system records leading.

### Record error messages

Every message below is the full text of a `lua`-kind error value. N, J, and K are the positions [Positions in a record error](#positions-in-a-record-error) describes, and the parts in braces are filled in as each entry says.

JSON form:

- `messages[N] must be a JSON-representable message table: {reason}` comes from a record given to `append` or `replace` that has no JSON form, meaning it holds a value that is not a string, number, boolean, or nested table, even in a field the list drops.
- `messages[N] {field} must be JSON-representable: {reason}` comes from a builder argument that has no JSON form, where `{field}` names `content`, `tool_calls`, or `tool_call_id`, and N is `#list + 1`.

The `{reason}` is the converter's own explanation of why the value has no JSON form, such as ``unsupported value type `function` ``. On a list holding one record, these messages read as follows: the first for a whole record that JSON cannot represent, or one with such a field inside it, passed to `append` or `replace`, and the others for a function in a tool call's arguments, as a `tool_call_id`, and as content or inside a content table:

````text
messages[2] must be a JSON-representable message table: unsupported value type `function`
messages[2] tool_calls must be JSON-representable: unsupported value type `function`
messages[2] tool_call_id must be JSON-representable: unsupported value type `function`
messages[2] content must be JSON-representable: unsupported value type `function`
````

Table shape and message role:

- `messages[N] must be a message table` comes from a record that is not a table. Added to an empty list it reads `messages[1] must be a message table`, and appended to a one-record list it reads `messages[2] must be a message table`.
- `messages[N] role must be a string, one of: system, user, assistant, tool` comes from a record whose `role` is missing or not a string.
- `messages[N] role "{role}" is unknown; known roles: system, user, assistant, tool` comes from a `role` string that is not one of the four, quoted in double quotes. The known-roles list is always the four message roles joined with a comma and a space.

Content:

- `messages[N] content must be a string or a non-empty array of content parts` comes from content that is missing, an empty array, or neither a string nor an array. `list:user()` with no content on an empty list raises `messages[1] content must be a string or a non-empty array of content parts`.
- `messages[N] content part J must be a table with a string type field` comes from a part that is not a table, or that lacks a string `type` field. A bare string inside the first record's content array raises `messages[1] content part 1 must be a table with a string type field`.
- `messages[N] content part J has unknown type "{type}"; known types: text, image_url` comes from a part whose `type` is neither `text` nor `image_url`, quoted in double quotes.
- `messages[N] content part J is a text part and must set a string text field` comes from a `text` part without a string `text` field.
- `messages[N] content part J is an image_url part and must set an image_url table with a string url field` comes from an `image_url` part with no `image_url` table, or whose table sets other keys but no string `url`.

Tool call ids:

- `messages[N] tool_call_id must be a string` comes from a `tool_call_id` that is present but not a string, such as `messages[1] tool_call_id must be a string` on a first record.
- `messages[N] is a tool message and must set a string tool_call_id` comes from a `tool` record without a string `tool_call_id`. `list:tool(text)` with no id, on a list holding one record, raises `messages[2] is a tool message and must set a string tool_call_id`.
- `messages[N] sets a tool_call_id but is not a tool message` comes from a `system`, `user`, or `assistant` record that sets `tool_call_id`.

Tool calls:

- `messages[N] tool_calls must be an array` comes from a `tool_calls` value that is present but not an array.
- `messages[N] tool_calls[K] must be a table` comes from a tool call that is not a table, such as `messages[1] tool_calls[1] must be a table`.
- `messages[N] tool_calls[K] must set a string id` comes from a call without a string `id`, such as `messages[2] tool_calls[1] must set a string id`.
- `messages[N] tool_calls[K] must set a string name` comes from a call without a string `name`, such as `messages[1] tool_calls[1] must set a string name`.
- `messages[N] tool_calls[K] arguments must be a table` comes from `arguments` that is anything other than a keyed table, a string or a non-empty sequence included.

System placement:

- `messages[N] is a system message after a non-system message; system messages lead the list, so change the leading block with replace(first, last, records...)` comes from a system record added after any non-system record, where N is the new record's position.

One rule spans records, so the edit that adds a record does not check it: only `assistant` records carry `tool_calls`, and a `tool_calls` array on a record of another message role fails at the `models.loop` call instead, with the message [Cross-record errors](#cross-record-errors) lists.

## Reading records

Here you read a list's length, its records by position, and each record's fields, and you walk a list with `pairs` and `ipairs`.

````markdown
---
name: read-records
description: Reads a message list by length, position, and walk.
runvil: 0
---

# Read records

## Read

```lua
local msgs = messages.new():user('u1'):assistant('a1'):user('u2')
local seen = {}
for i, record in pairs(msgs) do
  seen[#seen + 1] = i .. ':' .. record.role .. ':' .. record.content
end
return #msgs .. '|' .. msgs[#msgs].content .. '|' .. table.concat(seen, ' ')
```
````

````text
3|u2|1:user:u1 2:assistant:a1 3:user:u2
````

`#msgs` counts the records, `msgs[#msgs]` reads the last one, and `pairs(msgs)` visits each record in order with its position. These are the forms for reading a list:

| Form | Reads |
|---|---|
| `#list` | The number of records in the list |
| `list[i]` | A read-only view of record `i`, or `nil` past the end |
| `record.role` | `system`, `user`, `assistant`, or `tool` |
| `record.content` | A string, or a non-empty array of content parts |
| `record.tool_calls` | On an `assistant` record with calls, an array of `{ id, name, arguments }` entries, otherwise `nil` |
| `record.tool_call_id` | On a `tool` record, the string `id` of the call it answers, otherwise `nil` |
| `pairs(list)`, `ipairs(list)` | Each 1-based position and a view of the record there, in order |

### The length of a list

The length operator `#msgs` gives the number of records in a message list. It is `0` for a new, empty list, so `#l` is `0` right after `local l = messages.new()`. It is `1` after one `:user` call and `2` after `msgs:user('u'):assistant('a')`.

The length counts the records `models.loop` appends as well. It is `2` after a loop adds one reply to one user message, `4` after two user messages and the two replies the loops appended, and `8` after four user records and four rounds, and it is `1` again once the reply is removed from a two-record list. A list that held five records when the loop's last round was sent reads `#msgs` as `6` after the final reply `bye` joins it.

### Reading a record by position

You read a single record from a list by its 1-based index with `msgs[i]`, as in `msgs[2].tool_calls`, `msgs[3].role`, or `return msgs[#msgs].content`. `list[i]` returns the message record at position `i`, and you read its fields such as `role` and `content` directly, as in `history[2].content` or `fresh[1].role .. ':' .. fresh[1].content`.

Because `#list` gives the number of records, it names the position of the last one, so `list[#list]` is the latest message, such as the reply `models.loop` just appended. A block ending with `models.loop(msgs)` and then `return msgs[#msgs].content` returns the model's reply, such as `pong`. The same position works in other calls: after `msgs:system('s'):user('u'):assistant('a')`, the call `msgs:replace(#msgs, #msgs)` targets the third record (see [Editing a list with `replace`](#editing-a-list-with-replace)). `msgs[i]` also works inside a numeric `for` loop, as in `for i = 3, 6 do ids[#ids + 1] = msgs[i].tool_call_id end`.

A position outside the list reads `nil` instead of raising an error. Put this block in the `## Read` section of `read-records` in place of the first one:

````lua
local msgs = messages.new():user('u1'):assistant('a1')
return tostring(msgs[0]) .. '|' .. tostring(msgs[-1]) .. '|' .. tostring(msgs[3]) .. '|' .. tostring(msgs[1.5]) .. '|' .. msgs[2.0].content
````

````text
nil|nil|nil|nil|a1
````

On this two-record list, position 0, a negative index, and an index past the last record all read `nil`, and so does any non-integral number, such as `1.5`. An empty list reads `nil` at position 1, so `l[1] == nil` is `true` there. An integral float such as `2.0` reads the same record as `2`, because an index follows the same integer rule as the `replace` bounds. A key that is neither a method name nor a record position, such as a string that names no method or a boolean, reads `nil` as well.

### Reading a record's fields

Each record you read gives you its `role`, `content`, `tool_calls`, and `tool_call_id`. Put this block in the `## Read` section of `read-records`:

````lua
local msgs = messages.new()
  :user('say hi')
  :assistant('', { { id = 'call_1', name = 'web_search', arguments = { query = 'one' } } })
  :tool('three results for one', 'call_1')
  :assistant('done')
local roles = {}
for i = 1, #msgs do roles[i] = msgs[i].role end
local call = msgs[2].tool_calls[1]
return table.concat(roles, ',') .. '|' .. call.name .. ':' .. call.arguments.query .. '|' .. msgs[3].tool_call_id .. '|' .. tostring(msgs[4].tool_calls)
````

````text
user,assistant,tool,assistant|web_search:one|call_1|nil
````

- `record.role` returns the record's message role as a string, `system`, `user`, `assistant`, or `tool`, as in `msgs[#msgs].role == 'assistant'`. A system or user record's `role` names who spoke, `"system"` or `"user"`.
- `record.content` returns a string or a non-empty array of content parts. `msgs[i].content` reads a message's text, as in `msgs[2].content == 'final answer'` checking a reply, and for a tool result it is the tool's answer text, such as the answer from the `task_status` tool (see [Tool scope and the task built-ins](12-tools.md#tool-scope-and-the-task-built-ins)).
- `record.tool_calls`, on an `assistant` record that holds tool calls, returns an array of `{ id, name, arguments }` entries, where `arguments` is always a table, `{}` when the call set no `arguments` or set `arguments = nil`. In the block above, `msgs[2].tool_calls[1].id == 'call_1'`, `.name == 'web_search'`, and `.arguments.query == 'one'`, and a record the loop writes reads the same way.
- `record.tool_call_id`, on a `tool` record, returns the string `id` of the call it answers, as in `msgs[3].tool_call_id == 'call_1'`.

A system or user record holds only `role` and `content`, with `content` the text given to the builder method, so `:system('s')` reads as `{ "role": "system", "content": "s" }`, and its `tool_calls` and `tool_call_id` read `nil`. A record without tool calls reads `nil` for `tool_calls`, not an empty list.

A counted loop such as `for i = 1, #msgs do roles[i] = msgs[i].role end` reads every record's message role, and the message roles a loop leaves are `user`, `assistant`, and `tool`. One run gives the message roles `user,assistant,tool,assistant,tool,user,assistant`, where the second `user` is a task notice the loop appended (see [`await_tasks` and task notices](15-tasks.md#await_tasks-and-task-notices)). Reading such a record as `msgs[5].role .. ':' .. msgs[5].content` gives text that starts `user:Task id=0.0`.

### Walking a list

A counted loop, `ipairs`, and `pairs` all visit a list's records in order, each giving a 1-based position and a record view, the same read-only views that `list[i]` returns. You read each record's fields, such as `role`, `content`, and `tool_call_id`, directly, as in `for i, record in ipairs(msgs) do roles[i] = record.role end` or `for _, m in ipairs(msgs) do`. All three walk the same records:

- `for i = 1, #list do ... list[i] ... end` visits every record in order, because `#list` is the record count.
- `ipairs(msgs)` visits every record in order, giving the index and a record view.
- `pairs(list)` walks every record in order, starting at position 1 and yielding each position and a view of the record there.

A list's length, its numeric indexes, and a `pairs` walk over it cover only its records, never its methods, so `pairs(msgs)` over three records visits exactly positions 1, 2, and 3. Iterating an empty list with `pairs`, `ipairs`, or a counted loop visits nothing.

The two iterators differ when the loop edits the list. `pairs(list)` stops at the record count the list had when the walk began, as the sandbox `pairs` fixes a table's keys at its start, so a loop that appends on every pass still ends. Put this block in the `## Read` section of `read-records`:

````lua
local msgs = messages.new():user('u1'):assistant('a1'):user('u2')
local n = 0
for _ in pairs(msgs) do
  n = n + 1
  msgs:user('x')
end
return n .. '|' .. #msgs
````

````text
3|6
````

The walk ends after three passes, with `#msgs` at `6`. `ipairs(list)` reads the live list instead, so records appended during the walk are visited too, and a loop that appends a record on each pass keeps going as long as it keeps appending.

`pairs(list)` also reads the live list at every step. It stops early when the list shrinks below the next position it would visit, so `msgs:replace(2, 3)` run at `i == 1` on a three-record list ends the walk after position 1, and a record replaced ahead of the walk shows its new content. This block runs the same editing loop under both iterators:

````lua
local function walk(iter)
  local msgs = messages.new():user('u1'):assistant('a1'):user('u2')
  local seen = {}
  for i, record in iter(msgs) do
    if i == 1 then
      msgs:replace(3, 3, { role = 'user', content = 'u2b' })
      msgs:user('u3')
    end
    seen[#seen + 1] = i .. ':' .. record.content
  end
  return table.concat(seen, ' ')
end
return walk(pairs) .. '|' .. walk(ipairs)
````

````text
1:u1 2:a1 3:u2b|1:u1 2:a1 3:u2b 4:u3
````

Both walks read the replaced record at position 3 as `u2b`, and only `ipairs` visits the record appended at position 4.

### What a record view allows

A record view is what `list[i]` returns: a read-only view of the record at 1-based position `i`, whose fields you read through metamethods, and `pairs` and `ipairs` hand out the same views. Put this block in the `## Read` section of `read-records`:

````lua
local msgs = messages.new()
  :user('say hi')
  :assistant('working', { { id = 'call_1', name = 'web_search', arguments = { query = 'hi' } } })
local view = msgs[2]
local calls = view.tool_calls
calls[1].arguments.query = 'changed'
return type(view) .. '|' .. view.tool_calls[1].arguments.query .. '|' .. tostring(view.tool_calls == view.tool_calls)
````

````text
userdata|hi|false
````

A view's Lua type is `userdata`, not `table`, so `type(msgs[i])` reads `"userdata"` for every record shape, whether you read the record with `msgs[i]` or while iterating. Each read of `content` or `tool_calls` builds a fresh Lua table, converted from the record's JSON form, so changing the returned table, nested `arguments` included, leaves the list unchanged. Two reads of the same table field return two distinct tables, which is why `view.tool_calls == view.tool_calls` is `false`, and reading the field from the same view again after changing an earlier copy returns the original value, so the view reads its original content and its original tool calls again.

The four fields read like the fields of the record table you appended, value for value, including `nil` for a field the record does not hold:

- `view.content` returns the record's content as a Lua value: a string for string content, and a table for parts content.
- `view.tool_calls` returns the record's tool calls as a table, or `nil` when the record carries none, so a record without tool calls reads `nil` there, as `assert(m.tool_calls == nil, ...)` checks for a user record.
- `view.tool_call_id` returns the tool call id string the record carries, or `nil` when it carries none.

Any other key reads `nil` instead of raising an error. An unknown field such as `extra` reads `nil` even when the appended table set `extra = 1`. Field names match exactly, byte for byte, so a key in a different case, such as `Role`, reads `nil`. A non-string key such as `1` reads `nil` too, so a record view does not behave like an array.

A view holds its own reference to the record it was read from, so it keeps showing that record's fields rather than looking the record up in the list again. It keeps showing them even after a later `replace` changes that position, because an edit puts new records into the list instead of changing records already handed out.

`pairs(view)` visits only the fields the record has, in the fixed order `role`, `content`, `tool_calls`, `tool_call_id`, the same order on every run, skipping absent fields and ending after the last present one. It yields each field name as the key and the same value that dot access returns, including a fresh table for a table field. Put this block in the `## Read` section of `read-records`:

````lua
local msgs = messages.new()
  :user('say hi')
  :assistant('working', { { id = 'call_1', name = 'web_search' } })
  :tool('three results', 'call_1')
local function fields(view)
  local keys = {}
  for key in pairs(view) do keys[#keys + 1] = key end
  return table.concat(keys, ',')
end
return fields(msgs[2]) .. '|' .. fields(msgs[3])
````

````text
role,content,tool_calls|role,content,tool_call_id
````

### Using a view as a record

A view converts to the same record value as the table it was built from, so it works as the record argument of `append` and `replace`. Put this block in the `## Read` section of `read-records`:

````lua
local msgs = messages.new():system('s'):user('u')
local other = messages.new()
for i = 1, #msgs do other:append(msgs[i]) end
var.last = other[#other].content
return #other .. '|' .. var.last
````

````text
2|u
````

`other:append(msgs[i])` copies the same record into another list, so the loop leaves `other` holding the same records as `msgs`. A view also converts to the same JSON form as the record it stands for, so inside a table, or as an `append` or `replace` argument, it converts like the equivalent record table, and `{ msgs[1], msgs[2] }` converts to `[{ "role": "system", "content": "s" }, { "role": "user", "content": "u" }]`.

Store or insert a record's fields, such as `msgs[i].content`, rather than the view itself, as the block above does with `var.last` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)). The guards that refuse userdata still refuse a bare view:

- Assigning a view straight to a `var` field fails with `var.{name} must be JSON data, got userdata`.
- A prose reference to a global holding a view (see [Inserting Lua globals by bare name](07-substitution.md#inserting-lua-globals-by-bare-name)) fails with ``global `{name}` is a userdata; bare globals in prose must be JSON data``.

## Editing a list with `replace`

Here you rewrite, remove, insert, and merge the records of a message list with `replace`, and learn the rules every edit follows.

````markdown
---
name: edit-list
description: Rewrites one record of a message list.
runvil: 0
---

# Edit list

## Edit

```lua
local msgs = messages.new()
msgs:user('u1')
msgs:append({ role = 'assistant', content = 'a1' })
msgs:user('u2')
msgs:append({ role = 'assistant', content = 'a2' })
msgs:replace(3, 3, { role = 'user', content = 'u2b' })
local out = {}
for i = 1, #msgs do out[i] = msgs[i].content end
return table.concat(out, ' ')
```
````

````text
u1 a1 u2b a2
````

The block builds the four records `u1 a1 u2 a2`, then rewrites the third one. `replace` is the list method that changes records already in a list, while `append` adds a record at the end (see [Building a message list](#building-a-message-list)). `list:replace(first, last, records...)` replaces the records at 1-based positions `first` through `last`, inclusive, with the records that follow, and returns the list. Here the range is position 3 alone, which holds `u2`, and the one new record takes its place, so record 3 reads `u2b` and every other record stays where it is.

Each new record is a message record table (see [Message records](#message-records)). Its `role` field names the message role of the speaker, as in `{ role = 'user', content = 'changed' }`, and its `content` field holds the message text, as in `content = 'changed'`. To swap the last message for a short user message, pass the last position as both bounds:

````lua
msgs:replace(#msgs, #msgs, { role = 'user', content = 'zz' })
````

The records after the range may be none, and their count need not match the size of the range. The examples below start from the same four records `u1 a1 u2 a2`, with only the `replace` line of `edit-list` changed.

### Removing records with `replace`

Call `replace` with a range and no records to delete that range:

````lua
msgs:replace(2, 3)
````

````text
u1 a2
````

`list:replace(first, last)` with no records removes records `first` through `last` and keeps every record before `first` and after `last`. Passing fewer records than the range holds shrinks the list the same way, so replacing four records with two leaves a two-record list.

`msgs:replace(#msgs, #msgs)` with no records removes the last record, such as a trailing assistant reply, and every earlier record stays in place. The loop always leaves its terminal record in the list, so this is how you drop that record when you do not want it kept. Put this block in the `## Chat` section of `first-chat`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
models.loop(msgs)
msgs:replace(#msgs, #msgs)
return #msgs
````

````text
1
````

The loop leaves two records, and removing the terminal record shrinks the list back to the one user record. Because every earlier record stays in place, a list of a system record, a user record, and an assistant reply keeps its first 2 records when you drop the reply, so the next request starts with the same 2 messages as the 3-message request before it.

### Inserting records with `replace`

An empty range inserts records without removing any. When `first` is one more than `last`, the range holds no records, so `replace` keeps every old record and puts the new records before position `first`:

| Call on `u1 a1 u2 a2` | Records after |
| --- | --- |
| `msgs:replace(3, 2, { role = 'user', content = 'mid' })` | `u1 a1 mid u2 a2` |
| `msgs:replace(1, 0, { role = 'system', content = 'front' })` | `front u1 a1 u2 a2` |
| `msgs:replace(#msgs + 1, #msgs, { role = 'user', content = 'end' })` | `u1 a1 u2 a2 end` |

`list:replace(first, first - 1, records...)` inserts before position `first`, so `msgs:replace(3, 2, ...)` puts `mid` between `a1` and `u2`. `list:replace(1, 0, records...)` inserts at the front of the list, ahead of `u1`, which is how you put a system record first, as in `msgs:replace(1, 0, { role = 'system', content = 'Be brief.' })`. `list:replace(#list + 1, #list, records...)` adds records at the end of the list, the same as `append`, so `end` becomes the last record.

### Replacing a whole range

`list:replace(1, #list, records...)` swaps out every record, and the number of new records may differ from the size of the range:

````lua
msgs:replace(1, #msgs, { role = 'system', content = 's' }, { role = 'user', content = 'summary' })
````

````text
s summary
````

The four records `u1 a1 u2 a2` give way to two: a system record `s` and a user record `summary`.

Replacing a range with a single record merges the range. On a list whose records 2 to 4 are a user record, an assistant record, and a user record, `msgs:replace(2, 4, { role = 'user', content = 'one and two' })` merges the three into one user record at position 2, and a loop run on the list afterward still sends its next round.

### Editing the leading system block

`replace` is how you change or remove the [leading system block](#message-records), the system records at the front of a list. A replace may rewrite or grow that block, as long as every system record stays ahead of the first non-system record. Put this block in the `## Edit` section of `edit-list` in place of the first one:

````lua
local msgs = messages.new()
msgs:append({ role = 'system', content = 's1' })
msgs:append({ role = 'system', content = 's2' })
msgs:user('u')
msgs:replace(1, 2,
  { role = 'system', content = 'one' },
  { role = 'system', content = 'two' },
  { role = 'system', content = 'three' })
local out = {}
for i = 1, #msgs do out[i] = msgs[i].content end
return table.concat(out, ' ')
````

````text
one two three u
````

The two system records at positions 1 and 2 give way to three, and the user record moves to position 4, still behind every system record.

To rewrite the system message, replace position 1 with a record whose `role` is `system`, as in `list:replace(1, 1, { role = "system", content = "new rules" })`. This works even after a loop has sent the list, so `msgs:replace(1, 1, { role = 'system', content = 'other' })` between two loop calls changes the instructions the next call sends.

The leading-block rule is checked on the whole edited list. An edit that would leave a system record behind a non-system record raises the late-system error (see [Building a message list](#building-a-message-list)), which names the position of the first late record after the edit and advises `change the leading block with replace(first, last, records...)`. On the three records `s1 s2 u`:

- `msgs:replace(4, 3, { role = 'system', content = 'late' })` would append a system record after `u`, so it is refused naming `messages[4]`.
- `msgs:replace(1, 1, { role = 'user', content = 'u0' })` would leave `s2` behind the new user record, so it is refused naming `messages[2]`, where `s2` would land.

A refused late-system edit leaves the list unchanged.

### Editing a list between rounds

You can edit a list with `replace` after a loop has sent it, and the next request uses the edited messages. After `msgs:replace(4, 4, { role = 'user', content = 'changed' })` on a list that a loop has already sent, the next request matches the earlier one through position 3 and differs from it starting at position 4.

### Editing a record through its view

A [record view](#reading-records), the value `msgs[i]` reads, is read-only, so you change a record by rewriting it in the list with `replace`, building the new record from the view's fields. Put this block in the `## Chat` section of `first-chat`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
models.loop(msgs)
local last = msgs[#msgs]
msgs:replace(#msgs, #msgs, { role = last.role, content = last.content .. ' (edited)' })
return msgs[#msgs].content
````

The run result is the model's reply followed by ` (edited)`. Every view field, `role` and `content` included, reads the same value as the record it comes from, so `last.role` is `'assistant'` and `last.content` is the reply text.

A view also works as the record argument of `replace` and of `append`. A view saved in a local still holds its record after `replace` removes that record from the list, so these three lines put the same record back at the front:

````lua
local first = msgs[1]
msgs:replace(1, 1)
msgs:replace(1, 0, first)
````

Assigning to any field of a view raises an error whose message names `replace`, the method to use instead:

````text
records read from a messages.new() list are read-only; change the list with replace(first, last, records...)
````

The same error comes from assigning a record field (`role`, `content`, `tool_calls`, or `tool_call_id`), a key that is not a record field such as `extra`, or `nil`, so you cannot attach extra data to a view. A refused assignment to a view leaves the list unchanged.

The list itself takes no assignment either. Assigning to any index or field of a list, including `msgs[#msgs] = nil`, raises the error [A list belongs to the Engine](#a-list-belongs-to-the-engine) shows, which points you to `append` and `replace`. Assigning `nil` is refused like any other value, and a refused assignment leaves the list unchanged. To drop the last record, call `msgs:replace(#msgs, #msgs)`.

### The `replace` range rule

A `replace` range is valid when `1 <= first <= last + 1 <= #list + 1`, with whole-number bounds. The rule covers three cases:

- `first <= last` replaces records `first` through `last`.
- `first = last + 1` is an empty range, which inserts before position `first`.
- `first = #list + 1` with `last = #list` is the empty range one past the end, where `replace` appends.

A bound can be an integer or a float with no fractional part, which works as its integer, so `2.0` counts as `2`. `msgs:replace(2.0, 4.0)` on `u1 a1 u2 a2` removes records 2 through 4 and leaves only `u1`.

### When `replace` refuses an edit

A refused `replace` raises a `lua`-kind [error value](05-lua-environment.md#error-values) at the call and leaves the list unchanged. `replace` applies all of its records or none, so one bad record in the call leaves the list untouched. It checks `first`, then `last`, then the range, before it looks at any record, so a bad range is reported even when the records are also bad.

A bound that is not a whole number fails with `replace first must be an integer, got ...`, or the same message naming `last`. A number with a fraction shows its value, and any other value shows its Lua type:

| Call | Message |
| --- | --- |
| `msgs:replace(1.5, 2)` | `replace first must be an integer, got 1.5` |
| `msgs:replace(1, '2')` | `replace last must be an integer, got string` |
| `msgs:replace(1)` | `replace last must be an integer, got nil` |

A numeric string is not converted to a number, so `'2'` reports its type, `string`, and a missing `last` reports `nil`.

A range outside the rule fails with this message, which names both bounds as you write them, the record count, and the allowed range, with `{n + 1}` written out as a number:

````text
replace({first}, {last}) is out of bounds on a list of {n} records: it needs 1 <= first <= last + 1 <= {n + 1}
````

On the four records `u1 a1 u2 a2`, `msgs:replace(2, 5)` raises:

````text
replace(2, 5) is out of bounds on a list of 4 records: it needs 1 <= first <= last + 1 <= 5
````

Each part of the rule fails in its own way on those four records:

- `first` starts at position 1 or later, so `msgs:replace(0, 0)` and `msgs:replace(-1, 2)` are out of bounds. A negative bound appears in the message as written, as in `replace(-1, 0)`.
- `first` is at most `last + 1`, so the reversed range in `msgs:replace(3, 1)` raises `replace(3, 1) is out of bounds on a list of 4 records`.
- `first` is at most `#list + 1`, so `msgs:replace(6, 5)` is out of bounds.
- `last` is at most `#list`, so `msgs:replace(2, 5)` is out of bounds.

`replace` checks each new record on its own at the position it takes, the first new record at `first`, the next at `first + 1`, and so on, the same way `append` checks one. A bad record raises an error naming that position:

````lua
msgs:replace(1, 1, { role = 'user', content = 'ok' }, { role = 'user' })
````

````text
messages[2] content must be a string or a non-empty array of content parts
````

The second new record takes position 2 and has no `content`, so the call is refused, and the valid first record is not applied either. A refused bound, an out-of-bounds range, and a refused record each leave the list unchanged.

## How the loop runs

Here you follow one `models.loop` call round by round, run a conversation on a chosen model with `handle:loop`, and see what the loop returns and what each round reports. Put this block in the `## Chat` section of `first-chat`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
models.loop(msgs)
return #msgs .. ': ' .. msgs[1].role .. ', ' .. msgs[2].role
````

````text
2: user, assistant
````

In a section, you call `models.loop` to run a model-tool loop over a message list you own. The loop runs the conversation as a series of rounds: each round sends the message list to the model, and the loop judges the answer until it can return or must raise an error such as `empty_model_reply` (see [Empty replies and finish reasons](#empty-replies-and-finish-reasons)). This call goes through these steps:

1. A valid call opens by sending your message list to the model, so the first thing the loop does is send a round.
2. No tools are in scope for the section, so the loop offers the model no tools.
3. The model replies with text and no tool calls, so the loop appends one `assistant` record holding the reply text.
4. The loop returns after that single round, leaving the user record and the terminal record, so `#msgs` is 2.

When the model asks for tools instead, the loop runs them and sends another round, as [Tool rounds and the round cap](#tool-rounds-and-the-round-cap) shows.

### One round and one reply

When the model gives a non-empty text reply with no tool calls, the loop sends one request and appends `{ role = "assistant", content = reply }`, with no tool calls, as the one terminal record. The terminal record comes after the records of every tool round, and its `content` holds the model's reply text verbatim, so the loop's final reply is always `msgs[#msgs].content`. Every record the loop adds has plain text `content`.

A round like this ends `models.loop` normally and hands control back to the block, which goes on with its next statement. `models.loop` and `handle:loop` return `nil` when the loop ends normally, so you read the model's final reply and the rest of the outcome from the list the loop has updated. Each returns exactly one value:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
return select('#', models.loop(msgs)) .. '|' .. #msgs
````

````text
1|2
````

### The loop appends to your list

You pass `models.loop` a message list built with `messages.new()`, and that list is the one each round's request sends as its messages. The loop works on your own list rather than a copy. It appends every record it makes, model replies and tool replies alike, to that list in place, so after the call you read every record of the conversation from the same variable.

Every record added during `models.loop` appears in your list: assistant replies, tool results, and task notices (see [`await_tasks` and task notices](15-tasks.md#await_tasks-and-task-notices)), which arrive as `user` records whose text starts like `Task id=0.0 (## Child) completed: `.

### What each round sends

Each round is one stateless, tool-capable model request over your whole message list as it stands at that moment, including every record that earlier rounds and earlier calls have added. A round after a tool call therefore resends the whole conversation so far, with the message roles `user`, `assistant`, and `tool`. In one call with three one-call tool rounds before the final reply, the four requests hold 1, 3, 5, and 7 messages.

A loop round is the same request a nested `infer` makes (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)), only sent over your conversation together with the tools on offer for that round. Each round offers the model the tools in scope at the moment that round is sent (see [Tool scope and the task built-ins](12-tools.md#tool-scope-and-the-task-built-ins)), so a tool offered with `tools.offer` (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)) or `tools.offer_local` (see [Defining local tools with `tools.offer_local`](12-tools.md#defining-local-tools-with-toolsoffer_local)) between two `models.loop` calls is offered only to the later call.

The model's sampling options (see [Sampling options on `models.use`](10-models.md#sampling-options-on-modelsuse)), such as `temperature`, `max_tokens`, and `thinking`, apply to every round of the loop. Each round is sent with the options of the model it runs on, the conversation so far, and the tools on offer.

Before every round, the loop checks the whole list again against the [cross-record rules](#checking-and-composing-before-each-request), such as no duplicate tool-call id, no orphan tool record, and no assistant tool call without a result. It then builds the request from the records the list holds at that moment, so a list you edit between calls is checked each time. The per-record checks that a builder method such as `msgs:user` runs when it adds a record do not run again.

### How the loop judges an answer

Each round that returns an answer ends in exactly one of four outcomes the loop acts on, and the loop judges them in this fixed order:

1. A context overflow, which the compactor handles (see [Compactors and context exhaustion](#compactors-and-context-exhaustion)).
2. A request for tool calls, which starts a tool round (see [Tool rounds and the round cap](#tool-rounds-and-the-round-cap)).
3. A non-empty text reply, which the loop appends as the terminal record before it returns.
4. The clean exit: an empty reply that finishes with `stop` after at least one answered tool call (see [Empty replies and finish reasons](#empty-replies-and-finish-reasons)).

Any other answer raises an `empty_model_reply` error. Two more rules shape every call: `models.loop` and `handle:loop` stop at the round cap (see [Setting the round cap](#setting-the-round-cap)), and they use `compactors.fail` when no compactor is passed (see [Choosing a compactor for a call](#choosing-a-compactor-for-a-call)).

While the run is being cancelled, an error that a local tool handler or a compactor raises is raised again at once, unchanged, so the cancellation ends the conversation. A handler that returns normally during the cancellation still has its result reported.

### Running the loop again on the same list

You can reuse one message list across several `models.loop` calls, appending a new user record between them, and each call continues the conversation by sending what earlier calls leave in the list, earlier replies included. A call wrapped as `pcall(function() return models.loop(history) end)` runs the same way and still appends the reply, as the chat in [Common conversation shapes](#common-conversation-shapes) does.

When every reply is plain text with no tool calls, each `models.loop` call makes exactly one model request and returns, so two calls make two requests. Each such call adds exactly one `assistant` record, so two user records leave four records alternating `user` and `assistant`, with the replies at positions 2 and 4, as that chat's `4|first reply|second reply` shows.

`models.loop` also works in the H1 body (see [The H1 pass](04-how-a-prompt-runs.md#the-h1-pass)), so a prompt can hold a model conversation before the walk starts.

### Which model a loop round runs on

`models.loop(messages, compactor?)`, called with no model handle, runs each round on the section's current model. That is the model role selected with `models.use` (see [Selecting a role per section with `models.use`](10-models.md#selecting-a-role-per-section-with-modelsuse)), or else the prompt-wide default set with `models.default` (see [The prompt-wide default with `models.default`](10-models.md#the-prompt-wide-default-with-modelsdefault)). The loop looks the section's current model up again each time it sends a round, so one list can move between models:

````markdown
---
name: second-opinion
description: Drafts with one model role and reviews the draft with another.
runvil: 0
models:
  writer: {}
  reviewer: {}
---

# Second opinion

## Chat

```lua
local msgs = messages.new()
msgs:user('Write a short paragraph about the sea.')
models.use('writer')
models.loop(msgs)
msgs:user('Point out the weakest sentence in your paragraph.')
models.use('reviewer')
models.loop(msgs)
return msgs[#msgs].content
```
````

The first loop's round runs on the model bound to `writer`, and the second loop's round runs on the model bound to `reviewer`, because `models.use('reviewer')` changes the section's current model before the second call sends its round. The second request includes the first reply, so the reviewer sees the paragraph it comments on, and the run result is the reviewer's reply.

`models.loop(msgs)` runs on the section's current model even when other model handles are in scope, such as one from `local other = models.get('reviewer')`. With neither `models.use` nor `models.default` in effect, `models.loop` fails with the same missing-model error that `models.infer` raises (see [Infer errors and the missing-model error](10-models.md#infer-errors-and-the-missing-model-error)), because both pick the section's current model the same way.

### Running on a model handle with `handle:loop`

To run a conversation on one chosen model, call `loop` on a model handle (see [Getting handles with `models.get`](10-models.md#getting-handles-with-modelsget)). This version of `second-opinion` sets a prompt-wide default in its H1 body and runs its review on a handle:

````markdown
---
name: second-opinion
description: Drafts with one model role and reviews the draft with another.
runvil: 0
models:
  writer: {}
  reviewer: {}
---

# Second opinion

```lua
models.default('writer')
```

## Draft

```lua
local msgs = messages.new()
msgs:user('Write a short paragraph about the sea.')
models.loop(msgs)
var.draft = msgs[#msgs].content
```

## Review

```lua
local review = messages.new()
review:user('Point out the weakest sentence in this paragraph: ' .. var.draft)
models.get('reviewer'):loop(review)
return review[#review].content
```
````

`## Draft` runs on the prompt-wide default, `writer`, and keeps the draft in `var` (see [Keeping state in `var`](05-lua-environment.md#keeping-state-in-var)). `## Review` calls `h:loop(messages, compactor?)` on the handle that `models.get('reviewer')` returns, so every round of that call runs on the `reviewer` model, and the run result is the review.

A model handle (see [Model handles](10-models.md#model-handles)) has a `loop` method that runs the loop on that handle's model. You call it as `handle:loop(msgs, compactor?)`, or as `handle.loop(handle, msgs, compactor?)` with the handle passed first. Every round of a `handle:loop` call, at any point in the section, runs on the handle's own fixed model instead of the section's current model that `models.loop` uses. Every request `handle:loop` sends names that model handle, while `models.loop` sends its rounds with no handle, and both send your own message list.

The compactor is optional in both forms and follows the same rules on a handle, where it sets what happens when a round overflows the model's context window (see [Choosing a compactor for a call](#choosing-a-compactor-for-a-call)). Pass it second, as in `handle:loop(msgs, compactor)` or `other:loop(msgs, compactors.fail)`, or pass an explicit `nil`, as in `handle:loop(msgs, nil)`, for the default.

You can keep using one list after a loop returns, appending another user record and running the loop again, even on a different model, and each `handle:loop` call appends its own terminal record. Put this block in the `## Review` section of `second-opinion`:

````lua
local other = models.get('reviewer')
local msgs = messages.new()
msgs:user('first')
models.loop(msgs)
msgs:user('second')
other:loop(msgs)
return #msgs
````

````text
4
````

`models.loop(msgs)` runs its round on the prompt-wide default, `other:loop(msgs)` runs its round on `reviewer`, and each call adds one terminal record. Both entries also work as function values: `pcall(models.loop, msgs, compactor)` passes the list and the compactor, and `pcall(writer.loop, writer, msgs, compactor)`, where `writer` holds a model handle, supplies the handle as the first argument for the method form.

### The round counter and what each round reports

Each round that returns a reply advances the run's round counter by one, whether the reply is text, a batch of tool calls, or an empty reply. A round refused for context overflow, and a round that fails, leave the count unchanged. The records a loop reports hold this number in `turn`, so one `models.loop` call with two tool rounds and a final reply reports the `turn` values 1, 2, and 3.

Each round reports a model-turn observation, completed, failed, or truncated, and each tool call reports a succeeded or failed observation, in order (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)). A text reply that ends the loop takes one round, and the run reports one model-turn-completed event for it under the section that runs the loop. A failed round is reported as a model-turn-failed event before the error propagates, and the events for a failed round leave out the backend's response body and the messages you send.

A served round reports in this order:

1. The debug request and response pair, when debug capture is on.
2. The model-turn-completed event.
3. Any metadata diagnostics.
4. The model's thinking, when the model returns reasoning alongside its reply.
5. A truncation observation, when the model finishes with `length`.
6. The reply, or the tool-call batch.

When the model returns reasoning, the run reports a thinking event with the serving model's name and the reasoning text, always before that round's reply event. Each text reply is reported as an assistant reply event with the reply text, the provider's finish reason (see [Empty replies and finish reasons](#empty-replies-and-finish-reasons)), the name of the model that serves it, and usage metrics. A round that ends in a text reply answers the loop with the reply text, the finish reason, and the round's `turn`, and the final reply is reported as a `reply` record with its `turn`, the chat origin, the finish reason, and the text.

Each content event from a loop round is stamped with the round's id and the chat reply origin, which sets it apart from a reply to a nested `infer` (see [What an infer round reports](10-models.md#what-an-infer-round-reports)). Every event the loop reports includes the run's execution id. When the Host shows live output, each round's text fragments reach it as they arrive, in the order the model streams them, and the terminal record holds the whole text of those fragments.

## Loop arguments and errors

Here you pass `models.loop` and `handle:loop` the arguments they take, catch any loop failure with one `pcall`, and branch on its error kind. Put this block in the `## Chat` section of `first-chat`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
local ok, err = pcall(models.loop, msgs)
if not ok then
  if err.kind == 'tool_loop_exhausted' then
    return 'no final reply within the round cap'
  elseif err.kind == 'context_exhausted' then
    return 'the conversation outgrew the context window'
  end
  error(err)
end
return msgs[#msgs].content
````

When the round succeeds, `pcall` returns `true`, and the run result is the reply. Every failure of `models.loop` is raised at the call, so one `pcall` around the call catches them all (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). A failure that the loop or the Engine raises is an [error value](05-lua-environment.md#error-values) with a `kind` field, extra fields for that kind, and a readable message through `tostring(err)`. After a failed loop you branch on `err.kind`, here on `tool_loop_exhausted` (see [Tool rounds and the round cap](#tool-rounds-and-the-round-cap)) and `context_exhausted` (see [Compactors and context exhaustion](#compactors-and-context-exhaustion)), and raise any error you do not handle again with `error(err)`.

Without a `pcall`, a failed `models.loop` ends the block at the call, and the rest of the block after it does not run.

### The loop's arguments

`models.loop(messages, compactor?)` takes at most two arguments:

- `messages`, required: a message list built with `messages.new()`.
- `compactor`, optional: a function, which the loop uses as given, or `nil` for `compactors.fail` (see [Choosing a compactor for a call](#choosing-a-compactor-for-a-call)).

`handle:loop(messages, compactor?)` takes the same two arguments after the handle, which the colon supplies.

A call that breaks these rules is refused with a `lua`-kind error value whose message is the refusal text, raised at the call, so a `pcall` around the call catches it. A refused call raises before the loop starts, so no model request is sent for it, and it leaves your list unchanged: `#msgs` reports the same length after the refusal as before it. A refused call also leaves pending task notices (see [`await_tasks` and task notices](15-tasks.md#await_tasks-and-task-notices)) queued, so the next valid loop call's round delivers them to the list, for example as the user record `Task id=0.0 (## Child) was canceled: the author cancelled it`.

### Argument errors

Each loop entry checks its arguments in a fixed order and reports only the first failure:

1. `models.loop` refuses a model handle in first place, and `handle:loop` refuses a call with no model handle as its receiver.
2. Both check the argument count.
3. Both check the compactor's type.
4. Both check the message list.

So `models.loop(1, 5)` reports the compactor error, even though `1` is not a message list either.

The messages argument is a `messages.new()` list. Anything else in its place, including `nil`, a missing argument, a number, a string, or a plain table such as `{}`, raises this `lua`-kind error:

````text
models.loop needs a messages.new() list; build one with messages.new() and :user, :append, or :replace
````

A hand-written Lua array of message records, such as `{ { role = 'user', content = 'hi' } }`, gets the same error from both `models.loop` and `handle:loop` even when every record is well formed. Its records are never checked and no request is sent, because only a `messages.new()` list is accepted. A single record taken from a list, such as `msgs[1]`, gets the list error too. `handle:loop` raises the same list error, with a message that still names `models.loop`, for a number, a missing list, or a model handle where the message list goes. The list error raises even after you rebind the global `error` to a function that does nothing, because `models.loop` does not rely on your globals to raise, and no round runs.

`models.loop` takes the list first. A model handle in first place, with or without a compactor, raises this before any round is sent:

````text
models.loop takes (messages, compactor?); call handle:loop(messages, compactor?) to run on a model handle
````

`handle:loop` needs the handle as its receiver, which the colon supplies. Calling it with a dot, as in `handle.loop(msgs)`, or taking it off the handle and calling it bare, as in `local run = handle.loop; run(msgs)`, leaves no model handle as the receiver and fails before any round with:

````text
call loop on a model handle with a colon: handle:loop(messages, compactor?)
````

`models.loop` takes at most two arguments, and every value passed counts, trailing `nil`s included. So `models.loop(msgs, nil, nil)` raises this, and no round runs:

````text
models.loop takes (messages, compactor?)
````

`handle:loop` takes at most two arguments after the handle, with trailing `nil`s counted the same way. `handle:loop(list, nil)` starts the loop, while `handle:loop(list, nil, nil)` raises:

````text
handle:loop takes (messages, compactor?)
````

A compactor that is neither a function nor `nil`, passed to `models.loop` or to a handle's `loop`, raises an error naming the Lua type passed, before any request is sent:

````text
compactor must be a function, got integer
````

The message names an integer `integer` and every other value by Lua's `type()` name. A float reads `number`, a string reads `string`, and a table reads `table`. Any userdata reads `userdata`, including a message list, a model handle, and the JSON null that a tool result holds.

### Errors while a round runs

A loop round over a list that holds no records fails at the `models.loop` call with a `lua`-kind error, before any request is sent:

````text
messages must not be empty
````

The empty-list check runs on each round after that round's pending task notices are added, so `models.loop(messages.new())` runs one round over the pending notices when there are any.

Any failure while the loop prepares a round is raised at the `models.loop` call, where your `pcall` catches it like any other call error. That covers choosing the model, reading the section's tools, checking the list, the empty-list check among them, turning the list into the request, and reaching the model's provider.

An error from a model request or a tool call inside the loop reaches you as the Engine's own error value, unchanged. When the model request itself fails, the loop raises that request's error unchanged. A request failure other than a context-window rejection or an empty reply, such as a backend or transport error, is raised as the `models.loop` call's error without calling the compactor.

A failure that the loop's rules decide to raise reaches your block as that same value, with no position prefix added. A string or table that your own compactor or local tool handler raises with `error` reaches the `models.loop` caller unchanged: a string stays that string, and a table is the very same table, with no metatable added.

A round refused for context overflow, or for a compactor that returns instead of raising, adds no records to the list, and records from rounds that have already completed stay in the list.

`models.loop` raises kind `lua` when an argument or the message list breaks a rule, or when a compactor returns instead of raising, with the message `the selected compactor returned without raising: ...`. Left uncaught, a `lua`-kind loop error ends the run with run error kind `Lua` (see [Run error kinds](16-limits-and-errors.md#run-error-kinds)), and the run's message holds the error text and your prompt line, such as ``section `Only` prologue:14: in`` (see [Error line numbers and locations](05-lua-environment.md#error-line-numbers-and-locations)).

## Tool rounds and the round cap

Here you watch `models.loop` run the tools a model asks for, read the records each tool round adds to your list, and cap how many rounds one loop call may make.

````markdown
---
name: search-chat
description: Lets a model search the web before it answers.
runvil: 0
plugins:
  - web
models:
  writer: {}
---

# Search chat

## Ask

```lua
models.use('writer')
tools.offer('web/search')
local msgs = messages.new()
msgs:user('Find the tallest building in Paris, then answer in one sentence.')
models.loop(msgs)
local roles = {}
for i = 1, #msgs do roles[i] = msgs[i].role end
return table.concat(roles, ',')
```
````

The prompt declares the `web` Plugin (see [The Engine, the Harness, and the Host](01-what-a-prompt-is.md#the-engine-the-harness-and-the-host)) and offers its search tool to the model before the loop (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)). When the model makes one search and then answers with text, the run result is:

````text
user,assistant,tool,assistant
````

When tools are in scope for the section, a round can end with the model asking for tools instead of replying, and such a round is a tool round. When the model asks for tools, `models.loop` runs each requested tool, appends the calls and their results to the message list, and sends the list to the model again, ending when the model replies with text. So you never send single rounds or run the requested tools by hand. Here the loop makes two rounds:

1. Round 1 sends the user record, and the model asks for a search, so the round is a tool round.
2. The loop runs the call and appends two records: an `assistant` record holding the call, and a `tool` record holding its result.
3. Round 2 sends all three records, so the request includes a message whose message role is `tool` and whose content is the search result, and the model replies with text.
4. The loop appends the terminal record and returns, so the reply is `msgs[#msgs].content`.

The model can call every tool the section offers with `tools.offer` or `tools.offer_local` (see [Defining local tools with `tools.offer_local`](12-tools.md#defining-local-tools-with-toolsoffer_local)), with both kinds mixed in one loop, and `models.loop(msgs)` sends each tool call the model asks for to the matching offered tool. The loop keeps running rounds while the model answers with tool calls: it runs each round's calls, sends the conversation with the tool results back to the model for the next round, and can go through several tool rounds in a row within one call. It stops at the first reply that has plain text and no tool calls, and that text lands as the last record of the list.

After a tool round is answered, the loop starts the next round with a new request over the updated list. That request includes a `tool` message with each result, and the round after a batch of calls sends the model every result from that batch.

### The records a tool round adds

Put these lines in the `## Ask` block of `search-chat` in place of its last three lines:

````lua
local call = msgs[2].tool_calls[1]
return call.id .. '|' .. msgs[3].role .. '|' .. msgs[3].tool_call_id
````

When the model's search call has the id `call_1`, the run result is:

````text
call_1|tool|call_1
````

When the model requests tools, `models.loop` appends an assistant record whose content is empty text and whose `tool_calls` field lists every call in the round, so after one user record the first call reads as `msgs[2].tool_calls[1]`. Each entry in `tool_calls` has the call's `id`, `name`, and `arguments`, read as `msgs[2].tool_calls[1].id`, `.name`, and `.arguments`. Every call the loop records has a non-blank `id` unique within its round, a non-blank `name`, and `arguments` already decoded into a Lua table, never JSON text, so you read argument values directly: for a call whose arguments hold a `city` of `Paris`, `msgs[2].tool_calls[1].arguments.city` reads `'Paris'`.

Right after that assistant record, the loop appends one record per call whose message role is `tool`, with `tool_call_id` equal to the call's `id` and `content` holding the call's result text. The `tool` record your list holds is exactly what the loop sends to the model on the next round. The loop records calls in the same shape you write them yourself (see [Message records](#message-records)), so its records and yours replay the same way.

A model reply with a blank call id or name, an id repeated within the round, or arguments that are missing or do not decode to a JSON object fails the round as a malformed reply, raised at the call with kind `internal`. A round that requests tool calls is a tool round even when the provider finishes it with the finish reason `stop`.

Each tool round appends its records after the records already in the list. When each round asks for one call, every tool round appends exactly two records, an assistant record holding the call and a `tool` record holding its result, in round order. With one leading user record, the tool results sit at `msgs[3]`, `msgs[5]`, `msgs[7]`, and onward, and you read each one through `content`:

| Rounds in one call | Records after the loop |
| --- | --- |
| A text reply only | 2 |
| One one-call tool round, then the reply | 4 |
| Four one-call tool rounds, then the reply | 10 |

One user record, one tool round with one call, and the reply make 4 records: the user record, an assistant record with `tool_calls`, a `tool` record, and the final assistant record.

When the first round asks for two calls, the list after `models.loop` returns holds the user record, then the assistant's tool-call record, then one `tool` record per call, so `msgs[3]` and `msgs[4]` hold the first and second call's results. The `tool` records follow the order in which the model issues the calls.

After the loop, your list holds the whole conversation in order: the user record, each assistant call followed by its tool answer, any delivered task notices, and the final assistant reply, as in the message roles `user,assistant,tool,assistant,tool,user,assistant`. You read any tool's answer by index, such as a task tool's answer at `msgs[5].content` in that list.

### Batches of tool calls

When a round's answer has tool calls, the loop answers every call before it asks the model again. When one reply requests several tool calls, `models.loop` answers them as one batch, each call with its own arguments, before it asks for the next reply. The calls are issued in the order the model lists them, and the loop runs each call's handler once.

When one reply asks for several bound tools, tools offered with `tools.offer`, the loop issues those calls at once so they run in parallel. A local tool call, from `tools.offer_local`, or a task tool call such as `task_status` (see [Tool scope and the task built-ins](12-tools.md#tool-scope-and-the-task-built-ins)) stops the batch and runs on its own between its neighbors. So a batch of a bound call, a local call, a `task_status` call, and another bound call issues the first call alone, then answers the local call and the `task_status` call, then issues the last call.

The loop appends a round's records only once every call in the round has its result, and then it appends the whole batch at once: one `assistant` record with empty text whose `tool_calls` holds every call id of the round in the model's order, then one `tool` record per call in the same order, all ahead of the final text reply. The `tool` records follow the order the model lists the calls, not the order they finish in, so a call listed first has its record first even when it finishes second. A round that calls two tools therefore adds three records, and the loop never leaves a half-answered batch: nothing from a tool round reaches the list until every call in that round has an answer. While a batch's handlers run, the list shows none of that batch, so a handler sees the list as it stands before the round.

When a bound tool fails during the loop, the loop does not stop. The tool's failure message, wrapped as untrusted text inside `<untrusted_input_` tags, becomes that call's `tool` record, and the loop goes on to the next round.

Some failures end the loop partway through a round, and then none of that round's records land on the list:

- A local tool handler that raises ends `models.loop` with no further round. Neither the assistant tool-call record nor the results of calls that have already returned land on the list.
- When a tool call the model issues is dropped instead of answered, `models.loop` raises `cancelled` at the call and leaves the list as it was before the round, with no assistant or `tool` record appended.
- When the model calls a tool outside the section's tool scope (see [Tool scope and the task built-ins](12-tools.md#tool-scope-and-the-task-built-ins)), `models.loop` fails and appends nothing to the list.

You catch such a failure with `pcall` and read `err.kind` and `err.name`, for example to tell which tool was out of scope. An out-of-scope call to `task` gives `err.kind` as `out_of_scope_tool` and `err.name` as `task`, and a list that held one user record before the call still holds just that record.

Once the model gives a text reply, the loop runs no further rounds, even when a task the model has started (see [Starting a task with `tasks.spawn`](15-tasks.md#starting-a-task-with-tasksspawn)) ends later.

### Task notices and task tools in the list

Starting `models.loop` adds nothing to the message list on its own. Any notices for the model are added when a round's request is sent, and after a round that delivers task notices (see [`await_tasks` and task notices](15-tasks.md#await_tasks-and-task-notices)), your list holds one extra `user` record for each notice text. After `models.loop` returns, you read a notice like any other record, through its `role` and `content` fields, for example `user` and a content starting `Task id=0.0 (## Child) completed: `.

Each answer from a task tool (see [Tool scope and the task built-ins](12-tools.md#tool-scope-and-the-task-built-ins)) appears in the list as a `tool` record whose `tool_call_id` matches the `id` of the model's call. After `await_tasks` answers, the loop sends that answer to the model in the next round and keeps going until the model replies with plain text.

### Setting the round cap

The round cap is the most rounds a single `models.loop` call may make, and you set it with the `max_tool_iterations:` key in the frontmatter (see [Optional keys and unknown keys](02-file-structure.md#optional-keys-and-unknown-keys)). Add the key to `search-chat` and catch the error the cap raises:

````markdown
---
name: search-chat
description: Lets a model search the web before it answers.
runvil: 0
max_tool_iterations: 2
plugins:
  - web
models:
  writer: {}
---

# Search chat

## Ask

```lua
models.use('writer')
tools.offer('web/search')
local msgs = messages.new()
msgs:user('Find the tallest building in Paris, then answer in one sentence.')
local ok, err = pcall(models.loop, msgs)
if not ok then
  if err.kind == 'tool_loop_exhausted' then
    return tostring(err) .. ' after ' .. #msgs .. ' records'
  end
  error(err)
end
return msgs[#msgs].content
```
````

When the model keeps calling the search tool and never replies with text, the run result is:

````text
tool-call loop did not converge after 5 records
````

When the model answers with text within its two rounds, the run result is that reply.

Setting `max_tool_iterations:` to a positive integer, such as `3`, `5`, or `20`, sets the prompt's round cap to that number, so each `models.loop` call starts at most that many rounds. A prompt that omits the key runs with the default round cap, which is 24 unless the Host or Harness running the prompt sets another default. An explicit value replaces that default for that prompt only, whether it raises or lowers the cap. A cap tighter than the default suits a prompt whose prose tells the model its budget is limited: the shipped `research-person` prompt (see [Shipped example prompts](01-what-a-prompt-is.md#shipped-example-prompts)) sets `max_tool_iterations: 20` for its `models.loop(msgs)` call and tells the model it has a limited budget.

The cap counts how many times the model is asked within one `models.loop` call, one round per request:

- A round counts toward the cap as soon as it starts, including the final round where the model replies with text.
- A tool round counts once no matter how many calls it holds or how many dispatches it takes.
- The tally starts at zero for each loop call, so every `models.loop` call in a run gets the full cap, and two loops in one section can each make that many rounds.
- The cap counts only the current call's own rounds, separately from the run's round counter that every round advances (see [The round counter and what each round reports](#the-round-counter-and-what-each-round-reports)), so rounds made earlier in the run never use up a later loop's cap.
- The cap applies inside fanout arms (see [What an arm sees](14-fanout.md#what-an-arm-sees)) too.

### Reaching the round cap

With `max_tool_iterations` set to N, a model that never stops calling tools gets exactly N rounds: when the Nth tool round is answered, `models.loop` raises `tool_loop_exhausted` without sending another request, so `max_tool_iterations: 1` allows one round. The error's message is:

````text
tool-call loop did not converge
````

A model that keeps calling a failing bound tool reaches the cap the same way, because each failure is answered with failure text and the model never gives a text reply.

When `tool_loop_exhausted` is raised, the last allowed round's assistant tool-call record and its `tool` records are already on the list, and they stay there. The list holds only complete rounds, each with its assistant tool-call record and a `tool` record for each call, so one user record with `max_tool_iterations: 2` leaves 5 records, as in `search-chat`.

The error value has its `kind` field set to `tool_loop_exhausted`, and it has only its kind and message, with no extra fields. `tostring(err)` gives `tool-call loop did not converge`. You catch it at the call site with `pcall(models.loop, msgs)`, which returns `false` and that error value.

Left uncaught, the `tool_loop_exhausted` error ends the block, so the section's later code never runs, and it fails the run with run error kind `Tool` (see [Run error kinds](16-limits-and-errors.md#run-error-kinds)) and the message `tool-call loop did not converge`. When the model has started a task in that section, abandoning that task does not replace the `tool_loop_exhausted` error.

### Valid `max_tool_iterations` values

`max_tool_iterations:` accepts whole numbers from 1 through 1000 inclusive, with 1000 itself accepted, written as a bare YAML integer such as `max_tool_iterations: 50`. A value outside that range fails at parse, before the run starts, with parse error kind `Frontmatter` (see [Parse errors and their kinds](16-limits-and-errors.md#parse-errors-and-their-kinds)) in the `invalid frontmatter: {detail}` form. The detail is one of two messages, where `{raw}` echoes the value as written:

| Value | Detail message |
| --- | --- |
| Zero or below, such as `0` or `-1` | `max_tool_iterations must be a positive integer (>= 1), got {raw}` |
| Above 1000, such as `1001` or `100000000000` | `max_tool_iterations must be <= 1000, got {raw}` |

A value too large for 32 bits, such as `100000000000`, is range-checked rather than wrapped, so it gets the second message.

## What the model receives

Here you learn how the loop checks your message list and composes it into the conversation the model reads, so you can predict exactly what each request holds.

````markdown
---
name: first-chat
description: Says hello to a model and returns its reply.
runvil: 0
models:
  writer: {}
---

# First chat

## Chat

```lua
models.use('writer')
local msgs = messages.new()
msgs:system('You are terse.')
msgs:system('Answer in French.')
msgs:user('Hello.')
msgs:user('What time is it?')
models.loop(msgs)
return #msgs
```
````

The list holds four records, but the model receives two:

````text
{ role = 'system', content = 'You are terse.\n\nAnswer in French.' }
{ role = 'user', content = 'Hello.\n\nWhat time is it?' }
````

When the model replies with text, the run result is:

````text
5
````

The two leading system records compose into one system message, and the two user records merge into one user message, each pair joined by a blank line. Composing and merging shape only what the model receives. Your list still holds the four records you added plus the terminal record the loop appended, so `#msgs` is 5.

### Checking and composing before each request

Right before each request leaves, the loop takes whatever records the list holds at that moment and does two things with them. First it checks the cross-record rules, the rules about how records relate to one another rather than about one record's own fields:

- `tool_calls` appears only on `assistant` records.
- Every tool call id is unique across the whole list.
- Every tool call is paired with exactly one tool record that answers it.
- When two or more `system` records lead the list, each of them is plain text.

Then it composes what the model receives from those records, merging runs of records so the message roles alternate as a provider requires.

Builder methods do not check the cross-record rules when they add a record, and `models.loop` does not check a record's own fields again (see [When records are checked](#when-records-are-checked)). The cross-record rules run on every round, right before that round is sent, so records you append between calls and records the loop appends between its own rounds meet the same rules, and a list you edit between calls is checked and composed again on the next call.

When the list breaks a cross-record rule, the round fails. The run reports a `model_turn_failed` event (see [Run events and records](04-how-a-prompt-runs.md#run-events-and-records)), and the `models.loop` call raises a Lua error and sends nothing to the model. [Cross-record errors](#cross-record-errors) lists every message.

### A well-formed list reaches the model as written

A list with one leading `system` record followed by alternating `user` and `assistant` records reaches the model exactly as written: unchanged, in order, one message for each record. A list built as system `be terse`, user `hi`, assistant `hello`, user `thanks` reaches the model as those same four messages. Any other list reaches the model with its runs of records merged.

`models.loop` sends only the records in the list, in the order the builder methods such as `msgs:system` and `msgs:user` added them. It adds no system message or other text of its own. A record whose `content` is a string reaches the model as plain text, so `{ role = "system", content = "be terse" }` arrives as:

````text
{ "role": "system", "content": "be terse" }
````

### Same-role merging

Same-role merging is how the loop joins a run of neighboring records into one message for the model, so the message roles always alternate. Each message role has its own rule:

| Records in a row | What the model receives |
|---|---|
| Two or more leading `system` records | One system message, texts joined by a blank line |
| Consecutive `user` records | One user message, contents joined by a blank line |
| Consecutive text-only `assistant` records | One assistant message, texts joined with no separator |
| A text-only `assistant` record, then an `assistant` record with `tool_calls` | One assistant record holding the joined text and the later record's tool calls |
| Consecutive `tool` records | No merging; each tool result stays its own message |

Because user records merge, you never merge them by hand to satisfy a provider's alternation rule. Text-only assistant records reassemble the same way: streaming fragments of one reply that you append as they arrive, such as `Hello, `, `world`, and `!`, reach the model as one assistant message, `Hello, world!`.

Assistant text written just before an assistant record with `tool_calls` merges into that record as its visible content instead of breaking the alternation. A text-only assistant record `let me check`, followed by an assistant record whose `content` is empty and whose `tool_calls` hold one call, reaches the model as one assistant message whose content is `let me check` and whose tool calls are that call.

Tool records never merge, so each tool result reaches the model as its own message, even when several sit side by side.

Empty user and assistant records vanish into their run instead of adding blank lines to the joined text. When both records being merged are plain text, an empty one adds nothing, so no stray blank line or separator appears in the message the model receives. A list that repeats system records, holds empty fragments, and has same-role runs heals into a clean alternation. These eight records:

````text
system 's1', system 's2', user '', user 'hi', assistant '', assistant 'answer', assistant '', user 'next'
````

reach the model as four messages:

````text
system 's1\n\ns2', user 'hi', assistant 'answer', user 'next'
````

When either record being merged uses content parts (see [Message records](#message-records)), the model receives a parts array holding every part of both records in arrival order. Between user records, the blank-line separator goes in as its own text part, even when the other side is empty text. Assistant records have no separator, so their parts follow one another directly. Consecutive assistant fragments that mix plain text and content parts merge into one parts array in arrival order, and an empty fragment among them is absorbed.

### How the leading system block is composed

The leading system block is the run of `system` records at the start of a list (see [The leading system block](#the-leading-system-block)). Two or more leading system records reach the model as one system message whose text is their texts joined by a blank line. The provider accepts only one system message, so the loop composes the block rather than rejecting the list.

For that composition to work, every record in a block of two or more is plain text. A lone leading system record reaches the model unchanged: it may use plain text or content parts, and its parts stay intact rather than being flattened to text.

When the block holds two or more records and any of them uses content parts, the `models.loop` call fails and names the first such record by its position:

````text
messages[N] is a system message with content parts; only plain text system messages can be composed
````

### Your list stays as you wrote it

Sending a list never changes it. Composing system records and merging same-role records happen on a copy and change only what the model receives. Your list keeps its own records, count, and order, so `#msgs` counts the records as you appended them rather than the merged messages the model receives, and the list stays the conversation's continuity state. In the first example the model received two messages, yet `#msgs` read 5.

Each round composes the list again for whichever model that round runs on, so the same list can go to different models on later calls. One `models.loop` call can run on the default model, and a later call on the same list can run after `models.use('other')` selects another model role (see [Selecting a role per section with `models.use`](10-models.md#selecting-a-role-per-section-with-modelsuse)).

### Tool exchanges in a request

Put this block in the `## Chat` section of `first-chat` to send a list that already holds a finished tool exchange:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('Read the notes.')
msgs:append({ role = 'assistant', content = '', tool_calls = {
  { id = 'call_1', name = 'read', arguments = { path = 'notes.txt', limit = 3 } },
  { id = 'call_2', name = 'search', arguments = { query = 'tool_calls', recursive = true } },
} })
msgs:append({ role = 'tool', tool_call_id = 'call_2', content = 'two matches' })
msgs:append({ role = 'tool', tool_call_id = 'call_1', content = 'first line' })
models.loop(msgs)
return msgs[#msgs].content
````

The run result is the model's reply. The records right after an assistant record with `tool_calls` are that record's results: one `tool` record per call, each with a `tool_call_id` matching its call, and each answering its call exactly once. The results may come in any order, so `call_2` is answered before `call_1` here, but nothing else comes between the assistant record and its last tool record. After an assistant record with `tool_calls`, a tool record for every call comes before any record of another message role and before the list ends.

A complete tool exchange, meaning an assistant record with `tool_calls` followed by one tool record per call, reaches the model verbatim. One assistant record can hold several tool calls, and all of them replay to the model in that one message, with their ids unchanged and their arguments intact. The model receives them as its own earlier function calls, with each call's `id` and `name` and its structured `arguments` sent as JSON text, so the first call's arguments arrive as a JSON string that decodes back to `{ "path": "notes.txt", "limit": 3 }`.

When the loop records the model's own tool calls during a tool round (see [Tool rounds and the round cap](#tool-rounds-and-the-round-cap)), it writes them in the same `id`, `name`, and `arguments` shape you write. An assistant record with `tool_calls` that the loop appended and one that you appended reach the model the same way.

Each tool record answers exactly one still-pending call from the assistant record just before its run of results, and answers it only once. Tool call ids are unique across the whole list, not only within one assistant record, so a list that repeats an id anywhere is refused as a duplicate.

### Content parts in a request

Content parts keep their order and their type on the way to the model. A record whose content is a text part followed by an image part reaches the model in that order and as those types: each text part as text, and each image part as an image referenced by its URL. [Message records](#message-records) shows how to write content parts.

### Cross-record errors

A list that breaks a cross-record rule fails at the `models.loop` call with a Lua error whose message names the offending record by its 1-based position as `messages[N]`. Put this block in the `## Chat` section of `first-chat` to see one:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hi')
msgs:append({ role = 'assistant', content = '', tool_calls = {
  { id = 'call_1', name = 'read', arguments = { path = 'notes.txt' } },
} })
local ok, err = pcall(models.loop, msgs)
return tostring(err)
````

````text
messages[2] tool call "call_1" has no tool result
````

The list ends while the assistant record at position 2 still has an unanswered call, so the call fails before anything is sent, and the message names that record and the call's id. Ids always print in double quotes. These are the five messages:

| Message | Names |
|---|---|
| `messages[N] is a system message with content parts; only plain text system messages can be composed` | The first record with content parts in a leading block of two or more system records |
| `messages[N] sets tool_calls but is not an assistant message` | The `system`, `user`, or `tool` record that sets `tool_calls` |
| `messages[N] tool call id "ID" duplicates an earlier tool call` | The assistant record that repeats the id, and the id |
| `messages[N] tool call "ID" has no tool result` | The assistant record with an unanswered call, and its first unanswered call in call order |
| `messages[N] is an orphan tool record: no pending assistant tool call with id "ID"` | The tool record, and the `tool_call_id` it gives |

Each rule has its own triggers:

- Content parts in the leading block: any record of a leading block of two or more system records uses content parts, and the message names the first such record.
- `tool_calls` placement: only assistant records hold `tool_calls`. Adding a `system`, `user`, or `tool` record that sets `tool_calls` with `append` or `replace` succeeds, because a record's own fields allow it, but the next `models.loop` call fails naming that record.
- Duplicate ids: reusing a tool call id from any earlier record, or using one id twice in one assistant record, fails naming the assistant record that repeats it. This holds even after the first call was answered, and it includes a model that repeats an id from an earlier round, which fails at the next request. Two calls with id `call_1` in the first record give `messages[1] tool call id "call_1" duplicates an earlier tool call`, and a later assistant record at position 3 that reuses `call_1` gives `messages[3] tool call id "call_1" duplicates an earlier tool call`.
- Missing results: a call gets no tool record before a record of another message role arrives, before a new assistant record with `tool_calls` starts, or before the list ends. The message names the assistant record holding the unanswered call. When that record at position 1 has calls `call_1` and `call_2`, and only `call_1` is answered before a user record, the message is `messages[1] tool call "call_2" has no tool result`.
- Orphan tool records: a tool record does not answer a still-pending call from the open assistant record exactly once. That covers a tool record with no assistant tool call before it, a `tool_call_id` that is not among the open calls, a second answer to a call that already has its result, and an answer to a call whose assistant record is already fully answered. A tool record at position 2 answering `call_9` right after a user record gives `messages[2] is an orphan tool record: no pending assistant tool call with id "call_9"`.

The rules run in a fixed order, and only the first broken rule is reported:

1. The plain-text check on a leading block of two or more system records. It runs before every other cross-record rule, so it is the error reported even when later records also break another rule.
2. Each record in list order, with the `tool_calls` placement rule checked before the rules for that record's own message role.
3. The check for an assistant record whose calls are still open at the end of the list.

## Empty replies and finish reasons

Here you learn how the loop judges a round that comes back with nothing in it, so you can tell a model that finished on purpose from a round that failed.

````markdown
---
name: first-chat
description: Says hello to a model and returns its reply.
runvil: 0
models:
  writer: {}
---

# First chat

## Chat

```lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
local ok, err = pcall(models.loop, msgs)
if ok then return msgs[#msgs].content end
return err.kind .. '|' .. tostring(err.finish_reason) .. '|' .. tostring(err) .. '|' .. #msgs
```
````

When the model replies with no text and no tool calls, and the provider gives the finish reason `stop`, the run result is:

````text
empty_model_reply|stop|the model replied with no text and no tool calls|1
````

The round produced nothing, and no tool call came before it, so `models.loop` raises an error value of kind `empty_model_reply`. `pcall` catches it, returning `false` and the error value (see [Raising and catching errors](05-lua-environment.md#raising-and-catching-errors)). `err.finish_reason` holds the provider's finish reason, `tostring(err)` gives the message, and `#msgs` is still 1 because the empty reply was not appended.

### Finish reasons

A finish reason is the provider's own label for how a round's reply ended, such as `"stop"`, or `"length"` for a reply the provider cut short for length. A round's reply comes with a finish reason when the provider sends one. The finish reason the loop works with is the provider's own, passed through exactly as the provider sent it, and it is absent when the provider sent none or when no round ran.

`models.loop` judges a round by what it produced, not by its finish reason. A round that asks for tool calls runs those calls even when its finish reason is `stop`, because providers routinely finish tool-call rounds with `stop`. A round that asks for tool calls is a tool round, never an empty reply, even when its text is empty or null. The finish reason decides the outcome only for an empty reply.

### What counts as an empty reply

An empty reply is a round that produces neither reply text nor tool calls. `models.loop` judges an empty reply against that round's finish reason.

Text that is empty or only whitespace counts exactly as no reply, so content such as `"   \n\t "` is an empty reply, and a reply whose text is the empty string counts the same as a round that returns no reply at all. That includes an empty string in a reply the Host builds itself rather than reads from a model. A reply with no tool calls and no text other than whitespace is an empty reply.

A reply that holds only reasoning content, with no text and no tool calls, also counts as empty and raises `empty_model_reply`. Reasoning text from the model is never used as the reply, even when it is all the model sent.

### When an empty reply raises

The outcome of an empty reply depends on two things: whether the same `models.loop` call has already answered a tool call, and the round's finish reason.

| Tool calls answered earlier in this `models.loop` call | Finish reason | Outcome |
|---|---|---|
| None | Any, or none | `empty_model_reply` |
| One or more | `stop` | The clean exit |
| One or more | None | `empty_model_reply` with no `finish_reason` field |
| One or more | `length` or any other | `empty_model_reply` |

When a round finishes with no reply text and no tool calls, and the same `models.loop` call has answered no tool call before it, `models.loop` raises `empty_model_reply`. That holds for an empty `stop` reply too, whether or not the model had tools to call: a tool that was offered but never called does not count.

After at least one answered tool call, only the finish reason `stop` ends cleanly. An empty reply with no finish reason fails closed and raises `empty_model_reply` with no finish reason, even after a successful tool call, because a missing finish reason is not `stop`. An empty reply with finish reason `length` is never a clean exit, so it raises `empty_model_reply` with finish reason `length`, and it reports no truncation event, since there is no reply to have truncated.

### The clean exit

The clean exit is how a model ends a conversation on purpose after doing its work through tools: an empty reply with finish reason `stop`, after at least one answered tool call in the same `models.loop` call. In a section that offers the model a tool (see [Offering tools with `tools.offer` and `plugins.offer`](12-tools.md#offering-tools-with-toolsoffer-and-pluginsoffer)), this block reads the list after the loop:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
local result = models.loop(msgs)
return tostring(result) .. '|' .. #msgs .. '|' .. msgs[3].role .. '|' .. msgs[4].role .. ':' .. msgs[4].content
````

When the model calls the tool once and then sends an empty reply with finish reason `stop`, the run result is:

````text
nil|4|tool|assistant:
````

On the clean exit the loop does not raise. It appends `{ role = "assistant", content = "" }` as the terminal record and returns `nil`. The terminal record has no tool calls, so the last entry of the list has message role `assistant` and content `""`, after the user record, the assistant record with the call, and the tool record.

The clean-exit rule counts the tool calls answered across every round of the current `models.loop` call, and a call whose tool failed still counts. The run above returns `nil` the same way when the tool call fails, instead of raising `empty_model_reply`.

Each `models.loop` call starts its count at zero. When the current call has answered no tool call of its own, an empty `stop` reply raises `empty_model_reply`, even if an earlier `models.loop` call on the same list made tool calls. A first call that makes one tool call and gets the reply `done`, followed by a user record `second` and a second call that gets an empty `stop` reply, leaves `pcall(models.loop, msgs)` returning `false` with kind `empty_model_reply`, and `#msgs` at 5.

### The `empty_model_reply` error value

`empty_model_reply` is the error kind for a model round that returned no product, neither text nor tool calls. A model completion that comes back empty surfaces as this kind, and it is raised only for empty replies, including an empty reply that finished with `length`. A truncated reply that has text is accepted instead (see [Replies cut short for length](#replies-cut-short-for-length)).

Its error value (see [Error values](05-lua-environment.md#error-values)) holds:

- `kind`: `empty_model_reply`.
- `finish_reason`: the finish reason the provider reported for the empty round, such as `stop` or `length`, read as `err.finish_reason`. It is always a string, never a number. When the provider reports no finish reason, the error value has no `finish_reason` field at all, so `err.finish_reason` reads `nil`.
- The message, which `tostring(err)` returns.

The message for an empty reply read from a model is the fixed phrase:

````text
the model replied with no text and no tool calls
````

When the empty reply held reasoning content that was ignored, the message extends that phrase after a colon:

````text
the model replied with no text and no tool calls: reasoning content was present but ignored
````

Every reply read from a model backend names its empty-reply phrase, and that phrase becomes the `empty_model_reply` message, so you see the same text the failed reply itself gives. When an empty round reaches the loop without such a phrase, as an empty text reply that the Host builds itself does, the message falls back to:

````text
empty model reply
````

### What a rejected empty round leaves

`models.loop` does not retry an empty round. It raises on the first empty answer without sending a second request. An empty reply that raises `empty_model_reply` appends nothing to the message list, so the list keeps only what it held before the call. In the first example of this section the list held one message, and `#msgs` was still 1 after the error.

### Replies cut short for length

A text reply that the provider cut short for length, with finish reason `"length"`, is accepted as the terminal record. It still ends `models.loop`, and its partial text becomes the terminal record. With this block in the `## Chat` section of `first-chat`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user('hello')
models.loop(msgs)
return #msgs .. '|' .. msgs[#msgs].content
````

a reply of `partial answer` with finish reason `length` gives:

````text
2|partial answer
````

A text reply that finishes with reason `length` emits a truncation observation alongside its `assistant_reply` report: the run reports a `model_turn_truncated` event right after the `model_turn_completed` event.

A tool round is different. A tool-call round that the provider cuts short with finish reason `length` or `content_filter` fails as a model failure whose message names the truncation, so its partial arguments never run. `pcall` catches that failure as an error value of kind `internal`, and left uncaught it ends the run with run error kind `Completion` (see [Run error kinds](16-limits-and-errors.md#run-error-kinds)).

### Events for an empty round

An empty model reply still completes its round. It advances the round count and reports a `model_turn_completed` event holding the provider's finish reason and the message, and the loop then decides, from the earlier rounds of the same call, whether it is the clean exit or an `empty_model_reply` failure. The run reports every empty round as a completed model round this way. An empty reply records no debug capture, since the failed completion has no request or response bodies to record.

### When `empty_model_reply` goes uncaught

An `empty_model_reply` that the prompt does not catch ends the block as that same typed `empty_model_reply` error, and it fails the run with run error kind `Completion`. The run's `Completion` failure keeps the message you would have seen and the `finish_reason` field, which is there only when the provider gave one. The clean exit, by contrast, returns normally and fails nothing.

## The context precheck

Here you learn how the loop decides, before each request leaves, whether the conversation fits the model's context window, and how to tell its refusal from the provider's.

````markdown
---
name: long-notes
description: Sends notes far longer than the model's context window.
runvil: 0
models:
  writer: {}
---

# Long notes

## Send

```lua
models.use('writer')
local msgs = messages.new()
msgs:user(string.rep('x', 100000))
local ok, err = pcall(models.loop, msgs)
if ok then return 'sent' end
return err.kind .. ': ' .. tostring(err.reason)
```
````

When the model bound to `writer` has a 4096-token context window, the run result is:

````text
context_exhausted: precheck
````

The 100,000-character message estimates to 25,004 tokens, and with the 512 tokens kept free for the reply, the total is far past the 4096-token window. The precheck refuses the request, so nothing is sent to the provider, and with no compactor given, `models.loop` raises an error value of kind `context_exhausted` whose `reason` is `precheck`. [Compactors and context exhaustion](#compactors-and-context-exhaustion) covers that error.

### When the precheck runs

The context precheck is Runvil's own check, before each request leaves, that the conversation plus the room kept for the reply fits the context window of the model in use. A request that fits is sent. A request the precheck refuses never reaches the provider, so no model round runs for it.

The precheck runs on every `models.loop` call and before every round. A list that grows past the window between calls is refused before the next request leaves, and the precheck also guards the first request of a list, refusing it before any round runs when the list is already too large.

The precheck uses the context window and `max_tokens` of the model the round goes to. When the round comes from `h:loop`, that is the handle's own (see [Model handles](10-models.md#model-handles)). Otherwise it is the section's current model (see [How prepare binds and checks roles](10-models.md#how-prepare-binds-and-checks-roles)). A handle that `models.get` returns for a raw model id, rather than a declared model role, has a context window of 8192 tokens for the precheck, visible as `h.context`, so `models.get('qwen/qwen3-8b').context` reads `8192` (see [Getting handles with `models.get`](10-models.md#getting-handles-with-modelsget)).

Model-task notices that a round delivers are pushed onto your list as user records before the precheck (see [`await_tasks` and task notices](15-tasks.md#await_tasks-and-task-notices)). They count toward that request, and they stay on the list even when the precheck refuses it.

### The output reserve

The output reserve is the room each round keeps free for the reply, sized from the model's context window and the `max_tokens` setting of the model in use:

- When the model in use sets `max_tokens` (see [Sampling options on `models.use`](10-models.md#sampling-options-on-modelsuse)), the reserve equals that value.
- When `max_tokens` is unset, the reserve is one eighth of the context window, rounded down and capped at 8192 tokens.
- Either way, the reserve never exceeds half the context window.

| Context window | `max_tokens` | Output reserve |
|---|---|---|
| 4096 | unset | 512 |
| 65,536 | unset | 8192 |
| 1,000,000 | unset | 8192 |
| 4096 | 1000 | 1000 |
| 4096 | 100,000 | 2048 |
| 100 | 51 | 50 |

So a model with a 4096-token window and no `max_tokens` keeps a 512-token reserve, and a request must leave that room in the window. With a `max_tokens` of 1000 on the same window, the reserve is 1000 tokens, so a 12,000-character request, which estimates to 3004 tokens, is sent, while a 12,400-character request, at 3104 tokens, is refused. Without `max_tokens`, the smaller default reserve admits the longer request too.

Setting `max_tokens` at or above the context window is not rejected, and it does not make every request fail the precheck, because the reserve stops at half the window. An empty or small request still gets through: with a `max_tokens` of 100,000 on a 4096-token window, the reserve is 2048 tokens and a 400-character request is sent. The capped reserve still counts, though, so a request larger than half the window is still refused: an 8,400-character request, at 2104 tokens, is refused on that window. Any request that is too large for the model and still passes the precheck meets the provider's own limit, which refuses it.

### The admission rule

Before each round is sent, the precheck adds the request's token count to the output reserve. It refuses the round with reason `"precheck"` only when that total exceeds the model's context window, so an oversized request never leaves, and a total exactly equal to the window passes. A 103-token request with a `max_tokens` of 20 passes a 123-token window and is refused by a 122-token window.

### How the estimate counts tokens

The precheck estimates a request's token count from its text:

1. It adds up the text length of the whole conversation in UTF-8 bytes.
2. It divides that total by 4, once, dropping the remainder. This is a conservative rule of thumb of about four characters per token.
3. It adds 4 tokens of framing overhead for each record sent.

One record of 396 characters estimates to 396 / 4 = 99 tokens plus 4, or 103 tokens. The 100,000-character message in `long-notes` estimates to 25,000 plus 4, or 25,004 tokens.

The division runs once over the whole conversation, and records are counted after they merge (see [Same-role merging](#same-role-merging)), so merged records pay the per-record overhead once. Non-ASCII text weighs more per visible character, because each such character takes more than one byte.

The estimate counts:

- a record's plain string content,
- the `text` of each text part in a multipart content array,
- the whole serialized form of each tool call a record holds, not only its arguments.

Tool-call arguments on an assistant record therefore count even when the record's visible text is empty. For a multimodal message only the text parts count, and content that is neither text nor a parts array counts as zero. Image parts, tool schemas, and the system template add nothing to the estimate, so an image-heavy conversation can pass the precheck and still overflow at the provider, where the `provider` overflow reason covers it (see [Overflow reasons](#overflow-reasons)).

### The provider anchor

The provider anchor is the token count the precheck remembers from the provider's reported usage for the latest served round, and the count it starts from when the next request extends that round. After a round whose provider reported token usage, the precheck does not re-estimate the earlier request and the reply. It counts them as the provider's prompt tokens plus completion tokens, then adds the estimate for only the records appended after the reply. The output reserve still applies on top of that count.

Take a list whose first request holds the five-character user message `xxxxx`, answered with `ok` on a model with a 4096-token window, where the provider reports 3400 prompt tokens and 100 completion tokens. The anchor is 3500. When you add a 400-character user message and call `models.loop` again, the precheck counts 3500 plus 104 tokens for the new record, or 3604, and the 512-token reserve takes the total to 4116, so the request is refused. The plain estimate of the same request is about 113 tokens, which would pass.

The remembered total is the prompt tokens plus the completion tokens, less any reasoning tokens the provider reported, because the list never sends reasoning back to the model, and the total never drops below zero. A model's reasoning therefore does not count against the next request. A round reported as 3000 prompt tokens and 1200 completion tokens, 800 of them reasoning, anchors at 3400. When the provider reports no reasoning breakdown, the whole completion counts, so the same totals anchor at 4200.

Because the provider's counts include weight the estimate ignores, such as the system template and the tool schemas, the anchored count can differ from the estimate in either direction. It can refuse a request the plain estimate would admit, as in the example above. It can also admit a request the plain estimate would refuse, when the provider counted fewer tokens than the estimate, so a long conversation can pass a precheck the estimate alone would refuse. Once a round has reported usage, a conversation can pass or fail the precheck a round earlier or later than the estimate alone would decide.

### When the anchor applies

The anchor applies only when the new request extends the exact list that the measured round sent. The precheck uses the remembered total only when all three of these hold:

1. The request has more records than the measured round sent.
2. It starts with exactly the records that round sent.
3. The next record after them is an assistant record.

The count is then the remembered total plus the estimate for the records after the reply. The total already covers the reply, so the reply is never counted twice. Any other list is counted by the full estimate:

- The record right after the previously sent records is not an assistant reply, such as a user record or a tool record.
- No record follows the previously sent records, as when the exact sent request is sent again, or the list holds fewer records than were sent.
- The list no longer starts with exactly the records last sent, because you rewrote, re-roled, summarized, or merged earlier records, as with `msgs:replace` (see [Editing a list with `replace`](#editing-a-list-with-replace)). A rewritten, compacted, or merged list never reuses a provider count measured for another conversation.
- No round on the list has reported token usage yet, as when the first reply reports none.

Dropping the reply and adding another user message also drops the anchor. After `msgs:replace(#msgs, #msgs)` removes the reply and `msgs:user(...)` adds a new user record, the two adjacent user records merge into one in the request, so its first record no longer matches the one that was sent.

A round that reports no usage does not clear the remembered total, so a conversation keeps its provider-measured count across that round as long as later requests pass the same checks. A round that reports token usage becomes the measurement for the next round's precheck to the same model, whatever that round's outcome, while a round that fails outright leaves the measurement as it was.

### Which model the anchor belongs to

When a round extends the previous measured round and goes to the same model, the precheck starts its count from the provider's reported usage for that earlier round. The anchor follows the most recent served round, so a round served on a newly selected model anchors that model's next request.

- After `models.use` switches to a different model, the next request's precheck uses the estimate, because the remembered total is the first model's own token count, and another model's tokenizer count does not apply.
- Returning to an earlier model after another model served a round counts by the estimate, because only the latest round's measurement is kept.
- Switching with `models.use` to another model role that names the same model keeps the anchor, even when that role has different settings such as thinking.

### Overflow reasons

An overflow reason is the string that says which gate refused a request as too large for the model's context window. There are exactly two:

| Reason | Which gate refused | Described as |
|---|---|---|
| `precheck` | Runvil's own precheck, before sending | the request precheck overflowed the model's context window |
| `provider` | The provider, after the request was sent | the provider rejected the request as exceeding the model's context window |

You see the reason as the `reason` field of a `context_exhausted` error, testable as `err.reason == 'precheck'` or `err.reason == 'provider'`, and a compactor receives it as its argument (see [Compactors and context exhaustion](#compactors-and-context-exhaustion)).

Every refusal by the precheck has the reason `precheck`. A round overflows with reason `precheck` when the conversation as it would be sent, plus the output reserve, exceeds the model's context window. The check refuses it before any request is sent, and `models.loop` hands the overflow to the compactor with the reason `precheck`. With the default compactor, a precheck refusal makes `models.loop` raise `context_exhausted` with `reason` equal to `precheck`, as in `long-notes`.

A round overflows with reason `provider` when the provider rejects the request as too large for the model's context window, an overflow found on the provider's side rather than by Runvil's estimate. A request the precheck lets through can still overflow at the provider, so the provider's own overflow reply is the backstop. On a provider overflow, the loop calls the compactor with reason `provider` after that one request.

### How a provider overflow is recognized

A provider response with HTTP status 400 or 413 whose body contains one of these twelve phrases, ignoring case, surfaces as reason `"provider"`:

- `context length`
- `context window`
- `context size`
- `context_length_exceeded`
- `maximum context length`
- `prompt is too long`
- `too many tokens`
- `exceeds the available context size`
- `exceed_context_size`
- `input is too long`
- `exceeds the maximum number of tokens`
- `too large for model`

An error the provider sends inside an already started reply stream is classified as an overflow by the same phrases, with no HTTP status involved. Runvil classifies the provider's failure, and the loop acts only on that classification: a failure classified as a context overflow takes the provider overflow path during a section's model round instead of failing as a plain model error.

A provider rejection counts as a context overflow only when its body contains an overflow phrase. An unrelated 400 rejection never invokes a compactor, and it ends the loop as the model failure `the model backend rejected the request (status 400)`. Any other failed provider response, such as any 5xx status, any status other than 400 or 413, or a 400 or 413 without an overflow phrase, gets no compactor call and is raised at the `models.loop` call as an error of kind `internal`.

A provider context overflow is reported as a failed model round, with a `model_turn_failed` event, and it reaches the loop as an overflow with reason `provider`, not as an error.

### What a refusal leaves behind

A precheck refusal is reported as a `model_turn_failed` event, the same way a failed round is. Beyond that event, a refused request leaves no trace of a send:

- It writes no chat record to the run log, so the log holds only the requests that left.
- It records no send on the list, so the list's last recorded send, the one the [provider anchor](#the-provider-anchor) measures from, stays as it was.
- It appends no reply or tool records to the message list. A send refused for context overflow, by either gate, leaves the list as it was.
- It has no finish reason and leaves the round count unadvanced, because no round ran.

### Sending again after a refusal

Because a refused send appends nothing, the message that was too large stays where you left it, ready to edit in place with `msgs:replace`. Put this block in the `## Send` section of `long-notes`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user(string.rep('x', 100000))
local ok, err = pcall(models.loop, msgs)
if not ok and err.reason == 'precheck' then
  msgs:replace(#msgs, #msgs, { role = 'user', content = 'Summarize the notes in one line.' })
  models.loop(msgs)
end
return msgs[#msgs].content
````

With the 4096-token window, the first call is refused, `pcall` catches the error, and `msgs:replace(#msgs, #msgs, ...)` swaps the oversized message for a short one. The second `models.loop` call on the same list sends the request once it fits, and the run result is the model's reply.

The send after a refusal and an edit delivers the conversation as it now stands, earlier records plus the edited message, and the refused text never reaches the model. A caught precheck refusal does not fail the run, so the section keeps going and the run finishes with the section's result.

## Compactors and context exhaustion

Here you learn how to choose what happens when a round overflows the model's context window, and how to catch and read the `context_exhausted` error the default policy raises.

````markdown
---
name: long-notes
description: Sends notes far longer than the model's context window.
runvil: 0
models:
  writer: {}
---

# Long notes

## Send

```lua
models.use('writer')
local msgs = messages.new()
msgs:user(string.rep('x', 100000))
local ok, err = pcall(models.loop, msgs, function(reason)
  error('the notes are too long to send (' .. reason .. ')', 0)
end)
if ok then return 'sent' end
return err
```
````

With the 4096-token window, the run result is:

````text
the notes are too long to send (precheck)
````

The function passed as the second argument of `models.loop` is the compactor for this call. When the precheck refuses the round, the loop calls it with the overflow reason `precheck` instead of raising on its own. The compactor ends by raising its own error, and that error comes out of `models.loop` unchanged, so `pcall` catches the bare string `the notes are too long to send (precheck)`.

### Trimming the list after an overflow

A compactor ends the loop call; it does not shorten the list and let the loop continue. To trim the list, let the compactor raise, catch the error, edit the list, and call `models.loop` again. Put this block in the `## Send` section of `long-notes`:

````lua
models.use('writer')
local msgs = messages.new()
msgs:user(string.rep('x', 100000))
local overflow
local ok = pcall(models.loop, msgs, function(reason)
  overflow = reason
  error('the notes are too long to send (' .. reason .. ')', 0)
end)
if not ok and overflow then
  msgs:replace(#msgs, #msgs, { role = 'user', content = 'Summarize the notes in one line.' })
  models.loop(msgs)
end
return msgs[#msgs].content
````

The compactor records the reason in the enclosing `local overflow`, so the code after `pcall` knows the failure was an overflow. A round refused for overflow adds no records to the message list, while records from rounds that already completed stay, so you can shorten the last message with `msgs:replace(#msgs, #msgs, ...)` and call `models.loop(msgs)` again. The second call leaves the compactor out, so it uses the default, and the run result is the model's reply. A list you trim this way no longer starts with the records the last measured round sent, so its next request is counted by the full estimate (see [When the anchor applies](#when-the-anchor-applies)).

### The `compactors` global

A compactor is the policy that handles a conversation that overflows the model's context window: the function `models.loop` calls when a round overflows, with the overflow reason as its argument.

A prompt's Lua reads the `compactors` global, a table of the shipped compactor policies holding the single key `fail`. It is an Engine global, installed alongside `messages`, so every section and every H1 block runs with it already in place, and you use it, as in `compactors.fail`, without declaring it. `compactors.fail` is the only compactor policy Runvil ships, and it is the default compactor for `models.loop`.

### Choosing a compactor for a call

The compactor for a call is the optional second argument of `models.loop`, and of `handle:loop` (see [How the loop runs](#how-the-loop-runs)):

- Leaving the argument out selects `compactors.fail`, so `models.loop(msgs)` and `models.loop(msgs, compactors.fail)` behave the same, and the same holds for `handle:loop`.
- Passing `nil` selects the default just as leaving the argument out does.
- Passing `compactors.fail` explicitly works on either loop entry, as in `models.loop(msgs, compactors.fail)` or `other:loop(msgs, compactors.fail)`, and a normal round runs as usual.
- Passing any Lua function makes it the compactor, used as given.

The loop invokes the compactor only when an overflow happens. When a round's request cannot fit the model's context window, the loop calls the selected compactor with the overflow reason tag instead of handling a reply. The compactor also runs after a sent round comes back as a provider context overflow, so for a provider overflow it runs after exactly one request was sent and rejected.

The default is the `fail` field of the `compactors` table the Engine installed, so `models.loop(msgs)` on an oversized list fails with a `context_exhausted` error whose reason is `precheck`, and assigning another table to the `compactors` global does not change the default. [Replacing the default compactor](#replacing-the-default-compactor) shows when the loop reads that field.

An explicit compactor argument must be a function or `nil`. Any other value makes `models.loop` or `handle:loop` raise the `lua`-kind `compactor must be a function, got {type}` error before any request is sent, as [Argument errors](#argument-errors) shows. The default read from `compactors.fail` is not checked this way.

### `compactors.fail`

`compactors.fail` is the default overflow policy. It never compacts: under it, an overflowing round raises `context_exhausted`. It is a function that takes the overflow reason tag as its one string argument, `"precheck"` or `"provider"`, and it always raises a `context_exhausted` error value whose `reason` is that tag. It never returns normally.

You can also call it from your own Lua, as in `compactors.fail('precheck')`. Whether a `models.loop` round overflows under it or your Lua calls `compactors.fail(tag)`, a `pcall` around that call catches the same value: a `context_exhausted` error value with the message `context exhausted: {reason}`, where `{reason}` is the reason in words, and the field `reason` set to `"precheck"` or `"provider"`. The caught value keeps its kind and reason, never flattened to plain text. Under the loop the error surfaces at the `models.loop` call site, and from a direct call it surfaces at that call. Left uncaught, the two end the run differently, as [When context exhaustion goes uncaught](#when-context-exhaustion-goes-uncaught) shows.

`compactors.fail` accepts exactly two reason tags, the lowercase `precheck` and `provider`, matched exactly as written. Any other string raises a Lua argument error whose message names the text it received and the accepted tags:

````text
invalid overflow reason `{input}`: required one of `precheck`, `provider`
````

That error has kind `lua`. It is an authoring error, never context exhaustion.

When the provider rejects a sent request for exceeding its context, for example with status 400 and `This model's maximum context length is 4096 tokens.`, `compactors.fail` raises `context_exhausted` with the reason `provider`, instead of the request failing as an ordinary model failure.

### Reading a `context_exhausted` error

`context_exhausted` is the error kind for a request that overflows the model's context window when the selected compactor does not compact. When the conversation cannot fit, `models.loop` raises an error value whose `kind` is `context_exhausted`, and you catch it at the call site with `local ok, err = pcall(models.loop, msgs)`. The caught value is an error value that holds its kind, so `err.kind .. ':' .. err.reason` works on it, and the `long-notes` block at the start of [The context precheck](#the-context-precheck), with `err.kind .. ': ' .. tostring(err.reason)`, gives `context_exhausted: precheck`.

Every `context_exhausted` error, whether a compactor raises it or context exhaustion does, holds the overflow reason that triggered it in a string `reason` field. The overflow reason passed to a compactor is the same tag as the `reason` field of the `context_exhausted` error, so `err.reason` reads `precheck` or `provider`.

The message, which `tostring(err)` returns, begins with `context exhausted: ` and continues with the reason in words:

````text
context exhausted: the request precheck overflowed the model's context window
context exhausted: the provider rejected the request as exceeding the model's context window
````

The two reasons render different messages, so you can tell a `precheck` exhaustion from a `provider` exhaustion by the text as well as by `err.reason`.

Context exhaustion is not limited to `models.loop`: when `models.infer` fails because the context window is exhausted, the caught error also has kind `context_exhausted` and a `reason` field, such as `provider` (see [Running a round with `models.infer`](10-models.md#running-a-round-with-modelsinfer)).

### Writing your own compactor

Any Lua function is accepted as a custom compactor for `models.loop` or `handle:loop`. When a round overflows the model's context, the loop calls it instead of raising, passing the overflow reason `"precheck"` or `"provider"` as its single string argument. That reason is the whole compactor contract: the compactor receives exactly one argument, so `select('#', ...)` inside it reads 1. A compactor can record the reason in an enclosing `local` for the code after the loop call to read, as the `overflow` local does in [Trimming the list after an overflow](#trimming-the-list-after-an-overflow).

A compactor must raise. A compactor that returns at all, with or without a value, is refused, even when the value it returns is a record for the list, and `models.loop` raises an error of kind `lua` whose message names `compactors.fail`:

````text
the selected compactor returned without raising: replacement compactors are deferred; compactors.fail is the only shipped policy
````

The loop raises this error even while the run is being cancelled. When a compactor's return is refused, whatever it returned is not appended, and the message list is left unchanged. A context overflow therefore always ends the loop call with an error, because the loop raises whether the compactor returns or raises, and no further round starts.

A compactor can run its own model rounds, such as `models.infer('summarize')`, which suspend and resume inside the loop like any other suspending call. In this compactor, an infer round that replies `summary` makes the loop call fail with `compacted after summary`:

````lua
local ok, err = pcall(models.loop, msgs, function(reason)
  local summary = models.infer('summarize')
  error('compacted after ' .. summary, 0)
end)
````

### What a compactor's error becomes

A compactor runs inside the loop under a protected call, so an error it raises goes to the loop's rules as a failed outcome instead of escaping on its own, and only its first returned or raised value goes back to the loop. An error raised inside the compactor, such as `error('compacted', 0)`, ends `models.loop` and propagates out of it to the caller, and the author's message list stays unchanged, so `#msgs` stays the same.

When the compactor raises, `models.loop` raises that failure after normalizing it into an error value, the same normalization a `pcall` capture applies:

- An Engine failure, such as the `context_exhausted` error `compactors.fail` raises, arrives as the same error value `pcall` would give, with its `kind` and fields, such as `kind = "context_exhausted"` with `reason = "precheck"`.
- A string or a table the compactor raises reaches the caller unchanged, as [Errors while a round runs](#errors-while-a-round-runs) describes. You can embed the reason tag in your own message, so `error('custom failure: ' .. reason, 0)` arrives as the bare string `custom failure: provider`. With `own = { reason = 'mine' }` and `error(own)` in the compactor, `err == own` is true and `getmetatable(err)` is `nil`.

When the run is being cancelled while the compactor runs, and the compactor raises, `models.loop` raises the compactor's failure raw, unchanged, without normalizing it.

### Replacing the default compactor

You can assign your own function to `compactors.fail`, and later `models.loop` calls with no explicit compactor use it as the default. With this block in the `## Send` section of `long-notes`:

````lua
models.use('writer')
compactors.fail = function(reason) error('replaced: ' .. reason, 0) end
local big = messages.new()
big:user(string.rep('x', 100000))
local ok, err = pcall(models.loop, big)
return err
````

the 4096-token window gives:

````text
replaced: precheck
````

The default is never checked up front. Whatever `compactors.fail` holds is read when each loop call begins and called only when a round overflows, so when it holds a value that is not a function, `models.loop(msgs)` still runs ordinary rounds normally and fails only when a round overflows. At that point the call raises Lua's own call error as a plain string, not an error value with a `kind`, such as this one for a number:

````text
attempt to call a number value
````

### When context exhaustion goes uncaught

Context exhaustion raises at the `models.loop` call site. Left uncaught, it fails the section, so code after the call never runs, and it ends the run with run error kind `ContextExhausted` (see [Run error kinds](16-limits-and-errors.md#run-error-kinds)) and a message of the form `context exhausted: {reason}`. An uncaught `context_exhausted` error ends the block as that same `context_exhausted` error, with its fields, including `reason`, kept. A `context_exhausted` error value that escapes a block ends the run as a `ContextExhausted` failure with the same `reason`, whether `models.loop` raised it or your Lua caught it with `pcall` and raised it again with `error(err)`.

A direct call to `compactors.fail` ends the run differently. Left uncaught, that call fails the section as an ordinary Lua runtime error, and the run ends with run error kind `Lua`, not `ContextExhausted`. To end the run as context exhaustion from your own Lua, catch the call's error with `pcall` and raise it again:

````lua
local ok, err = pcall(compactors.fail, 'precheck')
error(err)
````

This block ends the run with run error kind `ContextExhausted` and the reason `precheck`.

A compactor's own error takes a different path. An uncaught string raised by a compactor fails the section as an ordinary Lua runtime error holding the compactor's message, and the code after the call never runs. Left uncaught in a walked section, that error ends the run with run error kind `Lua`, with the compactor's error text in the message.
