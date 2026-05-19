# r4301 base-bump notes

Live planning + status doc for bumping Boxer's embedded DOSBox tree from 0.74
to SVN r4301, so that jmarsh's `patch-r4301.diff` becomes a clean drop-in
against the revision it was authored for.

`PPC_JIT_NOTES.md` is the writeup of the prior JIT-on-0.74 integration. This
file is the live doc for the base-bump workstream — update it in the same turn
as any load-bearing decision. Don't defer.

## Branches

- `leopard_legacy` — the fork's **public release mainline** (local ==
  `origin/leopard_legacy`). It already carries the JIT-on-0.74 work
  (`da37579d` is reachable from it). Earlier note here ("clean Boxer
  leopard_legacy, untouched") was wrong — corrected during the v2.0.0
  merge prep. Before that merge its tip was `fe7b1960`.
- `r4301-basebump` — where Phase 0–3 happened; strict descendant of
  `leopard_legacy` (clean fast-forward, no divergence). Merged into
  `leopard_legacy` via `--no-ff` and tagged `v2.0.0`.
- `v1.0.0` — release tag at `4a454a1c`, the original JIT-on-0.74 work.
  That commit is an **orphan** (on no branch): the `leopard_legacy`
  line was rewritten at some point (`4a454a1c` and `fe7b1960` share an
  identical commit message — the rewrite signature), leaving `v1.0.0`
  pinning the pre-rewrite commit. Benign: the tag/release still
  resolves; a tag needs no branch. `v1.0.0` is immutable, untouched.
- `v2.0.0` — this project's release: r4301 base-bump + PPC JIT,
  hardware-validated, ~17% faster than the v1.0.0 JIT-on-0.74 build.

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

- [x] Resolve merge conflicts subsystem by subsystem (cpu/, dos/, hardware/,
      ints/, gui/, shell/, misc/, includes). One commit per subsystem.
      **All 73 hunks across 30 files resolved.**
- [x] Reset `core_dynrec/` and `core_dynrec.cpp` to r4301 vanilla form
      (drops the JIT-on-0.74 backport for those files). `risc_ppc.h` removed
      (Phase 2 re-creates it via `patch-r4301.diff`). `risc_armv4le-s3.h`
      removed (no longer in r4301). `risc_armv8le.h` lifted from r4301.
- [x] Disable Boxer's `__ppc__ / __ppc64__` `C_DYNREC` enable in `config.h`
      for build cycle 1. The PPC build runs normal-core only. Phase 2
      re-enables the clause alongside re-applying the JIT patch.
- [x] Triage new-in-r4301 files. Tracking table below.
- [x] Audit the load-bearing PPC fixes against the new r4301 code. Tracking
      checklist below.
- [ ] Update `Boxer.xcodeproj` to add/remove file references for the new
      tree. **Followup for the user — I can't reliably edit the project
      file blind.** Files that need adding to Compile Sources:
      - `DOSBox/src/hardware/mame/fmopl.cpp`
      - `DOSBox/src/hardware/mame/saa1099.cpp`
      - `DOSBox/src/hardware/mame/sn76496.cpp`
      - `DOSBox/src/hardware/mame/ymdeltat.cpp`
      - `DOSBox/src/hardware/mame/ymf262.cpp`
      - `DOSBox/src/dos/drive_overlay.cpp`
      Files that should *not* be referenced (they were never in Boxer's
      compile set, or are PPC-deferred): `risc_armv8le.h` (header-only,
      no compile-phase reference needed), `pci_bus.cpp` /
      `pci_devices.h` (deferred), `risc_armv4le-s3.h` (deleted).
- [x] **Build cycle 1 — PASSED on G4.** Builds, links, boots, runs games
      under the normal core. Two integration regressions found and fixed
      in-cycle (`d53a9c7d` DOS_ChangeDir trailing-backslash; `20bc7b5f`
      CMD_CHOICE quit hang), both verified fixed on the G4. Phase 1 done.

### Phase 2 — re-apply the JIT patch

- [x] Applied `patch-r4301.diff` with `patch -p0` from `DOSBox/` (patch
      paths are root-relative, not `-p1`). JIT backend files
      (`fpu.h`, `mem.h`, `risc_ppc.h` [new], `cache.h`, `decoder_basic.h`,
      `decoder_opcodes.h`, `core_dynrec.cpp`) taken **wholesale** — all
      landed clean. `cache.h` applied clean → the `PFLAG_HASCODE`
      divergence flagged in PPC_JIT_NOTES was a 0.74-vs-r4301 artifact
      the base-bump dissolved (the point of the bump). `Makefile.am`
      hunk skipped (autotools, irrelevant to the Xcode build).
- [x] Resolved `drive_fat.cpp` selectively. **A clean `patch` apply here
      is WRONG**: the patch's MBR/boot-sector/dir-entry hunks (#6–#10)
      apply cleanly but would double-swap on PPC because Boxer's
      `BXCoalfaceDrives` helpers already byteswap those structs (6 sites
      preserved in Phase 1, lines 737/781/1110/1222/1266/1313). Reverted
      `drive_fat.cpp` to Phase 1 and hand-applied **only** hunks #2–#5
      (the 7 `var_read`/`var_write` substitutions in
      `getClusterValue`/`setClusterValue` — the raw FAT-entry buffer the
      coalface never touches). Skipped #1 (unrelated EOC refactor, not in
      the PPC keep-list) and #6–#12 (coalface-owned). Matches
      `PPC_JIT_NOTES.md:43` prior art exactly.
- [x] Re-enabled the `__ppc__ || __ppc64__` `C_DYNREC` clause in
      `config.h`.
- [x] Verified `dyn_set_eip_end(HostReg,Bit32u)` has the patch's
      32-bit-load form: `gen_mov_word_to_reg(reg,&reg_eip,true)` (forced
      full-word load + `get_extend_word` mask), with the old
      `decode.big_op` form commented out. `decoder_basic.h` taken
      wholesale from the patch so this is the patch's form by
      construction; confirmed by inspection.
- [ ] **Xcode followup (user):** add `DOSBox/src/cpu/core_dynrec/risc_ppc.h`
      to the project as a **file reference only** (under the core_dynrec
      group, alongside `risc_x86.h`) — NOT a Compile Sources entry (it's
      a header). `risc_x86.h` has project refs which is why
      `#include "core_dynrec/risc_x86.h"` resolves for i386; `risc_ppc.h`
      has none, so the PPC build will fail to resolve
      `#include "core_dynrec/risc_ppc.h"` at `core_dynrec.cpp:153`
      (same failure mode as Phase 1 `midi.h`). Also `git add` it
      (currently untracked).
- [x] **Build cycle 2 — PASSED on G4.** Built clean (0 errors), JIT
      engages and runs on real hardware against the r4301 base.
      Phase 2 complete.

### Phase 3 — validation

- [x] **Phase 3 — PASSED (strict win) on the G4.** Six-title matrix
      runs under `core=dynamic`; Quake timedemo deterministic (969
      frames) across all builds. New r4301+JIT (8.7 fps) **beats** the
      JIT-on-0.74 golden build (`da37579d`, 7.4 fps) by ~17%; interpreter
      perf unchanged vs stock 0.74. No mod-replay regression found —
      performance improved. **Project complete: base-bump + JIT
      migration validated end to end.**

## Regression matrix & test methodology

Standing, fixed regression set. Run the **identical** matrix every build
cycle so results compare cycle-over-cycle. This is the floor, not the
ceiling — keep doing exploratory passes on top; a fixed list proves
"no regression vs last cycle" but can't find novel bugs.

Each title is pinned to the risk axis it exercises (chosen to cover what
the base-bump actually changed, not generic coverage).

| Title | Covers | Phase 1 (normal core) |
|-------|--------|-----------------------|
| Epic Pinball (demo) | GUS + SB16, fast real-mode CPU | ✅ G4-confirmed |
| Commander Keen | Adlib (new MAME `ymf262`) + PC Speaker, EGA timing/scroll | ✅ G4-confirmed |
| PC Player Benchmark | DOS/4GW protected-mode JIT path, deterministic number | ✅ G4-confirmed |
| Quake timedemo (DOSBENCH) | FPU path, **deterministic fps number** | ✅ G4-confirmed (record fps as JIT baseline) |
| Round 42 | CGA tweaked-mode → PPC big-endian `vga_draw.cpp` `CFSwapInt32HostToLittle` fix. Requires `machine=cga` in the gamebox `.conf` or the path isn't exercised. | ✅ G4-confirmed (renders correctly on PPC; wrong endianness can't accidentally produce correct colour) |
| Star Trek 25th Anniversary | CD-ROM → `cdrom_image.cpp` / `drive_iso.cpp` / cdromDrive ctor / MSCDEX | ✅ G4-confirmed |

**Phase 1 fully signed off — all six axes hardware-verified under the
normal core.**

### Differential rule (bug classifier)

When something breaks in Phase 2+ (`core=dynamic`), **first reproduce it
with `core=normal` on the same build**:

- Repros on normal core → base-bump bug that slipped Phase 1.
- Only on dynamic → JIT-side.

This stops JIT ghosts that are actually integration bugs (and vice
versa). Commit `43385c29` is the "normal core on r4301" reference point.

### Quake timedemo numbers (deterministic — 969-frame "THE NECROPOLIS" demo)

| Build | Core | fps | seconds |
|-------|------|-----|---------|
| stock leopard_legacy 0.74, no JIT | normal | 3.9 | 246.9 |
| `9fa7bc15` (Phase 2, r4301+JIT) | normal | 3.9 | 245.4 |
| `da37579d` (JIT-on-0.74 golden) | dynamic | 7.4 | 129.9 |
| `9fa7bc15` (Phase 2, r4301+JIT) | dynamic | **8.7** | **111.3** |

Same 969 frames + identical pickup log across **all four** runs ⇒ JIT
functionally deterministic everywhere (no desync; correct FPU/integer
results). Base-bump did NOT cost interpreter perf (new normal core 3.9
≈ stock 0.74 no-JIT 3.9, <1% noise) — base stable, JIT the only
variable. **Phase 3 headline: new r4301+JIT (8.7) beats the JIT-on-0.74
golden build (7.4) by ~17%.** Speedup over interpreter: 1.9× (old) →
2.2× (new). The migration improved JIT performance while preserving
correctness — strict win, not just no-regression.

### Reference baselines

- `43385c29` — Phase 1, r4301 normal-core, hardware-passed. Base-bump
  reference.
- `da37579d` (old `r4301` branch) — JIT-on-0.74 golden build. Phase 3
  compares against it; anything that regresses vs it is a mod-replay
  miss. **Do not overwrite/delete that binary.** For Quake, "same fps"
  is the JIT-correctness signal, not "ran fine".

### Hang procedure

Any hang: `sample <pid> 10 -file out.txt` the wedged process **first**,
before theorising. This cycle it collapsed a multi-layer speculative
trace into a one-line `CMD_CHOICE` fix. The emulation thread is the one
with DOSBox frames (`Normal_Loop`, `DOS_Shell::*`, `CALLBACK_*`).

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

### src/cpu/ (1 file, 2 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/cpu/core_dyn_x86/dyn_fpu_dh.h` | 2 | take r4301 | r4301 dropped two dead `//Bitu group=...` comments. Pure cleanup. |

**Scope note:** the `src/cpu/` lift deliberately excludes `core_dynrec/` and
`core_dynrec.cpp`. Those carry the JIT-on-0.74 backport from `da37579d` and
will be reset wholesale when Phase 2 re-applies `patch-r4301.diff`. Touching
them now would scramble the per-commit story.

### src/dos/ (8 files, 25 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/dos/cdrom_image.cpp` | 1 | take r4301 | r4301 added a member-initializer list (`:subUnit(subUnit)`) and renamed param from `_subUnit` to `subUnit`. Auto-merge took Boxer's `images[_subUnit] = this;` body line, leaving an inconsistent param/body name pair. Resolved by taking r4301's signature and fixing the body to `images[subUnit]` to match. |
| `src/dos/dos_memory.cpp` | 2 | keep Boxer | Boxer made `callbackhandler` a function-local instead of a static global, to avoid `CALLBACK_HandlerObject.Allocate` failing with "already-installed" on shutdown-and-restart. Keep both Boxer hunks (the comment block above and the local declaration). |
| `src/dos/dos_programs.cpp` | 8 | merge | H1/H7 (MOUNT/IMGMOUNT `-u` unmount): keep Boxer's inline form (preserves `boxer_driveDidUnmount(i_drive)`); skip r4301's `UnmountHelper(...)` refactor since it would drop the boxer notification. H2: take both — r4301's `path_relative_to_last_config` resolution **and** Boxer's `is_physfs` detection. H3: `if (!is_physfs && !S_ISDIR(test.st_mode))` — Boxer's physfs guard with r4301's macro. H4: combine — `if (type == "overlay") { ...Overlay_Drive... } else if (is_physfs) { ...physfsDrive... } else { ...localDrive... }`. Requires `getBasedir()` from drives.h batch (already in) and accepting `drive_overlay.cpp`. H5: take r4301 (`dirCache.SetLabel(...)` direct access). H6: take both — r4301's `incrementFDD()` and Boxer's `boxer_driveDidMount(...)`. H8: keep Boxer's `boxer_driveDidMount(...)` call; take r4301's spelling fix (`be careful` for `becareful`). |
| `src/dos/drive_cache.cpp` | 3 | merge | H1: `SetBaseDir(basePath, drive)` — the merged `dos_system.h` declares the 2-arg version, so use that. Drop `free[i] = true` loop (the `free[]` member is no longer in DOS_Drive_Cache). H2: take Boxer (matches Boxer's 3-arg `GetShortName(dirpath, filename, shortname)` signature, which the auto-merge already kept above the conflict). Simplified Boxer's body slightly (removed the printf debug lines and the dead binary-search comment block). H3: take Boxer (`drive->opendir(...)` / `drive->closedir(...)` matches the merged DOS_Drive_Cache API which has a `drive` member for dispatching directory access through the drive's overrides — required for `physfsDrive`). |
| `src/dos/drive_fat.cpp` | 3 | merge ✓ | (covered in earlier commit `8cce05c1`) |
| `src/dos/drive_iso.cpp` | 2 | merge | Same auto-merge param-name issue as cdrom_image.cpp. H1: keep Boxer's signature (`letter`, `name`, `_mediaid`) so the body compiles unchanged, but adopt r4301's member-initializer list (initializes `iso/dataCD/mediaid/subUnit/driveLetter` early). H2: keep Boxer's signature (`letter`, `_subUnit`); add r4301's missing line `_subUnit = MSCDEX_GetSubUnit(letter);` — that's a real bug fix Boxer was missing (without it, `UpdateMscdex` for an existing drive uses whatever subUnit the caller passed instead of the actual current subUnit). |
| `src/dos/drive_local.cpp` | 5 | merge | H1: take r4301 — drop the inline `class localFile` declaration (r4301 moved it to `include/dos_system.h`, which is already in via the merged header). H2: take both — Boxer's `boxer_shouldAllowWriteAccessToPath` permission check **and** r4301's `fopen_wrap` helper (instead of plain `fopen`). H3: take both — Boxer's permission check, then r4301's "flush handles" block (Betrayal in Antara fix), then `fopen_wrap`. H4: take both methods — `localFile::Flush()` from r4301 and `localFile::willBecomeUnavailable()` from Boxer (independent additions). H5: keep Boxer's signature (`letter` param) so the body compiles; adopt r4301's member-initializer list (`subUnit(0), driveLetter('\0')`). |
| `src/dos/drives.h` | 1 | merge | r4301 moved `GetLabel/SetLabel/EmptyCache` to the `DOS_Drive` base class in `include/dos_system.h` (already in via the merged header) and added `getBasedir()`. Boxer kept those overrides on `localDrive` plus added `opendir/closedir/read_directory_first/read_directory_next` (needed for `physfsDrive` polymorphic override) and `getShortName`. Keep Boxer's full set of `localDrive` declarations and add r4301's `getBasedir()` accessor. |

### src/ (root, 1 file, 1 hunk)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/dosbox.cpp` | 1 | take r4301 | r4301 expanded the `midiconfig` `Set_help` text (mentions `find the id/name with mixer/listmidi`). Pure docstring improvement. |

### src/gui/ (1 file, 8 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/gui/midi.cpp` | 8 | merge | Boxer disables all DOSBox-internal MIDI handling and routes through `BXCoalfaceAudio` (`boxer_sendMIDIMessage`, `boxer_sendMIDISysex`, `boxer_suggestMIDIHandler`). H1: keep Boxer's `#include "BXCoalfaceAudio.h"`. H2: keep Boxer's commented-out MIDI driver includes (BXCoalface replaces them entirely). H3: take r4301's `DB_Midi midi;` global (the struct definition moved to the new `include/midi.h`). H4: take r4301 (whitespace). H5: keep Boxer (`boxer_sendMIDISysex` instead of `midi.handler->PlaySysex`). H6: keep Boxer's "Colonel's Bequest" sysex-delay-clamp fix (`if (midi.sysex.delay < 40) midi.sysex.delay = 40`). H7: take r4301 (`static_cast<int>` for the printf format). H8: keep Boxer's `boxer_suggestMIDIHandler` call, the disabled-sysex-delay-parsing block, and the `goto getdefault` short-circuit. Adopting `include/midi.h` is mandatory (defines `DB_Midi`). |

### src/hardware/ (12 files, 28 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/hardware/adlib.cpp` | 1 | take r4301 | mixerChan scale 2.0 → 1.5 with explanatory comment ("measured to be too high"). r4301 already references `MAMEOPL2/3::Handler` for `oplemu="mame"`, which makes accepting the new `mame/fmopl.{cpp,h}` and `mame/ymf262.{cpp,h}` files mandatory. |
| `src/hardware/dbopl.cpp` | 1 | take r4301 | r4301 simplified the rounding correction to a single `guessAdd++`. |
| `src/hardware/dma.cpp` | 3 | take r4301 | H1: `dma_wrapping` becomes `static` (matches `include/dma.h` change — public API is now `DMA_SetWrapping(Bitu)`). H2/H3: `sBitfs(x)` size format macro for 64-bit-safe `Bitu` printing. |
| `src/hardware/gus.cpp` | 1 | **keep Boxer** | Boxer added a substantial `pantable[]` fix (~25-line block with comment): the upstream 0.74 code generated a panning table unrelated to the actual GUS panning register, locking GUS programs to mono. r4301 still has the broken upstream form. Boxer's fix is real and intentional — keep it. |
| `src/hardware/joystick.cpp` | 3 | merge | H1: take r4301 (drop stray `// Store writetime index` comment). H2: take r4301 (whitespace alignment). H3: **keep Boxer** — Boxer commented out the static `if(timed)` handler-install branch and replaced with `gameport_timed = ...; ReadHandler.Install(0x201, read_p201_switchable, IO_MB);` so Boxer can toggle gameport timing at runtime. Also keeps Boxer's disabled `stick[0/1].enabled = false` init (Boxer sets these elsewhere). |
| `src/hardware/mixer.cpp` | 2 | merge | H1: **keep Boxer** — `ShowVolume("MASTER", boxer_masterVolume(BXLeftChannel), boxer_masterVolume(BXRightChannel))` routes the master-volume display through Boxer's OS X mixer instead of `mixer.mastervol[]`. H2: take r4301 (`(Bit32u)obtained.freq` cast for 64-bit safety). |
| `src/hardware/pcspeaker.cpp` | 1 | take r4301 | `fabsf(...)` instead of `(float)(fabs(...))` — avoids double promotion. |
| `src/hardware/serialport/nullmodem.cpp` | 2 | take r4301 | H1: inner-shadow `Bits rxchar` declaration (semantically equivalent on this scope). H2: `control` instead of Boxer's `_control` — original upstream form; Boxer's `_` prefix was unconventional. |
| `src/hardware/serialport/softmodem.cpp` | 1 | take r4301 | Whitespace tweak in a `while` loop. |
| `src/hardware/tandy_sound.cpp` | 3 | take r4301 | H1: r4301 deleted ~70 lines of inline SN76496 register-handling code in `SN76496Write`; the SN76496 emulation moved to MAME's `mame/sn76496.{cpp,h}` (`device.write(data)` on line 78 dispatches to it). **Adopting `mame/sn76496.{cpp,h}` is mandatory** — without them this file won't link. H2/H3: whitespace (`data & 0xff` vs `data&0xff`). PPC_JIT_NOTES had this file flagged as the home of the `f4935f44` PPC fix; that was incorrect — the f4935f44 fix is in `vga_draw.cpp`, not here. |
| `src/hardware/vga_draw.cpp` | 4 | merge | **load-bearing — `f4935f44` PPC fix lives here.** H1 (CGA composite output): take r4301's improved algorithm but keep Boxer's `CFSwapInt32HostToLittle(...)` wrapper — r4301 still emits a packed `Bit32u` to TempLine, so big-endian hosts need the byteswap. H2/H3 (4BPP_Line and 4BPP_Line_Double): take r4301 entirely — r4301 refactored these from packed `Bit32u` writes to byte-by-byte `Bit8u` writes, which obsoletes the PPC fix in those two functions (byte stores are endianness-agnostic). H4 (text-mode cursor in `VGA_TEXT_Xlat16_Draw_Line`): take r4301's cleaner `if (...) { ... }` form (no `goto skip_cursor`/`font_addr`). |

**Followup for the user:** the new files under `DOSBox/src/hardware/mame/`
need to be added to `Boxer.xcodeproj`'s Compile Sources build phase
(`fmopl.cpp`, `saa1099.cpp`, `sn76496.cpp`, `ymdeltat.cpp`, `ymf262.cpp`).
Without them `adlib.cpp`/`tandy_sound.cpp`/`gameblaster.cpp` won't link.
I cannot reliably edit the project file blind; flagging here for the
build-cycle 1 prep.

### src/ints/ (2 files, 2 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/ints/bios_keyboard.cpp` | 1 | keep Boxer | Boxer commented out the `#if SDL_VERSION_ATLEAST(1, 2, 14)` block and unconditionally `#define CAN_USE_LOCK 1` because Boxer doesn't use SDL. r4301's bare `#endif` would close a `#if` that no longer exists. Drop r4301's `#endif` and the unused-on-Boxer comment about lower-SDL-version handling. |
| `src/ints/int10_char.cpp` | 1 | take r4301 | r4301 has a clearer comment about mode 6 vs INT 10h fn 09h. Same logic, better wording. |

### src/misc/ (1 file, 3 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/misc/setup.cpp` | 3 | take r4301 | All three hunks are r4301 cleanups. H1: `Bitu val` → `Bit32u value` (64-bit safety: `%u` matches uint32, not Bitu which can be 64-bit). H2/H3: whitespace tweaks in the help-text formatter. |

### src/shell/ (2 files, 6 hunks)  ✓

| File | Hunks | Decision | Rationale |
|------|-------|----------|-----------|
| `src/shell/shell.cpp` | 3 | merge | H1 (/INIT): keep Boxer's `boxer_autoexecDidStart()` / `boxer_autoexecDidFinish()` hooks; drop r4301's startup-banner branch (Boxer doesn't show DOSBox's welcome text — surrounding Boxer UI handles user-facing notification). H2/H3 (cmdline mount logic): structurally divergent — r4301 added a `while (FindCommand(dummy++, line) && !command_found)` loop with `continue` for retries, Boxer had a single-shot `if (FindCommand(1, line))` with `goto nomount`. Resolved by taking r4301's loop structure and threading Boxer's physfs-source check (`line.find(':')` for `archive.zip:internal/path` paths) into the top of the loop body, plus moving Boxer's `.ZIP`/`.7Z` PHYSFS detection from a `goto nomount` to `command_found = true; continue;`. The `nomount:` label is dropped entirely (no remaining `goto`). |
| `src/shell/shell_cmds.cpp` | 3 | merge | H1 (CD): keep Boxer's DWIM behavior — when given `cd D:\path`, Boxer changes to drive D first then `cd` to the path. r4301's "drive not found / illegal path" hint UX is dropped (Boxer doesn't ship the new `SHELL_EXECUTE_DRIVE_NOT_FOUND` / `SHELL_CMD_CHDIR_HINT` strings anyway). H2 (DIR): take both — r4301's new sort flags (`/ON`, `/OD`, `/OE`, `/OS`, `/A-D` + `reverseSort`) **and** Boxer's commented-out unrecognised-switch-bailout (preserves unix/style/paths support). H3 (COPY): keep Boxer's `*/` closing the comment block opened earlier in the function; take r4301's slightly cleaner `Gather all sources` comment text. |

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
| `include/midi.h` | _(tbd)_ | accept ✓ | Defines `DB_Midi` struct (which `src/gui/midi.cpp` H3 declares as `extern DB_Midi midi;`). Mandatory. |
| `include/pci_bus.h` | _(tbd)_ | defer | Boxer's hardware/ tree didn't reference PCI symbols — checked by grep. Phase 1 doesn't need it. If Phase 2 / build cycle 1 reveals a missing reference, bring in alongside `pci_bus.cpp` and `pci_devices.h`. |
| `src/cpu/core_dyn_x86/risc_x64.h` | _(tbd)_ | defer | x86_64 dynrec backend; only used when building for x86_64. Boxer's Legacy Release config builds i386 + ppc + (x86_64?) — verify during build cycle 1. If the x86_64 arch needs it, lift it then. |
| `src/cpu/core_dynrec/risc_armv8le.h` | _(tbd)_ | defer | ARMv8 dynrec backend; never built on any Boxer arch. Skip. |
| `src/dos/drive_overlay.cpp` | _(tbd)_ | accept ✓ | r4301's `Overlay_Drive` class is referenced from `dos_programs.cpp` H4 (`MOUNT -t overlay`). Either accept or strip the overlay branch. Accepted to preserve r4301's feature surface — Boxer users can ignore it but it shouldn't be cut. Needs adding to `Boxer.xcodeproj`. |
| `src/hardware/pci_bus.cpp` | _(tbd)_ | defer | No references in Boxer's hardware/ tree (grep clean). Skip until needed. |
| `src/hardware/pci_devices.h` | _(tbd)_ | defer | Same as above. |
| `src/hardware/mame/emu.h` | _(tbd)_ | accept ✓ | Header-only compat shim required by all the chip emulators below. Brought in with the hardware/ batch. |
| `src/hardware/mame/fmopl.{cpp,h}` | _(tbd)_ | accept ✓ | MAME OPL2/3 — `adlib.cpp` references `MAMEOPL2/3::Handler`. Mandatory for link. |
| `src/hardware/mame/saa1099.{cpp,h}` | _(tbd)_ | accept ✓ | MAME SAA1099 (Game Blaster / CMS) — `gameblaster.cpp` likely switched to it. Brought in for completeness; verify use during build cycle 1. |
| `src/hardware/mame/sn76496.{cpp,h}` | _(tbd)_ | accept ✓ | MAME SN76496 (Tandy / PCjr) — `tandy_sound.cpp` H1 swap requires it. Mandatory for link. |
| `src/hardware/mame/ymdeltat.{cpp,h}` | _(tbd)_ | accept ✓ | MAME ADPCM-A helper used by `ymf262`. Mandatory for link. |
| `src/hardware/mame/ymf262.{cpp,h}` | _(tbd)_ | accept ✓ | MAME OPL3 — `adlib.cpp` references it via `MAMEOPL3`. Mandatory for link. |
| `src/libs/zmbv/{makedll.mk,resource.rc,zmbv_mingw.def}` | n/a | drop | MinGW build artifacts, irrelevant to Xcode build |

## PPC-fix audit checklist

These fixes are load-bearing for PPC and must survive the tree replacement.
They live far from the patch hunks, so the merge can silently lose them if
upstream rewrote the surrounding code.

- [x] `6995713f` — FAT image endianness / coalface helpers preserved in
      drive_fat.cpp (six call sites). Coalface helpers themselves
      (`BXCoalfaceDrives.{h,mm}`) are in `Boxer/`, not in `DOSBox/`, and
      were not touched by this work.
- [x] `9456f8cf` — MT-32 endianness fix lived in `DOSBox/src/gui/midi_mt32.h`,
      which Boxer subsequently **deleted** at `f5662d29` ("Reverted MIDI handler
      source files to DOSBox 0.74-spec now that they're no longer used by Boxer
      at all" — Boxer moved MT-32 to the MT32Emu/Munt framework on the
      Boxer-side). The fix is dead code in the leopard_legacy branch and does
      not need to survive the base-bump. No carry-over needed.
- [x] `f4935f44` — Tandy / CGA pixel-column flip fix lives in
      `src/hardware/vga_draw.cpp` (commit message says "Tandy and CGA" but the
      fix is in vga_draw, not tandy_sound). Carried over selectively in the
      hardware/ batch: the CGA composite hunk (H1) keeps Boxer's
      `CFSwapInt32HostToLittle(...)` wrapper around r4301's improved algorithm
      (still emits packed `Bit32u`, so still needs the byteswap on PPC). The
      4BPP_Line and 4BPP_Line_Double hunks (H2/H3) take r4301 entirely —
      r4301 refactored those from packed `Bit32u` writes to byte-by-byte
      `Bit8u` writes, which obsoletes the PPC fix in those two functions.
- [x] `Segs::val[]` declared as `Bit16u[8]` in `include/regs.h`. **r4301
      already has the upstream fix** — confirmed by direct inspection. No
      manual carry-over needed; the include/ batch lift picks it up for free.

## Build cycle log

Each row: cycle number, what was sent, what was tested, outcome, follow-up.

| # | Sent | Tested | Outcome | Follow-up |
|---|------|--------|---------|-----------|
| 1 (attempt 1) | commits up to `5223159f` (Phase 1 complete + JIT-disable) | `xcodebuild -configuration "Legacy Release" -target Boxer ARCHS=ppc` | **failed**: 69 errors in 8 root-cause categories | See "Build cycle 1 attempt 1 — fixes" section below. Re-sync, rebuild. |
| 1 (attempt 2) | commits up to `9ad2fec9` (8 fixes + Xcode adds) | same | **failed at link**: 1 undefined symbol — `restart_program(std::vector<std::string>&)` referenced from `CONFIG::Run()` in programs.o. r4301 added the call in `programs.cpp`; the *definition* lives in `sdlmain.cpp` which Boxer doesn't compile. Compile stage clean. | Stubbed `restart_program` as a `static` no-op in `programs.cpp` itself, since Boxer's CONFIG -restart path is unreachable (Cocoa owns the runloop). Re-sync, rebuild. |
| 1 (attempt 3) | commits up to `bd067e82` (restart_program stub) | cold boot + run games on G4 | **builds, links, boots, runs games** under normal core. One regression: launch-panel targets in a subdirectory fail — shell stays at `C:\>` and the bare program name is not found. Manual `cd <dir>` then run works. | Root-caused to r4301's new trailing-backslash rejection in `DOS_ChangeDir` (auto-merge bug — `dos_files.cpp` was never in the conflict queue). Boxer's launch always passes a trailing-backslash dir path. Removed the rejection. Re-sync, rebuild, retest DOSBENCH subdir launch. |
| 1 (attempt 4) | commit `d53a9c7d` (DOS_ChangeDir fix) | DOSBENCH subdir launch + quit on G4 | **subdir launch fixed.** New regression: quitting while DOSBENCH.BAT's `CHOICE` menu is waiting hangs the app (window closes, must force-quit). `sample` of pid 278 showed the emulation thread in a 100% busy spin: `CMD_CHOICE → DOS_ReadFile → device_CON::Read → CALLBACK_RunRealInt → DOSBOX_RunMachine → Normal_Loop → boxer_runLoopShouldContinue`. Confirmed CHOICE-specific (answering the prompt lets quit work normally). | Internal `DOS_Shell::CMD_CHOICE` raw key-read loop never checks the shell `exit` flag, so Boxer's cancel (`shell->exit=YES`) can't break it. Added `!exit` to the loop condition + early `return` on exit. Re-sync, rebuild, retest quit-during-CHOICE. |
| 1 (attempt 5) | commit `20bc7b5f` (CMD_CHOICE fix) | quit-during-CHOICE on G4 | **PASSED.** Quit-during-CHOICE no longer hangs; subdir launch and normal CHOICE answering still work. Build cycle 1 complete — Phase 1 done. | Proceed to Phase 2 (re-apply `patch-r4301.diff`, re-enable PPC `C_DYNREC`). |
| 2 (attempt 1) | commits up to `9fa7bc15` (Phase 2 JIT patch + risc_ppc.h Xcode ref) | compile/link only | **BUILD SUCCEEDED.** 0 compile errors, 0 link errors, `-arch ppc`. `risc_ppc.h` resolved (Xcode file-ref worked); the 907-line PPC backend compiled clean as part of `core_dynrec.cpp`. 238 warnings (expected r4301 `-Wshadow` noise). JIT backend dropped in with zero manual code fixes — the base-bump payoff. | Run the six-title matrix on the G4 with `core=dynamic`. Differential rule in force. Bank Quake fps vs Phase 1. |
| 2 (G4 test) | same build (`9fa7bc15`) | matrix with `core=dynamic` on G4 | **PASSED — "Everything works. JIT engages."** Phase 2 complete: the PPC JIT runs on real hardware against the r4301 base. The full base-bump + JIT re-application is hardware-validated. | Phase 3: regression sweep vs the JIT-on-0.74 golden build (`da37579d`). Capture Quake `core=dynamic` fps vs the Phase 1 baseline for the JIT-correctness check. |

## Build cycle 1 attempt 1 — fixes

69 errors collapsed into 8 root causes:

1. **`sBitfs` macro undefined** (12 errors in cpu.cpp/dma.cpp/callback.cpp). r4301 uses `sBitfs(x)` printf-format-string helper for `Bitu` (32-or-64-bit). Defined by autoconf via `acinclude.m4` upstream; Boxer has a hand-written `config.h` and was missing it. Added to `config.h`: `sBit32fs(a) #a`, `sBit64fs(a) "ll" #a`, `sBitfs` switches on `__LP64__`.

2. **`midi.h: No such file or directory`** (cascade of 14 errors in midi.cpp + mixer.cpp). The header is on disk but Xcode 3's header maps only index files referenced from the project. **User followup: add `DOSBox/include/midi.h` to the Xcode project under the DOSBox/include group.**

3. **`pci_bus.h: No such file or directory`** (2 errors in dosbox.cpp + bios.cpp). r4301 added unconditional `#include "pci_bus.h"` to both files; the actual PCI calls are gated by `PCI_FUNCTIONALITY_ENABLED` which Boxer doesn't define, but the include itself runs unconditionally. Lifted `include/pci_bus.h` from r4301 (no .cpp needed since nothing references PCI symbols without the gate). **User followup: add `DOSBox/include/pci_bus.h` to the Xcode project.**

4. **`'id' Objective-C collision in CFileInfo`** (`dos_system.h:187: expected unqualified-id before '=' token`). r4301 added `Bit16u id;` to `CFileInfo`, which collides with the Objective-C `id` typedef pulled in transitively via Boxer's `BXCoalface.h` (included from `dosbox.h`, included from `dos_system.h`). Renamed `CFileInfo::id` → `CFileInfo::cacheID` and updated all 13 call sites in `drive_cache.cpp` (sed pattern `s/(dir|dirSearch\[id\])->id\b/\1->cacheID/g` — local `id` loop counters left alone).

5. **`'CALLBACK_HandlerObject' / 'CBRET_NONE' / 'lastint' not declared` in dos_memory.cpp**. Boxer's `DOS_default_handler` and the function-local `callbackhandler` use callback APIs but the merged file doesn't `#include "callback.h"`. Added the include.

6. **`localFile::willBecomeUnavailable` not declared** (drive_local.cpp:709). When we dropped the inline `class localFile` declaration from drive_local.cpp (it moved to `dos_system.h` in r4301), the override declaration for Boxer's `willBecomeUnavailable` went with it. Added the override declaration to `localFile` in `dos_system.h`. (Base class `DOS_File` already has `virtual void willBecomeUnavailable() { }` from auto-merge.)

7. **`localDrive::allocation is private`** (drives.h:105 cascade in drive_physfs.cpp). The merged `drives.h` has `private:` before the `allocation` struct — r4301 made it private; Boxer needs it `protected:` so `physfsDrive` (a subclass) can read its fields. Changed to `protected:`.

8. **`GetShortName(char[512], char[512])` no matching call** (drive_overlay.cpp:371). r4301's `drive_overlay.cpp` calls the upstream 2-arg `GetShortName(fullname, shortname)`, but Boxer changed `DOS_Drive_Cache::GetShortName` to a 3-arg form `(dirpath, filename, shortname)`. Added a 2-arg backward-compat overload in `dos_system.h` + implementation in `drive_cache.cpp` that splits `fullname` at the last `CROSS_FILESPLIT` and dispatches to the 3-arg form.

## Decisions

Append-only log of load-bearing decisions made during the work. One line each.

- 2026-05-04 — `patch-r4301.diff` at `fe7b1960` is bit-equivalent to whatever produced `~/repos/dosbox_r4301_patched`. Use the in-repo patch as authoritative.
- 2026-05-04 — Sandbox repo at `~/repos/r4301-merge-sandbox` is the canonical merge workspace. `boxer-074` overlays Boxer's `DOSBox/` on vanilla-074 *without* `--delete` so vanilla files Boxer doesn't track (Makefile.am, autogen.sh, etc.) stay in place — keeps the merge from raising spurious "deleted by us" conflicts.
- 2026-05-04 — `~/repos/dosbox_r4301`'s VERSION reads `0.74-2`; this is just trunk's unbumped string at r4301 (per `svn co -r4301 trunk`), not a tree mismatch.
- 2026-05-04 — Phase 1 commit strategy: per-subsystem commits in `boxer-ppcjit` are produced by `rsync -a --existing` from sandbox into `DOSBox/<subsystem>/`. New-in-r4301 files are deferred to their own commits when their consumers land. Intermediate commits are not individually buildable; build cycle 1 only fires after the final Phase 1 commit. This keeps `git log --oneline` legible at the cost of mid-bump compile-broken states.
- 2026-05-04 — `Segs::val[]` PPC fix already present in r4301 upstream — no manual carry-over needed.
- 2026-05-17 — `dos_files.cpp` auto-merged silently and took r4301's stricter `DOS_ChangeDir`, which hard-rejects any path ending in `\`. Boxer's launch (`BXEmulator+BXShell.mm -executeProgramAtPath:`) always passes a trailing-backslash dir, so every subdirectory launch-panel target regressed. Removed r4301's trailing-backslash rejection (0.74-compatible and DOS-accurate — real DOS accepts `cd c:\foo\`). Lesson: the "auto-merged, worth a glance" list was not exhaustive — `dos_files.cpp` wasn't even on it; a clean auto-merge can still take an upstream behavior change that breaks a Boxer contract.
- 2026-05-17 — Phase 2 patch apply: `patch -p0` from `DOSBox/` (not `-p1` — patch paths are source-root-relative). JIT backend files taken wholesale (all clean); `Makefile.am` skipped. `cache.h` applied clean, retiring the PPC_JIT_NOTES `PFLAG_HASCODE` concern (0.74-vs-r4301 artifact dissolved by the base-bump).
- 2026-05-17 — Phase 2 `drive_fat.cpp` is the one place a clean patch apply is actively wrong. Patch hunks #6–#10 (MBR/boot-sector/dir-entry `var_read`/`var_write`) apply cleanly but Boxer's `BXCoalfaceDrives` already byteswaps those structs → applying = double-swap → silent FAT-image corruption on PPC. Resolution: revert `drive_fat.cpp` to Phase 1, hand-apply only hunks #2–#5 (FAT-entry buffer, coalface-untouched). Skipped #1 (unrelated EOC refactor). This is the central PPC-correctness call of Phase 2; matches `PPC_JIT_NOTES.md:43`.
- 2026-05-17 — `risc_ppc.h` needs an Xcode **file reference** (not Compile Sources) for the PPC header map to resolve `#include "core_dynrec/risc_ppc.h"`, by direct analogy to `risc_x86.h` (project-referenced, resolves for i386). PPC_JIT_NOTES' "build-phase entry may not be necessary" is correct only about Compile Sources; a file reference IS required. User followup (can't edit pbxproj blind).
- 2026-05-17 — `DOS_Shell::CMD_CHOICE`'s raw `DOS_ReadFile(STDIN)` key-read loop never checked the shell `exit` flag. Boxer cancels a shell by setting `shell->exit=YES` (BXEmulator -cancel) and relies on read loops honoring it — the normal `InputCommand` does, via `boxer_handleCommandInput`. CMD_CHOICE bypassed all of that, so quitting while a CHOICE prompt was waiting (DOSBENCH.BAT menu) spun the emulation thread forever. Added `!exit` to the loop condition + early return. Diagnosed from a `sample` of the hung pid — runtime evidence collapsed a deep speculative trace into a one-line root cause. Lesson: any internal shell command with its own blocking read (not just InputCommand) must honor `exit` for Boxer's quit to work; CMD_CHOICE is fixed, but audit similar raw-read commands if more hang-on-quit reports appear.
