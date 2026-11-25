A fork of the DexNom community victory fork by Budaspec.

## Status

On an AMD Radeon RX 9060 XT (driver 32.0.32015.7029) the game loads into the world and runs for about 18 minutes, to frame ~11,600, without a GPU fault. Two problems remain: occasional shader-compiler crashes, which restart the game, and an unstable frame rate that makes button presses drop. It is not comfortably playable yet.

## Progress

### 2026-11-25, GPU page fault fixed

- **Cause.** Some shaders pick a buffer from the descriptor heap by an index the game supplies. The translator read the slot as a buffer when the runtime's stride table gave it an element size, and as a constant buffer otherwise. Texture slots and empty slots also have no element size, so they were read as constant buffers. On AMD, a texture descriptor read that way gives a wild address and the GPU faults (page fault at `0x5b3b000` in compute shader `fc5e66f8f873d3bb`). D3D12 found no live or recently freed allocation at that address, which rules out a use-after-free.
- **Fix.** The runtime marks real constant-buffer slots in the stride table with their own value. Only those are read as constant buffers; texture and empty slots now read zeros. The fixed shader passes Microsoft's DXIL validator, and all 236 translator tests pass.
- **Result.** Boot-loop run 3 ran for about 18 minutes and reached frame 11,583 with no GPU fault. Earlier runs stopped at frames 2,742 and 2,881. The run ended on a shader-compiler crash in compute shader `f16876929fa64f9f`; the launcher restarted the game, and the session was then stopped by hand.

### 2026-11-25, boot-loop testing

- **Further than before.** Two test runs passed the point where the game used to crash at world load (around frame 2,440). They reached frames 2,742 and 2,881 before a different failure, described under Known issues.
- **No more skipped shaders for compiler crashes.** The 38 pipelines that had been skipped are back in. When a pipeline crashes AMD's shader compiler, its shaders are now translated again on the next launch without the merging optimizer pass, instead of being dropped. Every crashing shader tested offline compiles that way: the 12 from before and one new one, `a567b9ef845fb680`. A pipeline is skipped only if it still crashes after that.
- **New diagnostic switch.** `VICTORY_REUSE_SKIP` turns off chosen kinds of the merging pass, such as `bin`, `call`, `join` or a `dx.op` name. It is used to find which merge a driver fails on, and the shader cache keys on it.

### 2026-11-25, setup and first fix

- **Setup.** The runtime builds and installs with `Setup.cmd`. Shader translation works for about 22,000 of the game's shaders.
- **AMD driver crash, found.** After the main menu the game hung for 10+ seconds ("waiting for a new frame sample") and then crashed with `0xc0000005`. The crash is inside AMD's shader compiler (`amdxc64.dll`), while it compiles some large compute shaders. Our translated shaders pass Microsoft's DXIL validator, so the code is valid and AMD's compiler mishandles it.
- **AMD driver crash, cause.** The crash was reproduced offline by compiling a single shader, without the game. The trigger is one optimizer step: a texture-size query (`getDimensions`) on a heap-indexed texture is computed once and reused in later blocks. AMD's compiler reads through a null pointer on that pattern.
- **AMD driver crash, fix.** The optimizer no longer reuses those size queries; each use computes its own, which costs next to nothing. All 12 crashing shaders that were tested offline now compile. The same shaders still crash without the fix.
- **Safety net.** If the driver crashes while compiling a pipeline, the runtime records it in `cache\pipelines\pipelines-crashed.txt` and the launcher restarts the game automatically.

## Known issues

- **Unstable frame rate (blocking for play).** The frame rate swings enough that many button presses do not register.
- **Shader-compiler crashes still happen.** AMD's compiler still crashes on some shaders, most recently compute `f16876929fa64f9f` after 18 minutes. The game restarts, and the next launch translates that shader without the merging pass. A targeted fix for each pattern is preferred to that fallback.
- **Ordered-pixel shaders are not drawn.** 10 pixel shaders use the console's ordered pixel shading (POPS), which keeps overlapping transparent pixels in draw order. The translator does not support it yet, so those draws are dropped. The planned fix maps them to D3D12 rasterizer-ordered views.
- **Full shader check not finished.** The offline check that builds pipelines for all ~22,000 shaders was stopped before it finished.
- **Inherited issues.** These come from the upstream runtime: some objects render black, frame times can be long in some scenes, and ray tracing is experimental.

## Credits

Built on DexNom Runtime and Marvisak's VIctory. See [README.upstream.md](README.upstream.md) for the original credits and the [license](LICENSE) (GPL-3.0-or-later).
