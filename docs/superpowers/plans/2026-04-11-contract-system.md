# BadgKatCareer Contract System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a complete, self-contained contract system for BadgKatCareer — 23 new contracts across 3 groups, replace all third-party contract packs, clean up stock contract handling.

**Architecture:** KSP ContractConfigurator (CC) `.cfg` files organized into three CONTRACT_GROUPs (Exploration, MissionControl, Milestones). Each contract is a CONTRACT_TYPE in its own file. Stock contracts disabled via CONTRACT_GROUP `disabledContractType` or ModuleManager patches. CKAN metapackage updated to move ResearchBodies to hard dependency.

**Tech Stack:** KSP ModuleManager config syntax, ContractConfigurator expression language, CKAN JSON metapackage format.

**Base path:** `C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\`

**Key CC patterns used throughout (reference):**
- `CONTRACT_GROUP { name = X; displayName = Y; minVersion = 1.15.0; disabledContractType = Z }` — group definition, can disable stock types
- `CONTRACT_TYPE { name = X; group = Y; ... }` — individual contract
- `DATA { type = CelestialBody; uniquenessCheck = CONTRACT_ACTIVE; targetBody1 = <expression>; title = "..." }` — data binding (title required when requiredValue = true)
- `REQUIREMENT { name = X; type = CompleteContract; contractType = Y }` — gate behind another contract
- `PARAMETER { name = X; type = VesselParameterGroup; ... }` — vessel checks
- `BEHAVIOUR { name = X; type = WaypointGenerator; WAYPOINT { ... } }` — place map markers
- All DATA nodes with `requiredValue = true` MUST have a `title` field
- Avoid apostrophes in title/expression fields (CC parses them as expression delimiters)
- `Orbit` parameter type requires at least one constraint (use `minAltitude = 0`)

---

### Task 1: Clean up ContractTweaks.cfg

**Files:**
- Modify: `ContractTweaks.cfg`

Remove all GAP marketplace patches (GAP is uninstalled) and add new stock contract disables.

- [ ] **Step 1: Replace ContractTweaks.cfg content**

Replace the entire file with:

```cfg
// ContractTweaks.cfg
// Disable stock contracts that are annoying, impossible, or redundant with
// BadgKatCareer's contract system. Keep contracts that add value.

// ============================================================================
// DISABLE PART TEST CONTRACTS
// "Test the RT-5 Flea between 28,000m and 42,000m while flying above Kerbin"
// Tedious, unrewarding filler. Replaced by TestFlight.
// ============================================================================
CONTRACT_GROUP
{
	name = BadgKatCareer_DisablePartTest
	minVersion = 1.15.0
	disabledContractType = PartTest
}

// ============================================================================
// DISABLE SURVEY CONTRACTS
// "Take a temperature reading above 18,000m at this exact spot"
// Often physically impossible. Replaced by BiomeSurvey.
// ============================================================================
CONTRACT_GROUP
{
	name = BadgKatCareer_DisableSurvey
	minVersion = 1.15.0
	disabledContractType = SurveyContract
}

// ============================================================================
// DISABLE PLANT FLAG CONTRACTS
// Covered by CrewedLanding contracts which include flag planting.
// ============================================================================
CONTRACT_GROUP
{
	name = BadgKatCareer_DisablePlantFlag
	minVersion = 1.15.0
	disabledContractType = PlantFlag
}

// ============================================================================
// DISABLE COLLECT SCIENCE CONTRACTS
// Covered by BiomeSurvey and DeepSpaceSurvey with better targeting.
// ============================================================================
CONTRACT_GROUP
{
	name = BadgKatCareer_DisableCollectScience
	minVersion = 1.15.0
	disabledContractType = CollectScience
}

// ============================================================================
// DISABLE SATELLITE CONTRACTS
// Covered by FirstRelay and RelayConstellation contracts.
// ============================================================================
CONTRACT_GROUP
{
	name = BadgKatCareer_DisableSatellite
	minVersion = 1.15.0
	disabledContractType = SatelliteContract
}
```

- [ ] **Step 2: Verify the file**

Open `ContractTweaks.cfg` and confirm:
- No GAP marketplace patches remain
- 5 CONTRACT_GROUPs disabling: PartTest, SurveyContract, PlantFlag, CollectScience, SatelliteContract
- Each has minVersion = 1.15.0

- [ ] **Step 3: Commit**

```bash
git add ContractTweaks.cfg
git commit -m "chore: clean up ContractTweaks, remove GAP patches, add new stock disables"
```

---

### Task 2: Update CKAN — move ResearchBodies to depends

**Files:**
- Modify: `BadgKatCareer.ckan`

- [ ] **Step 1: Move ResearchBodies from recommends to depends**

In `BadgKatCareer.ckan`, add `{ "name": "ResearchBodies" }` to the `depends` array (after `HarmonyKSP`), and remove it from `recommends`.

The `depends` block should become:
```json
"depends": [
    { "name": "ModuleManager" },
    { "name": "CommunityTechTree" },
    { "name": "CommunityResourcePack" },
    { "name": "CommunityCategoryKit" },
    { "name": "CommunityPartsTitles" },
    { "name": "B9PartSwitch" },
    { "name": "HarmonyKSP" },
    { "name": "ResearchBodies" }
],
```

- [ ] **Step 2: Also remove these uninstalled mods from recommends if still present**

Check that GAP, Constellations, CommNetRelays, Bases and Stations Reborn, Field Research, Tourism Expanded, Research Advancement Division, Kerbal Academy, and Clever Sats are NOT in recommends (they were uninstalled).

- [ ] **Step 3: Commit**

```bash
git add BadgKatCareer.ckan
git commit -m "chore: move ResearchBodies to depends, clean up recommends"
```

---

### Task 3: Create Milestones group definition

**Files:**
- Create: `ContractPacks/Milestones/Milestones.cfg`

- [ ] **Step 1: Create the Milestones directory and group file**

```cfg
// BadgKat Milestones — One-time achievements
// Aviation, SSTO, and rover milestones that give non-rocket paths
// purpose from day one through endgame.
//
// Philosophy: Three paths start on day one (rockets, planes, rovers).
// Each path has milestones that give you a reason to build and fly.
// Aviation graduates into spaceplanes. Rovers get surface survey missions.

CONTRACT_GROUP
{
	name = BadgKatMilestones
	displayName = Milestones
	agent = Milestones

	tip = Pushing the envelope...

	minVersion = 1.15.0
}

AGENT
{
	name = Milestones
	title = BadgKat Milestones
	description = The record keepers. First flights, speed records, and achievements that mark the progress of kerbalkind. Every milestone is a story worth telling.
	logoURL = Squad/Agencies/KerbinWorldFirstRecordKeepingSociety
	logoScaledURL = Squad/Agencies/KerbinWorldFirstRecordKeepingSociety_scaled
	mentality = Neutral
}
```

- [ ] **Step 2: Commit**

```bash
git add ContractPacks/Milestones/Milestones.cfg
git commit -m "feat: add Milestones contract group and agent"
```

---

### Task 4: Aviation milestones — FirstFlight

**Files:**
- Create: `ContractPacks/Milestones/FirstFlight.cfg`

- [ ] **Step 1: Create FirstFlight.cfg**

```cfg
// FirstFlight.cfg — Day one aviation milestone
// "Fly a winged vessel." The Wright Brothers moment.
// Available immediately — planes are a day-one path.

CONTRACT_TYPE
{
	name = BKML_FirstFlight
	group = BadgKatMilestones

	title = First Flight
	genericTitle = First Flight
	description = The engineering team has been sketching designs on napkins in the cafeteria. Wings, a fuselage, maybe an engine or two. It is time to see if their creation can actually leave the ground. Fly a winged vessel in the atmosphere. The Wright Brothers did it with wood and fabric — we have got rocket fuel and duct tape.
	synopsis = Fly a winged vessel in the atmosphere.
	completedMessage = Flight confirmed! The ground crew is cheering, the engineers are already arguing about the next design, and someone just ordered a second runway. Aviation has arrived at KSC.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 8000.0
	rewardReputation = 3.0
	rewardScience = 1.0
	failureFunds = 0
	advanceFunds = 0

	PARAMETER
	{
		name = FirstFlightParam
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
			name = IsFlying
			type = ReachState
			situation = FLYING
			title = Be airborne
		}
	}
}
```

- [ ] **Step 2: Commit**

```bash
git add ContractPacks/Milestones/FirstFlight.cfg
git commit -m "feat: add FirstFlight aviation milestone"
```

---

### Task 5: Aviation milestones — SoundBarrier, Mach3, Hypersonic

**Files:**
- Create: `ContractPacks/Milestones/SoundBarrier.cfg`
- Create: `ContractPacks/Milestones/Mach3.cfg`
- Create: `ContractPacks/Milestones/Hypersonic.cfg`

- [ ] **Step 1: Create SoundBarrier.cfg**

```cfg
// SoundBarrier.cfg — Break the sound barrier
// Mach 1 = 343 m/s. Your first real speed challenge.

CONTRACT_TYPE
{
	name = BKML_SoundBarrier
	group = BadgKatMilestones

	title = Break the Sound Barrier
	genericTitle = Break the Sound Barrier
	description = Our aircraft can fly, but can they fly FAST? The engineers want to know what happens when we push past Mach 1. The ground crew has been warned about the sonic boom. Jeb has been warned about pulling too hard on the stick. Neither will listen.
	synopsis = Exceed Mach 1 (343 m/s) in a winged vessel.
	completedMessage = BOOM! The sonic barrier has been shattered! Windows at KSC rattled, car alarms went off in the parking lot, and Jeb is asking when they can go faster. The answer is soon.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 20000.0
	rewardReputation = 5.0
	rewardScience = 2.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = FirstFlightDone
		type = CompleteContract
		contractType = BKML_FirstFlight
	}

	PARAMETER
	{
		name = SoundBarrierParam
		type = VesselParameterGroup
		title = Break the sound barrier

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
			name = GoFast
			type = ReachState
			situation = FLYING
			minSpeed = 343
			title = Exceed 343 m/s (Mach 1)
		}
	}
}
```

- [ ] **Step 2: Create Mach3.cfg**

```cfg
// Mach3.cfg — SR-71 territory
// Mach 3 = 1029 m/s. Needs serious engines and thermal management.

CONTRACT_TYPE
{
	name = BKML_Mach3
	group = BadgKatMilestones

	title = Mach 3 — Into the Heat
	genericTitle = Reach Mach 3
	description = Breaking the sound barrier was impressive. Now R&D wants to triple it. Mach 3 puts us in the realm of the legendary SR-71 — the kind of speed where the airframe heats up and the sky starts turning dark. We will need better engines, better aerodynamics, and a pilot with nerves of steel.
	synopsis = Exceed Mach 3 (1029 m/s) in a winged vessel.
	completedMessage = Mach 3 achieved! The aircraft returned trailing heat shimmer and the pilot returned trailing confidence. At this speed, the horizon curves. The SSTO team is taking notes.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 35000.0
	rewardReputation = 8.0
	rewardScience = 3.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = SoundBarrierDone
		type = CompleteContract
		contractType = BKML_SoundBarrier
	}

	PARAMETER
	{
		name = Mach3Param
		type = VesselParameterGroup
		title = Reach Mach 3

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
			name = GoFaster
			type = ReachState
			situation = FLYING
			minSpeed = 1029
			title = Exceed 1029 m/s (Mach 3)
		}
	}
}
```

- [ ] **Step 3: Create Hypersonic.cfg**

```cfg
// Hypersonic.cfg — Mach 5, bleeding edge
// Mach 5 = 1715 m/s. This is basically a spaceplane that hasn't left the atmosphere yet.

CONTRACT_TYPE
{
	name = BKML_Hypersonic
	group = BadgKatMilestones

	title = Hypersonic — Mach 5
	genericTitle = Reach Mach 5
	description = The speed records keep falling. Mach 5 is the threshold of hypersonic flight — at this speed the atmosphere is more plasma than air. The line between aircraft and spacecraft is getting very blurry. R&D suspects that the next step after this is orbit itself.
	synopsis = Exceed Mach 5 (1715 m/s) in a winged vessel.
	completedMessage = Mach 5! The aircraft is officially faster than most rockets at this altitude. The pilot reports the sky was black and the ground was a blur. The SSTO program just got its first test data.

	prestige = Exceptional
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 50000.0
	rewardReputation = 10.0
	rewardScience = 5.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = Mach3Done
		type = CompleteContract
		contractType = BKML_Mach3
	}

	PARAMETER
	{
		name = HypersonicParam
		type = VesselParameterGroup
		title = Reach Mach 5

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
			name = GoHypersonic
			type = ReachState
			situation = FLYING
			minSpeed = 1715
			title = Exceed 1715 m/s (Mach 5)
		}
	}
}
```

- [ ] **Step 4: Commit**

```bash
git add ContractPacks/Milestones/SoundBarrier.cfg ContractPacks/Milestones/Mach3.cfg ContractPacks/Milestones/Hypersonic.cfg
git commit -m "feat: add speed milestone contracts (Mach 1, 3, 5)"
```

---

### Task 6: Aviation milestones — Destination flights

**Files:**
- Create: `ContractPacks/Milestones/IslandHop.cfg`
- Create: `ContractPacks/Milestones/DessertRun.cfg`
- Create: `ContractPacks/Milestones/WoomerangDash.cfg`

- [ ] **Step 1: Create IslandHop.cfg**

```cfg
// IslandHop.cfg — Fly to the Island Airfield
// Short hop across the bay, ~50km. Your first cross-country flight.

CONTRACT_TYPE
{
	name = BKML_IslandHop
	group = BadgKatMilestones

	title = Island Hop — Fly to the Island Airfield
	genericTitle = Fly to the Island Airfield
	description = There is a runway across the bay from KSC — the Island Airfield. It is only about 50 kilometers away, but getting there by air and landing in one piece is a real test of aircraft design and piloting skill. Pack enough fuel, keep the wings level, and try not to overshoot the runway.
	synopsis = Fly a winged vessel to the Island Airfield and land.
	completedMessage = Touchdown at the Island Airfield! That is our first successful cross-country flight. The locals are waving from the beach. Someone left a flag here — apparently Jeb visited by boat last summer.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 20000.0
	rewardReputation = 5.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = FirstFlightDone
		type = CompleteContract
		contractType = BKML_FirstFlight
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		WAYPOINT
		{
			name = Island Airfield
			icon = marker
			latitude = -1.52
			longitude = -71.97
			altitude = 0
		}
	}

	PARAMETER
	{
		name = IslandHopParam
		type = VesselParameterGroup
		title = Fly to the Island Airfield and land

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
			name = ReachIsland
			type = VisitWaypoint
			index = 0
			distance = 500
			title = Reach the Island Airfield (within 500m)
		}

		PARAMETER
		{
			name = LandSafely
			type = ReachState
			situation = LANDED
			title = Land safely
		}
	}
}
```

- [ ] **Step 2: Create DessertRun.cfg**

```cfg
// DessertRun.cfg — Fly to the Dessert Airfield
// Long cross-continent flight, ~900km. Requires Making History DLC.

CONTRACT_TYPE:NEEDS[SquadExpansion]
{
	name = BKML_DessertRun
	group = BadgKatMilestones

	title = Dessert Run — Fly to the Dessert Airfield
	genericTitle = Fly to the Dessert Airfield
	description = The Dessert Airfield is a remote strip out in the badlands, roughly 900 kilometers from KSC. This is a real endurance flight — fuel management, navigation, and a landing on a strip surrounded by nothing but sand and regret. The ground crew out there says they have snacks. They might be lying.
	synopsis = Fly a winged vessel to the Dessert Airfield and land.
	completedMessage = Wheels down at the Dessert Airfield! The crew chief here says we are the first visitors in months. The snacks were real — slightly stale, but real. Our aircraft has proven it can handle long-range missions.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 30000.0
	rewardReputation = 6.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = IslandHopDone
		type = CompleteContract
		contractType = BKML_IslandHop
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		WAYPOINT
		{
			name = Dessert Airfield
			icon = marker
			latitude = -6.56
			longitude = -144.04
			altitude = 0
		}
	}

	PARAMETER
	{
		name = DessertRunParam
		type = VesselParameterGroup
		title = Fly to the Dessert Airfield and land

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
			name = ReachDessert
			type = VisitWaypoint
			index = 0
			distance = 500
			title = Reach the Dessert Airfield (within 500m)
		}

		PARAMETER
		{
			name = LandSafely
			type = ReachState
			situation = LANDED
			title = Land safely
		}
	}
}
```

- [ ] **Step 3: Create WoomerangDash.cfg**

```cfg
// WoomerangDash.cfg — Fly to Woomerang Launch Site
// Polar-ish flight, ~1200km. The longest aviation milestone. Requires Making History DLC.

CONTRACT_TYPE:NEEDS[SquadExpansion]
{
	name = BKML_WoomerangDash
	group = BadgKatMilestones

	title = Woomerang Dash — Fly to Woomerang
	genericTitle = Fly to the Woomerang Launch Site
	description = Woomerang is about as far from KSC as you can get without leaving the continent — over 1200 kilometers north, near the tundra line. This is the ultimate endurance flight. Fuel will be tight, the terrain is unforgiving, and the landing strip was built by optimists. But if we can fly here, we can fly anywhere on Kerbin.
	synopsis = Fly a winged vessel to Woomerang and land.
	completedMessage = Landed at Woomerang! The ground crew is bundled up in parkas and wondering why we did not just launch from here. Our aircraft has crossed the continent — every corner of Kerbin is now within reach by air.

	prestige = Exceptional
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 40000.0
	rewardReputation = 8.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = DessertRunDone
		type = CompleteContract
		contractType = BKML_DessertRun
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		WAYPOINT
		{
			name = Woomerang Launch Site
			icon = marker
			latitude = 45.29
			longitude = 136.11
			altitude = 0
		}
	}

	PARAMETER
	{
		name = WoomerangDashParam
		type = VesselParameterGroup
		title = Fly to Woomerang and land

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
			name = ReachWoomerang
			type = VisitWaypoint
			index = 0
			distance = 500
			title = Reach Woomerang Launch Site (within 500m)
		}

		PARAMETER
		{
			name = LandSafely
			type = ReachState
			situation = LANDED
			title = Land safely
		}
	}
}
```

- [ ] **Step 4: Commit**

```bash
git add ContractPacks/Milestones/IslandHop.cfg ContractPacks/Milestones/DessertRun.cfg ContractPacks/Milestones/WoomerangDash.cfg
git commit -m "feat: add destination flight milestones (Island, Dessert, Woomerang)"
```

---

### Task 7: Aviation milestones — Stunt contracts

**Files:**
- Create: `ContractPacks/Milestones/HighSpeedLowPass.cfg`
- Create: `ContractPacks/Milestones/EdgeOfSpace.cfg`

- [ ] **Step 1: Create HighSpeedLowPass.cfg**

```cfg
// HighSpeedLowPass.cfg — Buzz the tower
// Mach 2 below 1000m. G-Effects makes this spicy.

CONTRACT_TYPE
{
	name = BKML_HighSpeedLowPass
	group = BadgKatMilestones

	title = High-Speed Low Pass
	genericTitle = High-Speed Low Pass
	description = The flight test division has a dare for any pilot brave enough — or foolish enough — to try it. Fly a winged vessel at Mach 2 or faster below 1000 meters altitude. The control tower staff have been warned. Jeb has already volunteered three times. Gene said no twice but is running out of excuses.
	synopsis = Exceed Mach 2 (686 m/s) below 1000m altitude in a winged vessel.
	completedMessage = WHOOOOSH! The tower shook, coffee cups went flying, and Gene is pretending to be angry. The flight recorder data shows a clean pass — fast, low, and under control. Mostly. The pilot is already asking about doing it inverted next time.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 25000.0
	rewardReputation = 5.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = SoundBarrierDone
		type = CompleteContract
		contractType = BKML_SoundBarrier
	}

	PARAMETER
	{
		name = HighSpeedLowPassParam
		type = VesselParameterGroup
		title = High-speed low pass

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
			name = BuzzTheTower
			type = ReachState
			situation = FLYING
			minSpeed = 686
			maxAltitude = 1000
			title = Exceed 686 m/s (Mach 2) below 1000m
		}
	}
}
```

- [ ] **Step 2: Create EdgeOfSpace.cfg**

```cfg
// EdgeOfSpace.cfg — Push an aircraft to its ceiling
// Fly above 18,000m while still in atmosphere (FLYING, not SUB_ORBITAL)

CONTRACT_TYPE
{
	name = BKML_EdgeOfSpace
	group = BadgKatMilestones

	title = Edge of Space
	genericTitle = Edge of Space
	description = How high can a winged aircraft go before the atmosphere gives out? The science team wants data from the upper atmosphere — that thin slice of sky where jet engines sputter and wings barely bite. Push an aircraft above 18,000 meters while still maintaining aerodynamic flight. Up there, the sky is black and the horizon curves.
	synopsis = Fly a winged vessel above 18,000m while still in the atmosphere.
	completedMessage = 18,000 meters and still flying! The pilot reports the sky was dark blue fading to black, and the controls felt like they were made of suggestions. The upper atmosphere data is invaluable — and the photos are going on the break room wall.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 30000.0
	rewardReputation = 6.0
	rewardScience = 3.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = SoundBarrierDone
		type = CompleteContract
		contractType = BKML_SoundBarrier
	}

	PARAMETER
	{
		name = EdgeOfSpaceParam
		type = VesselParameterGroup
		title = Fly to the edge of space

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
			name = HighAlt
			type = ReachState
			situation = FLYING
			minAltitude = 18000
			title = Fly above 18,000m in atmosphere
		}
	}
}
```

- [ ] **Step 3: Commit**

```bash
git add ContractPacks/Milestones/HighSpeedLowPass.cfg ContractPacks/Milestones/EdgeOfSpace.cfg
git commit -m "feat: add stunt milestones (HighSpeedLowPass, EdgeOfSpace)"
```

---

### Task 8: SSTO milestones

**Files:**
- Create: `ContractPacks/Milestones/SuborbitalSpaceplane.cfg`
- Create: `ContractPacks/Milestones/OrbitalSpaceplane.cfg`
- Create: `ContractPacks/Milestones/SpaceplaneDocking.cfg`
- Create: `ContractPacks/Milestones/LaytheSSTO.cfg`

- [ ] **Step 1: Create SuborbitalSpaceplane.cfg**

```cfg
// SuborbitalSpaceplane.cfg — Bridge from aviation to spaceflight
// Reach space with wings and return. The first spaceplane.

CONTRACT_TYPE
{
	name = BKML_SuborbitalSpaceplane
	group = BadgKatMilestones

	title = Suborbital Spaceplane
	genericTitle = Suborbital Spaceplane
	description = We have pushed aircraft to Mach 5 and the edge of space. The next step is obvious — go all the way. Build a vessel with wings that can leave the atmosphere, touch space, and glide back to a runway landing. No parachutes, no splashdowns. This is a spaceplane — it goes up with wings and comes home on them.
	synopsis = Reach suborbital space in a winged vessel and return home.
	completedMessage = The spaceplane has landed! Wings took us to space and wings brought us home. The SSTO program is officially underway. The next question is not if we can reach orbit — it is when.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 45000.0
	rewardReputation = 8.0
	rewardScience = 5.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = HypersonicOrEdge
		type = Any

		REQUIREMENT
		{
			name = HypersonicDone
			type = CompleteContract
			contractType = BKML_Hypersonic
		}

		REQUIREMENT
		{
			name = EdgeOfSpaceDone
			type = CompleteContract
			contractType = BKML_EdgeOfSpace
		}
	}

	PARAMETER
	{
		name = SuborbitalSpaceplaneParam
		type = VesselParameterGroup
		title = Suborbital spaceplane flight

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
			name = ReachSpace
			type = ReachState
			situation = SUB_ORBITAL
			title = Reach suborbital space
		}
	}

	PARAMETER
	{
		name = ReturnHome
		type = ReturnHome
		title = Return home safely
	}
}
```

- [ ] **Step 2: Create OrbitalSpaceplane.cfg**

```cfg
// OrbitalSpaceplane.cfg — Orbit and return with wings
// The classic SSTO achievement.

CONTRACT_TYPE
{
	name = BKML_OrbitalSpaceplane
	group = BadgKatMilestones

	title = Orbital Spaceplane
	genericTitle = Orbital Spaceplane
	description = Suborbital was the warm-up. Now the real challenge — build a winged vessel that can reach stable orbit around Kerbin and return to land on a runway. This is the dream of reusable spaceflight. No boosters left behind, no stages dropped in the ocean. Everything that goes up comes back down in one piece.
	synopsis = Orbit Kerbin in a winged vessel and land safely.
	completedMessage = SSTO achieved! A winged vessel has reached orbit and returned to the runway — nothing dropped, nothing wasted. This changes everything. Reusable spaceflight is here.

	prestige = Exceptional
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 60000.0
	rewardReputation = 12.0
	rewardScience = 5.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = SuborbitalDone
		type = CompleteContract
		contractType = BKML_SuborbitalSpaceplane
	}

	PARAMETER
	{
		name = OrbitalSpaceplaneParam
		type = VesselParameterGroup
		title = Orbital spaceplane flight

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
			name = Orbit
			type = Orbit
			targetBody = HomeWorld()
			situation = ORBITING
			minAltitude = 0
			title = Reach stable orbit
		}
	}

	PARAMETER
	{
		name = ReturnHome
		type = ReturnHome
		title = Return home and land safely
	}
}
```

- [ ] **Step 3: Create SpaceplaneDocking.cfg**

```cfg
// SpaceplaneDocking.cfg — Reusable resupply vehicle concept
// Fly to orbit with wings and dock with a station.

CONTRACT_TYPE
{
	name = BKML_SpaceplaneDocking
	group = BadgKatMilestones

	title = Spaceplane Station Rendezvous
	genericTitle = Spaceplane Station Rendezvous
	description = Now that we can fly to orbit on wings, the logistics team has an idea — why keep throwing away rockets for every station resupply? Build a spaceplane that can fly to orbit, dock with a station, and return to the runway for reuse. This is the future of affordable spaceflight.
	synopsis = Fly a winged vessel to orbit and dock with a station.
	completedMessage = The spaceplane has docked with the station and returned home. Reusable orbital logistics — what used to cost a whole rocket now costs a tank of fuel and a runway. The accountants are thrilled for the first time in the history of the space program.

	prestige = Exceptional
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 80000.0
	rewardReputation = 15.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = OrbitalDone
		type = CompleteContract
		contractType = BKML_OrbitalSpaceplane
	}

	REQUIREMENT
	{
		name = HasStation
		type = Expression
		expression = AllVessels().Where(v => v.VesselType() == Station).Count() >= 1
		title = Must have at least one station deployed
	}

	PARAMETER
	{
		name = SpaceplaneDockingParam
		type = VesselParameterGroup
		title = Spaceplane station rendezvous

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
			name = Orbit
			type = Orbit
			targetBody = HomeWorld()
			situation = ORBITING
			minAltitude = 0
			title = Reach orbit
		}

		PARAMETER
		{
			name = DockWithStation
			type = Docking
			title = Dock with a station
		}
	}

	PARAMETER
	{
		name = ReturnHome
		type = ReturnHome
		title = Return home and land safely
	}
}
```

- [ ] **Step 4: Create LaytheSSTO.cfg**

```cfg
// LaytheSSTO.cfg — The ultimate aviation achievement
// Fly a winged vessel in Laythe's atmosphere. The contract people screenshot.

CONTRACT_TYPE
{
	name = BKML_LaytheSSTO
	group = BadgKatMilestones

	title = Laythe SSTO — Wings on Another World
	genericTitle = Fly on Laythe
	description = Laythe. An ocean moon orbiting a gas giant, with an atmosphere thick enough to fly in. Everything the aviation program has built toward leads here — from the first wobbly flight at KSC to wings on an alien world. Get a spaceplane to Laythe and fly in its atmosphere. This is not just an achievement. This is the achievement.
	synopsis = Fly a winged vessel in Laythe atmosphere.
	completedMessage = Wings on Laythe. A kerbal is flying through an alien sky, over alien oceans, around an alien gas giant. The entire space program — every test flight, every speed record, every failed landing — led to this moment. Someone at KSC is definitely crying. Several someones.

	prestige = Exceptional
	targetBody = Laythe
	maxCompletions = 1

	rewardFunds = 120000.0
	rewardReputation = 25.0
	rewardScience = 15.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = OrbitalDone
		type = CompleteContract
		contractType = BKML_OrbitalSpaceplane
	}

	REQUIREMENT
	{
		name = ReachedLaythe
		type = Expression
		expression = ReachedBodies().Where(b => b.Name() == "Laythe").Count() >= 1
		title = Must have reached Laythe
	}

	PARAMETER
	{
		name = LaytheSSTOParam
		type = VesselParameterGroup
		title = Fly on Laythe

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
			name = FlyOnLaythe
			type = ReachState
			targetBody = Laythe
			situation = FLYING
			title = Fly in Laythe atmosphere
		}
	}
}
```

- [ ] **Step 5: Commit**

```bash
git add ContractPacks/Milestones/SuborbitalSpaceplane.cfg ContractPacks/Milestones/OrbitalSpaceplane.cfg ContractPacks/Milestones/SpaceplaneDocking.cfg ContractPacks/Milestones/LaytheSSTO.cfg
git commit -m "feat: add SSTO milestone chain (suborbital through Laythe)"
```

---

### Task 9: Rover milestones

**Files:**
- Create: `ContractPacks/Milestones/FirstRover.cfg`
- Create: `ContractPacks/Milestones/RoverSurvey.cfg`

- [ ] **Step 1: Create FirstRover.cfg**

```cfg
// FirstRover.cfg — Day one rover milestone
// Drive a wheeled vehicle. Simple, immediate, available from start.

CONTRACT_TYPE
{
	name = BKML_FirstRover
	group = BadgKatMilestones

	title = First Rover Drive
	genericTitle = First Rover Drive
	description = The engineering team has bolted some wheels to a probe core and called it a rover. Time to see if it actually works. Drive a wheeled vehicle on the surface. Bonus points if it does not flip over immediately. (There are no bonus points.)
	synopsis = Drive a wheeled vehicle on the surface.
	completedMessage = The rover is rolling! It only flipped twice during testing, which the engineers are calling a personal best. Surface exploration by wheel is now a viable option for the space program.

	prestige = Trivial
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 8000.0
	rewardReputation = 3.0
	rewardScience = 1.0
	failureFunds = 0
	advanceFunds = 0

	PARAMETER
	{
		name = FirstRoverParam
		type = VesselParameterGroup
		title = Drive a rover

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
			name = IsDriving
			type = ReachState
			situation = LANDED
			minSpeed = 1
			title = Be moving on the surface
		}
	}
}
```

- [ ] **Step 2: Create RoverSurvey.cfg**

```cfg
// RoverSurvey.cfg — Repeatable rover science missions
// Drive to waypoints on a body, collect science. Gives rovers ongoing purpose.

CONTRACT_TYPE
{
	name = BKMC_RoverSurvey
	group = BadgKatMissionControl

	title = Rover survey of @/targetBody1
	genericTitle = Rover surface survey
	description = The Science Division has identified survey sites on @/targetBody1 that can only be reached by surface vehicle. Drive a rover to the marked locations and collect science at each waypoint. This is the kind of slow, methodical work that makes probes jealous — no orbiting allowed, only driving.
	genericDescription = Drive a rover to survey sites on a body and collect science.
	synopsis = Drive a rover to two survey sites on @/targetBody1 and collect science.
	completedMessage = Survey complete! The rover data from both sites is already changing our understanding of @/targetBody1 surface geology. The Science Division wants more — they always want more.

	prestige = Significant
	targetBody = @/targetBody1

	agent = Mission Control

	maxSimultaneous = 1
	maxCompletions = 0

	minExpiry = 30
	maxExpiry = 90

	rewardFunds = Random(30000.0, 60000.0)
	rewardReputation = Random(5.0, 10.0)
	rewardScience = 8.0
	failureReputation = Random(2.0, 5.0)
	failureFunds = 10000.0
	advanceFunds = 15000.0

	DATA
	{
		type = CelestialBody
		requiredValue = true
		uniquenessCheck = CONTRACT_ACTIVE
		targetBody1 = LandedBodies().Where(b => !b.IsHomeWorld() && b.HasSurface()).SelectUnique()
		title = Must have landed on a non-homeworld body with a surface
	}

	REQUIREMENT
	{
		name = HasLanded
		type = Expression
		expression = LandedBodies().Where(b => !b.IsHomeWorld()).Count() >= 1
		title = Must have landed on at least one non-homeworld body
	}

	REQUIREMENT
	{
		name = Cooldown
		type = CompleteContract
		contractType = BKMC_RoverSurvey
		minCount = 0
		cooldownDuration = 45d
	}

	BEHAVIOUR
	{
		name = WaypointGenerator
		type = WaypointGenerator

		RANDOM_WAYPOINT
		{
			name = Survey Site Alpha
			icon = marker
			targetBody = @/targetBody1
			altitude = 0.0
			waterAllowed = false
		}

		RANDOM_WAYPOINT
		{
			name = Survey Site Beta
			icon = marker
			targetBody = @/targetBody1
			altitude = 0.0
			waterAllowed = false
		}
	}

	PARAMETER
	{
		name = RoverSurveyParam
		type = VesselParameterGroup
		title = Rover survey of @/targetBody1

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
			name = VisitAlpha
			type = VisitWaypoint
			index = 0
			distance = 500
			title = Drive to Survey Site Alpha
		}

		PARAMETER
		{
			name = ScienceAlpha
			type = CollectScience
			targetBody = @/targetBody1
			situation = SrfLanded
			recoveryMethod = RecoverOrTransmit
			title = Collect surface science at Site Alpha
		}

		PARAMETER
		{
			name = VisitBeta
			type = VisitWaypoint
			index = 1
			distance = 500
			title = Drive to Survey Site Beta
		}

		PARAMETER
		{
			name = ScienceBeta
			type = CollectScience
			targetBody = @/targetBody1
			situation = SrfLanded
			recoveryMethod = RecoverOrTransmit
			title = Collect surface science at Site Beta
		}
	}
}
```

- [ ] **Step 3: Commit**

```bash
git add ContractPacks/Milestones/FirstRover.cfg ContractPacks/Milestones/RoverSurvey.cfg
git commit -m "feat: add rover contracts (FirstRover milestone, RoverSurvey repeatable)"
```

---

### Task 10: Exploration — Replace ReachSpace, add early crewed track

**Files:**
- Modify: `ContractPacks/Exploration/FirstSteps.cfg`
- Create: `ContractPacks/Exploration/EarlyCrewed.cfg`
- Create: `ContractPacks/Exploration/UnmannedOrbit.cfg`

- [ ] **Step 1: Update FirstSteps.cfg — replace ReachSpace with UnmannedSuborbital**

Replace the `BKEX_ReachSpace` CONTRACT_TYPE in `FirstSteps.cfg` with:

```cfg
CONTRACT_TYPE
{
	name = BKEX_UnmannedSuborbital
	group = BadgKatExploration

	title = Unmanned: Reach Space
	genericTitle = Send a probe to space
	description = Our first science is in, and the data is clear — there is more to learn above the atmosphere. Before we risk any kerbals, send an unmanned probe to suborbital space. Let the machines go first. This is how responsible space programs operate.
	synopsis = Launch an unmanned vessel to suborbital space.
	completedMessage = The probe has touched space! Telemetry confirms suborbital trajectory. The scientists are already arguing about what to send next — but one thing is certain, orbit is the next goal.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 30000.0
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
		name = UnmannedSuborbitalParam
		type = VesselParameterGroup
		title = Reach space with an unmanned vessel

		PARAMETER
		{
			name = IsUnmanned
			type = HasCrew
			maxCrew = 0
			title = Must be unmanned
		}

		PARAMETER
		{
			name = ReachSpace
			type = ReachState
			situation = SUB_ORBITAL
			title = Reach suborbital space
		}
	}
}
```

Keep the `BKEX_FirstScience` CONTRACT_TYPE unchanged.

- [ ] **Step 2: Create UnmannedOrbit.cfg**

```cfg
// UnmannedOrbit.cfg — First unmanned orbit of homeworld
// Gates ProbeFlyby and CrewedHomeOrbit

CONTRACT_TYPE
{
	name = BKEX_UnmannedOrbit
	group = BadgKatExploration

	title = Unmanned: Orbit Homeworld
	genericTitle = Place a probe in orbit
	description = Suborbital was just the beginning. Now we need to go all the way around — place an unmanned probe in stable orbit. This is the foundation everything else is built on. Relays, stations, interplanetary transfers — it all starts with proving we can stay up there.
	synopsis = Place an unmanned vessel in stable orbit around homeworld.
	completedMessage = Orbit achieved! The probe is circling the planet, sending back data with every pass. The tracking stations are locked on. The next step is reaching for the Mun — or wherever the telescopes point us.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 45000.0
	rewardReputation = 8.0
	rewardScience = 5.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = UnmannedSuborbitalDone
		type = CompleteContract
		contractType = BKEX_UnmannedSuborbital
	}

	PARAMETER
	{
		name = UnmannedOrbitParam
		type = VesselParameterGroup
		title = Orbit homeworld with an unmanned vessel

		PARAMETER
		{
			name = IsUnmanned
			type = HasCrew
			maxCrew = 0
			title = Must be unmanned
		}

		PARAMETER
		{
			name = Orbit
			type = Orbit
			targetBody = HomeWorld()
			situation = ORBITING
			minAltitude = 0
			title = Achieve stable orbit
		}
	}
}
```

- [ ] **Step 3: Create EarlyCrewed.cfg**

```cfg
// EarlyCrewed.cfg — Early crewed milestones
// Upper atmosphere and suborbital crewed flights, gated behind unmanned equivalents.
// The "we are putting a kerbal on a rocket" contracts.

// ============================================================================
// CREWED UPPER ATMOSPHERE — Not quite space, but higher than planes
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_CrewedUpperAtmo
	group = BadgKatExploration

	title = Crewed: Upper Atmosphere
	genericTitle = Send a crew to the upper atmosphere
	description = The probes have shown us the sky above. Now it is time to put a kerbal up there — not all the way to space, not yet, but high enough that the sky turns dark and the ground gets small. Fly a crewed vessel above 18,000 meters. This is the first step toward putting kerbals in space.
	synopsis = Fly a crewed vessel above 18,000m in the atmosphere.
	completedMessage = The crew has seen the curve of the horizon and the darkness above. They are back safely, and their report is simple — they want to go higher. Much higher.

	prestige = Significant
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 20000.0
	rewardReputation = 5.0
	rewardScience = 2.0
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
		name = CrewedUpperAtmoParam
		type = VesselParameterGroup
		title = Crewed upper atmosphere flight

		PARAMETER
		{
			name = HasCrew
			type = HasCrew
			minCrew = 1
			title = Must have at least 1 crew member
		}

		PARAMETER
		{
			name = HighAlt
			type = ReachState
			situation = FLYING
			minAltitude = 18000
			title = Fly above 18,000m in atmosphere
		}
	}
}

// ============================================================================
// CREWED SUBORBITAL — First kerbal in space
// ============================================================================
CONTRACT_TYPE
{
	name = BKEX_CrewedSuborbital
	group = BadgKatExploration

	title = Crewed: Reach Space
	genericTitle = Send a crew to space
	description = Probes have reached space. Crews have reached the upper atmosphere. Now it is time for the big one — send a kerbal to suborbital space and bring them home safely. This is the moment the space program has been building toward. The whole world will be watching.
	synopsis = Send a crewed vessel to suborbital space and return safely.
	completedMessage = A kerbal has been to space and lived to tell about it! The capsule is recovered, the crew is grinning, and the space program has just changed forever. Next stop — orbit.

	prestige = Exceptional
	targetBody = HomeWorld()
	maxCompletions = 1

	rewardFunds = 50000.0
	rewardReputation = 10.0
	rewardScience = 5.0
	failureFunds = 0
	advanceFunds = 0

	REQUIREMENT
	{
		name = UnmannedSuborbitalDone
		type = CompleteContract
		contractType = BKEX_UnmannedSuborbital
	}

	REQUIREMENT
	{
		name = CrewedUpperAtmoDone
		type = CompleteContract
		contractType = BKEX_CrewedUpperAtmo
	}

	PARAMETER
	{
		name = CrewedSuborbitalParam
		type = VesselParameterGroup
		title = Crewed suborbital flight

		PARAMETER
		{
			name = HasCrew
			type = HasCrew
			minCrew = 1
			title = Must have at least 1 crew member
		}

		PARAMETER
		{
			name = ReachSpace
			type = ReachState
			situation = SUB_ORBITAL
			title = Reach suborbital space
		}
	}

	PARAMETER
	{
		name = ReturnHome
		type = ReturnHome
		title = Return crew safely home
	}
}
```

- [ ] **Step 4: Commit**

```bash
git add ContractPacks/Exploration/FirstSteps.cfg ContractPacks/Exploration/UnmannedOrbit.cfg ContractPacks/Exploration/EarlyCrewed.cfg
git commit -m "feat: split early exploration into unmanned/crewed tracks"
```

---

### Task 11: Update CrewedExploration gates and ProbeFlyby query

**Files:**
- Modify: `ContractPacks/Exploration/CrewedExploration.cfg`
- Modify: `ContractPacks/Exploration/ProbeExploration.cfg`

- [ ] **Step 1: Update CrewedHomeOrbit gates in CrewedExploration.cfg**

In the `BKEX_CrewedHomeOrbit` CONTRACT_TYPE, add two new REQUIREMENT blocks after the existing `HasOrbited` requirement:

```cfg
	REQUIREMENT
	{
		name = UnmannedOrbitDone
		type = CompleteContract
		contractType = BKEX_UnmannedOrbit
		title = Must have completed unmanned orbit
	}

	REQUIREMENT
	{
		name = CrewedSuborbitalDone
		type = CompleteContract
		contractType = BKEX_CrewedSuborbital
		title = Must have completed crewed suborbital
	}
```

Also remove the placeholder `NotCrewedOrbit` requirement (the one with `expression = true` that does nothing):

```cfg
	// DELETE THIS BLOCK:
	REQUIREMENT
	{
		name = NotCrewedOrbit
		type = Expression
		// This will be offered once — after probe orbit, before crewed orbit
		expression = true
		title = Available after first probe orbit of homeworld
	}
```

- [ ] **Step 2: Update ProbeFlyby to not use NextUnreachedBody()**

In `ProbeExploration.cfg`, replace the `BKEX_ProbeFlyby` DATA node:

Replace:
```cfg
	DATA
	{
		type = CelestialBody
		uniquenessCheck = CONTRACT_ACTIVE
		targetBody1 = NextUnreachedBody()
		title = Target the next unreached body
	}
```

With:
```cfg
	DATA
	{
		type = CelestialBody
		uniquenessCheck = CONTRACT_ACTIVE
		targetBody1 = AllBodies().Where(b => !b.HaveReached() && !b.IsHomeWorld()).SelectUnique()
		title = Target an unreached body
	}
```

Also add a requirement gating behind UnmannedOrbit:

```cfg
	REQUIREMENT
	{
		name = UnmannedOrbitDone
		type = CompleteContract
		contractType = BKEX_UnmannedOrbit
	}
```

And replace the existing `ReachedSpace` requirement with:
```cfg
	REQUIREMENT
	{
		name = UnmannedOrbitDone
		type = CompleteContract
		contractType = BKEX_UnmannedOrbit
	}
```

- [ ] **Step 3: Commit**

```bash
git add ContractPacks/Exploration/CrewedExploration.cfg ContractPacks/Exploration/ProbeExploration.cfg
git commit -m "feat: update exploration gates for unmanned/crewed split"
```

---

### Task 12: Relay deployment contracts

**Files:**
- Create: `ContractPacks/MissionControl/FirstRelay.cfg`
- Create: `ContractPacks/MissionControl/RelayConstellation.cfg`

- [ ] **Step 1: Create FirstRelay.cfg**

```cfg
// FirstRelay.cfg — Deploy your first relay satellite
// Gates behind unmanned orbit. Starts the relay infrastructure chain.

CONTRACT_TYPE
{
	name = BKMC_FirstRelay
	group = BadgKatMissionControl

	title = Deploy first relay satellite
	genericTitle = Deploy a relay satellite
	description = Now that we can reach orbit, Mission Control wants to establish a communications backbone. Deploy a relay satellite in orbit around homeworld. This is the first node in what will eventually become a system-wide communications network. Mark the vessel as a Relay type so the tracking stations can identify it.
	synopsis = Deploy a relay satellite in orbit around homeworld.
	completedMessage = First relay is online! The tracking stations are picking up the signal loud and clear. This is the foundation of our communications network. Three more and we will have full coverage.

	prestige = Significant
	targetBody = HomeWorld()

	agent = Mission Control

	maxCompletions = 1

	rewardFunds = 30000.0
	rewardReputation = 5.0
	rewardScience = 0
	failureFunds = 0
	advanceFunds = 10000.0

	REQUIREMENT
	{
		name = UnmannedOrbitDone
		type = CompleteContract
		contractType = BKEX_UnmannedOrbit
	}

	PARAMETER
	{
		name = FirstRelayParam
		type = VesselParameterGroup
		title = Deploy a relay satellite

		PARAMETER
		{
			name = IsUnmanned
			type = HasCrew
			maxCrew = 0
			title = Must be unmanned
		}

		PARAMETER
		{
			name = IsRelay
			type = VesselIsType
			vesselType = Relay
			title = Vessel must be marked as Relay type
		}

		PARAMETER
		{
			name = InOrbit
			type = Orbit
			targetBody = HomeWorld()
			situation = ORBITING
			minAltitude = 0
			title = Be in stable orbit
		}
	}
}
```

- [ ] **Step 2: Create RelayConstellation.cfg**

```cfg
// RelayConstellation.cfg — Deploy 3 relays around a body
// Repeatable per body. Prereq for RelayRepair and RelayUpgrade.

CONTRACT_TYPE
{
	name = BKMC_RelayConstellation
	group = BadgKatMissionControl

	title = Establish relay network around @/targetBody1
	genericTitle = Establish a relay constellation
	description = A single relay is a start, but real coverage requires a constellation. Deploy at least 3 relay satellites in orbit around @/targetBody1 to ensure continuous communications coverage. This network will support all future missions to the area — probes, crews, and surface operations will all depend on it.
	genericDescription = Deploy three relay satellites around a body for full coverage.
	synopsis = Have at least 3 relay satellites orbiting @/targetBody1.
	completedMessage = Relay constellation established around @/targetBody1! Full communications coverage confirmed. Future missions to this body will have reliable contact with KSC. The network team is celebrating — this is infrastructure that will last.

	prestige = Significant
	targetBody = @/targetBody1

	agent = Mission Control

	maxSimultaneous = 1
	maxCompletions = 0

	minExpiry = 60
	maxExpiry = 180

	rewardFunds = 60000.0
	rewardReputation = 8.0
	rewardScience = 0
	failureReputation = 3.0
	failureFunds = 20000.0
	advanceFunds = 30000.0

	DATA
	{
		type = CelestialBody
		requiredValue = true
		uniquenessCheck = CONTRACT_ACTIVE
		targetBody1 = OrbitedBodies().SelectUnique()
		title = Must have orbited a body
	}

	REQUIREMENT
	{
		name = FirstRelayDone
		type = CompleteContract
		contractType = BKMC_FirstRelay
	}

	REQUIREMENT
	{
		name = HasOrbited
		type = Expression
		expression = OrbitedBodies().Count() >= 1
		title = Must have orbited at least one body
	}

	PARAMETER
	{
		name = ConstellationParam
		type = Expression
		expression = AllVessels().Where(v => v.VesselType() == Relay && v.CelestialBody() == @/targetBody1 && v.IsOrbiting()).Count() >= 3
		title = Have 3+ relay satellites orbiting @/targetBody1
	}
}
```

- [ ] **Step 3: Commit**

```bash
git add ContractPacks/MissionControl/FirstRelay.cfg ContractPacks/MissionControl/RelayConstellation.cfg
git commit -m "feat: add relay deployment contracts (FirstRelay, RelayConstellation)"
```

---

### Task 13: BiomeSurvey contract

**Files:**
- Create: `ContractPacks/MissionControl/BiomeSurvey.cfg`

- [ ] **Step 1: Create BiomeSurvey.cfg**

```cfg
// BiomeSurvey.cfg — Robust biome + experiment science contracts
// "Collect temperature data from the Mun Highlands"
// Replaces Field Research with specific experiment + biome targeting.

CONTRACT_TYPE
{
	name = BKMC_BiomeSurvey
	group = BadgKatMissionControl

	title = Biome survey: @/experiment at @/targetBiome on @/targetBody1
	genericTitle = Biome science survey
	description = The Science Division has identified a gap in our data for @/targetBody1. They need @/experiment data from the @/targetBiome region specifically. This is the kind of targeted science that builds real understanding — not just "collect something from somewhere" but a specific instrument in a specific place.
	genericDescription = Collect a specific type of science data from a specific biome.
	synopsis = Collect @/experiment data from @/targetBiome on @/targetBody1.
	completedMessage = Data received! The @/experiment readings from @/targetBiome are exactly what the Science Division needed. This fills a real gap in our understanding of @/targetBody1. They are already plotting the next survey site.

	prestige = Trivial
	targetBody = @/targetBody1

	agent = Mission Control

	maxSimultaneous = 2
	maxCompletions = 0

	minExpiry = 15
	maxExpiry = 45

	rewardFunds = Random(15000.0, 30000.0)
	rewardReputation = Random(2.0, 5.0)
	rewardScience = 5.0
	failureReputation = Random(1.0, 3.0)
	failureFunds = 5000.0
	advanceFunds = 5000.0

	DATA
	{
		type = CelestialBody
		requiredValue = true
		uniquenessCheck = CONTRACT_ACTIVE
		targetBody1 = LandedBodies().Where(b => !b.IsHomeWorld() && b.HasSurface()).SelectUnique()
		title = Must have landed on a non-homeworld body with a surface
	}

	DATA
	{
		type = Biome
		requiredValue = true
		targetBiome = @/targetBody1.Biomes().Random()
		title = Select a random biome on the target body
	}

	DATA
	{
		type = List<string>
		requiredValue = false
		experimentList = [ "temperatureScan", "barometerScan", "seismicScan", "gravityScan", "evaReport", "surfaceSample", "mysteryGoo", "mobileMaterialsLab" ]
	}

	DATA
	{
		type = string
		requiredValue = true
		experiment = @/experimentList.Random()
		title = Select a random experiment type
	}

	DATA
	{
		type = List<Kerbal>
		requiredValue = false
		availableScientists = AllKerbals().Where(k => k.ExperienceTrait() == Scientist && k.ExperienceLevel() >= 1 && k.RosterStatus() == Available && k.Type() == Crew)
	}

	REQUIREMENT
	{
		name = HasLanded
		type = Expression
		expression = LandedBodies().Where(b => !b.IsHomeWorld()).Count() >= 1
		title = Must have landed on at least one non-homeworld body
	}

	REQUIREMENT
	{
		name = HasScientist
		type = Expression
		expression = @/availableScientists.Count() >= 1
		title = Must have at least one Scientist level 1 or higher
	}

	REQUIREMENT
	{
		name = Cooldown
		type = CompleteContract
		contractType = BKMC_BiomeSurvey
		minCount = 0
		cooldownDuration = 30d
	}

	PARAMETER
	{
		name = BiomeSurveyParam
		type = VesselParameterGroup
		title = Collect @/experiment data from @/targetBiome

		PARAMETER
		{
			name = HasScientist
			type = HasCrew
			trait = Scientist
			minExperience = 1
			minCrew = 1
			title = Include a trained Scientist (level 1+)
		}

		PARAMETER
		{
			name = CollectData
			type = CollectScience
			targetBody = @/targetBody1
			biome = @/targetBiome
			situation = SrfLanded
			recoveryMethod = RecoverOrTransmit
			title = Collect @/experiment data from @/targetBiome
		}
	}
}
```

- [ ] **Step 2: Commit**

```bash
git add ContractPacks/MissionControl/BiomeSurvey.cfg
git commit -m "feat: add BiomeSurvey contract with experiment + biome targeting"
```

---

### Task 14: Final verification and cleanup

- [ ] **Step 1: Verify all new files exist**

Check the full file listing:

```bash
ls -la ContractPacks/Milestones/
ls -la ContractPacks/Exploration/
ls -la ContractPacks/MissionControl/
```

Expected new files:
- `ContractPacks/Milestones/`: Milestones.cfg, FirstFlight.cfg, SoundBarrier.cfg, Mach3.cfg, Hypersonic.cfg, IslandHop.cfg, DessertRun.cfg, WoomerangDash.cfg, HighSpeedLowPass.cfg, EdgeOfSpace.cfg, SuborbitalSpaceplane.cfg, OrbitalSpaceplane.cfg, SpaceplaneDocking.cfg, LaytheSSTO.cfg, FirstRover.cfg, RoverSurvey.cfg
- `ContractPacks/Exploration/`: EarlyCrewed.cfg, UnmannedOrbit.cfg (plus existing updated files)
- `ContractPacks/MissionControl/`: FirstRelay.cfg, RelayConstellation.cfg, BiomeSurvey.cfg

- [ ] **Step 2: Verify no apostrophes in titles/descriptions that could break CC expressions**

Search all new files for apostrophes in title or expression fields:

```bash
grep -rn "title.*'" ContractPacks/Milestones/ ContractPacks/Exploration/EarlyCrewed.cfg ContractPacks/Exploration/UnmannedOrbit.cfg ContractPacks/MissionControl/FirstRelay.cfg ContractPacks/MissionControl/RelayConstellation.cfg ContractPacks/MissionControl/BiomeSurvey.cfg
```

Fix any found by replacing `'` with nothing or rewording.

- [ ] **Step 3: Verify all DATA nodes with requiredValue = true have title fields**

```bash
grep -A5 "requiredValue = true" ContractPacks/Milestones/*.cfg ContractPacks/Exploration/EarlyCrewed.cfg ContractPacks/Exploration/UnmannedOrbit.cfg ContractPacks/MissionControl/FirstRelay.cfg ContractPacks/MissionControl/RelayConstellation.cfg ContractPacks/MissionControl/BiomeSurvey.cfg | grep -B1 "^--\|title"
```

Every `requiredValue = true` DATA block must have a `title` field.

- [ ] **Step 4: Verify all Orbit parameters have minAltitude**

```bash
grep -A3 "type = Orbit" ContractPacks/Milestones/*.cfg ContractPacks/Exploration/EarlyCrewed.cfg ContractPacks/Exploration/UnmannedOrbit.cfg ContractPacks/MissionControl/FirstRelay.cfg ContractPacks/MissionControl/RelayConstellation.cfg | grep "minAltitude"
```

Every `type = Orbit` parameter must have `minAltitude = 0` at minimum.

- [ ] **Step 5: Final commit if any fixes were needed**

```bash
git add -A
git commit -m "fix: address CC validation issues (apostrophes, titles, orbit constraints)"
```
