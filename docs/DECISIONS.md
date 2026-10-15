# Decisions

ADR-style log. Each entry: what we decided, why, and the rejected alternative.

---

## D1 — Native arm64, not runtime emulation
**Decision**: The end state is native execution on arm64; Box86/Winlator-style
runtime JIT is rejected as the *final* deliverable.
**Why**: The goal is a real native APK, not a container.
**Rejected**: Winlator (Wine + Box86 + DXVK) as the shipped solution. It may
still be used as an interim *walking skeleton* to prove the game runs on the
target device early.

## D2 — Translate the binary, don't reimplement the engine
**Decision**: Keep the original engine code (combat, AI, ~850 engine
functions) by statically recompiling `witcher.exe`, rather than writing a new
engine.
**Why**: Translation keeps byte-perfect behavior and gets combat/AI/functions
"for free" — the hardest parts of any reimplementation.
**Rejected**: xoreos-style reimplementation (recover behavior, write new
engine), and matching decompilation of the whole binary.

## D3 — Outcome: "plays perfectly, mechanism doesn't matter"
**Decision**: Success = the game plays perfectly on Android. Whether a given
layer is AOT-recompiled, JIT'd, or shimmed is an implementation detail.
**Why**: Anchors decisions on observable results, not purity.
**Implication**: The mature translator/stack is the default; push layers
"more native" only where there's a concrete win (footprint, startup, perf).

## D4 — Platform layer: Wine-arm64 + DXVK for now
**Decision**: Serve Win32 with Wine (native arm64) and D3D9 with DXVK
(D3D9→Vulkan).
**Why**: Mature, native, safe default; keeps the project bounded.
**Deferred**: a hand-built minimal Win32 shim (~200-400 APIs) for a
smaller/lighter APK. Revisit after the safe stack is proven.

## D5 — DRM-free content confirmed
**Decision**: The repack's content is DRM-free (packaged, not protected).
**Why**: DRM/anti-debug would be catastrophic for static recompilation.
**Action**: still verify the *installed* `witcher.exe` is clean (no
self-modifying code / anti-debug remnants) before lifting — §7.1 in PLAN.

## D6 — Repo lives in a child dir; game data not committed
**Decision**: The git repo is `/mnt/i/torrent/witcher1/port/`, a child of the
game-data directory. Game data is referenced from the parent (`..`) and never
committed.
**Why**: Keeps engine/repo separate from assets (licensing + size).

## D7 — Retargetable IR: PC (x86-64) first, then arm64
**Decision**: Lift to IR once, emit x86-64 for PC dev/test and arm64 for the
phone. Develop/test on PC first.
**Why**: Same-machine diffing against the original is far faster than
iterating on-device.

## D8 — (open) Recompiler toolchain
**Lean**: remill/McSema x86→LLVM semantics as the core + custom SEH/x87/import
fixes + AI-generated glue. Not yet finalized — see `OPEN-QUESTIONS.md`.
