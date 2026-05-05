# Boxer (leopard_legacy / PPC JIT work)

Boxer is a macOS DOS-game frontend that embeds DOSBox 0.74. This checkout is the
`leopard_legacy` branch — the active line for PowerPC and Mac OS X 10.5 support.
Mainline Boxer dropped PPC at commit `31952311`; do not assume mainline patches
apply here without checking.

## Toolchain (pinned, do not propose changing)

- Compiler: `com.apple.compilers.llvmgcc42` (LLVM-GCC 4.2)
- SDK: `macosx10.6`
- Deployment target: 10.5
- `VALID_ARCHS = "i386 x86_64 ppc"`

This means C++03 only. Do not propose C++11/14/17 features (no `nullptr`,
`std::atomic`, range-for, `auto`, lambdas, `constexpr`, `<chrono>`, etc.).

## Active work

Integrating a third-party PPC JIT patch (authored against DOSBox SVN r4301)
into Boxer's DOSBox 0.74 base. The patch itself is known-good — it ships as
working Mac G4/G5 builds at r4301. Bugs encountered here are integration-side
(0.74 ↔ r4301 deltas), not patch-side. Debug accordingly.

**Read `PPC_JIT_NOTES.md` for current state, known issues, and prior attempts
before proposing changes to JIT-related code.** Update it in the same turn you
make a load-bearing decision — don't defer.

A second workstream is now active: bumping the embedded DOSBox tree from 0.74
to SVN r4301 so the JIT patch becomes a clean drop-in. **Read
`R4301_BASEBUMP_NOTES.md` for the live phase plan, conflict queue, build-cycle
log, and decisions.** Same update-in-the-same-turn rule applies. The base-bump
work happens on the `r4301-basebump` branch.

## Working mode

The user builds in a Snow Leopard VM (Xcode 3.2.x for the Legacy Release
configuration — the only one that produces a PPC binary; see `Readme.txt`)
and tests the resulting binary on a real G4. Claude cannot build, run, or
observe anything. Every change is a round-trip through a human build-and-test
cycle, so:

- Make edits worth a build cycle. Don't churn.
- When you're guessing, say so. Don't present speculation as diagnosis.
- If a fix depends on runtime behavior you can't verify, name what the user
  should look for when they test it on the G4.

## Repo orientation

- `DOSBox/` — the embedded DOSBox 0.74 source, with Boxer modifications.
  Files containing `Boxer`/`boxer` markers are integration points.
- `DOSBox/src/cpu/core_dynrec/` — dynrec backend; PPC JIT lives here as
  `risc_ppc.h` (currently untracked while integration is in flight).
- `Boxer/` — the Cocoa app. `BXCoalface.{h,mm}` and `BXEmulator+*.mm` are
  the main bridge to DOSBox internals.
