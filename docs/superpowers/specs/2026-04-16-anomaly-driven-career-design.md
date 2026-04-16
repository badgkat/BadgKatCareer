# Anomaly-Driven Career Redesign — Master Spec

BadgKatCareer career mode restructured around an anomaly-driven narrative campaign. The monolith breadcrumb trail is the story spine; the space program exists because the mystery demands capability.

## Vision

The career feels like a campaign, not a contract browser. Six kerbal characters drive the player forward through dialogue. Anomaly discoveries gate each chapter of the story. The player always has 2-3 story contracts and a handful of radiant contracts available — never a wall of options, never stuck with nothing to do.

Target audience: a smart 12-year-old who wants to be an astronaut. Teaching happens through character personality and mission requirements, never tutorials.

## Dependencies

- ChatterBox v1.0.0 (DymndLab)
- ContractConfigurator >= 2.12.0
- All BadgKatCareer contract packs
- Kerbin Side Remastered (for base contracts)
- OPM, Kcalbeloh, MPE (for outer system content)

## Three-Tool Pattern

Every contract in the career falls into one of three categories:

### Real Contracts
Standard CC contracts with gameplay parameters and ChatterBox dialogue. Accept dialogue briefs the mission; completion dialogue reacts and points forward. These are the campaign beats the player plays.

### Bridge Contracts
Lightweight CC contracts that auto-accept, have zero or trivial parameters, and exist to deliver narrative dialogue based on game state. Multiple bridge contracts with mutually exclusive REQUIREMENT blocks allow context-aware branching — "if the player has done X but not Y, fire this dialogue; if both, fire that one." Bridge scenes are short (2-3 lines), always move the narrative forward, and can include teaching moments when natural.

### Radiant Contracts
Repeatable contracts with no narrative dialogue. Tourism, biome surveys, station resupply, crew rotation, rover surveys. Available when the player has the infrastructure. These provide funds and science between story beats. The economy that keeps the space program running.

---

## The Campaign Arc

### Act 1 — "Something Strange" (Kerbin Surface)

The campaign opens with Gene and Wernher welcoming the player. Weird readings from KSC. Two leads to investigate.

| # | Contract | Type | Gate | Opens |
|---|----------|------|------|-------|
| 1 | `FirstScience` | Real (auto-accept) | None — campaign start | Fork: RoverMonolith + FlyToIsland |
| 2a | `RoverMonolith` | Real (combined rover drive + monolith investigation) | FirstScience | On complete: bridge teases the other fork |
| 2b | `FlyToIsland` | Real (combined first flight + island airfield recon) | FirstScience | On complete: bridge teases the other fork |
| 3 | Bridge: "Both leads checked" | Bridge (auto) | 2a AND 2b | UnmannedSuborbital + Kerbin_Pyramids |
| 4 | `Kerbin_Pyramids` | Real | Bridge 3 | Kerbin_UFO |
| 5 | `Kerbin_UFO` | Real | Kerbin_Pyramids | (feeds anomaly count) |

Aviation milestones (SoundBarrier → Mach3 → Hypersonic) available after FlyToIsland, gating Kerbin Side bases sequentially. These run parallel to the main arc — side content, never blocking the story.

### Act 2 — "The Signal Points Up" (Reaching Space)

The monolith is transmitting off-planet. The space program exists because the mystery demands it.

| # | Contract | Type | Gate | Opens |
|---|----------|------|------|-------|
| 6 | `UnmannedSuborbital` | Real | Bridge 3 | UnmannedOrbit + CrewedUpperAtmo |
| 7 | `UnmannedOrbit` | Real | UnmannedSuborbital | FirstRelay |
| 8 | `CrewedUpperAtmo` | Real | UnmannedSuborbital | CrewedSuborbital |
| 9 | `CrewedSuborbital` | Real | CrewedUpperAtmo + UnmannedSuborbital | CrewedHomeOrbit (with UnmannedOrbit) |
| 10 | `FirstRelay` | Real | UnmannedOrbit | RelayConstellation (Kerbin) |
| 11 | `RelayConstellation` | Real (teach orbital mechanics) | FirstRelay | Mun/Minmus probes |
| 12 | `CrewedHomeOrbit` | Real | CrewedSuborbital + UnmannedOrbit | Crewed Mun/Minmus |

### Act 3 — "Something on the Mun" (Local System)

Relay network up, probes detect anomaly signals on the Mun. The trail continues.

| # | Contract | Type | Gate | Opens |
|---|----------|------|------|-------|
| 13 | `MunProbe` | Real | RelayConstellation | Mun_Monolith |
| 14 | `Mun_Monolith` | Real | MunProbe | Mun_Memorial, Mun_RockArch, Mun_UFO (sequential) |
| 15 | `Mun_Memorial` → `Mun_RockArch` → `Mun_UFO` | Real (sequential) | Mun_Monolith | Feed anomaly count |
| 16 | Bridge: "Trace the signal" | Bridge (auto) | Mun_Monolith + ≥2 Kerbin anomalies | ProbeFlyby_Duna + ProbeFlyby_Eve |
| — | Minmus exploration | Real (parallel side track) | RelayConstellation | Optional, not gating |

### Act 4 — "The Trail" (Interplanetary)

Signals detected across the solar system. Duna and Eve open together; completing either opens Jool.

| # | Contract | Type | Gate | Opens |
|---|----------|------|------|-------|
| 17 | `ProbeFlyby_Duna` | Real | Bridge 16 | Duna anomalies (Face, MSL) + DunaBase |
| 18 | `ProbeFlyby_Eve` | Real | Bridge 16 | Eve exploration |
| 19 | Duna anomalies (Face → MSL, sequential) | Real | ProbeFlyby_Duna | Feed progress |
| 20 | Bridge: "Going deeper" | Bridge (auto) | Duna OR Eve anomaly complete | Jool system opens |
| 21 | Moho, Dres | Real (optional side) | Interplanetary capability | Not gating |

### Act 5 — "The Edge" (Outer System + Endgame)

The trail leads to the outer system. The monoliths converge on something beyond.

| # | Contract | Type | Gate | Opens |
|---|----------|------|------|-------|
| 22 | Jool system probes → anomalies | Real (Vall_Icehenge → Bop_DeadKraken → Jool_Monolith, sequential) | Bridge 20 | Outer system |
| 23 | OPM bodies + anomalies (Tekto, Thatmo, Plock) | Real (optional side arc) | Jool capability | Not gating main story |
| 24 | `Kcalbeloh_Wormhole` | Real | Jool_Monolith + anomaly count threshold | Kcalbeloh_Beyond |
| 25 | `Kcalbeloh_Beyond` → `Kcalbeloh_Dipuc` → `Kcalbeloh_BlackHole` | Real (linear) | Sequential | Endgame |

**Endgame:** Kcalbeloh_BlackHole completion — "The monoliths were an invitation."

### Radiant Contracts (Available When Infrastructure Exists)

- Tourism — available after CrewedSuborbital
- Biome surveys — available per body after landing capability
- Station resupply / crew rotation — available when stations exist
- Base resupply / manufacturing — available when bases exist
- Rover surveys — available after FirstRover capability
- Test flights — available after aviation milestones

No narrative dialogue. Economy and sandbox gameplay between story beats.

---

## The Cast

Six characters, unchanged from the ChatterBox design spec. See `2026-04-15-chatterbox-dialogue-design.md` for full voice guide, verbal tics, models, and colors.

| Character | Model | Color | Role in Campaign |
|---|---|---|---|
| Gene Kerman | `Instructor_Gene` | `#FF8BE08A` | Mission director. Points the player forward. Emotional anchor. |
| Wernher von Kerman | `Instructor_Wernher` | `#FF82B4E8` | Chief scientist. Explains the anomaly data. Teaching through nerding out. |
| Gus Kerman | `Strategy_MechanicGuy` | `#FFFFC078` | Head engineer. Builds what the mission needs. Talks to machines. |
| Mortimer Kerman | `Strategy_Mortimer` | `#FFFFE066` | Finance director. Panics about cost. Comic relief. |
| Walt Kerman | `Strategy_PRGuy` | `#FFC8A0E8` | PR director. Shows up for crewed firsts. Drafts headlines. |
| Linus Kerman | `Strategy_ScienceGuy` | `#FF6ED4C8` | Junior scientist. Found the anomaly signals. Audience surrogate. |

**Bridge contract casting:** Linus reports data findings, Gene decides next steps. Wernher interprets when the science is interesting. Bridges are primarily Linus + Gene or Linus + Wernher scenes.

---

## Contract Condition Improvements

Requirements teach mission design. Multiple right answers, but the player has to think about it. Characters explain *why* in the briefing dialogue.

### Relay Contracts

Adapted from the RemoteTech constellation pattern for stock CommNet:

- `HasAntenna` with `minAntennaPower` scaled to target body distance
- `PartValidation` requiring power source (solar panels or RTG)
- `RelayConstellation` requires 3-4 distinct named vessels (`VesselParameterGroup` with `define` + `IsNotVessel`)
- Near-circular orbits: `maxEccentricity = 0.004`
- Near-equatorial: `maxInclination = 1`
- Minimum periapsis calculated from body radius for line-of-sight coverage: `minPeA = @/targetBody.Radius() * 0.7320508` (3-sat) or `0.4142135` (4-sat)
- `Duration` parameter for network shake-out testing (2 days)
- Dialogue: Wernher teaches why circular + equatorial + altitude = full coverage

### Station Contracts

- `PartValidation` requiring docking port module
- `HasCrewCapacity` minimum for the station type
- Power source required (solar or RTG)
- Dialogue: Gus explains why stations need docking ports and power

### Crewed Missions

- `HasResource` for Snacks, minimum scaled to crew count and estimated mission duration
- `HasCrew` with trait requirements: Pilot for orbital missions, Scientist for anomaly investigations, Engineer for base/station work
- Dialogue: Gene on crew safety, why you bring the right people

### Rovers

- `PartValidation` for wheel module
- Power source required
- Dialogue: Gus on surface mobility, why wheels + power

### Probes

- `HasAntenna` minimum for the target body
- `PartValidation` for science instrument
- Power source required
- Dialogue: Wernher on what makes a probe functional

### Anomaly Investigation

- Recruitment contracts: `SpawnKerbal` + `RecoverKerbal` (see Crew Recruitment below)
- Non-recruitment anomaly contracts: `HasCrew` with `trait = Scientist, minCrew = 1`
- Dialogue: why you bring a scientist to study anomalies

---

## Crew Recruitment System

Key anomaly contracts spawn a named scientist at the investigation site. The player must physically bring them home. On recovery, the kerbal joins the astronaut roster permanently. Kerbin Side bases spawn engineers.

### Recruitment Anomalies (Spawn Named Specialists)

| Contract | Kerbal Name | Trait | Narrative Hook |
|---|---|---|---|
| `Kerbin_Monolith` | Sidmund Kerman | Scientist | KSC's first field researcher |
| `Mun_Monolith` | Dilsby Kerman | Scientist | Deep space signal specialist |
| `Duna_Face` | Kenlie Kerman | Scientist | Exogeologist, alien surface formations |
| `Jool_Monolith` | Barlan Kerman | Scientist | Gas giant specialist |
| `Kcalbeloh_Wormhole` | Megbeth Kerman | Scientist | Theoretical physicist |

### Kerbin Side Base Recruitment (Spawn Engineers)

| Base Batch | Kerbal Name | Trait | Narrative Hook |
|---|---|---|---|
| KS_Batch1 (first base) | Derny Kerman | Engineer | Ground facilities specialist |
| KS_Batch2 (first base) | Rosby Kerman | Engineer | Communications infrastructure |
| KS_Batch3 (first base) | Tansey Kerman | Engineer | Launch site specialist |
| KS_Batch4 (first base) | Nelby Kerman | Engineer | Senior facilities engineer |

### Recruitment Contract Flow

1. Accept dialogue introduces the specialist: "Dr. Sidmund Kerman has volunteered for field work."
2. `SpawnKerbal` BEHAVIOUR places the kerbal at the anomaly/base coordinates
3. Player travels there, picks up the kerbal via EVA
4. `RecoverKerbal` PARAMETER requires bringing them home
5. Completion dialogue: "Welcome aboard, Dr. Kerman."
6. Kerbal joins the player's astronaut roster permanently

### Non-Recruitment Contracts

- Remaining anomaly contracts require `HasCrew` with `trait = Scientist, minCrew = 1`
- Remaining Kerbin Side bases require `HasCrew` with `trait = Engineer, minCrew = 1`
- This forces the player to use their growing specialist roster

---

## Dialogue Rewrite Approach

### What Stays
- All teaching content (Newton's cannonball, center of lift, Mach numbers, Yeager, Gagarin, space history)
- Character voices, verbal tics, personality
- Character models, colors, animations
- Writing rules from the ChatterBox design spec

### What Changes
- **Framing lines** rewritten to connect to the anomaly narrative. "We detected capability" becomes "The signal demands we get higher."
- **Completion lines** now point forward. Every story contract's completion scene connects to the next beat.
- **Some scenes move** if their contract moves in the progression order. Teaching content travels with the scene; framing changes.

### New Scenes Needed
- Bridge contract dialogues (~10) — narrative connectors, short (2-3 lines), can teach when natural
- Recruitment dialogues (~9) — accept scenes introducing spawned specialists
- Combined contract dialogues (~4) — RoverMonolith and FlyToIsland merge previous separate scenes

### Estimated Scene Count
- Reframed existing scenes: ~50
- New bridge scenes: ~10
- New recruitment scenes: ~9
- New combined contract scenes: ~4
- **Total: ~73 scenes**

---

## Phase Breakdown

### Phase 1 — Narrative Restructure + Dialogue Rewrite
The big one. Changes the skeleton of the career.

- Redesign the contract progression tree (new REQUIREMENT blocks across all contracts)
- Create combined contracts: `RoverMonolith` (rover + monolith), `FlyToIsland` (flight + island airfield)
- Create bridge contracts (~10) with mutually exclusive requirements for conditional narrative
- `FirstScience` becomes the sole auto-accept real contract (bridge contracts also auto-accept, but they are narrative-only with no gameplay)
- Rewrite/reframe all ChatterBox dialogue scenes for the anomaly narrative
- Add new bridge, recruitment, and combined contract scenes
- Preserve all teaching content, rearrange as needed for new progression order

**Produces:** A playable campaign with the new progression flow and narrative dialogue. Contract conditions unchanged (tightened in Phase 2).

### Phase 2 — Contract Condition Improvements
Tighten requirements to teach mission design.

- Add `HasAntenna` requirements to relay contracts
- Build relay constellation contract with multi-vessel orbital mechanics enforcement
- Add `PartValidation` requirements: stations (docking ports), rovers (wheels), probes (science instruments)
- Add `HasResource` checks for Snacks on crewed missions
- Add crew trait requirements where appropriate
- Teaching dialogue in briefings for each new requirement

**Depends on Phase 1:** Needs the new contract structure in place.

### Phase 3 — Crew Recruitment
Named specialists join the program through anomaly investigation.

- Add `SpawnKerbal`/`RecoverKerbal` to 5 key anomaly contracts
- Add engineer spawns to 4 Kerbin Side base batches (first base in each batch)
- Add scientist/engineer trait requirements to remaining anomaly and base contracts
- Recruitment-specific dialogue scenes (~9 new scenes)

**Depends on Phase 1:** Needs the anomaly contracts restructured.
**Can interleave with Phase 2:** Trait requirements overlap but are independently implementable.

---

## Scope Summary

- **Story contracts (real):** ~35-40 campaign beats
- **Bridge contracts:** ~10 narrative connectors
- **Radiant contracts:** ~15 repeatable (mostly existing, minor tweaks)
- **ChatterBox scenes total:** ~73
- **New combined contracts:** 2 (RoverMonolith, FlyToIsland)
- **Contracts with tightened conditions:** ~20 (Phase 2)
- **Crew recruitment contracts:** ~9 (Phase 3)
- **Files to modify:** ~55 existing contract files
- **Files to create:** ~12 (bridge contracts + combined contracts)
- **Three phases:** Phase 1 first (prerequisite), then Phase 2 and Phase 3 in either order
