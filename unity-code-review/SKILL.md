---
name: unity-code-review
description: Multi-axis review for Unity C# changes before merge. Use when reviewing a diff/PR/branch in a Unity project, after completing a feature, or when evaluating code another agent produced. Covers correctness, Unity lifecycle, architecture, performance, and asset changes. Complements unity-developer (coding standards).
---

# Unity Code Review

Approval standard: approve when the change definitely improves overall code health, even if imperfect. Findings get severity labels: **Critical** (blocks merge) / *(no prefix)* required / **Nit** / **Consider** / **FYI**.

Review tests first — they reveal intent and coverage. Then walk the diff on six axes.

## The Six Axes

### 1. Correctness
- Matches spec; edge cases (null, empty, boundary); error paths, not just happy path.
- Unity fake-null: destroyed-object checks use `if (obj)` semantics correctly; no `?.` or `??` on UnityEngine.Object fields (bypasses lifetime check).
- Coroutines: stopped on disable/destroy? `yield` inside try/catch constraints respected?
- async: no `async void` outside event handlers; cancellation on destroy (destroyCancellationToken); no awaiting on dead objects.

### 2. Lifecycle & events
- Every subscribe has matching unsubscribe (OnEnable/OnDisable pairing, or OnDestroy for cross-scene).
- No heavy work or cross-object access in Awake that belongs in Start (init-order dependency).
- Static state: reset correctly with domain-reload disabled (`[RuntimeInitializeOnLoadMethod]` or explicit reset).
- Update/FixedUpdate/LateUpdate used for the right thing (physics in Fixed, camera follow in Late).

### 3. Architecture
- Follows project's chosen pattern (Singleton vs DI per unity-developer decision) — no new pattern smuggled in.
- No feature logic leaking into shared/core modules; dependencies flow one way.
- Serialized data shape changes reviewed for prefab/scene breakage; renames use `[FormerlySerializedAs]`.
- asmdef boundaries respected; no Editor code in runtime assemblies (`#if UNITY_EDITOR` or Editor folder).

### 4. Performance (hot paths = Update, physics callbacks, per-frame UI)
- No GetComponent/Find/Camera.main/LINQ/string concat/boxing/closure alloc in per-frame code.
- No Instantiate/Destroy churn where pooling exists in project.
- Physics: no mesh collider on moving objects casually; layer masks on casts.
- UI: no per-frame Canvas dirtying (text set every frame without change check).

### 5. Security & data
- No secrets in code or serialized assets; server-authoritative checks not trusted to client.
- PlayerPrefs/save data: external data validated at boundaries.

### 6. Asset diffs (scenes/prefabs/SO/meta)
- Scene/prefab YAML diffs: intentional changes only? (Unity dirties files it merely opened.)
- No missing script references, no accidental GUID changes.
- .meta files accompany every new asset; none orphaned.

## Change sizing
~100 lines good; ~300 acceptable if one logical change; 1000+ split it. Scene/prefab changes count as opaque — isolate from script changes when possible.

## Red flags
- "LGTM" without walking the diff.
- Review checks only that tests pass.
- Bug fix without regression test.
- Asset diff bundled with large code diff, unreviewed.
- Refactor that relocates complexity instead of removing it. Prefer the remedy that removes moving pieces.

## Output format
Findings as `file:line — Severity: problem. Suggested fix.` End with verdict: approve / approve-with-required-changes / request-changes, plus the verification story observed (tests run? compile clean? manual check?).
