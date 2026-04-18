# BadgKatDirector — Administration Building Redesign

Replace the stock Administration Building strategy system with a character-driven program management layer. The six BadgKatCareer cast members live in the admin building, react to the player's decisions, and drive the campaign forward through dialogue, choices, and mechanical effects.

## Vision

The admin building is where you run your space program. Not a strategy picker, not a currency slider — a place where your staff argues about what to do next, where you make decisions that shape the campaign, and where story contracts originate before flowing to Mission Control.

Two layers of interaction:

- **Day-to-day (A-layer):** Soft, low-pressure. Set character focuses, read memos, pick up story contracts. Change anytime. Effects are gentle nudges on contract weights and crew bonuses.
- **Milestone directives (B-layer):** Weighty, permanent. At ~9 act transitions across the campaign, two characters argue and you pick a direction. Effects lock in for the chapter: unique contracts spawn, bonuses apply, the narrative branches. Both paths are fun — the choice is lateral, not "right vs wrong."

Target audience: same smart 12-year-old. Management should feel like running a scrappy space program, not optimizing a spreadsheet.

## Dependencies

- ContractConfigurator >= 2.12.0
- ChatterBox >= 1.0.0 (rendering pipeline, see Dialogue Strategy section)
- CustomBarnKit (admin building UI, facility-level gating)
- All BadgKatCareer contract packs
- Unity/KSP 1.12.x modding libraries

## Architecture Overview

Three-layer architecture designed for iterative expansion:

```
┌─────────────────────────────────────────┐
│  UI Layer                               │
│  Narrow: Admin building only            │
│  Medium: Effects reach R&D, tracking    │
│  Wide: Characters appear in all KSC     │
│         buildings                       │
├─────────────────────────────────────────┤
│  Effect Layer                           │
│  Modular IDirectorEffect classes        │
│  New effects = new classes, no rewrite  │
├─────────────────────────────────────────┤
│  Core Data Model                        │
│  Character states, choices, history     │
│  Persists in save file (SCENARIO)       │
│  Designed for full vision on day one    │
└─────────────────────────────────────────┘
```

Narrow ships first. Medium and wide add new effect types and UI surfaces on the same data model — no rewrites.

---

## Core Data Model

### Character State

Each of the six characters has persistent state in the save file:

| Field | Type | Purpose |
|---|---|---|
| `name` | string | Gene, Wernher, Gus, Mortimer, Walt, Linus |
| `disposition` | enum | Enthusiastic, Supportive, Neutral, Skeptical, Frustrated |
| `lastChoiceSided` | bool | Did the player back them in the last milestone directive? |
| `focusTag` | string (nullable) | Current A-layer focus assignment |
| `memory[]` | list | Notable choices this character cares about, for dialogue callbacks |

**Disposition is not morale.** It is a narrative state, not a number to optimize. It affects dialogue tone and which contracts a character advocates for. A Frustrated Wernher still does science — he grumbles about it and doesn't volunteer bonus missions. An Enthusiastic Gus proactively offers infrastructure contracts.

Disposition decays toward Neutral over time (measured in contracts completed, not real time). No death spiral — completing missions in a character's domain pulls them back. They are professionals.

### Character-Building Mapping

| Character | Domain | Natural Building |
|---|---|---|
| Gene | Mission direction, crew safety | Administration, Mission Control |
| Wernher | Science, anomaly research | R&D |
| Gus | Engineering, infrastructure | VAB/SPH |
| Mortimer | Budget, finance | Administration |
| Walt | PR, reputation, recruitment | Astronaut Complex |
| Linus | Anomaly signals, tracking | Tracking Station, Observatory |

Gene and Linus span two buildings each. Not every building needs a unique character — some characters have more authority than others.

### Choice History

```
ChoiceHistory
├── milestoneChoices[]     // B-layer: locked decisions at act transitions
│   ├── actId
│   ├── choiceId
│   ├── backedCharacter
│   └── timestamp
├── activeStances{}        // A-layer: current lightweight focuses
│   ├── key (e.g., "wernher_focus")
│   └── value (e.g., "deep_space_signals")
└── contractsOriginated[]  // story contracts spawned from admin building
```

### Effect Registry

All effects implement `IDirectorEffect`:

```
IDirectorEffect
├── OnChoiceMade(choice)
├── OnContractCompleted(contract)
├── ModifyContractWeight(contract) → float
├── ModifyReward(type, amount) → float
├── ModifyCrewBonuses(kerbal) → bonuses
└── GetAvailableContracts() → list
```

---

## The Reputation Economy

Reputation represents the public's confidence in the space program. Walt's obsession, Mortimer's funding source, the ceiling on the program's ambition.

### Three Functions, One Number

**1. Funding Pipeline (always on)**

Reputation drives passive income at regular intervals (~30 days, configurable). This supplements contract payouts as a steady funding source.

| Rep Range | Tier | Passive Income/Month | Feel |
|---|---|---|---|
| 0–100 | Startup | ~5,000 | Scraping by |
| 100–250 | Established | ~15,000 | Comfortable |
| 250–500 | Respected | ~35,000 | Well-funded |
| 500–750 | Prestigious | ~60,000 | Major program |
| 750+ | Legendary | ~100,000 | Money isn't the problem |

Numbers are illustrative — tuned in playtesting.

**2. Ambition Gate (at story beats)**

Certain milestone directives and story contracts require minimum reputation. Always transparent — Walt tells the player in the admin building:

> "We need at least 300 reputation before we can sell a crewed Duna mission to the public. We're at 260. A successful Mun investigation or a flashy first would put us over."

The player always knows: what the next gate is, where they stand, what actions would help. Gates are set generously — normal play reaches them naturally. They only bite after significant failures.

**3. Risk Budget (on ambitious missions)**

High-profile missions stake reputation. Before launch, the player sees the stakes. Success gains rep; failure costs it.

- Walt's assessment appears in briefing dialogue
- Risk scales with ambition (routine resupply stakes nothing, crewed Duna landing stakes a lot)
- Failures aren't game-ending but sting — income dips, may delay the next gate
- Recovery always possible through successful missions

### Reputation Decay

Reputation decays slowly over time (~1–2% per month) to prevent coasting. This encourages concurrent missions — while a Duna probe is in transit, the player runs aviation milestones, Mun anomalies, station resupply to stay relevant.

Safeguards:
- **Tier floors:** Once you cross a tier threshold, you can never decay below the previous tier's floor. Landing on Duna means you can't become a nobody.
- **Activity resets the clock:** Completing any contract resets the decay timer. Active programs don't decay.
- **Walt can help:** If rep is decaying, Walt offers a "PR campaign" action — costs funds, halts decay for a period. Spending money to stay relevant.

---

## The Admin Building Experience

### Building Roles

The admin building and Mission Control have distinct purposes:

- **Admin Building:** "What are we doing next?" — narrative, choices, character scenes, story contract origination
- **Mission Control:** "What do I need to do?" — contract details, parameters, tracking, radiant contracts

Story contracts (Tier 1, some Tier 2) originate in the admin building. The player accepts them through character scenes. They then appear in Mission Control as already-active for reference. Radiant contracts (station resupply, crew rotation, biome surveys) stay in Mission Control exclusively.

### UI Layout

```
┌──────────────┬──────────────────────────┬─────────────────┐
│  CHARACTERS  │     THE DASHBOARD        │   ON THE DESK   │
│              │                          │                 │
│  [Gene]      │  Program Status          │  Story Contracts│
│  Supportive  │  ───────────────         │  ─────────────  │
│              │  Act 3: "Something       │  ☐ Mun Probe    │
│  [Wernher]   │   on the Mun"           │    (Wernher)    │
│  Enthusiastic│                          │                 │
│              │  Reputation: 312         │  ☐ Mun Anomaly  │
│  [Gus]       │  ████████░░ Respected    │    (Linus)      │
│  Neutral     │  Monthly Income: 32,000  │                 │
│              │  Next Gate: 400          │                 │
│  [Mortimer]  │   (Crewed Interplanetary)│  Memos          │
│  Neutral     │                          │  ─────────────  │
│              │  Active Effects          │  Gus: relay     │
│  [Walt]      │  ───────────────         │     aging       │
│  Supportive  │  • Science +15% (Mun)   │  Walt: need     │
│              │  • Probe contracts +30%  │     a headline  │
│  [Linus]     │                          │                 │
│  Enthusiastic│                          │                 │
└──────────────┴──────────────────────────┴─────────────────┘
```

- **Left panel:** Character portraits with disposition shown as facial expression/animation
- **Center panel:** Active scene or status dashboard
- **Right panel:** Pending story contracts and character memos

### Three Interaction Modes

**1. Daily Briefing (A-layer, anytime visit)**

No pending events. The dashboard appears. The player can:

- **Check the board.** Current focuses and their effects.
- **Shift focus.** Click a character, pick from 2–3 contextual focus options. Options change as the campaign progresses — new ones unlock, old ones can disappear.
- **Read memos.** Short character notes generated from game state templates. Informational, not mandatory. Point toward available contracts and opportunities.
- **Pick up story contracts.** Unlocked story contracts appear on the desk. Clicking one triggers a short character pitch scene. Accept to activate, or leave it for next visit.

Focus effects are real but gentle: contract weight modifiers (+30% more of a type), small crew XP bonuses, minor cost adjustments. No wrong setting — just "what do you want more of right now."

**2. Staff Meeting (triggered by game state)**

At certain thresholds — returning from a major mission, discovering a body, completing an act milestone — the admin building flags a pending meeting (visual indicator on the KSC building).

The meeting is a ChatterBox scene: 2–4 characters, 4–8 lines, reacting to what just happened. At the end, one or more of:
- New focus options unlock
- A story contract appears on the desk
- Character dispositions shift
- Two characters propose competing next steps (soft A-layer effect, not a locked commitment)

No permanent choice required. These are the "your organization has opinions" moments.

**3. Milestone Directive (B-layer, ~9 across the campaign)**

At act transitions, a major scene plays. Two characters argue. The scene builds to a fork. The dialogue pauses and choice cards appear:

```
┌─────────────────────────────────────────────────┐
│  Gene: Your call, boss.                         │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Back Wernher: "Go big on science"      │    │
│  │  • Full Duna science lander mission     │    │
│  │  • +40% science from Duna biomes        │    │
│  │  • Wernher → Enthusiastic               │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Back Gus: "Prove the engineering"      │    │
│  │  • Duna rover + small lander mission    │    │
│  │  • -20% Duna vehicle costs              │    │
│  │  • Gus → Enthusiastic                   │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Compromise: "Do both, smaller"         │    │
│  │  • Medium lander with some science      │    │
│  │  • +15% science, -10% costs             │    │
│  │  • Both → Supportive                    │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
└─────────────────────────────────────────────────┘
```

**Mechanical effects are shown transparently.** The player knows exactly what they gain. Narrative effects (how characters treat you later, which dialogue variants fire) are hidden — that is the fun part.

After choosing, a short closing scene plays. The backed character reacts positively. The opposed character reacts in-character (Wernher sulks scientifically, Gus grumbles practically, Mortimer calculates the cost aloud).

**The other path is not destroyed.** Contracts from the unchosen path don't appear as story contracts, but some content may still be available as radiant contracts in Mission Control. The player missed the narrative version, not the gameplay entirely.

### Milestone Directive Map

| Act | Trigger | Characters | Choice Axis |
|---|---|---|---|
| 1 | Both surface leads complete | Gene vs Wernher | Push to space now vs investigate more on the ground |
| 2 | First orbit achieved | Mortimer vs Wernher | Conservative budget vs ambitious probe program |
| 2 | First crew in space | Gene vs Walt | Quiet professionalism vs media blitz |
| 3 | Mun monolith found | Wernher vs Gus | Study the monolith (science) vs build Mun infrastructure |
| 3 | Ready to go interplanetary | Gene vs Mortimer | Multiple targets vs focus resources on one |
| 4 | Duna/Eve probe data | Wernher vs Gus | Science-heavy vs engineering-first |
| 4 | First crewed interplanetary | Gene vs Walt | Low-profile safe mission vs high-profile showcase |
| 5 | Jool system reached | Wernher vs Linus | Systematic survey vs chase the anomaly signal |
| 5 | Kcalbeloh wormhole | Gene vs everyone | Go through (the whole staff argues) |

Each directive has 2–3 choices. Both/all paths are viable and fun — the choice shapes the texture of the playthrough, not its quality.

### Example Directive: "Going to Duna"

Act 4 transition. Probe data is in. Duna is the next target.

**The scene:**

> **Wernher:** The spectrographic data from our flyby is extraordinary. The signal is strongest near the northern basin. I want a full science package — surface experiments, atmospheric analysis, the works. We land heavy, we learn everything.
>
> **Gus:** Heavy means expensive. And we have never landed anything that big on another planet. I say we go lean — a rover and a small lander. Prove we can get there and back. Then we talk about the big science package.
>
> **Gene:** Your call, boss.

**Choice A — Back Wernher ("Go big on science"):**
- Story contracts: Full Duna science lander, then Duna anomaly investigation with heavy equipment
- Effects: +40% science from Duna biomes, anomaly contracts pay bonus science
- Wernher → Enthusiastic, Gus → Skeptical

**Choice B — Back Gus ("Prove the engineering first"):**
- Story contracts: Duna rover + small lander, then Duna base construction
- Effects: -20% Duna vehicle costs, infrastructure contracts appear for Duna
- Gus → Enthusiastic, Wernher → Skeptical

**Choice C — Compromise ("Do both, smaller"):**
- Story contracts: Medium lander with some science, no base
- Effects: +15% science, -10% costs
- Both → Supportive (not Enthusiastic)

---

## Effect Types

Effects are the mechanical consequences of choices. Each is a C# class implementing `IDirectorEffect`.

### Contract Effects
- **ContractWeightModifier** — shift probability of contract types appearing
- **ContractRewardModifier** — multiply rewards for specific types/bodies
- **StoryContractSpawner** — create CC contracts for the admin building desk

### Economic Effects
- **FundInjection** — one-time funds bonus on choice
- **PartCostModifier** — adjust part costs by category
- **LaunchCostModifier** — adjust vessel rollout costs
- **ReputationIncome** — passive funding pipeline (always active, scales with rep)

### Crew Effects
- **CrewXPModifier** — trait-specific XP rate changes
- **CrewHireCostModifier** — hiring cost adjustments
- **TraitBonusModifier** — small stat bonuses for a trait

### Progression Effects
- **EarlyPartAccess** — unlock specific parts before their tech node. Goes away if the effect is removed unless the player has since researched the node normally.
- **ScienceMultiplier** — bonus science from specific situations/bodies

### Reputation Effects
- **ReputationStake** — tag high-profile missions with rep risk/reward
- **ReputationRecovery** — path back after failures (Walt's PR campaign)

### Narrative Effects
- **DispositionShift** — move a character's disposition
- **DialogueBranch** — set flags that dialogue scenes check
- **MemoTrigger** — queue a character memo for next admin visit

---

## Dialogue Strategy

### The Problem

ChatterBox handles contract-triggered dialogue well but has limitations for admin building scenes:
- Tied to contract lifecycle (onAccept, onComplete, onFail) — admin scenes are not contract events
- No player choice capture — linear dialogue only, no branching mid-scene
- No callbacks — cannot chain "dialogue ends → effects apply"
- No branching — cannot select line variants by disposition within a single scene
- All key classes are `internal sealed` — cannot extend from another assembly

### The Approach

**Option 1 (preferred): PR to ChatterBox** making the rendering pipeline public.

Smallest useful change: make `ScenePopupDefinition`, `ScenePopupController.Enqueue()`, `CharacterConfig`, and `LineConfig` public. This lets any mod build a scene definition in code and queue it for display. ChatterBox's renderer handles portraits, dialogue, audio.

We build **BadgKatDialogue** as an extension layer on top:
- `DirectorScene` — higher-level wrapper building `ScenePopupDefinition` objects from .cfg, with branching, disposition-aware variants, and choice points
- `ChoiceOverlay` — custom UI rendering choice cards between dialogue segments
- `SceneCallback` — event hooks for "scene ended" / "choice made"

**Option 3 (fallback): Fork ChatterBox's rendering code.**

ChatterBox is MIT licensed. If the PR is rejected or slow, fork the rendering code (portrait system, dialogue sequencing, audio) into `BadgKatDialogue.dll`. Same API shape, but public and with extensions baked in.

Both options produce the same API from our plugin's perspective. `BadgKatDialogue` talks to a scene renderer — whether that is ChatterBox-with-public-API or our-own-fork is swappable.

**ChatterBox stays for contract dialogue.** The career overhaul's Tier 1/2 contract scenes continue using ChatterBox BEHAVIOUR nodes. Admin building scenes use BadgKatDialogue. The player does not notice — both show character portraits and dialogue.

---

## C# Plugin Architecture

Single assembly: `BadgKatDirector.dll` in `GameData/BadgKatCareer/Plugins/`.

### Class Structure

```
BadgKatDirector/
├── Core/
│   ├── DirectorScenario.cs        // ScenarioModule — save/load, all state
│   ├── CharacterState.cs          // Per-character data
│   ├── ChoiceHistory.cs           // Record of milestone decisions
│   ├── ReputationEconomy.cs       // Income, decay, gates, stakes
│   └── DirectorConfig.cs          // Static config from .cfg files
│
├── Effects/
│   ├── IDirectorEffect.cs         // Interface all effects implement
│   ├── ContractWeightModifier.cs
│   ├── ContractRewardModifier.cs
│   ├── StoryContractSpawner.cs
│   ├── FundInjection.cs
│   ├── PartCostModifier.cs
│   ├── CrewXPModifier.cs
│   ├── ReputationStake.cs
│   └── DispositionShift.cs
│
├── Admin/
│   ├── AdminOverhaul.cs           // KSPAddon — replaces admin building UI
│   ├── CharacterPanel.cs          // Left panel: portraits + dispositions
│   ├── ScenePanel.cs              // Center: scene or dashboard
│   ├── DeskPanel.cs               // Right: pending stories, effects
│   └── FocusSelector.cs           // Focus option picker
│
├── Contracts/
│   ├── StoryReleaseRequirement.cs // CC REQUIREMENT: admin building released?
│   ├── DirectiveContract.cs       // Contract type for milestone directives
│   └── ReputationGate.cs          // CC REQUIREMENT: minimum rep check
│
└── Narrative/
    ├── MemoGenerator.cs           // Game-state-driven character memos
    ├── StaffMeeting.cs            // Trigger and manage meeting scenes
    ├── MilestoneDirective.cs      // B-layer choice flow
    └── DialogueVariant.cs         // Disposition-aware dialogue flags
```

### Key Classes

**`DirectorScenario` (ScenarioModule):**
Central hub. Holds all `CharacterState` objects, `ChoiceHistory`, `ReputationEconomy`, pending/released story contracts, and active `IDirectorEffect` instances. Registers for game events: contract completion, vessel recovery, progress milestones, kerbal death.

**`ReputationEconomy`:**
Manages all three reputation functions:
- `GetMonthlyIncome(rep)` / `ProcessIncomeTick(time)` — passive funding
- `GetDecayRate(rep)` / `GetDecayFloor()` / `ProcessDecayTick(time)` — slow decay with tier floors
- `ResetDecayTimer()` — called on any contract completion
- `MeetsGate(gateId)` / `GetNextGate()` — ambition gating, what Walt reports
- `GetMissionStake(vessel)` — risk assessment for high-profile launches

**`AdminOverhaul` (KSPAddon, SpaceCentre):**
Intercepts the stock admin building screen (similar approach to Strategia's `AdminResizer`). Replaces content with three-panel character-driven layout. Checks for pending staff meetings or milestone directives on open.

**`StoryReleaseRequirement` (CC Requirement):**
Custom ContractConfigurator REQUIREMENT type added to story contracts:
```cfg
REQUIREMENT
{
    name = AdminRelease
    type = StoryRelease
    contractId = BKEX_ProbeFlyby_Duna
}
```
Returns false until the player accepts through the admin building. Keeps story contracts out of Mission Control until released.

**`MilestoneDirective`:**
Manages B-layer flow: detects trigger conditions → flags admin building → launches directive scene → captures choice → applies effects → records in history.

### Configuration (Data-Driven)

Directives, focuses, and memos are defined in .cfg files. The C# plugin provides the engine; the .cfg files provide the content.

**Directive definition:**
```cfg
DIRECTOR_DIRECTIVE
{
    id = duna_approach
    actId = 4

    TRIGGER
    {
        type = ContractComplete
        contract = BKEX_ProbeFlyby_Duna
    }

    scene = directive_duna_approach

    CHOICE
    {
        id = science_heavy
        backedCharacter = Wernher
        opposedCharacter = Gus

        EFFECT
        {
            type = ContractRewardModifier
            body = Duna
            scienceMultiplier = 1.4
        }
        EFFECT
        {
            type = StoryContractSpawner
            contract = BKEX_DunaFullScience
        }
        EFFECT
        {
            type = DispositionShift
            character = Wernher
            disposition = Enthusiastic
        }
        EFFECT
        {
            type = DispositionShift
            character = Gus
            disposition = Skeptical
        }
    }

    CHOICE
    {
        id = engineering_first
        backedCharacter = Gus
        opposedCharacter = Wernher
        // ... effects
    }

    CHOICE
    {
        id = compromise
        // ... effects
    }
}
```

**Focus definition:**
```cfg
DIRECTOR_FOCUS
{
    character = Wernher
    id = deep_space_signals
    title = Deep Space Signals
    description = Wernher focuses on tracking the anomaly signals.

    REQUIREMENT
    {
        type = ContractComplete
        contract = BKEX_UnmannedOrbit
    }

    EFFECT
    {
        type = ContractWeightModifier
        group = AnomalySurveyor
        multiplier = 1.3
    }
    EFFECT
    {
        type = ScienceMultiplier
        situation = InSpaceLow
        multiplier = 1.1
    }
}
```

**Memo template:**
```cfg
DIRECTOR_MEMO
{
    character = Gus
    id = relay_aging
    priority = low

    // Conditions use a simplified expression system evaluated by DirectorScenario.
    // Available queries: vessel counts by type, vessel age, body reach status,
    // contract completion status, reputation level, character disposition.
    CONDITION
    {
        type = VesselAge
        vesselType = Relay
        minAge = 200  // days
    }

    text = That relay is getting old. Might want to think about upgrades before something breaks.
}
```

### Facility-Level Gating (CustomBarnKit)

Admin building features unlock with facility upgrades:

| Level | Features |
|---|---|
| 1 | Basic focus system, story contract desk, memos |
| 2 | Staff meetings unlock, reputation dashboard visible |
| 3 | Full milestone directives, all effects available |

Gives the player a reason to upgrade the admin building early.

---

## Integration With Existing Systems

### Career Overhaul Compatibility

The career overhaul (anomaly-driven narrative, ChatterBox dialogue, contract progression) provides the foundation. The admin building system layers on top:

- Story contracts gain `StoryRelease` REQUIREMENT — held from Mission Control until the admin building releases them
- ChatterBox dialogue gains disposition-aware variant lines via `DialogueBranch` flags
- Contract rewards become modifiable via `CurrencyModifierQuery` (same API Strategia used)
- High-profile contracts gain `ReputationStake` metadata

Existing CC contracts need minimal changes. The integration points are additive REQUIREMENT blocks and optional metadata, not structural rewrites.

### Mod Compatibility

**Keep and hook into:**
- ContractConfigurator — contracts still run on CC
- ChatterBox — contract dialogue stays on ChatterBox; admin scenes use BadgKatDialogue extension
- CustomBarnKit — UI and facility gating
- ResearchBodies — Linus's focus options reference discovered bodies
- Snacks — untouched, still used for life support

**Replace:**
- Stock strategies — disabled via MM patch (same approach as Strategia's `StockStrategyDisabler.cfg`)

**No new mod dependencies required for narrow release.**

### Save File Structure

All state persists in a custom SCENARIO module:

```cfg
SCENARIO
{
    name = BadgKatDirector
    scene = 7, 8, 5

    CHARACTER
    {
        name = Gene
        disposition = Supportive
        lastChoiceSided = true
        focusTag = crew_safety
        MEMORY
        {
            choiceId = duna_directive
            sided = true
            timestamp = 1847293
        }
    }
    // ... other characters

    CHOICE_HISTORY
    {
        MILESTONE
        {
            actId = 4
            choiceId = duna_directive
            backedCharacter = Gus
            timestamp = 1847293
        }
    }

    REPUTATION_ECONOMY
    {
        lastIncomeTime = 1900000
        lastDecayTime = 1890000
        decayFloor = 250
        highestTierReached = 3
    }

    PENDING_STORIES
    {
        contractId = BKEX_ProbeFlyby_Duna
        originatedBy = Wernher
        released = false
    }
}
```

---

## Phasing Plan

### Phase 1 — Narrow (ships with career overhaul)

The admin building itself plus core systems:

- `DirectorScenario` with save/load
- `CharacterState` for all six characters
- `ReputationEconomy` (passive income, decay, gates)
- `AdminOverhaul` UI replacing stock admin building
- `StoryReleaseRequirement` for CC integration
- Character focus system (A-layer)
- Staff meetings (triggered ChatterBox scenes)
- Milestone directives (B-layer choices, ~9 across campaign)
- Memo system (game-state-driven character notes)
- Story contract desk (accept Tier 1/2 from admin building)
- Stock strategy disabler
- Effect types (Phase 1): ContractWeightModifier, ContractRewardModifier, StoryContractSpawner, FundInjection, PartCostModifier, CrewXPModifier, DispositionShift, DialogueBranch, ReputationStake, ReputationIncome, MemoTrigger
- BadgKatDialogue extension layer (ChatterBox PR or fork)

**Produces:** A playable admin building that drives the campaign narrative, with real mechanical effects from player choices.

### Phase 2 — Medium (extends effects to other systems)

Same UI, same data model, new effect types:

- `EarlyPartAccess` — unlock parts before tech nodes
- `PartCostModifier` — adjust part costs by category
- `ScienceMultiplier` — bonus science from situations/bodies
- `CrewXPModifier` — trait-specific XP rates
- `CrewHireCostModifier` — hiring cost adjustments
- `TraitBonusModifier` — crew stat bonuses
- Wernher's focus affects R&D costs
- Gus's focus affects part availability
- Linus's focus affects ResearchBodies discovery rates

### Phase 3 — Wide (characters in all buildings)

Same data model, same effects, new UI surfaces:

- Characters appear in "their" buildings (Gus in VAB, Wernher in R&D, Linus in Tracking Station)
- Per-building UI MonoBehaviours reading from shared character state
- Building-specific interactions (Gus comments on your rocket design, Wernher highlights interesting experiments)
- Full KSC feels like visiting an organization, not navigating menus

---

## Scope Summary

- **C# classes:** ~25 (core + effects + UI + contracts + narrative)
- **Config file types:** 3 (directives, focuses, memos)
- **Milestone directives:** ~9 hand-authored choice points
- **Character focus options:** ~3 per character, evolving with campaign progress (~18 total)
- **Memo templates:** ~15–20 game-state-driven
- **Staff meeting scenes:** ~10–15 triggered by progression
- **New CC REQUIREMENT types:** 2 (StoryRelease, ReputationGate)
- **Phases:** Narrow first (prerequisite), then Medium and Wide in either order
