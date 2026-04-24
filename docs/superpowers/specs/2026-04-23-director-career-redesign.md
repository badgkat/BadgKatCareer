# KerbalAdminKit + BadgKatCareer — Admin Building Redesign

Replace the stock Administration Building with a character-driven program management layer. This spec covers two mods: a generic admin-building-customizer toolkit (**KerbalAdminKit**, KAK) and the specific campaign content that uses it (**BadgKatCareer**).

**Supersedes:** [`2026-04-17-admin-building-design.md`](2026-04-17-admin-building-design.md). Original spec was written before KerbalDialogueKit shipped and KerbalCampaignKit was designed; most of its data model and effect classes have moved downstack.

**Note on naming:** This spec previously referred to the framework mod as `BadgKatDirector`. During implementation brainstorming the scope was honestly reframed as "admin-building customizer" and the mod renamed to `KerbalAdminKit` to match the `Kerbal*Kit` stack convention. Sections below use `KerbalAdminKit` / `KAK` throughout. Older cfg nodes like `DIRECTOR_CHARACTER`, `DIRECTOR_FOCUS`, `DIRECTOR_MEMO`, `DIRECTOR_BUILDING_SCENE`, `DIRECTOR_PR_CAMPAIGN`, `DIRECTOR_NOTIFICATION_STYLE`, and `DIRECTOR_KSC_MARKER_OFFSET` keep their names for continuity (content authoring stays stable across the rename). A new top-level `KERBAL_ADMIN_KIT` settings block is introduced for mod-level toggles.

**Detailed KAK design:** [`2026-04-23-admin-kit-design.md`](2026-04-23-admin-kit-design.md). This spec provides the stack overview and BadgKatCareer content plan; the KAK spec covers framework internals.

## The Four-Mod Stack

```
BadgKatCareer ─→ KerbalAdminKit ─→ KerbalCampaignKit ─→ KerbalDialogueKit ─→ ChatterBox
             └─→ KerbalCampaignKit (contract gating)
             └─→ KerbalDialogueKit (contract dialogue)
```

| Mod | Responsibility |
|---|---|
| **KerbalDialogueKit** (shipped v0.1.0) | Dialogue rendering, flags, choice capture, scene definitions |
| **KerbalCampaignKit** (shipped v0.1.0) | Chapters, event→action triggers, reputation economy, notification data API, CC REQUIREMENT types |
| **KerbalAdminKit** (Phase 2) | Admin-building UI replacement, character / focus / memo registries, per-building character overlays, KSC notification renderer, PR campaign action, disposition decay cfg convenience. Inert without content cfg; opt-in admin takeover via `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }`. |
| **BadgKatCareer** (Phase 3) | Campaign-specific content: 6 characters, 5-act chapter map, focuses, memos, scenes, rep tuning, chapter-gating on existing contracts |

See [`2026-04-23-campaign-kit-design.md`](2026-04-23-campaign-kit-design.md) for the CampaignKit spec and [`2026-04-23-admin-kit-design.md`](2026-04-23-admin-kit-design.md) for the KAK spec.

## Scope Shift From Original Spec

What the original spec put in the framework mod but we've moved downstack:

| Original concept | Now lives as |
|---|---|
| `ChoiceHistory` | KDK flags (choices auto-persist as flags) |
| `milestoneChoices[]` | CampaignKit chapter history + KDK choice flags |
| `activeFocuses{}` | KDK flags (e.g., `wernher_focus`) |
| `contractsOriginated[]` | Triggers that spawn contracts; no separate registry |
| Effect classes (ContractWeightModifier, FundInjection, etc.) | CampaignKit trigger actions |
| StoryReleaseRequirement | CampaignKit's `FlagEquals` CC REQUIREMENT |
| DirectiveContract | Cfg pattern: trigger + KDK scene with choice; no special contract type |
| Reputation economy math | CampaignKit (KAK just reads API for display) |
| "Effect layer" architecture | Collapses into CampaignKit's trigger action types |

What stays in KAK:

1. **UI framework** — opt-in admin building replacement, per-building character overlays, KSC notification renderer
2. **Character registry** — `DIRECTOR_CHARACTER` cfg, disposition flag conventions
3. **Focus registry** — `DIRECTOR_FOCUS` cfg, picker UI
4. **Memo pipeline** — `DIRECTOR_MEMO` cfg with game-state conditions, optional stock-mail mirroring
5. **Disposition decay** — `DISPOSITION_DECAY` cfg that compiles to CampaignKit time-triggers
6. **PR campaign action** — specific gameplay action tied to admin desk

KAK's C# codebase is ~60% smaller than the original Director spec implied, and activates lazily so an install without content cfg leaves stock KSP untouched.

---

# Part 1: KerbalAdminKit (Framework)

KAK is a standalone mod that customizes the KSP Administration Building when content mods supply the cfg for it. Full framework detail — activation model, cfg formats, UI layout, per-building overlays, KSC notification renderer, disposition decay compiler, save structure, build layout, theming, v0.2 expansion — lives in [`2026-04-23-admin-kit-design.md`](2026-04-23-admin-kit-design.md).

Key framework properties (content authors need to know):

- **Inert without content.** Install KAK with no `DIRECTOR_*` cfg and nothing changes. Stock admin, stock MessageSystem, stock KSC view all work normally.
- **Opt-in admin takeover.** Content must set `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }` for KAK to intercept the admin click. Without it, KSC markers / overlays / stock-mail memos activate but admin stays stock.
- **Per-feature degradation.** UI sections render only if relevant cfg is present. Missing `DIRECTOR_CHARACTER` cfg → no character panel. Missing `DIRECTOR_MEMO` → no memo section. Etc.
- **Chip rendering is themeable and layered.** `DIRECTOR_CHARACTER` can stack one or more `CHIP_LAYER { textureUrl = ... }` blocks for composed chips (e.g., background frame + portrait + corner badge). Without any layers, chip falls back to a color-swatch. Building overlays can override the chip stack for facility-specific looks. KSC notification glyphs are skinnable via `DIRECTOR_NOTIFICATION_STYLE`.
- **Memos can mirror to stock mail** via `postToStockMail = true` so the player gets mailbox pings mid-flight.

Everything a content mod hooks into:

| cfg node | purpose |
|---|---|
| `KERBAL_ADMIN_KIT` | mod-level toggles (replaceAdminBuilding, deskMemoCount, memoPollSeconds) |
| `DIRECTOR_CHARACTER` | character registry entry (+ optional portrait, dispositions) |
| `DIRECTOR_FOCUS` | focus registry entry |
| `DIRECTOR_MEMO` | memo definition with conditions |
| `DIRECTOR_BUILDING_SCENE` | per-building overlay chip |
| `DIRECTOR_NOTIFICATION_STYLE` | KSC marker glyph skin |
| `DIRECTOR_KSC_MARKER_OFFSET` | per-building marker pixel offset |
| `DIRECTOR_PR_CAMPAIGN` | PR Campaign button configuration |
| `DISPOSITION_DECAY` | compiles to KCK time-triggers at load |

All detail lives in the KAK spec. The rest of *this* spec focuses on what BadgKatCareer ships on top.

---

# Part 2: BadgKatCareer (Content)

All story-specific content. Additive to the existing mod — contracts stay; new KAK-facing content files added under `BadgKatCareer/DirectorContent/` (folder name preserved for cfg path consistency; no rename needed).

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

v0.1 ships color-swatch chips (no PNG portraits). Portrait artwork can be added later by dropping PNGs into `BadgKatCareer/Art/` and adding `CHIP_LAYER` blocks to each `DIRECTOR_CHARACTER` (e.g., a shared frame background + per-character portrait + optional badge). Existing character colors remain the fallback fill color when no layers are defined.

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

Nine directives total across the five chapters (some chapters have mid-chapter directives triggered by contract completions, not just on entry):

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

High-signal memos (low funds, low rep, crew ready) set `postToStockMail = true` so the player sees them mid-flight via the mailbox as well.

## Staff Meeting Scenes

~10-15 KDK scenes in `BadgKatCareer/DirectorContent/StaffMeetings/`. Triggered by CampaignKit triggers on events like first successful orbit, first crew return, major failure, each chapter transition (in addition to the directive scene).

## Reputation Config

`BadgKatCareer/DirectorContent/Reputation.cfg` ships:

```cfg
REPUTATION_INCOME
{
    enabled = true
    intervalDays = 30
    TIER { min = 0;   max = 100;  income = 5000 }
    TIER { min = 100; max = 250;  income = 15000 }
    TIER { min = 250; max = 500;  income = 35000 }
    TIER { min = 500; max = 750;  income = 60000 }
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

(Uses multi-line KSP cfg syntax in the real file; inline `{ k = v }` is illustrative.)

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

DIRECTOR_BUILDING_SCENE
{
    facility = Administration
    character = walt
    onClickScene = walt_admin_pr
}
```

These scenes can vary by chapter via `visibleIf` — a single `onClickScene` id references a scene whose lines conditionally fire based on current chapter.

## Admin Activation

`BadgKatCareer/DirectorContent/AdminKitSettings.cfg` sets the opt-in flag:

```cfg
KERBAL_ADMIN_KIT
{
    replaceAdminBuilding = true
    deskMemoCount = 5
    memoPollSeconds = 5
}
```

Players who install BadgKatCareer get the full admin-replacement experience. Players who install only KAK (without BadgKatCareer or another content mod that sets the flag) keep stock admin.

---

# Phasing

- **Phase 0 (shipped)** — KerbalDialogueKit v0.1.0
- **Phase 1 (shipped)** — KerbalCampaignKit v0.1.0 (spec in `2026-04-23-campaign-kit-design.md`). Headless, usable independently.
- **Phase 2** — KerbalAdminKit v0.1.0 (spec in `2026-04-23-admin-kit-design.md`). Character/focus/memo registries, opt-in admin building UI, Mission Control + Tracking Station overlays, KSC notification renderer, PR campaign action, disposition decay compiler.
- **Phase 3** — BadgKatCareer content integration. 6 characters, 5 chapters, 9 directives, 18 focuses, 20 memos, 10-15 staff meetings, rep config, chapter-gate REQUIREMENTs added to existing contracts, select rep-stake tags, `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }`.
- **Phase 4** — KAK v0.2: `DIRECTOR_POLICY` for long-running modifiers, action memos, directives browser, R&D / VAB / SPH / Astronaut overlays + second-pass polish.

Each phase is a shippable increment. Phase 1 is usable by third-party mods without waiting for KAK. Phase 2 without Phase 3 gives a generic admin-customizer framework (no content; stock admin untouched without cfg). Phase 3 is the first version that feels like the intended BadgKatCareer experience.

## Scope Summary

**KerbalCampaignKit (shipped):**
- ~40 classes, single DLL
- Detailed in `2026-04-23-campaign-kit-design.md`

**KerbalAdminKit (Phase 2):**
- ~25-30 classes
- Cfg formats: `KERBAL_ADMIN_KIT`, `DIRECTOR_CHARACTER`, `DIRECTOR_FOCUS`, `DIRECTOR_MEMO`, `DIRECTOR_BUILDING_SCENE`, `DIRECTOR_NOTIFICATION_STYLE`, `DIRECTOR_KSC_MARKER_OFFSET`, `DIRECTOR_PR_CAMPAIGN`, `DISPOSITION_DECAY`
- No campaign content — pure framework
- Detailed in `2026-04-23-admin-kit-design.md`

**BadgKatCareer additions (Phase 3):**
- 1 `KERBAL_ADMIN_KIT` settings block
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

**Out of Scope (all phases):**
- Stock building 3D asset replacement (overlays sit on top of stock scenes)
- Localization (all text is cfg, translation is content's concern)
- Multiplayer/shared state (single-save, per-save state)
- Debug/admin tooling UI (deferred to post-v0.1)
- Live 3D portraits in KAK's own UI (ChatterBox handles 3D in dialogue popups; KAK uses flat chips)
- Runtime cfg reload (restart required to pick up cfg changes)
