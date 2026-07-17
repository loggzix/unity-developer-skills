---
name: unity-developer
description: Unity 6 C# coding standards, per-project Singleton-vs-DI architecture decision, code review, and failure-mode fixes; background reference for rendering, performance, platforms. Triggers when writing, reviewing, or refactoring Unity C# code, implementing features, setting up architecture, working with events, or reviewing changes. Triggers - viết script, viet script, viết manager, viet manager, refactor C#, review code Unity, sửa lỗi C#, tối ưu code Unity, singleton hay DI, MonoBehaviour, UniTask, DOTween, EditorWindow, custom inspector, viết tool editor, lag, giật lag, hiệu năng, object pool, pooling, popup, tween, fx, addressables, spawner, wave, save data, PlayerPrefs, lưu dữ liệu. NOT for MCP scene/asset manipulation (unity-mcp-skill) or profiler-data deep dives (unity-performance).
---

# Unity Development Standards

> ⚠️ **Unity 6 (C# 9):** All patterns and examples are compatible with Unity 6, which uses C# 9. No C# 10+ features are used.

## 🧭 STEP 0: Choose the Architecture (ASK FIRST on a new project)

> Entry point: STEP 0 is invoked from Order-of-Operations step 1 — a CLAUDE.md `## Architecture:` line always supersedes this ask; never ask when that line exists.

> 🔴 **STOP · CHECKPOINT** — On a **new** project, do NOT write any architecture-level code (managers, services, events, data access) until the developer answers the question below. Ask, wait for the answer, then proceed. (Existing project → skip: detect and follow the architecture already in use.)

**Architecture is NOT fixed.** The question:

> **"Dự án này dùng kiến trúc nào — Singleton hay DI (VContainer/SignalBus)?"**

Then follow that choice consistently for the whole project. Quick guidance to help them decide:

| Use **Singleton** when… | Use **DI** when… |
| --- | --- |
| Team of **1–4 people** | Team of **5+ people** |
| Small / casual / hyper-casual game | Large, long-lived, live-ops game |
| Few global managers, clear lifecycle | Many interlocking systems, complex lifecycles |
| Little/no unit testing (test by playing) | Serious unit testing / TDD on business logic |
| Ship fast, low overhead | Need swappable interfaces, multi-platform/SDK variants |

> ✅ **Default recommendation to GIVE the user for small teams (1–4 devs): Singleton.** It informs their answer — it never replaces the ask: on a NEW project you still stop at the checkpoint even for a solo dev. Do not impose DI on a small project.

**Rules for choosing:**

- **New project →** ASK (checkpoint above), record the decision in the project's CLAUDE.md (`## Architecture: Singleton|DI`) so later sessions read it instead of re-detecting. No reply possible (non-interactive) → the git-log-majority fallback of the BOTH-signals failure row; empty repo → default Singleton for team ≤4, state the assumption in one line, record it provisionally.
- **Existing project →** user states the architecture explicitly in the prompt ("project theo Singleton pattern", "project dùng VContainer") → trust it, output `Architecture: <X> (stated by user)` — do NOT run detect just to verify; ONLY if opposite-path signals surface incidentally while editing (`LifetimeScope`/`[Inject]` in a stated-Singleton project or vice versa), say so and follow the code. Otherwise detect signals: a `Singleton<T>` base class means Singleton; a `LifetimeScope`/`[Inject]` means DI. Both or neither → Failure Modes table.
- After the choice, apply the matching section below: **[Path A: Singleton](#path-a-singleton-default-for-small-teams)** or **[Path B: DI](#path-b-di-vcontainer--signalbus-or-di-container--publishersubscriber)**.

## ▶️ Order of Operations (follow in sequence)

Quick map: Write new code → all steps | Review/fix pasted code → Surgical rule + 1a-1b, 2, 4b-4e, 6 | Perf complaint → 🔴 Performance measure-first checkpoint FIRST | Editor tool → Editor tooling patterns | Question-only → Architecture line + Checklist: n/a. Mid-task: re-read this card before replying.

0. Prompt contains a performance complaint (lag / giật / slow / tối ưu / fps drop — any language, any verb form) → jump to the [🔴 Performance measure-first checkpoint](#-performance-gate) NOW; no other step may emit code edits until that gate resolves.

1. **Architecture — sub-steps:**
   - 1a. Locate the project root: `git rev-parse --show-toplevel` from the edited file's dir; no repo → walk parents until one contains Assets/ (max 6 levels); no files identified yet → same walk from the cwd.
   - 1b. Read `<root>/CLAUDE.md` for a `## Architecture: Singleton|DI` line — found → output `Architecture: <X> (from CLAUDE.md)`, skip detect/ask entirely. The prompt explicitly states a CONTRADICTING architecture → follow the user for this session, output `Architecture: <X> (stated by user, overrides CLAUDE.md <Y>)`, offer to update the line — never silently follow either side.
   - 1c. Not found → run **STEP 0** (ask-vs-detect rules live there). Output: `Architecture: <X> (stated by user | detected via <signal> | asked)`.
   - 1d. Surgical request (pasted/pointed code to review/fix/refactor) → still do 1a-1b (cheap, never blocks), NEVER run STEP 0; start at step 2.
2. **Before writing any code** → apply **🔴 Code Quality Rules** (access modifiers, `readonly`/`const`, `nameof`, minimal comments). Output: none — enforced in the code, verified at step 6.
3. **Before writing any NEW system/helper** (popup/screen, pool, tween sequencer, navigator, data accessor…) → search first, in sub-steps:
   - 3a. Derive 2-3 nouns from the request ("reward popup" → Popup|Reward).
   - 3b. Run: Grep `class \w*(<noun1>|<noun2>|Manager|Controller|Service|System|View)` glob `**/*.cs`; Grep `<FeatureKeyword>` glob `**/*.cs`; Glob `**/*<FeatureNoun>*/**/*.cs`; plus the project's real type names (taken from usings/fields of the files being edited).
   - 3c. Open at most the 3 most recently modified hits (relevant = class name contains a step-3a noun OR its public members already cover the requested behavior); 0 relevant → read the remaining hits up to 5 total; still nothing → `Searched: … → nothing found, writing new`. Found one → extend/reuse it and say so. Output one line before any new code: `Searched: <patterns> → reusing <X>` or `Searched: <patterns> → nothing found, writing new`. No line = step skipped. A parallel system next to an existing one = defect.
4. **Writing/refactoring** — sub-steps, each with its own output:
   - 4a. Write, applying the matching Path A/B rules + Modern C# patterns.
   - 4b. For EACH changed file, run Grep with pattern `Find(First|Any)?Object|FindObjects?(Of|By)Type|GameObject\.Find|Instantiate\(|Destroy\(|DO[A-Z]\w*\(|DOTween\.Sequence|\.SetDelay\(|PlayerPrefs|\.Instance|Camera\.main|SendMessage|Invoke\("|Resources\.Load|\.Result|\.Wait\(\)|async void|StartCoroutine\(|new WaitForSeconds|InvokeRepeating\(|Addressables\.` (output per file: `<file>: N hits`). Zero files on disk (pasted snippet) → scan the snippet text with the same alternation.
   - 4c. Check the 6 structural triggers the regex cannot catch: popup/screen open-close · persistent one-shot flag · collecting ALL instances of a type · tween on a pooled object · coroutine started on a pooled/disable-able object · component lookup inside a per-frame or per-rebind path.
   - 4d. Every 4b hit or 4c match MUST name its Failure-Mode row (copy the row's First fix into the code) or be marked `(legit, no row)` naming ONE class from this closed whitelist:
     • one-shot Instantiate/Destroy (level build, single popup) • Camera.main cached once in Start • GetComponent(s) caching in Awake/Start • PlayerPrefs behind the project save wrapper • .Instance already null-guarded per HudView • StartCoroutine/new WaitForSeconds on a non-pooled, never-disabled object with the WaitForSeconds cached.
     Addressables hits are NEVER legit-only — name the handle-release or load-fail row. A hit naming neither a row nor a listed class = rewrite before step 6. 0 hits AND 0 matches → `Triggers: none`. Output: the `Triggers:` contract line.
   - 4e. Verify compile: `read_console(action="get", types=["error","warning"], count=20)` after save — errors gate the line; warnings never block: 0 errors → `Compile: clean (N warnings listed)`, each warning becomes a 🟢 suggestion. Editor still compiling → poll editor_state (ReadMcpResourceTool, uri mcpforunity://editor_state) until isCompiling=false, max 5 polls ~3s apart; still compiling → `Compile: unverified (editor busy)` + re-check 🔴 Quality Rules on the git diff. No MCP connection → `Compile: unverified (no editor connection)` + the same diff re-check. Pasted snippet with no file on disk → skip 4e, emit `Compile: n/a (pasted snippet — no file on disk)` and re-check the Quality Rules on the corrected snippet. Output: the `Compile:` contract line.
   - 4f. **Verify runtime** (compile != works) — code has a branch/loop/state/tween/save/player-visible flow → run it: `manage_editor(action="play")`, drive it minimally (click the button / invoke the entry via a cheat), `read_console(types=["error"])` to catch runtime errors, then `manage_editor(action="stop")`. Pure data/util class, editor tool, or no MCP connection → skip, state the reason in one line. Output: the `Verify:` contract line.
5. **Hit a runtime/edge failure AFTER writing** → consult **🔁 Failure Modes & Fallbacks**; name the matched row in one line before applying its fix. (Design-time trigger coverage lives in step 4's reconcile.)
6. **ALWAYS before finishing** → self-run the **Code Review Checklist** on your own diff (this is step 2's verification). Output (always, even unasked): one line — `Checklist: pass` or `Checklist: <N> violations fixed (<rule names>)`. Output contract — output only the APPLICABLE lines, each copied verbatim then filled; omit inapplicable lines and strip the trailing # comments. An omitted line whose condition applied = that step was skipped; go back and run it. Pure Q&A replies (no C# code, no review verdict) emit the Architecture line (when steps 1a-1c ran) plus `Checklist: n/a (no diff — question-only reply)`. A reply containing any C# code block MUST place this contract block FIRST, before the code — contract after code or absent = invalid reply, rewrite:

```
Architecture: <X> (from CLAUDE.md | stated by user | detected via <signal> | asked | n/a (surgical))
Searched: <patterns> → <reusing X | nothing found, writing new>   # only when proposing a new system
Compile: <clean | N errors fixed | unverified (editor busy) | unverified (no editor connection) | n/a (pasted snippet — no file on disk)>
Verify: <ran (behavior ok) | ran (fixed N runtime issue) | skipped (<reason>) | n/a (no runtime surface)>   # from step 4f; only when code written
Failure-Mode: <row name (direct | structural: same <lifecycle|ownership|churn> as <trigger>)>   # step-5 consults only; step-4 hits go ONLY in Triggers
Triggers: <none | n/a (report-only) | N hits: <row name or legit-class> each>   # from step 4; surgical review with no code written → n/a; do not repeat rows in Failure-Mode
Ease: <convention Ease.X (N hits, <family>) | default OutBack/OutQuad | default ease — no project to grep | n/a (no tween)>   # only when a tween was written
Surgical: <n/a | N edits / M report-only>   # surgical mode only — every edit names its 🔴 row or behavior defect
Persistent base: <found X | none | n/a>   # only when writing a scene-persistent manager

Checklist: <pass | N violations fixed (<rule names>) | n/a (no diff — question-only reply)>
```

Example (feature write in a Singleton project):
```
Architecture: Singleton (from CLAUDE.md)
Searched: class \w*(Popup|Reward|Manager) → reusing ScreenNavigator
Compile: clean (0 warnings)
Verify: ran (behavior ok)
Triggers: 2 hits: Instantiate ×1 → churn row; .Instance ×1 (legit, HudView-guarded)
Checklist: pass
```
Copy the SHAPE — values always from the current task, never from this example.

Surgical note: `n/a (surgical)` means never run STEP 0 or block on the architecture question; the Surgical rule surfaces it as a NON-blocking question ONLY when a path-specific rule would change a verdict; otherwise omit it. Review requests additionally append to the template, one line per finding: `🔴|🟡|🟢 <checklist rule name> — line N: <defect> → <one-line fix>`, ending `Verdict: <pass | N critical / M important>` — severity taken from Review Severity Levels, never invented.

## 🔴 CRITICAL: Code Quality Rules (CHECK FIRST — both architectures)

ALWAYS enforce these BEFORE writing any code, regardless of architecture:

1. **Use least accessible access modifier** — `private` by default
2. **Handle errors deliberately** — catch ONLY where a concrete recovery exists (name it); otherwise let it throw; empty catch or catch-log-continue with no fallback = 🔴 Critical
3. **Use `readonly` for fields** — Mark fields that aren't reassigned (`[SerializeField]`/inspector fields are exempt)
4. **Use `const` for constants** — Constants should be `const`, not `readonly`
5. **Use `nameof` for strings** — Never hardcode property/parameter names (the project's `[ClassName]` log-prefix convention is exempt)
6. **Comments** — Comment ONLY non-obvious "why" (a tricky reason, a gotcha). A comment that restates what the code says → delete it; code that needs a "what"-comment → rename identifiers instead. No verbose XML docs unless a public API genuinely needs one.
7. **Minimum structure for NEW code** — no state machine, interface, factory, or extra layer unless the feature demonstrably needs it TODAY; the reuse found in step 3 always beats a new system. Any new abstraction gets a one-line justification or gets deleted.

```csharp
// ✅ EXCELLENT: Quality rules enforced (architecture-neutral)
public sealed class DamageCalculator
{
    private const int CritMultiplier = 2;

    public int Calculate(int baseDamage, bool isCrit) =>
        isCrit ? baseDamage * CritMultiplier : baseDamage;
}

#if UNITY_EDITOR
public class EditorTool
{
    public void Process() => Debug.Log("Processing..."); // Debug.Log OK in editor
}
#endif
```

## 🟡 Modern C# Patterns (both architectures)

```csharp
// ✅ GOOD: LINQ instead of loops (NOT in hot paths — see Performance checklist)
var activeEnemies = allEnemies.Where(e => e.IsActive).ToArray();

// ✅ GOOD: Expression bodies
public int Health => currentHealth;

// ✅ GOOD: Null-coalescing
var name = playerName ?? "Unknown";

// ✅ GOOD: Pattern matching
if (obj is Player player) player.TakeDamage(10);
```

---

# Path A: Singleton (default for small teams)

Use this when STEP 0 picked Singleton. **This is the recommended default for 1–4 dev projects (matches the STEP 0 table).**

### Rules

Singleton-style projects typically use **three accepted forms of global access** — pick the one that matches the type, do NOT force everything into a `Singleton<T>` base. **Detect which forms the project already uses and follow them; don't impose ones it doesn't have.**

1. **MonoBehaviour singleton + `.Instance`** — for managers that ARE MonoBehaviours (audio, SDK, navigation, etc.). Inherit the project's shared singleton base; do NOT hand-roll a new one per class.
2. **`static class` + static accessor property** — for **plain-C# data/config holders**. These deliberately do NOT inherit the MonoBehaviour singleton base (they aren't MonoBehaviours). This is a valid global, NOT an "ad-hoc singleton".
3. **Service-locator (`Register`/`Resolve`)** — for **non-MonoBehaviour** objects created with `new` and shared globally.

- ✅ Access MonoBehaviour managers via `.Instance`; plain-C# data via its static accessor; `new`-created services via the service-locator. Match the existing convention.
- ✅ Use an `Initialize()` method on managers; call them in a defined order from a single top-level initializer.
- ✅ Manager that must survive scene loads (audio, SDK/ads, save) — decision chain: Grep `PersistentSingleton|DontDestroyOnLoad` glob `**/*.cs` → output `Persistent base: <found X | none>`. Found → inherit it. None → inherit the plain base with the exact Awake override below. scene-scoped managers → plain base. State which variant you picked. No persistent variant exists in the project → inherit the plain base with exactly this Awake override (guard BEFORE base.Awake()/DDOL — wrong order leaves a 1-frame duplicate or a stray DDOL object): `protected override void Awake() { if (Instance != null && Instance != this) { Destroy(gameObject); return; } base.Awake(); DontDestroyOnLoad(gameObject); }` — do NOT create a second generic base.
- ✅ Designer-tuned config (volumes, rates, costs, curves, balance numbers) → ScriptableObject/config asset referenced via `[SerializeField]` or the project's config layer — hardcoded magic numbers in managers = 🟡 Important.
- ✅ Static/global data access is fine — no Controller layer required. If a data holder is **async-initialized**, guard access with its init flag (the accessor is null/empty until init completes).
- ✅ Events: C# `event`/`Action` or `UnityEvent` (Singleton projects also commonly communicate via direct `.Instance` calls). When you DO use events, **always unsubscribe** in `OnDisable`/`OnDestroy`.
- ✅ Logging: `Debug.Log`/`Debug.LogError` is acceptable in runtime; prefix messages with `[ClassName]` for traceability.
- ✅ Use UniTask for async; thread a `CancellationToken` through awaits — e.g. `await UniTask.Delay(1000, cancellationToken: destroyCancellationToken);` in a MonoBehaviour, or accept a `CancellationToken ct` parameter and pass it to every await in the chain.

### Singleton base class

Before writing a MonoBehaviour singleton, **search the project for an existing shared base** (common names: `Singleton<T>`, `StaticInstance<T>`, `PersistentSingleton<T>`, `MonoSingleton<T>`). If one exists, **inherit it; never redefine it** (a duplicate generic base = compile error). Read the base before relying on it — implementations differ: some destroy duplicate instances and some don't, some auto-create the GameObject when missing and some return null. 🔴 CHECKPOINT — creating a brand-new base (nothing found): show the search patterns tried and their empty results, ask "Project chưa có singleton base — tạo mới Singleton<T>?", and wait for confirmation before writing it.

### Example manager

```csharp
public sealed class ScoreManager : Singleton<ScoreManager>
{
    private int score;
    public int Score => score;

    public void Initialize() => score = 0;

    public void Add(int amount)
    {
        score += amount;
        OnScoreChanged?.Invoke(score);
    }

    public event System.Action<int> OnScoreChanged;
}

// Subscriber must unsubscribe:
public sealed class HudView : MonoBehaviour
{
    private bool subscribed;

    private void OnEnable()
    {
        if (ScoreManager.Instance != null) { ScoreManager.Instance.OnScoreChanged += Refresh; subscribed = true; }
    }

    private void Start()
    {
        if (!subscribed && ScoreManager.Instance != null) { ScoreManager.Instance.OnScoreChanged += Refresh; subscribed = true; }
    }

    private void OnDisable()
    {
        if (subscribed && ScoreManager.Instance != null) ScoreManager.Instance.OnScoreChanged -= Refresh;
        subscribed = false;
    }

    private void Refresh(int score) { /* update UI */ }
}
```

---

# Path B: DI (VContainer + SignalBus OR DI Container + Publisher/Subscriber)

Use this ONLY when STEP 0 picked DI (large team / long-lived project).

### Choose ONE stack

**Detect before choosing:** project has `using VContainer`/`LifetimeScope`/`SignalBus` → Option 1; has `IPublisher<`/`ISubscriber<` → Option 2; brand-new DI project with neither → default to Option 1 and say the default was chosen. Never mix the two stacks.

**Option 1: VContainer + SignalBus**

- ✅ VContainer for dependency injection
- ✅ SignalBus for events
- ✅ `[Preserve]` attribute on constructors

**Option 2: DI Container + Publisher/Subscriber**

- ✅ DI container wrapper
- ✅ `IPublisher`/`ISubscriber` for events
- ✅ `[Inject]` attribute on constructors

**DI universal rules:**

- ✅ Use Data Controllers (NEVER direct data access)
- ✅ Inject an `ILogger` via constructor for runtime logging (no `#if` guards, no `[prefix]`; never CALL the logger inside a constructor — wiring the dependency there is required and fine); use `logger.Method()` directly (DI guarantees non-null)
- ✅ Always implement `IDisposable` and unsubscribe from every signal
- ✅ Unload assets in `Dispose`
- ✅ Use UniTask for async

Data Controller shape (the raw model type never leaves its controller):

```csharp
public sealed class LevelDataController
{
    private readonly LevelData data;

    [Preserve]
    public LevelDataController(LevelData data) { this.data = data; }

    public int CurrentLevel => this.data.currentLevel;
    public void SetLevel(int value) => this.data.currentLevel = value;
}
```

### Unity Architecture (VContainer)

```csharp
using UnityEngine.Scripting;
using VContainer.Unity;

public sealed class GameService : IInitializable, IDisposable
{
    private readonly SignalBus signalBus;
    private readonly LevelDataController levelController;

    [Preserve]
    public GameService(SignalBus signalBus, LevelDataController levelController)
    {
        this.signalBus = signalBus;
        this.levelController = levelController;
    }

    void IInitializable.Initialize() => this.signalBus.Subscribe<WonSignal>(this.OnWon);
    void IDisposable.Dispose()       => this.signalBus.TryUnsubscribe<WonSignal>(this.OnWon);
}
```

### Unity Architecture (DI Container + Publisher/Subscriber)

```csharp
public sealed class GameService : IAsyncEarlyLoadable, IDisposable
{
    private readonly IPublisher<WonSignal> publisher;
    private readonly IDisposable subscription;

    [Inject]
    public GameService(IPublisher<WonSignal> publisher, ISubscriber<WonSignal> subscriber)
    {
        this.publisher = publisher;
        this.subscription = subscriber.Subscribe(this.OnWon);
    }

    public void Dispose() => this.subscription?.Dispose();
}
```

---

## Code Review Checklist

### Surgical rule (any request to review/fix/refactor code the user pastes or points to, regardless of verb or language)

Refactor-to-standard = surgical mode: only checklist defects produce changed lines.

- Pasted snippet with no project context → do NOT run STEP 0 and do NOT ask the architecture question: apply Universal + Unity Specifics rows only; path-specific items (.Instance vs inject) become questions in the answer, not edits. Ask ONLY if a path-specific rule would change a verdict — never as a reflex. No file on disk → `Compile: n/a`; re-check the Quality Rules on the corrected snippet instead.
- Pasted/pointed code WITH stated or discoverable context (the prompt names the stack, or the file lives in a project with a CLAUDE.md `## Architecture:` line) → apply that path's checklist rows directly, output `Architecture: <X> (stated by user | from CLAUDE.md)`; reserve `n/a (surgical)` for the truly context-free case.
- Scene-scan Find APIs are ALWAYS edit-producing in surgical mode even in Awake/Start where severity is only 🟡; over-permissive visibility (public field that should be [SerializeField] private) is 🔴 Critical per Severity Levels and always edit-producing.

- Every changed line must trace to a real defect from the checklists/severity levels below.
- Do NOT add `sealed`, rename identifiers, restyle, or introduce interfaces/factories/abstractions the user didn't ask for — even if they match the standards for NEW code.
- Style upgrades you notice but don't apply → list them as 🟢 suggestions in the answer, never as silent edits.
- 🟡 style-tier items (verbose loop → LINQ, expression bodies, naming, missing readonly/const/nameof, magic-number extraction — none change behavior) are REPORT-ONLY in surgical mode: list as suggestions, never edit — only 🔴 rows and behavior defects (tween state, Find, async void, leaks) produce changed lines.
- Tween rows in Unity Specifics are BEHAVIOR defects, not style: a show-tween with no explicit start state relies on leftover scale/alpha — fix start state + deliberate ease + SetLink together. Only a purely cosmetic ease swap on an otherwise-correct tween is 🟢 report-only.

### Universal (both architectures)

- [ ] All access modifiers correct (`private` by default)
- [ ] Errors handled deliberately (throw vs log-and-fallback)
- [ ] `readonly` used for non-reassigned fields, `const` for constants
- [ ] `nameof` used instead of string literals
- [ ] LINQ used instead of manual loops (except hot paths = Update/FixedUpdate/LateUpdate or any code executed once per frame or more, incl. per-frame tween/scroll callbacks)
- [ ] Expression bodies (single-expression members), null-coalescing, pattern matching used when they shorten code without hiding logic
- [ ] UniTask used for async; `CancellationToken` threaded through
- [ ] No `async void` — fire-and-forget returns `async UniTaskVoid` (call with `.Forget()`); awaited flows return `UniTask`
- [ ] Events/signals unsubscribed (in `OnDisable`/`OnDestroy` or `Dispose`)

### Singleton path only

- [ ] Global access uses an accepted form (MonoBehaviour singleton + `.Instance`, `static class` + static accessor for plain-C# data, or service-locator for `new`-created services) — NOT a hand-rolled MonoBehaviour singleton
- [ ] MonoBehaviour managers inherit the project's shared singleton base; accessed via `.Instance`, null-guarded at every access point that can run during scene load/teardown (see HudView example)
- [ ] Async-initialized data/static holders guarded with their init flag — accessor is null/empty before init completes
- [ ] `Initialize()` called in a defined order from the top-level initializer

### DI path only

- [ ] VContainer/DI Container used correctly
- [ ] SignalBus/Publisher-Subscriber used correctly
- [ ] Data accessed through Controllers only
- [ ] Injected `ILogger` (no guards/prefixes/constructor logs); `logger.Method()` not `logger?.Method()`
- [ ] `[Preserve]` or `[Inject]` attribute on constructors
- [ ] `IDisposable` implemented; assets unloaded in `Dispose`

### Unity Specifics (both)

- [ ] No scene-scan Find API anywhere in runtime code: `FindObjectOfType`, `FindObjectsOfType`, `FindObjectsByType`, `FindAnyObjectByType`, `FindFirstObjectByType`, `GameObject.Find`, `GameObject.FindWithTag` — swapping a deprecated Find for its Unity 6 replacement is NOT a fix (→ Failure Modes table)
- [ ] No `GetComponent` in Update/runtime loops — cache in `Awake`/`Start`
- [ ] `TryGetComponent` used instead of `GetComponent` + null check
- [ ] Lifecycle contract: `Awake` = cache self refs/components only; `OnEnable` = subscribe; `Start` = touch other objects (`.Instance` safe here); `OnDisable` = unsubscribe + `DOKill`; `OnDestroy`/`Dispose` = release handles. Cross-object access in `Awake` = defect. Exception: subscribing to a manager's event in `OnEnable`: null-guard the `.Instance` access (see HudView example); Instance can be null at `OnEnable` time → re-attempt the subscribe in `Start`, otherwise the subscription is silently lost
- [ ] Tweens (DOTween) on pooled/disable-able objects killed on despawn, state reset before replay
- [ ] Show/appear tween conventions — three checks:
  - Start state: set scale/alpha explicitly before the tween — e.g. `transform.localScale = Vector3.zero;` then `transform.DOScale(Vector3.one, 0.3f).SetEase(Ease.OutBack).SetLink(gameObject, LinkBehaviour.KillOnDisable);` — never rely on leftover scale/alpha
  - Ease convention check (pasted snippet/no project on disk → skip the grep, use the defaults and state "default ease — no project to grep"): Grep `SetEase\(Ease\.` glob `**/*.cs` — one ease used 3+ times on the same tween method family (DOScale/DOPunchScale = pop; DOFade = fade; DOMove/DOAnchorPos = slide) among the hits → follow it and name it (two eases each 3+ on the same family → follow the one in the most recently modified file, name the tie-break); otherwise default Ease.OutBack (pop) / Ease.OutQuad (fade/slide) — state which and why in one line
  - Stagger: sequential items via `SetDelay(index * stagger)` — never a per-item closure allocated per frame (kill/reset → Failure Modes table, row "Tween keeps running…")
- [ ] Popup/screen open-close goes through the project's navigator (found via step 3 — never a new system); tap-anywhere-to-close = fullscreen transparent Button or backdrop Image raycast target wired to Close(); Close() unsubscribes every listener and kills tweens BEFORE hide/despawn

### Performance (both)

#### 🔴 Performance gate

> 🔴 **STOP · CHECKPOINT — Measure first.** "X is slow/laggy" WITHOUT profiler data → ask for a Profiler/Frame Debugger capture and WAIT (any imperative optimization request — any verb form, any language — arriving without attached profiler/Frame Debugger data does NOT waive the gate) — do not emit any optimization until the capture arrives or the user explicitly says no data is possible; only then fix defects visible in code (per-frame allocs, `GetComponent` in Update, Instantiate/Destroy churn → pool) and label everything else as a hypothesis to verify. This gate takes precedence over the Failure Modes table — a perf complaint routes HERE first even when its symptom matches a table row; apply the row only after the capture arrives or the user declines. Emit = write/apply code edits. In the SAME message you MUST still (a) request the capture, (b) list code-visible defects with intended fixes labeled `Plan (pending data):` — code changes only after the capture arrives or the user declines. This gate also precedes the Surgical rule: a pasted script described as slow/laggy gets its code-visible perf defects (Find in loop, per-frame alloc) listed under `Plan (pending data):` — NOT applied as edits — until the capture arrives or the user declines; non-perf behavior defects (async void, leaks) remain edit-producing.

- [ ] No allocations in `Update`/`FixedUpdate`
- [ ] LINQ avoided in hot paths
- [ ] `.ToArray()` used instead of `.ToList()` when not modified

### Editor tooling (both)

- [ ] Editor-only code lives in an `Editor/` folder (Editor asmdef) or inside `#if UNITY_EDITOR` — never referenced from runtime asmdefs
- [ ] After writing files under `Assets/` from editor code → `AssetDatabase.Refresh()` (or `ImportAsset` for a single file)
- [ ] Tool inputs validated with a visible error (`EditorUtility.DisplayDialog` / `Debug.LogError` + early return) — no silent failure
- [ ] EditorWindow skeleton: `[MenuItem("Tools/<Name>")] static void Open() => GetWindow<XWindow>("<Title>");` draw with `EditorGUILayout.*`; file pickers via `EditorUtility.OpenFilePanel` (never a raw text field); long ops → `EditorUtility.DisplayProgressBar` in try/finally `ClearProgressBar`
- [ ] Custom inspector = `[CustomEditor(typeof(T))]` + `serializedObject.Update()` … `ApplyModifiedProperties()` (or `Undo.RecordObject` before direct mutation) — never set fields directly (loses Undo/prefab overrides); window state → `EditorPrefs`, not static fields

## 🔁 Failure Modes & Fallbacks (if X → do Y)

> 🔴 **STOP · CHECKPOINT** — never add UniTask/DOTween/VContainer (or any package) to a project that lacks it; ask and WAIT for approval before touching manifest.json.

Recovery branches, not just "don't". Contract routing: see the Failure-Mode/Triggers comments in the step-6 template (canonical). Rows match by STRUCTURE, not keyword — apply a row to any code with the same lifecycle/ownership problem even when the trigger noun differs (a GDPR-consent flag is the one-shot-flag row; a mute-all toggle silencing every active audio source is the ALL-objects row; a kill-feed line is the churn row; a low-health vignette pulse on a pooled HUD is the pooled-tween row; a server-driven price is the async-init-data row). When you hit these at runtime/review, apply the fix:

| If (trigger) | First fix | Still failing → fallback |
| --- | --- | --- |
| `.Instance` null during scene load/teardown | Null-guard before use | Defer to next frame, or skip the call this frame |
| Async-init data/static holder read before ready | Check its init flag first | Await init, or return default/empty until flag is true |
| Need persistent small player-state (any small scalar that must survive restarts — a seen-once flag, a running counter, a best-combo record, a last-claim timestamp) | Use the project's save/data layer (grep `DataManager`, `*SaveData*`, `*UserData*`, PlayerPrefs wrapper); key = `const string`. Multi-field or progression-critical data → state the PlayerPrefs-vs-save-file tradeoff in one line (PlayerPrefs = small scalars, unencrypted, easily cleared; save file = structured/versionable/migratable) before picking. | No wrapper exists → raw `PlayerPrefs`: read once into a cached bool, `Save()` immediately on set |
| No shared singleton base found in project | Search names (`Singleton<T>`/`PersistentSingleton<T>`/…) before writing | Only then → run the 🔴 CHECKPOINT in Path A "Singleton base class" (show empty search results, ask, WAIT), then create ONE base — never a second generic base (duplicate = compile error) |
| Event/signal subscribed but object may be destroyed | Unsubscribe in `OnDisable`/`OnDestroy`/`Dispose` | If invoke still NREs, null-check handler at raise site |
| Addressables/asset handle not released | Release in `OnDestroy`/`Dispose` | Track handles in a list, release all on teardown |
| Addressables load fails / key missing at runtime | Check `handle.Status == AsyncOperationStatus.Succeeded` after await; failed → log the key + fall back to a `[SerializeField]` default asset | Recurring for one key → validate the key in the Addressables group at editor time; never retry-loop at runtime |
| `await` continues after object destroyed | Thread a `CancellationToken` through every await | Check `this == null`/`destroyCancellationToken` after resume |
| `GetComponent`/`Find` in a hot loop | Cache the reference in `Awake`/`Start` | `TryGetComponent` once + store result |
| Tempted to `FindObjectOfType` for a single object | `[SerializeField]` wire in Inspector, or `.Instance`/DI for managers | Event the target subscribes to, so no reference is needed at all |
| Need ALL objects of a type (spawned/pooled instances) | Objects self-register in a collection owned by their manager (e.g. `HashSet<T>`) in `OnEnable`, remove in `OnDisable` — stays correct under pooling. Iterate a SNAPSHOT (`foreach (var e in registry.ToArray())`) when the effect can despawn/disable members mid-loop — mutating the set during enumeration throws | Broadcast event the objects subscribe to |
| Tween keeps running on a pooled/disabled object | `SetLink(gameObject, LinkBehaviour.KillOnDisable)` at tween creation (plain `SetLink(go)` only kills on Destroy, not `SetActive(false)`) | `DOKill()` in `OnDisable`/before returning to pool; reset scale/pos/alpha before reuse |
| Repeated `Instantiate`/`Destroy` churn (list items, projectiles, spawn waves) | Object pool: reuse instances, reset state (scale/alpha/subscriptions) on rent/return | scrolling collections → reuse or build a view-recycler (step 3 search first); defer heavy per-item asset loads (icons/sprites) to the project's async asset layer — Addressables when the project uses it — releasing the handle on recycle; the view receives data via a `Bind(data)` call from the list owner — never a Find/GetComponent lookup per rebind |
| Existing project shows BOTH architecture signals (`Singleton<T>` AND `LifetimeScope`/`[Inject]`) or neither | Treat as a new project: ASK the user (STEP 0) | User unavailable or declines to decide → run `git log --pretty=format: --name-only -30 -- "*.cs"`, take the first 5 non-empty paths, open those files, count Singleton-vs-DI signals, follow the majority (0-vs-0 in those 5 → widen to -100 commits; still 0 → default Singleton for team ≤4 per the STEP 0 table); state the assumption in one line, record it in CLAUDE.md as provisional |
| Project lacks UniTask/DOTween (checklist items reference them) | Use Unity 6 `Awaitable` or coroutines / built-in animation; treat those checklist items as "when the package is present" | → package checkpoint above the table |
| Editor tool input invalid (no file picked, CSV missing columns, write path outside Assets/) | `EditorUtility.DisplayDialog` naming the exact bad field + early return — never throw or silently no-op | Recurring bad input → validate in OnGUI, disable the action button (`GUI.enabled = false`) until inputs pass |
| `read_console` shows compile errors after an edit | Fix the FIRST error (later ones usually cascade), recompile | Still failing after 2 rounds → 🔴 STOP · CHECKPOINT: run `git diff <file>` first — file holds uncommitted changes beyond your own edits → 🔴 ask before reverting; only session edits → show the exact diff being discarded, ask "Discard these edits?" and WAIT for approval before `git checkout -- <file>`. Report the exact error, propose a different approach |

## Common Mistakes to Avoid

### ❌ DON'T (both architectures)

- Anything the Review Checklist above bans — the checklist is canonical; do not re-derive rules from memory. Extra rationale worth keeping in mind:
- Scene-scan Find APIs: full variant list + canonical rule live in the Unity Specifics checklist row — banned even in Awake/Start (scene-scan cost + hidden dependency); when refactoring one away, say why it was removed (fixes → Failure Modes table); severity: see Review Severity Levels
- `async void` swallows exceptions and can't be awaited/cancelled → `async UniTaskVoid` + `.Forget()` for fire-and-forget, or return `UniTask`
- `.Result`/`.Wait()`/`GetAwaiter().GetResult()` on Task/UniTask → always await: sync-over-async blocks or deadlocks the main thread
- Mutate a ScriptableObject asset at runtime expecting it to persist — changes survive only in the editor; runtime-writable state goes through the save layer (→ one-shot-flag row)
- Poll state in Update (checking a flag/value every frame) when an event/callback exists → subscribe instead (OnScoreChanged pattern)
- `new WaitForSeconds(...)` inside a loop or repeatedly-invoked coroutine → cache the instance, or UniTask.Delay
- Coroutines on pooled/disable-able objects → silently stopped by SetActive(false); use UniTask + CancellationToken (→ await-after-destroy row)
- `Camera.main` in per-frame code → cache once in `Start` (even with Unity 2020.2+ internal caching it is an avoidable per-frame property call, and the cached field survives when no camera is tagged MainCamera) · string-based `Invoke("X", t)`/`SendMessage` → direct call or `UniTask.Delay` (no compile check, breaks on rename) · `Resources.Load` for new runtime content when the project uses Addressables

### ❌ DON'T (DI path only)

- Use `Debug.Log` in runtime code → Use an injected `ILogger`
- Add conditional guards or manual prefixes to logs → the logger handles them
- Log in constructors → keep constructors fast — no logging or I/O; DI subscription wiring is the one allowed side effect (see the DI Option 2 example)
- Use `logger?.Method()` → Use `logger.Method()` (DI guarantees non-null)
- Use Zenject → Use VContainer
- Use MessagePipe directly → Use SignalBus
- Access data models directly → Use Controllers

### ❌ DON'T (Singleton path only)

- Hand-roll a new **MonoBehaviour** singleton per class → inherit the project's shared singleton base
- ⚠️ Do NOT "fix" a plain-C# `static class` data/config holder by making it inherit the MonoBehaviour singleton base — it isn't a MonoBehaviour; the static-class form is intentional and correct
- Chain deep `A.Instance.B.Instance.C...` → chains ≥3 `.Instance` deep, or the same chain in 5+ call sites → flag it in review and raise the DI question with the team
- Forget to null-guard `.Instance` during scene load/teardown, or access an async-initialized data holder before its init flag is true

## Review Severity Levels

### 🔴 Critical (Must Fix — both)

- Missing or over-permissive access modifiers (e.g. a public field that should be [SerializeField] private)
- `async void`, or sync-over-async (`.Result`/`.Wait()`/`GetAwaiter().GetResult()`) in runtime code — behavior defects, always edit-producing even in surgical mode
- Empty catch, or catch-log-continue with no named recovery (Quality Rule 2)
- Memory leaks (not unsubscribing from events/signals)
- Any scene-scan Find variant (full variant list: Unity Specifics first row — canonical) in per-frame or repeatedly-called code

> 🟡 In one-shot code (`Awake`/`Start`): Important, not Critical — still replace (→ Failure Modes table).

### 🔴 Critical (DI path only)

- Any violation of the ❌ DON'T (DI path only) list above = Critical (Debug.Log in runtime, log guards/prefixes, constructor logging, `logger?.`, Zenject/MessagePipe, direct data access, missing `IDisposable`)

### 🟡 Important (Should Fix)

- Missing `readonly` (field not reassigned outside initializer/constructor, `[SerializeField]` exempt) or `const` (compile-time constant)
- Hardcoded strings (use `nameof()`)
- Verbose code instead of LINQ
- Performance issues in hot paths
- (DI) Missing `[Preserve]`/`[Inject]`; verbose unnecessary logs
- Hardcoded designer-tunable magic numbers in managers (Path A rule — move to ScriptableObject/config asset)
- (DI path, or project already has a test assembly) Missing unit tests for new business logic
- Public API of a shared library/package missing XML docs — internal game code needs none (Quality Rule 6)

### 🟢 Nice to Have (Suggestion)

- Could use expression body / null-conditional / pattern matching
- Could improve naming
- Could simplify with modern C# features

---
# Reference: Unity Capabilities (background only)

> Background knowledge — NOT defaults or recommendations. The **Standards + STEP 0 above govern all decisions.** Pull deeper detail only when a task needs it.

Unity 6 LTS across these areas: **rendering** (URP/HDRP, Shader Graph, HLSL, VFX Graph, post-processing) · **performance** (Profiler/Frame Debugger/Memory Profiler, LOD, culling, Job System + Burst, DOTS/ECS, GC tuning) · **assets** (Addressables, asset bundles, texture/audio compression, ScriptableObject data-driven design) · **UI** (UI Toolkit, uGUI, Input System, localization) · **physics/anim** (Unity/Havok physics, state machines + blend trees, Timeline, Cinemachine, IK) · **audio** (Audio Mixer, 3D spatial, Wwise/FMOD) · **networking** (Netcode for GameObjects, Mirror, relay/lobby) · **platforms** (mobile/console/PC/WebGL tuning + store cert) · **tooling/DevOps** (custom editor tools, Unity Cloud Build CI, Git LFS) · **testing** (Unity Test Framework, play/edit mode, memory-leak detection).

Need specifics on any area (exact API, workflow, trade-offs) → `unity_docs(action="lookup", query="<API or topic>")` (batch via `queries=`) against the project's Unity version; WebSearch only if unity_docs returns nothing; unity_docs unavailable (no Unity MCP) → WebSearch `site:docs.unity3d.com <API> <version>`; still nothing → say the API is unverified — never answer from memory.
