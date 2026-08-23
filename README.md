# greenlet for z/OS

[greenlet](https://github.com/python-greenlet/greenlet) 3.5.5 ported to z/OS.

> [!WARNING]
> **This port is incomplete and is not safe for production.** Core switching
> works, but some sequences kill the process outright — SIGKILL, no traceback,
> no dump. Five of greenlet's own test modules die this way, including its main
> suite. See [What does not work](#what-does-not-work). The CI does **not**
> publish a wheel.

## What works

Verified on Python 3.12, 3.13 and 3.14:

* greenlets run and return values; `dead` is set correctly
* control alternates between parent and child in the right order
* values cross a switch in both directions
* a 4096-element local survives being saved to the heap and restored
* exceptions propagate out of a greenlet
* `throw()` raises inside a suspended greenlet; a bare `throw()` kills it with `GreenletExit`
* greenlets nest
* 500 consecutive switches are clean
* producer/consumer patterns work

From greenlet's own suite, these modules pass in full: `test_contextvars` (9),
`test_gc` (4), `test_generator` (1), `test_generator_nested` (5),
`test_interpreter_shutdown` (23), `test_leaks` (9), `test_stack_saved` (1),
`test_weakref` (3).

## What does not work

Some sequences kill the process. The smallest reproducer found:

```python
g.switch()
unittest.TestCase().assertEqual(r, "ok")   # any deep-ish Python call
g.throw(RuntimeError)                      # process dies here
```

Replace the middle line with a bare `assert`, or with a shallow function call,
and it passes — so it is the *depth of the intervening call*, not the throw.
Recursing 160 frames deep after a throw is fine; 500 consecutive switch/throw
cycles are fine. Only the combination fails.

Modules that die: `test_greenlet` (the main suite), `test_throw`,
`test_tracing`, `test_greenlet_trash` (all SIGKILL) and
`test_extension_interface` (SIGSEGV, inside greenlet's C API test module).

Two further modules fail for reasons that are **not** greenlet's fault:
`test_cpp` decodes subprocess output as UTF-8 when on z/OS it is EBCDIC, and
`test_version` looks for source files relative to a layout this build does not
have.

Anything built on greenlet — gevent, SQLAlchemy's async support — should be
assumed affected.

## What the port changes

Three changes, in `patches/greenlet-3.5.5-zos.patch`.

### 1. A z/OS stack-switch backend

greenlet selects a `slp_switch()` implementation per platform. z/OS defines
`__s390x__` but not `__linux__`, so it previously selected nothing at all and
failed on `"greenlet needs to be ported to this platform"`.

Widening the Linux check would have been wrong. On z/OS the linkage is XPLINK,
which disagrees with the Linux ELF ABI about everything this code depends on:

| | Linux/s390x | z/OS XPLINK |
| --- | --- | --- |
| stack pointer | r15 | **r4** |
| return address | r14 | **r7**, returns via `b 2(7)` |
| callee-saved | r6–r13 | **r6–r15** |
| frame addressing | positive from SP | SP points **2048 bytes below** the frame |

Measured directly, with a local at `0x50082FF9F0`: `r4` was `0x50082FF060` and
`r15` was `0x1A70C010`. The Linux file would have captured and adjusted a value
unrelated to the stack.

The assembler behind inline asm is **HLASM**, not gas. An instruction must not
start in column 1 or it is parsed as a label — which fails as
`expected identifier` on the first instruction and `symbol 'LGR' is already
defined` on the second — and register operands are bare numbers, so `%0`
expands to `4`, not `%r4`.

### 2. The stack-pointer guard

This is the part that took longest to find and the reason the port exists.

On XPLINK **a callee saves its incoming registers into its caller's frame**,
1808 bytes *above* the caller's stack pointer. (Per LLVM's
`XPLINKSpillOffsetTable`, saved r7 sits at `own_R4 + 0x818`, and
`parent_R4 = current_R4 + dsa_size`.)

greenlet assumes the opposite — that everything above the stack pointer is the
greenlet's saved state and everything below is scratch. So
`slp_restore_state()`, the very function called to copy the incoming stack
back, parks its own return address *inside the bytes it is about to
overwrite*.

The symptom is not a crash at the memcpy. It is the callee returning 0 and the
caller reading garbage:

```
save_state(stackref=50082FDF00) -> 0     <- callee returned 0
macro saw 7104 -> returning -1           <- caller read 7104
```

`STACK_MAGIC` cannot separate them; the window is closed from both sides:

| STACK_MAGIC | bytes | result |
| --- | --- | --- |
| 0, 256 | 0, 2048 | trampoline's saved state overwritten by the restore |
| 288+ | 2304+ | slp_switch's *own* saved r7 excluded from the restore → **0C1 operation exception** |

2304 is exactly the 2048 bias plus slp_switch's 256-byte DSA — the two frames
meet with no gap.

The fix is to drop the stack pointer clear of the restore region across that
one call:

```c
__asm__ volatile (" agr 4,%0" : : "r" (stsizediff));   /* switch */
__asm__ volatile (" aghi 4,-4096");                    /* clear the region */
SLP_RESTORE_STATE();
__asm__ volatile (" aghi 4,4096");
```

The generated code was inspected to confirm no r4-relative access appears
between the two adjustments, so the guard is sound rather than lucky.

### 3. A pthread key instead of thread-local storage

Unrelated to stacks, and worth knowing for any C++ port: **this compiler
supports no thread-local storage at all.**

```
error: thread-local storage is not supported for the current target
```

`thread_local`, `_Thread_local` and `__thread` are all rejected. greenlet has
exactly one thread-local — `g_thread_state_global` — reached everywhere through
`GET_THREAD_STATE()`, which makes it a clean seam. The patch expresses it with
a `pthread_key_create` destructor, preserving the semantics that matter:
created on first use in each thread, destroyed when that thread exits.

## Notes for whoever finishes this

Things ruled out for the remaining crash, with evidence:

* **Not the leak-check harness.** `GREENLET_SKIP_LEAKCHECKS=1` changes nothing.
* **Not repetition.** 500 consecutive switch+throw cycles pass.
* **Not stack depth alone.** 200 frames of recursion before the scenario, or
  160 frames after the throw, both pass.
* **Not LE stack segmentation.** LE stacks *are* segmented — the floor moves
  from `0x5008200000` to `0x5008100000` as the stack grows, and
  `STACK64(64M,0M,64M)` stops that — but the floor was measured from inside
  Python at the point of failure and had not moved, with ~1 MB of headroom.
  Setting `STACK64` does not fix the crash.
* **Not the clobber list.** Removing r6/r7 gives byte-identical behaviour.
* **Not the guard.** The generated code around it is clean.

The failure is a SIGKILL with no CEEDUMP, which means it is not an LE abend.

## Building

```sh
zopen build --build stable -v
```

The build needs a current `check_python` — one that exports `ZOPEN_PYTHON_<major>_<minor>`
per interpreter. An older one reports
`Declared Python interpreter(s) not found: 3.12 3.13 3.14`, and the paths can
be supplied by hand as a workaround.
