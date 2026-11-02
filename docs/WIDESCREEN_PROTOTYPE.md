# Widescreen milestone 1: experimental 16:9 PPU scanlines

This is a **prototype**, not a finished widescreen port. Changes live only on `feature/widescreen-16-9`.

## What is implemented

- Opt-in `[Graphics] ExtendedAspectRatio = 16:9` selects a raw viewport of **398x224**, extending the original 256x224 image by 71 pixels on each side. It does not stretch characters. Original rendering is the default.
- Padded 400-pixel-wide PPU scanline buffers; native 256 pixels are still copied correctly when the option is disabled.
- In SNES Mode 1 and with mosaic disabled, the fast PPU attempts to render additional BG tiles and available sprite/OAM data in the side margins. The capacity and rendering loop follow concepts used by Zelda3; no external code has been copied verbatim.
- Unsupported modes (including Mode 7, mosaics and the old PPU renderer) render a **centered 256-pixel scene**. Margins are cleared per scanline to prevent old-frame garbage.
- The selected host margin is retained during PPU resets; the serialized SNES snapshot byte ranges remain unchanged.

## How to test on Windows

On the development branch, compile normally with `run_with_tcc.bat`. To avoid modifying tracked `smw.ini`, make a personal config:

```powershell
Copy-Item smw.ini smw.user.ini
notepad smw.user.ini
```

The personal config is now ignored by Git. Uncomment or add the following setting **inside `[Graphics]`**:

```ini
ExtendedAspectRatio = 16:9
```

Keep `IgnoreAspectRatio = 0` and `NewRenderer = 1`. For the original view instead use `ExtendedAspectRatio = 4:3` or comment out the setting. Restart the application after changing it.

Try title screen (256-pixel fallback expected), Yoshi's Island 1 and 2, enemy positions near left/right edges, scrolling, mosaic fades, overworld, and the `R` renderer toggle. Check SDL/OpenGL output and compare the 4:3 baseline. Share build errors and screenshots; this changes code only and hasn't been compiled on the owner's Windows PC yet.

## Known issues / why this is not finished

1. The game streams native-width tile data into VRAM. Side margins may show wrong, missing or repeating terrain until a Map16/world-aware renderer or wider streaming is implemented.
2. The game's original OAM, camera and enemy activation/despawn still operate in the native window. Actors beyond the original 256px are not guaranteed to be created or drawn.
3. The original HUD is not re-anchored; it can sit in the middle of the wide screen.
4. Mode 7, other modes and mosaic effects intentionally revert to native width. Some transitions have black side bars.
5. **Build and in-game behavior are unverified here.** Windows TCC compilation and real gameplay tests are the immediate next step.

## References

- https://github.com/snesrev/zelda3 — wide PPU buffers and margin concepts
- https://github.com/VitorVilela7/wide-snes — SMW gameplay, level and camera implications
- https://github.com/mstan/SuperMarioWorldRecomp/blob/main/docs/adaptive-renderer.md — an alternative Map16/OAM host rendering approach

Next: validate compilation and 4:3 parity, fix tile streaming, sprites/spawns, HUD and transitions in separate commits.

## Milestone 2 — enemy spawn/despawn preview

Placed ordinary enemies in stock horizontal levels are now preloaded inside the 398-pixel visible area plus 32 pixels beyond each side; the original loader and 12 normal sprite slots are preserved. Death/erase guards avoid repeated loading while a placement remains within the activation area, and the three normal offscreen-removal banks use the expanded boundary without overriding vertical removal. Drawing culls in Bank01 and Bank23 are also relaxed to the expanded boundary.

**Unverified:** Build TCC/Windows and observe on Yoshi's Island 1/2. Special sprites, generators, shooters, vertical levels, Lunar Magic and Mode 7 are unchanged. Guard state is not stored in savestates. OAM coordinate limitations may still hide objects despite earlier loading. Test a killed Koopa for immediate respawn, and move camera away and back; report exact level and screenshot.

## Hotfix: native gameplay mode for widescreen and duplicate spawn guard

The previous milestone 2 triggered `Memory compare failed`, `VRAM OAM compare failed` and `Verify failure!`. This is because the default project runner compares the original SNES ROM simulation against the reimplemented C simulation every frame. Early enemy spawns intentionally change C RAM and OAM, so the verifier rejects the difference and restores the original result, undoing the visible change.

When `ExtendedAspectRatio = 16:9`, **SMW now runs the reimplemented C simulation alone** (native mode), with a one-time console explanation. The ROM-vs-C verifier is unchanged for the standard `4:3` mode; do not interpret no verifier warnings in 16:9 as proof of gameplay correctness. The original sprite loader now also honors the widened loader's per-placement guard, preventing a native-range load from duplicating an enemy recently killed inside the wide activation zone.

**Retest required:** Build with TCC, confirm exactly one `Experimental widescreen: native C gameplay only; ROM/C comparison disabled.` line at game start, verify early spawning on Yoshi's Island 1/2 and whether extra sprites render in the right/left margins. Compare against `4:3`, where that line must not appear. Check death/respawn and save/load. If sprites remain absent, investigate OAM/graphics coordinates rather than suppressing diagnostics.

## Milestone 3 — OAM right margin and spawn identity (awaiting Windows validation)

**User observations (previous commit):** normal enemies remain absent from the wide margins; repeated left/right camera movement can produce duplicate living mobs; a bonus pipe room with a rotating ring of hittable blocks is broken (blocks too high and significant graphics corruption). The bonus-room bug is **deferred but tracked as a serious gameplay regression**.

The PPU originally interprets 9-bit OAM X values >=256 as negative (X-512). That makes the extended right margin inaccessible to sprites that only use the classic OAM encoding. The host PPU now accepts a per-tile **right-side X hint**, recorded by SMW C sprite draw paths (FinishOAMWrite and common one-/two-tile graphics), reset at the start of each game frame, and used only in widescreen PPU mode. Original SNES OAM bytes remain unchanged.

The extra sprite loader now checks whether a source index still belongs to an active normal sprite slot, even when load status has been cleared, before allowing duplicate spawning. Its placement guard is no longer cleared when the placement leaves the activation region but its enemy remains alive.

**Tests:** compile with TCC; check a basic Koopa appearing toward the right outside the old 256px area, moving into the old field, then back. Walk left/right to check duplicate enemies, kill an enemy and inspect return behavior. The change does *not* yet cover special drawing paths, alternative OAM routines or the bonus-room regression. Verify 4:3 parity. If all enemies still pop in at the old border, inspect the remaining OAM paths and the scene mode.

## Milestone 4 — Zelda3 9-bit sprite X decoding

The user reports that ordinary enemies still pop in at the native 256px edge. `snesrev/zelda3/snes/ppu.c::ppu_evaluateSprites` holds 9-bit X on the right until the extended edge; SMW's PPU used to wrap all 9-bit X >=256 by subtracting 512. X=300 thus became -212, hiding a sprite that legitimately belongs on the widened right side.

SMW now wraps at `x >= 256 + ppu->extraRightCur`. With the experimental 71 extra pixels this permits X=256..326; with 4:3 it wraps at the original 256. The previous host-only OAM-X markers remain in place. This is a targeted rendering correction, not a fix for enemy spawn semantics, every specialty graphics routine, duplicate actor lifecycle, or the pipe bonus room.

**Unverified Windows test**: rebuild with TCC, check normal Koopas at X=256..326 while entering from right, then move camera back and forth; compare to 4:3. If sprites still only appear at the old border, log the placement-loader and actual high OAM X bit for a specific enemy.

## Milestone 5 — BG3 water and rotating-platform safeguards (Windows tests pending)

**New user-confirmed regressions after OAM wrap fix:** enemies now largely appear in 16:9, but a bonus pipe area's bottom water renders at only native 256px, with misplaced/missing level objects. Rotating mobile platforms in a normal level are now invisible, breaking progression; they previously worked in an earlier build.

BG3 water root cause: `PpuWindows_Clear/Calc` systematically excluded BG3 from expanded margins, because SMW uses BG3 for its status bar. In this experiment BG3 is extended **only below the first 32 scanlines**, with top 32 still native-width. Tilemap streaming of BG3 is unchanged; side effects remain possible. Rotating platforms: the experimental loader indiscriminately pre-spawned IDs below `0xc9`, including platform IDs `0x5f`, `0x9e`, `0xa3`, potentially breaking slots/OAM ordering. Early spawning now only handles low-risk basic enemy IDs `0x00..​0x0c`, `0x0f..0x11`. Entire levels listing rotating/chained platforms use stock spawn order; the early loader keeps at least four of twelve normal slots free. Rendering sprite/tile budgets scale up only in 16:9 (50 objects, 54 slivers per scanline vs 32/34 stock).

**Re-test before claiming success**: ordinary Koopa stage (new 16:9 mobs still visible), the rotating-platform stage (platforms exist and carry Mario), watery bonus stage (water extends both edges), and 4:3 parity. Other bonus issues (high unreachable ring and missing geometry) remain open, and special platform behavior needs extra QA.

## Milestone 6 — HUD reserve item OAM stability and basic enemy activation

User test of milestone 5: **BG3 water works**, rotating platforms are visible again but still glitchy, enemy activation regressed to the native 256px boundary, and the status-bar reserve item jumps horizontally when the camera crosses coarse scroll boundaries.

The previous precaution disabled the *entire* widened placement parser when a complex rotating/chained platform existed anywhere in that stage. Removed that global veto. The parser retains its restricted basic-enemy allowlist and reserves four native sprite slots; platforms and other complex sprites still use the original loader and timing.

The OAM renderer also has a host-only right-side hint array. As native SMW reuses OAM slots for different drawings, a hint from a game sprite can later be accidentally applied to a HUD sprite, despite that item's actual screen X being fixed (120px). Hints are now used only if their low 8 bits match the current native OAM tile X, invalidating mismatched/stale positions. This is an evidence-based probable fix, **not yet gameplay validated**.

Windows acceptance: compile TCC; check an ordinary enemy in the right 16:9 margin inside a level with rotating platforms, platform rendering and ride/collision, BG3 water, and reserve box item during multiple camera scrolling boundaries. Item should remain anchored at X=120 in the native 256px center. Further bonus room regressions remain open.
