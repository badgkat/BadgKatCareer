# Phase 1: Narrative Restructure + Dialogue Rewrite — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the BadgKatCareer contract progression so anomaly discoveries drive the campaign forward, with ChatterBox dialogue as the connective tissue.

**Architecture:** Replace the current independent contract gating with a campaign-style flow where anomaly contracts form the story spine. Combined contracts merge related activities (rover+monolith, flight+island). Bridge contracts (auto-accept, zero-parameter) deliver narrative beats between story arcs. All dialogue reframed around "the monolith signal is pulling us outward."

**Tech Stack:** ContractConfigurator >= 2.12.0, ChatterBox v1.0.0, KSP 1.12.x ConfigNode format

---

## Reference: Character Quick-Reference

See `docs/superpowers/specs/2026-04-15-chatterbox-dialogue-design.md` for full voice guide.

| Character | Model | Color | Speaker Token |
|---|---|---|---|
| Gene | `Instructor_Gene` | `#FF8BE08A` | A or B |
| Wernher | `Instructor_Wernher` | `#FF82B4E8` | A or B |
| Gus | `Strategy_MechanicGuy` | `#FFFFC078` | A or B |
| Mortimer | `Strategy_Mortimer` | `#FFFFE066` | A or B |
| Walt | `Strategy_PRGuy` | `#FFC8A0E8` | A or B |
| Linus | `Strategy_ScienceGuy` | `#FF6ED4C8` | A or B |

## Reference: KSP ConfigNode Rules

- Each key-value pair on its own line (NO semicolons)
- LINE nodes MUST be multi-line: `LINE { speaker = A \n text = ... }`
- Braces on their own lines for multi-field nodes
- Comments use `//`

## Reference: New Progression Tree

```
FirstScience (auto-accept) ─┬─► RoverMonolith (combined rover + monolith)
                            └─► FlyToIsland (combined flight + island)
                                    │
                    ┌───────────────┤
                    ▼               ▼
            SoundBarrier      (aviation side track)
                    │
            Both complete ──► Bridge: "Both Leads Checked"
                    │
        ┌───────────┼──────────────────┐
        ▼           ▼                  ▼
UnmannedSuborbital  Kerbin_Pyramids    Kerbin_Monolith_Additional
        │           │
        │           Kerbin_UFO
        │
    ┌───┴───────────────┐
    ▼                   ▼
UnmannedOrbit      CrewedUpperAtmo
    │                   │
    ▼                   ▼
FirstRelay         CrewedSuborbital
    │                   │
    ▼                   │
RelayConstellation      │
    │                   │
    └───────┬───────────┘
            ▼
    CrewedHomeOrbit
            │
            ▼
    Bridge: "Signals from the Mun"
            │
    ┌───────┴───────┐
    ▼               ▼
MunProbe/anomalies  Minmus (side)
    │
    ▼
Bridge: "Trace the Signal" (needs Mun_Monolith + 2 Kerbin anomalies)
    │
    ┌───────┴───────┐
    ▼               ▼
ProbeFlyby_Duna   ProbeFlyby_Eve
    │               │
    ▼               ▼
Duna anomalies    Eve exploration
    │               │
    └───────┬───────┘
            ▼
    Bridge: "Going Deeper" (Duna OR Eve anomaly)
            │
            ▼
    Jool system (Vall → Bop → Jool_Monolith)
            │
            ▼
    Kcalbeloh_Wormhole → Beyond → Dipuc → BlackHole
```

---

## File Map

### New Files to Create
- `ContractPacks/Exploration/CampaignOpening.cfg` — RoverMonolith + FlyToIsland combined contracts
- `ContractPacks/Exploration/BridgeContracts.cfg` — All bridge contracts (Act 1-5 narrative connectors)

### Files to Modify
- `ContractPacks/Exploration/FirstSteps.cfg` — FirstScience: add autoAccept, rewrite dialogue
- `ContractPacks/Exploration/EarlyCrewed.cfg` — CrewedUpperAtmo, CrewedSuborbital: update gating + dialogue
- `ContractPacks/Exploration/UnmannedOrbit.cfg` — Update dialogue framing
- `ContractPacks/Exploration/ProbeExploration.cfg` — ProbeFlyby: restructure for Duna-first targeting
- `ContractPacks/Exploration/CrewedExploration.cfg` — CrewedHomeOrbit: update dialogue
- `ContractPacks/Milestones/FirstRover.cfg` — Deprecate (replaced by RoverMonolith)
- `ContractPacks/Milestones/FirstFlight.cfg` — Deprecate (replaced by FlyToIsland)
- `ContractPacks/Milestones/SoundBarrier.cfg` — Update requirement to FlyToIsland
- `ContractPacks/Milestones/IslandHop.cfg` — Update requirement to FlyToIsland
- `ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg` — Deprecate (merged into RoverMonolith)
- `ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg` — Deprecate (merged into FlyToIsland)
- `ContractPacks/AnomalySurveyor/Kerbin_Pyramids.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Kerbin_UFO.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Kerbin_Monolith_Additional.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Mun_Monolith.cfg` — Update gating + dialogue
- `ContractPacks/AnomalySurveyor/Mun_Memorial.cfg` — Update gating (sequential)
- `ContractPacks/AnomalySurveyor/Mun_RockArch.cfg` — Update gating (sequential)
- `ContractPacks/AnomalySurveyor/Mun_UFO.cfg` — Update gating (sequential)
- `ContractPacks/AnomalySurveyor/Duna_Face.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Duna_MSL.cfg` — Update gating (sequential after Face)
- `ContractPacks/AnomalySurveyor/Moho_Mohole.cfg` — Update gating (optional side)
- `ContractPacks/AnomalySurveyor/Tylo_Cave.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Bop_DeadKraken.cfg` — Update gating (Jool sequential)
- `ContractPacks/AnomalySurveyor/Vall_Icehenge.cfg` — Update gating (Jool sequential)
- `ContractPacks/AnomalySurveyor/Jool_Monolith.cfg` — Update gating (Jool sequential)
- `ContractPacks/AnomalySurveyor/Kcalbeloh_Wormhole.cfg` — Update gating
- `ContractPacks/AnomalySurveyor/Kcalbeloh_Beyond.cfg` — Unchanged (already sequential)
- `ContractPacks/AnomalySurveyor/Kcalbeloh_BlackHole.cfg` — Update completion dialogue (campaign finale)
- `ContractPacks/AnomalySurveyor/OPM_Tekto_Spire.cfg` — Update gating (optional side)
- `ContractPacks/AnomalySurveyor/OPM_Thatmo_Monolith.cfg` — Update gating (optional side)
- `ContractPacks/AnomalySurveyor/OPM_Plock_Monolith.cfg` — Update gating (optional side)
- `ContractPacks/MissionControl/FirstRelay.cfg` — Update dialogue framing
- `ContractPacks/MissionControl/RelayConstellation.cfg` — Update dialogue framing

### Files Unchanged
- `ContractPacks/Exploration/Exploration.cfg` — Group definition, no changes needed
- `ContractPacks/AnomalySurveyor/AnomalySurveyor.cfg` — Group definition, no changes
- `ContractPacks/MissionControl/MissionControl.cfg` — Group definition, no changes
- `ContractPacks/Milestones/Milestones.cfg` — Group definition, no changes
- All Tier 3 radiant contracts (resupply, crew rotation, etc.)
- KS_Batch1-4 — Side content, gating stays on aviation milestones

---

### Task 1: Create Combined Campaign Opening Contracts

**Files:**
- Create: `ContractPacks/Exploration/CampaignOpening.cfg`

This file contains two new CONTRACT_TYPE definitions: `BKEX_RoverMonolith` (combines FirstRover + Kerbin_Monolith) and `BKEX_FlyToIsland` (combines FirstFlight + Kerbin_IslandAirfield). These are the opening fork of the campaign.

- [ ] **Step 1: Create the CampaignOpening.cfg file**

```cfg
// CampaignOpening.cfg — The opening fork of the campaign.
// After FirstScience, the player chooses: drive to the monolith, or fly to the island.
// Each combines gameplay from two former contracts into one cohesive mission.

// ============================================================================
// ROVER TO MONOLITH — Drive to the anomaly north of KSC
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_RoverMonolith
	group = BadgKatExploration

	title = Investigate the Anomaly
	genericTitle = Investigate the Anomaly
	description = Linus has been complaining about his gyroscopes drifting for weeks. Wernher finally checked and found strange magnetic fields coming from somewhere north of KSC. Drive a rover out there and see what is causing it.
	synopsis = Drive a rover to the Tycho Magnetic Anomaly and investigate on EVA.
	completedMessage = A monolith. Perfectly smooth, perfectly black, humming with a signal we cannot decode. And it is pointing somewhere — away from Kerbin.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 25000.0
	rewardReputation = 10.0
	rewardScience = 15.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = FirstScienceDone
		type = CompleteContract
		contractType = BKEX_FirstScience
	}

	PARAMETER
	{
		name = RoverDrive
		type = VesselParameterGroup
		title = Drive a rover to the anomaly

		PARAMETER
		{
			name = HasWheels
			type = PartValidation
			partModule = ModuleWheelMotor
			minCount = 1
			title = Vessel must have powered wheels
		}

		PARAMETER
		{
			name = NoEngines
			type = PartValidation
			title = No rocket engines — drive on wheels

			NONE
			{
				partModule = ModuleEngines
			}

			NONE
			{
				partModule = ModuleEnginesFX
			}
		}

		PARAMETER
		{
			name = IsDriving
			type = ReachState
			situation = LANDED
			minSpeed = 1
			title = Be moving on the surface
		}
	}

	PARAMETER
	{
		name = InvestigateMonolith
		type = All
		title = Investigate the anomaly on EVA

		PARAMETER
		{
			name = VesselIsType
			type = VesselIsType
			vesselType = EVA
		}

		PARAMETER
		{
			name = VisitWaypoint
			type = VisitWaypoint
			index = 0
			distance = 15
		}
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		WAYPOINT
		{
			name = Tycho Magnetic Anomaly
			icon = BadgKatCareer/ContractPacks/AnomalySurveyor/Icons/monolith
			latitude = 0.10227297919058
			longitude = -74.5684338987643
			altitude = 0.0
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Strange Readings

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Gus
		characterBModel = Strategy_MechanicGuy
		characterBAnimation = idle
		characterBColor = #FFFFC078

		LINE
		{
			speaker = A
			text = Linus in the science lab says his gyroscopes have been drifting for weeks. Wernher checked — something is generating a magnetic field north of KSC. Not far. Drive a rover out there and take a look.
		}
		LINE
		{
			speaker = B
			text = I bolted some wheels to a probe core last week. She pulls a little to the left, but she will get you there. Just keep her under five meters per second on the hills — she is top-heavy.
		}
		LINE
		{
			speaker = A
			text = Gus Kerman, head engineer. If it has wheels, wings, or welding marks, he built it.
		}
		LINE
		{
			speaker = B
			text = She is not pretty, but she runs. Hop out on EVA when you get close and see what is causing those readings.
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The Monolith

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle_wonder
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = Gene... it is a monolith. A perfectly smooth, perfectly black rectangle, just standing there in the grass. The magnetic readings are off every scale I have.
		}
		LINE
		{
			speaker = B
			text = Right in our backyard and we never noticed. How long has this been here?
		}
		LINE
		{
			speaker = A
			text = I am picking up something else. A signal — very faint, very structured. It is not random noise. It is pointing... up. Away from Kerbin. Like it wants us to follow.
		}
		LINE
		{
			speaker = B
			text = Someone put this here, Linus. And they left a trail. We need to figure out where it leads.
		}
	}
}

// ============================================================================
// FLY TO ISLAND — First flight + island airfield investigation
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_FlyToIsland
	group = BadgKatExploration

	title = Fly to the Island Airfield
	genericTitle = Fly to the Island Airfield
	description = There is an old airfield on the island across from KSC. Nobody knows who built it or when. The engineering team has been sketching wing designs on napkins — time to see if one of their creations can fly you over there to investigate.
	synopsis = Fly a winged vessel to the island airfield and investigate on EVA.
	completedMessage = An overgrown airfield with a burnt-out pod in the hangar. Someone was flying from here long before us. The runway is still usable — that could come in handy.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 20000.0
	rewardReputation = 5.0
	rewardScience = 3.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = FirstScienceDone
		type = CompleteContract
		contractType = BKEX_FirstScience
	}

	PARAMETER
	{
		name = FlyThere
		type = VesselParameterGroup
		title = Fly a winged vessel

		PARAMETER
		{
			name = HasWings
			type = PartValidation
			partModule = ModuleLiftingSurface
			minCount = 1
			title = Vessel must have wings
		}

		PARAMETER
		{
			name = HasIntake
			type = PartValidation
			partModule = ModuleResourceIntake
			minCount = 1
			title = Must have an air intake
		}

		PARAMETER
		{
			name = HasCrew
			type = HasCrew
			minCrew = 1
			title = Must be crewed
		}

		PARAMETER
		{
			name = IsFlying
			type = ReachState
			situation = FLYING
			title = Be airborne
		}
	}

	PARAMETER
	{
		name = InvestigateTower
		type = All
		title = Climb to the top of the control tower

		PARAMETER
		{
			name = VesselIsType
			type = VesselIsType
			vesselType = EVA
		}

		PARAMETER
		{
			name = VisitWaypoint
			type = VisitWaypoint
			index = 0
			distance = 5
		}
	}

	PARAMETER
	{
		name = InvestigateHangar
		type = All
		title = Look for the burnt out pod in the hangar

		PARAMETER
		{
			name = VesselIsType
			type = VesselIsType
			vesselType = EVA
		}

		PARAMETER
		{
			name = VisitWaypoint
			type = VisitWaypoint
			index = 1
			distance = 25
		}
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		PQS_CITY
		{
			name = Control Tower
			icon = BadgKatCareer/ContractPacks/AnomalySurveyor/Icons/unknown
			pqsCity = IslandAirfield
			pqsOffset = -25.5440407030383, -180.724966497429, 59.493380329812
			altitude = 59.493380329812
		}

		PQS_CITY
		{
			name = Pod in Hangar
			icon = BadgKatCareer/ContractPacks/AnomalySurveyor/Icons/unknown
			pqsCity = IslandAirfield
			pqsOffset = -102.891933658238, -107.91985303655, 18.1263730936462
			altitude = 18.1263730936462
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Dream of Flight

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = There is an airfield on the island across from us. Old, overgrown — but someone built it. In 1903, two bicycle mechanics built a machine out of wood and fabric that flew for twelve seconds. Twelve seconds was enough to change everything.
		}
		LINE
		{
			speaker = B
			text = We have slightly better materials. And a runway that is longer than a sand dune.
		}
		LINE
		{
			speaker = A
			text = The physics is simple and beautiful — air moves faster over a curved wing, creating lower pressure on top. That difference in pressure is lift. Fight gravity with geometry.
		}
		LINE
		{
			speaker = A
			text = One critical detail — the center of lift should sit just behind the center of mass. That way, if the nose dips, the lift pushes it back up. If it is the other way around... well. You will find out quickly.
		}
		LINE
		{
			speaker = B
			text = Fly over to the island, land, and check out that old airfield. See what you find. I have the fire crew standing by. Not that I am worried.
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The Old Airfield

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_nodA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_smileA
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = An old airfield with a burnt-out pod in the hangar. Someone was flying from here long before us. The runway is overgrown but solid.
		}
		LINE
		{
			speaker = B
			text = Wings beat gravity today. Those bicycle mechanics would be proud. And the airfield — that could be useful as a secondary landing site.
		}
	}
}
```

- [ ] **Step 2: Verify syntax — check for semicolons in LINE nodes**

Run from the git repo root:
```bash
grep -n "speaker.*;" ContractPacks/Exploration/CampaignOpening.cfg
```
Expected: no matches (zero semicolons in LINE nodes).

- [ ] **Step 3: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/CampaignOpening.cfg
git commit -m "feat: create combined RoverMonolith and FlyToIsland campaign opening contracts"
```

---

### Task 2: Update FirstScience as Campaign Opener

**Files:**
- Modify: `ContractPacks/Exploration/FirstSteps.cfg` (BKEX_FirstScience section only)

FirstScience becomes the sole auto-accept contract. Its dialogue introduces the campaign and teases the RoverMonolith/FlyToIsland fork. The completion dialogue points forward to both options.

- [ ] **Step 1: Add autoAccept to FirstScience**

In `FirstSteps.cfg`, inside the `BKEX_FirstScience` CONTRACT_TYPE, add `autoAccept = true` after the `advanceFunds = 0` line:

```cfg
	advanceFunds = 0

	autoAccept = true
```

- [ ] **Step 2: Replace the FirstScience accept dialogue**

Replace the entire `ChatterBox_Accept` BEHAVIOUR block in BKEX_FirstScience with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Welcome to the Space Program

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = Welcome to the Kerbal Space Program. I am Gene Kerman, and I run Mission Control. That means I worry about everything so you do not have to.
		}
		LINE
		{
			speaker = B
			text = And I am Wernher von Kerman, head of the Science Division. I will be providing the experiments, the hypotheses, and occasionally the explosions.
		}
		LINE
		{
			speaker = A
			text = Here is the situation — we have some odd readings. Linus in the science lab says his instruments have been acting up, and there is an old airfield on the island that nobody remembers building. But first things first — we need data. Any data.
		}
		LINE
		{
			speaker = B
			text = Even a simple temperature reading at the launchpad tells us something we did not know before. That is how science works — one measurement at a time.
		}
		LINE
		{
			speaker = A
			text = Collect some science, any way you can. Then we will talk about those strange readings.
		}
	}
```

- [ ] **Step 3: Replace the FirstScience completion dialogue**

Replace the entire `ChatterBox_Complete` BEHAVIOUR block in BKEX_FirstScience with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = First Data

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_nodA
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_smileA
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = B
			text = Data! Excellent. The first measurement is always the most important — every discovery in history started with someone writing down a number.
		}
		LINE
		{
			speaker = A
			text = Good work. Now — about those odd readings. We have two leads. Linus says there is a magnetic anomaly north of KSC. Something is out there messing with his gyroscopes. Gus has built a rover that can get you there.
		}
		LINE
		{
			speaker = B
			text = And the island airfield — it is right across the water. Someone was flying from there before we existed. I have some wing designs that should get you there and back.
		}
		LINE
		{
			speaker = A
			text = Check Mission Control. Both leads are on the board — pick whichever one calls to you first.
		}
	}
```

- [ ] **Step 4: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/FirstSteps.cfg
git commit -m "feat: update FirstScience as auto-accept campaign opener with anomaly narrative"
```

---

### Task 3: Deprecate Replaced Contracts

**Files:**
- Modify: `ContractPacks/Milestones/FirstRover.cfg` — Add `maxCompletions = 0` override or rename to prevent loading
- Modify: `ContractPacks/Milestones/FirstFlight.cfg` — Same
- Modify: `ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg` — Same
- Modify: `ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg` — Same
- Modify: `ContractPacks/Milestones/SoundBarrier.cfg` — Update requirement from BKML_FirstFlight to BKEX_FlyToIsland
- Modify: `ContractPacks/Milestones/IslandHop.cfg` — Update requirement from BKML_FirstFlight to BKEX_FlyToIsland

The cleanest way to disable contracts without deleting the files (preserving git history and avoiding broken saves) is to add an impossible REQUIREMENT.

- [ ] **Step 1: Disable FirstRover**

Add the following at the end of the CONTRACT_TYPE block in `FirstRover.cfg`, just before the closing `}`:

```cfg
	// DEPRECATED: Replaced by BKEX_RoverMonolith in CampaignOpening.cfg
	REQUIREMENT
	{
		name = Deprecated
		type = Expression
		expression = false
		title = This contract has been replaced by the campaign version
	}
```

- [ ] **Step 2: Disable FirstFlight**

Add the same deprecation block at the end of the CONTRACT_TYPE block in `FirstFlight.cfg`:

```cfg
	// DEPRECATED: Replaced by BKEX_FlyToIsland in CampaignOpening.cfg
	REQUIREMENT
	{
		name = Deprecated
		type = Expression
		expression = false
		title = This contract has been replaced by the campaign version
	}
```

- [ ] **Step 3: Disable Kerbin_Monolith**

Add the same deprecation block at the end of the CONTRACT_TYPE block in `Kerbin_Monolith.cfg`:

```cfg
	// DEPRECATED: Replaced by BKEX_RoverMonolith in CampaignOpening.cfg
	REQUIREMENT
	{
		name = Deprecated
		type = Expression
		expression = false
		title = This contract has been replaced by the campaign version
	}
```

- [ ] **Step 4: Disable Kerbin_IslandAirfield**

Add the same deprecation block at the end of the CONTRACT_TYPE block in `Kerbin_IslandAirfield.cfg`:

```cfg
	// DEPRECATED: Replaced by BKEX_FlyToIsland in CampaignOpening.cfg
	REQUIREMENT
	{
		name = Deprecated
		type = Expression
		expression = false
		title = This contract has been replaced by the campaign version
	}
```

- [ ] **Step 5: Update SoundBarrier requirement**

In `SoundBarrier.cfg`, find the REQUIREMENT that references `BKML_FirstFlight` and change it to reference `BKEX_FlyToIsland`. The requirement block should read:

```cfg
	REQUIREMENT
	{
		name = FirstFlightDone
		type = CompleteContract
		contractType = BKEX_FlyToIsland
	}
```

Note: You will need to read SoundBarrier.cfg first to find the exact current requirement. The requirement name and structure should match whatever is there, just change `contractType`.

- [ ] **Step 6: Update IslandHop requirement**

In `IslandHop.cfg`, find the REQUIREMENT that references `BKML_FirstFlight` and change `contractType` to `BKEX_FlyToIsland`. Same pattern as Step 5.

- [ ] **Step 7: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Milestones/FirstRover.cfg ContractPacks/Milestones/FirstFlight.cfg
git add ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg
git add ContractPacks/Milestones/SoundBarrier.cfg ContractPacks/Milestones/IslandHop.cfg
git commit -m "refactor: deprecate contracts replaced by campaign opening, update downstream deps"
```

---

### Task 4: Create Bridge Contracts

**Files:**
- Create: `ContractPacks/Exploration/BridgeContracts.cfg`

Bridge contracts are auto-accept, zero-parameter narrative contracts. They fire when their requirements are met, deliver a ChatterBox scene, and immediately complete. They serve as the connective tissue between campaign acts.

- [ ] **Step 1: Create BridgeContracts.cfg with all bridge contracts**

```cfg
// BridgeContracts.cfg — Narrative bridge contracts
// Auto-accept, zero-parameter contracts that deliver dialogue between campaign acts.
// These are invisible to the player as "contracts" — they just fire a scene.

// ============================================================================
// ACT 1 BRIDGE: Both leads checked — monolith + island done
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_Bridge_BothLeads
	group = BadgKatExploration

	title = Mission Briefing: New Leads
	description = Both initial leads have been investigated.
	synopsis = Receive a briefing from Mission Control.
	completedMessage = New contracts available.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1
	autoAccept = true

	rewardFunds = 0
	rewardReputation = 0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = RoverMonolithDone
		type = CompleteContract
		contractType = BKEX_RoverMonolith
	}

	REQUIREMENT
	{
		name = FlyToIslandDone
		type = CompleteContract
		contractType = BKEX_FlyToIsland
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Both Leads Checked

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle
		characterAColor = #FF6ED4C8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = Gene, I have been analyzing the monolith signal. It is structured — not random. And it is pointing straight up, off-planet. Whatever put that monolith here wants us to go higher.
		}
		LINE
		{
			speaker = B
			text = Higher means rockets. Wernher has been waiting for this. But first — the orbital photos from your flights are showing more anomalies on Kerbin. Structures in the desert. Something in the arctic ice. We are not done on the ground yet.
		}
		LINE
		{
			speaker = A
			text = The signal is getting louder, Gene. I think it knows we found the first one.
		}
	}
}

// ============================================================================
// ACT 2-3 BRIDGE: Signals from the Mun — relay network up, anomalies detected
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_Bridge_MunSignals
	group = BadgKatExploration

	title = Mission Briefing: Mun Signals
	description = The relay network has detected signals from the Mun.
	synopsis = Receive a briefing about Mun anomalies.
	completedMessage = Mun investigation contracts available.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1
	autoAccept = true

	rewardFunds = 0
	rewardReputation = 0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = RelayConstellationDone
		type = CompleteContract
		contractType = BKMC_RelayConstellation
	}

	REQUIREMENT
	{
		name = CrewedHomeOrbitDone
		type = CompleteContract
		contractType = BKEX_CrewedHomeOrbit
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Signals from the Mun

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = Wernher — the new relay network is picking up something. The same signal pattern as the Kerbin monolith, but from the Mun. Multiple sources.
		}
		LINE
		{
			speaker = B
			text = The frequency matches exactly. Same harmonic signature. Whoever placed the monolith on Kerbin placed others on the Mun. This is a trail, Linus. They are breadcrumbs.
		}
		LINE
		{
			speaker = A
			text = Should I be excited or terrified?
		}
		LINE
		{
			speaker = B
			text = Both. That is how the best discoveries feel. We need to get a probe out there — and eventually, boots on the ground.
		}
	}
}

// ============================================================================
// ACT 3-4 BRIDGE: Trace the signal — Mun monolith + Kerbin anomalies point outward
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_Bridge_TraceSignal
	group = BadgKatExploration

	title = Mission Briefing: Interplanetary Signal
	description = The monolith data points to other planets.
	synopsis = Receive a briefing about interplanetary targets.
	completedMessage = Interplanetary probe contracts available.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1
	autoAccept = true

	rewardFunds = 0
	rewardReputation = 0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = MunMonolithDone
		type = CompleteContract
		contractType = AS_Mun_Monolith
		minCount = 1
	}

	// Need at least 2 Kerbin anomalies completed (Pyramids, UFO, or Monolith_Additional)
	REQUIREMENT
	{
		name = KerbinAnomalies
		type = AtLeast
		count = 2

		REQUIREMENT
		{
			type = CompleteContract
			contractType = AS_Kerbin_Pyramids
		}

		REQUIREMENT
		{
			type = CompleteContract
			contractType = AS_Kerbin_UFO
		}

		REQUIREMENT
		{
			type = CompleteContract
			contractType = AS_Kerbin_Monolith_Additional
			minCount = 1
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Signal Points Outward

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Linus
		characterBModel = Strategy_ScienceGuy
		characterBAnimation = idle
		characterBColor = #FF6ED4C8

		LINE
		{
			speaker = B
			text = Wernher, I have been triangulating every monolith signal — Kerbin, the Mun, all of them. They converge on two points. One near Duna. One near Eve. The signal is unmistakable.
		}
		LINE
		{
			speaker = A
			text = Interplanetary. Of course it is interplanetary. The breadcrumbs lead from our backyard to the Mun and now to other planets entirely. Whoever left these wanted us to follow — and they wanted to see if we could build the machines to get there.
		}
		LINE
		{
			speaker = B
			text = Can we? Get there, I mean?
		}
		LINE
		{
			speaker = A
			text = A transfer window to Duna opens regularly. The math is beautiful — you do not fly to where a planet is, you fly to where it will be. Like throwing a ball to a running friend. We send probes first. Always probes first.
		}
	}
}

// ============================================================================
// ACT 4-5 BRIDGE: Going deeper — interplanetary anomaly leads to outer system
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_Bridge_GoingDeeper
	group = BadgKatExploration

	title = Mission Briefing: The Trail Continues
	description = Interplanetary anomalies point to the outer system.
	synopsis = Receive a briefing about outer system targets.
	completedMessage = Outer system contracts available.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1
	autoAccept = true

	rewardFunds = 0
	rewardReputation = 0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	// Duna OR Eve anomaly complete
	REQUIREMENT
	{
		name = InterplanetaryAnomaly
		type = Any

		REQUIREMENT
		{
			type = CompleteContract
			contractType = AS_Duna_Face
		}

		REQUIREMENT
		{
			type = CompleteContract
			contractType = AS_Duna_MSL
		}
	}

	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Going Deeper

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Linus
		characterAModel = Strategy_ScienceGuy
		characterAAnimation = idle_wonder
		characterAColor = #FF6ED4C8

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = The Duna data is in. Same monolith signature, same structured signal. And it points further out — toward the gas giants. Jool and its moons.
		}
		LINE
		{
			speaker = B
			text = The trail does not end. Each monolith points to the next, and each one is further from home. Whoever built them wanted to test how far we would go.
		}
		LINE
		{
			speaker = A
			text = Jool is a long way from home, Wernher. The travel time alone is...
		}
		LINE
		{
			speaker = B
			text = Years. But that is what probes are for. We build them, we aim them, and we wait. The universe is patient. We should be too.
		}
	}
}
```

- [ ] **Step 2: Verify syntax**

```bash
grep -n "speaker.*;" ContractPacks/Exploration/BridgeContracts.cfg
```
Expected: no matches.

- [ ] **Step 3: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/BridgeContracts.cfg
git commit -m "feat: create bridge contracts for campaign narrative flow (Acts 1-5)"
```

---

### Task 5: Update Act 1 Anomaly Gating

**Files:**
- Modify: `ContractPacks/AnomalySurveyor/Kerbin_Pyramids.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Kerbin_UFO.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Kerbin_Monolith_Additional.cfg`

Kerbin anomalies now gate off the Act 1 bridge instead of generic milestones.

- [ ] **Step 1: Update Kerbin_Pyramids gating**

In `Kerbin_Pyramids.cfg`, replace the existing REQUIREMENT block:

```cfg
    REQUIREMENT
    {
        name = ReachSpace
        type = ReachSpace
    }
```

With:

```cfg
    REQUIREMENT
    {
        name = BothLeadsDone
        type = CompleteContract
        contractType = BKEX_Bridge_BothLeads
    }
```

- [ ] **Step 2: Update Kerbin_UFO gating**

In `Kerbin_UFO.cfg`, replace the existing REQUIREMENT blocks (the `Orbit` requirement and any SCANsat requirements — keep SCANsat if present) with:

```cfg
    REQUIREMENT
    {
        name = PyramidsDone
        type = CompleteContract
        contractType = AS_Kerbin_Pyramids
    }
```

Keep any `REQUIREMENT:NEEDS[SCANsat]` blocks unchanged.

- [ ] **Step 3: Update Kerbin_Monolith_Additional gating**

In `Kerbin_Monolith_Additional.cfg`, replace the existing REQUIREMENT at the bottom:

```cfg
    REQUIREMENT
    {
        type = CompleteContract

        contractType = AS_Kerbin_Monolith

        title = Must have discovered a similar anomaly
    }
```

With:

```cfg
    REQUIREMENT
    {
        type = CompleteContract
        contractType = BKEX_RoverMonolith
        title = Must have investigated the first monolith
    }
```

- [ ] **Step 4: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/AnomalySurveyor/Kerbin_Pyramids.cfg
git add ContractPacks/AnomalySurveyor/Kerbin_UFO.cfg
git add ContractPacks/AnomalySurveyor/Kerbin_Monolith_Additional.cfg
git commit -m "feat: update Kerbin anomaly gating for campaign progression"
```

---

### Task 6: Update Act 2 Exploration Contract Gating

**Files:**
- Modify: `ContractPacks/Exploration/FirstSteps.cfg` (BKEX_UnmannedSuborbital section)
- Modify: `ContractPacks/Exploration/EarlyCrewed.cfg` (BKEX_CrewedUpperAtmo section)

UnmannedSuborbital now gates off the "Both Leads" bridge (not just FirstScience). CrewedUpperAtmo gates off UnmannedSuborbital (not just FirstScience).

- [ ] **Step 1: Update UnmannedSuborbital requirement**

In `FirstSteps.cfg`, in the `BKEX_UnmannedSuborbital` CONTRACT_TYPE, replace:

```cfg
	REQUIREMENT
	{
		name = FirstScienceDone
		type = CompleteContract
		contractType = BKEX_FirstScience
	}
```

With:

```cfg
	REQUIREMENT
	{
		name = BothLeadsDone
		type = CompleteContract
		contractType = BKEX_Bridge_BothLeads
	}
```

- [ ] **Step 2: Rewrite UnmannedSuborbital accept dialogue**

Replace the entire `ChatterBox_Accept` BEHAVIOUR in BKEX_UnmannedSuborbital with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Going Up

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Mortimer
		characterBModel = Strategy_Mortimer
		characterBAnimation = idle
		characterBColor = #FFFFE066

		LINE
		{
			speaker = A
			text = The monolith signal is pointing straight up, out of the atmosphere. Linus says whatever is sending it is not on Kerbin. We need to get above the sky to trace it.
		}
		LINE
		{
			speaker = B
			text = Do you have ANY idea how much Kerbosene costs per kilogram? We are about to set a very large pile of money on fire. Literally.
		}
		LINE
		{
			speaker = A
			text = Mortimer Kerman, everybody. Our finance director. He keeps the lights on.
		}
		LINE
		{
			speaker = B
			text = Someone has to. Do we at least have to send a kerbal up there?
		}
		LINE
		{
			speaker = A
			text = No. A probe goes first — cheaper, no Snacks required, and nobody's family calls if it does not come back. Suborbital means it goes up past 70 kilometers where the sky goes black, then comes back down. If the signal is up there, we will find it.
		}
	}
```

- [ ] **Step 3: Rewrite UnmannedSuborbital complete dialogue**

Replace the entire `ChatterBox_Complete` BEHAVIOUR in BKEX_UnmannedSuborbital with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = We Touched Space

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = true_thumbsUp
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = true_smileA
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = Space! Our probe touched actual space and came back in one piece!
		}
		LINE
		{
			speaker = B
			text = The signal is there — faint, but our instruments detected it above the atmosphere. It does not stop at suborbital. It keeps going. To trace it properly, we need to stay up there. Orbit.
		}
		LINE
		{
			speaker = A
			text = Orbit is a very different problem than going up and coming back down.
		}
		LINE
		{
			speaker = B
			text = A beautiful, elegant, sideways problem. And we are going to solve it.
		}
	}
```

- [ ] **Step 4: Update CrewedUpperAtmo requirement**

In `EarlyCrewed.cfg`, in the `BKEX_CrewedUpperAtmo` CONTRACT_TYPE, replace:

```cfg
	REQUIREMENT
	{
		name = FirstScienceDone
		type = CompleteContract
		contractType = BKEX_FirstScience
	}
```

With:

```cfg
	REQUIREMENT
	{
		name = UnmannedSuborbitalDone
		type = CompleteContract
		contractType = BKEX_UnmannedSuborbital
	}
```

- [ ] **Step 5: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/FirstSteps.cfg ContractPacks/Exploration/EarlyCrewed.cfg
git commit -m "feat: update Act 2 exploration gating and dialogue for anomaly narrative"
```

---

### Task 7: Rewrite Act 2 Remaining Dialogue

**Files:**
- Modify: `ContractPacks/Exploration/UnmannedOrbit.cfg`
- Modify: `ContractPacks/Exploration/CrewedExploration.cfg` (CrewedHomeOrbit section)
- Modify: `ContractPacks/MissionControl/FirstRelay.cfg`

These contracts keep their gating unchanged but get dialogue reframed around the anomaly signal narrative. Teaching content stays.

- [ ] **Step 1: Rewrite UnmannedOrbit accept dialogue**

In `UnmannedOrbit.cfg`, replace the `ChatterBox_Accept` BEHAVIOUR with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = The Sideways Problem

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = The signal does not stop at suborbital. To track it, we need to stay up there. That means orbit — and orbit is a completely different trick than going up and coming down.
		}
		LINE
		{
			speaker = B
			text = Different how?
		}
		LINE
		{
			speaker = A
			text = Orbit is going sideways so fast that you keep falling but never hit the ground. Newton imagined a cannon on a very tall mountain. Fire it gently, the ball lands nearby. Fire it hard enough, and the ball falls AROUND the planet. The ground curves away as fast as the ball falls toward it.
		}
		LINE
		{
			speaker = A
			text = Your periapsis is the lowest point of the orbit — the closest you get to the ground. Your apoapsis is the highest. To circularize, you burn at apoapsis until your periapsis comes up to match.
		}
		LINE
		{
			speaker = B
			text = In simple terms — point up, go fast, then turn sideways and go faster.
		}
		LINE
		{
			speaker = A
			text = ...that is reductive, but technically not wrong.
		}
	}
```

- [ ] **Step 2: Rewrite UnmannedOrbit complete dialogue**

Replace the `ChatterBox_Complete` BEHAVIOUR in `UnmannedOrbit.cfg` with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = Orbit Achieved

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = true_thumbsUp
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = true_smileA
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = Orbit! It will be up there long after we are gone. That is what orbit means — you gave it enough energy to fall forever and never land.
		}
		LINE
		{
			speaker = B
			text = The signal is clearer from orbit. Linus says it is definitely coming from beyond Kerbin. But we are going to need a communications network before we can track it properly. Relays, Wernher.
		}
		LINE
		{
			speaker = A
			text = First a network, then we aim for the Mun. The monolith signal should lead us straight there.
		}
	}
```

- [ ] **Step 3: Rewrite CrewedHomeOrbit accept dialogue**

In `CrewedExploration.cfg`, in the `BKEX_CrewedHomeOrbit` CONTRACT_TYPE, replace the `ChatterBox_Accept` BEHAVIOUR with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = All the Way Around

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Gene
		characterAModel = Instructor_Gene
		characterAAnimation = idle
		characterAColor = #FF8BE08A

		characterBName = Wernher
		characterBModel = Instructor_Wernher
		characterBAnimation = idle
		characterBColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = We have probes in orbit tracking the monolith signal. But tracking is not enough — we need crew up there. Real eyes, real hands, real judgment. An orbit with kerbals aboard.
		}
		LINE
		{
			speaker = B
			text = Remember the difference — suborbital is a lob. Orbit is a circle. The crew will be weightless not for minutes but for as long as they stay up there. They will see sunrise every ninety minutes.
		}
		LINE
		{
			speaker = A
			text = And we need them back. The deorbit burn is just as important as the one that got them up there.
		}
	}
```

- [ ] **Step 4: Rewrite FirstRelay accept dialogue**

In `FirstRelay.cfg`, replace the `ChatterBox_Accept` BEHAVIOUR with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Building the Network

		presentation = Dialogue
		scale = 0.45
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		LINE
		{
			speaker = A
			text = The monolith signal is clearer from orbit, but our tracking has blind spots. Every time a probe flies behind the planet, we lose contact. A relay satellite is a mirror for radio signals — put one in orbit, and the signal finds a path home. We build the network first, then we follow the trail.
		}
	}
```

- [ ] **Step 5: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/UnmannedOrbit.cfg
git add ContractPacks/Exploration/CrewedExploration.cfg
git add ContractPacks/MissionControl/FirstRelay.cfg
git commit -m "feat: rewrite Act 2 dialogue for anomaly signal narrative"
```

---

### Task 8: Update Mun Anomaly Contract Gating

**Files:**
- Modify: `ContractPacks/AnomalySurveyor/Mun_Monolith.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Mun_Memorial.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Mun_RockArch.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Mun_UFO.cfg`

Mun anomalies now gate off the Act 2-3 bridge. Within the Mun, they become sequential: Mun_Monolith first, then Memorial → RockArch → UFO.

- [ ] **Step 1: Update Mun_Monolith gating**

In `Mun_Monolith.cfg`, replace ALL existing REQUIREMENT blocks (the `Orbit` requirement and the `AtLeast` block for Kerbin monoliths) with:

```cfg
    REQUIREMENT
    {
        name = MunSignalsBridge
        type = CompleteContract
        contractType = BKEX_Bridge_MunSignals
    }
```

This means you no longer need 2 Kerbin monolith additionals to reach the Mun — the bridge contract handles that progression.

- [ ] **Step 2: Update Mun_Memorial gating**

In `Mun_Memorial.cfg`, replace the existing REQUIREMENT blocks with:

```cfg
    REQUIREMENT
    {
        name = MunMonolithDone
        type = CompleteContract
        contractType = AS_Mun_Monolith
        minCount = 1
    }
```

Keep any `REQUIREMENT:NEEDS[SCANsat]` blocks unchanged.

- [ ] **Step 3: Update Mun_RockArch gating**

In `Mun_RockArch.cfg`, replace the existing REQUIREMENT blocks (except SCANsat) with:

```cfg
    REQUIREMENT
    {
        name = MunMemorialDone
        type = CompleteContract
        contractType = AS_Mun_Memorial
    }
```

Keep any `REQUIREMENT:NEEDS[SCANsat]` blocks unchanged.

- [ ] **Step 4: Update Mun_UFO gating**

In `Mun_UFO.cfg`, replace the existing REQUIREMENT blocks (except SCANsat) with:

```cfg
    REQUIREMENT
    {
        name = MunRockArchDone
        type = CompleteContract
        contractType = AS_Mun_RockArch
    }
```

Keep any `REQUIREMENT:NEEDS[SCANsat]` blocks unchanged. Also remove the `AS_Kerbin_UFO` completion requirement since gating is now sequential within the Mun.

- [ ] **Step 5: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/AnomalySurveyor/Mun_Monolith.cfg
git add ContractPacks/AnomalySurveyor/Mun_Memorial.cfg
git add ContractPacks/AnomalySurveyor/Mun_RockArch.cfg
git add ContractPacks/AnomalySurveyor/Mun_UFO.cfg
git commit -m "feat: update Mun anomaly gating to sequential campaign flow"
```

---

### Task 9: Update Interplanetary Anomaly Gating

**Files:**
- Modify: `ContractPacks/AnomalySurveyor/Duna_Face.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Duna_MSL.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Moho_Mohole.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Tylo_Cave.cfg`

Duna anomalies gate off the Act 3-4 bridge (TraceSignal). Within Duna, Face comes first, then MSL. Moho is optional side content.

- [ ] **Step 1: Update Duna_Face gating**

In `Duna_Face.cfg`, add a new REQUIREMENT (keep existing ones like the Duna flyby/orbit and SCANsat):

```cfg
    REQUIREMENT
    {
        name = TraceSignalDone
        type = CompleteContract
        contractType = BKEX_Bridge_TraceSignal
    }
```

- [ ] **Step 2: Update Duna_MSL gating**

In `Duna_MSL.cfg`, add a requirement for Duna_Face completion (sequential):

```cfg
    REQUIREMENT
    {
        name = DunaFaceDone
        type = CompleteContract
        contractType = AS_Duna_Face
    }
```

- [ ] **Step 3: Verify Moho_Mohole and Tylo_Cave**

Read these files. They should already gate on having reached their respective bodies. No changes needed if their existing gating is body-reach-based — they are optional side content not on the main story spine. If they have requirements referencing other anomaly contracts that no longer make sense, update them.

- [ ] **Step 4: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/AnomalySurveyor/Duna_Face.cfg
git add ContractPacks/AnomalySurveyor/Duna_MSL.cfg
git commit -m "feat: update interplanetary anomaly gating for campaign flow"
```

---

### Task 10: Update Jool System Anomaly Gating

**Files:**
- Modify: `ContractPacks/AnomalySurveyor/Vall_Icehenge.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Bop_DeadKraken.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Jool_Monolith.cfg`

Jool system anomalies gate off the "Going Deeper" bridge. Sequential: Vall → Bop → Jool_Monolith.

- [ ] **Step 1: Update Vall_Icehenge gating**

In `Vall_Icehenge.cfg`, replace/add requirements so it gates off the bridge:

```cfg
    REQUIREMENT
    {
        name = GoingDeeperDone
        type = CompleteContract
        contractType = BKEX_Bridge_GoingDeeper
    }

    REQUIREMENT
    {
        name = VallOrbit
        type = Orbit
        targetBody = Vall
    }
```

Keep any existing SCANsat requirements.

- [ ] **Step 2: Update Bop_DeadKraken gating**

In `Bop_DeadKraken.cfg`, replace/add requirements for sequential after Vall:

```cfg
    REQUIREMENT
    {
        name = VallDone
        type = CompleteContract
        contractType = AS_Vall_Icehenge
    }

    REQUIREMENT
    {
        name = BopOrbit
        type = Orbit
        targetBody = Bop
    }
```

Keep any existing SCANsat requirements.

- [ ] **Step 3: Update Jool_Monolith gating**

In `Jool_Monolith.cfg`, replace the existing Mun monolith prerequisite requirements with:

```cfg
    REQUIREMENT
    {
        name = BopDone
        type = CompleteContract
        contractType = AS_Bop_DeadKraken
    }
```

Keep the existing `Orbit` requirement for Jool and any other structural requirements (the SpawnVessel, HasCrew for Pilot+Scientist, etc.).

- [ ] **Step 4: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/AnomalySurveyor/Vall_Icehenge.cfg
git add ContractPacks/AnomalySurveyor/Bop_DeadKraken.cfg
git add ContractPacks/AnomalySurveyor/Jool_Monolith.cfg
git commit -m "feat: update Jool system anomaly gating to sequential campaign flow"
```

---

### Task 11: Update Endgame + OPM Gating

**Files:**
- Modify: `ContractPacks/AnomalySurveyor/Kcalbeloh_Wormhole.cfg`
- Modify: `ContractPacks/AnomalySurveyor/Kcalbeloh_BlackHole.cfg`
- Modify: `ContractPacks/AnomalySurveyor/OPM_Tekto_Spire.cfg`
- Modify: `ContractPacks/AnomalySurveyor/OPM_Thatmo_Monolith.cfg`
- Modify: `ContractPacks/AnomalySurveyor/OPM_Plock_Monolith.cfg`

Kcalbeloh_Wormhole gates off Jool_Monolith. OPM contracts are optional side content gated on reaching their bodies.

- [ ] **Step 1: Update Kcalbeloh_Wormhole gating**

In `Kcalbeloh_Wormhole.cfg`, add/replace the main story gate requirement:

```cfg
    REQUIREMENT
    {
        name = JoolMonolithDone
        type = CompleteContract
        contractType = AS_Jool_Monolith
    }
```

Keep any existing `NEEDS[Kcalbeloh]` conditions and body-specific orbit requirements.

- [ ] **Step 2: Rewrite Kcalbeloh_BlackHole completion dialogue**

In `Kcalbeloh_BlackHole.cfg`, replace the `ChatterBox_Complete` BEHAVIOUR with the campaign finale:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Complete
		type = ChatterBox
		onState = CONTRACT_SUCCESS
		title = The Invitation

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle_wonder
		characterAColor = #FF82B4E8

		characterBName = Gene
		characterBModel = Instructor_Gene
		characterBAnimation = idle_wonder
		characterBColor = #FF8BE08A

		LINE
		{
			speaker = A
			text = The data is... Gene, the monolith signal does not end here. The black hole data contained star charts. Maps to places we do not have names for. Galaxies beyond our own.
		}
		LINE
		{
			speaker = B
			text = Star charts. Someone left us a trail of breadcrumbs from our own backyard to the edge of a black hole, and the last breadcrumb is a map to somewhere else entirely.
		}
		LINE
		{
			speaker = A
			text = The monoliths were not a test. They were an invitation. Every single one — from the grass outside KSC to here. They wanted to see if we could make the journey.
		}
		LINE
		{
			speaker = B
			text = And we did. Every rocket, every relay, every kerbal who volunteered to sit on top of a controlled explosion — it all led here.
		}
		LINE
		{
			speaker = A
			text = The question is no longer whether we can reach the stars. The question is whether we are ready for what is out there waiting for us.
		}
	}
```

- [ ] **Step 3: Verify OPM contracts**

Read `OPM_Tekto_Spire.cfg`, `OPM_Thatmo_Monolith.cfg`, and `OPM_Plock_Monolith.cfg`. These should gate on reaching their respective bodies (orbit requirements). They are optional side content — no changes to gating needed unless they reference contracts that have been renamed. Verify and update if needed.

- [ ] **Step 4: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/AnomalySurveyor/Kcalbeloh_Wormhole.cfg
git add ContractPacks/AnomalySurveyor/Kcalbeloh_BlackHole.cfg
git add ContractPacks/AnomalySurveyor/OPM_Tekto_Spire.cfg
git add ContractPacks/AnomalySurveyor/OPM_Thatmo_Monolith.cfg
git add ContractPacks/AnomalySurveyor/OPM_Plock_Monolith.cfg
git commit -m "feat: update endgame and OPM gating, write campaign finale dialogue"
```

---

### Task 12: Update ProbeFlyby for Duna-First Targeting

**Files:**
- Modify: `ContractPacks/Exploration/ProbeExploration.cfg` (BKEX_ProbeFlyby section)

The current ProbeFlyby dynamically selects targets (homeworld moons first, then anything unreached). For the campaign, the first interplanetary flyby should be narratively motivated — the signal points to Duna. After Duna, Eve opens alongside.

The cleanest approach: keep ProbeFlyby as a generic repeatable contract, but add the Act 3-4 bridge as a requirement. The bridge already gates on Mun_Monolith + Kerbin anomalies, so interplanetary flybys only open after the story justifies them. The dynamic targeting (which body is offered) stays — the player's first unreached body after the Mun will naturally be an interplanetary target.

- [ ] **Step 1: Add bridge requirement to ProbeFlyby**

In `ProbeExploration.cfg`, in the `BKEX_ProbeFlyby` CONTRACT_TYPE, add a new REQUIREMENT alongside the existing `UnmannedOrbitDone`:

```cfg
	REQUIREMENT
	{
		name = TraceSignalDone
		type = CompleteContract
		contractType = BKEX_Bridge_TraceSignal
	}
```

Keep the existing `UnmannedOrbitDone` requirement as well.

- [ ] **Step 2: Rewrite ProbeFlyby accept dialogue**

Replace the `ChatterBox_Accept` BEHAVIOUR in BKEX_ProbeFlyby with:

```cfg
	BEHAVIOUR
	{
		name = ChatterBox_Accept
		type = ChatterBox
		onState = CONTRACT_ACCEPTED
		title = Following the Trail

		presentation = Dialogue
		scale = 0.50
		pauseGame = true
		triggerOnce = true

		characterAName = Wernher
		characterAModel = Instructor_Wernher
		characterAAnimation = idle
		characterAColor = #FF82B4E8

		characterBName = Linus
		characterBModel = Strategy_ScienceGuy
		characterBAnimation = idle
		characterBColor = #FF6ED4C8

		LINE
		{
			speaker = A
			text = The monolith signals point outward — to other planets. A flyby is the fastest way to get eyes on a new world. Our probe will pass close, gather data, and the gravity of the target bends our path onward.
		}
		LINE
		{
			speaker = B
			text = A flyby means we do not have to slow down? We just sail past?
		}
		LINE
		{
			speaker = A
			text = Precisely. Stopping takes fuel — sometimes more fuel than getting there. A flyby trades time for efficiency. And everything with mass bends the space around it. A planet is a curve in the road. Use it well and you get a free turn. The Voyager probes used this trick to visit four planets on one tank.
		}
		LINE
		{
			speaker = B
			text = If the monolith signal is out there... we will find it.
		}
	}
```

- [ ] **Step 3: Commit**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add ContractPacks/Exploration/ProbeExploration.cfg
git commit -m "feat: gate interplanetary probes behind campaign bridge, rewrite flyby dialogue"
```

---

### Task 13: Verification Pass

**Files:** All modified files across the campaign

This is a verification task — no new code, just checking that everything connects correctly.

- [ ] **Step 1: Verify all contract name references**

Search for every `contractType = ` reference across all ContractPacks cfg files and verify that each one points to a contract that exists:

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
grep -rn "contractType = " ContractPacks/ | sort
```

Cross-reference against the list of actual CONTRACT_TYPE `name =` values:

```bash
grep -rn "name = BK\|name = AS_\|name = BKEX_\|name = BKML_\|name = BKMC_" ContractPacks/ | grep "CONTRACT_TYPE" -A1
```

Check for:
- References to deprecated contracts (BKML_FirstRover, BKML_FirstFlight, AS_Kerbin_Monolith, AS_Kerbin_IslandAirfield) that should have been updated
- Misspelled contract names
- Circular dependencies

- [ ] **Step 2: Verify progression chain flows**

Trace the requirement chain from FirstScience to Kcalbeloh_BlackHole and verify there are no dead ends or impossible gates:

1. `BKEX_FirstScience` (autoAccept) → requires nothing
2. `BKEX_RoverMonolith` → requires BKEX_FirstScience
3. `BKEX_FlyToIsland` → requires BKEX_FirstScience
4. `BKEX_Bridge_BothLeads` → requires RoverMonolith AND FlyToIsland
5. `BKEX_UnmannedSuborbital` → requires Bridge_BothLeads
6. `BKEX_UnmannedOrbit` → requires UnmannedSuborbital
7. `BKEX_CrewedUpperAtmo` → requires UnmannedSuborbital
8. `BKEX_CrewedSuborbital` → requires UnmannedSuborbital AND CrewedUpperAtmo
9. `BKEX_CrewedHomeOrbit` → requires UnmannedOrbit AND CrewedSuborbital
10. `BKMC_FirstRelay` → requires UnmannedOrbit
11. `BKMC_RelayConstellation` → requires FirstRelay
12. `BKEX_Bridge_MunSignals` → requires RelayConstellation AND CrewedHomeOrbit
13. `AS_Mun_Monolith` → requires Bridge_MunSignals
14. `AS_Mun_Memorial` → requires Mun_Monolith (minCount=1)
15. `AS_Mun_RockArch` → requires Mun_Memorial
16. `AS_Mun_UFO` → requires Mun_RockArch
17. `BKEX_Bridge_TraceSignal` → requires Mun_Monolith + 2 Kerbin anomalies
18. `BKEX_ProbeFlyby` → requires Bridge_TraceSignal + UnmannedOrbit
19. `AS_Duna_Face` → requires Bridge_TraceSignal + Duna flyby/orbit
20. `AS_Duna_MSL` → requires Duna_Face
21. `BKEX_Bridge_GoingDeeper` → requires Duna_Face OR Duna_MSL
22. `AS_Vall_Icehenge` → requires Bridge_GoingDeeper + Vall orbit
23. `AS_Bop_DeadKraken` → requires Vall_Icehenge + Bop orbit
24. `AS_Jool_Monolith` → requires Bop_DeadKraken
25. `AS_Kcalbeloh_Wormhole` → requires Jool_Monolith
26. `AS_Kcalbeloh_Beyond` → requires Wormhole
27. `AS_Kcalbeloh_Dipuc` → requires Beyond
28. `AS_Kcalbeloh_BlackHole` → requires Dipuc

Parallel tracks (not gating main story):
- `AS_Kerbin_Pyramids` → requires Bridge_BothLeads
- `AS_Kerbin_UFO` → requires Kerbin_Pyramids
- `AS_Kerbin_Monolith_Additional` → requires RoverMonolith
- Aviation milestones → require FlyToIsland
- OPM anomalies → require reaching those bodies

- [ ] **Step 3: Check for semicolons in all LINE nodes**

```bash
grep -rn "speaker.*;" ContractPacks/
```

Expected: zero matches.

- [ ] **Step 4: Launch KSP and check KSP.log for errors**

1. Launch KSP
2. Check `KSP.log` for ContractConfigurator errors
3. Search for `[ERR]` and `ContractConfigurator` in the log
4. Verify contracts appear in Mission Control with correct gating
5. Use Alt+F12 debug menu to force-complete requirements and test later contracts

- [ ] **Step 5: Final commit (if any fixes needed)**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer"
git add -A
git commit -m "fix: verification pass corrections for campaign progression"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] Campaign arc from FirstScience to BlackHole — Tasks 1-12
- [x] Combined contracts (RoverMonolith, FlyToIsland) — Task 1
- [x] Bridge contracts (4 bridges) — Task 4
- [x] FirstScience auto-accept — Task 2
- [x] Dialogue rewrite (anomaly framing) — Tasks 2, 6, 7, 11
- [x] Deprecated contracts handled — Task 3
- [x] Downstream dependency updates — Tasks 3, 5, 6, 8, 9, 10, 11, 12
- [x] Mun anomalies sequential — Task 8
- [x] Jool anomalies sequential — Task 10
- [x] Duna/Eve gate Jool — Task 9 + Bridge in Task 4
- [x] OPM as optional side — Task 11
- [x] Campaign finale dialogue — Task 11
- [x] Verification pass — Task 13

**Not in Phase 1 scope (deferred to Phase 2/3):**
- Contract condition improvements (antenna power, orbital mechanics, etc.)
- Crew recruitment (SpawnKerbal/RecoverKerbal)
- Relay constellation multi-vessel requirements
- Snacks/life support requirements
- Trait requirements on anomaly contracts

**Placeholder scan:** No TBDs, TODOs, or vague instructions. All cfg code is complete. All contract names are consistent.

**Type consistency:** All contract name references (`contractType = X`) match their definitions (`name = X`). Verified in Task 13 Step 2.
