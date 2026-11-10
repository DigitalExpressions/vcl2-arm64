# Native Neon support for this VCL2 fork

Date: 2026-11-10. Repository inspected at `cce2d6cf238fe98c4e8c2f3a901b02940ce355ce`.

## Start here on the Mac: setup and implementation handoff

This section is the handoff for a new session opened in VS Code on an Apple
Silicon Mac. Read it together with the implementation phases below; the earlier
chat is not needed to understand the objective or the next steps.

### Current state and agreed scope

- The feasibility study and plan are complete. Implementation has not started,
  and no native Mac build, test, or benchmark has been run.
- The user wants a direct Neon backend for this DigitalExpressions VCL2 fork,
  initially on macOS ARM64, while preserving the ordinary VCL API, x86 support,
  and the selectable sse2neon compatibility backend.
- Core scope is signed/unsigned integer vectors, FP32/FP64, boolean masks,
  128/256/512-bit logical widths, the in-tree vector math, tests, and benchmarks.
  Native FP16 and additional operating systems are follow-up work.
- Retain the logical 512-bit types required by `dex-gfxmath`/`GfxMath.h` even
  though individual Neon registers are 128 bits. The consumer still needs to be
  located and inspected on the Mac.
- Use VS Code for editing and review, Apple Clang/LLDB for builds and debugging,
  CMake/CTest for repeatable tests, and Instruments when profiling is needed.
  Keep building and testing possible from the terminal and CI.
- The user explicitly requested that this document and the existing local
  `sse2neon.h` update both be committed. The header update predates this study;
  preserve its contents and investigate its upstream provenance in Phase 1.
  Its Git blob ID at handoff is
  `affc779889a00f7ec2e9c0b6e200e5d443ba4cf8`.
- The handoff commit is local unless separately pushed or transferred. A fresh
  clone of the remote will contain these changes only after that commit reaches
  the remote. Verify that both files arrive on the Mac.

### Mac prerequisites

| Item | Needed for |
| --- | --- |
| Apple Silicon Mac and native ARM64 terminal/tools | Actual ARM execution, compiler checks, and hardware measurements. Avoid running the development shell through Rosetta. |
| Apple command-line developer tools | Apple Clang, LLDB, Git, and the macOS SDK. Command Line Tools are sufficient for initial development. |
| Full Xcode, recommended for the profiling phase | Instruments and Apple's integrated performance/debugging tools. Daily editing can stay in VS Code. |
| CMake with CTest available on `PATH` | Building and running the planned test/benchmark harness. Ninja is optional; use the available generator. |
| Native Apple Silicon VS Code | Open the repository folder, review edits, and run build/debug tasks. |
| VS Code C/C++ and CMake Tools extensions | C++ navigation/debugging and CMake integration. These do not install the compiler or CMake themselves. |
| Local `dex-gfxmath` sources and a representative integration target | Verify consumer requirements and application behavior. Their location is not yet known. |
| Git/network access | Retrieve pinned upstream testbench/reference sources and inspect dependency revisions. |

If the Apple command-line tools are missing, run this in the Mac terminal and
complete the installer:

```sh
xcode-select --install
```

If full Xcode is used, finish its initial component/license setup and verify
that the active developer directory points to the intended installation.
Install a stable macOS build of [CMake](https://cmake.org/download/) and make its
command-line tools available on `PATH` if they are missing. Use a consistent
Apple Clang toolchain across command-line builds and the editor.

Setup references: [Apple command-line tools](https://developer.apple.com/documentation/xcode/installing-the-command-line-tools),
[VS Code with Clang on macOS](https://code.visualstudio.com/docs/cpp/config-clang-mac),
[VS Code CMake integration](https://code.visualstudio.com/docs/cpp/cmake-quickstart),
and [Apple Instruments profiling](https://help.apple.com/xcode/mac/current/en.lproj/devcc15e6912.html).

### Open the project and verify the environment

Open the repository root containing `vectorclass.h` and this document, then use
its integrated terminal. The Windows path from the original study is not a Mac
path; discover the actual checkout location. Run:

```sh
pwd
git status --short
git log -3 --oneline
sw_vers
uname -m
sysctl -n machdep.cpu.brand_string
xcode-select -p
xcrun --find clang++
xcrun clang++ --version
xcrun --sdk macosx --show-sdk-path
xcrun lldb --version
cmake --version
ctest --version
```

Expect `uname -m` to report `arm64`, Apple Clang to be selected, and a valid
macOS SDK path. Investigate any translated shell, missing tool, conflicting
toolchain, or unexpected architecture before collecting timings. Record the
chip, macOS, compiler, SDK, and CMake versions with the baseline results.

The following optional compiler-only check verifies that the selected toolchain
accepts the native types and basic Neon operations. It does not test this VCL
fork and does not create project files:

```sh
xcrun clang++ -arch arm64 -std=c++17 -x c++ -fsyntax-only - <<'CPP'
#include <arm_neon.h>
float32x4_t check_fma(float32x4_t a, float32x4_t b, float32x4_t c) {
    return vfmaq_f32(c, a, b);
}
float64x2_t check_div(float64x2_t a, float64x2_t b) {
    return vdivq_f64(a, b);
}
CPP
```

This checkout has no CMake harness yet. Creating `CMakeLists.txt`, test targets,
and reproducible configurations belongs to Phase 1. Do not expect existing
presets or launch targets, or use CMake Quick Start to replace repository
contents with a sample project. Once the harness exists, use separate build
directories/configurations for backends, widths, optimization, and sanitizers.
Use C++17 and an explicit ARM64 target, with an M1-compatible baseline; exclude
x86 compiler switches and fast-math from the initial correctness baseline.

### Instructions for the implementation session

Start a new session with this request:

> Read NEON_SUPPORT_PLAN.md, including the Mac handoff section. Implement the
> core native Neon backend following the plan. Start by checking the local Git
> state and Mac toolchain, locating dex-gfxmath/GfxMath.h, and establishing the
> repaired sse2neon test and benchmark baseline. Keep the compatibility backend
> available, preserve x86 behavior and the logical 512-bit API, and validate on
> this Mac as you go. Update the progress log with commands, results, remaining
> failures, and the next concrete step so work can resume in a later session.

On resuming, the agent should:

1. Read applicable repository instructions, this document, and any later
   progress entries. Inspect the current branch, commit, and local changes;
   use the current source as authoritative and refresh stale line references.
2. Run the environment checks above. Inspect both committed and local header
   changes; preserve user edits. Do not replace the bundled sse2neon header
   with a newly downloaded revision without understanding the differences.
3. Locate the consumer in the workspace or nearby known project directories.
   Inspect its build flags, direct intrinsic use, type/layout assumptions, and
   workloads. If unavailable, continue standalone VCL work and report the exact
   integration gap; do not claim consumer validation.
4. Execute Phase 1 before widening the implementation scope. Pin the testbench
   revision, reproduce baseline failures, and repair the compatibility path.
   Create a small CMake/CTest harness and deterministic reference cases. Keep
   initial benchmarks comparable and record the numerical configuration.
5. Continue through the core phases, using an experimental backend switch until
   its coverage gates pass. Do not mark an entire phase complete on the basis
   of an include-only compile or a small prototype.
6. Record x86 checks separately. Native ARM execution cannot validate AVX2 or
   AVX-512 machine code; arrange suitable CI/runners, or explicitly report
   compile-only coverage and missing runtime coverage.
7. Keep this document's progress log current at meaningful checkpoints. Record
   the exact build/test commands or committed preset names, test counts and
   failures, benchmark conditions, implementation commits, and next action.
   Request missing information only when it is necessary for further progress.

No native implementation, tests, build settings, or VS Code configuration were
created during the planning/documentation work. The Mac session starts with
environment validation and Phase 1, unless a later progress entry shows that
work has already been completed.

### Progress log

| Date | Completed and evidence | Outstanding / next action |
| --- | --- | --- |
| 2026-11-10 | Static feasibility study, phased plan, and Mac/VS Code handoff documented. User requested committing the plan and the pre-existing sse2neon header update. No Mac execution results exist. | Transfer the handoff commit to the Mac, verify prerequisites, inspect the consumer, and begin Phase 1. |

## Feasibility and recommendation

**Yes: an AArch64 Neon backend for Apple M-series processors is feasible.** Keep
VCL's C++ vector API, implement its 128-bit primitives directly with
`<arm_neon.h>`, and provide its wider logical vectors by composing those primitives.

This fork already has a partial Apple ARM port through `sse2neon.h`. That path
also compiles to native ARM instructions; it does not require Rosetta. A direct
backend would remove the SSE translation dependency from the core ARM path and
let implementations use VCL's actual contracts instead of reconstructing SSE
instructions. It is a substantial backend project, not an include-file change.

First repair and measure the existing path. Retain it as a selectable fallback
while the direct backend is developed. Promote the direct backend only after
correctness, application integration, and performance checks pass. Simple
arithmetic may compile identically through either backend; speedups are a
hypothesis to test, especially for permutations, masks, reductions, conversions,
and fused arithmetic.

Arm documents the relevant integer and floating-point lane shapes on its
[Neon overview](https://www.arm.com/technologies/neon). Its
[ACLE intrinsics reference](https://arm-software.github.io/acle/neon_intrinsics/advsimd.html)
specifies AArch64 operations for FP32/FP64 arithmetic, division, square root,
comparisons, table lookup, and fused multiply-add. These are sufficient building
blocks for the proposed backend; operations without direct matches need sequences
or documented fallbacks.

## What is already present

These are static code findings, not results from running the library on a Mac.

| Location | Finding | Consequence |
| --- | --- | --- |
| `instrset.h:39-52` | An ARM preamble includes sse2neon, forces `INSTRSET=6` and `MAX_VECTOR_SIZE=512`, replaces `_mm_getcsr()` with `1`, and defines an x86 header guard. It precedes VCL's own include guard. | There is already a translation backend, but configuration and control-state handling need repair. |
| `instrset.h:250-283` | The ARM `cpuid` branch returns four constant values of `1`. | These are not meaningful x86 CPUID feature words; they cannot support the existing detector. |
| `instrset.h:372-389` | GNU/Clang `bit_scan_reverse` still contains `bsrl`/`bsrq` x86 assembly. | ARM needs compiler builtins with the existing input contract preserved. |
| `instrset_detect.cpp:22-105` | Runtime detection contains x86 `xgetbv` assembly and CPUID feature tests. | This translation unit and the dispatcher examples need explicit architecture handling. |
| `vectorclass.h:44-85` | Header selection is controlled by an ordered x86 `INSTRSET` scale. | Add backend selection without assigning Neon a fictitious AVX capability level. |
| `vectori128.h`, `vectorf128.h` | The 128-bit classes are implemented around SSE intrinsics and raw SSE types. Together they contain about 10,000 lines including comments and inactive branches. | A direct backend must cover an extensive API, including integer support required by floating-point math. |
| `vectori256e.h:64-125`, `vectorf256e.h:50-105` | Wider emulated vectors compose 128-bit pieces, but their bases still store `__m128*` and some functions call SSE intrinsics directly. | Reuse the composition design after adapting these storage and operation boundaries. |
| `vectori512e.h`, `vectorf512e.h` | These files mostly compose smaller VCL vectors. | Much of the 512-bit implementation should be reusable; compilation and API coverage must verify this. |
| `vectorf128.h:1312-1370`, `:2282-2343` | Explicit fused-operation selection checks x86 FMA macros. | Add a Neon capability path; do not define `__FMA__` to impersonate x86 FMA support. |
| `vectorf128.h:972-1000`, `:1960-1985` | FP32 has a small ARM NaN-classification branch; FP64 lacks an equivalent branch. | Audit both types, including behavior under aggressive floating-point optimization. |
| `vectormath_exp.h:605-615` | Active 128-bit helper functions use SSE bit casts. | Port these helpers as well as the main vector classes. |
| `vectormath_lib.h` | This is an Intel SVML integration with x86 symbols and calling assumptions. | Use the in-tree vector math algorithms for the first Neon release. |
| Repository root | No test suite or build/CI definitions are checked in. | Establish a repeatable baseline before implementation. |

The working tree already contained a substantial modification to `sse2neon.h`
when inspected. The user subsequently requested including that existing update
in the documentation handoff commit. Preserve it and record its provenance
before any dependency update. The header was studied alongside upstream; its
exact revision should not be assumed to match upstream master.

Commit `481cb7f` explicitly says the 512-bit maximum was retained to avoid type
errors in `GfxMath.h`. Preserve that application requirement. The existing comment
about limiting to "128byte" is misleading: Neon vectors are 128 **bits**, while
`MAX_VECTOR_SIZE` describes logical VCL widths.

## Target architecture and compatibility contract

Initial platform: little-endian macOS `arm64`, Apple Clang, C++17, with an
M1-compatible baseline. Keep the x86 implementation supported. Linux AArch64 and
Windows ARM64 can follow after the Apple backend is validated.

| VCL logical width | Neon representation |
| --- | --- |
| 128 bits | One 128-bit vector |
| 256 bits | Two 128-bit vectors |
| 512 bits | Four 128-bit vectors, composed through two 256-bit halves |

These are representations, not throughput promises. There is no 256/512-bit Neon
instruction corresponding to those wider VCL types. Do not make SVE, SME, or
processor-generation-specific instructions prerequisites for this backend.

Preserve vector names, lane order, constructors from scalars, ordinary operators,
loads/stores, masks, conversions, compile-time permutations, and metadata such as
`size()` and `elementtype()`. Cover signed/unsigned 8-, 16-, 32-, and 64-bit integer
vectors, FP32/FP64 vectors, and their boolean types.

Source that directly constructs VCL types from `__m128*`, converts them to SSE
intrinsic types, or calls `_mm_*` is a separate compatibility surface. Keep that
source working through the selectable sse2neon backend. The direct backend should
expose explicit native accessors instead of globally defining SSE types. An
optional interoperability header can be considered after inspecting actual
consumer requirements. Do not promise that all intrinsic-level source works
unchanged without a translation layer.

Each translation unit in an application must agree on backend and ABI-affecting
configuration. Do not pass VCL objects across modules compiled with different
backend representations. Check sizes, alignment, overloads, and by-value calls
within each supported configuration; cross-architecture binary ABI compatibility
is not a goal.

## Proposed code organization

Introduce a small `vcl_config.h` for architecture detection and backend selection.
Use names such as `VCL_BACKEND_X86`, `VCL_BACKEND_SSE2NEON`, and
`VCL_BACKEND_NEON`, plus an explicit user override. Detect AArch64 with compiler
target macros such as `__aarch64__` and `__arm64__`; isolate any additional
compiler spellings there. Configuration must describe the compilation target,
including when building separate slices of a universal application.

Keep `INSTRSET` as the existing x86 selection mechanism. The sse2neon backend may
retain its internal level-6 compatibility setting during transition. Select the
direct backend explicitly in `vectorclass.h` and replace reachable x86-only
conditions with backend/capability checks. Do not set a high `INSTRSET` merely to
enable FMA or wider logical vectors. Keep native width separate from
`MAX_VECTOR_SIZE`, and respect a caller-specified maximum of 128, 256, or 512.

Suggested new files:

- `vcl_config.h`: target/backend selection and configuration checks.
- `instrset_neon.h`: ARM helpers and the documented control-state adapter.
- `vectori128_neon.h`: integer vectors and masks using native vector storage.
- `vectorf128_neon.h`: FP32/FP64 vectors and masks using native vector storage.
- `tests/` and `benchmarks/`: the coverage harness and application kernels.

Use small architecture-neutral helpers for bit reinterpretation, composition,
and compile-time permutation analysis. Preserve the existing x86 storage/API
where possible. Adapt the emulated 256-bit bases through selected storage types
and VCL operations; audit all reachable 512-bit and conversion code afterward.
Do not duplicate all wider headers unless an actual ABI or compiler constraint
requires it. Reuse the algorithmic templates in `instrset.h` after separating
them from x86-only helpers.

## Lessons from studying sse2neon

The upstream [sse2neon project](https://github.com/DLTcollab/sse2neon) covers SSE
through SSE4.2 and selected additional extensions, rather than AVX. Its
[header implementation](https://github.com/DLTcollab/sse2neon/blob/master/sse2neon.h)
is a useful reference for semantic corrections and ARM intrinsic selection.

In particular, its min/max compatibility path compares and selects, byte shuffle
masks indices before table lookup, and float-to-integer conversion fixes invalid
inputs. Its control-word adapter maps selected rounding and flush controls rather
than reproducing all MXCSR state. Precision switches improve selected behavior;
they do not establish universal bit-for-bit SSE equivalence.

Carry these lessons into VCL operations rather than copying an SSE instruction
interface into the new backend. Preserve the MIT notices if code is adapted
from sse2neon, alongside the existing VCL notices.

The upstream [performance classification](https://github.com/DLTcollab/sse2neon/blob/master/perf-tier.md)
can suggest places to inspect, but it uses static analysis and non-Apple cycle
assumptions. It is not an Apple M-series benchmark or evidence of a VCL speedup.

## Semantic decisions to implement and test

| Area | Required treatment |
| --- | --- |
| Masks | Preserve VCL broad masks: zero or all bits set per lane. Use native comparisons and bit selection. Test `to_bits`, `load_bits`, `horizontal_and/or`, counts, and first-match operations explicitly. |
| Integer arithmetic | Preserve modular and saturating behavior, signedness, widening/narrowing, high-half products, and constant/runtime divisor classes. Avoid introducing scalar signed-overflow UB in fallbacks. |
| Shifts | Test immediate and runtime counts around zero and element width, according to the existing API's defined domain. Neon signed-count shifts do not automatically reproduce every SSE shift contract. |
| Permutations | Preserve zero lanes, lane numbering, cross-half movement, and `V_DC` don't-care semantics. Prefer `vext`, zip/unzip, transpose, reverse, and table lookup as appropriate; arbitrary unused lanes need not match. |
| FP min/max | Preserve the existing API's operand ordering for NaNs and signed zeros. A plain Neon min/max is not a universal substitute. Keep any explicitly different NaN-propagating functions distinct. |
| Rounding/conversions | Preserve ties, explicit versus current rounding mode, signed/unsigned range boundaries, and documented NaN/overflow results. Native conversions can require fixups. |
| Division/sqrt | Use AArch64 division and square-root instructions for ordinary operations. Keep approximate reciprocal/rsqrt as separate APIs with stated error bounds and special-value handling. |
| FMA | Implement `mul_add`, `mul_sub`, `nmul_add`, and precision helpers deliberately with correct operand order. `vfmaq_f32(c,a,b)` computes `c+a*b`. Validate polynomial accuracy after enabling fused operations. |
| FP classification | Use integer-bit classification where robustness under finite-math assumptions is required. Check FP32 and FP64 behavior for NaNs, infinities, signed zeros, and subnormals. |
| Reductions | Preserve exact integer results and widening contracts. Evaluate FP reduction order against the current contract; algebraic equivalence does not guarantee the same rounded result. |
| FP environment | Remove the constant `_mm_getcsr()` override. Define a tested FPCR/FPSR mapping for the existing public control helpers, including supported rounding, flush behavior, exception fields, and unsupported distinctions. Save and restore thread state in tests. |
| Memory operations | Partial loads/stores must touch only their documented lanes. Preserve alignment preconditions. Implement `store_nt` with a correct ordinary store fallback initially; do not promise x86 cache-bypass behavior. |
| Gather/scatter/lookups | Neon has no general x86-style vector gather/scatter equivalent. Use correct scalar/lane sequences where necessary, preserving bounds and documented duplicate-index behavior. Optimize only after profiling. |

Use existing VCL behavior and documented contracts as the specification. Where
backends already differ, record an explicit policy and test it. NaN payloads,
exception behavior, approximate results, and fused rounding need separate
expectations; a blanket "exact SSE compatibility" claim would be misleading.

## Implementation phases and completion gates

### 1. Establish the baseline and consumer requirements

Record the VCL revision and the modified sse2neon dependency's provenance. Audit
the actual `dex-gfxmath`/`GfxMath.h` consumer for required types, direct intrinsic
access, control-word use, vector widths, and compile flags. That consumer has not
been inspected as part of this folder-only study.

Bring in a pinned, compatible revision of the upstream
[VCL testbench](https://github.com/vectorclass/testbench). Adapt its x86 feature
discovery and compiler flags for ARM. It already has separate general-operation,
permutation/lookup, math, and FP16 test programs; verify compatibility with this
fork's older VCL API before using its lists unchanged.

Repair target detection, configuration placement, bit scans, the control-word
override, and architecture handling in `instrset_detect.cpp` and the examples.
Keep x86-named feature queries honest about x86 extensions; add a separate Neon
capability query where needed. Define/document the ARM behavior of the legacy
numeric detector instead of inventing CPUID words or dispatching to x86 objects.

Gate: the repaired sse2neon path builds and passes the selected core tests on a
real Apple ARM Mac; x86 regression checks pass; representative consumer kernels
have recorded correctness results and timings. Enable/test relevant sse2neon
precision options for the chosen numerical contract.

### 2. Add backend selection and a complete dependency slice

Add the configuration/header boundary. Start the direct backend with `Vec128b`,
`Vec4i`/`Vec4ui`, their masks, `Vec4f`, and `Vec4fb`, then the 64-bit integer
types needed by `Vec2d`/`Vec2db`. Cover scalar construction, memory, arithmetic,
comparison, selection, casts, and basic permutation in this slice.

An incomplete slice stays behind an explicit experimental switch. Do not make
`#include "vectorclass.h"` expose wider or math APIs whose dependencies are not
implemented. Keep the sse2neon backend selectable for normal application builds.

Gate: a self-contained 128-bit slice compiles without sse2neon, passes its
published tests, and emits expected Neon code for representative kernels.

### 3. Complete the 128-bit API

Add remaining integer widths and masks, widening/narrowing, saturation, shifts,
multiplication helpers, divisor classes, reductions, masks-to-bits, permutations,
blends, lookups, gathers/scatters, and partial memory operations. Complete all
FP32/FP64 helpers and their semantic fixups. Reuse VCL algorithms where useful;
implement operations without native instructions with reviewed sequences.

Gate: every in-scope 128-bit public operation has compile and behavior coverage;
tests cover boundary values, not just random normal inputs. `MAX_VECTOR_SIZE=128`
works with namespace and include-order variations.

### 4. Restore 256/512-bit logical vectors

Adapt the emulated storage bases and residual intrinsic calls. Reuse composition
for wider integer/floating vectors, and audit `vector_convert.h`, masks, and
cross-half permutations. Preserve the 512-bit logical API required by GfxMath.

Gate: all three maximum-width configurations build and pass their applicable
tests. Wide operations agree with the corresponding smaller-vector composition,
including lane order and masks. The consumer builds against the direct backend.

### 5. Validate vector math and integration

Port residual casts such as `fraction_2` in `vectormath_exp.h`. Reuse and test
`vectormath_common.h`, `vectormath_exp.h`, `vectormath_trig.h`, and
`vectormath_hyp.h`, including FMA-dependent precision helpers. Guard x86-specific
branches by real capabilities. Give the Intel SVML header a clear ARM diagnostic
and direct users to the supported in-tree math headers.

Gate: stated accuracy limits pass for float/double at all supported widths,
including range-reduction boundaries and special values. Actual dex-gfxmath
workloads pass integration checks under their production compiler flags.

### 6. Tune and release

Benchmark both ARM backends on identical native builds and inputs. Focus on
application kernels plus permutations, mask extraction, conversions, reductions,
and fused math. Inspect optimized assembly for scalarization, stack traffic, and
unnecessary mask conversions. Report CPU, compiler, flags, input sizes, median
timings, and variability; compare equivalent numerical settings.

Gate: no unexplained correctness failures or material performance regressions
in the agreed workloads. Publish the support matrix and remaining operation
costs. Retain the explicit compatibility backend when promoting direct Neon.

### 7. Optional follow-up: native FP16 and additional platforms

FP16 has a separate API and substantial SSE-dependent implementation in
`vectorfp16e.h`. It is outside the first FP32/FP64/integer release. Gate optional
native half arithmetic on compiler/target capabilities and test its rounding
against VCL's widening-based emulation, which can differ. Publish FP16 as
unsupported in the direct backend until covered; continue offering the
compatibility path only where its FP16 behavior has actually been verified.

Then consider Linux AArch64 and Windows ARM64 as separate validation targets.

## Validation and release evidence

- Use deterministic scalar/reference cases and the VCL testbench as independent
  checks. Compare direct Neon with repaired sse2neon and x86 VCL; agreement
  between the two ARM paths alone is not proof of correctness.
- Require exact integer/mask/bit-operation results where defined. Distinguish
  floating-point bitwise tests from ULP/error-bound tests for approximations and
  transcendental functions. Test NaNs, infinities, signed zero, subnormals,
  conversion limits, rounding ties, shift boundaries, and saturation endpoints.
- Instantiate template APIs deliberately, including zero and don't-care
  permutation lanes, cross-half cases, and divisor edge cases. An include-only
  smoke test does not cover template code generation.
- Check partial loads/stores at protected page boundaries and use sanitizers
  where available. Test `VCL_NAMESPACE`, all maximum widths, standalone header
  inclusion, and configuration conflicts.
- Run native macOS ARM CI on the minimum supported Apple Clang and a current
  supported release. Run core tests in debug and production optimization modes;
  evaluate fast-math separately and document any unsupported contracts.
- Retain x86 CI with SSE2 and representative AVX2/FMA and AVX-512 configurations,
  using execution-capable runners or compile-only jobs as appropriate.
- Benchmark on M1 hardware plus a newer available M-series machine when possible.
  A generic ARM emulator or cross-compile can supplement correctness coverage,
  but cannot establish Apple hardware performance.

## Effort and first deliverable

Planning estimate for one engineer experienced with SIMD, with native Mac access:
baseline audit/repairs in roughly 3-5 working days; a focused direct-Neon prototype
in another 2-4 weeks; a tested core integer/FP32/FP64 backend with wider vectors,
math, integration, and tuning in roughly 8-14 weeks total. Broad FP16 support and
additional operating systems require separate estimates. Revise these ranges
after baseline testing and the consumer audit; they are not measured commitments.

The subsequent conversation provisionally budgeted about 30-60 hours of active
agent implementation and testing across multiple sessions, assuming a direct
native Mac test loop. The first environment/baseline checkpoint was estimated
at 3-6 hours, with a useful prototype around 6-12 hours from the start. These are
unvalidated planning ranges, not evidence of completion or guaranteed runtime;
reassess after actual baseline failures and consumer requirements are known.
The earlier week-based estimate describes human engineering effort.

The first implementation deliverable should be the repaired compatibility
backend, reproducible tests/benchmarks, and a small direct-Neon vertical slice.
That provides evidence for the larger migration while keeping an application
build available throughout development.

## Study limits

This document follows a static inspection of this checkout and the linked
primary sources. No native macOS build, runtime test, or performance benchmark
was run during this study. The developer shell is Windows, and no Apple build
environment was used. Implementation files and the pre-existing sse2neon changes
were left untouched by the agent. The user asked to commit the existing
`sse2neon.h` update alongside this plan; committing that snapshot does not imply
that the header or the existing ARM backend has passed native validation.
