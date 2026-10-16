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
| `capabilities` | `[(String, Json)]`                           | what else the server says it can do              |

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
