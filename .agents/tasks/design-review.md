# Design review, round 2: browser-native animated Gaussian-human runtime

Reviewed: `.agents/tasks/design.md` (revision 2, 1,455 lines, read in full) against `.agents/tasks/repo-analysis.md` (read in full) and the upstream source at
`.scratch/Gsplat-unity-virtual-human` @ `8ed42ea` (HEAD confirmed). The step prompt left `{{design}}` and `{{analysis}}` unfilled; I resolved them to those two files.
This supersedes the round-1 review (0 HIGH, 9 MEDIUM, 10 NIT). I re-checked each round-1 finding against the revised text; all 19 are resolved.

**Verdict: CHANGES_REQUESTED.** 0 HIGH, 3 MEDIUM, 10 NIT.

The algorithm port, binding record, handedness handling, asset byte layouts, pass gating and validation plan are sound and checked against source.
The three MEDIUM findings are: a renderer-decision rationale that I verified to be false for the PlayCanvas version the design cites; a canvas-size rule that distorts the image on this very machine; and no concrete procedure for the performance and per-stage profiling deliverable.
Each has a small, concrete fix.

---

## Findings

### F1 (MEDIUM) The stated reason for rejecting PlayCanvas (per-stage timing) is false for `playcanvas@2.23.0`, and 3.3 contradicts 3.4
Where: design 2 (decision paragraph), 3.1 (criterion C2), 3.2 (PlayCanvas row), 3.3 (reason 1, score table), 16.4 (F1).

The design says "C1 and C2 decide it". C2 = "can our single device and one encoder run deform, sort and render so each stage gets its own timestamp queries ... 1 = our compute passes can be timed but the library's internal passes cannot".
Section 2 says "per-stage timing would stop at the engine boundary"; 3.3 reason 1 says a library's internal passes "do not allow" separate timing.
I read the published `playcanvas@2.23.0` tarball:

- `platform/graphics/webgpu/webgpu-graphics-device.js` defines `setupTimeStampWrites(passDesc, name)`. It is called for render passes (`startRenderPass`, line 824) and for compute passes (line 864). Each pass gets `timestampWrites` keyed by its pass name.
- `WebgpuGpuProfiler` owns a 1024-slot timestamp query set and reports per-pass timings through `device.gpuProfiler`.

So the engine's internal passes are timed individually by name. Whether its gsplat sort, projection and draw passes carry distinct names I did not establish (unverified), but the categorical claim is wrong.

A second inconsistency: 3.3 says "the deformer would have to run as an engine `Compute` writing engine-owned storage". 3.4 says the deformer "takes a `GPUDevice` and never creates its own, so it could run on another engine's device".
Both cannot hold. The deformer could run as raw WebGPU on the engine's device (`app.graphicsDevice.wgpu`). What is engine-owned is the read side: `scene/gsplat/gsplat-streams.js` creates every splat stream as a `Texture` (`GSplatStreams.createTexture`).
So the handoff needs a storage-buffer-to-storage-texture pass. That supports the design's C1 = 1, and is the honest cost of adopting PlayCanvas.

Effect on the decision: with C2 = 2 the table reads in-house 20, PlayCanvas 19 (out of 24). The margin is one judgement point, and the deciding facts become C4 (code we own) and the texture conversion pass. Verified from source: 2.23.0 does have a GPU `ComputeRadixSort` in `gsplat-hybrid-renderer.js` and renderer modes AUTO / RASTER_CPU_SORT / RASTER_GPU_SORT / COMPUTE, so G2 holds.
The user asked whether PlayCanvas can be the base renderer. Right now the answer rests on a false premise and a spike that is deferred until after about 1,100 lines have been written and tested.

Fix (choose one, and correct the text in both cases):
1. Keep in-house. Rewrite C2 and 3.3 reason 1 to the facts: PlayCanvas times its own named passes; our reasons are (a) one encoder and a fixed 10-slot schema that we own, (b) an exact bitonic sort that can be compared element by element with `Array.sort`, (c) no per-frame buffer-to-texture conversion pass and no dependence on `GSplatFormat` internals. Rescore C2 for PlayCanvas as 2 (or "1-2, unverified") and show 19 versus 20 as a near tie, with the owner-approval note already present for C4.
2. Or run the half-day spike at the start of M1, before any of `project.wgsl`, `sort.wgsl`, `splat.wgsl` is written, with a go/no-go rule: adopt PlayCanvas if (i) a `Compute` pass fills `GSplatContainer` stream textures from a `DeformedSplatBuffers`-shaped buffer with no CPU readback, (ii) `gpuProfiler` shows separate timings for its sort and draw passes, (iii) the GPU sort path (not the CPU worker) is active. Otherwise record the result in `porting-notes.md` and proceed in-house.

### F2 (MEDIUM) Canvas-size rule clamps each dimension independently, which distorts the image at normal window sizes on this machine
Where: design 6.7 ("Canvas size = CSS size x `devicePixelRatio`, each dimension clamped to 2048 px"), NFR-2, 11.1.

Read literally, `w = min(2048, cssW*dpr)` and `h = min(2048, cssH*dpr)`. This MacBook has a Retina display (devicePixelRatio 2). A 1440x900 CSS canvas is 2880x1800 device pixels, clamped to 2048x1800. The buffer aspect is 1.14, the displayed aspect is 1.60, and the browser stretches the buffer to the CSS box.
The figure renders about 1.4x too wide. The project pass uses `W/H` for the Jacobian and viewport, so it is consistent internally and the distortion only shows on screen, which is exactly how it would slip past the numeric tests.

Separately, NFR-2 says "60 fps at 1920x1080" without saying whether that is CSS pixels or the drawing buffer. With the 2048 clamp and DPR 2 these differ (a 1920x1080 CSS window becomes 2048x1152).

Fix: replace the rule in 6.7 with a uniform scale:
```
s = min(1, 2048 / max(cssW * dpr, cssH * dpr))
width  = max(1, round(cssW * dpr * s))
height = max(1, round(cssH * dpr * s))
aspect = width / height            // used by the projection
```
Add a Playwright check (M1): viewport 1600x900 with `deviceScaleFactor: 2`, assert `|canvas.width/canvas.height - 1600/900| < 0.01`.
Define NFR-2's resolution as the drawing-buffer size (1920x1080) and give the benchmark a `?res=1920x1080` override that sets the buffer size directly (see F3).

### F3 (MEDIUM) No concrete procedure for the performance and per-stage profiling deliverable
Where: design 1.3 (NFR-2), 11.1, 11.2, 12.4, 13 (M1 "60 fps on static data", M7 "final timings table"), A5.

The profiler itself is well specified (10 slots, staging ring, stage mask). What is missing is how the numbers that the user asked for are produced and made comparable:

- No benchmark hook or script. The `__gsplat` hooks list has `setClip`, `setTime`, `renderOnce`, but nothing that runs N frames and returns per-stage and frame statistics. "Final timings table" would be read off the on-screen panel by hand.
- No conditions: headed or headless, drawing-buffer size (see F2), warm-up frames, measured frames, clip, camera, which statistic (mean, p50, p95), what counts as "60 fps". FPS is capped by the display refresh (this laptop's display may run at 120 Hz), so FPS alone cannot show headroom. Report GPU busy time per frame (sum of stage times) against 16.7 ms.
- No guard against a software adapter. Playwright-launched Chrome can hand out a fallback adapter. The M0 smoke test only checks "WebGPU adapter present", so parity would pass and timings would be meaningless.
- Apple-GPU timestamp support was observed for the design's probe, but the design never says whether `timestampWrites` was verified on both compute and render passes. The `render` stage uses a render pass.
- A5 claims the design "works to about 260,000 at interactive rates", with no way to measure it (the synthetic character has about 2,000 Gaussians and the real one 40,000).

Fix: add to the plan (11.2 and M7) a `verify:bench` step in `tools/verify/browser-checks.ts`:
1. Hook `__gsplat.bench({ warmup: 120, frames: 600 })` returns, per stage, median and p95 of GPU ms (only for frames in which the stage ran), CPU anim and encode ms, and the rAF interval distribution; plus adapter `vendor`, `architecture`, `description`, `isFallbackAdapter`.
2. Conditions: headed Chrome, `deviceScaleFactor: 1`, viewport 1920x1080 (buffer 1920x1080), first clip playing, default camera, deformation on, nothing else running; report machine, Chrome version, date.
3. M0 smoke test asserts `isFallbackAdapter === false` and vendor `apple`, and runs one compute pass and one render pass with `timestampWrites`, asserting a non-zero, plausible delta for each.
4. Report headroom (16.7 ms minus the GPU sum) next to FPS.
5. For A5, either add `build-synthetic --count 262144` (tile and jitter the real cloud) and measure, or delete the 260,000 claim.

### F4 (NIT) Keyboard handlers will conflict with focused controls
Where: design 6.7, 9.2. Space toggles play globally and arrow keys orbit, but a focused `<button>`, `<select>` or `<input type=range>` also reacts to Space and the arrows (double toggle, scrubber moves while orbiting). Say that the global handlers ignore events whose target is one of those elements, and that `R` is ignored while typing in an input.

### F5 (NIT) Skeleton overlay is hard-wired to 27 bones and 864 bytes
Where: design 3.4 (`updateSkeleton`), 6.6, 9.3. The segment count is `jointCount - rootCount`, 27 only for the LHM rig. The synthetic character has 2. Size the buffer from the rig: `segments = Float32Array(4 * 2 * boneCount)`.

### F6 (NIT) Binary readers: short files and unchecked fields leave an uncaught-exception path
Where: design 7.2, 7.3, 7.4, 10.1, acceptance 16.
- A file shorter than its header (0 to 31 bytes for GSRG, 0 to 63 for GSPL) reaches "magic" first and throws a `RangeError`, not an `AssetError`. Check `byteLength >= headerSize` first and raise `*_SIZE_MISMATCH`.
- `flags`, `reserved` words and GSPL `reserved[2]` are "zero" but never checked; say reject (preferred) or ignore. `binding.format === "GSRG/1"` is a required key with no rule.
- The loader does not state that parsed vertex, triangle and joint counts equal the manifest values, nor that every triangle index is below `vertexCount` (an out-of-range index reads zero on the GPU and silently deforms the wrong surface). Add both as `MANIFEST_INVALID` / `GLB_MALFORMED`.
- GSPL `bboxMin/Max` are min/max of Gaussian centres (not extents); say so, since the default camera is derived from them (checked: centre (0.0035, -0.4605, -0.0125), 1.2 x diagonal = 2.99 m, matching the stated 3.0).

### F7 (NIT) Acceptance 12 can pass vacuously
Where: design 1.4 #12. "Joint skin matrices at `t = 0` and `t = duration` agree" goes through the sampler that wraps `t mod duration`, so `t = duration` is `t = 0`. Compare the first and last keyframe values of every channel (or sample with wrapping disabled) instead.

### F8 (NIT) M0 exit lists an acceptance item its tooling cannot satisfy
Where: design 13 (M0), 1.4 #20. #20 includes "the baker fails with `BAKE_PROXY_SCALE`", but the baker arrives in M3. Split #20 into 20a (validator, M0) and 20b (baker clause, M3).

### F9 (NIT) Vite configuration details that the scripts need
Where: design 4.1, 12.4, 14. `vite --config web/vite.config.ts` run from the repo root uses `process.cwd()` as root unless the config sets `root`; set `root` to the `web/` directory in the config. `tests/*.browser.ts` sit outside that root, so the dev server needs `server.fs.allow` for the repo root, and the production build that `vite preview` serves needs `import.meta.glob('../../../tests/*.browser.ts')` (a variable dynamic `import()` is not bundled). `verify:browser` must run `vite build` first.

### F10 (NIT) README and reproducibility gaps
Where: design 14, 1.4 #18, 8.2. With `--synthetic` only, the default asset `lhm500m-woman` does not exist, so `npm run dev` fails with `ASSET_FETCH_FAILED`; say to open `?asset=synthetic`. The Blender path is hard-coded while Chrome has `CHROME_PATH`; add a `BLENDER` override. "Clean clone" needs a git repository and a committed `package-lock.json`; this workspace is not a git repository today.

### F11 (NIT) Calibration uses 5 seeds, verification 20
Where: design 12.3.1. The gate has 25 % headroom (0.75 to 1.0) but the extreme-value tail of 20 x 40,000 samples is larger than that of 5 x 40,000. Calibrate on seeds 101..120 so the two sample sizes match and the gate is not flaky.

### F12 (NIT) The renderer has no analytic anchor
Where: design 1.4 #10, 12.3 J. The GPU project pass is compared with a float64 CPU EWA written from the same prose, so a sign error in the Jacobian would match. Add a known-answer check: an isotropic Gaussian of scale `s` on the optical axis at depth `d` gives `sigma_px = sqrt((fx*s/d)^2 + dilation)`; and two Gaussians mirrored about the view axis give mirrored axes.

### F13 (NIT) Small frame-loop details
Where: design 9.1, 9.3. `toggled(deformOn)` must be true on the first frame so the initial `setSource(...)` happens. Clamp the rAF delta (for example to 100 ms) so a returning hidden tab does not jump the animation. Confirm in the M0 smoke test that `timestamp-query` can be requested on this adapter.

---

## Verified assumptions

Checked by me in this round, against source or by running something on this machine.

- **Kernel transcription.** `GsplatSkinning.compute` L340-462 against WGSL in 5.4: `qmul`, all four Shoemake branches of `LookRotationQuat` (component placement checked branch by branch), frame construction (`n`, `t`, `b`, re-orthogonalised `t`), `pProxy = v0 + e1*u + e2*v + n*off`, `rProxy = qmul(tfq, rl)`, `ProxyToScan` application, xyzw output.
- **Baker.** `GsplatRigBakerWindow.cs` L228-300: unclamped least-squares barycentrics with the `det` formula, `signedOff`, and `restLocalRot = Inverse(LookRotation(nrm, tan)) * (rotProxyFromScan * gRotScan)` with the PLY wxyz to xyzw reorder. Design 5.2 steps 2-5 match.
- **Handedness.** `r = up x f = t x n`, `u = f x r = n x (t x n) = t`, so the columns are `[t x n | t | n]` with determinant +1. The math is mirror-agnostic, so baking on the unmirrored right-handed proxy and reconstructing there is consistent. The quaternion to matrix columns in 6.2 are the standard ones.
- **Projection formulas.** Jacobian, 1.3 clamp, eigen step, extent formula and the Herf key order checked by algebra against the reference 3DGS; sort dispatch count 1 + 28 + 7 = 36 at Np = 65,536; `select` argument order in the bitonic step.
- **Numbers and files.** Input paths and byte sizes in 8.1 match the clone exactly (PLY 2,720,415; FBX 1,458,976; OBJ 4,201,976; rigproxy 1,035,540; riggedgsplat 1,280,032). Default camera from the sample bbox is consistent.
- **Machine.** Node v26.5.0, npm 11.17.0, Apple M3 Max, Chrome 154.0.8037.95, Blender 4.1.1. Every pinned npm version exists (`vite@8.3.2`, `typescript@5.9.3`, `@webgpu/types@0.1.74`, `@types/node@26.6.4`, `@gltf-transform/{core,extensions,functions}@4.5.1`, `playwright-core@1.63.0`); Vite 8.3.2 engines are `^20.19.0 || >=22.12.0`; TypeScript `latest` is 7.0.2.
- **Spark 2.3.1 (downloaded and searched).** The bundle contains no `navigator.gpu` and no `WebGPURenderer`; it uses GLSL3 `ShaderMaterial`, `WebGLRenderTarget`, Worker-based sorting and `readRenderTargetPixelsAsync` readbacks. This upgrades the G1/G2 rejection from docs-based to confirmed.
- **PlayCanvas 2.23.0 (tarball).** Hybrid renderer with GPU `ComputeRadixSort`; modes AUTO / RASTER_CPU_SORT / RASTER_GPU_SORT / COMPUTE; splat streams are `Texture`s; per-pass `timestampWrites` for render and compute passes (basis of F1).
- **Revision 2 fixes.** Each round-1 finding is addressed in the text: seam types and gating (3.4, 9.3), GSPL offsets formula (7.2), `validateProxyToScan` (7.1), tolerance model and pose generator (12.3.1), baker host rule (8.3), `test` script and `verbatimModuleSyntax` (4.1, 14), error-code list (10.1.1).

## Unverified or wrong assumptions

Wrong (verified):
- "A library's internal passes cannot be timed per stage" for PlayCanvas 2.23.0 (F1).
- "The deformer would have to run as an engine `Compute`" (contradicted by 3.4; F1).
- "Each canvas dimension clamped to 2048" as a sizing rule (F2).

Unverified (not checked by me, or only taken from the design):
- The project, splat and overlay WGSL do not exist yet; EWA correctness depends on the oracles (F12).
- Playwright driving system Chrome with a real (non-fallback) adapter; `timestampWrites` on a render pass on this GPU; every performance figure (F3).
- Whether PlayCanvas names its gsplat sort, projection and draw passes separately; whether any route lets a `GSplatFormat` read a storage buffer rather than a texture.
- The design's own probes in 12.5 (shader compile, sort equality, rest-pose parity numbers). I did not re-run them; the analysis reports the same rest-pose figures from an independent NumPy port.
- Blender export flags, Rocketbox retarget quality, the 50 % coverage guard and the calibration constant `c`.
- Unity itself was never run (A4); parity is to the HLSL transcription plus shipped data.

Housekeeping: I removed the scratch directories `/tmp/pcpack.*` and `/tmp/sparkpack.*` left by earlier steps, plus my own `/tmp/sparkrev.*` download. Nothing inside the workspace was changed except these two review files.
