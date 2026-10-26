# Transformer export cancel deadlocks `c2.google.av1.encoder` on Pixel 9 (Tensor G4)

**Component:** `media3-transformer` (`DefaultCodec`, `Transformer.cancel()`)
**Media3:** 1.11.1 (repo HEAD `8c6678b657`)
**Device:** Pixel 9 Pro Fold (`comet`, Tensor G4), Android 16, build `google/comet/comet:16/CP1A.260505.005/15081906:user/release-keys`
**Encoder:** `c2.google.av1.encoder` (hardware AV1, served by `google.hardware.media.c2@2.0-service`)
**Reproducer:** `demos/composition` (debug build), Export sheet → "Stress: 50x" button

---

## 1. Summary

Cancelling a `Transformer` export that uses the hardware AV1 encoder on Tensor G4 can make
`MediaCodec.release()` on that encoder never return. This is a deadlock inside Google's vendor
codec service (as reported by the downstream app that found it, see §8). It is triggered when
the encoder is released while it still has a frame in flight, which Media3's cancel path allows.

Consequences:

1. The stuck encoder is marked "pending removal" by the media resource manager. Every later
   `MediaCodec` creation in the same process waits for it, so decoding, thumbnails and
   subsequent exports all freeze.
2. Each incident leaks a deadlocked encoder and its threads inside the vendor service. After
   enough incidents the service has no free request threads and codec creation hangs system-wide
   until reboot.
3. In the demo, repeated cancels also pile up 4K pipelines until the app dies with an
   `OutOfMemoryError`.

Draining the encoder (signal end of input, discard output until end of stream) before
`release()` removes the problem in the reproducer: 50/50 clean cancels (§6).

---

## 2. Reproduction

### Steps (demo app)

1. Build and install: `./gradlew :demo-composition:installDebug`.
2. Push a 4K clip: `adb push portrait_4k60.mp4 /sdcard/Android/data/androidx.media3.demo.composition/files/stress_4k.mp4`
   (source: `https://storage.googleapis.com/exoplayer-test-media-1/mp4/portrait_4k60.mp4`, 3840x2160 H.264, ~33 MB).
3. Optional interval override (default 100 ms): write the number of ms to
   `.../files/stress_interval_ms.txt`. Use **300** to reproduce reliably (see §2.2).
4. Open the app → Export → leave "Drain encoder on release" **off** → tap
   "Stress: 50x (AV1 export, 100ms, cancel)".

The loop runs 50 times: `Transformer.start()` with `setVideoMimeType(video/av1)` → wait → `Transformer.cancel()`.
A watchdog thread logs main-thread stalls, and after the loop a probe creates an H.264 decoder and an AV1
encoder on a worker thread with a 5 s timeout. Logs use tags `CompPreviewVM` (lines prefixed `STRESS`) and `DefaultCodec`.

### 2.1 Observed (without drain), 300 ms interval

| Run | Outcome |
|---|---|
| 1 | Loop finished (50 iterations, `maxCancel=542ms`), then **H.264 decoder creation hung >5 s and AV1 encoder creation hung >5 s** (`MediaCodec.native_setup`, thread RUNNABLE in native). |
| 2 | Cancels at ~520–540 ms each from iteration 3; app killed by `OutOfMemoryError` at iteration 41. |
| 3 | Same; app killed by OOM at iteration 17–26. |
| 4 | Same; 15 of the first 26 cancels took 520–540 ms, 18 × "Player release timed out", OOM at iteration 26. |

Typical log excerpt:

```
I CompPreviewVM: STRESS #3: start=5ms cancel=529ms (main thread)
E ExoPlayerAssetLoader: Releasing the player timed out.
E ExoPlayerAssetLoader:   Caused by: androidx.media3.exoplayer.ExoTimeoutException: Player release timed out.
D MediaCodec: MediaCodec::reclaim(0xb400006de7474a80) c2.google.av1.encoder     <- same instance, repeatedly
E CompPreviewVM: STRESS PROBE: H264 decoder creation hung; thread state=RUNNABLE
E CompPreviewVM:   at android.media.MediaCodec.native_setup(Native Method)
E CompPreviewVM:   at android.media.MediaCodec.createDecoderByType(MediaCodec.java:2105)
```

Resource manager (`adb shell dumpsys media.resource_manager`) shows for the demo process:

```
markClientForPendingRemoval(pid <app>, clientId ...)
reclaimResourcesFromClientsPendingRemoval(pid <app>)
```

Vendor service after the runs (`adb shell ps -T -p <pid of google.hardware.media.c2@2.0-service>`):
4–5 `Output Looper` and 2 `Codec2.0 Looper` threads remained although no app held a Google codec,
i.e. encoder instances that were never released.

### 2.2 Why 100 ms does not reproduce in this demo

At 4K, Media3's pipeline has not created the encoder by 100 ms (decoder startup dominates), so the
cancel lands before any encoder exists. 50 iterations at 100 ms showed cancels of 4–154 ms and no hang.
The bug needs the cancel to arrive after the encoder is created and has accepted at least one frame.
Pipelines that create the encoder earlier (the downstream app does) hit it at shorter intervals.

---

## 3. Root cause analysis

### 3.1 Vendor deadlock (from the downstream app's native stacks, device-side evidence)

Two threads in `google.hardware.media.c2@2.0-service` wait on each other:

| Thread | Holds | Waits for |
|---|---|---|
| binder thread running `GC2AV1Enc::release` → `GC2Component::release` → `RunningState::release` → `ALooper::stop` → `Thread::requestExitAndWait` | encoder mutex | encoder's Output Looper to exit |
| `Output Looper`: `GC2Component::generateCb` → `GC2AV1Enc::subtask` → `std::mutex::lock` | nothing | encoder mutex |

I could not capture these stacks myself (device not rooted); the demo evidence in §2.1 (pending-removal
entries, repeated `reclaim` of one AV1 encoder, leaked Output Looper threads) is consistent with them.

### 3.2 Why Media3 triggers it

`Transformer.cancel()` → `TransformerInternal.endInternal()` releases asset loaders, the video
graph/frame processor and the encoder. For a surface-input encoder, frames queued to the input
`Surface` are encoded asynchronously by the vendor component, so at release time frames can still
be in flight. `DefaultCodec.release()` (before this change) is simply:

```java
outputBuffer = null;
if (inputSurface != null) inputSurface.release();
mediaCodec.release();
```

with no attempt to quiesce the encoder. `c2.google.av1.encoder` deadlocks if released in that state.

### 3.3 How a stuck release spreads (Android framework behavior)

`MediaCodec::init` → `ResourceManagerServiceProxy::init` → `reclaimResourcesFromClientsPendingRemoval(pid)`
→ `MediaCodec::reclaim(<stuck encoder>)`, which waits for the stuck release. So one stuck encoder blocks
creation of **any** codec (including software ones) in that process. This is why `ExoPlayerAssetLoader`
reports "Player release timed out" (its player cannot finish releasing/creating decoders) and why the
post-loop probe could not create even an H.264 decoder.

---

## 4. Impact

- Export, playback, thumbnailing and any other `MediaCodec` use in the process freeze until the process is killed.
- Killing the app clears the per-process freeze, but the leaked encoders and their threads stay in the vendor
  service. After repeated incidents only a reboot recovers codec creation for all apps.
- Any app that lets users start and cancel exports quickly on Tensor G4 with an AV1 output is exposed.
  `Transformer` selects AV1 when asked via `setVideoMimeType(video/av1)`, and downstream apps that rank AV1
  first on this chip (the reporting app does) will use it by default.

---

## 5. Proposed fix (prototype in this branch)

### 5.1 Change

`libraries/transformer/src/main/java/androidx/media3/transformer/DefaultCodec.java`:

```java
public static void setDrainVideoEncoderOnRelease(boolean enabled)   // opt-in prototype switch

@Override
public void release() {
  outputBuffer = null;
  if (drainVideoEncoderOnRelease && !isDecoder && isVideo && inputSurface != null) {
    drainBeforeRelease();
  }
  if (inputSurface != null) inputSurface.release();
  mediaCodec.release();
}

private void drainBeforeRelease() {
  if (outputStreamEnded) return;                       // already finished normally: nothing in flight
  // 1. return any dequeued-but-unreleased output buffer
  // 2. mediaCodec.signalEndOfInputStream()
  // 3. dequeueOutputBuffer(10 ms) + releaseOutputBuffer(render=false) until BUFFER_FLAG_END_OF_STREAM
  //    or 1 s elapsed; RuntimeExceptions are logged and ignored
}
```

Behavior: after the drain the encoder has emitted end of stream, so it has no frame in flight when
`mediaCodec.release()` is called. Output produced while draining is discarded (the export is being
cancelled). The call is bounded to 1 s. It is skipped for decoders, audio encoders, and encoders that
already reached end of stream.

### 5.2 Suggestions for the Media3 team (for the real implementation)

1. **Where to put it.** Either in `DefaultCodec.release()` (as prototyped, covers every caller) or in the
   cancel path of `VideoSampleExporter`/`EncoderWrapper` so only cancellation pays for it. Releasing after a
   normal end of stream already skips the drain.
2. **Avoid the 100 ms sleep.** `DefaultCodec.signalEndOfInputStream()` sleeps 100 ms when
   `videoOutputStarted` is false (b/301603935). The prototype calls `mediaCodec.signalEndOfInputStream()`
   directly during release, because the lost-frame workaround is irrelevant for discarded output.
3. **Threading.** `Transformer.cancel()` currently blocks the calling thread (main in most apps) while the
   encoder drains. Measured drain time is ~160–180 ms per cancel in the demo (the downstream app measured
   ~83 ms average at its settings), so the cancel cost grows by that amount. Consider releasing the encoder on
   the internal export thread so the caller is not blocked, as long as `cancel()` semantics (output file
   deleted, listener not called) are kept.
4. **Make it unconditional or per-vendor?** The prototype is opt-in and applies to all surface-input video
   encoders. An alternative is applying it by default; the cost (≤1 s, usually ~100–200 ms, only on cancel of
   an unfinished export) is small compared to the freeze risk. A narrower alternative is a deny-list for
   `c2.google.av1.encoder` in `DefaultEncoderFactory`, which avoids the bug but removes AV1 encoding on Tensor G4.
5. **Default selection.** Where Media3 or `CodecDbLite`-style data ranks AV1 first for Tensor G4, document
   or revisit that until the vendor fix ships.
6. **Report upstream.** The root cause is in the vendor HAL; Google should fix `GC2AV1Enc::release`
   vs `GC2AV1Enc::subtask` lock ordering. The Media3 mitigation is a workaround and should be gated/removable.

### 5.3 Limitations / not verified

- Only `c2.google.av1.encoder` on Pixel 9 Pro Fold was tested. Other encoders and Tensor generations were not.
- Drain relies on the encoder reaching end of stream within 1 s. If the encoder is already wedged, the timeout
  bounds the wait but `release()` can still hang inside the vendor service.
- Unit tests for the new path were not written or run (only the demo was exercised).
- The `OutOfMemoryError` in the unfixed runs is a demo artifact of repeatedly leaking 4K pipelines; the
  downstream app saw a freeze instead.

---

## 6. Results with the fix (drain on)

Same device and input, 300 ms interval, 50 iterations, on a vendor service that already had leaked encoders
from the earlier runs (device not yet rebooted):

| Metric | Without drain | With drain |
|---|---|---|
| Iterations completed | 17–41 before app death / hang | **50 / 50** |
| "Player release timed out" | 18 (single run) | **0** |
| Cancel time on main thread | 520–542 ms from iteration ~3 on | 71–269 ms, no stalls |
| AV1 encoders created | — | 50 |
| Encoders drained to end of stream | — | **50 / 50** (`ended=true`, ~156–181 ms each) |
| Post-loop H.264 decoder / AV1 encoder creation | hung >5 s (run 1) | OK (49 ms / 31 ms) |
| Vendor service thread count | grew (24 → 26) | unchanged (21) |
| App survived | no (OOM) | yes |

Log lines from `DefaultCodec`:

```
I DefaultCodec: Drained c2.google.av1.encoder before release: ended=true in 162ms
```

---

## 6.1 Cancel off the main thread (no Media3 change needed)

`Transformer.cancel()` is documented as callable only from the Transformer's application looper
(`Transformer.Builder.setLooper`, default: the creating thread's looper, else main). Internally
`TransformerInternal.cancel()` posts `MSG_END` to the `Transformer:Internal` thread and then **blocks the
calling thread** on `canceledConditionVariable` until `endInternal()` has released the sample exporters
(video encoder, including the drain), asset loaders and muxer. So with the default looper the main thread is
blocked for the whole encoder release (and indefinitely if the encoder release deadlocks).

Fix at app level: build the Transformer with its own looper and run `start()`/`cancel()` there.

- `Transformer.Builder.setLooper(exportThread.looper)` with a dedicated `HandlerThread("ExportTransformer")`.
- `start()` and `cancel()` are posted to that looper. All calls come from one producer thread to one
  consumer thread, so they run in the order issued: a new export can never start before the previous cancel
  has finished, which is the serialization the main thread gave implicitly. The output-file delete runs after
  `cancel()` on the same thread.
- Listener callbacks now arrive on the export thread (the demo only updates a thread-safe `StateFlow`).

Demo switch: "Run Transformer (cancel) off main thread". Result (drain on, 300 ms, 50 iterations, same device):

| Metric | Cancel on main thread (drain on) | Cancel on `ExportTransformer` thread (drain on) |
|---|---|---|
| Max cancel call time on main thread | 269 ms (typ. 70-190 ms) | **1 ms** (just a post) |
| Max main-thread stall (20 ms ping watchdog) | not measured | **54 ms** (threshold 150 ms never exceeded) |
| `Transformer.cancel()` duration on export thread | n/a | p50 84 ms, max 188 ms |
| Errors / wrong-thread exceptions / release timeouts | 0 / 0 / 0 | 0 / 0 / 0 |
| Iterations | 50/50 | 50/50 |

Without the drain, an off-main-thread cancel keeps the main thread responsive but the export thread (and every
later codec creation in the process) still blocks on the deadlocked release, so the drain is still required.

Suggestions for Media3: document that `cancel()` blocks until release completes; consider an asynchronous
cancel (post and return, with the listener or a future signalling completion) so apps need not own a looper.

---

## 6.2 Alternative release strategies compared (prototype, `DefaultCodec`)

Strategies added to `DefaultCodec` (all only for surface-input video encoders), selected in the demo via
`files/stress_mode.txt` (`none|drain|flush|serialize`):

| Mode | Behavior before `mediaCodec.release()` |
|---|---|
| `none` | nothing (stock behavior, control) |
| `drain` | `signalEndOfInputStream()`, discard output until end of stream (cap 1 s) |
| `flush` | `mediaCodec.flush()` then `mediaCodec.stop()` |
| `serialize` | poll and discard output until the encoder has been idle for 100 ms (cap 1 s), and take a process-wide lock around `MediaCodec.createByCodecName` and `release()` of every `DefaultCodec` |

50 x (AV1 export -> 300 ms -> cancel), cancel on main thread, 4K input, same device. **Not a clean experiment:**
the phone had **not** been rebooted (leaked encoders from earlier runs in the vendor service), and each mode ran once.

Round 1 (order flush, serialize, drain, then none), all against the same already-degraded vendor service:

| Mode | Iterations | Release timeouts | Cancel on main (p50 / max) | Post-loop codec probe | Result |
|---|---|---|---|---|---|
| `flush` | 50 | 0 | 45 / 65 ms | OK | no hang |
| `serialize` | 50 | 0 | 308 / 430 ms | OK | no hang |
| `drain` | 50 | 0 | 205 / 257 ms | OK | no hang (all encoders `ended=true`, ~110-160 ms) |
| `none` (control) | 32 (app died) | 20 | 529 / 538 ms | n/a | hang, OOM (7 events) |

Round 2 (flush, serialize, drain again, after the `none` control had leaked more encoders): `flush` and `drain`
finished but with every cancel at ~524-529 ms and the AV1 encoder probe **hung**; `serialize` died at iteration 15
with 14 release timeouts. Round 2 is invalid as a comparison because the vendor service was already
exhausted by the control run, which also shows the leak persists across processes.

### 6.2.1 Clean rerun (freshly rebooted phone, 2 runs per mode, `none` control last)

Same test (50 x AV1 export -> 300 ms -> cancel, cancel on main, 4K). Vendor service started with 1 thread / 0 Output Loopers.
All runs completed the whole loop and the post-loop H.264 decoder and AV1 encoder probes succeeded unless noted.

| Mode | Runs | Release timeouts | Cancel on main p50 / max | Max main-thread stall | Vendor service afterwards |
|---|---|---|---|---|---|
| `flush` (flush+stop) | 2 | 0 | 41-44 / 57-68 ms | 92-99 ms | 12 threads, 0 loopers |
| `serialize` | 2 | 0 | 320-326 / 348-351 ms | 382-386 ms | 13 threads, 0 loopers |
| `drain` | 2 | 0 | 211-214 / 234-245 ms | 263-275 ms | 13 threads, 0 loopers |
| `none` (control) | 1 | 18 | 525 / 540 ms | n/a (app killed: `OutOfMemoryError` x16, died at ~iteration 26) | 15 threads, **1 leaked Output Looper** |

(Counts of `STRESS #` lines are lower than 50 in some logs because the logcat ring buffer wrapped; every run printed its final result line.)

Conclusion: on a clean device, `flush`+`stop`, `drain` and `serialize` all avoided the deadlock in 2/2 runs with no leaked
encoders, while the stock behavior hung and leaked an encoder in the first run. `flush`+`stop` is by far the cheapest (~40 ms
cancel), `drain` guarantees end of stream (~210 ms), `serialize` is slowest (~320 ms). Remaining caveats: 2 runs per mode is still
small, and `flush`+`stop` has no end-of-stream guarantee, so its safety margin should be validated on more devices/encoders and
with native stacks before recommending it over `drain`.

Earlier (dirty-device) observations, kept for the record:

What this does and does not show:
- In round 1 every strategy avoided the hang once, while the control hung. That is consistent with the
  root cause (an in-flight frame at release) but **n=1 per mode on a dirty vendor service is not enough to rank them**.
- `flush`+`stop` was fastest (~10 ms) and surprisingly effective; it still needs repeat runs, since `stop()` on this
  component could take the same vendor lock under other timing.
- `serialize` costs the most on cancel (idle wait, ~200-250 ms) and, apart from the idle wait, adds nothing a single
  export needs (Media3 already releases the exporter before the asset loaders and cancel is synchronous).
- `drain` is the only one that guarantees the encoder has emitted end of stream, so it is the most principled.
- **To do:** after a reboot, run each mode in its own freshly rebooted state (reboot between modes, `none` last), several
  repetitions, ideally with native stacks (ANR/bugreport) to confirm no stuck `GC2AV1Enc` threads afterwards.

---

## 6.3 Is it specific to AV1? Control with HEVC

Same stress (4K input, cancel on main, no drain, `mode=none`), forcing the output MIME type with `files/stress_mime.txt`.
Device was not freshly rebooted (the vendor AV1 service already had 1 leaked encoder from earlier), which only makes a hang more likely.

| Output MIME | Encoder actually created | Interval | Iterations | Release timeouts | Cancel on main p50 / max | Stuck / pending removal |
|---|---|---|---|---|---|---|
| `video/hevc` | `c2.exynos.hevc.encoder` (Samsung/Exynos HAL, `samsung.hardware.media.c2@1.2-service`) | 200 ms | 50 | 0 | 39 / 75 ms | none |
| `video/hevc` | same | 300 ms | 50 | 0 | 34 / 62 ms | none |
| `video/hevc` | same | 450 ms | 50 | 0 | 37 / 59 ms | none |
| `video/hevc` | same | 700 ms | 50 | 0 | 39 / 51 ms | none |
| `video/av01` (positive control, run last) | `c2.google.av1.encoder` (Google HAL) | 300 ms | died at ~31 | 19 | 529 / 548 ms | yes, OOM kill |

Post-loop codec probes passed after every HEVC run, and the Exynos HAL showed 0 leaked looper threads afterwards.
(Per-run `STRESS #` counts in the logs are lower than 50 where the logcat ring buffer wrapped; every HEVC run printed its final result.)

Conclusion: with the same rapid start/cancel pattern, the HEVC hardware encoder did **not** deadlock at any of 4 intervals
(200 runs total), while the AV1 encoder deadlocked at 300 ms every time it was tried. So the bug is specific to
`c2.google.av1.encoder` (Google HAL), not to Media3's cancel path in general. HEVC therefore also did not need the drain, so
the drain could not be evaluated on it. This is consistent with Premier's finding of 150+ clean HEVC cycles.

Not valid: an `video/avc` run. The source is H.264, so Transformer transmuxed and never created an encoder
(only the probe's AV1 encoder was allocated); that run says nothing about H.264 encoders.

Caveat: this shows no hang for HEVC at these timings on this device, not that no other encoder can have the problem.

---

## 7. How to check device / run state

```bash
# Encoder sessions since boot (non-zero is expected when forcing AV1)
adb shell dumpsys media.metrics | grep -o 'mediacodec.codec=c2.google.av1.encoder' | wc -l

# Pending removal / reclaim events for the app's pid
adb shell dumpsys media.resource_manager | grep -E "markClientForPendingRemoval|reclaimResourcesFromClientsPendingRemoval"

# Vendor service threads: leaked encoders show up as extra "Output Looper" threads
adb shell ps -A | grep google.hardware.media.c2
adb shell ps -T -p <pid of google.hardware.media.c2@2.0-service>

# Live logs while running the stress test
adb logcat -v threadtime | grep -E "STRESS|DefaultCodec|Player release timed out|MediaCodec::reclaim"
```

Native stacks of the vendor deadlock require an ANR/bugreport on a device (the downstream app captured
them by forcing an ANR and reading `/data/anr` from `adb bugreport`).

---

## 8. Provenance of the root-cause stacks

The vendor-side native stacks in §3.1 come from the downstream app's investigation (Adobe Premiere for
Android, same device and Android build), obtained with JDWP thread dumps, ANR native stacks from a bugreport,
`dumpsys media.resource_manager` and `dumpsys media.metrics`. Their mitigation drains export encoders before
release (capped at 1 s) and also avoids this encoder in a separate workaround. This report's demo
reproduces the same symptoms with stock Media3 and shows the same drain fixes them.

---

## 9. Files changed in this repository (uncommitted)

| File | Change |
|---|---|
| `libraries/transformer/.../DefaultCodec.java` | Drain-before-release prototype and opt-in switch |
| `demos/composition/.../CompositionPreviewViewModel.kt` | Stress loop, watchdog, codec probe, drain toggle state |
| `demos/composition/.../CompositionPreviewActivity.kt` | Wires stress status and drain toggle |
| `demos/composition/.../ui/ExportOptions.kt` | "Stress" button, status text, drain and off-main-thread switches |
| `demos/composition/.../CompositionPreviewViewModel.kt` | also: dedicated `ExportTransformer` looper for start/cancel |
