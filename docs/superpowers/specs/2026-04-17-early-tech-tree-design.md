# Early Game Tech Tree Redesign

Rework the start and tier-1 tech nodes to align with the campaign opening. CTT stays as the base tree; we patch only the early game parts that the campaign specifically needs.

## Goal

Each campaign contract earns enough science to unlock the parts for the next contract. The tech tree becomes a progression engine driven by the anomaly narrative. The player never has parts they don't need yet, and never lacks parts they do.

## Dependencies

- Community Tech Tree (CTT) — base tree, unchanged
- KAX (Kerbal Aircraft eXpansion) — provides prop engines for early flight
- Stock KSP parts
- ReStockPlus (optional, patched if present)

## Scope

Early game only — `start` node and the first two 5-science nodes. Mid-to-late tree is handled by CTT and doesn't need changes. Science balance tuning is a separate future concern if needed.

---

## Node Layout

### Start (0 science)

The absolute minimum to do FirstScience: a command pod and a science experiment. Nothing else.

**Parts at start:**
- `mk1pod_v2` — Mk1 Command Pod (get a kerbal on the pad)
- `GooExperiment` — Mystery Goo Containment Unit (run the experiment)

**Removed from start (compared to old EarlyExploration.cfg):**
- `probeCoreSphere_v2` (Stayputnik) — moved to `engineering101`
- `restock-pod-sphere-1` (ReStockPlus probe) — moved to `engineering101`
- `seatExternalCmd` (command seat) — moved to `engineering101`
- `wingConnector5` (basic wing) — moved to `engineering101`
- `GearFixed` (landing gear) — moved to `engineering101`
- `smallCtrlSrf` (elevon) — moved to `engineering101`
- `wd_prop_engine_s0_1` (Waterbreather) — REMOVED, mod no longer installed
- `miniFuselage` (Mk0 fuel tank) — moved to `engineering101`
- `roverWheel2` (rover wheel) — moved to `basicRocketry`
- `batteryPack` (battery) — moved to `basicRocketry`

### basicRocketry (5 science) — Rover Path

Repurposed as the rover/ground exploration node for the campaign opening. Despite the name, this is where the player gets wheels and batteries to do RoverMonolith.

**Parts moved here:**
- `roverWheel2` — Small rover wheel
- `batteryPack` — Z-100 battery
- `probeCoreSphere_v2` — Stayputnik probe core (rovers are unmanned)
- `restock-pod-sphere-1` — ReStockPlus probe sphere (if installed, via NEEDS)

Stock rocket parts that CTT/stock already place here stay untouched. The player can also start building simple rockets from this node if they want.

### engineering101 (5 science) — Plane Path

Repurposed as the aviation/flight node for the campaign opening. This is where the player gets wings and a prop engine to do FlyToIsland.

**Parts moved here:**
- `KAXVintagePropelator` — Vintage Propelator B (3.5 kN prop engine, from KAX). Includes `ModuleResourceIntake` so satisfies the FlyToIsland intake parameter check.
- `seatExternalCmd` — External Command Seat (open cockpit, Wright Brothers style)
- `wingConnector5` — Wing Connector Type B (basic wing)
- `smallCtrlSrf` — Elevon 1 (control surface)
- `GearFixed` — LY-01 Fixed Landing Gear
- `miniFuselage` — Mk0 Liquid Fuel Fuselage (fuel tank for the prop engine)

### Tier 2+ (15+ science)

No changes. CTT handles rocketry progression from `generalRocketry` onward. By the time the player completes both Act 1 contracts (RoverMonolith + FlyToIsland), they will have earned enough science from Kerbin surface biomes to unlock rocket parts for UnmannedSuborbital.

---

## Implementation

One file change: rewrite `EarlyExploration.cfg` to:
1. Remove all old part moves
2. Move `mk1pod_v2` and `GooExperiment` to `start` (if not already there)
3. `mk1pod_v2` and `GooExperiment` are already at `start` in stock — no move needed
4. Move rover parts to `basicRocketry`
5. Move plane parts to `engineering101`
6. Move KAX Vintage Propelator from `aerodynamicSystems` to `engineering101`
7. Use `NEEDS[KAX]` for the KAX prop engine patch
8. Use `NEEDS[ReStockPlus]` for the ReStockPlus probe patch
9. Remove the dead Waterbreather/WaterDrinker reference

## Science Budget Estimate

- **FirstScience (launchpad):** ~5-8 science (goo + crew report on launchpad + runway)
- **After one Act 1 contract:** ~15-25 science (surface biomes near KSC, monolith area, island)
- **After both Act 1 contracts:** ~30-50 science (enough for several tier-2 nodes including rocketry)
- **Mid-game hump:** Addressed naturally by the campaign sending players to new bodies via the anomaly trail, providing fresh science from unexplored biomes

## What This Does NOT Cover

- Mid-to-late tree restructuring (CTT handles this)
- Science reward balancing on contracts (separate tuning pass)
- Part requirements on contracts (Phase 2 of career redesign)
- Custom tech tree nodes or connections (using stock/CTT structure as-is)
