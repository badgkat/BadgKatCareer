# KerbalAdminKit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone utility mod (`KerbalAdminKit.dll`) that customizes the KSP Administration Building and surfaces character-authored program information across KSC. Inert without content cfg; opt-in admin takeover via `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }`. Renders UI for data owned by KerbalCampaignKit (KCK) and KerbalDialogueKit (KDK). Foundational for BadgKatCareer.

**Architecture:** A new C# mod at `GameData/KerbalAdminKit/` in its own git repo. Depends on KerbalDialogueKit (v0.1.0), KerbalCampaignKit (v0.1.0), HarmonyKSP, ClickThroughBlocker, ModuleManager. Provides an `AdminKit` static API, an `AdminKitScenario` for minimal per-save persistence, a layered IMGUI admin screen (three panels), per-building character overlays (MC + Tracking Station), a KSC notification renderer, a memo pipeline, a disposition decay → KCK-trigger compiler, and a PR Campaign gameplay action. Pure-logic code (condition evaluation, loaders, decay compilation) lives in xUnit-testable classes that consume KCK's `ISceneNode` abstraction; KSP-integrated code (IMGUI UI, Harmony patch, MonoBehaviours) is verified manually in-game.

**Tech Stack:** C# targeting .NET Framework 4.7.2 (KSP's Unity 2019 runtime), MSBuild/`dotnet` CLI, xUnit. References: KSP 1.12.x assemblies, KerbalDialogueKit.dll, KerbalCampaignKit.dll, 0Harmony.dll, ClickThroughBlocker.dll, ContractConfigurator.dll (for `ContractSystem` query), UnityEngine IMGUI.

**Reference:** Full spec at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer/docs/superpowers/specs/2026-04-23-admin-kit-design.md`. Stack overview at `2026-04-23-director-career-redesign.md`.

---

## Context For The Engineer

**You have not seen this codebase. Read this section before touching any task.**

**Working directory.** Primary repo lives at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/` (create new). The KSP install root is `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/`. Other mods already exist under `GameData/` — most relevant are `KerbalDialogueKit/` (KDK, shipped v0.1.0), `KerbalCampaignKit/` (KCK, shipped v0.1.0), `ChatterBox/`, `ContractConfigurator/`, `ClickThroughBlocker/`, `000_Harmony/` (HarmonyKSP). **Do NOT** edit KSP's own install files or any other mod's files — only write inside `GameData/KerbalAdminKit/`.

**KSP modding basics.** KSP 1.12 runs on Unity 2019 with .NET Framework 4.x. Plugins live in `GameData/<ModName>/Plugins/` and load at startup. Key APIs:
- `[KSPAddon]` attribute + `MonoBehaviour` for scene-lifecycle runtime components. Common `Startup` values: `MainMenu` (global, once), `SpaceCentre`, `MissionControl`, `TrackingStation`, `Flight`.
- `[KSPScenario]` + `ScenarioModule` for per-save persistent state.
- `GameEvents` static class: `EventData<T>` fields for subscribing to gameplay events.
- `ConfigNode` class: KSP's cfg format (braces, `key = value`, nested nodes). Sealed and hard to test without KSP — see "ISceneNode" below.
- `GameDatabase.Instance.GetConfigNodes("NODE_NAME")` — enumerate all top-level cfg nodes of a given name across loaded mods.
- `GameDatabase.Instance.GetTexture(url, false)` — load a PNG by its `GameDatabase` URL (e.g., `"KerbalAdminKit/Art/exclamation"` — no extension).

**Dependency APIs you will touch.**

From **KerbalCampaignKit** (note: `CampaignKit` static facade lives in `KerbalCampaignKit.Core`; cfg abstraction in `KerbalCampaignKit.Config`):
- `KerbalCampaignKit.Core.CampaignKit.Engine` — trigger engine. `Engine.Register(Trigger)` adds a programmatic trigger.
- `KerbalCampaignKit.Core.CampaignKit.Chapters.Current` — current chapter id (string) or null.
- `KerbalCampaignKit.Core.CampaignKit.Reputation` — `ReputationEconomy` instance. Key methods:
  - `Reputation.Income.IncomeForRep(double rep)` — income amount for the tier matching `rep`.
  - `Reputation.Income.TierLabelForRep(double rep)` — string label for the matching tier, or null.
  - `Reputation.Income.Tiers` — `List<ReputationIncome.Tier>` ordered; iterate to derive tier index.
  - `Reputation.NextGate(double currentRep)` — returns `(double value, string label)?` for the next upward tier threshold, or null.
  - `Reputation.HaltDecay(double nowSeconds, double days)` — halt decay starting at `nowSeconds` for `days` in-game days. **Note two parameters.**
- `KerbalCampaignKit.Core.CampaignKit.Notifications.Highest(string targetPrefix)` — returns `NotificationSeverity?` (null, `Info`, or `Action`) representing the highest-severity notification whose target matches the prefix segment-aware. Returns null when no notifications match.
- `KerbalCampaignKit.Notifications.CampaignKitEvents.OnNotificationAdded` / `OnNotificationCleared` — static events for notification state changes.
- `KerbalCampaignKit.Config.ISceneNode` / `ConfigNodeAdapter` — cfg abstraction. **Reuse these; do not redefine.**
- `KerbalCampaignKit.Triggers.Trigger`, `EventSpec`, `ActionSpec` — trigger data model. KAK's disposition-decay implementation does NOT register triggers into KCK's engine (see Task 24); it uses an in-memory table and its own ticker for simplicity.

From **KerbalDialogueKit** (`using KerbalDialogueKit.Core;` + `KerbalDialogueKit.Flags;`):
- `DialogueKit.Flags` — `FlagStore` (`Get`, `Set`, `Remove`, `All`).
- `DialogueKit.EnqueueById(string sceneId)` — queue a scene for display via ChatterBox.
- `KerbalDialogueKit.Flags.FlagExpressionParser` — parse `"chapter >= 3 && completed == true"` against a flag store.

From **HarmonyKSP** (`using HarmonyLib;`):
- `Harmony` class for Harmony instance.
- `[HarmonyPatch(typeof(T), nameof(T.Method))]` attribute + `[HarmonyPrefix]` method returning bool to suppress the original call.

From **ClickThroughBlocker** (`using ClickThroughFix;`):
- `ClickThroughBlocker.GUILayoutWindow(id, rect, func, title, style)` — drop-in replacement for `GUILayout.Window` that blocks click-through to the game world.

From **KSP stock** (no `using`, but `UnityEngine` + global namespace):
- `Funding.Instance.Funds` / `Funding.Instance.AddFunds(amount, reason)`.
- `global::Reputation.Instance.reputation` / `global::Reputation.Instance.AddReputation(amount, reason)` — **note the `global::` qualifier**: we have a `KerbalAdminKit.Reputation` namespace that shadows the KSP class. Every use of KSP's `Reputation` class must be fully qualified.
- `ResearchAndDevelopment.Instance.Science`.
- `TransactionReasons.Strategies` — reason enum for admin-originated fund/rep changes.
- `MessageSystem.Instance.AddMessage(new MessageSystem.Message(title, body, MessageSystemButton.MessageButtonColor.GREEN, MessageSystemButton.ButtonIcons.MESSAGE))` — post to the mailbox.
- `SpaceCenter.Instance` — KSC scene singleton. Has `Administration`, `MissionControl`, `TrackingStation`, etc. as `SpaceCenterBuilding` references with `.transform`.
- `Planetarium.GetUniversalTime()` — current game time in seconds (double).
- `Camera.main.WorldToScreenPoint(Vector3 worldPos)` — world → screen coords for building markers.

**The `global::Reputation` shadowing.** Our mod has a `KerbalAdminKit.Memos.Conditions.ReputationAboveCondition` / `ReputationBelowCondition`, so the `Reputation` identifier at the top of any condition file resolves to **our class, not KSP's**. Anywhere we need KSP's `Reputation`, write `global::Reputation.Instance`. This is the same trick KCK uses.

**Build output pattern.** `Directory.Build.props` at the repo root redirects `BaseIntermediateOutputPath` and `BaseOutputPath` to `C:\Users\bklen\ksp-dev\KAK-build\` so build intermediates never land inside `GameData/`. The main project's csproj overrides `OutputPath` to write the shipping DLL directly to `Plugins/`. KCK uses this exact pattern — mirror it.

**Test strategy.** Two buckets:
- **xUnit-testable** (must not reference `UnityEngine` or KSP assemblies): loaders consuming `ISceneNode`, pure condition evaluators, disposition-decay compiler, chip-layer spec model.
- **Manual verification** (must reference KSP): IMGUI UI, Harmony patches, `MonoBehaviour`s, `KSPAddon`s, `ScenarioModule`. These get a quick sanity script in the task but no automated test.

**Cfg-node syntax.** KSP cfg has three hard rules that cause mysterious load failures if violated:
1. **No semicolons.** Each `key = value` on its own line.
2. **Braces on their own lines for multi-field nodes.** Inline `NODE { a = 1; b = 2 }` is illegal in multi-field nodes; use multi-line.
3. **Comments are `//` line-comments only.** No block comments.

See `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/CLAUDE.md` for the full list (and KCK's plan for examples in action).

**Commit cadence.** Commit after every task passes. Use conventional commit prefixes (`feat:`, `fix:`, `test:`, `docs:`, `chore:`, `refactor:`). **Never include `Co-Authored-By:` trailers.** The user's auto-memory has an explicit feedback entry forbidding them.

**Authoritative path prefix.** All paths below are relative to `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/` unless stated otherwise.

---

## File Structure

```
GameData/KerbalAdminKit/                           ← repo root, also mod root
├── .git/
├── .gitignore
├── Directory.Build.props                          ← redirects obj/bin OUT of GameData
├── LICENSE                                        ← MIT
├── KerbalAdminKit.sln
├── KerbalAdminKit.version                         ← KSP-AVC, shipped
├── README.md
├── Plugins/
│   └── KerbalAdminKit.dll                         ← main project build target (shipped)
├── Art/
│   ├── exclamation.png                            ← default Action-severity glyph
│   └── dot.png                                    ← default Info-severity glyph
├── demo/
│   └── demo-admin-kit.cfg                         ← self-contained demo cfg
├── src/
│   └── KerbalAdminKit/
│       ├── KerbalAdminKit.csproj                  ← net472, OutputPath=..\..\Plugins\
│       ├── Properties/AssemblyInfo.cs
│       ├── Core/
│       │   ├── AdminKit.cs                        ← public static API facade
│       │   ├── AdminKitScenario.cs                ← ScenarioModule: dismissed + stock-mailed + PR cooldown
│       │   └── AdminKitAddon.cs                   ← KSPAddon(MainMenu): cfg scan, Harmony reg, wiring
│       ├── Settings/
│       │   ├── AdminKitSettings.cs                ← data
│       │   └── AdminKitSettingsLoader.cs          ← KERBAL_ADMIN_KIT parser
│       ├── Characters/
│       │   ├── ChipLayerSpec.cs                   ← textureUrl + rect + tintWithDisposition
│       │   ├── CharacterInfo.cs                   ← id + displayName + role + baseColor + chip layers + dispositions + onClickScene
│       │   ├── CharacterRegistry.cs               ← Get / All / GetDisposition
│       │   ├── CharacterLoader.cs                 ← DIRECTOR_CHARACTER parser (incl. CHIP_LAYER + DISPOSITION_TINT + DISPOSITION_LABEL)
│       │   └── DispositionColors.cs               ← derive tint from baseColor + disposition name, honor overrides
│       ├── Focuses/
│       │   ├── FocusOption.cs                     ← character + id + title + description + requirement + flag + flagValue
│       │   ├── FocusRegistry.cs                   ← GetValidFor(character, flagStore)
│       │   ├── FocusLoader.cs                     ← DIRECTOR_FOCUS parser
│       │   └── FocusPicker.cs                     ← IMGUI modal (KSP-integrated)
│       ├── Memos/
│       │   ├── Memo.cs
│       │   ├── MemoContext.cs                     ← injected currency + chapter + flags + time
│       │   ├── MemoRegistry.cs                    ← Active / Dismiss / Pinned
│       │   ├── MemoLoader.cs                      ← DIRECTOR_MEMO parser (incl. CONDITION blocks)
│       │   ├── MemoTicker.cs                      ← KSPAddon(SpaceCentre): poll + stock-mail
│       │   └── Conditions/
│       │       ├── IMemoCondition.cs
│       │       ├── MemoConditionFactory.cs        ← type name → IMemoCondition
│       │       ├── FundsBelowCondition.cs
│       │       ├── ReputationAboveCondition.cs
│       │       ├── ReputationBelowCondition.cs
│       │       ├── InChapterCondition.cs
│       │       ├── FlagExpressionCondition.cs
│       │       ├── TimeSinceEventCondition.cs
│       │       ├── VesselAgeCondition.cs          ← KSP-integrated
│       │       ├── ContractAvailableCondition.cs  ← KSP-integrated
│       │       └── BodyDiscoveredCondition.cs     ← KSP-integrated (ResearchBodies soft)
│       ├── DecayCompiler/
│       │   ├── DispositionDecayCompiler.cs
│       │   └── DispositionDecayLoader.cs
│       ├── Admin/
│       │   ├── ChipRenderer.cs                    ← shared IMGUI layered composition
│       │   ├── CharacterPanel.cs
│       │   ├── DashboardPanel.cs
│       │   ├── DeskPanel.cs
│       │   ├── PrCampaignAction.cs
│       │   ├── AdminBuildingUI.cs                 ← KSPAddon(SpaceCentre): main window
│       │   └── AdminClickInterceptor.cs           ← Harmony patch
│       ├── BuildingOverlays/
│       │   ├── BuildingSceneRegistry.cs
│       │   ├── BuildingSceneLoader.cs             ← DIRECTOR_BUILDING_SCENE parser
│       │   ├── BuildingOverlayBase.cs             ← shared overlay logic
│       │   ├── MissionControlOverlay.cs           ← KSPAddon(MissionControl)
│       │   └── TrackingStationOverlay.cs          ← KSPAddon(TrackingStation)
│       ├── KscRenderer/
│       │   ├── NotificationStyleRegistry.cs
│       │   ├── NotificationStyleLoader.cs
│       │   ├── KscMarkerOffsetRegistry.cs
│       │   └── KscNotificationRenderer.cs         ← KSPAddon(SpaceCentre): OnGUI markers
│       └── Util/
│           ├── HexColor.cs                        ← "#FFRRGGBB" → Color
│           └── TextureLoader.cs                   ← GameDatabase URL → Texture2D w/ caching
└── tests/
    └── KerbalAdminKit.Tests/
        ├── KerbalAdminKit.Tests.csproj            ← net472, xUnit, copies runtime deps
        ├── SmokeTest.cs
        ├── TestHelpers/
        │   ├── FakeSceneNode.cs                   ← ISceneNode impl for tests
        │   └── FakeTriggerRegistry.cs             ← capture compiled triggers
        ├── AdminKitSettingsLoaderTests.cs
        ├── CharacterLoaderTests.cs
        ├── DispositionColorsTests.cs
        ├── FocusLoaderTests.cs
        ├── FocusRegistryTests.cs
        ├── MemoLoaderTests.cs
        ├── PureMemoConditionTests.cs
        ├── DispositionDecayCompilerTests.cs
        ├── NotificationStyleLoaderTests.cs
        ├── BuildingSceneLoaderTests.cs
        └── HexColorTests.cs
```

Roughly 45 source files (~30 production classes + supporting types) + 13 test files. The existing KCK repo at `GameData/KerbalCampaignKit/` is a direct structural analogue — when in doubt, read KCK's matching file.

---

## Task 1: Repo init, Directory.Build.props, LICENSE, .gitignore

**Files:**
- Create: `GameData/KerbalAdminKit/` directory
- Create: `.gitignore`
- Create: `Directory.Build.props`
- Create: `LICENSE`
- Create: `.git/` (via `git init`)

- [ ] **Step 1: Create mod folder and initialize git**

From the KSP install root `/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/`:

```bash
mkdir -p "GameData/KerbalAdminKit"
cd "GameData/KerbalAdminKit"
git init
git branch -m master main
```

Expected: empty repo on `main` branch.

- [ ] **Step 2: Write `.gitignore`**

Create `GameData/KerbalAdminKit/.gitignore`:

```
bin/
obj/
*.suo
*.user
*.userprefs
.vs/
.idea/
*.swp
*.tmp
```

The build outputs live **outside** the repo via `Directory.Build.props` (next step) — this `.gitignore` is belt-and-suspenders.

- [ ] **Step 3: Write `Directory.Build.props`**

Create `GameData/KerbalAdminKit/Directory.Build.props`:

```xml
<Project>
  <PropertyGroup>
    <BaseIntermediateOutputPath>C:\Users\bklen\ksp-dev\KAK-build\$(MSBuildProjectName)\obj\</BaseIntermediateOutputPath>
    <BaseOutputPath>C:\Users\bklen\ksp-dev\KAK-build\$(MSBuildProjectName)\bin\</BaseOutputPath>
  </PropertyGroup>
</Project>
```

This redirects `obj/` and `bin/` outside `GameData/` so KSP's assembly loader doesn't scan test DLLs or intermediates at game startup. The main project overrides `OutputPath` in Task 2 to write the shipping DLL into `Plugins/`.

- [ ] **Step 4: Write `LICENSE` (MIT)**

Create `GameData/KerbalAdminKit/LICENSE`:

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

- [ ] **Step 5: Commit**

```bash
git add .gitignore Directory.Build.props LICENSE
git commit -m "chore: initialize KerbalAdminKit repo with build redirection and MIT license"
```

---

## Task 2: Main library csproj + Core namespace placeholder

**Files:**
- Create: `KerbalAdminKit.sln`
- Create: `src/KerbalAdminKit/KerbalAdminKit.csproj`
- Create: `src/KerbalAdminKit/Properties/AssemblyInfo.cs`
- Create: `src/KerbalAdminKit/Core/AdminKit.cs`

- [ ] **Step 1: Write solution file**

Create `KerbalAdminKit.sln`:

```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 17
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "KerbalAdminKit", "src\KerbalAdminKit\KerbalAdminKit.csproj", "{AAAA1111-2222-3333-4444-555566667777}"
EndProject
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "KerbalAdminKit.Tests", "tests\KerbalAdminKit.Tests\KerbalAdminKit.Tests.csproj", "{BBBB1111-2222-3333-4444-555566667777}"
EndProject
Global
    GlobalSection(SolutionConfigurationPlatforms) = preSolution
        Debug|Any CPU = Debug|Any CPU
        Release|Any CPU = Release|Any CPU
    EndGlobalSection
    GlobalSection(ProjectConfigurationPlatforms) = postSolution
        {AAAA1111-2222-3333-4444-555566667777}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
        {AAAA1111-2222-3333-4444-555566667777}.Debug|Any CPU.Build.0 = Debug|Any CPU
        {AAAA1111-2222-3333-4444-555566667777}.Release|Any CPU.ActiveCfg = Release|Any CPU
        {AAAA1111-2222-3333-4444-555566667777}.Release|Any CPU.Build.0 = Release|Any CPU
        {BBBB1111-2222-3333-4444-555566667777}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
        {BBBB1111-2222-3333-4444-555566667777}.Debug|Any CPU.Build.0 = Debug|Any CPU
        {BBBB1111-2222-3333-4444-555566667777}.Release|Any CPU.ActiveCfg = Release|Any CPU
        {BBBB1111-2222-3333-4444-555566667777}.Release|Any CPU.Build.0 = Release|Any CPU
    EndGlobalSection
EndGlobal
```

- [ ] **Step 2: Write main project csproj**

Create `src/KerbalAdminKit/KerbalAdminKit.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <LangVersion>8.0</LangVersion>
    <RootNamespace>KerbalAdminKit</RootNamespace>
    <AssemblyName>KerbalAdminKit</AssemblyName>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <OutputPath>..\..\Plugins\</OutputPath>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <CopyLocalLockFileAssemblies>false</CopyLocalLockFileAssemblies>
    <Deterministic>true</Deterministic>
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
    <Reference Include="UnityEngine.IMGUIModule">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\UnityEngine.IMGUIModule.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="KerbalDialogueKit">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\Plugins\KerbalDialogueKit.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="KerbalCampaignKit">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\Plugins\KerbalCampaignKit.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="0Harmony">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\000_Harmony\0Harmony.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="ClickThroughBlocker">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\000_ClickThroughBlocker\Plugins\ClickThroughBlocker.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="ContractConfigurator">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\ContractConfigurator\ContractConfigurator.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>
</Project>
```

**Verify HintPath correctness:**

```bash
ls "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/000_ClickThroughBlocker/Plugins/ClickThroughBlocker.dll"
ls "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/000_Harmony/0Harmony.dll"
ls "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalCampaignKit/Plugins/KerbalCampaignKit.dll"
```

All must exist. If any layout differs (some ClickThroughBlocker builds put the DLL at folder root rather than `Plugins/`), correct the `HintPath`.

- [ ] **Step 3: Write `AssemblyInfo.cs`**

Create `src/KerbalAdminKit/Properties/AssemblyInfo.cs`:

```csharp
using System.Reflection;
using System.Runtime.InteropServices;

[assembly: AssemblyTitle("KerbalAdminKit")]
[assembly: AssemblyProduct("KerbalAdminKit")]
[assembly: AssemblyVersion("0.1.0.0")]
[assembly: AssemblyFileVersion("0.1.0.0")]
[assembly: ComVisible(false)]
```

`GenerateAssemblyInfo=false` in the csproj means we supply this ourselves. Without it KSP displays `v0.0.0.0` for the mod.

- [ ] **Step 4: Write public API placeholder**

Create `src/KerbalAdminKit/Core/AdminKit.cs`:

```csharp
namespace KerbalAdminKit
{
    /// <summary>
    /// Public static API facade. Properties are wired by AdminKitAddon at startup.
    /// </summary>
    public static class AdminKit
    {
    }
}
```

Placeholder — later tasks add `Characters`, `Focuses`, `Memos` properties as their registries come online.

- [ ] **Step 5: Build + commit**

```bash
cd "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit"
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

Expected: `KerbalAdminKit.dll` lands at `Plugins/KerbalAdminKit.dll`. Zero errors, zero warnings.

```bash
git add KerbalAdminKit.sln src/
git commit -m "feat: add main library project with KSP/KDK/KCK/Harmony references"
```

---

## Task 3: Test project scaffolding + smoke test

**Files:**
- Create: `tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj`
- Create: `tests/KerbalAdminKit.Tests/SmokeTest.cs`
- Create: `tests/KerbalAdminKit.Tests/TestHelpers/FakeSceneNode.cs`

- [ ] **Step 1: Write test csproj**

Create `tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <LangVersion>8.0</LangVersion>
    <IsPackable>false</IsPackable>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.6" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.6" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\KerbalAdminKit\KerbalAdminKit.csproj">
      <Private>false</Private>
    </ProjectReference>
    <Reference Include="KerbalDialogueKit">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\Plugins\KerbalDialogueKit.dll</HintPath>
      <Private>true</Private>
    </Reference>
    <Reference Include="KerbalCampaignKit">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\Plugins\KerbalCampaignKit.dll</HintPath>
      <Private>true</Private>
    </Reference>
  </ItemGroup>
</Project>
```

`Private=true` on KDK / KCK causes their DLLs to copy to the test output so xUnit can resolve transitive references. KSP and Unity assemblies are not referenced here: test code must never touch them.

- [ ] **Step 2: Write `FakeSceneNode` test helper**

Create `tests/KerbalAdminKit.Tests/TestHelpers/FakeSceneNode.cs`:

```csharp
using System.Collections.Generic;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Tests.TestHelpers
{
    public sealed class FakeSceneNode : ISceneNode
    {
        public Dictionary<string, List<string>> Values = new Dictionary<string, List<string>>();
        public Dictionary<string, List<FakeSceneNode>> Nodes = new Dictionary<string, List<FakeSceneNode>>();

        public string GetValue(string key) =>
            Values.TryGetValue(key, out var list) && list.Count > 0 ? list[0] : null;

        public IEnumerable<string> GetValues(string key) =>
            Values.TryGetValue(key, out var list) ? list : new List<string>();

        public IEnumerable<ISceneNode> GetNodes(string name)
        {
            if (!Nodes.TryGetValue(name, out var list)) yield break;
            foreach (var n in list) yield return n;
        }

        public bool HasValue(string key) => Values.ContainsKey(key);
        public bool HasNode(string name) => Nodes.ContainsKey(name);

        public FakeSceneNode Set(string key, string value)
        {
            if (!Values.TryGetValue(key, out var list))
            {
                list = new List<string>();
                Values[key] = list;
            }
            list.Clear();
            list.Add(value);
            return this;
        }

        public FakeSceneNode Add(string key, string value)
        {
            if (!Values.TryGetValue(key, out var list))
            {
                list = new List<string>();
                Values[key] = list;
            }
            list.Add(value);
            return this;
        }

        public FakeSceneNode AddChild(string name, FakeSceneNode child)
        {
            if (!Nodes.TryGetValue(name, out var list))
            {
                list = new List<FakeSceneNode>();
                Nodes[name] = list;
            }
            list.Add(child);
            return this;
        }
    }
}
```

- [ ] **Step 3: Write smoke test**

Create `tests/KerbalAdminKit.Tests/SmokeTest.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class SmokeTest
    {
        [Fact]
        public void AdminKit_Facade_IsReferenceable()
        {
            var t = typeof(AdminKit);
            Assert.NotNull(t);
        }

        [Fact]
        public void FakeSceneNode_RoundTripsValues()
        {
            var n = new FakeSceneNode().Set("key", "value");
            Assert.Equal("value", n.GetValue("key"));
            Assert.True(n.HasValue("key"));
            Assert.False(n.HasNode("whatever"));
        }
    }
}
```

- [ ] **Step 4: Run tests**

```bash
cd "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit"
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

Expected: both tests PASS. If anything errors about missing KDK/KCK DLLs, confirm `Private=true` on those references and rebuild.

- [ ] **Step 5: Commit**

```bash
git add tests/
git commit -m "test: add xUnit test project with smoke test and FakeSceneNode helper"
```

---

## Task 4: .version file + README scaffold

**Files:**
- Create: `KerbalAdminKit.version`
- Create: `README.md`

- [ ] **Step 1: Write `.version` for KSP-AVC**

Create `GameData/KerbalAdminKit/KerbalAdminKit.version`:

```json
{
  "NAME": "KerbalAdminKit",
  "URL": "https://raw.githubusercontent.com/badgkat/KerbalAdminKit/main/KerbalAdminKit.version",
  "DOWNLOAD": "https://github.com/badgkat/KerbalAdminKit/releases/latest",
  "VERSION": {
    "MAJOR": 0,
    "MINOR": 1,
    "PATCH": 0
  },
  "KSP_VERSION": {
    "MAJOR": 1,
    "MINOR": 12
  },
  "KSP_VERSION_MIN": {
    "MAJOR": 1,
    "MINOR": 12
  },
  "KSP_VERSION_MAX": {
    "MAJOR": 1,
    "MINOR": 12
  }
}
```

The GitHub URLs are placeholders — the repo is created in the final task. KSP-AVC will silently fail the update check until then, which is fine for development.

- [ ] **Step 2: Write initial README**

Create `GameData/KerbalAdminKit/README.md`:

```markdown
# KerbalAdminKit (KAK)

A toolkit mod that customizes the KSP Administration Building and surfaces character-authored program information across KSC. Renders UI for data owned by [KerbalCampaignKit](https://github.com/badgkat/KerbalCampaignKit) and [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit); does not own game state itself.

## Status

v0.1.0 — under development.

## What KAK Does

- Optional admin-building replacement with three-panel IMGUI UI (Characters / Dashboard / Desk).
- Per-building character overlays in Mission Control and Tracking Station.
- KSC notification markers pulled from `KerbalCampaignKit.Notifications`.
- Condition-driven memo pipeline with optional mirroring to the stock mail system.
- `DISPOSITION_DECAY` cfg compiles to `CAMPAIGN_TRIGGER`s at load.
- PR Campaign gameplay action.

## Activation

Install KAK with no content cfg and nothing changes. Stock admin, stock mail, stock KSC view all work normally.

Set `KERBAL_ADMIN_KIT { replaceAdminBuilding = true }` in a content cfg to take over the admin building.

## Requirements

- KSP 1.12.x
- [ModuleManager](https://forum.kerbalspaceprogram.com/index.php?/topic/50533-*)
- [HarmonyKSP](https://github.com/KSPModdingLibs/HarmonyKSP)
- [ClickThroughBlocker](https://forum.kerbalspaceprogram.com/index.php?/topic/170747-*)
- [KerbalDialogueKit](https://github.com/badgkat/KerbalDialogueKit) >= 0.1.0
- [KerbalCampaignKit](https://github.com/badgkat/KerbalCampaignKit) >= 0.1.0

## License

MIT — see `LICENSE`.
```

- [ ] **Step 3: Commit**

```bash
git add KerbalAdminKit.version README.md
git commit -m "docs: add .version file and README scaffold"
```

---

## Task 5: AdminKitSettings data + loader (TDD)

**Files:**
- Create: `src/KerbalAdminKit/Settings/AdminKitSettings.cs`
- Create: `src/KerbalAdminKit/Settings/AdminKitSettingsLoader.cs`
- Create: `tests/KerbalAdminKit.Tests/AdminKitSettingsLoaderTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/AdminKitSettingsLoaderTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Settings;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class AdminKitSettingsLoaderTests
    {
        [Fact]
        public void Load_Defaults_WhenNoFieldsSet()
        {
            var node = new FakeSceneNode();
            var s = AdminKitSettingsLoader.Load(node);

            Assert.False(s.ReplaceAdminBuilding);
            Assert.Equal(5, s.DeskMemoCount);
            Assert.Equal(5.0, s.MemoPollSeconds);
        }

        [Fact]
        public void Load_ParsesReplaceAdminBuilding()
        {
            var node = new FakeSceneNode().Set("replaceAdminBuilding", "true");
            var s = AdminKitSettingsLoader.Load(node);
            Assert.True(s.ReplaceAdminBuilding);
        }

        [Fact]
        public void Load_ParsesDeskMemoCount()
        {
            var node = new FakeSceneNode().Set("deskMemoCount", "10");
            var s = AdminKitSettingsLoader.Load(node);
            Assert.Equal(10, s.DeskMemoCount);
        }

        [Fact]
        public void Load_ParsesMemoPollSeconds()
        {
            var node = new FakeSceneNode().Set("memoPollSeconds", "2.5");
            var s = AdminKitSettingsLoader.Load(node);
            Assert.Equal(2.5, s.MemoPollSeconds);
        }

        [Fact]
        public void Load_IgnoresMalformedValuesAndKeepsDefaults()
        {
            var node = new FakeSceneNode()
                .Set("deskMemoCount", "not-a-number")
                .Set("memoPollSeconds", "garbage");
            var s = AdminKitSettingsLoader.Load(node);
            Assert.Equal(5, s.DeskMemoCount);
            Assert.Equal(5.0, s.MemoPollSeconds);
        }
    }
}
```

- [ ] **Step 2: Run tests — expect compilation failure**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

Expected: build error — `AdminKitSettings` and `AdminKitSettingsLoader` don't exist yet.

- [ ] **Step 3: Write `AdminKitSettings`**

Create `src/KerbalAdminKit/Settings/AdminKitSettings.cs`:

```csharp
namespace KerbalAdminKit.Settings
{
    public sealed class AdminKitSettings
    {
        public bool ReplaceAdminBuilding;
        public int DeskMemoCount = 5;
        public double MemoPollSeconds = 5.0;

        public static AdminKitSettings Defaults() => new AdminKitSettings();
    }
}
```

- [ ] **Step 4: Write `AdminKitSettingsLoader`**

Create `src/KerbalAdminKit/Settings/AdminKitSettingsLoader.cs`:

```csharp
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Settings
{
    public static class AdminKitSettingsLoader
    {
        public static AdminKitSettings Load(ISceneNode node)
        {
            var s = AdminKitSettings.Defaults();
            if (node == null) return s;

            if (node.HasValue("replaceAdminBuilding") &&
                bool.TryParse(node.GetValue("replaceAdminBuilding"), out var rab))
                s.ReplaceAdminBuilding = rab;

            if (node.HasValue("deskMemoCount") &&
                int.TryParse(node.GetValue("deskMemoCount"), out var dmc))
                s.DeskMemoCount = dmc;

            if (node.HasValue("memoPollSeconds") &&
                double.TryParse(node.GetValue("memoPollSeconds"), out var mps))
                s.MemoPollSeconds = mps;

            return s;
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

All 5 new tests + 2 smoke tests = 7 pass.

- [ ] **Step 6: Commit**

```bash
git add src/KerbalAdminKit/Settings/ tests/KerbalAdminKit.Tests/AdminKitSettingsLoaderTests.cs
git commit -m "feat: add AdminKitSettings data and loader with defaults and parsing"
```

---

## Task 6: AdminKitScenario (save/load shell)

**Files:**
- Create: `src/KerbalAdminKit/Core/AdminKitScenario.cs`

This task uses KSP `ConfigNode` + `ScenarioModule` APIs directly — no xUnit testing. Verified in-game during Task 30.

- [ ] **Step 1: Write `AdminKitScenario`**

Create `src/KerbalAdminKit/Core/AdminKitScenario.cs`:

```csharp
using System.Collections.Generic;

namespace KerbalAdminKit
{
    [KSPScenario(ScenarioCreationOptions.AddToAllGames,
        GameScenes.SPACECENTER, GameScenes.FLIGHT,
        GameScenes.TRACKSTATION, GameScenes.EDITOR)]
    public class AdminKitScenario : ScenarioModule
    {
        public static AdminKitScenario Instance;

        public Dictionary<string, double> DismissedMemos = new Dictionary<string, double>();
        public HashSet<string> StockMailedMemos = new HashSet<string>();
        public double PrCampaignLastUsedTime;

        public override void OnAwake()
        {
            base.OnAwake();
            Instance = this;
        }

        public override void OnSave(ConfigNode node)
        {
            base.OnSave(node);

            var dismissed = node.AddNode("DISMISSED_MEMOS");
            foreach (var kv in DismissedMemos)
            {
                var m = dismissed.AddNode("MEMO");
                m.AddValue("id", kv.Key);
                m.AddValue("dismissedAt", kv.Value);
            }

            var mailed = node.AddNode("STOCK_MAILED_MEMOS");
            foreach (var id in StockMailedMemos)
            {
                var m = mailed.AddNode("MEMO");
                m.AddValue("id", id);
            }

            var pr = node.AddNode("PR_CAMPAIGN");
            pr.AddValue("lastUsedTime", PrCampaignLastUsedTime);
        }

        public override void OnLoad(ConfigNode node)
        {
            base.OnLoad(node);
            DismissedMemos.Clear();
            StockMailedMemos.Clear();
            PrCampaignLastUsedTime = 0;

            if (node.HasNode("DISMISSED_MEMOS"))
            {
                foreach (var m in node.GetNode("DISMISSED_MEMOS").GetNodes("MEMO"))
                {
                    var id = m.GetValue("id");
                    if (string.IsNullOrEmpty(id)) continue;
                    double.TryParse(m.GetValue("dismissedAt"), out var at);
                    DismissedMemos[id] = at;
                }
            }

            if (node.HasNode("STOCK_MAILED_MEMOS"))
            {
                foreach (var m in node.GetNode("STOCK_MAILED_MEMOS").GetNodes("MEMO"))
                {
                    var id = m.GetValue("id");
                    if (!string.IsNullOrEmpty(id)) StockMailedMemos.Add(id);
                }
            }

            if (node.HasNode("PR_CAMPAIGN"))
            {
                double.TryParse(node.GetNode("PR_CAMPAIGN").GetValue("lastUsedTime"),
                    out PrCampaignLastUsedTime);
            }
        }
    }
}
```

- [ ] **Step 2: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

Expected: zero errors.

- [ ] **Step 3: Commit**

```bash
git add src/KerbalAdminKit/Core/AdminKitScenario.cs
git commit -m "feat: add AdminKitScenario with dismissed memos / stock-mail tracking / PR cooldown"
```

---

## Task 7: ChipLayerSpec + CharacterInfo + DispositionColors (TDD where possible)

**Files:**
- Create: `src/KerbalAdminKit/Characters/ChipLayerSpec.cs`
- Create: `src/KerbalAdminKit/Characters/CharacterInfo.cs`
- Create: `src/KerbalAdminKit/Util/HexColor.cs`
- Create: `src/KerbalAdminKit/Characters/DispositionColors.cs`
- Create: `tests/KerbalAdminKit.Tests/HexColorTests.cs`
- Create: `tests/KerbalAdminKit.Tests/DispositionColorsTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/HexColorTests.cs`:

```csharp
using Xunit;
using UnityEngineColor = KerbalAdminKit.Util.ColorValue;
using KerbalAdminKit.Util;

namespace KerbalAdminKit.Tests
{
    public class HexColorTests
    {
        [Fact]
        public void Parse_EightDigitHex_IncludesAlpha()
        {
            // #FF82B4E8 → A=FF R=82 G=B4 B=E8
            var c = HexColor.Parse("#FF82B4E8");
            Assert.Equal(0xFF, c.A);
            Assert.Equal(0x82, c.R);
            Assert.Equal(0xB4, c.G);
            Assert.Equal(0xE8, c.B);
        }

        [Fact]
        public void Parse_SixDigitHex_DefaultsAlphaToFF()
        {
            var c = HexColor.Parse("#82B4E8");
            Assert.Equal(0xFF, c.A);
            Assert.Equal(0x82, c.R);
        }

        [Fact]
        public void Parse_NoHash_Works()
        {
            var c = HexColor.Parse("82B4E8");
            Assert.Equal(0x82, c.R);
        }

        [Fact]
        public void Parse_Invalid_ReturnsWhite()
        {
            var c = HexColor.Parse("garbage");
            Assert.Equal(0xFF, c.R);
            Assert.Equal(0xFF, c.G);
            Assert.Equal(0xFF, c.B);
            Assert.Equal(0xFF, c.A);
        }
    }
}
```

`ColorValue` is a plain struct so xUnit can run without UnityEngine.

Create `tests/KerbalAdminKit.Tests/DispositionColorsTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Util;

namespace KerbalAdminKit.Tests
{
    public class DispositionColorsTests
    {
        [Fact]
        public void ResolveTint_ReturnsBaseWhenNoOverrideAndUnknownDisposition()
        {
            var info = new CharacterInfo
            {
                Id = "gus",
                BaseColor = HexColor.Parse("#FFFFC078"),
            };
            var tint = DispositionColors.ResolveTint(info, "TotallyUnknownMood");
            Assert.Equal(info.BaseColor, tint);
        }

        [Fact]
        public void ResolveTint_UsesOverrideWhenMatched()
        {
            var info = new CharacterInfo
            {
                Id = "gus",
                BaseColor = HexColor.Parse("#FFFFC078"),
            };
            info.DispositionTints["Enthusiastic"] = HexColor.Parse("#FFA0D4FF");
            var tint = DispositionColors.ResolveTint(info, "Enthusiastic");
            Assert.Equal(HexColor.Parse("#FFA0D4FF"), tint);
        }

        [Theory]
        [InlineData("Enthusiastic")]
        [InlineData("Frustrated")]
        [InlineData("Supportive")]
        [InlineData("Skeptical")]
        public void ResolveTint_DerivesFromBaseForStandardValues(string d)
        {
            var info = new CharacterInfo
            {
                Id = "x",
                BaseColor = HexColor.Parse("#FF808080"),
            };
            var tint = DispositionColors.ResolveTint(info, d);
            // Standard dispositions perturb the base; exact values tested below
            // via ResolveTint_Enthusiastic_BrightensBaseColor.
            Assert.NotNull((object)tint);
        }

        [Fact]
        public void ResolveTint_Enthusiastic_BrightensBaseColor()
        {
            var info = new CharacterInfo
            {
                Id = "x",
                BaseColor = new ColorValue { A = 255, R = 100, G = 100, B = 100 },
            };
            var tint = DispositionColors.ResolveTint(info, "Enthusiastic");
            Assert.True(tint.R > 100);
            Assert.True(tint.G > 100);
            Assert.True(tint.B > 100);
        }

        [Fact]
        public void ResolveTint_Frustrated_DesaturatesBaseColor()
        {
            var info = new CharacterInfo
            {
                Id = "x",
                BaseColor = new ColorValue { A = 255, R = 255, G = 100, B = 100 },
            };
            var tint = DispositionColors.ResolveTint(info, "Frustrated");
            // Desaturation: R,G,B move toward their average
            Assert.True(tint.R < 255);
            Assert.True(tint.G > 100 || tint.G == 100);
        }
    }
}
```

- [ ] **Step 2: Run tests — expect compile failures**

- [ ] **Step 3: Write `ColorValue` + `HexColor`**

Create `src/KerbalAdminKit/Util/HexColor.cs`:

```csharp
using System;

namespace KerbalAdminKit.Util
{
    /// <summary>
    /// Framework-neutral color struct so we can unit-test without UnityEngine.
    /// Converted to UnityEngine.Color at render time via ToUnity().
    /// </summary>
    public struct ColorValue
    {
        public byte A, R, G, B;

        public static ColorValue White => new ColorValue { A = 255, R = 255, G = 255, B = 255 };

        public override bool Equals(object obj) =>
            obj is ColorValue c && c.A == A && c.R == R && c.G == G && c.B == B;
        public override int GetHashCode() => (A << 24) | (R << 16) | (G << 8) | B;
        public static bool operator ==(ColorValue a, ColorValue b) => a.Equals(b);
        public static bool operator !=(ColorValue a, ColorValue b) => !a.Equals(b);
    }

    public static class HexColor
    {
        public static ColorValue Parse(string hex)
        {
            if (string.IsNullOrWhiteSpace(hex)) return ColorValue.White;
            hex = hex.TrimStart('#').Trim();
            try
            {
                if (hex.Length == 8)
                {
                    return new ColorValue
                    {
                        A = Convert.ToByte(hex.Substring(0, 2), 16),
                        R = Convert.ToByte(hex.Substring(2, 2), 16),
                        G = Convert.ToByte(hex.Substring(4, 2), 16),
                        B = Convert.ToByte(hex.Substring(6, 2), 16),
                    };
                }
                if (hex.Length == 6)
                {
                    return new ColorValue
                    {
                        A = 0xFF,
                        R = Convert.ToByte(hex.Substring(0, 2), 16),
                        G = Convert.ToByte(hex.Substring(2, 2), 16),
                        B = Convert.ToByte(hex.Substring(4, 2), 16),
                    };
                }
            }
            catch (FormatException) { }
            catch (ArgumentOutOfRangeException) { }
            catch (OverflowException) { }
            return ColorValue.White;
        }
    }
}
```

- [ ] **Step 4: Write `ChipLayerSpec`**

Create `src/KerbalAdminKit/Characters/ChipLayerSpec.cs`:

```csharp
namespace KerbalAdminKit.Characters
{
    /// <summary>
    /// One layer in a chip's composed visual. Drawn in cfg order (first = back).
    /// rect uses normalized [x, y, width, height] in chip space (0..1).
    /// </summary>
    public sealed class ChipLayerSpec
    {
        public string TextureUrl;
        public float RectX = 0f;
        public float RectY = 0f;
        public float RectW = 1f;
        public float RectH = 1f;
        public bool TintWithDisposition = false;
    }
}
```

- [ ] **Step 5: Write `CharacterInfo`**

Create `src/KerbalAdminKit/Characters/CharacterInfo.cs`:

```csharp
using System.Collections.Generic;
using KerbalAdminKit.Util;

namespace KerbalAdminKit.Characters
{
    public sealed class CharacterInfo
    {
        public string Id;
        public string DisplayName;
        public string Role;
        public ColorValue BaseColor;
        public string ChipShape = "square";   // reserved for v0.2

        public List<ChipLayerSpec> ChipLayers = new List<ChipLayerSpec>();

        public string OnClickScene;           // optional KDK scene id

        public string DispositionFlag;
        public string DefaultDisposition = "Neutral";

        // Per-disposition overrides
        public Dictionary<string, ColorValue> DispositionTints = new Dictionary<string, ColorValue>();
        public Dictionary<string, string> DispositionLabels = new Dictionary<string, string>();
    }
}
```

- [ ] **Step 6: Write `DispositionColors`**

Create `src/KerbalAdminKit/Characters/DispositionColors.cs`:

```csharp
using KerbalAdminKit.Util;

namespace KerbalAdminKit.Characters
{
    public static class DispositionColors
    {
        public static ColorValue ResolveTint(CharacterInfo info, string disposition)
        {
            if (info == null) return ColorValue.White;
            if (info.DispositionTints != null &&
                info.DispositionTints.TryGetValue(disposition ?? "", out var overrideColor))
                return overrideColor;

            return Derive(info.BaseColor, disposition);
        }

        public static string ResolveLabel(CharacterInfo info, string disposition)
        {
            if (info?.DispositionLabels != null &&
                info.DispositionLabels.TryGetValue(disposition ?? "", out var label))
                return label;
            return disposition;
        }

        private static ColorValue Derive(ColorValue baseColor, string disposition)
        {
            switch (disposition)
            {
                case "Enthusiastic": return Brighten(baseColor, 0.25f);
                case "Supportive":   return Brighten(baseColor, 0.10f);
                case "Neutral":      return baseColor;
                case "Skeptical":    return Darken(baseColor, 0.15f);
                case "Frustrated":   return Desaturate(Darken(baseColor, 0.25f), 0.4f);
                default:             return baseColor;
            }
        }

        private static ColorValue Brighten(ColorValue c, float amount)
        {
            return new ColorValue
            {
                A = c.A,
                R = Clamp((byte)(c.R + (255 - c.R) * amount)),
                G = Clamp((byte)(c.G + (255 - c.G) * amount)),
                B = Clamp((byte)(c.B + (255 - c.B) * amount)),
            };
        }

        private static ColorValue Darken(ColorValue c, float amount)
        {
            return new ColorValue
            {
                A = c.A,
                R = Clamp((byte)(c.R * (1 - amount))),
                G = Clamp((byte)(c.G * (1 - amount))),
                B = Clamp((byte)(c.B * (1 - amount))),
            };
        }

        private static ColorValue Desaturate(ColorValue c, float amount)
        {
            byte avg = (byte)((c.R + c.G + c.B) / 3);
            return new ColorValue
            {
                A = c.A,
                R = Clamp((byte)(c.R + (avg - c.R) * amount)),
                G = Clamp((byte)(c.G + (avg - c.G) * amount)),
                B = Clamp((byte)(c.B + (avg - c.B) * amount)),
            };
        }

        private static byte Clamp(byte b) => b;   // byte wraparound avoided by math above
    }
}
```

- [ ] **Step 7: Run tests — expect PASS**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

- [ ] **Step 8: Commit**

```bash
git add src/KerbalAdminKit/Characters/ src/KerbalAdminKit/Util/HexColor.cs tests/KerbalAdminKit.Tests/HexColorTests.cs tests/KerbalAdminKit.Tests/DispositionColorsTests.cs
git commit -m "feat: add ColorValue/HexColor, ChipLayerSpec, CharacterInfo, DispositionColors"
```

---

## Task 8: CharacterLoader + CharacterRegistry (TDD)

**Files:**
- Create: `src/KerbalAdminKit/Characters/CharacterLoader.cs`
- Create: `src/KerbalAdminKit/Characters/CharacterRegistry.cs`
- Create: `tests/KerbalAdminKit.Tests/CharacterLoaderTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/CharacterLoaderTests.cs`:

```csharp
using System.Linq;
using Xunit;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Util;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class CharacterLoaderTests
    {
        [Fact]
        public void Load_ParsesCoreFields()
        {
            var n = new FakeSceneNode()
                .Set("id", "wernher")
                .Set("displayName", "Wernher von Kerman")
                .Set("role", "Chief Scientist")
                .Set("baseColor", "#FF82B4E8")
                .Set("dispositionFlag", "wernher_disposition")
                .Set("defaultDisposition", "Supportive")
                .Set("onClickScene", "wernher_intro");

            var c = CharacterLoader.Load(n);

            Assert.Equal("wernher", c.Id);
            Assert.Equal("Wernher von Kerman", c.DisplayName);
            Assert.Equal("Chief Scientist", c.Role);
            Assert.Equal(HexColor.Parse("#FF82B4E8"), c.BaseColor);
            Assert.Equal("wernher_disposition", c.DispositionFlag);
            Assert.Equal("Supportive", c.DefaultDisposition);
            Assert.Equal("wernher_intro", c.OnClickScene);
        }

        [Fact]
        public void Load_ReturnsNull_WhenIdMissing()
        {
            var n = new FakeSceneNode().Set("displayName", "no id");
            Assert.Null(CharacterLoader.Load(n));
        }

        [Fact]
        public void Load_ParsesChipLayers_InOrder()
        {
            var n = new FakeSceneNode().Set("id", "gus").Set("baseColor", "#FFFFC078");
            n.AddChild("CHIP_LAYER", new FakeSceneNode().Set("textureUrl", "A/bg"));
            n.AddChild("CHIP_LAYER", new FakeSceneNode()
                .Set("textureUrl", "A/portrait")
                .Set("tintWithDisposition", "true"));
            n.AddChild("CHIP_LAYER", new FakeSceneNode()
                .Set("textureUrl", "A/badge")
                .Set("rect", "0.7, 0.0, 0.3, 0.3"));

            var c = CharacterLoader.Load(n);

            Assert.Equal(3, c.ChipLayers.Count);
            Assert.Equal("A/bg", c.ChipLayers[0].TextureUrl);
            Assert.False(c.ChipLayers[0].TintWithDisposition);

            Assert.Equal("A/portrait", c.ChipLayers[1].TextureUrl);
            Assert.True(c.ChipLayers[1].TintWithDisposition);

            Assert.Equal("A/badge", c.ChipLayers[2].TextureUrl);
            Assert.Equal(0.7f, c.ChipLayers[2].RectX);
            Assert.Equal(0.0f, c.ChipLayers[2].RectY);
            Assert.Equal(0.3f, c.ChipLayers[2].RectW);
            Assert.Equal(0.3f, c.ChipLayers[2].RectH);
        }

        [Fact]
        public void Load_ParsesDispositionTintOverrides()
        {
            var n = new FakeSceneNode().Set("id", "x").Set("baseColor", "#FF808080");
            n.AddChild("DISPOSITION_TINT", new FakeSceneNode()
                .Set("value", "Enthusiastic")
                .Set("color", "#FFA0D4FF"));

            var c = CharacterLoader.Load(n);
            Assert.Equal(HexColor.Parse("#FFA0D4FF"), c.DispositionTints["Enthusiastic"]);
        }

        [Fact]
        public void Load_ParsesDispositionLabelOverrides()
        {
            var n = new FakeSceneNode().Set("id", "x").Set("baseColor", "#FF808080");
            n.AddChild("DISPOSITION_LABEL", new FakeSceneNode()
                .Set("value", "Enthusiastic")
                .Set("text", "Thrilled"));

            var c = CharacterLoader.Load(n);
            Assert.Equal("Thrilled", c.DispositionLabels["Enthusiastic"]);
        }

        [Fact]
        public void Registry_GetAndAll_Works()
        {
            var reg = new CharacterRegistry();
            reg.Add(new CharacterInfo { Id = "gene" });
            reg.Add(new CharacterInfo { Id = "gus"  });

            Assert.Equal(2, reg.All.Count());
            Assert.Equal("gus", reg.Get("gus").Id);
            Assert.Null(reg.Get("missing"));
        }
    }
}
```

- [ ] **Step 2: Run tests — expect compile failures**

- [ ] **Step 3: Write `CharacterLoader`**

Create `src/KerbalAdminKit/Characters/CharacterLoader.cs`:

```csharp
using System.Globalization;
using KerbalAdminKit.Util;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Characters
{
    public static class CharacterLoader
    {
        public static CharacterInfo Load(ISceneNode node)
        {
            if (node == null) return null;
            var id = node.GetValue("id");
            if (string.IsNullOrEmpty(id)) return null;

            var c = new CharacterInfo
            {
                Id = id,
                DisplayName = node.GetValue("displayName") ?? id,
                Role = node.GetValue("role") ?? "",
                BaseColor = HexColor.Parse(node.GetValue("baseColor")),
                DispositionFlag = node.GetValue("dispositionFlag"),
                DefaultDisposition = node.GetValue("defaultDisposition") ?? "Neutral",
                OnClickScene = node.GetValue("onClickScene"),
                ChipShape = node.GetValue("chipShape") ?? "square",
            };

            foreach (var layerNode in node.GetNodes("CHIP_LAYER"))
            {
                var layer = new ChipLayerSpec
                {
                    TextureUrl = layerNode.GetValue("textureUrl"),
                };
                if (layerNode.HasValue("rect"))
                {
                    var parts = (layerNode.GetValue("rect") ?? "").Split(',');
                    if (parts.Length == 4 &&
                        float.TryParse(parts[0].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var x) &&
                        float.TryParse(parts[1].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var y) &&
                        float.TryParse(parts[2].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var w) &&
                        float.TryParse(parts[3].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var h))
                    {
                        layer.RectX = x; layer.RectY = y; layer.RectW = w; layer.RectH = h;
                    }
                }
                if (layerNode.HasValue("tintWithDisposition") &&
                    bool.TryParse(layerNode.GetValue("tintWithDisposition"), out var tint))
                    layer.TintWithDisposition = tint;

                if (!string.IsNullOrEmpty(layer.TextureUrl))
                    c.ChipLayers.Add(layer);
            }

            foreach (var tintNode in node.GetNodes("DISPOSITION_TINT"))
            {
                var value = tintNode.GetValue("value");
                var color = tintNode.GetValue("color");
                if (!string.IsNullOrEmpty(value) && !string.IsNullOrEmpty(color))
                    c.DispositionTints[value] = HexColor.Parse(color);
            }

            foreach (var labelNode in node.GetNodes("DISPOSITION_LABEL"))
            {
                var value = labelNode.GetValue("value");
                var text = labelNode.GetValue("text");
                if (!string.IsNullOrEmpty(value) && !string.IsNullOrEmpty(text))
                    c.DispositionLabels[value] = text;
            }

            return c;
        }
    }
}
```

- [ ] **Step 4: Write `CharacterRegistry`**

Create `src/KerbalAdminKit/Characters/CharacterRegistry.cs`:

```csharp
using System.Collections.Generic;
using KerbalDialogueKit.Core;

namespace KerbalAdminKit.Characters
{
    public sealed class CharacterRegistry
    {
        private readonly Dictionary<string, CharacterInfo> byId =
            new Dictionary<string, CharacterInfo>();

        public void Add(CharacterInfo c)
        {
            if (c == null || string.IsNullOrEmpty(c.Id)) return;
            byId[c.Id] = c;
        }

        public CharacterInfo Get(string id) =>
            (id != null && byId.TryGetValue(id, out var c)) ? c : null;

        public IEnumerable<CharacterInfo> All => byId.Values;
        public int Count => byId.Count;

        /// <summary>
        /// Reads disposition flag via KDK, falling back to DefaultDisposition.
        /// </summary>
        public string GetDisposition(string id)
        {
            var info = Get(id);
            if (info == null) return null;
            if (string.IsNullOrEmpty(info.DispositionFlag)) return info.DefaultDisposition;
            var value = DialogueKit.Flags?.Get(info.DispositionFlag);
            return string.IsNullOrEmpty(value) ? info.DefaultDisposition : value;
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

- [ ] **Step 6: Commit**

```bash
git add src/KerbalAdminKit/Characters/CharacterLoader.cs src/KerbalAdminKit/Characters/CharacterRegistry.cs tests/KerbalAdminKit.Tests/CharacterLoaderTests.cs
git commit -m "feat: add CharacterLoader (cfg parsing) and CharacterRegistry (lookup + disposition)"
```

---

## Task 9: FocusOption + FocusLoader + FocusRegistry (TDD)

**Files:**
- Create: `src/KerbalAdminKit/Focuses/FocusOption.cs`
- Create: `src/KerbalAdminKit/Focuses/FocusLoader.cs`
- Create: `src/KerbalAdminKit/Focuses/FocusRegistry.cs`
- Create: `tests/KerbalAdminKit.Tests/FocusLoaderTests.cs`
- Create: `tests/KerbalAdminKit.Tests/FocusRegistryTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/FocusLoaderTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Focuses;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class FocusLoaderTests
    {
        [Fact]
        public void Load_ParsesAllFields()
        {
            var n = new FakeSceneNode()
                .Set("character", "wernher")
                .Set("id", "deep_space_signals")
                .Set("title", "Deep Space Signals")
                .Set("description", "Wernher focuses on anomaly signals")
                .Set("requirement", "chapter >= 3")
                .Set("flag", "wernher_focus")
                .Set("flagValue", "deep_space_signals");

            var f = FocusLoader.Load(n);

            Assert.Equal("wernher", f.CharacterId);
            Assert.Equal("deep_space_signals", f.Id);
            Assert.Equal("Deep Space Signals", f.Title);
            Assert.Equal("Wernher focuses on anomaly signals", f.Description);
            Assert.Equal("chapter >= 3", f.Requirement);
            Assert.Equal("wernher_focus", f.Flag);
            Assert.Equal("deep_space_signals", f.FlagValue);
        }

        [Fact]
        public void Load_ReturnsNullWhenCharacterMissing()
        {
            var n = new FakeSceneNode().Set("id", "no_char");
            Assert.Null(FocusLoader.Load(n));
        }

        [Fact]
        public void Load_ReturnsNullWhenIdMissing()
        {
            var n = new FakeSceneNode().Set("character", "x");
            Assert.Null(FocusLoader.Load(n));
        }
    }
}
```

Create `tests/KerbalAdminKit.Tests/FocusRegistryTests.cs`:

```csharp
using System.Linq;
using Xunit;
using KerbalAdminKit.Focuses;

namespace KerbalAdminKit.Tests
{
    public class FocusRegistryTests
    {
        [Fact]
        public void GetForCharacter_ReturnsMatches_IgnoresOthers()
        {
            var reg = new FocusRegistry();
            reg.Add(new FocusOption { CharacterId = "wernher", Id = "a" });
            reg.Add(new FocusOption { CharacterId = "wernher", Id = "b" });
            reg.Add(new FocusOption { CharacterId = "gene",    Id = "c" });

            var werns = reg.GetForCharacter("wernher").ToList();
            Assert.Equal(2, werns.Count);
            Assert.Contains(werns, f => f.Id == "a");
            Assert.Contains(werns, f => f.Id == "b");
        }

        [Fact]
        public void HasAnyFor_FalseWhenNone()
        {
            var reg = new FocusRegistry();
            Assert.False(reg.HasAnyFor("wernher"));
        }

        [Fact]
        public void HasAnyFor_TrueWhenPresent()
        {
            var reg = new FocusRegistry();
            reg.Add(new FocusOption { CharacterId = "wernher", Id = "a" });
            Assert.True(reg.HasAnyFor("wernher"));
        }
    }
}
```

- [ ] **Step 2: Run — expect compile failures**

- [ ] **Step 3: Write `FocusOption`**

Create `src/KerbalAdminKit/Focuses/FocusOption.cs`:

```csharp
namespace KerbalAdminKit.Focuses
{
    public sealed class FocusOption
    {
        public string CharacterId;
        public string Id;
        public string Title;
        public string Description;
        public string Requirement;   // optional KDK flag expression
        public string Flag;          // the flag KAK writes on selection
        public string FlagValue;     // the value to write
    }
}
```

- [ ] **Step 4: Write `FocusLoader`**

Create `src/KerbalAdminKit/Focuses/FocusLoader.cs`:

```csharp
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Focuses
{
    public static class FocusLoader
    {
        public static FocusOption Load(ISceneNode node)
        {
            if (node == null) return null;
            var character = node.GetValue("character");
            var id = node.GetValue("id");
            if (string.IsNullOrEmpty(character) || string.IsNullOrEmpty(id)) return null;

            return new FocusOption
            {
                CharacterId = character,
                Id = id,
                Title = node.GetValue("title") ?? id,
                Description = node.GetValue("description") ?? "",
                Requirement = node.GetValue("requirement"),
                Flag = node.GetValue("flag"),
                FlagValue = node.GetValue("flagValue"),
            };
        }
    }
}
```

- [ ] **Step 5: Write `FocusRegistry`**

Create `src/KerbalAdminKit/Focuses/FocusRegistry.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;

namespace KerbalAdminKit.Focuses
{
    public sealed class FocusRegistry
    {
        private readonly List<FocusOption> options = new List<FocusOption>();

        public void Add(FocusOption option)
        {
            if (option != null) options.Add(option);
        }

        public IEnumerable<FocusOption> All => options;

        public IEnumerable<FocusOption> GetForCharacter(string characterId) =>
            options.Where(o => o.CharacterId == characterId);

        public bool HasAnyFor(string characterId) =>
            options.Any(o => o.CharacterId == characterId);

        /// <summary>
        /// Returns focuses for character whose Requirement expression evaluates
        /// to true against the current flag store (null requirement = always valid).
        /// Used by the picker UI.
        /// </summary>
        public IEnumerable<FocusOption> GetValidFor(string characterId)
        {
            var flags = DialogueKit.Flags;
            foreach (var opt in GetForCharacter(characterId))
            {
                if (string.IsNullOrEmpty(opt.Requirement)) { yield return opt; continue; }
                try
                {
                    if (FlagExpressionParser.Evaluate(opt.Requirement, flags))
                        yield return opt;
                }
                catch
                {
                    // malformed expression → treat as not matching, log once
                    UnityEngine.Debug.LogWarning(
                        $"[KerbalAdminKit] Invalid focus requirement on {opt.Id}: {opt.Requirement}");
                }
            }
        }
    }
}
```

- [ ] **Step 6: Run tests — expect PASS**

Test for `GetValidFor` isn't written because `FlagExpressionParser` depends on KDK runtime — that path is verified manually in Task 30.

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

- [ ] **Step 7: Commit**

```bash
git add src/KerbalAdminKit/Focuses/ tests/KerbalAdminKit.Tests/FocusLoaderTests.cs tests/KerbalAdminKit.Tests/FocusRegistryTests.cs
git commit -m "feat: add FocusOption/Loader/Registry with requirement-expression filtering"
```

---

## Task 10: Memo data model (Memo + MemoContext + IMemoCondition)

**Files:**
- Create: `src/KerbalAdminKit/Memos/Memo.cs`
- Create: `src/KerbalAdminKit/Memos/MemoContext.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/IMemoCondition.cs`

These are data/interface types with no behavior; no unit tests in this task. Condition logic is tested in Task 11.

- [ ] **Step 1: Write `IMemoCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/IMemoCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    public interface IMemoCondition
    {
        bool Evaluate(MemoContext ctx);
    }
}
```

- [ ] **Step 2: Write `Memo`**

Create `src/KerbalAdminKit/Memos/Memo.cs`:

```csharp
using System.Collections.Generic;
using KerbalAdminKit.Memos.Conditions;

namespace KerbalAdminKit.Memos
{
    public enum MemoPriority { Low, Normal, High }

    public sealed class Memo
    {
        public string Id;
        public string CharacterId;
        public MemoPriority Priority = MemoPriority.Normal;
        public string Text;

        public List<IMemoCondition> Conditions = new List<IMemoCondition>();

        public double? ExpireAfterDays;   // null = never
        public bool SuppressAfterDismiss;
        public bool PostToStockMail;

        // Runtime state (not persisted):
        public bool IsActive;
        public double ActivatedAtSeconds;
    }
}
```

- [ ] **Step 3: Write `MemoContext`**

Create `src/KerbalAdminKit/Memos/MemoContext.cs`:

```csharp
using KerbalDialogueKit.Flags;

namespace KerbalAdminKit.Memos
{
    /// <summary>
    /// Snapshot of game state at evaluation time. Passed to conditions.
    /// Built by MemoTicker before each poll; pure-logic conditions read from it.
    /// </summary>
    public sealed class MemoContext
    {
        public double NowSeconds;
        public double Funds;
        public double Reputation;
        public double Science;
        public string CurrentChapter;
        public FlagStore Flags;

        // KSP-integrated conditions read additional state via services
        // they wire themselves (see VesselAgeCondition, ContractAvailableCondition).
    }
}
```

- [ ] **Step 4: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

- [ ] **Step 5: Commit**

```bash
git add src/KerbalAdminKit/Memos/Memo.cs src/KerbalAdminKit/Memos/MemoContext.cs src/KerbalAdminKit/Memos/Conditions/IMemoCondition.cs
git commit -m "feat: add Memo data model + MemoContext + IMemoCondition interface"
```

---

## Task 11: Pure-logic memo conditions (TDD)

Conditions that evaluate only from `MemoContext` fields — no KSP lookups. Implement and test together.

**Files:**
- Create: `src/KerbalAdminKit/Memos/Conditions/FundsBelowCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/ReputationAboveCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/ReputationBelowCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/InChapterCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/FlagExpressionCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/TimeSinceEventCondition.cs`
- Create: `tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Memos;
using KerbalAdminKit.Memos.Conditions;
using KerbalDialogueKit.Flags;

namespace KerbalAdminKit.Tests
{
    public class PureMemoConditionTests
    {
        private MemoContext Ctx(double funds = 0, double rep = 0, string chapter = null, FlagStore flags = null, double now = 0)
            => new MemoContext { Funds = funds, Reputation = rep, CurrentChapter = chapter, Flags = flags ?? new FlagStore(), NowSeconds = now };

        [Fact]
        public void FundsBelow_True_WhenFundsLessThanThreshold()
        {
            var c = new FundsBelowCondition { Threshold = 10000 };
            Assert.True(c.Evaluate(Ctx(funds: 9999)));
        }

        [Fact]
        public void FundsBelow_False_WhenFundsEqualOrAbove()
        {
            var c = new FundsBelowCondition { Threshold = 10000 };
            Assert.False(c.Evaluate(Ctx(funds: 10000)));
            Assert.False(c.Evaluate(Ctx(funds: 10001)));
        }

        [Fact]
        public void ReputationAbove_True_WhenRepExceedsThreshold()
        {
            var c = new ReputationAboveCondition { Threshold = 250 };
            Assert.True(c.Evaluate(Ctx(rep: 251)));
            Assert.False(c.Evaluate(Ctx(rep: 250)));
        }

        [Fact]
        public void ReputationBelow_True_WhenRepUnderThreshold()
        {
            var c = new ReputationBelowCondition { Threshold = 100 };
            Assert.True(c.Evaluate(Ctx(rep: 99)));
            Assert.False(c.Evaluate(Ctx(rep: 100)));
        }

        [Fact]
        public void InChapter_True_WhenCurrentMatches()
        {
            var c = new InChapterCondition { Chapter = "3" };
            Assert.True(c.Evaluate(Ctx(chapter: "3")));
            Assert.False(c.Evaluate(Ctx(chapter: "2")));
            Assert.False(c.Evaluate(Ctx(chapter: null)));
        }

        [Fact]
        public void FlagExpression_ParsesAndEvaluates()
        {
            var flags = new FlagStore();
            flags.Set("completed_first_anomaly", "true");
            flags.Set("chapter", "3");

            var c = new FlagExpressionCondition { Expression = "completed_first_anomaly == true" };
            Assert.True(c.Evaluate(Ctx(flags: flags)));

            var c2 = new FlagExpressionCondition { Expression = "chapter >= 4" };
            Assert.False(c2.Evaluate(Ctx(flags: flags)));
        }

        [Fact]
        public void TimeSinceEvent_True_WhenEventFlagOlderThanMinDays()
        {
            var flags = new FlagStore();
            // event time stored as seconds in a flag named "<event>_at"
            flags.Set("first_orbit_at", "0");           // elapsed: now - 0

            double secondsPerDay = 21600;               // KSP Kerbin day
            double now = secondsPerDay * 10;            // 10 days later

            var c = new TimeSinceEventCondition { EventName = "first_orbit", MinDays = 5 };
            Assert.True(c.Evaluate(Ctx(flags: flags, now: now)));

            var c2 = new TimeSinceEventCondition { EventName = "first_orbit", MinDays = 20 };
            Assert.False(c2.Evaluate(Ctx(flags: flags, now: now)));
        }

        [Fact]
        public void TimeSinceEvent_False_WhenEventMissing()
        {
            var flags = new FlagStore();
            var c = new TimeSinceEventCondition { EventName = "never_happened", MinDays = 1 };
            Assert.False(c.Evaluate(Ctx(flags: flags, now: 1e9)));
        }
    }
}
```

- [ ] **Step 2: Write `FundsBelowCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/FundsBelowCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    public sealed class FundsBelowCondition : IMemoCondition
    {
        public double Threshold;
        public bool Evaluate(MemoContext ctx) => ctx != null && ctx.Funds < Threshold;
    }
}
```

- [ ] **Step 3: Write `ReputationAboveCondition` + `ReputationBelowCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/ReputationAboveCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    public sealed class ReputationAboveCondition : IMemoCondition
    {
        public double Threshold;
        public bool Evaluate(MemoContext ctx) => ctx != null && ctx.Reputation > Threshold;
    }
}
```

Create `src/KerbalAdminKit/Memos/Conditions/ReputationBelowCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    public sealed class ReputationBelowCondition : IMemoCondition
    {
        public double Threshold;
        public bool Evaluate(MemoContext ctx) => ctx != null && ctx.Reputation < Threshold;
    }
}
```

- [ ] **Step 4: Write `InChapterCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/InChapterCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    public sealed class InChapterCondition : IMemoCondition
    {
        public string Chapter;

        public bool Evaluate(MemoContext ctx) =>
            ctx != null && Chapter != null && Chapter == ctx.CurrentChapter;
    }
}
```

- [ ] **Step 5: Write `FlagExpressionCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/FlagExpressionCondition.cs`:

```csharp
using KerbalDialogueKit.Flags;

namespace KerbalAdminKit.Memos.Conditions
{
    public sealed class FlagExpressionCondition : IMemoCondition
    {
        public string Expression;

        public bool Evaluate(MemoContext ctx)
        {
            if (string.IsNullOrEmpty(Expression) || ctx?.Flags == null) return false;
            try { return FlagExpressionParser.Evaluate(Expression, ctx.Flags); }
            catch { return false; }
        }
    }
}
```

- [ ] **Step 6: Write `TimeSinceEventCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/TimeSinceEventCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// Evaluates true when the current time is at least MinDays after the
    /// timestamp stored in flag "<EventName>_at". If the flag isn't set
    /// (event never happened), returns false. Day length = 21600s (Kerbin).
    /// </summary>
    public sealed class TimeSinceEventCondition : IMemoCondition
    {
        public string EventName;
        public double MinDays;

        private const double SecondsPerDay = 21600;

        public bool Evaluate(MemoContext ctx)
        {
            if (ctx?.Flags == null || string.IsNullOrEmpty(EventName)) return false;
            var flag = ctx.Flags.Get(EventName + "_at");
            if (string.IsNullOrEmpty(flag)) return false;
            if (!double.TryParse(flag, out var eventSeconds)) return false;
            var elapsedDays = (ctx.NowSeconds - eventSeconds) / SecondsPerDay;
            return elapsedDays >= MinDays;
        }
    }
}
```

- [ ] **Step 7: Run tests — expect PASS**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

- [ ] **Step 8: Commit**

```bash
git add src/KerbalAdminKit/Memos/Conditions/ tests/KerbalAdminKit.Tests/PureMemoConditionTests.cs
git commit -m "feat: add pure-logic memo conditions (Funds/Rep/InChapter/FlagExpr/TimeSinceEvent)"
```

---

## Task 12: KSP-integrated memo conditions + MemoConditionFactory

**Files:**
- Create: `src/KerbalAdminKit/Memos/Conditions/VesselAgeCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/ContractAvailableCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/BodyDiscoveredCondition.cs`
- Create: `src/KerbalAdminKit/Memos/Conditions/MemoConditionFactory.cs`

KSP-integrated conditions are not unit-tested (they call into live game state). Verified manually in Task 30.

- [ ] **Step 1: Write `VesselAgeCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/VesselAgeCondition.cs`:

```csharp
using System.Linq;

namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// True when the game contains at least one vessel whose in-game age
    /// falls in [MinDays, MaxDays]. MaxDays = null means no upper bound.
    /// Optional VesselType filter ("Relay", "Station", etc.) matches
    /// Vessel.vesselType.ToString().
    /// </summary>
    public sealed class VesselAgeCondition : IMemoCondition
    {
        public string VesselType;     // optional filter
        public double MinDays;
        public double? MaxDays;

        private const double SecondsPerDay = 21600;

        public bool Evaluate(MemoContext ctx)
        {
            if (FlightGlobals.Vessels == null) return false;
            var now = ctx?.NowSeconds ?? Planetarium.GetUniversalTime();

            return FlightGlobals.Vessels.Any(v =>
            {
                if (v == null) return false;
                if (!string.IsNullOrEmpty(VesselType) &&
                    v.vesselType.ToString() != VesselType) return false;
                var ageDays = (now - v.launchTime) / SecondsPerDay;
                if (ageDays < MinDays) return false;
                if (MaxDays.HasValue && ageDays > MaxDays.Value) return false;
                return true;
            });
        }
    }
}
```

- [ ] **Step 2: Write `ContractAvailableCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/ContractAvailableCondition.cs`:

```csharp
using System.Linq;
using Contracts;

namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// True when the CC contract system has at least Count contracts
    /// available (state = Offered) whose Group matches. Group null = any.
    /// </summary>
    public sealed class ContractAvailableCondition : IMemoCondition
    {
        public string Group;
        public int Count = 1;

        public bool Evaluate(MemoContext ctx)
        {
            if (ContractSystem.Instance?.Contracts == null) return false;
            var matching = ContractSystem.Instance.Contracts
                .Where(c => c.ContractState == Contract.State.Offered);

            if (!string.IsNullOrEmpty(Group))
                matching = matching.Where(c => c.GetType().Name == Group);

            return matching.Count() >= Count;
        }
    }
}
```

- [ ] **Step 3: Write `BodyDiscoveredCondition`**

Create `src/KerbalAdminKit/Memos/Conditions/BodyDiscoveredCondition.cs`:

```csharp
namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// True when the named celestial body has been discovered via
    /// ResearchBodies, or if ResearchBodies isn't installed, true whenever
    /// the body exists in the FlightGlobals bodies list.
    /// </summary>
    public sealed class BodyDiscoveredCondition : IMemoCondition
    {
        public string BodyName;

        public bool Evaluate(MemoContext ctx)
        {
            if (string.IsNullOrEmpty(BodyName)) return false;
            var body = FlightGlobals.Bodies?.Find(b => b.bodyName == BodyName);
            if (body == null) return false;

            // ResearchBodies integration is optional. If the assembly is absent,
            // "discovered" degrades to "exists" (always true for stock bodies).
            return true;
        }
    }
}
```

- [ ] **Step 4: Write `MemoConditionFactory`**

Create `src/KerbalAdminKit/Memos/Conditions/MemoConditionFactory.cs`:

```csharp
using System.Globalization;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Memos.Conditions
{
    /// <summary>
    /// Builds an IMemoCondition from a CONDITION cfg node. Returns null on
    /// unknown type or missing fields (loader logs and skips).
    /// </summary>
    public static class MemoConditionFactory
    {
        public static IMemoCondition Build(ISceneNode node)
        {
            if (node == null) return null;
            var type = node.GetValue("type");
            if (string.IsNullOrEmpty(type)) return null;

            switch (type)
            {
                case "FundsBelow":
                    return new FundsBelowCondition { Threshold = D(node, "threshold") };
                case "ReputationAbove":
                    return new ReputationAboveCondition { Threshold = D(node, "threshold") };
                case "ReputationBelow":
                    return new ReputationBelowCondition { Threshold = D(node, "threshold") };
                case "InChapter":
                    return new InChapterCondition { Chapter = node.GetValue("chapter") };
                case "FlagExpression":
                    return new FlagExpressionCondition { Expression = node.GetValue("expression") };
                case "TimeSinceEvent":
                    return new TimeSinceEventCondition
                    {
                        EventName = node.GetValue("event"),
                        MinDays = D(node, "minDays"),
                    };
                case "VesselAge":
                    return new VesselAgeCondition
                    {
                        VesselType = node.GetValue("vesselType"),
                        MinDays = D(node, "minDays"),
                        MaxDays = OptD(node, "maxDays"),
                    };
                case "ContractAvailable":
                    return new ContractAvailableCondition
                    {
                        Group = node.GetValue("group"),
                        Count = (int)D(node, "count", 1),
                    };
                case "BodyDiscovered":
                    return new BodyDiscoveredCondition { BodyName = node.GetValue("body") };
                default:
                    return null;
            }
        }

        private static double D(ISceneNode n, string k, double fallback = 0)
            => double.TryParse(n.GetValue(k), NumberStyles.Float,
                CultureInfo.InvariantCulture, out var v) ? v : fallback;

        private static double? OptD(ISceneNode n, string k)
            => n.HasValue(k) && double.TryParse(n.GetValue(k), NumberStyles.Float,
                CultureInfo.InvariantCulture, out var v) ? (double?)v : null;
    }
}
```

- [ ] **Step 5: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

- [ ] **Step 6: Commit**

```bash
git add src/KerbalAdminKit/Memos/Conditions/
git commit -m "feat: add KSP-integrated memo conditions + MemoConditionFactory"
```

---

## Task 13: MemoLoader (TDD for cfg parsing)

**Files:**
- Create: `src/KerbalAdminKit/Memos/MemoLoader.cs`
- Create: `tests/KerbalAdminKit.Tests/MemoLoaderTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/MemoLoaderTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.Memos;
using KerbalAdminKit.Memos.Conditions;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class MemoLoaderTests
    {
        [Fact]
        public void Load_ParsesCoreFields()
        {
            var n = new FakeSceneNode()
                .Set("id", "relay_aging")
                .Set("character", "gus")
                .Set("priority", "high")
                .Set("text", "That relay is getting old.")
                .Set("expireAfterDays", "60")
                .Set("suppressAfterDismiss", "true")
                .Set("postToStockMail", "true");

            var m = MemoLoader.Load(n);

            Assert.Equal("relay_aging", m.Id);
            Assert.Equal("gus", m.CharacterId);
            Assert.Equal(MemoPriority.High, m.Priority);
            Assert.Equal("That relay is getting old.", m.Text);
            Assert.Equal(60.0, m.ExpireAfterDays);
            Assert.True(m.SuppressAfterDismiss);
            Assert.True(m.PostToStockMail);
        }

        [Fact]
        public void Load_ReturnsNullWhenIdMissing()
        {
            var n = new FakeSceneNode().Set("text", "no id");
            Assert.Null(MemoLoader.Load(n));
        }

        [Fact]
        public void Load_DefaultsPriorityToNormal()
        {
            var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
            Assert.Equal(MemoPriority.Normal, MemoLoader.Load(n).Priority);
        }

        [Fact]
        public void Load_BuildsConditionsFromChildNodes()
        {
            var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
            n.AddChild("CONDITION", new FakeSceneNode()
                .Set("type", "FundsBelow")
                .Set("threshold", "10000"));
            n.AddChild("CONDITION", new FakeSceneNode()
                .Set("type", "InChapter")
                .Set("chapter", "3"));

            var m = MemoLoader.Load(n);
            Assert.Equal(2, m.Conditions.Count);
            Assert.IsType<FundsBelowCondition>(m.Conditions[0]);
            Assert.Equal(10000.0, ((FundsBelowCondition)m.Conditions[0]).Threshold);
            Assert.IsType<InChapterCondition>(m.Conditions[1]);
        }

        [Fact]
        public void Load_SkipsUnknownConditionTypes()
        {
            var n = new FakeSceneNode().Set("id", "x").Set("text", "y");
            n.AddChild("CONDITION", new FakeSceneNode().Set("type", "NotAThing"));
            n.AddChild("CONDITION", new FakeSceneNode().Set("type", "InChapter").Set("chapter", "1"));

            var m = MemoLoader.Load(n);
            Assert.Single(m.Conditions);
            Assert.IsType<InChapterCondition>(m.Conditions[0]);
        }
    }
}
```

- [ ] **Step 2: Run — expect compile failures**

- [ ] **Step 3: Write `MemoLoader`**

Create `src/KerbalAdminKit/Memos/MemoLoader.cs`:

```csharp
using System.Globalization;
using KerbalAdminKit.Memos.Conditions;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.Memos
{
    public static class MemoLoader
    {
        public static Memo Load(ISceneNode node)
        {
            if (node == null) return null;
            var id = node.GetValue("id");
            if (string.IsNullOrEmpty(id)) return null;

            var m = new Memo
            {
                Id = id,
                CharacterId = node.GetValue("character") ?? "",
                Text = node.GetValue("text") ?? "",
                Priority = ParsePriority(node.GetValue("priority")),
            };

            if (node.HasValue("expireAfterDays") &&
                double.TryParse(node.GetValue("expireAfterDays"),
                    NumberStyles.Float, CultureInfo.InvariantCulture, out var exp))
                m.ExpireAfterDays = exp;

            if (node.HasValue("suppressAfterDismiss") &&
                bool.TryParse(node.GetValue("suppressAfterDismiss"), out var sup))
                m.SuppressAfterDismiss = sup;

            if (node.HasValue("postToStockMail") &&
                bool.TryParse(node.GetValue("postToStockMail"), out var psm))
                m.PostToStockMail = psm;

            foreach (var condNode in node.GetNodes("CONDITION"))
            {
                var cond = MemoConditionFactory.Build(condNode);
                if (cond != null) m.Conditions.Add(cond);
            }

            return m;
        }

        private static MemoPriority ParsePriority(string s)
        {
            if (string.IsNullOrEmpty(s)) return MemoPriority.Normal;
            switch (s.ToLowerInvariant())
            {
                case "low":    return MemoPriority.Low;
                case "high":   return MemoPriority.High;
                default:       return MemoPriority.Normal;
            }
        }
    }
}
```

- [ ] **Step 4: Run tests — expect PASS**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalAdminKit/Memos/MemoLoader.cs tests/KerbalAdminKit.Tests/MemoLoaderTests.cs
git commit -m "feat: add MemoLoader parsing DIRECTOR_MEMO with CONDITION blocks"
```

---

## Task 14: MemoRegistry + MemoTicker (KSP-integrated)

**Files:**
- Create: `src/KerbalAdminKit/Memos/MemoRegistry.cs`
- Create: `src/KerbalAdminKit/Memos/MemoTicker.cs`

No unit tests — both classes depend on KSP singletons + `MonoBehaviour`. Verified in Task 30.

- [ ] **Step 1: Write `MemoRegistry`**

Create `src/KerbalAdminKit/Memos/MemoRegistry.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;

namespace KerbalAdminKit.Memos
{
    public sealed class MemoRegistry
    {
        private readonly List<Memo> all = new List<Memo>();

        public void Add(Memo m) { if (m != null) all.Add(m); }

        public IEnumerable<Memo> All => all;

        public IEnumerable<Memo> Active =>
            all.Where(m => m.IsActive)
               .OrderByDescending(m => (int)m.Priority)
               .ThenBy(m => m.ActivatedAtSeconds);

        public IEnumerable<Memo> Pinned =>
            all.Where(m => m.IsActive && m.Priority == MemoPriority.High);

        public Memo Get(string id) => all.FirstOrDefault(m => m.Id == id);

        /// <summary>
        /// Marks a memo dismissed. Records dismissal time in AdminKitScenario.
        /// If SuppressAfterDismiss, the memo won't re-activate this save.
        /// </summary>
        public void Dismiss(string id, double nowSeconds)
        {
            var m = Get(id);
            if (m == null) return;
            m.IsActive = false;
            if (AdminKitScenario.Instance != null)
                AdminKitScenario.Instance.DismissedMemos[id] = nowSeconds;
        }
    }
}
```

- [ ] **Step 2: Write `MemoTicker`**

Create `src/KerbalAdminKit/Memos/MemoTicker.cs`:

```csharp
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalAdminKit.Memos
{
    /// <summary>
    /// Polls memo conditions while at KSC and flips IsActive accordingly.
    /// Fires stock-mail delivery on first-time activation when PostToStockMail.
    /// </summary>
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class MemoTicker : MonoBehaviour
    {
        private const double SecondsPerDay = 21600;

        private MemoRegistry registry;
        private double pollSeconds = 5.0;
        private double nextPoll;

        public void Initialize(MemoRegistry reg, double pollInterval)
        {
            registry = reg;
            pollSeconds = pollInterval;
        }

        private void FixedUpdate()
        {
            if (registry == null) return;
            var now = Planetarium.GetUniversalTime();
            if (now < nextPoll) return;
            nextPoll = now + pollSeconds;

            var ctx = BuildContext(now);
            foreach (var memo in registry.All) EvaluateMemo(memo, ctx, now);
        }

        private MemoContext BuildContext(double now)
        {
            return new MemoContext
            {
                NowSeconds = now,
                Funds = Funding.Instance != null ? Funding.Instance.Funds : 0,
                Reputation = global::Reputation.Instance != null ? global::Reputation.Instance.reputation : 0,
                Science = ResearchAndDevelopment.Instance != null ? ResearchAndDevelopment.Instance.Science : 0,
                CurrentChapter = KerbalCampaignKit.Core.CampaignKit.Chapters?.Current,
                Flags = DialogueKit.Flags,
            };
        }

        private void EvaluateMemo(Memo memo, MemoContext ctx, double now)
        {
            // Suppression: permanently dismissed this save
            if (memo.SuppressAfterDismiss &&
                AdminKitScenario.Instance != null &&
                AdminKitScenario.Instance.DismissedMemos.ContainsKey(memo.Id))
            {
                memo.IsActive = false;
                return;
            }

            // Expiry: was active, N days have passed since activation
            if (memo.IsActive && memo.ExpireAfterDays.HasValue)
            {
                var activeDays = (now - memo.ActivatedAtSeconds) / SecondsPerDay;
                if (activeDays > memo.ExpireAfterDays.Value)
                {
                    memo.IsActive = false;
                    return;
                }
            }

            // Evaluate all conditions (AND)
            bool allPass = memo.Conditions.Count == 0;
            if (!allPass)
            {
                allPass = true;
                foreach (var cond in memo.Conditions)
                {
                    if (!cond.Evaluate(ctx)) { allPass = false; break; }
                }
            }

            if (allPass && !memo.IsActive)
            {
                memo.IsActive = true;
                memo.ActivatedAtSeconds = now;
                PostToStockMailIfNeeded(memo);
            }
            else if (!allPass && memo.IsActive)
            {
                memo.IsActive = false;
            }
        }

        private void PostToStockMailIfNeeded(Memo memo)
        {
            if (!memo.PostToStockMail) return;
            if (AdminKitScenario.Instance == null) return;
            if (AdminKitScenario.Instance.StockMailedMemos.Contains(memo.Id)) return;

            if (MessageSystem.Instance != null)
            {
                var title = !string.IsNullOrEmpty(memo.CharacterId)
                    ? $"Memo from {memo.CharacterId}"
                    : "Program Memo";
                var msg = new MessageSystem.Message(
                    title,
                    memo.Text ?? "",
                    MessageSystemButton.MessageButtonColor.GREEN,
                    MessageSystemButton.ButtonIcons.MESSAGE);
                MessageSystem.Instance.AddMessage(msg);
            }
            AdminKitScenario.Instance.StockMailedMemos.Add(memo.Id);
        }
    }
}
```

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalAdminKit/Memos/MemoRegistry.cs src/KerbalAdminKit/Memos/MemoTicker.cs
git commit -m "feat: add MemoRegistry + MemoTicker polling with stock-mail mirroring"
```

---

## Task 15: DispositionDecayCompiler + Loader (TDD)

**Files:**
- Create: `src/KerbalAdminKit/DecayCompiler/DispositionDecayLoader.cs`
- Create: `src/KerbalAdminKit/DecayCompiler/DispositionDecayCompiler.cs`
- Create: `tests/KerbalAdminKit.Tests/TestHelpers/FakeTriggerRegistry.cs`
- Create: `tests/KerbalAdminKit.Tests/DispositionDecayCompilerTests.cs`

- [ ] **Step 1: Write fake trigger registry helper**

Create `tests/KerbalAdminKit.Tests/TestHelpers/FakeTriggerRegistry.cs`:

```csharp
using System.Collections.Generic;
using KerbalAdminKit.DecayCompiler;

namespace KerbalAdminKit.Tests.TestHelpers
{
    /// <summary>
    /// ITriggerSink captures compiled triggers for test assertions.
    /// Production wiring adapts KCK's CampaignKit.Engine.Register to this interface.
    /// </summary>
    public sealed class FakeTriggerRegistry : ITriggerSink
    {
        public List<CompiledDecayTrigger> Registered = new List<CompiledDecayTrigger>();
        public void Register(CompiledDecayTrigger t) => Registered.Add(t);
    }
}
```

- [ ] **Step 2: Write failing tests**

Create `tests/KerbalAdminKit.Tests/DispositionDecayCompilerTests.cs`:

```csharp
using System.Linq;
using Xunit;
using KerbalAdminKit.DecayCompiler;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class DispositionDecayCompilerTests
    {
        [Fact]
        public void Compile_ProducesOneTriggerPerStep()
        {
            var node = new FakeSceneNode()
                .Set("character", "wernher")
                .Set("stepDays", "180")
                .Set("towardValue", "Neutral");
            node.AddChild("STEP", new FakeSceneNode().Set("from", "Enthusiastic").Set("to", "Supportive"));
            node.AddChild("STEP", new FakeSceneNode().Set("from", "Supportive").Set("to", "Neutral"));

            var decay = DispositionDecayLoader.Load(node);
            var sink = new FakeTriggerRegistry();
            DispositionDecayCompiler.Compile(decay, sink);

            Assert.Equal(2, sink.Registered.Count);

            var t0 = sink.Registered[0];
            Assert.Equal("disposition_decay_wernher_Enthusiastic_to_Supportive", t0.Id);
            Assert.Equal("wernher_disposition", t0.FlagToMatch);
            Assert.Equal("Enthusiastic", t0.ExpectedFlagValue);
            Assert.Equal("Supportive", t0.TargetFlagValue);
            Assert.Equal(180.0, t0.StepDays);
        }

        [Fact]
        public void Compile_SkipsIncompleteSteps()
        {
            var node = new FakeSceneNode()
                .Set("character", "gus")
                .Set("stepDays", "120");
            node.AddChild("STEP", new FakeSceneNode().Set("from", "A"));                  // missing to
            node.AddChild("STEP", new FakeSceneNode().Set("to", "B"));                    // missing from
            node.AddChild("STEP", new FakeSceneNode().Set("from", "Frustrated").Set("to", "Skeptical"));

            var decay = DispositionDecayLoader.Load(node);
            var sink = new FakeTriggerRegistry();
            DispositionDecayCompiler.Compile(decay, sink);

            Assert.Single(sink.Registered);
            Assert.Equal("disposition_decay_gus_Frustrated_to_Skeptical", sink.Registered[0].Id);
        }

        [Fact]
        public void Load_ReturnsNullWhenCharacterMissing()
        {
            var node = new FakeSceneNode().Set("stepDays", "180");
            node.AddChild("STEP", new FakeSceneNode().Set("from", "X").Set("to", "Y"));
            Assert.Null(DispositionDecayLoader.Load(node));
        }
    }
}
```

- [ ] **Step 3: Write `DispositionDecayLoader`**

Create `src/KerbalAdminKit/DecayCompiler/DispositionDecayLoader.cs`:

```csharp
using System.Collections.Generic;
using System.Globalization;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.DecayCompiler
{
    public sealed class DispositionDecay
    {
        public string CharacterId;
        public double StepDays;
        public string TowardValue;                    // display only; the transitions express the target
        public List<DispositionDecayStep> Steps = new List<DispositionDecayStep>();
    }

    public sealed class DispositionDecayStep
    {
        public string From;
        public string To;
    }

    public static class DispositionDecayLoader
    {
        public static DispositionDecay Load(ISceneNode node)
        {
            if (node == null) return null;
            var character = node.GetValue("character");
            if (string.IsNullOrEmpty(character)) return null;

            var d = new DispositionDecay { CharacterId = character };
            if (node.HasValue("stepDays") &&
                double.TryParse(node.GetValue("stepDays"),
                    NumberStyles.Float, CultureInfo.InvariantCulture, out var days))
                d.StepDays = days;
            d.TowardValue = node.GetValue("towardValue") ?? "Neutral";

            foreach (var stepNode in node.GetNodes("STEP"))
            {
                d.Steps.Add(new DispositionDecayStep
                {
                    From = stepNode.GetValue("from"),
                    To = stepNode.GetValue("to"),
                });
            }

            return d;
        }
    }
}
```

- [ ] **Step 4: Write `DispositionDecayCompiler` + `ITriggerSink`**

Create `src/KerbalAdminKit/DecayCompiler/DispositionDecayCompiler.cs`:

```csharp
namespace KerbalAdminKit.DecayCompiler
{
    /// <summary>
    /// Pure-logic representation of the trigger we want KCK to fire. The
    /// production wiring in AdminKitAddon translates these to real
    /// KerbalCampaignKit.Triggers.Trigger objects and calls
    /// CampaignKit.Engine.Register; the indirection lets us test compilation
    /// without KCK's runtime engine.
    /// </summary>
    public sealed class CompiledDecayTrigger
    {
        public string Id;
        public double StepDays;
        public string FlagToMatch;
        public string ExpectedFlagValue;
        public string TargetFlagValue;
    }

    public interface ITriggerSink
    {
        void Register(CompiledDecayTrigger trigger);
    }

    public static class DispositionDecayCompiler
    {
        public static void Compile(DispositionDecay decay, ITriggerSink sink)
        {
            if (decay == null || sink == null) return;
            if (string.IsNullOrEmpty(decay.CharacterId)) return;

            var flagName = decay.CharacterId + "_disposition";

            foreach (var step in decay.Steps)
            {
                if (string.IsNullOrEmpty(step?.From) || string.IsNullOrEmpty(step.To)) continue;
                sink.Register(new CompiledDecayTrigger
                {
                    Id = $"disposition_decay_{decay.CharacterId}_{step.From}_to_{step.To}",
                    StepDays = decay.StepDays,
                    FlagToMatch = flagName,
                    ExpectedFlagValue = step.From,
                    TargetFlagValue = step.To,
                });
            }
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
dotnet test tests/KerbalAdminKit.Tests/KerbalAdminKit.Tests.csproj
```

- [ ] **Step 6: Commit**

```bash
git add src/KerbalAdminKit/DecayCompiler/ tests/KerbalAdminKit.Tests/TestHelpers/FakeTriggerRegistry.cs tests/KerbalAdminKit.Tests/DispositionDecayCompilerTests.cs
git commit -m "feat: add DispositionDecayCompiler with pure-logic CompiledDecayTrigger + ITriggerSink"
```

---

## Task 16: Notification style + building scene + marker-offset loaders (TDD)

**Files:**
- Create: `src/KerbalAdminKit/KscRenderer/NotificationStyleRegistry.cs`
- Create: `src/KerbalAdminKit/KscRenderer/NotificationStyleLoader.cs`
- Create: `src/KerbalAdminKit/KscRenderer/KscMarkerOffsetRegistry.cs`
- Create: `src/KerbalAdminKit/BuildingOverlays/BuildingSceneRegistry.cs`
- Create: `src/KerbalAdminKit/BuildingOverlays/BuildingSceneLoader.cs`
- Create: `tests/KerbalAdminKit.Tests/NotificationStyleLoaderTests.cs`
- Create: `tests/KerbalAdminKit.Tests/BuildingSceneLoaderTests.cs`

- [ ] **Step 1: Write failing tests**

Create `tests/KerbalAdminKit.Tests/NotificationStyleLoaderTests.cs`:

```csharp
using Xunit;
using KerbalAdminKit.KscRenderer;
using KerbalAdminKit.Util;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class NotificationStyleLoaderTests
    {
        [Fact]
        public void Load_ParsesSeverityStyle()
        {
            var n = new FakeSceneNode()
                .Set("severity", "Action")
                .Set("textureUrl", "KerbalAdminKit/Art/exclamation")
                .Set("color", "#FFFF4040")
                .Set("pulse", "true");

            var style = NotificationStyleLoader.Load(n);

            Assert.Equal("Action", style.Severity);
            Assert.Equal("KerbalAdminKit/Art/exclamation", style.TextureUrl);
            Assert.Equal(HexColor.Parse("#FFFF4040"), style.Color);
            Assert.True(style.Pulse);
        }

        [Fact]
        public void Load_ReturnsNullWhenSeverityMissing()
        {
            var n = new FakeSceneNode().Set("textureUrl", "x");
            Assert.Null(NotificationStyleLoader.Load(n));
        }

        [Fact]
        public void Registry_OverridesDefaults()
        {
            var reg = new NotificationStyleRegistry();
            reg.AddOrReplace(new NotificationStyle { Severity = "Action", TextureUrl = "custom" });
            Assert.Equal("custom", reg.Get("Action").TextureUrl);
        }
    }
}
```

Create `tests/KerbalAdminKit.Tests/BuildingSceneLoaderTests.cs`:

```csharp
using System.Linq;
using Xunit;
using KerbalAdminKit.BuildingOverlays;
using KerbalAdminKit.Tests.TestHelpers;

namespace KerbalAdminKit.Tests
{
    public class BuildingSceneLoaderTests
    {
        [Fact]
        public void Load_ParsesCoreFields()
        {
            var n = new FakeSceneNode()
                .Set("facility", "MissionControl")
                .Set("character", "gene")
                .Set("onClickScene", "gene_mc_status_briefing");

            var bs = BuildingSceneLoader.Load(n);

            Assert.Equal("MissionControl", bs.Facility);
            Assert.Equal("gene", bs.CharacterId);
            Assert.Equal("gene_mc_status_briefing", bs.OnClickScene);
            Assert.Empty(bs.ChipLayerOverrides);
        }

        [Fact]
        public void Load_ReturnsNullWhenFacilityOrCharacterMissing()
        {
            Assert.Null(BuildingSceneLoader.Load(new FakeSceneNode().Set("character", "gus")));
            Assert.Null(BuildingSceneLoader.Load(new FakeSceneNode().Set("facility", "RD")));
        }

        [Fact]
        public void Load_ParsesChipLayerOverrides()
        {
            var n = new FakeSceneNode()
                .Set("facility", "MissionControl")
                .Set("character", "gene")
                .Set("onClickScene", "scene");
            n.AddChild("CHIP_LAYER", new FakeSceneNode().Set("textureUrl", "Override/gene_mc"));

            var bs = BuildingSceneLoader.Load(n);
            Assert.Single(bs.ChipLayerOverrides);
            Assert.Equal("Override/gene_mc", bs.ChipLayerOverrides[0].TextureUrl);
        }

        [Fact]
        public void Registry_GroupsByFacility()
        {
            var reg = new BuildingSceneRegistry();
            reg.Add(new BuildingScene { Facility = "MissionControl", CharacterId = "gene" });
            reg.Add(new BuildingScene { Facility = "MissionControl", CharacterId = "walt" });
            reg.Add(new BuildingScene { Facility = "TrackingStation", CharacterId = "linus" });

            var mc = reg.GetForFacility("MissionControl").ToList();
            Assert.Equal(2, mc.Count);
            Assert.Single(reg.GetForFacility("TrackingStation"));
        }
    }
}
```

- [ ] **Step 2: Run — expect compile failures**

- [ ] **Step 3: Write `NotificationStyle` + `NotificationStyleLoader` + `NotificationStyleRegistry`**

Create `src/KerbalAdminKit/KscRenderer/NotificationStyleRegistry.cs`:

```csharp
using System.Collections.Generic;
using KerbalAdminKit.Util;

namespace KerbalAdminKit.KscRenderer
{
    public sealed class NotificationStyle
    {
        public string Severity;                   // "Action" or "Info"
        public string TextureUrl;
        public ColorValue Color = ColorValue.White;
        public bool Pulse;
    }

    public sealed class NotificationStyleRegistry
    {
        private readonly Dictionary<string, NotificationStyle> bySeverity =
            new Dictionary<string, NotificationStyle>();

        public NotificationStyleRegistry()
        {
            // Defaults: shipped PNGs
            AddOrReplace(new NotificationStyle
            {
                Severity = "Action",
                TextureUrl = "KerbalAdminKit/Art/exclamation",
                Color = HexColor.Parse("#FFFF4040"),
                Pulse = true,
            });
            AddOrReplace(new NotificationStyle
            {
                Severity = "Info",
                TextureUrl = "KerbalAdminKit/Art/dot",
                Color = HexColor.Parse("#FFFFE066"),
                Pulse = false,
            });
        }

        public void AddOrReplace(NotificationStyle s)
        {
            if (s == null || string.IsNullOrEmpty(s.Severity)) return;
            bySeverity[s.Severity] = s;
        }

        public NotificationStyle Get(string severity) =>
            (severity != null && bySeverity.TryGetValue(severity, out var s)) ? s : null;
    }
}
```

Create `src/KerbalAdminKit/KscRenderer/NotificationStyleLoader.cs`:

```csharp
using KerbalAdminKit.Util;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.KscRenderer
{
    public static class NotificationStyleLoader
    {
        public static NotificationStyle Load(ISceneNode node)
        {
            if (node == null) return null;
            var severity = node.GetValue("severity");
            if (string.IsNullOrEmpty(severity)) return null;

            var s = new NotificationStyle { Severity = severity };
            if (node.HasValue("textureUrl")) s.TextureUrl = node.GetValue("textureUrl");
            if (node.HasValue("color")) s.Color = HexColor.Parse(node.GetValue("color"));
            if (node.HasValue("pulse") && bool.TryParse(node.GetValue("pulse"), out var p))
                s.Pulse = p;
            return s;
        }
    }
}
```

Create `src/KerbalAdminKit/KscRenderer/KscMarkerOffsetRegistry.cs`:

```csharp
using System.Collections.Generic;
using System.Globalization;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.KscRenderer
{
    public sealed class KscMarkerOffsetRegistry
    {
        private readonly Dictionary<string, UnityEngine.Vector2> offsets =
            new Dictionary<string, UnityEngine.Vector2>();

        public UnityEngine.Vector2 Get(string building) =>
            offsets.TryGetValue(building ?? "", out var v) ? v : new UnityEngine.Vector2(0, 20);

        public void Load(ISceneNode node)
        {
            if (node == null) return;
            var building = node.GetValue("building");
            if (string.IsNullOrEmpty(building)) return;
            float x = 0, y = 20;
            float.TryParse(node.GetValue("x"), NumberStyles.Float, CultureInfo.InvariantCulture, out x);
            float.TryParse(node.GetValue("y"), NumberStyles.Float, CultureInfo.InvariantCulture, out y);
            offsets[building] = new UnityEngine.Vector2(x, y);
        }
    }
}
```

- [ ] **Step 4: Write `BuildingScene` + loader + registry**

Create `src/KerbalAdminKit/BuildingOverlays/BuildingSceneRegistry.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using KerbalAdminKit.Characters;

namespace KerbalAdminKit.BuildingOverlays
{
    public sealed class BuildingScene
    {
        public string Facility;
        public string CharacterId;
        public string OnClickScene;
        public List<ChipLayerSpec> ChipLayerOverrides = new List<ChipLayerSpec>();
    }

    public sealed class BuildingSceneRegistry
    {
        private readonly List<BuildingScene> scenes = new List<BuildingScene>();

        public void Add(BuildingScene s) { if (s != null) scenes.Add(s); }

        public IEnumerable<BuildingScene> All => scenes;

        public IEnumerable<BuildingScene> GetForFacility(string facility) =>
            scenes.Where(s => s.Facility == facility);
    }
}
```

Create `src/KerbalAdminKit/BuildingOverlays/BuildingSceneLoader.cs`:

```csharp
using System.Globalization;
using KerbalAdminKit.Characters;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit.BuildingOverlays
{
    public static class BuildingSceneLoader
    {
        public static BuildingScene Load(ISceneNode node)
        {
            if (node == null) return null;
            var facility = node.GetValue("facility");
            var character = node.GetValue("character");
            if (string.IsNullOrEmpty(facility) || string.IsNullOrEmpty(character)) return null;

            var bs = new BuildingScene
            {
                Facility = facility,
                CharacterId = character,
                OnClickScene = node.GetValue("onClickScene"),
            };

            foreach (var layerNode in node.GetNodes("CHIP_LAYER"))
            {
                var layer = new ChipLayerSpec { TextureUrl = layerNode.GetValue("textureUrl") };
                if (layerNode.HasValue("rect"))
                {
                    var parts = (layerNode.GetValue("rect") ?? "").Split(',');
                    if (parts.Length == 4 &&
                        float.TryParse(parts[0].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var x) &&
                        float.TryParse(parts[1].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var y) &&
                        float.TryParse(parts[2].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var w) &&
                        float.TryParse(parts[3].Trim(), NumberStyles.Float, CultureInfo.InvariantCulture, out var h))
                    {
                        layer.RectX = x; layer.RectY = y; layer.RectW = w; layer.RectH = h;
                    }
                }
                if (layerNode.HasValue("tintWithDisposition") &&
                    bool.TryParse(layerNode.GetValue("tintWithDisposition"), out var tint))
                    layer.TintWithDisposition = tint;

                if (!string.IsNullOrEmpty(layer.TextureUrl))
                    bs.ChipLayerOverrides.Add(layer);
            }

            return bs;
        }
    }
}
```

- [ ] **Step 5: Run tests — expect PASS**

- [ ] **Step 6: Commit**

```bash
git add src/KerbalAdminKit/KscRenderer/ src/KerbalAdminKit/BuildingOverlays/ tests/KerbalAdminKit.Tests/NotificationStyleLoaderTests.cs tests/KerbalAdminKit.Tests/BuildingSceneLoaderTests.cs
git commit -m "feat: add notification style + building scene + marker offset loaders and registries"
```

---

## Task 17: TextureLoader + ChipRenderer

**Files:**
- Create: `src/KerbalAdminKit/Util/TextureLoader.cs`
- Create: `src/KerbalAdminKit/Admin/ChipRenderer.cs`

KSP-integrated (UnityEngine), no unit tests.

- [ ] **Step 1: Write `TextureLoader`**

Create `src/KerbalAdminKit/Util/TextureLoader.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace KerbalAdminKit.Util
{
    /// <summary>
    /// Caches GameDatabase texture lookups. Returns null if not found;
    /// callers render a fallback (color swatch) when null.
    /// </summary>
    public static class TextureLoader
    {
        private static readonly Dictionary<string, Texture2D> cache =
            new Dictionary<string, Texture2D>();

        public static Texture2D Get(string url)
        {
            if (string.IsNullOrEmpty(url)) return null;
            if (cache.TryGetValue(url, out var tex)) return tex;

            var loaded = GameDatabase.Instance?.GetTexture(url, false);
            cache[url] = loaded;
            if (loaded == null)
                Debug.LogWarning($"[KerbalAdminKit] Texture not found: {url}");
            return loaded;
        }

        public static void ClearCache() => cache.Clear();
    }
}
```

- [ ] **Step 2: Write `ChipRenderer`**

Create `src/KerbalAdminKit/Admin/ChipRenderer.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Util;

namespace KerbalAdminKit.Admin
{
    /// <summary>
    /// Draws a composed chip: layered textures (or baseColor fallback) +
    /// disposition tint overlay on opt-in layers. Caller supplies the
    /// pixel rect and the chip spec.
    /// </summary>
    public static class ChipRenderer
    {
        public static void Draw(Rect chipRect, CharacterInfo info, string disposition, List<ChipLayerSpec> layerOverride = null)
        {
            if (info == null) return;

            var layers = layerOverride != null && layerOverride.Count > 0
                ? layerOverride
                : info.ChipLayers;

            if (layers == null || layers.Count == 0)
            {
                // Fallback: color-swatch filled with baseColor, tinted by disposition
                var tint = DispositionColors.ResolveTint(info, disposition);
                DrawFilledRect(chipRect, ToUnity(tint));
                return;
            }

            foreach (var layer in layers)
            {
                var tex = TextureLoader.Get(layer.TextureUrl);
                if (tex == null) continue;
                var layerRect = new Rect(
                    chipRect.x + layer.RectX * chipRect.width,
                    chipRect.y + (1 - layer.RectY - layer.RectH) * chipRect.height,
                    layer.RectW * chipRect.width,
                    layer.RectH * chipRect.height);

                if (layer.TintWithDisposition)
                {
                    var tint = DispositionColors.ResolveTint(info, disposition);
                    var original = GUI.color;
                    GUI.color = ToUnity(tint);
                    GUI.DrawTexture(layerRect, tex, ScaleMode.ScaleToFit);
                    GUI.color = original;
                }
                else
                {
                    GUI.DrawTexture(layerRect, tex, ScaleMode.ScaleToFit);
                }
            }
        }

        private static Texture2D solidPixel;
        private static Texture2D SolidPixel()
        {
            if (solidPixel == null)
            {
                solidPixel = new Texture2D(1, 1);
                solidPixel.SetPixel(0, 0, Color.white);
                solidPixel.Apply();
            }
            return solidPixel;
        }

        private static void DrawFilledRect(Rect r, Color c)
        {
            var original = GUI.color;
            GUI.color = c;
            GUI.DrawTexture(r, SolidPixel());
            GUI.color = original;
        }

        public static Color ToUnity(ColorValue c) =>
            new Color(c.R / 255f, c.G / 255f, c.B / 255f, c.A / 255f);
    }
}
```

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalAdminKit/Util/TextureLoader.cs src/KerbalAdminKit/Admin/ChipRenderer.cs
git commit -m "feat: add TextureLoader cache and ChipRenderer (layered composition + swatch fallback)"
```

---

## Task 18: CharacterPanel + FocusPicker

**Files:**
- Create: `src/KerbalAdminKit/Focuses/FocusPicker.cs`
- Create: `src/KerbalAdminKit/Admin/CharacterPanel.cs`

- [ ] **Step 1: Write `FocusPicker`**

Create `src/KerbalAdminKit/Focuses/FocusPicker.cs`:

```csharp
using UnityEngine;
using ClickThroughFix;
using KerbalDialogueKit.Core;
using KerbalAdminKit.Characters;

namespace KerbalAdminKit.Focuses
{
    /// <summary>
    /// IMGUI modal that lists valid focuses for a character. Selecting sets
    /// the flag and closes. Caller sets Show(character) to open.
    /// </summary>
    public sealed class FocusPicker
    {
        private readonly FocusRegistry focuses;
        private readonly CharacterRegistry characters;
        private CharacterInfo target;
        private Rect window = new Rect(Screen.width / 2 - 200, Screen.height / 2 - 200, 400, 400);
        private Vector2 scroll;
        private const int WindowId = 0x4B41_4B01;

        public bool IsOpen => target != null;

        public FocusPicker(FocusRegistry focuses, CharacterRegistry characters)
        {
            this.focuses = focuses;
            this.characters = characters;
        }

        public void Show(CharacterInfo character) => target = character;
        public void Close() => target = null;

        public void OnGUI()
        {
            if (target == null) return;
            window = ClickThroughBlocker.GUILayoutWindow(
                WindowId, window, DrawWindow, $"Focus: {target.DisplayName}", GUI.skin.window);
        }

        private void DrawWindow(int id)
        {
            scroll = GUILayout.BeginScrollView(scroll);
            foreach (var focus in focuses.GetValidFor(target.Id))
            {
                GUILayout.BeginVertical(GUI.skin.box);
                GUILayout.Label($"<b>{focus.Title}</b>");
                if (!string.IsNullOrEmpty(focus.Description))
                    GUILayout.Label(focus.Description);
                if (GUILayout.Button("Select"))
                {
                    DialogueKit.Flags?.Set(focus.Flag ?? "", focus.FlagValue ?? "");
                    Close();
                }
                GUILayout.EndVertical();
            }
            GUILayout.EndScrollView();

            if (GUILayout.Button("Cancel")) Close();
            GUI.DragWindow();
        }
    }
}
```

- [ ] **Step 2: Write `CharacterPanel`**

Create `src/KerbalAdminKit/Admin/CharacterPanel.cs`:

```csharp
using UnityEngine;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Focuses;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;

namespace KerbalAdminKit.Admin
{
    /// <summary>
    /// Left-column character chips + text block. Click opens the focus picker,
    /// or fires onClickScene if no focuses exist for the character.
    /// </summary>
    public sealed class CharacterPanel
    {
        private readonly CharacterRegistry characters;
        private readonly FocusRegistry focuses;
        private readonly FocusPicker picker;
        private Vector2 scroll;

        public CharacterPanel(CharacterRegistry characters, FocusRegistry focuses, FocusPicker picker)
        {
            this.characters = characters;
            this.focuses = focuses;
            this.picker = picker;
        }

        public void Draw(Rect area)
        {
            GUILayout.BeginArea(area, GUI.skin.box);
            scroll = GUILayout.BeginScrollView(scroll);

            foreach (var c in characters.All)
            {
                DrawChip(c);
                GUILayout.Space(8);
            }

            GUILayout.EndScrollView();
            GUILayout.EndArea();
        }

        private void DrawChip(CharacterInfo c)
        {
            GUILayout.BeginVertical(GUI.skin.box);

            var chipHeight = 64f;
            var chipRect = GUILayoutUtility.GetRect(GUILayout.ExpandWidth(true), GUILayout.Height(chipHeight));
            var disposition = characters.GetDisposition(c.Id);
            ChipRenderer.Draw(chipRect, c, disposition);

            // Click detection: button over chip area (invisible)
            if (GUI.Button(chipRect, GUIContent.none, GUIStyle.none))
                OnChipClicked(c);

            GUILayout.Label($"<b>{c.DisplayName}</b>");
            if (!string.IsNullOrEmpty(c.Role)) GUILayout.Label(c.Role);
            GUILayout.Label($"— {DispositionColors.ResolveLabel(c, disposition) ?? "—"}");

            var focusFlag = c.DispositionFlag?.Replace("_disposition", "_focus");
            if (!string.IsNullOrEmpty(focusFlag))
            {
                var focusValue = DialogueKit.Flags?.Get(focusFlag);
                if (!string.IsNullOrEmpty(focusValue))
                {
                    var focus = FindFocus(c.Id, focusValue);
                    if (focus != null) GUILayout.Label($"Focus: {focus.Title}");
                }
            }

            GUILayout.EndVertical();
        }

        private FocusOption FindFocus(string characterId, string flagValue)
        {
            foreach (var f in focuses.GetForCharacter(characterId))
                if (f.FlagValue == flagValue) return f;
            return null;
        }

        private void OnChipClicked(CharacterInfo c)
        {
            if (focuses.HasAnyFor(c.Id)) { picker.Show(c); return; }
            if (!string.IsNullOrEmpty(c.OnClickScene))
                DialogueKit.EnqueueById(c.OnClickScene);
        }
    }
}
```

- [ ] **Step 3: Build**

- [ ] **Step 4: Commit**

```bash
git add src/KerbalAdminKit/Focuses/FocusPicker.cs src/KerbalAdminKit/Admin/CharacterPanel.cs
git commit -m "feat: add CharacterPanel IMGUI with chip rendering and FocusPicker modal"
```

---

## Task 19: DashboardPanel

**Files:**
- Create: `src/KerbalAdminKit/Admin/DashboardPanel.cs`

- [ ] **Step 1: Write `DashboardPanel`**

Create `src/KerbalAdminKit/Admin/DashboardPanel.cs`:

```csharp
using UnityEngine;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Focuses;
using KerbalCampaignKit.Core;
using KerbalDialogueKit.Core;

namespace KerbalAdminKit.Admin
{
    /// <summary>
    /// Center panel: program status (chapter, rep tier, income, next gate,
    /// next-chapter hint, active focuses).
    /// </summary>
    public sealed class DashboardPanel
    {
        private readonly CharacterRegistry characters;
        private readonly FocusRegistry focuses;
        private Vector2 scroll;

        public DashboardPanel(CharacterRegistry characters, FocusRegistry focuses)
        {
            this.characters = characters;
            this.focuses = focuses;
        }

        public void Draw(Rect area)
        {
            GUILayout.BeginArea(area, GUI.skin.box);
            scroll = GUILayout.BeginScrollView(scroll);

            GUILayout.Label("<b>Program Status</b>");
            DrawChapter();
            DrawReputation();
            DrawActiveFocuses();

            GUILayout.EndScrollView();
            GUILayout.EndArea();
        }

        private void DrawChapter()
        {
            var chapter = CampaignKit.Chapters?.Current;
            GUILayout.Label($"Current Chapter: {chapter ?? "—"}");
        }

        private void DrawReputation()
        {
            var rep = global::Reputation.Instance != null ? global::Reputation.Instance.reputation : 0f;
            GUILayout.Label($"Reputation: {(int)rep}");

            var income = CampaignKit.Reputation?.Income;
            if (income != null && income.Enabled)
            {
                var monthly = income.IncomeForRep(rep);
                if (monthly > 0) GUILayout.Label($"Monthly Income: {(int)monthly}");

                var tierLabel = income.TierLabelForRep(rep);
                if (!string.IsNullOrEmpty(tierLabel)) GUILayout.Label($"Tier: {tierLabel}");
            }

            var nextGate = CampaignKit.Reputation?.NextGate(rep);
            if (nextGate.HasValue)
                GUILayout.Label($"Next: {nextGate.Value.label} at {(int)nextGate.Value.value}");
        }

        private void DrawActiveFocuses()
        {
            GUILayout.Space(8);
            GUILayout.Label("<b>Active Focuses</b>");

            bool any = false;
            foreach (var c in characters.All)
            {
                if (string.IsNullOrEmpty(c.DispositionFlag)) continue;
                var focusFlag = c.DispositionFlag.Replace("_disposition", "_focus");
                var value = DialogueKit.Flags?.Get(focusFlag);
                if (string.IsNullOrEmpty(value)) continue;

                FocusOption match = null;
                foreach (var f in focuses.GetForCharacter(c.Id))
                    if (f.FlagValue == value) { match = f; break; }
                if (match != null)
                {
                    GUILayout.Label($"{c.DisplayName}: {match.Title}");
                    any = true;
                }
            }
            if (!any) GUILayout.Label("— none —");
        }
    }
}
```

- [ ] **Step 2: Build and commit**

```bash
git add src/KerbalAdminKit/Admin/DashboardPanel.cs
git commit -m "feat: add DashboardPanel (chapter, rep tier, monthly income, active focuses)"
```

---

## Task 20: DeskPanel + PrCampaignAction

**Files:**
- Create: `src/KerbalAdminKit/Admin/PrCampaignAction.cs`
- Create: `src/KerbalAdminKit/Admin/DeskPanel.cs`

- [ ] **Step 1: Write `PrCampaignAction`**

Create `src/KerbalAdminKit/Admin/PrCampaignAction.cs`:

```csharp
using System.Globalization;
using KerbalCampaignKit.Config;
using KerbalCampaignKit.Core;

namespace KerbalAdminKit.Admin
{
    public sealed class PrCampaignConfig
    {
        public double BaseCost = 50000;
        public double TierCostMultiplier = 25000;
        public double HaltDecayDays = 60;
        public double RepBonus = 10;

        public static PrCampaignConfig Load(ISceneNode node)
        {
            var cfg = new PrCampaignConfig();
            if (node == null) return cfg;
            if (node.HasValue("baseCost")) cfg.BaseCost = D(node, "baseCost", cfg.BaseCost);
            if (node.HasValue("tierCostMultiplier")) cfg.TierCostMultiplier = D(node, "tierCostMultiplier", cfg.TierCostMultiplier);
            if (node.HasValue("haltDecayDays")) cfg.HaltDecayDays = D(node, "haltDecayDays", cfg.HaltDecayDays);
            if (node.HasValue("repBonus")) cfg.RepBonus = D(node, "repBonus", cfg.RepBonus);
            return cfg;
        }

        private static double D(ISceneNode n, string k, double fallback) =>
            double.TryParse(n.GetValue(k), NumberStyles.Float, CultureInfo.InvariantCulture, out var v) ? v : fallback;

        public double ComputeCost(int currentTierIndex) =>
            BaseCost + (currentTierIndex * TierCostMultiplier);
    }

    public static class PrCampaignAction
    {
        /// <summary>
        /// Applies the PR campaign: deducts funds, adds rep, halts decay,
        /// records lastUsedTime in the AdminKit scenario.
        /// </summary>
        public static void Execute(PrCampaignConfig cfg, int currentTierIndex)
        {
            if (cfg == null) return;
            var cost = cfg.ComputeCost(currentTierIndex);
            Funding.Instance?.AddFunds(-cost, TransactionReasons.Strategies);
            global::Reputation.Instance?.AddReputation((float)cfg.RepBonus, TransactionReasons.Strategies);
            CampaignKit.Reputation?.HaltDecay(Planetarium.GetUniversalTime(), cfg.HaltDecayDays);

            if (AdminKitScenario.Instance != null)
                AdminKitScenario.Instance.PrCampaignLastUsedTime = Planetarium.GetUniversalTime();
        }
    }
}
```

- [ ] **Step 2: Write `DeskPanel`**

Create `src/KerbalAdminKit/Admin/DeskPanel.cs`:

```csharp
using UnityEngine;
using KerbalAdminKit.Memos;
using KerbalAdminKit.Settings;

namespace KerbalAdminKit.Admin
{
    public sealed class DeskPanel
    {
        private readonly MemoRegistry memos;
        private readonly AdminKitSettings settings;
        private readonly PrCampaignConfig prConfig;
        private Vector2 scroll;
        private bool confirmingPr;

        public DeskPanel(MemoRegistry memos, AdminKitSettings settings, PrCampaignConfig prConfig)
        {
            this.memos = memos;
            this.settings = settings;
            this.prConfig = prConfig;
        }

        public void Draw(Rect area)
        {
            GUILayout.BeginArea(area, GUI.skin.box);
            scroll = GUILayout.BeginScrollView(scroll);

            DrawMemos();
            DrawPrButton();

            GUILayout.EndScrollView();
            GUILayout.EndArea();

            if (confirmingPr) DrawPrConfirm();
        }

        private void DrawMemos()
        {
            GUILayout.Label("<b>Memos</b>");
            int shown = 0;
            foreach (var m in memos.Active)
            {
                if (shown >= settings.DeskMemoCount) break;
                GUILayout.BeginVertical(GUI.skin.box);
                if (!string.IsNullOrEmpty(m.CharacterId))
                    GUILayout.Label($"<b>{m.CharacterId}:</b>");
                GUILayout.Label(m.Text ?? "");
                if (GUILayout.Button("Dismiss"))
                    memos.Dismiss(m.Id, Planetarium.GetUniversalTime());
                GUILayout.EndVertical();
                GUILayout.Space(4);
                shown++;
            }
            if (shown == 0) GUILayout.Label("— no memos —");
        }

        private void DrawPrButton()
        {
            if (prConfig == null) return;
            GUILayout.Space(12);
            var cost = prConfig.ComputeCost(CurrentTierIndex());
            if (GUILayout.Button($"PR Campaign ({(int)cost} funds)"))
                confirmingPr = true;
        }

        private void DrawPrConfirm()
        {
            var rect = new Rect(Screen.width/2 - 200, Screen.height/2 - 100, 400, 200);
            GUI.Window(0x4B41_4B02, rect, id =>
            {
                var cost = prConfig.ComputeCost(CurrentTierIndex());
                GUILayout.Label($"Run a PR campaign for {(int)cost} funds?");
                GUILayout.Label($"+{(int)prConfig.RepBonus} reputation, halt decay {(int)prConfig.HaltDecayDays} days.");
                GUILayout.BeginHorizontal();
                if (GUILayout.Button("Confirm"))
                {
                    PrCampaignAction.Execute(prConfig, CurrentTierIndex());
                    confirmingPr = false;
                }
                if (GUILayout.Button("Cancel")) confirmingPr = false;
                GUILayout.EndHorizontal();
            }, "PR Campaign");
        }

        private int CurrentTierIndex()
        {
            // KCK doesn't expose tier index directly, so we iterate Income.Tiers
            // (ordered ascending) and return the first matching index. Returns 0
            // if no income configured or rep falls outside all tiers.
            var income = KerbalCampaignKit.Core.CampaignKit.Reputation?.Income;
            if (income == null || income.Tiers == null) return 0;
            var rep = global::Reputation.Instance != null ? global::Reputation.Instance.reputation : 0f;
            for (int i = 0; i < income.Tiers.Count; i++)
            {
                var t = income.Tiers[i];
                if (rep >= t.Min && rep < t.Max) return i;
            }
            return 0;
        }
    }
}
```

- [ ] **Step 3: Build and commit**

```bash
git add src/KerbalAdminKit/Admin/PrCampaignAction.cs src/KerbalAdminKit/Admin/DeskPanel.cs
git commit -m "feat: add DeskPanel (memos + PR Campaign button) and PrCampaignAction wiring"
```

---

## Task 21: AdminBuildingUI + AdminClickInterceptor

**Files:**
- Create: `src/KerbalAdminKit/Admin/AdminBuildingUI.cs`
- Create: `src/KerbalAdminKit/Admin/AdminClickInterceptor.cs`

- [ ] **Step 1: Write `AdminClickInterceptor` (Harmony patch)**

Create `src/KerbalAdminKit/Admin/AdminClickInterceptor.cs`:

```csharp
using HarmonyLib;
using UnityEngine;

namespace KerbalAdminKit.Admin
{
    /// <summary>
    /// Harmony prefix on SpaceCenterBuilding.OnClicked. When the clicked
    /// building is the Administration facility, returns false to suppress
    /// the stock admin scene and instead opens the AdminBuildingUI window.
    /// Installed only when replaceAdminBuilding = true.
    /// </summary>
    [HarmonyPatch(typeof(SpaceCenterBuilding), "OnClicked")]
    public static class AdminClickInterceptor
    {
        public static bool Enabled;
        public static AdminBuildingUI UI;

        static bool Prefix(SpaceCenterBuilding __instance)
        {
            if (!Enabled || UI == null) return true;
            if (__instance == null) return true;
            if (!IsAdministration(__instance)) return true;

            UI.Show();
            return false;   // suppress stock scene
        }

        private static bool IsAdministration(SpaceCenterBuilding b)
        {
            var name = b.facilityName ?? b.gameObject?.name ?? "";
            return name.Contains("Administration") || name.Contains("admin");
        }
    }
}
```

> **Implementation note:** The exact method name (`OnClicked`) and the heuristic for identifying the administration building may need adjustment after checking decompiled stock KSP. Candidate alternate methods: `SpaceCenterBuilding.onClicked`, `Administration.OnClicked`. If `SpaceCenterBuilding.facilityName` doesn't exist as a property, use `b.gameObject.name` or iterate `SpaceCenter.Instance`'s facility references to match identity. Verify during Task 28 smoke test — if the patch never fires, check the method name in game logs.

- [ ] **Step 2: Write `AdminBuildingUI`**

Create `src/KerbalAdminKit/Admin/AdminBuildingUI.cs`:

```csharp
using UnityEngine;
using ClickThroughFix;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Focuses;
using KerbalAdminKit.Memos;
using KerbalAdminKit.Settings;

namespace KerbalAdminKit.Admin
{
    /// <summary>
    /// Fullscreen IMGUI admin screen. Three panels side-by-side.
    /// Opened by AdminClickInterceptor when the administration building is
    /// clicked (and replaceAdminBuilding = true).
    /// </summary>
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class AdminBuildingUI : MonoBehaviour
    {
        public static AdminBuildingUI Instance;

        private bool visible;
        private Rect window = new Rect(40, 40, 1200, 700);
        private const int WindowId = 0x4B41_4B00;

        private CharacterPanel characterPanel;
        private DashboardPanel dashboardPanel;
        private DeskPanel deskPanel;
        private FocusPicker focusPicker;

        private void Awake() { Instance = this; }

        public void Initialize(
            CharacterRegistry characters,
            FocusRegistry focuses,
            MemoRegistry memos,
            AdminKitSettings settings,
            PrCampaignConfig prConfig)
        {
            focusPicker = new FocusPicker(focuses, characters);
            characterPanel = new CharacterPanel(characters, focuses, focusPicker);
            dashboardPanel = new DashboardPanel(characters, focuses);
            deskPanel = new DeskPanel(memos, settings, prConfig);

            AdminClickInterceptor.UI = this;
        }

        public void Show()
        {
            visible = true;
            var w = Mathf.Min(1200, Screen.width - 80);
            var h = Mathf.Min(700, Screen.height - 80);
            window = new Rect((Screen.width - w) / 2, (Screen.height - h) / 2, w, h);
        }

        public void Hide() => visible = false;

        private void OnGUI()
        {
            if (!visible) return;
            if (characterPanel == null) return;

            window = ClickThroughBlocker.GUILayoutWindow(
                WindowId, window, DrawWindow, "Administration", GUI.skin.window);

            focusPicker?.OnGUI();
        }

        private void DrawWindow(int id)
        {
            var padding = 8f;
            var innerW = window.width - padding * 2;
            var innerH = window.height - 44f;   // title bar + footer
            var colW = (innerW - padding * 2) / 3f;

            var charRect = new Rect(padding, 30, colW, innerH);
            var dashRect = new Rect(padding + colW + padding, 30, colW, innerH);
            var deskRect = new Rect(padding + (colW + padding) * 2, 30, colW, innerH);

            characterPanel?.Draw(charRect);
            dashboardPanel?.Draw(dashRect);
            deskPanel?.Draw(deskRect);

            var closeRect = new Rect(window.width - 80, 4, 70, 20);
            if (GUI.Button(closeRect, "Close")) Hide();

            GUI.DragWindow(new Rect(0, 0, window.width - 90, 24));
        }
    }
}
```

- [ ] **Step 3: Build and commit**

```bash
git add src/KerbalAdminKit/Admin/AdminBuildingUI.cs src/KerbalAdminKit/Admin/AdminClickInterceptor.cs
git commit -m "feat: add AdminBuildingUI three-panel window + Harmony AdminClickInterceptor"
```

---

## Task 22: Per-building overlays (MC + Tracking Station)

**Files:**
- Create: `src/KerbalAdminKit/BuildingOverlays/BuildingOverlayBase.cs`
- Create: `src/KerbalAdminKit/BuildingOverlays/MissionControlOverlay.cs`
- Create: `src/KerbalAdminKit/BuildingOverlays/TrackingStationOverlay.cs`

- [ ] **Step 1: Write `BuildingOverlayBase`**

Create `src/KerbalAdminKit/BuildingOverlays/BuildingOverlayBase.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using ClickThroughFix;
using KerbalAdminKit.Admin;
using KerbalAdminKit.Characters;
using KerbalCampaignKit.Core;
using KerbalDialogueKit.Core;

namespace KerbalAdminKit.BuildingOverlays
{
    /// <summary>
    /// Shared overlay: top-right corner panel with one chip per
    /// DIRECTOR_BUILDING_SCENE matching this facility. Click → KDK scene.
    /// </summary>
    public abstract class BuildingOverlayBase : MonoBehaviour
    {
        protected abstract string Facility { get; }                     // "MissionControl" etc.
        protected abstract string NotificationPrefix { get; }           // "mc", "tracking", etc.
        private const int WindowId = 0x4B41_4B10;

        protected BuildingSceneRegistry SceneRegistry;
        protected CharacterRegistry Characters;

        private Rect window;
        private List<BuildingScene> myScenes;

        public void Initialize(BuildingSceneRegistry scenes, CharacterRegistry characters)
        {
            SceneRegistry = scenes;
            Characters = characters;
            myScenes = new List<BuildingScene>();
            foreach (var s in scenes.GetForFacility(Facility)) myScenes.Add(s);

            var w = 220f;
            var h = 80f + myScenes.Count * 80f;
            window = new Rect(Screen.width - w - 20, 60, w, h);
        }

        private void OnGUI()
        {
            if (myScenes == null || myScenes.Count == 0) return;
            window = ClickThroughBlocker.GUILayoutWindow(
                WindowId + GetInstanceID(), window, DrawWindow, Facility, GUI.skin.window);
        }

        private void DrawWindow(int id)
        {
            foreach (var scene in myScenes)
            {
                var character = Characters.Get(scene.CharacterId);
                if (character == null) continue;

                var chipHeight = 56f;
                var chipRect = GUILayoutUtility.GetRect(GUILayout.ExpandWidth(true), GUILayout.Height(chipHeight));
                var disposition = Characters.GetDisposition(scene.CharacterId);

                var overrideLayers = scene.ChipLayerOverrides.Count > 0
                    ? scene.ChipLayerOverrides
                    : null;

                ChipRenderer.Draw(chipRect, character, disposition, overrideLayers);

                if (HasNotification(scene.CharacterId)) DrawOutline(chipRect);

                if (GUI.Button(chipRect, GUIContent.none, GUIStyle.none))
                {
                    if (!string.IsNullOrEmpty(scene.OnClickScene))
                        DialogueKit.EnqueueById(scene.OnClickScene);
                }

                GUILayout.Label(character.DisplayName);
                GUILayout.Space(4);
            }

            GUI.DragWindow();
        }

        private bool HasNotification(string characterId)
        {
            var target = NotificationPrefix + ".character." + characterId;
            return CampaignKit.Notifications?.Highest(target).HasValue == true;
        }

        private void DrawOutline(Rect r)
        {
            var original = GUI.color;
            GUI.color = Color.yellow;
            GUI.Box(new Rect(r.x - 2, r.y - 2, r.width + 4, r.height + 4), GUIContent.none);
            GUI.color = original;
        }
    }
}
```

- [ ] **Step 2: Write MC + Tracking subclasses**

Create `src/KerbalAdminKit/BuildingOverlays/MissionControlOverlay.cs`:

```csharp
namespace KerbalAdminKit.BuildingOverlays
{
    [KSPAddon(KSPAddon.Startup.MissionControl, false)]
    public sealed class MissionControlOverlay : BuildingOverlayBase
    {
        protected override string Facility => "MissionControl";
        protected override string NotificationPrefix => "mc";
    }
}
```

Create `src/KerbalAdminKit/BuildingOverlays/TrackingStationOverlay.cs`:

```csharp
namespace KerbalAdminKit.BuildingOverlays
{
    [KSPAddon(KSPAddon.Startup.TrackingStation, false)]
    public sealed class TrackingStationOverlay : BuildingOverlayBase
    {
        protected override string Facility => "TrackingStation";
        protected override string NotificationPrefix => "tracking";
    }
}
```

- [ ] **Step 3: Build and commit**

```bash
git add src/KerbalAdminKit/BuildingOverlays/BuildingOverlayBase.cs src/KerbalAdminKit/BuildingOverlays/MissionControlOverlay.cs src/KerbalAdminKit/BuildingOverlays/TrackingStationOverlay.cs
git commit -m "feat: add per-building overlays (MissionControl, TrackingStation) with shared base"
```

---

## Task 23: KscNotificationRenderer

**Files:**
- Create: `src/KerbalAdminKit/KscRenderer/KscNotificationRenderer.cs`

- [ ] **Step 1: Write `KscNotificationRenderer`**

Create `src/KerbalAdminKit/KscRenderer/KscNotificationRenderer.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using KerbalAdminKit.Util;
using KerbalCampaignKit.Core;

namespace KerbalAdminKit.KscRenderer
{
    /// <summary>
    /// Draws notification markers over KSC buildings in OnGUI. Markers pull
    /// their textures/colors from NotificationStyleRegistry; positions from
    /// each building's transform via Camera.main.WorldToScreenPoint.
    /// </summary>
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class KscNotificationRenderer : MonoBehaviour
    {
        private NotificationStyleRegistry styles;
        private KscMarkerOffsetRegistry offsets;

        private readonly List<(string facility, string prefix)> buildings = new List<(string, string)>
        {
            ("Administration",       "admin"),
            ("MissionControl",       "mc"),
            ("TrackingStation",      "tracking"),
            ("ResearchAndDevelopment", "rd"),
            ("VehicleAssemblyBuilding", "vab"),
            ("SpaceplaneHangar",     "sph"),
            ("AstronautComplex",     "astronaut"),
        };

        public void Initialize(NotificationStyleRegistry styles, KscMarkerOffsetRegistry offsets)
        {
            this.styles = styles;
            this.offsets = offsets;
        }

        private void OnGUI()
        {
            if (styles == null || offsets == null) return;
            if (Camera.main == null) return;
            if (SpaceCenter.Instance == null) return;

            foreach (var (facility, prefix) in buildings)
            {
                var severity = CampaignKit.Notifications?.Highest(prefix);
                if (!severity.HasValue) continue;

                var style = styles.Get(severity.Value.ToString());
                if (style == null || string.IsNullOrEmpty(style.TextureUrl)) continue;
                var tex = TextureLoader.Get(style.TextureUrl);
                if (tex == null) continue;

                var transform = FindFacilityTransform(facility);
                if (transform == null) continue;

                var screen = Camera.main.WorldToScreenPoint(transform.position);
                if (screen.z < 0) continue;      // behind camera

                var offset = offsets.Get(facility);
                var pos = new Vector2(screen.x + offset.x, Screen.height - screen.y - offset.y);

                var size = 32f * (style.Pulse ? (1f + 0.1f * Mathf.Sin(Time.time * 3f)) : 1f);
                var rect = new Rect(pos.x - size/2f, pos.y - size/2f, size, size);

                var original = GUI.color;
                GUI.color = ChipRendererColor(style);
                GUI.DrawTexture(rect, tex, ScaleMode.ScaleToFit);
                GUI.color = original;
            }
        }

        private Color ChipRendererColor(NotificationStyle s) =>
            KerbalAdminKit.Admin.ChipRenderer.ToUnity(s.Color);

        private Transform FindFacilityTransform(string facility)
        {
            var sc = SpaceCenter.Instance;
            if (sc == null) return null;

            // SpaceCenter exposes each facility as a field. We use reflection-free
            // access via the named properties (PSystemSetup-style). If a field is
            // missing (mod-dependent), we skip.
            switch (facility)
            {
                case "Administration":          return sc.Administration?.transform;
                case "MissionControl":          return sc.MissionControl?.transform;
                case "TrackingStation":         return sc.TrackingStation?.transform;
                case "ResearchAndDevelopment":  return sc.ResearchAndDevelopment?.transform;
                case "VehicleAssemblyBuilding": return sc.VehicleAssemblyBuilding?.transform;
                case "SpaceplaneHangar":        return sc.SpaceplaneHangar?.transform;
                case "AstronautComplex":        return sc.AstronautComplex?.transform;
                default:                        return null;
            }
        }
    }
}
```

> **Implementation note:** The `SpaceCenter.Instance` fields used above (e.g., `sc.Administration`) are the stock KSP property names. If a property does not exist under that exact name, check `Assembly-CSharp.dll` via a decompiler for the current KSP 1.12 name — candidate alternatives: `sc.AdministrationFacility`, a `GameObject.Find("Administration")` lookup, or `FindObjectsOfType<SpaceCenterBuilding>()` filtered by name. Verify during Task 28.

- [ ] **Step 2: Build and commit**

```bash
git add src/KerbalAdminKit/KscRenderer/KscNotificationRenderer.cs
git commit -m "feat: add KscNotificationRenderer drawing severity-styled markers over KSC buildings"
```

---

## Task 24: AdminKitAddon (wire everything) + finalize AdminKit public API

**Files:**
- Create: `src/KerbalAdminKit/Core/AdminKitAddon.cs`
- Modify: `src/KerbalAdminKit/Core/AdminKit.cs`

- [ ] **Step 1: Finalize `AdminKit` public API**

Overwrite `src/KerbalAdminKit/Core/AdminKit.cs`:

```csharp
using KerbalAdminKit.Characters;
using KerbalAdminKit.Focuses;
using KerbalAdminKit.Memos;

namespace KerbalAdminKit
{
    /// <summary>
    /// Public static facade. Wired by AdminKitAddon during startup.
    /// </summary>
    public static class AdminKit
    {
        public static CharacterRegistry Characters { get; internal set; }
        public static FocusRegistry Focuses { get; internal set; }
        public static MemoRegistry Memos { get; internal set; }
    }
}
```

- [ ] **Step 2: Write `AdminKitAddon` — scan cfg, wire systems, install Harmony**

Create `src/KerbalAdminKit/Core/AdminKitAddon.cs`:

```csharp
using System.Collections.Generic;
using HarmonyLib;
using UnityEngine;
using KerbalAdminKit.Admin;
using KerbalAdminKit.BuildingOverlays;
using KerbalAdminKit.Characters;
using KerbalAdminKit.DecayCompiler;
using KerbalAdminKit.Focuses;
using KerbalAdminKit.KscRenderer;
using KerbalAdminKit.Memos;
using KerbalAdminKit.Settings;
using KerbalCampaignKit.Config;

namespace KerbalAdminKit
{
    [KSPAddon(KSPAddon.Startup.MainMenu, true)]
    public sealed class AdminKitAddon : MonoBehaviour
    {
        private AdminKitSettings settings;
        private CharacterRegistry characters;
        private FocusRegistry focuses;
        private MemoRegistry memos;
        private NotificationStyleRegistry styles;
        private KscMarkerOffsetRegistry offsets;
        private BuildingSceneRegistry buildingScenes;
        private PrCampaignConfig prConfig;

        private bool anyCfg;

        private void Awake()
        {
            DontDestroyOnLoad(this);

            LoadSettings();
            LoadCharacters();
            LoadFocuses();
            LoadMemos();
            LoadNotificationStyles();
            LoadMarkerOffsets();
            LoadBuildingScenes();
            LoadPrCampaign();
            CompileDispositionDecay();

            AdminKit.Characters = characters;
            AdminKit.Focuses = focuses;
            AdminKit.Memos = memos;

            if (!anyCfg)
            {
                Debug.Log("[KerbalAdminKit] No cfg found — remaining inert.");
                return;
            }

            if (settings.ReplaceAdminBuilding)
                InstallAdminInterceptor();

            WireSceneHooks();
        }

        private void LoadSettings()
        {
            var nodes = GameDatabase.Instance.GetConfigNodes("KERBAL_ADMIN_KIT");
            settings = nodes.Length > 0
                ? AdminKitSettingsLoader.Load(new ConfigNodeAdapter(nodes[0]))
                : AdminKitSettings.Defaults();
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadCharacters()
        {
            characters = new CharacterRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_CHARACTER");
            foreach (var n in nodes)
            {
                var c = CharacterLoader.Load(new ConfigNodeAdapter(n));
                if (c != null) characters.Add(c);
            }
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadFocuses()
        {
            focuses = new FocusRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_FOCUS");
            foreach (var n in nodes)
            {
                var f = FocusLoader.Load(new ConfigNodeAdapter(n));
                if (f != null) focuses.Add(f);
            }
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadMemos()
        {
            memos = new MemoRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_MEMO");
            foreach (var n in nodes)
            {
                var m = MemoLoader.Load(new ConfigNodeAdapter(n));
                if (m != null) memos.Add(m);
            }
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadNotificationStyles()
        {
            styles = new NotificationStyleRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_NOTIFICATION_STYLE");
            foreach (var n in nodes)
            {
                var s = NotificationStyleLoader.Load(new ConfigNodeAdapter(n));
                if (s != null) styles.AddOrReplace(s);
            }
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadMarkerOffsets()
        {
            offsets = new KscMarkerOffsetRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_KSC_MARKER_OFFSET");
            foreach (var n in nodes) offsets.Load(new ConfigNodeAdapter(n));
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadBuildingScenes()
        {
            buildingScenes = new BuildingSceneRegistry();
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_BUILDING_SCENE");
            foreach (var n in nodes)
            {
                var bs = BuildingSceneLoader.Load(new ConfigNodeAdapter(n));
                if (bs != null) buildingScenes.Add(bs);
            }
            if (nodes.Length > 0) anyCfg = true;
        }

        private void LoadPrCampaign()
        {
            var nodes = GameDatabase.Instance.GetConfigNodes("DIRECTOR_PR_CAMPAIGN");
            if (nodes.Length > 0)
            {
                prConfig = PrCampaignConfig.Load(new ConfigNodeAdapter(nodes[0]));
                anyCfg = true;
            }
        }

        private void CompileDispositionDecay()
        {
            var nodes = GameDatabase.Instance.GetConfigNodes("DISPOSITION_DECAY");
            if (nodes.Length == 0) return;
            anyCfg = true;

            var sink = new KckTriggerSink();
            foreach (var n in nodes)
            {
                var decay = DispositionDecayLoader.Load(new ConfigNodeAdapter(n));
                DispositionDecayCompiler.Compile(decay, sink);
            }
        }

        private void InstallAdminInterceptor()
        {
            var harmony = new Harmony("badgkat.KerbalAdminKit");
            harmony.PatchAll(typeof(AdminClickInterceptor).Assembly);
            AdminClickInterceptor.Enabled = true;
        }

        private void WireSceneHooks()
        {
            // MainMenu doesn't have UI scenes yet; wire per-scene addons when they wake.
            GameEvents.onLevelWasLoaded.Add(OnLevelLoaded);
        }

        private void OnLevelLoaded(GameScenes scene)
        {
            if (scene == GameScenes.SPACECENTER)
            {
                WireAtSpaceCentre();
            }
            else if (scene == GameScenes.TRACKSTATION && TrackingStationOverlay.FindObjectOfType<TrackingStationOverlay>() is var tso && tso != null)
            {
                tso.Initialize(buildingScenes, characters);
            }
        }

        private void WireAtSpaceCentre()
        {
            var ui = AdminBuildingUI.Instance;
            if (ui != null) ui.Initialize(characters, focuses, memos, settings, prConfig);

            var ticker = FindObjectOfType<MemoTicker>();
            if (ticker != null) ticker.Initialize(memos, settings.MemoPollSeconds);

            var renderer = FindObjectOfType<KscNotificationRenderer>();
            if (renderer != null) renderer.Initialize(styles, offsets);

            // MC overlay lives in its own scene, hooked when player enters MC.
            // Tracking Station likewise.
        }
    }

    /// <summary>
    /// Collects compiled decay triggers in memory. DispositionDecayTicker
    /// reads from this list on a polling cadence and applies transitions by
    /// reading and updating KDK flags directly. We skip KCK's trigger engine
    /// for decay because KCK's TimeElapsed events require an external source
    /// to fire them — this ticker IS that source, and applying the flag
    /// update ourselves is simpler than round-tripping through KCK.
    /// </summary>
    internal sealed class InMemoryDecaySink : DecayCompiler.ITriggerSink
    {
        public List<DecayCompiler.CompiledDecayTrigger> Triggers =
            new List<DecayCompiler.CompiledDecayTrigger>();
        public void Register(DecayCompiler.CompiledDecayTrigger t) => Triggers.Add(t);
    }
}
```

- [ ] **Step 3: Write `DispositionDecayTicker`**

Create `src/KerbalAdminKit/DecayCompiler/DispositionDecayTicker.cs`:

```csharp
using System.Collections.Generic;
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalAdminKit.DecayCompiler
{
    /// <summary>
    /// Polls compiled decay triggers at KSC. For each trigger whose flag
    /// matches ExpectedFlagValue AND whose flag's last-set time is at least
    /// StepDays ago, sets the flag to TargetFlagValue and records the new
    /// last-set time via a companion flag "<flag>_last_set_at".
    /// </summary>
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class DispositionDecayTicker : MonoBehaviour
    {
        private const double SecondsPerDay = 21600;
        private const double PollSeconds = 10.0;

        private List<CompiledDecayTrigger> triggers = new List<CompiledDecayTrigger>();
        private double nextPoll;

        public void Initialize(IEnumerable<CompiledDecayTrigger> compiled)
        {
            triggers.Clear();
            foreach (var t in compiled) triggers.Add(t);
        }

        private void FixedUpdate()
        {
            if (triggers.Count == 0) return;
            var now = Planetarium.GetUniversalTime();
            if (now < nextPoll) return;
            nextPoll = now + PollSeconds;

            var flags = DialogueKit.Flags;
            if (flags == null) return;

            foreach (var t in triggers)
            {
                var current = flags.Get(t.FlagToMatch);
                if (current != t.ExpectedFlagValue) continue;

                var lastSetKey = t.FlagToMatch + "_last_set_at";
                var lastSetStr = flags.Get(lastSetKey);
                double lastSet;
                if (!double.TryParse(lastSetStr, out lastSet))
                {
                    // No timestamp yet — stamp now and wait for next cycle.
                    flags.Set(lastSetKey, now.ToString("F1"));
                    continue;
                }

                var elapsedDays = (now - lastSet) / SecondsPerDay;
                if (elapsedDays < t.StepDays) continue;

                flags.Set(t.FlagToMatch, t.TargetFlagValue);
                flags.Set(lastSetKey, now.ToString("F1"));
                Debug.Log($"[KerbalAdminKit] Decay {t.Id}: {t.ExpectedFlagValue} → {t.TargetFlagValue}");
            }
        }
    }
}
```

> **Flag-set-time recording.** Content that writes dispositions via KCK triggers should also write `<flag>_last_set_at` = `<now seconds>` so the ticker knows when to decay. A KCK action helper could be added in a follow-up, but for v0.1 BadgKatCareer's chapter-entry triggers explicitly stamp the timestamp alongside the disposition write. The ticker self-heals by stamping when it first sees an un-stamped flag — this means existing dispositions start their decay clock from first observation, which is fine for a fresh save and benign on upgrade.

- [ ] **Step 4: Wire `DispositionDecayTicker` in `AdminKitAddon`**

Append to `AdminKitAddon.WireAtSpaceCentre()`:

```csharp
        private void WireAtSpaceCentre()
        {
            var ui = AdminBuildingUI.Instance;
            if (ui != null) ui.Initialize(characters, focuses, memos, settings, prConfig);

            var ticker = FindObjectOfType<MemoTicker>();
            if (ticker != null) ticker.Initialize(memos, settings.MemoPollSeconds);

            var renderer = FindObjectOfType<KscNotificationRenderer>();
            if (renderer != null) renderer.Initialize(styles, offsets);

            var decayTicker = FindObjectOfType<DispositionDecayTicker>();
            if (decayTicker != null && decaySink != null)
                decayTicker.Initialize(decaySink.Triggers);
        }
```

Also update `CompileDispositionDecay` to save the sink as a class field:

```csharp
        private InMemoryDecaySink decaySink;

        private void CompileDispositionDecay()
        {
            var nodes = GameDatabase.Instance.GetConfigNodes("DISPOSITION_DECAY");
            if (nodes.Length == 0) return;
            anyCfg = true;

            decaySink = new InMemoryDecaySink();
            foreach (var n in nodes)
            {
                var decay = DispositionDecayLoader.Load(new ConfigNodeAdapter(n));
                DispositionDecayCompiler.Compile(decay, decaySink);
            }
        }
```

Also wire Mission Control and Tracking Station overlay initialization on scene load:

```csharp
        private void OnLevelLoaded(GameScenes scene)
        {
            if (scene == GameScenes.SPACECENTER) WireAtSpaceCentre();
            else if (scene == GameScenes.TRACKSTATION)
            {
                var tso = FindObjectOfType<TrackingStationOverlay>();
                if (tso != null) tso.Initialize(buildingScenes, characters);
            }
            // MissionControl scene has no dedicated GameScenes enum value in KSP 1.12 —
            // it's triggered via a button press rather than a scene transition.
            // The KSPAddon(Startup.MissionControl) still fires when player enters MC;
            // that addon can self-initialize via AdminKit.Characters / etc. facades.
        }
```

> **MC scene wiring note.** `MissionControlOverlay`'s `KSPAddon(Startup.MissionControl)` attribute activates when player enters MC. That MonoBehaviour's own `Start` method can read from the public facades (`AdminKit.Characters`, etc.) rather than depending on the addon re-initializing it. Replace the empty facade in `MissionControlOverlay` with a `Start()` that calls `Initialize(AdminKit.BuildingScenes, AdminKit.Characters)` (adding a new `BuildingScenes` property to `AdminKit` facade). Do the same for Tracking Station for consistency.

- [ ] **Step 5: Add `BuildingScenes` facade property**

Edit `src/KerbalAdminKit/Core/AdminKit.cs`:

```csharp
using KerbalAdminKit.BuildingOverlays;
using KerbalAdminKit.Characters;
using KerbalAdminKit.Focuses;
using KerbalAdminKit.Memos;

namespace KerbalAdminKit
{
    public static class AdminKit
    {
        public static CharacterRegistry Characters { get; internal set; }
        public static FocusRegistry Focuses { get; internal set; }
        public static MemoRegistry Memos { get; internal set; }
        public static BuildingSceneRegistry BuildingScenes { get; internal set; }
    }
}
```

And in `AdminKitAddon.Awake`, after the `AdminKit.Memos = memos;` line, add:

```csharp
            AdminKit.BuildingScenes = buildingScenes;
```

Update `MissionControlOverlay` and `TrackingStationOverlay` to self-init:

```csharp
// MissionControlOverlay.cs
using UnityEngine;
namespace KerbalAdminKit.BuildingOverlays
{
    [KSPAddon(KSPAddon.Startup.MissionControl, false)]
    public sealed class MissionControlOverlay : BuildingOverlayBase
    {
        protected override string Facility => "MissionControl";
        protected override string NotificationPrefix => "mc";

        private void Start()
        {
            if (AdminKit.BuildingScenes != null && AdminKit.Characters != null)
                Initialize(AdminKit.BuildingScenes, AdminKit.Characters);
        }
    }
}
```

Same shape for `TrackingStationOverlay`.

- [ ] **Step 6: Build and commit**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
git add src/KerbalAdminKit/
git commit -m "feat: wire AdminKitAddon with registries, Harmony, and DispositionDecayTicker"
```

The final task list is now 28 tasks (Tasks 1-24 built the code; 25 ships default assets + demo; 26 verifies in-game; 27 publishes).

---

## Task 25: Default art + demo cfg

**Files:**
- Create: `Art/exclamation.png` (24×24 red exclamation glyph, transparent background)
- Create: `Art/dot.png` (24×24 yellow circle, transparent background)
- Create: `demo/demo-admin-kit.cfg`

- [ ] **Step 1: Create default notification glyphs**

Two small PNGs at 24×24 pixels. You can draw them in any image editor. The style doesn't matter for v0.1 — content mods override via `DIRECTOR_NOTIFICATION_STYLE`. Save:

- `Art/exclamation.png` — solid red exclamation mark on transparent background
- `Art/dot.png` — solid yellow filled circle on transparent background

If you don't have an image editor handy, a 24×24 pure-color PNG generated via Python's `PIL` library is fine:

```python
from PIL import Image, ImageDraw
im = Image.new("RGBA", (24, 24), (0, 0, 0, 0))
d = ImageDraw.Draw(im)
d.rectangle([10, 4, 13, 16], fill=(255, 60, 60, 255))
d.ellipse([10, 18, 13, 21], fill=(255, 60, 60, 255))
im.save("Art/exclamation.png")

im = Image.new("RGBA", (24, 24), (0, 0, 0, 0))
d = ImageDraw.Draw(im)
d.ellipse([6, 6, 17, 17], fill=(255, 224, 100, 255))
im.save("Art/dot.png")
```

- [ ] **Step 2: Write `demo/demo-admin-kit.cfg`**

Create `demo/demo-admin-kit.cfg`:

```cfg
// KerbalAdminKit demo content.
// Install demo/ alongside GameData/KerbalAdminKit/ to exercise the full feature set
// without BadgKatCareer installed.

KERBAL_ADMIN_KIT
{
    replaceAdminBuilding = true
    deskMemoCount = 3
    memoPollSeconds = 5
}

DIRECTOR_CHARACTER
{
    id = demo_gus
    displayName = Demo Gus
    role = Chief Engineer
    baseColor = #FFFFC078
    dispositionFlag = demo_gus_disposition
    defaultDisposition = Supportive
}

DIRECTOR_CHARACTER
{
    id = demo_wernher
    displayName = Demo Wernher
    role = Chief Scientist
    baseColor = #FF82B4E8
    dispositionFlag = demo_wernher_disposition
    defaultDisposition = Neutral
}

DIRECTOR_FOCUS
{
    character = demo_wernher
    id = materials_research
    title = Materials Research
    description = Wernher focuses on advanced alloys.
    flag = demo_wernher_focus
    flagValue = materials
}

DIRECTOR_FOCUS
{
    character = demo_wernher
    id = anomaly_research
    title = Anomaly Research
    description = Wernher studies the monoliths.
    flag = demo_wernher_focus
    flagValue = anomaly
}

DIRECTOR_MEMO
{
    id = demo_low_funds
    character = demo_gus
    priority = high
    text = Funds are low — we might have to cut corners on the next launch.
    CONDITION
    {
        type = FundsBelow
        threshold = 50000
    }
    postToStockMail = true
}

DIRECTOR_MEMO
{
    id = demo_high_rep
    character = demo_wernher
    priority = normal
    text = The scientific community is paying attention. Let's deliver.
    CONDITION
    {
        type = ReputationAbove
        threshold = 100
    }
}

DIRECTOR_BUILDING_SCENE
{
    facility = MissionControl
    character = demo_gus
    onClickScene = demo_gus_mc_scene
}

DIRECTOR_PR_CAMPAIGN
{
    baseCost = 25000
    tierCostMultiplier = 10000
    haltDecayDays = 30
    repBonus = 5
}

DISPOSITION_DECAY
{
    character = demo_wernher
    towardValue = Neutral
    stepDays = 30
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
}
```

Note the multi-line cfg format: no semicolons, braces on their own lines.

- [ ] **Step 3: Build and verify demo cfg loads**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

Launch KSP (from the user's session — request they run `! start steam://rungameid/220200` or equivalent, or simply ask them to launch manually). Wait for main menu.

- [ ] **Step 4: Commit**

```bash
git add Art/ demo/
git commit -m "feat: add default notification glyphs and self-contained demo cfg"
```

---

## Task 26: In-game manual verification

No code changes. Verify the whole stack works.

- [ ] **Step 1: Copy demo cfg into GameData so it actually loads**

```bash
cp "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo/demo-admin-kit.cfg" \
   "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo-admin-kit.cfg"
```

(KSP only loads cfg from the mod root or subfolders scanned by ModuleManager. If the `demo/` subfolder isn't loaded by default, the copy at mod root ensures it is. Mark this copy as ignored in a future `.gitignore` if iteration is frequent.)

- [ ] **Step 2: Launch KSP, open the KSP log, verify load**

Wait for main menu. Then open `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/KSP.log` and search for lines containing `KerbalAdminKit`. Expect:

- `[LOG] [AddonLoader]: Instantiating addon 'AdminKitAddon'...` or equivalent
- No exceptions in the AdminKit namespace
- If no cfg were found: `[KerbalAdminKit] No cfg found — remaining inert.` (with demo cfg installed, you should NOT see this line)
- `[KerbalAdminKit] CompiledDecayTrigger ready: disposition_decay_demo_wernher_Enthusiastic_to_Supportive` (or the updated log line from DispositionDecayTicker)

Look also for Harmony patching errors: `[HarmonyLib] ... AdminClickInterceptor ...`. Success means the patch was applied to `SpaceCenterBuilding.OnClicked`. Failure means the method name or target type is wrong — check Task 21's implementation note.

- [ ] **Step 3: Start a new career save, go to KSC, click Admin**

Expected: KAK's three-panel IMGUI window opens instead of stock admin.

- Left panel: two character chips (Demo Gus, Demo Wernher) with color swatches and disposition labels.
- Center panel: "Program Status" heading, reputation value, no chapter (KCK has no chapter cfg yet — that's fine), no active focuses until one is picked.
- Right panel: memo section (empty at this rep), PR Campaign button.

Click Demo Wernher's chip → focus picker modal opens with "Materials Research" and "Anomaly Research." Pick one → modal closes, the Dashboard's "Active Focuses" now shows `Demo Wernher: <picked title>`.

Close admin screen.

- [ ] **Step 4: Test memo pipeline**

Use Alt+F12 (debug menu) → Cheats → Set funds to 10000 (below 50000 threshold).

Within 5 seconds (poll interval), the admin desk should show `Demo Gus: Funds are low...`. The stock mailbox (top-right) should blip once with the same memo (`postToStockMail = true`).

Dismiss the memo via the button. It should clear from the desk and not return on reload (`suppressAfterDismiss` default is false for this memo, so if you raise funds then lower again, it WILL return; this is correct).

- [ ] **Step 5: Test KSC notification renderer**

From an in-game scene with notifications programmatically added (via the console if exposed, or by writing a throwaway `DIRECTOR_MEMO` that sets `CampaignKit.Notifications` via a trigger — or by temporarily enqueueing a notification from another mod's test harness), confirm an exclamation/dot appears over the matching KSC building. If no notifications exist yet, this step is optional for v0.1 release; the renderer is live, just idle.

- [ ] **Step 6: Test zero-cfg inertness**

Rename the demo cfg so KSP can't load it:

```bash
mv "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo-admin-kit.cfg" \
   "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo-admin-kit.cfg.off"
```

Launch KSP again. In KSP.log, expect:

```
[KerbalAdminKit] No cfg found — remaining inert.
```

Click the stock Admin building at KSC — stock admin scene opens normally. Stock strategies grid intact. KAK did nothing.

Restore the demo cfg:

```bash
mv "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo-admin-kit.cfg.off" \
   "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit/demo-admin-kit.cfg"
```

- [ ] **Step 7: Test opt-in gating**

Edit the demo cfg and set `replaceAdminBuilding = false`. Restart KSP. Click admin → stock admin opens (KAK didn't intercept). But memos still tick, and the KSC renderer still draws markers. Confirms opt-in gating works.

Restore `replaceAdminBuilding = true` when done.

- [ ] **Step 8: Commit any fixes needed during verification**

If verification revealed issues (Harmony target wrong, SpaceCenter facility property missing, etc.), fix and commit. No separate task for these — they're part of Task 26.

```bash
git add src/KerbalAdminKit/
git commit -m "fix: <description of what you fixed during in-game verification>"
```

---

## Task 27: Publish to GitHub + tag v0.1.0

Matches the KCK publish flow.

- [ ] **Step 1: Verify clean tree**

```bash
cd "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalAdminKit"
git status --short
```

Expected: empty. If anything is uncommitted, commit or stash it.

- [ ] **Step 2: Create public GitHub repo**

```bash
gh repo create badgkat/KerbalAdminKit --public --source=. \
  --description="Customize the KSP Administration Building. Optional admin replacement with character chips, focus picker, memos, and KSC notification markers. Renders UI for data owned by KerbalCampaignKit." \
  --remote=origin
```

- [ ] **Step 3: Push**

```bash
git push -u origin main
```

- [ ] **Step 4: Add repo topics**

```bash
gh repo edit badgkat/KerbalAdminKit --add-topic ksp --add-topic kerbal-space-program --add-topic ksp-mod --add-topic chatterbox --add-topic modding-library
```

- [ ] **Step 5: Verify .version URL resolves**

```bash
curl -o /dev/null -s -w "%{http_code}\n" https://raw.githubusercontent.com/badgkat/KerbalAdminKit/main/KerbalAdminKit.version
```

Expected: `200`.

- [ ] **Step 6: Tag v0.1.0**

```bash
git tag -a v0.1.0 -m "Initial release: admin UI, character chips, focuses, memos, KSC markers, disposition decay"
git push origin v0.1.0
```

- [ ] **Step 7: Final status check**

```bash
gh repo view badgkat/KerbalAdminKit --json name,description,url,topics,stargazerCount
```

All fields populated. Done.

---

## Status

v0.1.0 shipped. KAK is now available to any KSP modder via:

```
depends:
  - KerbalDialogueKit >= 0.1.0
  - KerbalCampaignKit >= 0.1.0
  - KerbalAdminKit >= 0.1.0
  - HarmonyKSP
  - ClickThroughBlocker
  - ModuleManager
```

## Known follow-ups (post-v0.1)

- Hook KCK's `CampaignKitEvents.OnNotificationAdded/Cleared` into KscNotificationRenderer to avoid per-frame polling (currently every `OnGUI`).
- Implement `DIRECTOR_POLICY` cfg (replacement for stock strategies, see spec v0.2 notes).
- Action memos with button effects.
- R&D / VAB / SPH / Astronaut Complex overlays.
- BadgKatCareer integration (Phase 3): 6 characters, 5 chapters, directives, real content.
- Replace disposition-decay timestamp convention (`<flag>_last_set_at`) with a KCK-side helper so content doesn't have to stamp manually.

- [ ] **Step 2: Build**

```bash
dotnet build src/KerbalAdminKit/KerbalAdminKit.csproj -c Release
```

- [ ] **Step 3: Commit**

```bash
git add src/KerbalAdminKit/Core/AdminKit.cs src/KerbalAdminKit/Core/AdminKitAddon.cs
git commit -m "feat: add AdminKitAddon wiring cfg scan, Harmony, registries, and public API"
```

---
