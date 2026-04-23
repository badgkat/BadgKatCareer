# KerbalCampaignKit — Career State Machine & Event Wiring

A headless utility library that extends KerbalDialogueKit with a career-scale state machine: named narrative chapters, a general event→action trigger system, a reputation economy, and a hierarchical notification API. Designed for any mod building a story-driven career, not tied to BadgKatCareer.

## Vision

KerbalDialogueKit renders character-driven dialogue and stores flags. What it does not do is coordinate *when* scenes play, *what* narrative state the campaign is in, or *how* gameplay events cascade into story consequences. KerbalCampaignKit fills that gap.

The library is deliberately generic. It does not define chapters, does not assume a campaign structure, does not provide content. It provides three kinds of infrastructure:

- **Chapters:** A named narrative state machine with entry/exit conditions. Contracts and UI can gate on chapter state.
- **Triggers:** A general "when X happens, do Y" wiring fabric. X can be any KSP event or cross-mod signal; Y can enqueue a scene, set a flag, advance a chapter, adjust currencies, raise a notification, or publish a custom GameEvent.
- **Reputation economy:** Opt-in career funding mechanics — passive income by tier, decay, rep-minimum gates, mission-level stakes.
- **Notifications:** A hierarchical attention-marker API so building overlays and UI mods know what needs the player's attention.

CampaignKit is headless. It provides data and API; it does not render UI. A consumer mod (BadgKatDirector, or any other) renders the notification markers and dashboards over CampaignKit state.

## Dependencies

- KerbalDialogueKit >= 0.1.0 (flag store, scene enqueue, flag expression parser)
- ContractConfigurator >= 2.12.0 (for CC REQUIREMENT types)
- KSP 1.12.x modding libraries

Optional soft-dependencies used by specific trigger event types:

- ResearchBodies (for `BodyDiscovered` event)
- Any mod publishing custom `GameEvents` can be hooked via `GameEventTrigger`

---

## Core Concepts

### Chapter

A named narrative state. At any time exactly one chapter is active. Chapters advance through triggers. CC contracts and other mods can gate content on chapter state.

### Trigger

A rule: "when event *X* happens, if condition *Y* holds, execute actions *Z*." The central wiring primitive. Everything narrative in a consumer mod ends up expressed as triggers.

### Notification

An attention marker attached to a hierarchical target (e.g., `admin`, `admin.desk`, `character.wernher`). Severities: `info` (soft indicator) or `action` (prominent). The notification data is headless; renderers subscribe to add/clear events to draw markers.

### Reputation Economy

Three opt-in functions over the stock rep currency:

- **Passive income** — tiered monthly funds dispensed from current rep
- **Minimum gates** — CC REQUIREMENT; contracts unavailable below a threshold
- **Mission stakes** — per-contract rep risk/reward amounts

Any consumer can enable any subset via cfg. None are mandatory.

---

## Chapters

### Chapter Definition (Config)

```cfg
CAMPAIGN_CHAPTER
{
    id = 3
    name = Beyond the Mun
    description = The anomaly signals stretch past Kerbin's moons.

    ENTRY_TRIGGER = chapter_3_entry
    EXIT_TRIGGER  = chapter_3_exit  // optional; exit may be implicit via ENTRY to next
}
```

Chapter IDs can be numeric or strings — `3` and `act_interplanetary` are both valid. Consumer content decides the convention.

### ChapterScenario State

A `ScenarioModule` persists:

```cfg
SCENARIO
{
    name = CampaignKitScenario
    currentChapter = 3
    HISTORY
    {
        ENTERED { chapter = 1; timestamp = 0 }
        ENTERED { chapter = 2; timestamp = 500000 }
        ENTERED { chapter = 3; timestamp = 2800000 }
    }
}
```

History is append-only. Useful for "have we been in chapter X" checks.

### Chapter API (C#)

```csharp
CampaignKit.Chapters.Current;                // "3"
CampaignKit.Chapters.IsAtLeast("3");         // true
CampaignKit.Chapters.HasEntered("2");        // true
CampaignKit.Chapters.Advance("4");           // explicit advance, fires ChapterEntered event
```

`ChapterEntered` / `ChapterExited` fire as CampaignKit events that triggers can subscribe to.

### CC REQUIREMENT types

```cfg
REQUIREMENT { name = InChapter; type = InChapter; chapter = 3 }
REQUIREMENT { name = ChapterAtLeast; type = ChapterAtLeast; chapter = 3 }
```

Also provides generic flag-gating REQUIREMENTs (below).

---

## Triggers

### Trigger Definition (Config)

```cfg
CAMPAIGN_TRIGGER
{
    id = chapter_4_on_duna_flyby
    once = true                            // fire once per save (default: true)

    ON_EVENT
    {
        type = ContractComplete
        contract = BKEX_ProbeFlyby_Duna
    }
    // Multiple ON_EVENT blocks OR together

    WHEN
    {
        flagExpression = chapter == 3 && duna_directive_fired != true
    }

    ACTIONS
    {
        SET_FLAG { name = duna_directive_fired; value = true }
        ADVANCE_CHAPTER { target = 4 }
        ENQUEUE_SCENE
        {
            sceneId = directive_duna_approach
            when = OnFacilityEnter
            facility = Administration
        }
        NOTIFY
        {
            target = admin
            severity = action
            source = chapter_4_directive
            clearOn = SceneEnded
            clearSceneId = directive_duna_approach
        }
    }
}
```

### Event Source Types

| Type | Parameters |
|---|---|
| `ContractComplete` | `contract` (CC type name) |
| `ContractAccepted` | `contract` |
| `ContractFailed` | `contract` |
| `ContractCancelled` | `contract` |
| `VesselRecovered` | `vesselType?` (Probe/Base/Ship/Rover/etc.) |
| `VesselCrashed` | `vesselType?` |
| `KerbalKilled` | `kerbalName?`, `trait?` |
| `KerbalLeveled` | `trait?`, `level?` |
| `BodyDiscovered` | `body?` (ResearchBodies soft-dep) |
| `FlagChanged` | `name`, `value?` |
| `ReputationCrossed` | `threshold`, `direction` (Rising\|Falling) |
| `FundsCrossed` | `threshold`, `direction` |
| `ScienceCrossed` | `threshold` |
| `FacilityUpgraded` | `facility` |
| `FacilityEntered` | `facility` |
| `ChapterEntered` | `chapter?` |
| `ChapterExited` | `chapter?` |
| `TimeElapsed` | `days`, `ref` (Startup\|LastTrigger\|LastEvent) |
| `SceneEnded` | `sceneId?` |
| `SceneChoiceMade` | `sceneId?`, `choiceId?`, `value?` |
| `GameEvent` | `eventName` (arbitrary `GameEvents` hook) |

### Action Types

| Type | Parameters |
|---|---|
| `SET_FLAG` | `name`, `value` |
| `CLEAR_FLAG` | `name` |
| `ENQUEUE_SCENE` | `sceneId`, `when?` (Immediate\|OnFacilityEnter), `facility?` |
| `ADVANCE_CHAPTER` | `target?` (omit for "next defined") |
| `ADJUST_FUNDS` | `amount`, `reason?` |
| `ADJUST_REPUTATION` | `amount`, `reason?` |
| `ADJUST_SCIENCE` | `amount`, `reason?` |
| `NOTIFY` | `target`, `severity`, `source`, `clearOn?`, `clearSceneId?` |
| `CLEAR_NOTIFICATION` | `target`, `source?` |
| `PUBLISH_EVENT` | `name`, arbitrary k=v params |

### Deferred Scene Enqueue

`ENQUEUE_SCENE { when = OnFacilityEnter; facility = Administration }` creates a pending scene record. When the player next enters the target facility, CampaignKit enqueues the scene to KDK.

Pending scenes persist across save/load in `CampaignKitScenario`. A single trigger firing can create a pending scene that doesn't play until hours later of real time — the player does other missions, loads and reloads — and still plays correctly on first admin entry.

### Trigger Evaluation

- Events fire synchronously from KSP GameEvents, ContractConfigurator callbacks, or KDK callbacks.
- All registered triggers listening to that event evaluate their `WHEN` expression.
- Matching triggers execute actions in cfg order.
- `once = true` triggers mark themselves "fired" in save state after first execution.
- Multiple triggers can fire from one event; ordering is cfg-declaration order (stable).

### Requirement Expressions

`WHEN { flagExpression = ... }` reuses KDK's `FlagExpressionParser`. Full operator set: `==`, `!=`, `<`, `>`, `<=`, `>=`, `in (...)`, `&&`, `||`, parens. Missing flags evaluate as empty string.

This means any state reachable as a flag — KDK flags, chapter state exposed as flags, disposition state exposed as flags, current rep as numeric flag — can gate a trigger.

CampaignKit mirrors structural state into well-known flags so expressions can reach it:

- `chapter` — current chapter id
- `reputation` — current rep value (numeric)
- `funds` — current funds value (numeric)
- `science` — current science value (numeric)
- `facility.<name>` — facility level (numeric, e.g., `facility.Administration == 3`)

Consumer mods should avoid stomping these names.

---

## Reputation Economy

All three functions are independent and opt-in. A consumer can use any subset.

### Passive Income

```cfg
REPUTATION_INCOME
{
    enabled = true
    intervalDays = 30

    TIER { min = 0;    max = 100;  income = 5000 }
    TIER { min = 100;  max = 250;  income = 15000 }
    TIER { min = 250;  max = 500;  income = 35000 }
    TIER { min = 500;  max = 750;  income = 60000 }
    TIER { min = 750;  max = 9999; income = 100000 }
}
```

If no `REPUTATION_INCOME` block is defined, no income is paid.

### Decay

```cfg
REPUTATION_DECAY
{
    enabled = true
    ratePercentPerMonth = 1.5
    resetOnContractComplete = true    // completing any contract resets decay timer
    tierFloors = true                 // never decay below highest tier reached
}
```

API exposed for consumer actions:

```csharp
CampaignKit.Reputation.HaltDecay(days: 60);   // for "PR campaign" style actions
CampaignKit.Reputation.NextGate();            // returns (value, description) or null
```

### Reputation Gate

CC REQUIREMENT:

```cfg
REQUIREMENT
{
    name = ReputationMinimum
    type = ReputationMinimum
    value = 300
}
```

Optional and transparent — contract description gets "(requires 300 reputation)" suffix automatically if the requirement is present.

### Reputation Stake

Per-contract opt-in:

```cfg
CONTRACT_TYPE
{
    name = BKEX_CrewedDunaLanding
    ...
    BEHAVIOUR
    {
        name = ReputationStake
        type = ReputationStake
        successBonus = 50
        failurePenalty = 100
    }
}
```

Applied on contract completion/failure in addition to stock rep rewards.

---

## Notifications

### Target Hierarchy

Targets are dot-separated strings:

- `admin` — admin building
- `admin.desk` — desk panel in admin building
- `admin.desk.contracts` — contracts section of desk
- `tracking` — tracking station
- `tracking.signals` — signals panel in tracking
- `character.wernher` — any surface showing Wernher

Convention: the first segment is the building or domain, subsequent segments are panels/sub-surfaces.

Queries support prefix matching:

```csharp
CampaignKit.Notifications.At("admin");           // exact matches only
CampaignKit.Notifications.AtOrBelow("admin");    // "admin" + "admin.*" descendants
```

### Severities

- **Info** — soft indicator. "Something's new here but not urgent."
- **Action** — prominent marker. "Player decision needed."

Renderers aggregate: if any `action` notification is at-or-below a target, show action marker; else if any `info`, show info marker.

### Auto-Clearing

```cfg
NOTIFY
{
    target = admin
    severity = action
    source = chapter_4_directive
    clearOn = SceneEnded
    clearSceneId = directive_duna_approach
}
```

`clearOn` options:

- `Manual` (default) — content must `CLEAR_NOTIFICATION` explicitly
- `FacilityEntered` — clear when player enters the target's root facility
- `SceneEnded` — clear when `clearSceneId` ends in KDK
- `FlagSet` — clear when `clearFlag` is set to `clearFlagValue`

### API (C#)

```csharp
CampaignKit.Notifications.Add(target: "admin", severity: Action, source: "x");
CampaignKit.Notifications.Clear(target: "admin", source: "x");
CampaignKit.Notifications.ClearAll("admin");         // clears all at "admin" and descendants
CampaignKit.Notifications.HasAny("admin");           // bool
CampaignKit.Notifications.Highest("admin");          // returns Info/Action/None
```

### GameEvents

Renderers subscribe to:

- `CampaignKitEvents.OnNotificationAdded`
- `CampaignKitEvents.OnNotificationCleared`

Also general state-change events for UI refresh:

- `CampaignKitEvents.OnChapterChanged`
- `CampaignKitEvents.OnReputationIncomeTick`

---

## Generic CC REQUIREMENT Types

Beyond chapter and rep requirements, CampaignKit provides generic flag-gating:

```cfg
REQUIREMENT { type = FlagEquals;    name = duna_approach; value = science_heavy }
REQUIREMENT { type = FlagNotEquals; name = wernher_disposition; value = Frustrated }
REQUIREMENT { type = FlagExpression; expression = "chapter == 3 && duna_directive_fired != true" }
```

`FlagExpression` is the general form — reuses KDK's expression parser. Other types are shorthand.

These replace the original spec's `StoryReleaseRequirement` — a story contract is "released" by setting a flag in a trigger; the contract gates on `FlagEquals` against that flag.

---

## Library Structure

Single assembly: `KerbalCampaignKit.dll` in `GameData/KerbalCampaignKit/Plugins/`.

```
KerbalCampaignKit/
├── Core/
│   ├── CampaignKit.cs              // Public static API entry point
│   ├── CampaignKitScenario.cs      // Save-persisted state
│   ├── CampaignKitAddon.cs         // KSPAddon — wires KSP events, pending scene playback
│   └── StateMirror.cs              // Mirrors chapter/rep/funds/science/facility levels to flags
│
├── Chapters/
│   ├── Chapter.cs                  // Data: id, name, entryTriggerId
│   ├── ChapterManager.cs           // Current chapter, advance/rollback
│   └── ChapterLoader.cs            // Parse CAMPAIGN_CHAPTER cfg
│
├── Triggers/
│   ├── Trigger.cs                  // Data: id, events, when, actions, once
│   ├── TriggerLoader.cs            // Parse CAMPAIGN_TRIGGER cfg
│   ├── TriggerEngine.cs            // Event dispatch, dedup, action execution
│   ├── Events/
│   │   ├── IEventSource.cs
│   │   ├── ContractEventSource.cs
│   │   ├── VesselEventSource.cs
│   │   ├── KerbalEventSource.cs
│   │   ├── CurrencyEventSource.cs
│   │   ├── FacilityEventSource.cs
│   │   ├── TimeEventSource.cs
│   │   ├── ChapterEventSource.cs
│   │   ├── FlagEventSource.cs
│   │   ├── SceneEventSource.cs
│   │   └── GameEventSource.cs
│   └── Actions/
│       ├── IAction.cs
│       ├── SetFlagAction.cs
│       ├── EnqueueSceneAction.cs
│       ├── AdvanceChapterAction.cs
│       ├── AdjustCurrencyAction.cs
│       ├── NotifyAction.cs
│       ├── ClearNotificationAction.cs
│       └── PublishEventAction.cs
│
├── Reputation/
│   ├── ReputationEconomy.cs        // Income tick, decay, gate/stake API
│   ├── ReputationIncomeLoader.cs   // Parse REPUTATION_INCOME cfg
│   ├── ReputationDecayLoader.cs    // Parse REPUTATION_DECAY cfg
│   └── ReputationStakeBehaviour.cs // CC BEHAVIOUR
│
├── Notifications/
│   ├── Notification.cs             // Target, severity, source, clearOn
│   ├── NotificationStore.cs        // Hierarchical add/clear/query
│   ├── AutoClearWatcher.cs         // Listens for SceneEnded, FacilityEntered, FlagSet
│   └── CampaignKitEvents.cs        // GameEvents publisher
│
├── Contracts/
│   ├── InChapterRequirement.cs     // CC REQUIREMENT
│   ├── ChapterAtLeastRequirement.cs
│   ├── ReputationMinimumRequirement.cs
│   ├── FlagEqualsRequirement.cs
│   ├── FlagNotEqualsRequirement.cs
│   └── FlagExpressionRequirement.cs
│
└── Config/
    └── ConfigNodeAdapter.cs        // Abstraction over ConfigNode for testability
```

Roughly 40 classes, ~4,000 lines. Most classes are small (one event source, one action, one requirement).

---

## Save Structure

```cfg
SCENARIO
{
    name = CampaignKitScenario

    currentChapter = 3

    CHAPTER_HISTORY
    {
        ENTERED { chapter = 1; timestamp = 0 }
        ENTERED { chapter = 2; timestamp = 500000 }
        ENTERED { chapter = 3; timestamp = 2800000 }
    }

    FIRED_TRIGGERS
    {
        trigger = chapter_1_startup
        trigger = chapter_2_on_first_orbit
    }

    PENDING_SCENES
    {
        SCENE { sceneId = directive_duna_approach; facility = Administration; fromTrigger = chapter_4_on_duna_flyby }
    }

    NOTIFICATIONS
    {
        NOTIFICATION { target = admin; severity = Action; source = chapter_4_directive; clearOn = SceneEnded; clearSceneId = directive_duna_approach }
    }

    REPUTATION_STATE
    {
        lastIncomeTime = 1900000
        lastDecayTime  = 1890000
        decayHaltUntil = 1950000
        highestTierReached = 3
    }
}
```

Flag state itself lives in KDK's `FlagScenario` — CampaignKit doesn't duplicate it.

---

## Integration Points

### With KerbalDialogueKit

- `ENQUEUE_SCENE` actions call `DialogueKit.EnqueueById`
- `SET_FLAG` / `CLEAR_FLAG` actions call `DialogueKit.Flags.Set`/`Remove`
- Flag expressions reuse `FlagExpressionParser`
- `SceneEnded`, `SceneChoiceMade` events subscribe to KDK's scene callbacks

### With ContractConfigurator

- CC REQUIREMENT types registered on assembly load
- `ReputationStake` contract BEHAVIOUR
- `ContractComplete`/`ContractAccepted`/`ContractFailed`/`ContractCancelled` event sources subscribe to CC's `OnContractCompleted` etc. events

### With ResearchBodies (soft)

- `BodyDiscovered` event source uses reflection or soft-reference to subscribe to ResearchBodies' discovery GameEvent. No hard dependency — consumers can still use the event type; it just never fires if ResearchBodies isn't installed.

### With Any Mod Using GameEvents

- `GameEvent` event source type subscribes to an arbitrary GameEvents name. Useful for music mods, other utility mods publishing custom events.

---

## Scope Summary

- **Single library DLL:** `KerbalCampaignKit.dll`
- **Public API surface:** ~15 classes in `CampaignKit.*` static
- **Dependencies:** KDK (>=0.1.0), ContractConfigurator, KSP 1.12.x
- **Content:** None — infrastructure only
- **Consumers:** BadgKatDirector, BadgKatCareer, any future narrative career mod
- **License recommendation:** MIT (matches KDK, encourages reuse)

### Out of Scope for v0.1

- Built-in UI — entirely consumer's job
- Save-file branching / multi-path narrative graphs — achievable via triggers + flags but not a first-class concept
- Persistent cross-save state — chapters and flags are per-save
- Localization — all text pulled from cfg, translation is consumer's problem for v0.1

### Future (v0.2+)

- Debug/inspector UI (see triggers that fired, chapter history, pending scenes)
- Additional event sources as consumers request them
- Performance profiling if trigger count grows past a few hundred
