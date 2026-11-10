# The language server: `lsp!`

`lsp!` declares a language server over a compiler's database. Each feature is
a function of the database and a file; the server speaks the protocol and says
it can do exactly what it was given.

```meadow
lsp! {
  pub Server "miniml" for Session
    | text = setSessionSource
    | diagnostics = diagnosticsOf
    | tree = treeOf
    | hover = hoverAt
    | definition = definitionAt
    | references = referencesAt
    | completion = completionAt
    | symbols = symbolsOf
    | format = formattedOf
    | resolved = resolvedOf
    | requests = serverRequests
}
```

This writes `serveServer ()`, the server on standard input and output until
the editor says `exit`, and, with a `tree`, `tokensServer db file`
(see [highlighting.md](highlighting.md)). `for Session` means the server makes
its database with `newSession ()`.

## Features

Only `text` is required. Files are numbered by the server as they are opened;
places are byte offsets, which is what a compiler counts in. The server
converts to and from the protocol's lines and UTF-16 columns.

| feature        | type                                         | answers                                          |
| -------------- | -------------------------------------------- | ------------------------------------------------ |
| `text`         | `db -> Int -> String -> ()`                  | how an edit reaches the compiler                 |
| `opened`       | `db -> Int -> String -> ()`                  | where a file is, when the editor opens it: its uri |
| `closed`       | `db -> Int -> ()`                            | a file the editor closed, for the language to forget |
| `diagnostics`  | `db -> Int -> [Diagnostic]`                  | published for every open file after each edit    |
| `tree`         | `db -> Int -> Green k`                       | semantic tokens, folding ranges, selection ranges |
| `resolved`     | `db -> Int -> [(Int, String)]`               | what name resolution says particular tokens are  |
| `hover`        | `db -> Int -> Int -> Maybe (Meta, String)`   | hover, as markdown                               |
| `definition`   | `db -> Int -> Int -> [(Int, Meta)]`          | go to definition                                 |
| `references`   | `db -> Int -> Int -> [(Int, Meta)]`          | references, highlights, prepare rename, rename   |
| `completion`   | `db -> Int -> Int -> [(String, String)]`     | completion: each a label and a detail            |
| `symbols`      | `db -> Int -> [(String, String, Meta)]`      | document symbols: name, kind, place              |
| `format`       | `db -> Int -> String`                        | formatting, as one edit (see [format.md](format.md)) |
| `requests`     | `[(String, db -> Asked -> Json)]`            | the language's own requests, below               |
| `refuse`       | `db -> Asked -> Maybe (Int, String)`         | a request answered with an error instead, below  |
| `notifications` | `[(String, db -> Asked -> Bool)]`           | the language's own notifications, below          |
| `capabilities` | `[(String, Json)]`                           | what else the server says it can do              |

`opened` is called when a file is opened, before its `text`, with the
file's number and its uri. `Lingua.Lsp.pathOf` turns a `file:` uri into the
path on disk, so `file:///c%3A/a%20b.idl` becomes `c:/a b.idl`. Like every
feature, its function can do anything the server can, reading the disk
included. A language that checks a file together with the rest of its
package can use it to find the package's manifest and set the other modules'
texts as inputs. A query can't read the disk, but `opened` can, before
anything is asked.

`closed` is called when the editor closes a file, with the file's number.
The editor is told nothing is wrong in that file any more. A language that
keeps its own copy of each open file's text drops it here, and goes back to
what is on disk. That can change what is wrong in the files still open, so
after `closed` they are all looked at again. On threads, any looking still
under way is stopped first, as for an edit. A server without `closed` only
clears the closed file.

## A language's own requests

`requests` is a list of methods and what answers each. It is for two things:

- **a method of the protocol's that Lingua has no feature for**, such as
  `textDocument/inlayHint`, `textDocument/codeLens`, `workspace/symbol` or
  `completionItem/resolve`;
- **a method only this language has**, such as a request for a definition's
  intermediate code.

```meadow
def serverRequests =
  [
    ("miniml/type", \db (a : Asked) -> J.jString (typeIn db a.file)),
    ("textDocument/inlayHint", hintsOf)
  ]

fun hintsOf db (a : Asked) =
  let t = typeIn db a.file in
  if S.isEmpty t then J.jArray []
  else
    J.jArray
      [J.jObject [
         ("position", positionIn a (stringByteLength (S.trimEnd a.text))),
         ("label", J.jString ": ${t}"),
         ("paddingLeft", J.jBool True)
       ]]
```

An answer is a function of the database and what was asked,
`Lingua.Lsp.Asked`, and is the request's result as JSON (`Std.Json`):

| field     | holds                                                              |
| --------- | ------------------------------------------------------------------ |
| `method`  | the method                                                         |
| `file`    | the file its `textDocument` names, or `-1`                         |
| `text`    | that file's text                                                   |
| `offset`  | the byte its `position` is at, or `0` if it has none               |
| `params`  | its parameters, as they came                                       |
| `docs`    | the documents open: `uriOf`, `textOf`, `fileOf`, `openFiles`       |

For writing the answer in the protocol's terms:

| function                    | answers                                                    |
| --------------------------- | ---------------------------------------------------------- |
| `rangeIn a meta`            | a `Meta` of the file asked about, as a range               |
| `positionIn a offset`       | a byte of it, as a position                                |
| `locationIn a (file, meta)` | a place in any open file, as a location                    |
| `byteIn a ["range", "start"]` | the byte a position among the parameters is at           |

Rules:

- **The language's answer comes first.** A request given for a method Lingua
  does answer, `textDocument/completion` say, is asked in Lingua's place.
- **A request about no open file** is asked only if the method is one of the
  language's own, with `file = -1` and an empty text. `workspace/symbol` is
  such a request.
- **The server says it can.** For `textDocument/inlayHint`,
  `textDocument/codeLens` (with `codeLens/resolve`), `workspace/symbol` and
  `completionItem/resolve` the capability is written for you. For any other
  method of the protocol's that an editor must be told of, give it in
  `capabilities`; of two that say the same thing, yours is the one sent.
- A method of the language's own needs no capability. The editor's extension
  sends it.
- **A method nobody answers** gets the error `-32601` with the message
  `unhandled request: <method>`, whether or not the file it names is open.
  A method Lingua answers, asked about a file that isn't open, gets `null`.

## Refusing a request

`refuse` is asked about every request before anything answers it, Lingua's
own included. Return `Just (code, message)` to answer with that error
instead, or `None` to let the request be answered as usual.

```meadow
fun refusalOf db (a : Asked) : Maybe (Int, String) =
  if a.method != "textDocument/rename" then None
  else
    let name = newName a.params in
    if isName name then None else Just (requestFailed, "`${name}` is not a name")
```

`Lingua.Lsp` names the codes: `requestFailed` (`-32803`), `invalidParams`
(`-32602`), `methodNotFound` (`-32601`), `invalidRequest` (`-32600`) and
`serverNotInitialized` (`-32002`). A refusal keeps Lingua's own answer for
every other case, so a language that can't rename a module refuses that one
rename and still has Lingua rename everything else.

## A language's own notifications

The server handles `initialized`, `exit` and the document notifications
itself. `notifications` gives what the language does when it hears any other
notification, such as `workspace/didChangeWatchedFiles`. Each is asked with
the notification as an `Asked`, its file `-1` unless the notification names an
open document. It returns whether what is wrong in the open files may have
changed. On `True`, every open file is looked at again and its diagnostics
published.

```meadow
def serverNotifications =
  [("workspace/didChangeWatchedFiles", \db (a : Asked) -> restudied db a.params)]
```

A threaded server stops any looking under way before the language hears a
notification, since what it does may set an input. Whatever the looking hadn't
finished is looked at again, even when the language returns `False`.

Any other notification is ignored.

## The session

The server follows the protocol's lifecycle:

- **Before the editor says `initialized`**, every request except `initialize`
  is answered `serverNotInitialized`.
- **After `shutdown`**, every request is answered `invalidRequest` with the
  message `the server is shutting down`. `exit` ends the server.
- **Closing a document forgets it.** The server publishes an empty list of
  diagnostics for it, since nothing will update them now. After that it
  isn't looked at after edits, and a request about it is asked as about a file
  that isn't open. Opened again, it keeps its number, and `opened` is told
  again.

`initialize` is answered with the capabilities alone. The server doesn't
write `positionEncoding` (UTF-16 is what the protocol assumes) or
`serverInfo`.

## Testing a server

`Lingua.Lsp.told messages server` runs a server against the messages an editor
would send, and answers what it wrote, each message as JSON:

```meadow
@test fun theServerAnswersARequestOfMiniMLsOwn () =
  let said =
    told
      [opening "file:///a.ml" "1 + 2\n", asking 1 "miniml/type" (inFile "file:///a.ml")]
      serveServer
  in assertEq
       (M.andThen (J.get "result") (V.get said 1))
       (Just (J.jString "Int"))
       "what the program is, from the session an edit set the text of"
```

The first thing written after an edit is the diagnostics of every open file,
so a request's reply comes after those.

`told` begins the session as an editor does. If `messages` have no
`initialize`, it sends `initialize` and `initialized` first, and leaves its
own `initialize`'s reply out of what it returns. If they have `initialize` but
no `initialized`, it sends `initialized` right after. To send exactly the
messages given, as a test of the lifecycle must, use `toldAsSent`.
[Editor.mw](../examples/MiniML/src/Editor.mw) has the helpers `opening`,
`asking` and `inFile`.

## On threads

By default a server does one thing at a time: a message is read when the one
before it has been answered, and each edit publishes the diagnostics of every
open file before the next message is looked at. A request waits behind that.

`threaded`, after the database's name, makes it a server on three threads:

```meadow
lsp! {
  pub Server "miniml" for Session threaded
    | text = setSessionSource
    …
}
```

| thread            | does                                                          |
| ----------------- | ------------------------------------------------------------- |
| one that reads    | reads each message as it comes, so the editor is never kept waiting to be read |
| one that looks    | runs `diagnostics` over every open file after an edit         |
| one that answers  | keeps the open documents, answers requests, and is the only one that writes |

What this changes:

- **A request is answered while the files are still being looked at.**
  Formatting a file needs only its tree, and no longer waits for it to be
  checked. A request that needs what the other thread is working out waits
  for that query and no longer.
- **An edit stops the looking.** The answering thread interrupts the database
  (`interrupt<Db>`, see [drivers.md](drivers.md#stopping-work-under-way)),
  waits for the thread to end, sets the text and starts again. So a burst of
  edits publishes diagnostics once, for the text that is there at the end.
  Twenty edits sent together to MiniML's server publish once.
- **What was found for a text that has gone is not sent.**

What it asks of the language:

- **`diagnostics` must answer and not raise.** It runs on a thread of its
  own, and a query stopped under it raises `QueryError.Interrupted`. Catch
  it and answer nothing, as MiniML's does with `Std.Exn.toResult`; what it
  answers then is thrown away. A thread starts with no handlers
  (`Std.Thread`), so there is nothing on it to catch what `diagnostics` lets
  out.
- Every other feature runs on the answering thread, as before.

A threaded server also writes `serve<Name>InTurn ()`, the same server one
thing at a time. It is what a test tells with `told`, since a thread that
reads does not read what a test supplies.

Leave `threaded` out for a server whose language checks a file faster than
anyone types.

## Limits

- **Requests are answered one at a time.** On threads, a request that takes
  long keeps the next request waiting, though not the reading of edits.
- **Whole documents.** The server asks the editor for a file's whole text on
  each change.
- **A request is not cancelled.** `$/cancelRequest` is ignored; only looking
  for what is wrong is stopped, and by an edit.
- **Semantic tokens have classes and no modifiers.**
- **The server sends nothing but replies and diagnostics.** A language's
  request cannot make the server ask the editor for something.
