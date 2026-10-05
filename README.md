<!---
{
  "id": "14d07a93-2c67-4dba-ab7b-4ae61fbd884b",
  "teaches": "Debugging: Stepping Through a Loop with GDB",
  "depends_on": [
    "f5d53abd-0e66-4120-9045-4575182d18fe",
    "9264c227-8d04-4a33-bbc7-df86ba0e9a5a"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-05",
  "keywords": [
    "gdb",
    "debugger",
    "breakpoint",
    "next",
    "step",
    "print",
    "-g",
    "-O0",
    "off-by-one",
    "TUI"
  ]
}
--->

# Debugging: Stepping Through a Loop with GDB

> In this exercise you will use GDB for the first time: build with debug symbols and no optimisation, set a breakpoint, step through a loop, print variables and find an off-by-one error that `printf` would have hidden. You will also see what happens when you try the same on an optimised binary. No `sudo` is required.

## Introduction

`DBG` and logging both require you to decide *in advance* which variables to print and where. A debugger removes that requirement: it stops the program at a line you choose and lets you look at *any* variable, call *any* function, and continue. No code changes, no rebuild between questions.

The price is two compiler flags. `-g` stores the mapping from machine code back to source lines and variable names in the binary. `-O0` keeps the compiler from reordering, inlining or removing code, so that "line 13" still means line 13 and `i` still lives somewhere GDB can read. The debug configuration of your `Makefile` sets exactly these.

### 1.1) Further Readings and Other Sources

* [GDB Manual: Sample Session](https://sourceware.org/gdb/current/onlinedocs/gdb/Sample-Session.html)
* [Beej's Quick Guide to GDB](https://beej.us/guide/bggdb/)
* [GDB Manual: TUI mode](https://sourceware.org/gdb/current/onlinedocs/gdb/TUI.html)
* [Greg Law, CppCon 2015: Give me 15 minutes and I'll change your view of GDB](https://www.youtube.com/watch?v=PorfLSr3DDI)

## Tasks

### Task 1: The Program

```bash
mkdir -p ~/dbg-gdb && cd ~/dbg-gdb
```

```c
// file: loop.c
#include <stdio.h>

int sum_to(int n) {
    int s = 0;
    for (int i = 1; i <= n; i++) {
        s += i;
    }
    return s;
}

int main(void) {
    int values[5] = {1, 2, 3, 4, 5};
    int total = 0;
    for (int i = 0; i <= 5; i++) {
        total += values[i];
    }
    printf("total = %d, sum_to(5) = %d\n", total, sum_to(5));
    return 0;
}
```

```bash
gcc -Wall -Wextra -O0 -g loop.c -o loop
./loop
```

The output is `total = 15`, which happens to be right. Do not fix anything yet.

### Task 2: Start GDB, Set a Breakpoint, Run

```bash
gdb -q ./loop
```

```
(gdb) break 15
(gdb) run
(gdb) print i
(gdb) print total
(gdb) print values
```

GDB stops *before* line 15 executes. `print values` shows the whole array because GDB knows its type from the debug info.

### Task 3: Step and Continue

```
(gdb) next
(gdb) next
(gdb) print i
(gdb) continue
(gdb) print i
```

`next` executes one source line. `continue` runs to the next breakpoint, which is the same line in the next iteration. Press Enter on an empty line: GDB repeats the previous command.

### Task 4: Catch the Sixth Iteration

```
(gdb) continue 4
(gdb) print i
(gdb) print values[i]
(gdb) print &values[i]
(gdb) print &values[4]
```

`i` is 5 and `values[5]` is memory past the end of the array. Today it holds 0, tomorrow it holds whatever lies there. Why did `./loop` print the correct result anyway?

### Task 5: Conditional Breakpoints

Quit (`quit`, confirm with `y`), start again, and stop only at the interesting iteration:

```
(gdb) break 15 if i == 5
(gdb) run
(gdb) print i
(gdb) info breakpoints
```

Fix the loop condition to `i < 5` afterwards and rebuild.

### Task 6: Into a Function and Out Again

```
(gdb) break 17
(gdb) run
(gdb) step
(gdb) step
(gdb) step
(gdb) info locals
(gdb) finish
```

`step` enters `sum_to`, `next` would have stepped over it. `info locals` lists all local variables of the current function. `finish` runs until the function returns and prints `Value returned`.

### Task 7: The Source View

```bash
gdb -q -tui ./loop
```

```
(gdb) break 15
(gdb) run
(gdb) next
(gdb) layout next
(gdb) layout src
```

The TUI shows the source with the current line highlighted. `Ctrl-L` redraws if the screen gets garbled. Most people use `layout src` and nothing else.

### Task 8: What Optimisation Does

```bash
gcc -Wall -Wextra -O2 -g loop.c -o loop_O2
gdb -q ./loop_O2
```

```
(gdb) break sum_to
(gdb) run
(gdb) next
(gdb) next
(gdb) print s
(gdb) print i
```

Lines are skipped, variables are `<optimized out>`, the function may have disappeared entirely because the compiler inlined it. Read the warning that `gcc -O2` printed about line 15 in Task 1 as well: the compiler *saw* the bug and told you, if you read warnings.

## Questions

- What does `-g` add to the binary, and why does the program run at the same speed with it?
- Why does GDB need `-O0` to show sensible values? What does `<optimized out>` mean?
- `next` versus `step`: which one would you use to skip a call to `printf`?
- In Task 4 the program read `values[5]`. Why was there no crash?
- `break 15 if i == 5`: how does GDB implement this, and why is it slower than an unconditional breakpoint plus `continue 4`?
- Compare the three tools so far: for the off-by-one in this program, what would `DBG`, a log and GDB each have shown you?

## Advice

Ten commands cover ninety percent of GDB use: `break`, `run`, `next`, `step`, `continue`, `print`, `finish`, `info locals`, `backtrace`, `quit`. Learn them on this toy program until they are reflexes; the next two exercises use the same commands on harder problems.

Keep the debug and release builds in separate directories, as the `make` exercise set up. Debugging an `-O2` binary is a frustrating waste of an afternoon, and it happens to everyone exactly once.
