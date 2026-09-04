# picolibc (source package)

A freestanding C library for mcpp, **compiled with the consuming program's own
flags**.

```toml
[dependencies]
picolibc.picolibc = "1.8.12"
```

That is the whole of it. No `sysroot` line, no payload path, no multilib
directory to name — the library arrives through the dependency graph and mcpp
reports `c-abi picolibc (picolibc@1.8.12, graph)`. The version is
upstream's: every file under `picolibc/` is byte-identical to the 1.8.12
release, and a packaging-only change would be `1.8.12.1`.

## Why source rather than a prebuilt payload

A prebuilt C library ships one build per ABI the target table can name — seven
for Cortex-M alone — and every consumer then finds the right one through a
`libdir` convention that has to match the payload byte for byte.

| prebuilt | this |
|---|---|
| seven multilibs, built separately | none: one compile, with your flags |
| `libdir` must match the payload exactly | no `libdir` at all |
| builtins must ship together or the first `printf` fails to link | a dependency edge, overridable |
| five host mirrors, `.sha256`, CDN propagation | a source tree, host-independent |
| the version pinned in the target table, outside the lock | in `mcpp.lock` |

**And on ARM the prebuilt route is actively wrong.** The multilib key
`<march>/<mabi>` cannot separate the float ABI, because `mabi` there names the
procedure call standard and is `aapcs` either way. Measured while building one:
the seven profiles collapsed into five directories and the soft-float row
received a library carrying `Tag_ABI_HardFP_use`. Nothing failed at build time.

**The headers were never per-profile anyway.** Measured across seven builds:
the whole include tree — including `picolibc.h` and `newlib.h`, which meson
*generates* — is byte-identical. A prebuilt ships seven copies of one directory.

## What is measured

* Builds for all seven M-profile rows.
* A `printf("%.2f")` program built against this package plus
  [`cortex-m-rt`](https://github.com/mcpplibs/cortex-m-rt) boots under QEMU's
  `mps2-an385`, prints `13.00` and exits 0.
* The source list is upstream's, read out of the `build.ninja` meson produced —
  1130 files, identical for every profile.

## What this package does *not* carry

**`picocrt`.** A startup object decides where execution begins, what it
initialises and how it reaches the host — which board is running. picolibc ships
nine variants for that reason, and choosing among them is a board-support
package's job. `cortex-m-rt` supplies its own vector table, `Reset_Handler` and
TLS initialisation.

A board that supplies startup must set the **thread pointer**: picolibc
reaches `stdout` through thread-local storage, and without it a program links
cleanly, runs, prints nothing and hangs. There is no diagnostic for that state.

## Scope

This release serves the **ARM** family. Sources and machine directories for
aarch64, riscv and x86 are vendored, but only the seven M-profile profiles have
been built and run. Adding an architecture is a `cfg` block, its long-double
directory, and a measurement — the last being the part that cannot be skipped.

## Licence

BSD-3-Clause / BSD-2-Clause, as picolibc. See `LICENSE.picolibc`.
