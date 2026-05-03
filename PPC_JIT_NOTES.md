# Boxer PPC JIT integration notes

This file documents the integration of a third-party DOSBox PowerPC dynrec backend (`patch-r4301.diff` against vanilla DOSBox SVN r4301) into Boxer's older `leopard_legacy` DOSBox tree, the divergences that mattered, and the one bug that prevented the JIT from running until it was found.

The work was done with substantial help from an LLM (Claude). Most of the hours went into back-and-forth diagnosis with debug instrumentation; the actual patches are small. This document is a writeup of the result, not a narrative of the process. The bug described below was found after several false starts (including one wrong "smoking gun" claim earlier in the session); take the confidence level of the conclusions accordingly and verify against the source before relying on them.

These notes assume basic familiarity with DOSBox's two recompiler cores (`core_dyn_x86`, which is x86-only inline-asm, and `core_dynrec`, which is the portable recompiler with per-architecture backends in `src/cpu/core_dynrec/risc_*.h`), and with DOSBox's BIOS-callback machinery (opcodes `FE 38 NN MM`).

## Goal

Boxer's existing PPC build had a working DOSBox tree but no recompiler — it ran games using the slow `normal` core. There is an old third-party DOSBox patch, `patch-r4301.diff` (jmarsh's PPC backend, originally posted on Vogons and packaged with binaries on SourceForge by Dr. Cameron Kaiser), which adds a `risc_ppc.h` backend to `core_dynrec`. Vanilla DOSBox SVN at r4301 plus that patch builds and runs correctly on OS X 10.4 PPC.

Boxer's DOSBox tree is older than r4301 and has a number of Boxer-specific modifications, so the patch does not apply cleanly. The work was to merge the two carefully and get the JIT working.

## Reference checkouts (external to this repo)

The integration work relied on two reference trees kept outside this repo. If you are continuing this work, set up equivalents and tell Claude where they are:

- A vanilla DOSBox SVN r4301 checkout — ground truth for what the patch's base looks like.
- The same checkout with `patch-r4301.diff` applied and built — ground truth for what the patch produces when it goes on cleanly. This is also the "known-good Mac G4/G5 build at r4301" referenced elsewhere.

When debugging a divergence between Boxer's tree and the patch, compare against both: vanilla shows the pre-patch state, patched shows the intended post-patch state, and Boxer's tree is whatever fell out of merging the patch into older, modified code.

## Files touched by the patch (against vanilla)

- `include/fpu.h` — adds a tag name `FPU_rec` to the `typedef struct {} FPU_rec;` so it can be forward-declared.
- `include/mem.h` — reorganizes the `host_read*`/`host_write*` helpers, adds a `__builtin_bswap16/32` fast path for big-endian + GCC ≥ 4.3, and adds two `var_read` overloads (sister functions to the existing `var_write`).
- `src/dos/drive_fat.cpp` — replaces raw `*(Bit{16,32}u *)…` byte-buffer accesses on FAT-image data with `var_read` / `var_write` so the FAT image bytes (which are little-endian on disk) are byte-swapped on big-endian hosts. Also adds a `copyDirEntry` helper and byte-swaps MBR / boot-sector / dir-entry fields after reading them.
- `src/cpu/core_dynrec/risc_ppc.h` — **new file**, ~907 lines: the PPC code generator (instruction encodings, the entry/epilog trampoline, the lazy-flag inline fast paths in `gen_fill_function_ptr`, the icache-flush hook, etc.).
- `src/cpu/core_dynrec/cache.h` — three changes:
  - The `write_map` byte-counter dereferences are switched from `host_read{b,w,d}(&write_map[…])` to raw `*(Bit{8,16,32}u *)&write_map[…]`. `write_map` is host-endian counter data, not little-endian guest data, so the host-endian-aware helpers were the wrong tool here.
  - A few `#if defined(WORDS_BIGENDIAN) || !defined(C_UNALIGNED_MEMORY)` are simplified to `#if !defined(C_UNALIGNED_MEMORY)` for `invalidation_map` writes (now safe on big-endian because of the previous change).
  - `cache_init` is rearranged: the `runcode` trampoline is emitted at the start of the link-blocks page, the two link-block stubs go at fixed offsets near the end (`PAGESIZE_TEMP-64` and `PAGESIZE_TEMP-32`), and after each one is written the new hook pair `cache_block_before_close()` / `cache_block_closing(start, size)` is called. On PPC those defines call `sys_icache_invalidate` (or the inline `dcbst`/`icbi` loop) so the just-emitted instructions are visible to the icache when execution jumps to them.
- `src/cpu/core_dynrec/decoder_basic.h` — switches a handful of EA-builder calls from `gen_add` / `gen_mov_word_to_reg` to new `gen_add_LE` / `gen_mov_LE_word_to_reg` variants. On little-endian hosts the new names are macros aliasing the old ones; on big-endian PPC they emit `lwbrx`/`lhbrx` (byte-reversing loads). The reason: those EA-builder paths read 16/32-bit values that came directly out of the guest's instruction stream, which is little-endian.
- `src/cpu/core_dynrec/decoder_opcodes.h` — same `gen_*_LE` substitutions, plus a small rewrite of `dyn_ret_near` (drops a now-redundant `gen_extend_word` after `dynrec_pop_word` and uses `decode.big_op` for the size of the writeback).
- `src/cpu/core_dynrec/Makefile.am` — adds `risc_ppc.h` to the source list. (Boxer uses Xcode, so this file isn't relevant to the integrated build.)
- `src/cpu/core_dynrec.cpp` — adds a `POWERPC` arch token and the `#elif C_TARGETCPU == POWERPC` branch that includes `risc_ppc.h`, plus `#define gen_*_LE gen_*` aliases on little-endian hosts.

## Boxer-side divergences that overlapped with the patch

Most of the patch goes in cleanly. These are the places where Boxer had already changed the code in a way that interacted with the patch:

- `src/dos/drive_fat.cpp` — Boxer already had a different fix for the same FAT image endianness problem the patch addresses. Boxer's tree adds `BXCoalfaceDrives.{h,mm}` with whole-struct byteswap helpers (`boxer_FATPartitionTableLittleToHost`, `boxer_FATBootstrapLittleToHost`, `boxer_FATDirEntry{LittleToHost,HostToLittle}`) and calls them at the appropriate read/write boundaries. The patch's MBR / boot-sector / dir-entry hunks were skipped (applying them on top would double-swap). The patch's per-FAT-entry `var_read` / `var_write` substitutions in `getClusterValue` / `setClusterValue` were applied, because the FAT sector buffer is a raw `Bit8u[]` indexed at arbitrary offsets and never goes through the coalface helpers — those five substitutions fix a real Boxer-on-PPC bug that the coalface code did not cover.

- `src/cpu/core_dynrec/cache.h` — Boxer used the older single `PFLAG_HASCODE` flag (no 16- vs 32-bit distinction). The patch's hunks did not touch that line, so this divergence is benign.

- `src/cpu/core_dynrec/decoder_basic.h` — Boxer had:
  - `decode.modrm.val` retained as a struct field rather than a local.
  - `MakeCodePage` simpler (no stale-cph-clear logic).
  - `dyn_set_eip_end` using `decode.big_op` instead of `,true` for the EIP load.

  The first two are unrelated to the patch. The third one **is the bug** (see below). It was deliberately skipped in the first integration pass on the (mistaken) reasoning that Boxer's form looked self-consistent and the patch's change looked like an optimization. It is not an optimization on PPC.

- `src/cpu/core_dynrec/decoder_opcodes.h` — Boxer's `dyn_pop_ev` was substantially older (no protected-ESP / fault-rollback). That code is far away from any of the patch hunks, so it is unrelated.

- `src/cpu/core_dynrec.cpp` — Boxer already had a `POWERPC` arch token and a `#elif POWERPC` branch including `risc_ppc.h` (the file the patch adds), but the token's value was `0x04` — same as `ARMV4LE`. That clash was harmless in practice (you don't build for both at once), but `POWERPC` was bumped to `0x06` to match the patch and avoid future confusion.

- `src/cpu/core_dynrec/Makefile.am` — does not exist in Boxer (Xcode builds). The Xcode project picks up `risc_ppc.h` automatically because the `#include` is conditional on `C_TARGETCPU == POWERPC` and headers don't need to be in the build phase.

There were also two divergences far from any patch hunk that turned out to matter once the JIT was actually running:

- `include/regs.h` had `Segs::val[]` declared as `Bitu[8]` (4 bytes per slot on PPC32). The PPC dynrec backend's segment-load helpers (`gen_mov_seg16_to_reg` etc.) emit `lhz` (16-bit zero-extending load) at the byte offset of `val[i]`. With a 4-byte slot on big-endian, those two loaded bytes are the high half of the slot — always zero, since segment selectors are 16-bit. The patch keeps `Bit16u[8]` (which the upstream DOSBox tree fixed in 2018; that fix was never pulled into Boxer's `leopard_legacy`). Reverted to `Bit16u[8]`.

- `config.h` only enables `C_DYNREC` on `__x86_64__ && !C_DYNAMIC_X86`. On PPC, `C_DYNREC` was never set, so `core_dynrec.cpp` (the whole file is `#if (C_DYNREC)`) compiled to nothing and `cpu.cpp`'s decoder dispatch silently fell through to the normal core regardless of what the user's conf said. Added a clause to enable `C_DYNREC` on `__ppc__ || __ppc64__`.

## The bug

After everything compiled and the JIT was actually running, programs froze within the first second of execution. The hang was not a CPU-burning infinite loop — Boxer sat at near-0% CPU, the emulator window was frozen, and quitting required force-quit because the emulator thread had wandered off into bad state and the Boxer wrapper was waiting for it.

What was eventually visible in the trace was that within ~30 JIT'd blocks, execution would land at a control-flow instruction whose **target was completely unrelated to the source EIP**. The trace showed a JIT block at `cs:eip = 0x0070:0x4711` exiting with `eip = 0xfe3f`. The first byte of the block was `e8 3c fe …` — a near `CALL` with displacement bytes `3c fe`.

The arithmetic that should have happened:

```
target = current_eip + instruction_length + sign_extended(displacement)
       = 0x4711 + 3 + (Bit16s)0xfe3c
       = 0x4711 + 3 + (-0x1c4)
       = 0x4550
```

The arithmetic that actually happened:

```
target = 0 + 3 + (Bit16s)0xfe3c
       = 0xfe3f
```

The `current_eip` term was being dropped entirely. After the bad CALL, execution drifted into BIOS data area, eventually hit an invalid opcode, faulted into INT 6 → INT 13 → IRET, and the IRET popped garbage off the wrong stack frame and asserted in `CPU_IRET`.

Where the EIP went missing: in `dyn_set_eip_end(reg, imm)` (`src/cpu/core_dynrec/decoder_basic.h`):

```c
static INLINE void dyn_set_eip_end(HostReg reg, Bit32u imm = 0) {
    gen_mov_word_to_reg(reg, &reg_eip, decode.big_op);   // <-- the trap
    gen_add_imm(reg, (Bit32u)(decode.code - decode.code_start + imm));
    if (!decode.big_op) gen_extend_word(false, reg);
}
```

`reg_eip` is `cpu_regs.ip.dword[0]` — a `Bit32u`. In 16-bit code mode (`decode.big_op == false`), the call passes `false` for the `dword` argument to `gen_mov_word_to_reg`, asking it to emit a 16-bit load. The PPC backend's implementation:

```c
static void gen_mov_word_to_reg(HostReg dest_reg, void *data, bool dword) {
    Bit32s addr = (Bit32s)data;
    HostReg ld = gen_addr(addr, dest_reg);
    IMM_OP(dword ? 32 : 40, dest_reg, ld, addr);  // lwz / lhz dest, addr@l(ld)
}
```

It emits `lhz` at the **raw byte address** of the `Bit32u`. On big-endian PPC, bytes 0–1 of a `Bit32u` are the high 16 bits, and the low 16 bits (the actual IP) live in bytes 2–3. So in 16-bit code mode, the function loads the **high half** of `reg_eip` — which is always zero, because IP only has 16 significant bits — adds the displacement, masks to 16 bits, and the result is just `(displacement + length) & 0xFFFF`. Every relative CALL, JMP, and Jcc in 16-bit protected-mode code jumped to a wrong, EIP-independent address.

Note that the sister helper `gen_add_direct_word` in `risc_ppc.h` *does* compensate for this:

```c
static void gen_add_direct_word(void *dest, Bit32u imm, bool dword) {
    Bit32s addr = (Bit32s)dest;
    if (!dword) {
        imm &= 0xFFFF;
        addr += 2;          // <-- skip the high half on big-endian
    }
    …
}
```

`gen_mov_word_to_reg` does not. That's why everywhere else in the dynrec that does a 16-bit access through `gen_mov_word_to_reg` happens to be correct: the callers either pass an actual `Bit16u*` (`fpu.sw`), or pass `&cpu_regs.regs[r].word[W_INDEX]` where `W_INDEX=1` on big-endian (so the address already points to bytes 2–3), or pass `&Segs.val[i]` after the `Bit16u[8]` fix above. The `dyn_set_eip_end` call site was the one place that passed `&reg_eip` (a `Bit32u*`) and asked for a 16-bit read.

The patch's fix for `dyn_set_eip_end` is to load 32 bits unconditionally and drop the redundant extend (the writeback path masks to 16 bits when storing back to `reg_ip`):

```c
static INLINE void dyn_set_eip_end(HostReg reg, Bit32u imm = 0) {
    gen_mov_word_to_reg(reg, &reg_eip, true);   // 32-bit load: reads all 4 bytes correctly
    gen_add_imm(reg, (Bit32u)(decode.code - decode.code_start + imm));
}
```

That is the entire fix. Two lines.

## Lessons (for the next person doing something similar)

- When integrating a working patch into a divergent tree, hunks that look "purely cosmetic" can be load-bearing on the host architecture the patch targeted. The `dyn_set_eip_end` change in this patch reads as an optimization (drop a redundant `gen_extend_word` because the load is now full-width). On x86 it *is* purely an optimization. On PPC it is the difference between a working JIT and one that mis-translates every 16-bit-mode relative branch. If a patch has been tested working on the target architecture and a hunk is skipped because "the local form looks fine", that is debt that is invisible until execution actually runs.

- "Big-endian half-word access through a `Bit32u*`" is a category of bug that bites once and then bites again. Boxer had the same shape of mistake in two places: `Segs::val[]` declared as `Bitu` rather than `Bit16u`, and `dyn_set_eip_end` calling `gen_mov_word_to_reg(reg, &reg_eip, false)`. Both produced the same symptom (a 16-bit read that returns the wrong two bytes), in different code paths, with different triggering conditions. If one such bug appears, grep for the pattern.

- DOSBox's recompilers fall back to the normal core for any opcode they don't translate (returning `BR_Opcode`). This means a buggy JIT'd CALL can land somewhere the JIT can't even decode, and the `INT 6` (invalid opcode) will come from the *normal* core's `default:` arm — not the JIT. When debugging, remember the failure site and the JIT bug site can be far apart.

- A debugging ring-buffer that records `{block #, CS:EIP at entry, return code, CS:EIP at exit, first 32 bytes of the x86 instruction stream being translated}` is invaluable. Once that was in place the bug was found in two iterations: the first iteration showed the wrong target EIP, the second (with the byte dump) made the displacement arithmetic obvious.

## Final list of files modified vs. Boxer's `leopard_legacy`

```
DOSBox/config.h                                    # enable C_DYNREC on PPC
DOSBox/include/fpu.h                               # add FPU_rec tag
DOSBox/include/mem.h                               # var_read overloads, bswap fast path
DOSBox/include/regs.h                              # Segs::val[] back to Bit16u
DOSBox/src/dos/drive_fat.cpp                       # var_read/var_write for FAT entries
DOSBox/src/cpu/core_dynrec.cpp                     # POWERPC=0x06, LE shim macros, no-op stubs
DOSBox/src/cpu/core_dynrec/cache.h                 # write_map raw deref, link-block restructure
DOSBox/src/cpu/core_dynrec/decoder_basic.h         # gen_*_LE substitutions + dyn_set_eip_end fix
DOSBox/src/cpu/core_dynrec/decoder_opcodes.h       # gen_*_LE substitutions, dyn_ret_near rewrite
DOSBox/src/cpu/core_dynrec/risc_ppc.h              # NEW FILE — PPC backend (from patch, unmodified)
```

## Future work: r4301 base-bump (findings, not a plan)

A possible future direction is to stop backporting the JIT into Boxer's 0.74 base and instead bring the base forward to DOSBox SVN r4301, so the JIT patch becomes a clean drop-in against the revision it was authored for. Not started; this section records what was concluded about feasibility, not a committed plan.

- **r4301 is the patch's authoring revision.** Matching the base eliminates the integration backporting that produced the bugs documented above. The patch itself is known-good at r4301.

- **Madd's commit `4a85221c` on the `64bit/master` branch is the template.** That commit jumped Boxer's DOSBox base from 0.74 to SVN r4068 — ~250 revs short of r4301, same shape of work. 237 files, ~12k inserts / ~8k deletes. Boxer-side glue (`BXCoalface`, `BXEmulator+BXDOSFileSystem`) needed only single-digit-line touches.

- **C++11 is not a risk.** r4301 builds under Tiger / Xcode 2.5 / GCC 4.0.1 (confirmed empirically), which predates llvm-gcc-4.2 by years. Mainline DOSBox SVN through this era stayed C++03-clean for portability. Don't waste time auditing for `nullptr`, `std::atomic`, etc.

- **PPC-specific endianness fixes must be preserved across the rebase.** Boxer accumulated hand-fixed PPC bugs that the surrounding code has shifted under: `6995713f` (FAT images), `9456f8cf` (MT-32), `f4935f44` (Tandy/CGA), plus the FAT coalface helpers. `64bit/master` may have lost some of these when it dropped PPC support entirely at `31952311`, so its diff cannot be trusted blind for PPC correctness — audit each surviving Boxer PPC fix against the new upstream code.

- **Boxer wrote `sdlmain` out of existence at `2702c15a`.** Watch for r4068 → r4301 deltas that re-introduce SDL bootstrap code; it has to stay out.

- **A working copy of the DOSBox SVN repository will be available locally for this work.** Use it directly to browse history and produce per-revision diffs (e.g. `svn diff -r4068:4301`) rather than fetching upstream changes through web tools or asking the user to extract them.

## Credits

The PPC dynrec backend (`risc_ppc.h` and the supporting infrastructure changes in `cache.h`, `decoder_basic.h`, `decoder_opcodes.h`, `core_dynrec.cpp`, `mem.h`, and `drive_fat.cpp`) is from the third-party DOSBox community patch known as `patch-r4301.diff`, authored by jmarsh and posted on Vogons (https://www.vogons.org/viewtopic.php?f=32&t=65057), with binaries packaged by Dr. Cameron Kaiser on SourceForge (https://sourceforge.net/projects/dosbox-ppcjit/). All credit for the JIT itself goes to those parties. This integration glues that patch into Boxer's older, divergent tree.
