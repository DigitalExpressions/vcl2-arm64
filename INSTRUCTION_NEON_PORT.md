# Start the native Neon port on your Mac

Open this repository's root folder in VS Code on an Apple Silicon Mac. Make
sure the checkout includes this file, [NEON_SUPPORT_PLAN.md](NEON_SUPPORT_PLAN.md),
and the updated `sse2neon.h` from commit `6b3471e` or a descendant. Complete the
Mac prerequisites in the plan's **Start here on the Mac** section.

Paste this prompt into a new session:

```text
Read NEON_SUPPORT_PLAN.md, especially the Mac setup and handoff section, and
implement the core native Neon backend following that plan.

First inspect the Git state and verify the native ARM64 Apple Clang toolchain.
Locate dex-gfxmath/GfxMath.h if available, then establish tests and benchmarks
and repair the existing sse2neon baseline before implementing the direct Neon
backend. If the consumer sources are unavailable, continue standalone VCL work
and record the integration gap.

Preserve the VCL API, x86 support, selectable sse2neon backend, and logical
512-bit vector types. Preserve existing user changes. Investigate the bundled
sse2neon.h update's provenance before replacing or updating that dependency.

Proceed through the implementation phases, compiling and testing on this Mac.
Keep NEON_SUPPORT_PLAN.md updated with completed work, exact validation commands,
remaining failures, and the next step so another session can resume. Keep
unvalidated code behind the experimental backend switch and report missing
x86 runtime coverage separately from native ARM validation.

Begin the work now. Ask only when missing information prevents meaningful
progress.
```

Alternatively, use this shorter prompt to refer to the instructions above:

```text
Read INSTRUCTION_NEON_PORT.md and NEON_SUPPORT_PLAN.md, then begin the native
Neon port according to the startup prompt and implementation plan. If work has
already started, verify the current source and resume from the progress log.
```

The detailed plan is the source for architecture, scope, phases, and completion
gates. Its progress log is the source for completed work and the next action.
At this handoff, implementation has not started and no native Mac validation
has been run. Opening these files alone does not initiate implementation; send
one of the prompts when ready to start.
