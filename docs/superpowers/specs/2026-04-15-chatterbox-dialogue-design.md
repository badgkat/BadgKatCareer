# ChatterBox Dialogue System Design

BadgKatCareer integration with ChatterBox v1.0.0 to add character-driven dialogue to the contract career experience.

## Goal

Make the career feel like a story being told by a cast of kerbal characters who teach spaceflight concepts, drop space history, and build an in-universe narrative — all through goofy kerbal personality. Target audience: a smart 12-year-old who wants to be an astronaut. Teaching should feel like characters nerding out, never like a tutorial.

## Dependencies

- ChatterBox v1.0.0 (DymndLab) — installed at `GameData/ChatterBox/`
- ContractConfigurator >= 2.12.0
- All BadgKatCareer contract packs (Exploration, Milestones, MissionControl, AnomalySurveyor)

## Approach: Tiered Coverage

Three tiers of dialogue treatment based on narrative importance:

- **Tier 1 — Cinematic (3-8 lines, two speakers, on accept AND complete):** ~15 landmark contracts. Full character banter, teaching moments, space history. The story beats.
- **Tier 2 — Quick exchange (1-2 lines, one speaker, accept OR complete):** ~25 routine contracts. A quip or congrats. Keeps characters present without wearing out welcome.
- **Tier 3 — No ChatterBox:** Repeatable infrastructure contracts. Fire too often for dialogue. Existing description text carries personality.

---

## The Cast

Six characters, each introduced in an early game contract.

### Gene Kerman — Mission Control Director
- **Role:** Sends you on missions, worries about you coming back. Emotional anchor.
- **Voice:** Dry, understated, fatherly. Buries concern under deadpan humor.
- **Verbal tics:** "Not that I'm worried, but...", calls things "perfectly routine" when they aren't, "unscheduled disassembly" for explosions, "Try to bring back more pieces than you left with"
- **Introduced:** FirstScience (accept)
- **Model:** `Instructor_Gene`
- **Color:** `#FF8BE08A` (green — Mission Control, go/no-go)

### Wernher von Kerman — Chief Scientist
- **Role:** The reason you're going anywhere. Explains science, drops space history. Gleefully reckless.
- **Voice:** Academic vocabulary masking chaos. Treats explosions as data. Every disaster is a "learning opportunity."
- **Verbal tics:** Germanic phrasing ("This is most excellent"), cites failed experiments as proof of concept, compares missions to real space history, can't help nerding out
- **Introduced:** FirstScience (accept) — paired with Gene as the opening duo
- **Model:** `Instructor_Wernher`
- **Color:** `#FF82B4E8` (blue — science, cerebral)

### Gus Kerman — Head Engineer / Facilities
- **Role:** Builds and fixes things. Rovers, stations, repairs. Talks about machines like pets.
- **Voice:** Blue-collar, practical, slightly exasperated. Knows every bolt in KSC.
- **Verbal tics:** "She's temperamental" about machines, duct tape as engineering material, skeptical of Wernher, uses "she/her" for all vehicles
- **Introduced:** FirstRover (accept) — he built the rover
- **Model:** `Strategy_MechanicGuy`
- **Color:** `#FFFFC078` (orange — sparks and rust)

### Mortimer Kerman — Finance Director
- **Role:** Pays for everything, wishes you wouldn't. Comic relief through anxiety.
- **Voice:** Nervous, penny-pinching, catastrophizing. Every launch is money on fire.
- **Verbal tics:** Quotes costs mid-sentence, treats fuel as liquid money, "Do you have ANY idea how much that costs?", grudging admissions when things work out
- **Introduced:** UnmannedSuborbital (accept) — first real budget rocket
- **Model:** `Strategy_Mortimer`
- **Color:** `#FFFFE066` (yellow — gold, money)

### Walt Kerman — Public Relations
- **Role:** Shows up when reputation matters. Crewed firsts, big achievements.
- **Voice:** Smarmy PR-speak, always drafting the headline. Performatively enthusiastic.
- **Verbal tics:** "The public is going to LOVE this", drafts press releases in real time, reframes disasters as "unplanned rapid events", genuinely starstruck beneath the polish
- **Introduced:** CrewedUpperAtmo (accept) — first crew on camera
- **Model:** `Strategy_PRGuy`
- **Color:** `#FFC8A0E8` (purple — showbiz)

### Linus Kerman — Junior Scientist
- **Role:** Found the anomaly signals. Audience surrogate — asks what the player is thinking.
- **Voice:** Earnest, nervous, genuinely awed. Appropriately scared of things Wernher finds exciting.
- **Verbal tics:** "Is that... supposed to look like that?", stammers and ellipses when overwhelmed, "Wernher, how concerned should I be?"
- **Introduced:** Kerbin_Monolith (accept) — his gyroscopes are acting up
- **Model:** `Strategy_ScienceGuy`
- **Color:** `#FF6ED4C8` (teal — junior science, distinct from Wernher's blue)

---

## Writing Rules

- **No fourth-wall breaking.** Characters don't know they're in a game.
- **Kerbal terms always.** Kerbosene, Koxide, Snacks. Never "food" or "gasoline."
- **Real science, kerbal context.** Periapsis yes, NASA no. "In 1903, two bicycle mechanics flew for twelve seconds" yes. "On Earth" no.
- **Humor from character, not jokes.** Gene is funny because he's dry. Wernher is funny because he's oblivious to danger. Nobody is trying to be funny.
- **Teaching through nerding out.** Wernher doesn't say "orbit means going sideways fast enough." He says "The trick is to go sideways so fast you keep falling but never hit the ground. Newton figured this out with a thought experiment about a very large cannon."
- **Practical advice as character knowledge.** "The center of lift should sit just behind the center of mass — that way, if the nose dips, the lift pushes it back up. If it is the other way around... well. You will find out quickly." This saves the player an hour without ever feeling like a tooltip.
- **Short lines.** 1-2 sentences per LINE node. Respect the click-through format.
- **Speaker name tokens.** Use `<characterA>` / `<characterB>` in LINE text when characters address each other.

---

## ChatterBox Technical Patterns

### Tier 1 Template (cinematic, two speakers)

```cfg
BEHAVIOUR
{
    name = ChatterBox
    type = ChatterBox
    onState = CONTRACT_ACCEPTED
    title = Scene Title

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

    LINE { speaker = A; text = Line of dialogue here. }
    LINE { speaker = B; text = Response here. }
    // 3-8 lines total
}
```

Tier 1 contracts have **two BEHAVIOUR nodes** — one `onState = CONTRACT_ACCEPTED`, one `onState = CONTRACT_SUCCESS`.

### Tier 2 Template (quick exchange, single speaker)

```cfg
BEHAVIOUR
{
    name = ChatterBox
    type = ChatterBox
    onState = CONTRACT_ACCEPTED
    title = Scene Title

    presentation = Dialogue
    scale = 0.45
    pauseGame = true
    triggerOnce = true

    characterAName = Gene
    characterAModel = Instructor_Gene
    characterAAnimation = true_nodA
    characterAColor = #FF8BE08A

    LINE { speaker = A; text = One or two lines here. }
}
```

Tier 2 contracts have **one BEHAVIOUR node** — either accept or complete, not both.

### Animation Reference

| Animation | Use For |
|---|---|
| `idle` | Default talking |
| `true_nodA` | Agreeing, congratulating |
| `true_smileA` | Pleased, mission success |
| `true_thumbsUp` | Big wins (first orbit, first crew) |
| `idle_wonder` | Anomaly discoveries, mystery |
| `idle_disappointed` | Something went wrong |
| `idle_sadA` | Genuine concern (risky missions) |

---

## Contract Tier Assignments

### Tier 1 — Cinematic Scenes

| Contract | Accept Scene | Complete Scene | Speakers |
|---|---|---|---|
| FirstScience | Program opens, meet Gene + Wernher | "We're in business" — Wernher wants more | Gene, Wernher |
| FirstRover | Gus introduced, built the rover | Proud dad, "She held together" | Gene, Gus |
| FirstFlight | Wright brothers moment, CoL/CoM teaching | Aviation has arrived | Gene, Wernher |
| Kerbin_Monolith | Linus introduced, gyroscopes are weird | "Someone put this here" — mystery begins | Linus, Gene |
| UnmannedSuborbital (accept) | Mortimer introduced via budget panic | — | Gene, Mortimer |
| UnmannedSuborbital (complete) | — | "Space! Actual space!" + Sputnik parallel | Gene, Wernher |
| CrewedUpperAtmo | Walt introduced, first crew on camera | PR frenzy | Gene, Walt |
| UnmannedOrbit | Newton's cannonball, what orbit means | "It will be up there long after we are gone" | Gene, Wernher |
| CrewedSuborbital | First kerbal in space, Gagarin parallel | Gene drops the deadpan — genuine pride | Gene, Wernher |
| CrewedHomeOrbit | Orbit with crew — higher stakes | Crew is floating, wonder | Gene, Wernher |
| ProbeFlyby (first) | Leaving home, interplanetary travel | Data from another world | Wernher, Linus |
| SoundBarrier | (none) | Yeager parallel, Mach 1 explained | Gene, Wernher |
| SuborbitalSpaceplane | SSTO concept — runway to space | Gus: "She held together... barely" | Wernher, Gus |
| Kcalbeloh_Wormhole | "There's a hole in the sky" — Einstein-Rosen | Through the looking glass | Wernher, Linus |
| Kcalbeloh_BlackHole | (none) | End of the monolith road — awe and vertigo | Wernher, Gene |

**Teaching moments per Tier 1 scene:**

| Contract | Teaching Content |
|---|---|
| FirstScience | What science collection is, why data matters, science is incremental |
| FirstRover | Surface exploration without rockets, rovers as science platforms |
| FirstFlight | Lift, wings, center of lift behind center of mass for stability |
| Kerbin_Monolith | Magnetic anomalies, 2001 monolith parallel (in-universe) |
| UnmannedSuborbital | What suborbital means (up and back down), why probes go first |
| UnmannedOrbit | What orbit is (Newton's cannonball), going sideways fast enough |
| CrewedSuborbital | Gagarin/Shepard history, difference between probe and crewed |
| CrewedHomeOrbit | Orbit vs suborbital revisited with crew stakes |
| ProbeFlyby (first) | Interplanetary travel, transfer windows hinted at |
| SoundBarrier | Speed of sound, Mach numbers, Yeager 1947 |
| SuborbitalSpaceplane | What makes SSTOs special — one vehicle, runway to space |
| Kcalbeloh_Wormhole | Einstein-Rosen bridges, wormhole theory |
| Kcalbeloh_BlackHole | Black holes, event horizons, gravity |

### Tier 2 — Quick Exchanges

| Contract(s) | When | Speaker | Content |
|---|---|---|---|
| Mach3 | complete | Wernher | Mach numbers, brief congrats |
| Hypersonic | complete | Gene | Speed pride, brief |
| HighSpeedLowPass | accept | Gene | "Please don't hit anything" |
| EdgeOfSpace | complete | Wernher | Where atmosphere meets vacuum |
| IslandHop | accept | Gene | "There's an island out there..." |
| DessertRun | accept | Gene | Quick DLC airfield flavor |
| WoomerangDash | accept | Gus | Quick DLC airfield flavor |
| KS_Batch1 (4 bases) | accept | Gene | One-liner per base |
| KS_Batch2 (4 bases) | accept | Gene | One-liner per base |
| KS_Batch3 (4 bases) | accept | Gene | One-liner per base |
| KS_Batch4 (7 bases) | accept | Gene | One-liner per base |
| OrbitalSpaceplane | complete | Gus | "She held together!" |
| SpaceplaneDocking | complete | Gene | Brief "well done" |
| FirstRelay | accept | Wernher | Why communication relays matter |
| RelayConstellation | accept | Wernher | Network coverage concept |
| ProbeOrbit | complete | Wernher | Quick science excitement |
| ProbeLanding | complete | Linus | First surface data from another world |
| CrewedFlyby | accept | Gene | Crew safety concern |
| CrewedOrbit | accept | Gene | Brief crew concern |
| CrewedLanding | accept | Gene | "Bring them home" |
| ExoFlight | accept | Wernher | Alien atmosphere excitement |
| Anomaly Surveyor (stock bodies, ~12) | complete | Linus | Quick reaction to each discovery |
| OPM anomalies (3) | complete | Linus | Edge-of-system wonder |
| Kcalbeloh_Beyond | accept | Wernher | Brief narrative continuation |
| Kcalbeloh_Dipuc | complete | Wernher | Gravitational anomaly reaction |

### Tier 3 — No ChatterBox

StationResupply, BaseResupply, CrewRotation, BiomeSurvey, DeepSpaceSurvey, RoverSurvey, RelayRepair, RelayUpgrade, StationMaintenance, BaseManufacturing, EmergencyRescueRepair, TestFlight, PrecisionLanding, LegacyMission, VeteranRetirement.

These are repeatable infrastructure contracts that fire often enough for dialogue to get annoying. The existing `description` and `completedMessage` text already has personality baked in.

---

## Existing DialogBox Migration

The AnomalySurveyor contracts currently use stock CC `DialogBox` BEHAVIOUR nodes. These will be **replaced** with ChatterBox BEHAVIOUR nodes. The old DialogBox nodes are removed and replaced with equivalent (but better) ChatterBox scenes. Contract logic (waypoints, requirements, parameters) stays untouched.

Affected files:
- `Kerbin_Monolith.cfg` (upgraded to Tier 1)
- `Kerbin_Pyramids.cfg`, `Kerbin_UFO.cfg`, `Kerbin_IslandAirfield.cfg` (Tier 2)
- `Kerbin_Monolith_Additional.cfg` (Tier 2)
- `Mun_Monolith.cfg`, `Mun_Memorial.cfg`, `Mun_RockArch.cfg`, `Mun_UFO.cfg` (Tier 2)
- `Bop_DeadKraken.cfg`, `Duna_Face.cfg`, `Duna_MSL.cfg` (Tier 2)
- `Moho_Mohole.cfg`, `Tylo_Cave.cfg`, `Vall_Icehenge.cfg`, `Jool_Monolith.cfg` (Tier 2)
- `OPM_Tekto_Spire.cfg`, `OPM_Thatmo_Monolith.cfg`, `OPM_Plock_Monolith.cfg` (Tier 2)
- `Kcalbeloh_Wormhole.cfg` (Tier 1), `Kcalbeloh_Beyond.cfg`, `Kcalbeloh_Dipuc.cfg` (Tier 2), `Kcalbeloh_BlackHole.cfg` (Tier 1)

---

## Scope Summary

- **~15 Tier 1 contracts:** 2 ChatterBox BEHAVIOURs each (accept + complete) = ~30 scenes, ~150 lines of dialogue
- **~25 Tier 2 contracts** + **~19 Kerbin Side bases** + **~15 anomaly upgrades:** 1 BEHAVIOUR each = ~59 scenes, ~80 lines of dialogue
- **~15 Tier 3 contracts:** No changes
- **Total new ChatterBox BEHAVIOUR nodes:** ~89
- **Total lines of dialogue to write:** ~230
- **Files to modify:** ~55 (add BEHAVIOUR nodes to existing contracts)
- **Files to create:** 0 (all dialogue lives inside existing contract files)
- **Existing DialogBox nodes to replace:** ~20 (in AnomalySurveyor files)
