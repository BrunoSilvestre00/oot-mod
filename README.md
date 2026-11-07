# Ultrawide Mod for The Legend of Zelda: Ocarina of Time (Nintendo Switch 2)

Play the Ocarina of Time remake in 21:9 or 32:9 on an emulator. The 3D view gets a genuinely wider field of view, while the HUD and pre-rendered videos stay at their original 16:9 proportions, centred on screen.

Distributed as small IPS patches only. No game files are included.

## Versions

| Download | For screens | Target aspect |
| --- | --- | --- |
| [`oot-ultrawide-21x9-3440x1440.zip`](releases/oot-ultrawide-21x9-3440x1440.zip) | 3440×1440 | 2.389 (43:18) |
| [`oot-ultrawide-21x9-2560x1080.zip`](releases/oot-ultrawide-21x9-2560x1080.zip) | 2560×1080, 5120×2160 | 2.370 (64:27) |
| [`oot-super-ultrawide-32x9.zip`](releases/oot-super-ultrawide-32x9.zip) | 5120×1440, 3840×1080 | 3.556 (32:9) |

Enable **only one** at a time, since they all patch the same code.

## Compatibility

- **Game:** The Legend of Zelda: Ocarina of Time, title ID `0400486024DEC000`
- **Executable:** `main` build ID `E0F3CCFAF50065AE4EA01EB5AEB13B1A45DA3348`. Other builds are not supported.
- **Tested on:** [Lichii](https://github.com/zortos293/Lichii) `nx2-build-88`. Works alongside Lichii's built-in "Max Resolution" and "Dynamic FPS" patches.

## Installation

1. Download the zip for your screen from [`releases/`](releases/).
2. In Lichii, right-click the game → **Properties → Add-Ons → Import Mod from ZIP**, select the zip and accept the suggested mod name. If you installed another version before, untick or delete it in the same list.
3. In the same Properties window, set **Graphics → Aspect Ratio → Stretch to Window** and play fullscreen.
   The game still renders a 16:9 frame; the emulator stretches it to your screen and the mod pre-compensates for that stretch.
4. Restart the game. The emulator log should show `Applying IPS patch from mod "<mod name>"`.

For other emulators, or to install by hand, extract the zip into the emulator's `load/` directory. It contains `0400486024DEC000/<mod name>/exefs/E0F3CCFAF50065AE4EA01EB5AEB13B1A45DA3348.ips`.

## What it changes

- **3D:** every 16:9 camera projection is switched to the target aspect, so you see more to the sides instead of a stretched image. The new aspect is written back to the camera, so other systems that read it (e.g. culling) see the wider view too. Non-16:9 views such as cubemaps are left alone.
- **HUD and menus:** full-screen 2D is drawn into the centre 16:9 area. Edge-anchored elements (hearts, buttons, minimap) sit at the edges of that area. Markers that track the 3D world, such as the Z-target reticle and NPC speech bubbles, stay in the right place. Menu backgrounds and fade transitions still cover the whole screen.
- **Pre-rendered videos:** drawn in the centre 16:9 area instead of being stretched.

## Known limitations

- Only the build ID above is supported. A game update will need new offsets.

## How it works

The patch places small ARM64 routines in unused executable padding after `.text` (`main+0x3C46400`) and branches to them from eight hook sites. Each hook reruns the instruction it replaced and jumps back.

| Hook (`main+`) | Purpose |
| --- | --- |
| `0x3485CC0` | `sead::PerspectiveProjection::doUpdateMatrix`: if the aspect is within 0.01 of 16:9, use the target aspect and write it back to `mAspect` |
| `0x004F1E8`, `0x023B9A8`, `0x033B82C` | Inlined copies of the same matrix builder (the game mostly uses these) |
| `0x025D76C` | Render-view setup: the width/height aspect passed to shaders gets the same mapping |
| `0x0774368` | 2D canvas setup: if the projection is the full-screen 3840×2160 layout ortho, scale x by `(16/9) / target`. The backgrounds of the L/R item menu, pause menu and system menu are skipped, because they redraw a capture of the whole frame. So are the loading and cutscene fades, so they cover the sides |
| `0x3B5C758` | Movie layer draw: shrink the full-target viewport to the centre `(16/9) / target` of the width |
| `0x020C2B0` | NPC speech bubble anchor: the layout x of the speaker (`ndc.x × 1920`) is divided by the HUD squeeze, so the bubble sits over the speaker |

Offsets are module offsets. In the IPS32 files they are shifted by the `0x100`-byte NSO header. All routines only use registers that the original code overwrites immediately afterwards, or that are dead at that point.

## Building

`build.py` generates the three zips in `releases/` using only the Python 3 standard library. The zips are reproducible: rebuilding gives byte-identical files.

```sh
python3 build.py
```

To add another aspect ratio, add an entry to `VARIANTS` in `build.py`: mod name, zip file name and target aspect.

If you have a decompressed image of the `main` module (module offset 0 = file offset 0), you can check every replaced instruction against it:

```sh
python3 build.py --verify-main /path/to/main.bin
```

## Disclaimer

This is an unofficial fan modification and is not affiliated with or endorsed by Nintendo. It contains no Nintendo code, assets, keys or executables. Use it with a legally obtained copy of the game, and back up your saves.
