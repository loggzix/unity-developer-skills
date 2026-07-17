---
name: unity-plan-review
description: Two-role adversarial review of an implementation plan BEFORE coding starts. Use when a Unity plan is ready (from unity-task-breakdown or user-provided) and it spans 3+ files, adds a new system, or the user asks to "review the plan" / "review plan này" / "duyệt plan" / "xem lại kế hoạch trước khi code". Role 1 challenges scope/product value, Role 2 challenges architecture/engineering. Not for reviewing written code (use unity-code-review) or single decisions (use doubt-driven-dev).
---

# Unity Plan Review

A plan reviewed only by its author is a plan reviewed by nobody. Run two fresh-context reviewers over the plan artifact sequentially, then reconcile.

Flow: Step 0 (input: raw plan → output: verbatim artifact) → Phase 1 → Phase 2 (each phase outputs ≤8 findings [F#], renumbered P#-n) → Reconcile (output: classified findings + DRAFT revised plan) → 🔴 Final gate (output: report + wait for user).

## Step 0 — Acquire artifact

1. Acquire the artifact via the branches below.
2. Apply the Completeness check.
3. Only then apply the S-size skip rule from Rules — size is judged from the written plan (announce in one line: "PLAN REVIEW SKIPPED: S-size (1-2 files, obvious scope)"); explicit review request always overrides.

Before spawning any reviewer, confirm you have the written plan. Completeness check = (a) numbered task list AND (b) each task names the files/assets it touches; missing either → 🔴 STOP, name the failed criterion ((a) or (b)), ask the user to complete the plan or route to unity-task-breakdown.
- User provided it inline → apply the Completeness check. Complete → proceed.
- Plan just produced by unity-task-breakdown in this conversation → use its final approved task list verbatim as the artifact; do not re-summarize.
- Plan lives in a file → read it first (`Read` tool), then apply the Completeness check — do not ask them to re-paste a file they already gave. Paste the file CONTENT verbatim into the reviewer prompt — never a path (reviewers are forbidden from reading files).
- Only described verbally → 🔴 STOP — no written artifact, no review: ask the user to paste the plan (or route to unity-task-breakdown first). Do not review a verbal summary or reconstruct the plan yourself; do not proceed past this point.
- Artifact = the verbatim plan PLUS user-stated project facts (existing systems, constraints — data of the plan, not framing); EXCLUDE your own rationale/preferences. Include the project's chosen architecture pattern (Singleton vs DI, one line) ONLY if the user or project CLAUDE.md states it — quote the source; never infer it from the codebase yourself at this step (inference happens in Reconcile).

## Pipeline

Each phase spawns a fresh-context agent that receives ONLY the plan artifact. Check this session's tool list first and use whichever of Agent/Task exists (neither listed → go straight to the Agent-tool-unavailable row of the If-X table); subagent_type: general-purpose. Spawn synchronously — do not spawn Phase 2 until Phase 1's output is in hand. The reviewer judges the artifact only and must NOT explore the codebase (verifying claims against the repo is Reconcile's job). Independence is the point.

Reviewer prompt template (the ONLY insertion point is the artifact per Step 0 — plan + user-stated project facts; never your rationale or framing):

```
You are reviewing an implementation plan for a Unity project. Find defects; do not validate, do not praise.
Judge ONLY the plan text below. Do NOT read files, grep, or explore any codebase — treat every claim in the plan as an unverified hypothesis and flag it if load-bearing.
Plan and stated project facts (verbatim):
<artifact per Step 0>
Answer EVERY checklist bullet below, one by one, in order — echo each bullet's bold title as a heading, then under it either report finding(s) or write `CLEAR — <one sentence of evidence>`; a skipped bullet invalidates the review. Report ONLY real defects — zero findings is a valid and respected answer. If every bullet is CLEAR, the FIRST line of your reply must be exactly NO DEFECTS FOUND, followed by the per-bullet CLEAR lines; never manufacture findings to appear useful. Hard cap 8 findings (a cap, not a quota). Each finding MUST name (a) a specific task/file/symbol from the plan and (b) a concrete replacement (API, pattern, or split); findings missing either will be discarded. Format each finding exactly as: **[F#] <title>** — <concrete defect>. Alternative: <specific replacement>.
Write your findings in the same language as the plan text below.
Derive every replacement from the plan's own domain and APIs; any illustrative mappings in the checklist are examples, never answers to copy — a replacement that does not fit this plan's domain is a discarded finding.
<Phase 1 or Phase 2 bullet list — copied VERBATIM from the phase section below; never paraphrase, reorder, or drop bullets: each is a mandatory checklist item>
```

The final report at the gate is always written in the conversation language.

### Phase 1 — Scope review (product hat)
Prompt the reviewer to answer:
- **Premise challenge:** is this the right problem? Is there a simpler feature that delivers the same player value?
- **Minimum viable scope:** what can be cut and shipped later? Produce an explicit **"NOT in scope"** list.
- **Leverage map:** which existing project systems (event system, pooling, save/load, UI navigator, addressables setup) already cover parts of this plan? New system proposed where one exists = finding.
- **Complexity trigger:** 8+ files touched, or any brand-new manager/singleton/system → propose a reduced-scope alternative.
- **Player-facing check:** for each task, what does the player see/feel? A task with no answer = a finding proposing the cut (goes on the NOT-in-scope list) unless the plan states a non-player-facing justification (infra, save integrity, compliance). A sub-task that only verifies or wires another task in the plan inherits that task's justification — judge the parent, never propose cutting the verification.

### Phase 2 — Engineering review (architect hat)
Prompt the reviewer to answer:
- **Architecture:** fits the pattern named in the stated project facts (Singleton vs DI)? No pattern stated AND the plan adds or restructures a system/manager → that omission is a finding; plan only edits existing files with no new system → write `CLEAR — pattern UNVERIFIED` for this bullet; no finding. Dependencies flow one way? asmdef boundaries respected?
- **Serialization impact:** any serialized field/class rename or data-shape change? Prefabs/scenes/SO assets that break? `[FormerlySerializedAs]` planned?
- **Lifecycle risks:** Category rule (governs): a lifetime must not exceed the widest consumer scope — for every proposed object lifetime, compare it to the scope of ALL its consumers; lifetime broader than every consumer's scope = wrong lifetime scope finding. Instances — illustrative, not exhaustive; derive from the plan's own lifetimes/tasks, never copy these as findings: execution order dependencies, domain-reload static state, scene load/unload, event subscribe/unsubscribe pairing; persistent/static/DontDestroyOnLoad serving a single scene, screen, or popup; async/loading lifetime — async/await or coroutines continuing after object destruction, Addressables/AsyncOperationHandle never released, tweens not killed on disable.
- **Test plan:** which parts are pure C# (EditMode-testable) vs MonoBehaviour glue (PlayMode) vs Inspector wiring (OnValidate assertion)? Untested critical path = finding.
- **Task granularity:** Category rule (governs): one task = one revertable unit — tasks must fit one session; a task too large or mixed to review/revert as one unit = finding; alternative: split into one code task + one wiring task with its own verification. Instances — illustrative, not exhaustive; derive from the plan's own lifetimes/tasks, never copy these as findings: a task touching a scene file + 5+ scripts is L regardless of line count; a task mixing script edits with prefab/scene rewiring across 3+ assets is unreviewable/unrevertable.
- **Performance budget:** per-frame cost of the design, GC allocation, draw-call impact for UI/rendering features.
  - Category rule (governs): never re-check per frame what changes rarely or is event-reported — flag ANY such polling; the finding must DERIVE the exact replacement for the plan's own domain (named event/callback/cached-timestamp API; "use events" alone is vague), even when it is not in the mappings below. To derive it, ask: which system already KNOWS the exact moment the condition becomes true (physics, animation, UI, tween, input, time)? That system's callback/event/cached timestamp is the replacement — name its concrete API.

  - Illustrative mappings (not exhaustive): collision/trigger → OnTriggerEnter/OnCollisionEnter; input → Input System callbacks; tween completion → OnComplete; e.g. waiting for a system to reach a state → that system's completion callback (arrival event, state-exit callback), not a per-frame state compare. A mapping matching the plan verbatim is not the finding — the finding must still cite the plan's own task/symbol and derive the replacement in the plan's terms.
  - Also flag per-frame Find/GetComponent.

As EACH phase returns (before spawning the next), renumber that phase's [F#] IDs to [P1-n] / [P2-n] — all later references (report table, changed-line suffixes) use the P#-n form.

Example finding:
> **[P2-1] Change `List<int> unlockedLevels` to `int[]` in SaveData with no migration** — shipped saves deserialize to an empty array, players lose progress. Alternative: keep the shape, or add a one-time migration step reading the old field.

## Reconcile — classify every finding yourself

Numbered recipe — run in execution order:
1. Locate the plan's target project root: Glob the named project folder, an Assets/ marker, or an .asmdef; ambiguous → ask the user; root unreachable from this session → mark EVERY codebase claim UNVERIFIED, downgrade to Taste.
2. For every finding that asserts a codebase fact (file exists, system exists, pattern used), verify it yourself BEFORE classifying:
   - 2a. User-stated facts count as VERIFIED: a fact the user stated in this conversation (existing system, chosen pattern) needs no recipe — cite the user's message as the log line; run 2b/2c only for claims neither the user nor the stated project facts cover.
   - 2b. Pattern inference (only when Phase 2 flagged a pattern omission): Grep the root for `static\s+\w+\s+Instance` (Singleton) vs `Container\.Bind|builder\.Register|\[Inject\]` (DI) — log hits per pattern, majority wins, note it as INFERRED.
   - 2c. Per-claim recipe, run against that root, not the cwd: Glob the claimed file name; Grep the claimed class/symbol (declaration AND usage); log one line per classification: pattern run + hit count — e.g. `P2-3: Glob **/BattlePassTooltip.cs → 1 hit; Grep "class BattlePassTooltip" → decl 1 / usage 4 → VERIFIED` (0 hits on both → claim false ONLY when the root was found; declaration-only hits → system exists but unused, say so). An unverified assertion cannot be Mechanical (no repo access → downgrade to Taste, mark UNVERIFIED). Phase 2's architecture bullet returned CLEAR — pattern UNVERIFIED → append PATTERN UNVERIFIED to Mode in the final report.
3. Dedupe cross-phase findings: Phase 1 and Phase 2 flag the same defect on the same task → keep the more specific finding, log the other as `duplicate-of P#-n` (one line); it counts in its phase's Findings column only.
4. Coverage re-check before closing: re-scan the plan — a defect matching ANY Phase-2 category bullet (architecture, serialization, lifecycle, granularity, performance) returned CLEAR → re-prompt that phase ONCE — re-spawn a FRESH agent with the full reviewer prompt template, replacing the bullet list with ONLY the missed bullet (copied verbatim; never the answer); same format rules apply; never reuse the prior reviewer's context; still CLEAR → record the miss and raise it yourself labeled RECONCILER-ADDED with ID [R-n], classified like any other finding; the report table gets a `| Reconciler | ... |` row and its changed lines are suffixed [R-n].
5. Re-read the plan against each finding and classify (discarding a bad finding is success, not waste):
- **Invalid** (finding is wrong, speculative scope expansion, or proposes a new system/abstraction the plan doesn't need) → discard, log one line why.
- **Mechanical** (one clearly right answer) → apply to the DRAFT revised plan in your report, log it — 🔴 never Edit/Write the plan file on disk before the user approves at the Final gate.
- **Taste** (reasonable disagreement) → apply the cleaner option, surface at final gate with your recommendation.
- **User challenge** (both reviewers flag the same premise, or strong evidence = a VERIFIED codebase fact — Glob/Grep with logged hits — directly contradicts the user's stated direction) → present to user with downside analysis (see Anti-patterns).

Decision principles when applying: explicit over clever (10 obvious lines beat 200-line abstraction), reuse over rebuild (DRY), pick the option covering more edge cases, bias toward action — flag concerns without blocking.

## If X → do Y

| If (situation) | Do | Still failing → fallback |
| --- | --- | --- |
| ONE reviewer run fails (errors out, empty, or off-format = per-bullet headings/CLEAR lines missing, or findings present without the **[F#]** pattern — a NO DEFECTS FOUND reply WITH per-bullet CLEAR lines is on-format, never re-spawn it) | Re-spawn that phase once with the identical prompt — always try this row before the Agent-tool row below | Second failure → self-simulate that phase only, label it SELF-SIMULATED (main conversation only — a subagent follows the subagent row below, which outranks this fallback) |
| Agent tool unavailable because you ARE a subagent | Escalate to the main conversation — never self-simulate from a nested context | Escalation impossible (autonomous run) → return the plan unreviewed, labeled UNREVIEWED — PLAN REVIEW PENDING; never self-simulate or implement |
| Agent tool itself unavailable or repeatedly failing in the main conversation (row-1 re-spawn already failed, or the tool does not exist) | Simulate the two roles yourself with hard separation — finish Phase 1 output completely before starting Phase 2; a self-simulated run still uses the exact reviewer prompt template: per-bullet headings/CLEAR lines, [F#] format, 8-cap — every format row of this table applies; label the report SELF-SIMULATED (weaker independence) | User needs full independence anyway → tell them to re-run from the main conversation where Agent is available |
| A reviewer returns vague findings ("consider improving X") | Vague = missing either (a) a named task/file/symbol from the plan, or (b) a named concrete replacement (API, pattern, or split). Reject and re-prompt once | Drop findings that stay vague |
| Reviewer exceeds the 8-finding cap or breaks the [F#] format | Keep the 8 highest-severity findings (severity order: wrong premise > data loss/serialization > lifecycle/crash > performance > scope/style), presented in the reviewer's original order; log dropped IDs one line each | Format still broken → re-prompt once with the cap + format as the FIRST line |
| Reviewer output shows it explored the codebase (cites file contents/paths not in the artifact) | Discard that phase's output — contaminated independence; re-spawn once with the prohibition as the FIRST line of the prompt | Second contamination → self-simulate that phase per the "ONE reviewer run fails" row's fallback, label SELF-SIMULATED |
| Plan artifact too large for one reviewer prompt (>400 lines OR >25 tasks) | Split by plan section/milestone into chunks of ≤200 lines AND ≤12 tasks, splitting only at section boundaries; run each phase per chunk with the same template; merge = drop duplicate findings hitting the same task/defect (keep the more specific), apply the 8-cap per phase across the merged set using the cap row's severity order, then renumber continuously per phase in merged order (P1-1..P1-8, P2-1..P2-8) — chunk-local IDs never appear in the report | Still too large → send back to unity-task-breakdown to split the plan itself |
| Reviewer asserts something about the codebase (file exists, pattern used) | Verify per the Reconcile recipe (Glob file / Grep symbol, log pattern + hit count) — reviewer claims AND plan premises are hypotheses | Can't conclude either way → ask the user, list what you checked |
| The plan's premise itself is wrong (feature exists, cited field/file doesn't) | That outranks all other findings — lead the report with it as a User challenge | User doesn't answer → terminal: plan stays blocked at the gate, log it as pending, do not implement any part |
| Both reviewers approve an obviously risky plan | Check your prompts: did they receive rationale or leading framing? Re-run the tainted phase clean | Clean re-run still approves → escalate to the user with your own risk analysis |
| Plan changed substantively during review | One re-run of the affected phase only — not an endless loop | Still churning after the re-run → the plan isn't stable enough to review; send it back to unity-task-breakdown |
| Phase 1 and Phase 2 findings directly contradict | Reconcile decides: correctness beats scope preference; verify the disputed claim in the codebase | Both defensible → surface as a Taste decision with your recommendation |
| User's gate answer is ambiguous or partial | Apply the clear parts; re-present ONLY the unanswered challenges (not the whole report) | Still unanswered after round 2 → plan stays blocked, log pending, implement nothing |

## 🔴 Final gate — STOP, present to user

```markdown
## PLAN REVIEW REPORT
| Phase | Findings | Applied | Taste | User challenges |
|-------|----------|---------|-------|-----------------|
| Phase N | n | n (P#-a, ...) | n (P#-b) | n (P#-c) |
**VERDICT:** approve (zero findings applied to the DRAFT — neither Mechanical nor Taste) / approve-with-changes (any Mechanical or Taste applied) / rethink-scope (any surviving premise/scope finding or User challenge). Evaluate top-down: rethink-scope, else approve-with-changes, else approve — a User challenge forces rethink-scope even with zero applied findings.
("Surviving" = an unanswered User challenge. There is no unsettled Taste — Reconcile always applies the cleaner option; a Taste the reconciler cannot decide is reclassified as a User challenge. A scope finding already applied as Mechanical/Taste, its task cut and logged under NOT in scope, counts as applied → approve-with-changes.)
**Mode:** independent / Phase N SELF-SIMULATED / chunk-merged (k chunks) / UNREVIEWED — PLAN REVIEW PENDING [+ " PATTERN UNVERIFIED" when Phase 2's architecture bullet returned CLEAR — pattern UNVERIFIED]
**NOT in scope:** [explicit list]
**User challenges:** [each with downside analysis, or "none"]
**Taste decisions:** [each with recommendation]
```

Each table cell = count (IDs), e.g. `| Phase 1 | 4 | 2 (P1-1, P1-3) | 1 (P1-2) | 1 (P1-4) |` — fill with real counts, never copy the example. The Findings column is the raw count including Invalid; Invalid findings appear in no other column — list each under the table as `Invalid: P#-n — <one-line reason>`. Applied column = Mechanical findings ONLY; Taste findings appear only in the Taste column even though their cleaner option is already applied to the DRAFT. No finding appears in two columns. Then the FULL revised plan verbatim (not a diff), each changed line suffixed [P1-n]/[P2-n] pointing at its finding. Exception: verdict approve → write PLAN UNCHANGED — ORIGINAL STANDS instead of re-pasting the plan; the report table and NOT-in-scope list are still mandatory. After the user answers: apply the answers and output: FINAL PLAN verbatim + one line per resolved finding (P#-n → user decision) + updated NOT-in-scope list — do NOT re-run the full review (re-run one phase only if an answer changed the plan substantively, per the If-X table). **User challenges block the gate until answered** (see Anti-patterns). The gate always waits: regardless of verdict — including approve — do not begin implementing until the user responds to the report.

## Anti-patterns

| Anti-pattern | Why it breaks | Instead |
| --- | --- | --- |
| Auto-deciding a User challenge | Overrides the user's stated direction | Present with downside analysis; wait |
| Coding "while waiting" on a User challenge | Sunk work prejudges the answer | Gate blocked until answered |
| Silent scope drift | Plan changes nobody approved or can trace | Every plan change traces to a logged finding |
| Nesting this skill from a subagent | Can't spawn reviewers or stop at the gate | Escalate to main conversation |
| Editing the plan file on disk before the Final gate | Destroys the user's original; approval becomes rubber-stamping a fait accompli | Apply findings only to the DRAFT in the report; disk writes only after gate approval |
| Feeding reviewers your rationale / leading framing | Anchors them — your own opinion echoed back | Plan artifact only |
| Reviewers seeing each other's output | Findings converge; independence lost | Sequential, isolated phases |
| Letting a reviewer grep/Read the codebase (or passing it a file path) | Anchors on the repo; plan claims stop being hypotheses | Paste plan content only; verification happens in Reconcile |
| Inventing findings on a clean plan to look useful | False positives erode trust in the whole review | Every finding points at a real defect; a clean plan gets verdict approve, said plainly |

## Rules
- One review cycle by default — re-run only if the plan changed substantively (= tasks added/removed, touched-file set changed, or a task's approach changed; wording/naming edits don't count). Sequencing and main-conversation-only are enforced in Pipeline and the If-X table.
- Skip entirely for S-size tasks (1-2 files AND no new system/manager, no serialized-field change, no prefab/scene rewiring) — announce it in one line: "PLAN REVIEW SKIPPED: S-size (1-2 files, obvious scope)" so the user can override — UNLESS the user explicitly asked for the review — then run both phases anyway, stating up front that S-size + a clean result (verdict: approve) is the expected outcome.