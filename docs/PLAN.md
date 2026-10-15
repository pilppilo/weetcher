# PLAN — The Witcher 1 → Android (native)

First draft. This captures the full brainstorm as a technical plan, not a
schedule. Timelines are deliberately absent; milestones are defined by *what
you can run and see*.

---

## 1. Target

- **Runtime**: Android on **Pixel 11 / Google Tensor G6** (`arm64-v8a`).
- **GPU**: PowerVR CXTP-48-1536 (Imagination), Vulkan + GLES3.
- **Dev/test**: PC (x86-64) — the recompiled game is retargetable, so the PC is
  the fast iteration surface and the original binary is the oracle.
- **Outcome**: a **native APK** — the game's machine code runs natively on the
  ARM cores. No Box86/Wine-JIT at runtime.

## 2. Inputs (what we're starting from)

- **Game**: The Witcher: Enhanced Edition — Director's Cut, v1.5.
- **Repack**: FitGirl (`setup.exe` + FreeArc `fg-*.bin`), ~11 GB compressed.
  Content is **DRM-free** (confirmed), but verify the installed binary is
  clean for lifting (see §7.1).
- **Engine**: BioWare **Aurora**, heavily modified by CDPR.
  - Binary: `system/witcher.exe`, 32-bit x86, MSVC C++, **D3D9**.
  - Data: KEY/BIF, GFF, 2DA, TLK, MDB (models), DDS textures, `.bik` (Bink)
    video, ogg/mp3 audio.
  - Scripts: **NCS (NWScript) bytecode + Lua `.luc`**.
- **Host**: Windows + WSL; game dir is `/mnt/i/torrent/witcher1`.

## 3. Strategy (decision D2 — "translate the binary")

Keep the original engine code. Do not reimplement combat/AI/engine functions
by hand. Instead:

1. **Statically recompile** `witcher.exe` from x86 → IR → native machine code.
2. **Provide the platform** (Win32 + D3D9) natively.
3. **Bridge** the two with import shims.

The key insight that licenses this: **Wine and DXVK are already native** — Wine
is a native Win32 reimplementation (not an emulator), DXVK is native. The *only*
non-native layer in a Box86+Wine+DXVK stack is Box86's runtime JIT. Replacing
that JIT with AOT translation is the whole "port" — a **bounded** problem.

## 4. Architecture

```
witcher.exe (x86 32-bit, DRM-free)
        │  static recompilation (lift → IR)
        ▼
   LLVM IR / C  ◄────────────── one artifact, retargetable
        │
        ├───────────────────────────────┐
        ▼                               ▼
   arm64 (ship: Pixel 11)           x86-64 (dev: PC)
        │                               │
   import shims                     import shims
        │                               │
   native arm64:                    native x86-64:
   Wine + DXVK + audio              Wine + DXVK + audio
        │                               │
   Android (libc/EGL/Vulkan)        Windows/Linux (dev box)
```

- **Retargetability** is the core win: lift once, emit both. Recompile to
  **x86-64 first**, run it *next to the original* on the same PC, diff
  behavior, debug fast. Emit **arm64** only after x86-64 is proven.
- **Import shims** translate the x86 calling conventions (stdcall/thiscall)
  to arm64 and forward to native Wine/DXVK.

## 5. Platform layer (decision D4)

For now: **Wine-arm64 + DXVK** (native, mature, safe).

- **Wine** (LGPL) compiled for arm64 serves Win32 (`kernel32`, `user32`,
  `dinput`, `dsound`, `winmm`, `msvcrt`, ...).
- **DXVK** (zlib) translates D3D9 → Vulkan; Vulkan is native on PowerVR.

Deferred (the "grander" option): a **minimal hand-built Win32 shim** covering
only the ~200-400 APIs `witcher.exe` actually imports, mapped directly to
libc/EGL/Android. Yields a smaller/lighter APK but is its own sub-project.
Revisit after the safe stack is proven working.

## 6. Tooling fork (open — decision D8)

| Route | Tool | Gives | Tradeoff |
|---|---|---|---|
| Lift → LLVM IR | **remill + McSema** | x86→IR→arm64/x86-64, compiler-grade opts | SEH/x87 incomplete; semi-abandoned; we extend |
| Decompile → C | RetDec / Hex-Rays | readable C to fix and recompile | not built for faithful recompile |
| x86→C→native | xboxrecomp-style | proven on 32-bit x86 games (x87, indirect) | x86→x86-64 today; add arm64 emitter |
| Custom + AI | our own lifter | tailored to TW1's quirks | build semantics (reuse remill's) |

**Lean**: remill/McSema's x86→LLVM *semantics* as the core (don't re-derive
the instruction semantics), with **our own SEH/x87/import fixes** on top and
**AI-generated shim glue + platform scaffolding**. AI is the force multiplier
on boilerplate, shader/glue, and interpretation — not on the semantics table.

## 7. Hard problems & attack plans

### 7.1 Binary cleanliness / DRM (gate — go/no-go)
Retail TW1 shipped TAGES/SecuROM (anti-debug, possibly self-modifying).
- **Attack**: use the **DRM-free** installed binary; scan for
  self-modification / anti-debug before lifting. If unclean, source the GOG or
  Steam DRM-free build. **This gates everything.**

### 7.2 x87 FPU stack
32-bit MSVC circa 2007 uses x87 (register stack, 80-bit) unless `/arch:SSE2`.
- **Attack**: first determine the binary's FP mix (disassemble, count
  x87 vs SSE). If SSE2, this mostly evaporates. Else emulate the x87 stack in
  the lifted output, or convert to SSE2 semantics accepting a tiny precision
  delta. `xboxrecomp` already solved x87 for OG Xbox (same family).

### 7.3 SEH (Structured Exception Handling) — the biggest blocker
MSVC C++ uses SEH for throw/catch, CRT error handling, stack probes; tied to
the FS segment + frame unwinding. This kills most naive static recompilers.
- **Attack**: reimplement the SEH frame mechanism as explicit code in the
  lifted output (setjmp/longjmp-style unwinder, or map to arm64 native
  `.eh_frame` via `-fexceptions`). Reuse Wine's SEH model as reference.

### 7.4 Indirect control flow
C++ vtables, function pointers, `switch` jump tables, Win32 callbacks.
- **Attack**: every function is a potential target; **runtime jump-table
  dispatcher** for unresolved indirects (the conservative TCG trick). Win32
  callbacks (window procs, D3D9 device callbacks) get explicit shim thunks.

### 7.5 Platform surface (imports)
Recompiled code still calls Win32/D3D9. Must bridge x86→arm64 calling
conventions and enumerate the exact API set.
- **Attack**: dump `witcher.exe`'s import table first — it's a finite, known
  list. Decide Wine-arm64 coverage vs minimal shim per API.

### 7.6 CRT / runtime
`witcher.exe` uses the MSVC CRT (malloc/memcpy/operator new/...).
- **Attack**: route through Wine's native `msvcrt.dll` where possible; retarget
  C++ runtime bits to NDK's libc++/libstdc++ as needed.

### 7.7 Memory model
32-bit binary → 64-bit host. Pointers are 32-bit.
- **Advantage**: keep a 4 GB virtual-address model; no pointer-width
  mismatch. Standard for 32-bit static recomp.

### 7.8 Video (`.bik`)
Bink is CPU-heavy x86 code.
- **Attack**: short term, run `binkw32.dll` through the same translation;
  long term, swap in an FFmpeg Bink decoder for native arm64 video.

## 8. Testing strategy

- **Oracle**: the original `witcher.exe` on PC (Windows) is ground truth.
- **x86-64 build**: recompiled game runs natively on PC next to the original;
  diff rendering/output/state. Fast iteration.
- **arm64 build**: validate on Pixel 11 (PowerVR Vulkan driver + DXVK).
- **Save-file compatibility** as an integration test: our engine must load a
  save the original wrote.

## 9. Milestones (tangible — run it, see it)

- **M0 — "the binary is clean and understood"**: install the repack natively,
  produce DRM-free `system/witcher.exe`; confirm clean for lifting; identify
  MSVC version + FP mix; dump import table. *No code yet — but a go/no-go.*
- **M1 — "the pipeline recompiles something"**: lift `witcher.exe` to IR and
  emit an x86-64 binary that runs (even trivially). Proves the toolchain.
- **M2 — "the recompiled game boots on PC"**: x86-64 build renders the menu /
  an area via Wine+DXVK. First visible output.
- **M3 — "the recompiled game plays on PC"**: parity with the original on the
  dev box (walk, talk, fight). The translation is correct.
- **M4 — "it runs on the phone"**: arm64 build boots and renders on Pixel 11.
- **M5 — "native APK, optimized"**: packaged, tuned for Tensor G6
  (thermals, battery, framerate).

## 10. Legal / licensing

- **No asset redistribution** — engine/binaries only; users supply game data
  (ScummVM/OpenMW model).
- Recompiled game code is derivative of CDPR's binary: keep it for personal
  use/interoperability research; don't publish the recompiled `witcher.exe`
  output.
- Reused components carry their own licenses: Wine (LGPL-2.1+), DXVK (zlib),
  remill/McSema (Apache-2.0), RetDec (MIT). Respect each.

## 11. Non-goals (for now)

- Reimplementing the engine from scratch (xoreos path — rejected, D2).
- A from-scratch minimal Win32 shim (deferred, D4).
- Matching decompilation of the whole binary (rejected, D2).
- Distributing game assets.
