# Open questions

Unresolved unknowns. Each blocks (or de-risks) a specific part of the plan.
Answer in Milestone 0 unless noted.

## Binary

- [ ] **Compiler**: exact MSVC version + flags. Determines decompiler/lifter
      fidelity and matching. (`strings`/PE timestamps/RTTI signatures can hint.)
- [ ] **FP mix**: was `/arch:SSE2` used, or is x87 dominant? Sizes the §7.2
      work.
- [ ] **CRT linkage**: static CRT or `msvcrt.dll` import? Shapes §7.6.
- [ ] **Cleanliness**: is the installed `witcher.exe` a clean, un-packed,
      non-self-modifying binary? Is it byte-identical to the GOG/Steam DRM-free
      build (useful cross-check)?

## Platform surface

- [ ] **Import table**: exact Win32 + D3D9 API list. Sizes the shim and checks
      Wine-arm64 coverage.
- [ ] **D3D9 feature set**: which D3D9 features/materials/shaders does CDPR's
      material system use? Checks DXVK coverage.
- [ ] **Wine-arm64 maturity** for this game's specific usage (dinput, dsound,
      winmm, binkw32, msvcrt).
- [ ] **PowerVR CXTP Vulkan driver** behavior with DXVK on Tensor G6 (the
      device-specific unknown).

## Game behavior

- [ ] **Threading**: is TW1 single-threaded (2007)? Affects the recompiler's
      memory/thread model.
- [ ] **Self-modifying / anti-debug remnants** even in the DRM-free build.
- [ ] **Bink `.bik`**: confirm decode path (translate `binkw32.dll` vs FFmpeg).
- [ ] **Mac port**: the Steam manifest lists a `The Witcher.app`. Determine if
      it has an OpenGL (rather than D3D9) renderer — could de-risk the graphics
      layer (still x86/closed, so CPU layer unaffected).

## Toolchain (decision D8)

- [ ] Confirm current state of remill/McSema (maintenance, arm64 output).
- [ ] Confirm xboxrecomp's x86→C approach and whether an arm64 emitter exists.
- [ ] Decide lifter core + who owns SEH/x87/import fixes (us vs tool).

## Legal

- [ ] Confirm we keep everything engine-only and don't redistribute the
      recompiled game binary or assets.
