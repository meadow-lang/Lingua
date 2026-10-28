# Passes: what is automatic, and the newer parts of `pass!`

[DESIGN.md §5](DESIGN.md#5-lang-and-pass) introduces `pass!`, contexts, `later`
and `raw`. This guide covers what a pass does without being told, and five
later additions: saying which sort a case is for, `node`, cases that answer
something other than a node, passes that start partway down a program, and
passes that fold a program into a value.

## What a pass does on its own

A pass is written as the cases that change something. Everything else is
generated, and it behaves much like a recursion scheme: you say what happens at
the productions you care about, and the traversal carries it through the rest
of the structure.

Specifically, for `pass! { lower : Surface -> Core … }`:

- **One function per sort**: `lowerExpr`, `lowerStmt`. Each takes a node of the
  source and answers the target's.
- **Children are translated first.** A field a case names arrives already
  translated, as a row of the target. A case for `Bin { lhs, rhs }` never calls
  the pass on `lhs` itself.
- **Unchanged productions are copied.** Any production the target kept as the
  source has it needs no case. Its node is written again with its children
  translated.
- **Provenance is kept.** A node a case writes gets the `meta` of the node it
  replaces.

So a pass with a case only for a leaf reaches every such leaf however deep it
is:

```meadow
pass! {
  deepen : Shapes -> Depths with depth
    | Expr.Var { name } -> Var { name, depth }
    | Lam { param, later body } -> Lam { param, body = body (depth + 1) }
}
```

`Pat.Var` and `Pat.Wild` have no case and are copied. `Lam` has a case only
because it changes the context for its body; without a context to change, the
`Var` case alone would be enough.

A case for a recursive production is needed when there is additional work at
that node: a context to extend (`later`), a child to inspect before translating
it (`raw`), or a different node to write. It is never needed just to keep the
traversal going.

What is **not** inferred: a production the target dropped or changed must have
a case. Leaving one out is an error at the pass, naming the production.

## Which production a case is for

A case names its production bare, and a node written in a body does too:

```meadow
| Lam { param, body } -> Lam { param, body }
```

When two sorts each have a production of that name, say which with the sort,
as a constructor is said of its type:

```meadow
| Expr.Var { name } -> Var { name, depth }
| Pat.Var { name }  -> Pat.Var { name }
```

The rules:

| written                   | means                                                                      |
| ------------------------- | -------------------------------------------------------------------------- |
| a case `Var { … }`        | the source's only `Var`; an error naming the choices if two sorts have one |
| a case `Pat.Var { … }`    | `Pat`'s                                                                    |
| `Var { … }` in a body     | the `Var` of the sort being written, if it has one; otherwise the only one |
| `Pat.Var { … }` in a body | `Pat`'s                                                                    |

The error for an ambiguous bare name is:

```text
`Shapes` has a `Var` in `Expr` and `Pat`: say which, `Expr.Var` or `Pat.Var`
```

The flat and dotted names from [languages.md](languages.md) are never written
inside a pass.

## What a case has in scope

| name    | what it is                                                                   |
| ------- | ---------------------------------------------------------------------------- |
| fields  | each named field, already translated (or a function, `later`; or untranslated, `raw`) |
| `here`  | the `meta` of the node being rewritten: where a diagnostic about it points   |
| `node`  | the node itself as the source has it                                         |
| `src`   | the source's tables, to read a `raw` row with                                |
| `out`   | the target being written                                                     |
| context | the name after `with`, if the pass has one                                   |

### `node`

`node` is the node the case is for: a `Syntax` when the source is a tree, a row
when it is tables. Use it when a case needs more of the node than its fields.

```meadow
pass! {
  onlyExprs : Calc -> Core from Expr
    | Bin { lhs, op, rhs } -> Prim { op, args = [lhs, rhs] }
    | Literal { number } -> Literal { number = "${number}@${show (R.offset node)}" }
}
```

`here` is still the right thing for a diagnostic; `node` is for everything else
the tree knows, such as a token the language dropped, or the node's parent.

## Cases that do not answer a node

What a case answers is its sort's function's answer. That is usually a node of
the target, but it can be anything every case of the sort agrees on. One
declaration may become several nodes, for the case above to put together:

```meadow
pass! {
  spread : Calc -> Core from File
    | File _ -> []
    | Stmt _ -> []
    | File { stmts } -> V.concat stmts
    | Let { value } -> [value]
    | ExprStmt { expr } -> [expr, expr]
    | Bin { lhs, op, rhs } -> Prim { op, args = [lhs, rhs] }
}
```

Here `spreadStmt` answers a list of expressions and `spreadFile` concatenates
them.

**From a tree, such a sort also needs a fallback.** A lossless tree can hold a
node that is not what its rule says, because something is missing in it. For a
sort that answers nodes, the generated function answers "no node" for those.
For a sort that answers something else, there is no value Lingua could invent,
so the pass must say:

```meadow
| Stmt _ -> []
```

`| Sort _ -> expr` is what `passSort` answers for a malformed node of that
sort. Naming a sort the source does not have is an error at the case.

## A pass over part of a program: `from`

A pass need not be of a whole program. `from` lists the sorts it starts at:

```meadow
pass! {
  lowerTy : Grouped -> Ast from Type, Row
    | Arrow { param, result } -> …
}
```

With a context, `from` follows it: `pass! { p : A -> B with env from Type … }`.

A rooted pass differs from a whole one in three ways:

- **It has a function for each listed sort and each sort reachable from
  them**, and none for the rest. `from Type, Row` gives `lowerTyType`,
  `lowerTyRow` and whatever those hold.
- **Only those sorts are checked.** A production elsewhere in the source that
  the target dropped does not need a case.
- **It has no entry point.** There is no `lowerTy program`. The caller calls
  the sort's function directly.

The functions take the source, the target being written, the context if there
is one, and the node:

```meadow
lowerTyType src out node            -- no context
elaborateExpr src out env node      -- with one
```

When the source is a tree there are no source tables, and `src` is `()`:

```meadow
let p =
  runSt (\() ->
    let b = newCoreBuilder () in
    let r = onlyExprsExpr () b e in      -- `e : Syntax Calc`, an expression
    CoreProgram (freezeCore b) r)
```

**What it is for.** A compiler written by hand is moved onto passes one part at
a time. The hand-written code calls the part that is already a pass, and the
pass grows a group of rules at a step. Without `from`, a pass must cover every
production the target changed before it compiles at all.

**Limits.**

- A sort in `from` must be one the source has.
- A pass that writes a **tree** cannot be rooted. It already has a function for
  every sort, so there is nothing for `from` to choose, and saying it is an
  error.

## A pass that folds: `fold`

Some passes don't write a language. They collect something from a program: the
names it mentions, the constants it uses, whether it calls anything. Written as
an ordinary pass, every production needs a case, so the pass is no shorter than
the `match` it replaces. `fold` gives the answer for nothing and a way to put
two answers together. Then a production with no case answers its children's
answers put together:

```meadow
pass! {
  mentions : Elaborated -> [Int] with depth from Term
    fold [] V.append
    | Opaque { unique } -> [unique]
    | Lam { later body } -> body (depth + 1)
    | Var { index } -> if index >= depth then [index - depth] else []
}
```

- **After the arrow is the answer's type.** It documents what the pass
  answers, and nothing checks it.
- **`fold` takes two parts.** The first is a name, possibly qualified by
  modules (`[]`, `0`, `HashSet.empty`), a literal, or something in brackets.
  Everything after it, up to the first case, is the second part: `V.append`,
  `HashSet.union`, `(\a b -> a or b)`.
- **A production with no case** answers its children's answers combined from
  left to right, in field order, starting from the empty answer:
  `combine (combine empty first) second`. A list field adds each element in
  turn. An optional field adds its answer when it is present. A field that
  isn't a node, such as a name, a number or a token, adds nothing. Because the
  order is left to right, an answer that depends on order, like names in order
  of first appearance, comes out in source order.
- **A case** takes fields the same way any case does: already folded, `later`
  (a function from a context to the child's answer, for a binder), or `raw`.
  It may use `here`, `node` and `src`. There is no `out`, because a fold
  writes no target.
- **From a tree**, a node that doesn't match its rule answers the empty
  answer, unless the pass says otherwise with `| Stmt _ -> …`.

With `from`, a fold has a function for each sort it reaches and no entry point.
Each function takes the source, the context if there is one, and the node:
`mentionsTerm src depth node`. From a tree, `src` is `()`. A whole fold's entry
point is `mentions depth program`, or `mentions depth root` for a tree.

A truth is a fold too:

```meadow
pass! {
  calls : Calc -> Bool
    fold False (\a b -> a or b)
    | Call {} -> True
}
```

The tests in [the Calc example](../examples/Calc/src/Lib.mw) cover both table
and tree languages, a context with `later`, and `from`.

