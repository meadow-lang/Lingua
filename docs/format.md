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
| `formatStyleWithin : Int -> Green Mini -> String` | the same, a width given: 0 only indents |
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
| `skip "…"`   | a node with a child reading as this, blank space aside, is kept as it was written; may be given more than once | none |
| `reparse "|" f else g` | a macro body's entry starting `|` is read by `f`, any other by `g`, and laid out as a tree of the language where it reads with no error | none |

`indent` comes before `width`, and `width` before `skip`. Whether there is a `width` decides which of two
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
  It is the kind the grammar's token table gives that name, so one the lexer
  spells many ways is found however it is spelled: `"interp_start"` for
  `"interp_start" = InterpStart _`, whatever piece of a string it is.
- A **child** is written by its label: `value`, `body`. A child the grammar did
  not label goes by the name its accessor has: `Paren` holds an unlabelled
  `Expr`, so its rule says `expr`.

**Layout**, between and around them:

| written     | meaning                                                                   |
| ----------- | ------------------------------------------------------------------------- |
| `indent(…)` | what it encloses is one level in from where the node began                |
| `indent 3(…)` | what it encloses is 3 columns in; `indent -1(…)` is one column out      |
| `group(…)`  | what it encloses goes on one line if it fits, and breaks together if not  |
| `group 60(…)` | the same, but on one line only while that line is 60 columns at most, however wide the width |
| `align(…)`  | a line it breaks to is set in to the column it started at                 |
| `flat(…)`   | on one line whatever the width, every group in it too                     |
| `tiers(…)(head tail)` | a chain of operators broken at its loosest first, by a table of tiers |
| `entries own K N(…)(…)` | a macro's body of token trees, an entry a line |
| `line`      | a space, or a line break where its group breaks                           |
| `cut`       | nothing, or a line break where its group breaks                           |
| `newline`   | a line break, always; its group and every group around it break           |
| `tight`     | nothing: the two things touch                                             |
| `space`     | a space, always: one that does not break when its group does              |
| `ifbroke(a, b)` | after a group: parting `a` where that group broke, `b` where it did not |
| `when K(…)(…)` | a child laid out by the first where it is a `K`, and by the second where it is not |
| `when 1 xs(…)(…)`, `when 7+ xs(…)(…)` | the first where `xs` is exactly one node and no separators, or seven nodes or more; the second where not |
| `when + "," ";" x(…)(…)` | the first where child `x` holds one of those separators, at any depth but inside a bracket within it; the second where not |
| `chained x` | child `x` is no group of its own: its rule breaks with this group |
| `broken x`  | child `x`'s rule breaks, however short it is |
| `wide x`    | child `x`'s `group N(…)`s are plain groups: on one line as far as the width |
| `filled N xs` | where each node of `xs` is N tokens at most with no comment and no bracket, a `line` in each node's own rule is a `fill`; `as K` counts a node of kind K one token |
| `hug K…(xs)` | the last node of `xs`, where it is a `K`, may start on the line of what is before it and break inside itself before `xs` breaks |
| `fill N`    | between a separated child's nodes: a space, or a break before a node that does not fit whole; only where each node is N tokens at most (0 for any), with no bracket, and there are no comments, and `line` otherwise; `as K` counts a node of kind K one token |
| `kept`      | `line` where the text had blank space between the two things, `tight` where it had none |
| `aswritten` | as the text had it: `newline` where it had a break (a blank line where it had one), a space where it had blank space on one line, nothing where the two touched |
| (nothing)   | as the text had it: nothing where the two things touched, a space where they did not |

A rule as a whole is a group. When several separators come in a row the
strongest wins, in the order `newline`, `line`, `cut`, `tight`. A separator
that a rule begins or ends with belongs to the parent: it separates that node
from whatever is next to it, and it counts in the run of separators there. A
child whose rule begins with one gets no extra space before it, and one whose
rule ends with one gets none after it. The same goes for a separator inside
an `indent(…)` at the start or end of a rule: it stays inside, at that
indentation, but a separator beside it adds only what it asks for beyond
that. A line break there and a line break beside it make one line break.

`indent`, `group` and `align` are layout only when followed by parentheses,
or (`indent` and `group`) by a number and parentheses. Anywhere else they are
a child's name, so a grammar may label a child `group`.

**`group 60(…)`** is for what reads as one line only while it is short, such
as a `match` with its arms or a `let` with its body. It measures itself: what
it encloses, flat, from where it starts. So a rule that is short as a whole
writes its body as `group 60(…)`.

**`align(…)`** sets what it encloses in from the column it starts at, not
from where the node began: an `if` whose branches go under its condition
wherever the `if` is.

```meadow
| If = "if" align(cond line "then" yes line "else" no)
```

```text
fun z -> if x < 3
            then (x)
            else y
```

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

**Separators.** A child that is many, with tokens between its nodes, such as
`items:(Expr ("," Expr)* ","?)`, is written with what goes between two of them
in parentheses after its name:

```meadow
| List = "[" indent(cut items(tight "," line)) cut "]"
```

Each token in the parentheses is the separator the tree has there. The
partings say how it stands: here a comma touches the item before it, and a
space or a break comes after it. A separator after the last node, a trailing
comma, is kept as it was: written with what of the parentheses comes before
it, and none of what comes after. So `[a, b,]` stays `[a, b,]`, and broken it
is:

```text
[
  aaaaaaaaaa,
  bbbbbbbbbb,
]
```

Separators are never added or dropped. A blank line between two nodes is kept,
as it is for any child that is many. With no tokens in the parentheses,
`arms(newline)`, it is a child that is many with the partings between its
nodes.

**`breaks N child`** makes the group it is written in break when `child` is
long enough, however much room there is:

- For a child that is many, when it has N nodes or more, or a blank line
  between two. `| Match = "match" scrutinee "with" indent(arms) breaks 3 arms`
  puts an arm on each line once there are three.
- For a child of the rule's own kind, such as a `let`'s body that is another
  `let`, N counts along the chain: `let a = x in let b = y in b` is two.
  Every `let` in the chain breaks, the inner ones too. The chain is followed
  through nodes that hold nothing but one node, as a grammar with a node for
  each level of precedence has between a `let` and its body.

```meadow
| LetE = "let" name "=" value "in" line body breaks 2 body
```

```text
let a = x in
let b = y in
b
```

A group that must break makes every group around it break too. `breaks`
writes nothing; it is written in the rule only to say which group it breaks.

**`kept`** is for what a language spaces as its writer chose, such as an
operator that means one thing spaced and another touching: `a+b`, `f -1`,
`xs.len`. With `| Tail = kept op kept operand`, `a+b` stays `a+b`, `a + b`
stays `a + b`, and `a  +` followed by a line break becomes `a + b`. Two things
that touch are never parted, and two apart are never joined. A separator a
child's rule begins with stands with its parent's, so the parent needs a rule
too (`| Expr = first rest`); a node with no rule keeps the space the text had
before each child.

**`aswritten`** is for what the writer lays out and the formatter should
not: an attribute on a line of its own before what it is on, or beside it.
`| Decl = attrs(aswritten) aswritten item` keeps

```text
@derive(Debug, Display)
@pub data Maybe a = Nothing | Just a
```

on two lines and `@pub fun f x = x` on one. The space it keeps is a space,
not a place to break, so a break elsewhere in the group does not move it.

`kept` and `aswritten` part the last thing written from the next. As a
rule's first element they part the node from what comes before it, as in
`Tail = kept op kept operand`. After children the node does not have, such
as a declaration's attributes when it has none, they part nothing.

**`ifbroke(a, b)`** chooses the parting after a group by whether the group
broke. `a` and `b` are each one of `tight`, `cut`, `space`, `line` and
`newline`. A `let` whose body goes on its `in`'s line when the binding broke,
and on a line of its own when the binding did not but the whole `let` does not
fit:

```meadow
| LetE = group 60(group("let" name "=" indent(line value) line "in") ifbroke(space, line) body)
```

```text
let a =                                     let a = x in
  [bbbbbbbbbb, cccccccccc, dddddddddd]      [bbbbbbbbbb, cccccccccc, dddddddddd]
in a
```

The group is tried on one line, followed by `b`. If that does not fit, it is
laid out broken, followed by `a`.

**`when K(…)(…)`** lays a child out one way when it is a `K` (through nodes
that hold nothing but one node, as for `breaks`) and another way when it is
not. Several kinds may be named, `when IfE MatchE(…)(…)`, and both layouts
name the same child. An `else if` that goes on with the chain:

```meadow
| IfE = "if" cond "then" group(indent(line yes)) line "else" when IfE(no)(group(indent(line no)))
```

```text
if aaaaaaaa then xxxxxxxx
else if bbbbbbbb then yyyyyyyy
else zzzzzzzz
```

**`when` by count.** `when 1 elems(a)(b)` takes `a` where `elems` is one
node and no separator, and `b` otherwise. Parentheses around one thing are
then not a tuple:

```meadow
| Paren = when 1 elems("(" elems ")")("(" indent(cut elems(tight "," line)) cut ")")
```

`when 7+ items(a)(b)` takes `a` where `items` has seven nodes or more. The
child counted may be any the rule names. The two layouts read the node's
children alike: a child separated in either is counted with its separators.

**`chained x`** writes child `x` as no group of its own. Neither are the nodes
between it and what it holds, where they hold nothing but one node. Its rule's
partings break with this group, as one more link of a chain. With it, an
`else if` chain breaks every `else` once one breaks:

```meadow
| IfE = "if" cond "then" group(indent(line yes)) line "else" when IfE(chained no)(group(indent(line no)))
```

```text
if aaaaaaaa then xxxxxxxx
else if bbbb then yyyy
else zzzz
```

**`fill N`** runs a separated child's nodes on, as many to a line as fit,
where every node is N tokens at most, none has a comment, and none holds a
bracket (`(`, `[` or `{`):

```meadow
| List = when 7+ items("[" indent(cut items(tight "," fill 1)) cut "]")("[" indent(cut items(tight "," line)) cut "]")
```

```text
[
  aaaa, bbbb, cccc, dddd,
  eeee, ffff, gggg, hhhh
]
```

A `ifbroke` that a `when` layout begins with sees the group written before
the `when`.

`when + "," ";" body(a)(b)` asks whether a child holds a separator where it
is not one node of its own: a list whose commas are in its items' nodes, as
`CommaItem = "," expr` has them. It looks at any depth, but not inside a node
that begins and ends with a token, so the comma of a tuple inside the bracket
does not make the bracket a list.

**`broken x`** breaks child `x`'s own group, however short it is, and the
groups of the nodes between it and what it holds where they hold nothing but
one node. It is for a node laid out one way in one place and another way in
another. A record's block is a field a line in a declaration, and stays on one
line where it is a type inline:

```meadow
| RecordDecl = "record" name params "=" broken fields
```

```text
record Person = {
  name : String
}
```

**`wide x`** lets child `x`'s `group N(…)`s, and those of the nodes between it
and what it holds, go on one line as far as the width, as plain groups do. A
`let` that reads as one line only while it is short elsewhere may be as long
as the line where it is alone in parentheses:

```meadow
| LetE = group 60("let" binds line "in" body)
| Paren = "(" tight wide elems tight ")"
```

**`filled N xs`** is `fill` for a child whose separators are in its nodes'
own rules, as `CommaItem = "," expr` has its comma:

```meadow
| VecBody = first filled 2 tail
| VecItem = nogroup tight "," line expr
```

Where every node of `tail` is N tokens at most, none has a comment and none
holds a bracket, the `line` in each node's rule is a `fill`, so the nodes run
on, as many to a line as fit. Each node is laid out on one line, and goes on
the next line whole where it does not fit: `show level` is not broken after
`show`. Otherwise the rules are as written.

**`as`** after either counts a node of the kinds it names as one token,
however many it holds: `fill 3 as InterpE`, or `filled 4 items as (InterpE
CharE)`. A string with holes is one token to a lexer that reads it whole, and
many to one that splits it at its holes.

**A parting before what a node lacks.** A parting with nothing after it in the
node, where the rule names something after it that the node does not have,
parts nothing. `SemiItem = nogroup tight ";" line expr` writes `[2;]`, not
`[2; ]`, where the expression is missing. A parting that is the rule's own
last element still parts the node from what follows it.

**`hug K…(xs)`** lets the last node of a child that is many hug what comes
before it, as an application's last argument does when it is a lambda or a
bracket. Where everything fits it goes on one line. Otherwise, where the last
node may hug, everything before it goes on one line and the node is laid out
from the end of that line, set in from that line. It hugs where that goes on
one line and the first line of the node fits after it; a later line of the
node too long for any width does not stop it. Where it does not hug, the
rule's own layout breaks as usual.

```meadow
| Call = "call" head indent(line hug LamE List Paren.LamE Paren+(args(line)))
```

```text
call f aaaa (\x ->              call f a [
  bbbbbbbbbbbbbbbbbbbb)           bbbbbbbbbb,
                                  cccccccccc
                                ]
```

Each kind is read through nodes that hold nothing but one node, so an
argument that is a node around a node is read as what it holds, and one with
more after its first piece (`xs.f(…)`) hugs nothing. A kind is written:

| written  | hugs where it is                                           |
| -------- | ---------------------------------------------------------- |
| `K`      | a `K`                                                      |
| `K.L`    | a `K` holding one node that is, through such nodes, an `L`: parentheses around a lambda |
| `K+`     | a `K` with separators among what it holds: a tuple         |
| `K+ "," ";"` | a `K` holding one of those separators at any depth, but not inside a node that begins and ends with a token (a bracket within it): a list whose commas are in nodes of their own |

A node that holds no node, such as an empty bracket, never hugs. Since it
cannot break, hugging it would lay it out the same anyway.

**Nothing between two things.** Where a rule writes nothing between two
things, they are parted as the text had them: touching where they touched,
and a space where they did not. In a language where touching means
something, such as `0XFF` read as two tokens, or `x'`, formatting keeps what
it meant. A parting the rule does write (`line`, `cut`, `tight`, `newline`)
is the rule's to say, so a bracket still breaks after its `[`.

**`flat(…)`** lays out what it encloses on one line whatever the width, every
group in it too, as nothing inside a string should break:

```meadow
| InterpE = "interp_start" flat(first mids) "interp_end"
```

A break it must have, after a comment, it keeps. A `group N(…)` inside it, or
a group that `breaks` says must break, is on one line all the same: a hole
longer than `N` that holds a `match` is still one line.

**`tiers(…)(head tail)`** lays out a chain of operators that the grammar keeps
flat, `Ops = first:Operand rest:InfixTail*` with each `InfixTail` an operator
and its operand, by a fixed table of tiers rather than the program's own
fixities:

```meadow
| Ops = tiers("|"; "=" ":=" "<-" trailing; "->" "=>" trailing; "or" "||"; "and" "&&";
              "==" "!=" "<" ">" "<=" ">="; other; "+" "-"; "*" "/" "%")(first rest)
```

The tiers are written loosest first and parted by `;`, each a list of
operators. `trailing` puts a tier's break after its operators instead of
before them. `sep` makes a tier's operators separators: the break goes after
them, what follows is no further in than what is before (but a level in, in
one entry of a macro body that has several, as what goes on with the entry),
and they split the chain however they are spaced, so `a, b` and `a,b` both split at the comma.
`sep N M` is a `sep` whose items run on, as many to a line as fit, where
there are `N` items or more and each is `M` tokens at most with no bracket in
it (any number of tokens, where `M` is not given). Otherwise it is a plain
`sep`. `"," sep 7 3` is how `meadow fmt` lays out a brace's list: more than
six short items run on, and otherwise each is a line of its own.
`other` is the tier of any operator the table does not name (the last tier,
where no tier says `other`). The chain is split at the
loosest tier present, each part split the same way at the next. Each split
is a group: the first part, then, a level in, each operator and the part
after it.

```text
aaaaaaaaaa + bbbbbbbbbb
  == cccccccccc * dddddddddd
```

Only an operator with blank space on both sides takes part. `a+b` stays
glued into its operand and is never broken at. `tiers 4(…)(…)` sets what
breaks four columns in rather than a level. A chain with a comment in it is
laid out as the rule's children would be without `tiers`, so a comment is
never moved or lost.

**`entries own K N(TABLE)(ELSE)`** lays out a macro's body, which is token
trees (`BraceGroup = "{" trees:Tt* "}"`) rather than nodes of the language,
an entry a line:

```meadow
| BraceGroup = "{" entries own MacCall 4(…the tiers…)(indent(cut trees(tight "," line)) cut) "}"
```

- **Where an entry starts:** at the first tree, then at a tree that begins a
  line and is `|`, or is no further in than the entry before it and none of
  the table's operators. A line that starts with an operator goes on with
  the entry before it.
- **When:** where there is more than one entry, and the body is the own
  bracket of a `K` (`own MacCall`, a call's), or has no `,` at its top, or
  an entry that starts with `|`. Otherwise it is laid out as `ELSE` says,
  which names the same child.
- **Each entry a line,** a level in from the bracket, a blank line between
  two kept where the text had one, and the bracket closing on a line of its
  own. A `|` entry after one that is not is a level further in.
- **Within an entry,** the trees are split at its spaced operators as `tiers`
  splits a chain, and are otherwise as the text had them. A `sep` tier, such
  as `"," ";" sep`, splits an entry's list at its commas.
- **`chain(TABLE)(trees)`** lays out a child of token trees as one entry,
  split by the table, as a bracket's `ELSE` layout may: `indent(cut
  chain("," sep)(trees)) cut` breaks a list in a body an item a line. Where
  the loosest operators in the trees are a `sep` tier's, that tier breaks
  with what holds the chain: when the bracket breaks, each item goes on a
  line of its own, even if they would all fit on one, unless a `sep N M`
  tier runs them on. The body of a `|`
  entry is `N` columns in from its `|` (two levels, where `N` is not given).
  An entry with a comment among its trees is written as it was.
- **Things side by side** in an entry that does not read as the language are
  an application: on one line if it fits, and each a line a step in if not,
  with a bracket it ends with on the line before it where that line has room
  for its opening. An entry with a word the language spells out in it
  (`if`, `then`) is as the text spaced it.
- **`chain(TABLE)(a b) through(K L)`** is a chain over the language's own
  nodes: children `a` and `b` and what lies between them, and inside any node
  of kind `K` or `L` -- at any depth -- what that node holds. The operators
  found so are split by the table, and every other node is laid out by its
  own rule. A child that is many is found across the table's operators
  between its nodes. `chain N(…)` sets what follows a part's first operand
  `N` columns in from it, rather than a level, and leaves that operand where
  it is: a signature's type, `chain 4("->" "=>" trailing; "!")(ty implies
  eff)`, as rustfmt sets one. Its loosest tier breaks with what holds the
  chain, so a brace's commas go a line each where the brace breaks:

  ```meadow
  | BraceE =
      "{"
      indent(line chain("," ";" sep; "|"; "=" trailing)(items tail)
        through(BraceItem FieldValue BraceTail UpdFields UpdSet UpdField UpdValue))
      line "}"
  ```

  ```text
  {
    cx
      | bound =
        V.foldl step cx.bound r.binds,
    types = t
  }
  ```
- **As the language, where it reads as the language.** With `reparse "|"
  parseSurfaceArm else parseSurfaceExpr` in the header, an entry is first
  read by one of those: an entry starting with `|` by the first, any other by
  the second. Where it reads whole with no error, it is laid out by the
  formatter's rules as that tree, as the same text anywhere else would be:
  an application's arguments, a hug, a `let`, all as the language has them.
  Where it does not, such as a field of a `lang!` or a rule of a `syntax!`,
  it is split by its tokens as above. A `|` entry that does not read as an
  arm, such as a `pass!` arm whose pattern says `later`, or a `database!`
  query's `| query name (x : T) : U =`, may still have a body that reads as
  the language: what follows its `->`, or its first `=` where it has no
  `->`. That body is laid out by the rules, `N` in from the `|`, as are the
  lines of what is before it after the first. After `->`, the body goes on the
  line the pattern ends where it fits there (`} -> 1`). After `=`, it goes on
  a line of its own wherever what is before it broke, as a definition's body
  does. The readers are what `syntax!` writes
  for the rules a `starts { Arm, Expr }` section names: `parse<Lang><Rule>`,
  the text read as that rule, with an error where the rule ends before the
  text does.

**`skip "@fmt(skip)"`** in the header keeps a node exactly as it was written,
comments and all, when one of its children reads as that text, blank space
aside. In a grammar where attributes are a declaration's children, such as
`Decl = attrs:Attr* item:Item`, that is the declaration the attribute is on,
whether the attribute shares its line or stands on a line of its own. The
node's lines stay as they were, wherever it is put. When indenting only, a
line that starts inside it is left alone.

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
- `` `x` is one thing, not many ``: separators after a child that is not many.
- `` `breaks` counts `x`, which this rule does not write ``, or ``x is one T``:
  `breaks` counts a child the rule names, which is many or leads to the rule's
  own kind.

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

In this mode `line`, `cut`, `newline` and `tight` say nothing, and a `group`,
a `group 60(…)` and an `align(…)` are just what they enclose. Only `indent(…)` matters, and `indent 3(…)`, which sets
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

- **Nothing but blank space changes, or nothing does.** `format<Name>` checks
  what it laid out against the text, blank space aside. Where they differ,
  which would be a fault in the formatter, it gives the text back as it was:
  better unformatted than changed. `Lingua.Format.layoutUnchecked` is the
  same layout unchecked, for a test that looks for such faults.
- **Every token survives, in order.** The rules are matched against the
  lossless tree's children as they are met. A child a rule did not name,
  something the parser could not place, is written where it was.
- **Broken programs are formatted as far as they read.** A rule whose node is
  missing a child lays out what is there. `let x =\n(1 +\nin x` still gets its
  value indented.
- **Comments keep a line of their own** before what they were written before.
  A comment inside a node breaks whatever group holds it. A comment written at
  the end of a line, after the token before it, stays at the end of that
  line, with a line break after it, whatever separator the rule puts before
  what follows. A blank line before or after a comment is kept, one at most.
- **Skipped text kept as runs.** A grammar whose lexer skips blank space and
  comments (`@skip`), rather than declaring them `trivia`, has them in the
  tree as runs: a comment and the line breaks around it in one piece. Such a
  run's breaks are kept as if they were pieces of their own, so a comment
  never takes in the line after it. Two comments in one run, or a comment
  that runs over several lines, are written a line at a time: each line after
  the first as far in from the first as it was in the text, counted from where
  the first is now. So a run of comments under an `align(…)` lines up at its
  column.
- **An `indent(…)` with nothing in it** writes nothing, including the
  separators inside it. So `indent(newline entries)` with no entries adds no
  line break.
- **Text the lexer skipped**, such as blank space before the first token that
  a layout pass doesn't hand over, is trivia. The parser keeps it as
  `Unknown`.
- **It is idempotent.** Formatting twice gives what formatting once did.
- **A trailing newline is kept** if the text had one. A line that would be
  nothing but its indent is left empty, and no line ends with blank space,
  unless a token of the text has blank space before a line's end in it (a
  string over several lines), which is left as it was.
- When laying out, a node with no rule keeps one blank line between two
  things; a longer run of blank lines becomes one. A node with a rule keeps
  one in the same way between two items of a child that is many, such as a
  file's declarations. A line break a child's rule asks for is not added on
  top of one the text already has.
- **A node with no rule keeps how far in each of its lines was.** A line the
  text breaks to, and no rule asked for, continues the line before. It is set
  in by as much as it was further in than the line the node began on, and by
  at least one level, on top of whatever sets in the node. The root is the
  exception: a line at its column is one of its items, so a file with no rule
  keeps its lines where they were. In a layout
  language a line at the node's own column would be a new item, never a
  continuation. A line a rule broke to (a layout `;`, say, or a child whose
  rule begins with a break) is set in as the rules say. That holds after a
  comment on a line of its own, too.
- **Blank space before anything is written** is dropped. So a text whose
  first line is indented is set out whole, on the first run.

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
| `tok`, `child`, `indented`, `indentedBy n`, `grouped`, `groupedWithin n`, `aligned`, `separated kinds between`, `breaksAt n leaf` | the pieces of a rule |
| `nogroup`                | a rule's first element, for one that is no group |
| `line`, `cut`, `newline`, `tight`, `space`, `kept`, `aswritten` | the separators |
| `ifBroke broken flat`, `whenKind kinds yes no`, `whenCount n orMore leaf yes no` | a parting by whether a group broke; a layout by a child's kind or count |
| `chained e`, `broken e`, `wide e`, `fillUpTo n`, `hugging kinds e` | a child that is no group of its own; one that always breaks; running on; a last node that hugs |
| `layout st g`            | what `format<Name>` calls when there is a width |
| `layoutUnchecked st g`   | the same, not checked against the text          |
| `reindent st g`          | what it calls when there is none               |
| `continuation st g`      | what `indent<Name>` calls                      |
| `within st width`        | the formatter at another width                 |
| `skipping st texts`      | the formatter, keeping nodes with such a child as they were |
| `formatWithin st width g` | what `format<Name>Within` calls               |
