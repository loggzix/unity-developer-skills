---
name: unity-plan-review
description: Two-role adversarial review of an implementation plan BEFORE coding starts. Use when a Unity plan is ready (from unity-task-breakdown or user-provided) and it spans 3+ files, adds a new system, or the user asks to "review the plan". Role 1 challenges scope/product value, Role 2 challenges architecture/engineering. Not for reviewing written code (use unity-code-review) or single decisions (use doubt-driven-dev).
---

# Unity Plan Review

A plan reviewed only by its author is a plan reviewed by nobody. Run two fresh-context reviewers over the plan artifact sequentially, then reconcile. Adapted from gstack's plan-ceo-review / plan-eng-review / autoplan pipeline.

## Pipeline

Each phase spawns a fresh-context agent (Agent tool) that receives ONLY the plan artifact — never your rationale, never the prior phase's findings. Independence is the point.

### Phase 1 — Scope review (product hat)
Prompt the reviewer to answer:
- **Premise challenge:** is this the right problem? Is there a simpler feature that delivers the same player value?
- **Minimum viable scope:** what can be cut and shipped later? Produce an explicit **"NOT in scope"** list.
- **Leverage map:** which existing project systems (event system, pooling, save/load, UI navigator, addressables setup) already cover parts of this plan? New system proposed where one exists = finding.
- **Complexity trigger:** 8+ files touched, or any brand-new manager/singleton/system → propose a reduced-scope alternative.
- **Player-facing check:** for each task, what does the player see/feel? Tasks with no answer are candidates for cutting.

### Phase 2 — Engineering review (architect hat)
Prompt the reviewer to answer:
- **Architecture:** fits project's chosen pattern (Singleton vs DI per unity-developer)? Dependencies flow one way? asmdef boundaries respected?
- **Serialization impact:** any serialized field/class rename or data-shape change? Prefabs/scenes/SO assets that break? `[FormerlySerializedAs]` planned?
- **Lifecycle risks:** execution order dependencies, domain-reload static state, scene load/unload, event subscribe/unsubscribe pairing.
- **Test plan:** which parts are pure C# (EditMode-testable) vs MonoBehaviour glue (PlayMode) vs Inspector wiring (OnValidate assertion)? Untested critical path = finding.
- **Performance budget:** per-frame cost of the design, GC allocation, draw-call impact for UI/rendering features. Flag designs that force per-frame Find/GetComponent.

Max 8 findings per phase. Each finding: concrete, with a proposed alternative — not just "consider".

## Reconcile — classify every finding yourself

Re-read the plan against each finding, then classify:
- **Mechanical** (one clearly right answer) → apply to plan silently, log it.
- **Taste** (reasonable disagreement) → apply the cleaner option, surface at final gate with your recommendation.
- **User challenge** (both reviewers or strong evidence say the user's stated direction is wrong) → NEVER auto-decide. Present to user with downside analysis.

Decision principles when applying: explicit over clever (10 obvious lines beat 200-line abstraction), reuse over rebuild (DRY), pick the option covering more edge cases, bias toward action — flag concerns without blocking.

## Final gate — present to user

```markdown
## PLAN REVIEW REPORT
| Phase | Findings | Applied | Taste | User challenges |
|-------|----------|---------|-------|-----------------|
**VERDICT:** approve / approve-with-changes / rethink-scope
**NOT in scope:** [explicit list]
**User challenges:** [each with downside analysis, or "none"]
**Taste decisions:** [each with recommendation]
```

Then the revised plan. No silent scope drift: every change to the plan traces to a logged finding.

## Rules
- Sequential phases; reviewers never see each other's output.
- Runs from main conversation only; escalate instead of nesting from subagents.
- One review cycle by default; re-run only if the plan changed substantively.
- Skip entirely for S-size tasks (1-2 files, obvious scope) — the review costs more than the risk.