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
