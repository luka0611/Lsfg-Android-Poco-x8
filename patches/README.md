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
worker actually processed**. Frames dropped by the `queueDepth` bound never update it. So
once the worker is the bottleneck, `captureInterval` converges on *the worker's own
throughput*, and `remainingBudget = captureInterval - workTime` converges on **zero**.

`step` then goes to zero, and both layers of vsync alignment switch themselves off:

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

### The setting to check first, before any rebuild

The CPU blit path is selected whenever **NPU or CPU post-processing is enabled**:

```cpp
const bool cpuPostActive = g.npuPostProcessing || g.cpuPostProcessing;
if (kEnableWsiSwapchain && !cpuPostActive && g.vk.hasSwapchain && ...) { /* WSI */ }
```

On that path every posted frame costs a full-resolution `AHardwareBuffer_lock`, an
`ANativeWindow_lock`, and a per-row `memcpy` — on top of losing the drop-free FIFO queue.
**Turn NPU and CPU post-processing off** and the session moves to the GPU-only path. GPU
post-processing does *not* trigger this (it is not part of `cpuPostActive`), so that one is
safe to leave on.

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
- `blitWork` dominant → you are on the CPU blit path; turn off NPU/CPU post-processing.
- `copy` dominant → full-resolution AHB copies; capture scale again.
- All small but `wallEnd` large → the time is going to pacing sleeps, and fix (2) above is
  the answer.

Also worth capturing, since it measures the stutter directly rather than inferring it:
`getRecentPostIntervalsNs` feeds the frame-graph HUD with real inter-post intervals. A
clean 2× looks like evenly spaced posts; the bunching described above looks like pairs of
posts with a gap after each pair.
