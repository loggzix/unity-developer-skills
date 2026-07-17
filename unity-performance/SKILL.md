---
name: unity-performance
description: Measure-first performance optimization for Unity. Use when frame rate drops, GC spikes, load times are slow, memory grows, build size balloons, or the user asks to "optimize" anything in a Unity project. Forbids optimizing without profiler data (sole exception: one UNVERIFIED lowest-risk fix when the user refuses to measure). Triggers - optimize, lag, stutter, FPS drop, GC spike, build size, tối ưu, giật, khựng, tụt fps, nặng máy, ngốn RAM, load lâu, giảm dung lượng build, memory leak, rò rỉ bộ nhớ, crash OOM, hitch, jank, frame drop, draw calls, batching, slow loading, app size, startup slow, mở app lâu, nóng máy, thermal throttling, texture memory.
---

# Unity Performance Optimization

Measure before optimizing. Performance work without measurement is guessing — guessing adds complexity without improving what matters. Never accept "this looks slow" as a bottleneck claim.

> 🔴 **STOP · GATE — no numbers, no fix.**
> - RULE: no profiler capture / build report / frame timing for the symptom → the ONLY next step is the capture (exact tool, exact scenario).
> - LOCUS UNSTATED: user chưa nói Editor hay device/build → ask exactly one question "Editor hay máy thật/build?" BEFORE the capture instruction — never default to Editor. EXEMPT: build-size và app-startup symptoms — locus cố định sẵn (Build Report / device boot capture); skip câu hỏi, đi thẳng capture instruction.
> - WHO RUNS: symptom reproduces in-Editor AND MCP manage_profiler is connected → capture it yourself now (MEASURE step 1); instruct-and-wait applies when the capture needs the user (device, build, manual play scenario).
> - DEVICE-ONLY SYMPTOM: the instructed capture IS the device capture (Development Build + Autoconnect Profiler); offer an Editor capture ONLY IF the user states the device is unreachable right now — label it COARSE-INTERIM per the MEASURE rule, never self-run it.
> - NAMED SUSPECT: a user-named prefab/shader/script/asset is a hypothesis to rank for measurement, not a work order — do not edit code first.
> - SOLE EXCEPTION: the "User refuses to measure" row in If X → do Y — one lowest-risk catalog fix, labeled UNVERIFIED.
> - WHILE WAITING: no edits, no scaffolding, no drafts.
> - GATE OUTPUT: (a) exact capture instruction — tool | steps | scenario | which profiler view to read; (b) 2-3 ranked hypotheses, each with the marker/counter that would confirm it.

## Workflow

1. **MEASURE** — baseline with real data:
   - Unity Profiler (CPU/GPU/Memory/Rendering) — baseline capture:
     - 1a. MCP path: `manage_profiler(action="profiler_start")` → enter Play Mode (`manage_editor(action="play")`) and perform (or ask the user to perform) the exact reported action once → `manage_profiler(action="get_frame_timing")` + `manage_profiler(action="get_counters", category="Render"|"Scripts"|"Memory")` → `manage_profiler(action="profiler_stop")` → `manage_editor(action="stop")` → `read_console`.
     - 1b. No MCP → Editor (Window > Analysis > Profiler), manual.
     - 1c. DEVICE RULE (CANONICAL — referenced everywhere as "the MEASURE coarse-interim rule"): symptom reported on device or in builds → the baseline MUST be a device/build capture; an Editor capture for a device symptom = COARSE-INTERIM only — usable to rank hypotheses, never as the baseline or the VERIFY capture; Editor numbers lie (especially GC and rendering).
     - Record the baseline as `metric | value | scenario | device`.
   - Frame Debugger for draw-call/batching questions.
   - Memory Profiler package for memory growth/leaks.
   - Build-size symptom → the baseline is the Build Report ranked asset/category list (Editor.log after a build — Windows `%LOCALAPPDATA%\Unity\Editor\Editor.log`, macOS `~/Library/Logs/Unity/Editor.log`, search "Build Report"; or Build Report Inspector), recorded as `category | MB | top asset` — the Profiler is not the tool for size.
   - Record specific numbers: ms per frame by category, GC alloc per frame, draw calls, setpass calls.
2. **IDENTIFY** — find the actual bottleneck: CPU main thread vs render thread vs GPU vs GC. Fixing the wrong one changes nothing. Classify by the marker on top:

   | Marker on top | Verdict | Fix domain |
   | --- | --- | --- |
   | `Gfx.WaitForPresent`/`Gfx.PresentFrame` (main thread) | GPU-bound | rendering, not scripts |
   | `Gfx.WaitForCommands` (render thread) | render thread starving | main-thread CPU |
   | `WaitForTargetFPS` | vsync/FPS cap | not a bottleneck |
   | `GC.Alloc`/`GC.Collect` | GC | allocations |
   | `Semaphore.WaitForSignal` (render thread) | main-thread CPU is the culprit | scripts |

   **Output:** a table `rank | marker | self ms | % frame | catalog fix` (rank by self-time ms, descending; top 3 minimum) — this is what the CHECKPOINT presents. e.g. `1 | Physics.Simulate | 7.2 ms | 43% | layer-mask casts, lower FixedUpdate rate` / `2 | ParticleSystem.Update | 3.1 ms | 19% | cap max particles` / `3 | Animators.Update | 2.7 ms | 12% | culling mode BasedOnRenderers`.

   > 🔴 **CHECKPOINT** — present the measured numbers + ranked bottleneck to the user, get confirmation BEFORE editing code. Wait for an explicit reply — silence or a follow-up question is not consent, and the ORIGINAL optimize request (in any wording or language) is NOT confirmation of the ranking, even in the message that delivered the numbers; confirmation must come AFTER the ranked table is presented. Valid confirmation = the user names a rank, names the marker, or replies an explicit yes/ok to the proposed #1 — anything else → re-ask in one line containing only the #1 marker and its ms — do not re-print the table or add new analysis while waiting. User picks a lower-ranked bottleneck → follow their pick, note its rank.

3. **FIX** —
   - 3a. Input: the single bottleneck confirmed at the CHECKPOINT — default #1; the user explicitly picked a lower-ranked one there → that pick IS the input (note its rank). Never an unconfirmed bottleneck, never two at once. User explicitly authorizes batching multiple fixes → still refuse: one change per VERIFY cycle (→ If-X batch row).
   - 3b. Match it to a Common-bottlenecks entry and apply that fix. Fix is a bulk texture/audio import override → 🔴 CHECKPOINT: asset count + exact setting changes, confirm first (Memory-section gate).
   - 3c. No catalog entry matches → Deep Profile that marker's children for one short window (≤5 s / ≤300 frames around the exact action, then Deep Profile OFF); project-code marker (e.g. MyGame.X.Update) → remove the per-frame allocations/redundant work its children show; engine-internal marker → `unity_docs(action="lookup", query="<marker>")` before touching anything. Label the fix CATALOG-MISS.

   > 🔴 CHECKPOINT (CATALOG-MISS) — present the Deep-Profile finding + proposed change, same confirmation rules as step 2; no edit until explicit pick — still one change per VERIFY cycle.

   - 3d. Output: one focused change + one line `change made | metric expected to move | expected value` before VERIFY.
4. **VERIFY** — measure again, same scenario, compare numbers. Symptom was reported on device/build → the after-capture MUST be on the same device/build tier; an Editor after-capture does not count. Output: `metric | before | after | delta % | same-scenario? y/n` — no table = not verified. No before/after numbers = not done. Numbers worse or unchanged → revert the change, return to IDENTIFY with the next hypothesis. Unchanged = |delta| within run-to-run noise (compare medians of 3 runs per the Numbers-noisy row); an improvement smaller than the spread between runs counts as unchanged → revert.
5. **GUARD** — 🔴 task is not complete without (a) or (b): a VERIFY table without a GUARD artifact is an unfinished task.
   - (a) package present (check `Packages/manifest.json` contains `com.unity.test-framework.performance`, or `manage_packages(action="get_package_info", package="com.unity.test-framework.performance")`) → 🔴 creating PerfGuard.cs + its asmdef writes new files into the user's project — state the two file paths in one line and get an OK first → write PerfGuard.cs + asmdef. Place in `Assets/Tests/Performance/PerfGuard.cs` inside a test asmdef referencing Unity.PerformanceTesting — asmdef: `{"name":"PerfGuard.Tests","references":["Unity.PerformanceTesting","UnityEngine.TestRunner","UnityEditor.TestRunner"],"defineConstraints":["UNITY_INCLUDE_TESTS"]}` (Unity 2019.3+; Unity cũ hơn → optionalUnityReferences:["TestAssemblies"]) — with `using System.Collections; using System.Linq; using NUnit.Framework; using UnityEngine.TestTools; using Unity.PerformanceTesting;` — `const double BUDGET_MS = 16.6; // from Frame budget targets: 16.6 = 60 FPS, 33.3 = 30 FPS mobile — use the project's declared target, never copy blind` then `[UnityTest, Performance] public IEnumerator Scenario_Frame() { yield return Measure.Frames().WarmupCount(5).MeasurementCount(20).Run(); var median = PerformanceTest.Active.SampleGroups.First(g => g.Name == "Time").Median; Assert.Less(median, BUDGET_MS); /* "Time" is ms */ }` (a [Test] void form never iterates the IEnumerator Run() returns — zero frames recorded).
   - (b) package absent or user declines → record one line `scenario | metric | budget | date measured` appended to the project's perf-notes file if one exists (glob `Docs/PERF*.md`, `**/PERF_BUDGET*`), else create `Docs/PERF_BUDGET.md` and append there so the exact scenario can be re-checked — 🔴 creating this file writes into the user's project: state the path in one line and get an OK first (same rule as (a)); appending to an EXISTING perf-notes file needs no gate; user declines both → report the budget line inline in chat and mark GUARD=declined. Package absent → do NOT install com.unity.test-framework.performance unprompted (long resolve + recompile) — use (b).

## Frame budget targets
- 60 FPS → 16.6 ms/frame; 30 FPS (mobile) → 33.3 ms.
- GC allocation in steady-state gameplay: target 0 B/frame (spikes cause hitches).
- Set a per-project budget (draw calls, setpass, texture memory) and check against it, not against "feels fine".

## Common Unity bottlenecks (check profiler FIRST, then look for these)

Thermal symptom (nóng máy, throttling): sustained CPU+GPU load, not a separate bottleneck class — capture on device after 10+ min of play (throttled state), then classify per IDENTIFY; cooling-dependent numbers → the Numbers-noisy If-X row.

**CPU / scripts:**
- GetComponent, Find, FindObjectOfType, Camera.main in per-frame code → cache.
- LINQ, string concat/interpolation, boxing, closures in hot paths → GC spikes.
- Instantiate/Destroy churn → object pooling (`UnityEngine.Pool.ObjectPool<T>`); one-shot spike on open (popup/scene) → prewarm the pool at load, or spread instantiation across frames (≤2 ms per frame via coroutine/async).
- Physics: per-frame casts without layer mask; SendMessage; excessive FixedUpdate rate (>50Hz / Fixed Timestep <0.02 without physics need).
- Debug.Log left in hot paths (allocates + serializes even in builds unless stripped).

**Rendering:**
- Draw calls / setpass: batching broken by material instances (`renderer.material` leak → use sharedMaterial or MaterialPropertyBlock).
- Overdraw (transparent UI/particles on mobile).
- Missing SRP batcher compatibility / GPU instancing on repeated meshes.
- GPU-bound verdict (`Gfx.WaitForPresent` on top of main thread) → in order: reduce URP Render Scale 0.7-0.85 on mobile; cut post-processing (Bloom/DoF first); reduce overdraw (Scene view Overdraw mode; flatten stacked transparents, cap particle screen size — Renderer module → Max Particle Size 0.5); simplify the heaviest shader (Frame Debugger → most expensive draw): URP Lit → Simple Lit/Baked Lit, or strip unused shader_feature variants; MSAA 2x/off.

**Loading / startup:**
- `Addressables .WaitForCompletion` or `Resources.Load` during gameplay → `LoadAssetAsync` + preload on the prior screen/transition.
- `Shader.CreateGPUProgram`/`Shader.Parse` spikes on first use → `ShaderVariantCollection` warmup at boot.
- JSON/save parse on main thread during load → move behind the load screen; markers: `Loading.LoadFileAsync`, `Shader.CreateGPUProgram`, `GC.Collect` right after load.

**UI (uGUI):**
- Canvas rebuild: any dirtied element rebuilds the whole canvas → split canvases by update frequency; don't set text/color every frame without change check.
- Layout groups + Content Size Fitter nesting in dynamic lists.

**Runtime memory (leaks/OOM) & build size** — leak/OOM symptoms use [RUNTIME] items; the Build-size If-X row uses [SIZE] items; [BOTH] applies to either:

> 🔴 CHECKPOINT — texture/audio import overrides touch many assets and trigger long reimports: list the affected asset count + exact setting changes and get user confirmation BEFORE applying in bulk.
- [RUNTIME] Textures unnecessarily readable (Read/Write flag doubles memory); [SIZE] uncompressed textures.
- [BOTH] Android textures not ASTC (default ETC/RGBA32) → per-platform override ASTC 6x6 (UI/albedo) or 8x8; cap Max Size — 2048 for UI atlases, 1024 for world albedo on 2-3GB-RAM devices.
- [BOTH] Audio: long clips on Decompress On Load / PCM → Load Type=Streaming + Vorbis; short SFX → Compressed In Memory.
- [SIZE] Duplicated sprites outside an atlas (each copy counts separately) → SpriteAtlas; confirm via Build Report ranked assets.
- [RUNTIME] Addressables/AssetBundles not released; scenes loaded additively and never unloaded.
- [SIZE] Resources folder ballooning build size.

## If X → do Y

| Trigger | Fix | Still failing → fallback |
| --- | --- | --- |
| Symptom only on device, fine in Editor | Profile ON the device (Development Build + Autoconnect Profiler, or `adb` + Profiler attach) — the MEASURE device rule applies | No target device available → Development Build + Profiler on any device of the same tier (same GPU family/RAM class), state the tier mismatch in the baseline; an Editor capture stays COARSE-INTERIM (→ MEASURE rule) |
| GPU-bound verdict but no GPU timing data on the device (Unity GPU profiler unsupported on most Android) | Use main-thread `Gfx.WaitForPresent` wait ms as the GPU-cost proxy; rank shaders/overdraw via Frame Debugger in Editor at matched resolution (Render Scale, MSAA) — verdict stays device-sourced | Vendor tool: Android GPU Inspector / Xcode GPU capture; state proxy-only confidence |
| Can't attach profiler to device | Development Build with Profiler stats overlay | `Debug.Log(Time.deltaTime)` bracket around the suspect scenario — coarse numbers beat none; say precision is limited |
| MCP `manage_profiler` unavailable | Editor Profiler manually (Window > Analysis > Profiler) | Development Build + stats overlay |
| MCP capture gives counters but no per-marker ms | `get_counters` returns category totals only — for marker self-time, read the Editor Profiler Hierarchy sorted by Self ms (Window > Analysis > Profiler), or `profiler_start(log_file="perf.raw")` and open the .raw in the Profiler window | Editor unreachable → `Profiler.BeginSample`/`EndSample` around the 2-3 ranked suspects, log ms |
| Editor.log has no Build Report section | Rebuild, then read the log right after the build | Install the Build Report Inspector package |
| Hitch not reproducible on demand | Capture with Profiler recording running while playing until it fires; use Deep Profile only for short windows (≤5 s / ≤300 frames, then OFF — it distorts timing) | Still never fires → `Profiler.BeginSample`/`EndSample` around the 2-3 ranked suspects + a long soak capture on device |
| Numbers noisy between runs | Fix the scenario (same scene, same save, 3 runs), compare medians; thermal throttling → let device cool between runs | Still noisy → 5 runs, lock the device (fixed brightness, airplane mode, cool 5 min between runs), report P50 and P95 separately; P95-only regression → treat as hitch (see Hitch row) |
| User provides numbers already | Accept when they include per-marker ms + a clear scenario AND were captured where the symptom occurs (device symptom → device/build capture; Editor numbers for a device symptom → coarse ranking only, instruct a device re-capture per the device row) → go to IDENTIFY, do NOT demand re-measuring | Totals only ("frame 90ms") without breakdown → ask for one Deep Profile capture of that exact action |
| User provides a profiler screenshot / partial capture only | Read what is visible; top marker + ms legible → treat as numbers-provided, go to IDENTIFY with a stated confidence caveat | Marker names illegible or totals-only → ask for the Hierarchy view sorted by Self ms, or one .raw export (profiler_start(log_file=...)) |
| User refuses to measure, demands an immediate fix | Offer the cheapest measurement once: one Deep Profile capture of that exact action (~2 min), state the cost | Still refused → apply ONLY the lowest-risk catalog fix matching the symptom (lowest-risk = settings/asset-only change with no code edit — one import override, one Load Type; none matches → a single cache-the-lookup one-liner; NEVER a pooling/architecture change under UNVERIFIED), label it UNVERIFIED, require after-numbers before any second fix |
| User confirms the ranking but demands several fixes in one batch | Refuse the batch even with a confirmed ranking — one change per VERIFY cycle is a measurement-attribution rule, not a preference: apply #1 only, VERIFY, then #2; say so in one line | User insists again → hold the rule; offer the compressed loop instead: fix #1 now → one quick re-capture → #2 in the same session |
| Capture is flat, no clear hotspot | Sort by peak frame, not average (hitches hide in averages); widen the capture; Deep Profile a short window around the exact action | Timeline view: hunt single spikes (GC.Collect, Loading.*, Shader.Parse); `Profiler.BeginSample` around 2-3 suspects |
| App startup slow (mở app lâu) | Baseline = time-to-first-interactive on device: Development Build + Autoconnect Profiler from boot, or `adb logcat -s Unity ActivityManager` Displayed time; rank: Shader.CreateGPUProgram (→ ShaderVariantCollection warmup), Addressables/RemoteConfig init on main thread, sync JSON/save parse, first-scene Instantiate burst | Profiler can't attach at boot → `Debug.Log(Time.realtimeSinceStartup)` brackets at splash/first-frame/first-input, compare stages |
| Play Mode won't start / compile errors block the self-run capture | read_console, fix compile FIRST (a capture mid-error is meaningless), then re-run MEASURE step 1 | Errors out of scope right now → instruct the user to capture manually in the Editor Profiler; treat as instruct-and-wait |
| Build size question | Build Report in Editor.log (or Build Report Inspector) FIRST — ranked size by category/asset. Usual suspects AFTER the report: the [SIZE] items in the **Runtime memory (leaks/OOM) & build size** list above, plus the StreamingAssets folder (copied verbatim into the build, never compressed/stripped). Fix one group → rebuild → compare | Can't rebuild now → read the LAST build's report in Editor.log; no dominant asset in the report → compare IL2CPP/code size, stripping level, duplicated bundles (Addressables → Analyze: Check Duplicate Bundle Dependencies) |
| GC spike but no per-frame alloc visible | Check load-time/one-shot allocs (Addressables, JSON parse, LINQ in Initialize) and incremental GC settings | Still no source → re-capture with allocation callstacks (`manage_profiler(action="profiler_start", enable_callstacks=true)` or Profiler Call Stacks: GC.Alloc), read GC.Alloc callstacks in Timeline; enable Incremental GC only AFTER the alloc source is named, never as the fix itself |
| Memory grows over session (ngốn RAM, leak suspicion, OOM crash) | Snapshot diff: `manage_profiler(action="memory_take_snapshot")` at t0 → repeat the scenario → snapshot t1 → `memory_compare_snapshots` (or Memory Profiler package Diff). Rank growth: unreleased Addressables handles, event subscriptions, static caches, Read/Write textures, additive scenes never unloaded | Memory Profiler package missing → Profiler window Memory module (Simple view) totals at t0/t1 as a coarse diff, offer to install com.unity.memoryprofiler; diff unclear → 3 snapshots over time, compare the growth curve; still blind → `get_object_memory` on the top 2-3 suspect assets |
| The fix helped but budget still missed | Re-profile — the NEXT bottleneck is now on top; repeat MEASURE→FIX, don't keep polishing the old one | After 3 MEASURE→FIX cycles still over budget → stop, present the remaining gap (current vs budget) + the cost of the next candidate; re-negotiate budget or scope before cycle 4 |
| User goes silent at a CHECKPOINT | Do nothing — no edits, no scaffolding; the session ends with the ranked table as the deliverable | Next user message resumes → restate the table in one line, re-ask for the pick |

## Anti-patterns
- Optimizing code the profiler doesn't show as hot (premature).
- Micro-optimizing (for vs foreach) while a 5 ms GetComponent-in-Update sits there.
- Caching everything "just in case" — adds state bugs without measured need.
- Async/threading added for work that isn't the bottleneck.
- Deep Profile left on for the whole session, or concluding from a single run (→ If X table).
- Applying two or more fixes before re-measuring (→ If-X batch row).
- Concluding device performance from an Editor capture (→ If X table, device row).
- Enabling Incremental GC (or raising heap size) as the fix instead of naming the allocation source first (→ If X table, GC-spike row).

## Verification checklist
- [ ] Before/after numbers exist with specific values (ms, B/frame, draw calls)
- [ ] Bottleneck identified in profiler, not assumed
- [ ] Same test scenario both measurements
- [ ] No new GC allocation introduced
- [ ] Tests still pass, gameplay unchanged
- [ ] GUARD recorded: `[UnityTest, Performance]` coroutine test (per GUARD step 5) committed OR a `scenario | metric | budget | date` line in the project's perf notes — VERIFY without GUARD = not done
