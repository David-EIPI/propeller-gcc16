# propgcc-16 — a GCC 16.2.0 C compiler for the Parallax Propeller 1 (P8X32A)

This is the Propeller back end from **propgcc / propeller-gcc**
(<https://github.com/dbetz/propeller-gcc>, GCC **6.0.0 20150430 experimental**,
`propellergcc-alpha_v1_9_0`) forwarded to the **GCC 16.2.0** release.

The back end itself is unchanged in intent: same memory models (LMM, CMM, COG,
XMM, XMMC), same ABI, same multilibs, same `__builtin_propeller_*` intrinsics,
same `_COGMEM` / `cogmem` / `native` / `fcache` / `naked` attributes.  What
follows is what is different.

---

## 1. Bugs fixed

Nine defects in the original back end and runtime, or in how it interacts
with current GCC.  Each is a wrong-code or crash bug, not a missed
optimisation.

**1. `CNT`, `INA`, `INB` and `PAR` could be placed in an instruction's d-field.**
The Propeller Manual v1.2 (register table, p.23, note 1) states that these
registers are *"only accessible as a source register"*.  The compiler could
emit, for example, `wrlong CNT, <addr>`, which does not read the live counter.
Measured on hardware, the stored value came back as 0, and code that then
waited on it stalled for a full 2^32-cycle counter wrap.  The operand
predicates and the store patterns now keep these registers out of the d-field.
`PHSA`/`PHSB`, which carry the same restriction for read-modify-write, are
covered too.

**2. 64-bit unsigned division and modulo were wrong at `-O2` and `-Os`.**
A `udivmoddi4` pattern existed whose body was an unconditional `FAIL`.  Naming
an expander registers an optab handler regardless, so the middle end believed
the target had a divmod instruction, fused adjacent `a / b` and `a % b` into a
single internal call, and then used the un-initialised results.  Both the
quotient and the remainder came out as garbage.  The dead pattern is removed.

**3. `float` to `double` conversion produced 29 junk mantissa bits.**
`__extendsfdf2` unpacked the float into one register and then called the double
packer, which expects the mantissa spread across two — the second was never
cleared.  This affected every widening conversion, including the default
promotion when a `float` is passed to `printf`.  Against exact IEEE-754
values, the original toolchain gets 2 of 20 test conversions right; this one
gets 20 of 20.

**4. Combine could reverse two volatile cog-register reads.**
`unsigned t = CNT; return CNT - t;` compiled to `t - CNT`, i.e. the negated
elapsed time.  Two separate volatile reads were folded into one instruction,
where RTL no longer fixes their order.  A `TARGET_LEGITIMATE_COMBINED_INSN`
hook now rejects any combined insn that would perform more than one volatile
read; single-location read-modify-write (`DIRA |= mask`) is still folded.

**5. `__builtin_ctz` did not link.**  `__CTZSI` was defined in the runtime but
never declared `.global`.

**6. Internal compiler error on 64-bit compares in `-mcog`.**
`cmpdi` printed its operands with codes that can only render a register or a
constant, while the predicates also admitted cog memory.

**7. Internal compiler error on 64-bit memory access.**
The post-reload 64-bit move splitter assumed the address was always a plain
register and asserted otherwise.

**8. Cog images were linked without their start code.**
propgcc builds a cog overlay with `gcc -mcog -r`: a relocatable object that is
nevertheless a complete program, copied into a cog and started at cog address
0.  GCC 6's link spec added the start files to any `-r` link; current GCC
guards them with `%{!r:...}`, which is right for an ordinary partial link but
left cog images with no start code - they began with whatever literal pool the
compiler had emitted, and the loader jumped straight into data.
`LINK_COMMAND_SPEC` is now overridden so that the start files, default
libraries and end files are linked when `-r` is combined with `-mcog`.  An
ordinary partial link is unaffected.

**9. Naked functions were left without a return.**
On this target `naked` is used to suppress the frame, not to hand-write the
whole function: cog code often has no stack, so ordinary C functions are marked
naked purely to stop a frame being built.  GCC 6 still gave them a return, via
a path that current GCC no longer takes for naked functions, so they were left
falling through into whatever followed.  A non-native naked function now gets
the ordinary return again.  A *native* naked function does not: its return is a
named `<func>_ret` label, which such functions normally define themselves, and
emitting a second one is a duplicate symbol error.

---

## 2. New instruction support

Seven P8X32A instructions the original back end had no pattern for and could
never emit:

| instruction | now generated for |
|---|---|
| `MOVS`, `MOVD`, `MOVI` | 9-bit field inserts at bits 0-8, 9-17 and 23-31 — via the `insv` optab and the equivalent mask/or idiom.  A field assignment that previously took 4-6 instructions now takes one. |
| `ADDS`, `SUBS` | signed overflow (`__builtin_add_overflow`, `__builtin_sub_overflow`, `-ftrapv`).  An overflow-checked add drops from six instructions plus a frame to `adds` and a branch. |
| `TJZ`, `TJNZ` | test-against-zero and branch, fused into one instruction.  Restricted to code running from cog memory, since the branch target is a 9-bit cog address. |

MOVS/MOVD/MOVI masking behaviour was verified against real hardware before
being relied on.

---

## 3. Optimisations

* **Redundant zero tests removed.**  `ANDN` and the hub loads `RDLONG`,
  `RDWORD`, `RDBYTE` all set Z from their result, so `r = a & ~b; if (r)` and
  `if (*p)` no longer emit a separate compare.  (Most other operations already
  did this.  `MIN`/`MAX`/`MINS`/`MAXS` cannot: they set Z from the source
  operand, not the result.)
* Patterns were also added for `NEG`/`ABS` tested against their source and for
  `XOR` feeding an equality test.  These are correct but fire rarely, because
  the middle end usually sinks the arithmetic past the branch.
* Everything else is the GCC 16 middle end, which is 10 major releases newer
  than the original.

GCC's generic `compare-elim` pass is deliberately **not** enabled: it requires
every flag-setting instruction to clobber the flags, whereas on the Propeller
flags change only when `wz`/`wc` is requested — which is exactly what lets the
back end keep a condition live across arithmetic and use predicated
instructions.

### New warning: `-Wcog-stack`

Code generated for cog memory will build a stack frame if register pressure
demands it.  That is fine for a standalone `-mcog` program, whose start code
sets `sp` up, but an overlay copied straight into a cog usually has no stack,
and the frame then writes through whatever `sp` happens to hold.  `-Wcog-stack`
names any function this happens to:

    driver.c:300:1: warning: 'check_error' needs a 8 byte stack frame, but cog
      code has no stack unless sp is set up for it [-Wcog-stack]

It is off by default, since a standalone cog program legitimately has a stack.
Turn it on, with `-Werror=cog-stack` if you like, when building cog overlays.

### Measured effect

On a set of four large, hand-optimised closed-source Propeller applications
(LMM, with cog and ecog drivers, 21-29 KB of code each):

| | |
|---|---|
| total code size | **-2.2%** |
| best single application | **-4.6%** |
| cog/ecog driver modules | smaller in every case, **-5% to -22%** |

Because Propeller instructions are almost all fixed-time, a size reduction of
this kind is also a speed reduction.  The cog-driver figure matters
disproportionately: an ecog module must fit in 496 longs, and the
largest driver measured went from 477 longs to 453.

---

## 4. Testing

A differential test suite compares this compiler against the original GCC 6
build, running both on the Propeller and comparing a checksum over every
computed value.  17 test modules cover integer and 64-bit arithmetic, logic,
shifts, comparisons and branches, memory access of every width, structs and
pointers, `switch` in dense/sparse/computed forms, calls with up to eight
arguments plus sibcalls, recursion, varargs and struct arguments, soft float
and double, the Propeller intrinsics, locks, multi-cog startup, field inserts,
overflow builtins, and computational kernels taken from production
applications.

| | |
|---|---|
| simulator configurations compared | **150** (2 memory models x 5 optimisation levels) |
| individual computed values compared | **357,190** |
| cog-mode configurations compared | **13** (3 memory models x 5 levels, all that fit in a cog) |
| compile-only combinations | **1,260** (7 memory models x 5 levels x both `double` sizes) |
| modules re-verified on real hardware | **13**, built with production compiler flags |

Results: all 150 simulator configurations agree except three in which the
*original* compiler fails (a pre-existing wrong-code bug of its own in CMM at
`-O2` and above); all 13 cog configurations agree; all 1,260 compile
combinations succeed; and every hardware run reproduces the simulator
checksum exactly.

Two checks are deliberately *not* parity checks, because the original compiler
is wrong and this one is right: float-to-double conversion is compared against
exact IEEE-754 values computed on the host (20/20 against 2/20), and 9-bit
field inserts and overflow builtins are compared against independently
computed references (0 mismatches over 72 and 289 operand pairs respectively).

---

## 5. Two changes carried over from the source tree

The port was made from a working tree that carried two small local changes not
present in the upstream repository.  Both are kept, so that a build from the
supplied patches reproduces exactly the compiler that was tested:

* `propeller.cc` — cog memory is priced at zero in `rtx_costs`, which is
  accurate: a cog-memory operand is addressed directly by the instruction, at
  the same cost as a register.
* `libgcc/config/propeller/crtbegin.S` — the startup bss clear is an inline
  byte loop instead of a call to `memset`, which removes a `memset` dependency
  from every program's startup path.

Neither is my work; both are noted here for provenance.

## 6. Known limitations

* **Cog code and the stack.**  Code compiled for cog memory will build a stack
  frame if register pressure demands it.  That is fine for a standalone `-mcog`
  program, whose startup code sets `sp` up, but an overlay module loaded
  directly into a cog usually has no stack, and the frame would then corrupt
  the module's own cog image.  This is inherited behaviour, not new — the
  original compiler does the same thing under enough pressure — but GCC 16's
  register allocator reaches the threshold somewhat sooner.  Build cog overlays
  with `-Wcog-stack` (see above) and the compiler will name any function this
  happens to; `-Werror=cog-stack` turns it into a build failure.  Nothing in
  code generation avoids it: making the callee-saved registers caller-saved, or
  fixing them out of the allocation order, or repricing memory moves, all leave
  the allocator spilling instead, which is worse.
* The C++ compiler is not built; this is a C toolchain.
* Propeller 2 is not supported, exactly as before.
