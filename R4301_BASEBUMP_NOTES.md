# r4301 base-bump notes

Live planning + status doc for bumping Boxer's embedded DOSBox tree from 0.74
to SVN r4301, so that jmarsh's `patch-r4301.diff` becomes a clean drop-in
against the revision it was authored for.

`PPC_JIT_NOTES.md` is the writeup of the prior JIT-on-0.74 integration. This
file is the live doc for the base-bump workstream — update it in the same turn
as any load-bearing decision. Don't defer.

## Branches

- `leopard_legacy` — clean Boxer leopard_legacy, untouched.
- `r4301` (deleted; renamed to `r4301-basebump`) — formerly held the
  JIT-on-0.74 backport. The previous JIT-on-0.74 work is reachable via commit
  hashes / reflog if needed.
- `r4301-basebump` — **active branch**. The base-bump work happens here.

The JIT-on-0.74 backport commit (`da37579d`) sits in this branch's history but
will be superseded once the `DOSBox/` tree is replaced with the r4301-merged
result. Reachable for reference via the commit hash regardless.

## Reference checkouts (read-only, outside this repo)

- `~/repos/dosbox-0.74` — vanilla DOSBox at the `RELEASE_0_74` tag (r3609).
  VERSION reads `0.74`.
- `~/repos/dosbox_r4301` — vanilla DOSBox SVN trunk at r4301
  (`svn co -r4301 trunk`). VERSION reads `0.74-2`; the lag is just an unbumped
  string in trunk, not a tree mismatch.
- `~/repos/dosbox_r4301_patched` — r4301 with `patch-r4301.diff` applied and
  built. Ground truth for what the patch produces when it goes on cleanly.
- `~/repos/Boxer` — unmodified Boxer leopard_legacy, used as the source of
  truth for "Boxer's modifications to DOSBox 0.74" via diff vs `dosbox-0.74`.

## Strategy

1. Diff `~/repos/Boxer/DOSBox` against `~/repos/dosbox-0.74` to enumerate every
   Boxer-side modification.
2. Forward-port those modifications onto unpatched r4301. Get a working PPC
   binary that boots and runs DOS programs under the **normal** core. No JIT
   yet — that is a separate milestone so a JIT failure cannot be confused
   with a base-bump failure.
3. Apply `patch-r4301.diff` on the forward-ported tree. Most should land
   cleanly because the base now matches the patch's authoring revision.

## Phase plan

### Phase 0 — sandbox

- [ ] Stand up a throwaway git repo (location TBD, outside this tree) with
      three branches off a `vanilla-074` initial commit:
      - `vanilla-r4301` — `~/repos/dosbox_r4301` contents
      - `boxer-074` — `~/repos/Boxer/DOSBox/` contents
- [ ] `git merge boxer-074` into `vanilla-r4301`. Capture conflict list.
- [ ] Confirm `~/repos/dosbox_r4301` vs `~/repos/dosbox_r4301_patched` matches
      the `patch-r4301.diff` committed at `fe7b1960`. Avoids surprises.

### Phase 1 — forward-port Boxer mods onto r4301 (no JIT)

- [ ] Resolve merge conflicts subsystem by subsystem (cpu/, dos/, hardware/,
      ints/, gui/, shell/, misc/, includes). One commit per subsystem.
- [ ] Triage new-in-r4301 files. Cross-check Madd's `4a85221c` for what
      mainline-Boxer kept vs dropped. Tracking table below.
- [ ] Audit the load-bearing PPC fixes against the new r4301 code. Tracking
      checklist below.
- [ ] Update `Boxer.xcodeproj` to add/remove file references for the new tree.
- [ ] **Build cycle 1**. Test plan: cold-boot, a known-working game runs
      under the normal core on the G4, no regressions vs current PPC build.

### Phase 2 — re-apply the JIT patch

- [ ] `patch -p1 < patch-r4301.diff` against the Phase 1 result.
- [ ] Resolve residual conflicts. Known ones from PPC_JIT_NOTES: FAT coalface
      (skip MBR/boot-sector hunks, keep FAT-entry hunks), `cache.h`
      `PFLAG_HASCODE` divergence.
- [ ] Re-add Boxer's `config.h` clause enabling `C_DYNREC` on
      `__ppc__ || __ppc64__`.
- [ ] Verify `dyn_set_eip_end` ended up with the patch's 32-bit-load form.
- [ ] Add `risc_ppc.h` to Xcode project if needed (per PPC_JIT_NOTES the
      `#include` is conditional so a build-phase entry may not be necessary).
- [ ] **Build cycle 2**. Test plan: same workload as cycle 1 but with
      `core=dynamic`.

### Phase 3 — validation

- [ ] Workload sweep on the G4. Anything that regresses vs the current
      JIT-on-0.74 build (commit `da37579d` on the old `r4301` branch) is a
      sign Phase 1 mod-replay missed something.

## Phase 0 conflict queue

Populated after the 3-way merge runs. Each row: file, conflict shape, decision,
rationale.

| File | Shape | Decision | Rationale |
|------|-------|----------|-----------|
| _(pending)_ | | | |

## New-in-r4301 file triage

Files added between 0.74 and r4301 that Boxer must explicitly accept, drop, or
stub. Cross-check `4a85221c` for prior art.

| File | Mainline (`4a85221c`) | This branch | Rationale |
|------|----------------------|-------------|-----------|
| `src/cpu/core_dyn_x86/risc_x64.h` | _(tbd)_ | _(tbd)_ | x86_64-only; PPC build does not need it but i386/x86_64 builds may |
| `src/cpu/core_dynrec/risc_armv8le.h` | _(tbd)_ | _(tbd)_ | ARMv8 backend; not built on any Boxer arch |
| `src/dos/drive_overlay.cpp` | _(tbd)_ | _(tbd)_ | New overlay drive feature |
| `src/hardware/pci_bus.cpp` | _(tbd)_ | _(tbd)_ | New PCI bus emulation |
| `src/hardware/pci_devices.h` | _(tbd)_ | _(tbd)_ | Header for above |
| `src/hardware/mame/` | _(tbd)_ | _(tbd)_ | New mame-derived chip emulation directory |

## PPC-fix audit checklist

These fixes are load-bearing for PPC and must survive the tree replacement.
They live far from the patch hunks, so the merge can silently lose them if
upstream rewrote the surrounding code.

- [ ] `6995713f` — FAT image endianness / coalface helpers
      (`BXCoalfaceDrives.{h,mm}` in `Boxer/`, plus call sites in
      `src/dos/drive_fat.cpp`)
- [ ] `9456f8cf` — MT-32 endianness fix
- [ ] `f4935f44` — Tandy / CGA endianness fix
- [ ] `Segs::val[]` declared as `Bit16u[8]` in `include/regs.h`. PPC_JIT_NOTES
      says upstream fixed this in 2018; r4301 may already have it. Verify
      rather than assume.

## Build cycle log

Each row: cycle number, what was sent, what was tested, outcome, follow-up.

| # | Sent | Tested | Outcome | Follow-up |
|---|------|--------|---------|-----------|
| _(pending)_ | | | | |

## Decisions

Append-only log of load-bearing decisions made during the work. One line each.

- _(none yet)_
