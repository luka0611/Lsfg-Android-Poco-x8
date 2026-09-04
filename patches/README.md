# Poco X8 — diagnosis and fixes

Analysis of why base FPS drops even at `multiplier=2` / `flow_scale=0.25`, and why
Shizuku capture never starts.

Analysed against:

| Repo | Commit |
|---|---|
| `FrankBarretta/LSFG-Android-Application` | `b8475419` (the commit this fork pins) |
| `FrankBarretta/lsfg-vk-android` | `release` @ `3e89e543` |

The fixes are committed to your app fork, on branch
`claude/lsfg-mobile-performance-nhe9vx`:

<https://github.com/luka0611/lsfg-android-application-poco-x8/tree/claude/lsfg-mobile-performance-nhe9vx>

**Not built as an APK and not run on a device** — there is no Android SDK/NDK in the
environment this was written in. What *was* verified: `kotlinc` type-checks every changed
Kotlin file against an `android.jar` with no error referencing the new code, and the
rewritten `captureContentHash` compiles under `clang -std=c++20`. Treat it as reviewed and
type-checked, not tested. The reasoning behind each change is below so you can judge it
yourself.

---

## 1. Shizuku capture: the reflection target no longer exists on Android 14+

`PrivilegedScreenCapture` (used by **both** the Shizuku and the root capture paths)
resolves the screenshot API by reflection and requires a **single-argument**
`captureDisplay`:

```kotlin
// PrivilegedScreenCapture.kt
captureDisplay = findSingleArgMethod(captureClass, "captureDisplay", argsClass),
```

```kotlin
private fun findSingleArgMethod(cls: Class<*>, name: String, argClass: Class<*>): Method =
    ...firstOrNull { it.name == name && it.parameterTypes.size == 1 && ... }
        ?: throw NoSuchMethodException(...)
```

That signature was correct on Android 12/13. On Android 14 (U) the screenshot entry
points moved to `android.window.ScreenCapture` **and became asynchronous**:

```java
// android/window/ScreenCapture.java (Android 14+)
public static int captureDisplay(DisplayCaptureArgs captureArgs,
                                 ScreenCaptureListener captureListener)
public static SynchronousScreenCaptureListener createSyncCaptureListener()
```

There is no one-argument overload any more. So on Android 14/15:

1. `buildBackend("android.window.ScreenCapture", …)` throws `NoSuchMethodException`.
2. `buildBackend("android.view.SurfaceControl", …)` throws too — the class kept the name
   but not that overload.
3. `firstNotNullOfOrNull` yields null →
   `IllegalStateException("No privileged ScreenCapture backend with UID filter is available")`.
4. `ShizukuCaptureUserService.runCaptureLoop` catches it and reports
   `"Shizuku capture unavailable: …"`, then sets `running = false`.

The capture loop exits before a single frame. Nothing downstream is at fault — every UID
filter, display token and builder lookup in that class is fine; the call itself is
unreachable.

Grepping confirms nothing in the tree handles the listener form:

```
$ grep -rn "ScreenCaptureListener\|createSyncCaptureListener" app/src/
(no matches)
```

**Fix:** probe for `createSyncCaptureListener()` +
`captureDisplay(args, listener)` first, and fall back to the legacy one-arg form on older
releases. A sync listener holds exactly one result, so a fresh one is created per capture
and `getBuffer()` blocks for it.

This also fixes root capture, which shares the same class.

### Check it on device

```sh
adb logcat -s PrivilegedCapture ShizukuUserCapture ShizukuCapture
```

Before the patch you should see the backend warnings followed by
`No privileged ScreenCapture backend with UID filter is available`. After it, expect
`android.window.ScreenCapture: using async captureDisplay(args, listener)` and then
`captured frame #1 …`.

If instead you get a `SecurityException` at that point, the problem is a different one
(HyperOS restricting `READ_FRAME_BUFFER` for the shell UID) and the fix is elsewhere —
worth reporting back.

---

## 2. Base FPS: the cost is fixed, which is why multiplier and flow scale don't move it

The flow-scale mapping is *correct* — I checked it against the framegen library rather
than assuming:

```cpp
// lsfg_render_loop.cpp
const float userFlow = (cfg.flowScale >= 0.25f && cfg.flowScale <= 1.0f) ? cfg.flowScale : 1.0f;
g.flowScale = 1.0f / userFlow;   // 0.25 → 4.0
```

```cpp
// lsfg-vk-android/src/context.cpp:103 — same reciprocal upstream
conf.hdr, 1.0F / conf.flowScale, conf.multiplier - 1, …
// lsfg-vk-android/framegen/v3.1_src/shaders/mipmaps.cpp:33
inImg_0.getExtent().width / vk.flowScale
```

So `0.25` really does shrink the optical-flow pyramid 4×, and it really is the cheapest
setting. The reason it doesn't help is that **the flow pyramid isn't where the money
goes**. Four other things dominate, and neither slider touches any of them.

### 2a. A full-resolution CPU lock of a GPU buffer, on the capture thread, every frame

`pushFrame` hashes every captured frame to detect duplicates:

```cpp
// lsfg_render_loop.cpp — comment above captureContentHash
// Cost: ~50-100 μs total — 1× AHardwareBuffer_lock (the AHB was allocated
// with CPU_READ_OFTEN so the lock is cheap), 64× 4-byte reads, unlock.
```

```cpp
AHardwareBuffer_lock(ahb, AHARDWAREBUFFER_USAGE_CPU_READ_OFTEN, -1, nullptr, &ptr)
```

**That comment is describing the wrong buffer pool.** It's true of the framegen *output*
AHBs, which `ahb_image_bridge.cpp` allocates itself with `CPU_READ_OFTEN` and documents at
length. But `captureContentHash` runs on the *capture* buffers, which come from the
`ImageReader` in `CaptureEngine.setLsfgMode`:

```kotlin
android.hardware.HardwareBuffer.USAGE_GPU_SAMPLED_IMAGE or
        android.hardware.HardwareBuffer.USAGE_GPU_COLOR_OUTPUT
```

No CPU-read usage at all. Locking for an access the allocation never declared is outside
the `AHardwareBuffer` contract, and drivers split two ways — both bad:

- **Lock refused** → `captureContentHash` returns 0 → the dedup branch is skipped, so
  duplicate frames reach the worker. This is less severe than it first looks: `pushFrame`
  bounds `g.pending` at `queueDepth` and drops the oldest, so the pipeline is *not*
  unbounded — the cost is wasted interpolation on identical pairs, plus a failed ioctl per
  captured frame. The HUD's "real fps" also freezes, because `uniqueCaptures` stops
  incrementing — worth checking on your device, it's a free diagnostic.
- **Lock honoured** → the driver resolves the whole compressed surface (AFBC on Mali, UBWC
  on Adreno) to linear and waits on the producing GPU work first — a full-resolution
  detile plus a pipeline sync, on the capture thread, once per frame.

Either way it back-pressures SurfaceFlinger's VirtualDisplay, which is exactly the
"base FPS drops even when I'm not in a game" symptom.

**Fix:** check `desc.usage` before locking and self-disable hashing when the buffers
aren't CPU-readable (or after 8 consecutive lock failures) instead of retrying forever.

Declaring `USAGE_CPU_READ_OFTEN` on the `ImageReader` was the other option, and I decided
against it *because* you're on Mali: it makes gralloc pick a CPU-cached, uncompressed
layout, which loses AFBC and costs bandwidth on every framegen read of that buffer. On a
bandwidth-bound part that's the wrong trade — better to lose duplicate-skip, which
`queueDepth` already partly covers, than to lose compression on the hottest surface in the
pipeline. If it turns out the hash was carrying more weight than expected, the one-line
experiment is adding `HardwareBuffer.USAGE_CPU_READ_OFTEN` to the `ImageReader` usage in
`CaptureEngine.setLsfgMode` and comparing.

### 2b. The overlay pins your panel to its maximum refresh rate

```kotlin
// OverlayManager.kt
requestedRefreshRateHz = wm.defaultDisplay.supportedModes
    .maxOfOrNull { it.refreshRate }
    ?: wm.defaultDisplay.refreshRate
```

…and that value is handed to `Surface.setFrameRate(…, CHANGE_FRAME_RATE_ALWAYS)` for as
long as the overlay lives. The **Refresh override** setting in the drawer only feeds the
native pacer's `setVsyncPeriodNs` — it never reaches `setFrameRate`. So selecting 60 Hz
today still leaves the panel pinned at 120.

At 120 Hz, SurfaceFlinger composites both the screen *and* the capture VirtualDisplay
twice as often as at 60. Add a full-screen `TRANSLUCENT` overlay, which generally forces
client (GPU) composition instead of hardware overlay planes, and the fixed cost roughly
doubles for no benefit when the target app renders at 30–60.

**Fix:** make `requestedRefreshRateHz` honour the override, capped at what the panel
supports. AUTO still means maximum, so nobody's defaults change.

### 2c. Two full GPU pipeline drains per frame

```cpp
// copyAhbImage — once per real frame
g.vk.fn.vkQueueWaitIdle(g.vk.computeQueue);
```
```cpp
// workerThread — once per real frame
if (g.performanceMode) LSFG_3_1P::waitIdle();
else                   LSFG_3_1::waitIdle();
```

The second is `vkDeviceWaitIdle` on framegen's device. The code is honest about why:

> Framegen and our session use different VkDevices, so vkDeviceWaitIdle on either is
> necessary — without an explicit shared semaphore this is the only correct sync.

Correct, but it serialises CPU and GPU completely: no frame's GPU work overlaps the next
frame's CPU work, so the pipeline runs at the *sum* of every stage rather than the max.
`blitOutputToWindow` then CPU-locks the output AHB and memcpys it on top.

**Not in the patch** — this is the real fix and it's a proper piece of work, not a hunk.
`VK_KHR_EXTERNAL_SEMAPHORE_FD` is already in the required extension list in
`android_vk_session.cpp`, so the pieces are there: export a semaphore from the framegen
device, import it into the session device, and have the blit wait on it instead of
draining. I'd want to do that with a device in hand.

### 2d. Everything runs at full native resolution

There is no capture-resolution setting — `LsfgConfig` has `flowScale`, `multiplier`,
`gpuUpscaleFactor`, `npuUpscaleFactor`, but nothing that scales the capture itself. The
`VirtualDisplay`, the `ImageReader`, framegen's I/O images and the swapchain blit all run
at the panel's native resolution.

**Built.** There is now a **Capture scale** slider in the drawer, next to Flow scale:
0.50–1.00, default 1.00 (so nothing changes until you move it). It scales the
`VirtualDisplay`, the `ImageReader`, framegen's images and the output image together,
while the overlay Surface stays native.

It works because both output paths already upscale, which I checked before building it
rather than assuming: the WSI path blits `src.extent` → `swap.extent` with
`VK_FILTER_LINEAR`, and the CPU path calls `ANativeWindow_setBuffersGeometry` with the
produced size and lets SurfaceFlinger scale. So only the render dimensions move;
`setOutputSurface` keeps the native geometry throughout.

**0.7 is the setting to try first** — roughly half the pixels, and through a 2×
interpolation most people won't see it. 0.5 is a quarter of the pixels and is where flow
tracking starts to visibly suffer.

---

## What to try right now, before rebuilding anything

These need no code changes and will tell us which of the above dominates on your device:

1. **Set Refresh override to 60 Hz** in the drawer. On your *current* build that only
   changes pacing, not the panel pin (2b) — so if it already helps noticeably, pacing is a
   factor; if it does nothing, that's consistent with the pin being the problem, and the
   new build should help because the override now reaches `setFrameRate` too.
2. **Turn off all post-processing** — NPU, GPU and CPU stages each add a full-resolution
   pass. `nnapi_postprocess.cpp` is 745 lines of per-frame work.
3. **Turn off the frame graph HUD.** It polls native counters at 5 Hz and drives an
   overlay redraw; the FPS counter at 1 Hz is much cheaper.
4. **Drop queue depth to 2.** Default is 4 (`LsfgPreferences.QUEUE_DEPTH`); each slot is a
   full-resolution AHB held live.
5. **Watch for thermal throttling.** `FLAG_KEEP_SCREEN_ON` plus a pinned 120 Hz panel plus
   sustained GPU load will throttle a mid-range SoC within minutes, and that shows up as
   base FPS decaying over time rather than dropping immediately. If your FPS is fine for
   the first minute and sags after, this is the cause and the patch's 2b hunk is the
   relevant one.

## What would help most to collect

```sh
adb logcat -s LSFG PrivilegedCapture ShizukuUserCapture ShizukuCapture lsfg_native > lsfg.log
```

Three things in there settle the open questions above:

- the `frame profile (avg over N): copy=… present=… waitIdle=… blitWork=…` line the render
  loop logs periodically — it says directly which stage dominates;
- the `pushFrame #N ahb=… usage=0x…` lines — the usage bits confirm whether the capture
  buffers are CPU-lockable on your device, which decides the 2a A/B;
- your exact model and Android version — Poco X8 and X8 Pro are different SoCs and GPU
  vendors, and the AFBC/UBWC reasoning in 2a differs between them.

---

## 3. "Base FPS stuck at 50, very stuttery" — pacing is bypassed exactly when it's needed

This is a separate mechanism from §2, and it is the most likely cause of the stutter.

Each captured frame produces `multiplier - 1` generated frames plus the real one. The
worker spaces them across the capture interval:

```cpp
auto remainingBudget = captureInterval - (now - frameWorkStartedAt);
if (remainingBudget < State::Clock::duration::zero()) remainingBudget = 0;
const auto slotCount = static_cast<int64_t>(g.outputs.size() + 1);
const auto step = remainingBudget / slotCount;
```

`captureInterval` is an EMA of the delta between the capture timestamps of frames **the
worker actually processed** — `prevCaptureTimestampNs` is only updated for dequeued frames,
and frames dropped by the `queueDepth` bound never touch it. While the worker keeps up,
that delta is the source's render interval, which is what the pacer wants. Once the worker
is the bottleneck it becomes *the worker's own period* instead, and the pacer starts
measuring itself.

That feedback loop has a stable bad equilibrium. Worker period = copy + present + waitIdle
+ blits + sleeps, so

```
remainingBudget = captureInterval − (copy + present + waitIdle) = blits + sleeps
```

Start from sleeps = 0. Then `step = blitTime / slotCount` — a few ms at most, which is
below the `period − slack` threshold (6.33 ms at 120 Hz), so no separation is inserted,
so sleeps stay 0 on the next cycle, which keeps `step` small. The pacer cannot bootstrap
its way out: the budget that would let it separate posts is only available if it had
separated them already. That is the "stuck" in "stuck at 50".

Both layers of vsync alignment switch themselves off in that state:

```cpp
deadline += step;
if (step > State::Clock::duration::zero()) {          // ← skipped entirely at step == 0
    deadline = sleepUntilVsyncAligned(deadline, lastPostedAt, step);
}
```

```cpp
const auto minSeparatedSlot = period - slack;
if (slotBudget < minSeparatedSlot) {                   // ← 8.33ms − 2ms = 6.33ms at 120 Hz
    std::this_thread::sleep_until(deadline);           //   any smaller step: no separation
    return deadline;
}
```

So every generated frame is posted back-to-back with the real one, with no spacing. The
function's own comment says what that produces:

> if the computed deadline lies in the same vsync slot as that post, we push it to the NEXT
> boundary so the SurfaceFlinger queue doesn't collapse two buffers onto one flip (**which
> is what produces the steady-state "bunched" stutter we see with multiplier≥2**)

The guard disables that protection in precisely the regime where the collision happens.
Pacing works when there is idle budget and stops working as soon as the GPU is saturated.

Consequences differ by output path, and both are consistent with the report:

- **WSI swapchain path** (the default when no NPU/CPU post-processing is on): present mode
  is `VK_PRESENT_MODE_FIFO_KHR`, so nothing is dropped — the two frames are shown at
  consecutive vsyncs instead of at their correct 10 ms spacing. Interpolated frames
  displayed at the wrong times *is* judder, and `vkAcquireNextImageKHR` back-pressure then
  throttles the worker.
- **CPU blit path**: `ANativeWindow_unlockAndPost` twice inside one vsync means
  SurfaceFlinger latches the newer buffer and drops the older, so the generated frame is
  computed at full cost and then discarded — the displayed rate collapses back to the real
  capture rate, which is what "base FPS stuck at 50" looks like.

### Which of those two applies here: the WSI one

The CPU blit path is selected whenever NPU or CPU post-processing is enabled:

```cpp
const bool cpuPostActive = g.npuPostProcessing || g.cpuPostProcessing;
if (kEnableWsiSwapchain && !cpuPostActive && g.vk.hasSwapchain && ...) { /* WSI */ }
```

On this build neither can be enabled. Both default to false, and every entry point to the
image-quality UI is compiled out:

```kotlin
// FeatureFlags.kt
internal const val SHOW_IMAGE_QUALITY: Boolean = false
```

`SettingsDrawerOverlay.kt:566` gates the drawer's GPU/NPU/CPU sections on it, and
`HomeScreen.kt:507` gates the card that navigates to `PARAMS_IMAGE_QUALITY`. The route is
still registered in the NavHost, but nothing reaches it — so the screen is unreachable and
the two prefs keep their `false` default unless an older build once wrote them.

So `cpuPostActive` is false, the WSI swapchain path is the live one, and the drop variant
above does **not** apply. What remains is the FIFO variant: `VK_PRESENT_MODE_FIFO_KHR`
never drops, so the bunched frames are all presented, one per vsync, at the panel's cadence
instead of at their correct spacing — and `vkAcquireNextImageKHR` on a 3-image swapchain
then throttles the worker to roughly one cycle per (frames-posted × vsync period).

Confirm which path a session actually took with:

```sh
adb logcat -s LSFG lsfg_native | grep "Output surface attached"
```

`path=WSI` is the expected one. `path=CPU (post-process=on)` would mean a stale pref did
get set and the drop variant is live after all.

### Why this isn't fixed in the branch

Two candidate fixes, and picking between them needs measurements from the device rather
than a guess:

1. Drop the `slotBudget < minSeparatedSlot` early-return so posts are always separated by a
   vsync. Simple, but it makes the worker sleep ~8 ms per capture when it is already behind,
   which would *lower* the real frame rate.
2. Post only as many generated frames as there are whole vsync slots in the remaining
   budget, aligned, and skip the rest. Strictly better in principle — the skipped frames
   were being discarded or mistimed anyway, and not computing them frees GPU time — but it
   is a rewrite of the frame scheduler.

Changing frame scheduling blind, with no device to measure on, is how a build gets worse
instead of better. The instrumentation already in the render loop settles it in one line.

### The measurement that decides it

```sh
adb logcat -s LSFG lsfg_native | grep "frame profile"
```

```
frame profile (avg over N): copy=… present=… waitIdle=… blitWork=… wallEnd=… queue=… latency=…
```

- `waitIdle` dominant → the cross-device sync in §2c is the bottleneck; the capture-scale
  slider is the lever, and a shared semaphore is the real fix.
- `blitWork` dominant → the CPU blit path is somehow active (see above) or the GPU
  post-processing stage is heavy.
- `copy` dominant → full-resolution AHB copies; capture scale again.
- All small but `wallEnd` large → the time is going to pacing sleeps, and fix (2) above is
  the answer.

Also worth capturing, since it measures the stutter directly rather than inferring it:
`getRecentPostIntervalsNs` feeds the frame-graph HUD with real inter-post intervals. A
clean 2× looks like evenly spaced posts; the bunching described above looks like pairs of
posts with a gap after each pair.

---

## 4. Measured on device — what the numbers settled

Benchmark report + logcat from the actual hardware. Device is **not** what I assumed
earlier: Xiaomi `2511FPC34G` / `klee`, **MediaTek mt6899**, **Mali-G720 MC8**,
**Android 16 (SDK 36)**, panel modes 1268x2756 @ 30/60/90/120. Target app
`com.xd.TLglobal` (Torchlight), which renders ~60 fps unassisted.

Benchmark runs used capture scale 0.70 → `render_size = 1928x888`. That is exactly
`scaleDimension(2756, 0.70) = 1928` and `scaleDimension(1268, 0.70) = 888`, so the new
slider does what it was built to do.

### 2a is confirmed outright

```
pushFrame #1 ahb=2756x1268 stride=2816 fmt=1 usage=0x300
```

`0x300` = `GPU_SAMPLED_IMAGE | GPU_COLOR_OUTPUT`, every `CPU_READ` bit clear. Exactly the
predicted allocation. Consequence, in all three runs:

```
real_fps        = 0.00
unique_captures = 0
```

`captureContentHash` can never succeed on this device, so the metric was pinned at zero —
which is also what the user saw in the HUD. Fixed by counting delivered captures once
hashing is known impossible.

### waitIdle is the bottleneck, and it is not close

| run | copy | present | **waitIdle** | blit | total | waitIdle share |
|---|---|---|---|---|---|---|
| x2 | 4.31 | 1.70 | **12.29** | 3.54 | 25.55 | **48%** |
| x3 | 4.12 | 2.15 | **19.95** | 5.29 | 39.31 | **51%** |
| x4 | 3.84 | 3.01 | **24.04** | 8.60 | 51.33 | **47%** |

Roughly half of every frame is spent in the cross-device `vkDeviceWaitIdle` of §2c.

Capture scale earns its place against the same measurement — 2x at 1.00 (from logcat)
versus 2x at 0.70 (benchmark): waitIdle 19.0 → 12.29 ms (−35%), total 34.5 → 25.55 ms
(−26%).

### 3 is confirmed, and the model predicts the exact numbers

| run | vsync_alignment | stalls | jitter | pacing_min |
|---|---|---|---|---|
| x2 | 21.3% | 0 | 0.613 | 0.61 ms |
| x3 | 1.6% | 5 | 0.828 | 0.50 ms |
| x4 | **0.0%** | 14 | 0.985 | 0.54 ms |

Posts landing 0.5 ms apart is the bunching, measured. Working the model with the session's
actual pacing preset (`slack=1.5ms`, so `minSeparatedSlot = 8.33 − 1.5 = 6.83 ms` at
120 Hz):

| run | remainingBudget = total − (copy+pres+wait) | step = budget / slotCount | ≥ 6.83 ms? |
|---|---|---|---|
| x2 | 25.55 − 18.30 = 7.25 | 3.63 | no |
| x3 | 39.31 − 26.22 = 13.09 | 4.36 | no |
| x4 | 51.33 − 30.88 = 20.45 | 5.11 | no |

`step` never reaches the threshold, so alignment never engages — matching 21.3 / 1.6 / 0.0%.

### The conclusion that changes the plan

**The pacing fix cannot be made to work first.** Both options in §3 assumed there was budget
to redistribute. There is not: at x3 the GPU work is 26.2 ms of a 39.3 ms capture interval,
and three posts spaced one vsync apart need 20.5 ms that does not exist. Forcing separation
would just stall the worker and cut the real rate further.

Remove `waitIdle` and the arithmetic inverts: work drops to ~6.3 ms of a 39 ms interval,
leaving ~33 ms to place three posts ~11 ms apart — comfortably above the 6.83 ms floor.
**Pacing becomes fixable only after the semaphore work, and is largely fixed by it.**

So §3's "two candidate fixes" is superseded: the answer is neither, it is §2c.

### The semaphore path already exists in the API

```cpp
// framegen/public/lsfg_3_1.hpp
void presentContext(int32_t id, int inSem, const std::vector<int>& outSem);
//   @param inSem  Semaphore to wait on before starting the generation.
//   @param outSem Semaphores to signal once each output image is ready.
```

The Android wrapper passes neither:

```cpp
std::vector<int> outSems;  // empty
LSFG_3_1::presentContext(g.framegenCtxId, /*inSem*/ -1, outSems);
```

and framegen honours them only when `inSem >= 0` (`context.cpp:142,166,216`). The catch is
the handle type: framegen imports with `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_OPAQUE_FD_BIT`
(`framegen/src/core/semaphore.cpp:63`), while Android's native external type is `SYNC_FD` —
the same mismatch the `createContextFromAHB` comment describes for *memory*. Whether Mali
supports OPAQUE_FD semaphores decides if this is an app-only change or needs framegen
changing too, so the build now probes and logs it:

```
external semaphore OPAQUE_FD: exportable=? importable=? — semaphore-based framegen sync ...
```

### Secondary observations

- **x4 is counterproductive.** posted_fps peaks at x3 (76.40) and *falls* at x4 (72.81)
  while stalls nearly triple (5 → 14). x3 is the practical ceiling on this device.
- **Real rate falls as multiplier rises** — derived `posted − generated`: 30.5 (x2),
  25.5 (x3), 18.2 (x4), against ~60 fps unassisted. At x2 the total (60.9) merely matches
  what the game already did on its own, with 67–95 ms of added latency.
- **Crash at 4x + capture 1.00.** Last line before process death is
  `Re-init LSFG context pass=1 2756x1268 render=2756x1268 multiplier=4`. The same 4x run
  completes fine at 0.70 (half the pixels), so this looks like an allocation failure at
  full resolution — three output AHBs at ~14 MB each plus input slots plus framegen's
  pyramid. Not yet confirmed; the crash file itself was not captured.
- **`display_refresh = 60.0` in the report is the idle rate**, sampled after the session
  ended — not what the panel ran at. `posted_fps = 76.40` could not have been achieved
  against a 60 Hz panel, so it was at 90 or 120 during the run.
- **The log file reached 176 MB.** `LsfgLog.append` opens a `FileWriter` per call with no
  rotation or size cap. Worth bounding regardless of anything else here.
