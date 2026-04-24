# KerbalAdminKit (KAK) — Design

A toolkit mod that customizes the KSP Administration Building and surfaces character-authored program information across KSC. Renders UI for data owned by KerbalCampaignKit (KCK) and KerbalDialogueKit (KDK); does not own game state itself.

**Companion specs:**
- [`2026-04-23-campaign-kit-design.md`](2026-04-23-campaign-kit-design.md) — KerbalCampaignKit (shipped v0.1.0)
- [`2026-04-23-director-career-redesign.md`](2026-04-23-director-career-redesign.md) — high-level four-mod stack overview and BadgKatCareer content plan

## What KAK Is

KAK is a **configurable admin-building replacement**. Content mods declare characters, focuses, memos, building scenes, and reputation/decay tuning via cfg; KAK renders the corresponding admin UI, KSC notification markers, and per-building character overlays. Economic math lives in KCK; KAK reads KCK's API and provides one admin-scoped gameplay action (PR Campaign).

## Design Principles

- **Inert without content.** Install KAK alone, nothing changes. Stock admin, stock strategies, stock MessageSystem all behave normally.
- **Opt-in admin takeover.** A top-level `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }` is required for KAK to intercept the admin building click. Without it, KAK only activates the lightweight features content has declared (KSC markers, overlays, optional stock-mail memos).
- **Per-feature activation.** Each UI section and hook checks for its cfg. No characters → no character panel. No memos → no memo section. No `DIRECTOR_PR_CAMPAIGN` → no PR button.
- **Economy math belongs to KCK.** KAK displays rep tier / gate / monthly income by reading `CampaignKit.Reputation.*`. The only direct economy action is PR Campaign, which calls KCK's public API.
- **Dialogue belongs to KDK.** Clicking any character chip or overlay chip enqueues a KDK scene id. KAK never renders a dialogue popup — that's ChatterBox's job via KDK.
- **Themeable at the chip level.** Every character and every notification glyph can be skinned via cfg (`portraitTexture`, `textureUrl`, color, tint). Defaults ship; content overrides piecewise.

## Dependencies

- KerbalCampaignKit ≥ 0.1.0
- KerbalDialogueKit ≥ 0.1.0
- ModuleManager ≥ 4.2.3
- HarmonyKSP (for admin-click interception)
- ClickThroughBlocker (for IMGUI window click isolation)
- KSP 1.12.x

No CustomBarnKit dependency. CustomBarnKit only tunes facility stats and is orthogonal to KAK's scope.

---

## Activation Model

At `MainMenu` startup, `AdminKitAddon` (a `KSPAddon(Startup.MainMenu, true)` MonoBehaviour) scans `GameDatabase` for KAK cfg nodes and decides what to wire:

| Condition | Effect |
|---|---|
| No `DIRECTOR_*` or `KERBAL_ADMIN_KIT` cfg anywhere | No hooks registered. Stock everything. |
| Cfg exists, `replaceAdminBuilding = false` or absent | KSC notification renderer, per-building overlays, stock-mail memos activate (each gated by its own cfg). Admin click → stock admin. |
| `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }` | Harmony prefix patch installed on admin click handler. Click opens KAK admin UI instead of stock scene. |

Detection happens once at `MainMenu`. Adding cfg mid-save requires a restart.

### `KERBAL_ADMIN_KIT` Settings Block

```cfg
KERBAL_ADMIN_KIT
{
    replaceAdminBuilding = true
    deskMemoCount = 5
    memoPollSeconds = 5
}
```

All fields optional with defaults: `replaceAdminBuilding = false`, `deskMemoCount = 5`, `memoPollSeconds = 5`.

---

## Character Registry

### `DIRECTOR_CHARACTER` Definition

```cfg
DIRECTOR_CHARACTER
{
    id = wernher
    displayName = Wernher von Kerman
    role = Chief Scientist
    baseColor = #FF82B4E8

    // Optional visual overrides
    portraitTexture = BadgKatCareer/Art/wernher.png
    chipShape = square

    // Optional click-to-scene
    onClickScene = wernher_admin_entry

    // Convention: disposition read from this flag
    dispositionFlag = wernher_disposition
    defaultDisposition = Neutral

    // Optional per-value visual/label overrides
    DISPOSITION_TINT  { value = Enthusiastic; color = #FFA0D4FF }
    DISPOSITION_LABEL { value = Enthusiastic; text  = Thrilled }
}
```

Fields:

- `id` — required, unique per character.
- `displayName`, `role`, `baseColor` — required for default chip rendering.
- `portraitTexture` — optional `GameDatabase` URL. If set, chip renders the texture with disposition tint overlay; else renders a filled `baseColor` rectangle.
- `chipShape` — reserved for v0.2; v0.1 always renders square rounded-corner.
- `onClickScene` — optional KDK scene id. If set, clicking the chip fires `DialogueKit.EnqueueById(onClickScene)`.
- `dispositionFlag` — KDK flag KAK reads for the disposition label and tint.
- `defaultDisposition` — fallback value when the flag is unset.
- `DISPOSITION_TINT` blocks — override default tint derivation for specific disposition values.
- `DISPOSITION_LABEL` blocks — override display text for specific disposition values.

Standard disposition values KAK recognizes: `Enthusiastic`, `Supportive`, `Neutral`, `Skeptical`, `Frustrated`. Unknown values render as Neutral with a one-time log warning.

### Character API

```csharp
AdminKit.Characters.Get("wernher");            // CharacterInfo
AdminKit.Characters.GetDisposition("wernher"); // string, reads KDK flag
AdminKit.Characters.All;                       // IEnumerable<CharacterInfo>
```

---

## Focus Registry

### `DIRECTOR_FOCUS` Definition

```cfg
DIRECTOR_FOCUS
{
    character = wernher
    id = deep_space_signals
    title = Deep Space Signals
    description = Wernher focuses on tracking the anomaly signals.

    // Optional unlock condition (KDK flag expression syntax)
    requirement = chapter >= 3 && completed_first_anomaly == true

    // Convention: Director writes to this flag on selection
    flag = wernher_focus
    flagValue = deep_space_signals
}
```

`requirement` reuses KDK's `FlagExpressionParser`. Focuses with unmet requirements are omitted from the picker.

### Focus Picker UI

Modal IMGUI window centered on the admin screen. Lists all valid `DIRECTOR_FOCUS` entries for the clicked character, sorted by `id` (content can order with integer prefixes in ids if needed). Selection:

1. Calls `DialogueKit.Flags.Set(focus.flag, focus.flagValue)`.
2. Closes the picker.

No save button. No "clear focus" button in v0.1 (deferred). Changing to a different focus overwrites the flag.

Focus *effects* (contract weight, bonus science, etc.) are not defined here. Content expresses effects as KCK triggers watching the focus flag or as CC REQUIREMENTs reading it via `FlagEquals`.

---

## Memo Pipeline

Memos are short character-attributed notes surfaced in the admin desk panel, driven by game state.

### `DIRECTOR_MEMO` Definition

```cfg
DIRECTOR_MEMO
{
    character = gus
    id = relay_aging
    priority = low          // low | normal | high
    text = That relay is getting old. Might want to think about upgrades before something breaks.

    CONDITION
    {
        type = VesselAge
        vesselType = Relay
        minDays = 200
    }

    expireAfterDays = 60
    suppressAfterDismiss = true

    postToStockMail = true
}
```

- Multiple `CONDITION` blocks AND together. All must match for the memo to activate.
- `expireAfterDays` — once active, auto-remove from active list after N days regardless of conditions.
- `suppressAfterDismiss = true` — once player dismisses, never re-activates for this save.
- `postToStockMail = true` — the first time a memo activates in a save, also calls `MessageSystem.Instance.AddMessage` with character color and text. Player sees the mailbox blip mid-flight; memo also appears on desk when they return to KSC. Tracked in the KAK save (`STOCK_MAILED_MEMOS`) so a reload doesn't re-send — MessageSystem already persists the stock message itself, so one delivery per memo per save is the guarantee.

### Condition Types (v0.1)

| Type | Parameters |
|---|---|
| `VesselAge` | `vesselType?`, `minDays`, `maxDays?` |
| `FundsBelow` | `threshold` |
| `ReputationBelow` | `threshold` |
| `ReputationAbove` | `threshold` |
| `ContractAvailable` | `group?`, `count?` |
| `BodyDiscovered` | `body?` (ResearchBodies soft dep) |
| `InChapter` | `chapter` |
| `FlagExpression` | `expression` |
| `TimeSinceEvent` | `event`, `minDays` |

Conditions implement `IMemoCondition.Evaluate(MemoContext)` returning bool. Pure-logic conditions (`InChapter`, `FlagExpression`, `ReputationAbove/Below`, `FundsBelow`) are unit-testable with fakes. KSP-integrated conditions (`VesselAge`, `ContractAvailable`, `BodyDiscovered`) are tested manually.

### Memo Ticker

A `KSPAddon(Startup.SpaceCentre, false)` MonoBehaviour polls active memos every `memoPollSeconds` (default 5) while at KSC. In flight, editor, or tracking, the ticker sleeps — memos can only surface on return to KSC. This bounds overhead.

### Memo API

```csharp
AdminKit.Memos.Active;                 // IEnumerable<Memo>, sorted by priority desc
AdminKit.Memos.Dismiss("relay_aging");
AdminKit.Memos.Pinned;                 // high-priority, always-visible
```

---

## Admin Building UI

### Entry

With `replaceAdminBuilding = true`, a Harmony prefix patch on the admin building's click handler returns `false` to suppress the stock scene and opens KAK's IMGUI window at KSC. Likely target is `SpaceCenterBuilding.OnClicked` specialized via instance-identity check against the administration facility; the implementation plan confirms the exact method during research (stock KSP class inspection).

### Layout

Fixed three-panel layout in v0.1:

```
┌──────────────┬──────────────────────────┬─────────────────┐
│  CHARACTERS  │        DASHBOARD         │       DESK      │
│              │                          │                 │
│  (chips)     │  Program status          │  Story contracts│
│              │  Reputation tier + gate  │  ──────────     │
│              │  Monthly income          │  Memos          │
│              │  Next chapter condition  │  ──────────     │
│              │  Active focuses          │  [PR Campaign]  │
│              │                          │                 │
└──────────────┴──────────────────────────┴─────────────────┘
```

Panels degrade:

- No `DIRECTOR_CHARACTER` cfg → character column hidden; admin becomes two-panel (Dashboard + Desk).
- No focuses defined for any character → Active Focuses subsection hidden.
- No `DIRECTOR_MEMO` cfg → memo section hidden; desk may be empty aside from PR button.
- No `DIRECTOR_PR_CAMPAIGN` cfg → PR button hidden.

All IMGUI windows wrap `ClickThroughBlocker.GUILayoutWindow` to prevent click leak to KSC behind.

### Character Chip

Per character, repeated in the left column:

1. Rectangle rendered as either the character's `baseColor` fill or `portraitTexture` (if set).
2. Disposition tint overlay on top (from `DISPOSITION_TINT` or derived from `baseColor`).
3. Name (line 1), role (line 2), disposition label (line 3), current focus title (line 4, if focus set).
4. Outline pulse when `CampaignKit.Notifications.Highest("character." + id)` returns non-null. Pulse intensity by severity.
5. Click behavior:
   - If character has any focuses → open focus picker modal.
   - Else if `onClickScene` set → fire `DialogueKit.EnqueueById(onClickScene)`.
   - Else → no action.

### Dashboard Panel

Dynamic content:

- **Chapter name + description.** From `CampaignKit.Chapters.Current` and `ChapterInfo.Description`.
- **Reputation tier.** Current rep value, tier label from `REPUTATION_INCOME` tiers (if configured), progress bar to next tier boundary.
- **Monthly income.** `CampaignKit.Reputation.Income.CurrentMonthlyIncome()` (if configured).
- **Next gate.** `CampaignKit.Reputation.NextGate()` — content-defined string or null.
- **Next chapter condition.** Display-only string from `CAMPAIGN_CHAPTER { nextCondition = "..." }`. KAK does not evaluate this; it renders the content-provided hint.
- **Active focuses.** For each character with a focus flag set, render "Character: focus title."

All sections render conditionally. Missing data → section collapses.

### Desk Panel

- **Story contracts.** Contracts flagged for admin-originated release (by content convention; typically a `DIRECTOR_STORY_CONTRACT` cfg naming the contract id). Shows pitch text + Accept / Decline buttons. Accept fires a KDK scene for the pitch; Decline clears the release flag. (Full `DIRECTOR_STORY_CONTRACT` cfg design deferred — v0.1 reads flags named `<contractId>_pending_release` and defers the formal cfg to v0.2.)
- **Memos.** Top `deskMemoCount` (default 5) from `AdminKit.Memos.Active`, sorted priority desc then addedAt asc. Each row: character color stripe, attribution "Gus:", body text, `×` dismiss button.
- **PR Campaign button.** Visible iff `DIRECTOR_PR_CAMPAIGN` cfg present. Cost computed live from current rep tier. Click opens confirmation modal; confirm applies effects.

### Focus Picker Modal

Centered IMGUI window over the admin screen. Lists valid focuses for the clicked character. Row: title + description. Click → set flag, close modal.

### PR Campaign Action

```cfg
DIRECTOR_PR_CAMPAIGN
{
    baseCost = 50000
    tierCostMultiplier = 25000

    haltDecayDays = 60
    repBonus = 10
}
```

On confirm:
1. `Funding.Instance.AddFunds(-cost, TransactionReasons.Strategies)`
2. `Reputation.Instance.AddReputation(repBonus, TransactionReasons.Strategies)`
3. `CampaignKit.Reputation.HaltDecay(haltDecayDays)`
4. Records `lastUsedTime` in `KerbalAdminKitScenario`.

Cost formula: `baseCost + (currentTierIndex * tierCostMultiplier)`. Tier index derived from `REPUTATION_INCOME` configured tiers (0 if none configured).

---

## Per-Building Character Overlays

### Phase 1 Coverage

- **Administration** — full UI (above).
- **MissionControl** — IMGUI corner panel with chip(s).
- **TrackingStation** — IMGUI corner panel with chip(s).

Deferred to v0.2: R&D, VAB, SPH, Astronaut Complex. Same overlay pattern.

### `DIRECTOR_BUILDING_SCENE` Definition

```cfg
DIRECTOR_BUILDING_SCENE
{
    facility = MissionControl
    character = gene
    onClickScene = gene_mc_status_briefing
}
```

Multiple entries per facility allowed — each renders its own chip in the corner panel.

### Overlay Implementation

Per-facility `KSPAddon(Startup.MissionControl, false)` and `KSPAddon(Startup.TrackingStation, false)` MonoBehaviours:

1. On `Start`, read all `DIRECTOR_BUILDING_SCENE` cfg matching the facility.
2. In `OnGUI`, draw a top-right corner panel with one chip per entry (reusing character-chip rendering).
3. Click → `DialogueKit.EnqueueById(entry.onClickScene)`.
4. Notification glow when `CampaignKit.Notifications.Highest("<facility-prefix>.character." + characterId)` returns non-null.

Facility prefix map:
- `MissionControl` → `mc`
- `TrackingStation` → `tracking`

### KSC Notification Renderer

`KSPAddon(Startup.SpaceCentre, false)` MonoBehaviour renders markers over KSC buildings in `OnGUI`.

Per-frame logic:

1. For each tracked facility (`Administration`, `MissionControl`, `TrackingStation`, `ResearchAndDevelopment`, `VehicleAssemblyBuilding`, `SpaceplaneHangar`, `AstronautComplex`):
   - Query `CampaignKit.Notifications.Highest(prefix)` for the facility's target prefix.
   - If `Action` severity → draw prominent glyph (exclamation + pulse).
   - Else if `Info` severity → draw subtle glyph (dot).
   - Else → skip.
2. Position: `Camera.main.WorldToScreenPoint(facility.transform.position) + DIRECTOR_KSC_MARKER_OFFSET`.
3. Subscribe to `CampaignKitEvents.OnNotificationAdded/Cleared` to update cached marker states; draw loop uses cache so it stays cheap.

### `DIRECTOR_NOTIFICATION_STYLE` Definition

```cfg
DIRECTOR_NOTIFICATION_STYLE
{
    severity = Action
    textureUrl = KerbalAdminKit/Art/exclamation
    color = #FFFF4040
    pulse = true
}

DIRECTOR_NOTIFICATION_STYLE
{
    severity = Info
    textureUrl = KerbalAdminKit/Art/dot
    color = #FFFFE066
    pulse = false
}
```

Both severities ship with defaults. Content overrides by writing cfg blocks with matching `severity`.

### `DIRECTOR_KSC_MARKER_OFFSET` Definition

```cfg
DIRECTOR_KSC_MARKER_OFFSET
{
    building = Administration
    x = 0
    y = 20
}
```

Optional per-building offset. Defaults centered above each building's transform; content tunes.

---

## Disposition Decay Compiler

A cfg convenience that emits KCK triggers at startup.

### `DISPOSITION_DECAY` Definition

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

### Compiler Behavior

At `MainMenu` startup, after KCK's trigger engine initializes, `DispositionDecayCompiler` iterates `DISPOSITION_DECAY` cfg and registers one synthesized `CAMPAIGN_TRIGGER` per STEP:

```
id: disposition_decay_<character>_<from>_to_<to>
event: TimeElapsed (stepDays) referencing the character's disposition flag's last-changed time
condition: <character>_disposition == <from>
action: SET_FLAG <character>_disposition = <to>
once: false (re-fires as steps repeat)
```

Triggers are registered programmatically via `CampaignKit.Engine.Register(...)` — never serialized to cfg. Compiler is pure-logic and unit-testable with a fake trigger registry.

Content can ship no `DISPOSITION_DECAY` — dispositions then stay sticky until content shifts them via other triggers.

---

## Save Structure

```cfg
SCENARIO
{
    name = KerbalAdminKitScenario

    DISMISSED_MEMOS
    {
        MEMO { id = relay_aging; dismissedAt = 1800000 }
    }

    STOCK_MAILED_MEMOS
    {
        MEMO { id = relay_aging }
        MEMO { id = funds_low_warning }
    }

    PR_CAMPAIGN
    {
        lastUsedTime = 1750000
    }
}
```

That's it. Dispositions, focuses, chapter state, notifications, triggers all live in KDK flags and KCK's scenario. `STOCK_MAILED_MEMOS` is a flat id list recording which memos have already fired a stock MessageSystem entry so reloads don't duplicate. KAK's save footprint is intentionally tiny.

---

## Build & Plugin Structure

Mirrors the KCK pattern:

```
KerbalAdminKit/
├── Directory.Build.props              // redirects BaseIntermediateOutputPath/BaseOutputPath out of GameData
├── .gitignore
├── LICENSE                             // MIT
├── README.md
├── KerbalAdminKit.sln
├── KerbalAdminKit.version              // KSP-AVC, published at known URL
├── demo/                               // self-contained demo cfg
│   └── demo-admin-kit.cfg
├── Plugins/                            // build target; DLL lands here
├── Art/                                // default glyphs: exclamation.png, dot.png
├── src/
│   └── KerbalAdminKit/
│       ├── Core/
│       │   ├── AdminKit.cs                  // public static API
│       │   ├── AdminKitScenario.cs          // dismissed memos, PR cooldown
│       │   └── AdminKitAddon.cs             // startup: cfg scan, Harmony patch, wiring
│       │
│       ├── Settings/
│       │   └── AdminKitSettings.cs          // KERBAL_ADMIN_KIT cfg loader
│       │
│       ├── Characters/
│       │   ├── CharacterInfo.cs
│       │   ├── CharacterRegistry.cs
│       │   └── DispositionColors.cs         // tint derivation + overrides
│       │
│       ├── Focuses/
│       │   ├── FocusOption.cs
│       │   ├── FocusRegistry.cs
│       │   └── FocusPicker.cs               // modal UI
│       │
│       ├── Memos/
│       │   ├── Memo.cs
│       │   ├── MemoContext.cs
│       │   ├── MemoRegistry.cs
│       │   ├── MemoTicker.cs
│       │   ├── MemoLoader.cs
│       │   └── Conditions/
│       │       ├── IMemoCondition.cs
│       │       ├── VesselAgeCondition.cs
│       │       ├── FundsBelowCondition.cs
│       │       ├── ReputationAboveCondition.cs
│       │       ├── ReputationBelowCondition.cs
│       │       ├── ContractAvailableCondition.cs
│       │       ├── BodyDiscoveredCondition.cs
│       │       ├── InChapterCondition.cs
│       │       ├── FlagExpressionCondition.cs
│       │       └── TimeSinceEventCondition.cs
│       │
│       ├── DecayCompiler/
│       │   └── DispositionDecayCompiler.cs
│       │
│       ├── Admin/
│       │   ├── AdminBuildingUI.cs           // main IMGUI window, KSPAddon(SpaceCentre)
│       │   ├── AdminClickInterceptor.cs     // Harmony patch
│       │   ├── CharacterPanel.cs
│       │   ├── DashboardPanel.cs
│       │   ├── DeskPanel.cs
│       │   ├── ChipRenderer.cs              // shared chip-drawing routine
│       │   └── PrCampaignAction.cs
│       │
│       ├── BuildingOverlays/
│       │   ├── BuildingOverlayBase.cs
│       │   ├── MissionControlOverlay.cs
│       │   ├── TrackingStationOverlay.cs
│       │   └── BuildingSceneRegistry.cs
│       │
│       ├── KscRenderer/
│       │   ├── KscNotificationRenderer.cs
│       │   └── NotificationStyleRegistry.cs
│       │
│       └── Util/
│           └── TextureLoader.cs             // GameDatabase texture URL → Texture2D
│
└── tests/
    └── KerbalAdminKit.Tests/
        ├── MemoConditionTests.cs
        ├── DispositionDecayCompilerTests.cs
        ├── CharacterRegistryTests.cs
        ├── FocusRegistryTests.cs
        ├── MemoRegistryTests.cs
        ├── NotificationStyleRegistryTests.cs
        └── Helpers/
            └── FakeSceneNode.cs             // reuses KCK's ISceneNode abstraction
```

Roughly 25–30 classes. Test project covers pure-logic: condition evaluation, decay compilation, cfg loading via `ISceneNode` fakes. KSP-integrated code (UI, Harmony patch, overlays) verified manually.

---

## v0.2 — Desk Expansion (Not Building)

Noted here so v0.1 planning accounts for extension points:

- **`DIRECTOR_POLICY`** — content-defined long-running modifiers (name, cost, duration, activation triggers, deactivation triggers). Renders as toggle in desk. Replaces stock-strategy concept with content-authored equivalents. Requires new desk subpanel and trigger-system integration.
- **Action memos** — memos with an action button beyond dismiss. E.g., "Request emergency budget: -2 rep, +50000 funds." Adds `ACTION { label = ... ; triggerId = ... }` block to `DIRECTOR_MEMO`.
- **Directives browser** — desk subpanel listing KCK-triggered directive scenes the player can replay or initiate on demand.
- **`DIRECTOR_STORY_CONTRACT`** cfg — formal story-contract declaration replacing the v0.1 flag convention.
- **Additional building overlays** — R&D (Wernher), VAB/SPH (Gus), Astronaut Complex (Walt).

---

## Out of Scope (v0.1 and v0.2)

- Stock building 3D asset replacement (overlays sit on top of stock scenes).
- Localization (all text is cfg; translation is content's concern).
- Multiplayer / shared state.
- 3D portrait rendering via `KerbalInstructor`. Chips render color swatches or flat textures. Live 3D portraits happen in KDK scenes via ChatterBox.
- Runtime cfg reload. Cfg scanned once at `MainMenu`; changes require restart.
- Admin-mode toggle to switch between stock and KAK admin within a save. Opt-in is per-install via cfg.
