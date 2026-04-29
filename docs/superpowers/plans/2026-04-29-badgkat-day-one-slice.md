# BadgKat Day One Vertical Slice — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire BadgKatCareer's first end-to-end content slice (Prologue → Day One → Into Orbit stub) onto the KDK / KCK / KAK framework, including a player-choice directive with mechanical contract-gating teeth.

**Architecture:** Three repos touched. Pre-slice cleanup deletes obsoleted bridge contracts and deprecation tombstones from BadgKatCareer. Two small framework additions to KCK (auto-publish contract status flags) and KAK (composite ANY/ALL memo conditions, ChapterAtLeast condition) close API gaps motivated by this slice. Then BadgKatCareer ships pure-cfg DirectorContent driving the kits, plus chapter/flag REQUIREMENT patches on existing contracts.

**Tech Stack:** C# (.NET Framework 4.7.2 / net472, KSP 1.12.x). xUnit for plugin unit tests. KSP ConfigNode cfg + ContractConfigurator + ModuleManager for content.

**Repos and branches:**
- `GameData/BadgKatCareer/` (branch: `master`) — cleanup + content cfg
- `GameData/KerbalCampaignKit/` (branch: `main`) — auto-flag feature
- `GameData/KerbalAdminKit/` (branch: `main`) — composite conditions + ChapterAtLeast condition

**Three separate git repos.** Always `cd` into the right one before running git commands. Cross-repo work = three sets of commits.

---

## Phase A — Pre-slice cleanup (BadgKatCareer)

Delete obsoleted bridge contracts and `expression = false` deprecation tombstones. Repoint references that survive into the cleaned graph. Pre-delete dialogue lift if needed.

### Task 1: Audit dialogue lift requirements

**Files:**
- Read: `ContractPacks/Exploration/CampaignOpening.cfg` (defines BKEX_RoverMonolith, BKEX_FlyToIsland)
- Read: `ContractPacks/Milestones/FirstFlight.cfg`, `Milestones/FirstRover.cfg`, `Milestones/IslandHop.cfg`, `AnomalySurveyor/Kerbin_IslandAirfield.cfg`, `AnomalySurveyor/Kerbin_Monolith.cfg`

- [ ] **Step 1: Read campaign-version contracts**

```bash
cd "GameData/BadgKatCareer"
grep -A 200 "name = BKEX_RoverMonolith" ContractPacks/Exploration/CampaignOpening.cfg | head -250
grep -A 200 "name = BKEX_FlyToIsland"   ContractPacks/Exploration/CampaignOpening.cfg | head -250
```

- [ ] **Step 2: For each deprecated file, list LINE blocks not present in the campaign target**

Build a 5-row table in your head (one row per deprecated file). For each, decide: (a) campaign target already has equivalent dialogue → mark "no lift needed"; (b) campaign target lacks equivalent → mark "lift" with the LINE numbers to copy.

| Deprecated source | Campaign target | Lift needed? |
|---|---|---|
| `Milestones/FirstFlight.cfg` | `BKEX_FlyToIsland` | ? |
| `Milestones/FirstRover.cfg` | `BKEX_RoverMonolith` | ? |
| `Milestones/IslandHop.cfg` | `BKEX_FlyToIsland` | ? |
| `AnomalySurveyor/Kerbin_IslandAirfield.cfg` | `BKEX_FlyToIsland` | ? |
| `AnomalySurveyor/Kerbin_Monolith.cfg` | `BKEX_RoverMonolith` | ? |

- [ ] **Step 3: No commit. This task is pure analysis; subsequent tasks act on its output.**

### Task 2: Lift dialogue (per-file, conditional on Task 1 audit)

**Files:**
- Modify: `ContractPacks/Exploration/CampaignOpening.cfg` (only if lifts are needed)

- [ ] **Step 1: For each "lift" row in Task 1's table, append the LINEs to the campaign contract's existing `BEHAVIOUR { type = ChatterBox; onState = CONTRACT_ACCEPTED }` block**

If `BKEX_RoverMonolith` or `BKEX_FlyToIsland` lacks a ChatterBox BEHAVIOUR entirely, copy the whole BEHAVIOUR block from the deprecated file (including character A/B blocks). Indentation must match CampaignOpening.cfg's existing indent style (tabs, look at neighboring blocks).

- [ ] **Step 2: Verify with grep that lifted text appears in the new home**

```bash
grep -F "Wright Brothers" ContractPacks/Exploration/CampaignOpening.cfg   # FirstFlight Wernher line
grep -F "left wheel" ContractPacks/Exploration/CampaignOpening.cfg        # FirstRover Gus line
# (etc. for each lifted phrase)
```

Expected: every lifted phrase returns a match.

- [ ] **Step 3: Commit (only if any lifts were performed; otherwise skip)**

```bash
cd "GameData/BadgKatCareer"
git add ContractPacks/Exploration/CampaignOpening.cfg
git commit -m "content: lift dialogue from deprecated milestones into campaign contracts"
```

### Task 3: Delete bridge contracts file

**Files:**
- Delete: `ContractPacks/Exploration/BridgeContracts.cfg` (4 contracts: BKEX_Bridge_BothLeads, BKEX_Bridge_MunSignals, BKEX_Bridge_TraceSignal, BKEX_Bridge_GoingDeeper)

- [ ] **Step 1: Confirm the 4 bridge contract names appear nowhere else as definitions**

```bash
cd "GameData/BadgKatCareer"
grep -rn "name = BKEX_Bridge_" ContractPacks/
```

Expected: only matches inside `BridgeContracts.cfg` itself.

- [ ] **Step 2: List all callers (these will become broken refs once we delete; we'll handle in Task 7)**

```bash
grep -rn "contractType = BKEX_Bridge_" ContractPacks/
```

Expected references (already known): `Exploration/FirstSteps.cfg`, `Exploration/ProbeExploration.cfg`, `AnomalySurveyor/Duna_Face.cfg`, `AnomalySurveyor/Kerbin_Pyramids.cfg`, `AnomalySurveyor/Mun_Monolith.cfg`, `AnomalySurveyor/Vall_Icehenge.cfg`. Note these for Task 7.

- [ ] **Step 3: Delete the file**

```bash
git rm ContractPacks/Exploration/BridgeContracts.cfg
```

- [ ] **Step 4: Commit**

```bash
git commit -m "cleanup: remove obsoleted bridge contracts (replaced by KCK trigger + KDK ENQUEUE_SCENE)"
```

### Task 4: Delete deprecation tombstones

**Files:**
- Delete: `ContractPacks/Milestones/FirstFlight.cfg`
- Delete: `ContractPacks/Milestones/FirstRover.cfg`
- Delete: `ContractPacks/Milestones/IslandHop.cfg`
- Delete: `ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg`
- Delete: `ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg`

- [ ] **Step 1: Re-confirm each file has the `expression = false` tombstone**

```bash
cd "GameData/BadgKatCareer"
grep -l "expression = false" \
  ContractPacks/Milestones/FirstFlight.cfg \
  ContractPacks/Milestones/FirstRover.cfg \
  ContractPacks/Milestones/IslandHop.cfg \
  ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg \
  ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg
```

Expected: all 5 paths returned.

- [ ] **Step 2: Delete all 5 files**

```bash
git rm \
  ContractPacks/Milestones/FirstFlight.cfg \
  ContractPacks/Milestones/FirstRover.cfg \
  ContractPacks/Milestones/IslandHop.cfg \
  ContractPacks/AnomalySurveyor/Kerbin_IslandAirfield.cfg \
  ContractPacks/AnomalySurveyor/Kerbin_Monolith.cfg
```

- [ ] **Step 3: Commit**

```bash
git commit -m "cleanup: remove expression=false deprecation tombstones (BKML_FirstFlight/FirstRover/IslandHop, AS Kerbin Monolith/IslandAirfield)"
```

### Task 5: Repoint BKML_IslandHop references

**Files:**
- Modify: `ContractPacks/Milestones/KS_Batch1.cfg` (4 occurrences)
- Modify: `ContractPacks/Milestones/DessertRun.cfg` (1 occurrence)

- [ ] **Step 1: Replace all `BKML_IslandHop` references with `BKEX_FlyToIsland`**

```bash
cd "GameData/BadgKatCareer"
sed -i 's/BKML_IslandHop/BKEX_FlyToIsland/g' ContractPacks/Milestones/KS_Batch1.cfg ContractPacks/Milestones/DessertRun.cfg
```

(Bash sed; on Windows use `sed -i ''` if needed, or open in editor and replace.)

- [ ] **Step 2: Verify no `BKML_IslandHop` references remain anywhere**

```bash
grep -rn "BKML_IslandHop" ContractPacks/
```

Expected: zero matches.

- [ ] **Step 3: Verify substitution count**

```bash
grep -c "BKEX_FlyToIsland" ContractPacks/Milestones/KS_Batch1.cfg ContractPacks/Milestones/DessertRun.cfg
```

Expected: KS_Batch1.cfg → 4, DessertRun.cfg → 1.

- [ ] **Step 4: Commit**

```bash
git add ContractPacks/Milestones/KS_Batch1.cfg ContractPacks/Milestones/DessertRun.cfg
git commit -m "cleanup: repoint BKML_IslandHop refs to BKEX_FlyToIsland (BKML_IslandHop deleted)"
```

### Task 6: Audit other tombstone references (BKML_FirstFlight, BKML_FirstRover, AS_Kerbin_*)

**Files:**
- Investigate: any cfg referencing `BKML_FirstFlight`, `BKML_FirstRover`, `AS_Kerbin_Monolith`, `AS_Kerbin_IslandAirfield`

- [ ] **Step 1: Grep for each name**

```bash
cd "GameData/BadgKatCareer"
grep -rn "BKML_FirstFlight\|BKML_FirstRover\|AS_Kerbin_Monolith\b\|AS_Kerbin_IslandAirfield" ContractPacks/
```

Note: `AS_Kerbin_Monolith` uses `\b` to avoid matching `AS_Kerbin_Monolith_Additional` (which is a different valid contract).

Expected results based on prior research: matches inside the deleted bridge contracts (already gone) and possibly inside other deleted tombstones (also gone). Any remaining match is a real broken reference.

- [ ] **Step 2: For each remaining match, decide repoint target**

Likely all matches will be from the just-deleted files (and thus already gone with their parent file), but if any survive in a file we haven't touched: `BKML_FirstFlight` → `BKEX_FlyToIsland`; `BKML_FirstRover` → `BKEX_RoverMonolith`; `AS_Kerbin_Monolith` → `BKEX_RoverMonolith`; `AS_Kerbin_IslandAirfield` → `BKEX_FlyToIsland`. Apply with `sed` or editor.

- [ ] **Step 3: Re-grep until clean**

```bash
grep -rn "BKML_FirstFlight\|BKML_FirstRover\|AS_Kerbin_Monolith\b\|AS_Kerbin_IslandAirfield" ContractPacks/
```

Expected: zero matches.

- [ ] **Step 4: Commit if any changes were made (otherwise skip)**

```bash
git add ContractPacks/
git commit -m "cleanup: repoint stragglers from deleted tombstone contracts"
```

### Task 7: Cleanup audit

**Files:** none modified — verification only.

- [ ] **Step 1: Final grep across all the deleted names**

```bash
cd "GameData/BadgKatCareer"
grep -rn "BKML_FirstFlight\|BKML_FirstRover\|BKML_IslandHop\|BKEX_Bridge_BothLeads\|BKEX_Bridge_MunSignals\|BKEX_Bridge_TraceSignal\|BKEX_Bridge_GoingDeeper\|AS_Kerbin_Monolith\b\|AS_Kerbin_IslandAirfield" ContractPacks/
```

Expected: zero matches.

- [ ] **Step 2: Confirm git working tree is clean**

```bash
git status
```

Expected: "nothing to commit, working tree clean".

- [ ] **Step 3: No commit. Audit only.**

---

## Phase B — Framework additions

Two repos: KerbalCampaignKit and KerbalAdminKit. Build new features TDD-style (xUnit tests first, then implementation), then build and copy DLLs to Plugins/.

### Task 8: KCK — failing test for contract-status flag publication

**Files:**
- Modify: `tests/KerbalCampaignKit.Tests/TriggerEngineTests.cs` (or create a new test file `ContractEventSourceTests.cs` — see step 1)
- Test: `tests/KerbalCampaignKit.Tests/ContractEventSourceTests.cs`

- [ ] **Step 1: Check whether `ContractEventSource` is testable without KSP**

```bash
cd "GameData/KerbalCampaignKit"
grep -rn "GameEvents.Contract" src/
```

`ContractEventSource.cs` calls `GameEvents.Contract.onCompleted.Add(...)`, which depends on KSP. The publishable-flag side-effect (writing to `DialogueKit.Flags`) is the new behavior — that's testable in isolation if we extract the flag-write into a static helper.

- [ ] **Step 2: Decide test boundary**

Plan: extract a `static void PublishContractStateFlag(string contractName, string state)` method on `ContractEventSource` (or a separate `ContractStateFlagPublisher` class). The Fire method calls this helper. Tests call the helper directly with a fake/test FlagStore.

- [ ] **Step 3: Create test file with failing tests**

Create `tests/KerbalCampaignKit.Tests/ContractStateFlagPublisherTests.cs`:

```csharp
using Xunit;
using KerbalCampaignKit.Triggers.Events;
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Tests
{
    public class ContractStateFlagPublisherTests
    {
        [Fact]
        public void Publish_SetsCompleteFlag_WhenStateIsComplete()
        {
            var flags = new FlagStore();
            ContractStateFlagPublisher.Publish(flags, "BKEX_RoverMonolith", "complete");
            Assert.Equal("true", flags.Get("contract:BKEX_RoverMonolith.complete"));
        }

        [Fact]
        public void Publish_SetsAcceptedFlag_WhenStateIsAccepted()
        {
            var flags = new FlagStore();
            ContractStateFlagPublisher.Publish(flags, "BKEX_FlyToIsland", "accepted");
            Assert.Equal("true", flags.Get("contract:BKEX_FlyToIsland.accepted"));
        }

        [Fact]
        public void Publish_DoesNothing_WhenFlagsNull()
        {
            // Should not throw.
            ContractStateFlagPublisher.Publish(null, "BKEX_X", "complete");
        }

        [Fact]
        public void Publish_DoesNothing_WhenContractNameNullOrEmpty()
        {
            var flags = new FlagStore();
            ContractStateFlagPublisher.Publish(flags, null, "complete");
            ContractStateFlagPublisher.Publish(flags, "", "complete");
            Assert.Empty(flags.AllKeys());
        }

        [Fact]
        public void Publish_DoesNothing_WhenStateNullOrEmpty()
        {
            var flags = new FlagStore();
            ContractStateFlagPublisher.Publish(flags, "BKEX_X", null);
            ContractStateFlagPublisher.Publish(flags, "BKEX_X", "");
            Assert.Empty(flags.AllKeys());
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they fail (compile error: no `ContractStateFlagPublisher`)**

```bash
cd "GameData/KerbalCampaignKit"
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter "FullyQualifiedName~ContractStateFlagPublisher"
```

Expected: build fails with `CS0103: The name 'ContractStateFlagPublisher' does not exist` or equivalent.

- [ ] **Step 5: Verify FlagStore has an `AllKeys()` method**

```bash
grep -n "AllKeys\|Keys" "../KerbalDialogueKit/src/KerbalDialogueKit/Flags/FlagStore.cs"
```

If FlagStore doesn't expose `AllKeys()`, replace those test assertions with explicit `Assert.Null(flags.Get("contract:X.complete"))` or `Assert.False(flags.HasFlag("..."))` checks instead. Adjust the test file if needed.

### Task 9: KCK — implement ContractStateFlagPublisher

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Events/ContractStateFlagPublisher.cs`

- [ ] **Step 1: Create the publisher class**

```csharp
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Publishes contract status as KDK flags so trigger whenExpression
    /// and CC FlagEquals/FlagExpression requirements can query contract state directly.
    /// Flag format: contract:&lt;name&gt;.&lt;state&gt; = "true"
    /// where state is one of accepted | complete | failed | cancelled.
    /// </summary>
    public static class ContractStateFlagPublisher
    {
        public static void Publish(FlagStore flags, string contractName, string state)
        {
            if (flags == null) return;
            if (string.IsNullOrEmpty(contractName)) return;
            if (string.IsNullOrEmpty(state)) return;
            flags.Set($"contract:{contractName}.{state}", "true");
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they pass**

```bash
cd "GameData/KerbalCampaignKit"
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter "FullyQualifiedName~ContractStateFlagPublisher"
```

Expected: all 5 tests pass.

- [ ] **Step 3: Commit**

```bash
git add tests/KerbalCampaignKit.Tests/ContractStateFlagPublisherTests.cs src/KerbalCampaignKit/Triggers/Events/ContractStateFlagPublisher.cs
git commit -m "feat(kck): add ContractStateFlagPublisher for whenExpression queries"
```

### Task 10: KCK — wire ContractEventSource to publish on each event

**Files:**
- Modify: `src/KerbalCampaignKit/Triggers/Events/ContractEventSource.cs`

- [ ] **Step 1: Update `Fire` method to call the publisher**

Replace the `Fire` method body in `ContractEventSource.cs`:

```csharp
private void Fire(EventType type, Contract contract)
{
    if (sink == null || contract == null) return;
    var name = ContractTypeName(contract);
    var record = new EventRecord { Type = type };
    record.Params["contract"] = name;
    sink(record);

    var state = StateForType(type);
    ContractStateFlagPublisher.Publish(KerbalDialogueKit.Core.DialogueKit.Flags, name, state);
}

private static string StateForType(EventType type)
{
    switch (type)
    {
        case EventType.ContractAccepted:  return "accepted";
        case EventType.ContractComplete:  return "complete";
        case EventType.ContractFailed:    return "failed";
        case EventType.ContractCancelled: return "cancelled";
        default: return null;
    }
}
```

- [ ] **Step 2: Verify EventType enum values**

```bash
grep "ContractAccepted\|ContractComplete\|ContractFailed\|ContractCancelled" src/KerbalCampaignKit/Triggers/Events/EventType.cs
```

Expected: all four names exist as enum members. If any name differs (e.g., `ContractCompleted` instead of `ContractComplete`), adjust the switch in `StateForType` accordingly.

- [ ] **Step 3: Build to confirm compile**

```bash
cd "GameData/KerbalCampaignKit"
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: build succeeds.

- [ ] **Step 4: Run all KCK tests (regression check)**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
```

Expected: all tests pass (no regressions).

- [ ] **Step 5: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Events/ContractEventSource.cs
git commit -m "feat(kck): ContractEventSource publishes contract:<name>.<state> flags on each Contract event"
```

### Task 11: KAK — failing tests for ChapterAtLeast memo condition

**Files:**
- Modify: `tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs`

- [ ] **Step 1: Add tests after the existing `InChapter_True_WhenCurrentMatches` test**

Append these `[Fact]` methods inside the existing `PureMemoConditionTests` class:

```csharp
[Fact]
public void ChapterAtLeast_True_WhenCurrentMatches()
{
    var c = new ChapterAtLeastCondition { Chapter = 1 };
    Assert.True(c.Evaluate(Ctx(chapter: "1")));
}

[Fact]
public void ChapterAtLeast_True_WhenCurrentExceeds()
{
    var c = new ChapterAtLeastCondition { Chapter = 1 };
    Assert.True(c.Evaluate(Ctx(chapter: "2")));
    Assert.True(c.Evaluate(Ctx(chapter: "5")));
}

[Fact]
public void ChapterAtLeast_False_WhenCurrentBelow()
{
    var c = new ChapterAtLeastCondition { Chapter = 2 };
    Assert.False(c.Evaluate(Ctx(chapter: "1")));
    Assert.False(c.Evaluate(Ctx(chapter: "0")));
}

[Fact]
public void ChapterAtLeast_False_WhenCurrentNullOrUnparseable()
{
    var c = new ChapterAtLeastCondition { Chapter = 1 };
    Assert.False(c.Evaluate(Ctx(chapter: null)));
    Assert.False(c.Evaluate(Ctx(chapter: "not-a-number")));
}
```

- [ ] **Step 2: Run tests to verify they fail (compile error)**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj --filter "FullyQualifiedName~ChapterAtLeast"
```

Expected: build fails with `CS0103: The name 'ChapterAtLeastCondition' does not exist`.

### Task 12: KAK — implement ChapterAtLeastCondition

**Files:**
- Create: `src/KerbalAdminKit/Memos/Conditions/ChapterAtLeastCondition.cs`
- Modify: `src/KerbalAdminKit/Memos/Conditions/MemoConditionFactory.cs`

- [ ] **Step 1: Create the condition class**

```csharp
using System.Globalization;

namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// True when the current chapter id is parseable as an integer
    /// and >= the configured threshold.
    /// </summary>
    public sealed class ChapterAtLeastCondition : IMemoCondition
    {
        public int Chapter;

        public bool Evaluate(MemoContext ctx)
        {
            if (string.IsNullOrEmpty(ctx.CurrentChapter)) return false;
            if (!int.TryParse(ctx.CurrentChapter, NumberStyles.Integer,
                CultureInfo.InvariantCulture, out var current)) return false;
            return current >= Chapter;
        }
    }
}
```

- [ ] **Step 2: Register it in MemoConditionFactory**

In `MemoConditionFactory.Build`, add a new case before the `default`:

```csharp
case "ChapterAtLeast":
    return new ChapterAtLeastCondition { Chapter = (int)D(node, "chapter") };
```

- [ ] **Step 3: Run the new tests**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj --filter "FullyQualifiedName~ChapterAtLeast"
```

Expected: all 4 tests pass.

- [ ] **Step 4: Commit**

```bash
git add tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs \
        src/KerbalAdminKit/Memos/Conditions/ChapterAtLeastCondition.cs \
        src/KerbalAdminKit/Memos/Conditions/MemoConditionFactory.cs
git commit -m "feat(kak): add ChapterAtLeast memo condition"
```

### Task 13: KAK — failing tests for ANY/ALL composite conditions

**Files:**
- Modify: `tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs`

- [ ] **Step 1: Append composite-condition tests**

Add to the existing `PureMemoConditionTests` class:

```csharp
[Fact]
public void All_True_WhenAllChildrenTrue()
{
    var c = new CompositeCondition
    {
        Mode = CompositeMode.All,
        Children =
        {
            new FundsBelowCondition { Threshold = 10000 },
            new ReputationAboveCondition { Threshold = 50 },
        },
    };
    Assert.True(c.Evaluate(Ctx(funds: 5000, rep: 100)));
}

[Fact]
public void All_False_WhenAnyChildFalse()
{
    var c = new CompositeCondition
    {
        Mode = CompositeMode.All,
        Children =
        {
            new FundsBelowCondition { Threshold = 10000 },
            new ReputationAboveCondition { Threshold = 50 },
        },
    };
    Assert.False(c.Evaluate(Ctx(funds: 5000, rep: 10)));
    Assert.False(c.Evaluate(Ctx(funds: 50000, rep: 100)));
}

[Fact]
public void All_True_WhenChildrenEmpty()
{
    var c = new CompositeCondition { Mode = CompositeMode.All };
    Assert.True(c.Evaluate(Ctx()));
}

[Fact]
public void Any_True_WhenAnyChildTrue()
{
    var c = new CompositeCondition
    {
        Mode = CompositeMode.Any,
        Children =
        {
            new FundsBelowCondition { Threshold = 10000 },
            new ReputationAboveCondition { Threshold = 50 },
        },
    };
    Assert.True(c.Evaluate(Ctx(funds: 5000, rep: 0)));
    Assert.True(c.Evaluate(Ctx(funds: 50000, rep: 100)));
}

[Fact]
public void Any_False_WhenAllChildrenFalse()
{
    var c = new CompositeCondition
    {
        Mode = CompositeMode.Any,
        Children =
        {
            new FundsBelowCondition { Threshold = 10000 },
            new ReputationAboveCondition { Threshold = 50 },
        },
    };
    Assert.False(c.Evaluate(Ctx(funds: 50000, rep: 10)));
}

[Fact]
public void Any_False_WhenChildrenEmpty()
{
    var c = new CompositeCondition { Mode = CompositeMode.Any };
    Assert.False(c.Evaluate(Ctx()));
}

[Fact]
public void Composite_Nests()
{
    // ALL { ANY { funds<10k, rep>50 }, chapter>=1 }
    var c = new CompositeCondition
    {
        Mode = CompositeMode.All,
        Children =
        {
            new CompositeCondition
            {
                Mode = CompositeMode.Any,
                Children =
                {
                    new FundsBelowCondition { Threshold = 10000 },
                    new ReputationAboveCondition { Threshold = 50 },
                },
            },
            new ChapterAtLeastCondition { Chapter = 1 },
        },
    };
    Assert.True(c.Evaluate(Ctx(funds: 5000, rep: 0, chapter: "1")));
    Assert.False(c.Evaluate(Ctx(funds: 5000, rep: 0, chapter: "0")));
    Assert.False(c.Evaluate(Ctx(funds: 50000, rep: 10, chapter: "1")));
}
```

- [ ] **Step 2: Run tests to verify they fail (compile error)**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj --filter "FullyQualifiedName~Composite|FullyQualifiedName~All_|FullyQualifiedName~Any_"
```

Expected: build fails with `CS0103: The name 'CompositeCondition' does not exist`.

### Task 14: KAK — implement CompositeCondition

**Files:**
- Create: `src/KerbalAdminKit/Memos/Conditions/CompositeCondition.cs`

- [ ] **Step 1: Create the class with enum**

```csharp
using System.Collections.Generic;

namespace KerbalAdminKit.Memos.Conditions
{
    public enum CompositeMode { All, Any }

    /// <summary>
    /// Combines child conditions with All (AND) or Any (OR) semantics.
    /// Empty All evaluates true (vacuous truth); empty Any evaluates false.
    /// Nestable.
    /// </summary>
    public sealed class CompositeCondition : IMemoCondition
    {
        public CompositeMode Mode = CompositeMode.All;
        public List<IMemoCondition> Children = new List<IMemoCondition>();

        public bool Evaluate(MemoContext ctx)
        {
            if (Mode == CompositeMode.All)
            {
                foreach (var c in Children)
                    if (!c.Evaluate(ctx)) return false;
                return true;
            }
            else
            {
                foreach (var c in Children)
                    if (c.Evaluate(ctx)) return true;
                return false;
            }
        }
    }
}
```

- [ ] **Step 2: Run the new tests**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj --filter "FullyQualifiedName~Composite|FullyQualifiedName~All_|FullyQualifiedName~Any_"
```

Expected: all 7 tests pass.

- [ ] **Step 3: Commit**

```bash
git add tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs \
        src/KerbalAdminKit/Memos/Conditions/CompositeCondition.cs
git commit -m "feat(kak): add CompositeCondition for ANY/ALL memo condition combinators"
```

### Task 15: KAK — failing test for MemoLoader composite parsing

**Files:**
- Modify: `tests/KerbalAdminKit.Tests/MemoLoaderTests.cs`

- [ ] **Step 1: Append two tests covering ALL and ANY block parsing**

```csharp
[Fact]
public void Load_ParsesAllBlock()
{
    var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
    var all = new FakeSceneNode();
    all.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "FundsBelow").Set("threshold", "10000"));
    all.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "ChapterAtLeast").Set("chapter", "1"));
    n.AddChild("ALL", all);

    var m = MemoLoader.Load(n);
    Assert.Single(m.Conditions);
    var composite = Assert.IsType<CompositeCondition>(m.Conditions[0]);
    Assert.Equal(CompositeMode.All, composite.Mode);
    Assert.Equal(2, composite.Children.Count);
}

[Fact]
public void Load_ParsesAnyBlock()
{
    var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
    var any = new FakeSceneNode();
    any.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "FundsBelow").Set("threshold", "10000"));
    any.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "ReputationAbove").Set("threshold", "50"));
    n.AddChild("ANY", any);

    var m = MemoLoader.Load(n);
    Assert.Single(m.Conditions);
    var composite = Assert.IsType<CompositeCondition>(m.Conditions[0]);
    Assert.Equal(CompositeMode.Any, composite.Mode);
    Assert.Equal(2, composite.Children.Count);
}

[Fact]
public void Load_NestsCompositesRecursively()
{
    var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
    var outer = new FakeSceneNode();
    var inner = new FakeSceneNode();
    inner.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "FundsBelow").Set("threshold", "10000"));
    outer.AddChild("ANY", inner);
    outer.AddChild("CONDITION", new FakeSceneNode()
        .Set("type", "ChapterAtLeast").Set("chapter", "1"));
    n.AddChild("ALL", outer);

    var m = MemoLoader.Load(n);
    Assert.Single(m.Conditions);
    var topAll = Assert.IsType<CompositeCondition>(m.Conditions[0]);
    Assert.Equal(CompositeMode.All, topAll.Mode);
    Assert.Equal(2, topAll.Children.Count);

    var nestedAny = Assert.IsType<CompositeCondition>(topAll.Children[0]);
    Assert.Equal(CompositeMode.Any, nestedAny.Mode);
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj --filter "FullyQualifiedName~Load_ParsesAllBlock|FullyQualifiedName~Load_ParsesAnyBlock|FullyQualifiedName~Load_NestsComposites"
```

Expected: tests fail (assertions fail because MemoLoader currently ignores ALL/ANY child nodes).

### Task 16: KAK — extend MemoLoader to parse composite blocks

**Files:**
- Modify: `src/KerbalAdminKit/Memos/MemoLoader.cs`

- [ ] **Step 1: Replace the condition-loading loop with a recursive helper**

In `MemoLoader.Load`, replace this block:

```csharp
foreach (var condNode in node.GetNodes("CONDITION"))
{
    var cond = MemoConditionFactory.Build(condNode);
    if (cond != null) m.Conditions.Add(cond);
}
```

with this:

```csharp
foreach (var c in BuildConditions(node))
    m.Conditions.Add(c);
```

Then add a private static method to the class:

```csharp
private static System.Collections.Generic.IEnumerable<IMemoCondition> BuildConditions(ISceneNode node)
{
    foreach (var condNode in node.GetNodes("CONDITION"))
    {
        var cond = MemoConditionFactory.Build(condNode);
        if (cond != null) yield return cond;
    }
    foreach (var allNode in node.GetNodes("ALL"))
    {
        var c = new CompositeCondition { Mode = CompositeMode.All };
        foreach (var child in BuildConditions(allNode)) c.Children.Add(child);
        yield return c;
    }
    foreach (var anyNode in node.GetNodes("ANY"))
    {
        var c = new CompositeCondition { Mode = CompositeMode.Any };
        foreach (var child in BuildConditions(anyNode)) c.Children.Add(child);
        yield return c;
    }
}
```

- [ ] **Step 2: Run tests**

```bash
cd "GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

Expected: all tests pass (new and existing).

- [ ] **Step 3: Commit**

```bash
git add tests/KerbalAdminKit.Tests/MemoLoaderTests.cs \
        src/KerbalAdminKit/Memos/MemoLoader.cs
git commit -m "feat(kak): MemoLoader parses ANY/ALL composite blocks recursively"
```

### Task 17: Build and deploy framework changes

**Files:**
- Generated: `GameData/KerbalCampaignKit/Plugins/KerbalCampaignKit.dll`
- Generated: `GameData/KerbalAdminKit/Plugins/KerbalAdminKit.dll`

- [ ] **Step 1: Build KCK in Release**

```bash
cd "GameData/KerbalCampaignKit"
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: build succeeds; `Plugins/KerbalCampaignKit.dll` updated (newer mtime).

- [ ] **Step 2: Build KAK in Release**

```bash
cd "../KerbalAdminKit"
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

Expected: build succeeds; `Plugins/KerbalAdminKit.dll` updated.

- [ ] **Step 3: Run all tests in both repos as a final regression check**

```bash
cd "../KerbalCampaignKit" && dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
cd "../KerbalAdminKit" && dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

Expected: 100% pass on both.

- [ ] **Step 4: Confirm DLL changes are tracked by git in both repos**

```bash
cd "../KerbalCampaignKit" && git status Plugins/
cd "../KerbalAdminKit"   && git status Plugins/
```

Expected: each repo shows its own `Plugins/<Name>.dll` as modified.

- [ ] **Step 5: Commit DLL updates in each repo**

```bash
cd "../KerbalCampaignKit"
git add Plugins/KerbalCampaignKit.dll
git commit -m "build: KCK Plugins/ rebuild after framework additions"
cd "../KerbalAdminKit"
git add Plugins/KerbalAdminKit.dll
git commit -m "build: KAK Plugins/ rebuild after framework additions"
```

---

## Phase C — Content cfg (BadgKatCareer)

All cfg files. Each task is "create file with content shown, then verify by launching KSP and checking the log." For tasks that are purely additive cfg, the validation is loading without errors. The chapter/trigger machine validation is in Phase D.

**Pattern for each cfg task:**
1. Create file with exact content shown.
2. Launch KSP, wait for the main menu, exit.
3. Check `KSP.log` (root of KSP install) for relevant ERROR or WRN lines.
4. Commit if log is clean.

### Task 18: Create AdminKitSettings.cfg

**Files:**
- Create: `DirectorContent/AdminKitSettings.cfg`

- [ ] **Step 1: Create the directory structure**

```bash
cd "GameData/BadgKatCareer"
mkdir -p DirectorContent/Scenes
```

- [ ] **Step 2: Write AdminKitSettings.cfg**

```cfg
// AdminKitSettings.cfg — BadgKatCareer activates KAK's admin-building replacement.
// Without this file (or this block), KAK stays inert and stock admin remains.

KERBAL_ADMIN_KIT
{
    replaceAdminBuilding = true
    deskMemoCount = 5
    memoPollSeconds = 5
}
```

- [ ] **Step 3: Launch KSP, watch for KAK initialization**

Launch KSP. From `KSP.log`, expect a line like `[KerbalAdminKit] AdminKitSettings: replaceAdminBuilding=true ...`. No `[ERR]`/`[WRN]` from `KerbalAdminKit`.

- [ ] **Step 4: Commit**

```bash
git add DirectorContent/AdminKitSettings.cfg
git commit -m "content: enable KAK admin-building replacement (KERBAL_ADMIN_KIT block)"
```

### Task 19: Create Characters.cfg

**Files:**
- Create: `DirectorContent/Characters.cfg`

- [ ] **Step 1: Write Characters.cfg with all 6 characters**

```cfg
// Characters.cfg — 6 DIRECTOR_CHARACTER entries for the BadgKatCareer cast.
// Color-swatch chips (no portraits in v0.1 slice). dispositionFlag follows the
// <id>_disposition convention so KCK actions can target dispositions by flag.

DIRECTOR_CHARACTER
{
    id = gene
    displayName = Gene Kerman
    role = Mission Director
    baseColor = #FF8BE08A
    dispositionFlag = gene_disposition
    defaultDisposition = Neutral
}

DIRECTOR_CHARACTER
{
    id = wernher
    displayName = Wernher von Kerman
    role = Chief Scientist
    baseColor = #FF82B4E8
    dispositionFlag = wernher_disposition
    defaultDisposition = Neutral
}

DIRECTOR_CHARACTER
{
    id = gus
    displayName = Gus Kerman
    role = Chief Engineer
    baseColor = #FFFFC078
    dispositionFlag = gus_disposition
    defaultDisposition = Neutral
}

DIRECTOR_CHARACTER
{
    id = mortimer
    displayName = Mortimer Kerman
    role = Director of Finance
    baseColor = #FFFFE066
    dispositionFlag = mortimer_disposition
    defaultDisposition = Skeptical
}

DIRECTOR_CHARACTER
{
    id = walt
    displayName = Walt Kerman
    role = Director of Public Relations
    baseColor = #FFC8A0E8
    dispositionFlag = walt_disposition
    defaultDisposition = Neutral
}

DIRECTOR_CHARACTER
{
    id = linus
    displayName = Linus Kerman
    role = Signals Analyst
    baseColor = #FF6ED4C8
    dispositionFlag = linus_disposition
    defaultDisposition = Enthusiastic
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect log line `[KerbalAdminKit] CharacterRegistry loaded 6 characters` (or similar). No errors.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Characters.cfg
git commit -m "content: register 6 DIRECTOR_CHARACTER entries (BadgKatCareer cast)"
```

### Task 20: Create Focuses.cfg

**Files:**
- Create: `DirectorContent/Focuses.cfg`

- [ ] **Step 1: Write Focuses.cfg with 6 focuses (1 per character)**

```cfg
// Focuses.cfg — 1 DIRECTOR_FOCUS per character for the slice.
// Placeholder titles/descriptions; mechanical effects of focus choice
// are not queried in this slice (validates picker UI + flag plumbing only).

DIRECTOR_FOCUS
{
    character = gene
    id = mission_pacing
    title = Mission Pacing
    description = Gene watches the schedule. Things will get done when they get done.
    flag = gene_focus
    flagValue = pacing
}

DIRECTOR_FOCUS
{
    character = wernher
    id = anomaly_research
    title = Anomaly Research
    description = Wernher studies what should not be there.
    flag = wernher_focus
    flagValue = anomaly
}

DIRECTOR_FOCUS
{
    character = gus
    id = launch_infrastructure
    title = Launch Infrastructure
    description = Gus keeps the pad standing and the rockets pointed up.
    flag = gus_focus
    flagValue = infra
}

DIRECTOR_FOCUS
{
    character = mortimer
    id = cost_optimization
    title = Cost Optimization
    description = Mortimer counts the funds twice and the receipts three times.
    flag = mortimer_focus
    flagValue = cost
}

DIRECTOR_FOCUS
{
    character = walt
    id = public_outreach
    title = Public Outreach
    description = Walt wants the program in every kerbal's living room.
    flag = walt_focus
    flagValue = outreach
}

DIRECTOR_FOCUS
{
    character = linus
    id = signal_analysis
    title = Signal Analysis
    description = Linus listens to the static and hears patterns.
    flag = linus_focus
    flagValue = signals
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalAdminKit] FocusRegistry loaded 6 focuses` or similar. No errors.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Focuses.cfg
git commit -m "content: register 6 DIRECTOR_FOCUS entries (1 per character, slice placeholders)"
```

### Task 21: Create Reputation.cfg

**Files:**
- Create: `DirectorContent/Reputation.cfg`

- [ ] **Step 1: Write Reputation.cfg**

```cfg
// Reputation.cfg — KCK reputation income tiers, decay, and KAK's PR campaign.
// Numbers are baseline from the redesign spec; tuning happens after playtest.

REPUTATION_INCOME
{
    enabled = true
    intervalDays = 30

    TIER
    {
        min = 0
        max = 100
        income = 5000
        label = Startup
    }

    TIER
    {
        min = 100
        max = 250
        income = 15000
        label = Established
    }

    TIER
    {
        min = 250
        max = 500
        income = 35000
        label = Respected
    }

    TIER
    {
        min = 500
        max = 750
        income = 60000
        label = Renowned
    }

    TIER
    {
        min = 750
        max = 9999
        income = 100000
        label = Prestigious
    }
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

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalCampaignKit] ReputationIncome: 5 tiers ...` and `[KerbalAdminKit] PR campaign: baseCost=50000 ...`. No errors.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Reputation.cfg
git commit -m "content: configure rep income/decay tiers and PR campaign action"
```

### Task 22: Create DispositionDecay.cfg

**Files:**
- Create: `DirectorContent/DispositionDecay.cfg`

- [ ] **Step 1: Write DispositionDecay.cfg with 6 entries**

```cfg
// DispositionDecay.cfg — 1 entry per character. Slow decay (180 days/step)
// validates the KCK time-trigger compilation path. In normal play, contract
// completions in a character's domain snap them back up.

DISPOSITION_DECAY
{
    character = gene
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}

DISPOSITION_DECAY
{
    character = wernher
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}

DISPOSITION_DECAY
{
    character = gus
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}

DISPOSITION_DECAY
{
    character = mortimer
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}

DISPOSITION_DECAY
{
    character = walt
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}

DISPOSITION_DECAY
{
    character = linus
    towardValue = Neutral
    stepDays = 180

    STEP
    {
        from = Enthusiastic
        to = Supportive
    }

    STEP
    {
        from = Supportive
        to = Neutral
    }

    STEP
    {
        from = Frustrated
        to = Skeptical
    }

    STEP
    {
        from = Skeptical
        to = Neutral
    }
}
```

**Note:** every cfg node in this file uses multi-line form per KSP cfg parser requirements (no semicolons, no inline `{ k = v; k = v }`). This is verbose but correct.

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalCampaignKit] DispositionDecay compiled 6 entries to time-triggers`. No parse errors. If errors appear, fix per the WARNING note above and re-launch.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/DispositionDecay.cfg
git commit -m "content: configure 180-day disposition decay for all 6 characters"
```

### Task 23: Create Chapters.cfg

**Files:**
- Create: `DirectorContent/Chapters.cfg`

- [ ] **Step 1: Write Chapters.cfg**

```cfg
// Chapters.cfg — 3 chapters covering the slice arc: Prologue → Day One → stub.
// ENTRY_TRIGGER values reference triggers defined in Triggers.cfg.

CAMPAIGN_CHAPTER
{
    id = 0
    name = Prologue: Telemetry
    description = Before there was a space program, there was a question: can we build something that flies, drives, or floats — and bring back data?
    ENTRY_TRIGGER = ch0_entry
}

CAMPAIGN_CHAPTER
{
    id = 1
    name = Day One
    description = The rover and the biplane prove their worth. Two teams, two paths, both pointing toward the same horizon.
    ENTRY_TRIGGER = ch1_entry
}

CAMPAIGN_CHAPTER
{
    id = 2
    name = Into Orbit
    description = The signal points up. Time to build something that can chase it.
    ENTRY_TRIGGER = ch2_entry
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalCampaignKit] ChapterLoader loaded 3 chapters`. The triggers are not yet defined — KCK may log warnings about unresolved ENTRY_TRIGGER ids; that's expected and resolved in Task 24.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Chapters.cfg
git commit -m "content: define 3 chapters (Prologue, Day One, Into Orbit stub)"
```

### Task 24: Create Triggers.cfg

**Files:**
- Create: `DirectorContent/Triggers.cfg`

- [ ] **Step 1: Write Triggers.cfg**

```cfg
// Triggers.cfg — chapter machine wiring + the directive scene gate.
// Uses KCK's whenExpression and the new auto-published contract:<name>.<state> flags.

CAMPAIGN_TRIGGER
{
    id = ch0_entry
    once = true

    ON_EVENT
    {
        type = FacilityEntered
        facility = SpaceCenter
    }

    ACTIONS
    {
        ADVANCE_CHAPTER
        {
            target = 0
        }
    }
}

CAMPAIGN_TRIGGER
{
    id = ch1_entry
    once = true

    ON_EVENT
    {
        type = ContractCompleted
        contractType = BKEX_FirstScience
    }

    ACTIONS
    {
        ADVANCE_CHAPTER
        {
            target = 1
        }

        ENQUEUE_SCENE
        {
            sceneId = staff_first_recovery
        }

        NOTIFY
        {
            target = admin
            severity = Info
            source = ch1_welcome
        }
    }
}

CAMPAIGN_TRIGGER
{
    id = ch2_entry
    once = true

    ON_EVENT
    {
        type = ContractCompleted
        contractType = BKEX_RoverMonolith
    }

    ON_EVENT
    {
        type = ContractCompleted
        contractType = BKEX_FlyToIsland
    }

    whenExpression = contract:BKEX_RoverMonolith.complete && contract:BKEX_FlyToIsland.complete

    ACTIONS
    {
        ENQUEUE_SCENE
        {
            sceneId = ch1_directive_gene_wernher
        }
    }
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalCampaignKit] TriggerLoader loaded 3 triggers`. No `whenExpression` parse errors. Chapter ENTRY_TRIGGER warnings from Task 23 should now resolve.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Triggers.cfg
git commit -m "content: chapter-transition triggers using whenExpression on auto-published contract flags"
```

### Task 25: Create Memos.cfg

**Files:**
- Create: `DirectorContent/Memos.cfg`

- [ ] **Step 1: Write Memos.cfg**

```cfg
// Memos.cfg — 5 memos covering 4 distinct condition-type paths plus a directive-followup split.
// Text is functional placeholder; rewrite in the polish pass after in-game validation.

DIRECTOR_MEMO
{
    id = mortimer_low_funds
    character = mortimer
    priority = high
    text = Funds are getting tight. We can stretch what we have, but the next big build is going to need careful planning.
    postToStockMail = true

    CONDITION
    {
        type = FundsBelow
        threshold = 25000
    }
}

DIRECTOR_MEMO
{
    id = walt_rep_milestone
    character = walt
    priority = normal
    text = Coverage is picking up. The press wants to know what we are doing next — keep delivering and the cycle keeps spinning our way.

    CONDITION
    {
        type = ReputationAbove
        threshold = 50
    }
}

DIRECTOR_MEMO
{
    id = gus_day_one_kit
    character = gus
    priority = normal
    text = The rover prototype is on the workshop floor. The biplane is on the runway. Pick your poison — both teams are ready.

    CONDITION
    {
        type = ChapterAtLeast
        chapter = 1
    }
}

DIRECTOR_MEMO
{
    id = wernher_directive_followup_space
    character = wernher
    priority = normal
    text = Pushing to space changes everything. I am moving telescope time and signals work to the top of the queue.

    CONDITION
    {
        type = FlagExpression
        expression = ch1_choice == space
    }

    CONDITION
    {
        type = ChapterAtLeast
        chapter = 2
    }
}

DIRECTOR_MEMO
{
    id = wernher_directive_followup_ground
    character = wernher
    priority = normal
    text = More ground work it is. There are anomalies left to find and we have not exhausted what Kerbin is willing to tell us.

    CONDITION
    {
        type = FlagExpression
        expression = ch1_choice == ground
    }

    CONDITION
    {
        type = ChapterAtLeast
        chapter = 2
    }
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalAdminKit] MemoLoader loaded 5 memos`. No parse errors. The `FlagExpression` condition syntax `ch1_choice == space` should be valid per KCK's `RequirementExpression` parser (already used by KDK's `visibleIf`).

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Memos.cfg
git commit -m "content: 5 DIRECTOR_MEMO entries spanning Funds/Rep/Chapter/Flag conditions"
```

### Task 26: Create BuildingScenes.cfg

**Files:**
- Create: `DirectorContent/BuildingScenes.cfg`

- [ ] **Step 1: Write BuildingScenes.cfg**

```cfg
// BuildingScenes.cfg — 4 character chips overlaid on KSC building exteriors.
// Each onClickScene id resolves against the SCENE blocks in Scenes/building_overlays.cfg.

DIRECTOR_BUILDING_SCENE
{
    facility = MissionControl
    character = gene
    onClickScene = gene_mc_status
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

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalAdminKit] BuildingSceneLoader loaded 4 building scenes`. KAK may log warnings about unresolved onClickScene ids — these are resolved when Task 28 lands.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/BuildingScenes.cfg
git commit -m "content: 4 DIRECTOR_BUILDING_SCENE chips on MC/Tracking/Admin"
```

### Task 27: Create Scenes/staff_first_recovery.cfg

**Files:**
- Create: `DirectorContent/Scenes/staff_first_recovery.cfg`

- [ ] **Step 1: Write the scene**

```cfg
// staff_first_recovery.cfg — fires on Ch0→Ch1 transition (first recovered science).
// Stub voice; rewrite in polish pass.

SCENE
{
    id = staff_first_recovery
    title = Data In Hand
    pauseGame = true

    LINE
    {
        speaker = gene
        text = Data is in. Vessel recovered, telemetry intact, no one set anything on fire. That is what we call a good day.
    }

    LINE
    {
        speaker = walt
        text = And that is what we call a press release. I will draft something. Something modest. Something with the word "historic" in it.
    }

    LINE
    {
        speaker = gene
        text = Modest, Walt.

    }

    LINE
    {
        speaker = walt
        text = Modestly historic. Got it.
    }
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalDialogueKit] SceneRegistry registered staff_first_recovery`. No parse errors.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Scenes/staff_first_recovery.cfg
git commit -m "content: staff scene for Ch0→Ch1 transition (first recovery)"
```

### Task 28: Create Scenes/building_overlays.cfg

**Files:**
- Create: `DirectorContent/Scenes/building_overlays.cfg`

- [ ] **Step 1: Write 4 stub scenes**

```cfg
// building_overlays.cfg — 4 stub one-liner scenes for the building chip clicks.
// Voice work happens in the polish pass; these only validate the click→scene plumbing.

SCENE
{
    id = gene_mc_status
    title = Mission Control
    pauseGame = false

    LINE
    {
        speaker = gene
        text = Boards are green. Vessels accounted for. We are in good shape, all things considered.
    }
}

SCENE
{
    id = linus_tracking_signals
    title = Tracking Station
    pauseGame = false

    LINE
    {
        speaker = linus
        text = Antennas are clear. I am chasing some structured noise from the monolith side, but nothing actionable yet.
    }
}

SCENE
{
    id = mortimer_admin_budget
    title = The Numbers
    pauseGame = false

    LINE
    {
        speaker = mortimer
        text = Funds are what they are. I will keep the lights on if you keep the rockets pointed up.
    }
}

SCENE
{
    id = walt_admin_pr
    title = The Story
    pauseGame = false

    LINE
    {
        speaker = walt
        text = Public is paying attention. We can spin almost anything into a win, but we cannot spin a hole in the ground. Try not to make holes.
    }
}
```

- [ ] **Step 2: Launch KSP, verify load**

Expect `[KerbalDialogueKit] SceneRegistry registered 4 scenes`. The unresolved-onClickScene warnings from Task 26 should clear.

- [ ] **Step 3: Commit**

```bash
git add DirectorContent/Scenes/building_overlays.cfg
git commit -m "content: 4 stub scenes for building-chip overlays (MC/Tracking/Admin)"
```

### Task 29: Lift BothLeads dialogue + author the directive scene

**Files:**
- Read: git history (`git log --all --diff-filter=D -- ContractPacks/Exploration/BridgeContracts.cfg`)
- Create: `DirectorContent/Scenes/ch1_directive_gene_wernher.cfg`

- [ ] **Step 1: Recover the BothLeads dialogue from the deleted bridge file**

```bash
cd "GameData/BadgKatCareer"
git show HEAD~6:ContractPacks/Exploration/BridgeContracts.cfg | head -100
```

(`HEAD~6` assumes Phase A's commits sequenced 1-7 + framework + earlier content tasks landed in order; adjust the offset if not. Easier alternative: `git log --all --oneline -- ContractPacks/Exploration/BridgeContracts.cfg`, then `git show <commit>:ContractPacks/Exploration/BridgeContracts.cfg`.)

Capture the 3 LINE blocks under `BKEX_Bridge_BothLeads`'s ChatterBox BEHAVIOUR (Linus + Gene about the structured signal pointing up).

- [ ] **Step 2: Write the directive scene**

```cfg
// ch1_directive_gene_wernher.cfg — fires on Ch1→Ch2 boundary.
// Opens with Linus's lifted BothLeads dialogue, then forks via CHOICE_CARD.
// Both branches advance to chapter 2; the flag value differentiates downstream.

SCENE
{
    id = ch1_directive_gene_wernher
    title = Something Is Pointing at the Sky
    pauseGame = true
    onFacilityEnter = Administration

    LINE
    {
        speaker = linus
        text = The monolith signal — I have been running it through every algorithm I have. It is structured, Gene. Not natural noise, not interference. Something encoded it deliberately. And the geometry of the field lines is pointing up. Off-planet.
    }

    LINE
    {
        speaker = gene
        text = So whatever left that thing wants us to follow it somewhere we cannot drive to. We are going to need rockets. Bigger ones than we have built so far.
    }

    LINE
    {
        speaker = wernher
        text = There are also structures we have not visited. The desert pyramids, the ice formations, more of Kerbin to investigate. Two directions, both legitimate.
    }

    LINE
    {
        speaker = gene
        text = So which is it — push to space, or keep working the ground first? It is your program.
    }

    CHOICE_CARD
    {
        title = Where do we point next?

        OPTION
        {
            id = space
            text = Push to space — the signal is calling.
            setFlag = ch1_choice
            flagValue = space
            advanceChapter = 2
        }

        OPTION
        {
            id = ground
            text = More ground work first — Kerbin still has secrets.
            setFlag = ch1_choice
            flagValue = ground
            advanceChapter = 2
        }
    }
}
```

**Note on `advanceChapter` and `setFlag` in OPTION blocks:** check whether KDK's `OPTION` cfg supports these inline action attributes. If not (and KDK currently only sets a single `setFlag/flagValue` pair), then split the action into a KCK trigger that listens on the flag-set event and calls `ADVANCE_CHAPTER`. Verify by grep:

```bash
grep -A 30 "class.*Option\|class.*ChoiceCard" "../KerbalDialogueKit/src/KerbalDialogueKit/"**/*.cs
```

If `advanceChapter` is not a recognized OPTION attribute, replace the OPTION block with `setFlag` only, and add this companion trigger to `Triggers.cfg`:

```cfg
CAMPAIGN_TRIGGER
{
    id = ch2_advance_on_choice
    once = true

    ON_EVENT
    {
        type = FlagSet
        flag = ch1_choice
    }

    ACTIONS
    {
        ADVANCE_CHAPTER
        {
            target = 2
        }
    }
}
```

(KCK has a `FlagEventSource` per the file listing; FlagSet event type should exist.)

- [ ] **Step 3: Launch KSP, verify load**

Expect `[KerbalDialogueKit] SceneRegistry registered ch1_directive_gene_wernher`. The `ENQUEUE_SCENE { sceneId = ch1_directive_gene_wernher }` warning from Task 24 should resolve.

- [ ] **Step 4: Commit**

```bash
git add DirectorContent/Scenes/ch1_directive_gene_wernher.cfg
# If the trigger fallback was needed, also:
# git add DirectorContent/Triggers.cfg
git commit -m "content: Ch1→Ch2 directive scene (Gene vs Wernher) with lifted BothLeads opening"
```

### Task 30: Append chapter-gate REQUIREMENTs to existing contracts

**Files:**
- Modify: `ContractTweaks.cfg` (append patches to existing file)

- [ ] **Step 1: Append `@CONTRACT_TYPE` patches**

Append to the end of `ContractTweaks.cfg`:

```cfg
// ============================================================================
// CHAPTER GATES — added by BadgKatCareer Day One slice
// Requires KerbalCampaignKit's ChapterAtLeast REQUIREMENT type.
// ============================================================================

@CONTRACT_TYPE[BKEX_RoverMonolith]:NEEDS[KerbalCampaignKit]
{
    REQUIREMENT
    {
        name = ChapterGate
        type = ChapterAtLeast
        chapter = 1
    }
}

@CONTRACT_TYPE[BKEX_FlyToIsland]:NEEDS[KerbalCampaignKit]
{
    REQUIREMENT
    {
        name = ChapterGate
        type = ChapterAtLeast
        chapter = 1
    }
}

@CONTRACT_TYPE[BKEX_UnmannedSuborbital]:NEEDS[KerbalCampaignKit]
{
    REQUIREMENT
    {
        name = ChapterGate
        type = ChapterAtLeast
        chapter = 2
    }
}
```

- [ ] **Step 2: Launch KSP, verify ModuleManager applies the patches**

After launch, `KSP.log` should show 3 lines like `[ModuleManager] Applying update BadgKatCareer/ContractTweaks/@CONTRACT_TYPE[BKEX_RoverMonolith] to ...`. No CC errors about unknown REQUIREMENT type `ChapterAtLeast`.

- [ ] **Step 3: Commit**

```bash
git add ContractTweaks.cfg
git commit -m "content: chapter gates on RoverMonolith / FlyToIsland (Ch1) and UnmannedSuborbital (Ch2)"
```

### Task 31: Append directive flag-gate REQUIREMENTs

**Files:**
- Modify: `ContractTweaks.cfg`

- [ ] **Step 1: Append the flag-gate patches**

```cfg
// ============================================================================
// DIRECTIVE FLAG GATES — Ch1 directive choice splits Ch2 contract availability.
// space branch unlocks orbital track first; ground branch unlocks aviation tier-2.
// Both branches eventually unlock both — only sequencing differs.
// ============================================================================

@CONTRACT_TYPE[BKEX_UnmannedSuborbital]:NEEDS[KerbalCampaignKit]
{
    REQUIREMENT
    {
        name = DirectiveSpaceGate
        type = FlagEquals
        flag = ch1_choice
        value = space
    }
}

@CONTRACT_TYPE[BKML_SoundBarrier]:NEEDS[KerbalCampaignKit]
{
    REQUIREMENT
    {
        name = ChapterGate
        type = ChapterAtLeast
        chapter = 2
    }

    REQUIREMENT
    {
        name = DirectiveGroundGate
        type = FlagEquals
        flag = ch1_choice
        value = ground
    }
}
```

- [ ] **Step 2: Launch KSP, verify**

Expect 2 more `Applying update` log lines (one for each `@CONTRACT_TYPE` block above). `BKEX_UnmannedSuborbital` now has 2 added REQUIREMENTs (chapter gate from Task 30 + directive gate from this task).

- [ ] **Step 3: Commit**

```bash
git add ContractTweaks.cfg
git commit -m "content: directive flag gates on UnmannedSuborbital (space) and SoundBarrier (ground)"
```

---

## Phase D — Validation

In-game playtest. No new files; no commits unless bugs surface and are fixed (each fix is its own follow-up commit).

### Task 32: Cold-load smoke check

**Files:** none modified.

- [ ] **Step 1: Delete `GameData/ModuleManager.ConfigCache` and `GameData/ModuleManager.ConfigSHA` to force a fresh patch run**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program"
rm -f GameData/ModuleManager.ConfigCache GameData/ModuleManager.ConfigSHA
```

- [ ] **Step 2: Launch KSP. Wait for the main menu. Exit.**

- [ ] **Step 3: Inspect `KSP.log`**

```bash
grep -E "\[ERR\]|\[WRN\]|Exception" KSP.log | grep -E "KerbalAdminKit|KerbalCampaignKit|KerbalDialogueKit|ContractConfigurator|ModuleManager.*BadgKatCareer"
```

Expected: zero output. Any error or warning is a defect; fix the responsible cfg/code, rebuild if needed, re-run this step.

- [ ] **Step 4: Confirm patch count**

```bash
grep -c "Applying update.*BadgKatCareer/ContractTweaks" KSP.log
```

Expected: 5 (3 chapter gates + 2 directive gates from Tasks 30-31).

### Task 33: Manual playtest sequence

**Files:** none modified.

- [ ] **Step 1: New career save, enter SpaceCenter**

Verify:
- KSC view shows 4 exterior building chips: Mortimer + Walt over Admin, Gene over MC, Linus over Tracking
- `KSP.log`: `ch0_entry` trigger fires, chapter == 0

- [ ] **Step 2: Click Administration building**

Verify: KAK admin UI opens with 6 character chips in the character panel (all DIRECTOR_CHARACTER).

- [ ] **Step 3: Click each character chip in the admin UI**

Verify: focus picker appears for each (1 focus available); selecting any focus dismisses the picker and clears the alert badge on the chip.

- [ ] **Step 4: Click Mortimer's exterior chip on the Admin building**

Verify: `mortimer_admin_budget` scene fires with Mortimer as speaker. Repeat for Walt (admin), Gene (MC), Linus (Tracking). All 4 should fire correctly.

- [ ] **Step 5: Build a vessel, get any science, recover (this satisfies BKEX_FirstScience)**

Verify (from log + game state):
- `BKEX_FirstScience` completes
- `ch1_entry` trigger fires
- chapter == 1
- `staff_first_recovery` scene plays
- `gus_day_one_kit` memo activates (visible on admin desk)

- [ ] **Step 6: Build a rover, drive to Kerbin Monolith (BKEX_RoverMonolith completes)**

Verify (KSP.log):
- `BKEX_RoverMonolith` completes
- KCK writes `contract:BKEX_RoverMonolith.complete = true` flag
- `ch2_entry` trigger does NOT fire (FlyToIsland flag still unset; whenExpression false)

- [ ] **Step 7: Build a biplane, fly to island airfield (BKEX_FlyToIsland completes)**

Verify (KSP.log):
- `BKEX_FlyToIsland` completes
- KCK writes `contract:BKEX_FlyToIsland.complete = true` flag
- `ch2_entry` trigger evaluates whenExpression — both flags now true — fires
- `ch1_directive_gene_wernher` scene queued (waiting for Administration entry)

- [ ] **Step 8: Walk into Administration building**

Verify:
- Directive scene plays
- Linus opening lines (lifted from BothLeads) display correctly
- CHOICE_CARD appears with 2 options

- [ ] **Step 9: Pick "Push to space"**

Verify:
- `ch1_choice = space` flag set
- chapter advances to 2
- In Mission Control: `BKEX_UnmannedSuborbital` is offered (or accept-able from agencies); `BKML_SoundBarrier` is NOT

- [ ] **Step 10: Reload save before step 9, pick "More ground work"**

Verify: `BKEX_UnmannedSuborbital` is NOT offered; `BKML_SoundBarrier` IS offered.

- [ ] **Step 11: Trigger and dismiss `gus_day_one_kit` memo**

(Memo is already active at chapter 1.) Dismiss it via the admin desk. Then wait 5+ seconds (one poll interval). Verify: memo does not re-activate. The rearm-pending machinery from KAK v0.1 should hold it silent until conditions transition false→true.

- [ ] **Step 12: Quicksave, alt-F4, reload**

Verify (after reload):
- chapter == correct value
- `ch1_choice` flag preserved
- Dismissed memos remain dismissed
- Pending-rearm memos still pending

- [ ] **Step 13: Verify both completion orders**

Reload before step 6. Do biplane FIRST (FlyToIsland), then rover (RoverMonolith). Repeat verification of step 7-8 — chapter 2 transition must work in both completion orders.

### Task 34: Reverse validation — mod-disabled and mid-save enable

**Files:** none modified.

- [ ] **Step 1: Disable BadgKatCareer**

Move `GameData/BadgKatCareer/` outside GameData (e.g., to a sibling `_disabled/` folder). Leave KAK / KCK / KDK enabled.

- [ ] **Step 2: Launch KSP, start a stock career save**

Verify:
- KAK is inert: no admin replacement, no exterior building chips, no notification markers
- KCK is inert: no chapter UI, no rep tier income
- KDK is inert: no scenes triggering
- Stock admin building works as in vanilla KSP
- Stock MessageSystem works as in vanilla

- [ ] **Step 3: Re-enable BadgKatCareer mid-save**

Move BadgKatCareer back into GameData. Reload the existing stock save (do NOT start fresh).

Verify:
- chapter starts at 0 (NOT retroactively advanced)
- Previously-completed contracts in the save do NOT retro-fire any triggers
- No crashes, no exception spam in KSP.log

### Task 35: Final commit + push (per repo)

**Files:** none modified — house-keeping only.

- [ ] **Step 1: Confirm working trees clean in all three repos**

```bash
cd "GameData/BadgKatCareer"   && git status
cd "../KerbalCampaignKit"     && git status
cd "../KerbalAdminKit"        && git status
```

Expected: all three "nothing to commit, working tree clean".

- [ ] **Step 2: Push all three branches**

```bash
cd "GameData/BadgKatCareer"   && git push origin master
cd "../KerbalCampaignKit"     && git push origin main
cd "../KerbalAdminKit"        && git push origin main
```

Expected: all three pushes succeed.

- [ ] **Step 3: No further commits. Slice complete.**

---

## Out-of-scope cleanup notes (for later plans)

These are NOT in this slice but were noted during brainstorming and should be tracked for future plans:

- **Voice rewrite pass** for memos and stub scenes — placeholder text in this slice, voice work happens after in-game validation
- **Ch2 vertical slice** — Into Orbit content + Mortimer-vs-Wernher and Gene-vs-Walt directives
- **Bridge dialogue migration** for MunSignals / TraceSignal / GoingDeeper — those will be lifted from git history into proper KCK trigger + KDK scene patterns when their respective chapters are planned
- **Full focus expansion** — bump from 1 focus per character to 3 once axes have proven gameplay weight
- **Portrait artwork** — drop PNGs into `BadgKatCareer/Art/`, add `CHIP_LAYER` blocks
