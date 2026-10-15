# The Witcher 1 → Android (native)

Port **The Witcher: Enhanced Edition — Director's Cut** to run **natively** on
Android (target: Pixel 11 / Google Tensor G6, `arm64`), by statically
translating the game binary. Development and testing happen on **PC (x86-64)**
via retargetable intermediate representation.

## Goal

A native Android app that plays The Witcher 1 perfectly, with **no CPU
emulation or runtime JIT** — the game's own machine code runs directly on
the ARM cores.

## Approach (one line)

Lift `witcher.exe` (32-bit x86) to retargetable IR once, emit **arm64** for
Android and **x86-64** for PC, and stand it on a native platform layer
(Wine-arm64 + DXVK) with import shims.

## Directory layout

```
..  (this repo's parent)   = /mnt/i/torrent/witcher1
    ├── setup.exe          = FitGirl installer (FreeArc)
    ├── fg-*.bin           = FreeArc archives (the game data, compressed)
    └── port/              = THIS REPO (git)
        ├── README.md
        ├── docs/
        │   ├── PLAN.md            = full technical plan (the brainstorm, written down)
        │   ├── DECISIONS.md       = decision log (ADR-style)
        │   └── OPEN-QUESTIONS.md  = outstanding unknowns to resolve
        └── .gitignore
```

**The game data is NOT in this repo** — it lives in the parent directory and is
never committed. After install, the game binary is expected at
`../The Witcher Enhanced Edition/system/witcher.exe` (verify exact path during
Milestone 0).

## Status

Brainstorming / planning. No code yet. See `docs/PLAN.md`.

## First steps (Milestone 0)

1. Install the repack natively on Windows (`..\setup.exe`, base + English only)
   to produce the real game tree and a **DRM-free** `system/witcher.exe`.
2. Confirm the binary is clean for static analysis (no self-modifying code /
   anti-debug remnants).
3. Identify the compiler (MSVC version) and FP instruction mix (x87 vs SSE2).

See `docs/PLAN.md` → "Milestones".
