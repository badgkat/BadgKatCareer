> **⚠️ SUPERSEDED — 2026-04-23**
>
> This spec has been superseded by:
> - [`2026-04-23-campaign-kit-design.md`](2026-04-23-campaign-kit-design.md) — KerbalCampaignKit (new mod, extracted from the original Director design)
> - [`2026-04-23-director-career-redesign.md`](2026-04-23-director-career-redesign.md) — revised Director + Career spec
>
> **What changed:** KerbalDialogueKit shipped (v0.1.0) after this spec was written. With KDK's flag store, scene enqueue, and choice capture in hand, much of this spec's data model (ChoiceHistory, effect classes, milestone directive as a first-class concept, StoryReleaseRequirement) collapses into lower-level primitives. A new utility — `KerbalCampaignKit` — was factored out to own chapter state, event→action triggers, the reputation economy, and notification APIs. Director shrinks to UI + conventions over that stack.
>
> Retained for historical record. Do not implement from this document.

---

# BadgKatDirector + BadgKatCareer — Admin Building & Campaign Integration

Replace the stock Administration Building strategy system with a character-driven program management layer. This spec covers two mods: a generic framework (**BadgKatDirector**) and the specific campaign content that uses it (**BadgKatCareer**).

## Architecture: Three-Mod Split

The work splits into three independently valuable mods:

| Mod | Role | Content | Consumers |
|---|---|---|---|
| **KerbalDialogueKit** | Dialogue infrastructure | None (library) | Any mod wanting character dialogue |
| **BadgKatDirector** | Admin building + economy framework | Generic framework | Any career wanting character-driven admin |
| **BadgKatCareer** | Story, characters, contracts | Campaign content | The BadgKat playthrough |

Dependency chain:

```
BadgKatCareer ─┬─→ BadgKatDirector ─→ KerbalDialogueKit ─→ ChatterBox
               └─→ KerbalDialogueKit (directly, for contract dialogue)
```

**KerbalDialogueKit** is specified separately in `2026-04-18-dialogue-kit-design.md`. This spec focuses on the Director framework and Career content.

## Vision

The admin building is where you run your space program. Not a strategy picker, not a currency slider — a place where your staff argues about what to do next, where you make decisions that shape the campaign, and where story contracts originate before flowing to Mission Control.

Two layers of interaction:

- **Day-to-day (A-layer):** Soft, low-pressure. Set character focuses, read memos, pick up story contracts. Change anytime. Effects are gentle nudges on contract weights and crew bonuses.
- **Milestone directives (B-layer):** Weighty, permanent. At ~9 act transitions across the campaign, two characters argue and you pick a direction. Effects lock in for the chapter: unique contracts spawn, bonuses apply, the narrative branches. Both paths are fun — the choice is lateral, not "right vs wrong."

Target audience: same smart 12-year-old as the rest of BadgKatCareer. Management should feel like running a scrappy space program, not optimizing a spreadsheet.

## Dependencies

**BadgKatDirector:**
- KerbalDialogueKit (our dialogue library)
- ContractConfigurator >= 2.12.0
- CustomBarnKit (admin building UI, facility-level gating)
- KSP 1.12.x modding libraries

**BadgKatCareer:**
- BadgKatDirector
- KerbalDialogueKit
- All existing contract pack dependencies (ResearchBodies, Snacks, KerbinSideRemastered, etc.)

## Architecture Overview

Three-layer architecture within BadgKatDirector designed for iterative expansion:

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

# Part 1: BadgKatDirector (Framework)

Generic, content-free. Defines the system. A modder could ship their own cast and directives on top of this.

## Core Data Model

### Character State

Each character has persistent state in the save file:

| Field | Type | Purpose |
|---|---|---|
| `id` | string | Unique identifier (e.g., "gene", "wernher") |
| `displayName` | string | "Gene Kerman" |
| `disposition` | enum | Enthusiastic, Supportive, Neutral, Skeptical, Frustrated |
| `lastChoiceSided` | bool | Did the player back them in the last milestone directive? |
| `focusTag` | string (nullable) | Current A-layer focus assignment |
| `memory[]` | list | Notable choices this character cares about, for dialogue callbacks |

Character definitions (id, display name, portrait model, color) are loaded from `DIRECTOR_CHARACTER` cfg nodes supplied by consumer mods.

**Disposition is not morale.** It is a narrative state, not a number to optimize. It affects dialogue tone (via KerbalDialogueKit flag system) and which contracts a character advocates for. A Frustrated Wernher still does science — he grumbles about it and doesn't volunteer bonus missions. An Enthusiastic Gus proactively offers infrastructure contracts.

Disposition decays toward Neutral over time (measured in contracts completed, not real time). No death spiral — completing missions in a character's domain pulls them back. They are professionals.

### Choice History

```
ChoiceHistory
├── milestoneChoices[]     // B-layer: locked decisions at act transitions
│   ├── actId
│   ├── choiceId
│   ├── backedCharacter
│   └── timestamp
├── activeFocuses{}        // A-layer: current lightweight focuses
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

## The Reputation Economy

Reputation represents the public's confidence in the space program. Three functions, one number.

### Passive Income

Reputation drives passive income at regular intervals (~30 days, configurable). Supplements contract payouts as a steady funding source.

| Rep Range | Tier | Passive Income/Month | Feel |
|---|---|---|---|
| 0–100 | Startup | ~5,000 | Scraping by |
| 100–250 | Established | ~15,000 | Comfortable |
| 250–500 | Respected | ~35,000 | Well-funded |
| 500–750 | Prestigious | ~60,000 | Major program |
| 750+ | Legendary | ~100,000 | Money isn't the problem |

Numbers are illustrative — tuned in playtesting. The framework supports configurable tiers via cfg.

### Ambition Gates

Certain milestone directives and story contracts require minimum reputation. Always transparent — the framework provides a "next gate" API that consumer UIs display to the player:

> "We need at least 300 reputation before we can sell a crewed Duna mission to the public. We're at 260."

Gates are set generously — normal play reaches them naturally. They bite only after significant failures.

### Risk Stakes

High-profile missions stake reputation. Success gains; failure costs. The framework provides a `ReputationStake` effect that tags contracts with risk/reward amounts. Consumer content decides which contracts stake rep and how much.

- Risk scales with ambition
- Failures aren't game-ending but sting — income dips, may delay the next gate
- Recovery always possible through successful missions

### Reputation Decay

Reputation decays slowly over time (~1–2% per month) to prevent coasting. Encourages concurrent missions — while a Duna probe is in transit, the player runs aviation milestones, Mun anomalies, station resupply to stay relevant.

Safeguards:
- **Tier floors:** Once you cross a tier threshold, you can never decay below the previous tier's floor. Landing on Duna means you can't become a nobody.
- **Activity resets the clock:** Completing any contract resets the decay timer.
- **PR campaign action:** A consumer-defined action (e.g., Walt's "PR campaign" in BadgKatCareer) costs funds and halts decay for a period.

## The Admin Building UI

### Building Roles

The admin building and Mission Control have distinct purposes:

- **Admin Building:** "What are we doing next?" — narrative, choices, character scenes, story contract origination
- **Mission Control:** "What do I need to do?" — contract details, parameters, tracking, radiant contracts

Story contracts originate in the admin building. The player accepts them through character scenes. They appear in Mission Control as already-active for reference. Radiant contracts stay in Mission Control exclusively.

### UI Layout

```
┌──────────────┬──────────────────────────┬─────────────────┐
│  CHARACTERS  │     THE DASHBOARD        │   ON THE DESK   │
│              │                          │                 │
│  [Gene]      │  Program Status          │  Story Contracts│
│  Supportive  │  ───────────────         │  ─────────────  │
│              │  Current Chapter: 3      │  ☐ Mun Probe    │
│  [Wernher]   │                          │    (Wernher)    │
│  Enthusiastic│  Reputation: 312         │                 │
│              │  ████████░░ Respected    │  ☐ Mun Anomaly  │
│  [Gus]       │  Monthly Income: 32,000  │    (Linus)      │
│  Neutral     │  Next Gate: 400          │                 │
│              │                          │  Memos          │
│  [Mortimer]  │  Active Effects          │  ─────────────  │
│  Neutral     │  ───────────────         │  Gus: relay     │
│              │  • Science +15% (Mun)   │     aging       │
│  [Walt]      │  • Probe contracts +30%  │  Walt: need     │
│  Supportive  │                          │     a headline  │
│              │                          │                 │
│  [Linus]     │                          │                 │
│  Enthusiastic│                          │                 │
└──────────────┴──────────────────────────┴─────────────────┘
```

The UI is framework-provided. Content (character names, focus options, memos, directive text) comes from consumer cfg files.

### Three Interaction Modes

**1. Daily Briefing (A-layer, anytime visit)**

No pending events. The dashboard appears. The player can:

- **Check the board.** Current focuses and their effects.
- **Shift focus.** Click a character, pick from available focus options (defined by consumer content). Options change as the campaign progresses.
- **Read memos.** Short character notes generated from game state templates.
- **Pick up story contracts.** Unlocked story contracts appear on the desk. Clicking one triggers a short character pitch scene via KerbalDialogueKit. Accept to activate, or leave for next visit.

Focus effects are real but gentle: contract weight modifiers, small crew XP bonuses, minor cost adjustments. No wrong setting.

**2. Staff Meeting (triggered by game state)**

At thresholds defined by consumer content — returning from a major mission, discovering a body, completing an act — the admin building flags a pending meeting.

The meeting is a KerbalDialogueKit scene. At the end, one or more of:
- New focus options unlock
- A story contract appears on the desk
- Character dispositions shift
- Two characters propose competing next steps (soft A-layer effect)

No permanent choice required.

**3. Milestone Directive (B-layer)**

At act transitions (triggers defined by consumer content), a major scene plays. Two characters argue. The scene builds to a fork. KerbalDialogueKit's ChoiceOverlay presents cards:

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

Mechanical effects are shown transparently. Narrative effects (how characters treat you later, which dialogue variants fire) are hidden.

After choosing, a short closing scene plays. The backed character reacts positively. The opposed character reacts in-character.

Unchosen paths' contracts may still be available as radiant contracts in Mission Control — the player missed the narrative version, not the gameplay entirely.

## Effect Types

All effects implement `IDirectorEffect`. Phase 1 ships with:

### Contract Effects
- **ContractWeightModifier** — shift probability of contract types appearing
- **ContractRewardModifier** — multiply rewards for specific types/bodies
- **StoryContractSpawner** — create CC contracts for the admin building desk

### Economic Effects
- **FundInjection** — one-time funds bonus on choice
- **PartCostModifier** — adjust part costs by category
- **LaunchCostModifier** — adjust vessel rollout costs
- **ReputationIncome** — passive funding pipeline (always active)

### Crew Effects
- **CrewXPModifier** — trait-specific XP rate changes
- **CrewHireCostModifier** — hiring cost adjustments
- **TraitBonusModifier** — small stat bonuses for a trait

### Progression Effects (Phase 2)
- **EarlyPartAccess** — unlock specific parts before their tech node
- **ScienceMultiplier** — bonus science from specific situations/bodies

### Reputation Effects
- **ReputationStake** — tag high-profile missions with rep risk/reward
- **ReputationRecovery** — path back after failures

### Narrative Effects
- **DispositionShift** — move a character's disposition
- **DialogueBranch** — set KerbalDialogueKit flags (disposition-aware dialogue)
- **MemoTrigger** — queue a character memo for next admin visit

## C# Plugin Architecture (BadgKatDirector)

Single assembly: `BadgKatDirector.dll`.

```
BadgKatDirector/
├── Core/
│   ├── DirectorScenario.cs        // ScenarioModule — save/load, all state
│   ├── CharacterState.cs          // Per-character data
│   ├── CharacterConfigLoader.cs   // Loads DIRECTOR_CHARACTER cfg
│   ├── ChoiceHistory.cs           // Record of milestone decisions
│   ├── ReputationEconomy.cs       // Income, decay, gates, stakes
│   └── DirectorConfig.cs          // Static config from cfg files
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
│   ├── DeskPanel.cs               // Right: pending stories, memos
│   └── FocusSelector.cs           // Focus option picker
│
├── Contracts/
│   ├── StoryReleaseRequirement.cs // CC REQUIREMENT: admin released?
│   ├── DirectiveContract.cs       // Contract type for directives
│   └── ReputationGate.cs          // CC REQUIREMENT: min rep check
│
└── Narrative/
    ├── MemoGenerator.cs           // Game-state-driven character memos
    ├── StaffMeeting.cs            // Trigger and manage meeting scenes
    ├── MilestoneDirective.cs      // B-layer choice flow
    └── DispositionFlagSync.cs     // Syncs character dispositions to
                                   // KerbalDialogueKit flags
```

### Consumer-Facing Config Formats

Consumer mods define content through these cfg node types:

**Character definition:**
```cfg
DIRECTOR_CHARACTER
{
    id = wernher
    displayName = Wernher von Kerman
    portraitModel = Instructor_Wernher
    color = #FF82B4E8
    role = Chief Scientist
}
```

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

    // References a DIALOGUE_SCENE defined via KerbalDialogueKit
    scene = directive_duna_approach

    CHOICE
    {
        id = science_heavy
        backedCharacter = wernher
        opposedCharacter = gus
        EFFECT { type = ContractRewardModifier; body = Duna; scienceMultiplier = 1.4 }
        EFFECT { type = StoryContractSpawner; contract = BKEX_DunaFullScience }
        EFFECT { type = DispositionShift; character = wernher; disposition = Enthusiastic }
        EFFECT { type = DispositionShift; character = gus; disposition = Skeptical }
    }
    CHOICE { id = engineering_first; ... }
    CHOICE { id = compromise; ... }
}
```

**Focus definition:**
```cfg
DIRECTOR_FOCUS
{
    character = wernher
    id = deep_space_signals
    title = Deep Space Signals
    description = Wernher focuses on tracking the anomaly signals.

    REQUIREMENT
    {
        type = ContractComplete
        contract = BKEX_UnmannedOrbit
    }

    EFFECT { type = ContractWeightModifier; group = AnomalySurveyor; multiplier = 1.3 }
    EFFECT { type = ScienceMultiplier; situation = InSpaceLow; multiplier = 1.1 }
}
```

**Memo template:**
```cfg
DIRECTOR_MEMO
{
    character = gus
    id = relay_aging
    priority = low

    CONDITION
    {
        type = VesselAge
        vesselType = Relay
        minAge = 200
    }

    text = That relay is getting old. Might want to think about upgrades before something breaks.
}
```

### Facility-Level Gating (CustomBarnKit)

| Admin Level | Features |
|---|---|
| 1 | Basic focus system, story contract desk, memos |
| 2 | Staff meetings unlock, reputation dashboard visible |
| 3 | Full milestone directives, all effects available |

Gives the player a reason to upgrade the admin building early.

### Save File Structure

```cfg
SCENARIO
{
    name = BadgKatDirector
    scene = 7, 8, 5

    CHARACTER
    {
        id = gene
        disposition = Supportive
        lastChoiceSided = true
        focusTag = crew_safety
        MEMORY { choiceId = duna_directive; sided = true; timestamp = 1847293 }
    }
    // ... other characters

    CHOICE_HISTORY
    {
        MILESTONE
        {
            actId = 4
            choiceId = duna_directive
            backedCharacter = gus
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
        originatedBy = wernher
        released = false
    }
}
```

---

# Part 2: BadgKatCareer (Content)

All the story-specific content that plugs into BadgKatDirector. This is the existing mod — we add new content files but keep all existing contract packs and dialogue.

## The Cast

Six characters defined via `DIRECTOR_CHARACTER` nodes in `BadgKatCareer/DirectorContent/Characters.cfg`:

| Character | Domain | Natural Building | Portrait |
|---|---|---|---|
| Gene | Mission direction, crew safety | Administration, Mission Control | Instructor_Gene |
| Wernher | Science, anomaly research | R&D | Instructor_Wernher |
| Gus | Engineering, infrastructure | VAB/SPH | Strategy_MechanicGuy |
| Mortimer | Budget, finance | Administration | Strategy_Mortimer |
| Walt | PR, reputation, recruitment | Astronaut Complex | Strategy_PRGuy |
| Linus | Anomaly signals, tracking | Tracking Station, Observatory | Strategy_ScienceGuy |

Gene and Linus span two buildings each. Not every building needs a unique character — some have more authority than others.

## Milestone Directive Map

Nine hand-authored directives across the five-act campaign:

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

**The scene (in KerbalDialogueKit DIALOGUE_SCENE format):**

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
- Both → Supportive

## Character Focus Options

~3 focus options per character, evolving through the campaign. Stored in `BadgKatCareer/DirectorContent/Focuses.cfg`.

Examples:
- **Wernher:** Anomaly Research, Deep Space Signals, Material Science, Biological Sciences (late), Theoretical Physics (endgame)
- **Gus:** Launch Infrastructure, Surface Operations, Station Engineering, Interplanetary Vehicles (mid-game)
- **Walt:** Public Outreach, Media Relations, International Partnerships (late)
- **Mortimer:** Cost Optimization, Investor Relations, Austerity (under rep pressure)
- **Linus:** Signal Analysis, Body Discovery, Anomaly Cataloging
- **Gene:** Crew Safety, Mission Pacing, Program Direction

## Memo Templates

~20 game-state-driven character memos. Stored in `BadgKatCareer/DirectorContent/Memos.cfg`. Examples:

- Gus (relay aging): "That relay over Kerbin is getting old. Might want to think about upgrades before something breaks."
- Walt (low rep): "Boss, we need a headline. Public interest is fading."
- Mortimer (low funds): "We can't afford another launch this month unless we complete a contract."
- Wernher (new body discovered): "The spectrographic data from the telescope is fascinating. I'd love to send a probe."
- Linus (anomaly signal detected): "I'm picking up signals from the Mun. The pattern matches what we saw on Kerbin."

## Staff Meeting Scenes

~10–15 triggered scenes reacting to campaign progression. Defined as KerbalDialogueKit DIALOGUE_SCENE nodes in `BadgKatCareer/DirectorContent/StaffMeetings/`.

Triggers include:
- First successful orbit
- First crew return from space
- First body discovery
- First anomaly investigation complete
- Each act transition (in addition to directives)
- Major failures (crew loss, mission disaster)

## Reputation Economy Tuning

Tier thresholds and income rates tuned for BadgKatCareer's pacing:
- Act 1 typically finishes around rep 100–150 (Established tier)
- Act 2 at ~250 (Respected)
- Act 4 at ~500 (Prestigious)
- Endgame Kcalbeloh at 750+ (Legendary)

## Walt's PR Campaign Action

BadgKatCareer defines the rep decay counter-action: Walt's "PR Campaign" button in the admin building. Costs funds based on current tier, halts decay for 60 days, small rep bump.

---

## Integration With Existing Systems

### Career Overhaul Compatibility

The anomaly-driven career overhaul (separate spec, currently being implemented) provides the contract progression, ChatterBox dialogue on contracts, bridge contracts, and crew recruitment. This Director work layers on top:

- Story contracts gain `StoryRelease` REQUIREMENT — held from Mission Control until admin building releases them
- Contract-triggered ChatterBox scenes gain disposition-aware variant lines via KerbalDialogueKit flags
- Contract rewards become modifiable via `CurrencyModifierQuery`
- High-profile contracts gain `ReputationStake` metadata

Existing contract cfg files need minimal changes — additive REQUIREMENT blocks and optional metadata, no structural rewrites.

### Mod Compatibility

**Keep and hook into:**
- ContractConfigurator — contracts run on CC
- ChatterBox — contract dialogue stays on ChatterBox (KerbalDialogueKit extends it)
- CustomBarnKit — UI and facility gating
- ResearchBodies — Linus's focuses reference discovered bodies
- Snacks — untouched

**Replace:**
- Stock strategies — disabled via MM patch

**No new mod dependencies beyond the three-mod split.**

---

## Phasing Plan

### Phase 0 — KerbalDialogueKit (foundation)

See `2026-04-18-dialogue-kit-design.md`. Must ship before Director. Also useful to the career overhaul for contract dialogue improvements (branching, disposition variants).

### Phase 1 — BadgKatDirector Narrow + BadgKatCareer Initial Content

Core framework and enough content to be playable:

**BadgKatDirector:**
- `DirectorScenario` with save/load
- Character system loading from DIRECTOR_CHARACTER cfg
- `ReputationEconomy` (passive income, decay, gates)
- `AdminOverhaul` UI replacing stock admin building
- `StoryReleaseRequirement` for CC integration
- Focus system, staff meeting system, milestone directive system (frameworks only)
- Memo framework with condition types: VesselAge, ContractComplete, ReputationLevel, BodyDiscovered
- Story contract desk
- Stock strategy disabler
- Phase 1 effects: ContractWeightModifier, ContractRewardModifier, StoryContractSpawner, FundInjection, PartCostModifier, CrewXPModifier, DispositionShift, DialogueBranch, ReputationStake, ReputationIncome, MemoTrigger

**BadgKatCareer content:**
- Character definitions for the six
- All ~9 milestone directives with full scenes
- ~18 character focus options
- ~20 memo templates
- ~10–15 staff meeting scenes
- Walt's PR Campaign action config
- Reputation tier tuning for the campaign
- Integration REQUIREMENTs added to existing story contracts

**Produces:** A playable admin building that drives the BadgKatCareer campaign narrative with real mechanical effects.

### Phase 2 — Medium (effect types extend to other systems)

Same UI, same data model, new effect types in BadgKatDirector:
- `EarlyPartAccess` — unlock parts before tech nodes
- `ScienceMultiplier` — bonus science from situations/bodies
- `CrewHireCostModifier`, `TraitBonusModifier`
- `ReputationRecovery` (richer than Phase 1 PR campaign)
- Wernher's focus affects R&D costs
- Gus's focus affects part availability
- Linus's focus affects ResearchBodies discovery rates

### Phase 3 — Wide (characters in all buildings)

Same data model, same effects, new UI surfaces:
- Characters appear in "their" buildings (Gus in VAB, Wernher in R&D, Linus in Tracking Station)
- Per-building UI MonoBehaviours reading from shared character state
- Building-specific interactions
- Full KSC feels like visiting an organization

---

## Scope Summary

**BadgKatDirector (framework):**
- ~25 C# classes
- 4 config file types (DIRECTOR_CHARACTER, DIRECTOR_DIRECTIVE, DIRECTOR_FOCUS, DIRECTOR_MEMO)
- 2 new CC REQUIREMENT types (StoryRelease, ReputationGate)
- No content — pure framework

**BadgKatCareer (content additions):**
- 6 character definitions
- ~9 milestone directives + dialogue scenes
- ~18 focus option definitions
- ~20 memo templates
- ~10–15 staff meeting scenes
- Reputation tier tuning
- Integration REQUIREMENTs added to ~30 existing story contracts

**Phasing:**
- Phase 0 (KerbalDialogueKit): Foundation, ships first
- Phase 1 (Director narrow + Career content): Playable system
- Phase 2 (Medium effects): Extends reach
- Phase 3 (Wide UI): Characters in all buildings
