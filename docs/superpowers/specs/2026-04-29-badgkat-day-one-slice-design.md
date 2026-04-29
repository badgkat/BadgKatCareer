# BadgKatCareer — Day One Vertical Slice

First production cfg slice on top of the KDK / KCK / KAK framework stack. End-to-end validation through one full chapter transition with a player choice that has real mechanical teeth.

## Why a slice instead of full Phase 3

The redesign spec (`2026-04-23-director-career-redesign.md`) describes the full Phase 3 content target: 6 characters, 5 chapters, 9 directives, 18 focuses, ~20 memos, 10-15 staff scenes, rep tuning, decay, building scenes, chapter gates, rep stakes. Authoring all of that before validating the framework wiring would be weeks of cfg work that we'd likely have to revise once the actual chapter/trigger/scene plumbing meets KSP at runtime.

This slice is the smallest possible vertical that:
- Activates the admin-building replacement
- Exercises every framework hook (characters, focuses, memos with multiple condition types, building scenes, disposition decay, chapter machine, triggers, reputation, PR campaign, KDK scenes with branching choices, contract gating)
- Validates two real chapter transitions
- Proves the directive-choice mechanic with chapter-2 contract gating

Once it works in-game and we shake out the inevitable wiring bugs, subsequent plans add Ch2 / Ch3 / Ch4 / Ch5 content as pattern repetition.

## Scope

### Chapters

Three chapters covering the slice arc:

| ID | Name | Entry | Exit |
|---|---|---|---|
| 0 | Prologue: Telemetry | Game start (FacilityEntered: SpaceCenter, once) | `BKEX_FirstScience` completes |
| 1 | Day One | Ch0 exit fires | `BKEX_RoverMonolith` AND `BKEX_FlyToIsland` complete (either order) |
| 2 | Into Orbit | Ch1 directive scene resolves | (out of slice scope — stub only) |

### Cast (6 characters)

Per the redesign spec — Gene, Wernher, Gus, Mortimer, Walt, Linus. Color-swatch chips, no portraits in this slice. Default dispositions: Linus = Enthusiastic, Mortimer = Skeptical, all others = Neutral.

### Focuses (6, one per character)

Pure placeholders. Picker UI validation; no game-state queries against focus flags in this slice.

| Character | Title | Flag value |
|---|---|---|
| Gene | Mission Pacing | `pacing` |
| Wernher | Anomaly Research | `anomaly` |
| Gus | Launch Infrastructure | `infra` |
| Mortimer | Cost Optimization | `cost` |
| Walt | Public Outreach | `outreach` |
| Linus | Signal Analysis | `signals` |

### Memos (4-5)

Each exercises a distinct condition type or condition combinator:

1. `mortimer_low_funds` — `FundsBelow` (numeric game-state)
2. `walt_rep_milestone` — `ReputationAbove` (rep-tier read)
3. `gus_day_one_kit` — `ChapterAtLeast` (chapter-state read)
4. `wernher_directive_followup_space` — `FlagEquals` + `ChapterAtLeast` (AND of multiple conditions, including a KDK flag read)
5. `wernher_directive_followup_ground` — same shape, opposite flag value (split-text pattern for branch-specific memos)

### Staff scenes (2)

- `staff_first_recovery` — fires on Ch0→Ch1 transition. Brief Gene+Walt beat.
- `ch1_directive_gene_wernher` — fires on Ch1→Ch2 transition with `OnFacilityEnter = Administration`. Lifts Linus's existing BothLeads opening verbatim, then forks into the directive choice.

### Building scenes (4)

| Facility | Character | Scene |
|---|---|---|
| MissionControl | Gene | `gene_mc_status` |
| TrackingStation | Linus | `linus_tracking_signals` |
| Administration | Mortimer | `mortimer_admin_budget` |
| Administration | Walt | `walt_admin_pr` |

Each is a one-line stub. Validates click→scene plumbing, not voice work.

### Disposition decay (6)

One per character, identical 180-day step structure (Enthusiastic → Supportive → Neutral; Frustrated → Skeptical → Neutral). Compiles to KCK time-triggers; validation is the load-time log line, not in-playtest decay (180 days is far beyond a slice playtest).

### Reputation

Full 5-tier income table, 1.5%/month decay with `resetOnContractComplete = true` and `tierFloors = true`. PR campaign cost = 50000 base + 25000/tier, halts decay 60 days, rep bonus 10. All numbers from the redesign spec, unchanged.

### Activation

```cfg
KERBAL_ADMIN_KIT
{
    replaceAdminBuilding = true
    deskMemoCount = 5
    memoPollSeconds = 5
}
```

### Out of scope

- Ch2 / Ch3 / Ch4 / Ch5 content (chapters, contracts, directives, scenes)
- Other 8 directives
- Other ~16 memos
- Other 12 focuses (full 3/character is post-slice)
- Most staff meeting scenes
- ReputationStake BEHAVIOURs on high-profile contracts
- Portrait artwork (color-swatch only)
- Voice/tone polish — text in this slice is functional placeholder; rewrite pass after in-game validation

## Pre-slice cleanup (kill dead workarounds)

Must run before new content cfg lands so reference rot doesn't compound.

### Files to delete entirely

| File | Reason |
|---|---|
| `ContractPacks/Exploration/BridgeContracts.cfg` | Auto-accept zero-effect ChatterBox-trigger workarounds (4 contracts: BKEX_Bridge_BothLeads, BKEX_Bridge_MunSignals, BKEX_Bridge_TraceSignal, BKEX_Bridge_GoingDeeper). Replaced by KCK trigger + KDK ENQUEUE_SCENE. |
| `ContractPacks/Milestones/FirstFlight.cfg` | `expression = false` deprecation tombstone. Replaced by `BKEX_FlyToIsland`. |
| `ContractPacks/Milestones/FirstRover.cfg` | `expression = false` tombstone. Replaced by `BKEX_RoverMonolith`. |
| `ContractPacks/Milestones/IslandHop.cfg` | `expression = false` tombstone. Replaced by `BKEX_FlyToIsland`. |
| `ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg` | `expression = false` tombstone. Replaced by campaign version. |
| `ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg` | `expression = false` tombstone. Replaced by campaign version. |

### References to repoint

| File | Current ref | Update to |
|---|---|---|
| `ContractPacks/Milestones/KS_Batch1.cfg` (4 occurrences) | `contractType = BKML_IslandHop` | `contractType = BKEX_FlyToIsland` |
| `ContractPacks/Milestones/DessertRun.cfg` (1 occurrence) | `contractType = BKML_IslandHop` | `contractType = BKEX_FlyToIsland` |

### Pre-delete dialogue lift

Several deprecated contracts carry on-character ChatterBox dialogue worth keeping. Before deleting each file, compare its `LINE` blocks against the campaign-version contract and lift any unique dialogue:

| Deprecated source | Campaign target | Notable dialogue |
|---|---|---|
| `Milestones/FirstFlight.cfg` | `BKEX_FlyToIsland` | Wernher's Wright Brothers monologue, Gene's "fire crew standing by" |
| `Milestones/FirstRover.cfg` | `BKEX_RoverMonolith` | Gus's "she pulls a little to the left" intro |
| `Milestones/IslandHop.cfg` | `BKEX_FlyToIsland` | Gene's "navigation practice" briefing |
| `AnomalySurveyor/Kerbin_IslandAirfield.cfg` | `BKEX_FlyToIsland` | Mystery-airfield reveal lines |
| `AnomalySurveyor/Kerbin_Monolith.cfg` | `BKEX_RoverMonolith` | Linus's "structured signal pointing away" lines |

If the campaign version already has equivalent dialogue, just delete the deprecated file. If not, lift the unique LINEs into the campaign contract's `BEHAVIOUR { type = ChatterBox }` block, then delete.

### Cleanup audit

Final grep across `ContractPacks/` for the deprecated names — must return zero matches before commit:

```
BKML_FirstFlight | BKML_FirstRover | BKML_IslandHop |
BKEX_Bridge_BothLeads | BKEX_Bridge_MunSignals |
BKEX_Bridge_TraceSignal | BKEX_Bridge_GoingDeeper |
AS_Kerbin_Monolith | AS_Kerbin_IslandAirfield
```

## Framework additions

Two small features added to KCK and KAK before content cfg lands. Each pays for itself on this slice and every future chapter plan.

### KCK: auto-publish contract status as a flag

`ContractEventSource.Fire` currently writes the contract type to `record.Params["contract"]` but doesn't update KDK flags. Change: on each Contract event, also call `DialogueKit.Flags.Set("contract:" + name + "." + state, "true")` where state is one of `accepted | complete | failed | cancelled`.

Per-state boolean flags rather than a single status string — `RequirementExpression` (the engine that already powers `visibleIf` and `whenExpression`) handles boolean flag presence cleanly without needing string-literal parsing.

After the change, any trigger's `whenExpression` can query contract state directly:

```cfg
whenExpression = contract:BKEX_RoverMonolith.complete && contract:BKEX_FlyToIsland.complete
```

No helper triggers, no manual `SET_FLAG` boilerplate. Estimated effort: ~1 hour including unit test for the flag-write path.

### KAK: composite memo conditions (`ANY` / `ALL`)

Currently `Memo.Conditions` is a flat list ANDed implicitly. Adds `CompositeCondition` with `Mode = All | Any` and corresponding cfg blocks:

```cfg
DIRECTOR_MEMO
{
    id = example
    ANY
    {
        CONDITION { type = FundsBelow; threshold = 25000 }
        CONDITION { type = ChapterAtLeast; chapter = 3 }
    }
    ALL
    {
        CONDITION { type = ReputationAbove; threshold = 50 }
        CONDITION { type = FlagEquals; flag = ch1_choice; value = space }
    }
}
```

`ANY` and `ALL` can nest. Top-level `CONDITION` blocks keep their implicit-AND behavior — no breaking change. Estimated effort: ~2 hours including unit tests covering nested compositions and edge cases (empty composites, single-child composites).

## File layout

All new content under `BadgKatCareer/DirectorContent/`:

```
BadgKatCareer/
├── DirectorContent/
│   ├── AdminKitSettings.cfg        # KERBAL_ADMIN_KIT block
│   ├── Characters.cfg              # 6 DIRECTOR_CHARACTER
│   ├── Focuses.cfg                 # 6 DIRECTOR_FOCUS
│   ├── BuildingScenes.cfg          # 4 DIRECTOR_BUILDING_SCENE
│   ├── DispositionDecay.cfg        # 6 DISPOSITION_DECAY
│   ├── Memos.cfg                   # 4-5 DIRECTOR_MEMO
│   ├── Reputation.cfg              # REPUTATION_INCOME, REPUTATION_DECAY, DIRECTOR_PR_CAMPAIGN
│   ├── Chapters.cfg                # 3 CAMPAIGN_CHAPTER
│   ├── Triggers.cfg                # CAMPAIGN_TRIGGER (3 chapter triggers + any helpers)
│   └── Scenes/
│       ├── ch1_directive_gene_wernher.cfg
│       ├── staff_first_recovery.cfg
│       └── building_overlays.cfg   # 4 stub scenes for the building chips
└── ContractTweaks.cfg              # APPENDED with @CONTRACT_TYPE patches
```

Per-concern files (not per-chapter) — when Ch2+ content lands, files grow with `// Chapter N ──────` section comments rather than fragmenting. Scenes get one file per scene (multi-line LINE blocks grow fast).

## Chapter machine

### Triggers

```cfg
CAMPAIGN_TRIGGER
{
    id = ch0_entry
    once = true
    ON_EVENT { type = FacilityEntered; facility = SpaceCenter }
    ACTIONS { ADVANCE_CHAPTER { target = 0 } }
}

CAMPAIGN_TRIGGER
{
    id = ch1_entry
    once = true
    ON_EVENT { type = ContractCompleted; contractType = BKEX_FirstScience }
    ACTIONS
    {
        ADVANCE_CHAPTER { target = 1 }
        ENQUEUE_SCENE { sceneId = staff_first_recovery }
        NOTIFY { target = admin; severity = Info; source = ch1_welcome }
    }
}

CAMPAIGN_TRIGGER
{
    id = ch2_entry
    once = true
    ON_EVENT { type = ContractCompleted; contractType = BKEX_RoverMonolith }
    ON_EVENT { type = ContractCompleted; contractType = BKEX_FlyToIsland }
    whenExpression = contract:BKEX_RoverMonolith.complete && contract:BKEX_FlyToIsland.complete
    ACTIONS { ENQUEUE_SCENE { sceneId = ch1_directive_gene_wernher } }
}
```

The Ch1→Ch2 trigger relies on the new KCK auto-flag feature: each `ContractCompleted` event also writes `contract:<name> = complete` to KDK flags, and `whenExpression` evaluates against those. Either completion order works — the trigger re-evaluates on either event, fires only when both flags are set.

### Directive scene

`Scenes/ch1_directive_gene_wernher.cfg`:

- `OnFacilityEnter = Administration` — plays next time the player enters admin after the trigger fires (not immediately on contract completion). Avoids interrupting recovery flow.
- Opens with Linus's existing BothLeads dialogue (3 lines), lifted verbatim from `BridgeContracts.cfg` before that file is deleted.
- Forks via `CHOICE_CARD` into:
  - `space` → sets `ch1_choice = space`, calls `advance_chapter_2` action
  - `ground` → sets `ch1_choice = ground`, calls `advance_chapter_2` action
- Both branches call the same `ADVANCE_CHAPTER { target = 2 }`. Mechanically the chapter advances regardless; only the flag value differs.

## Contract gating

Two layers added via `@CONTRACT_TYPE[...]` patches in `ContractTweaks.cfg`:

### Layer 1: Chapter gates

| Contract | Add REQUIREMENT |
|---|---|
| `BKEX_RoverMonolith` | `ChapterAtLeast = 1` |
| `BKEX_FlyToIsland` | `ChapterAtLeast = 1` |
| `BKEX_UnmannedSuborbital` | `ChapterAtLeast = 2` |

`BKEX_FirstScience` gets no chapter gate — it's the Ch0 contract.

### Layer 2: Directive flag gates

| Contract | Add REQUIREMENT |
|---|---|
| `BKEX_UnmannedSuborbital` | `FlagEquals { flag = ch1_choice; value = space }` (in addition to chapter gate) |
| `BKML_SoundBarrier` | `FlagEquals { flag = ch1_choice; value = ground }` + `ChapterAtLeast = 2` |

Both REQUIREMENTs compose as AND by default in CC. This is what gives the directive choice mechanical teeth: `space` players unlock orbital first; `ground` players unlock aviation tier-2 first. Both eventually unlock both — sequencing differs.

## Validation plan

### Build-time checks

After first launch with the slice loaded:

- Zero `[ERROR]` or `[WRN]` lines from `ContractConfigurator`, `ModuleManager`, `KerbalCampaignKit`, `KerbalAdminKit`, `KerbalDialogueKit` in `KSP.log`
- No `expression = false` warnings (deprecated requirements we missed)
- No unresolvable contract type references
- No malformed `whenExpression` parses
- No missing scene IDs from `ENQUEUE_SCENE` actions
- `ModuleManager.ConfigCache` shows the expected new patches

### Manual playtest sequence

| Step | Action | Verifies |
|---|---|---|
| 1a | New career save, enter SpaceCenter view | `ch0_entry` fires; chapter == 0; KSC view shows 4 exterior building chips (Mortimer + Walt over Admin, Gene over MC, Linus over Tracking) |
| 1b | Click Administration building | KAK admin UI loads with 6 character chips in the character panel (all DIRECTOR_CHARACTER) |
| 2 | Click each character chip in admin UI | Picker shows for each (1 focus each); selecting clears the alert badge |
| 3 | Click Mortimer's exterior building chip on Admin | `mortimer_admin_budget` stub scene fires with Mortimer as speaker. Repeat for Walt / Gene / Linus chips on their respective buildings. |
| 4 | Build vessel, get any science, recover | `BKEX_FirstScience` completes → `ch1_entry` fires → chapter == 1 → `staff_first_recovery` plays → `gus_day_one_kit` memo activates |
| 5 | Build cheap rover, drive to Kerbin Monolith | `BKEX_RoverMonolith` completes; `contract:BKEX_RoverMonolith.complete` flag set; `ch2_entry` does NOT fire (FlyToIsland still pending) |
| 6 | Build biplane, fly to island airfield | `BKEX_FlyToIsland` completes; `whenExpression` now passes; directive scene queues |
| 7 | Walk into admin building | `ch1_directive_gene_wernher` plays (Linus opening + Gene-vs-Wernher fork) |
| 8 | Pick "push to space" | `ch1_choice = space` set; chapter == 2; `BKEX_UnmannedSuborbital` appears in MC; `BKML_SoundBarrier` does NOT |
| 9 | Reload before step 8, pick "ground" | `BKEX_UnmannedSuborbital` does NOT appear; `BKML_SoundBarrier` does |
| 10 | Dismiss `gus_day_one_kit` memo | Memo disappears; doesn't re-fire on next 5s poll |
| 11 | Quicksave, alt-F4, reload | Chapter, flags, dismissed memos, rearm-pending all persist correctly |

**Critical:** steps 5 and 6 must be runnable in either order. The `whenExpression` is the load-bearing piece of the new framework features.

### Reverse validation

- Disable BadgKatCareer, run a clean stock career save: KAK / KCK / KDK all inert (no admin replacement, no markers, no chapter UI). Stock KSP unaffected.
- Re-enable BadgKatCareer mid-save (existing save without DirectorContent): chapter starts at 0; previously-completed contracts don't retro-fire triggers; no crashes.

## Post-slice next steps

After this slice ships and shakes out wiring bugs:

1. **Voice rewrite pass** — replace placeholder text in memos and stub scenes with on-character voice work. Per-character voices are documented in `2026-04-15-chatterbox-dialogue-design.md`.
2. **Ch2 vertical slice** — Into Orbit chapter content + the Mortimer-vs-Wernher and Gene-vs-Walt directives at end of Ch2. Lifts the MunSignals bridge dialogue into a KCK trigger + KDK scene during this plan.
3. **Ch3-5 plans** — Beyond the Mun, Interplanetary, Kcalbeloh. Each plan's chapter-transition trigger can lift the corresponding bridge's dialogue from git history.
4. **Focus expansion** — bump from 1 focus per character (slice) to 3 per character (full per redesign spec) once we have a sense of which focus axes carry meaningful gameplay weight.
5. **Portrait artwork** — drop PNGs into `BadgKatCareer/Art/`, add `CHIP_LAYER` blocks to each character.
