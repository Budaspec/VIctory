A fork of the DexNom community victory fork by Budaspec.

## At a glance

| | |
|---|---|
| **Platform** | AMD Radeon RX 9060 XT 16 GB (RDNA 4), driver 32.0.32015.7029, Ryzen 5 9600X, 32 GB RAM, Windows 11 Pro |
| **Milestone** | Playable: in the world, character control, HUD working |
| **Story progress** | Prologue cleared: reached Jason's first mission, "Gone Fishing", and played part of it |
| **Rendering** | The world draws, but many textures and objects are black or missing |
| **Performance** | About 3 to 6 fps in the world; some frames take 1 to 2 seconds |
| **Stability** | Rarely crashes: recent test runs of 8 to 30 minutes with no crash or GPU fault |
| **Evidence** | [Gameplay video (YouTube)](https://youtu.be/HYebox6DnVQ) |

**What works**
- Builds from source with `Setup.cmd`, based on DexNom Runtime (Marvisak's VIctory); GPL-3.0.
- Runs on RDNA 4 (RX 9060 XT): fixes for AMD shader-compiler crashes on this driver (resource-size queries, a merging optimizer pass) and for a GPU page fault from reading texture descriptors as constant buffers.
- Gamepad presses register even at low frame rates (presses between frames are latched).

**What's blocking**
- Low, uneven frame rate: game threads wait on runtime locks (descriptor mirror, submission, texture uploads). That's being worked on now.
- Many textures and objects are black or missing.
- Tested on one PC only.

## Status

**Nearly crash-free.** On an AMD Radeon RX 9060 XT (driver 32.0.32015.7029) the game loads into the world and ran for over 30 minutes in the latest test with no GPU fault. Correction to the earlier note on this page: that test did crash 6 times in its first 6 minutes. Three mesh and pixel pipelines crashed AMD's shader compiler (a different spot in it, `amdxc64.dll+0xe8d9e5`), twice each. The launcher restarted the game each time, then skipped those 3 pipelines, and the next 24 minutes ran clean. Every compute shader that crashed the compiler before now compiles; the 3 mesh/pixel pipelines are being investigated.

**Current focus: frame rate and frame timing.** The world runs at about 3 to 6 fps, and some frames take 1 to 2 seconds, so it is not comfortably playable yet. The first goal is a frame time stable enough that the game is mostly playable, keeping the current stability. The end goal is 30 fps without major dips. Missing or black textures and objects are known and set aside for now.

## Progress

### 2026-11-25, frame timing work and a regression fix

- **Regression found and fixed.** The fix for the compute-shader compiler crashes was also applied to pixel shaders, which made AMD's compiler crash elsewhere (`amdxc64.dll+0xe8d9e5`) on 3 mesh/pixel pipelines. That caused run 4's 6 restarts. The fix now applies only to compute shaders. All 15 compute shaders and the 6 mesh/pixel shaders compile, and test run 6 had **zero crashes** with all of them in.
- **Frame-time spikes reduced.** Texture uploads for shaders that pick textures by index are capped at 24 ms a frame (100 ms in the first minute), down from 250 ms (1.5 s). Crash breadcrumbs, which cost about 156 ms of CPU a frame, are now off in test runs. Run 5 compared with run 4: median frame 230 ms vs 256 ms, worst frame 2 s vs 23 s. Frames of 1 to 2 seconds still happen in about 12% of seconds.
- **Where the slow frames go.** In the 1 to 2 second frames the game uses little CPU and goes about a second without submitting work, so its threads are waiting. A sampling profile (every thread's stack once a second for 6 minutes, 20,216 game-thread samples) shows that when the game's threads are in runtime code, they mostly wait on runtime locks. Ranked by samples:
  1. `HostHeap::mirror`: the descriptor heap mirror lock, taken when draws translate descriptor slots (99).
  2. `Host::submit_part`: the submission lock (82).
  3. `texture::upload_from`: the texture upload lock (78).
  4. `draw::State::bind` (48).

  Next step: shorten or split these locks so game threads stop queuing behind each other.

### 2026-11-25, compiler crashes, dropped presses, frame pacing

- **Shader-compiler crash fixed.** Compute shader `f16876929fa64f9f` crashed AMD's compiler with every optimizer setting. A new diagnostic (`VICTORY_DROP_OPS`) removes chosen operations from a shader, and it showed that resource-size queries on resources picked from the descriptor heap trigger the crash. The translator now marks the heap indices those queries use as non-uniform. That computes the same result and avoids the crash. All 15 shaders that ever crashed the compiler now compile. `VICTORY_SIZE_NONUNIFORM=0` turns this off.
- **Dropped button presses.** The game finds a press by comparing the current controller reading with the previous one. The runtime made a new reading every 8 ms whenever a stick moved, so at 3 to 6 fps a tap usually started and ended between two of the game's reads. Readings since the game's last read are now merged into one, which includes every button pressed in between. Its previous reading is the one the game saw last. A test reproduces the old failure. `VICTORY_INPUT_LATCH=0` turns this off.
- **Frame pacing.** Every 16th frame, the pass that prepares heap buffer views rescanned all ~83,000 slots, a hitch of 0.1 s or more. The rescan is now spread over 16 frames, and the slot part of the pass dropped to about 10 ms a frame. The frame-rate line now splits the pass into slots, memory copies and textures.
- **Result.** Test run 4 ran for over 30 minutes and passed frame 6,000 with no GPU fault. It restarted 6 times in its first 6 minutes on 3 new mesh/pixel compiler crashes (see Status), then ran 24 minutes clean.

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

- **Low, uneven frame rate (blocking for play).** The world runs at about 3 to 6 fps. Measured causes, largest first:
  - Uploads of textures that shaders pick by index run on the frame's own thread, with a budget of up to 250 ms a frame (1.5 s during the first minute).
  - Change detection on game memory checks about 11 GB a frame, nearly all of it unchanged.
  - Copying GPU results back to game memory takes 70 to 130 ms a frame.
  - Some frames take 1 to 2 seconds while the game's own threads have not submitted any work.
- **Ordered-pixel shaders are not drawn.** 10 pixel shaders use the console's ordered pixel shading (POPS), which keeps overlapping transparent pixels in draw order. The translator does not support it yet, so those draws are dropped. The planned fix maps them to D3D12 rasterizer-ordered views.
- **Full shader check not finished.** The offline check that builds pipelines for all ~22,000 shaders was stopped before it finished.
- **Inherited issues.** These come from the upstream runtime: some objects render black, frame times can be long in some scenes, and ray tracing is experimental.

## Credits

Built on DexNom Runtime and Marvisak's VIctory. See [README.upstream.md](README.upstream.md) for the original credits and the [license](LICENSE) (GPL-3.0-or-later).
