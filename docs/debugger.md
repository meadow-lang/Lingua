# The debugger: `dap!`

`dap!` declares a debugger: a debug adapter an editor speaks the Debug Adapter
Protocol to. A debugger is two things. One is a machine that runs a program a
little at a time and can say where it is and what it holds; that is the
language's, since only the language knows how its programs run. The other is
the protocol, the request queue and the console, which are the same for every
language; those are Lingua's.

```meadow
dap! {
  pub Debugger "miniml"
    | launch = launched
    | resume = resumed
    | run = ran
    | breakpoints = placed
    | frames = framesOf
    | scopes = scopesOf
    | variables = variablesOf
    | evaluate = evaluated
}
```

This writes `debugDebugger ()`, the adapter on standard input and output until
the editor disconnects, and `machineDebugger`, the machine as a
`Lingua.Dap.Machine`. MiniML gives it a command, so `miniml debug` is what an
editor runs ([Lib.mw](../examples/MiniML/src/Lib.mw)).

## The machine

The machine is a value `m`, and each feature is a function of it. Nothing is
mutated: a function that changes the machine answers the new one.

| feature       | type                                         | does                                                      |
| ------------- | -------------------------------------------- | --------------------------------------------------------- |
| `launch`      | `String -> Json -> Result String m`          | the program's path, and the launch's arguments as they came: a machine at its start, or why there is none |
| `resume`      | `m -> Mode -> m`                             | how it is to run until it next stops                      |
| `run`         | `m -> Int -> (m, Maybe Stop)`                | run at most that many steps: why it stopped, if it did    |
| `breakpoints` | `m -> String -> [Int] -> (m, [Maybe Int])`   | a file's breakpoints, by line from 1: the line each landed on, or `None` where there is nothing to stop at |
| `frames`      | `m -> [Frame]`                               | where it is, innermost first                              |
| `scopes`      | `m -> Int -> [Scope]`                        | a frame's scopes, by the frame's place in `frames`        |
| `variables`   | `m -> Int -> [Variable]`                     | what a scope or an openable value holds, by its number    |
| `evaluate`    | `m -> Int -> String -> Maybe Variable`       | an expression in a frame: hover and the debug console     |
| `output`      | `m -> (m, String)`                           | what the program wrote since it was last asked            |

`evaluate` and `output` may be left out; the rest are required.

The types are in `Lingua.Dap`:

| type       | is                                                                       |
| ---------- | ------------------------------------------------------------------------ |
| `Mode`     | `Continue`, `StepIn`, `StepOver`, `StepOut`                              |
| `Stop`     | `Entry`, `Breakpoint`, `Step`, `Pause`, `Exception why` (stopped, can be looked at, cannot run on), `Exited value`, `Failed why` (ended) |
| `Frame`    | `{ name, path, line, column }`: lines and columns from 1, `path` empty for none |
| `Scope`    | `{ name, reference, expensive }`                                         |
| `Variable` | `{ name, value, kind, reference }`: `kind` is the type, or empty; `reference` is 0 for a value that cannot be opened |

**How the machine runs.** The adapter calls `resume` with the mode a request
asked for (`continue`, `next`, `stepIn`, `stepOut`), then `run` again and
again. Each `run` is a slice of 20,000 steps, and between slices the adapter
answers what has arrived. So `run` should stop where `resume`'s mode says to:
at a breakpoint, after a step, or at the program's end. A `run` that answers
`None` is still running.

**Output.** The program's output cannot go to standard output, because that
is the protocol. A machine keeps what its program writes, and the adapter asks
`output` after each slice and sends it to the editor's console.

## What the adapter does

Lingua answers:

- `initialize`, `launch`, `configurationDone`, `disconnect`, `terminate`;
- `setBreakpoints`, which may come before `launch` (they are kept and given to
  the machine once there is one), and `setExceptionBreakpoints`;
- `threads` (one thread, `main`), `stackTrace`, `scopes`, `variables`,
  `evaluate`;
- `continue`, `next`, `stepIn`, `stepOut`, `pause`.

It sends `initialized`, `stopped` (`entry`, `breakpoint`, `step`, `pause`,
`exception`), `output`, `exited` and `terminated`. It refuses, with a reason,
a request it does not answer, a run of a program that has failed or finished,
and a launch with no `program`.

`stopOnEntry` in the launch stops before the first step. The editor's
`linesStartAt1` and `columnsStartAt1` are honoured: the machine counts from 1
whatever the editor does.

## Running without blocking

While the program runs, the adapter still answers: a pause has to land at
once, and an editor sets breakpoints and asks for threads when it likes. So
one thread reads what the editor sends, and the program runs in slices with
what has arrived handled between them. A pause stops the program at the end
of the slice it arrives in.

## Testing a debugger

`Lingua.Dap.debugged messages slices machine` runs an adapter against the
requests an editor would send, one at a time. After each request, the program
runs until it stops, or for `slices` slices if it does not. It answers what
the adapter wrote, each message as JSON.

```meadow
@test fun aStepIntoACallStopsInsideIt () =
  let path = written fact in
  let said = debugged (V.append (launching path 3) [req 5 "stepIn" [], req 6 "stackTrace" []]) 100 machineDebugger in
  let frames = field (response said 6) ["body", "stackFrames"] in
  assertEq
    (field (item frames 0) ["line"], field (response said 6) ["body", "totalFrames"])
    (Just (J.jInt 2), Just (J.jInt 3))
    "on the first line of the call it stepped into, one frame deeper"
```

## MiniML's machine

[Debug.mw](../examples/MiniML/src/Debug.mw) is MiniML's `Core` run by a CEK
machine: the expression being worked on, the environment it is worked in, and
what waits for its value. A step is one transition.

- **Where it is** is the start of the expression being worked on. A
  breakpoint stops it the first time it enters an expression on that line,
  so a breakpoint in a recursive function stops at each call.
- **Frames** are the function bodies waiting on the continuation: each one's
  name and the place it was called from, innermost first, then `main`.
- **Variables** are a frame's environment, each name once, the nearest.
- **`evaluate`** looks a name up in a frame.
- **Steps:** `stepIn` stops on the next line it enters, at any depth;
  `next`, on the next line in the same function or one it returns to;
  `stepOut`, once the function returns.

`Eval`'s evaluator is the reference, and a test checks the machine comes to
the same value on the same programs.

## Limits

- **One thread.** `threads` answers one, and the machine is one program.
- **No conditional breakpoints, logpoints or data breakpoints;** no
  `setVariable`, `restart`, `goto` or `stepBack`.
- **`setExceptionBreakpoints` is answered and ignored:** a failure always
  stops the program.
- **A pause lands at the end of a slice:** at most 20,000 steps late.
- **Breakpoints are by line.** A machine that knows columns can only use the
  line.
