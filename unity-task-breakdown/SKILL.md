---
name: unity-task-breakdown
description: Decompose Unity features into small, verifiable tasks before implementation. Use when a Unity feature/refactor spans 3+ files, touches scenes/prefabs/ScriptableObjects, feels too large or vague to start, or implementation order isn't obvious. Not for single-file changes with obvious scope.
---

# Unity Task Breakdown

Decompose work into tasks small enough to implement, compile-check, and verify in one focused session. Complements the `unity-developer` skill (coding standards) — this covers process, not code style.

## Process

### 1. Plan read-only first
- Read spec + relevant code (prefer codegraph_explore over raw Read).
- Identify existing patterns: how does this project do events, DI/Singleton, data assets?
- Map dependencies AND asset dependencies (which prefabs/scenes/SO assets reference the scripts you'll touch — renaming a serialized field or class breaks references).
- Note risks: domain reload behavior, execution order, serialization changes, platform differences.

### 2. Slice vertically
One complete playable path at a time — e.g. "enemy spawns and takes damage" before "all enemy types". Each task leaves the project compiling and the scene playable.

### 3. Write tasks in this shape

```markdown
## Task N: [title]
**Description:** one paragraph.
**Acceptance criteria:**
- [ ] Specific, testable in-Editor condition
**Verification:**
- [ ] Compiles clean (read_console: no errors/warnings from changed files)
- [ ] EditMode/PlayMode tests pass (if test asmdef exists)
- [ ] Manual: enter Play Mode, [exact steps + expected behavior]
**Dependencies:** Task numbers or "None"
**Files/assets touched:** scripts, prefabs, scenes, SO assets
**Scope:** S (1-2 files) | M (3-5) | L (5+ → break further)
```

### 4. Unity-specific ordering rules
- Data structures (serialized fields, ScriptableObjects) FIRST — changing them later loses serialized data or breaks prefabs.
- Pure C# logic before MonoBehaviour glue — testable without Play Mode.
- Scene/prefab wiring LAST — hardest to review in diffs, do it once when scripts are stable.
- Anything that renames a public serialized field: flag it, use `[FormerlySerializedAs]`.

## Sizing
Agent performs best on S/M tasks. A task touching a scene file + 5 scripts is L regardless of line count — scene diffs are unreviewable, isolate them.

## Red flags
- Starting implementation without written task list.
- Task mixing script changes and scene rewiring.
- No manual Play Mode verification step (tests alone don't prove gameplay).
- Serialized data shape changed mid-plan instead of first.

## Exit criteria
Every task has acceptance criteria + verification; dependencies ordered; scene/prefab edits isolated; user approved the plan.