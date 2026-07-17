---
name: unity-mcp-skill
description: Orchestrate Unity Editor via MCP (Model Context Protocol) tools and resources. Use when working with Unity projects through MCP for Unity - creating/modifying GameObjects, editing scripts, managing scenes, editing prefabs, running tests, installing packages, profiling, or play-mode control via MCP. Triggers - GameObject, prefab, scene, component, batch_execute, screenshot, play mode, compile errors, Unity console, UXML/USS, gắn component, sửa prefab, tạo scene, chụp màn hình scene, lỗi compile, chạy test Unity, material, shader, animation clip, particle, VFX, ProBuilder, vật liệu, hiệu ứng particle, audit asset, missing reference, kiểm tra reference thiếu, plan-only, dry-run, Cinemachine, UI Toolkit, cài package Unity, post-processing, profiler.
---

# Unity-MCP Operator Guide

## Template Notice

Examples in `references/workflows.md` and `references/tools-reference.md` are reusable templates. Templates are NOT guaranteed across Unity versions/package setups (UGUI/TMP/Input System). After applying one: run `read_console(types=["error"], count=10)`; if the change is visual → also `manage_camera(action="screenshot", include_image=True, max_resolution=512)`; payload rejected → read `mcpforunity://scene/gameobject/{id}/components` for the real shape, re-send.

Before applying a template:
- Validate targets/components first via resources and `find_gameobjects`.
- Treat names, enum values, and property payloads as placeholders — resolve real field/enum names by reading `mcpforunity://scene/gameobject/{id}/components` BEFORE filling them in.

## Quick Start: Resource-First Workflow

**Always read relevant resources before using tools.** This prevents errors and provides the necessary context.
How to read a resource: use the MCP client's resource-read call (e.g. `ReadMcpResourceTool(server="UnityMCP", uri="mcpforunity://editor/state")`, or your client's equivalent) — every "Read mcpforunity://..." line below means exactly this call; a client without resource reads falls back per-resource: console → `read_console`; scene state → `manage_scene(action="get_active")`; `editor/state` has NO direct tool equivalent — probe readiness by issuing `read_console(count=1)` and treating a busy error as not-ready (`get_active` does not expose is_compiling).

```
0. HARD GATE, every task (read-only included): read mcpforunity://editor/state.
   Mutating task → proceed only when ready_for_tools == true.
   Read-only task on a not-ready editor → note staleness, resource reads only.
1. Check editor state     → mcpforunity://editor/state
2. Understand the scene   → manage_scene(action="get_hierarchy", page_size=50); GO resource schema doc: mcpforunity://scene/gameobject-api
3. Find what you need     → find_gameobjects or resources
4. Take action            → pick the tool from the Core Tool Categories table below (one row per domain; two rows match → the more specific domain row wins; NO row matches → stop and ask — execute_code is a last resort per its row, never a default)
5. Verify results         → read_console, manage_camera(action="screenshot"), resources
```

## Critical Best Practices

### 1. After Writing/Editing Scripts or Major Changes: Wait for Compilation and Check Console

```python
# COMPILE-WAIT — mandatory after create_script, script_apply_edits, OR apply_text_edits:
# 0. Baseline the console BEFORE the edit: read_console(types=["error"], count=10), record existing errors.
#    After compile, only errors NOT in the baseline block progress — pre-existing errors are
#    reported to the user, never auto-fixed as part of this task.
# Both tools already trigger AssetDatabase.ImportAsset + RequestScriptCompilation automatically.
# No need to call refresh_unity — just wait for compilation to finish, then check console.

# 1. Poll editor state until compilation completes. Loop:
#   (a) read mcpforunity://editor/state
#   (b) is_compiling or is_domain_reload_pending → wait recommended_retry_after_ms if present, else 1000ms
#   (c) exit when is_compiling == false AND ready_for_tools == true
#   (d) after 60 iterations OR 180s total elapsed (whichever first — recommended_retry_after_ms can be multi-second) → read blocking_reasons: mentions a dialog/modal → tell the user to close it in the Editor; empty/unreadable → editor appears hung, suggest focusing the Unity window; either way fire NO more tool calls
#   (e) editor/state read fails mid-loop → domain reload in progress: follow Error Recovery "Connection lost"
#       (wait 5s → mcpforunity://instances → set_active_instance → resume); failed reads do NOT count toward the 60
#   (f) client cannot read resources → same loop via probe: read_console(count=1), busy error = still compiling;
#       same 1000ms wait and 60-iteration cap

# 2. Check for compilation errors
read_console(types=["error"], count=10, include_stacktrace=True)

# After any other major change (scene/prefab/asset edits), check console too:
read_console(action="get", types=["error", "warning"], count=10, format="detailed")
```

**Why:** Unity must compile scripts before they're usable. `create_script`, `script_apply_edits`, and `apply_text_edits` already trigger import and compilation automatically — calling `refresh_unity` afterward is redundant. `refresh_unity` has exactly one use: files changed OUTSIDE MCP script tools (git pull, external editor) — then `refresh_unity(scope="all", compile="request")`.

### 2. Use `batch_execute` for Multiple Operations

```python
# 10-100x faster than sequential calls
batch_execute(
    commands=[
        {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube1", "primitive_type": "Cube"}},
        {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube2", "primitive_type": "Cube"}},
        {"tool": "manage_gameobject", "params": {"action": "create", "name": "Cube3", "primitive_type": "Cube"}}
    ],
    parallel=True  # Hint only — never depend on ordering; dependent chains MUST use fail_fast=True instead
)
```

**Max 25 commands per batch by default (configurable in Unity MCP Tools window, max 100).** More than 25 operations → split into sequential batch_execute calls of ≤25 each (30 creates = 25 + 5); never fall back to per-command calls. Use `fail_fast=True` for dependent operations.

**Rule:** 3+ read-only lookups (find_gameobjects, manage_asset search, get_info) in one task → send them as ONE batch_execute; sequential same-shape reads are a DON'T:
```python
batch_execute(commands=[
    {"tool": "find_gameobjects", "params": {"search_term": "Camera", "search_method": "by_component"}},
    {"tool": "find_gameobjects", "params": {"search_term": "Player", "search_method": "by_tag"}},
    {"tool": "find_gameobjects", "params": {"search_term": "GameManager", "search_method": "by_name"}}
])
```

### 3. Use Screenshots to Verify Visual Results

```python
# Basic screenshot (saves to Assets/Screenshots/, returns file path only)
manage_camera(action="screenshot")

# Inline screenshot (returns base64 PNG directly to the AI)
manage_camera(action="screenshot", include_image=True, max_resolution=512)

# Use a specific camera and cap resolution for smaller payloads
manage_camera(action="screenshot", camera="MainCamera", include_image=True, max_resolution=512)

# Batch surround: captures front/back/left/right/top/bird_eye around the scene
manage_camera(action="screenshot", batch="surround", max_resolution=256)

# Batch surround centered on a specific object
manage_camera(action="screenshot", batch="surround", view_target="Player", max_resolution=256)

# Positioned screenshot: place a temp camera and capture in one call
manage_camera(action="screenshot", view_target="Player", view_position=[0, 10, -10], max_resolution=512)

# Scene View screenshot: capture what the developer sees in the editor
manage_camera(action="screenshot", capture_source="scene_view", include_image=True, max_resolution=512)

# Scene View framed on a specific object
manage_camera(action="screenshot", capture_source="scene_view", view_target="Canvas", include_image=True, max_resolution=512)

# Agentic camera loop: point, shoot, analyze
manage_gameobject(action="look_at", target="MainCamera", look_at_target="Player")
manage_camera(action="screenshot", camera="MainCamera", include_image=True, max_resolution=512)
# → Analyze image, decide next action

# Multi-view screenshot (6-angle contact sheet)
manage_camera(action="screenshot", batch="surround", include_image=True, max_resolution=480)  # if screenshot_multiview exists on your package version it is equivalent; unknown-action error → use this batch="surround" form
```

**Best practices for AI scene understanding:**
- Use `include_image=True` when you need to *see* the scene, not just save a file.
- Use `batch="surround"` for a comprehensive overview (6 angles, one command).
- Use `view_target`/`view_position` to capture from a specific viewpoint without needing a scene camera.
- Use `capture_source="scene_view"` to see the editor viewport (gizmos, wireframes, grid).
- Keep `max_resolution` at 256–512 to balance quality vs. token cost.
- Verifying UI (Canvas): do NOT pass `camera=` — camera-rendered captures EXCLUDE Screen Space-Overlay canvases; use the default ScreenCapture path (no camera param). UI missing from a camera screenshot → re-shoot without `camera=` before concluding the UI is broken.

### 4. Always Check `editor_state` Before Complex Operations

```python
# Read mcpforunity://editor/state to check:
# - is_compiling: Wait if true
# - is_domain_reload_pending: Wait if true  
# - ready_for_tools: Only proceed if true
# - blocking_reasons: Why tools might fail
```

## Parameter Type Conventions

Common accepted shapes below. `manage_components.set_property` payload shapes can vary by component/property; if a template fails, inspect the component resource payload and re-send in the stored shape. Universal retry order for ANY rejected payload: (1) native type → (2) JSON-string form → (3) the exact shape stored in mcpforunity://scene/gameobject/{id}/component/{name} — read it, mirror it, retry ONCE; still rejected → report the raw error, do not loop. The Vectors/Booleans/Colors subsections below are instances of this rule.

### Vectors (position, rotation, scale, color)
```python
# Default: send the native list first:
position=[1.0, 2.0, 3.0]
# Rejected → re-send as JSON string: position="[1.0, 2.0, 3.0]"
```

### Booleans
```python
# Default: send the native boolean first:
include_inactive=True
# Rejected → re-send as string: include_inactive="true"
```

### Colors
```python
# Default: send normalized floats first (most component color properties store normalized):
color=[1.0, 0.0, 0.0, 1.0]
# Rejected → re-send 0-255 ints: color=[255, 0, 0, 255]
```

### Paths
```python
# Assets-relative (default):
path="Assets/Scripts/MyScript.cs"

# URI forms:
uri="mcpforunity://path/Assets/Scripts/MyScript.cs"
uri="file:///full/path/to/file.cs"
```

## Core Tool Categories

| Category | Key Tools | Use For |
|----------|-----------|---------|
| **Scene** | `manage_scene`, `find_gameobjects` | Scene operations, finding objects. Multi-scene editing (additive load, close, set active, move GO between scenes), scene templates (`3d_basic`, `2d_basic`, `empty`, `default`), scene validation with `auto_repair`. For build settings, use `manage_build(action="scenes")`. |
| **Objects** | `manage_gameobject`, `manage_components` | Creating/modifying GameObjects |
| **Scripts** | `create_script`, `script_apply_edits`, `validate_script` | C# code management (auto-refreshes on create/edit) |
| **Assets** | `manage_asset`, `manage_prefabs` | Asset operations. **Prefab instantiation** is done via `manage_gameobject(action="create", prefab_path="...")`, not `manage_prefabs`. |
| **Editor** | `manage_editor`, `execute_menu_item`, `read_console` | Editor control, package deployment (`deploy_package`/`restore_package`), undo/redo (`undo`/`redo` actions). Play-mode control: `manage_editor(action="play"/"pause"/"stop")` — read `editor_state.is_playing` first, verify the transition via editor/state; script edits while playing → see 🔴 STOP |
| **Testing** | `run_tests`, `get_test_job` | Unity Test Framework |
| **Batch** | `batch_execute` | Parallel/bulk operations |
| **Camera** | `manage_camera` | Cameras + Cinemachine (Tier 2 needs `com.unity.cinemachine` — `ping` to check); screenshots incl. batch/surround. Details: [tools-reference.md](references/tools-reference.md#camera-tools) |
| **Graphics** | `manage_graphics` | Volumes/post-processing, light baking, render stats, pipeline settings, URP features — `ping` to check pipeline. Details: [tools-reference.md](references/tools-reference.md#graphics-tools) |
| **Packages** | `manage_packages` | Install, remove, search, and manage Unity packages and scoped registries. Query actions: list installed, search registry, get info, ping, poll status. Mutating actions: add/remove packages, embed for editing, add/remove scoped registries, force resolve. Validates identifiers, warns on git URLs, checks dependents before removal (`force=true` to override). See [tools-reference.md](references/tools-reference.md#package-tools). |
| **ProBuilder** | `manage_probuilder` | 3D modeling, mesh editing, complex geometry. **`com.unity.probuilder` installed AND geometry needs face/edge editing, multi-material faces, or complex shapes → use `manage_probuilder`, NOT primitives; otherwise primitives.** Supports 12 shape types, face/edge/vertex editing, smoothing, and per-face materials. Examples: `manage_probuilder(action="create_shape", properties={"shape_type": "Stair", "size": [2,4,3], "steps": 6})`; `manage_probuilder(action="extrude_faces", target="Wall", properties={"faceIndices": [0], "distance": 0.5})`; verify: read_console(types=["error"]) + screenshot. |
| **Physics & Animation** | `manage_physics`, `manage_animation` | Physics settings/joints/raycasts/collision matrix, Animator/clips. Details: [physics](references/tools-reference.md#physics-tools); Action prefixes: manage_animation → animator_*/controller_*/clip_* (e.g. `manage_animation(action="clip_create", clip_path="Assets/Animations/Walk.anim")`). Unknown-action error → read the error's supported-action list, re-send |
| **VFX, Materials & Textures** | `manage_vfx`, `manage_material`, `manage_texture`, `manage_shader`, `manage_scriptable_object` | Particles/trails, material colors/shaders, procedural textures, ScriptableObject CRUD. Details: [material/shader + texture](references/tools-reference.md#material--shader-tools); Action prefixes: manage_vfx → particle_*/vfx_*/line_*/trail_* (e.g. `manage_vfx(action="particle_create", target="FX_Hit")`); manage_scriptable_object → create/modify. Unknown-action error → read the error's supported-action list, re-send. Example: `manage_material(action="set_material_color", target="Player", color=[1.0,0.0,0.0,1.0], mode="property_block")` |
| **Profiling & Code** | `manage_profiler`, `execute_code` | Profiler sessions/counters/memory snapshots; `execute_code` runs C# in-editor — use it ONLY when no dedicated tool covers the operation |
| **UI** | `manage_ui`, `batch_execute` with `manage_gameobject` + `manage_components` | **UI Toolkit**: Use `manage_ui` to create UXML/USS files, attach UIDocument, inspect visual trees. **uGUI (Canvas)**: Use `batch_execute` for Canvas, Panel, Button, Text, Slider, Toggle, Input Field. **Read `mcpforunity://project/info` first** to detect uGUI/TMP/Input System/UI Toolkit availability. (see [UI workflows](references/workflows.md#ui-creation-workflows)) |
| **Docs** | `unity_reflect`, `unity_docs` | API verification and documentation lookup. **`unity_reflect`** inspects live C# APIs via reflection (requires Unity connection): `search` types across assemblies, `get_type` for member summary, `get_member` for full signatures. **`unity_docs`** fetches official docs from docs.unity3d.com (no Unity connection needed): `get_doc` (ScriptReference), `get_manual` (Manual pages), `get_package_doc` (package docs), `lookup` (parallel search all sources + project assets). **Trust hierarchy: reflection > project assets > docs.** Workflow: `unity_reflect` search -> get_type -> get_member -> `unity_docs` lookup. See [tools-reference.md](references/tools-reference.md#docs-tools). |

## Common Workflows

Step 0 for EVERY task: apply the HARD GATE from Quick Start step 0 (read `mcpforunity://editor/state`; mutating → ready_for_tools==true; read-only → note staleness, resource reads only).

### Creating a New Script and Using It

```python
# 1. Create the script (automatically triggers import + compilation)
create_script(
    path="Assets/Scripts/PlayerController.cs",
    contents="using UnityEngine;\n\npublic class PlayerController : MonoBehaviour\n{\n    void Update() { }\n}"
)

# 2. Poll mcpforunity://editor/state — wait recommended_retry_after_ms if present, else 1000ms; max 60 iterations — until is_compiling==false AND ready_for_tools==true
# 3. read_console(types=["error"], count=10, include_stacktrace=True) — must return zero errors before step 4 (full loop details: Critical Best Practices → "1. After Writing/Editing Scripts")

# 4. Only then attach to GameObject
manage_gameobject(action="modify", target="Player", components_to_add=["PlayerController"])
```

### Finding and Modifying GameObjects

```python
# 1. Find by name/tag/component (returns IDs only)
result = find_gameobjects(search_term="Enemy", search_method="by_tag", page_size=50)

# 2. Get full data via resource
# mcpforunity://scene/gameobject/{instance_id}

# 3. Modify using the ID
manage_gameobject(action="modify", target=instance_id, position=[10, 0, 0])
```

### Editing a Prefab Asset (not a scene instance)

🔴 CHECKPOINT: `modify_contents` writes the .prefab to disk IMMEDIATELY (no prefab-stage cancel) — list the intended changes and get user confirmation first (see STOP · CHECKPOINT).

Two paths - pick one:

```python
# A) Headless (DEFAULT for scripted edits - writes the .prefab to disk immediately; use B ONLY to inspect live component values):
manage_prefabs(action="get_hierarchy", prefab_path="Assets/Prefabs/My.prefab")  # resolve child paths first
manage_prefabs(action="modify_contents", prefab_path="Assets/Prefabs/My.prefab",
    components_to_add=["MyScript"],
    component_properties={"MyScript": {
        "targetButton": {"path": "Header/BtnOk"},     # child by relative path
        "bgSprite":     {"guid": "abc123..."}         # asset by GUID; {"path": "Assets/..."} also works
    }})
# Field names and child paths are ALWAYS resolved from get_hierarchy output —
# never copied from examples or assumed from the request.

# B) Interactive (prefab stage - when you must inspect live components):
manage_prefabs(action="open_prefab_stage", prefab_path="Assets/Prefabs/My.prefab")
# ...edit objects inside the stage with manage_gameobject / manage_components...
# set_property object references inside the stage use the same shapes as modify_contents:
# instance ID (int), "Assets/..." path, or {"guid": ...}; for child GameObjects, resolve the
# child instance ID first via find_gameobjects(search_method="by_path") within the stage.
# 🔴 save_prefab_stage writes the .prefab to disk — same checkpoint as modify_contents:
# list changes and get user confirmation BEFORE calling it (close_prefab_stage without saving = the cancel path)
manage_prefabs(action="save_prefab_stage")
manage_prefabs(action="close_prefab_stage")

# Verify either path:
manage_prefabs(action="get_hierarchy", prefab_path="Assets/Prefabs/My.prefab")
read_console(types=["error"], count=10)
```

Prefab stage actions belong to `manage_prefabs` - NOT `manage_editor`.

### Plan-Only Mode

Output format (mandatory SHAPE — the lines below are an EXAMPLE; substitute the matching workflow's tool calls and verify steps; every domain uses the same shape: numbered calls with full params, 🔴 on gated calls, that workflow's verify calls last, end with the confirmation question):
```
Plan:
1. find_gameobjects(search_term="Player", search_method="by_name")   # resolve ID
2. 🔴 manage_gameobject(action="delete", target=<id>)   # 🔴 IF target has children or 3+ deletes (STOP list) — resolve child count via gameobject resource in step 1; unresolved → assume gated
Verify:
N. read_console(types=["error"], count=10)
Chưa chạy gì — xác nhận để thực thi?
```
The closing not-yet-run statement + confirmation question are mandatory and rendered in the USER'S language — the Vietnamese line above is an example rendering (same rule as the checkpoint language rule).
Second example, different gated domain: `1. manage_asset(action="get_info", path="Assets/Materials/Hero.mat")` → `2. 🔴 manage_material(action="set_material_color", material_path="Assets/Materials/Hero.mat", color=[1.0,0.0,0.0,1.0])` → `Verify: read_console(types=["error"], count=10)`. Never reuse example paths/fields — every param comes from THIS task's read-only resolution calls.
When the request forbids execution or asks only for a plan, in any language/phrasing (plan only, dry-run, do-not-run, đừng chạy, chỉ lập kế hoạch): (1) resolve unknowns with read-only calls only (find_gameobjects, get_hierarchy, resources — open/save/close_prefab_stage count as EXECUTION, not inspection: resolve prefab internals via manage_prefabs(action="get_hierarchy") only); (2) output a numbered list of intended tool calls with full params; (3) mark every call that would trip a 🔴 checkpoint; (4) the plan's final numbered items MUST be the matching workflow's verify calls (e.g. get_hierarchy + read_console after prefab writes) — a plan without verify steps is incomplete; (5) execute NOTHING — end by asking for confirmation.

### Read-Only Asset Audit (missing references / GUIDs)

```python
# 0. Read mcpforunity://editor/state (read-only tasks included — Step 0).
# 1. Locate targets (no mutation):
manage_asset(action="search", path="Assets", search_pattern=<the user's asset name pattern>, filter_type="Prefab", page_size=50, generate_preview=False)
# 2. Extract referenced GUIDs from each asset:
find_in_file(uri="Assets/Path/My.prefab", pattern="guid: [0-9a-f]{32}")
# Asset-type-specific audit → narrow to the serialized field (bare guid matches ALL reference types and over-reports):
# sprites: pattern="m_Sprite: \\{fileID: \\d+, guid: [0-9a-f]{32}"; textures: pattern="m_Texture: .*guid: [0-9a-f]{32}"
# 3. Resolve each GUID via manage_asset(action="get_info"/"search") - unresolvable GUID -> report as missing.
# Rule: audit tasks use ONLY read-only actions (search, get_info, find_in_file, resources).
# Same recipe for any missing-reference audit — scenes (*.unity) and ScriptableObjects (*.asset): swap filter_type and the find_in_file target.
# Report honestly even when 0 issues found; never "repair" during an audit.
```

### Profiling a Scene

```python
manage_profiler(action="profiler_start")
# run the scenario: manage_editor(action="play") → confirm editor_state.is_playing → drive it 5-10s → capture below → manage_editor(action="stop"); the play-mode script-edit gate applies while profiling
manage_profiler(action="get_frame_timing"); manage_profiler(action="get_counters", category="Render")
manage_profiler(action="profiler_stop")    # details: tools-reference.md#profiler-tools
```

### Running and Monitoring Tests

```python
# 1. Start test run (async)
result = run_tests(mode="EditMode", test_names=["MyTests.TestSomething"])
job_id = result["job_id"]

# 2. Poll for completion
result = get_test_job(job_id=job_id, wait_timeout=60, include_failed_tests=True)
# mode="PlayMode" → always pass init_timeout=120000 on run_tests (domain reload delays init; the 15000 default times out)
```

## Pagination Pattern

Large queries return paginated results. Always follow `next_cursor`:

```python
cursor = 0
all_items = []
page_count = 0
while True:
    if page_count > 40 or len(all_items) > 2000: break  # runaway cursor — report partial + last cursor
    result = manage_scene(action="get_hierarchy", page_size=50, cursor=cursor)
    all_items.extend(result.get("data", {}).get("items") or result.get("ids") or [])
    nc = result.get("next_cursor") or result.get("data", {}).get("next_cursor")
    if not nc or nc == cursor:  # no cursor, or server echoed the same one → stop, report partial
        break
    cursor = nc
    page_count += 1
```

Field names vary by tool: `find_gameobjects` returns ids/cursor at the top level; `get_hierarchy` nests under `data.items`. Follow `next_cursor` wherever it appears in the response, not a fixed path.

## Multi-Instance Workflow

When multiple Unity Editors are running:

```python
# 1. List instances via resource: mcpforunity://instances
# 2. Set active instance
set_active_instance(instance="MyProject@abc123")
# 3. All subsequent calls route to that instance
```

## Error Recovery

| Symptom | Fix | Still failing → fallback |
|---------|-----|--------------------------|
| Tools return "busy" | Compilation in progress — wait, check `editor_state` | Poll until `ready_for_tools`, inspect `blocking_reasons` |
| "stale_file" error | File changed since SHA — re-fetch with `get_sha`, retry | Re-read the file and re-apply the edit |
| Connection lost | Domain reload — wait 5s → read `mcpforunity://instances` → `set_active_instance` → retry the failed call once | Still failing after that one retry → verify the Editor is running, report to the user; do not loop |
| Commands fail silently | Wrong instance — check `set_active_instance` | List `mcpforunity://instances` and pin the right one |
| `batch_execute` fails mid-batch | Read per-command results, retry only the failed commands | Use `fail_fast=True` for dependent chains |
| `get_test_job` exceeds `wait_timeout` | Poll again with a longer `wait_timeout` | Still hanging → check `editor_state` for a dialog/compile block |
| Screenshot returns `pending=true` (Play Mode) | Call it a second time to retrieve the saved PNG | Still pending → re-run without `include_image` and Read the saved PNG from Assets/Screenshots/; still empty → `capture_source="scene_view"` instead |
| `validate_script` reports errors after an edit | Fix per the diagnostics | Badly broken → restore via `manage_editor` undo |
| Console shows CS errors after create_script/script_apply_edits | Fix errors in console order — FIRST error first (later CS errors are usually cascades of it); parse file:line, fix via script_apply_edits, re-poll compile | 2 failed fix attempts → stop, show the errors to the user; offer undo or delete_script (🔴 checkpoint applies) |
| `find_gameobjects` returns empty | Retry with include_inactive=True, then search_method="by_path" with the full path | Locate via manage_scene(action="get_hierarchy", parent=<nearest known parent>) |
| Screenshot returns black/empty image | Camera renders nothing — verify a camera exists via find_gameobjects(search_method="by_component", search_term="Camera"), try `capture_source="scene_view"` | Fall back to `batch="surround"` without a camera param |
| `editor/state` unreadable at Step 0 | MCP server up but no editor connected — read `mcpforunity://instances` | Empty list → report to user: open the Unity Editor with the MCP bridge; do NOT proceed |
| modify_contents rejects component_properties (unknown field/child path) | Re-run manage_prefabs(action="get_hierarchy") and copy field names/child paths exactly from its output; re-send | Still rejected → open_prefab_stage + read the component resource for the live payload shape, mirror it (🔴 save gate still applies) |
| add_package/remove_package fails or hangs | Poll manage_packages(action="status", job_id=...), read_console for resolver errors | Report the exact package error to the user; do NOT retry-loop (each attempt can trigger a domain reload) |
| Tool call fails with unknown/hidden tool | Tool group deactivated — manage_tools(action="list_groups"), then manage_tools(action="activate", group=<group>) and retry | Toggled in the Unity MCP window instead → manage_tools(action="sync"); still absent → report to user, do NOT substitute execute_code |
| manage_ui render_ui returns pending=true or blank | Play Mode: call render_ui a second time to retrieve the saved PNG (hasContent==true) | Editor-mode render is best-effort → fall back to manage_camera(action="screenshot", include_image=True, max_resolution=512) of the Canvas |

## ❌ DON'T (common MCP mistakes)

- Call tools while `is_compiling`/`is_domain_reload_pending` is true → poll `mcpforunity://editor/state` until `ready_for_tools`
- Skip the compile-wait + console check after script edits, or call `refresh_unity` after them (redundant — see Critical Best Practices §1)
- Use `manage_gameobject` to search (use `find_gameobjects`) or `manage_prefabs` to instantiate (use `manage_gameobject(action="create", prefab_path=...)`)
- Fire 3+ same-shape calls sequentially when one `batch_execute` does it (10-100x slower) — applies to read-only lookups too (find_gameobjects, manage_asset search)
- Treat a batch_execute top-level success as per-command success — ALWAYS read each command's result; partial failures are silent otherwise
- Leave a prefab stage open after the task — close_prefab_stage (or discard) before moving on
- Dump full `get_hierarchy` when you need one object — `find_gameobjects` + the gameobject resource is cheaper
- Read/dump whole serialized YAML (.prefab/.unity/.asset) during audits — files can be multi-MB; extract with find_in_file(pattern=...) only, never a full-file Read
- Request screenshots without `max_resolution` (256–512 is enough for scene understanding; full-res burns tokens)
- Edit scripts while the Editor is in Play Mode — recompile kills play state; check `editor_state` first
- Delete + recreate a script to rewrite it — regenerates the .meta GUID and breaks every scene/prefab reference; rewrite in place with `apply_text_edits` full-range replacement or `script_apply_edits` replace_method

## 🔴 STOP · CHECKPOINT (destructive operations)

Gated operations — one bullet each:
- `delete_script`, `manage_asset(action="delete")`, and every other tool's file-delete/overwrite action — `manage_ui(action="delete")` (.uxml/.uss), `manage_shader(action="delete")`, `manage_texture(action="delete")` — same gate
- `manage_scene(action="close_scene")` with unsaved changes — silently discards edits (read mcpforunity://editor/state → active-scene dirty flag; field absent/unreadable → treat the scene as dirty, gate applies)
- `manage_prefabs(action="modify_contents")` — writes the .prefab to disk immediately, no prefab-stage cancel
- `manage_prefabs(action="save_prefab_stage")` — writes the prefab to disk, same gate as modify_contents
- `manage_editor(action="deploy_package")` — no confirmation dialog
- `manage_scene(action="create"/"load")` — replaces the open scene; unsaved changes silently discarded (save first)
- `manage_packages(action="add_package"/"remove_package")` — domain reload clears the undo stack
- `execute_code` — arbitrary C#, can mutate anything; safety_checks is not a sandbox
- `execute_menu_item` on mutating menu paths (Assets/Delete, File/Save…)
- `manage_asset(action="move"/"rename")` on assets referenced by scenes/prefabs — decide referenced-ness first: manage_asset(action="get_info") for the GUID, then find_in_file over *.unity/*.prefab for that GUID; any hit → gate applies, zero hits → proceed without confirmation
- `manage_gameobject(action="delete")` — decide children-ness FIRST: read mcpforunity://scene/gameobject/{id} (children/child_count); resource unreadable → treat as having children (gate applies). Gate khi: có children, bị reference, hoặc 3+ objects. Exemption: objects created EARLIER IN THIS SAME TASK may be deleted ungated.
- `manage_scene(action="save")` targeting a scene file other than the one currently loaded — silent overwrite of the target file

For ANY of the above, send this message and WAIT: "Sẽ thay đổi: [exact files/objects]. Xác nhận? (chưa chạy gì)" (in the user's language — the exact-files list and wait-for-yes semantics are mandatory regardless of language) — call the tool only after the user answers yes. Undo exists (`manage_editor` undo) but domain reload can clear the undo stack - do not rely on it.

Also STOP before ANY mutating call when the user asked for read-only, audit, or plan-only work - present the plan and wait for explicit confirmation.

Also STOP if `editor_state.is_playing == true` and the task edits scripts — recompile kills play state; confirm with the user first.

## Reference Files

For detailed schemas and examples:

- **[tools-reference.md](references/tools-reference.md)**: Complete tool documentation with all parameters
- **[workflows.md](references/workflows.md)**: Extended workflow examples and patterns
