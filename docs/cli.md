# The command line: `cli!`

`cli!` declares a program's command line: its commands, what each takes, and
what help says of each. It writes a type with a constructor per command, the
function that reads the arguments into it, help, a bash completion script, and
every mistake drawn against the command line.

```meadow
cli! {
  pub Cli "miniml" "MiniML, typed and run by a compiler written with Lingua"
    | check "Type-check each file and say its type" { files : [String] "the files to check" }
    | run "Run a program and say its value and type" { file : String "the program" }
    | repl "Type and run each line of standard input" {}
}

fun main () =
  match argsCli () with
  | Cli.Check r -> check r.files
  | Cli.Run r -> run r.file
  | Cli.Repl -> repl ()
```

For a type `Cli` this writes:

| written            | is                                                               |
| ------------------ | ---------------------------------------------------------------- |
| `data Cli`         | a constructor per command, holding its fields as a record        |
| `argsCli ()`       | the command the process was started with; help and completions are printed and a mistake drawn, and each ends the process (a mistake with status 2) |
| `askCli args`      | the same for any arguments, as a `Result`, ending nothing        |
| `cliSpecCli`       | the command line as data, which `Lingua.Cli` reads one with      |

Every program also answers `help`, `help <command>`, `--help`, `-h` and
`completions bash`.

## Fields

A field is `name : Kind "what help says"`. Its kind decides whether it is an
argument, read by position, or a flag, read by name.

| kind                  | on the line              | the field holds     |
| --------------------- | ------------------------ | ------------------- |
| `String`, `Int`       | an argument that must be there | `String`, `Int` |
| `Maybe T`             | an argument that may be left out | `Maybe T`     |
| `[T]`                 | the rest of the arguments | `[T]`              |
| `Flag`                | `--name`                 | `Bool`              |
| `Option T`            | `--name value` or `--name=value` | `Maybe T`   |
| `Option [T]`          | the same, any number of times | `[T]`          |
| `Option ("a" \| "b")` | one of those texts       | `Maybe String`      |
| `Option T = d`        | `d` where it is not given | `T`                |

After `--`, everything is an argument.

**One of a few values.** `runtime : Option ("glade" | "silo")` refuses anything
else, saying which there are, and help lists them.

**A value where none is given.** `width : Option Int = 100` holds `100` unless
the line says otherwise. The field is an `Int`, not a `Maybe`, and help says
`[default: 100]`. The value after `=` is a number or a text in quotes.

**Given more than once.** `cfg : Option [String]` keeps each: `--cfg a --cfg b`
is `["a", "b"]`. Any other option given twice is a mistake, drawn at the
second:

```text
Error: `--opt-level` is given twice
```

A `Flag` given twice is still just there.

## How a flag is written

A flag's name on the line is its field's name with a dash for each capital:
`noPrune` is `--no-prune`, `optLevel` is `--opt-level`. After what help says
of it, a field may say more:

```meadow
optLevel : Option Int "How hard to optimise" short "O" value "LEVEL"
workspace : Flag "Every member of the workspace" alias "all"
clientProcessId : Option Int "The editor's process" as "clientProcessId"
emit : Option [String] "What to write" split ","
```

| said            | means                                                               |
| --------------- | ------------------------------------------------------------------- |
| `short "O"`     | one letter: `-O 2`, `-O2` or `-O=2`                                 |
| `alias "all"`   | another name it answers to, `--all`; a field may have several       |
| `as "x"`        | its name exactly, where a dash for each capital is wrong            |
| `value "LEVEL"` | what help calls the value: `--opt-level <LEVEL>`                    |
| `split ","`     | for an `Option [T]`: each value is parted there, so `--emit a,b` is two |

An argument that starts with `-` and a letter is a flag, and one the command
does not have is a mistake. `-` alone, and `-1`, are arguments.

## Options several commands take

Declare them once, before the commands, and name them where they are taken:

```meadow
cli! {
  Tool "tool" "A compiler's command line"
    options Build {
      release : Flag "Build with the release profile"
      optLevel : Option Int "How hard to optimise" short "O" value "LEVEL"
      runtime : Option ("glade" | "silo") = "glade" "The runtime system"
    }
    | build "Build a package" { path : Maybe String "the package" } with Build
    | run "Build a package and run it" { path : Maybe String "the package" } with Build
}
```

This writes `type ToolBuild = { release : Bool, optLevel : Maybe Int, runtime :
String }`, and each command that says `with Build` holds one in a field named
for it: `Tool.Build r` and `Tool.Run r` both have `r.build : ToolBuild`. What
reads those options is then one function for every command that takes them,
and two commands cannot come to take them differently. A command may take
several: `with Build, Test`.

## Options every command takes

`options { … }` with no name are taken by every command, and may be written
before the command or after it:

```meadow
options {
  offline : Flag "Never fetch a dependency"
  locked : Flag "Refuse to change the lock file"
}
```

```text
tool --offline build
tool build --offline
```

They are not a command's, so they are not in its record. With them,
`argsTool ()` and `askTool args` answer a pair: a `ToolOptions` and the
command.

```meadow
match argsTool () with
| (every, Tool.Build r) -> build every.offline r.path r.build
```

Only flags can be taken by every command, not arguments.

## The program itself

```meadow
version "1.2.0"
default repl
```

`version` gives the program `--version` and `-V`, which print `tool 1.2.0`.

`default` names the command that runs when the line names none: `tool` alone
is `tool repl`, and so is `tool --offline`. A line that starts with a flag is
that command's too. A word that is no command is still a mistake.

Both come before the commands, with `options`, in any order.

## Help

`tool help build` says how each flag is written and what its declaration
says of it:

```text
tool build -- Build a package

Usage: tool build [<path>] [--release] [--opt-level <LEVEL>] [--runtime <runtime>]

Arguments:
  [<path>]   the package

Flags:
  --release                 Build with the release profile
  -O, --opt-level <LEVEL>   How hard to optimise
  --runtime <runtime>       The runtime system [possible values: glade, silo] [default: glade]

Flags every command takes:
  --offline   Never fetch a dependency
  --locked    Refuse to change the lock file
```

## Mistakes in the declaration

Each is reported where it was written:

- a command declared twice, or called `help` or `completions`;
- an argument that can never be reached: after the rest of them, or one that
  is needed after one that may be left out;
- `with X` where no options are called `X`;
- a field of a command and of the options it takes with the same name, or
  two flags written the same;
- `short`, `alias`, `as`, `value`, `split` or `= d` on an argument; `split`
  on an option of one value; `= d` on a `Flag` or an `Option [T]`; a `short`
  of more than one letter;
- `default x` where there is no command `x`.

[Tool.mw](../examples/MiniML/src/Tool.mw) declares a command line with all of
the above and tests each part of it.

## Limits

- **Letters are not written together.** `-ab` is `-a` with the value `b`, not
  two flags.
- **Completions are for bash**, and offer a command's flags and files, not the
  values a flag takes.
- **One of a few values is a `String`** in the field, not a type of its own.
- **Help's layout is Lingua's.** It says what a `clap` program's does, in a
  different arrangement.
- **A command has no commands of its own.**

## By hand

`cli!` writes a `Lingua.Cli.Spec` and calls `ask` on it. Both can be used
directly: `spec`, `specWith`, `command`, `arg`, `argWritten`, `ask`, `usage`,
`commandUsage`, `bashCompletion`, and `field`, `text`, `num`, `flag`, `maybe`,
`many` to read a `Ran` command's values back.
