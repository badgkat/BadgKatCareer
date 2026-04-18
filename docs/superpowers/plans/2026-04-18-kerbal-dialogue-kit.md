# KerbalDialogueKit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a standalone utility library (`KerbalDialogueKit.dll`) that extends ChatterBox's dialogue rendering with code-triggered scenes, branching via flag-based `visibleIf`, mid-scene choice cards, and lifecycle callbacks. Foundational for BadgKatDirector and any future mod wanting character-driven interactive dialogue.

**Architecture:** A new C# mod at `GameData/KerbalDialogueKit/` in its own git repo. Reuses ChatterBox's rendering (portraits, line sequencing, audio) via reflection since its key classes are `internal sealed`. Adds a public `DialogueKit` static API, a `DialogueScene` builder, a flag system with a `visibleIf` expression parser, a custom choice overlay UI using KSP's IMGUI, and a `ScenarioModule` persisting flags per-save. Core logic (parser, scene model, flags) is unit-testable in an xUnit project without KSP.

**Tech Stack:** C# targeting .NET Framework 4.8 (KSP's Unity runtime), MSBuild/`dotnet` CLI for builds, xUnit for unit tests, reflection against ChatterBox internals with a planned PR to make those internals public (swap-in replacement later).

**Reference:** Full spec at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/BadgKatCareer/docs/superpowers/specs/2026-04-18-dialogue-kit-design.md`

---

## Context For The Engineer

**Working directory setup.** The user's KSP install is at `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/`. Mods are installed under `GameData/`. Each mod is its own folder and, in this project, its own git repo. We will create a new folder `GameData/KerbalDialogueKit/` and `git init` inside it. The **source code and build artifacts** (csproj, test project, README) live at the repo root; only `Plugins/` (compiled DLL), `KerbalDialogueKit.version`, and example `.cfg` files get installed to `GameData/` — matching the ChatterBox convention.

**KSP modding basics.** KSP 1.12 runs on Unity 2019 with .NET Framework 4.x. Plugin DLLs are placed in `GameData/<ModName>/Plugins/` and loaded at game startup. KSP exposes:
- `MonoBehaviour` + `[KSPAddon]` attribute to auto-load runtime components at specific scenes (MainMenu, SpaceCentre, Flight, etc.)
- `ScenarioModule` + `[KSPScenario]` attribute for per-save persistent state that saves/loads with the game
- `GameEvents` static class with `EventData<T>` fields for subscribing to game events (contract completed, vessel recovered, etc.)
- `ConfigNode` class for reading/writing KSP's cfg format (braces, `key = value`, nested nodes)
- `PopupDialog` / `DialogGUI*` classes for pop-up dialogs. For in-game overlays drawn every frame we use Unity's IMGUI (`OnGUI()` with `GUI.*` / `GUILayout.*`)

**ChatterBox internals (what we reflect against).**
ChatterBox is a single source file at [DymndLab/ChatterBox](https://github.com/DymndLab/ChatterBox). The key internal classes:
- `ChatterBox.ScenePopupDefinition` — data object describing a scene (title, characters, lines, audio, presentation mode)
- `ChatterBox.CharacterConfig` — portrait model, color, animation, name
- `ChatterBox.LineConfig` — speaker enum, text, animation, per-line audio, text color
- `ChatterBox.SceneSpeaker` — enum with values `A` and `B`
- `ChatterBox.ScenePresentationMode` — enum with `Dialogue` and `Action`
- `ChatterBox.ScenePopupController` — `MonoBehaviour` with a public static `Enqueue(ScenePopupDefinition)` method that queues a scene for display

Because these are `internal` we cannot reference them directly from another assembly. Reflection is the workaround: get `Type` via `Assembly.GetType("ChatterBox.ScenePopupDefinition")`, instantiate via `Activator.CreateInstance(type)`, set fields via `FieldInfo.SetValue`, invoke the Enqueue method via `MethodInfo.Invoke`.

We centralize reflection in one file (`ChatterBoxBridge.cs`) so it's swappable once our upstream PR lands.

**PR to ChatterBox.** Send early (Task 1) so it has the longest time to land. Scope: make `ScenePopupDefinition`, `CharacterConfig`, `LineConfig`, and `ScenePopupController.Enqueue` public. Non-breaking — existing consumers unaffected. If it lands during development, we swap reflection for direct calls in a cleanup task.

**Testing strategy.**
- **Pure logic (flag system, expression parser, scene builder):** unit-tested in a side-by-side xUnit project. These classes must not reference `UnityEngine` or KSP assemblies.
- **KSP-dependent code (renderer bridge, overlay UI, ScenarioModule):** manual in-game verification. We will build a tiny `KerbalDialogueKit.Demo` mod (also in this repo) that triggers a test scene from a toolbar button so you can visually confirm the system works end-to-end.

**Commit cadence.** Commit after every task in this plan completes and its tests pass. Follow the existing repo style (conventional commits: `feat:`, `fix:`, `docs:`, `test:`, `chore:`, `refactor:`).

**Authoritative path prefix.** All paths below are relative to `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalDialogueKit/` (the new mod repo) unless explicitly stated otherwise.

---

## File Structure

The new repo will contain:

```
GameData/KerbalDialogueKit/                    ← repo root, also the mod root
├── .gitignore
├── README.md
├── LICENSE                                     ← MIT
├── KerbalDialogueKit.sln
├── src/
│   └── KerbalDialogueKit/
│       ├── KerbalDialogueKit.csproj
│       ├── Properties/
│       │   └── AssemblyInfo.cs
│       ├── Core/
│       │   ├── DialogueKit.cs                  ← public static API entry
│       │   ├── DialogueScene.cs                ← scene data + fluent builder
│       │   ├── DialogueLine.cs                 ← single line with visibleIf
│       │   ├── DialogueChoice.cs               ← choice prompt
│       │   ├── DialogueChoiceOption.cs         ← one option within a choice
│       │   ├── DialogueCharacter.cs            ← speaker data (id, model, color)
│       │   ├── SceneCallbacks.cs               ← callback delegates container
│       │   └── SceneQueue.cs                   ← internal queue management
│       ├── Flags/
│       │   ├── FlagStore.cs                    ← in-memory flag kv store
│       │   ├── FlagScenario.cs                 ← ScenarioModule persisting flags
│       │   ├── FlagExpression.cs               ← parsed expression tree
│       │   ├── FlagExpressionParser.cs         ← text → FlagExpression
│       │   └── FlagExpressionTokenizer.cs      ← text → token stream
│       ├── Rendering/
│       │   ├── ChatterBoxBridge.cs             ← reflection wrapper (swappable)
│       │   ├── SceneRenderer.cs                ← DialogueScene → ChatterBox
│       │   ├── ChoiceOverlay.cs                ← MonoBehaviour + IMGUI overlay
│       │   └── LineEvaluator.cs                ← filters lines by visibleIf
│       ├── Config/
│       │   └── DialogueSceneLoader.cs          ← parse DIALOGUE_SCENE cfg
│       ├── Events/
│       │   └── DialogueEvents.cs               ← GameEvents for music cues
│       └── Runtime/
│           └── KerbalDialogueKitAddon.cs       ← [KSPAddon] entry point
├── tests/
│   └── KerbalDialogueKit.Tests/
│       ├── KerbalDialogueKit.Tests.csproj
│       ├── FlagStoreTests.cs
│       ├── FlagExpressionTokenizerTests.cs
│       ├── FlagExpressionParserTests.cs
│       ├── FlagExpressionEvaluationTests.cs
│       ├── DialogueSceneBuilderTests.cs
│       ├── LineEvaluatorTests.cs
│       └── DialogueSceneLoaderTests.cs
├── demo/
│   └── KerbalDialogueKit.Demo/
│       ├── KerbalDialogueKit.Demo.csproj
│       └── DemoTrigger.cs                      ← [KSPAddon] toolbar button
├── Plugins/                                    ← build output lands here
│   └── (KerbalDialogueKit.dll after build)
├── Examples/
│   └── demo-scene.cfg                          ← example DIALOGUE_SCENE
└── KerbalDialogueKit.version                   ← KSP-AVC version file
```

**File responsibilities.**

| File | Responsibility |
|---|---|
| `DialogueKit.cs` | Public static entry: `Enqueue(scene)`, `EnqueueById(id, callbacks)`, `Flags` property, `IsPlaying()`, `QueueLength()` |
| `DialogueScene.cs` | Scene data (title, character slots, ordered line/choice items) + fluent builder methods |
| `DialogueLine.cs` | `Speaker`, `Text`, `Animation`, `VisibleIf` (string expression), `LineAudio`, `LineLevel` |
| `DialogueChoice.cs` | `Id`, `Prompt` (optional text above cards), `Options` list |
| `DialogueChoiceOption.cs` | `Value`, `Text`, `Description` (optional effects preview) |
| `DialogueCharacter.cs` | `Id` (slot "A"/"B"), `Name`, `PortraitModel`, `Color`, `Animation` |
| `SceneCallbacks.cs` | Holder of optional `Action`/`Action<...>` delegates for lifecycle events |
| `SceneQueue.cs` | FIFO queue + priority insertion; handles "is a scene running" state |
| `FlagStore.cs` | Pure in-memory Dictionary<string,string>; no Unity/KSP deps |
| `FlagScenario.cs` | `ScenarioModule` wrapping FlagStore, persisting to save file |
| `FlagExpression.cs` | AST node types (Literal, Ref, Comparison, LogicalAnd, LogicalOr, InList) |
| `FlagExpressionTokenizer.cs` | Pure tokenizer; produces Token list |
| `FlagExpressionParser.cs` | Pure parser; Token list → FlagExpression AST |
| `ChatterBoxBridge.cs` | All reflection against ChatterBox; single swap point when PR lands |
| `SceneRenderer.cs` | Maps `DialogueScene` to ChatterBox scene; handles line filtering via visibleIf; drives playback |
| `ChoiceOverlay.cs` | `MonoBehaviour` that draws choice cards with IMGUI; captures click and resumes |
| `LineEvaluator.cs` | Pure: given a line's VisibleIf + current flags, returns bool |
| `DialogueSceneLoader.cs` | Parses `DIALOGUE_SCENE` ConfigNode → `DialogueScene` |
| `DialogueEvents.cs` | Publishes `GameEvents.FindEvent` / custom events for music cues |
| `KerbalDialogueKitAddon.cs` | `[KSPAddon(Startup.Instantly, true)]` — initializes SceneRenderer, loads cfg scenes, wires up ChoiceOverlay |

---

## Task 1: Prepare the ChatterBox PR

This runs in parallel with everything else. It's a separate repo and does not block our work.

**Files:**
- Modify: `/tmp/ChatterBox/Source/ShowScenePopup.cs` (your fork of [DymndLab/ChatterBox](https://github.com/DymndLab/ChatterBox))

- [ ] **Step 1: Fork and clone ChatterBox locally**

Run:
```bash
gh repo fork DymndLab/ChatterBox --clone=true --remote-name=origin
cd ChatterBox
git checkout -b public-api
```

- [ ] **Step 2: Make the four key classes public**

Open `Source/ShowScenePopup.cs` and change visibility:
- `internal sealed class ScenePopupDefinition` → `public sealed class ScenePopupDefinition`
- `internal sealed class CharacterConfig` → `public sealed class CharacterConfig`
- `internal sealed class LineConfig` → `public sealed class LineConfig`
- `internal sealed class ScenePopupController` → `public sealed class ScenePopupController`
- `internal enum SceneSpeaker` → `public enum SceneSpeaker`
- `internal enum ScenePresentationMode` → `public enum ScenePresentationMode`
- `internal enum SceneAnchor` → `public enum SceneAnchor`
- `internal enum ClampToMode` → `public enum ClampToMode`
- `internal enum LineJustify` → `public enum LineJustify`
- `internal enum PortraitSourceMode` → `public enum PortraitSourceMode`

The `ScenePopupController.Enqueue` method is already `public`; making the class public allows external assemblies to reference the parameter type.

- [ ] **Step 3: Build ChatterBox against your local KSP DLLs to verify it still compiles**

The ChatterBox repo's build instructions may vary. Worst case, open the .csproj (if present), adjust reference paths to `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/KSP_x64_Data/Managed/` for KSP DLLs and `C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/ContractConfigurator/ContractConfigurator.dll` for CC, and run `dotnet build`. Expected: clean build.

- [ ] **Step 4: Commit and push to your fork, then open PR**

```bash
git add Source/ShowScenePopup.cs
git commit -m "Expose scene rendering API for external mod consumption

Changes ScenePopupDefinition, CharacterConfig, LineConfig, ScenePopupController
and supporting enums from internal to public. ScenePopupController.Enqueue
was already public; making the parameter types public allows other mods to
build and queue scene definitions programmatically.

Non-breaking: no behavioural changes. Existing ChatterBox consumers
(contract-triggered dialogue via BEHAVIOUR nodes) continue working
unchanged. External mods gain the ability to trigger scenes from code.

Rationale: enables utility libraries (e.g. KerbalDialogueKit) to extend
ChatterBox with branching, choice capture, and code-driven scene triggers
without forking the rendering pipeline."
git push origin public-api
gh pr create --title "Expose scene rendering API for external mod consumption" \
    --body "Non-breaking change making scene rendering classes public so other mods can programmatically build and enqueue scenes. See commit message for details."
```

Record the PR URL somewhere (add it to a `docs/UPSTREAM.md` in our mod repo once created).

**No commit to our repo in this task.** Continue to Task 2 regardless of PR status — reflection path works either way.

---

## Task 2: Initialize the KerbalDialogueKit Repo

**Files:**
- Create: `GameData/KerbalDialogueKit/` (new directory)
- Create: `GameData/KerbalDialogueKit/.gitignore`
- Create: `GameData/KerbalDialogueKit/README.md`
- Create: `GameData/KerbalDialogueKit/LICENSE`
- Create: `GameData/KerbalDialogueKit/KerbalDialogueKit.version`

- [ ] **Step 1: Create the repo folder and initialize git**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData"
mkdir KerbalDialogueKit
cd KerbalDialogueKit
git init
```

- [ ] **Step 2: Write `.gitignore`**

File: `.gitignore`
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
demo/*/bin/
demo/*/obj/

# But DO ship Plugins/KerbalDialogueKit.dll — that's the installed artifact
!Plugins/*.dll

# Editor
.idea/
*.swp
```

- [ ] **Step 3: Write LICENSE (MIT)**

File: `LICENSE`
```
MIT License

Copyright (c) 2026 BadgKat

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

- [ ] **Step 4: Write the KSP-AVC version file**

File: `KerbalDialogueKit.version`
```json
{
  "NAME": "KerbalDialogueKit",
  "URL": "",
  "DOWNLOAD": "",
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

Leave URL/DOWNLOAD empty for now; fill in when we publish.

- [ ] **Step 5: Write a stub README**

File: `README.md`
```markdown
# KerbalDialogueKit

A ChatterBox extension library that lets other mods trigger character-driven
dialogue scenes from code, with branching, mid-scene choice capture, and
lifecycle callbacks.

## Status

Under active development. Not yet released.

## Dependencies

- ChatterBox v1.0.0 or later
- KSP 1.12.x

## License

MIT — see LICENSE.
```

- [ ] **Step 6: Initial commit**

```bash
git add .gitignore README.md LICENSE KerbalDialogueKit.version
git commit -m "chore: initial commit — repo scaffolding"
```

---

## Task 3: Set Up the Solution and Main Project

**Files:**
- Create: `KerbalDialogueKit.sln`
- Create: `src/KerbalDialogueKit/KerbalDialogueKit.csproj`
- Create: `src/KerbalDialogueKit/Properties/AssemblyInfo.cs`
- Create: `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs` (stub)

- [ ] **Step 1: Create the directory structure**

```bash
mkdir -p src/KerbalDialogueKit/Properties
mkdir -p src/KerbalDialogueKit/Core
mkdir -p src/KerbalDialogueKit/Flags
mkdir -p src/KerbalDialogueKit/Rendering
mkdir -p src/KerbalDialogueKit/Config
mkdir -p src/KerbalDialogueKit/Events
mkdir -p src/KerbalDialogueKit/Runtime
mkdir -p Plugins
```

- [ ] **Step 2: Write the main csproj targeting .NET Framework 4.8**

File: `src/KerbalDialogueKit/KerbalDialogueKit.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <AssemblyName>KerbalDialogueKit</AssemblyName>
    <RootNamespace>KerbalDialogueKit</RootNamespace>
    <LangVersion>7.3</LangVersion>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <OutputPath>$(MSBuildThisFileDirectory)..\..\Plugins\</OutputPath>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
    <AutoGenerateBindingRedirects>false</AutoGenerateBindingRedirects>
  </PropertyGroup>

  <ItemGroup>
    <!-- KSP engine DLLs (Reference with CopyLocal=false so we don't ship them) -->
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

    <!-- Third-party mods (reflection targets; reference with CopyLocal=false) -->
    <Reference Include="ChatterBox">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\ChatterBox\Plugins\ChatterBox.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>

</Project>
```

Note: we reference ChatterBox.dll even though we use reflection. That lets us load the assembly at build time and makes `Assembly.GetAssembly(typeof(...))` work cleanly at runtime.

- [ ] **Step 3: Write AssemblyInfo.cs**

File: `src/KerbalDialogueKit/Properties/AssemblyInfo.cs`
```csharp
using System.Reflection;
using System.Runtime.InteropServices;

[assembly: AssemblyTitle("KerbalDialogueKit")]
[assembly: AssemblyDescription("ChatterBox extension library: code-triggered dialogue with branching and choices.")]
[assembly: AssemblyCompany("BadgKat")]
[assembly: AssemblyProduct("KerbalDialogueKit")]
[assembly: AssemblyCopyright("Copyright © BadgKat 2026")]
[assembly: ComVisible(false)]
[assembly: AssemblyVersion("0.1.0.0")]
[assembly: AssemblyFileVersion("0.1.0.0")]
[assembly: AssemblyInformationalVersion("0.1.0")]
[assembly: Guid("DDF2A8E0-4FB1-4C6F-9C6A-1B5F8F6B7A11")]
```

- [ ] **Step 4: Write a minimal KSPAddon stub that logs on load**

File: `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs`
```csharp
using UnityEngine;

namespace KerbalDialogueKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.Instantly, true)]
    public sealed class KerbalDialogueKitAddon : MonoBehaviour
    {
        public void Awake()
        {
            Debug.Log("[KerbalDialogueKit] Initialized v0.1.0");
            DontDestroyOnLoad(this);
        }
    }
}
```

- [ ] **Step 5: Create the solution file**

From the repo root:
```bash
dotnet new sln -n KerbalDialogueKit
dotnet sln add src/KerbalDialogueKit/KerbalDialogueKit.csproj
```

- [ ] **Step 6: Build and verify the DLL lands in Plugins/**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: `Build succeeded.` and `Plugins/KerbalDialogueKit.dll` exists.

Verify:
```bash
ls -la Plugins/
```

Expected: `KerbalDialogueKit.dll` present.

- [ ] **Step 7: Launch KSP briefly and check the log for our init line**

Launch KSP manually. After it reaches the main menu, quit. Then:
```bash
grep "KerbalDialogueKit" "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/KSP.log" | head -5
```

Expected: one or more log lines including `[KerbalDialogueKit] Initialized v0.1.0`.

If you do not see the line, the DLL did not load. Check KSP.log for `AssemblyLoader` errors mentioning KerbalDialogueKit.

- [ ] **Step 8: Commit**

```bash
git add KerbalDialogueKit.sln src/
git commit -m "feat: scaffold main csproj and KSPAddon entry point

Targets net48, outputs to Plugins/KerbalDialogueKit.dll, references
KSP engine DLLs and ChatterBox.dll (all CopyLocal=false)."
```

---

## Task 4: Set Up the xUnit Test Project

**Files:**
- Create: `tests/KerbalDialogueKit.Tests/KerbalDialogueKit.Tests.csproj`
- Create: `tests/KerbalDialogueKit.Tests/SmokeTest.cs`

- [ ] **Step 1: Create tests directory and project**

```bash
mkdir -p tests/KerbalDialogueKit.Tests
```

- [ ] **Step 2: Write the test csproj**

We target `net48` for the test project too so we can link against the same assembly without multi-target complications. xUnit works on net48.

File: `tests/KerbalDialogueKit.Tests/KerbalDialogueKit.Tests.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
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
    <ProjectReference Include="..\..\src\KerbalDialogueKit\KerbalDialogueKit.csproj" />
  </ItemGroup>

</Project>
```

- [ ] **Step 3: Write a trivial smoke test**

File: `tests/KerbalDialogueKit.Tests/SmokeTest.cs`
```csharp
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class SmokeTest
    {
        [Fact]
        public void TrueIsTrue()
        {
            Assert.True(true);
        }
    }
}
```

- [ ] **Step 4: Add test project to the solution**

```bash
dotnet sln add tests/KerbalDialogueKit.Tests/KerbalDialogueKit.Tests.csproj
```

- [ ] **Step 5: Run tests to verify the harness works**

```bash
dotnet test tests/KerbalDialogueKit.Tests/KerbalDialogueKit.Tests.csproj
```

Expected: `Passed: 1, Failed: 0`.

Note: the test project references the main project, which references `Assembly-CSharp.dll` etc. That's fine — the test project builds against those references but doesn't need them at test-runtime as long as your tests don't touch KSP types. If xUnit fails to load the test assembly due to Unity/KSP resolution errors, add `<Private>true</Private>` temporarily on the KSP references in the main csproj. This is unlikely but keep it in your back pocket.

- [ ] **Step 6: Commit**

```bash
git add tests/
git commit -m "test: add xUnit test project with passing smoke test"
```

---

## Task 5: FlagStore — In-Memory Flag Key-Value Store

A pure C# class with no Unity/KSP references. Fully unit-testable.

**Files:**
- Create: `src/KerbalDialogueKit/Flags/FlagStore.cs`
- Create: `tests/KerbalDialogueKit.Tests/FlagStoreTests.cs`

- [ ] **Step 1: Write the failing tests**

File: `tests/KerbalDialogueKit.Tests/FlagStoreTests.cs`
```csharp
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class FlagStoreTests
    {
        [Fact]
        public void GetReturnsNullForUnsetFlag()
        {
            var store = new FlagStore();
            Assert.Null(store.Get("missing"));
        }

        [Fact]
        public void SetAndGetRoundTrip()
        {
            var store = new FlagStore();
            store.Set("mood", "Enthusiastic");
            Assert.Equal("Enthusiastic", store.Get("mood"));
        }

        [Fact]
        public void SetOverwritesExistingValue()
        {
            var store = new FlagStore();
            store.Set("mood", "Neutral");
            store.Set("mood", "Frustrated");
            Assert.Equal("Frustrated", store.Get("mood"));
        }

        [Fact]
        public void HasReturnsTrueForSetFlag()
        {
            var store = new FlagStore();
            store.Set("k", "v");
            Assert.True(store.Has("k"));
        }

        [Fact]
        public void HasReturnsFalseForUnsetFlag()
        {
            var store = new FlagStore();
            Assert.False(store.Has("missing"));
        }

        [Fact]
        public void RemoveDeletesFlag()
        {
            var store = new FlagStore();
            store.Set("k", "v");
            store.Remove("k");
            Assert.False(store.Has("k"));
            Assert.Null(store.Get("k"));
        }

        [Fact]
        public void RemoveOfUnsetFlagIsNoOp()
        {
            var store = new FlagStore();
            store.Remove("missing");  // Should not throw
        }

        [Fact]
        public void ClearRemovesAllFlags()
        {
            var store = new FlagStore();
            store.Set("a", "1");
            store.Set("b", "2");
            store.Clear();
            Assert.False(store.Has("a"));
            Assert.False(store.Has("b"));
        }

        [Fact]
        public void KeysReturnsAllSetKeys()
        {
            var store = new FlagStore();
            store.Set("a", "1");
            store.Set("b", "2");
            var keys = store.Keys;
            Assert.Contains("a", keys);
            Assert.Contains("b", keys);
            Assert.Equal(2, keys.Count);
        }
    }
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: build fails — `FlagStore` not defined.

- [ ] **Step 3: Implement FlagStore**

File: `src/KerbalDialogueKit/Flags/FlagStore.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Flags
{
    /// <summary>
    /// In-memory key-value flag store. No Unity or KSP dependencies.
    /// Persistence is handled separately by FlagScenario.
    /// </summary>
    public sealed class FlagStore
    {
        private readonly Dictionary<string, string> values = new Dictionary<string, string>();

        public string Get(string key)
        {
            values.TryGetValue(key, out var v);
            return v;
        }

        public void Set(string key, string value)
        {
            values[key] = value;
        }

        public bool Has(string key)
        {
            return values.ContainsKey(key);
        }

        public void Remove(string key)
        {
            values.Remove(key);
        }

        public void Clear()
        {
            values.Clear();
        }

        public IReadOnlyCollection<string> Keys => values.Keys;

        internal IReadOnlyDictionary<string, string> Snapshot => values;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all 9 FlagStoreTests pass.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalDialogueKit/Flags/FlagStore.cs tests/KerbalDialogueKit.Tests/FlagStoreTests.cs
git commit -m "feat: add FlagStore for in-memory flag key-value storage"
```

---

## Task 6: FlagExpressionTokenizer — Tokenize visibleIf Strings

Pure logic. Converts a string like `"mood == Enthusiastic && score > 5"` into a token stream.

**Token kinds:**
- `Identifier` — bareword `mood`, `approach`, etc.
- `Value` — quoted or unquoted literal; everything after `==`, `!=`, `>`, `<`, `>=`, `<=`, or inside `(...)` list
- `OpEq` (`==`), `OpNeq` (`!=`), `OpGt` (`>`), `OpLt` (`<`), `OpGte` (`>=`), `OpLte` (`<=`)
- `OpAnd` (`&&`), `OpOr` (`||`)
- `KwIn` (`in`)
- `LParen` (`(`), `RParen` (`)`)
- `Comma` (`,`)
- `Eof`

**Files:**
- Create: `src/KerbalDialogueKit/Flags/FlagExpressionTokenizer.cs`
- Create: `tests/KerbalDialogueKit.Tests/FlagExpressionTokenizerTests.cs`

- [ ] **Step 1: Write the failing tests**

File: `tests/KerbalDialogueKit.Tests/FlagExpressionTokenizerTests.cs`
```csharp
using System.Collections.Generic;
using System.Linq;
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class FlagExpressionTokenizerTests
    {
        private static List<TokenKind> Kinds(string input)
        {
            return FlagExpressionTokenizer.Tokenize(input).Select(t => t.Kind).ToList();
        }

        private static List<string> Texts(string input)
        {
            return FlagExpressionTokenizer.Tokenize(input).Select(t => t.Text).ToList();
        }

        [Fact]
        public void EmptyInputYieldsOnlyEof()
        {
            Assert.Equal(new[] { TokenKind.Eof }, Kinds(""));
        }

        [Fact]
        public void WhitespaceOnlyYieldsOnlyEof()
        {
            Assert.Equal(new[] { TokenKind.Eof }, Kinds("   \t  "));
        }

        [Fact]
        public void SimpleEquality()
        {
            Assert.Equal(
                new[] { TokenKind.Identifier, TokenKind.OpEq, TokenKind.Value, TokenKind.Eof },
                Kinds("mood == Enthusiastic"));
        }

        [Fact]
        public void AllComparisonOperators()
        {
            Assert.Equal(TokenKind.OpEq, FlagExpressionTokenizer.Tokenize("a == b").ElementAt(1).Kind);
            Assert.Equal(TokenKind.OpNeq, FlagExpressionTokenizer.Tokenize("a != b").ElementAt(1).Kind);
            Assert.Equal(TokenKind.OpGte, FlagExpressionTokenizer.Tokenize("a >= b").ElementAt(1).Kind);
            Assert.Equal(TokenKind.OpLte, FlagExpressionTokenizer.Tokenize("a <= b").ElementAt(1).Kind);
            Assert.Equal(TokenKind.OpGt, FlagExpressionTokenizer.Tokenize("a > b").ElementAt(1).Kind);
            Assert.Equal(TokenKind.OpLt, FlagExpressionTokenizer.Tokenize("a < b").ElementAt(1).Kind);
        }

        [Fact]
        public void LogicalAndOr()
        {
            Assert.Equal(
                new[] {
                    TokenKind.Identifier, TokenKind.OpEq, TokenKind.Value,
                    TokenKind.OpAnd,
                    TokenKind.Identifier, TokenKind.OpEq, TokenKind.Value,
                    TokenKind.Eof },
                Kinds("a == 1 && b == 2"));

            Assert.Equal(
                new[] {
                    TokenKind.Identifier, TokenKind.OpEq, TokenKind.Value,
                    TokenKind.OpOr,
                    TokenKind.Identifier, TokenKind.OpEq, TokenKind.Value,
                    TokenKind.Eof },
                Kinds("a == 1 || b == 2"));
        }

        [Fact]
        public void InOperatorWithList()
        {
            Assert.Equal(
                new[] {
                    TokenKind.Identifier, TokenKind.KwIn,
                    TokenKind.LParen,
                    TokenKind.Value, TokenKind.Comma, TokenKind.Value,
                    TokenKind.RParen, TokenKind.Eof },
                Kinds("mood in (Enthusiastic, Supportive)"));
        }

        [Fact]
        public void TokenTextIsPreserved()
        {
            var texts = Texts("mood == Enthusiastic");
            Assert.Equal(new[] { "mood", "==", "Enthusiastic", "" }, texts);
        }

        [Fact]
        public void QuotedValuePreservesInnerSpaces()
        {
            var tokens = FlagExpressionTokenizer.Tokenize("name == \"Dr. Sidmund\"").ToList();
            Assert.Equal(TokenKind.Value, tokens[2].Kind);
            Assert.Equal("Dr. Sidmund", tokens[2].Text);
        }

        [Fact]
        public void NumericValuesRemainAsText()
        {
            var tokens = FlagExpressionTokenizer.Tokenize("score > 5").ToList();
            Assert.Equal(TokenKind.Value, tokens[2].Kind);
            Assert.Equal("5", tokens[2].Text);
        }
    }
}
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: compile errors about missing types.

- [ ] **Step 3: Implement Token and TokenKind**

File: `src/KerbalDialogueKit/Flags/FlagExpressionTokenizer.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Text;

namespace KerbalDialogueKit.Flags
{
    public enum TokenKind
    {
        Identifier,
        Value,
        OpEq, OpNeq, OpGt, OpLt, OpGte, OpLte,
        OpAnd, OpOr,
        KwIn,
        LParen, RParen,
        Comma,
        Eof
    }

    public struct Token
    {
        public TokenKind Kind;
        public string Text;

        public Token(TokenKind kind, string text)
        {
            Kind = kind;
            Text = text;
        }
    }

    /// <summary>
    /// Pure tokenizer for visibleIf expressions.
    /// Grammar:
    ///   expr   := or
    ///   or     := and ("||" and)*
    ///   and    := cmp ("&&" cmp)*
    ///   cmp    := ident op value | ident "in" "(" value ("," value)* ")"
    ///   ident  := [a-zA-Z_][a-zA-Z0-9_]*
    ///   value  := quoted string | bareword | number
    /// </summary>
    public static class FlagExpressionTokenizer
    {
        public static IEnumerable<Token> Tokenize(string input)
        {
            var tokens = new List<Token>();
            int i = 0;
            int n = input?.Length ?? 0;

            // State: after a comparison operator or KwIn or LParen or Comma, the next
            // word is a Value; after operators like && / || or at the start, a word is
            // an Identifier or keyword. This is a simple state flag.
            bool expectValue = false;

            while (i < n)
            {
                char c = input[i];
                if (char.IsWhiteSpace(c)) { i++; continue; }

                // Multi-char operators first
                if (i + 1 < n)
                {
                    string two = input.Substring(i, 2);
                    if (two == "==") { tokens.Add(new Token(TokenKind.OpEq, "==")); i += 2; expectValue = true; continue; }
                    if (two == "!=") { tokens.Add(new Token(TokenKind.OpNeq, "!=")); i += 2; expectValue = true; continue; }
                    if (two == ">=") { tokens.Add(new Token(TokenKind.OpGte, ">=")); i += 2; expectValue = true; continue; }
                    if (two == "<=") { tokens.Add(new Token(TokenKind.OpLte, "<=")); i += 2; expectValue = true; continue; }
                    if (two == "&&") { tokens.Add(new Token(TokenKind.OpAnd, "&&")); i += 2; expectValue = false; continue; }
                    if (two == "||") { tokens.Add(new Token(TokenKind.OpOr, "||")); i += 2; expectValue = false; continue; }
                }

                if (c == '>') { tokens.Add(new Token(TokenKind.OpGt, ">")); i++; expectValue = true; continue; }
                if (c == '<') { tokens.Add(new Token(TokenKind.OpLt, "<")); i++; expectValue = true; continue; }
                if (c == '(') { tokens.Add(new Token(TokenKind.LParen, "(")); i++; expectValue = true; continue; }
                if (c == ')') { tokens.Add(new Token(TokenKind.RParen, ")")); i++; expectValue = false; continue; }
                if (c == ',') { tokens.Add(new Token(TokenKind.Comma, ",")); i++; expectValue = true; continue; }

                if (c == '"')
                {
                    // Quoted value
                    int start = ++i;
                    var sb = new StringBuilder();
                    while (i < n && input[i] != '"')
                    {
                        if (input[i] == '\\' && i + 1 < n) { sb.Append(input[i + 1]); i += 2; }
                        else { sb.Append(input[i]); i++; }
                    }
                    if (i >= n) throw new FormatException("Unterminated quoted string in expression");
                    i++; // skip closing quote
                    tokens.Add(new Token(TokenKind.Value, sb.ToString()));
                    expectValue = false;
                    continue;
                }

                // Bareword — identifier, keyword, or value
                if (IsIdentStart(c))
                {
                    int start = i;
                    while (i < n && IsIdentChar(input[i])) i++;
                    string word = input.Substring(start, i - start);

                    if (!expectValue)
                    {
                        // It's an identifier or the "in" keyword
                        if (word == "in") { tokens.Add(new Token(TokenKind.KwIn, "in")); expectValue = true; }
                        else tokens.Add(new Token(TokenKind.Identifier, word));
                    }
                    else
                    {
                        tokens.Add(new Token(TokenKind.Value, word));
                        expectValue = false;
                    }
                    continue;
                }

                // Numeric bareword (starts with digit)
                if (char.IsDigit(c) || c == '-')
                {
                    int start = i;
                    if (c == '-') i++;
                    while (i < n && (char.IsDigit(input[i]) || input[i] == '.')) i++;
                    string word = input.Substring(start, i - start);
                    tokens.Add(new Token(TokenKind.Value, word));
                    expectValue = false;
                    continue;
                }

                throw new FormatException($"Unexpected character '{c}' at position {i} in expression '{input}'");
            }

            tokens.Add(new Token(TokenKind.Eof, ""));
            return tokens;
        }

        private static bool IsIdentStart(char c) => char.IsLetter(c) || c == '_';
        private static bool IsIdentChar(char c) => char.IsLetterOrDigit(c) || c == '_';
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all tokenizer tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalDialogueKit/Flags/FlagExpressionTokenizer.cs tests/KerbalDialogueKit.Tests/FlagExpressionTokenizerTests.cs
git commit -m "feat: add FlagExpressionTokenizer for visibleIf expressions"
```

---

## Task 7: FlagExpression AST

Define the AST node types produced by the parser and consumed by the evaluator.

**Files:**
- Create: `src/KerbalDialogueKit/Flags/FlagExpression.cs`

- [ ] **Step 1: Write FlagExpression types**

File: `src/KerbalDialogueKit/Flags/FlagExpression.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Flags
{
    /// <summary>
    /// AST node for a visibleIf expression. Evaluated against a FlagStore to produce
    /// a bool. Construction is parser-only; evaluation is pure given the flag snapshot.
    /// </summary>
    public abstract class FlagExpression
    {
        public abstract bool Evaluate(FlagStore flags);
    }

    public sealed class AlwaysTrueExpression : FlagExpression
    {
        public override bool Evaluate(FlagStore flags) => true;
    }

    public enum ComparisonOp
    {
        Equal, NotEqual, Greater, Less, GreaterOrEqual, LessOrEqual
    }

    public sealed class ComparisonExpression : FlagExpression
    {
        public string FlagName;
        public ComparisonOp Op;
        public string RightValue;

        public override bool Evaluate(FlagStore flags)
        {
            string left = flags.Get(FlagName) ?? string.Empty;
            switch (Op)
            {
                case ComparisonOp.Equal: return left == RightValue;
                case ComparisonOp.NotEqual: return left != RightValue;
                case ComparisonOp.Greater:
                case ComparisonOp.Less:
                case ComparisonOp.GreaterOrEqual:
                case ComparisonOp.LessOrEqual:
                    return CompareNumeric(left, RightValue, Op);
                default: return false;
            }
        }

        private static bool CompareNumeric(string a, string b, ComparisonOp op)
        {
            if (!double.TryParse(a, out var da)) return false;
            if (!double.TryParse(b, out var db)) return false;
            switch (op)
            {
                case ComparisonOp.Greater: return da > db;
                case ComparisonOp.Less: return da < db;
                case ComparisonOp.GreaterOrEqual: return da >= db;
                case ComparisonOp.LessOrEqual: return da <= db;
                default: return false;
            }
        }
    }

    public sealed class InListExpression : FlagExpression
    {
        public string FlagName;
        public List<string> Values = new List<string>();

        public override bool Evaluate(FlagStore flags)
        {
            string left = flags.Get(FlagName) ?? string.Empty;
            return Values.Contains(left);
        }
    }

    public sealed class AndExpression : FlagExpression
    {
        public FlagExpression Left;
        public FlagExpression Right;

        public override bool Evaluate(FlagStore flags) =>
            Left.Evaluate(flags) && Right.Evaluate(flags);
    }

    public sealed class OrExpression : FlagExpression
    {
        public FlagExpression Left;
        public FlagExpression Right;

        public override bool Evaluate(FlagStore flags) =>
            Left.Evaluate(flags) || Right.Evaluate(flags);
    }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: build succeeds (no tests here yet; evaluation is tested in Task 9).

- [ ] **Step 3: Commit**

```bash
git add src/KerbalDialogueKit/Flags/FlagExpression.cs
git commit -m "feat: add FlagExpression AST node types"
```

---

## Task 8: FlagExpressionParser — Tokens to AST

Recursive-descent parser over the token stream. Pure logic, fully unit-testable.

**Files:**
- Create: `src/KerbalDialogueKit/Flags/FlagExpressionParser.cs`
- Create: `tests/KerbalDialogueKit.Tests/FlagExpressionParserTests.cs`

- [ ] **Step 1: Write parser tests**

File: `tests/KerbalDialogueKit.Tests/FlagExpressionParserTests.cs`
```csharp
using System;
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class FlagExpressionParserTests
    {
        [Fact]
        public void EmptyStringParsesAsAlwaysTrue()
        {
            var expr = FlagExpressionParser.Parse("");
            Assert.IsType<AlwaysTrueExpression>(expr);
        }

        [Fact]
        public void SimpleEqualityParsesAsComparison()
        {
            var expr = FlagExpressionParser.Parse("mood == Enthusiastic") as ComparisonExpression;
            Assert.NotNull(expr);
            Assert.Equal("mood", expr.FlagName);
            Assert.Equal(ComparisonOp.Equal, expr.Op);
            Assert.Equal("Enthusiastic", expr.RightValue);
        }

        [Fact]
        public void AllComparisonOperators()
        {
            Assert.Equal(ComparisonOp.Equal, ((ComparisonExpression)FlagExpressionParser.Parse("a == b")).Op);
            Assert.Equal(ComparisonOp.NotEqual, ((ComparisonExpression)FlagExpressionParser.Parse("a != b")).Op);
            Assert.Equal(ComparisonOp.Greater, ((ComparisonExpression)FlagExpressionParser.Parse("a > b")).Op);
            Assert.Equal(ComparisonOp.Less, ((ComparisonExpression)FlagExpressionParser.Parse("a < b")).Op);
            Assert.Equal(ComparisonOp.GreaterOrEqual, ((ComparisonExpression)FlagExpressionParser.Parse("a >= b")).Op);
            Assert.Equal(ComparisonOp.LessOrEqual, ((ComparisonExpression)FlagExpressionParser.Parse("a <= b")).Op);
        }

        [Fact]
        public void InListParses()
        {
            var expr = FlagExpressionParser.Parse("mood in (Enthusiastic, Supportive)") as InListExpression;
            Assert.NotNull(expr);
            Assert.Equal("mood", expr.FlagName);
            Assert.Equal(new[] { "Enthusiastic", "Supportive" }, expr.Values);
        }

        [Fact]
        public void AndExpressionParses()
        {
            var expr = FlagExpressionParser.Parse("a == 1 && b == 2") as AndExpression;
            Assert.NotNull(expr);
            Assert.IsType<ComparisonExpression>(expr.Left);
            Assert.IsType<ComparisonExpression>(expr.Right);
        }

        [Fact]
        public void OrExpressionParses()
        {
            var expr = FlagExpressionParser.Parse("a == 1 || b == 2") as OrExpression;
            Assert.NotNull(expr);
        }

        [Fact]
        public void AndBindsTighterThanOr()
        {
            // "a && b || c && d" should parse as "(a && b) || (c && d)"
            var expr = FlagExpressionParser.Parse("a == 1 && b == 2 || c == 3 && d == 4") as OrExpression;
            Assert.NotNull(expr);
            Assert.IsType<AndExpression>(expr.Left);
            Assert.IsType<AndExpression>(expr.Right);
        }

        [Fact]
        public void MalformedExpressionThrows()
        {
            Assert.Throws<FormatException>(() => FlagExpressionParser.Parse("a =="));
            Assert.Throws<FormatException>(() => FlagExpressionParser.Parse("== b"));
            Assert.Throws<FormatException>(() => FlagExpressionParser.Parse("a in b"));
            Assert.Throws<FormatException>(() => FlagExpressionParser.Parse("a in ("));
        }
    }
}
```

- [ ] **Step 2: Run tests to confirm they fail**

Expected: compile errors about missing `FlagExpressionParser`.

- [ ] **Step 3: Implement the parser**

File: `src/KerbalDialogueKit/Flags/FlagExpressionParser.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace KerbalDialogueKit.Flags
{
    /// <summary>
    /// Recursive-descent parser for visibleIf expressions.
    /// Empty input returns AlwaysTrueExpression.
    /// </summary>
    public static class FlagExpressionParser
    {
        public static FlagExpression Parse(string input)
        {
            if (string.IsNullOrWhiteSpace(input))
                return new AlwaysTrueExpression();

            var tokens = FlagExpressionTokenizer.Tokenize(input).ToList();
            int pos = 0;
            var expr = ParseOr(tokens, ref pos);
            if (tokens[pos].Kind != TokenKind.Eof)
                throw new FormatException($"Unexpected token '{tokens[pos].Text}' after expression end");
            return expr;
        }

        private static FlagExpression ParseOr(List<Token> tokens, ref int pos)
        {
            var left = ParseAnd(tokens, ref pos);
            while (tokens[pos].Kind == TokenKind.OpOr)
            {
                pos++;
                var right = ParseAnd(tokens, ref pos);
                left = new OrExpression { Left = left, Right = right };
            }
            return left;
        }

        private static FlagExpression ParseAnd(List<Token> tokens, ref int pos)
        {
            var left = ParseComparison(tokens, ref pos);
            while (tokens[pos].Kind == TokenKind.OpAnd)
            {
                pos++;
                var right = ParseComparison(tokens, ref pos);
                left = new AndExpression { Left = left, Right = right };
            }
            return left;
        }

        private static FlagExpression ParseComparison(List<Token> tokens, ref int pos)
        {
            if (tokens[pos].Kind != TokenKind.Identifier)
                throw new FormatException($"Expected flag name, got '{tokens[pos].Text}' ({tokens[pos].Kind})");
            string flagName = tokens[pos].Text;
            pos++;

            // "in" list
            if (tokens[pos].Kind == TokenKind.KwIn)
            {
                pos++;
                if (tokens[pos].Kind != TokenKind.LParen)
                    throw new FormatException("Expected '(' after 'in'");
                pos++;

                var values = new List<string>();
                if (tokens[pos].Kind != TokenKind.Value)
                    throw new FormatException("Expected value inside 'in' list");
                values.Add(tokens[pos].Text);
                pos++;

                while (tokens[pos].Kind == TokenKind.Comma)
                {
                    pos++;
                    if (tokens[pos].Kind != TokenKind.Value)
                        throw new FormatException("Expected value after ',' in 'in' list");
                    values.Add(tokens[pos].Text);
                    pos++;
                }

                if (tokens[pos].Kind != TokenKind.RParen)
                    throw new FormatException("Expected ')' closing 'in' list");
                pos++;

                return new InListExpression { FlagName = flagName, Values = values };
            }

            // comparison operator
            ComparisonOp op;
            switch (tokens[pos].Kind)
            {
                case TokenKind.OpEq: op = ComparisonOp.Equal; break;
                case TokenKind.OpNeq: op = ComparisonOp.NotEqual; break;
                case TokenKind.OpGt: op = ComparisonOp.Greater; break;
                case TokenKind.OpLt: op = ComparisonOp.Less; break;
                case TokenKind.OpGte: op = ComparisonOp.GreaterOrEqual; break;
                case TokenKind.OpLte: op = ComparisonOp.LessOrEqual; break;
                default:
                    throw new FormatException($"Expected comparison operator or 'in' after '{flagName}', got '{tokens[pos].Text}'");
            }
            pos++;

            if (tokens[pos].Kind != TokenKind.Value)
                throw new FormatException($"Expected value after operator, got '{tokens[pos].Text}'");
            string rightValue = tokens[pos].Text;
            pos++;

            return new ComparisonExpression { FlagName = flagName, Op = op, RightValue = rightValue };
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all parser tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalDialogueKit/Flags/FlagExpressionParser.cs tests/KerbalDialogueKit.Tests/FlagExpressionParserTests.cs
git commit -m "feat: add recursive-descent FlagExpressionParser"
```

---

## Task 9: FlagExpression Evaluation Tests

Test the AST's `Evaluate(FlagStore)` method end-to-end. Exercises AST + parser + store together.

**Files:**
- Create: `tests/KerbalDialogueKit.Tests/FlagExpressionEvaluationTests.cs`

- [ ] **Step 1: Write the tests**

File: `tests/KerbalDialogueKit.Tests/FlagExpressionEvaluationTests.cs`
```csharp
using KerbalDialogueKit.Flags;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class FlagExpressionEvaluationTests
    {
        private FlagStore Store(params (string k, string v)[] kv)
        {
            var s = new FlagStore();
            foreach (var pair in kv) s.Set(pair.k, pair.v);
            return s;
        }

        [Fact]
        public void AlwaysTrueEvaluatesTrue()
        {
            var expr = FlagExpressionParser.Parse("");
            Assert.True(expr.Evaluate(new FlagStore()));
        }

        [Fact]
        public void EqualityTrue()
        {
            var s = Store(("mood", "Enthusiastic"));
            Assert.True(FlagExpressionParser.Parse("mood == Enthusiastic").Evaluate(s));
        }

        [Fact]
        public void EqualityFalse()
        {
            var s = Store(("mood", "Neutral"));
            Assert.False(FlagExpressionParser.Parse("mood == Enthusiastic").Evaluate(s));
        }

        [Fact]
        public void MissingFlagTreatedAsEmptyString()
        {
            var s = new FlagStore();
            Assert.True(FlagExpressionParser.Parse("mood != Enthusiastic").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("mood == Enthusiastic").Evaluate(s));
        }

        [Fact]
        public void NumericComparison()
        {
            var s = Store(("score", "7"));
            Assert.True(FlagExpressionParser.Parse("score > 5").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("score > 10").Evaluate(s));
            Assert.True(FlagExpressionParser.Parse("score >= 7").Evaluate(s));
            Assert.True(FlagExpressionParser.Parse("score <= 7").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("score < 7").Evaluate(s));
        }

        [Fact]
        public void InListMatch()
        {
            var s = Store(("mood", "Supportive"));
            Assert.True(FlagExpressionParser.Parse("mood in (Enthusiastic, Supportive)").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("mood in (Frustrated, Skeptical)").Evaluate(s));
        }

        [Fact]
        public void AndShortCircuits()
        {
            var s = Store(("a", "1"), ("b", "2"));
            Assert.True(FlagExpressionParser.Parse("a == 1 && b == 2").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("a == 1 && b == 99").Evaluate(s));
        }

        [Fact]
        public void OrShortCircuits()
        {
            var s = Store(("a", "99"), ("b", "2"));
            Assert.True(FlagExpressionParser.Parse("a == 1 || b == 2").Evaluate(s));
            Assert.False(FlagExpressionParser.Parse("a == 1 || b == 99").Evaluate(s));
        }

        [Fact]
        public void PrecedenceAndBeatsOr()
        {
            var s = Store(("a", "1"), ("b", "2"), ("c", "0"), ("d", "0"));
            // (a==1 && b==2) || (c==1 && d==1) → true || false → true
            Assert.True(FlagExpressionParser.Parse("a == 1 && b == 2 || c == 1 && d == 1").Evaluate(s));
        }

        [Fact]
        public void NumericComparisonAgainstNonNumericIsFalse()
        {
            var s = Store(("mood", "Enthusiastic"));
            Assert.False(FlagExpressionParser.Parse("mood > 5").Evaluate(s));
        }
    }
}
```

- [ ] **Step 2: Run tests**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all evaluation tests pass.

- [ ] **Step 3: Commit**

```bash
git add tests/KerbalDialogueKit.Tests/FlagExpressionEvaluationTests.cs
git commit -m "test: add end-to-end FlagExpression evaluation tests"
```

---

## Task 10: FlagScenario — ScenarioModule Persistence

Wraps a `FlagStore` in a KSP `ScenarioModule` so flags persist per-save. This class references KSP, so we test it only manually in-game.

**Files:**
- Create: `src/KerbalDialogueKit/Flags/FlagScenario.cs`

- [ ] **Step 1: Implement the scenario module**

File: `src/KerbalDialogueKit/Flags/FlagScenario.cs`
```csharp
using System;
using UnityEngine;

namespace KerbalDialogueKit.Flags
{
    /// <summary>
    /// Per-save persistence for the FlagStore. Loaded in Career, Science, and
    /// Sandbox so flag-driven dialogue can be used in any save mode.
    /// </summary>
    [KSPScenario(
        ScenarioCreationOptions.AddToAllGames,
        GameScenes.SPACECENTER, GameScenes.FLIGHT, GameScenes.TRACKSTATION, GameScenes.EDITOR)]
    public sealed class FlagScenario : ScenarioModule
    {
        public static FlagScenario Instance { get; private set; }
        public FlagStore Flags { get; } = new FlagStore();

        public override void OnAwake()
        {
            base.OnAwake();
            Instance = this;
        }

        public override void OnLoad(ConfigNode node)
        {
            base.OnLoad(node);
            Flags.Clear();
            var flagsNode = node.GetNode("FLAGS");
            if (flagsNode == null) return;
            foreach (ConfigNode.Value v in flagsNode.values)
            {
                Flags.Set(v.name, v.value);
            }
        }

        public override void OnSave(ConfigNode node)
        {
            base.OnSave(node);
            var flagsNode = node.AddNode("FLAGS");
            foreach (var key in Flags.Keys)
            {
                flagsNode.AddValue(key, Flags.Get(key));
            }
        }
    }
}
```

- [ ] **Step 2: Build and verify**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 3: Commit**

```bash
git add src/KerbalDialogueKit/Flags/FlagScenario.cs
git commit -m "feat: add FlagScenario for per-save flag persistence"
```

---

## Task 11: Scene Data Model

The domain objects consumers build up and pass to `DialogueKit.Enqueue`. Pure data, no Unity references.

**Files:**
- Create: `src/KerbalDialogueKit/Core/DialogueCharacter.cs`
- Create: `src/KerbalDialogueKit/Core/DialogueLine.cs`
- Create: `src/KerbalDialogueKit/Core/DialogueChoiceOption.cs`
- Create: `src/KerbalDialogueKit/Core/DialogueChoice.cs`
- Create: `src/KerbalDialogueKit/Core/SceneCallbacks.cs`
- Create: `src/KerbalDialogueKit/Core/DialogueScene.cs`

- [ ] **Step 1: DialogueCharacter**

File: `src/KerbalDialogueKit/Core/DialogueCharacter.cs`
```csharp
namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// One speaker slot in a scene. "id" is the slot label ("A" or "B" usually);
    /// Name is the display name, PortraitModel is the KSP instructor model path
    /// (e.g. "Instructor_Gene"), Color is in KSP hex format ("#FFxxxxxx").
    /// </summary>
    public sealed class DialogueCharacter
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string PortraitModel { get; set; }
        public string Color { get; set; } = "#FFFFFFFF";
        public string Animation { get; set; } = "idle";
    }
}
```

- [ ] **Step 2: DialogueLine**

File: `src/KerbalDialogueKit/Core/DialogueLine.cs`
```csharp
namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// One line of dialogue. Speaker is the character slot id ("A" or "B").
    /// VisibleIf, if set, is a flag expression. If the expression evaluates
    /// false at scene build time, the line is skipped.
    /// </summary>
    public sealed class DialogueLine : IDialogueItem
    {
        public string Speaker { get; set; }
        public string Text { get; set; }
        public string Animation { get; set; }
        public string VisibleIf { get; set; }
        public string LineAudio { get; set; }
        public float LineLevel { get; set; }
    }
}
```

- [ ] **Step 3: DialogueChoiceOption and DialogueChoice**

File: `src/KerbalDialogueKit/Core/DialogueChoiceOption.cs`
```csharp
namespace KerbalDialogueKit.Core
{
    public sealed class DialogueChoiceOption
    {
        public string Value { get; set; }
        public string Text { get; set; }
        public string Description { get; set; }
    }
}
```

File: `src/KerbalDialogueKit/Core/DialogueChoice.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// A pause point in a scene presenting 2-4 option cards. When the player
    /// selects one, the flag named Id is set to the chosen option's Value, and
    /// the scene resumes. Subsequent lines' VisibleIf can check this flag.
    /// </summary>
    public sealed class DialogueChoice : IDialogueItem
    {
        public string Id { get; set; }
        public string Prompt { get; set; }
        public List<DialogueChoiceOption> Options { get; } = new List<DialogueChoiceOption>();
    }
}
```

- [ ] **Step 4: IDialogueItem marker interface**

File: `src/KerbalDialogueKit/Core/DialogueLine.cs` — add interface next to the DialogueLine type. Actually put the interface in its own file for clarity:

File: `src/KerbalDialogueKit/Core/IDialogueItem.cs`
```csharp
namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// Marker for items that can appear in a scene's ordered item list:
    /// DialogueLine or DialogueChoice.
    /// </summary>
    public interface IDialogueItem { }
}
```

(DialogueLine and DialogueChoice already implement it via the `: IDialogueItem` in their class declarations above.)

- [ ] **Step 5: SceneCallbacks**

File: `src/KerbalDialogueKit/Core/SceneCallbacks.cs`
```csharp
using System;

namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// Optional callbacks fired at scene lifecycle events. All are nullable;
    /// the renderer skips any that are null.
    /// </summary>
    public sealed class SceneCallbacks
    {
        public Action<DialogueScene> OnSceneStart;
        public Action<int, DialogueLine> OnLineActivated;
        public Action<string, string> OnChoiceMade;  // (choiceId, value)
        public Action<DialogueScene> OnSceneEnd;
        public Action<DialogueScene> OnSceneCancelled;
    }
}
```

- [ ] **Step 6: DialogueScene with fluent builder**

File: `src/KerbalDialogueKit/Core/DialogueScene.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// A scene: title, up to two characters, and an ordered sequence of lines
    /// and choices. Scenes are constructed by consumers (either in code via
    /// the fluent API, or loaded from DIALOGUE_SCENE cfg nodes) and enqueued
    /// via DialogueKit.Enqueue.
    /// </summary>
    public sealed class DialogueScene
    {
        public string Id { get; set; }
        public string Title { get; set; }
        public DialogueCharacter CharacterA { get; set; }
        public DialogueCharacter CharacterB { get; set; }
        public List<IDialogueItem> Items { get; } = new List<IDialogueItem>();
        public SceneCallbacks Callbacks { get; } = new SceneCallbacks();

        public DialogueScene() { }

        public DialogueScene(string id)
        {
            Id = id;
        }

        // ---- Fluent builder ----

        public DialogueScene WithTitle(string title)
        {
            Title = title;
            return this;
        }

        public DialogueScene WithCharacter(string slot, DialogueCharacter character)
        {
            character.Id = slot;
            if (slot == "A") CharacterA = character;
            else if (slot == "B") CharacterB = character;
            return this;
        }

        public DialogueScene AddLine(string speaker, string text, string animation = "idle", string visibleIf = null)
        {
            Items.Add(new DialogueLine
            {
                Speaker = speaker,
                Text = text,
                Animation = animation,
                VisibleIf = visibleIf
            });
            return this;
        }

        public DialogueScene AddChoice(string id, params DialogueChoiceOption[] options)
        {
            var choice = new DialogueChoice { Id = id };
            choice.Options.AddRange(options);
            Items.Add(choice);
            return this;
        }

        public DialogueScene OnSceneStart(System.Action<DialogueScene> cb)
        { Callbacks.OnSceneStart = cb; return this; }

        public DialogueScene OnLineActivated(System.Action<int, DialogueLine> cb)
        { Callbacks.OnLineActivated = cb; return this; }

        public DialogueScene OnChoiceMade(System.Action<string, string> cb)
        { Callbacks.OnChoiceMade = cb; return this; }

        public DialogueScene OnSceneEnd(System.Action<DialogueScene> cb)
        { Callbacks.OnSceneEnd = cb; return this; }

        public DialogueScene OnSceneCancelled(System.Action<DialogueScene> cb)
        { Callbacks.OnSceneCancelled = cb; return this; }
    }
}
```

- [ ] **Step 7: Build and commit**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build.

```bash
git add src/KerbalDialogueKit/Core/
git commit -m "feat: add DialogueScene data model with fluent builder"
```

---

## Task 12: DialogueSceneBuilder Tests

Unit tests for the fluent API. Confirms the builder wires items correctly.

**Files:**
- Create: `tests/KerbalDialogueKit.Tests/DialogueSceneBuilderTests.cs`

- [ ] **Step 1: Write the tests**

File: `tests/KerbalDialogueKit.Tests/DialogueSceneBuilderTests.cs`
```csharp
using KerbalDialogueKit.Core;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class DialogueSceneBuilderTests
    {
        private DialogueCharacter CharA() => new DialogueCharacter
        {
            Name = "Wernher",
            PortraitModel = "Instructor_Wernher",
            Color = "#FF82B4E8"
        };

        private DialogueCharacter CharB() => new DialogueCharacter
        {
            Name = "Gus",
            PortraitModel = "Strategy_MechanicGuy",
            Color = "#FFFFC078"
        };

        [Fact]
        public void EmptySceneHasNoItems()
        {
            var scene = new DialogueScene("empty");
            Assert.Empty(scene.Items);
        }

        [Fact]
        public void WithCharacterAssignsSlot()
        {
            var scene = new DialogueScene()
                .WithCharacter("A", CharA())
                .WithCharacter("B", CharB());

            Assert.Equal("Wernher", scene.CharacterA.Name);
            Assert.Equal("Gus", scene.CharacterB.Name);
            Assert.Equal("A", scene.CharacterA.Id);
            Assert.Equal("B", scene.CharacterB.Id);
        }

        [Fact]
        public void AddLineAppendsDialogueLine()
        {
            var scene = new DialogueScene()
                .AddLine("A", "Hello.");
            Assert.Single(scene.Items);
            var line = (DialogueLine)scene.Items[0];
            Assert.Equal("A", line.Speaker);
            Assert.Equal("Hello.", line.Text);
            Assert.Equal("idle", line.Animation);
            Assert.Null(line.VisibleIf);
        }

        [Fact]
        public void AddLineRespectsVisibleIf()
        {
            var scene = new DialogueScene()
                .AddLine("A", "Only if.", visibleIf: "mood == Enthusiastic");
            var line = (DialogueLine)scene.Items[0];
            Assert.Equal("mood == Enthusiastic", line.VisibleIf);
        }

        [Fact]
        public void AddChoiceAppendsDialogueChoice()
        {
            var scene = new DialogueScene()
                .AddChoice("approach",
                    new DialogueChoiceOption { Value = "safe", Text = "Play it safe" },
                    new DialogueChoiceOption { Value = "bold", Text = "Go for it" });
            Assert.Single(scene.Items);
            var choice = (DialogueChoice)scene.Items[0];
            Assert.Equal("approach", choice.Id);
            Assert.Equal(2, choice.Options.Count);
            Assert.Equal("safe", choice.Options[0].Value);
            Assert.Equal("bold", choice.Options[1].Value);
        }

        [Fact]
        public void ItemsOrderedAsBuilt()
        {
            var scene = new DialogueScene()
                .AddLine("A", "First")
                .AddChoice("c", new DialogueChoiceOption { Value = "x", Text = "X" })
                .AddLine("A", "Third");
            Assert.Equal(3, scene.Items.Count);
            Assert.IsType<DialogueLine>(scene.Items[0]);
            Assert.IsType<DialogueChoice>(scene.Items[1]);
            Assert.IsType<DialogueLine>(scene.Items[2]);
        }

        [Fact]
        public void CallbacksFluentApiStoresDelegates()
        {
            bool sceneStartCalled = false;
            var scene = new DialogueScene()
                .OnSceneStart(s => sceneStartCalled = true);
            scene.Callbacks.OnSceneStart?.Invoke(scene);
            Assert.True(sceneStartCalled);
        }
    }
}
```

- [ ] **Step 2: Run tests**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all tests pass.

- [ ] **Step 3: Commit**

```bash
git add tests/KerbalDialogueKit.Tests/DialogueSceneBuilderTests.cs
git commit -m "test: add DialogueScene fluent builder tests"
```

---

## Task 13: LineEvaluator — Filter Lines by visibleIf

Given the current flag state, filter a list of `IDialogueItem` and emit only those whose `VisibleIf` is empty or evaluates true. `DialogueChoice` items always pass through (they're not flag-gated — they're choice points).

**Files:**
- Create: `src/KerbalDialogueKit/Rendering/LineEvaluator.cs`
- Create: `tests/KerbalDialogueKit.Tests/LineEvaluatorTests.cs`

- [ ] **Step 1: Write the tests**

File: `tests/KerbalDialogueKit.Tests/LineEvaluatorTests.cs`
```csharp
using System.Linq;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;
using KerbalDialogueKit.Rendering;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class LineEvaluatorTests
    {
        [Fact]
        public void LineWithNoVisibleIfPasses()
        {
            var line = new DialogueLine { Text = "always" };
            var flags = new FlagStore();
            Assert.True(LineEvaluator.IsVisible(line, flags));
        }

        [Fact]
        public void LineWithEmptyVisibleIfPasses()
        {
            var line = new DialogueLine { Text = "always", VisibleIf = "" };
            var flags = new FlagStore();
            Assert.True(LineEvaluator.IsVisible(line, flags));
        }

        [Fact]
        public void LineWithMatchingVisibleIfPasses()
        {
            var line = new DialogueLine { Text = "happy", VisibleIf = "mood == Enthusiastic" };
            var flags = new FlagStore();
            flags.Set("mood", "Enthusiastic");
            Assert.True(LineEvaluator.IsVisible(line, flags));
        }

        [Fact]
        public void LineWithNonMatchingVisibleIfFails()
        {
            var line = new DialogueLine { Text = "happy", VisibleIf = "mood == Enthusiastic" };
            var flags = new FlagStore();
            flags.Set("mood", "Frustrated");
            Assert.False(LineEvaluator.IsVisible(line, flags));
        }

        [Fact]
        public void ChoiceItemsAlwaysVisible()
        {
            var choice = new DialogueChoice { Id = "x" };
            var flags = new FlagStore();
            Assert.True(LineEvaluator.IsVisible(choice, flags));
        }

        [Fact]
        public void FilterPreservesOrder()
        {
            var flags = new FlagStore();
            flags.Set("mood", "Enthusiastic");
            var items = new IDialogueItem[]
            {
                new DialogueLine { Text = "a", VisibleIf = "mood == Enthusiastic" },
                new DialogueLine { Text = "b", VisibleIf = "mood == Frustrated" },   // skipped
                new DialogueLine { Text = "c" },
                new DialogueChoice { Id = "q" },
                new DialogueLine { Text = "d", VisibleIf = "mood == Enthusiastic" }
            };
            var kept = LineEvaluator.Filter(items, flags).ToList();
            Assert.Equal(4, kept.Count);
            Assert.Equal("a", ((DialogueLine)kept[0]).Text);
            Assert.Equal("c", ((DialogueLine)kept[1]).Text);
            Assert.IsType<DialogueChoice>(kept[2]);
            Assert.Equal("d", ((DialogueLine)kept[3]).Text);
        }
    }
}
```

- [ ] **Step 2: Implement LineEvaluator**

File: `src/KerbalDialogueKit/Rendering/LineEvaluator.cs`
```csharp
using System.Collections.Generic;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;

namespace KerbalDialogueKit.Rendering
{
    /// <summary>
    /// Filters scene items by their VisibleIf expression against the current
    /// flag state. DialogueChoice items always pass through; they are choice
    /// points, not flag-gated content.
    /// </summary>
    public static class LineEvaluator
    {
        public static bool IsVisible(IDialogueItem item, FlagStore flags)
        {
            if (item is DialogueChoice) return true;
            if (item is DialogueLine line)
            {
                if (string.IsNullOrWhiteSpace(line.VisibleIf)) return true;
                var expr = FlagExpressionParser.Parse(line.VisibleIf);
                return expr.Evaluate(flags);
            }
            return true;
        }

        public static IEnumerable<IDialogueItem> Filter(IEnumerable<IDialogueItem> items, FlagStore flags)
        {
            foreach (var item in items)
            {
                if (IsVisible(item, flags)) yield return item;
            }
        }
    }
}
```

- [ ] **Step 3: Run tests**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all LineEvaluator tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalDialogueKit/Rendering/LineEvaluator.cs tests/KerbalDialogueKit.Tests/LineEvaluatorTests.cs
git commit -m "feat: add LineEvaluator for flag-driven line visibility"
```

---

## Task 14: ChatterBoxBridge — Reflection Wrapper

Single file that contains all reflection against ChatterBox internals. If our upstream PR lands, we swap this file's guts for direct calls without touching any caller.

**Files:**
- Create: `src/KerbalDialogueKit/Rendering/ChatterBoxBridge.cs`

- [ ] **Step 1: Implement the bridge**

File: `src/KerbalDialogueKit/Rendering/ChatterBoxBridge.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Reflection;
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalDialogueKit.Rendering
{
    /// <summary>
    /// Wraps reflection against ChatterBox's internal rendering classes.
    /// If ChatterBox exposes these types as public (upstream PR), this file
    /// can be rewritten to use direct type references without touching
    /// consumers. That is its only reason to exist.
    /// </summary>
    internal static class ChatterBoxBridge
    {
        private static Assembly chatterBoxAssembly;
        private static Type scenePopupDefinitionType;
        private static Type characterConfigType;
        private static Type lineConfigType;
        private static Type scenePopupControllerType;
        private static Type sceneSpeakerType;
        private static Type scenePresentationModeType;
        private static MethodInfo enqueueMethod;
        private static bool loaded;
        private static bool loadSucceeded;

        public static bool EnsureLoaded()
        {
            if (loaded) return loadSucceeded;
            loaded = true;

            try
            {
                chatterBoxAssembly = AppDomain.CurrentDomain.GetAssemblies()
                    .FirstOrDefault(a => a.GetName().Name == "ChatterBox");
                if (chatterBoxAssembly == null)
                {
                    Debug.LogWarning("[KerbalDialogueKit] ChatterBox assembly not found at runtime.");
                    return false;
                }

                scenePopupDefinitionType = chatterBoxAssembly.GetType("ChatterBox.ScenePopupDefinition", throwOnError: false);
                characterConfigType = chatterBoxAssembly.GetType("ChatterBox.CharacterConfig", throwOnError: false);
                lineConfigType = chatterBoxAssembly.GetType("ChatterBox.LineConfig", throwOnError: false);
                scenePopupControllerType = chatterBoxAssembly.GetType("ChatterBox.ScenePopupController", throwOnError: false);
                sceneSpeakerType = chatterBoxAssembly.GetType("ChatterBox.SceneSpeaker", throwOnError: false);
                scenePresentationModeType = chatterBoxAssembly.GetType("ChatterBox.ScenePresentationMode", throwOnError: false);

                if (scenePopupDefinitionType == null || characterConfigType == null ||
                    lineConfigType == null || scenePopupControllerType == null)
                {
                    Debug.LogWarning("[KerbalDialogueKit] One or more ChatterBox types missing; integration disabled.");
                    return false;
                }

                enqueueMethod = scenePopupControllerType.GetMethod(
                    "Enqueue",
                    BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Static,
                    binder: null,
                    types: new[] { scenePopupDefinitionType },
                    modifiers: null);

                if (enqueueMethod == null)
                {
                    Debug.LogWarning("[KerbalDialogueKit] ChatterBox.ScenePopupController.Enqueue not found.");
                    return false;
                }

                loadSucceeded = true;
                Debug.Log("[KerbalDialogueKit] ChatterBox reflection bindings initialized.");
                return true;
            }
            catch (Exception ex)
            {
                Debug.LogError($"[KerbalDialogueKit] ChatterBox bridge init failed: {ex}");
                return false;
            }
        }

        /// <summary>
        /// Build a ChatterBox ScenePopupDefinition from our DialogueScene.
        /// Pre-filters items by VisibleIf — any line whose expression evaluates
        /// false is skipped. Choice points are elided here; choices are handled
        /// out-of-band by splitting scenes and opening ChoiceOverlay between them.
        /// </summary>
        public static object BuildScenePopupDefinition(
            string title,
            DialogueCharacter charA,
            DialogueCharacter charB,
            IEnumerable<DialogueLine> lines)
        {
            if (!EnsureLoaded()) return null;

            var def = Activator.CreateInstance(scenePopupDefinitionType);
            SetField(def, "Title", title ?? "");
            SetField(def, "Scale", 0.50f);
            SetField(def, "DimInactiveSpeaker", true);
            SetField(def, "DimGameAudio", true);
            SetField(def, "PauseGame", true);
            SetField(def, "NextButtonText", "Next");
            SetField(def, "CloseButtonText", "Close");
            SetField(def, "TitleColor", Color.white);
            SetField(def, "DefaultTextColor", new Color(0.9f, 0.9f, 0.9f, 1f));

            // Presentation: Dialogue (value 0 in ChatterBox's enum)
            SetField(def, "Presentation", Enum.ToObject(scenePresentationModeType, 0));

            if (charA != null) SetField(def, "CharacterA", BuildCharacterConfig(charA));
            if (charB != null) SetField(def, "CharacterB", BuildCharacterConfig(charB));

            var lineListType = typeof(List<>).MakeGenericType(lineConfigType);
            var lineList = (System.Collections.IList)Activator.CreateInstance(lineListType);
            foreach (var line in lines)
            {
                lineList.Add(BuildLineConfig(line));
            }
            SetField(def, "Lines", lineList);

            return def;
        }

        public static void Enqueue(object scenePopupDefinition)
        {
            if (!EnsureLoaded()) return;
            try
            {
                enqueueMethod.Invoke(null, new[] { scenePopupDefinition });
            }
            catch (Exception ex)
            {
                Debug.LogError($"[KerbalDialogueKit] Failed to enqueue scene: {ex}");
            }
        }

        private static object BuildCharacterConfig(DialogueCharacter c)
        {
            var cc = Activator.CreateInstance(characterConfigType);
            SetField(cc, "Name", c.Name ?? "");
            SetField(cc, "Model", c.PortraitModel ?? "");
            SetField(cc, "Animation", c.Animation ?? "idle");
            SetField(cc, "Color", ParseColor(c.Color));
            return cc;
        }

        private static object BuildLineConfig(DialogueLine line)
        {
            var lc = Activator.CreateInstance(lineConfigType);
            SetField(lc, "Text", line.Text ?? "");
            SetField(lc, "Animation", line.Animation ?? "idle");

            // Speaker: A=0, B=1 (ChatterBox's SceneSpeaker enum)
            int speakerVal = (line.Speaker == "B") ? 1 : 0;
            SetField(lc, "Speaker", Enum.ToObject(sceneSpeakerType, speakerVal));

            if (!string.IsNullOrEmpty(line.LineAudio))
            {
                SetField(lc, "LineAudio", line.LineAudio);
                SetField(lc, "LineLevel", line.LineLevel);
            }
            return lc;
        }

        private static void SetField(object target, string name, object value)
        {
            var type = target.GetType();
            var f = type.GetField(name, BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
            if (f != null) { f.SetValue(target, value); return; }
            var p = type.GetProperty(name, BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);
            if (p != null && p.CanWrite) { p.SetValue(target, value, null); return; }
            Debug.LogWarning($"[KerbalDialogueKit] Could not set '{name}' on {type.FullName}");
        }

        private static Color ParseColor(string hex)
        {
            if (string.IsNullOrWhiteSpace(hex)) return Color.white;
            hex = hex.TrimStart('#');
            if (hex.Length == 8)
            {
                byte a = Convert.ToByte(hex.Substring(0, 2), 16);
                byte r = Convert.ToByte(hex.Substring(2, 2), 16);
                byte g = Convert.ToByte(hex.Substring(4, 2), 16);
                byte b = Convert.ToByte(hex.Substring(6, 2), 16);
                return new Color(r / 255f, g / 255f, b / 255f, a / 255f);
            }
            if (hex.Length == 6)
            {
                byte r = Convert.ToByte(hex.Substring(0, 2), 16);
                byte g = Convert.ToByte(hex.Substring(2, 2), 16);
                byte b = Convert.ToByte(hex.Substring(4, 2), 16);
                return new Color(r / 255f, g / 255f, b / 255f);
            }
            return Color.white;
        }
    }
}
```

Note on the field names: I'm inferring these from reading ChatterBox source earlier. If any don't match (e.g., `Model` vs `ModelPath`), the `SetField` helper logs a warning and continues. During in-game validation (Task 18) we iterate on this.

- [ ] **Step 2: Build**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 3: Commit**

```bash
git add src/KerbalDialogueKit/Rendering/ChatterBoxBridge.cs
git commit -m "feat: add ChatterBoxBridge reflection wrapper"
```

---

## Task 15: SceneQueue and SceneRenderer

The renderer takes a `DialogueScene`, splits it at each `DialogueChoice` into segments, and plays them in order: segment → choice overlay → next segment → ... until done. Line filtering via `LineEvaluator` happens per segment, so choices in an earlier segment can enable lines in a later segment.

**Files:**
- Create: `src/KerbalDialogueKit/Core/SceneQueue.cs`
- Create: `src/KerbalDialogueKit/Rendering/SceneRenderer.cs`

- [ ] **Step 1: Implement SceneQueue**

File: `src/KerbalDialogueKit/Core/SceneQueue.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// Queue of scenes awaiting playback. Enqueue appends; EnqueuePriority
    /// prepends. The renderer pulls one scene at a time.
    /// </summary>
    internal sealed class SceneQueue
    {
        private readonly LinkedList<DialogueScene> scenes = new LinkedList<DialogueScene>();

        public int Count => scenes.Count;
        public bool IsEmpty => scenes.Count == 0;

        public void Enqueue(DialogueScene scene)
        {
            scenes.AddLast(scene);
        }

        public void EnqueuePriority(DialogueScene scene)
        {
            scenes.AddFirst(scene);
        }

        public DialogueScene Dequeue()
        {
            if (scenes.First == null) return null;
            var scene = scenes.First.Value;
            scenes.RemoveFirst();
            return scene;
        }

        public void Clear()
        {
            scenes.Clear();
        }
    }
}
```

- [ ] **Step 2: Implement SceneRenderer**

File: `src/KerbalDialogueKit/Rendering/SceneRenderer.cs`
```csharp
using System.Collections.Generic;
using UnityEngine;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;

namespace KerbalDialogueKit.Rendering
{
    /// <summary>
    /// Orchestrates playback of a DialogueScene: splits into segments at choices,
    /// filters lines via LineEvaluator, hands segments to ChatterBoxBridge, and
    /// opens ChoiceOverlay between segments when a choice is present.
    /// </summary>
    internal sealed class SceneRenderer
    {
        private readonly SceneQueue queue = new SceneQueue();
        private readonly FlagStore flags;
        private readonly System.Func<ChoiceOverlay> overlayFactory;

        private DialogueScene currentScene;
        private List<IDialogueItem> segments;
        private int segmentIndex;
        private bool playingChatterBox;

        public SceneRenderer(FlagStore flags, System.Func<ChoiceOverlay> overlayFactory)
        {
            this.flags = flags;
            this.overlayFactory = overlayFactory;
        }

        public int QueueLength => queue.Count;
        public bool IsPlaying => currentScene != null;

        public void Enqueue(DialogueScene scene) => queue.Enqueue(scene);
        public void EnqueuePriority(DialogueScene scene) => queue.EnqueuePriority(scene);

        public void ClearQueue() => queue.Clear();

        /// <summary>
        /// Called each frame from the Addon's Update. Advances whichever stage
        /// we're in — starts next scene, opens overlay for pending choice, etc.
        /// </summary>
        public void Tick()
        {
            if (currentScene == null && !queue.IsEmpty)
            {
                StartNextScene();
                return;
            }

            // We delegate line playback to ChatterBox. When we're waiting on it,
            // there's nothing to do here — we resume via the overlay close callback
            // or via polling ChatterBox's "is active" state (not currently done —
            // see note below).
        }

        private void StartNextScene()
        {
            currentScene = queue.Dequeue();
            segments = BuildSegments(currentScene);
            segmentIndex = 0;
            currentScene.Callbacks.OnSceneStart?.Invoke(currentScene);
            PlayNextSegment();
        }

        /// <summary>
        /// Split scene.Items at every DialogueChoice into a sequence:
        /// [lines...], choice, [lines...], choice, [lines...].
        /// The segments alternate between "run of lines" (List&lt;DialogueLine&gt;)
        /// and "choice" (DialogueChoice).
        /// </summary>
        private static List<IDialogueItem> BuildSegments(DialogueScene scene)
        {
            var segs = new List<IDialogueItem>();
            var currentRun = new LineRun();
            foreach (var item in scene.Items)
            {
                if (item is DialogueChoice)
                {
                    if (currentRun.Lines.Count > 0) { segs.Add(currentRun); currentRun = new LineRun(); }
                    segs.Add(item);
                }
                else if (item is DialogueLine l)
                {
                    currentRun.Lines.Add(l);
                }
            }
            if (currentRun.Lines.Count > 0) segs.Add(currentRun);
            return segs;
        }

        private void PlayNextSegment()
        {
            if (currentScene == null) { FinishScene(); return; }

            while (segmentIndex < segments.Count)
            {
                var seg = segments[segmentIndex];
                if (seg is LineRun run)
                {
                    var filtered = new List<DialogueLine>();
                    foreach (var line in run.Lines)
                    {
                        if (LineEvaluator.IsVisible(line, flags))
                            filtered.Add(line);
                    }
                    if (filtered.Count > 0)
                    {
                        var def = ChatterBoxBridge.BuildScenePopupDefinition(
                            currentScene.Title,
                            currentScene.CharacterA,
                            currentScene.CharacterB,
                            filtered);
                        if (def != null)
                        {
                            ChatterBoxBridge.Enqueue(def);
                            playingChatterBox = true;
                            segmentIndex++;
                            return;  // wait for ChatterBox playback to finish
                        }
                    }
                    segmentIndex++;
                    continue;
                }
                if (seg is DialogueChoice choice)
                {
                    var overlay = overlayFactory();
                    overlay.Show(choice, selected =>
                    {
                        flags.Set(choice.Id, selected.Value);
                        currentScene.Callbacks.OnChoiceMade?.Invoke(choice.Id, selected.Value);
                        segmentIndex++;
                        PlayNextSegment();
                    });
                    return;
                }
                segmentIndex++;
            }
            FinishScene();
        }

        /// <summary>
        /// Called by the addon when ChatterBox finishes its playback (we poll
        /// "is controller active" — see Addon).
        /// </summary>
        public void OnChatterBoxFinished()
        {
            if (!playingChatterBox) return;
            playingChatterBox = false;
            PlayNextSegment();
        }

        private void FinishScene()
        {
            var scene = currentScene;
            currentScene = null;
            segments = null;
            segmentIndex = 0;
            scene?.Callbacks.OnSceneEnd?.Invoke(scene);
        }

        public void CancelCurrent()
        {
            if (currentScene == null) return;
            var scene = currentScene;
            currentScene = null;
            segments = null;
            segmentIndex = 0;
            playingChatterBox = false;
            scene.Callbacks.OnSceneCancelled?.Invoke(scene);
        }

        private sealed class LineRun : IDialogueItem
        {
            public List<DialogueLine> Lines { get; } = new List<DialogueLine>();
        }
    }
}
```

**Note on playback synchronization:** We rely on the addon polling ChatterBox's "is a controller active" state each frame. The addon calls `SceneRenderer.OnChatterBoxFinished()` when it detects the transition from active → inactive. See Task 17 for that wiring.

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build (ChoiceOverlay referenced but not yet defined — will fail. Create a stub so build passes.)

If the build fails with "ChoiceOverlay not found", create the stub now:

File: `src/KerbalDialogueKit/Rendering/ChoiceOverlay.cs` (stub, expanded in Task 16)
```csharp
using System;
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalDialogueKit.Rendering
{
    internal sealed class ChoiceOverlay : MonoBehaviour
    {
        public void Show(DialogueChoice choice, Action<DialogueChoiceOption> onSelect)
        {
            // Stub - implementation in Task 16
            onSelect?.Invoke(choice.Options.Count > 0 ? choice.Options[0] : null);
            Destroy(gameObject);
        }
    }
}
```

Rebuild. Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalDialogueKit/Core/SceneQueue.cs src/KerbalDialogueKit/Rendering/SceneRenderer.cs src/KerbalDialogueKit/Rendering/ChoiceOverlay.cs
git commit -m "feat: add SceneRenderer orchestration with choice-based segmentation"
```

---

## Task 16: ChoiceOverlay — IMGUI Choice Card UI

Replaces the stub. Draws choice cards using Unity's IMGUI (`OnGUI`). KSP scenes often use IMGUI for overlays — it's unstyled but functional, and we're optimizing for shippable-in-two-weeks, not pretty.

**Files:**
- Modify: `src/KerbalDialogueKit/Rendering/ChoiceOverlay.cs`

- [ ] **Step 1: Replace the stub with a real IMGUI implementation**

File: `src/KerbalDialogueKit/Rendering/ChoiceOverlay.cs`
```csharp
using System;
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalDialogueKit.Rendering
{
    /// <summary>
    /// MonoBehaviour overlay drawn via Unity IMGUI. Displays choice prompt
    /// and option cards; invokes onSelect with the chosen option then destroys
    /// its own GameObject.
    /// </summary>
    internal sealed class ChoiceOverlay : MonoBehaviour
    {
        private DialogueChoice choice;
        private Action<DialogueChoiceOption> onSelect;
        private bool chosen;

        private const float CardWidth = 520f;
        private const float CardHeightEstimate = 96f;
        private const float Padding = 12f;

        private GUIStyle windowStyle;
        private GUIStyle cardButtonStyle;
        private GUIStyle promptStyle;
        private GUIStyle descStyle;
        private bool stylesReady;

        public void Show(DialogueChoice choice, Action<DialogueChoiceOption> onSelect)
        {
            this.choice = choice;
            this.onSelect = onSelect;
            this.chosen = false;
        }

        public void OnGUI()
        {
            if (choice == null) return;
            if (!stylesReady) InitStyles();

            int count = choice.Options.Count;
            float totalHeight = 60f + (count * (CardHeightEstimate + Padding)) + Padding;
            Rect window = new Rect(
                (Screen.width - CardWidth) / 2f,
                Mathf.Max(80f, (Screen.height - totalHeight) / 2f),
                CardWidth,
                totalHeight);

            GUI.ModalWindow(typeof(ChoiceOverlay).GetHashCode(), window, DrawContents,
                choice.Prompt ?? "Make your choice", windowStyle);
        }

        private void DrawContents(int windowId)
        {
            GUILayout.Space(4);
            foreach (var opt in choice.Options)
            {
                if (DrawCard(opt))
                {
                    if (chosen) return;
                    chosen = true;
                    var callback = onSelect;
                    var selected = opt;
                    onSelect = null;
                    choice = null;
                    try { callback?.Invoke(selected); }
                    finally { Destroy(gameObject); }
                    return;
                }
                GUILayout.Space(Padding);
            }
        }

        private bool DrawCard(DialogueChoiceOption opt)
        {
            GUILayout.BeginVertical(GUI.skin.box);
            bool click = GUILayout.Button(opt.Text ?? "", cardButtonStyle, GUILayout.Height(28f));
            if (!string.IsNullOrEmpty(opt.Description))
            {
                GUILayout.Label(opt.Description, descStyle);
            }
            GUILayout.EndVertical();
            return click;
        }

        private void InitStyles()
        {
            windowStyle = new GUIStyle(GUI.skin.window) { padding = new RectOffset(12, 12, 28, 12) };
            cardButtonStyle = new GUIStyle(GUI.skin.button) { fontStyle = FontStyle.Bold, alignment = TextAnchor.MiddleLeft };
            promptStyle = new GUIStyle(GUI.skin.label) { fontStyle = FontStyle.Bold, fontSize = 14 };
            descStyle = new GUIStyle(GUI.skin.label) { wordWrap = true, fontStyle = FontStyle.Italic };
            stylesReady = true;
        }
    }
}
```

- [ ] **Step 2: Build**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 3: Commit**

```bash
git add src/KerbalDialogueKit/Rendering/ChoiceOverlay.cs
git commit -m "feat: implement ChoiceOverlay with IMGUI choice cards"
```

---

## Task 17: DialogueKit Public API and Addon Wiring

The user-facing entry point and the `KSPAddon` that wires everything together — creates the SceneRenderer, hooks `Update()` to drive playback and detect when ChatterBox finishes.

**Files:**
- Create: `src/KerbalDialogueKit/Core/DialogueKit.cs`
- Modify: `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs`
- Create: `src/KerbalDialogueKit/Events/DialogueEvents.cs`

- [ ] **Step 1: DialogueEvents — GameEvents for music cues**

File: `src/KerbalDialogueKit/Events/DialogueEvents.cs`
```csharp
namespace KerbalDialogueKit.Events
{
    /// <summary>
    /// Events fired by KerbalDialogueKit. Music mods and others can subscribe
    /// via GameEvents.FindEvent&lt;EventData&lt;string&gt;&gt;(name).
    /// </summary>
    public static class DialogueEvents
    {
        public const string MusicCueEventName = "KerbalDialogueKit.MusicCue";
    }
}
```

- [ ] **Step 2: DialogueKit static API**

File: `src/KerbalDialogueKit/Core/DialogueKit.cs`
```csharp
using KerbalDialogueKit.Flags;
using KerbalDialogueKit.Rendering;

namespace KerbalDialogueKit.Core
{
    /// <summary>
    /// Public static entry point. Consumers call these methods to interact
    /// with KerbalDialogueKit.
    /// </summary>
    public static class DialogueKit
    {
        private static SceneRenderer renderer;
        private static FlagStore flagsRef;

        /// <summary>
        /// Called once by KerbalDialogueKitAddon at startup. Not for public use.
        /// </summary>
        internal static void Initialize(SceneRenderer sceneRenderer, FlagStore flags)
        {
            renderer = sceneRenderer;
            flagsRef = flags;
        }

        /// <summary>
        /// Current flag store. Flags survive across scene changes; persisted
        /// per-save by FlagScenario.
        /// </summary>
        public static FlagStore Flags => flagsRef;

        /// <summary>Add a scene to the end of the queue.</summary>
        public static void Enqueue(DialogueScene scene) => renderer?.Enqueue(scene);

        /// <summary>Add a scene to the front of the queue.</summary>
        public static void EnqueuePriority(DialogueScene scene) => renderer?.EnqueuePriority(scene);

        /// <summary>True if a scene is currently playing.</summary>
        public static bool IsPlaying() => renderer != null && renderer.IsPlaying;

        /// <summary>Number of scenes waiting in the queue (excluding any current).</summary>
        public static int QueueLength() => renderer?.QueueLength ?? 0;

        /// <summary>Drop all queued scenes. The current scene continues.</summary>
        public static void ClearQueue() => renderer?.ClearQueue();
    }
}
```

- [ ] **Step 3: Replace the addon with real wiring**

File: `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs`
```csharp
using System;
using System.Linq;
using System.Reflection;
using UnityEngine;
using KerbalDialogueKit.Core;
using KerbalDialogueKit.Flags;
using KerbalDialogueKit.Rendering;

namespace KerbalDialogueKit.Runtime
{
    /// <summary>
    /// Single persistent object that initializes KerbalDialogueKit, owns the
    /// SceneRenderer, and drives playback each frame.
    /// </summary>
    [KSPAddon(KSPAddon.Startup.Instantly, true)]
    public sealed class KerbalDialogueKitAddon : MonoBehaviour
    {
        private SceneRenderer renderer;
        private FlagStore flags;

        // Reflection handles for polling ChatterBox "is playing" state
        private Type scenePopupControllerType;
        private FieldInfo currentField;
        private object controllerInstance;
        private bool bridgeReady;

        // Track last-seen state so we fire OnChatterBoxFinished on transitions
        private bool chatterBoxWasActive;

        public void Awake()
        {
            DontDestroyOnLoad(this);

            // Use FlagScenario's FlagStore if it exists in-save; otherwise a scratch
            // store. Swapped by the addon below whenever a scenario loads.
            flags = new FlagStore();
            renderer = new SceneRenderer(flags, CreateChoiceOverlay);
            DialogueKit.Initialize(renderer, flags);

            GameEvents.onGameStateLoad.Add(OnGameStateLoaded);
            Debug.Log("[KerbalDialogueKit] Initialized v0.1.0");
        }

        public void OnDestroy()
        {
            GameEvents.onGameStateLoad.Remove(OnGameStateLoaded);
        }

        private void OnGameStateLoaded(ConfigNode _)
        {
            // Swap to the persistent store once FlagScenario is loaded
            if (FlagScenario.Instance != null)
            {
                // Copy scratch values into persistent store
                foreach (var k in flags.Keys)
                    FlagScenario.Instance.Flags.Set(k, flags.Get(k));
                flags = FlagScenario.Instance.Flags;
                // Recreate renderer against the new store
                renderer = new SceneRenderer(flags, CreateChoiceOverlay);
                DialogueKit.Initialize(renderer, flags);
                Debug.Log("[KerbalDialogueKit] Swapped to per-save flag store.");
            }
        }

        private ChoiceOverlay CreateChoiceOverlay()
        {
            var go = new GameObject("KDK-ChoiceOverlay");
            var overlay = go.AddComponent<ChoiceOverlay>();
            return overlay;
        }

        public void Update()
        {
            EnsureBridgeHandles();
            renderer?.Tick();

            bool active = IsChatterBoxActive();
            if (chatterBoxWasActive && !active)
            {
                renderer?.OnChatterBoxFinished();
            }
            chatterBoxWasActive = active;
        }

        private void EnsureBridgeHandles()
        {
            if (bridgeReady) return;
            try
            {
                var asm = AppDomain.CurrentDomain.GetAssemblies()
                    .FirstOrDefault(a => a.GetName().Name == "ChatterBox");
                if (asm == null) return;
                scenePopupControllerType = asm.GetType("ChatterBox.ScenePopupController", false);
                if (scenePopupControllerType == null) return;

                var instanceField = scenePopupControllerType.GetField(
                    "instance", BindingFlags.NonPublic | BindingFlags.Static);
                if (instanceField == null) return;
                controllerInstance = instanceField.GetValue(null);  // may be null until a scene has been enqueued
                currentField = scenePopupControllerType.GetField(
                    "current", BindingFlags.NonPublic | BindingFlags.Instance);
                bridgeReady = currentField != null;
            }
            catch (Exception e)
            {
                Debug.LogError($"[KerbalDialogueKit] Addon handle init failed: {e}");
            }
        }

        private bool IsChatterBoxActive()
        {
            if (!bridgeReady) return false;
            try
            {
                if (controllerInstance == null)
                {
                    var instanceField = scenePopupControllerType.GetField(
                        "instance", BindingFlags.NonPublic | BindingFlags.Static);
                    controllerInstance = instanceField?.GetValue(null);
                    if (controllerInstance == null) return false;
                }
                var cur = currentField.GetValue(controllerInstance);
                return cur != null;
            }
            catch { return false; }
        }
    }
}
```

- [ ] **Step 4: Build**

```bash
dotnet build src/KerbalDialogueKit/KerbalDialogueKit.csproj -c Release
```

Expected: clean build.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalDialogueKit/Core/DialogueKit.cs src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs src/KerbalDialogueKit/Events/DialogueEvents.cs
git commit -m "feat: wire SceneRenderer into KSPAddon; DialogueKit public API

Addon owns renderer, polls ChatterBox 'is playing' state each frame via
reflection, fires SceneRenderer.OnChatterBoxFinished on the transition
from active to inactive. ChoiceOverlay created on demand per choice."
```

---

## Task 18: Demo Mod — End-to-End Manual Verification

A tiny sibling project that exercises the public API from in-game so you can visually confirm the system works. Without this, we have no way to know whether the reflection bridge actually succeeds in KSP's runtime.

**Files:**
- Create: `demo/KerbalDialogueKit.Demo/KerbalDialogueKit.Demo.csproj`
- Create: `demo/KerbalDialogueKit.Demo/DemoTrigger.cs`

- [ ] **Step 1: Create the demo project**

```bash
mkdir -p demo/KerbalDialogueKit.Demo
```

File: `demo/KerbalDialogueKit.Demo/KerbalDialogueKit.Demo.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <AssemblyName>KerbalDialogueKit.Demo</AssemblyName>
    <RootNamespace>KerbalDialogueKit.Demo</RootNamespace>
    <LangVersion>7.3</LangVersion>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <OutputPath>$(MSBuildThisFileDirectory)..\..\Plugins\</OutputPath>
    <GenerateAssemblyInfo>true</GenerateAssemblyInfo>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="Assembly-CSharp">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp.dll</HintPath>
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
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\KerbalDialogueKit\KerbalDialogueKit.csproj">
      <Private>false</Private>
    </ProjectReference>
  </ItemGroup>

</Project>
```

- [ ] **Step 2: Demo trigger with an IMGUI button**

File: `demo/KerbalDialogueKit.Demo/DemoTrigger.cs`
```csharp
using UnityEngine;
using KerbalDialogueKit.Core;

namespace KerbalDialogueKit.Demo
{
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class DemoTrigger : MonoBehaviour
    {
        public void OnGUI()
        {
            if (GUI.Button(new Rect(20, 20, 180, 28), "KDK Demo Scene"))
            {
                RunDemo();
            }
        }

        private static void RunDemo()
        {
            var wernher = new DialogueCharacter
            {
                Name = "Wernher",
                PortraitModel = "Instructor_Wernher",
                Color = "#FF82B4E8"
            };
            var gus = new DialogueCharacter
            {
                Name = "Gus",
                PortraitModel = "Strategy_MechanicGuy",
                Color = "#FFFFC078"
            };

            var scene = new DialogueScene("kdk_demo")
                .WithTitle("Demo Scene")
                .WithCharacter("A", wernher)
                .WithCharacter("B", gus)
                .AddLine("A", "This is the dialogue kit working end-to-end.")
                .AddLine("B", "Let's see if the choice cards actually render.")
                .AddChoice("demo_choice",
                    new DialogueChoiceOption { Value = "science", Text = "Back science", Description = "The scientific path." },
                    new DialogueChoiceOption { Value = "engineering", Text = "Back engineering", Description = "The practical path." })
                .AddLine("A", "Ah, an excellent scientific decision.",
                    visibleIf: "demo_choice == science")
                .AddLine("B", "A sensible engineering call.",
                    visibleIf: "demo_choice == engineering")
                .OnChoiceMade((id, val) => Debug.Log($"[KDK.Demo] {id} = {val}"))
                .OnSceneEnd(s => Debug.Log("[KDK.Demo] Scene ended"));

            DialogueKit.Enqueue(scene);
        }
    }
}
```

- [ ] **Step 3: Add demo project to solution**

```bash
dotnet sln add demo/KerbalDialogueKit.Demo/KerbalDialogueKit.Demo.csproj
```

- [ ] **Step 4: Build everything**

```bash
dotnet build KerbalDialogueKit.sln -c Release
```

Expected: clean build; `Plugins/KerbalDialogueKit.dll` AND `Plugins/KerbalDialogueKit.Demo.dll` present.

- [ ] **Step 5: Launch KSP, load a save, go to Space Center**

Launch KSP. Start or load a career (or sandbox) save. In the Space Center, look for a "KDK Demo Scene" button in the top-left corner.

Click it. Expected:
1. A ChatterBox-style dialogue window opens with Wernher on the left, Gus on the right
2. "This is the dialogue kit working end-to-end." appears
3. Click Next: "Let's see if the choice cards actually render."
4. Click Next: the ChatterBox window closes and the ChoiceOverlay appears
5. Two cards: "Back science" and "Back engineering"
6. Click one
7. Another ChatterBox window opens with only the line matching your choice
8. KSP.log contains `[KDK.Demo] demo_choice = science` (or engineering) and `[KDK.Demo] Scene ended`

If any step fails, that's the thing to diagnose. The most likely failure points:
- **Reflection field name mismatch**: ChatterBox renamed `ScenePopupDefinition.Lines` or similar. Fix in `ChatterBoxBridge.cs`. The `SetField` helper logs warnings — grep KSP.log for them.
- **Animation name not recognized**: the `DialogueLine.Animation = "idle"` doesn't match what ChatterBox expects. Check `InstructorPortrait.SupportedAnimationNames` in ChatterBox source.
- **ChoiceOverlay doesn't render**: rare, but check that the `MonoBehaviour` is being added to a valid `GameObject` and `OnGUI` fires (put a `Debug.Log` in `OnGUI`).

Iterate with the user until all steps pass. This is the integration-test moment.

- [ ] **Step 6: Commit**

```bash
git add demo/
git commit -m "feat: add demo mod for end-to-end manual verification"
```

---

## Task 19: DialogueSceneLoader — Load Scenes from Cfg

Consumer mods should be able to define scenes in `.cfg` files (the `DIALOGUE_SCENE` node format from the spec). The loader parses these at startup and registers them so code can enqueue them by ID.

**Files:**
- Create: `src/KerbalDialogueKit/Config/DialogueSceneLoader.cs`
- Modify: `src/KerbalDialogueKit/Core/DialogueKit.cs` (add `EnqueueById`)
- Modify: `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs` (call loader at init)
- Create: `tests/KerbalDialogueKit.Tests/DialogueSceneLoaderTests.cs`
- Create: `Examples/demo-scene.cfg`

- [ ] **Step 1: Write loader tests**

Mocking `ConfigNode` directly is painful; instead we write the loader so it works off of a simple structure we can construct in tests. In practice at runtime it reads KSP `ConfigNode`s. Do this by factoring the loader against a lightweight interface:

File: `tests/KerbalDialogueKit.Tests/DialogueSceneLoaderTests.cs`
```csharp
using System.Collections.Generic;
using KerbalDialogueKit.Config;
using KerbalDialogueKit.Core;
using Xunit;

namespace KerbalDialogueKit.Tests
{
    public class DialogueSceneLoaderTests
    {
        private static SceneNode MakeSceneNode()
        {
            var node = new SceneNode();
            node.SetValue("id", "test_scene");
            node.SetValue("title", "A Test Scene");

            var ca = new SceneNode();
            ca.SetValue("id", "A");
            ca.SetValue("name", "Wernher");
            ca.SetValue("model", "Instructor_Wernher");
            ca.SetValue("color", "#FF82B4E8");
            node.AddChild("CHARACTER", ca);

            var line1 = new SceneNode();
            line1.SetValue("speaker", "A");
            line1.SetValue("text", "Hello there.");
            node.AddChild("LINE", line1);

            var choice = new SceneNode();
            choice.SetValue("id", "approach");
            var opt1 = new SceneNode();
            opt1.SetValue("value", "safe");
            opt1.SetValue("text", "Play it safe");
            choice.AddChild("OPTION", opt1);
            var opt2 = new SceneNode();
            opt2.SetValue("value", "bold");
            opt2.SetValue("text", "Go for it");
            choice.AddChild("OPTION", opt2);
            node.AddChild("CHOICE", choice);

            var line2 = new SceneNode();
            line2.SetValue("speaker", "A");
            line2.SetValue("text", "Conservative choice.");
            line2.SetValue("visibleIf", "approach == safe");
            node.AddChild("LINE", line2);
            return node;
        }

        [Fact]
        public void LoadParsesId()
        {
            var scene = DialogueSceneLoader.Load(MakeSceneNode());
            Assert.Equal("test_scene", scene.Id);
            Assert.Equal("A Test Scene", scene.Title);
        }

        [Fact]
        public void LoadParsesCharacter()
        {
            var scene = DialogueSceneLoader.Load(MakeSceneNode());
            Assert.NotNull(scene.CharacterA);
            Assert.Equal("Wernher", scene.CharacterA.Name);
            Assert.Equal("Instructor_Wernher", scene.CharacterA.PortraitModel);
        }

        [Fact]
        public void LoadPreservesItemOrder()
        {
            var scene = DialogueSceneLoader.Load(MakeSceneNode());
            Assert.Equal(3, scene.Items.Count);
            Assert.IsType<DialogueLine>(scene.Items[0]);
            Assert.IsType<DialogueChoice>(scene.Items[1]);
            Assert.IsType<DialogueLine>(scene.Items[2]);
        }

        [Fact]
        public void LoadParsesChoiceOptions()
        {
            var scene = DialogueSceneLoader.Load(MakeSceneNode());
            var choice = (DialogueChoice)scene.Items[1];
            Assert.Equal("approach", choice.Id);
            Assert.Equal(2, choice.Options.Count);
            Assert.Equal("safe", choice.Options[0].Value);
            Assert.Equal("Play it safe", choice.Options[0].Text);
        }

        [Fact]
        public void LoadParsesVisibleIf()
        {
            var scene = DialogueSceneLoader.Load(MakeSceneNode());
            var line = (DialogueLine)scene.Items[2];
            Assert.Equal("approach == safe", line.VisibleIf);
        }
    }

    // Lightweight stand-in for KSP ConfigNode. The loader works against this
    // interface; DialogueSceneLoader.Load(ConfigNode) wraps a real KSP
    // ConfigNode in a ConfigNodeAdapter (added in the implementation).
    internal sealed class SceneNode : ISceneNode
    {
        private readonly Dictionary<string, string> values = new Dictionary<string, string>();
        private readonly List<(string Name, ISceneNode Node)> children = new List<(string, ISceneNode)>();

        public string GetValue(string key) => values.TryGetValue(key, out var v) ? v : null;
        public void SetValue(string key, string val) => values[key] = val;

        public IEnumerable<ISceneNode> GetChildren(string name)
        {
            foreach (var (n, node) in children)
                if (n == name) yield return node;
        }

        public IEnumerable<(string Name, ISceneNode Node)> AllChildren() => children;

        public void AddChild(string name, ISceneNode node) => children.Add((name, node));
    }
}
```

- [ ] **Step 2: Define ISceneNode abstraction and implement loader**

File: `src/KerbalDialogueKit/Config/DialogueSceneLoader.cs`
```csharp
using System.Collections.Generic;
using KerbalDialogueKit.Core;

namespace KerbalDialogueKit.Config
{
    /// <summary>
    /// Lightweight abstraction over KSP's ConfigNode so loader logic stays
    /// testable without depending on UnityEngine. Real code wraps a
    /// KSP ConfigNode in ConfigNodeAdapter (below).
    /// </summary>
    public interface ISceneNode
    {
        string GetValue(string key);
        IEnumerable<ISceneNode> GetChildren(string name);
        IEnumerable<(string Name, ISceneNode Node)> AllChildren();
    }

    public static class DialogueSceneLoader
    {
        public static DialogueScene Load(ISceneNode node)
        {
            var scene = new DialogueScene(node.GetValue("id"))
                .WithTitle(node.GetValue("title"));

            foreach (var charNode in node.GetChildren("CHARACTER"))
            {
                var ch = new DialogueCharacter
                {
                    Id = charNode.GetValue("id"),
                    Name = charNode.GetValue("name"),
                    PortraitModel = charNode.GetValue("model"),
                    Color = charNode.GetValue("color") ?? "#FFFFFFFF",
                    Animation = charNode.GetValue("animation") ?? "idle"
                };
                if (ch.Id == "A") scene.CharacterA = ch;
                else if (ch.Id == "B") scene.CharacterB = ch;
            }

            foreach (var (name, child) in node.AllChildren())
            {
                if (name == "LINE")
                {
                    var line = new DialogueLine
                    {
                        Speaker = child.GetValue("speaker"),
                        Text = child.GetValue("text") ?? "",
                        Animation = child.GetValue("animation") ?? "idle",
                        VisibleIf = child.GetValue("visibleIf")
                    };
                    scene.Items.Add(line);
                }
                else if (name == "CHOICE")
                {
                    var choice = new DialogueChoice
                    {
                        Id = child.GetValue("id"),
                        Prompt = child.GetValue("prompt")
                    };
                    foreach (var opt in child.GetChildren("OPTION"))
                    {
                        choice.Options.Add(new DialogueChoiceOption
                        {
                            Value = opt.GetValue("value"),
                            Text = opt.GetValue("text"),
                            Description = opt.GetValue("description")
                        });
                    }
                    scene.Items.Add(choice);
                }
            }

            return scene;
        }
    }
}
```

- [ ] **Step 3: Write the KSP ConfigNode adapter**

File: `src/KerbalDialogueKit/Config/ConfigNodeAdapter.cs`
```csharp
using System.Collections.Generic;

namespace KerbalDialogueKit.Config
{
    /// <summary>
    /// Wraps a KSP ConfigNode in the ISceneNode interface so DialogueSceneLoader
    /// can parse it without depending on UnityEngine directly.
    /// </summary>
    internal sealed class ConfigNodeAdapter : ISceneNode
    {
        private readonly ConfigNode node;

        public ConfigNodeAdapter(ConfigNode node) { this.node = node; }

        public string GetValue(string key) => node.HasValue(key) ? node.GetValue(key) : null;

        public IEnumerable<ISceneNode> GetChildren(string name)
        {
            foreach (ConfigNode child in node.GetNodes(name))
                yield return new ConfigNodeAdapter(child);
        }

        public IEnumerable<(string Name, ISceneNode Node)> AllChildren()
        {
            foreach (ConfigNode child in node.nodes)
                yield return (child.name, new ConfigNodeAdapter(child));
        }
    }
}
```

- [ ] **Step 4: Add a scene registry + EnqueueById to DialogueKit**

Modify `src/KerbalDialogueKit/Core/DialogueKit.cs`. Replace the existing file contents with:

```csharp
using System.Collections.Generic;
using KerbalDialogueKit.Flags;
using KerbalDialogueKit.Rendering;

namespace KerbalDialogueKit.Core
{
    public static class DialogueKit
    {
        private static SceneRenderer renderer;
        private static FlagStore flagsRef;
        private static readonly Dictionary<string, DialogueScene> registry = new Dictionary<string, DialogueScene>();

        internal static void Initialize(SceneRenderer sceneRenderer, FlagStore flags)
        {
            renderer = sceneRenderer;
            flagsRef = flags;
        }

        public static FlagStore Flags => flagsRef;

        public static void Enqueue(DialogueScene scene) => renderer?.Enqueue(scene);

        public static void EnqueuePriority(DialogueScene scene) => renderer?.EnqueuePriority(scene);

        public static bool IsPlaying() => renderer != null && renderer.IsPlaying;

        public static int QueueLength() => renderer?.QueueLength ?? 0;

        public static void ClearQueue() => renderer?.ClearQueue();

        // ---- Registry / config scenes ----

        internal static void Register(DialogueScene scene)
        {
            if (string.IsNullOrEmpty(scene?.Id)) return;
            registry[scene.Id] = scene;
        }

        public static bool HasScene(string id) => registry.ContainsKey(id);

        /// <summary>
        /// Enqueue a scene loaded from cfg. Returns false if no scene with that
        /// id was registered. Callbacks is optional; when non-null, its callbacks
        /// REPLACE any set via the fluent builder.
        /// </summary>
        public static bool EnqueueById(string id, SceneCallbacks callbacks = null)
        {
            if (!registry.TryGetValue(id, out var template)) return false;

            // Clone the template so per-invocation callbacks don't mutate the registry copy
            var scene = CloneScene(template);
            if (callbacks != null)
            {
                scene.Callbacks.OnSceneStart = callbacks.OnSceneStart;
                scene.Callbacks.OnLineActivated = callbacks.OnLineActivated;
                scene.Callbacks.OnChoiceMade = callbacks.OnChoiceMade;
                scene.Callbacks.OnSceneEnd = callbacks.OnSceneEnd;
                scene.Callbacks.OnSceneCancelled = callbacks.OnSceneCancelled;
            }
            Enqueue(scene);
            return true;
        }

        private static DialogueScene CloneScene(DialogueScene source)
        {
            var clone = new DialogueScene(source.Id) { Title = source.Title };
            clone.CharacterA = source.CharacterA;
            clone.CharacterB = source.CharacterB;
            foreach (var item in source.Items) clone.Items.Add(item);
            return clone;
        }
    }
}
```

- [ ] **Step 5: Load DIALOGUE_SCENE cfg nodes at addon startup**

Modify `src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs`'s `Awake()`. Add at the end (before `Debug.Log("[KerbalDialogueKit] Initialized v0.1.0");`):

```csharp
            LoadConfigScenes();
```

And add the method to the class:

```csharp
        private void LoadConfigScenes()
        {
            int count = 0;
            foreach (var urlConfig in GameDatabase.Instance.GetConfigs("DIALOGUE_SCENE"))
            {
                try
                {
                    var adapter = new KerbalDialogueKit.Config.ConfigNodeAdapter(urlConfig.config);
                    var scene = KerbalDialogueKit.Config.DialogueSceneLoader.Load(adapter);
                    KerbalDialogueKit.Core.DialogueKit.Register(scene);
                    count++;
                }
                catch (Exception ex)
                {
                    Debug.LogError($"[KerbalDialogueKit] Failed to load DIALOGUE_SCENE '{urlConfig.url}': {ex}");
                }
            }
            Debug.Log($"[KerbalDialogueKit] Loaded {count} scene(s) from cfg.");
        }
```

(Note: the addon file already has `using` for `KerbalDialogueKit.Core`, `KerbalDialogueKit.Flags`, and `KerbalDialogueKit.Rendering`. The fully-qualified names in `LoadConfigScenes` make the intent clear and avoid accidental conflicts.)

- [ ] **Step 6: Add ConfigNodeAdapter visibility**

`ConfigNodeAdapter` is `internal` but is referenced from the addon (same assembly — fine). No changes needed.

- [ ] **Step 7: Example cfg file**

```bash
mkdir -p Examples
```

File: `Examples/demo-scene.cfg`
```cfg
// Example DIALOGUE_SCENE loadable by KerbalDialogueKit.
// Drop a file like this into any GameData mod's folder and the scene will be
// registered under its `id`. Enqueue from code with:
//    KerbalDialogueKit.Core.DialogueKit.EnqueueById("kdk_example_duna");
DIALOGUE_SCENE
{
    id = kdk_example_duna
    title = Going to Duna

    CHARACTER
    {
        id = A
        name = Wernher
        model = Instructor_Wernher
        color = #FF82B4E8
    }

    CHARACTER
    {
        id = B
        name = Gus
        model = Strategy_MechanicGuy
        color = #FFFFC078
    }

    LINE
    {
        speaker = A
        text = The spectrographic data is extraordinary. I want a full science package.
    }

    LINE
    {
        speaker = B
        text = Heavy means expensive. I say we start lean and prove the landing first.
        animation = idle_disappointed
    }

    CHOICE
    {
        id = duna_approach
        OPTION
        {
            value = science_heavy
            text = Back Wernher: Go big on science
            description = Full science package. +40 percent science from Duna.
        }
        OPTION
        {
            value = engineering_first
            text = Back Gus: Prove the engineering
            description = Rover and small lander. -20 percent vehicle costs.
        }
        OPTION
        {
            value = compromise
            text = Compromise: Do both, smaller
            description = Mid-size lander. Modest bonuses to both.
        }
    }

    LINE
    {
        speaker = A
        visibleIf = duna_approach == science_heavy
        text = This is most excellent! Let us begin preparations.
        animation = true_thumbsUp
    }

    LINE
    {
        speaker = B
        visibleIf = duna_approach == engineering_first
        text = Smart call, boss. She will hold together.
    }

    LINE
    {
        speaker = A
        visibleIf = duna_approach == compromise
        text = A reasonable compromise.
    }
}
```

- [ ] **Step 8: Run unit tests**

```bash
dotnet test tests/KerbalDialogueKit.Tests/
```

Expected: all loader tests pass (plus all previous tests still green).

- [ ] **Step 9: Build and verify the cfg is loaded at runtime**

```bash
dotnet build KerbalDialogueKit.sln -c Release
```

Copy the example cfg into the install so KSP's GameDatabase finds it. Since we ship from the same folder, just move it there:

```bash
cp Examples/demo-scene.cfg .
```

(The file needs to be under `GameData/KerbalDialogueKit/` or any other GameData subfolder for GameDatabase to scan it.)

Launch KSP, reach the main menu, quit. Then:
```bash
grep "KerbalDialogueKit" "/c/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/KSP.log"
```

Expected: you see `[KerbalDialogueKit] Loaded 1 scene(s) from cfg.`

- [ ] **Step 10: Update demo to use EnqueueById as well**

Modify `demo/KerbalDialogueKit.Demo/DemoTrigger.cs`. Add a second button that uses `EnqueueById`:

Replace the `OnGUI` method:

```csharp
        public void OnGUI()
        {
            if (GUI.Button(new Rect(20, 20, 180, 28), "KDK Demo Scene (code)"))
            {
                RunDemo();
            }
            if (GUI.Button(new Rect(20, 56, 180, 28), "KDK Demo Scene (cfg)"))
            {
                if (!DialogueKit.EnqueueById("kdk_example_duna",
                    new SceneCallbacks { OnChoiceMade = (id, v) => Debug.Log($"[KDK.Demo.cfg] {id}={v}") }))
                {
                    Debug.LogWarning("[KDK.Demo] Scene id not registered.");
                }
            }
        }
```

Rebuild and launch KSP. Expected: both buttons work. The cfg version fires the Duna scene from the example file.

- [ ] **Step 11: Commit**

```bash
git add src/KerbalDialogueKit/Config/ src/KerbalDialogueKit/Core/DialogueKit.cs src/KerbalDialogueKit/Runtime/KerbalDialogueKitAddon.cs tests/KerbalDialogueKit.Tests/DialogueSceneLoaderTests.cs Examples/demo-scene.cfg demo-scene.cfg demo/KerbalDialogueKit.Demo/DemoTrigger.cs
git commit -m "feat: load scenes from DIALOGUE_SCENE cfg nodes

Adds DialogueSceneLoader + ISceneNode/ConfigNodeAdapter split so the
loader is testable without UnityEngine. Scenes loaded at addon startup
and registered by id; enqueue them with DialogueKit.EnqueueById."
```

---

## Task 20: README and API Documentation

Ship-ready documentation so consumers know how to use the library.

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Write the full README**

File: `README.md`
```markdown
# KerbalDialogueKit

A utility library that extends [ChatterBox](https://github.com/DymndLab/ChatterBox)
with programmatically-triggered dialogue scenes, branching, mid-scene choice
capture, and lifecycle callbacks. Built for mods that want character-driven
interactive dialogue in KSP 1.12.x.

## Status

Early release. v0.1.0.

## Features

- Trigger dialogue scenes from code, not just from ContractConfigurator state
- Branch dialogue by flag — line variants based on `visibleIf` expressions
- Pause mid-scene and present 2-4 option cards; capture the player's choice
- Receive lifecycle callbacks (scene start/end, line activated, choice made)
- Queue multiple scenes; manage them via a simple static API
- Load scenes from `.cfg` files OR build them fluently in C#
- Per-save flag persistence so choices stick across saves/loads

## Requirements

- KSP 1.12.x
- [ChatterBox](https://github.com/DymndLab/ChatterBox) v1.0.0 or later
- [ContractConfigurator](https://forum.kerbalspaceprogram.com/index.php?/topic/91625-)

## Installation

Download the latest release from (link TBD). Extract so the folder structure is:

```
GameData/
  KerbalDialogueKit/
    KerbalDialogueKit.version
    Plugins/
      KerbalDialogueKit.dll
```

CKAN support is planned.

## Usage — Code

```csharp
using KerbalDialogueKit.Core;

var wernher = new DialogueCharacter
{
    Name = "Wernher",
    PortraitModel = "Instructor_Wernher",
    Color = "#FF82B4E8"
};
var gus = new DialogueCharacter
{
    Name = "Gus",
    PortraitModel = "Strategy_MechanicGuy",
    Color = "#FFFFC078"
};

var scene = new DialogueScene("my_directive")
    .WithTitle("The Approach Debate")
    .WithCharacter("A", wernher)
    .WithCharacter("B", gus)
    .AddLine("A", "The data is extraordinary.")
    .AddLine("B", "Heavy means expensive.")
    .AddChoice("approach",
        new DialogueChoiceOption { Value = "science", Text = "Back Wernher: Go big" },
        new DialogueChoiceOption { Value = "engineering", Text = "Back Gus: Start lean" })
    .AddLine("A", "This is most excellent!",
        visibleIf: "approach == science")
    .AddLine("B", "Smart call.",
        visibleIf: "approach == engineering")
    .OnChoiceMade((id, value) => Debug.Log($"{id}={value}"))
    .OnSceneEnd(s => Debug.Log("Done"));

DialogueKit.Enqueue(scene);
```

## Usage — Config

Define a scene in any `.cfg` file inside `GameData`:

```cfg
DIALOGUE_SCENE
{
    id = my_directive
    title = The Approach Debate

    CHARACTER
    {
        id = A
        name = Wernher
        model = Instructor_Wernher
        color = #FF82B4E8
    }
    CHARACTER
    {
        id = B
        name = Gus
        model = Strategy_MechanicGuy
        color = #FFFFC078
    }

    LINE { speaker = A; text = The data is extraordinary. }
    LINE { speaker = B; text = Heavy means expensive. }

    CHOICE
    {
        id = approach
        OPTION { value = science; text = Back Wernher: Go big }
        OPTION { value = engineering; text = Back Gus: Start lean }
    }

    LINE
    {
        speaker = A
        visibleIf = approach == science
        text = This is most excellent!
    }
    LINE
    {
        speaker = B
        visibleIf = approach == engineering
        text = Smart call.
    }
}
```

Then enqueue by id:

```csharp
DialogueKit.EnqueueById("my_directive",
    new SceneCallbacks { OnChoiceMade = (id, value) => Debug.Log($"{id}={value}") });
```

## visibleIf Expressions

Supported operators:
- `==`, `!=`, `>`, `<`, `>=`, `<=`
- `in (value1, value2)`
- `&&`, `||` with `&&` binding tighter

Examples:
- `mood == Enthusiastic`
- `score > 5 && completedMun == true`
- `mood in (Enthusiastic, Supportive)`

Missing flags are treated as empty string.

## Flags

Flags persist per-save via `FlagScenario`. Set them yourself or via choice
selection:

```csharp
DialogueKit.Flags.Set("wernherDisposition", "Enthusiastic");
var mood = DialogueKit.Flags.Get("wernherDisposition");
```

## Music Integration

When a scene fires a music cue (future feature), KerbalDialogueKit publishes
a named GameEvent so music mods can subscribe. Event name:
`KerbalDialogueKit.MusicCue`.

## License

MIT — see LICENSE.

## Credits

Built on top of ChatterBox by DymndLab. Portrait rendering, dialogue
sequencing, and audio handling remain ChatterBox's work — we extend it
with the interactive layer.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: full README with usage examples"
```

---

## Task 21: Create docs/UPSTREAM.md Tracking the PR

**Files:**
- Create: `docs/UPSTREAM.md`

- [ ] **Step 1: Record PR status**

```bash
mkdir -p docs
```

File: `docs/UPSTREAM.md`
```markdown
# Upstream Dependencies

## ChatterBox

- Repo: https://github.com/DymndLab/ChatterBox
- Depend on: v1.0.0+

### PR: Expose scene rendering API

- URL: (fill in from Task 1)
- Status: pending / merged / rejected

### If merged

Once the PR is merged in a released ChatterBox version we depend on, the
reflection bridge in `src/KerbalDialogueKit/Rendering/ChatterBoxBridge.cs`
can be replaced with direct type references. The public API of
ChatterBoxBridge (BuildScenePopupDefinition, Enqueue) stays identical —
only the implementation changes.

Steps:
1. Update `src/KerbalDialogueKit/KerbalDialogueKit.csproj` ChatterBox
   reference to the minimum version that contains the public API
2. Rewrite `ChatterBoxBridge.cs` to use `new ChatterBox.ScenePopupDefinition()`
   and so on instead of reflection
3. Delete the `SetField` helper and field-name strings
4. Run `dotnet test` (all green) and manual in-game verification

### If rejected

Reflection stays. Long-term, if the library proves worth the investment,
fork ChatterBox's rendering code (MIT license) into a standalone rendering
assembly that lives in KerbalDialogueKit. This removes the ChatterBox
dependency entirely.
```

- [ ] **Step 2: Commit**

```bash
git add docs/UPSTREAM.md
git commit -m "docs: track upstream ChatterBox PR status"
```

---

## Task 22: Final Verification and v0.1.0 Tag

- [ ] **Step 1: Clean build from scratch**

```bash
cd "C:/Program Files (x86)/Steam/steamapps/common/Kerbal Space Program/GameData/KerbalDialogueKit"
find . -name bin -type d -exec rm -rf {} + 2>/dev/null; find . -name obj -type d -exec rm -rf {} + 2>/dev/null; rm -f Plugins/*.dll
dotnet build KerbalDialogueKit.sln -c Release
```

Expected: `Build succeeded. 0 Error(s)`. `Plugins/KerbalDialogueKit.dll` and `Plugins/KerbalDialogueKit.Demo.dll` present.

- [ ] **Step 2: Full test run**

```bash
dotnet test tests/KerbalDialogueKit.Tests/ -c Release
```

Expected: all tests pass.

- [ ] **Step 3: In-game smoke test**

Launch KSP. Load a career save. In Space Center:
1. Click "KDK Demo Scene (code)" — verify dialogue, choices, branches work end-to-end
2. Click "KDK Demo Scene (cfg)" — verify the cfg-loaded scene works
3. Quit and reload the save — verify flags (choices) persist via FlagScenario

Check KSP.log for any `[KerbalDialogueKit]` warnings or errors; iterate if needed.

- [ ] **Step 4: Tag the release**

```bash
git tag v0.1.0
git log --oneline -10
```

(Actual push of the tag to a remote is deferred until a GitHub repo exists for KerbalDialogueKit.)

- [ ] **Step 5: Announce next steps**

KerbalDialogueKit v0.1.0 is shippable. Next step: BadgKatDirector plan (separate plan, references this library as a dependency).

---

## Summary

**Build artifact:** `GameData/KerbalDialogueKit/Plugins/KerbalDialogueKit.dll`

**Consumers import:**
```csharp
using KerbalDialogueKit.Core;  // DialogueKit, DialogueScene, DialogueCharacter, DialogueLine, DialogueChoice, DialogueChoiceOption, SceneCallbacks
using KerbalDialogueKit.Flags; // FlagStore (if directly poking flags)
```

**Cfg entry points:** `DIALOGUE_SCENE { ... }` nodes in any GameData cfg file, loaded at startup.

**Test coverage:**
- Flag storage (FlagStoreTests)
- Expression tokenization (FlagExpressionTokenizerTests)
- Expression parsing (FlagExpressionParserTests)
- Expression evaluation (FlagExpressionEvaluationTests)
- Scene builder (DialogueSceneBuilderTests)
- Line visibility filter (LineEvaluatorTests)
- Cfg loader (DialogueSceneLoaderTests)

**Manual verification:** demo mod with code-built and cfg-loaded scenes exercised in-game.

**Upstream:** PR to ChatterBox (Task 1, parallel). Once merged, refactor ChatterBoxBridge.cs (documented in docs/UPSTREAM.md).
