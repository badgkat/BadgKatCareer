# KerbalCampaignKit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone utility library (`KerbalCampaignKit.dll`) that extends KerbalDialogueKit with a career-scale state machine: named narrative chapters, a general event→action trigger system, a reputation economy, and a hierarchical notification API. Foundational for BadgKatDirector and any future mod wanting a story-driven career.

**Architecture:** A new C# mod at `GameData/KerbalCampaignKit/` in its own git repo. Depends on KerbalDialogueKit (already shipped v0.1.0) for flag storage, scene enqueue, and flag-expression parsing. Provides a `CampaignKit` static API, a `ChapterScenario` for save persistence, a trigger engine that dispatches KSP GameEvents to cfg-defined actions, and CC REQUIREMENT types for contract gating. Core logic (chapter state, notification store, trigger dispatch, reputation math) is unit-testable in an xUnit project without KSP.

**Tech Stack:** C# targeting .NET Framework 4.7.2 (KSP's Unity runtime), MSBuild/`dotnet` CLI for builds, xUnit for unit tests. References: KSP 1.12.x assemblies, KerbalDialogueKit.dll (installed via ChatterBox-compatible public-API fork), ContractConfigurator.dll.

**Reference:** Full spec at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer/docs/superpowers/specs/2026-04-23-campaign-kit-design.md`

---

## Context For The Engineer

**Working directory setup.** The user's KSP install is at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/`. Mods are installed under `GameData/`. We create `GameData/KerbalCampaignKit/` as a new folder and `git init` inside it. **Source and build artifacts** (csproj, test project, README) live at the repo root; only `Plugins/KerbalCampaignKit.dll`, the `.version` file, and example `.cfg` files are the shipped mod content — matching KerbalDialogueKit (KDK)'s convention.

**KSP modding basics.** KSP 1.12 runs on Unity 2019 with .NET Framework 4.x. Plugins live in `GameData/<ModName>/Plugins/` and load at startup. Key APIs we use:
- `[KSPAddon]` attribute + `MonoBehaviour` for scene-lifecycle runtime components
- `[KSPScenario]` + `ScenarioModule` for per-save persistent state
- `GameEvents` static class: `EventData<T>` fields for subscribing to contract complete, vessel recovered, facility enter, etc.
- `ConfigNode` class: KSP's cfg format (braces, `key = value`, nested nodes)
- `ContractConfigurator.ContractRequirement` base class for CC REQUIREMENT types
- `ContractConfigurator.ContractBehaviour` base class for CC BEHAVIOUR types

**KDK integration.** KDK ships at `GameData/KerbalDialogueKit/Plugins/KerbalDialogueKit.dll`. We reference it directly in our csproj with `<Private>false</Private>`. We use these KDK types:
- `KerbalDialogueKit.Core.DialogueKit` — static entry: `Enqueue`, `EnqueueById`, `Flags`
- `KerbalDialogueKit.Flags.FlagStore` — flag KV store (`Get`, `Set`, `Remove`, `All`)
- `KerbalDialogueKit.Flags.FlagExpression` + `FlagExpressionParser` — reusable expression parser
- `KerbalDialogueKit.Core.SceneCallbacks` — for scene lifecycle hooks

**Build output pattern.** Directory.Build.props redirects `obj/` and `bin/` outside `GameData/` (to `C:\Users\bklen\ksp-dev\KCK-build\`) so KSP's assembly loader doesn't pick up stale test/intermediate DLLs. The main project's csproj overrides `OutputPath` to write the shipping DLL directly to `Plugins/`. This pattern is proven — it's exactly what KDK uses.

**Testing strategy.**
- **Pure logic** (chapters, notifications, triggers, actions, reputation math, loaders): unit-tested via xUnit. Classes must not reference `UnityEngine` or KSP assemblies.
- **KSP-dependent** (event sources, ScenarioModule, CC REQUIREMENTs/BEHAVIOURs, the addon): manual in-game verification via a small test `.cfg` that exercises the trigger pipeline end-to-end.

**Configuration patterns for testability.** KSP's `ConfigNode` type is sealed and hard to test without KSP. We abstract parsing behind an `ISceneNode`-style interface (see KDK's `ConfigNodeAdapter`) so loaders consume an interface in tests but wrap real `ConfigNode` in production. KDK already has this pattern — reuse the shape.

**Commit cadence.** Commit after every task completes and its tests pass. Follow conventional commit prefixes (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `refactor:`). Never include `Co-Authored-By:` trailers.

**Authoritative path prefix.** All paths below are relative to `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalCampaignKit/` unless stated.

---

## File Structure

```
GameData/KerbalCampaignKit/                    ← repo root, also mod root
├── .gitignore
├── README.md
├── LICENSE                                     ← MIT
├── KerbalCampaignKit.sln
├── Directory.Build.props                       ← redirect obj/bin outside GameData
├── src/
│   └── KerbalCampaignKit/
│       ├── KerbalCampaignKit.csproj
│       ├── Core/
│       │   ├── CampaignKit.cs                  ← public static API entry
│       │   ├── CampaignKitScenario.cs          ← ScenarioModule persists state
│       │   ├── CampaignKitAddon.cs             ← [KSPAddon] runtime bootstrap
│       │   └── StateMirror.cs                  ← mirrors career state into KDK flags
│       ├── Chapters/
│       │   ├── Chapter.cs                      ← chapter data (id, name, triggers)
│       │   ├── ChapterHistory.cs               ← list of ChapterEntry records
│       │   ├── ChapterManager.cs               ← current chapter, Advance, IsAtLeast
│       │   └── ChapterLoader.cs                ← parse CAMPAIGN_CHAPTER cfg
│       ├── Triggers/
│       │   ├── Trigger.cs                      ← trigger data model
│       │   ├── EventSpec.cs                    ← parsed ON_EVENT block
│       │   ├── ActionSpec.cs                   ← parsed ACTIONS entry
│       │   ├── TriggerLoader.cs                ← parse CAMPAIGN_TRIGGER cfg
│       │   ├── TriggerEngine.cs                ← dispatch, dedup, action execution
│       │   ├── RequirementExpression.cs        ← wraps KDK FlagExpressionParser
│       │   ├── Events/
│       │   │   ├── IEventSource.cs
│       │   │   ├── EventType.cs                ← enum of supported event types
│       │   │   ├── EventRecord.cs              ← event + payload passed to triggers
│       │   │   ├── ContractEventSource.cs
│       │   │   ├── VesselEventSource.cs
│       │   │   ├── KerbalEventSource.cs
│       │   │   ├── CurrencyEventSource.cs
│       │   │   ├── FacilityEventSource.cs
│       │   │   ├── TimeEventSource.cs
│       │   │   ├── ChapterEventSource.cs
│       │   │   ├── FlagEventSource.cs
│       │   │   ├── SceneEventSource.cs
│       │   │   ├── BodyDiscoveredEventSource.cs
│       │   │   └── GameEventBridgeSource.cs
│       │   └── Actions/
│       │       ├── IAction.cs
│       │       ├── SetFlagAction.cs
│       │       ├── ClearFlagAction.cs
│       │       ├── AdvanceChapterAction.cs
│       │       ├── AdjustCurrencyAction.cs
│       │       ├── EnqueueSceneAction.cs
│       │       ├── NotifyAction.cs
│       │       ├── ClearNotificationAction.cs
│       │       └── PublishEventAction.cs
│       ├── Reputation/
│       │   ├── ReputationEconomy.cs            ← tick, halt, gates
│       │   ├── ReputationIncome.cs             ← tier table + math
│       │   ├── ReputationDecay.cs              ← decay math + floors
│       │   ├── ReputationLoader.cs             ← parse REPUTATION_INCOME/DECAY
│       │   └── ReputationStakeBehaviour.cs     ← CC BEHAVIOUR
│       ├── Notifications/
│       │   ├── Notification.cs                 ← target, severity, source, clearOn
│       │   ├── NotificationSeverity.cs         ← enum
│       │   ├── NotificationClearOn.cs          ← enum
│       │   ├── NotificationStore.cs            ← hierarchical add/clear/query
│       │   ├── AutoClearWatcher.cs             ← wires auto-clear rules
│       │   └── CampaignKitEvents.cs            ← GameEvents publisher
│       ├── Contracts/
│       │   ├── InChapterRequirement.cs
│       │   ├── ChapterAtLeastRequirement.cs
│       │   ├── FlagEqualsRequirement.cs
│       │   ├── FlagNotEqualsRequirement.cs
│       │   ├── FlagExpressionRequirement.cs
│       │   └── ReputationMinimumRequirement.cs
│       ├── PendingScenes/
│       │   ├── PendingScene.cs
│       │   └── PendingSceneQueue.cs
│       └── Config/
│           └── ConfigNodeAdapter.cs            ← abstraction for testability
├── tests/
│   └── KerbalCampaignKit.Tests/
│       ├── KerbalCampaignKit.Tests.csproj
│       ├── TestHelpers/
│       │   ├── FakeConfigNode.cs
│       │   └── TestFlagStore.cs
│       ├── ChapterManagerTests.cs
│       ├── ChapterLoaderTests.cs
│       ├── NotificationStoreTests.cs
│       ├── AutoClearWatcherTests.cs
│       ├── TriggerLoaderTests.cs
│       ├── TriggerEngineTests.cs
│       ├── RequirementExpressionTests.cs
│       ├── ActionTests.cs
│       ├── PendingSceneQueueTests.cs
│       ├── ReputationIncomeTests.cs
│       ├── ReputationDecayTests.cs
│       └── SmokeTest.cs
├── Plugins/                                    ← build output lands here
├── Examples/
│   └── demo-campaign.cfg
└── KerbalCampaignKit.version                   ← KSP-AVC version file
```

---

## Task 1: Repo init + Directory.Build.props + LICENSE + .gitignore

**Files:**
- Create: `GameData/KerbalCampaignKit/.gitignore`
- Create: `GameData/KerbalCampaignKit/Directory.Build.props`
- Create: `GameData/KerbalCampaignKit/LICENSE`
- Create: `GameData/KerbalCampaignKit/KerbalCampaignKit.sln`
- Create: `GameData/KerbalCampaignKit/Plugins/.gitkeep`

- [ ] **Step 1: Initialize the repo and create top-level files**

Run:
```bash
mkdir -p "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalCampaignKit"
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalCampaignKit"
git init
mkdir -p Plugins src tests Examples
touch Plugins/.gitkeep
```

- [ ] **Step 2: Create `.gitignore`**

Contents:
```
# Build outputs
bin/
obj/
*.user
*.suo
.vs/

# Exclude compiled DLL from source tree (built into Plugins/ by build)
src/*/bin/
src/*/obj/
tests/*/bin/
tests/*/obj/

# But DO ship Plugins/KerbalCampaignKit.dll — that's the installed artifact
!Plugins/*.dll
# Don't commit debug symbol files
Plugins/*.pdb

# Editor
.idea/
*.swp
```

- [ ] **Step 3: Create `Directory.Build.props`**

Contents:
```xml
<Project>
  <PropertyGroup>
    <!-- Redirect intermediate outputs (obj/) outside GameData so KSP
         doesn't load stale/debug copies of our assemblies. -->
    <BaseIntermediateOutputPath>C:\Users\bklen\ksp-dev\KCK-build\obj\$(MSBuildProjectName)\</BaseIntermediateOutputPath>

    <!-- Default BaseOutputPath (bin/) also goes outside GameData. Projects
         that ship override OutputPath to go to Plugins/ explicitly. -->
    <BaseOutputPath>C:\Users\bklen\ksp-dev\KCK-build\bin\$(MSBuildProjectName)\</BaseOutputPath>
  </PropertyGroup>
</Project>
```

- [ ] **Step 4: Create `LICENSE` (MIT)**

Contents:
```
MIT License

Copyright (c) 2026 badgkat

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 5: Create `KerbalCampaignKit.sln`**

Run:
```bash
dotnet new sln -n KerbalCampaignKit
```

- [ ] **Step 6: Commit**

```bash
git add .gitignore Directory.Build.props LICENSE KerbalCampaignKit.sln Plugins/.gitkeep
git commit -m "chore: initialize repo scaffolding"
```

---

## Task 2: Main library csproj + Core namespace placeholder

**Files:**
- Create: `src/KerbalCampaignKit/KerbalCampaignKit.csproj`
- Create: `src/KerbalCampaignKit/Core/CampaignKit.cs` (stub)

- [ ] **Step 1: Create `src/KerbalCampaignKit/KerbalCampaignKit.csproj`**

Contents:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <AssemblyName>KerbalCampaignKit</AssemblyName>
    <RootNamespace>KerbalCampaignKit</RootNamespace>
    <LangVersion>7.3</LangVersion>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <OutputPath>$(MSBuildThisFileDirectory)..\..\Plugins\</OutputPath>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <AutoGenerateBindingRedirects>false</AutoGenerateBindingRedirects>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="Assembly-CSharp">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Assembly-CSharp-firstpass">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp-firstpass.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="UnityEngine">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\UnityEngine.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="UnityEngine.CoreModule">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\UnityEngine.CoreModule.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="KerbalDialogueKit">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\Plugins\KerbalDialogueKit.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="ContractConfigurator">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\ContractConfigurator\Plugins\ContractConfigurator.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>

</Project>
```

- [ ] **Step 2: Create `src/KerbalCampaignKit/Core/CampaignKit.cs` (stub)**

Contents:
```csharp
namespace KerbalCampaignKit.Core
{
    /// <summary>
    /// Public static entry point. Consumer mods call these methods to
    /// interact with KerbalCampaignKit.
    /// </summary>
    public static class CampaignKit
    {
    }
}
```

- [ ] **Step 3: Add project to solution and build**

Run:
```bash
dotnet sln KerbalCampaignKit.sln add src/KerbalCampaignKit/KerbalCampaignKit.csproj
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: build succeeds, `Plugins/KerbalCampaignKit.dll` exists.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/KerbalCampaignKit.csproj src/KerbalCampaignKit/Core/CampaignKit.cs KerbalCampaignKit.sln Plugins/KerbalCampaignKit.dll
git commit -m "feat: scaffold main library project (net472, KDK + CC references)"
```

---

## Task 3: Test project scaffolding

**Files:**
- Create: `tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj`
- Create: `tests/KerbalCampaignKit.Tests/SmokeTest.cs`

- [ ] **Step 1: Create test csproj**

Contents of `tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <IsPackable>false</IsPackable>
    <LangVersion>7.3</LangVersion>
    <GenerateAssemblyInfo>true</GenerateAssemblyInfo>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.6" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.6" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\KerbalCampaignKit\KerbalCampaignKit.csproj" />
  </ItemGroup>

</Project>
```

- [ ] **Step 2: Create `tests/KerbalCampaignKit.Tests/SmokeTest.cs`**

Contents:
```csharp
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class SmokeTest
    {
        [Fact]
        public void CanReferenceCampaignKit()
        {
            // If this compiles and runs, references are wired correctly.
            var type = typeof(KerbalCampaignKit.Core.CampaignKit);
            Assert.NotNull(type);
        }
    }
}
```

- [ ] **Step 3: Add to solution and run tests**

Run:
```bash
dotnet sln KerbalCampaignKit.sln add tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
```

Expected: 1 test passes.

- [ ] **Step 4: Commit**

```bash
git add tests/ KerbalCampaignKit.sln
git commit -m "test: add xunit test project with smoke test"
```

---

## Task 4: Version file + README scaffold

**Files:**
- Create: `KerbalCampaignKit.version`
- Create: `README.md`

- [ ] **Step 1: Create `KerbalCampaignKit.version`**

Contents:
```json
{
  "NAME": "KerbalCampaignKit",
  "URL": "https://raw.githubusercontent.com/badgkat/KerbalCampaignKit/main/KerbalCampaignKit.version",
  "DOWNLOAD": "https://github.com/badgkat/KerbalCampaignKit/releases",
  "VERSION": {
    "MAJOR": 0,
    "MINOR": 1,
    "PATCH": 0
  },
  "KSP_VERSION_MIN": {
    "MAJOR": 1,
    "MINOR": 12,
    "PATCH": 0
  },
  "KSP_VERSION_MAX": {
    "MAJOR": 1,
    "MINOR": 12,
    "PATCH": 99
  }
}
```

- [ ] **Step 2: Create `README.md` scaffold**

Contents:
```markdown
# KerbalCampaignKit

A utility library that extends [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit)
with a career-scale state machine: named narrative chapters, a general event→action trigger
system, a reputation economy, and a hierarchical notification API. Built for KSP 1.12.x
mods that want a story-driven career.

## Status

Early development. v0.1.0 in progress.

## Requirements

- KSP 1.12.x
- [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit) v0.1.0 or later
- [ContractConfigurator](https://forum.kerbalspaceprogram.com/index.php?/topic/91625-112x-contract-configurator/) v2.12.0 or later

## License

MIT — see `LICENSE`.
```

- [ ] **Step 3: Commit**

```bash
git add KerbalCampaignKit.version README.md
git commit -m "docs: add version file and README scaffold"
```

---

## Task 5: ConfigNode adapter (testability abstraction)

**Files:**
- Create: `src/KerbalCampaignKit/Config/ConfigNodeAdapter.cs`
- Create: `tests/KerbalCampaignKit.Tests/TestHelpers/FakeConfigNode.cs`

This abstraction lets loaders consume an interface so tests don't need KSP's sealed `ConfigNode`.

- [ ] **Step 1: Write the interface and adapter**

Contents of `src/KerbalCampaignKit/Config/ConfigNodeAdapter.cs`:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Config
{
    /// <summary>
    /// Abstracts a cfg node for loaders so they can be unit-tested without
    /// KSP's sealed ConfigNode. Wrap a real ConfigNode via ConfigNodeAdapter.
    /// </summary>
    public interface ISceneNode
    {
        string GetValue(string key);
        IEnumerable<string> GetValues(string key);
        IEnumerable<ISceneNode> GetNodes(string name);
        bool HasValue(string key);
        bool HasNode(string name);
    }

    /// <summary>
    /// Wraps a real KSP ConfigNode. Uses duck-typed reflection to avoid a hard
    /// compile-time dependency in test assemblies, but in production this
    /// adapter compiles against the KSP ConfigNode directly.
    /// </summary>
    public sealed class ConfigNodeAdapter : ISceneNode
    {
        private readonly ConfigNode node;

        public ConfigNodeAdapter(ConfigNode node)
        {
            this.node = node;
        }

        public string GetValue(string key) => node.GetValue(key);

        public IEnumerable<string> GetValues(string key) => node.GetValues(key);

        public IEnumerable<ISceneNode> GetNodes(string name)
        {
            foreach (var child in node.GetNodes(name))
                yield return new ConfigNodeAdapter(child);
        }

        public bool HasValue(string key) => node.HasValue(key);
        public bool HasNode(string name) => node.HasNode(name);
    }
}
```

- [ ] **Step 2: Write the test fake**

Contents of `tests/KerbalCampaignKit.Tests/TestHelpers/FakeConfigNode.cs`:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Config;

namespace KerbalCampaignKit.Tests.TestHelpers
{
    /// <summary>
    /// In-memory ISceneNode implementation for tests. Mirrors KSP ConfigNode
    /// semantics: a key can appear multiple times (GetValues); nodes can nest.
    /// </summary>
    public sealed class FakeConfigNode : ISceneNode
    {
        private readonly List<(string key, string value)> values = new List<(string, string)>();
        private readonly List<(string name, FakeConfigNode node)> nodes = new List<(string, FakeConfigNode)>();

        public FakeConfigNode Add(string key, string value)
        {
            values.Add((key, value));
            return this;
        }

        public FakeConfigNode AddNode(string name, FakeConfigNode node)
        {
            nodes.Add((name, node));
            return this;
        }

        public string GetValue(string key)
        {
            foreach (var (k, v) in values)
                if (k == key) return v;
            return null;
        }

        public IEnumerable<string> GetValues(string key)
        {
            foreach (var (k, v) in values)
                if (k == key) yield return v;
        }

        public IEnumerable<ISceneNode> GetNodes(string name)
        {
            foreach (var (n, node) in nodes)
                if (n == name) yield return node;
        }

        public bool HasValue(string key)
        {
            foreach (var (k, _) in values)
                if (k == key) return true;
            return false;
        }

        public bool HasNode(string name)
        {
            foreach (var (n, _) in nodes)
                if (n == name) return true;
            return false;
        }
    }
}
```

- [ ] **Step 3: Build and run tests to verify nothing broke**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
```

Expected: build succeeds, 1 smoke test still passes.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Config/ tests/KerbalCampaignKit.Tests/TestHelpers/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: add ISceneNode abstraction + FakeConfigNode test double"
```

---

## Task 6: Chapter data model (Chapter + ChapterEntry + ChapterHistory)

**Files:**
- Create: `src/KerbalCampaignKit/Chapters/Chapter.cs`
- Create: `src/KerbalCampaignKit/Chapters/ChapterHistory.cs`
- Create: `tests/KerbalCampaignKit.Tests/ChapterHistoryTests.cs`

- [ ] **Step 1: Write the failing test**

Contents of `tests/KerbalCampaignKit.Tests/ChapterHistoryTests.cs`:
```csharp
using KerbalCampaignKit.Chapters;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ChapterHistoryTests
    {
        [Fact]
        public void RecordsEntries_InOrder()
        {
            var h = new ChapterHistory();
            h.RecordEntry("1", 0);
            h.RecordEntry("2", 500);
            h.RecordEntry("3", 1500);

            Assert.Equal(3, h.Entries.Count);
            Assert.Equal("1", h.Entries[0].ChapterId);
            Assert.Equal(0, h.Entries[0].TimestampSeconds);
            Assert.Equal("3", h.Entries[2].ChapterId);
        }

        [Fact]
        public void HasEntered_TrueIfAnyEntryMatches()
        {
            var h = new ChapterHistory();
            h.RecordEntry("1", 0);
            h.RecordEntry("2", 500);

            Assert.True(h.HasEntered("1"));
            Assert.True(h.HasEntered("2"));
            Assert.False(h.HasEntered("3"));
        }
    }
}
```

- [ ] **Step 2: Run test — expect FAIL (no ChapterHistory type yet)**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ChapterHistoryTests
```

Expected: compile error "ChapterHistory not found".

- [ ] **Step 3: Write `Chapter.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Chapters
{
    /// <summary>
    /// A named narrative chapter. IDs can be numeric strings ("3") or
    /// semantic ("act_interplanetary") — consumer content picks the convention.
    /// </summary>
    public sealed class Chapter
    {
        public string Id;
        public string Name;
        public string Description;
        public string EntryTriggerId;
        public string ExitTriggerId;
    }
}
```

- [ ] **Step 4: Write `ChapterHistory.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Chapters
{
    /// <summary>
    /// Append-only record of chapter entries. Useful for "has this save ever
    /// been in chapter X?" checks, which `currentChapter` alone can't answer
    /// (you might be past X now).
    /// </summary>
    public sealed class ChapterHistory
    {
        public struct Entry
        {
            public string ChapterId;
            public double TimestampSeconds;
        }

        public List<Entry> Entries { get; } = new List<Entry>();

        public void RecordEntry(string chapterId, double timestampSeconds)
        {
            Entries.Add(new Entry { ChapterId = chapterId, TimestampSeconds = timestampSeconds });
        }

        public bool HasEntered(string chapterId)
        {
            foreach (var e in Entries)
                if (e.ChapterId == chapterId) return true;
            return false;
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ChapterHistoryTests
```

Expected: 2 tests pass.

- [ ] **Step 6: Commit**

```bash
git add src/KerbalCampaignKit/Chapters/ tests/KerbalCampaignKit.Tests/ChapterHistoryTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: add Chapter data model and append-only ChapterHistory"
```

---

## Task 7: ChapterLoader (parse CAMPAIGN_CHAPTER cfg)

**Files:**
- Create: `src/KerbalCampaignKit/Chapters/ChapterLoader.cs`
- Create: `tests/KerbalCampaignKit.Tests/ChapterLoaderTests.cs`

- [ ] **Step 1: Write the failing test**

Contents of `tests/KerbalCampaignKit.Tests/ChapterLoaderTests.cs`:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Tests.TestHelpers;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ChapterLoaderTests
    {
        [Fact]
        public void ParsesBasicChapter()
        {
            var node = new FakeConfigNode()
                .Add("id", "3")
                .Add("name", "Beyond the Mun")
                .Add("description", "The anomaly signals stretch past Kerbin's moons.")
                .Add("ENTRY_TRIGGER", "chapter_3_entry")
                .Add("EXIT_TRIGGER", "chapter_3_exit");

            var chapter = ChapterLoader.Load(node);

            Assert.Equal("3", chapter.Id);
            Assert.Equal("Beyond the Mun", chapter.Name);
            Assert.Equal("The anomaly signals stretch past Kerbin's moons.", chapter.Description);
            Assert.Equal("chapter_3_entry", chapter.EntryTriggerId);
            Assert.Equal("chapter_3_exit", chapter.ExitTriggerId);
        }

        [Fact]
        public void AllowsMissingExitTrigger()
        {
            var node = new FakeConfigNode()
                .Add("id", "1")
                .Add("name", "First Steps")
                .Add("ENTRY_TRIGGER", "chapter_1_startup");

            var chapter = ChapterLoader.Load(node);
            Assert.Null(chapter.ExitTriggerId);
        }

        [Fact]
        public void ReturnsNullForMissingId()
        {
            var node = new FakeConfigNode().Add("name", "Nameless");
            var chapter = ChapterLoader.Load(node);
            Assert.Null(chapter);
        }
    }
}
```

- [ ] **Step 2: Write `ChapterLoader.cs`**

Contents:
```csharp
using KerbalCampaignKit.Config;

namespace KerbalCampaignKit.Chapters
{
    /// <summary>
    /// Parses a CAMPAIGN_CHAPTER cfg node into a Chapter. Returns null if
    /// required fields are missing — caller logs and skips.
    /// </summary>
    public static class ChapterLoader
    {
        public static Chapter Load(ISceneNode node)
        {
            if (!node.HasValue("id")) return null;
            var id = node.GetValue("id");
            if (string.IsNullOrEmpty(id)) return null;

            return new Chapter
            {
                Id = id,
                Name = node.HasValue("name") ? node.GetValue("name") : id,
                Description = node.HasValue("description") ? node.GetValue("description") : string.Empty,
                EntryTriggerId = node.HasValue("ENTRY_TRIGGER") ? node.GetValue("ENTRY_TRIGGER") : null,
                ExitTriggerId = node.HasValue("EXIT_TRIGGER") ? node.GetValue("EXIT_TRIGGER") : null,
            };
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 3 PASS**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ChapterLoaderTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Chapters/ChapterLoader.cs tests/KerbalCampaignKit.Tests/ChapterLoaderTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: parse CAMPAIGN_CHAPTER cfg into Chapter"
```

---

## Task 8: ChapterManager (current state + advance/query)

**Files:**
- Create: `src/KerbalCampaignKit/Chapters/ChapterManager.cs`
- Create: `tests/KerbalCampaignKit.Tests/ChapterManagerTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/ChapterManagerTests.cs`:
```csharp
using KerbalCampaignKit.Chapters;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ChapterManagerTests
    {
        [Fact]
        public void InitiallyNoChapter()
        {
            var m = new ChapterManager();
            Assert.Null(m.Current);
        }

        [Fact]
        public void AdvanceSetsCurrentAndRecordsHistory()
        {
            var m = new ChapterManager();
            m.Advance("1", 0);
            m.Advance("2", 500);

            Assert.Equal("2", m.Current);
            Assert.Equal(2, m.History.Entries.Count);
            Assert.True(m.HasEntered("1"));
            Assert.True(m.HasEntered("2"));
        }

        [Theory]
        [InlineData("1", "1", true)]
        [InlineData("2", "1", true)]
        [InlineData("1", "2", false)]
        public void IsAtLeast_NumericComparison(string current, string threshold, bool expected)
        {
            var m = new ChapterManager();
            m.Advance(current, 0);
            Assert.Equal(expected, m.IsAtLeast(threshold));
        }

        [Fact]
        public void IsAtLeast_NonNumeric_ExactMatchOnly()
        {
            var m = new ChapterManager();
            m.Advance("act_interplanetary", 0);
            Assert.True(m.IsAtLeast("act_interplanetary"));
            Assert.False(m.IsAtLeast("act_endgame"));
        }

        [Fact]
        public void FiresCallback_OnAdvance()
        {
            var m = new ChapterManager();
            string fromSeen = "UNSET", toSeen = "UNSET";
            m.OnChapterChanged += (from, to) => { fromSeen = from; toSeen = to; };

            m.Advance("1", 0);
            Assert.Null(fromSeen);
            Assert.Equal("1", toSeen);

            m.Advance("2", 0);
            Assert.Equal("1", fromSeen);
            Assert.Equal("2", toSeen);
        }
    }
}
```

- [ ] **Step 2: Write `ChapterManager.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Chapters
{
    /// <summary>
    /// Tracks the current chapter and history. All mutation goes through
    /// Advance, which fires OnChapterChanged so interested subsystems
    /// (trigger engine, UI renderers) can react.
    /// </summary>
    public sealed class ChapterManager
    {
        public string Current { get; private set; }
        public ChapterHistory History { get; } = new ChapterHistory();

        /// <summary>Args: (fromChapterId, toChapterId). fromChapterId is null on first advance.</summary>
        public event Action<string, string> OnChapterChanged;

        public void Advance(string chapterId, double timestampSeconds)
        {
            if (string.IsNullOrEmpty(chapterId)) return;
            var previous = Current;
            Current = chapterId;
            History.RecordEntry(chapterId, timestampSeconds);
            OnChapterChanged?.Invoke(previous, chapterId);
        }

        public bool HasEntered(string chapterId) => History.HasEntered(chapterId);

        /// <summary>
        /// True if the current chapter is >= threshold. For numeric chapter IDs
        /// uses integer comparison; for non-numeric IDs falls back to exact match.
        /// </summary>
        public bool IsAtLeast(string threshold)
        {
            if (string.IsNullOrEmpty(Current) || string.IsNullOrEmpty(threshold)) return false;

            if (int.TryParse(Current, out var cur) && int.TryParse(threshold, out var thr))
                return cur >= thr;

            return Current == threshold;
        }
    }
}
```

- [ ] **Step 3: Run tests — expect PASS**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ChapterManagerTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Chapters/ChapterManager.cs tests/KerbalCampaignKit.Tests/ChapterManagerTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: ChapterManager with current-state, history, advance callback"
```

---

## Task 9: Notification data + severity + clear rules

**Files:**
- Create: `src/KerbalCampaignKit/Notifications/NotificationSeverity.cs`
- Create: `src/KerbalCampaignKit/Notifications/NotificationClearOn.cs`
- Create: `src/KerbalCampaignKit/Notifications/Notification.cs`

- [ ] **Step 1: Write `NotificationSeverity.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Notifications
{
    public enum NotificationSeverity
    {
        Info,
        Action
    }
}
```

- [ ] **Step 2: Write `NotificationClearOn.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Notifications
{
    /// <summary>
    /// How a notification clears itself. Manual means the content must
    /// explicitly CLEAR_NOTIFICATION; the others fire when a watched event
    /// matches the notification's clear* parameters.
    /// </summary>
    public enum NotificationClearOn
    {
        Manual,
        FacilityEntered,
        SceneEnded,
        FlagSet
    }
}
```

- [ ] **Step 3: Write `Notification.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Notifications
{
    /// <summary>
    /// A single attention marker attached to a hierarchical target string.
    /// </summary>
    public sealed class Notification
    {
        public string Target;
        public NotificationSeverity Severity;
        public string Source;
        public NotificationClearOn ClearOn = NotificationClearOn.Manual;
        public string ClearSceneId;
        public string ClearFlag;
        public string ClearFlagValue;
        public double AddedAtSeconds;
    }
}
```

- [ ] **Step 4: Build — no tests yet (covered by store tests)**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalCampaignKit/Notifications/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: Notification data types with severity and clear-on rules"
```

---

## Task 10: NotificationStore (hierarchical add/clear/query)

**Files:**
- Create: `src/KerbalCampaignKit/Notifications/NotificationStore.cs`
- Create: `tests/KerbalCampaignKit.Tests/NotificationStoreTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/NotificationStoreTests.cs`:
```csharp
using KerbalCampaignKit.Notifications;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class NotificationStoreTests
    {
        private static Notification Make(string target, NotificationSeverity sev, string source)
            => new Notification { Target = target, Severity = sev, Source = source };

        [Fact]
        public void At_ReturnsOnlyExactMatches()
        {
            var store = new NotificationStore();
            store.Add(Make("admin", NotificationSeverity.Action, "x"));
            store.Add(Make("admin.desk", NotificationSeverity.Info, "y"));

            var at = store.At("admin");
            Assert.Single(at);
            Assert.Equal("x", at[0].Source);
        }

        [Fact]
        public void AtOrBelow_ReturnsExactAndDescendants()
        {
            var store = new NotificationStore();
            store.Add(Make("admin", NotificationSeverity.Action, "x"));
            store.Add(Make("admin.desk", NotificationSeverity.Info, "y"));
            store.Add(Make("admin.desk.contracts", NotificationSeverity.Info, "z"));
            store.Add(Make("tracking", NotificationSeverity.Info, "w"));

            var below = store.AtOrBelow("admin");
            Assert.Equal(3, below.Count);
        }

        [Fact]
        public void AtOrBelow_DoesNotMatchPrefixesOfOtherNames()
        {
            // "admin" must not match "administration" — prefix matching is
            // segment-aware (requires dot or end after the prefix).
            var store = new NotificationStore();
            store.Add(Make("administration", NotificationSeverity.Action, "a"));

            Assert.Empty(store.AtOrBelow("admin"));
        }

        [Fact]
        public void Highest_ReturnsActionOverInfo()
        {
            var store = new NotificationStore();
            store.Add(Make("admin", NotificationSeverity.Info, "x"));
            store.Add(Make("admin.desk", NotificationSeverity.Action, "y"));

            Assert.Equal(NotificationSeverity.Action, store.Highest("admin"));
        }

        [Fact]
        public void Highest_NullWhenNoneFound()
        {
            var store = new NotificationStore();
            Assert.Null(store.Highest("admin"));
        }

        [Fact]
        public void Clear_BySourceRemovesOnlyMatch()
        {
            var store = new NotificationStore();
            store.Add(Make("admin", NotificationSeverity.Action, "x"));
            store.Add(Make("admin", NotificationSeverity.Info, "y"));

            store.Clear("admin", "x");
            var remaining = store.At("admin");
            Assert.Single(remaining);
            Assert.Equal("y", remaining[0].Source);
        }

        [Fact]
        public void ClearAll_RemovesDescendants()
        {
            var store = new NotificationStore();
            store.Add(Make("admin", NotificationSeverity.Action, "x"));
            store.Add(Make("admin.desk", NotificationSeverity.Info, "y"));
            store.Add(Make("tracking", NotificationSeverity.Info, "z"));

            store.ClearAll("admin");
            Assert.Empty(store.AtOrBelow("admin"));
            Assert.Single(store.At("tracking"));
        }
    }
}
```

- [ ] **Step 2: Write `NotificationStore.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Notifications
{
    /// <summary>
    /// Holds notifications keyed by hierarchical target. Query methods support
    /// exact match (At) and descendant match (AtOrBelow). Prefix matching is
    /// segment-aware: "admin" matches "admin.desk" but not "administration".
    /// </summary>
    public sealed class NotificationStore
    {
        private readonly List<Notification> items = new List<Notification>();

        public IReadOnlyList<Notification> All => items;

        public void Add(Notification n) => items.Add(n);

        public List<Notification> At(string target)
        {
            var result = new List<Notification>();
            foreach (var n in items)
                if (n.Target == target) result.Add(n);
            return result;
        }

        public List<Notification> AtOrBelow(string target)
        {
            var result = new List<Notification>();
            foreach (var n in items)
                if (IsAtOrBelow(n.Target, target)) result.Add(n);
            return result;
        }

        public NotificationSeverity? Highest(string target)
        {
            var any = false;
            var highest = NotificationSeverity.Info;
            foreach (var n in items)
            {
                if (!IsAtOrBelow(n.Target, target)) continue;
                any = true;
                if (n.Severity == NotificationSeverity.Action) return NotificationSeverity.Action;
            }
            return any ? (NotificationSeverity?)highest : null;
        }

        public void Clear(string target, string source = null)
        {
            items.RemoveAll(n => n.Target == target && (source == null || n.Source == source));
        }

        public void ClearAll(string targetPrefix)
        {
            items.RemoveAll(n => IsAtOrBelow(n.Target, targetPrefix));
        }

        public bool HasAny(string target)
        {
            foreach (var n in items)
                if (IsAtOrBelow(n.Target, target)) return true;
            return false;
        }

        private static bool IsAtOrBelow(string candidate, string ancestor)
        {
            if (candidate == ancestor) return true;
            if (candidate == null || ancestor == null) return false;
            return candidate.StartsWith(ancestor + ".");
        }
    }
}
```

- [ ] **Step 3: Run tests — expect all pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter NotificationStoreTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Notifications/NotificationStore.cs tests/KerbalCampaignKit.Tests/NotificationStoreTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: hierarchical NotificationStore with segment-aware prefix match"
```

---

## Task 11: CampaignKitEvents (GameEvents publisher — stub for now)

**Files:**
- Create: `src/KerbalCampaignKit/Notifications/CampaignKitEvents.cs`

This centralizes the `GameEvents` (or pseudo-events for tests) that CampaignKit publishes. We write a stub first; wire it into `NotificationStore`, `ChapterManager`, and `ReputationEconomy` later via their `On*Changed` callbacks.

- [ ] **Step 1: Write `CampaignKitEvents.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Notifications
{
    /// <summary>
    /// C# events that CampaignKit publishes for subscribers (Director's UI,
    /// building overlays, other mods). In production the CampaignKitAddon
    /// bridges these to KSP GameEvents; in tests subscribers can just wire
    /// these delegates directly.
    /// </summary>
    public static class CampaignKitEvents
    {
        public static event Action<Notification> OnNotificationAdded;
        public static event Action<Notification> OnNotificationCleared;
        /// <summary>Args: (fromChapter, toChapter). fromChapter null on first entry.</summary>
        public static event Action<string, string> OnChapterChanged;
        /// <summary>Args: (amountPaid).</summary>
        public static event Action<double> OnReputationIncomeTick;

        internal static void FireNotificationAdded(Notification n) => OnNotificationAdded?.Invoke(n);
        internal static void FireNotificationCleared(Notification n) => OnNotificationCleared?.Invoke(n);
        internal static void FireChapterChanged(string from, string to) => OnChapterChanged?.Invoke(from, to);
        internal static void FireReputationIncomeTick(double amount) => OnReputationIncomeTick?.Invoke(amount);
    }
}
```

- [ ] **Step 2: Hook `NotificationStore` to fire events on add/clear**

Edit `src/KerbalCampaignKit/Notifications/NotificationStore.cs`:

Replace `Add` with:
```csharp
public void Add(Notification n)
{
    items.Add(n);
    CampaignKitEvents.FireNotificationAdded(n);
}
```

Replace `Clear` with:
```csharp
public void Clear(string target, string source = null)
{
    var removed = items.FindAll(n => n.Target == target && (source == null || n.Source == source));
    items.RemoveAll(n => n.Target == target && (source == null || n.Source == source));
    foreach (var r in removed) CampaignKitEvents.FireNotificationCleared(r);
}
```

Replace `ClearAll` with:
```csharp
public void ClearAll(string targetPrefix)
{
    var removed = items.FindAll(n => IsAtOrBelow(n.Target, targetPrefix));
    items.RemoveAll(n => IsAtOrBelow(n.Target, targetPrefix));
    foreach (var r in removed) CampaignKitEvents.FireNotificationCleared(r);
}
```

- [ ] **Step 3: Add regression test for event firing**

Append to `tests/KerbalCampaignKit.Tests/NotificationStoreTests.cs`:
```csharp
[Fact]
public void FiresAddedEvent()
{
    Notification seen = null;
    CampaignKitEvents.OnNotificationAdded += n => seen = n;
    try
    {
        var store = new NotificationStore();
        var note = Make("admin", NotificationSeverity.Info, "x");
        store.Add(note);
        Assert.Same(note, seen);
    }
    finally
    {
        CampaignKitEvents.OnNotificationAdded -= n => seen = n;
    }
}
```

- [ ] **Step 4: Run tests + build**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter NotificationStoreTests
```

Expected: all pass.

- [ ] **Step 5: Hook `ChapterManager` to fire chapter-changed event**

Edit `src/KerbalCampaignKit/Chapters/ChapterManager.cs`:

Add `using KerbalCampaignKit.Notifications;` at top.

Replace `Advance` body to append a CampaignKitEvents fire:
```csharp
public void Advance(string chapterId, double timestampSeconds)
{
    if (string.IsNullOrEmpty(chapterId)) return;
    var previous = Current;
    Current = chapterId;
    History.RecordEntry(chapterId, timestampSeconds);
    OnChapterChanged?.Invoke(previous, chapterId);
    CampaignKitEvents.FireChapterChanged(previous, chapterId);
}
```

- [ ] **Step 6: Run all tests + commit**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
git add src/KerbalCampaignKit/Notifications/CampaignKitEvents.cs src/KerbalCampaignKit/Notifications/NotificationStore.cs src/KerbalCampaignKit/Chapters/ChapterManager.cs tests/KerbalCampaignKit.Tests/NotificationStoreTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: CampaignKitEvents bus + hooks on store and chapter manager"
```

---

## Task 12: EventType enum + EventRecord + IEventSource skeleton

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Events/EventType.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/EventRecord.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/IEventSource.cs`

- [ ] **Step 1: Write `EventType.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// All event types CampaignKit triggers can listen to. Each IEventSource
    /// publishes events tagged with one of these values plus a parameter bag
    /// (EventRecord.Params).
    /// </summary>
    public enum EventType
    {
        ContractComplete,
        ContractAccepted,
        ContractFailed,
        ContractCancelled,
        VesselRecovered,
        VesselCrashed,
        KerbalKilled,
        KerbalLeveled,
        BodyDiscovered,
        FlagChanged,
        ReputationCrossed,
        FundsCrossed,
        ScienceCrossed,
        FacilityUpgraded,
        FacilityEntered,
        ChapterEntered,
        ChapterExited,
        TimeElapsed,
        SceneEnded,
        SceneChoiceMade,
        GameEvent
    }
}
```

- [ ] **Step 2: Write `EventRecord.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// A concrete event fired by an IEventSource. Type is required; Params
    /// holds string-keyed payload (e.g. {"contract":"BKEX_ProbeFlyby_Duna"}).
    /// Triggers match by Type + exact-or-any param equality.
    /// </summary>
    public sealed class EventRecord
    {
        public EventType Type;
        public Dictionary<string, string> Params = new Dictionary<string, string>();

        public string Get(string key)
        {
            return Params.TryGetValue(key, out var v) ? v : null;
        }
    }
}
```

- [ ] **Step 3: Write `IEventSource.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Abstraction for something that publishes events. The TriggerEngine
    /// subscribes with Register and receives EventRecord instances via the
    /// delegate. Each concrete source wraps a KSP GameEvent or KDK callback.
    /// </summary>
    public interface IEventSource
    {
        void Register(Action<EventRecord> onEvent);
        void Unregister();
    }
}
```

- [ ] **Step 4: Build**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

- [ ] **Step 5: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: event-source abstraction (EventType, EventRecord, IEventSource)"
```

---

## Task 13: Trigger data model (EventSpec + ActionSpec + Trigger)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/EventSpec.cs`
- Create: `src/KerbalCampaignKit/Triggers/ActionSpec.cs`
- Create: `src/KerbalCampaignKit/Triggers/Trigger.cs`

- [ ] **Step 1: Write `EventSpec.cs`**

Contents:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Triggers.Events;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// Parsed ON_EVENT block. Matches an EventRecord if Type matches and each
    /// specified parameter matches the record's param exactly. Missing params
    /// on the spec mean "don't care" for that param (matches anything).
    /// </summary>
    public sealed class EventSpec
    {
        public EventType Type;
        public Dictionary<string, string> ParamMatch = new Dictionary<string, string>();

        public bool Matches(EventRecord record)
        {
            if (record.Type != Type) return false;
            foreach (var kvp in ParamMatch)
            {
                if (!record.Params.TryGetValue(kvp.Key, out var actual)) return false;
                if (actual != kvp.Value) return false;
            }
            return true;
        }
    }
}
```

- [ ] **Step 2: Write `ActionSpec.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// Raw parsed action entry. Kind is the action type name (e.g. "SET_FLAG");
    /// Params holds all key-value fields. The TriggerEngine maps Kind to a
    /// registered IAction by lookup.
    /// </summary>
    public sealed class ActionSpec
    {
        public string Kind;
        public Dictionary<string, string> Params = new Dictionary<string, string>();

        public string Get(string key) =>
            Params.TryGetValue(key, out var v) ? v : null;
    }
}
```

- [ ] **Step 3: Write `Trigger.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// A parsed CAMPAIGN_TRIGGER. Events OR together (any event source can
    /// fire); the When expression gates whether actions execute; actions
    /// execute in declaration order.
    /// </summary>
    public sealed class Trigger
    {
        public string Id;
        public bool Once = true;
        public List<EventSpec> Events = new List<EventSpec>();
        public string WhenExpression;    // raw; parsed at engine load
        public List<ActionSpec> Actions = new List<ActionSpec>();
    }
}
```

- [ ] **Step 4: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Triggers/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: Trigger data model (EventSpec, ActionSpec, Trigger)"
```

---

## Task 14: TriggerLoader (parse CAMPAIGN_TRIGGER cfg)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/TriggerLoader.cs`
- Create: `tests/KerbalCampaignKit.Tests/TriggerLoaderTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/TriggerLoaderTests.cs`:
```csharp
using KerbalCampaignKit.Tests.TestHelpers;
using KerbalCampaignKit.Triggers;
using KerbalCampaignKit.Triggers.Events;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class TriggerLoaderTests
    {
        [Fact]
        public void ParsesIdAndOnceFlag()
        {
            var node = new FakeConfigNode()
                .Add("id", "chapter_4_on_duna")
                .Add("once", "false");

            var trigger = TriggerLoader.Load(node);
            Assert.Equal("chapter_4_on_duna", trigger.Id);
            Assert.False(trigger.Once);
        }

        [Fact]
        public void OnceDefaultsTrue()
        {
            var node = new FakeConfigNode().Add("id", "t1");
            Assert.True(TriggerLoader.Load(node).Once);
        }

        [Fact]
        public void ParsesEvent_WithParams()
        {
            var eventNode = new FakeConfigNode()
                .Add("type", "ContractComplete")
                .Add("contract", "BKEX_ProbeFlyby_Duna");
            var node = new FakeConfigNode()
                .Add("id", "t1")
                .AddNode("ON_EVENT", eventNode);

            var t = TriggerLoader.Load(node);
            Assert.Single(t.Events);
            Assert.Equal(EventType.ContractComplete, t.Events[0].Type);
            Assert.Equal("BKEX_ProbeFlyby_Duna", t.Events[0].ParamMatch["contract"]);
        }

        [Fact]
        public void ParsesWhenExpressionFromNestedNode()
        {
            var when = new FakeConfigNode().Add("flagExpression", "chapter == 3");
            var node = new FakeConfigNode()
                .Add("id", "t1")
                .AddNode("WHEN", when);

            var t = TriggerLoader.Load(node);
            Assert.Equal("chapter == 3", t.WhenExpression);
        }

        [Fact]
        public void ParsesActions_InOrder()
        {
            var setFlag = new FakeConfigNode()
                .Add("name", "foo")
                .Add("value", "bar");
            var advance = new FakeConfigNode().Add("target", "4");
            var actions = new FakeConfigNode()
                .AddNode("SET_FLAG", setFlag)
                .AddNode("ADVANCE_CHAPTER", advance);
            var node = new FakeConfigNode()
                .Add("id", "t1")
                .AddNode("ACTIONS", actions);

            var t = TriggerLoader.Load(node);
            Assert.Equal(2, t.Actions.Count);
            Assert.Equal("SET_FLAG", t.Actions[0].Kind);
            Assert.Equal("foo", t.Actions[0].Params["name"]);
            Assert.Equal("ADVANCE_CHAPTER", t.Actions[1].Kind);
            Assert.Equal("4", t.Actions[1].Params["target"]);
        }

        [Fact]
        public void ReturnsNull_IfIdMissing()
        {
            var node = new FakeConfigNode();
            Assert.Null(TriggerLoader.Load(node));
        }
    }
}
```

- [ ] **Step 2: Write `TriggerLoader.cs`**

Contents:
```csharp
using System;
using System.Collections.Generic;
using KerbalCampaignKit.Config;
using KerbalCampaignKit.Triggers.Events;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// Parses a CAMPAIGN_TRIGGER ISceneNode into a Trigger instance.
    /// Returns null if required fields are missing.
    /// </summary>
    public static class TriggerLoader
    {
        public static Trigger Load(ISceneNode node)
        {
            if (!node.HasValue("id")) return null;

            var trigger = new Trigger
            {
                Id = node.GetValue("id"),
                Once = !node.HasValue("once") || node.GetValue("once") != "false",
            };

            foreach (var eventNode in node.GetNodes("ON_EVENT"))
            {
                var typeText = eventNode.GetValue("type");
                if (string.IsNullOrEmpty(typeText)) continue;
                if (!Enum.TryParse<EventType>(typeText, out var evType)) continue;
                var spec = new EventSpec { Type = evType };
                // Collect all non-"type" values as match params.
                // FakeConfigNode and ConfigNodeAdapter both expose keys via HasValue/GetValue,
                // so we walk known param keys explicitly rather than enumerating.
                foreach (var key in KnownEventParams)
                {
                    if (eventNode.HasValue(key))
                        spec.ParamMatch[key] = eventNode.GetValue(key);
                }
                trigger.Events.Add(spec);
            }

            if (node.HasNode("WHEN"))
            {
                foreach (var when in node.GetNodes("WHEN"))
                {
                    if (when.HasValue("flagExpression"))
                    {
                        trigger.WhenExpression = when.GetValue("flagExpression");
                        break;
                    }
                }
            }

            if (node.HasNode("ACTIONS"))
            {
                foreach (var actions in node.GetNodes("ACTIONS"))
                {
                    foreach (var actionKind in KnownActionKinds)
                    {
                        foreach (var actionNode in actions.GetNodes(actionKind))
                        {
                            var spec = new ActionSpec { Kind = actionKind };
                            foreach (var key in KnownActionParams)
                            {
                                if (actionNode.HasValue(key))
                                    spec.Params[key] = actionNode.GetValue(key);
                            }
                            trigger.Actions.Add(spec);
                        }
                    }
                    break;  // only first ACTIONS block
                }
            }

            return trigger;
        }

        private static readonly string[] KnownEventParams = {
            "contract", "vesselType", "kerbalName", "trait", "level", "body",
            "name", "value", "threshold", "direction", "facility", "chapter",
            "days", "ref", "sceneId", "choiceId", "eventName"
        };

        private static readonly string[] KnownActionKinds = {
            "SET_FLAG", "CLEAR_FLAG", "ENQUEUE_SCENE", "ADVANCE_CHAPTER",
            "ADJUST_FUNDS", "ADJUST_REPUTATION", "ADJUST_SCIENCE",
            "NOTIFY", "CLEAR_NOTIFICATION", "PUBLISH_EVENT"
        };

        private static readonly string[] KnownActionParams = {
            "name", "value", "sceneId", "when", "facility", "target",
            "amount", "reason", "severity", "source", "clearOn",
            "clearSceneId", "clearFlag", "clearFlagValue", "eventName"
        };
    }
}
```

- [ ] **Step 3: Run tests — expect 6 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter TriggerLoaderTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/TriggerLoader.cs tests/KerbalCampaignKit.Tests/TriggerLoaderTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: parse CAMPAIGN_TRIGGER cfg into Trigger instance"
```

---

## Task 15: RequirementExpression (wrap KDK parser)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/RequirementExpression.cs`
- Create: `tests/KerbalCampaignKit.Tests/RequirementExpressionTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/RequirementExpressionTests.cs`:
```csharp
using KerbalDialogueKit.Flags;
using KerbalCampaignKit.Triggers;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class RequirementExpressionTests
    {
        [Fact]
        public void NullExpression_EvaluatesTrue()
        {
            var flags = new FlagStore();
            Assert.True(RequirementExpression.Evaluate(null, flags));
            Assert.True(RequirementExpression.Evaluate("", flags));
        }

        [Fact]
        public void SimpleEquality()
        {
            var flags = new FlagStore();
            flags.Set("chapter", "3");
            Assert.True(RequirementExpression.Evaluate("chapter == 3", flags));
            Assert.False(RequirementExpression.Evaluate("chapter == 4", flags));
        }

        [Fact]
        public void AndOr()
        {
            var flags = new FlagStore();
            flags.Set("chapter", "3");
            flags.Set("mood", "happy");
            Assert.True(RequirementExpression.Evaluate(
                "chapter == 3 && mood == happy", flags));
            Assert.False(RequirementExpression.Evaluate(
                "chapter == 3 && mood == sad", flags));
            Assert.True(RequirementExpression.Evaluate(
                "chapter == 99 || mood == happy", flags));
        }

        [Fact]
        public void InvalidExpression_EvaluatesFalse_NoCrash()
        {
            var flags = new FlagStore();
            Assert.False(RequirementExpression.Evaluate("not a real expression", flags));
        }
    }
}
```

- [ ] **Step 2: Write `RequirementExpression.cs`**

Contents:
```csharp
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// Thin wrapper over KDK's FlagExpressionParser so CampaignKit callers
    /// don't have to import KDK types directly. Null/empty expressions
    /// evaluate to true (vacuously satisfied). Invalid expressions evaluate
    /// to false to avoid crashes on bad content — KDK logs the parse error.
    /// </summary>
    public static class RequirementExpression
    {
        public static bool Evaluate(string expression, FlagStore flags)
        {
            if (string.IsNullOrEmpty(expression)) return true;

            FlagExpression parsed;
            try
            {
                parsed = FlagExpressionParser.Parse(expression);
            }
            catch
            {
                return false;
            }
            if (parsed == null) return false;
            return parsed.Evaluate(flags);
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 4 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter RequirementExpressionTests
```

Note: this depends on `KerbalDialogueKit.Flags.FlagExpressionParser` being accessible. If the KDK parser's entry point is named differently (e.g., `Parse` vs `ParseExpression`), adjust to match KDK's actual API — check `GameData/KerbalDialogueKit/src/KerbalDialogueKit/Flags/FlagExpressionParser.cs`.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/RequirementExpression.cs tests/KerbalCampaignKit.Tests/RequirementExpressionTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: RequirementExpression wraps KDK parser with safe defaults"
```

---

## Task 16: IAction interface + ActionContext

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Actions/IAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/ActionContext.cs`

`ActionContext` is the ambient state an action needs to do its work (flags, chapter manager, notification store, currency adapter, pending-scene queue). We inject it so actions stay testable without KSP.

- [ ] **Step 1: Write `ActionContext.cs`**

Contents:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.PendingScenes;
using KerbalCampaignKit.Reputation;
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Triggers.Actions
{
    /// <summary>
    /// Ambient state injected into action execution. In production the
    /// CampaignKitAddon constructs one of these from live systems; in tests
    /// it's constructed from in-memory fakes.
    /// </summary>
    public sealed class ActionContext
    {
        public FlagStore Flags;
        public ChapterManager Chapters;
        public NotificationStore Notifications;
        public PendingSceneQueue PendingScenes;
        public ICurrencyAdapter Currencies;
        public ISceneEnqueuer SceneEnqueuer;
        public double NowSeconds;
    }

    /// <summary>Abstracts currency mutation so tests don't need KSP Funding/etc.</summary>
    public interface ICurrencyAdapter
    {
        void AddFunds(double amount);
        void AddReputation(double amount);
        void AddScience(double amount);
        double Funds { get; }
        double Reputation { get; }
        double Science { get; }
    }

    /// <summary>Abstracts KDK's DialogueKit.Enqueue so tests can verify calls.</summary>
    public interface ISceneEnqueuer
    {
        bool EnqueueById(string sceneId);
    }
}
```

- [ ] **Step 2: Write `IAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    /// <summary>
    /// An action that fires when a trigger's conditions are met. Stateless;
    /// all state lives in ActionContext.
    /// </summary>
    public interface IAction
    {
        /// <summary>The cfg-level kind name this action handles (e.g. "SET_FLAG").</summary>
        string Kind { get; }

        /// <summary>Execute against the given spec and context.</summary>
        void Execute(ActionSpec spec, ActionContext ctx);
    }
}
```

- [ ] **Step 3: Build (no tests yet — each concrete action has its own tests)**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: one compile error — references to `PendingScenes` types not yet created. Fix by creating the stubs:

Create `src/KerbalCampaignKit/PendingScenes/PendingScene.cs`:
```csharp
namespace KerbalCampaignKit.PendingScenes
{
    public sealed class PendingScene
    {
        public string SceneId;
        public string Facility;        // null = immediate
        public string FromTriggerId;
    }
}
```

Create `src/KerbalCampaignKit/PendingScenes/PendingSceneQueue.cs`:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.PendingScenes
{
    public sealed class PendingSceneQueue
    {
        private readonly List<PendingScene> items = new List<PendingScene>();

        public IReadOnlyList<PendingScene> Pending => items;

        public void Add(PendingScene scene) => items.Add(scene);

        public List<PendingScene> TakeForFacility(string facility)
        {
            var matched = new List<PendingScene>();
            foreach (var s in items)
                if (s.Facility == facility) matched.Add(s);
            items.RemoveAll(s => s.Facility == facility);
            return matched;
        }

        public List<PendingScene> TakeImmediate()
        {
            var matched = new List<PendingScene>();
            foreach (var s in items)
                if (string.IsNullOrEmpty(s.Facility)) matched.Add(s);
            items.RemoveAll(s => string.IsNullOrEmpty(s.Facility));
            return matched;
        }

        public void Clear() => items.Clear();
    }
}
```

- [ ] **Step 4: Build (now clean)**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

- [ ] **Step 5: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Actions/ src/KerbalCampaignKit/PendingScenes/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: IAction interface, ActionContext, PendingSceneQueue skeleton"
```

---

## Task 17: Flag + chapter + currency actions (TDD)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Actions/SetFlagAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/ClearFlagAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/AdvanceChapterAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/AdjustCurrencyAction.cs`
- Create: `tests/KerbalCampaignKit.Tests/ActionTests.cs`
- Create: `tests/KerbalCampaignKit.Tests/TestHelpers/FakeCurrencyAdapter.cs`

- [ ] **Step 1: Write `FakeCurrencyAdapter.cs`**

Contents:
```csharp
using KerbalCampaignKit.Triggers.Actions;

namespace KerbalCampaignKit.Tests.TestHelpers
{
    public sealed class FakeCurrencyAdapter : ICurrencyAdapter
    {
        public double Funds { get; private set; }
        public double Reputation { get; private set; }
        public double Science { get; private set; }

        public void AddFunds(double amount) { Funds += amount; }
        public void AddReputation(double amount) { Reputation += amount; }
        public void AddScience(double amount) { Science += amount; }
    }
}
```

- [ ] **Step 2: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/ActionTests.cs`:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.PendingScenes;
using KerbalCampaignKit.Tests.TestHelpers;
using KerbalCampaignKit.Triggers;
using KerbalCampaignKit.Triggers.Actions;
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ActionTests
    {
        private static ActionContext NewCtx()
        {
            return new ActionContext
            {
                Flags = new FlagStore(),
                Chapters = new ChapterManager(),
                Notifications = new NotificationStore(),
                PendingScenes = new PendingSceneQueue(),
                Currencies = new FakeCurrencyAdapter(),
                NowSeconds = 1000
            };
        }

        [Fact]
        public void SetFlag_WritesToFlagStore()
        {
            var ctx = NewCtx();
            var spec = new ActionSpec { Kind = "SET_FLAG" };
            spec.Params["name"] = "mood";
            spec.Params["value"] = "happy";

            new SetFlagAction().Execute(spec, ctx);
            Assert.Equal("happy", ctx.Flags.Get("mood"));
        }

        [Fact]
        public void ClearFlag_Removes()
        {
            var ctx = NewCtx();
            ctx.Flags.Set("mood", "happy");
            var spec = new ActionSpec { Kind = "CLEAR_FLAG" };
            spec.Params["name"] = "mood";

            new ClearFlagAction().Execute(spec, ctx);
            Assert.Null(ctx.Flags.Get("mood"));
        }

        [Fact]
        public void AdvanceChapter_UpdatesManager()
        {
            var ctx = NewCtx();
            var spec = new ActionSpec { Kind = "ADVANCE_CHAPTER" };
            spec.Params["target"] = "3";

            new AdvanceChapterAction().Execute(spec, ctx);
            Assert.Equal("3", ctx.Chapters.Current);
        }

        [Fact]
        public void AdjustFunds_DelegatesToCurrencyAdapter()
        {
            var ctx = NewCtx();
            var spec = new ActionSpec { Kind = "ADJUST_FUNDS" };
            spec.Params["amount"] = "5000";

            new AdjustCurrencyAction("ADJUST_FUNDS").Execute(spec, ctx);
            Assert.Equal(5000, ((FakeCurrencyAdapter)ctx.Currencies).Funds);
        }

        [Fact]
        public void AdjustReputation_DelegatesToCurrencyAdapter()
        {
            var ctx = NewCtx();
            var spec = new ActionSpec { Kind = "ADJUST_REPUTATION" };
            spec.Params["amount"] = "-25";

            new AdjustCurrencyAction("ADJUST_REPUTATION").Execute(spec, ctx);
            Assert.Equal(-25, ((FakeCurrencyAdapter)ctx.Currencies).Reputation);
        }
    }
}
```

- [ ] **Step 3: Write `SetFlagAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    public sealed class SetFlagAction : IAction
    {
        public string Kind => "SET_FLAG";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var name = spec.Get("name");
            var value = spec.Get("value");
            if (string.IsNullOrEmpty(name)) return;
            ctx.Flags.Set(name, value ?? string.Empty);
        }
    }
}
```

- [ ] **Step 4: Write `ClearFlagAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    public sealed class ClearFlagAction : IAction
    {
        public string Kind => "CLEAR_FLAG";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var name = spec.Get("name");
            if (string.IsNullOrEmpty(name)) return;
            ctx.Flags.Remove(name);
        }
    }
}
```

- [ ] **Step 5: Write `AdvanceChapterAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    public sealed class AdvanceChapterAction : IAction
    {
        public string Kind => "ADVANCE_CHAPTER";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var target = spec.Get("target");
            if (string.IsNullOrEmpty(target)) return;
            ctx.Chapters.Advance(target, ctx.NowSeconds);
        }
    }
}
```

- [ ] **Step 6: Write `AdjustCurrencyAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    /// <summary>
    /// Handles ADJUST_FUNDS, ADJUST_REPUTATION, ADJUST_SCIENCE. A single
    /// class with three registrations, because the logic is identical
    /// aside from which adapter method to call.
    /// </summary>
    public sealed class AdjustCurrencyAction : IAction
    {
        private readonly string kind;

        public AdjustCurrencyAction(string kind) { this.kind = kind; }
        public string Kind => kind;

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var amountText = spec.Get("amount");
            if (string.IsNullOrEmpty(amountText)) return;
            if (!double.TryParse(amountText, out var amount)) return;

            switch (kind)
            {
                case "ADJUST_FUNDS": ctx.Currencies.AddFunds(amount); break;
                case "ADJUST_REPUTATION": ctx.Currencies.AddReputation(amount); break;
                case "ADJUST_SCIENCE": ctx.Currencies.AddScience(amount); break;
            }
        }
    }
}
```

- [ ] **Step 7: Run tests — expect 5 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ActionTests
```

- [ ] **Step 8: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Actions/ tests/KerbalCampaignKit.Tests/ActionTests.cs tests/KerbalCampaignKit.Tests/TestHelpers/FakeCurrencyAdapter.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: SET_FLAG, CLEAR_FLAG, ADVANCE_CHAPTER, ADJUST_* actions"
```

---

## Task 18: EnqueueSceneAction + PendingSceneQueue tests

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Actions/EnqueueSceneAction.cs`
- Create: `tests/KerbalCampaignKit.Tests/PendingSceneQueueTests.cs`
- Create: `tests/KerbalCampaignKit.Tests/TestHelpers/FakeSceneEnqueuer.cs`
- Modify: `tests/KerbalCampaignKit.Tests/ActionTests.cs` (add enqueue tests)

- [ ] **Step 1: Write `FakeSceneEnqueuer.cs`**

Contents:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Triggers.Actions;

namespace KerbalCampaignKit.Tests.TestHelpers
{
    public sealed class FakeSceneEnqueuer : ISceneEnqueuer
    {
        public List<string> Enqueued { get; } = new List<string>();

        public bool EnqueueById(string sceneId)
        {
            Enqueued.Add(sceneId);
            return true;
        }
    }
}
```

- [ ] **Step 2: Write the failing tests (append to ActionTests.cs)**

Append to `tests/KerbalCampaignKit.Tests/ActionTests.cs`:
```csharp
        [Fact]
        public void EnqueueScene_Immediate_CallsEnqueuer()
        {
            var ctx = NewCtx();
            var fake = new FakeSceneEnqueuer();
            ctx.SceneEnqueuer = fake;
            var spec = new ActionSpec { Kind = "ENQUEUE_SCENE" };
            spec.Params["sceneId"] = "my_scene";

            new EnqueueSceneAction().Execute(spec, ctx);
            Assert.Contains("my_scene", fake.Enqueued);
            Assert.Empty(ctx.PendingScenes.Pending);
        }

        [Fact]
        public void EnqueueScene_OnFacilityEnter_QueuesNotImmediate()
        {
            var ctx = NewCtx();
            var fake = new FakeSceneEnqueuer();
            ctx.SceneEnqueuer = fake;
            var spec = new ActionSpec { Kind = "ENQUEUE_SCENE" };
            spec.Params["sceneId"] = "my_scene";
            spec.Params["when"] = "OnFacilityEnter";
            spec.Params["facility"] = "Administration";

            new EnqueueSceneAction().Execute(spec, ctx);
            Assert.Empty(fake.Enqueued);
            Assert.Single(ctx.PendingScenes.Pending);
            Assert.Equal("Administration", ctx.PendingScenes.Pending[0].Facility);
        }
```

Add `using KerbalCampaignKit.Tests.TestHelpers;` if not present.

- [ ] **Step 3: Write `EnqueueSceneAction.cs`**

Contents:
```csharp
using KerbalCampaignKit.PendingScenes;

namespace KerbalCampaignKit.Triggers.Actions
{
    /// <summary>
    /// Either enqueues a scene immediately via the ISceneEnqueuer (KDK),
    /// or queues a PendingScene to be enqueued when the player next enters
    /// the target facility.
    /// </summary>
    public sealed class EnqueueSceneAction : IAction
    {
        public string Kind => "ENQUEUE_SCENE";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var sceneId = spec.Get("sceneId");
            if (string.IsNullOrEmpty(sceneId)) return;

            var when = spec.Get("when") ?? "Immediate";
            if (when == "OnFacilityEnter")
            {
                var facility = spec.Get("facility");
                if (string.IsNullOrEmpty(facility)) return;
                ctx.PendingScenes.Add(new PendingScene
                {
                    SceneId = sceneId,
                    Facility = facility
                });
            }
            else
            {
                ctx.SceneEnqueuer?.EnqueueById(sceneId);
            }
        }
    }
}
```

- [ ] **Step 4: Write `PendingSceneQueueTests.cs`**

Contents:
```csharp
using KerbalCampaignKit.PendingScenes;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class PendingSceneQueueTests
    {
        [Fact]
        public void TakeForFacility_ReturnsAndRemovesMatching()
        {
            var q = new PendingSceneQueue();
            q.Add(new PendingScene { SceneId = "a", Facility = "Administration" });
            q.Add(new PendingScene { SceneId = "b", Facility = "MissionControl" });
            q.Add(new PendingScene { SceneId = "c", Facility = "Administration" });

            var taken = q.TakeForFacility("Administration");
            Assert.Equal(2, taken.Count);
            Assert.Single(q.Pending);
            Assert.Equal("MissionControl", q.Pending[0].Facility);
        }

        [Fact]
        public void TakeImmediate_TakesNullFacility()
        {
            var q = new PendingSceneQueue();
            q.Add(new PendingScene { SceneId = "a", Facility = null });
            q.Add(new PendingScene { SceneId = "b", Facility = "Administration" });

            var taken = q.TakeImmediate();
            Assert.Single(taken);
            Assert.Equal("a", taken[0].SceneId);
        }
    }
}
```

- [ ] **Step 5: Run tests — expect all pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
```

- [ ] **Step 6: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Actions/EnqueueSceneAction.cs tests/KerbalCampaignKit.Tests/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: ENQUEUE_SCENE action with pending queue for facility-gated scenes"
```

---

## Task 19: Notify + ClearNotification + PublishEvent actions

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Actions/NotifyAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/ClearNotificationAction.cs`
- Create: `src/KerbalCampaignKit/Triggers/Actions/PublishEventAction.cs`
- Modify: `tests/KerbalCampaignKit.Tests/ActionTests.cs` (add notification tests)

- [ ] **Step 1: Append notification tests to ActionTests.cs**

```csharp
        [Fact]
        public void Notify_AddsToStoreWithSeverity()
        {
            var ctx = NewCtx();
            var spec = new ActionSpec { Kind = "NOTIFY" };
            spec.Params["target"] = "admin";
            spec.Params["severity"] = "Action";
            spec.Params["source"] = "chapter_4_directive";

            new NotifyAction().Execute(spec, ctx);
            var found = ctx.Notifications.At("admin");
            Assert.Single(found);
            Assert.Equal(Notifications.NotificationSeverity.Action, found[0].Severity);
            Assert.Equal("chapter_4_directive", found[0].Source);
        }

        [Fact]
        public void ClearNotification_RemovesMatchingSource()
        {
            var ctx = NewCtx();
            ctx.Notifications.Add(new Notifications.Notification
            {
                Target = "admin", Severity = Notifications.NotificationSeverity.Action, Source = "x"
            });
            var spec = new ActionSpec { Kind = "CLEAR_NOTIFICATION" };
            spec.Params["target"] = "admin";
            spec.Params["source"] = "x";

            new ClearNotificationAction().Execute(spec, ctx);
            Assert.Empty(ctx.Notifications.At("admin"));
        }
```

Add `using KerbalCampaignKit.Notifications;` to the test file if not present. Note it's already imported as `Notifications.Notification` in the test helpers, so qualify as shown.

- [ ] **Step 2: Write `NotifyAction.cs`**

Contents:
```csharp
using System;
using KerbalCampaignKit.Notifications;

namespace KerbalCampaignKit.Triggers.Actions
{
    public sealed class NotifyAction : IAction
    {
        public string Kind => "NOTIFY";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var target = spec.Get("target");
            if (string.IsNullOrEmpty(target)) return;

            var sevText = spec.Get("severity") ?? "Info";
            if (!Enum.TryParse<NotificationSeverity>(sevText, out var severity))
                severity = NotificationSeverity.Info;

            var clearOnText = spec.Get("clearOn") ?? "Manual";
            if (!Enum.TryParse<NotificationClearOn>(clearOnText, out var clearOn))
                clearOn = NotificationClearOn.Manual;

            ctx.Notifications.Add(new Notification
            {
                Target = target,
                Severity = severity,
                Source = spec.Get("source") ?? string.Empty,
                ClearOn = clearOn,
                ClearSceneId = spec.Get("clearSceneId"),
                ClearFlag = spec.Get("clearFlag"),
                ClearFlagValue = spec.Get("clearFlagValue"),
                AddedAtSeconds = ctx.NowSeconds,
            });
        }
    }
}
```

- [ ] **Step 3: Write `ClearNotificationAction.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Triggers.Actions
{
    public sealed class ClearNotificationAction : IAction
    {
        public string Kind => "CLEAR_NOTIFICATION";

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var target = spec.Get("target");
            if (string.IsNullOrEmpty(target)) return;
            ctx.Notifications.Clear(target, spec.Get("source"));
        }
    }
}
```

- [ ] **Step 4: Write `PublishEventAction.cs`**

Contents:
```csharp
using System.Collections.Generic;
using UnityEngine;

namespace KerbalCampaignKit.Triggers.Actions
{
    /// <summary>
    /// Publishes a named GameEvent so other mods (music switchers, UI) can
    /// subscribe. Params besides "eventName" are passed in the event payload
    /// as a dictionary; listeners decide how to interpret them.
    /// </summary>
    public sealed class PublishEventAction : IAction
    {
        public string Kind => "PUBLISH_EVENT";

        // Custom GameEvents registered on first use; keyed by name.
        private static readonly Dictionary<string, EventData<Dictionary<string, string>>> registered
            = new Dictionary<string, EventData<Dictionary<string, string>>>();

        public void Execute(ActionSpec spec, ActionContext ctx)
        {
            var name = spec.Get("eventName") ?? spec.Get("name");
            if (string.IsNullOrEmpty(name)) return;

            if (!registered.TryGetValue(name, out var ev))
            {
                ev = new EventData<Dictionary<string, string>>(name);
                registered[name] = ev;
            }

            var payload = new Dictionary<string, string>();
            foreach (var kvp in spec.Params)
                if (kvp.Key != "eventName" && kvp.Key != "name")
                    payload[kvp.Key] = kvp.Value;

            ev.Fire(payload);
        }
    }
}
```

Note: this file references `UnityEngine` so it won't be covered by pure-logic unit tests. It compiles under the main assembly (which references Unity) but we skip testing it here — manual verification only.

- [ ] **Step 5: Build, run tests**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj
```

Expected: clean build, all tests pass (including the two new notification tests).

- [ ] **Step 6: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Actions/ tests/KerbalCampaignKit.Tests/ActionTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: NOTIFY, CLEAR_NOTIFICATION, PUBLISH_EVENT actions"
```

---

## Task 20: TriggerEngine (dispatch + dedup + action registry)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/TriggerEngine.cs`
- Create: `tests/KerbalCampaignKit.Tests/TriggerEngineTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/TriggerEngineTests.cs`:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.PendingScenes;
using KerbalCampaignKit.Tests.TestHelpers;
using KerbalCampaignKit.Triggers;
using KerbalCampaignKit.Triggers.Actions;
using KerbalCampaignKit.Triggers.Events;
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class TriggerEngineTests
    {
        private static (TriggerEngine engine, ActionContext ctx) NewEngine()
        {
            var ctx = new ActionContext
            {
                Flags = new FlagStore(),
                Chapters = new ChapterManager(),
                Notifications = new NotificationStore(),
                PendingScenes = new PendingSceneQueue(),
                Currencies = new FakeCurrencyAdapter(),
                NowSeconds = 0
            };
            var engine = new TriggerEngine(ctx);
            engine.RegisterAction(new SetFlagAction());
            engine.RegisterAction(new AdvanceChapterAction());
            return (engine, ctx);
        }

        private static Trigger TriggerMatchingContract(string id, string contractName)
        {
            var t = new Trigger { Id = id };
            var ev = new EventSpec { Type = EventType.ContractComplete };
            ev.ParamMatch["contract"] = contractName;
            t.Events.Add(ev);
            return t;
        }

        [Fact]
        public void Dispatch_FiresMatchingTrigger()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "BKEX_FlybyDuna");
            trigger.Actions.Add(new ActionSpec { Kind = "SET_FLAG" });
            trigger.Actions[0].Params["name"] = "fired";
            trigger.Actions[0].Params["value"] = "true";
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "BKEX_FlybyDuna";
            engine.Dispatch(record);

            Assert.Equal("true", ctx.Flags.Get("fired"));
        }

        [Fact]
        public void Dispatch_SkipsNonMatchingTriggers()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "BKEX_FlybyDuna");
            trigger.Actions.Add(new ActionSpec { Kind = "SET_FLAG" });
            trigger.Actions[0].Params["name"] = "fired";
            trigger.Actions[0].Params["value"] = "true";
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "DifferentContract";
            engine.Dispatch(record);

            Assert.Null(ctx.Flags.Get("fired"));
        }

        [Fact]
        public void Dispatch_HonorsWhenExpression()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "X");
            trigger.WhenExpression = "chapter == 3";
            trigger.Actions.Add(new ActionSpec { Kind = "SET_FLAG" });
            trigger.Actions[0].Params["name"] = "fired";
            trigger.Actions[0].Params["value"] = "true";
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "X";
            engine.Dispatch(record);
            Assert.Null(ctx.Flags.Get("fired"));

            ctx.Flags.Set("chapter", "3");
            engine.Dispatch(record);
            Assert.Equal("true", ctx.Flags.Get("fired"));
        }

        [Fact]
        public void OnceTrigger_DoesNotFireTwice()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "X");
            trigger.Once = true;
            trigger.Actions.Add(new ActionSpec { Kind = "SET_FLAG" });
            trigger.Actions[0].Params["name"] = "count";
            trigger.Actions[0].Params["value"] = "1";
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "X";
            engine.Dispatch(record);
            ctx.Flags.Set("count", "old");
            engine.Dispatch(record);
            Assert.Equal("old", ctx.Flags.Get("count"));
        }

        [Fact]
        public void RepeatableTrigger_FiresEveryTime()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "X");
            trigger.Once = false;
            trigger.Actions.Add(new ActionSpec { Kind = "SET_FLAG" });
            trigger.Actions[0].Params["name"] = "last";
            trigger.Actions[0].Params["value"] = "fired";
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "X";
            engine.Dispatch(record);
            ctx.Flags.Set("last", "overwritten");
            engine.Dispatch(record);
            Assert.Equal("fired", ctx.Flags.Get("last"));
        }

        [Fact]
        public void UnregisteredActionKind_SkipsSilently()
        {
            var (engine, ctx) = NewEngine();
            var trigger = TriggerMatchingContract("t1", "X");
            trigger.Actions.Add(new ActionSpec { Kind = "UNKNOWN_ACTION" });
            engine.Register(trigger);

            var record = new EventRecord { Type = EventType.ContractComplete };
            record.Params["contract"] = "X";
            engine.Dispatch(record);  // should not throw
        }
    }
}
```

- [ ] **Step 2: Write `TriggerEngine.cs`**

Contents:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Triggers.Actions;
using KerbalCampaignKit.Triggers.Events;

namespace KerbalCampaignKit.Triggers
{
    /// <summary>
    /// Routes EventRecords to matching triggers and executes their actions.
    /// Tracks which Once-triggers have fired (persisted externally by
    /// CampaignKitScenario). Thread-unsafe — KSP is single-threaded.
    /// </summary>
    public sealed class TriggerEngine
    {
        private readonly ActionContext ctx;
        private readonly List<Trigger> triggers = new List<Trigger>();
        private readonly HashSet<string> firedOnceIds = new HashSet<string>();
        private readonly Dictionary<string, IAction> actions =
            new Dictionary<string, IAction>();

        public TriggerEngine(ActionContext ctx) { this.ctx = ctx; }

        public void Register(Trigger trigger) => triggers.Add(trigger);

        public void RegisterAction(IAction action) => actions[action.Kind] = action;

        public IReadOnlyCollection<string> FiredOnceIds => firedOnceIds;

        public void MarkFired(string triggerId) => firedOnceIds.Add(triggerId);

        public void Dispatch(EventRecord record)
        {
            foreach (var trigger in triggers)
            {
                if (trigger.Once && firedOnceIds.Contains(trigger.Id)) continue;
                if (!AnyEventMatches(trigger, record)) continue;
                if (!RequirementExpression.Evaluate(trigger.WhenExpression, ctx.Flags)) continue;

                foreach (var spec in trigger.Actions)
                {
                    if (!actions.TryGetValue(spec.Kind, out var action)) continue;
                    action.Execute(spec, ctx);
                }

                if (trigger.Once) firedOnceIds.Add(trigger.Id);
            }
        }

        private static bool AnyEventMatches(Trigger trigger, EventRecord record)
        {
            foreach (var spec in trigger.Events)
                if (spec.Matches(record)) return true;
            return false;
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 6 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter TriggerEngineTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/TriggerEngine.cs tests/KerbalCampaignKit.Tests/TriggerEngineTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: TriggerEngine dispatch with dedup and action registry"
```

---

## Task 21: ReputationIncome (tier table + math)

**Files:**
- Create: `src/KerbalCampaignKit/Reputation/ReputationIncome.cs`
- Create: `tests/KerbalCampaignKit.Tests/ReputationIncomeTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/ReputationIncomeTests.cs`:
```csharp
using KerbalCampaignKit.Reputation;
using KerbalCampaignKit.Tests.TestHelpers;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ReputationIncomeTests
    {
        private static ReputationIncome Build(params (double min, double max, double income)[] tiers)
        {
            var inc = new ReputationIncome { Enabled = true, IntervalDays = 30 };
            foreach (var t in tiers)
                inc.Tiers.Add(new ReputationIncome.Tier { Min = t.min, Max = t.max, Income = t.income });
            return inc;
        }

        [Fact]
        public void IncomeForRep_PicksMatchingTier()
        {
            var inc = Build((0, 100, 5000), (100, 250, 15000), (250, 500, 35000));
            Assert.Equal(5000, inc.IncomeForRep(0));
            Assert.Equal(5000, inc.IncomeForRep(99));
            Assert.Equal(15000, inc.IncomeForRep(100));
            Assert.Equal(15000, inc.IncomeForRep(249));
            Assert.Equal(35000, inc.IncomeForRep(300));
        }

        [Fact]
        public void IncomeForRep_ReturnsZeroWhenDisabled()
        {
            var inc = Build((0, 100, 5000));
            inc.Enabled = false;
            Assert.Equal(0, inc.IncomeForRep(50));
        }

        [Fact]
        public void IncomeForRep_ReturnsZeroWhenNoMatchingTier()
        {
            var inc = Build((100, 200, 5000));
            Assert.Equal(0, inc.IncomeForRep(50));
            Assert.Equal(0, inc.IncomeForRep(500));
        }

        [Fact]
        public void TierLabelForRep_ReturnsFirstMatch()
        {
            var inc = Build((0, 100, 5000), (100, 250, 15000));
            inc.Tiers[0].Label = "Startup";
            inc.Tiers[1].Label = "Established";
            Assert.Equal("Startup", inc.TierLabelForRep(50));
            Assert.Equal("Established", inc.TierLabelForRep(200));
        }
    }
}
```

- [ ] **Step 2: Write `ReputationIncome.cs`**

Contents:
```csharp
using System.Collections.Generic;

namespace KerbalCampaignKit.Reputation
{
    /// <summary>
    /// Pure-logic model for tiered reputation income. Tier matching is
    /// [min, max) half-open. Disabled or missing tier ranges return 0.
    /// </summary>
    public sealed class ReputationIncome
    {
        public bool Enabled;
        public double IntervalDays = 30;
        public List<Tier> Tiers { get; } = new List<Tier>();

        public sealed class Tier
        {
            public double Min;
            public double Max;
            public double Income;
            public string Label;
        }

        public double IncomeForRep(double rep)
        {
            if (!Enabled) return 0;
            var t = MatchingTier(rep);
            return t?.Income ?? 0;
        }

        public string TierLabelForRep(double rep) => MatchingTier(rep)?.Label;

        private Tier MatchingTier(double rep)
        {
            foreach (var t in Tiers)
                if (rep >= t.Min && rep < t.Max) return t;
            return null;
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 4 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ReputationIncomeTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Reputation/ReputationIncome.cs tests/KerbalCampaignKit.Tests/ReputationIncomeTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: ReputationIncome tier matching + math"
```

---

## Task 22: ReputationDecay (math + halt + tier floor)

**Files:**
- Create: `src/KerbalCampaignKit/Reputation/ReputationDecay.cs`
- Create: `tests/KerbalCampaignKit.Tests/ReputationDecayTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/ReputationDecayTests.cs`:
```csharp
using KerbalCampaignKit.Reputation;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class ReputationDecayTests
    {
        [Fact]
        public void NoDecayIfDisabled()
        {
            var d = new ReputationDecay { Enabled = false, RatePercentPerMonth = 10 };
            Assert.Equal(0, d.ComputeDecay(currentRep: 500, elapsedDays: 30, floor: 0));
        }

        [Fact]
        public void DecayScalesWithElapsedTime()
        {
            var d = new ReputationDecay { Enabled = true, RatePercentPerMonth = 2 };
            // 2% of 500 over 30 days = 10
            Assert.Equal(10, d.ComputeDecay(currentRep: 500, elapsedDays: 30, floor: 0));
            // 60 days = double
            Assert.Equal(20, d.ComputeDecay(currentRep: 500, elapsedDays: 60, floor: 0));
        }

        [Fact]
        public void DecayRespectsFloor()
        {
            var d = new ReputationDecay { Enabled = true, RatePercentPerMonth = 50 };
            // 50% of 300 = 150, but floor is 250, so max decay is 300 - 250 = 50
            Assert.Equal(50, d.ComputeDecay(currentRep: 300, elapsedDays: 30, floor: 250));
        }

        [Fact]
        public void DecayIsZeroBelowFloor()
        {
            var d = new ReputationDecay { Enabled = true, RatePercentPerMonth = 50 };
            Assert.Equal(0, d.ComputeDecay(currentRep: 200, elapsedDays: 30, floor: 250));
        }

        [Fact]
        public void DecayIsHaltedIfHaltUntilInFuture()
        {
            var d = new ReputationDecay { Enabled = true, RatePercentPerMonth = 10, HaltUntilSeconds = 1000 };
            Assert.True(d.IsHalted(nowSeconds: 500));
            Assert.False(d.IsHalted(nowSeconds: 1500));
        }
    }
}
```

- [ ] **Step 2: Write `ReputationDecay.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Reputation
{
    /// <summary>
    /// Pure-logic model for reputation decay. `RatePercentPerMonth` is a
    /// percentage of current rep. Decay respects `floor` — the rep value
    /// can't be reduced below it. `HaltUntilSeconds` is an in-game time
    /// deadline before which decay does not apply (for PR Campaign actions).
    /// </summary>
    public sealed class ReputationDecay
    {
        public bool Enabled;
        public double RatePercentPerMonth = 1.5;
        public bool ResetOnContractComplete = true;
        public bool TierFloors = true;
        public double HaltUntilSeconds;

        public bool IsHalted(double nowSeconds) => nowSeconds < HaltUntilSeconds;

        /// <summary>
        /// Decay amount (positive) to subtract from currentRep.
        /// Never exceeds currentRep - floor.
        /// </summary>
        public double ComputeDecay(double currentRep, double elapsedDays, double floor)
        {
            if (!Enabled) return 0;
            if (currentRep <= floor) return 0;

            // monthly percentage scaled by days-per-month (30)
            var monthlyDecay = currentRep * (RatePercentPerMonth / 100.0);
            var decay = monthlyDecay * (elapsedDays / 30.0);

            var maxDecay = currentRep - floor;
            if (decay > maxDecay) decay = maxDecay;
            if (decay < 0) decay = 0;
            return decay;
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 5 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ReputationDecayTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Reputation/ReputationDecay.cs tests/KerbalCampaignKit.Tests/ReputationDecayTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: ReputationDecay math with floor, halt, and rate"
```

---

## Task 23: ReputationLoader (parse REPUTATION_INCOME, REPUTATION_DECAY cfg)

**Files:**
- Create: `src/KerbalCampaignKit/Reputation/ReputationLoader.cs`

Small enough to combine with the loader test written inline.

- [ ] **Step 1: Write `ReputationLoader.cs`**

Contents:
```csharp
using KerbalCampaignKit.Config;

namespace KerbalCampaignKit.Reputation
{
    /// <summary>
    /// Parses REPUTATION_INCOME and REPUTATION_DECAY cfg nodes into models.
    /// </summary>
    public static class ReputationLoader
    {
        public static ReputationIncome LoadIncome(ISceneNode node)
        {
            var inc = new ReputationIncome();
            inc.Enabled = !node.HasValue("enabled") || node.GetValue("enabled") == "true";
            if (node.HasValue("intervalDays") && double.TryParse(node.GetValue("intervalDays"), out var i))
                inc.IntervalDays = i;

            foreach (var tierNode in node.GetNodes("TIER"))
            {
                var tier = new ReputationIncome.Tier();
                if (tierNode.HasValue("min") && double.TryParse(tierNode.GetValue("min"), out var min))
                    tier.Min = min;
                if (tierNode.HasValue("max") && double.TryParse(tierNode.GetValue("max"), out var max))
                    tier.Max = max;
                if (tierNode.HasValue("income") && double.TryParse(tierNode.GetValue("income"), out var income))
                    tier.Income = income;
                tier.Label = tierNode.HasValue("label") ? tierNode.GetValue("label") : null;
                inc.Tiers.Add(tier);
            }
            return inc;
        }

        public static ReputationDecay LoadDecay(ISceneNode node)
        {
            var d = new ReputationDecay();
            d.Enabled = !node.HasValue("enabled") || node.GetValue("enabled") == "true";
            if (node.HasValue("ratePercentPerMonth") && double.TryParse(node.GetValue("ratePercentPerMonth"), out var r))
                d.RatePercentPerMonth = r;
            if (node.HasValue("resetOnContractComplete"))
                d.ResetOnContractComplete = node.GetValue("resetOnContractComplete") == "true";
            if (node.HasValue("tierFloors"))
                d.TierFloors = node.GetValue("tierFloors") == "true";
            return d;
        }
    }
}
```

- [ ] **Step 2: Append tests to `ReputationIncomeTests.cs`**

Append:
```csharp
        [Fact]
        public void LoadIncome_ReadsTiers()
        {
            var root = new FakeConfigNode()
                .Add("enabled", "true")
                .Add("intervalDays", "15");
            var t1 = new FakeConfigNode()
                .Add("min", "0").Add("max", "100").Add("income", "5000").Add("label", "Startup");
            var t2 = new FakeConfigNode()
                .Add("min", "100").Add("max", "250").Add("income", "15000");
            root.AddNode("TIER", t1).AddNode("TIER", t2);

            var inc = ReputationLoader.LoadIncome(root);
            Assert.True(inc.Enabled);
            Assert.Equal(15, inc.IntervalDays);
            Assert.Equal(2, inc.Tiers.Count);
            Assert.Equal("Startup", inc.Tiers[0].Label);
            Assert.Equal(15000, inc.Tiers[1].Income);
        }
```

Add `using KerbalCampaignKit.Tests.TestHelpers;` at top if not present.

- [ ] **Step 3: Run tests + commit**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter ReputationIncomeTests
git add src/KerbalCampaignKit/Reputation/ReputationLoader.cs tests/KerbalCampaignKit.Tests/ReputationIncomeTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: parse REPUTATION_INCOME/DECAY cfg into models"
```

---

## Task 24: ReputationEconomy (tick integration)

**Files:**
- Create: `src/KerbalCampaignKit/Reputation/ReputationEconomy.cs`

The economy binds income + decay + the halt/gate API. Stateful (tracks `lastIncomeTime`, `lastDecayTime`, `highestTierReached`), but the mutations are pure. We integrate with KSP's `Reputation.Instance` and `Funding.Instance` through `ICurrencyAdapter`, so the economy itself is testable.

- [ ] **Step 1: Write `ReputationEconomy.cs`**

Contents:
```csharp
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.Triggers.Actions;

namespace KerbalCampaignKit.Reputation
{
    public sealed class ReputationEconomy
    {
        public ReputationIncome Income { get; set; } = new ReputationIncome();
        public ReputationDecay Decay { get; set; } = new ReputationDecay();

        public double LastIncomeTimeSeconds;
        public double LastDecayTimeSeconds;
        public double HighestRepReached;

        /// <summary>
        /// Called periodically by CampaignKitAddon. Pays out accumulated income
        /// and applies decay since the last tick. Uses the ambient currency
        /// adapter to read current rep and write deltas.
        /// </summary>
        public void Tick(ICurrencyAdapter currencies, double nowSeconds)
        {
            var rep = currencies.Reputation;
            if (rep > HighestRepReached) HighestRepReached = rep;

            // Income payout at each interval boundary.
            var intervalSeconds = Income.IntervalDays * 24 * 60 * 60;
            if (Income.Enabled && intervalSeconds > 0 && nowSeconds - LastIncomeTimeSeconds >= intervalSeconds)
            {
                var payout = Income.IncomeForRep(rep);
                if (payout > 0) currencies.AddFunds(payout);
                LastIncomeTimeSeconds = nowSeconds;
                CampaignKitEvents.FireReputationIncomeTick(payout);
            }

            // Decay — continuous; compute over elapsed interval.
            if (Decay.Enabled && !Decay.IsHalted(nowSeconds))
            {
                var elapsed = nowSeconds - LastDecayTimeSeconds;
                if (elapsed > 0)
                {
                    var elapsedDays = elapsed / (24.0 * 60 * 60);
                    var floor = Decay.TierFloors ? TierFloorBelow(HighestRepReached) : 0;
                    var amount = Decay.ComputeDecay(rep, elapsedDays, floor);
                    if (amount > 0) currencies.AddReputation(-amount);
                    LastDecayTimeSeconds = nowSeconds;
                }
            }
        }

        public void HaltDecay(double nowSeconds, double days)
        {
            var until = nowSeconds + days * 24 * 60 * 60;
            if (until > Decay.HaltUntilSeconds) Decay.HaltUntilSeconds = until;
        }

        public void ResetDecayTimer(double nowSeconds)
        {
            LastDecayTimeSeconds = nowSeconds;
        }

        /// <summary>
        /// Returns the floor for the current-or-lower tier that `highestRep`
        /// has entered. The lowest tier's Min is the global floor. If no tiers
        /// configured, returns 0.
        /// </summary>
        public double TierFloorBelow(double highestRep)
        {
            double floor = 0;
            foreach (var t in Income.Tiers)
            {
                if (highestRep >= t.Min && t.Min > floor) floor = t.Min;
            }
            return floor;
        }

        /// <summary>Next ambition gate above current rep, or null if none.</summary>
        public (double value, string label)? NextGate(double currentRep)
        {
            foreach (var t in Income.Tiers)
            {
                if (t.Min > currentRep)
                    return (t.Min, t.Label ?? $">= {t.Min} reputation");
            }
            return null;
        }
    }
}
```

- [ ] **Step 2: Build (no direct tests for Tick integration — covered by manual test later)**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

- [ ] **Step 3: Commit**

```bash
git add src/KerbalCampaignKit/Reputation/ReputationEconomy.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: ReputationEconomy tick with income payout, decay, halt, gates"
```

---

## Task 25: Event sources — Contract, Vessel, Currency (KSP-integration)

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Events/ContractEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/VesselEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/CurrencyEventSource.cs`

These subscribe to KSP GameEvents and publish `EventRecord`s to the TriggerEngine. Not unit-testable (need KSP runtime); verified via manual in-game test in the final task.

- [ ] **Step 1: Write `ContractEventSource.cs`**

Contents:
```csharp
using System;
using Contracts;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Publishes ContractComplete/Accepted/Failed/Cancelled events by
    /// subscribing to the stock contract lifecycle GameEvents.
    /// </summary>
    public sealed class ContractEventSource : IEventSource
    {
        private Action<EventRecord> sink;

        public void Register(Action<EventRecord> onEvent)
        {
            sink = onEvent;
            GameEvents.Contract.onCompleted.Add(OnCompleted);
            GameEvents.Contract.onAccepted.Add(OnAccepted);
            GameEvents.Contract.onFailed.Add(OnFailed);
            GameEvents.Contract.onCancelled.Add(OnCancelled);
        }

        public void Unregister()
        {
            GameEvents.Contract.onCompleted.Remove(OnCompleted);
            GameEvents.Contract.onAccepted.Remove(OnAccepted);
            GameEvents.Contract.onFailed.Remove(OnFailed);
            GameEvents.Contract.onCancelled.Remove(OnCancelled);
            sink = null;
        }

        private void OnCompleted(Contract c) => Fire(EventType.ContractComplete, c);
        private void OnAccepted(Contract c)  => Fire(EventType.ContractAccepted, c);
        private void OnFailed(Contract c)    => Fire(EventType.ContractFailed, c);
        private void OnCancelled(Contract c) => Fire(EventType.ContractCancelled, c);

        private void Fire(EventType type, Contract contract)
        {
            if (sink == null || contract == null) return;
            var record = new EventRecord { Type = type };
            record.Params["contract"] = ContractTypeName(contract);
            sink(record);
        }

        /// <summary>
        /// Returns the CC contract type name for ConfiguredContract, else
        /// the runtime class name. This is the value content will reference
        /// in ON_EVENT { contract = ... }.
        /// </summary>
        private static string ContractTypeName(Contract c)
        {
            // ContractConfigurator's ConfiguredContract exposes its type name
            // via a property we read by reflection to avoid a hard CC dep.
            var configuredType = c.GetType();
            var prop = configuredType.GetProperty("contractType");
            if (prop != null)
            {
                var val = prop.GetValue(c, null);
                if (val != null)
                {
                    var nameProp = val.GetType().GetProperty("name");
                    if (nameProp != null)
                        return (string)nameProp.GetValue(val, null);
                }
            }
            return configuredType.Name;
        }
    }
}
```

- [ ] **Step 2: Write `VesselEventSource.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Triggers.Events
{
    public sealed class VesselEventSource : IEventSource
    {
        private Action<EventRecord> sink;

        public void Register(Action<EventRecord> onEvent)
        {
            sink = onEvent;
            GameEvents.OnVesselRecoveryRequested.Add(OnRecovery);
            GameEvents.onCrash.Add(OnCrash);
        }

        public void Unregister()
        {
            GameEvents.OnVesselRecoveryRequested.Remove(OnRecovery);
            GameEvents.onCrash.Remove(OnCrash);
            sink = null;
        }

        private void OnRecovery(Vessel v)
        {
            if (sink == null || v == null) return;
            var record = new EventRecord { Type = EventType.VesselRecovered };
            record.Params["vesselType"] = v.vesselType.ToString();
            sink(record);
        }

        private void OnCrash(EventReport report)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.VesselCrashed };
            if (report?.origin?.vessel != null)
                record.Params["vesselType"] = report.origin.vessel.vesselType.ToString();
            sink(record);
        }
    }
}
```

- [ ] **Step 3: Write `CurrencyEventSource.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Publishes ReputationCrossed / FundsCrossed / ScienceCrossed when a
    /// currency crosses a threshold. Holds the last observed value so we
    /// can detect direction (rising/falling).
    /// </summary>
    public sealed class CurrencyEventSource : IEventSource
    {
        private Action<EventRecord> sink;
        private double lastRep = -1, lastFunds = -1, lastScience = -1;

        public void Register(Action<EventRecord> onEvent)
        {
            sink = onEvent;
            GameEvents.OnReputationChanged.Add(OnRepChanged);
            GameEvents.OnFundsChanged.Add(OnFundsChanged);
            GameEvents.OnScienceChanged.Add(OnScienceChanged);

            if (Reputation.Instance != null) lastRep = Reputation.Instance.reputation;
            if (Funding.Instance != null) lastFunds = Funding.Instance.Funds;
            if (ResearchAndDevelopment.Instance != null) lastScience = ResearchAndDevelopment.Instance.Science;
        }

        public void Unregister()
        {
            GameEvents.OnReputationChanged.Remove(OnRepChanged);
            GameEvents.OnFundsChanged.Remove(OnFundsChanged);
            GameEvents.OnScienceChanged.Remove(OnScienceChanged);
            sink = null;
        }

        private void OnRepChanged(float newValue, TransactionReasons reason)
            => FireCrossed(EventType.ReputationCrossed, lastRep, newValue, ref lastRep);

        private void OnFundsChanged(double newValue, TransactionReasons reason)
            => FireCrossed(EventType.FundsCrossed, lastFunds, newValue, ref lastFunds);

        private void OnScienceChanged(float newValue, TransactionReasons reason)
            => FireCrossed(EventType.ScienceCrossed, lastScience, newValue, ref lastScience);

        private void FireCrossed(EventType type, double oldValue, double newValue, ref double storage)
        {
            if (sink == null)
            {
                storage = newValue;
                return;
            }
            // Fire with params; trigger-level threshold filtering happens in EventSpec.Matches.
            // We include value + direction as params; trigger can filter on direction = Rising|Falling.
            var record = new EventRecord { Type = type };
            record.Params["value"] = newValue.ToString("F0");
            record.Params["direction"] = newValue > oldValue ? "Rising" : "Falling";
            sink(record);
            storage = newValue;
        }
    }
}
```

Note: `TransactionReasons` is in `KSPUtil` / stock; no extra using needed.

- [ ] **Step 4: Build (no tests; KSP-integration code)**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalCampaignKit/Triggers/Events/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: contract/vessel/currency event sources subscribe to GameEvents"
```

---

## Task 26: Event sources — Facility, Time, Chapter, Flag, Scene

**Files:**
- Create: `src/KerbalCampaignKit/Triggers/Events/FacilityEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/TimeEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/ChapterEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/FlagEventSource.cs`
- Create: `src/KerbalCampaignKit/Triggers/Events/SceneEventSource.cs`

- [ ] **Step 1: Write `FacilityEventSource.cs`**

Contents:
```csharp
using System;
using Upgradeables;

namespace KerbalCampaignKit.Triggers.Events
{
    public sealed class FacilityEventSource : IEventSource
    {
        private Action<EventRecord> sink;

        public void Register(Action<EventRecord> onEvent)
        {
            sink = onEvent;
            GameEvents.OnKSCFacilityUpgraded.Add(OnUpgraded);
            GameEvents.OnKSCFacilityUpgrading.Add(OnUpgrading);
        }

        public void Unregister()
        {
            GameEvents.OnKSCFacilityUpgraded.Remove(OnUpgraded);
            GameEvents.OnKSCFacilityUpgrading.Remove(OnUpgrading);
            sink = null;
        }

        private void OnUpgraded(UpgradeableFacility f, int level)
        {
            if (sink == null || f == null) return;
            var record = new EventRecord { Type = EventType.FacilityUpgraded };
            record.Params["facility"] = f.name;
            record.Params["level"] = level.ToString();
            sink(record);
        }

        private void OnUpgrading(UpgradeableFacility f, int level) { /* no-op; upgraded is authoritative */ }

        /// <summary>
        /// Called by CampaignKitAddon when a facility scene loads. Separate
        /// hook because it's scene-change-driven, not upgrade-driven.
        /// </summary>
        public void FireEntered(string facility)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.FacilityEntered };
            record.Params["facility"] = facility;
            sink(record);
        }
    }
}
```

- [ ] **Step 2: Write `TimeEventSource.cs`**

Contents:
```csharp
using System;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Time-based event firing. The engine polls this from the addon's
    /// update loop; the source itself holds no KSP subscription.
    /// </summary>
    public sealed class TimeEventSource : IEventSource
    {
        private Action<EventRecord> sink;
        public void Register(Action<EventRecord> onEvent) { sink = onEvent; }
        public void Unregister() { sink = null; }

        /// <summary>
        /// Emit a TimeElapsed event. Callers (usually a scheduled trigger
        /// evaluator in the addon) decide when to fire based on elapsed days.
        /// </summary>
        public void FireElapsed(double days, string reference)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.TimeElapsed };
            record.Params["days"] = days.ToString("F1");
            if (!string.IsNullOrEmpty(reference)) record.Params["ref"] = reference;
            sink(record);
        }
    }
}
```

- [ ] **Step 3: Write `ChapterEventSource.cs`**

Contents:
```csharp
using System;
using KerbalCampaignKit.Chapters;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Bridges ChapterManager.OnChapterChanged into EventRecords.
    /// </summary>
    public sealed class ChapterEventSource : IEventSource
    {
        private readonly ChapterManager chapters;
        private Action<EventRecord> sink;

        public ChapterEventSource(ChapterManager chapters) { this.chapters = chapters; }

        public void Register(Action<EventRecord> onEvent)
        {
            sink = onEvent;
            chapters.OnChapterChanged += OnChanged;
        }

        public void Unregister()
        {
            chapters.OnChapterChanged -= OnChanged;
            sink = null;
        }

        private void OnChanged(string from, string to)
        {
            if (sink == null) return;
            if (!string.IsNullOrEmpty(from))
            {
                var exit = new EventRecord { Type = EventType.ChapterExited };
                exit.Params["chapter"] = from;
                sink(exit);
            }
            if (!string.IsNullOrEmpty(to))
            {
                var enter = new EventRecord { Type = EventType.ChapterEntered };
                enter.Params["chapter"] = to;
                sink(enter);
            }
        }
    }
}
```

- [ ] **Step 4: Write `FlagEventSource.cs`**

Contents:
```csharp
using System;
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Wraps KDK's FlagStore.OnFlagChanged (if present) or polls for changes.
    /// KDK v0.1.0 may not expose change events; in that case the addon polls
    /// every few seconds and diffs. For Phase 1, we call FireChanged manually
    /// from any code path that writes flags (SetFlagAction, ClearFlagAction).
    /// </summary>
    public sealed class FlagEventSource : IEventSource
    {
        private Action<EventRecord> sink;
        public void Register(Action<EventRecord> onEvent) { sink = onEvent; }
        public void Unregister() { sink = null; }

        public void FireChanged(string name, string value)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.FlagChanged };
            record.Params["name"] = name;
            record.Params["value"] = value ?? string.Empty;
            sink(record);
        }
    }
}
```

- [ ] **Step 5: Write `SceneEventSource.cs`**

Contents:
```csharp
using System;
using KerbalDialogueKit.Core;

namespace KerbalCampaignKit.Triggers.Events
{
    /// <summary>
    /// Bridges KDK scene-level callbacks into EventRecords. KDK's
    /// SceneCallbacks are per-scene; we wrap DialogueKit so every scene
    /// ends up firing our aggregate callback.
    /// </summary>
    public sealed class SceneEventSource : IEventSource
    {
        private Action<EventRecord> sink;

        public void Register(Action<EventRecord> onEvent) { sink = onEvent; }
        public void Unregister() { sink = null; }

        /// <summary>
        /// Called by the addon when wiring new scenes. The addon chains this
        /// method onto each scene's callbacks.OnSceneEnd.
        /// </summary>
        public void FireSceneEnded(string sceneId)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.SceneEnded };
            record.Params["sceneId"] = sceneId;
            sink(record);
        }

        public void FireChoiceMade(string sceneId, string choiceId, string value)
        {
            if (sink == null) return;
            var record = new EventRecord { Type = EventType.SceneChoiceMade };
            record.Params["sceneId"] = sceneId;
            record.Params["choiceId"] = choiceId;
            record.Params["value"] = value;
            sink(record);
        }
    }
}
```

- [ ] **Step 6: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Triggers/Events/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: facility/time/chapter/flag/scene event sources"
```

---

## Task 27: AutoClearWatcher (notification auto-clear rules)

**Files:**
- Create: `src/KerbalCampaignKit/Notifications/AutoClearWatcher.cs`
- Create: `tests/KerbalCampaignKit.Tests/AutoClearWatcherTests.cs`

- [ ] **Step 1: Write the failing tests**

Contents of `tests/KerbalCampaignKit.Tests/AutoClearWatcherTests.cs`:
```csharp
using KerbalCampaignKit.Notifications;
using Xunit;

namespace KerbalCampaignKit.Tests
{
    public class AutoClearWatcherTests
    {
        [Fact]
        public void SceneEnded_ClearsMatchingNotification()
        {
            var store = new NotificationStore();
            store.Add(new Notification
            {
                Target = "admin",
                Severity = NotificationSeverity.Action,
                Source = "x",
                ClearOn = NotificationClearOn.SceneEnded,
                ClearSceneId = "directive"
            });
            var watcher = new AutoClearWatcher(store);

            watcher.OnSceneEnded("directive");
            Assert.Empty(store.At("admin"));
        }

        [Fact]
        public void SceneEnded_IgnoresNonMatchingSceneId()
        {
            var store = new NotificationStore();
            store.Add(new Notification
            {
                Target = "admin",
                ClearOn = NotificationClearOn.SceneEnded,
                ClearSceneId = "directive"
            });
            var watcher = new AutoClearWatcher(store);

            watcher.OnSceneEnded("other_scene");
            Assert.Single(store.At("admin"));
        }

        [Fact]
        public void FacilityEntered_ClearsNotificationAtThatFacility()
        {
            var store = new NotificationStore();
            store.Add(new Notification
            {
                Target = "admin",
                ClearOn = NotificationClearOn.FacilityEntered
            });
            store.Add(new Notification
            {
                Target = "admin.desk",
                ClearOn = NotificationClearOn.FacilityEntered
            });
            store.Add(new Notification
            {
                Target = "tracking",
                ClearOn = NotificationClearOn.FacilityEntered
            });

            var watcher = new AutoClearWatcher(store);
            watcher.OnFacilityEntered("admin");

            Assert.Empty(store.AtOrBelow("admin"));
            Assert.Single(store.At("tracking"));
        }

        [Fact]
        public void FlagSet_ClearsMatchingNotification()
        {
            var store = new NotificationStore();
            store.Add(new Notification
            {
                Target = "admin",
                ClearOn = NotificationClearOn.FlagSet,
                ClearFlag = "done",
                ClearFlagValue = "true"
            });
            var watcher = new AutoClearWatcher(store);

            watcher.OnFlagSet("done", "true");
            Assert.Empty(store.At("admin"));
        }

        [Fact]
        public void FlagSet_IgnoresMismatch()
        {
            var store = new NotificationStore();
            store.Add(new Notification
            {
                Target = "admin",
                ClearOn = NotificationClearOn.FlagSet,
                ClearFlag = "done",
                ClearFlagValue = "true"
            });
            var watcher = new AutoClearWatcher(store);

            watcher.OnFlagSet("done", "false");
            Assert.Single(store.At("admin"));

            watcher.OnFlagSet("other", "true");
            Assert.Single(store.At("admin"));
        }

        [Fact]
        public void ManualClearOn_NeverClearsAutomatically()
        {
            var store = new NotificationStore();
            store.Add(new Notification { Target = "admin", ClearOn = NotificationClearOn.Manual });
            var watcher = new AutoClearWatcher(store);

            watcher.OnSceneEnded("any");
            watcher.OnFacilityEntered("admin");
            watcher.OnFlagSet("anything", "true");
            Assert.Single(store.At("admin"));
        }
    }
}
```

- [ ] **Step 2: Write `AutoClearWatcher.cs`**

Contents:
```csharp
namespace KerbalCampaignKit.Notifications
{
    /// <summary>
    /// Applies NotificationClearOn rules to the store when the appropriate
    /// event fires. Addon wires this to KDK callbacks, facility-entered
    /// events, and flag-change events.
    /// </summary>
    public sealed class AutoClearWatcher
    {
        private readonly NotificationStore store;

        public AutoClearWatcher(NotificationStore store) { this.store = store; }

        public void OnSceneEnded(string sceneId)
        {
            var matches = store.All;
            // Copy to a list because Clear modifies the store during iteration.
            var toRemove = new System.Collections.Generic.List<Notification>();
            foreach (var n in matches)
                if (n.ClearOn == NotificationClearOn.SceneEnded && n.ClearSceneId == sceneId)
                    toRemove.Add(n);
            foreach (var n in toRemove)
                store.Clear(n.Target, n.Source);
        }

        public void OnFacilityEntered(string facility)
        {
            // Clear any notification whose target is at-or-below the facility
            // AND whose ClearOn = FacilityEntered.
            var toRemove = new System.Collections.Generic.List<Notification>();
            foreach (var n in store.All)
            {
                if (n.ClearOn != NotificationClearOn.FacilityEntered) continue;
                if (TargetIsAtOrBelow(n.Target, facility)) toRemove.Add(n);
            }
            foreach (var n in toRemove) store.Clear(n.Target, n.Source);
        }

        public void OnFlagSet(string name, string value)
        {
            var toRemove = new System.Collections.Generic.List<Notification>();
            foreach (var n in store.All)
            {
                if (n.ClearOn != NotificationClearOn.FlagSet) continue;
                if (n.ClearFlag != name) continue;
                if (n.ClearFlagValue != null && n.ClearFlagValue != value) continue;
                toRemove.Add(n);
            }
            foreach (var n in toRemove) store.Clear(n.Target, n.Source);
        }

        private static bool TargetIsAtOrBelow(string candidate, string ancestor)
        {
            if (candidate == ancestor) return true;
            if (candidate == null || ancestor == null) return false;
            return candidate.StartsWith(ancestor + ".");
        }
    }
}
```

- [ ] **Step 3: Run tests — expect 6 pass**

```bash
dotnet test tests/KerbalCampaignKit.Tests/KerbalCampaignKit.Tests.csproj --filter AutoClearWatcherTests
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalCampaignKit/Notifications/AutoClearWatcher.cs tests/KerbalCampaignKit.Tests/AutoClearWatcherTests.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: AutoClearWatcher applies notification clear-on rules"
```

---

## Task 28: CC REQUIREMENT types — InChapter + ChapterAtLeast

**Files:**
- Create: `src/KerbalCampaignKit/Contracts/InChapterRequirement.cs`
- Create: `src/KerbalCampaignKit/Contracts/ChapterAtLeastRequirement.cs`

KSP-integration code. CC auto-discovers types that extend `ContractRequirement`; no explicit registration needed. Manual test in the final verification task.

- [ ] **Step 1: Write `InChapterRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;
using KerbalCampaignKit.Core;

namespace KerbalCampaignKit.Contracts
{
    /// <summary>
    /// CC REQUIREMENT: requires the campaign to be exactly in the specified chapter.
    /// Usage in cfg:
    ///     REQUIREMENT { name = InChapter; type = InChapter; chapter = 3 }
    /// </summary>
    public class InChapterRequirement : ContractRequirement
    {
        protected string chapter;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "chapter", x => chapter = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            return CampaignKit.Chapters?.Current == chapter;
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("chapter", chapter);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            chapter = configNode.GetValue("chapter");
        }

        protected override string RequirementText()
        {
            return $"Campaign must be in chapter {chapter}";
        }
    }
}
```

- [ ] **Step 2: Write `ChapterAtLeastRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;
using KerbalCampaignKit.Core;

namespace KerbalCampaignKit.Contracts
{
    /// <summary>
    /// CC REQUIREMENT: requires the campaign to be at or past the specified chapter.
    /// Numeric chapter IDs compared as integers; non-numeric requires exact match.
    /// </summary>
    public class ChapterAtLeastRequirement : ContractRequirement
    {
        protected string chapter;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "chapter", x => chapter = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            return CampaignKit.Chapters?.IsAtLeast(chapter) ?? false;
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("chapter", chapter);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            chapter = configNode.GetValue("chapter");
        }

        protected override string RequirementText()
        {
            return $"Campaign must be at or past chapter {chapter}";
        }
    }
}
```

Note: `CampaignKit.Chapters` static accessor is added in Task 32.

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
```

Expected: compile error referencing `CampaignKit.Chapters` — that's fine, it'll be defined in Task 32. Add a placeholder to `Core/CampaignKit.cs` so it builds now:

Edit `src/KerbalCampaignKit/Core/CampaignKit.cs` to:
```csharp
using KerbalCampaignKit.Chapters;

namespace KerbalCampaignKit.Core
{
    public static class CampaignKit
    {
        public static ChapterManager Chapters { get; internal set; }
    }
}
```

- [ ] **Step 4: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Contracts/ src/KerbalCampaignKit/Core/CampaignKit.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: CC REQUIREMENT types InChapter + ChapterAtLeast"
```

---

## Task 29: CC REQUIREMENT types — FlagEquals, FlagNotEquals, FlagExpression, ReputationMinimum

**Files:**
- Create: `src/KerbalCampaignKit/Contracts/FlagEqualsRequirement.cs`
- Create: `src/KerbalCampaignKit/Contracts/FlagNotEqualsRequirement.cs`
- Create: `src/KerbalCampaignKit/Contracts/FlagExpressionRequirement.cs`
- Create: `src/KerbalCampaignKit/Contracts/ReputationMinimumRequirement.cs`

- [ ] **Step 1: Write `FlagEqualsRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;
using KerbalDialogueKit.Core;

namespace KerbalCampaignKit.Contracts
{
    /// <summary>
    /// CC REQUIREMENT: requires KDK flag `name` to equal `value`.
    /// Missing flags are treated as empty string.
    /// </summary>
    public class FlagEqualsRequirement : ContractRequirement
    {
        protected string flagName;
        protected string flagValue;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "name", x => flagName = x, this);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "value", x => flagValue = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            var actual = DialogueKit.Flags?.Get(flagName) ?? string.Empty;
            return actual == flagValue;
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("name", flagName);
            configNode.AddValue("value", flagValue);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            flagName = configNode.GetValue("name");
            flagValue = configNode.GetValue("value");
        }

        protected override string RequirementText()
        {
            return $"Flag '{flagName}' must equal '{flagValue}'";
        }
    }
}
```

- [ ] **Step 2: Write `FlagNotEqualsRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;
using KerbalDialogueKit.Core;

namespace KerbalCampaignKit.Contracts
{
    public class FlagNotEqualsRequirement : ContractRequirement
    {
        protected string flagName;
        protected string flagValue;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "name", x => flagName = x, this);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "value", x => flagValue = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            var actual = DialogueKit.Flags?.Get(flagName) ?? string.Empty;
            return actual != flagValue;
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("name", flagName);
            configNode.AddValue("value", flagValue);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            flagName = configNode.GetValue("name");
            flagValue = configNode.GetValue("value");
        }

        protected override string RequirementText()
        {
            return $"Flag '{flagName}' must not equal '{flagValue}'";
        }
    }
}
```

- [ ] **Step 3: Write `FlagExpressionRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;
using KerbalCampaignKit.Triggers;
using KerbalDialogueKit.Core;

namespace KerbalCampaignKit.Contracts
{
    /// <summary>
    /// CC REQUIREMENT: evaluates a KDK flag expression (full syntax).
    /// Usage: REQUIREMENT { type = FlagExpression; expression = "chapter == 3 && approach == science_heavy" }
    /// </summary>
    public class FlagExpressionRequirement : ContractRequirement
    {
        protected string expression;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<string>(configNode, "expression", x => expression = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            if (DialogueKit.Flags == null) return false;
            return RequirementExpression.Evaluate(expression, DialogueKit.Flags);
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("expression", expression);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            expression = configNode.GetValue("expression");
        }

        protected override string RequirementText()
        {
            return $"Flag expression: {expression}";
        }
    }
}
```

- [ ] **Step 4: Write `ReputationMinimumRequirement.cs`**

Contents:
```csharp
using ContractConfigurator;
using Contracts;

namespace KerbalCampaignKit.Contracts
{
    /// <summary>
    /// CC REQUIREMENT: requires current reputation >= value.
    /// </summary>
    public class ReputationMinimumRequirement : ContractRequirement
    {
        protected double minRep;

        public override bool LoadFromConfig(ConfigNode configNode)
        {
            bool valid = base.LoadFromConfig(configNode);
            valid &= ConfigNodeUtil.ParseValue<double>(configNode, "value", x => minRep = x, this);
            return valid;
        }

        public override bool RequirementMet(ConfiguredContract contract)
        {
            if (Reputation.Instance == null) return false;
            return Reputation.Instance.reputation >= minRep;
        }

        public override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("value", minRep);
        }

        public override void OnLoad(ConfigNode configNode)
        {
            double.TryParse(configNode.GetValue("value"), out minRep);
        }

        protected override string RequirementText()
        {
            return $"Requires at least {minRep:F0} reputation";
        }
    }
}
```

- [ ] **Step 5: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Contracts/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: FlagEquals, FlagNotEquals, FlagExpression, ReputationMinimum REQUIREMENTs"
```

---

## Task 30: ReputationStake contract BEHAVIOUR

**Files:**
- Create: `src/KerbalCampaignKit/Reputation/ReputationStakeBehaviour.cs`
- Create: `src/KerbalCampaignKit/Reputation/ReputationStakeFactory.cs`

CC BEHAVIOUR types require both a `ContractBehaviour` subclass and a `BehaviourFactory` that parses config and instantiates the behaviour. We ship both.

- [ ] **Step 1: Write `ReputationStakeBehaviour.cs`**

Contents:
```csharp
using ContractConfigurator.Behaviour;
using Contracts;

namespace KerbalCampaignKit.Reputation
{
    /// <summary>
    /// Applies a per-contract reputation bonus on success and penalty on
    /// failure, layered on top of stock rep rewards.
    /// </summary>
    public class ReputationStakeBehaviour : ContractBehaviour
    {
        public float SuccessBonus;
        public float FailurePenalty;

        public ReputationStakeBehaviour() { }

        public ReputationStakeBehaviour(float successBonus, float failurePenalty)
        {
            SuccessBonus = successBonus;
            FailurePenalty = failurePenalty;
        }

        protected override void OnCompleted()
        {
            if (SuccessBonus != 0 && Reputation.Instance != null)
                Reputation.Instance.AddReputation(SuccessBonus, TransactionReasons.ContractReward);
        }

        protected override void OnFailed()
        {
            if (FailurePenalty != 0 && Reputation.Instance != null)
                Reputation.Instance.AddReputation(-FailurePenalty, TransactionReasons.ContractPenalty);
        }

        protected override void OnSave(ConfigNode configNode)
        {
            configNode.AddValue("successBonus", SuccessBonus);
            configNode.AddValue("failurePenalty", FailurePenalty);
        }

        protected override void OnLoad(ConfigNode configNode)
        {
            float.TryParse(configNode.GetValue("successBonus"), out SuccessBonus);
            float.TryParse(configNode.GetValue("failurePenalty"), out FailurePenalty);
        }
    }
}
```

- [ ] **Step 2: Write `ReputationStakeFactory.cs`**

Contents:
```csharp
using ContractConfigurator;
using ContractConfigurator.Behaviour;
using Contracts;

namespace KerbalCampaignKit.Reputation
{
    public class ReputationStakeFactory : BehaviourFactory
    {
        protected float successBonus;
        protected float failurePenalty;

        public override bool Load(ConfigNode configNode)
        {
            bool valid = base.Load(configNode);
            valid &= ConfigNodeUtil.ParseValue<float>(configNode, "successBonus", x => successBonus = x, this, 0f);
            valid &= ConfigNodeUtil.ParseValue<float>(configNode, "failurePenalty", x => failurePenalty = x, this, 0f);
            return valid;
        }

        public override ContractBehaviour Generate(ConfiguredContract contract)
        {
            return new ReputationStakeBehaviour(successBonus, failurePenalty);
        }
    }
}
```

- [ ] **Step 3: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Reputation/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: ReputationStake CC behaviour with success/failure rep deltas"
```

---

## Task 31: CampaignKitScenario (save/load)

**Files:**
- Create: `src/KerbalCampaignKit/Core/CampaignKitScenario.cs`

Persists: current chapter, chapter history, fired-once trigger ids, pending scenes, notifications, reputation economy state. Manual-test only; ScenarioModule requires KSP.

- [ ] **Step 1: Write `CampaignKitScenario.cs`**

Contents:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.PendingScenes;
using KerbalCampaignKit.Reputation;
using KerbalCampaignKit.Triggers;

namespace KerbalCampaignKit.Core
{
    [KSPScenario(ScenarioCreationOptions.AddToAllGames,
        GameScenes.SPACECENTER, GameScenes.FLIGHT, GameScenes.TRACKSTATION, GameScenes.EDITOR)]
    public class CampaignKitScenario : ScenarioModule
    {
        public static CampaignKitScenario Instance;

        public ChapterManager Chapters = new ChapterManager();
        public NotificationStore Notifications = new NotificationStore();
        public PendingSceneQueue PendingScenes = new PendingSceneQueue();
        public ReputationEconomy Reputation = new ReputationEconomy();
        public TriggerEngine Engine;  // wired by addon after scenario loads

        public override void OnAwake()
        {
            base.OnAwake();
            Instance = this;
        }

        public override void OnSave(ConfigNode node)
        {
            base.OnSave(node);
            node.AddValue("currentChapter", Chapters.Current ?? string.Empty);

            var history = node.AddNode("CHAPTER_HISTORY");
            foreach (var entry in Chapters.History.Entries)
            {
                var e = history.AddNode("ENTERED");
                e.AddValue("chapter", entry.ChapterId);
                e.AddValue("timestamp", entry.TimestampSeconds);
            }

            var fired = node.AddNode("FIRED_TRIGGERS");
            if (Engine != null)
                foreach (var id in Engine.FiredOnceIds)
                    fired.AddValue("trigger", id);

            var pending = node.AddNode("PENDING_SCENES");
            foreach (var ps in PendingScenes.Pending)
            {
                var s = pending.AddNode("SCENE");
                s.AddValue("sceneId", ps.SceneId);
                if (!string.IsNullOrEmpty(ps.Facility)) s.AddValue("facility", ps.Facility);
                if (!string.IsNullOrEmpty(ps.FromTriggerId)) s.AddValue("fromTrigger", ps.FromTriggerId);
            }

            var notes = node.AddNode("NOTIFICATIONS");
            foreach (var n in Notifications.All)
            {
                var nn = notes.AddNode("NOTIFICATION");
                nn.AddValue("target", n.Target);
                nn.AddValue("severity", n.Severity.ToString());
                nn.AddValue("source", n.Source ?? string.Empty);
                nn.AddValue("clearOn", n.ClearOn.ToString());
                if (!string.IsNullOrEmpty(n.ClearSceneId)) nn.AddValue("clearSceneId", n.ClearSceneId);
                if (!string.IsNullOrEmpty(n.ClearFlag)) nn.AddValue("clearFlag", n.ClearFlag);
                if (!string.IsNullOrEmpty(n.ClearFlagValue)) nn.AddValue("clearFlagValue", n.ClearFlagValue);
                nn.AddValue("addedAt", n.AddedAtSeconds);
            }

            var rep = node.AddNode("REPUTATION_STATE");
            rep.AddValue("lastIncomeTime", Reputation.LastIncomeTimeSeconds);
            rep.AddValue("lastDecayTime", Reputation.LastDecayTimeSeconds);
            rep.AddValue("decayHaltUntil", Reputation.Decay.HaltUntilSeconds);
            rep.AddValue("highestRepReached", Reputation.HighestRepReached);
        }

        public override void OnLoad(ConfigNode node)
        {
            base.OnLoad(node);
            var current = node.GetValue("currentChapter");
            if (!string.IsNullOrEmpty(current))
                Chapters.Advance(current, 0);

            if (node.HasNode("CHAPTER_HISTORY"))
            {
                // Rebuild history without re-firing Advance events.
                Chapters.History.Entries.Clear();
                foreach (var e in node.GetNode("CHAPTER_HISTORY").GetNodes("ENTERED"))
                {
                    double.TryParse(e.GetValue("timestamp"), out var ts);
                    Chapters.History.RecordEntry(e.GetValue("chapter"), ts);
                }
            }

            if (node.HasNode("FIRED_TRIGGERS") && Engine != null)
            {
                foreach (var id in node.GetNode("FIRED_TRIGGERS").GetValues("trigger"))
                    Engine.MarkFired(id);
            }

            if (node.HasNode("PENDING_SCENES"))
            {
                foreach (var s in node.GetNode("PENDING_SCENES").GetNodes("SCENE"))
                {
                    PendingScenes.Add(new PendingScene
                    {
                        SceneId = s.GetValue("sceneId"),
                        Facility = s.HasValue("facility") ? s.GetValue("facility") : null,
                        FromTriggerId = s.HasValue("fromTrigger") ? s.GetValue("fromTrigger") : null,
                    });
                }
            }

            if (node.HasNode("NOTIFICATIONS"))
            {
                foreach (var nn in node.GetNode("NOTIFICATIONS").GetNodes("NOTIFICATION"))
                {
                    var n = new Notification
                    {
                        Target = nn.GetValue("target"),
                        Source = nn.GetValue("source"),
                        ClearSceneId = nn.HasValue("clearSceneId") ? nn.GetValue("clearSceneId") : null,
                        ClearFlag = nn.HasValue("clearFlag") ? nn.GetValue("clearFlag") : null,
                        ClearFlagValue = nn.HasValue("clearFlagValue") ? nn.GetValue("clearFlagValue") : null,
                    };
                    System.Enum.TryParse(nn.GetValue("severity"), out NotificationSeverity sev);
                    n.Severity = sev;
                    System.Enum.TryParse(nn.GetValue("clearOn"), out NotificationClearOn clearOn);
                    n.ClearOn = clearOn;
                    double.TryParse(nn.GetValue("addedAt"), out n.AddedAtSeconds);
                    Notifications.Add(n);
                }
            }

            if (node.HasNode("REPUTATION_STATE"))
            {
                var rep = node.GetNode("REPUTATION_STATE");
                double.TryParse(rep.GetValue("lastIncomeTime"), out Reputation.LastIncomeTimeSeconds);
                double.TryParse(rep.GetValue("lastDecayTime"), out Reputation.LastDecayTimeSeconds);
                double.TryParse(rep.GetValue("decayHaltUntil"), out Reputation.Decay.HaltUntilSeconds);
                double.TryParse(rep.GetValue("highestRepReached"), out Reputation.HighestRepReached);
            }
        }
    }
}
```

- [ ] **Step 2: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Core/CampaignKitScenario.cs Plugins/KerbalCampaignKit.dll
git commit -m "feat: CampaignKitScenario persists chapter, pending scenes, notifications, rep state"
```

---

## Task 32: StateMirror + CampaignKitAddon + public API

**Files:**
- Create: `src/KerbalCampaignKit/Core/StateMirror.cs`
- Create: `src/KerbalCampaignKit/Core/CampaignKitAddon.cs`
- Modify: `src/KerbalCampaignKit/Core/CampaignKit.cs`

The addon wires everything together at runtime. It:
1. On SpaceCenter load, pulls scenario instance
2. Loads CAMPAIGN_CHAPTER, CAMPAIGN_TRIGGER, REPUTATION_INCOME, REPUTATION_DECAY cfg nodes
3. Constructs ActionContext + TriggerEngine + event sources
4. Registers all built-in actions
5. Starts event sources
6. Polls ReputationEconomy.Tick() and TimeEventSource each FixedUpdate
7. Plays pending scenes when facility enter fires

- [ ] **Step 1: Write `StateMirror.cs`**

Contents:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalDialogueKit.Flags;

namespace KerbalCampaignKit.Core
{
    /// <summary>
    /// Mirrors career state (chapter, reputation, funds, science, facility levels)
    /// into well-known KDK flags so RequirementExpression / visibleIf can reach them.
    /// </summary>
    public sealed class StateMirror
    {
        private readonly FlagStore flags;
        private readonly ChapterManager chapters;

        public StateMirror(FlagStore flags, ChapterManager chapters)
        {
            this.flags = flags;
            this.chapters = chapters;
            chapters.OnChapterChanged += (_, to) => flags.Set("chapter", to ?? string.Empty);
        }

        public void SyncAll()
        {
            flags.Set("chapter", chapters.Current ?? string.Empty);
            if (Reputation.Instance != null)
                flags.Set("reputation", ((int)Reputation.Instance.reputation).ToString());
            if (Funding.Instance != null)
                flags.Set("funds", ((long)Funding.Instance.Funds).ToString());
            if (ResearchAndDevelopment.Instance != null)
                flags.Set("science", ((int)ResearchAndDevelopment.Instance.Science).ToString());

            SyncFacilityLevel("Administration");
            SyncFacilityLevel("MissionControl");
            SyncFacilityLevel("TrackingStation");
            SyncFacilityLevel("ResearchAndDevelopment");
            SyncFacilityLevel("VehicleAssemblyBuilding");
            SyncFacilityLevel("SpaceplaneHangar");
            SyncFacilityLevel("AstronautComplex");
            SyncFacilityLevel("LaunchPad");
            SyncFacilityLevel("Runway");
        }

        private void SyncFacilityLevel(string facilityName)
        {
            var level = (int)(ScenarioUpgradeableFacilities.GetFacilityLevel(
                $"SpaceCenter/{facilityName}") * 2) + 1;
            flags.Set($"facility.{facilityName}", level.ToString());
        }
    }
}
```

- [ ] **Step 2: Update `Core/CampaignKit.cs`**

Contents:
```csharp
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.Reputation;
using KerbalCampaignKit.Triggers;
using KerbalCampaignKit.Triggers.Events;

namespace KerbalCampaignKit.Core
{
    /// <summary>
    /// Public static entry point. Accessors resolve to the active scenario's
    /// subsystems. All null if called before the scenario initializes.
    /// </summary>
    public static class CampaignKit
    {
        public static ChapterManager Chapters { get; internal set; }
        public static NotificationStore Notifications { get; internal set; }
        public static ReputationEconomy Reputation { get; internal set; }
        public static TriggerEngine Engine { get; internal set; }

        /// <summary>
        /// Facility-entered hook for building overlays. Called by consumer UIs
        /// or internal addon when a KSC facility scene loads.
        /// </summary>
        public static void FireFacilityEntered(string facility)
        {
            facilityEventSource?.FireEntered(facility);
        }

        internal static FacilityEventSource facilityEventSource;
    }
}
```

- [ ] **Step 3: Write `CampaignKitAddon.cs`**

Contents:
```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Chapters;
using KerbalCampaignKit.Config;
using KerbalCampaignKit.Notifications;
using KerbalCampaignKit.Reputation;
using KerbalCampaignKit.Triggers;
using KerbalCampaignKit.Triggers.Actions;
using KerbalCampaignKit.Triggers.Events;
using KerbalDialogueKit.Core;
using UnityEngine;

namespace KerbalCampaignKit.Core
{
    [KSPAddon(KSPAddon.Startup.MainMenu, true)]
    public sealed class CampaignKitAddon : MonoBehaviour
    {
        private TriggerEngine engine;
        private ActionContext ctx;
        private StateMirror mirror;
        private AutoClearWatcher autoClear;
        private readonly List<IEventSource> sources = new List<IEventSource>();

        private FlagEventSource flagSource;
        private TimeEventSource timeSource;
        private FacilityEventSource facilitySource;
        private SceneEventSource sceneSource;

        private double lastTickSeconds;

        private void Awake()
        {
            DontDestroyOnLoad(this);
            GameEvents.onLevelWasLoaded.Add(OnLevelLoaded);
            GameEvents.OnGameSettingsApplied.Add(OnSettings);
        }

        private void OnDestroy()
        {
            GameEvents.onLevelWasLoaded.Remove(OnLevelLoaded);
            GameEvents.OnGameSettingsApplied.Remove(OnSettings);
            foreach (var s in sources) s.Unregister();
        }

        private void OnLevelLoaded(GameScenes scene)
        {
            if (CampaignKitScenario.Instance == null) return;
            if (engine != null) return;  // already initialized
            InitializeAll();

            // Fire facility-entered for relevant scenes
            switch (scene)
            {
                case GameScenes.SPACECENTER: CampaignKit.FireFacilityEntered("SpaceCenter"); break;
                case GameScenes.EDITOR: CampaignKit.FireFacilityEntered("VehicleAssemblyBuilding"); break;
                case GameScenes.TRACKSTATION: CampaignKit.FireFacilityEntered("TrackingStation"); break;
            }

            PlayImmediatePending();
        }

        private void OnSettings() { /* placeholder */ }

        private void InitializeAll()
        {
            var scenario = CampaignKitScenario.Instance;
            ctx = new ActionContext
            {
                Flags = DialogueKit.Flags,
                Chapters = scenario.Chapters,
                Notifications = scenario.Notifications,
                PendingScenes = scenario.PendingScenes,
                Currencies = new KspCurrencyAdapter(),
                SceneEnqueuer = new KspSceneEnqueuer(),
                NowSeconds = Planetarium.GetUniversalTime()
            };

            engine = new TriggerEngine(ctx);
            scenario.Engine = engine;
            RegisterActions();

            // Load chapters + triggers + reputation from GameDatabase
            LoadContent();

            // Wire event sources
            flagSource = new FlagEventSource();
            timeSource = new TimeEventSource();
            facilitySource = new FacilityEventSource();
            sceneSource = new SceneEventSource();
            sources.Add(new ContractEventSource());
            sources.Add(new VesselEventSource());
            sources.Add(new CurrencyEventSource());
            sources.Add(new FacilityEventSource());  // for upgrades
            sources.Add(new ChapterEventSource(scenario.Chapters));
            sources.Add(flagSource);
            sources.Add(timeSource);
            sources.Add(facilitySource);
            sources.Add(sceneSource);
            foreach (var s in sources) s.Register(engine.Dispatch);
            CampaignKit.facilityEventSource = facilitySource;

            autoClear = new AutoClearWatcher(scenario.Notifications);

            // State mirror: push career state into flags
            mirror = new StateMirror(DialogueKit.Flags, scenario.Chapters);
            mirror.SyncAll();

            // Public API
            CampaignKit.Chapters = scenario.Chapters;
            CampaignKit.Notifications = scenario.Notifications;
            CampaignKit.Reputation = scenario.Reputation;
            CampaignKit.Engine = engine;

            lastTickSeconds = Planetarium.GetUniversalTime();
        }

        private void RegisterActions()
        {
            engine.RegisterAction(new SetFlagAction());
            engine.RegisterAction(new ClearFlagAction());
            engine.RegisterAction(new AdvanceChapterAction());
            engine.RegisterAction(new AdjustCurrencyAction("ADJUST_FUNDS"));
            engine.RegisterAction(new AdjustCurrencyAction("ADJUST_REPUTATION"));
            engine.RegisterAction(new AdjustCurrencyAction("ADJUST_SCIENCE"));
            engine.RegisterAction(new EnqueueSceneAction());
            engine.RegisterAction(new NotifyAction());
            engine.RegisterAction(new ClearNotificationAction());
            engine.RegisterAction(new PublishEventAction());
        }

        private void LoadContent()
        {
            var scenario = CampaignKitScenario.Instance;

            foreach (var chapterCfg in GameDatabase.Instance.GetConfigNodes("CAMPAIGN_CHAPTER"))
            {
                var chapter = ChapterLoader.Load(new ConfigNodeAdapter(chapterCfg));
                // We don't maintain a registry of chapters here (not needed for Phase 1);
                // content declares all chapters, and the trigger engine advances between
                // them by ID. Loaders for description, name are available via API extensions.
            }

            foreach (var triggerCfg in GameDatabase.Instance.GetConfigNodes("CAMPAIGN_TRIGGER"))
            {
                var trigger = TriggerLoader.Load(new ConfigNodeAdapter(triggerCfg));
                if (trigger != null) engine.Register(trigger);
            }

            var incomeNodes = GameDatabase.Instance.GetConfigNodes("REPUTATION_INCOME");
            if (incomeNodes.Length > 0)
                scenario.Reputation.Income = ReputationLoader.LoadIncome(new ConfigNodeAdapter(incomeNodes[0]));

            var decayNodes = GameDatabase.Instance.GetConfigNodes("REPUTATION_DECAY");
            if (decayNodes.Length > 0)
                scenario.Reputation.Decay = ReputationLoader.LoadDecay(new ConfigNodeAdapter(decayNodes[0]));
        }

        private void PlayImmediatePending()
        {
            var scenario = CampaignKitScenario.Instance;
            if (scenario == null) return;
            // Scene with an OnFacilityEnter for the current scene plays now.
            string facility = FacilityNameForCurrentScene();
            if (string.IsNullOrEmpty(facility)) return;
            foreach (var ps in scenario.PendingScenes.TakeForFacility(facility))
                DialogueKit.EnqueueById(ps.SceneId);
        }

        private static string FacilityNameForCurrentScene()
        {
            // Map KSP scene → facility name used by ENQUEUE_SCENE's facility param.
            switch (HighLogic.LoadedScene)
            {
                case GameScenes.SPACECENTER: return null;  // non-specific — need sub-scene
                case GameScenes.EDITOR: return "VehicleAssemblyBuilding";
                case GameScenes.TRACKSTATION: return "TrackingStation";
                default: return null;
            }
        }

        private void FixedUpdate()
        {
            if (engine == null) return;
            var now = Planetarium.GetUniversalTime();
            ctx.NowSeconds = now;

            // Run reputation tick every 10 game-seconds
            if (now - lastTickSeconds > 10)
            {
                CampaignKitScenario.Instance.Reputation.Tick(ctx.Currencies, now);
                lastTickSeconds = now;
            }
        }
    }

    /// <summary>Wraps KSP's currency singletons.</summary>
    internal sealed class KspCurrencyAdapter : ICurrencyAdapter
    {
        public double Funds => Funding.Instance != null ? Funding.Instance.Funds : 0;
        public double Reputation => global::Reputation.Instance != null ? global::Reputation.Instance.reputation : 0;
        public double Science => ResearchAndDevelopment.Instance != null ? ResearchAndDevelopment.Instance.Science : 0;

        public void AddFunds(double amount) =>
            Funding.Instance?.AddFunds(amount, TransactionReasons.None);
        public void AddReputation(double amount) =>
            global::Reputation.Instance?.AddReputation((float)amount, TransactionReasons.None);
        public void AddScience(double amount) =>
            ResearchAndDevelopment.Instance?.AddScience((float)amount, TransactionReasons.None);
    }

    internal sealed class KspSceneEnqueuer : ISceneEnqueuer
    {
        public bool EnqueueById(string sceneId) => DialogueKit.EnqueueById(sceneId);
    }
}
```

Note: the addon references `ConfigNodeAdapter`, which currently lives in the `KerbalCampaignKit.Config` namespace. Adjust usings if the namespace path differs.

- [ ] **Step 4: Build + commit**

```bash
dotnet build src/KerbalCampaignKit/KerbalCampaignKit.csproj -c Release
git add src/KerbalCampaignKit/Core/ Plugins/KerbalCampaignKit.dll
git commit -m "feat: CampaignKitAddon wires scenario, engine, event sources, and public API"
```

---

## Task 33: Example campaign cfg + README fleshing

**Files:**
- Create: `Examples/demo-campaign.cfg`
- Create: `demo-campaign.cfg` (at repo root — KSP loads it directly for manual testing)
- Modify: `README.md`

- [ ] **Step 1: Write `Examples/demo-campaign.cfg`**

Contents:
```cfg
// Demo campaign for KerbalCampaignKit.
// Shows chapters, triggers, reputation income, and notifications.

CAMPAIGN_CHAPTER
{
    id = 1
    name = First Steps
    description = Your space program is just getting off the ground.
    ENTRY_TRIGGER = demo_chapter_1_entry
}

CAMPAIGN_CHAPTER
{
    id = 2
    name = Into Orbit
    description = You are ready to reach Kerbin orbit.
    ENTRY_TRIGGER = demo_chapter_2_entry
}

CAMPAIGN_TRIGGER
{
    id = demo_chapter_1_entry
    once = true
    ON_EVENT
    {
        type = FacilityEntered
        facility = SpaceCenter
    }
    ACTIONS
    {
        ADVANCE_CHAPTER { target = 1 }
        NOTIFY
        {
            target = admin
            severity = info
            source = demo_welcome
        }
    }
}

REPUTATION_INCOME
{
    enabled = true
    intervalDays = 30
    TIER { min = 0;    max = 100;  income = 5000;  label = Startup }
    TIER { min = 100;  max = 250;  income = 15000; label = Established }
    TIER { min = 250;  max = 500;  income = 35000; label = Respected }
    TIER { min = 500;  max = 9999; income = 60000; label = Prestigious }
}

REPUTATION_DECAY
{
    enabled = true
    ratePercentPerMonth = 1.5
    resetOnContractComplete = true
    tierFloors = true
}
```

- [ ] **Step 2: Copy to root for easy in-game loading**

```bash
cp Examples/demo-campaign.cfg demo-campaign.cfg
```

- [ ] **Step 3: Update `README.md`**

Replace contents with:
```markdown
# KerbalCampaignKit

A utility library that extends [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit)
with a career-scale state machine: named narrative chapters, a general event→action
trigger system, a reputation economy, and a hierarchical notification API. Built for
KSP 1.12.x mods that want story-driven career progression.

## Status

Early development. v0.1.0 in progress.

## Features

- **Chapters** — named narrative state with entry/exit triggers, save-persisted
- **Triggers** — "when X happens, do Y" wiring with 18+ event types and 10 action types
- **Reputation economy** — opt-in tiered passive income, decay, gates, stakes
- **Notifications** — hierarchical attention markers with auto-clear rules
- **CC integration** — `InChapter`, `ChapterAtLeast`, `FlagEquals`, `FlagExpression`, `ReputationMinimum` REQUIREMENTs + `ReputationStake` BEHAVIOUR

## Requirements

- KSP 1.12.x
- [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit) v0.1.0 or later
- [ContractConfigurator](https://forum.kerbalspaceprogram.com/index.php?/topic/91625-112x-contract-configurator/) v2.12.0 or later

## Installation

Extract so the folder structure is:

```
GameData/
  KerbalCampaignKit/
    KerbalCampaignKit.version
    Plugins/
      KerbalCampaignKit.dll
```

## Quick Example

Define a chapter and a trigger in any `.cfg` file inside `GameData`:

```cfg
CAMPAIGN_CHAPTER
{
    id = 2
    name = Into Orbit
    ENTRY_TRIGGER = chapter_2_entry
}

CAMPAIGN_TRIGGER
{
    id = chapter_2_entry
    once = true
    ON_EVENT
    {
        type = ContractComplete
        contract = BKEX_UnmannedOrbit
    }
    ACTIONS
    {
        ADVANCE_CHAPTER { target = 2 }
        ENQUEUE_SCENE
        {
            sceneId = chapter_2_intro
            when = OnFacilityEnter
            facility = Administration
        }
    }
}
```

See `Examples/demo-campaign.cfg` for a more complete example.

## License

MIT — see `LICENSE`.
```

- [ ] **Step 4: Commit**

```bash
git add Examples/demo-campaign.cfg demo-campaign.cfg README.md
git commit -m "docs: demo campaign cfg and expanded README"
```

---

## Task 34: In-game manual verification

**Files:** none — this task runs KSP and inspects the result.

- [ ] **Step 1: Confirm install location**

The repo lives at `GameData/KerbalCampaignKit/`. The built DLL should already be at `GameData/KerbalCampaignKit/Plugins/KerbalCampaignKit.dll`. KDK should already be installed at `GameData/KerbalDialogueKit/Plugins/KerbalDialogueKit.dll`.

Verify:
```bash
ls "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalCampaignKit/Plugins/"
ls "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalDialogueKit/Plugins/"
```

Both should show their respective `.dll` files.

- [ ] **Step 2: Launch KSP and watch the log**

Start KSP via Steam. Watch `KSP.log` at the KSP install root for:
- No exceptions mentioning `KerbalCampaignKit`
- Message: `[ModuleManager] ... CAMPAIGN_CHAPTER: 2` (or similar) confirming cfg loaded
- No `ContractConfigurator` errors referencing our REQUIREMENTs

- [ ] **Step 3: Start a new career save and verify scenario loads**

1. Main Menu → Start Game → Career
2. Enter the Space Center
3. In `KSP.log`, search for `CampaignKitScenario` — should appear once during scenario module loading
4. Alt+F12 → Debug → Database → search for `CAMPAIGN_CHAPTER` and `CAMPAIGN_TRIGGER` — our demo nodes should be listed

- [ ] **Step 4: Verify chapter advance fires**

The `demo_chapter_1_entry` trigger fires on `FacilityEntered { facility = SpaceCenter }`. On entering Space Center:

- Save the game (Esc → Save)
- Open the saved persistent.sfs and find the `SCENARIO { name = CampaignKitScenario ... }` block
- Expect `currentChapter = 1` and a `CHAPTER_HISTORY / ENTERED { chapter = 1 ... }` entry

- [ ] **Step 5: Verify notification was created**

In the persistent.sfs scenario block, look for `NOTIFICATIONS { NOTIFICATION { target = admin; severity = Info; source = demo_welcome ... } }`. Expected: present after first Space Center entry.

- [ ] **Step 6: Verify reputation income fires**

1. Use Alt+F12 → Cheats → Set Reputation = 150
2. Advance game time by 30+ days (Alt+F12 → Time Control → advance days; or use a launch + warp)
3. Check `KSP.log` for `CampaignKitEvents.OnReputationIncomeTick` or verify Funds increased by ~15000 (Established tier)

- [ ] **Step 7: Verify CC REQUIREMENTs load without errors**

Create a small test contract in any `.cfg`:
```cfg
CONTRACT_TYPE
{
    name = KCKTestContract
    title = KCK Test: Chapter Gating
    description = Test contract for chapter gating.
    synopsis = Complete to pass the test.
    completedMessage = Pass.
    prestige = Trivial
    agent = World-Firsts

    PARAMETER
    {
        name = Time
        type = Duration
        duration = 0s
    }

    REQUIREMENT
    {
        name = InChapter
        type = InChapter
        chapter = 1
    }
}
```

Launch, check Mission Control, verify the contract appears in Available (because chapter = 1) and disappears after a manual ADVANCE_CHAPTER to 2 via the debug console.

- [ ] **Step 8: Report**

Summarize in a short note:
- Scenario loads: ✓/✗
- Chapter advance works: ✓/✗
- Notification created and persisted: ✓/✗
- Reputation income paid at interval: ✓/✗
- CC REQUIREMENTs load without errors: ✓/✗

If any step fails, investigate `KSP.log` for the specific exception and either fix or file an issue.

- [ ] **Step 9: Final commit**

```bash
git add -A
git commit -m "chore: verified end-to-end in-game with demo campaign" --allow-empty
```

Tag the release:
```bash
git tag -a v0.1.0 -m "Initial release: chapters, triggers, reputation, notifications"
```

---

## Wrap-up

At this point `KerbalCampaignKit.dll` is installed at `GameData/KerbalCampaignKit/Plugins/` and the mod is functional. Subsequent phases (BadgKatDirector framework and BadgKatCareer content) build on this foundation.

Publish steps (optional, do when ready):
1. Create remote `badgkat/KerbalCampaignKit` on GitHub
2. `git remote add origin ...` + `git push -u origin main`
3. `gh release create v0.1.0 Plugins/KerbalCampaignKit.dll`
4. Update KSP-AVC `DOWNLOAD` URL if needed

## Known follow-ups (post-v0.1)

These are declared in the EventType enum / spec but not implemented in Phase 1 tasks. Add them if a consumer trigger references one of these events:

- **KerbalEventSource** — subscribes to `GameEvents.onCrewKilled` and `GameEvents.OnCrewmemberLeveledUp` to publish `KerbalKilled` / `KerbalLeveled` records. Similar shape to `VesselEventSource`.
- **BodyDiscoveredEventSource** — uses reflection to hook ResearchBodies' discovery GameEvent so we don't take a hard dependency on that mod.
- **GameEventBridgeSource** — takes an eventName in cfg and uses `EventData<>.Find(name)` to subscribe at runtime, publishing a `GameEvent`-typed record when it fires.
- **FlagChanged auto-fire** — `SetFlagAction` / `ClearFlagAction` could fire `FlagEventSource.FireChanged` so triggers listening on `FlagChanged` work without the addon polling. One-line add: inject `FlagEventSource` into `ActionContext`, call it from flag actions after the store write.

None of these are required for the demo campaign in Task 33. Add when a trigger in real content needs them.
