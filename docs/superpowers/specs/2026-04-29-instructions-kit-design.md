# Kerbal Instructions Kit (KIK) — Design

**Date:** 2026-04-29
**Status:** Draft — pending implementation plan
**Repo (planned):** `GameData/KerbalInstructionsKit/`

## Motivation

The existing sibling mods cover narrative dialogue (KDK / ChatterBox), campaign chapter
state (KCK), and KSC character flavor (KAK). None of them surface **concrete instructional
content** — the "click on this button," "go to the SPH," "drag the prograde handle outward"
guidance a new player needs to actually operate the game.

KIK fills that gap. It is a content-driven, plugin-only framework for showing instructional
panels — text plus images, multi-page, with cross-links to other lessons, KSPedia entries,
and external URLs. Pack authors define lessons in cfg; KIK loads them, gates them by per-save
unlock state, and renders them in a non-modal IMGUI panel accessible from the toolbar in
every scene plus a contextual button injected into the contract details view.

The immediate audience is a 12-year-old learning KSP. The longer-term audience is any campaign
mod that wants to teach the player how to operate the game.

## Scope

**In scope (v0.1):**

- ConfigNode loaders for `INSTRUCTION_LESSON`, `LESSON_TRIGGER`, and a CC `BEHAVIOUR` of
  type `AttachLesson`.
- Per-save unlock state via `ScenarioModule`.
- Trigger engine wired to KSP `GameEvents` and (optionally) ContractConfigurator events.
- IMGUI panel with stacked layout (lesson view + footer "All Lessons" → fullscreen archive
  with search, tag-chip filters, categorized grouping).
- Toolbar button registered via `ToolbarController` in KSC, Editor, Flight, MapView,
  TrackingStation.
- Harmony patch injecting a "📖 Instructions" button into the contract details view.
- KSPedia and external URL link buttons on lesson pages.
- xUnit test suite for pure-logic layers.

**Out of scope (v0.1):**

- Live on-screen pointers / arrows highlighting KSP UI elements.
- Per-element rich page composition (the v0.1 page model is text + image + caption + links;
  not an ordered list of typed elements).
- Tracking "lessons read" / completion percent.
- Resume-mid-lesson restoration when opened from a contract-window button (toolbar reopens
  to last viewed; contract-window 📖 always starts at page 1).
- Localization of structural strings (button labels, search placeholder, etc.) — strings
  are English-only in the plugin; lesson content can use Localizer keys.

## Architecture

KIK is a standalone .NET Framework 4.7.2 plugin DLL. Pattern matches KDK / KCK / KAK:
its own git repo, soft dependencies on adjacent mods, inert without cfg content.

### Dependencies

| Dep | Tier | Notes |
|---|---|---|
| `ToolbarController` | Hard | toolbar registration |
| `ClickThroughBlocker` (`ClickThruBlocker`) | Hard | IMGUI window |
| `Harmony` (`HarmonyLib`) | Hard | contract-window button injection |
| `ContractConfigurator` | **Soft** | runtime-detected via `AssemblyLoader`; AttachLesson BEHAVIOUR is reflection-registered if CC is present |
| KDK / KCK / ChatterBox | None | KIK does not depend on any of them |

### Module layout

```
KerbalInstructionsKit/
├── src/KerbalInstructionsKit/
│   ├── Core/             InstructionsKit static API, LessonState, LessonRegistry
│   ├── Config/           Lesson + Page + Link loaders, trigger compilers
│   ├── Runtime/          Per-scene KSPAddons, ScenarioModule, contract-window patch
│   ├── Triggers/         LessonTriggerEngine, GameEvent / Contract / Flag triggers
│   ├── Rendering/        LessonPanel (IMGUI), archive view, page renderer
│   └── Util/             ISceneNode, IPauseController, IExpressionContext abstractions
├── tests/KerbalInstructionsKit.Tests/  xUnit
├── Plugins/              built DLL output + icons
└── docs/
```

### Variation points (resolved)

- **CC dependency: soft.** Reflection-registered AttachLesson when CC is present;
  silent fallback to `LESSON_TRIGGER` otherwise.
- **Contract→lesson trigger: BEHAVIOUR + standalone trigger.** `AttachLesson` BEHAVIOUR
  for CC users is the ergonomic path; `LESSON_TRIGGER { onContract = ...; state = ... }`
  is the standalone fallback and supports non-contract `GameEvents`.
- **Per-scene addons.** One `KSPAddon` per scene (KSC, Editor, Flight, TrackingStation).
  Long-term state in the scenario; transient panel state per-addon.

## Cfg Authoring Grammar

Three top-level node types.

### `INSTRUCTION_LESSON`

```cfg
INSTRUCTION_LESSON
{
    id = LSN_ManeuverNodes              // unique global ID; convention <PACK>_LSN_<Name>
    title = How Maneuver Nodes Work
    category = Flight Basics            // optional; lessons with no category go to "Misc"
    sortOrder = 200                     // optional; lower = earlier within category, default 1000
    tags = Maneuver, Orbits, Beginner   // optional; drives filter chips in archive
    visibleIf = !LSN_ManeuverNodesV2.unlocked   // optional KIK expression (grammar below); default true

    PAGE
    {
        title = Setting up a node       // optional per-page subtitle
        text = To plan a burn, you need a <b>maneuver node</b>.\nClick the gauge above the navball.
        image = BadgKatCareer/Lessons/Images/maneuver-gauge.png   // GameDatabase URL, optional
        caption = The maneuver gauge sits at the top of the navball.   // optional

        LINK { type = lesson;  target = LSN_DeltaV;       label = What is delta-v? }
        LINK { type = kspedia; target = ManeuverNodes;    label = KSPedia: Maneuver Nodes }
        LINK { type = url;     target = https://wiki.kerbalspaceprogram.com/wiki/Maneuver_node; label = KSP Wiki }
    }

    PAGE { ... }
    PAGE { ... }
}
```

- Text is plain string with Unity rich-text tags supported (`<b>`, `<i>`, `<color=#xxx>`,
  `<size=N>`). Newlines via `\n`.
- `title`, `text`, `caption`, and link `label` accept Localizer `#autoLOC_xxxxx` keys.
- Lesson IDs are global; duplicate IDs log a warning and the second loses.

#### KIK expression grammar (`visibleIf`)

KIK ships its own minimal expression evaluator (does not depend on CC's expression engine).

```
expr        := orExpr
orExpr      := andExpr ('||' andExpr)*
andExpr     := unary ('&&' unary)*
unary       := '!' unary | atom
atom        := '(' expr ')' | lessonRef | flagRef | bool
lessonRef   := IDENT '.unlocked'    // IDENT must match a known lesson ID
flagRef     := 'flag(' IDENT ')'
bool        := 'true' | 'false'
IDENT       := [A-Za-z_][A-Za-z0-9_]*
```

Examples:
- `LSN_ManeuverNodes.unlocked`
- `!LSN_ManeuverNodesV2.unlocked`
- `LSN_RocketBasics.unlocked && !flag(SkipRocketLessons)`
- `flag(BeginnerMode) || LSN_AdvancedToggle.unlocked`

Evaluator is implemented behind `IExpressionContext`, which is mocked in unit tests.

### `LESSON_TRIGGER`

A standalone unlock trigger for environments without CC, or for unlock conditions tied to
KSP `GameEvents` rather than contract state.

```cfg
LESSON_TRIGGER
{
    lesson = LSN_RocketBasics

    onGameStart = true
    // OR
    onGameEvent = onLaunch
    // OR
    onContract = BKEX_FirstScience
    state = OFFERED
    // OR
    onFlag = ManeuverNodeTutorialReady
}
```

Exactly one of `onGameStart` / `onGameEvent` / `onContract` / `onFlag` per block. Multiple
blocks per lesson are allowed (any one fires → unlocked).

### CC `BEHAVIOUR` — `AttachLesson`

```cfg
CONTRACT_TYPE
{
    name = BKEX_FirstScience
    // ...

    BEHAVIOUR
    {
        name = AttachLesson_FirstScience
        type = AttachLesson
        lesson = LSN_GettingScience
        unlockOn = OFFERED              // OFFERED | ACCEPTED | COMPLETED; default OFFERED
        showButton = true               // default true; injects 📖 button on contract details
    }
}
```

When `showButton = true`, KIK's Harmony patch on the contract details view injects a
"📖 Instructions" button next to the contract title. Clicking it opens the panel directly
to the linked lesson (page 1).

## Public API

```csharp
namespace KerbalInstructionsKit.Core;

public static class InstructionsKit
{
    public static LessonRegistry Lessons { get; }
    public static LessonState State { get; }

    public static void OpenLesson(string lessonId);
    public static void OpenArchive();
    public static void Close();

    public static void SetFlag(string name, bool value);   // drives flag(...) in visibleIf
    public static bool GetFlag(string name);

    public static event Action<string> LessonUnlocked;
}
```

Used by the AttachLesson BEHAVIOUR, the contract-window Harmony patch, and any future
campaign-pack code that wants to surface lessons programmatically.

## Runtime

### ScenarioModule — `KerbalInstructionsKitScenario`

Per-save persistence, attached to all game modes. Save format:

```cfg
SCENARIO
{
    name = KerbalInstructionsKitScenario
    UNLOCKED
    {
        LSN_RocketBasics = true
        LSN_GettingScience = true
    }
    FLAGS
    {
        ManeuverTutorialDone = true
    }
    LAST_VIEWED = LSN_GettingScience
    LAST_VIEWED_PAGE = 2
}
```

`LAST_VIEWED` and `LAST_VIEWED_PAGE` capture the most recent lesson position so reopening
the toolbar mid-session returns the player to where they were. UNLOCKED entries that don't
match any loaded lesson are preserved silently — they may resolve later if a different mod
loads the matching cfg.

### Per-scene addons

| Scene flag | Addon | Toolbar visible in |
|---|---|---|
| `KSPAddon.Startup.SpaceCentre` | `KikKscAddon` | KSC, MissionControl |
| `KSPAddon.Startup.EditorAny` | `KikEditorAddon` | VAB, SPH |
| `KSPAddon.Startup.Flight` | `KikFlightAddon` | Flight, MapView |
| `KSPAddon.Startup.TrackingStation` | `KikTrackingStationAddon` | TrackingStation |

Each addon:
1. Registers a toolbar button via `ToolbarController` (stock + Blizzy).
2. Hosts the `LessonPanel` IMGUI window using `ClickThruBlocker.GUILayoutWindow`.
3. Reads `LessonState` from the scenario; writes back on close.
4. Unregisters toolbar on `OnDestroy`.

The panel is a single `LessonPanel` instance per addon. Transient state (current lesson,
page, view mode, back stack) lives on the addon, not the scenario.

### Trigger engine

A single `LessonTriggerEngine` instantiated by the scenario. On startup it:
1. Compiles every `LESSON_TRIGGER` block into a typed condition.
2. Registers `AttachLesson` BEHAVIOUR triggers (one per BEHAVIOUR-bearing contract).
3. Subscribes to `GameEvents.Contracts.OnOffered`, `OnAccepted`, `OnCompleted`, `OnFailed`,
   plus any specific `GameEvents` named in `onGameEvent` blocks.
4. Fires `onGameStart` triggers once when first attached to a fresh save.

On each event, it iterates registered triggers, sets `UNLOCKED` for matched lessons, and
fires `InstructionsKit.LessonUnlocked`. The engine is idempotent — already-unlocked lessons
ignore subsequent matches.

Internal flag bus: `InstructionsKit.SetFlag(name, true/false)` fires triggers tied to
`onFlag = name` and is read by `flag(...)` in `visibleIf` expressions. Flag values persist
in the scenario's `FLAGS` block. KIK ships no built-in flag setters; downstream code
(future KCK integration, campaign-pack glue, etc.) drives the bus.

### Contract-window button injection

A Harmony postfix on the contract details panel renderer (specific class verified during
implementation; targets `MissionControl` UI in the SpaceCentre scene). The patch:
1. Reads the currently selected contract.
2. Looks up any `AttachLesson` BEHAVIOUR with `showButton = true`.
3. If the linked lesson is unlocked, draws a "📖 Instructions" button next to the title.
4. On click, calls `InstructionsKit.OpenLesson(id)`.

If the patch fails to apply (KSP version drift, UI restructure), log error and skip — toolbar
entry remains functional.

### CC dependency probe

At plugin load, KIK iterates `AssemblyLoader.loadedAssemblies`. If `ContractConfigurator`
is present, KIK locates its `BEHAVIOUR` factory registry via reflection and registers
`AttachLesson` as a behaviour type. If absent, KIK logs once at INFO level:

> [KIK] ContractConfigurator not present; AttachLesson BEHAVIOUR disabled. Use LESSON_TRIGGER for unlock control.

Standalone `LESSON_TRIGGER` blocks load and fire either way.

## Rendering & UX

### Window

Single `ClickThruBlocker.GUILayoutWindow` per scene addon. Default ~900×640px, draggable,
position persisted per-scene via `PluginConfiguration`. Non-modal (game keeps running).

### Lesson view

```
┌────────────────────────────────────────────────────────────────┐
│ How Maneuver Nodes Work               ⏸ Pause   ✕ Close        │
│ ── Setting up a node ─────────────────────────────────────────│
│                                                                │
│   [          maneuver-gauge.png (centered, max 600px wide)   ] │
│   The maneuver gauge sits at the top of the navball.           │
│                                                                │
│   To plan a burn, you need a maneuver node.                    │
│   Click the gauge above the navball.                           │
│                                                                │
│   [ → What is delta-v? ]  [ ↗ KSPedia: Maneuver Nodes ]        │
│                                                                │
│ ── ◀ Prev ─────── Page 2 of 4 ─────── Next ▶ ─────────────────│
│ ──── 📚 All Lessons ────────────────────────────────────────── │
└────────────────────────────────────────────────────────────────┘
```

- Pause button hidden outside Flight (no-op there).
- LINK buttons render below the body, in cfg order.
- Internal lesson links push the previous lesson onto a back stack (a "← Back" link
  appears at top-left when the back stack is non-empty).
- Prev/Next disabled (greyed) on first/last page.

### Archive view

```
┌────────────────────────────────────────────────────────────────┐
│ All Lessons                                       ⏸ Pause   ✕  │
│ 🔎 [ search ─────────────────────] (filters title, category, tags) │
│ [ Beginner ] [ Orbits ] [ Planes ] [ Surface ] [ Spaceplanes ]│
│                                                                │
│ ▾ Flight Basics                                                │
│   • Reading the Navball                                        │
│   • Throttle & Stage                                           │
│   • Maneuver Nodes              ◀ last viewed                  │
│ ▾ Building Rockets                                             │
│   • Your First Stage                                           │
│   • Fuel & Engines                                             │
│ ▾ Misc                                                         │
│                                                                │
│ ──── ◀ Back to Lesson ─────────────────────────────────────── │
└────────────────────────────────────────────────────────────────┘
```

- Tag chips toggle filters; multiple chips combined as OR.
- Search is plain substring match across title / category / tags, case-insensitive.
- Hidden lessons (locked or `visibleIf = false`) don't appear at all.
- Categories with all lessons hidden collapse out of view.
- Empty archive shows: "No lessons unlocked yet — keep playing!"
- "Back to Lesson" hidden when no lesson is currently loaded.

### Open behaviors

| Trigger | Opens to |
|---|---|
| Toolbar click | Last-viewed lesson (`LAST_VIEWED` set), else archive view |
| Contract-window 📖 button | Linked lesson, page 1 |
| `InstructionsKit.OpenLesson(id)` | That lesson, page 1 |
| `InstructionsKit.OpenArchive()` | Archive view |
| Internal LINK to lesson | That lesson, page 1; previous pushed to back stack |

### Pause behavior

In Flight:
- ⏸ button calls `FlightDriver.SetPause(true)`. Label changes to "▶ Resume".
- On opening the panel, KIK records the value of `FlightDriver.Pause` as `wasPausedOnOpen`.
- KIK also tracks `kikPausedFlag` — set true when the user clicks ⏸ inside the panel,
  reset to false when they click ▶ Resume.
- Closing the panel:
  - If `kikPausedFlag` is true, KIK calls `FlightDriver.SetPause(wasPausedOnOpen)` — i.e.,
    restore the game to its pre-open pause state.
  - If `kikPausedFlag` is false (user never paused via KIK, or paused then resumed), close
    leaves `FlightDriver.Pause` as-is.
- Pause state itself is not persisted across panel reopens.

Outside Flight: button hidden.

### Toolbar icons

`KIK_icon.png` (38×38, stock) and `KIK_icon_blizzy.png` (24×24) ship in
`GameData/KerbalInstructionsKit/Plugins/Icons/`.

## Error Handling

KIK is content-driven; most errors are malformed cfg or content drift. Philosophy: **log
loudly, never crash, fail soft.** All logs prefixed `[KIK]`.

### Cfg load

| Problem | Behavior |
|---|---|
| Lesson with no `id` | Skip lesson, log error with file path. |
| Duplicate `id` | First wins, second logged as warning. |
| Lesson with no `PAGE` blocks | Skip lesson, log error. |
| Page with no `text` AND no `image` | Skip page, log warning. |
| `image` URL not in `GameDatabase` | Page renders without image; image space collapses; log warning. |
| `LINK { type = lesson; target = LSN_NotReal }` | Render disabled with "Lesson not found" tooltip; log warning at first render. |
| `LINK` with missing `target` | Skip link, log warning. |
| `LINK` with unknown `type` (not `lesson` / `kspedia` / `url`) | Skip link, log warning. |
| `LINK { type = url; target = ... }` not http(s) | Skip link, log warning. |
| `visibleIf` parse error | Treated as `true`, log warning once at load. |
| `visibleIf` runtime error | Treated as `false` for that frame, log once-per-lesson. |
| `LESSON_TRIGGER` referencing unknown lesson | Skip trigger, log warning. |
| `LESSON_TRIGGER onGameEvent` with unknown event name | Skip trigger, log warning. |

### Runtime

| Problem | Behavior |
|---|---|
| Scenario `UNLOCKED` IDs that don't match any lesson | Preserve silently (other mods may load matching cfg later). |
| ContractConfigurator absent | One-time INFO log; AttachLesson BEHAVIOUR disabled; standalone triggers continue. |
| ToolbarController missing despite hard dep | Log error, skip toolbar registration; panel still openable via API. |
| Harmony patch on contract details fails | Log error, skip injection; toolbar still works. |
| `KSPediaSpawner.SpawnKSPedia()` throws / page not found | Log warning, no UI feedback. |
| `Application.OpenURL()` blocked | Caught, logged, no UI feedback. |

### What we don't validate

- `visibleIf` expressions at load time — CC's expression engine doesn't expose pre-validation;
  errors surface lazily on first evaluation.
- KSPedia link targets — no API to enumerate KSPedia pages.
- `onContract` triggers referencing a real CC contract name — could resolve later via
  add-pack ordering.

## Testing

### `tests/KerbalInstructionsKit.Tests/` — xUnit

Pure-logic coverage:

- **Cfg parsing.** `LessonLoader` round-trips: all fields parsed correctly; malformed
  nodes produce expected warnings/skips; duplicate IDs handled.
- **Page parsing.** Text + image + caption + LINK blocks; missing fields handled cleanly.
- **Lesson registry.** Lookup, listing, hidden-lesson filtering by `visibleIf`.
- **Trigger compilation.** `LESSON_TRIGGER` blocks → `ITriggerCondition` instances;
  matching against synthetic events; `onGameStart` once-only semantics.
- **Visibility expression.** `visibleIf` against a fake `IExpressionContext`; happy paths
  and parse-error fallback to `true`.
- **State serialization.** `LessonState.Save` / `Load` round-trip through `ConfigNode`.
- **Search filter.** Substring match, case-insensitive, across title/category/tags.
- **Tag filter.** Multi-tag OR semantics.
- **Back stack.** Push/pop correctness when navigating internal links.

Pure-logic abstractions: `ISceneNode`, `IPauseController`, `IExpressionContext` mirror the
KAK pattern so the panel state machine and trigger engine are testable without Unity loaded.

### Manual KSP testing

Required for verification of integrations the test harness can't cover:

- Toolbar button appears in each registered scene.
- IMGUI window renders, drags, persists position across sessions.
- Contract-window 📖 button injection works on a real CC contract.
- KSPedia link opens KSPedia.
- URL link opens the system browser.
- Pause toggle behaves correctly; close-after-pause restores time.
- Image loading from `GameDatabase` URL renders correctly.
- Rich-text tags render (bold, italic, color, size).
- Localizer key resolution.

A small `KIK_TestPack/` cfg pack with 3-4 lesson stubs lives in the repo at
`dev/test-pack/KIK_TestPack/`. Files use `.cfg.example` extensions so KSP's recursive
GameData scan doesn't pick them up. Devs rename to `.cfg` and copy the folder out to
`GameData/KIK_TestPack/` for in-game smoke-testing; release artifacts and CKAN don't
include the dev directory.

## File Locations

- Plugin DLL output: `GameData/KerbalInstructionsKit/Plugins/KerbalInstructionsKit.dll`
- Toolbar icons: `GameData/KerbalInstructionsKit/Plugins/Icons/`
- Test pack (dev-only, not shipped): `GameData/KerbalInstructionsKit/dev/test-pack/KIK_TestPack/`
- Lesson images (in BadgKatCareer): `GameData/BadgKatCareer/Lessons/Images/`
- Lesson cfgs (in BadgKatCareer): `GameData/BadgKatCareer/Lessons/*.cfg`

## Dependency Chain Update

After KIK ships, the dependency chain becomes:

```
BadgKatCareer
  ├─► KerbalAdminKit ─► KerbalCampaignKit ─► KerbalDialogueKit ─► ChatterBox
  └─► KerbalInstructionsKit ─► (ToolbarController, ClickThruBlocker, Harmony)
                              soft: ContractConfigurator
```

KIK is a **leaf** plugin — it has no upstream KSP-mod dependencies beyond the dependency
utilities, and nothing in the existing chain depends on it. BadgKatCareer becomes the only
mod that requires both branches.
