# BadgKatCareer Contract System — Complete Design

## Overview

Replace all third-party contract packs with a unified, self-contained contract system. The player installs BadgKatCareer and gets the complete contract experience — no other contract packs needed (except Contract Configurator as the framework).

**Design pillars:**
1. Probe-first exploration — probes scout, crew follows
2. Three playable paths from day one — rockets, planes, rovers
3. Lightweight life support with Snacks — not spreadsheet-y
4. Maintenance contracts — infrastructure feels alive
5. Kerbal progression matters — experience gates contracts
6. Off-world manufacturing — endgame arc
7. Discovery-driven progression — ResearchBodies gates which bodies get contracts

## Dependencies

ResearchBodies is a hard dependency (moves from `recommends` to `depends` in CKAN). It gates the exploration progression:
- **Known from start:** Kerbin, Mun, Minmus (ResearchBodies default)
- **Must discover via telescope:** Everything else (Duna, Eve, Jool system, outer planets)
- **Discovery → contracts:** CC's body queries only return discovered bodies, so contracts for a body appear naturally after telescope discovery

This gives telescopes real purpose and creates a discovery → probe → crew pipeline for every body.

Tarsier Space Technology stays in `recommends` — it provides better orbital telescopes (ModuleTrackBodies) that feed into ResearchBodies' discovery system, but ResearchBodies has its own basic infrared telescope part.

### ResearchBodies Integration

ResearchBodies provides its own 4 contract types for body discovery:
- `RB_SearchSkies` — ground observatory search
- `RB_ResearchBody` — ground observatory long-term research
- `RB_TelescopeSearchSkies` — orbital telescope search
- `RB_TelescopeResearchBody` — orbital telescope research

**We do NOT duplicate these.** ResearchBodies owns the discovery pipeline. Our contracts pick up after discovery.

**ProbeFlyby fix:** Currently uses `NextUnreachedBody()` which may return undiscovered bodies. Must switch to a query that only returns bodies ResearchBodies has already revealed. CC's body queries likely respect RB's discovery state (RB hides undiscovered bodies from the tracking station), but needs verification during implementation. Fallback: filter using `ReachedBodies()` or check RB's discovery API if exposed to CC expressions.

### Full Discovery → Exploration Pipeline

```
ResearchBodies contract (use telescope) → body discovered → appears in tracking station
    → BadgKat ProbeFlyby appears for that body
    → ProbeOrbit → ProbeLanding
        → CrewedOrbit → CrewedLanding
```

## Body Progression Gating

Each body follows its own independent chain once discovered:

- **Mun/Minmus:** Known from day one (ResearchBodies default), probe contracts available immediately
- **Other bodies:** Must be discovered via ResearchBodies telescope contracts first, then follow the chain
- **Gas giants (Jool):** No landing contracts (no surface), chain stops at orbit
- **Multiple bodies in parallel:** Player can work multiple body chains simultaneously
- **Flyby discovery:** ResearchBodies config has `discoveryByFlyby = 30` (30% chance) — if a probe stumbles into an undiscovered body's SOI, there's a chance it gets discovered automatically

## Contract Groups

### BadgKatExploration (exists)
Two-track exploration: probes go first (cheaper, one-way OK), crew follows (harder, more rewarding). Replaces stock ExplorationContract.

### BadgKatMissionControl (exists)
Maintenance and operations. Other contracts BUILD things, this pack MAINTAINS them. Engineers, pilots, and scientists all have role-specific contracts gated by experience level.

### BadgKatMilestones (NEW)
One-time achievement contracts for aviation and rover paths. These fire early in career and give the non-rocket paths a reason to exist from day one. All maxCompletions = 1.

### AnomalySurveyor (exists, bundled fork)
14 contracts for KSP easter eggs. Already working. No changes needed.

## Stock Contract Handling

### Disabled
| Type | Reason |
|------|--------|
| PartTest | Tedious, replaced by TestFlight |
| SurveyContract | Often impossible, replaced by BiomeSurvey |
| ExplorationContract | Replaced by BadgKatExploration |
| PlantFlag | Covered by CrewedLanding |
| CollectScience | Covered by BiomeSurvey and DeepSpaceSurvey |
| SatelliteContract | Covered by relay contracts |

### Kept
| Type | Reason |
|------|--------|
| TourismContract | Harmless income, Tourism Plus may replace later |
| RecoverAsset (rescue) | Gives free crew, feeds kerbal progression |
| ARMContract (asteroid) | Fun, unique gameplay, no overlap |
| GrandTour | Rare aspirational contract, no overlap |

## New Contracts to Build (21 total)

### Exploration Group — Early Progression (3 new contracts)

Split the early game into unmanned/crewed tracks with an upper atmosphere crewed gate.

#### BKEX_UnmannedSuborbital (replaces BKEX_ReachSpace)
- **Trigger:** After FirstScience
- **Goal:** Reach suborbital space with an unmanned vessel. Probes go first.
- **Detection:** ReachState SUB_ORBITAL + HasCrew maxCrew = 0
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~30,000 funds, 5 rep
- **Requirement:** CompleteContract BKEX_FirstScience
- **Note:** Replaces the old ReachSpace contract. Now explicitly unmanned — probes first philosophy starts here.

#### BKEX_UnmannedOrbit
- **Trigger:** After UnmannedSuborbital
- **Goal:** Place an unmanned vessel in stable orbit around homeworld.
- **Detection:** HasCrew maxCrew = 0 + Orbit HomeWorld with minAltitude = 0
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~45,000 funds, 8 rep
- **Requirement:** CompleteContract BKEX_UnmannedSuborbital

#### BKEX_CrewedUpperAtmo
- **Trigger:** After FirstScience (available alongside UnmannedSuborbital)
- **Goal:** Fly a crewed vessel above 18,000m in atmosphere. The "we're putting a kerbal on a rocket" gateway — not quite space, but higher than planes can easily reach.
- **Detection:** HasCrew minCrew = 1 + ReachState FLYING with minAltitude = 18000
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~20,000 funds, 5 rep
- **Requirement:** CompleteContract BKEX_FirstScience

#### BKEX_CrewedSuborbital
- **Trigger:** After UnmannedSuborbital + CrewedUpperAtmo
- **Goal:** Send a crew to suborbital space and return them safely.
- **Detection:** HasCrew minCrew = 1 + ReachState SUB_ORBITAL + ReturnHome
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~50,000 funds, 10 rep
- **Requirement:** CompleteContract BKEX_UnmannedSuborbital + CompleteContract BKEX_CrewedUpperAtmo

#### BKEX_CrewedHomeOrbit (exists — update gates)
- **Update:** Add requirements for UnmannedOrbit + CrewedSuborbital
- **Existing gates stay:** probe orbit of homeworld
- **New gates:** CompleteContract BKEX_UnmannedOrbit + CompleteContract BKEX_CrewedSuborbital

**Early game exploration flow:**
```
FirstScience
    ├── UnmannedSuborbital ──→ UnmannedOrbit ──→ ProbeFlyby ──→ ...
    │       │                       │
    │       ▼                       ▼
    └── CrewedUpperAtmo ──→ CrewedSuborbital ──→ CrewedHomeOrbit ──→ ...
```

### Milestones Group — Aviation (9 contracts)

All aviation milestones are one-time achievements (maxCompletions = 1). They require a winged vessel (PartValidation with ModuleLiftingSurface) unless noted. G-Effects mod makes all speed/maneuver contracts organically harder via blackout mechanics without us needing to detect G-forces.

#### BKML_FirstFlight
- **Trigger:** Day one (available immediately alongside FirstScience)
- **Goal:** Fly a vessel with wings in atmosphere
- **Detection:** VesselParameterGroup with PartValidation (ModuleLiftingSurface) + ReachState FLYING
- **Prestige:** Trivial
- **maxCompletions:** 1
- **Rewards:** ~8,000 funds, 3 rep

#### BKML_SoundBarrier
- **Trigger:** After completing FirstFlight
- **Goal:** Exceed Mach 1 (343 m/s) in atmosphere
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING with minSpeed = 343
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~20,000 funds, 5 rep
- **Requirement:** CompleteContract BKML_FirstFlight

#### BKML_Mach3
- **Trigger:** After completing SoundBarrier
- **Goal:** Exceed Mach 3 (1029 m/s) in atmosphere. SR-71 territory.
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING with minSpeed = 1029
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~35,000 funds, 8 rep
- **Requirement:** CompleteContract BKML_SoundBarrier

#### BKML_Hypersonic
- **Trigger:** After completing Mach3
- **Goal:** Exceed Mach 5 (1715 m/s) in atmosphere. Bleeding edge — basically a spaceplane.
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING with minSpeed = 1715
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~50,000 funds, 10 rep
- **Requirement:** CompleteContract BKML_Mach3

#### BKML_IslandHop
- **Trigger:** After completing FirstFlight
- **Goal:** Fly to the Island Airfield and land (~50km from KSC)
- **Detection:** WaypointGenerator (fixed waypoint at island airfield) + VisitWaypoint (500m) + ReachState LANDED
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~20,000 funds, 5 rep
- **Requirement:** CompleteContract BKML_FirstFlight
- **Note:** Waypoint at approximately -1.52, -71.97. 500m radius — land on the runway.

#### BKML_DessertRun
- **Trigger:** After completing IslandHop
- **Goal:** Fly to the Dessert Airfield and land (~900km from KSC). Requires Making History.
- **Detection:** WaypointGenerator (fixed waypoint at Dessert Airfield) + VisitWaypoint (500m) + ReachState LANDED
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~30,000 funds, 6 rep
- **Requirement:** CompleteContract BKML_IslandHop, NEEDS[MakingHistory]

#### BKML_WoomerangDash
- **Trigger:** After completing DessertRun
- **Goal:** Fly to the Woomerang Launch Site and land (~1200km from KSC). Requires Making History.
- **Detection:** WaypointGenerator (fixed waypoint at Woomerang) + VisitWaypoint (500m) + ReachState LANDED
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~40,000 funds, 8 rep
- **Requirement:** CompleteContract BKML_DessertRun, NEEDS[MakingHistory]

#### BKML_HighSpeedLowPass
- **Trigger:** After completing SoundBarrier
- **Goal:** "Buzz the tower" — exceed Mach 2 (686 m/s) below 1000m altitude. G-Effects makes this spicy.
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING with minSpeed = 686, maxAltitude = 1000
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~25,000 funds, 5 rep
- **Requirement:** CompleteContract BKML_SoundBarrier

#### BKML_EdgeOfSpace
- **Trigger:** After completing SoundBarrier
- **Goal:** Push an aircraft to its ceiling — fly above 18,000m while still in atmosphere (FLYING, not SUB_ORBITAL)
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING with minAltitude = 18000
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~30,000 funds, 6 rep
- **Requirement:** CompleteContract BKML_SoundBarrier

### Milestones Group — SSTO (4 contracts)

The natural evolution of the aviation path. Planes don't dead-end — they graduate into spaceplanes. All require wings (ModuleLiftingSurface).

#### BKML_SuborbitalSpaceplane
- **Trigger:** After completing Hypersonic or EdgeOfSpace
- **Goal:** Reach space with a winged vessel and return. The bridge from aviation to spaceflight.
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState SUB_ORBITAL + ReturnHome
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~45,000 funds, 8 rep
- **Requirement:** CompleteContract BKML_Hypersonic OR CompleteContract BKML_EdgeOfSpace (Any group)

#### BKML_OrbitalSpaceplane
- **Trigger:** After completing SuborbitalSpaceplane
- **Goal:** Orbit Kerbin with a winged vessel and land safely.
- **Detection:** PartValidation (ModuleLiftingSurface) + Orbit HomeWorld + ReturnHome
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~60,000 funds, 12 rep
- **Requirement:** CompleteContract BKML_SuborbitalSpaceplane

#### BKML_SpaceplaneDocking
- **Trigger:** After completing OrbitalSpaceplane + having a station
- **Goal:** Fly a winged vessel to orbit and dock with a station. The "reusable resupply vehicle" concept.
- **Detection:** PartValidation (ModuleLiftingSurface) + Orbit HomeWorld + Docking with station
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~80,000 funds, 15 rep
- **Requirement:** CompleteContract BKML_OrbitalSpaceplane + HasStation (Expression)

#### BKML_LaytheSSTO
- **Trigger:** After completing OrbitalSpaceplane + having reached Laythe
- **Goal:** Fly a winged vessel in Laythe's atmosphere. The ultimate aviation achievement. The contract people screenshot.
- **Detection:** PartValidation (ModuleLiftingSurface) + ReachState FLYING on Laythe
- **Prestige:** Exceptional
- **maxCompletions:** 1
- **Rewards:** ~120,000 funds, 25 rep, 15 science
- **Requirement:** CompleteContract BKML_OrbitalSpaceplane + Reached Laythe
- **Note:** Does not require return — getting there is achievement enough. But could add optional ReturnHome bonus parameter.

### Milestones Group — Rover (2 contracts)

#### BKML_FirstRover
- **Trigger:** Day one (available immediately)
- **Goal:** Drive a wheeled vehicle on the surface
- **Detection:** PartValidation (ModuleWheelMotor) + ReachState LANDED + minSpeed = 1
- **Prestige:** Trivial
- **maxCompletions:** 1
- **Rewards:** ~8,000 funds, 3 rep

#### BKMC_RoverSurvey
- **Trigger:** After landing on a non-homeworld body
- **Goal:** Drive a rover to two waypoints in different biomes on a body, collect science at each
- **Detection:** PartValidation (ModuleWheelMotor) + VisitWaypoint (two RANDOM_WAYPOINTs) + CollectScience at each
- **Prestige:** Significant
- **maxCompletions:** 0 (repeatable)
- **Rewards:** ~30,000-60,000 funds, 5-10 rep, 8 science
- **Requirement:** LandedBodies (non-homeworld) >= 1
- **Cooldown:** 45 days
- **Note:** Two waypoints placed in different biomes via WaypointGenerator. Player drives between them collecting science. This is the rover's ongoing purpose — surface exploration missions.

### MissionControl Group — Relay Deployment (2 contracts)

#### BKMC_FirstRelay
- **Trigger:** After reaching orbit of homeworld
- **Goal:** Deploy a relay satellite in Kerbin orbit
- **Detection:** VesselParameterGroup with VesselIsType Relay + Orbit around HomeWorld + HasCrew maxCrew = 0
- **Prestige:** Significant
- **maxCompletions:** 1
- **Rewards:** ~30,000 funds, 5 rep
- **Requirement:** Orbit of HomeWorld achieved
- **Note:** Uses VesselIsType = Relay to check player has marked vessel as relay type. KSP prompts this naturally.

#### BKMC_RelayConstellation
- **Trigger:** After completing FirstRelay, for any body you've orbited
- **Goal:** Deploy 3 relay satellites around a target body
- **Detection:** Expression checking AllVessels().Where(v => v.VesselType() == Relay && v.CelestialBody() == targetBody).Count() >= 3
- **Prestige:** Significant
- **maxCompletions:** 0 (repeatable per body)
- **uniquenessCheck:** CONTRACT_ACTIVE
- **Rewards:** ~60,000 funds, 8 rep
- **Requirement:** CompleteContract BKMC_FirstRelay
- **Gate:** This is the prereq for RelayRepair and RelayUpgrade (those require 3+ relays already)

### MissionControl Group — Science (1 contract)

#### BKMC_BiomeSurvey
- **Trigger:** After landing on a non-homeworld body
- **Goal:** Collect a specific experiment from a specific biome on a body you've landed on
- **Detection:** CollectScience with experiment and biome parameters + situation SrfLanded
- **Prestige:** Trivial
- **maxCompletions:** 0 (repeatable)
- **Rewards:** ~15,000-30,000 funds, 2-5 rep, 5 science
- **Requirement:** LandedBodies count >= 1 (non-homeworld), Scientist level 1+
- **Cooldown:** 30 days
- **Implementation:** Uses DATA nodes to select a random body from LandedBodies, then a random biome on that body, then a random experiment. Contract reads like: "Collect temperature data from the Mun's Highlands." CC supports biome selection via `targetBody.Biomes()` and experiment via experiment ID lists.
- **Experiment pool:** temperatureScan, barometerScan, seismicScan, gravityScan, evaReport, surfaceSample, mysteryGoo, scienceExperiment (materials bay). Filter by what's unlocked via TechResearched requirements or just let CollectScience handle it (player can't collect what they haven't unlocked).

## Existing Contracts (no changes needed)

### Exploration Group (5 existing + 4 new = 11 contracts, after ReachSpace replaced)
| Contract | Phase | Gate | Status |
|----------|-------|------|--------|
| BKEX_FirstScience | Day 1 | Before first launch | exists |
| BKEX_UnmannedSuborbital | Day 1 | After FirstScience | NEW (replaces ReachSpace) |
| BKEX_UnmannedOrbit | Early | After UnmannedSuborbital | NEW |
| BKEX_CrewedUpperAtmo | Day 1 | After FirstScience | NEW |
| BKEX_CrewedSuborbital | Early | After UnmannedSuborbital + CrewedUpperAtmo | NEW |
| BKEX_CrewedHomeOrbit | Early-Mid | After UnmannedOrbit + CrewedSuborbital | exists (updated gates) |
| BKEX_ProbeFlyby | Early | After UnmannedOrbit + body discovered | exists |
| BKEX_ProbeOrbit | Early-Mid | After flyby of target body | exists |
| BKEX_ProbeLanding | Mid | After orbiting target body | exists |
| BKEX_CrewedOrbit | Mid | After probe orbit of target body, pilot 1+ | exists |
| BKEX_CrewedLanding | Mid-Late | After probe landing on target, pilot 2+ | exists |

### MissionControl Group (13 existing + 4 new = 17 contracts)
| Contract | Role Gate | Phase | Status |
|----------|-----------|-------|--------|
| BKMC_FirstRelay | None | Early | NEW |
| BKMC_RelayConstellation | None | Early-Mid | NEW |
| BKMC_TestFlight | Pilot 1+ | Mid | exists |
| BKMC_PrecisionLanding | Pilot 2+ | Mid-Late | exists |
| BKMC_StationMaintenance | Any Engineer | Mid | exists |
| BKMC_RelayRepair | Engineer 1+ | Mid | exists |
| BKMC_RelayUpgrade | Engineer 2+ | Mid-Late | exists |
| BKMC_EmergencyRepair | Engineer 3+ | Late | exists |
| BKMC_DeepSpaceSurvey | Scientist 2+ | Mid-Late | exists |
| BKMC_BiomeSurvey | Scientist 1+ | Mid | NEW |
| BKMC_RoverSurvey | None | Mid | NEW |
| BKMC_StationResupply | Any | Mid | exists |
| BKMC_CrewRotation | Any | Mid | exists |
| BKMC_BaseManufacturing | Engineer | Late | exists |
| BKMC_BaseResupply | Any | Late | exists |
| BKMC_LegacyMission | Veteran 3+ & Rookie | Late | exists |
| BKMC_VeteranRetirement | Level 4+ | Late | exists |

### Anomaly Surveyor (14 contracts)
No changes. Already bundled and working.

## Career Progression Flow

```
DAY ONE (fresh career)
│
├── ROCKETS (unmanned track):
│   FirstScience ──→ UnmannedSuborbital ──→ UnmannedOrbit
│                                               │
│   ROCKETS (crewed track):                ProbeFlyby (Mun/Minmus known)
│   FirstScience ──→ CrewedUpperAtmo           │
│                        │                 ProbeOrbit
│                    CrewedSuborbital           │
│                        │                 ProbeLanding
│                    CrewedHomeOrbit            │
│                        │              ┌──────┘
│                    CrewedOrbit ←───────┤ (gates behind probe orbit)
│                        │              │
│                    CrewedLanding ←─────┘ (gates behind probe landing)
│
├── AVIATION:
│   FirstFlight ──→ SoundBarrier ──→ Mach3 ──→ Hypersonic
│       │                │                        │
│       ├── IslandHop    ├── HighSpeedLowPass      │
│       │   → DessertRun ├── EdgeOfSpace ──────────┤
│       │     → WoomerangDash                      │
│       │                              SuborbitalSpaceplane
│       │                                  → OrbitalSpaceplane
│       │                                      → SpaceplaneDocking
│       │                                      → LaytheSSTO (endgame)
│
├── ROVERS:
│   FirstRover ──→ RoverSurvey (repeatable, after landing elsewhere)
│
├── DISCOVERY:
│   ResearchBodies telescope ──→ body discovered ──→ ProbeFlyby for that body
│   (Mun/Minmus known from start, everything else must be discovered)
│
EARLY-MID (first orbits)
├── FirstRelay ──→ RelayConstellation
│                      │
│                      ▼
│              RelayRepair ──→ RelayUpgrade
│
MID GAME (stations, infrastructure)
├── StationMaintenance          BiomeSurvey (specific experiment + biome)
├── StationResupply             RoverSurvey (waypoint-to-waypoint science)
├── CrewRotation                DeepSpaceSurvey
├── TestFlight                  PrecisionLanding
│
LATE GAME (bases, legacy)
├── BaseManufacturing
├── BaseResupply
├── EmergencyRepair
├── LegacyMission
└── VeteranRetirement

THROUGHOUT
└── Anomaly Surveyor (14 easter egg contracts)
    Stock: Rescue, Tourism, Asteroid, GrandTour
```

## Total Contract Count

| Group | Existing | New | Total |
|-------|----------|-----|-------|
| BadgKatExploration | 7 | 4 | 11 |
| BadgKatMilestones | 0 | 15 | 15 |
| BadgKatMissionControl | 13 | 4 | 17 |
| AnomalySurveyor | 14 | 0 | 14 |
| **Total** | **34** | **23** | **57** |

Plus stock: Rescue, Tourism, Asteroid, GrandTour.

Note: ReachSpace is replaced by UnmannedSuborbital (not counted as both).

## Cleanup Tasks

1. **ContractTweaks.cfg** — Remove all 14 GAP marketplace patches (GAP is uninstalled). Add disables for PlantFlag, CollectScience, SatelliteContract.
2. **Milestones group** — New file `ContractPacks/Milestones/Milestones.cfg` with CONTRACT_GROUP + AGENT definition.
3. **Exploration early game** — Replace ReachSpace with UnmannedSuborbital, add UnmannedOrbit, CrewedUpperAtmo, CrewedSuborbital. Update CrewedHomeOrbit gates.
4. **CKAN update** — Move ResearchBodies from `recommends` to `depends`.
5. **Aviation milestones** — 9 new files: FirstFlight, SoundBarrier, Mach3, Hypersonic, IslandHop, DessertRun, WoomerangDash, HighSpeedLowPass, EdgeOfSpace.
4. **SSTO milestones** — 4 new files: SuborbitalSpaceplane, OrbitalSpaceplane, SpaceplaneDocking, LaytheSSTO.
5. **Rover milestones + contracts** — FirstRover (milestone), RoverSurvey (repeatable, in MissionControl).
6. **Relay deployment** — 2 new files: FirstRelay, RelayConstellation (in MissionControl).
7. **Biome survey** — 1 new file: BiomeSurvey (in MissionControl, robust version with experiment + biome targeting).

## File Structure (after implementation)

```
BadgKatCareer/
├── BadgKatCareer.ckan
├── ContractTweaks.cfg
├── EarlyExploration.cfg
├── SnacksIntegration.cfg
├── ContractPacks/
│   ├── Milestones/
│   │   ├── Milestones.cfg              (NEW — group + agent)
│   │   ├── FirstFlight.cfg             (NEW)
│   │   ├── SoundBarrier.cfg            (NEW)
│   │   ├── Mach3.cfg                   (NEW)
│   │   ├── Hypersonic.cfg              (NEW)
│   │   ├── IslandHop.cfg               (NEW)
│   │   ├── DessertRun.cfg              (NEW — NEEDS MakingHistory)
│   │   ├── WoomerangDash.cfg           (NEW — NEEDS MakingHistory)
│   │   ├── HighSpeedLowPass.cfg        (NEW)
│   │   ├── EdgeOfSpace.cfg             (NEW)
│   │   ├── SuborbitalSpaceplane.cfg    (NEW)
│   │   ├── OrbitalSpaceplane.cfg       (NEW)
│   │   ├── SpaceplaneDocking.cfg       (NEW)
│   │   ├── LaytheSSTO.cfg              (NEW)
│   │   ├── FirstRover.cfg              (NEW)
│   │   └── RoverSurvey.cfg             (NEW — repeatable)
│   ├── Exploration/
│   │   ├── Exploration.cfg
│   │   ├── FirstSteps.cfg              (updated — ReachSpace → UnmannedSuborbital)
│   │   ├── EarlyCrewed.cfg             (NEW — CrewedUpperAtmo, CrewedSuborbital)
│   │   ├── UnmannedOrbit.cfg           (NEW)
│   │   ├── ProbeExploration.cfg
│   │   └── CrewedExploration.cfg       (updated — CrewedHomeOrbit gates)
│   ├── MissionControl/
│   │   ├── MissionControl.cfg
│   │   ├── FirstRelay.cfg              (NEW)
│   │   ├── RelayConstellation.cfg      (NEW)
│   │   ├── RelayRepair.cfg
│   │   ├── RelayUpgrade.cfg
│   │   ├── EmergencyRescueRepair.cfg
│   │   ├── StationMaintenance.cfg
│   │   ├── StationResupply.cfg
│   │   ├── CrewRotation.cfg
│   │   ├── BaseManufacturing.cfg
│   │   ├── BaseResupply.cfg
│   │   ├── BiomeSurvey.cfg             (NEW — robust biome+experiment)
│   │   ├── TestFlight.cfg
│   │   ├── PrecisionLanding.cfg
│   │   ├── DeepSpaceSurvey.cfg
│   │   ├── LegacyMission.cfg
│   │   └── VeteranRetirement.cfg
│   └── AnomalySurveyor/
│       └── (14 existing contracts, no changes)
```

## Open Questions / Risks

1. **Making History detection:** DessertRun and WoomerangDash need `NEEDS[SquadExpansion]` or similar to gate on Making History DLC. Need to verify the exact folder name MM recognizes for Making History.

2. **FirstRover driving detection:** ReachState LANDED + ModuleWheelMotor + minSpeed = 1. The minSpeed check might be finicky (rover needs to be actively moving when checked). Alternative: just validate wheels + landed and trust the player.

3. **BiomeSurvey biome selection:** CC supports `targetBody.Biomes()` to get available biomes. Need to verify the DATA node syntax for selecting a random biome and passing it to CollectScience. The experiment pool should filter naturally — players can't collect what they haven't unlocked.

4. **RoverSurvey waypoint biomes:** WaypointGenerator RANDOM_WAYPOINT can be placed, but ensuring two waypoints land in different biomes is not guaranteed. May need to just place two distant waypoints and accept they might be same biome — the driving is the point.

5. **Airfield coordinates:** Need to verify exact lat/long for all three destinations:
   - Island Airfield: approximately -1.52, -71.97
   - Dessert Airfield: approximately 0 (need to verify)
   - Woomerang: approximately 0 (need to verify)
   All use 500m VisitWaypoint radius.

6. **SSTO "wings" detection:** ModuleLiftingSurface is on wing parts but also on some fairings and nose cones. This is acceptable — if someone reaches orbit with just a fairing providing lift, that's impressive enough to count.

7. **LaytheSSTO Laythe detection:** Need to verify CC can check if Laythe has been reached. May use `ReachedBodies().Contains(Laythe)` or similar expression. Need to check if CC recognizes body names directly or needs CelestialBody lookup.
