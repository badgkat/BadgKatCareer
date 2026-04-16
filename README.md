# BadgKatCareer

A CKAN metapackage for a curated KSP 1.12.x career experience. No compiled code — everything is ModuleManager `.cfg` patches and ContractConfigurator contract packs.

## Philosophy

KSP is a fun, goofy, bug-filled game. These patches add light complexity to mission planning without killing the vibe. No hyper-realistic life support, no spreadsheet simulators — just enough to make decisions interesting.

- **Probe-first exploration** — unmanned probes scout before crew follows
- **Three paths from day one** — rockets, Wright-brothers-style planes, or rovers
- **Airplanes matter** — aviation is a real career path, not an afterthought
- **Infrastructure feels alive** — contracts create reasons to maintain stations, resupply bases, and repair relays
- **Snacks, not O2** — life support should be funny, not stressful

## What's Included

### ModuleManager Patches

| File | What it does |
|------|-------------|
| `EarlyExploration.cfg` | Moves parts to `start` tech node to enable three day-one paths |
| `ContractTweaks.cfg` | Disables stock contracts replaced by our contract packs |
| `FuelNames.cfg` | Renames resource display names (LiquidFuel → Kerbosene, etc.) |
| `SnacksIntegration.cfg` | Adds Soil Recyclers/Snack Processors to modded habitats |
| `MPE_ScatterFix.cfg` | Removes broken scatter textures from Minor Planets Expansion bodies |

### Contract Packs

| Pack | Group | Description |
|------|-------|-------------|
| `Exploration/` | `BadgKatExploration` | Two-track progression: unmanned probes then crewed missions |
| `Milestones/` | `BadgKatMilestones` | Aviation milestones, rover contracts, SSTO progression, Kerbin Side base visits |
| `MissionControl/` | `BadgKatMissionControl` | Repeatable infrastructure: relay deployment, station resupply, crew rotation, biome surveys |
| `AnomalySurveyor/` | `AnomalySurveyor` | Anomaly investigation for stock bodies + OPM + Kcalbeloh |

Contracts use ChatterBox BEHAVIOUR nodes for character-driven dialogue with six recurring KSC staff characters (Gene, Wernher, Gus, Mortimer, Walt, Linus).

### Contract Progression

```
Day 1: FirstScience (any science) + FirstRover + Kerbin_Monolith (anomaly)
  ├─► UnmannedSuborbital (probe to space)
  │     ├─► UnmannedOrbit → FirstRelay → RelayConstellation
  │     │     └─► ProbeFlyby → ProbeOrbit → ProbeLanding (per body)
  │     └─► CrewedSuborbital (also needs CrewedUpperAtmo)
  │           └─► CrewedHomeOrbit → CrewedFlyby → CrewedOrbit → CrewedLanding
  └─► FirstFlight → SoundBarrier → Mach3 → Hypersonic
        ├─► IslandHop → KS_Batch1 (4 bases)
        ├─► SoundBarrier → KS_Batch2 (4 bases)
        ├─► Mach3 → KS_Batch3 (4 bases)
        ├─► Hypersonic → KS_Batch4 (7 bases)
        └─► SuborbitalSpaceplane → OrbitalSpaceplane → SpaceplaneDocking
              └─► ExoFlight (any atmospheric non-homeworld body, repeatable)
```

## Install via CKAN

BadgKatCareer is a metapackage. Installing it pulls in the full recommended mod list. Visual/audio mods (Scatterer, Waterfall, ReStock, Chatterer, etc.) are listed as suggestions so lower-spec machines can skip them.

See `BadgKatCareer.ckan` for the full dependency tree.

## Mod List

The full categorized mod list with ~163 mods is maintained in `modlist-categorized.md` at the KSP install root. Categories:

- **Dependencies / Utilities** — libraries and frameworks everything else needs
- **QoL** — autopilot, planning tools, VAB organization, UI improvements
- **Flavor — Visual** — atmosphere, terrain, plumes, skybox, lighting
- **Flavor — Audio** — mission chatter, engine sounds
- **Flavor — IVA** — cockpit interiors, walkable cabins, probe control rooms
- **Parts** — Near Future suite, cryo/atomic/far-future propulsion, stations, rovers, bases
- **Planets** — Kopernicus, OPM, Kcalbeloh, Minor Planets Expansion
- **Gameplay Modifications** — aerodynamics, life support, crew lifecycle, CommNet, construction, thermal

### Hard Dependencies

These mods are required — BadgKatCareer will not function without them:

| Mod | Why |
|-----|-----|
| ModuleManager | Patching engine |
| Community Tech Tree | Tech tree structure all parts reference |
| Community Resource Pack | Shared resource definitions |
| Community Category Kit | Part category definitions |
| B9 Part Switch | Fuel switching system |
| Harmony 2 | Runtime patching library |
| Contract Configurator | Contract system the packs run on |
| ChatterBox | Dialogue system used in contract BEHAVIOURs |
| Research Bodies | Planet discovery mechanic, referenced in contracts |
| Snacks | Life support system, integrated via patches |

### Planet Packs (Soft Required)

Not hard dependencies, but large portions of contract content use `NEEDS[]` checks for these. Without them, those contracts simply won't load:

| Mod | Content that depends on it |
|-----|---------------------------|
| Kopernicus | Required by all planet packs below |
| Outer Planets Mod | Exploration + Anomaly contracts for Sarnus, Urlum, Neidon, Plock |
| Minor Planets Expansion | Exploration contracts for minor bodies |
| Kcalbeloh System | Exploration + Anomaly contracts for the black hole system |
