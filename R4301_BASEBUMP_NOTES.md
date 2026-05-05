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

### Phase 0 — sandbox  ✓

- [x] Sandbox repo at `~/repos/r4301-merge-sandbox` with three branches off
      `vanilla-074` initial commit:
      - `vanilla-r4301` — `~/repos/dosbox_r4301` contents (excluding `.svn`)
      - `boxer-074` — `~/repos/Boxer/DOSBox/` contents overlaid on vanilla-074
        (no `--delete`, so vanilla files Boxer doesn't track stay in place;
        keeps the merge from seeing "boxer deleted Makefile.am" as a conflict
        against r4301's edits to those files)
- [x] `git merge boxer-074` into `vanilla-r4301` ran. Result: 30 conflicted
      files (73 conflict hunks total), 33 auto-merged, 7 added-by-Boxer,
      no rename detection issues. Sandbox is left in pre-commit conflicted
      state so individual conflicts can be inspected with `git diff` and
      `git mergetool`. Do not commit the merge in the sandbox — it is a
      scratchpad, not authoritative output.
- [x] Confirmed: `patch -p0 < patch-r4301.diff` against vanilla r4301
      produces a tree identical (modulo build artifacts and user noise:
      `.o`, `.deps`, `Makefile`, `Makefile.in`, `.DS_Store`, `.claude`) to
      `~/repos/dosbox_r4301_patched`. The patch file in the repo is bit-good.

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

30 files with merge conflicts; 73 hunks total. Hunk counts in parens.
Conflict-resolution decisions go in the "Decision" column as we work each one.
Subsystems are batched so Phase 1 commits group naturally.

### include/ (3 files, 3 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `include/dma.h` | 1 | take r4301 | r4301 made `dma_wrapping` `static` in dma.cpp and exposed `DMA_SetWrapping(Bitu)` as the public API. Boxer's hardware/dma.cpp already defines `DMA_SetWrapping`, so the header just lines up with what r4301 expects. The dma.cpp conflict in the hardware batch will need Boxer's `Bit32u dma_wrapping` made `static` to match. |
| `include/dosbox.h` | 1 | merge | Keep Boxer's `#include "BXCoalface.h"` (BXCoalface `#define`s `E_Exit` to `boxer_die`, so the underlying declaration must stay hidden). Update the commented-out marker line to r4301's improved `GCC_ATTRIBUTE(noreturn)` form for documentation only. |
| `include/setup.h` | 1 | take r4301 | Adds `getMin()`/`getMax()` accessors and changes `Prop_int::SetValue` from `void` to `bool`. No Boxer caller depends on the void return (verified by grep). |

### src/cpu/ (1 file, 2 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/cpu/core_dyn_x86/dyn_fpu_dh.h` | 2 | _(pending)_ | |

### src/dos/ (8 files, 25 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/dos/cdrom_image.cpp` | 1 | _(pending)_ | |
| `src/dos/dos_memory.cpp` | 2 | _(pending)_ | |
| `src/dos/dos_programs.cpp` | 8 | _(pending)_ | likely heavy: imgmount, mount, etc. |
| `src/dos/drive_cache.cpp` | 3 | _(pending)_ | |
| `src/dos/drive_fat.cpp` | 3 | merge ✓ | All three conflicts: keep Boxer's coalface byteswap **and** r4301's new code, in the right order. Hunk 1 (boot sector): r4301 added a new floppy-format-detection block that reads `bootbuffer.nearjmp/mediadescriptor/oemname` directly — Boxer's `boxer_FATBootstrapLittleToHost(bootbuffer)` call must run *before* this block so the new code sees host-endian fields on PPC. Hunks 2+3 (directoryChange / addDirectoryEntry writes): r4301 added a `writeSector(sectnum, data)` helper that wraps the absolute-vs-CHS distinction. Use the helper, but keep Boxer's `boxer_FATDirEntryHostToLittle` byteswap before the write. All six Boxer coalface call sites (lines 737, 781, 1110, 1222, 1266, 1313 in the merged file) present and correct. |
| `src/dos/drive_iso.cpp` | 2 | _(pending)_ | |
| `src/dos/drive_local.cpp` | 5 | _(pending)_ | |
| `src/dos/drives.h` | 1 | _(pending)_ | |

### src/ (root, 1 file, 1 hunk)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/dosbox.cpp` | 1 | _(pending)_ | |

### src/gui/ (1 file, 8 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/gui/midi.cpp` | 8 | _(pending)_ | likely heavy: r4301 added new MIDI infra; Boxer has CoreMIDI hooks |

### src/hardware/ (12 files, 28 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/hardware/adlib.cpp` | 1 | _(pending)_ | r4301 switched to MAME OPL backend |
| `src/hardware/dbopl.cpp` | 1 | _(pending)_ | |
| `src/hardware/dma.cpp` | 3 | _(pending)_ | |
| `src/hardware/gus.cpp` | 1 | _(pending)_ | |
| `src/hardware/joystick.cpp` | 3 | _(pending)_ | |
| `src/hardware/mixer.cpp` | 2 | _(pending)_ | |
| `src/hardware/pcspeaker.cpp` | 1 | _(pending)_ | |
| `src/hardware/serialport/nullmodem.cpp` | 2 | _(pending)_ | |
| `src/hardware/serialport/softmodem.cpp` | 1 | _(pending)_ | |
| `src/hardware/tandy_sound.cpp` | 3 | _(pending)_ | **load-bearing** — `f4935f44` PPC fix lives here |
| `src/hardware/vga_draw.cpp` | 4 | _(pending)_ | |

### src/ints/ (2 files, 2 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/ints/bios_keyboard.cpp` | 1 | _(pending)_ | |
| `src/ints/int10_char.cpp` | 1 | _(pending)_ | |

### src/misc/ (1 file, 3 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/misc/setup.cpp` | 3 | _(pending)_ | |

### src/shell/ (2 files, 6 hunks)

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/shell/shell.cpp` | 3 | _(pending)_ | |
| `src/shell/shell_cmds.cpp` | 3 | _(pending)_ | |

### Auto-merged (33 files, no conflicts but worth a glance)

These auto-merged cleanly. Most should be fine, but git's auto-merge happily
takes adjacent non-conflicting hunks from both sides — which can produce a
syntactically valid file that does the wrong thing if the two sides were
making coordinated changes. Worth a spot-check during Phase 1, especially
for files where the auto-merged result references new-in-r4301 APIs.

`include/dos_system.h`, `include/joystick.h`, `include/keyboard.h`,
`include/timer.h`, `src/cpu/core_dyn_x86/cache.h`,
`src/cpu/core_dyn_x86/decoder.h`, `src/cpu/core_dyn_x86/dyn_fpu.h`,
`src/cpu/cpu.cpp`, `src/dos/cdrom.cpp`, `src/dos/dev_con.h`,
`src/dos/dos.cpp`, `src/dos/dos_execute.cpp`,
`src/dos/dos_keyboard_layout.cpp`, `src/dos/drive_virtual.cpp`,
`src/dos/drives.cpp`, `src/fpu/fpu_instructions_x86.h`,
`src/gui/render.cpp`, `src/hardware/hardware.cpp`,
`src/hardware/iohandler.cpp`, `src/hardware/ipx.cpp`,
`src/hardware/keyboard.cpp`, `src/hardware/opl.cpp`,
`src/hardware/pic.cpp`, `src/hardware/serialport/directserial.cpp`,
`src/hardware/serialport/misc_util.cpp`,
`src/hardware/serialport/serialdummy.cpp`, `src/hardware/vga_other.cpp`,
`src/hardware/vga_xga.cpp`, `src/ints/bios_disk.cpp`, `src/ints/ems.cpp`,
`src/ints/mouse.cpp`, `src/misc/messages.cpp`, `src/misc/support.cpp`,
`src/shell/shell_misc.cpp`.

## New-in-r4301 file triage

Files added between 0.74 and r4301 that Boxer must explicitly accept, drop, or
stub. Cross-check `4a85221c` for prior art.

21 files added between 0.74 and r4301 under `include/` and `src/`. Cross-check
`4a85221c` for prior art on each.

| File | Mainline (`4a85221c`) | This branch | Rationale |
|------|----------------------|-------------|-----------|
| `include/midi.h` | _(tbd)_ | _(tbd)_ | New MIDI header — referenced by r4301 `src/gui/midi.cpp` rewrite |
| `include/pci_bus.h` | _(tbd)_ | _(tbd)_ | Header for new PCI bus emulation |
| `src/cpu/core_dyn_x86/risc_x64.h` | _(tbd)_ | _(tbd)_ | x86_64 dynrec backend; not used by PPC build, may be wanted for x86_64 |
| `src/cpu/core_dynrec/risc_armv8le.h` | _(tbd)_ | _(tbd)_ | ARMv8 dynrec backend; never built on a Boxer arch — drop or keep dormant |
| `src/dos/drive_overlay.cpp` | _(tbd)_ | _(tbd)_ | New overlay drive feature; user-facing, decide whether Boxer wants it |
| `src/hardware/pci_bus.cpp` | _(tbd)_ | _(tbd)_ | New PCI bus emulation |
| `src/hardware/pci_devices.h` | _(tbd)_ | _(tbd)_ | PCI device IDs |
| `src/hardware/mame/emu.h` | _(tbd)_ | _(tbd)_ | MAME compat shim for the new chip emulators below |
| `src/hardware/mame/fmopl.{cpp,h}` | _(tbd)_ | _(tbd)_ | MAME OPL2/3 — `adlib.cpp` switched to it in r4301 |
| `src/hardware/mame/saa1099.{cpp,h}` | _(tbd)_ | _(tbd)_ | MAME SAA1099 (Game Blaster / CMS) |
| `src/hardware/mame/sn76496.{cpp,h}` | _(tbd)_ | _(tbd)_ | MAME SN76496 (Tandy / PCjr) — `tandy_sound.cpp` may switch to it |
| `src/hardware/mame/ymdeltat.{cpp,h}` | _(tbd)_ | _(tbd)_ | MAME ADPCM-A helper, used by ymf262 |
| `src/hardware/mame/ymf262.{cpp,h}` | _(tbd)_ | _(tbd)_ | MAME OPL3 |
| `src/libs/zmbv/{makedll.mk,resource.rc,zmbv_mingw.def}` | n/a | drop | MinGW build artifacts, irrelevant to Xcode build |

## PPC-fix audit checklist

These fixes are load-bearing for PPC and must survive the tree replacement.
They live far from the patch hunks, so the merge can silently lose them if
upstream rewrote the surrounding code.

- [x] `6995713f` — FAT image endianness / coalface helpers preserved in
      drive_fat.cpp (six call sites). Coalface helpers themselves
      (`BXCoalfaceDrives.{h,mm}`) are in `Boxer/`, not in `DOSBox/`, and
      were not touched by this work.
- [ ] `9456f8cf` — MT-32 endianness fix
- [ ] `f4935f44` — Tandy / CGA endianness fix
- [x] `Segs::val[]` declared as `Bit16u[8]` in `include/regs.h`. **r4301
      already has the upstream fix** — confirmed by direct inspection. No
      manual carry-over needed; the include/ batch lift picks it up for free.

## Build cycle log

Each row: cycle number, what was sent, what was tested, outcome, follow-up.

| # | Sent | Tested | Outcome | Follow-up |
|---|------|--------|---------|-----------|
| _(pending)_ | | | | |

## Decisions

Append-only log of load-bearing decisions made during the work. One line each.

- 2026-05-04 — `patch-r4301.diff` at `fe7b1960` is bit-equivalent to whatever produced `~/repos/dosbox_r4301_patched`. Use the in-repo patch as authoritative.
- 2026-05-04 — Sandbox repo at `~/repos/r4301-merge-sandbox` is the canonical merge workspace. `boxer-074` overlays Boxer's `DOSBox/` on vanilla-074 *without* `--delete` so vanilla files Boxer doesn't track (Makefile.am, autogen.sh, etc.) stay in place — keeps the merge from raising spurious "deleted by us" conflicts.
- 2026-05-04 — `~/repos/dosbox_r4301`'s VERSION reads `0.74-2`; this is just trunk's unbumped string at r4301 (per `svn co -r4301 trunk`), not a tree mismatch.
- 2026-05-04 — Phase 1 commit strategy: per-subsystem commits in `boxer-ppcjit` are produced by `rsync -a --existing` from sandbox into `DOSBox/<subsystem>/`. New-in-r4301 files are deferred to their own commits when their consumers land. Intermediate commits are not individually buildable; build cycle 1 only fires after the final Phase 1 commit. This keeps `git log --oneline` legible at the cost of mid-bump compile-broken states.
- 2026-05-04 — `Segs::val[]` PPC fix already present in r4301 upstream — no manual carry-over needed.
