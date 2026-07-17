---
name: unity-performance
description: Measure-first performance optimization for Unity. Use when frame rate drops, GC spikes, load times are slow, memory grows, build size balloons, or the user asks to "optimize" anything in a Unity project. Forbids optimizing without profiler data.
---

# Unity Performance Optimization

Measure before optimizing. Performance work without measurement is guessing — guessing adds complexity without improving what matters. Never accept "this looks slow" as a bottleneck claim.

## Workflow

1. **MEASURE** — baseline with real data:
   - Unity Profiler (CPU/GPU/Memory/Rendering) — via MCP `manage_profiler` or Editor. Profile on TARGET device/build when possible; Editor numbers lie (especially GC and rendering).
   - Frame Debugger for draw-call/batching questions.
   - Memory Profiler package for memory growth/leaks.
   - Record specific numbers: ms per frame by category, GC alloc per frame, draw calls, setpass calls.
2. **IDENTIFY** — find the actual bottleneck: CPU main thread vs render thread vs GPU vs GC. Fixing the wrong one changes nothing.
3. **FIX** — the specific bottleneck only. One change at a time.
4. **VERIFY** — measure again, same scenario, compare numbers. No before/after numbers = not done.
5. **GUARD** — add a perf test or document the budget; note the profiler scenario for re-checking.

## Frame budget targets
- 60 FPS → 16.6 ms/frame; 30 FPS (mobile) → 33.3 ms.
- GC allocation in steady-state gameplay: target 0 B/frame (spikes cause hitches).
- Set a per-project budget (draw calls, setpass, texture memory) and check against it, not against "feels fine".

## Common Unity bottlenecks (check profiler FIRST, then look for these)

**CPU / scripts:**
- GetComponent, Find, FindObjectOfType, Camera.main in per-frame code → cache.
- LINQ, string concat/interpolation, boxing, closures in hot paths → GC spikes.
- Instantiate/Destroy churn → object pooling.
- Physics: per-frame casts without layer mask; SendMessage; excessive FixedUpdate rate.
- Debug.Log left in hot paths (allocates + serializes even in builds unless stripped).

**Rendering:**
- Draw calls / setpass: batching broken by material instances (`renderer.material` leak → use sharedMaterial or MaterialPropertyBlock).
- Overdraw (transparent UI/particles on mobile).
- Missing SRP batcher compatibility / GPU instancing on repeated meshes.

**UI (uGUI):**
- Canvas rebuild: any dirtied element rebuilds the whole canvas → split canvases by update frequency; don't set text/color every frame without change check.
- Layout groups + Content Size Fitter nesting in dynamic lists.

**Memory / loading:**
- Textures uncompressed or unnecessarily readable (Read/Write flag doubles memory).
- Addressables/AssetBundles not released; scenes loaded additively and never unloaded.
- Resources folder ballooning build size.

## Anti-patterns in the fix itself
- Optimizing code the profiler doesn't show as hot (premature).
- Micro-optimizing (for vs foreach) while a 5 ms GetComponent-in-Update sits there.
- Caching everything "just in case" — adds state bugs without measured need.
- Async/threading added for work that isn't the bottleneck.

## Verification checklist
- [ ] Before/after numbers exist with specific values (ms, B/frame, draw calls)
- [ ] Bottleneck identified in profiler, not assumed
- [ ] Same test scenario both measurements
- [ ] No new GC allocation introduced
- [ ] Tests still pass, gameplay unchanged
