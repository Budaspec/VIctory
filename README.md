A fork of the DexNom community victory fork by Budaspec.

## Status

The game boots, reaches the main menu and starts loading into the world on an AMD Radeon RX 9060 XT (driver 32.0.32015.7029). Gameplay is not stable yet: the fixes below have been tested shader by shader but not yet in a full play session.

## Progress

### 2026-11-25

- **Setup.** The runtime builds and installs with `Setup.cmd`. Shader translation works for about 22,000 of the game's shaders.
- **AMD driver crash, found.** After the main menu the game hung for 10+ seconds ("waiting for a new frame sample") and then crashed with `0xc0000005`. The crash is inside AMD's shader compiler (`amdxc64.dll`), while it compiles some large compute shaders. Our translated shaders pass Microsoft's DXIL validator, so the code is valid and AMD's compiler mishandles it.
- **AMD driver crash, cause.** The crash was reproduced offline by compiling a single shader, without the game. The trigger is one optimizer step: a texture-size query (`getDimensions`) on a heap-indexed texture is computed once and reused in later blocks. AMD's compiler reads through a null pointer on that pattern.
- **AMD driver crash, fix.** The optimizer no longer reuses those size queries; each use computes its own, which costs next to nothing. All 12 crashing shaders that were tested offline now compile. The same shaders still crash without the fix.
- **Safety net.** If the driver still crashes while compiling a pipeline, the runtime records that pipeline in `cache\pipelines\pipelines-crashed.txt` and skips it on the next launch. The launcher then restarts the game automatically. This is only a fallback: a skipped pipeline is an effect that does not draw.

## Known issues

- **Untested in play.** The driver crash fix has not been confirmed in a full play session. An offline check that compiles all ~22,000 shaders is in progress.
- **Ordered-pixel shaders are not drawn.** 10 pixel shaders use the console's ordered pixel shading (POPS), which keeps overlapping transparent pixels in draw order. The translator does not support it yet, so those draws are dropped. The planned fix maps them to D3D12 rasterizer-ordered views.
- **Old skip list.** Pipelines recorded as crashing before the fix stay skipped until `pipelines-crashed.txt` is cleared.
- **Inherited issues.** These come from the upstream runtime: some objects render black, frame times can be long in some scenes, and ray tracing is experimental.

## Credits

Built on DexNom Runtime and Marvisak's VIctory. See [README.upstream.md](README.upstream.md) for the original credits and the [license](LICENSE) (GPL-3.0-or-later).
