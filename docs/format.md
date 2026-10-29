# Formatting: `format!`

`format!` declares a formatter against a grammar's rules. From one declaration
it writes two functions: one that sets a program out, and one that says how far
in the next line of an unfinished program should start.

```meadow
format! {
  pub Style for Mini
  indent 2
  width 80
    | Let = "let" recursive name params "=" group(indent(line value) line "in") newline body
    | Lam = "fun" params "->" indent(line body)
    | If = "if" cond indent(line "then" yes) indent(line "else" no)
    | Paren = "(" indent(cut expr) cut ")"
}
```

For a formatter `Style` of a grammar `Mini` this writes:

| function                              | answers                                              |
| ------------------------------------- | ---------------------------------------------------- |
| `formatStyle : Green Mini -> String`  | the tree's text, set out                             |
| `indentStyle : Green Mini -> String`  | what the line after an unfinished entry opens with   |

Both take the lossless tree `parseMini` makes, so both work on a program with
errors in it.

## The header

| part         | meaning                                                          | default         |
| ------------ | ---------------------------------------------------------------- | --------------- |
| `pub`        | export what is written                                           | private         |
| `Style`      | the formatter's name: `formatStyle`, `indentStyle`               |                 |
| `for Mini`   | the grammar, one that `syntax!` declared                         |                 |
| `indent N`   | how many spaces a level is                                       | 2               |
| `width N`    | how wide a line may be                                           | none: indent only |

`indent` comes before `width`. Whether there is a `width` decides which of two
modes the formatter works in; see [Two modes](#two-modes).

## Rules

A formatter rule is a grammar rule written again as what a node of it holds,
in the grammar's order, with layout between the pieces.

Take the grammar rule

```
Let = "let" recursive:"rec"? name:Name params:Param* "=" value:Expr "in" body:Expr
Paren = "(" Expr ")"
```

Its formatter rule names the same things in the same order:

```meadow
| Let = "let" recursive name params "=" group(indent(line value) line "in") newline body
```

**Pieces.**

- A **token** is written as it is in the grammar, in quotes: `"let"`, `"="`.
- A **child** is written by its label: `value`, `body`. A child the grammar did
  not label goes by the name its accessor has: `Paren` holds an unlabelled
  `Expr`, so its rule says `expr`.

**Layout**, between and around them:

| written     | meaning                                                                   |
| ----------- | ------------------------------------------------------------------------- |
| `indent(…)` | what it encloses is one level in from where the node began                |
| `indent 3(…)` | what it encloses is 3 columns in; `indent -1(…)` is one column out      |
| `group(…)`  | what it encloses goes on one line if it fits, and breaks together if not  |
| `line`      | a space, or a line break where its group breaks                           |
| `cut`       | nothing, or a line break where its group breaks                           |
| `newline`   | a line break, always; its group and every group around it break           |
| `tight`     | nothing: the two things touch                                             |
| (nothing)   | a single space                                                            |

A rule as a whole is a group. When several separators come in a row the
strongest wins, in the order `newline`, `line`, `cut`, `tight`. A separator
that a rule begins or ends with belongs to the parent: it separates that node
from whatever is next to it, and it counts in the run of separators there. A
child whose rule begins with one gets no extra space before it, and one whose
rule ends with one gets none after it. The same goes for a separator inside
an `indent(…)` at the start or end of a rule: it stays inside, at that
indentation, but a separator beside it adds only what it asks for beyond
that. A line break there and a line break beside it make one line break.

`indent` and `group` are layout only when followed by parentheses, or by a
number and parentheses. Anywhere else they are a child's name, so a grammar
may label a child `group`.

**`nogroup`.** A rule that begins with `nogroup` is not a group of its own.
Its separators break when the enclosing group breaks. That is what a
right-nested chain needs, such as `a -> b -> c` read as `a -> (b -> c)`. With
each link a group, the outer link breaks and an inner one still fits, so the
chain comes out half broken. With `nogroup` the whole chain breaks together:

```meadow
format! {
  Hung for Blocks
  indent 2
  width 30
    | Bind  = name indent 2(line indent 1("=") value)
    | Chain = nogroup from line "->" to
}
```

```text
x
   = aaaaaaaaaa
  -> bbbbbbbbbb
  -> cccccccccc
  -> dddddddddd
```

The line break before `=` is set in to its own column (2 + 1), and the
value's are set in to 2, so whatever follows `=` and each `->` starts in one
column.

**Not every rule needs one.** A node with no formatter rule is written as it
was, with its line breaks where they were, set in by whatever is around it. A
formatter can be adopted a rule at a time.

**Errors**, each reported where the name was written:

- `no grammar is called X`: the name after `for` is not a `syntax!` grammar.
- `X is kept as tables: a formatter is of a tree`: `for` named a table language.
- `no rule of the grammar is called X -- or it is a choice of rules`: a rule
  for something the grammar lacks, or for an enum such as `Expr`, which has no
  node of its own. Write rules for its alternatives instead.
- `Let has no child called x: it has …`: a child the rule does not have. The
  message lists the ones it does.

## Two modes

### With a width: laying out

`formatStyle` lays the program out within the width. Each node with a rule goes
on one line if that fits, and breaks where its rule says if not. Each child is
then laid out by its own rule.

```meadow
formatStyle (fst (parseMini "let x =\n1 + 2\nin\n      if x < 3\nthen (\nx\n)\nelse fun y ->\ny\n"))
-- "let x = 1 + 2 in\nif x < 3 then (x) else fun y -> y\n"
```

The same rules at `width 20`:

```text
let x = 1 + 2 in
if x < 3
  then (x)
  else fun y -> y
```

and with a value too long for its line, the `group` in `Let` breaks as one:

```text
let total =
  (
    first + second + third
  )
in
total
```

A `newline` forces its surroundings to break. `let x = let y = 1 in y in x`
cannot go on one line because the inner `let` has a body that starts a line:

```text
let x =
  let y = 1 in
  y
in
x
```

Line breaking is done by [Pretty](https://github.com/meadow-lang/Pretty),
Wadler's printer.

### Layout-sensitive grammars

Where indentation is part of the syntax, as in Idyll or Haskell, a layout pass
between the lexer and the parser usually inserts tokens with no text: one
where a block opens, one where each item starts, one where the block closes.
The formatter writes such a token as nothing, and nothing gets separated
around it either. The separators on each side apply as if it weren't there.
Three rules are enough to keep each block's items at the block's column:

```meadow
format! {
  Laid for Blocks
  indent 2
  width 40
    | Sep  = newline sep tight
    | Bind = name "=" indent(line value)
    | LetE = "let" open indent(line entries) close "in" body
}
```

`Sep`, the layout's `;`, is a separator and nothing more. Between two items it
is exactly one line break, so every item starts a line at the same level.
`open` and `close` print nothing, so `let a = 1 in a` stays on one line.
Two bindings break it:

```text
x =
  let
    a = 1
    b = 2 in a
```

[Blocks.mw](../examples/MiniML/src/Blocks.mw) has the layout pass, the grammar
and tests that read the result back as the same program.

### Without a width: indenting only

Leave `width` out and the formatter never moves a line break. It only sets each
line in:

```meadow
format! {
  Indents for Mini
  indent 2
    | Let = "let" recursive name params "=" indent(value) "in" body
    | Lam = "fun" params "->" indent(body)
    | If = "if" cond indent("then" yes) indent("else" no)
    | Paren = "(" indent(expr) ")"
}
```

```text
let x =                 let x =
1 + 2                     1 + 2
in                      in
      if x < 3     ──▶  if x < 3
then (                    then (
x                           x
)                         )
else fun y ->             else fun y ->
y                           y
```

In this mode `line`, `cut`, `newline` and `tight` say nothing, and a `group` is
just what it encloses. Only `indent(…)` matters, and `indent 3(…)`, which sets
a line in by those columns rather than by a level.

**A level is per line, not per node.** A line is set in by one level for each
*line* on which a node began whose indented part the line starts in.
`let f = fun y ->` opens a `let` and a `fun` on one line, so what follows is in
by one level, not two:

```text
let f = fun y ->        let f =
  y + 1                   fun y ->
in f 2                      y + 1
                        in f 2
```

A line that starts inside a token, such as a string or comment that runs over
several lines, is left alone.

## What formatting never changes

Formatting changes the space between tokens and nothing else. This is checked
by the tests for both whole and broken programs.

- **Every token survives, in order.** The rules are matched against the
  lossless tree's children as they are met. A child a rule did not name,
  something the parser could not place, is written where it was.
- **Broken programs are formatted as far as they read.** A rule whose node is
  missing a child lays out what is there. `let x =\n(1 +\nin x` still gets its
  value indented.
- **Comments keep a line of their own** before what they were written before.
  A comment inside a node breaks whatever group holds it. A comment written at
  the end of a line, after the token before it, stays at the end of that
  line, with a line break after it.
- **An `indent(…)` with nothing in it** writes nothing, including the
  separators inside it. So `indent(newline entries)` with no entries adds no
  line break.
- **Text the lexer skipped**, such as blank space before the first token that
  a layout pass doesn't hand over, is trivia. The parser keeps it as
  `Unknown`.
- **It is idempotent.** Formatting twice gives what formatting once did.
- **A trailing newline is kept** if the text had one. A line that would be
  nothing but its indent is left empty.
- When laying out, a node with no rule keeps one blank line between two
  things; a longer run of blank lines becomes one. A node with a rule keeps
  one in the same way between two items of a child that is many, such as a
  file's declarations. A line break a child's rule asks for is not added on
  top of one the text already has.
- **A node with no rule keeps how far in each of its lines was.** A line it
  breaks to is set in by as much as it was further in than the line the node
  began on, on top of whatever sets in the node. A line at or left of that line
  gets nothing extra. In a layout language a line further in continues the one
  before, so this keeps what the program means. A child whose rule begins
  with a line break is the exception: its rule sets it in.

## The next line's indent

`indentStyle` answers what the line after an **unfinished** text should open
with. A prompt asks it when Enter does not finish an entry, so an entry is
indented as it is typed the way it would be formatted once whole.

It answers one level for each line on which a node began that is still open at
the end of the text, in its indented part. A node is open there when either:

- its indented part is there and runs to the end, as in `let x = 1 +`; or
- its indented part has not started and nothing follows it, as in `let x =`.

With the rules above and `indent 2`:

| text so far            | next line opens with |
| ---------------------- | -------------------- |
| `let x =`              | 2 spaces             |
| `let x = 1 +`          | 2 spaces             |
| `let x = 1 in`         | nothing              |
| `let f = fun y ->`     | 2 spaces             |
| `let f =` ⏎ `  fun y ->` | 4 spaces           |
| `if a`                 | 2 spaces             |
| `(`                    | 2 spaces             |
| `(a)`                  | nothing              |

`indentStyle` works the same whether or not the formatter has a `width`.

## Using it

**In the editor.** Give `lsp!` a function that formats a file, and the server
answers `textDocument/formatting` with one edit replacing the file, or with no
edit when the text is already set out:

```meadow
fun formattedOf db file = formatStyle (treeOf db file)

lsp! {
  pub Server "miniml" for Session
    | text = setSessionSource
    | tree = treeOf
    | format = formattedOf
}
```

**At the prompt.** Give `repl!` the indent function directly:

```meadow
repl! {
  pub Prompt "miniml" for Session
    …
    | indent = indentStyle
}
```

See [repl.md](repl.md) and [highlighting.md](highlighting.md).

## Underneath

A formatter is a value, `Lingua.Format.Style k`: the rules as data, matched
against a node's children at run time. `format!` writes that value and two
one-line functions over it. The same module can be used by hand:

| function                 | what it does                                   |
| ------------------------ | ---------------------------------------------- |
| `style unit width name trivia rules` | a formatter; `width` 0 for indent only |
| `tok`, `child`, `indented`, `indentedBy n`, `grouped` | the pieces of a rule |
| `nogroup`                | a rule's first element, for one that is no group |
| `line`, `cut`, `newline`, `tight`     | the separators                  |
| `layout st g`            | what `format<Name>` calls when there is a width |
| `reindent st g`          | what it calls when there is none               |
| `continuation st g`      | what `indent<Name>` calls                      |
