# BadgKatCareer

Custom ModuleManager patches for a curated KSP career experience.

## Philosophy

KSP is a fun, goofy, bug-filled game. These patches add light complexity to
mission planning without killing the vibe. No hyper-realistic life support,
no spreadsheet simulators — just enough to make decisions interesting.

- **Probe-first capable** — probes are viable early
- **Early surface exploration** — rovers and command seats available before going to space
- **Airplanes matter** — aviation is a real career path, not an afterthought
- **Snacks, not O2** — life support should be funny, not stressful

## Patches

### EarlyExploration.cfg
Moves surface exploration parts earlier in the Community Tech Tree:
- Command seat → Survivability (tier 2, 15 science)
- Rover wheels + body → Basic Science (tier 3, 45 science)

### SnacksIntegration.cfg
Adds Soil Recyclers and Snack Processors to modded crew habitats that Snacks
doesn't patch by default:
- Station Parts Expansion Redux: all habitation modules, inflatables, centrifuges, greenhouses
- Near Future Spacecraft: utility pod
- MarkIV System: crew cabin
- Lynx rover: crew cabin

Without this, long-duration OPM missions need unreasonable amounts of snacks.

## Install via CKAN

BadgKatCareer is a metapackage. Installing it pulls in the full recommended mod
list. Visual mods (Parallax, Scatterer, ReStock, Waterfall) are listed as
suggestions so lower-spec PCs can skip them.

See `BadgKatCareer.ckan` for the full dependency tree.

## Dependencies
- ModuleManager 4.2.3+
- Community Tech Tree
- B9PartSwitch
- Community Resource Pack
