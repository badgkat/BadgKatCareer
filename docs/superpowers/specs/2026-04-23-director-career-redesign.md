# BadgKatDirector + BadgKatCareer — Admin Building Redesign

Replace the stock Administration Building with a character-driven program management layer. This spec covers two mods: a generic UI + convention framework (**BadgKatDirector**) and the specific campaign content that uses it (**BadgKatCareer**).

**Supersedes:** [`2026-04-17-admin-building-design.md`](2026-04-17-admin-building-design.md). Original spec was written before KerbalDialogueKit shipped and KerbalCampaignKit was designed; most of its data model and effect classes have moved downstack. This spec re-scopes Director around what remains: UI, conventions, and character registry.

## The Four-Mod Stack

```
BadgKatCareer ─→ BadgKatDirector ─→ KerbalCampaignKit ─→ KerbalDialogueKit ─→ ChatterBox
             └─→ KerbalCampaignKit (contract gating)
             └─→ KerbalDialogueKit (contract dialogue)
```

| Mod | Responsibility |
|---|---|
| **KerbalDialogueKit** (shipped v0.1.0) | Dialogue rendering, flags, choice capture, scene definitions |
| **KerbalCampaignKit** (Phase 1) | Chapters, event→action triggers, reputation economy, notification data API, CC REQUIREMENT types |
| **BadgKatDirector** (Phase 2) | Character registry, focus registry, memo pipeline, admin building UI, per-building character overlays, notification renderer, PR campaign action, disposition decay cfg convenience |
| **BadgKatCareer** (Phase 3) | Campaign-specific content: 6 characters, 5-act chapter map, focuses, memos, scenes, rep tuning, chapter-gating on existing contracts |

See [`2026-04-23-campaign-kit-design.md`](2026-04-23-campaign-kit-design.md) for the CampaignKit spec.

## Scope Shift From Original Spec

What the original spec put in Director but we've moved downstack:

| Original concept | Now lives as |
|---|---|
| `ChoiceHistory` | KDK flags (choices auto-persist as flags) |
| `milestoneChoices[]` | CampaignKit chapter history + KDK choice flags |
| `activeFocuses{}` | KDK flags (e.g., `wernher_focus`) |
| `contractsOriginated[]` | Triggers that spawn contracts; no separate registry |
| Effect classes (ContractWeightModifier, FundInjection, etc.) | CampaignKit trigger actions |
| StoryReleaseRequirement | CampaignKit's `FlagEquals` CC REQUIREMENT |
| DirectiveContract | Cfg pattern: trigger + KDK scene with choice; no special contract type |
| Reputation economy math | CampaignKit (Director just reads API for display) |
| "Effect layer" architecture | Collapses into CampaignKit's trigger action types |

What stays in Director:

1. **UI framework** — admin building replacement, per-building character overlays, KSC notification renderer
2. **Character registry** — `DIRECTOR_CHARACTER` cfg, disposition flag conventions
3. **Focus registry** — `DIRECTOR_FOCUS` cfg, picker UI
4. **Memo pipeline** — `DIRECTOR_MEMO` cfg with game-state conditions
5. **Disposition decay** — `DISPOSITION_DECAY` cfg that compiles to CampaignKit time-triggers
6. **PR campaign action** — specific gameplay action tied to admin UI

Director's C# codebase is ~60% smaller than the original spec implied.

---

# Part 1: BadgKatDirector (Framework)

Generic. Any consumer mod can ship its own cast, focuses, memos, and admin UI text on top.

## Design Philosophy

Director provides **UI + conventions** over CampaignKit/KDK state. Every piece of data Director cares about is a KDK flag or CampaignKit chapter state; Director just knows how to render it.

This means:

- Disposition is a flag — Director reads it for portrait tinting, CampaignKit triggers write it
- Focus is a flag — Director's picker UI writes it, contracts gate on it via `FlagEquals`
- Memory is flag history — Director doesn't maintain a separate memory list; content reads flags directly
- "Milestone directive" is not a Director class — it's a cfg pattern: chapter-entry trigger that enqueues a KDK scene with a choice. Director doesn't know or care.

## Dependencies

- KerbalCampaignKit >= 0.1.0
- KerbalDialogueKit >= 0.1.0
- CustomBarnKit (admin building UI replacement, facility-level gating)
- KSP 1.12.x

## Character Registry

### Character Definition

```cfg
DIRECTOR_CHARACTER
{
    id = wernher
    displayName = Wernher von Kerman
    role = Chief Scientist
    portraitModel = Instructor_Wernher
    baseColor = #FF82B4E8

    // Convention: Director reads disposition from this flag
    dispositionFlag = wernher_disposition
    defaultDisposition = Neutral

    // Optional: buildings where this character appears as an overlay
    buildings = Administration, RD, TrackingStation
}
```

### Disposition Values

Standard values: `Enthusiastic`, `Supportive`, `Neutral`, `Skeptical`, `Frustrated`. Content can define custom values; Director's UI treats unknown values as Neutral (with a log warning).

Each disposition has a default UI tint derived from the character's `baseColor` — Enthusiastic brightens, Frustrated desaturates, etc. Content can override per-character tints:

```cfg
DIRECTOR_CHARACTER
{
    id = wernher
    ...
    DISPOSITION_TINT { value = Enthusiastic; color = #FFA0D4FF }
    DISPOSITION_TINT { value = Frustrated;   color = #FF4E6E8A }
}
```

### Character API (C#)

```csharp
Director.Characters.Get("wernher");                  // CharacterInfo
Director.Characters.GetDisposition("wernher");       // string — reads KDK flag
Director.Characters.All;                              // enumerable
```

Directory-level is a thin facade; all mutation goes through `DialogueKit.Flags.Set` (typically from CampaignKit triggers).

## Focus Registry

### Focus Definition

```cfg
DIRECTOR_FOCUS
{
    character = wernher
    id = deep_space_signals
    title = Deep Space Signals
    description = Wernher focuses on tracking the anomaly signals.

    // Optional: when this focus is unlocked
    requirement = chapter >= 3 && completed_first_anomaly == true

    // Convention: Director writes to this flag on selection
    flag = wernher_focus
    flagValue = deep_space_signals
}
```

`requirement` reuses KDK's `FlagExpressionParser`. Focuses with unmet requirements don't appear in the picker.

Focus effects (contract weight modifiers, science bonuses, etc.) are not defined here — they're expressed as CampaignKit triggers or contract REQUIREMENTs that read `wernher_focus`. Director's job is UI + flag management; downstream mechanical effect is content's responsibility.

### Focus Picker UI

Character panel in admin building shows current focus per character. Clicking a character portrait opens a picker with all valid `DIRECTOR_FOCUS` entries for that character. Selecting sets the flag and closes the picker.

No "save" button — focus changes apply immediately. They can be changed any time (A-layer promise from original spec).

## Memo Pipeline

Memos are short character notes surfaced in the admin UI desk panel, driven by game state.

### Memo Definition

```cfg
DIRECTOR_MEMO
{
    character = gus
    id = relay_aging
    priority = low                     // low, normal, high
    text = That relay is getting old. Might want to think about upgrades before something breaks.

    CONDITION
    {
        type = VesselAge
        vesselType = Relay
        minDays = 200
    }

    // Optional: memo expires
    expireAfterDays = 60
    // Optional: don't re-show if dismissed
    suppressAfterDismiss = true
}
```

### Condition Types (Phase 1)

| Type | Parameters |
|---|---|
| `VesselAge` | `vesselType?`, `minDays`, `maxDays?` |
| `FundsBelow` | `threshold` |
| `ReputationBelow` | `threshold` |
| `ReputationAbove` | `threshold` |
| `ContractAvailable` | `group?`, `count?` |
| `BodyDiscovered` | `body?` (ResearchBodies soft) |
| `InChapter` | `chapter` |
| `FlagExpression` | `expression` |
| `TimeSinceEvent` | `event`, `minDays` |

Multiple `CONDITION` blocks AND together.

### Memo Evaluation

A `MemoTicker` polls active memos every few seconds (lightweight — just condition checks). Active memos queue into a display list sorted by priority then recency. The admin UI's desk panel renders the top N (configurable; default 5).

### Memo API

```csharp
Director.Memos.Active;                  // enumerable of active Memo
Director.Memos.Dismiss("relay_aging");  // hide until suppression clears
Director.Memos.Pinned;                  // memos with high priority, always visible
```

## Disposition Decay

A cfg-driven convenience that compiles to CampaignKit time-triggers at load.

### Disposition Decay Definition

```cfg
DISPOSITION_DECAY
{
    character = wernher
    towardValue = Neutral
    stepDays = 180

    STEP { from = Enthusiastic; to = Supportive }
    STEP { from = Supportive;   to = Neutral }
    STEP { from = Frustrated;   to = Skeptical }
    STEP { from = Skeptical;    to = Neutral }
}
```

Director loads these at startup and generates equivalent CampaignKit triggers:

```cfg
CAMPAIGN_TRIGGER
{
    id = disposition_decay_wernher_Enthusiastic_to_Supportive
    ON_EVENT { type = TimeElapsed; days = 180; ref = LastTrigger }
    WHEN { flagExpression = wernher_disposition == Enthusiastic }
    ACTIONS { SET_FLAG { name = wernher_disposition; value = Supportive } }
    once = false
}
```

Content can optionally ship no `DISPOSITION_DECAY` — dispositions then stay sticky until content explicitly shifts them.

## Admin Building UI — Layered Design

### Layout

```
┌──────────────┬──────────────────────────┬─────────────────┐
│  CHARACTERS  │     DASHBOARD / SCENE    │       DESK      │
│              │                          │                 │
│  [Gene]      │  (when nothing pending)  │  Story Contracts│
│  Supportive  │                          │  ─────────────  │
│              │  Program Status          │  ☐ Mun Probe    │
│  [Wernher]   │  Current Chapter: 3      │  ☐ Mun Anomaly  │
│  Enthusiastic│  Reputation: 312         │                 │
│              │  ████████░░ Respected    │  Memos          │
│  [Gus]       │  Monthly Income: 32,000  │  ─────────────  │
│  Neutral     │  Next Chapter: after     │  Gus: relay     │
│              │    Duna probe flyby      │     aging       │
│  [Mortimer]  │                          │  Walt: need     │
│  Neutral     │  Active Focuses          │     a headline  │
│              │  ─────────────           │                 │
│  [Walt]      │  Wernher: deep signals   │  [PR Campaign]  │
│  Supportive  │  Gus: launch infra       │                 │
│              │  Linus: signal analysis  │                 │
│  [Linus]     │                          │                 │
│  Enthusiastic│                          │                 │
└──────────────┴──────────────────────────┴─────────────────┘
```

Three panels: **Characters**, **Dashboard/Scene**, **Desk**. Dashboard center can switch to scene rendering when KDK is playing a scene — KDK renders in a modal overlay, Director dims the dashboard behind it.

### Layered Behavior (option D from brainstorming)

On admin entry:

1. **Scene layer (if pending).** CampaignKit has pending scenes flagged `OnFacilityEnter = Administration` — these auto-enqueue to KDK. Scene plays immediately. The dashboard is backdrop.
2. **Dashboard layer (always).** Program status, rep, focuses. Always visible as the room's baseline state.
3. **Desk layer (always).** Pending memos and story contracts pile up. Player processes at leisure.
4. **Notification layer.** Desk items with pending `action` notifications glow. Character portraits with pending `info` notifications get a subtle dot.

No "modes." The same screen has layered content; the foreground varies by what's pending.

### Character Panel

- Character portraits vertically stacked (6 for BadgKatCareer)
- Current disposition shown as tinted overlay + text label
- Current focus shown below name (if any)
- Click portrait → focus picker modal
- Right-click portrait → character scene (if content provides an on-demand scene id for that character, otherwise nothing)

### Program Status Panel

- Chapter name + description
- Current reputation value + tier label + progress to next tier
- Monthly passive income (if `REPUTATION_INCOME` configured)
- Next gate (if any) — queried from `CampaignKit.Reputation.NextGate()`
- Next chapter condition (from CampaignKit chapter state)
- Active focuses list

### Desk Panel

- Pending story contracts — contracts marked with a "story contract" flag that originate from admin rather than MC. Clicking opens a KDK pitch scene. Accepting activates the contract in CC.
- Memos — sorted by priority. Each memo dismissable.
- PR Campaign button (if content defines the action) — costs funds, halts decay, small rep bump.

### Stock UI Replacement

- MM patch disables stock strategies (already in BadgKatCareer)
- Director's UI replaces the admin building scene via CustomBarnKit integration
- Leaving admin returns to KSC normally

## Per-Building Character Overlays

### Phase 1 Buildings

Three buildings get overlays in Phase 1:

- **Administration** — full UI (above)
- **Mission Control** — light overlay: Gene's portrait in a corner panel, click for a scene about current mission status. Respects MC's existing contract UI (not replaced).
- **Tracking Station** — light overlay: Linus's portrait, click for scenes about anomaly signals or body discoveries. Doesn't replace tracking's vessel list or ResearchBodies observatory.

### Overlay Pattern

Each overlay is a small `KSPAddon(Startup.Flight)` MonoBehaviour (or equivalent per-scene) that:

1. Registers for the relevant facility scene (e.g., `MissionControl`)
2. Draws a small IMGUI corner panel with one or more character portraits
3. Each portrait shows its disposition tint
4. Clicking triggers a KDK scene (scene id determined by content via `DIRECTOR_BUILDING_SCENE` cfg)
5. Notification markers from CampaignKit appear as glows on portraits

```cfg
DIRECTOR_BUILDING_SCENE
{
    facility = MissionControl
    character = gene
    // When player clicks Gene's portrait in MC, this scene enqueues:
    onClickScene = gene_mc_briefing
    // Optional: show notification when any condition is met
    NOTIFICATION
    {
        severity = info
        source = gene_mc_briefing
        condition = chapter_transition_pending == true
    }
}
```

### Future Buildings (Phase 4)

- **R&D** — Wernher overlay
- **VAB/SPH** — Gus overlay
- **Astronaut Complex** — Walt overlay

Same overlay pattern. Deferred to Phase 4 because they're value-add rather than load-bearing for the BadgKatCareer narrative.

## KSC Notification Renderer

A `KSPAddon(KSPAddon.Startup.SpaceCentre)` MonoBehaviour renders CampaignKit notification markers over KSC buildings.

### Rendering Logic

Each frame:

1. For each KSC building (Admin, MC, Tracking, R&D, VAB, SPH, Astronaut):
   - Query `CampaignKit.Notifications.Highest("<building-target>")`
   - If Action → draw prominent marker (exclamation + pulse)
   - Else if Info → draw subtle marker (dot)
   - Else → nothing
2. Position markers using `Camera.main.WorldToScreenPoint(building.transform.position)` with fixed offset
3. Subscribe to `CampaignKitEvents.OnNotificationAdded/Cleared` to skip polling when nothing changed

### Notification Targets

Convention: the first path segment of a notification target is the building:

- `admin`, `admin.desk.contracts` → Admin marker
- `mc`, `mc.contracts` → Mission Control marker
- `tracking`, `tracking.signals` → Tracking Station marker
- `rd` → R&D marker
- `vab`, `sph` → VAB/SPH markers
- `astronaut` → Astronaut Complex marker
- `character.*` → wherever that character is on-screen (handled by building overlays, not KSC renderer)

## PR Campaign Action

The one bespoke gameplay action Director provides.

```cfg
DIRECTOR_PR_CAMPAIGN
{
    // Cost formula: base + (currentRepTier * tierMultiplier)
    baseCost = 50000
    tierCostMultiplier = 25000

    // Effects
    haltDecayDays = 60
    repBonus = 10
}
```

Rendered as a button in the admin desk panel. Clicking shows confirmation with cost. On confirm:

- `ADJUST_FUNDS { amount = -cost }`
- `CampaignKit.Reputation.HaltDecay(days: 60)`
- `ADJUST_REPUTATION { amount = 10 }`

Content mods can skip this by not defining `DIRECTOR_PR_CAMPAIGN`.

## Save Structure

Director's save state is minimal — most of what Director renders lives in KDK flags and CampaignKit state.

```cfg
SCENARIO
{
    name = DirectorScenario

    // Dismissed memos (so they don't re-show)
    DISMISSED_MEMOS
    {
        memo = relay_aging; dismissedAt = 1800000
    }

    // Track last time PR campaign was used, for cooldown display
    PR_CAMPAIGN
    {
        lastUsedTime = 1750000
    }
}
```

## Plugin Structure

```
BadgKatDirector/
├── Core/
│   ├── Director.cs                 // Public static API
│   ├── DirectorScenario.cs         // Dismissed memos, PR cooldown
│   └── DirectorAddon.cs            // Loads cfg, compiles disposition decay, wires UI
│
├── Characters/
│   ├── CharacterInfo.cs
│   ├── CharacterRegistry.cs        // Loads DIRECTOR_CHARACTER
│   └── DispositionColors.cs        // Tint derivation + overrides
│
├── Focuses/
│   ├── FocusOption.cs
│   ├── FocusRegistry.cs
│   └── FocusPicker.cs              // Picker modal UI
│
├── Memos/
│   ├── Memo.cs
│   ├── MemoRegistry.cs
│   ├── MemoConditions/
│   │   ├── IMemoCondition.cs
│   │   ├── VesselAgeCondition.cs
│   │   ├── FundsBelowCondition.cs
│   │   ├── ReputationCondition.cs
│   │   ├── ContractAvailableCondition.cs
│   │   ├── BodyDiscoveredCondition.cs
│   │   ├── InChapterCondition.cs
│   │   └── FlagExpressionCondition.cs
│   └── MemoTicker.cs               // Periodic evaluation
│
├── DecayCompiler/
│   └── DispositionDecayCompiler.cs // DISPOSITION_DECAY → CampaignKit triggers
│
├── Admin/
│   ├── AdminBuildingUI.cs          // KSPAddon, replaces stock admin screen
│   ├── CharacterPanel.cs
│   ├── DashboardPanel.cs
│   ├── DeskPanel.cs
│   └── PrCampaignButton.cs
│
├── BuildingOverlays/
│   ├── BuildingOverlay.cs          // Base overlay
│   ├── MissionControlOverlay.cs
│   ├── TrackingStationOverlay.cs
│   └── BuildingSceneRegistry.cs    // DIRECTOR_BUILDING_SCENE
│
└── KscRenderer/
    └── KscNotificationRenderer.cs  // Markers over buildings in KSC view
```

Roughly 25-30 classes.

---

# Part 2: BadgKatCareer (Content)

All story-specific content. Additive to the existing mod — contracts stay; new Director-facing content files added.

## The Cast

Six `DIRECTOR_CHARACTER` entries in `BadgKatCareer/DirectorContent/Characters.cfg`. Already defined in the existing codebase:

| Character | Role | Model | Color |
|---|---|---|---|
| Gene | Mission direction, crew safety | `Instructor_Gene` | `#FF8BE08A` |
| Wernher | Science, anomaly research | `Instructor_Wernher` | `#FF82B4E8` |
| Gus | Engineering, infrastructure | `Strategy_MechanicGuy` | `#FFFFC078` |
| Mortimer | Budget, finance | `Strategy_Mortimer` | `#FFFFE066` |
| Walt | PR, reputation | `Strategy_PRGuy` | `#FFC8A0E8` |
| Linus | Signals, anomalies | `Strategy_ScienceGuy` | `#FF6ED4C8` |

## Chapter Map

Five chapters mapped to the existing campaign's five acts. Each chapter gets a `CAMPAIGN_CHAPTER` entry + entry trigger.

| Chapter | Name | Entry | Exit (→ next) |
|---|---|---|---|
| 1 | First Steps | Startup | First surface science + first recovered flight |
| 2 | Into Orbit | Probe suborbital | First orbit |
| 3 | Beyond the Mun | First Mun return | First interplanetary flyby |
| 4 | Interplanetary | Interplanetary flyby success | Jool system arrival |
| 5 | The Kcalbeloh Wormhole | Jool orbit | (endgame) |

### Milestone Directives

Each chapter-entry trigger (except Chapter 1) enqueues a directive KDK scene marked `OnFacilityEnter = Administration`. Player's first admin visit after the chapter changes plays the scene. Scene has 2-3 choices; choice flags feed downstream triggers.

Nine directives total across the five chapters (some chapters have mid-chapter directives triggered by contract completions, not just on entry). Same map as the original spec:

| Chapter | Trigger | Characters | Choice Axis |
|---|---|---|---|
| 1 | Both surface leads complete | Gene vs Wernher | Push to space vs more ground work |
| 2 | First orbit | Mortimer vs Wernher | Conservative budget vs ambitious probes |
| 2 | First crew in space | Gene vs Walt | Quiet professionalism vs media blitz |
| 3 | Mun monolith found | Wernher vs Gus | Study monolith vs Mun infrastructure |
| 3 | Ready for interplanetary | Gene vs Mortimer | Multiple targets vs focus one |
| 4 | Duna/Eve probe data | Wernher vs Gus | Science-heavy vs engineering-first |
| 4 | First crewed interplanetary | Gene vs Walt | Low-profile vs showcase |
| 5 | Jool system reached | Wernher vs Linus | Systematic survey vs chase signal |
| 5 | Kcalbeloh wormhole | Gene vs everyone | Go through |

## Focuses

~18 `DIRECTOR_FOCUS` entries in `BadgKatCareer/DirectorContent/Focuses.cfg`, 3 per character. Example:

- **Wernher:** Anomaly Research, Deep Space Signals, Material Science
- **Gus:** Launch Infrastructure, Surface Operations, Interplanetary Vehicles
- **Gene:** Crew Safety, Mission Pacing, Program Direction
- **Mortimer:** Cost Optimization, Investor Relations, Austerity
- **Walt:** Public Outreach, Media Relations, International Partnerships
- **Linus:** Signal Analysis, Body Discovery, Anomaly Cataloging

Unlock requirements scale with chapter progression.

## Memos

~20 `DIRECTOR_MEMO` entries in `BadgKatCareer/DirectorContent/Memos.cfg`. Sample:

- Gus: relay aging, low funds = tightening belt, new part available
- Wernher: new body discovered, low science output, anomaly pattern found
- Walt: low rep warning, new media opportunity, investor visiting
- Mortimer: low funds, high funds = reinvest?, approaching gate
- Linus: faint signal detected, body conjunction, Kraken sighting
- Gene: crew ready for promotion, veteran returning, all crew grounded

## Staff Meeting Scenes

~10-15 KDK scenes in `BadgKatCareer/DirectorContent/StaffMeetings/`. Triggered by CampaignKit triggers on events like first successful orbit, first crew return, major failure, each chapter transition (in addition to the directive scene).

## Reputation Config

`BadgKatCareer/DirectorContent/Reputation.cfg` ships:

```cfg
REPUTATION_INCOME
{
    enabled = true
    intervalDays = 30
    TIER { min = 0;   max = 100; income = 5000 }
    TIER { min = 100; max = 250; income = 15000 }
    TIER { min = 250; max = 500; income = 35000 }
    TIER { min = 500; max = 750; income = 60000 }
    TIER { min = 750; max = 9999; income = 100000 }
}

REPUTATION_DECAY
{
    enabled = true
    ratePercentPerMonth = 1.5
    resetOnContractComplete = true
    tierFloors = true
}

DIRECTOR_PR_CAMPAIGN
{
    baseCost = 50000
    tierCostMultiplier = 25000
    haltDecayDays = 60
    repBonus = 10
}
```

No `ReputationMinimum` REQUIREMENTs on BadgKatCareer contracts — chapter gating handles that. Select high-profile crewed contracts get `ReputationStake` behaviours (Duna landing, Jool mission, Kcalbeloh).

## Disposition Decay

`BadgKatCareer/DirectorContent/DispositionDecay.cfg`:

```cfg
DISPOSITION_DECAY
{
    character = wernher
    towardValue = Neutral
    stepDays = 180
    STEP { from = Enthusiastic; to = Supportive }
    STEP { from = Supportive;   to = Neutral }
    STEP { from = Frustrated;   to = Skeptical }
    STEP { from = Skeptical;    to = Neutral }
}
// Repeated for all six characters
```

Slow decay — six months per step. Completing a contract in a character's domain snaps them back up (handled by CampaignKit triggers content adds).

## Contract Integration

Additive, no rewrites:

- **Chapter gates:** Each existing story contract gets a `ChapterAtLeast` REQUIREMENT matching where it should unlock.
- **Stake tags:** Major crewed landings (Duna, Eve, Jool system, Kcalbeloh) get `ReputationStake` BEHAVIOURs.
- **ChatterBox BEHAVIOURs stay** on existing contracts. New contracts (story contracts originated in admin) use KDK scenes instead, triggered via trigger + `ENQUEUE_SCENE`.
- **Story contract "release" flag:** Contracts greenlit from admin desk get a flag set on the scene's acceptance choice; `FlagEquals` REQUIREMENT holds them from MC until released. Not used on existing contracts.

## Building Scenes

Per-building overlay scene references:

```cfg
DIRECTOR_BUILDING_SCENE
{
    facility = MissionControl
    character = gene
    onClickScene = gene_mc_status_briefing
}

DIRECTOR_BUILDING_SCENE
{
    facility = TrackingStation
    character = linus
    onClickScene = linus_tracking_signals
}

DIRECTOR_BUILDING_SCENE
{
    facility = Administration
    character = mortimer
    onClickScene = mortimer_admin_budget
}
// And Walt, since Walt's natural building is Admin for Phase 1
DIRECTOR_BUILDING_SCENE
{
    facility = Administration
    character = walt
    onClickScene = walt_admin_pr
}
```

These scenes can vary by chapter via `visibleIf` — a single `onClickScene` id references a scene whose lines conditionally fire based on current chapter.

---

# Phasing

- **Phase 0 (shipped)** — KerbalDialogueKit v0.1.0
- **Phase 1** — KerbalCampaignKit v0.1.0 (spec in `2026-04-23-campaign-kit-design.md`). Headless, usable independently. Deliverable: chapters, triggers, rep economy, notifications, CC REQUIREMENTs.
- **Phase 2** — BadgKatDirector v0.1.0. Character/focus/memo registries, admin building UI, Mission Control + Tracking Station overlays, KSC notification renderer, PR campaign action, disposition decay compiler.
- **Phase 3** — BadgKatCareer content integration. 6 characters, 5 chapters, 9 directives, 18 focuses, 20 memos, 10-15 staff meetings, rep config, chapter-gate REQUIREMENTs added to existing contracts, select rep-stake tags.
- **Phase 4** — R&D, VAB/SPH, Astronaut Complex overlays + second-pass polish.

Each phase is a shippable increment. Phase 1 is usable by third-party mods without waiting for Director. Phase 2 without Phase 3 gives a playable admin building with placeholder content. Phase 3 is the first version that feels like the intended BadgKatCareer experience.

## Scope Summary

**KerbalCampaignKit:**
- ~40 classes, single DLL
- Detailed in `2026-04-23-campaign-kit-design.md`

**BadgKatDirector:**
- ~25-30 classes
- Cfg formats: `DIRECTOR_CHARACTER`, `DIRECTOR_FOCUS`, `DIRECTOR_MEMO`, `DISPOSITION_DECAY`, `DIRECTOR_BUILDING_SCENE`, `DIRECTOR_PR_CAMPAIGN`
- No campaign content — pure framework

**BadgKatCareer additions:**
- 6 `DIRECTOR_CHARACTER`
- 5 `CAMPAIGN_CHAPTER`
- ~30 `CAMPAIGN_TRIGGER` (chapter entry, milestone directives, memo shifts, reputation events)
- ~18 `DIRECTOR_FOCUS`
- ~20 `DIRECTOR_MEMO`
- ~10-15 staff meeting KDK scenes
- ~9 directive KDK scenes
- `REPUTATION_INCOME`, `REPUTATION_DECAY`, `DIRECTOR_PR_CAMPAIGN`
- 6 `DISPOSITION_DECAY` (one per character)
- `DIRECTOR_BUILDING_SCENE` entries for Admin/MC/Tracking characters
- Additive `ChapterAtLeast` REQUIREMENTs on existing contracts
- `ReputationStake` BEHAVIOURs on selected high-profile missions

**Out of Scope:**
- Stock building 3D asset replacement (overlays sit on top of stock scenes)
- Localization (all text is cfg, translation is content's concern)
- Multiplayer/shared state (single-save, per-save state)
- Debug/admin tooling UI (deferred to post-v0.1)
