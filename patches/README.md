# Poco X8 — diagnosis and fixes

Analysis of why base FPS drops even at `multiplier=2` / `flow_scale=0.25`, and why
Shizuku capture never starts.

Analysed against:

| Repo | Commit |
|---|---|
| `FrankBarretta/LSFG-Android-Application` | `b8475419` (the commit this fork pins) |
| `FrankBarretta/lsfg-vk-android` | `release` @ `3e89e543` |

`0001-poco-x8-shizuku-and-overhead-fixes.patch` applies to the application repo:

```sh
cd LSFG-Android-Application
git apply ../patches/0001-poco-x8-shizuku-and-overhead-fixes.patch
```

**None of this has been compiled or run on a device** — there is no Android SDK/NDK in
the environment it was written in. Treat the patch as a reviewed proposal, not a tested
one. The reasoning behind each hunk is below so you can judge it yourself.

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

**Fix (in the patch):** probe for `createSyncCaptureListener()` +
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
goes**. Three costs are paid per frame regardless of multiplier and flow scale.

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

- **Lock refused** → `captureContentHash` returns 0 → the dedup branch is skipped → *every*
  capture runs the full `import → copy → framegen → waitIdle → present` pipeline. Since
  the VirtualDisplay produces frames at the panel rate, that is up to 4× the intended work
  at 30 fps content. The HUD's "real fps" also freezes, because `uniqueCaptures` stops
  incrementing — worth checking on your device, it's a free diagnostic.
- **Lock honoured** → the driver resolves the whole compressed surface (AFBC on Mali, UBWC
  on Adreno) to linear and waits on the producing GPU work first — a full-resolution
  detile plus a pipeline sync, on the capture thread, once per frame.

Either way it back-pressures SurfaceFlinger's VirtualDisplay, which is exactly the
"base FPS drops even when I'm not in a game" symptom.

**Fix (in the patch):** declare `USAGE_CPU_READ_OFTEN` on the `ImageReader` so the lock is
legal, check `desc.usage` before locking, and self-disable hashing after 8 consecutive
failures instead of retrying forever. The privileged paths can't declare usage — the
buffer comes from SurfaceFlinger — so there the usage check turns dedup off cleanly rather
than paying a failed ioctl per frame.

> **Worth A/B testing:** `CPU_READ_OFTEN` makes gralloc pick a CPU-cached, usually
> *uncompressed* layout, which costs GPU bandwidth on every framegen read of that buffer.
> On a bandwidth-bound mid-range part, deleting the hash entirely (accepting no
> duplicate-skip, and a "real fps" readout that tracks the panel) may beat keeping it.
> I can't tell which wins without your device. Try the patch first; if it's still heavy,
> stub `captureContentHash` to `return 0` and compare.

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

**Fix (in the patch):** make `requestedRefreshRateHz` honour the override, capped at what
the panel supports. AUTO still means maximum, so nobody's defaults change.

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

**Not in the patch** — it needs a pref, drawer UI, and plumbing through
`setLsfgMode`/`initContext`, and it changes the overlay/capture alignment invariant that
`OverlayManager` maintains. But it's the single biggest lever available: `createVirtualDisplay`
already takes explicit `width, height`, and SurfaceFlinger scales during composition, so a
0.7× capture cuts framegen's per-frame work to roughly half at a cost most people won't
see through a 2× interpolation. Say the word and I'll build it.

---

## What to try right now, before rebuilding anything

These need no code changes and will tell us which of the above dominates on your device:

1. **Set Refresh override to 60 Hz** in the drawer. Today that only changes pacing, not the
   panel pin (2b) — so if it already helps noticeably, pacing is a factor; if it does
   nothing, that's consistent with the pin being the problem and the patch should help.
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
