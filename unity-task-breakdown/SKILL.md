---
name: unity-task-breakdown
description: Decompose Unity features into small, verifiable tasks before implementation. Use when a Unity feature/refactor spans 3+ files, touches scenes/prefabs/ScriptableObjects, feels too large or vague to start, or implementation order is not obvious. Not for single-file changes with obvious scope. Triggers - breakdown, chia task, chia nhỏ task, chia nho task, chia thành task list, break into a task list, plan feature này, lên kế hoạch làm, plan feature nay, len ke hoach lam, breakdown task này, break this down, break down this feature, task breakdown, plan this feature.
---

# Unity Task Breakdown

Decompose work into tasks small enough to implement, compile-check, and verify in one focused session. Complements the `unity-developer` skill (coding standards) — this covers process, not code style.

## Process

### 1. Plan read-only first
Input: the user's feature request/spec (+ any linked docs).
- Read spec + relevant code (codegraph_explore if connected+indexed; otherwise grep/Read — never skip the read or guess a pattern from memory).
- Identify existing patterns: how does this project do events, DI/Singleton, data assets?
- Map dependencies AND asset dependencies (which prefabs/scenes/SO assets reference the scripts you'll touch — renaming a serialized field or class breaks references).
- Note risks: domain reload behavior, execution order, serialization changes, platform differences.
- **Output:** four lists (patterns to reuse / scripts to touch / asset dependencies / risks) delivered as a "Khảo sát" section at the top of the breakdown message — step 2's input (the estimate formula consumes scripts + assets from here) and the user's evidence the read happened.

### 2. Slice vertically
Input: the four lists from step 1 (patterns to reuse / scripts to touch / asset dependencies / risks).
One complete playable path at a time — e.g. "enemy spawns and takes damage" before "all enemy types". Each task leaves the project compiling and the scene playable.
**Output:**
- Named slices; slice 1 = the thinnest end-to-end playable path crossing the riskiest item on step 1's risk list; state the risk it retires.
- Risk order: serialized-data change > save/persistence migration > SDK callback > domain-reload/execution-order > pure UI. A risk not in this list:
  - Blast radius B: `guid=$(grep -o "guid: [0-9a-f]*" <Script>.cs.meta | cut -d" " -f2); grep -rl "$guid" Assets/ --include=*.prefab --include=*.unity --include=*.asset | wc -l` (or codegraph_explore blast radius). Script is NEW (no .meta yet) → B = number of prefabs/scenes/SO assets the plan says will reference it. Run from the Unity project root (the dir containing Assets/); <Script>.cs.meta sits beside the script. No bash → Read the .meta for the guid, then Grep pattern=<guid> glob=*.{prefab,unity,asset} — B = files_with_matches count.
  - Threshold R = blast radius of this feature's own serialized-data risk (compute each B with the guid-grep above; R = max B across the scripts named in that risk); no serialized-data risk → R = 3.
    | B vs R | Tier placement |
    |---|---|
    | B ≥ R | top tier (serialized-data) |
    | R > B ≥ ceil(R/2) | between save-migration and SDK-callback |
    | B < ceil(R/2) | between domain-reload and pure UI |

    Example: no serialized-data risk → R=3, ceil(R/2)=2; unlisted risk script with B=4 → top tier; B=2 → middle band; B=1 → bottom band.
- Estimate per slice (= task count): ceil((scripts + assets THIS slice touches) / 3), minimum 1 — partition step 1's lists across slices first; an item touched by multiple slices counts only in the earliest one. Scene/prefab wiring adds +1 isolated task AFTER the ceil (see Sizing).
- **Output thêm:** an Estimate table delivered with the breakdown — one line per slice `Slice N → scripts: X, assets: Y → ceil((X+Y)/3) + W wiring = Z tasks` (W = 1 if the slice ends in scene/prefab wiring, else 0) + a total line `Sum = T`. Step 3's full-vs-outline decision MUST cite T; no table → step 3 does not start.

### 3. Write tasks in this shape
Order within step 3:
- 3.0 Glob **/*Test*.asmdef ONCE (matches Test/Tests/PlayModeTests naming) — found → keep the tests row in EVERY task; not found → omit it from EVERY task (no dangling criterion).
- 3.1 Write DRAFT tasks for the in-scope slices using the template below.
- 3.2 Derive "chốt trước khi làm" questions via Probe lenses AGAINST the draft (impact rank: per the Probe lenses section).
- Then: 0 questions → do NOT deliver here; continue to step 4 and present ONCE at the Exit gate. ≥1 question → deliver questions block FIRST + the task list marked DRAFT (pre-ordering), STOP at the 🔴 CHECKPOINT. Never present the same list twice as final.
Input: the slices from step 2. Sum of step-2 estimates ≤ 8 tasks → write full tasks for ALL slices in slice order; larger → full tasks for slice 1 only, one-line outline per remaining slice (name + goal + expected task count + risks), and state explicitly in the delivered list: **"Slices 2+ are outlines — full tasks written after slice 1 is approved and green."** Outline shape: `**Slice 2 — Garage paint-shop UI** · goal: car recolors instantly on swatch pick · ~3 tasks · risk: serialized paint state.` Each slice becomes 1-N tasks; a task never spans two slices.

```markdown
## Task N: [title]
**Description:** one paragraph.
**Acceptance criteria:**
- [ ] Specific, testable in-Editor condition
**Verification:**
- [ ] Compiles clean (read_console filtered per changed filename via filter_text: 0 errors, 0 warnings; Unity MCP not connected → check the Editor Console manually, or run Unity -batchmode -quit -projectPath <project root> -logFile build.log (Windows: full Editor path, e.g. "C:\Program Files\Unity\Hub\Editor\<ver>\Editor\Unity.exe") and grep -E "(error|warning) CS" — 0 error hits required, warning hits from changed files = fail, pre-existing warnings elsewhere = pass)
- [ ] EditMode/PlayMode tests pass (keep/omit per step 3.0)
- [ ] Manual verification — pick exactly one: (a) task touches any scene/prefab/SO asset → enter Play Mode, [exact steps + expected behavior]; (b) pure-C# task AND a Tests asmdef exists (step 3.0) → EditMode test replaces Play Mode; (c) pure-C# task, no asmdef → compile-clean + note "Play-Mode observable via slice-final task N". The slice's FINAL task always uses (a)
**Dependencies:** Task numbers or "None"
**Files/assets touched:** scripts, prefabs, scenes, SO assets
**Scope:** S (1-2 files) | M (3-5) | L (6+ files) — never plan an L task directly: split until every piece is S/M; the ONLY deliverable L is L-EXCEPTION per the If-X compiling row. Scene/prefab + scripts mixed in one task → split per Sizing
```

**Filled example (anchor your acceptance-criteria granularity to this):**

> **## Task 1: WaveConfig SO + wave progress field**
> **Description:** Create `WaveSpawnConfig` (ScriptableObject) with enemy prefab refs + spawn interval; add `lastClearedWave` to save data (append-only).
> **Acceptance criteria:** [ ] SO asset creatable via Create menu, fields visible in Inspector; [ ] wave index increments after wave clear and survives domain reload.
> **Verification:** [ ] Compiles clean (read_console: 0 errors, 0 warnings from changed files); [ ] Manual: enter Play Mode, clear wave 1 → wave 2 spawns after interval, restart Play Mode → resumes at wave 2.
> **Dependencies:** None · **Files:** WaveSpawnConfig.cs, ProgressSave.cs, Assets/Configs/Wave01.asset · **Scope:** S

**Output:** the full task list in this exact template, every field filled — no "TBD"; anything unknown becomes a question in "chốt trước khi làm" instead. Deliver in chat; write to a file only when the user asks to persist it.

🔴 **CHECKPOINT:**
- **STOP condition:** if "chốt trước khi làm" has ≥1 question, present the questions + assumed answers and STOP for the user's answers BEFORE running step 4 ordering — unless the user already authorized proceeding on assumptions (then mark affected tasks ASSUMED and continue).
- **Pre-authorized exception:** The lowest-risk-reading ASSUMED fallback (If-table ambiguity row) applies ONLY when the user pre-authorized assumptions or the run is non-interactive — in an interactive session, STOP means wait.
- **When answers arrive:** an answer that changes slice composition or risk order → return to step 2 and re-slice; otherwise re-run step 3 for every affected task (update Description/AC/estimates). Then proceed to step 4 with the revised list — never order a list written against superseded assumptions.
- **Question shape:** **Q[n]:** [question] | Options: [A/B] | Assumed: [X — lowest-risk because ...] | Impacts: Task #, #.

### Probe lenses (sub-step 3.2 — run against the DRAFT, before the 🔴 CHECKPOINT)
Walk the feature's state lifecycle through four lenses: (a) trigger/clock — what starts or resets state, by which clock/timezone/window; (b) failure path — what happens on miss, cancel, error, offline; (c) data source — where do the tunable numbers live: hardcoded constant vs SO asset vs remote config; (d) persistence & migration — where state lives, what happens to old saves. Examples from other domains — crafting queue: slot count source, cancel/refund behavior, offline progression, queue save format; inventory stacking: max-stack source, overflow behavior on full bag, stack merge on load. Each axis whose plausible answers change the task list becomes a question — cap 5, ranked by impact = number of tasks whose Description/AC would change between the plausible answers (tie → serialized-data impact first, per step 2's risk order); same task list either way → a stated assumption, not a question.

### 4. Unity-specific ordering rules
- Data structures (serialized fields, ScriptableObjects) FIRST — changing them later loses serialized data or breaks prefabs.
- Pure C# logic before MonoBehaviour glue — testable without Play Mode.
- Scene/prefab wiring LAST, isolated (see Sizing), do it once when scripts are stable.
- Anything that renames a public serialized field: flag it, use `[FormerlySerializedAs]`.
- Analytics/tracking calls ride inside the task that owns the state change they report (the purchase task fires the purchase event) — never a separate analytics task or slice.
**Output:** the task list reordered per the first three bullets (data → pure C# → wiring), applying the FormerlySerializedAs and analytics-placement rules while reordering; tasks renumbered and every Dependencies field updated to the new numbers — verify: every Task N depends only on tasks < N (forward dependency found → reorder again). (Input: the task list from step 3.)

## Sizing
Agent performs best on S/M tasks. A task touching a scene file + 5 scripts is L regardless of line count — scene diffs are unreviewable, isolate them into their own task. A wiring-ONLY task sizes normally by file count, never auto-L. VALUE tweak = changing an existing serialized value on one prefab/scene/SO with no field added/renamed/retyped → sub-threshold (below breakdown); any data-STRUCTURE change is always above threshold.

## If X → do Y

| Trigger | Fix | Still stuck → fallback |
| --- | --- | --- |
| Request is sub-threshold (1-2 files, obvious scope; a VALUE tweak per the Sizing note counts as sub-threshold) | Skill mis-fire — state in one line "Below breakdown threshold" and implement (or explain) directly; do NOT produce a task list, even a 1-task list | User insists on a list anyway → deliver a 1-task list marked SUB-THRESHOLD, skip steps 1-2 |
| Feature (or part of it) already exists in the codebase | Shrink the breakdown to the delta — say explicitly which requirements are already covered and where; don't plan work that's done | Unsure whether existing code fully covers a requirement → add a leading VERIFY-EXISTING task (read + one Play Mode check) instead of assuming coverage |
| Requirement ambiguous (2+ readings change the task list) | Put the questions at the TOP as "chốt trước khi làm" with an assumed answer per option — don't silently pick one; derive them via **Probe lenses** (under step 3), cap 5 | Pre-authorized / non-interactive → lowest-risk reading, mark affected tasks ASSUMED; interactive → STOP per the 🔴 CHECKPOINT (the full exception rules live THERE — do not re-derive them here) |
| A task won't leave the project compiling on its own | Merge it with its pair (interface + first implementor together); never plan a red-compile checkpoint | Merged task becomes L → keep the merge (compiling beats sizing), mark L-EXCEPTION with the reason; truly circular → add a temporary stub in the same task, delete it in the follow-up task, list the stub under Files/assets touched |
| Serialized data change discovered mid-breakdown | Reorder: it becomes Task 1 (or its own prerequisite task); flag `[FormerlySerializedAs]`/append-only ordering (MemoryPack et al.) | Change can't be append-only (old shipped saves break) → add a save-migration task before Task 1, AC: old save loads with no data loss |
| Task depends on an external decision (store product id, PM copy, remote config) | Isolate it into its own task marked BLOCKED-ON-X so the rest can proceed | Blocker sits on slice 1's critical path → re-slice so slice 1 routes around it, or stub the value as a placeholder constant (listed under Files touched, replaced when unblocked) |
| Task involves an async SDK callback (IAP, rewarded ad, cloud save, remote config — may arrive after scene unload or off the main thread) | Isolate the callback flow into its own task; acceptance criteria MUST enumerate success / fail / cancel / timeout separately; include a guard: marshal to main thread + scene-validity check before touching UI | SDK thread model unverifiable from docs → mark BLOCKED-ON-SDK, or stub the callback interface and mark ASSUMED-SYNC |
| Sum of step-2 estimates T > 8 (decided upfront by the Estimate table; drafted count exceeds 8 despite T ≤ 8 → this row re-fires, trailing slices convert to outlines) | The feature is an epic — apply step 3's >8 rule (full tasks for slice 1 only, one-line outline per remaining slice); slices not yet formed → group into 2-3 vertical slices first | Slice 1 alone still >8 → re-slice it into a thinner end-to-end path; can't shrink → the riskiest item becomes its own spike slice |
| Can't find how the project does X (events, saves, pooling) | That's a read gap, not a guess opportunity — locate the existing pattern first (codegraph/grep), then break down | Both codegraph and grep whiff → mark the task BLOCKED-ON-PATTERN and ask the user; never invent a new pattern |

## 🔴 Exit gate
**STOP — present the task list and wait for user approval before implementation starts.** Open questions ("chốt trước khi làm") must be answered or explicitly deferred by the user, not by you. If the user rejects or edits the list: revise only the affected tasks, re-check the ordering rules, re-present the full list — never implement from a partially approved list.

**Gate checks (before presenting):** verify mechanically: (1) every Dependencies number < the task's own number; (2) every Scope letter matches the Files/assets count bands; (3) no task lists both a scene/prefab AND scripts; (4) every AC names an action + observable result. Any check fails → fix, then present; a check still failing after 2 fix passes → present anyway with that check flagged FAILED-GATE + a one-line reason — never silently drop or loop the check.

**After approval:** implement exactly one task at a time in dependency order, running that task's Verification block green before opening the next (red after 2 fix attempts → STOP on that task, load the unity-debugging skill — do not open the next task, do not silently reorder); load the unity-developer skill before the first code edit; task list changes mid-implementation → return to this gate. For >8-task epics: when slice 1's tasks are all verified green, return to step 3, expand the NEXT slice's outline into full tasks (re-reading only what changed), and re-pass this gate before implementing it — outlines are never implemented directly.

## Red flags
- Starting implementation without written task list — nothing to review at the exit gate.
- Task mixing script changes and scene rewiring (see Sizing).
- No manual Play Mode verification at the END of each slice — tests alone don't prove gameplay (per-task exceptions: template options b/c).
- Duplicate work and mid-plan serialized changes: covered in the If X → do Y table above.
- Do NOT estimate hours/days — size only via S/M/L.
- Do NOT produce a task list for a sub-threshold change (see the first If-X row) — even a 1-task list is waste (sole exception: user insists after the refusal → SUB-THRESHOLD 1-task list per that row's fallback).
- Do NOT write acceptance criteria as "works correctly"/"chạy đúng" — name an observable in-Editor behavior (what you click, what you see).
- Do NOT slice horizontally (Task 1: all data, Task 2: all logic, Task 3: all UI). Observability is per SLICE, not per task: a pure-C# task verifies via EditMode test/compile-clean, but the slice it belongs to must end Play-Mode observable.
