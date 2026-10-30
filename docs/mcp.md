# Tools for agents: `mcp!`

`mcp!` serves a language's tools to an agent over the
[Model Context Protocol](https://modelcontextprotocol.io). An agent can check a
file, list what's left to prove, ask for a type, and run whatever else the
language offers. The tools are the same functions a language's `lsp!` and
`repl!` call, over the same `database!` session.

```meadow
mcp! {
  pub Agent "miniml" for Session
    | instructions = "Type and run MiniML programs."
    | tool typeOf "the type of a MiniML program" (source : String) = typeTool
    | tool describe "a program's type and value, in a file" (source : String, file : Maybe Int) = describeTool
}
```

## Settings and tools

| line | what it is |
| ---- | ---------- |
| `instructions = "…"` | what the server is for; an agent is told when it connects |
| `version = "…"` | the server's version, `0.1.0` unless it says |
| `tool name "what it does" (params) = handler` | a tool |

A tool's description is what an agent reads when choosing a tool, so write it
for the agent. Each parameter is a `String`, an `Int` or a `Bool`, or `Maybe`
one of them if the agent may leave it out. The schema the agent is shown is
derived from the parameters: their names, their JSON types, and which ones
are required. A tool with no parameters may leave out the parentheses.

A handler takes the database and the arguments, in order, and returns one of
two things:

- **`Result String String`**: the text the agent reads, or `Err` with the
  reason the tool failed. A failure is reported as the tool's own error
  (`isError`), which the agent can act on. It is not a protocol error.
- **`Result String Json`**: the same, with a `Std.Json` value. The agent gets
  it as text, and also as structured data when it is an object.

```meadow
fun typeTool db (source : String) =
  let u = setSessionSource db 1 source in
  match sessionTyped db 1 with
  | Ok t -> Ok t
  | Err _ -> Err "the program has no type: it has a mistake in it"
```

A handler can do anything an ordinary function can: ask the database, read
the disk, run proof search. A handler doesn't need to catch what the database
raises (for example, asking for an input nobody set, or a query that depends
on itself). The server catches it and reports it as the tool's error, and
keeps serving. A handler that raises any other type of exception must catch
it itself.

## What is written

For a server `Agent` over a database `Session`:

| function | what it is |
| -------- | ---------- |
| `serveAgent ()` | the server: a new session, then every message read from standard input answered on standard output, until the input ends |
| `answerAgent db message` | the reply to one message, or `None` for a notification: for a test, or another transport |
| `toolsAgent ()` | the tools, as `Lingua.Mcp.tool` builds them |

## The protocol

Messages are JSON-RPC 2.0, one per line, which is the protocol's stdio
transport. The server answers:

- `initialize` with the protocol version the client asked for (or
  `2025-06-18`), its name and version, that it has tools, and its
  instructions;
- `tools/list` with each tool and its input schema;
- `tools/call` with the handler's answer. If an argument is missing or has
  the wrong type, or there's no such tool, it answers with error `-32602`;
- `ping` with an empty result.

Notifications, `notifications/initialized` among them, get no reply. Any other
method gets error `-32601`.

## Testing a server

`Lingua.Mcp.told messages server` returns what `server` writes when it is sent
`messages` and then the input ends: its replies, in order, parsed back into
JSON.

```meadow
let said = Mcp.told [listTools, callTypeOf "fun x -> x + 1"] serveAgent in
…
```

[Editor.mw](../examples/MiniML/src/Editor.mw) has MiniML's `Agent` and a test of
it. [Mcp.mw](../src/Mcp.mw) has a server built by hand from `Lingua.Mcp.tool`,
and tests of the protocol.
