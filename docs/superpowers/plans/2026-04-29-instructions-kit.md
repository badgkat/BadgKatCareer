# Kerbal Instructions Kit Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `KerbalInstructionsKit` v0.1.0 — a content-driven framework plugin that lets campaign packs define multi-page lessons (text + image + cross-links) and surface them via a toolbar button in every scene plus an injected button on the contract details view. Inert without cfg content.

**Architecture:** Standalone .NET Framework 4.7.2 plugin DLL at `GameData/KerbalInstructionsKit/`, mirroring the KAK / KCK / KDK pattern. Pure-logic layers (cfg loaders, registry, expression evaluator, trigger engine, panel state machine) tested with xUnit; KSP-integrated layers (per-scene `KSPAddon`s, `ScenarioModule`, IMGUI panel, Harmony patch on contract details) verified manually in-game. Soft dependency on ContractConfigurator detected via reflection.

**Tech Stack:** C# (.NET Framework 4.7.2 / net472, KSP 1.12.x). xUnit for unit tests. ToolbarController + ClickThroughBlocker + Harmony for KSP integration. Reflection-detected ContractConfigurator.

**Repo:** New git repo at `GameData/KerbalInstructionsKit/` (created in Task 1).

**Spec:** `GameData/BadgKatCareer/docs/superpowers/specs/2026-04-29-instructions-kit-design.md`

---

## Phase 0 — Repo and project scaffolding

### Task 1: Initialize KerbalInstructionsKit repository

**Files:**
- Create: `GameData/KerbalInstructionsKit/.gitignore`
- Create: `GameData/KerbalInstructionsKit/README.md`
- Create: `GameData/KerbalInstructionsKit/LICENSE`
- Create: `GameData/KerbalInstructionsKit/src/KerbalInstructionsKit/KerbalInstructionsKit.csproj`
- Create: `GameData/KerbalInstructionsKit/src/KerbalInstructionsKit/Properties/AssemblyInfo.cs`

- [ ] **Step 1: Create the directory tree**

```bash
cd "GameData"
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Properties
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Core
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Config
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Triggers
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Runtime
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Rendering
mkdir -p KerbalInstructionsKit/src/KerbalInstructionsKit/Util
mkdir -p KerbalInstructionsKit/Plugins/Icons
mkdir -p KerbalInstructionsKit/dev/test-pack
mkdir -p KerbalInstructionsKit/docs
```

- [ ] **Step 2: Write `.gitignore`**

```gitignore
bin/
obj/
*.suo
*.user
*.userprefs
.vs/
.idea/
*.swp
*.tmp
.superpowers/
```

Note: `Plugins/` is NOT gitignored because the built DLL is checked in (matching KAK/KCK/KDK pattern — KSP loads from there).

- [ ] **Step 3: Write `README.md` (skeleton)**

```markdown
# Kerbal Instructions Kit (KIK)

Content-driven instructional panel framework for KSP 1.12.x. Inert without cfg content.

See [design spec](https://github.com/badgkat/BadgKatCareer/blob/master/docs/superpowers/specs/2026-04-29-instructions-kit-design.md) for the authoring grammar and architecture.

## Status

v0.1.0 — initial release.

## Dependencies

- ToolbarController
- ClickThroughBlocker
- Harmony (000_Harmony)
- ContractConfigurator (soft / optional)

## Authoring

See `dev/test-pack/` for example cfg.

## License

MIT — see LICENSE.
```

- [ ] **Step 4: Write `LICENSE` (MIT, matching KAK)**

Copy the MIT license text from `GameData/KerbalAdminKit/LICENSE`. Replace the copyright line with `Copyright (c) 2026 badgkat`.

- [ ] **Step 5: Write `src/KerbalInstructionsKit/Properties/AssemblyInfo.cs`**

```csharp
using System.Reflection;
using System.Runtime.InteropServices;

[assembly: AssemblyTitle("KerbalInstructionsKit")]
[assembly: AssemblyDescription("Content-driven instructional panel framework for KSP")]
[assembly: AssemblyCompany("badgkat")]
[assembly: AssemblyProduct("KerbalInstructionsKit")]
[assembly: AssemblyCopyright("Copyright (c) 2026 badgkat")]
[assembly: ComVisible(false)]
[assembly: AssemblyVersion("0.1.0.0")]
[assembly: AssemblyFileVersion("0.1.0.0")]
[assembly: KSPAssembly("KerbalInstructionsKit", 0, 1)]
```

- [ ] **Step 6: Write `src/KerbalInstructionsKit/KerbalInstructionsKit.csproj`**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net472</TargetFramework>
    <LangVersion>8.0</LangVersion>
    <RootNamespace>KerbalInstructionsKit</RootNamespace>
    <AssemblyName>KerbalInstructionsKit</AssemblyName>
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
    <Reference Include="0Harmony">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\000_Harmony\0Harmony.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="ClickThroughBlocker">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\000_ClickThroughBlocker\Plugins\ClickThroughBlocker.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="ToolbarControl">
      <HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\001_ToolbarControl\Plugins\ToolbarControl.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>
</Project>
```

Note: ContractConfigurator is intentionally NOT referenced — KIK probes for it via reflection at runtime.

- [ ] **Step 7: Build to verify the project compiles (empty)**

```bash
cd "GameData/KerbalInstructionsKit"
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: build succeeds, produces empty `Plugins/KerbalInstructionsKit.dll`.

- [ ] **Step 8: Initialize git repo and make first commit**

```bash
cd "GameData/KerbalInstructionsKit"
git init
git branch -m main
git add .gitignore README.md LICENSE src/
git commit -m "chore: scaffold KerbalInstructionsKit project"
```

Note: `Plugins/KerbalInstructionsKit.dll` is built but committed in a later task once it has actual content.

### Task 2: Set up xUnit test project

**Files:**
- Create: `GameData/KerbalInstructionsKit/tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj`
- Create: `GameData/KerbalInstructionsKit/tests/KerbalInstructionsKit.Tests/SmokeTest.cs`

- [ ] **Step 1: Create the test project directory**

```bash
cd "GameData/KerbalInstructionsKit"
mkdir -p tests/KerbalInstructionsKit.Tests/TestHelpers
```

- [ ] **Step 2: Write `tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj`**

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
    <ProjectReference Include="..\..\src\KerbalInstructionsKit\KerbalInstructionsKit.csproj">
      <Private>true</Private>
    </ProjectReference>
  </ItemGroup>
</Project>
```

- [ ] **Step 3: Write a smoke test**

`tests/KerbalInstructionsKit.Tests/SmokeTest.cs`:

```csharp
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class SmokeTest
    {
        [Fact]
        public void TestProjectBuildsAndRuns()
        {
            Assert.Equal(2, 1 + 1);
        }
    }
}
```

- [ ] **Step 4: Run the test**

```bash
cd "GameData/KerbalInstructionsKit"
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

Expected: 1 test passes.

- [ ] **Step 5: Commit**

```bash
cd "GameData/KerbalInstructionsKit"
git add tests/
git commit -m "test: add xUnit test project with smoke test"
```

---

## Phase 1 — Pure-logic foundation (TDD)

### Task 3: ISceneNode abstraction and FakeSceneNode test helper

KIK needs its own ConfigNode abstraction so the loaders can be unit-tested without KSP loaded. Mirror the KCK pattern but keep it independent (KIK doesn't depend on KCK).

**Files:**
- Create: `src/KerbalInstructionsKit/Util/ISceneNode.cs`
- Create: `src/KerbalInstructionsKit/Util/ConfigNodeAdapter.cs`
- Create: `tests/KerbalInstructionsKit.Tests/TestHelpers/FakeSceneNode.cs`
- Create: `tests/KerbalInstructionsKit.Tests/TestHelpers/FakeSceneNodeTests.cs`

- [ ] **Step 1: Write the failing tests for FakeSceneNode**

`tests/KerbalInstructionsKit.Tests/TestHelpers/FakeSceneNodeTests.cs`:

```csharp
using System.Linq;
using Xunit;

namespace KerbalInstructionsKit.Tests.TestHelpers
{
    public class FakeSceneNodeTests
    {
        [Fact]
        public void Set_StoresSingleValue()
        {
            var n = new FakeSceneNode().Set("k", "v");
            Assert.Equal("v", n.GetValue("k"));
            Assert.True(n.HasValue("k"));
        }

        [Fact]
        public void Add_AppendsMultipleValues()
        {
            var n = new FakeSceneNode().Add("k", "a").Add("k", "b");
            Assert.Equal(new[] { "a", "b" }, n.GetValues("k").ToArray());
        }

        [Fact]
        public void AddChild_StoresNamedChildren()
        {
            var parent = new FakeSceneNode();
            var child = new FakeSceneNode().Set("id", "X");
            parent.AddChild("CHILD", child);
            Assert.True(parent.HasNode("CHILD"));
            Assert.Single(parent.GetNodes("CHILD"));
            Assert.Equal("X", parent.GetNodes("CHILD").First().GetValue("id"));
        }

        [Fact]
        public void GetValue_MissingKey_ReturnsNull()
        {
            Assert.Null(new FakeSceneNode().GetValue("missing"));
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail (compile error: types don't exist)**

```bash
cd "GameData/KerbalInstructionsKit"
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

Expected: build error referencing `FakeSceneNode`.

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Util/ISceneNode.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Util
{
    public interface ISceneNode
    {
        string GetValue(string key);
        IEnumerable<string> GetValues(string key);
        IEnumerable<ISceneNode> GetNodes(string name);
        bool HasValue(string key);
        bool HasNode(string name);
    }
}
```

- [ ] **Step 4: Write `src/KerbalInstructionsKit/Util/ConfigNodeAdapter.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Util
{
    public sealed class ConfigNodeAdapter : ISceneNode
    {
        private readonly ConfigNode node;
        public ConfigNodeAdapter(ConfigNode node) { this.node = node; }
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

- [ ] **Step 5: Write `tests/KerbalInstructionsKit.Tests/TestHelpers/FakeSceneNode.cs`**

```csharp
using System.Collections.Generic;
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Tests.TestHelpers
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

- [ ] **Step 6: Run tests, expect 4 pass**

```bash
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

- [ ] **Step 7: Commit**

```bash
git add src/KerbalInstructionsKit/Util/ tests/KerbalInstructionsKit.Tests/TestHelpers/
git commit -m "feat: add ISceneNode abstraction + FakeSceneNode test helper"
```

### Task 4: Lesson, Page, Link data model

**Files:**
- Create: `src/KerbalInstructionsKit/Core/Lesson.cs`
- Create: `src/KerbalInstructionsKit/Core/Page.cs`
- Create: `src/KerbalInstructionsKit/Core/Link.cs`
- Create: `src/KerbalInstructionsKit/Core/LinkType.cs`

- [ ] **Step 1: Write `Core/LinkType.cs`**

```csharp
namespace KerbalInstructionsKit.Core
{
    public enum LinkType { Lesson, Kspedia, Url }
}
```

- [ ] **Step 2: Write `Core/Link.cs`**

```csharp
namespace KerbalInstructionsKit.Core
{
    public sealed class Link
    {
        public LinkType Type;
        public string Target;
        public string Label;
    }
}
```

- [ ] **Step 3: Write `Core/Page.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Core
{
    public sealed class Page
    {
        public string Title;     // optional
        public string Text;      // may contain Unity rich-text tags
        public string Image;     // GameDatabase URL, optional
        public string Caption;   // optional
        public List<Link> Links = new List<Link>();
    }
}
```

- [ ] **Step 4: Write `Core/Lesson.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Core
{
    public sealed class Lesson
    {
        public string Id;
        public string Title;
        public string Category;       // null/empty → "Misc"
        public int SortOrder = 1000;
        public List<string> Tags = new List<string>();
        public string VisibleIfRaw;   // raw expression text, compiled lazily
        public List<Page> Pages = new List<Page>();
    }
}
```

- [ ] **Step 5: Build to verify the data classes compile**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: success.

- [ ] **Step 6: Commit**

```bash
git add src/KerbalInstructionsKit/Core/
git commit -m "feat: add Lesson/Page/Link data model"
```

### Task 5: LinkLoader

**Files:**
- Create: `src/KerbalInstructionsKit/Config/LinkLoader.cs`
- Create: `tests/KerbalInstructionsKit.Tests/LinkLoaderTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/LinkLoaderTests.cs`:

```csharp
using KerbalInstructionsKit.Config;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Tests.TestHelpers;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class LinkLoaderTests
    {
        [Fact]
        public void Load_ParsesLessonLink()
        {
            var n = new FakeSceneNode().Set("type", "lesson").Set("target", "LSN_X").Set("label", "Go to X");
            var link = LinkLoader.Load(n);
            Assert.NotNull(link);
            Assert.Equal(LinkType.Lesson, link.Type);
            Assert.Equal("LSN_X", link.Target);
            Assert.Equal("Go to X", link.Label);
        }

        [Fact]
        public void Load_ParsesKspediaLink()
        {
            var n = new FakeSceneNode().Set("type", "kspedia").Set("target", "ManeuverNodes").Set("label", "KSPedia");
            var link = LinkLoader.Load(n);
            Assert.Equal(LinkType.Kspedia, link.Type);
        }

        [Fact]
        public void Load_ParsesUrlLink_Http()
        {
            var n = new FakeSceneNode().Set("type", "url").Set("target", "https://example.com").Set("label", "Wiki");
            var link = LinkLoader.Load(n);
            Assert.Equal(LinkType.Url, link.Type);
            Assert.Equal("https://example.com", link.Target);
        }

        [Fact]
        public void Load_RejectsUrlLink_NonHttp()
        {
            var n = new FakeSceneNode().Set("type", "url").Set("target", "file:///etc/passwd").Set("label", "X");
            Assert.Null(LinkLoader.Load(n));
        }

        [Fact]
        public void Load_RejectsLink_MissingTarget()
        {
            var n = new FakeSceneNode().Set("type", "lesson").Set("label", "X");
            Assert.Null(LinkLoader.Load(n));
        }

        [Fact]
        public void Load_RejectsLink_UnknownType()
        {
            var n = new FakeSceneNode().Set("type", "weird").Set("target", "X").Set("label", "X");
            Assert.Null(LinkLoader.Load(n));
        }

        [Fact]
        public void Load_RejectsLink_MissingType()
        {
            var n = new FakeSceneNode().Set("target", "X").Set("label", "X");
            Assert.Null(LinkLoader.Load(n));
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

```bash
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

Expected: build error referencing `LinkLoader`.

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Config/LinkLoader.cs`**

```csharp
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Config
{
    public static class LinkLoader
    {
        public static Link Load(ISceneNode node)
        {
            if (node == null) return null;
            var typeStr = node.GetValue("type");
            var target = node.GetValue("target");
            if (string.IsNullOrEmpty(typeStr))
            {
                Debug.LogWarning("[KIK] LINK missing 'type', skipping");
                return null;
            }
            if (string.IsNullOrEmpty(target))
            {
                Debug.LogWarning("[KIK] LINK missing 'target', skipping");
                return null;
            }

            LinkType type;
            switch (typeStr.ToLowerInvariant())
            {
                case "lesson":  type = LinkType.Lesson; break;
                case "kspedia": type = LinkType.Kspedia; break;
                case "url":     type = LinkType.Url; break;
                default:
                    Debug.LogWarning($"[KIK] LINK has unknown type '{typeStr}', skipping");
                    return null;
            }

            if (type == LinkType.Url &&
                !target.StartsWith("http://") && !target.StartsWith("https://"))
            {
                Debug.LogWarning($"[KIK] LINK url target '{target}' is not http(s), skipping");
                return null;
            }

            return new Link
            {
                Type = type,
                Target = target,
                Label = node.GetValue("label") ?? target,
            };
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 7 pass**

```bash
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Config/LinkLoader.cs tests/KerbalInstructionsKit.Tests/LinkLoaderTests.cs
git commit -m "feat: add LinkLoader with type/target validation"
```

### Task 6: PageLoader

**Files:**
- Create: `src/KerbalInstructionsKit/Config/PageLoader.cs`
- Create: `tests/KerbalInstructionsKit.Tests/PageLoaderTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/PageLoaderTests.cs`:

```csharp
using KerbalInstructionsKit.Config;
using KerbalInstructionsKit.Tests.TestHelpers;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class PageLoaderTests
    {
        [Fact]
        public void Load_ParsesAllFields()
        {
            var n = new FakeSceneNode()
                .Set("title", "Page A")
                .Set("text", "hello world")
                .Set("image", "MyMod/Img/foo.png")
                .Set("caption", "img caption");

            var p = PageLoader.Load(n);
            Assert.Equal("Page A", p.Title);
            Assert.Equal("hello world", p.Text);
            Assert.Equal("MyMod/Img/foo.png", p.Image);
            Assert.Equal("img caption", p.Caption);
            Assert.Empty(p.Links);
        }

        [Fact]
        public void Load_TextOnly_IsValid()
        {
            var n = new FakeSceneNode().Set("text", "hi");
            var p = PageLoader.Load(n);
            Assert.NotNull(p);
            Assert.Equal("hi", p.Text);
            Assert.Null(p.Image);
        }

        [Fact]
        public void Load_ImageOnly_IsValid()
        {
            var n = new FakeSceneNode().Set("image", "X/Y/z.png");
            var p = PageLoader.Load(n);
            Assert.NotNull(p);
            Assert.Equal("X/Y/z.png", p.Image);
        }

        [Fact]
        public void Load_NoTextNoImage_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("caption", "orphan caption");
            Assert.Null(PageLoader.Load(n));
        }

        [Fact]
        public void Load_ParsesLinks()
        {
            var n = new FakeSceneNode().Set("text", "hi");
            n.AddChild("LINK", new FakeSceneNode()
                .Set("type", "lesson").Set("target", "LSN_X").Set("label", "X"));
            n.AddChild("LINK", new FakeSceneNode()
                .Set("type", "kspedia").Set("target", "Foo").Set("label", "KSPedia: Foo"));

            var p = PageLoader.Load(n);
            Assert.Equal(2, p.Links.Count);
        }

        [Fact]
        public void Load_SkipsInvalidLinks()
        {
            var n = new FakeSceneNode().Set("text", "hi");
            n.AddChild("LINK", new FakeSceneNode().Set("type", "lesson"));   // missing target
            n.AddChild("LINK", new FakeSceneNode()
                .Set("type", "url").Set("target", "https://example.com").Set("label", "OK"));

            var p = PageLoader.Load(n);
            Assert.Single(p.Links);
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Config/PageLoader.cs`**

```csharp
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Config
{
    public static class PageLoader
    {
        public static Page Load(ISceneNode node)
        {
            if (node == null) return null;
            var text = node.GetValue("text");
            var image = node.GetValue("image");
            if (string.IsNullOrEmpty(text) && string.IsNullOrEmpty(image))
            {
                Debug.LogWarning("[KIK] PAGE has neither text nor image, skipping");
                return null;
            }

            var page = new Page
            {
                Title = node.GetValue("title"),
                Text = text,
                Image = image,
                Caption = node.GetValue("caption"),
            };

            foreach (var linkNode in node.GetNodes("LINK"))
            {
                var link = LinkLoader.Load(linkNode);
                if (link != null) page.Links.Add(link);
            }

            return page;
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 6 pass**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Config/PageLoader.cs tests/KerbalInstructionsKit.Tests/PageLoaderTests.cs
git commit -m "feat: add PageLoader with text/image/link parsing"
```

### Task 7: LessonLoader

**Files:**
- Create: `src/KerbalInstructionsKit/Config/LessonLoader.cs`
- Create: `tests/KerbalInstructionsKit.Tests/LessonLoaderTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/LessonLoaderTests.cs`:

```csharp
using System.Linq;
using KerbalInstructionsKit.Config;
using KerbalInstructionsKit.Tests.TestHelpers;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class LessonLoaderTests
    {
        private static FakeSceneNode OnePage(string text = "hi") =>
            new FakeSceneNode().Set("text", text);

        [Fact]
        public void Load_ParsesCoreFields()
        {
            var n = new FakeSceneNode()
                .Set("id", "LSN_X")
                .Set("title", "Lesson X")
                .Set("category", "Flight Basics")
                .Set("sortOrder", "200")
                .Set("tags", "Beginner, Orbits")
                .Set("visibleIf", "!LSN_Y.unlocked");
            n.AddChild("PAGE", OnePage());

            var l = LessonLoader.Load(n);
            Assert.Equal("LSN_X", l.Id);
            Assert.Equal("Lesson X", l.Title);
            Assert.Equal("Flight Basics", l.Category);
            Assert.Equal(200, l.SortOrder);
            Assert.Equal(new[] { "Beginner", "Orbits" }, l.Tags.ToArray());
            Assert.Equal("!LSN_Y.unlocked", l.VisibleIfRaw);
            Assert.Single(l.Pages);
        }

        [Fact]
        public void Load_DefaultSortOrder_1000()
        {
            var n = new FakeSceneNode().Set("id", "LSN_X").Set("title", "X");
            n.AddChild("PAGE", OnePage());
            var l = LessonLoader.Load(n);
            Assert.Equal(1000, l.SortOrder);
        }

        [Fact]
        public void Load_NoTags_EmptyList()
        {
            var n = new FakeSceneNode().Set("id", "LSN_X").Set("title", "X");
            n.AddChild("PAGE", OnePage());
            var l = LessonLoader.Load(n);
            Assert.Empty(l.Tags);
        }

        [Fact]
        public void Load_NoId_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("title", "X");
            n.AddChild("PAGE", OnePage());
            Assert.Null(LessonLoader.Load(n));
        }

        [Fact]
        public void Load_NoPages_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("id", "LSN_X").Set("title", "X");
            Assert.Null(LessonLoader.Load(n));
        }

        [Fact]
        public void Load_AllPagesInvalid_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("id", "LSN_X").Set("title", "X");
            n.AddChild("PAGE", new FakeSceneNode().Set("caption", "no body"));
            Assert.Null(LessonLoader.Load(n));
        }

        [Fact]
        public void Load_TagsStripsWhitespace()
        {
            var n = new FakeSceneNode()
                .Set("id", "LSN_X").Set("title", "X")
                .Set("tags", "  A  ,B,  C  ");
            n.AddChild("PAGE", OnePage());
            var l = LessonLoader.Load(n);
            Assert.Equal(new[] { "A", "B", "C" }, l.Tags.ToArray());
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Config/LessonLoader.cs`**

```csharp
using System.Globalization;
using System.Linq;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Config
{
    public static class LessonLoader
    {
        public static Lesson Load(ISceneNode node)
        {
            if (node == null) return null;
            var id = node.GetValue("id");
            if (string.IsNullOrEmpty(id))
            {
                Debug.LogError("[KIK] INSTRUCTION_LESSON missing 'id', skipping");
                return null;
            }

            var lesson = new Lesson
            {
                Id = id,
                Title = node.GetValue("title") ?? id,
                Category = node.GetValue("category"),
                VisibleIfRaw = node.GetValue("visibleIf"),
            };

            if (node.HasValue("sortOrder") &&
                int.TryParse(node.GetValue("sortOrder"),
                    NumberStyles.Integer, CultureInfo.InvariantCulture, out var so))
            {
                lesson.SortOrder = so;
            }

            var tagsRaw = node.GetValue("tags");
            if (!string.IsNullOrEmpty(tagsRaw))
            {
                lesson.Tags = tagsRaw.Split(',')
                    .Select(t => t.Trim())
                    .Where(t => t.Length > 0)
                    .ToList();
            }

            foreach (var pageNode in node.GetNodes("PAGE"))
            {
                var page = PageLoader.Load(pageNode);
                if (page != null) lesson.Pages.Add(page);
            }

            if (lesson.Pages.Count == 0)
            {
                Debug.LogError($"[KIK] INSTRUCTION_LESSON '{id}' has no valid PAGE blocks, skipping");
                return null;
            }

            return lesson;
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 7 pass**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Config/LessonLoader.cs tests/KerbalInstructionsKit.Tests/LessonLoaderTests.cs
git commit -m "feat: add LessonLoader for INSTRUCTION_LESSON parsing"
```

### Task 8: KIK expression AST and parser

The `visibleIf` evaluator. Spec grammar:

```
expr      := orExpr
orExpr    := andExpr ('||' andExpr)*
andExpr   := unary ('&&' unary)*
unary     := '!' unary | atom
atom      := '(' expr ')' | lessonRef | flagRef | bool
lessonRef := IDENT '.unlocked'
flagRef   := 'flag(' IDENT ')'
bool      := 'true' | 'false'
```

**Files:**
- Create: `src/KerbalInstructionsKit/Util/Expression/IExpression.cs`
- Create: `src/KerbalInstructionsKit/Util/Expression/IExpressionContext.cs`
- Create: `src/KerbalInstructionsKit/Util/Expression/ExpressionNodes.cs`
- Create: `src/KerbalInstructionsKit/Util/Expression/ExpressionParser.cs`
- Create: `tests/KerbalInstructionsKit.Tests/ExpressionParserTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/ExpressionParserTests.cs`:

```csharp
using System.Collections.Generic;
using KerbalInstructionsKit.Util.Expression;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class ExpressionParserTests
    {
        private sealed class FakeCtx : IExpressionContext
        {
            public HashSet<string> UnlockedLessons = new HashSet<string>();
            public Dictionary<string, bool> Flags = new Dictionary<string, bool>();
            public bool IsLessonUnlocked(string id) => UnlockedLessons.Contains(id);
            public bool GetFlag(string name) => Flags.TryGetValue(name, out var v) && v;
        }

        [Fact]
        public void Parse_BoolLiteral_True()
        {
            var e = ExpressionParser.Parse("true");
            Assert.True(e.Evaluate(new FakeCtx()));
        }

        [Fact]
        public void Parse_BoolLiteral_False()
        {
            var e = ExpressionParser.Parse("false");
            Assert.False(e.Evaluate(new FakeCtx()));
        }

        [Fact]
        public void Parse_LessonUnlocked_True()
        {
            var ctx = new FakeCtx(); ctx.UnlockedLessons.Add("LSN_X");
            var e = ExpressionParser.Parse("LSN_X.unlocked");
            Assert.True(e.Evaluate(ctx));
        }

        [Fact]
        public void Parse_LessonUnlocked_False()
        {
            var e = ExpressionParser.Parse("LSN_X.unlocked");
            Assert.False(e.Evaluate(new FakeCtx()));
        }

        [Fact]
        public void Parse_Negation()
        {
            var e = ExpressionParser.Parse("!LSN_X.unlocked");
            Assert.True(e.Evaluate(new FakeCtx()));
        }

        [Fact]
        public void Parse_FlagRef()
        {
            var ctx = new FakeCtx(); ctx.Flags["F"] = true;
            var e = ExpressionParser.Parse("flag(F)");
            Assert.True(e.Evaluate(ctx));
        }

        [Fact]
        public void Parse_AndOr_Precedence()
        {
            var ctx = new FakeCtx(); ctx.UnlockedLessons.Add("A");
            // A.unlocked && B.unlocked || true   →   (A&&B) || true   →   true
            var e = ExpressionParser.Parse("A.unlocked && B.unlocked || true");
            Assert.True(e.Evaluate(ctx));
        }

        [Fact]
        public void Parse_Parens()
        {
            var e = ExpressionParser.Parse("(true || false) && false");
            Assert.False(e.Evaluate(new FakeCtx()));
        }

        [Fact]
        public void Parse_ComplexExpression()
        {
            var ctx = new FakeCtx();
            ctx.UnlockedLessons.Add("LSN_Rocket");
            ctx.Flags["BeginnerMode"] = true;
            var e = ExpressionParser.Parse("LSN_Rocket.unlocked && !flag(SkipRocketLessons)");
            Assert.True(e.Evaluate(ctx));
        }

        [Fact]
        public void Parse_GarbageReturnsNull()
        {
            Assert.Null(ExpressionParser.Parse("(("));
            Assert.Null(ExpressionParser.Parse("LSN_X."));
            Assert.Null(ExpressionParser.Parse(""));
            Assert.Null(ExpressionParser.Parse(null));
        }

        [Fact]
        public void Parse_LessonRefMissingUnlockedSuffix_ReturnsNull()
        {
            // "LSN_X" alone is ambiguous and not allowed
            Assert.Null(ExpressionParser.Parse("LSN_X"));
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Util/Expression/IExpressionContext.cs`**

```csharp
namespace KerbalInstructionsKit.Util.Expression
{
    public interface IExpressionContext
    {
        bool IsLessonUnlocked(string lessonId);
        bool GetFlag(string flagName);
    }
}
```

- [ ] **Step 4: Write `src/KerbalInstructionsKit/Util/Expression/IExpression.cs`**

```csharp
namespace KerbalInstructionsKit.Util.Expression
{
    public interface IExpression
    {
        bool Evaluate(IExpressionContext ctx);
    }
}
```

- [ ] **Step 5: Write `src/KerbalInstructionsKit/Util/Expression/ExpressionNodes.cs`**

```csharp
namespace KerbalInstructionsKit.Util.Expression
{
    internal sealed class BoolLiteralExpr : IExpression
    {
        public bool Value;
        public bool Evaluate(IExpressionContext ctx) => Value;
    }

    internal sealed class LessonUnlockedExpr : IExpression
    {
        public string LessonId;
        public bool Evaluate(IExpressionContext ctx) => ctx.IsLessonUnlocked(LessonId);
    }

    internal sealed class FlagExpr : IExpression
    {
        public string FlagName;
        public bool Evaluate(IExpressionContext ctx) => ctx.GetFlag(FlagName);
    }

    internal sealed class NotExpr : IExpression
    {
        public IExpression Inner;
        public bool Evaluate(IExpressionContext ctx) => !Inner.Evaluate(ctx);
    }

    internal sealed class AndExpr : IExpression
    {
        public IExpression Left, Right;
        public bool Evaluate(IExpressionContext ctx) =>
            Left.Evaluate(ctx) && Right.Evaluate(ctx);
    }

    internal sealed class OrExpr : IExpression
    {
        public IExpression Left, Right;
        public bool Evaluate(IExpressionContext ctx) =>
            Left.Evaluate(ctx) || Right.Evaluate(ctx);
    }
}
```

- [ ] **Step 6: Write `src/KerbalInstructionsKit/Util/Expression/ExpressionParser.cs`**

```csharp
using System;
using System.Collections.Generic;
using System.Text;

namespace KerbalInstructionsKit.Util.Expression
{
    public static class ExpressionParser
    {
        public static IExpression Parse(string source)
        {
            if (string.IsNullOrEmpty(source)) return null;
            try
            {
                var tokens = Tokenize(source);
                var p = new Parser(tokens);
                var expr = p.ParseOr();
                return p.AtEnd ? expr : null;
            }
            catch (Exception)
            {
                return null;
            }
        }

        private enum Tok { Ident, Bool, LParen, RParen, Bang, And, Or, Dot, Comma, Eof }

        private struct Token
        {
            public Tok Kind;
            public string Text;
        }

        private static List<Token> Tokenize(string s)
        {
            var list = new List<Token>();
            int i = 0;
            while (i < s.Length)
            {
                char c = s[i];
                if (char.IsWhiteSpace(c)) { i++; continue; }
                if (c == '(') { list.Add(new Token { Kind = Tok.LParen, Text = "(" }); i++; continue; }
                if (c == ')') { list.Add(new Token { Kind = Tok.RParen, Text = ")" }); i++; continue; }
                if (c == '.') { list.Add(new Token { Kind = Tok.Dot, Text = "." }); i++; continue; }
                if (c == ',') { list.Add(new Token { Kind = Tok.Comma, Text = "," }); i++; continue; }
                if (c == '!')
                {
                    list.Add(new Token { Kind = Tok.Bang, Text = "!" }); i++; continue;
                }
                if (c == '&' && i + 1 < s.Length && s[i + 1] == '&')
                { list.Add(new Token { Kind = Tok.And, Text = "&&" }); i += 2; continue; }
                if (c == '|' && i + 1 < s.Length && s[i + 1] == '|')
                { list.Add(new Token { Kind = Tok.Or, Text = "||" }); i += 2; continue; }
                if (char.IsLetter(c) || c == '_')
                {
                    var sb = new StringBuilder();
                    while (i < s.Length && (char.IsLetterOrDigit(s[i]) || s[i] == '_'))
                    { sb.Append(s[i]); i++; }
                    var word = sb.ToString();
                    if (word == "true" || word == "false")
                        list.Add(new Token { Kind = Tok.Bool, Text = word });
                    else
                        list.Add(new Token { Kind = Tok.Ident, Text = word });
                    continue;
                }
                throw new Exception($"unexpected character '{c}' at {i}");
            }
            list.Add(new Token { Kind = Tok.Eof, Text = "" });
            return list;
        }

        private sealed class Parser
        {
            private readonly List<Token> tokens;
            private int pos;
            public Parser(List<Token> t) { tokens = t; pos = 0; }
            public bool AtEnd => tokens[pos].Kind == Tok.Eof;

            private Token Peek() => tokens[pos];
            private Token Next() => tokens[pos++];
            private bool Match(Tok k)
            {
                if (tokens[pos].Kind == k) { pos++; return true; }
                return false;
            }
            private void Expect(Tok k)
            {
                if (tokens[pos].Kind != k) throw new Exception($"expected {k}, got {tokens[pos].Kind}");
                pos++;
            }

            public IExpression ParseOr()
            {
                var left = ParseAnd();
                while (Match(Tok.Or))
                {
                    var right = ParseAnd();
                    left = new OrExpr { Left = left, Right = right };
                }
                return left;
            }

            private IExpression ParseAnd()
            {
                var left = ParseUnary();
                while (Match(Tok.And))
                {
                    var right = ParseUnary();
                    left = new AndExpr { Left = left, Right = right };
                }
                return left;
            }

            private IExpression ParseUnary()
            {
                if (Match(Tok.Bang))
                    return new NotExpr { Inner = ParseUnary() };
                return ParseAtom();
            }

            private IExpression ParseAtom()
            {
                var t = Peek();
                if (t.Kind == Tok.LParen)
                {
                    Next();
                    var inner = ParseOr();
                    Expect(Tok.RParen);
                    return inner;
                }
                if (t.Kind == Tok.Bool)
                {
                    Next();
                    return new BoolLiteralExpr { Value = t.Text == "true" };
                }
                if (t.Kind == Tok.Ident)
                {
                    Next();
                    if (t.Text == "flag")
                    {
                        Expect(Tok.LParen);
                        var inner = Next();
                        if (inner.Kind != Tok.Ident) throw new Exception("expected ident in flag()");
                        Expect(Tok.RParen);
                        return new FlagExpr { FlagName = inner.Text };
                    }
                    // lessonRef: IDENT '.unlocked'
                    Expect(Tok.Dot);
                    var suffix = Next();
                    if (suffix.Kind != Tok.Ident || suffix.Text != "unlocked")
                        throw new Exception($"expected '.unlocked', got '.{suffix.Text}'");
                    return new LessonUnlockedExpr { LessonId = t.Text };
                }
                throw new Exception($"unexpected token {t.Kind}");
            }
        }
    }
}
```

- [ ] **Step 7: Run tests, expect 11 pass**

- [ ] **Step 8: Commit**

```bash
git add src/KerbalInstructionsKit/Util/Expression/ tests/KerbalInstructionsKit.Tests/ExpressionParserTests.cs
git commit -m "feat: add KIK expression parser and evaluator"
```

### Task 9: LessonRegistry

**Files:**
- Create: `src/KerbalInstructionsKit/Core/LessonRegistry.cs`
- Create: `tests/KerbalInstructionsKit.Tests/LessonRegistryTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/LessonRegistryTests.cs`:

```csharp
using System.Linq;
using KerbalInstructionsKit.Core;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class LessonRegistryTests
    {
        private static Lesson L(string id, string title = null) =>
            new Lesson { Id = id, Title = title ?? id, Pages = { new Page { Text = "x" } } };

        [Fact]
        public void Register_AddsLesson()
        {
            var r = new LessonRegistry();
            Assert.True(r.Register(L("LSN_X")));
            Assert.Equal("LSN_X", r.Get("LSN_X").Id);
        }

        [Fact]
        public void Register_DuplicateId_FirstWins()
        {
            var r = new LessonRegistry();
            Assert.True(r.Register(L("LSN_X", "first")));
            Assert.False(r.Register(L("LSN_X", "second")));
            Assert.Equal("first", r.Get("LSN_X").Title);
        }

        [Fact]
        public void Get_Missing_ReturnsNull()
        {
            var r = new LessonRegistry();
            Assert.Null(r.Get("LSN_NoSuch"));
        }

        [Fact]
        public void All_ReturnsRegisteredLessons()
        {
            var r = new LessonRegistry();
            r.Register(L("A"));
            r.Register(L("B"));
            r.Register(L("C"));
            Assert.Equal(3, r.All.Count());
        }

        [Fact]
        public void Contains_ReportsMembership()
        {
            var r = new LessonRegistry();
            r.Register(L("A"));
            Assert.True(r.Contains("A"));
            Assert.False(r.Contains("B"));
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Core/LessonRegistry.cs`**

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace KerbalInstructionsKit.Core
{
    public sealed class LessonRegistry
    {
        private readonly Dictionary<string, Lesson> byId = new Dictionary<string, Lesson>();

        public bool Register(Lesson lesson)
        {
            if (lesson == null || string.IsNullOrEmpty(lesson.Id)) return false;
            if (byId.ContainsKey(lesson.Id))
            {
                Debug.LogWarning($"[KIK] duplicate lesson id '{lesson.Id}', first wins");
                return false;
            }
            byId[lesson.Id] = lesson;
            return true;
        }

        public Lesson Get(string id) =>
            id != null && byId.TryGetValue(id, out var l) ? l : null;

        public bool Contains(string id) => id != null && byId.ContainsKey(id);

        public IEnumerable<Lesson> All => byId.Values;

        public void Clear() => byId.Clear();
    }
}
```

- [ ] **Step 4: Run tests, expect 5 pass**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Core/LessonRegistry.cs tests/KerbalInstructionsKit.Tests/LessonRegistryTests.cs
git commit -m "feat: add LessonRegistry"
```

### Task 10: LessonState (per-save persistence)

**Files:**
- Create: `src/KerbalInstructionsKit/Core/LessonState.cs`
- Create: `tests/KerbalInstructionsKit.Tests/LessonStateTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/LessonStateTests.cs`:

```csharp
using KerbalInstructionsKit.Core;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class LessonStateTests
    {
        [Fact]
        public void NewState_NothingUnlocked()
        {
            var s = new LessonState();
            Assert.False(s.IsUnlocked("LSN_X"));
            Assert.False(s.GetFlag("F"));
            Assert.Null(s.LastViewedLesson);
            Assert.Equal(0, s.LastViewedPage);
        }

        [Fact]
        public void Unlock_RecordsLesson()
        {
            var s = new LessonState();
            Assert.True(s.Unlock("LSN_X"));   // returns true on first unlock
            Assert.True(s.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void Unlock_Idempotent()
        {
            var s = new LessonState();
            s.Unlock("LSN_X");
            Assert.False(s.Unlock("LSN_X"));  // returns false on subsequent calls
        }

        [Fact]
        public void SetFlag_RecordsValue()
        {
            var s = new LessonState();
            s.SetFlag("F", true);
            Assert.True(s.GetFlag("F"));
            s.SetFlag("F", false);
            Assert.False(s.GetFlag("F"));
        }

        [Fact]
        public void RoundTrip_PreservesAllState()
        {
            var s = new LessonState();
            s.Unlock("LSN_X");
            s.Unlock("LSN_Y");
            s.SetFlag("F", true);
            s.LastViewedLesson = "LSN_X";
            s.LastViewedPage = 3;

            var node = new ConfigNode("SCENARIO");
            s.Save(node);

            var s2 = new LessonState();
            s2.Load(node);

            Assert.True(s2.IsUnlocked("LSN_X"));
            Assert.True(s2.IsUnlocked("LSN_Y"));
            Assert.True(s2.GetFlag("F"));
            Assert.Equal("LSN_X", s2.LastViewedLesson);
            Assert.Equal(3, s2.LastViewedPage);
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Core/LessonState.cs`**

```csharp
using System.Collections.Generic;
using System.Globalization;

namespace KerbalInstructionsKit.Core
{
    public sealed class LessonState
    {
        private readonly HashSet<string> unlocked = new HashSet<string>();
        private readonly Dictionary<string, bool> flags = new Dictionary<string, bool>();

        public string LastViewedLesson;
        public int LastViewedPage;

        public bool IsUnlocked(string id) => id != null && unlocked.Contains(id);

        public bool Unlock(string id)
        {
            if (string.IsNullOrEmpty(id)) return false;
            return unlocked.Add(id);
        }

        public IEnumerable<string> AllUnlocked => unlocked;

        public bool GetFlag(string name) => name != null && flags.TryGetValue(name, out var v) && v;

        public void SetFlag(string name, bool value)
        {
            if (string.IsNullOrEmpty(name)) return;
            flags[name] = value;
        }

        public void Save(ConfigNode parent)
        {
            var unlockedNode = parent.AddNode("UNLOCKED");
            foreach (var id in unlocked) unlockedNode.AddValue(id, "true");

            var flagsNode = parent.AddNode("FLAGS");
            foreach (var kv in flags)
                flagsNode.AddValue(kv.Key, kv.Value.ToString(CultureInfo.InvariantCulture));

            if (!string.IsNullOrEmpty(LastViewedLesson))
            {
                parent.AddValue("LAST_VIEWED", LastViewedLesson);
                parent.AddValue("LAST_VIEWED_PAGE", LastViewedPage.ToString(CultureInfo.InvariantCulture));
            }
        }

        public void Load(ConfigNode parent)
        {
            unlocked.Clear();
            flags.Clear();
            LastViewedLesson = null;
            LastViewedPage = 0;

            if (parent.HasNode("UNLOCKED"))
            {
                foreach (ConfigNode.Value v in parent.GetNode("UNLOCKED").values)
                {
                    if (bool.TryParse(v.value, out var b) && b)
                        unlocked.Add(v.name);
                }
            }
            if (parent.HasNode("FLAGS"))
            {
                foreach (ConfigNode.Value v in parent.GetNode("FLAGS").values)
                {
                    if (bool.TryParse(v.value, out var b))
                        flags[v.name] = b;
                }
            }
            if (parent.HasValue("LAST_VIEWED"))
                LastViewedLesson = parent.GetValue("LAST_VIEWED");
            if (parent.HasValue("LAST_VIEWED_PAGE") &&
                int.TryParse(parent.GetValue("LAST_VIEWED_PAGE"),
                    NumberStyles.Integer, CultureInfo.InvariantCulture, out var p))
                LastViewedPage = p;
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 5 pass**

```bash
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Core/LessonState.cs tests/KerbalInstructionsKit.Tests/LessonStateTests.cs
git commit -m "feat: add LessonState with ConfigNode round-trip"
```

### Task 11: Trigger model and TriggerLoader

**Files:**
- Create: `src/KerbalInstructionsKit/Triggers/TriggerKind.cs`
- Create: `src/KerbalInstructionsKit/Triggers/LessonTrigger.cs`
- Create: `src/KerbalInstructionsKit/Triggers/TriggerLoader.cs`
- Create: `tests/KerbalInstructionsKit.Tests/TriggerLoaderTests.cs`

`LessonTrigger` is a plain value object (kind + parameters); the engine in Task 12 evaluates it against events.

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Triggers/TriggerKind.cs`**

```csharp
namespace KerbalInstructionsKit.Triggers
{
    public enum TriggerKind { GameStart, GameEvent, Contract, Flag }
    public enum ContractState { Offered, Accepted, Completed, Failed }
}
```

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Triggers/LessonTrigger.cs`**

```csharp
namespace KerbalInstructionsKit.Triggers
{
    public sealed class LessonTrigger
    {
        public string LessonId;
        public TriggerKind Kind;
        public string EventName;          // for GameEvent
        public string ContractName;       // for Contract
        public ContractState ContractState;   // for Contract
        public string FlagName;           // for Flag
    }
}
```

- [ ] **Step 3: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/TriggerLoaderTests.cs`:

```csharp
using KerbalInstructionsKit.Tests.TestHelpers;
using KerbalInstructionsKit.Triggers;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class TriggerLoaderTests
    {
        [Fact]
        public void Load_GameStart()
        {
            var n = new FakeSceneNode().Set("lesson", "LSN_X").Set("onGameStart", "true");
            var t = TriggerLoader.Load(n);
            Assert.Equal(TriggerKind.GameStart, t.Kind);
            Assert.Equal("LSN_X", t.LessonId);
        }

        [Fact]
        public void Load_GameEvent()
        {
            var n = new FakeSceneNode().Set("lesson", "LSN_X").Set("onGameEvent", "onLaunch");
            var t = TriggerLoader.Load(n);
            Assert.Equal(TriggerKind.GameEvent, t.Kind);
            Assert.Equal("onLaunch", t.EventName);
        }

        [Fact]
        public void Load_Contract_Offered()
        {
            var n = new FakeSceneNode()
                .Set("lesson", "LSN_X")
                .Set("onContract", "BKEX_FirstScience")
                .Set("state", "OFFERED");
            var t = TriggerLoader.Load(n);
            Assert.Equal(TriggerKind.Contract, t.Kind);
            Assert.Equal("BKEX_FirstScience", t.ContractName);
            Assert.Equal(ContractState.Offered, t.ContractState);
        }

        [Fact]
        public void Load_Contract_DefaultStateIsAccepted()
        {
            var n = new FakeSceneNode().Set("lesson", "LSN_X").Set("onContract", "C");
            var t = TriggerLoader.Load(n);
            Assert.Equal(ContractState.Accepted, t.ContractState);
        }

        [Fact]
        public void Load_Flag()
        {
            var n = new FakeSceneNode().Set("lesson", "LSN_X").Set("onFlag", "MyFlag");
            var t = TriggerLoader.Load(n);
            Assert.Equal(TriggerKind.Flag, t.Kind);
            Assert.Equal("MyFlag", t.FlagName);
        }

        [Fact]
        public void Load_NoLesson_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("onGameStart", "true");
            Assert.Null(TriggerLoader.Load(n));
        }

        [Fact]
        public void Load_NoTrigger_ReturnsNull()
        {
            var n = new FakeSceneNode().Set("lesson", "LSN_X");
            Assert.Null(TriggerLoader.Load(n));
        }
    }
}
```

- [ ] **Step 4: Run tests, expect compile errors**

- [ ] **Step 5: Write `src/KerbalInstructionsKit/Triggers/TriggerLoader.cs`**

```csharp
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Triggers
{
    public static class TriggerLoader
    {
        public static LessonTrigger Load(ISceneNode node)
        {
            if (node == null) return null;
            var lessonId = node.GetValue("lesson");
            if (string.IsNullOrEmpty(lessonId))
            {
                Debug.LogWarning("[KIK] LESSON_TRIGGER missing 'lesson', skipping");
                return null;
            }

            var t = new LessonTrigger { LessonId = lessonId };

            if (node.HasValue("onGameStart") &&
                bool.TryParse(node.GetValue("onGameStart"), out var b) && b)
            {
                t.Kind = TriggerKind.GameStart;
                return t;
            }

            var ev = node.GetValue("onGameEvent");
            if (!string.IsNullOrEmpty(ev))
            {
                t.Kind = TriggerKind.GameEvent;
                t.EventName = ev;
                return t;
            }

            var contract = node.GetValue("onContract");
            if (!string.IsNullOrEmpty(contract))
            {
                t.Kind = TriggerKind.Contract;
                t.ContractName = contract;
                t.ContractState = ParseContractState(node.GetValue("state"));
                return t;
            }

            var flag = node.GetValue("onFlag");
            if (!string.IsNullOrEmpty(flag))
            {
                t.Kind = TriggerKind.Flag;
                t.FlagName = flag;
                return t;
            }

            Debug.LogWarning($"[KIK] LESSON_TRIGGER for '{lessonId}' has no recognized trigger condition, skipping");
            return null;
        }

        private static ContractState ParseContractState(string s)
        {
            if (string.IsNullOrEmpty(s)) return ContractState.Accepted;
            switch (s.ToUpperInvariant())
            {
                case "OFFERED":   return ContractState.Offered;
                case "COMPLETED": return ContractState.Completed;
                case "FAILED":    return ContractState.Failed;
                default:          return ContractState.Accepted;
            }
        }
    }
}
```

- [ ] **Step 6: Run tests, expect 7 pass**

- [ ] **Step 7: Commit**

```bash
git add src/KerbalInstructionsKit/Triggers/ tests/KerbalInstructionsKit.Tests/TriggerLoaderTests.cs
git commit -m "feat: add LessonTrigger model and TriggerLoader"
```

### Task 12: LessonTriggerEngine (pure-logic)

`LessonTriggerEngine` is the runtime side of triggers — feed it events, it unlocks lessons. The KSP wiring (subscribing to `GameEvents`) lives in a runtime addon (Task 21); this task is pure-logic and unit-tested.

**Files:**
- Create: `src/KerbalInstructionsKit/Triggers/LessonTriggerEngine.cs`
- Create: `tests/KerbalInstructionsKit.Tests/LessonTriggerEngineTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/LessonTriggerEngineTests.cs`:

```csharp
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Triggers;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class LessonTriggerEngineTests
    {
        private static LessonTrigger T(TriggerKind k, string lesson = "LSN_X") =>
            new LessonTrigger { Kind = k, LessonId = lesson };

        [Fact]
        public void OnGameStart_FiresOnce()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(T(TriggerKind.GameStart, "LSN_X"));

            engine.RunGameStartTriggers();

            Assert.True(state.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void OnGameEvent_MatchByName()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(new LessonTrigger { Kind = TriggerKind.GameEvent, EventName = "onLaunch", LessonId = "LSN_X" });

            engine.OnGameEvent("onLaunch");
            Assert.True(state.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void OnGameEvent_NameMismatch_DoesNotUnlock()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(new LessonTrigger { Kind = TriggerKind.GameEvent, EventName = "onLaunch", LessonId = "LSN_X" });

            engine.OnGameEvent("onLand");
            Assert.False(state.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void OnContract_MatchesNameAndState()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(new LessonTrigger
            {
                Kind = TriggerKind.Contract,
                ContractName = "BKEX_X",
                ContractState = ContractState.Accepted,
                LessonId = "LSN_X"
            });

            engine.OnContractEvent("BKEX_X", ContractState.Offered);
            Assert.False(state.IsUnlocked("LSN_X"));   // wrong state
            engine.OnContractEvent("BKEX_X", ContractState.Accepted);
            Assert.True(state.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void OnFlag_TriggersWhenSetTrue()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(new LessonTrigger { Kind = TriggerKind.Flag, FlagName = "F", LessonId = "LSN_X" });

            engine.OnFlagSet("F", false);
            Assert.False(state.IsUnlocked("LSN_X"));
            engine.OnFlagSet("F", true);
            Assert.True(state.IsUnlocked("LSN_X"));
        }

        [Fact]
        public void Unlock_FiresLessonUnlockedEventOnce()
        {
            var state = new LessonState();
            var engine = new LessonTriggerEngine(state);
            engine.Register(T(TriggerKind.GameStart, "LSN_X"));

            int hits = 0;
            engine.LessonUnlocked += id => hits++;
            engine.RunGameStartTriggers();
            engine.RunGameStartTriggers();   // second run: already unlocked, no event

            Assert.Equal(1, hits);
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Triggers/LessonTriggerEngine.cs`**

```csharp
using System;
using System.Collections.Generic;
using KerbalInstructionsKit.Core;

namespace KerbalInstructionsKit.Triggers
{
    public sealed class LessonTriggerEngine
    {
        private readonly LessonState state;
        private readonly List<LessonTrigger> triggers = new List<LessonTrigger>();

        public event Action<string> LessonUnlocked;

        public LessonTriggerEngine(LessonState state)
        {
            this.state = state;
        }

        public void Register(LessonTrigger t)
        {
            if (t != null) triggers.Add(t);
        }

        public IEnumerable<LessonTrigger> All => triggers;

        public void RunGameStartTriggers()
        {
            foreach (var t in triggers)
                if (t.Kind == TriggerKind.GameStart) Fire(t.LessonId);
        }

        public void OnGameEvent(string name)
        {
            foreach (var t in triggers)
                if (t.Kind == TriggerKind.GameEvent && t.EventName == name)
                    Fire(t.LessonId);
        }

        public void OnContractEvent(string contractName, ContractState eventState)
        {
            foreach (var t in triggers)
                if (t.Kind == TriggerKind.Contract &&
                    t.ContractName == contractName &&
                    t.ContractState == eventState)
                    Fire(t.LessonId);
        }

        public void OnFlagSet(string flagName, bool value)
        {
            if (!value) return;   // flag triggers fire on true only
            foreach (var t in triggers)
                if (t.Kind == TriggerKind.Flag && t.FlagName == flagName)
                    Fire(t.LessonId);
        }

        private void Fire(string lessonId)
        {
            if (state.Unlock(lessonId))
                LessonUnlocked?.Invoke(lessonId);
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 6 pass**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Triggers/LessonTriggerEngine.cs tests/KerbalInstructionsKit.Tests/LessonTriggerEngineTests.cs
git commit -m "feat: add LessonTriggerEngine pure-logic"
```

### Task 13: Archive filter logic

Pure-logic that takes the registry, state, an active filter (search string + selected tags), and an `IExpressionContext`, and produces a sorted, grouped, filtered view. Tested independently of UI.

**Files:**
- Create: `src/KerbalInstructionsKit/Core/ArchiveQuery.cs`
- Create: `src/KerbalInstructionsKit/Core/ArchiveFilter.cs`
- Create: `tests/KerbalInstructionsKit.Tests/ArchiveFilterTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/ArchiveFilterTests.cs`:

```csharp
using System.Collections.Generic;
using System.Linq;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Util.Expression;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class ArchiveFilterTests
    {
        private sealed class Ctx : IExpressionContext
        {
            public HashSet<string> U = new HashSet<string>();
            public Dictionary<string, bool> F = new Dictionary<string, bool>();
            public bool IsLessonUnlocked(string id) => U.Contains(id);
            public bool GetFlag(string n) => F.TryGetValue(n, out var v) && v;
        }

        private static Lesson Mk(string id, string cat, int order, params string[] tags)
        {
            var l = new Lesson { Id = id, Title = id, Category = cat, SortOrder = order };
            l.Tags = tags.ToList();
            l.Pages.Add(new Page { Text = "x" });
            return l;
        }

        private static LessonRegistry Reg(params Lesson[] ls)
        {
            var r = new LessonRegistry();
            foreach (var l in ls) r.Register(l);
            return r;
        }

        [Fact]
        public void Query_OnlyShowsUnlocked()
        {
            var reg = Reg(Mk("A", "X", 100), Mk("B", "X", 200));
            var ctx = new Ctx(); ctx.U.Add("A");
            var query = new ArchiveQuery();

            var groups = ArchiveFilter.Apply(reg, ctx, query);

            Assert.Single(groups);
            Assert.Single(groups[0].Lessons);
            Assert.Equal("A", groups[0].Lessons[0].Id);
        }

        [Fact]
        public void Query_GroupsByCategory_NullCategoryGoesToMisc()
        {
            var reg = Reg(Mk("A", "Cat1", 100), Mk("B", null, 100), Mk("C", "Cat1", 50));
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B"); ctx.U.Add("C");

            var groups = ArchiveFilter.Apply(reg, ctx, new ArchiveQuery());

            Assert.Equal(2, groups.Count);
            var cat1 = groups.First(g => g.Category == "Cat1");
            var misc = groups.First(g => g.Category == "Misc");
            Assert.Equal(new[] { "C", "A" }, cat1.Lessons.Select(l => l.Id).ToArray());
            Assert.Equal(new[] { "B" }, misc.Lessons.Select(l => l.Id).ToArray());
        }

        [Fact]
        public void Query_SortOrderWithinCategory_AscendingThenTitle()
        {
            var reg = Reg(Mk("A", "X", 200), Mk("B", "X", 100), Mk("C", "X", 100));
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B"); ctx.U.Add("C");

            var groups = ArchiveFilter.Apply(reg, ctx, new ArchiveQuery());
            Assert.Equal(new[] { "B", "C", "A" },
                groups[0].Lessons.Select(l => l.Id).ToArray());
        }

        [Fact]
        public void Query_SearchMatchesTitleSubstring_CaseInsensitive()
        {
            var a = Mk("A", "X", 100); a.Title = "Maneuver Nodes";
            var b = Mk("B", "X", 100); b.Title = "Rocket Basics";
            var reg = Reg(a, b);
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B");

            var groups = ArchiveFilter.Apply(reg, ctx, new ArchiveQuery { Search = "maneuver" });
            Assert.Single(groups);
            Assert.Equal("A", groups[0].Lessons[0].Id);
        }

        [Fact]
        public void Query_SearchMatchesTag()
        {
            var a = Mk("A", "X", 100, "Beginner");
            var b = Mk("B", "X", 100, "Advanced");
            var reg = Reg(a, b);
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B");

            var groups = ArchiveFilter.Apply(reg, ctx, new ArchiveQuery { Search = "Beg" });
            Assert.Equal("A", groups[0].Lessons[0].Id);
        }

        [Fact]
        public void Query_TagFilter_OrSemantics()
        {
            var a = Mk("A", "X", 100, "Orbits");
            var b = Mk("B", "X", 100, "Planes");
            var c = Mk("C", "X", 100, "Surface");
            var reg = Reg(a, b, c);
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B"); ctx.U.Add("C");

            var q = new ArchiveQuery();
            q.SelectedTags.Add("Orbits");
            q.SelectedTags.Add("Planes");

            var groups = ArchiveFilter.Apply(reg, ctx, q);
            Assert.Equal(new[] { "A", "B" },
                groups[0].Lessons.Select(l => l.Id).OrderBy(x => x).ToArray());
        }

        [Fact]
        public void Query_VisibleIfFalse_HidesLesson()
        {
            var a = Mk("A", "X", 100);
            a.VisibleIfRaw = "false";
            var b = Mk("B", "X", 100);
            var reg = Reg(a, b);
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B");

            var groups = ArchiveFilter.Apply(reg, ctx, new ArchiveQuery());
            Assert.Single(groups);
            Assert.Equal("B", groups[0].Lessons[0].Id);
        }

        [Fact]
        public void Query_AllVisibleTags_AggregatesFromUnlocked()
        {
            var a = Mk("A", "X", 100, "Orbits", "Beginner");
            var b = Mk("B", "X", 100, "Planes");
            var reg = Reg(a, b);
            var ctx = new Ctx(); ctx.U.Add("A"); ctx.U.Add("B");

            var tags = ArchiveFilter.AllVisibleTags(reg, ctx);
            Assert.Equal(new[] { "Beginner", "Orbits", "Planes" },
                tags.OrderBy(t => t).ToArray());
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Core/ArchiveQuery.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Core
{
    public sealed class ArchiveQuery
    {
        public string Search = "";
        public List<string> SelectedTags = new List<string>();
    }

    public sealed class ArchiveGroup
    {
        public string Category;
        public List<Lesson> Lessons = new List<Lesson>();
    }
}
```

- [ ] **Step 4: Write `src/KerbalInstructionsKit/Core/ArchiveFilter.cs`**

```csharp
using System.Collections.Generic;
using System.Linq;
using KerbalInstructionsKit.Util.Expression;

namespace KerbalInstructionsKit.Core
{
    public static class ArchiveFilter
    {
        private static readonly Dictionary<string, IExpression> visibleIfCache =
            new Dictionary<string, IExpression>();
        private static readonly HashSet<string> visibleIfFailed =
            new HashSet<string>();

        internal static void ResetCache()   // for tests
        {
            visibleIfCache.Clear();
            visibleIfFailed.Clear();
        }

        private static bool IsVisible(Lesson l, IExpressionContext ctx)
        {
            if (string.IsNullOrEmpty(l.VisibleIfRaw)) return true;
            if (!visibleIfCache.TryGetValue(l.Id, out var expr))
            {
                expr = ExpressionParser.Parse(l.VisibleIfRaw);
                visibleIfCache[l.Id] = expr;
                if (expr == null)
                {
                    UnityEngine.Debug.LogWarning(
                        $"[KIK] visibleIf parse error on lesson '{l.Id}': '{l.VisibleIfRaw}'. Treated as true.");
                }
            }
            if (expr == null) return true;
            try { return expr.Evaluate(ctx); }
            catch
            {
                if (visibleIfFailed.Add(l.Id))
                    UnityEngine.Debug.LogWarning(
                        $"[KIK] visibleIf runtime error on lesson '{l.Id}'. Treated as false.");
                return false;
            }
        }

        public static List<ArchiveGroup> Apply(
            LessonRegistry reg, IExpressionContext ctx, ArchiveQuery query)
        {
            var search = (query.Search ?? "").Trim().ToLowerInvariant();
            var tags = query.SelectedTags ?? new List<string>();

            var visible = reg.All
                .Where(l => ctx.IsLessonUnlocked(l.Id))
                .Where(l => IsVisible(l, ctx))
                .Where(l => MatchesSearch(l, search))
                .Where(l => MatchesTags(l, tags))
                .ToList();

            return visible
                .GroupBy(l => string.IsNullOrEmpty(l.Category) ? "Misc" : l.Category)
                .Select(g => new ArchiveGroup
                {
                    Category = g.Key,
                    Lessons = g.OrderBy(l => l.SortOrder)
                               .ThenBy(l => l.Title)
                               .ToList(),
                })
                .OrderBy(g => g.Category == "Misc" ? 1 : 0)   // Misc last
                .ThenBy(g => g.Category)
                .ToList();
        }

        public static List<string> AllVisibleTags(
            LessonRegistry reg, IExpressionContext ctx)
        {
            return reg.All
                .Where(l => ctx.IsLessonUnlocked(l.Id))
                .Where(l => IsVisible(l, ctx))
                .SelectMany(l => l.Tags)
                .Distinct()
                .OrderBy(t => t)
                .ToList();
        }

        private static bool MatchesSearch(Lesson l, string search)
        {
            if (string.IsNullOrEmpty(search)) return true;
            if ((l.Title ?? "").ToLowerInvariant().Contains(search)) return true;
            if ((l.Category ?? "").ToLowerInvariant().Contains(search)) return true;
            foreach (var t in l.Tags)
                if (t.ToLowerInvariant().Contains(search)) return true;
            return false;
        }

        private static bool MatchesTags(Lesson l, List<string> tags)
        {
            if (tags.Count == 0) return true;
            foreach (var t in tags)
                if (l.Tags.Contains(t)) return true;
            return false;
        }
    }
}
```

- [ ] **Step 5: Run tests, expect 8 pass**

```bash
dotnet test tests/KerbalInstructionsKit.Tests/KerbalInstructionsKit.Tests.csproj
```

If `Apply` test caches a `visibleIf` between tests, call `ArchiveFilter.ResetCache()` from a `[Fact]` setup or use `IClassFixture`. For now the cache key is lesson Id and tests use unique lessons, so no collision.

- [ ] **Step 6: Commit**

```bash
git add src/KerbalInstructionsKit/Core/ArchiveQuery.cs src/KerbalInstructionsKit/Core/ArchiveFilter.cs tests/KerbalInstructionsKit.Tests/ArchiveFilterTests.cs
git commit -m "feat: add ArchiveFilter with search/tag/visibleIf logic"
```

### Task 14: PanelStateMachine

Tracks the current view (Lesson | Archive), current lesson, current page, and back stack. Pure logic. Used by both the rendering layer and tests.

**Files:**
- Create: `src/KerbalInstructionsKit/Core/PanelStateMachine.cs`
- Create: `tests/KerbalInstructionsKit.Tests/PanelStateMachineTests.cs`

- [ ] **Step 1: Write the failing tests**

`tests/KerbalInstructionsKit.Tests/PanelStateMachineTests.cs`:

```csharp
using KerbalInstructionsKit.Core;
using Xunit;

namespace KerbalInstructionsKit.Tests
{
    public class PanelStateMachineTests
    {
        private static LessonRegistry RegWith(int pages, params string[] ids)
        {
            var r = new LessonRegistry();
            foreach (var id in ids)
            {
                var l = new Lesson { Id = id, Title = id };
                for (int i = 0; i < pages; i++) l.Pages.Add(new Page { Text = $"page {i}" });
                r.Register(l);
            }
            return r;
        }

        [Fact]
        public void Initial_State_IsArchive()
        {
            var sm = new PanelStateMachine(RegWith(2, "A"));
            Assert.Equal(PanelView.Archive, sm.View);
            Assert.Null(sm.CurrentLesson);
        }

        [Fact]
        public void OpenLesson_SwitchesToLessonView_Page0()
        {
            var sm = new PanelStateMachine(RegWith(3, "A"));
            sm.OpenLesson("A");
            Assert.Equal(PanelView.Lesson, sm.View);
            Assert.Equal("A", sm.CurrentLesson.Id);
            Assert.Equal(0, sm.CurrentPage);
        }

        [Fact]
        public void OpenLesson_Unknown_NoEffect()
        {
            var sm = new PanelStateMachine(RegWith(2, "A"));
            sm.OpenLesson("ZZ");
            Assert.Equal(PanelView.Archive, sm.View);
            Assert.Null(sm.CurrentLesson);
        }

        [Fact]
        public void NextPage_AdvancesUntilLast()
        {
            var sm = new PanelStateMachine(RegWith(3, "A"));
            sm.OpenLesson("A");
            sm.NextPage();
            Assert.Equal(1, sm.CurrentPage);
            sm.NextPage();
            Assert.Equal(2, sm.CurrentPage);
            sm.NextPage();
            Assert.Equal(2, sm.CurrentPage);   // clamped at last page
        }

        [Fact]
        public void PrevPage_GoesBackUntilFirst()
        {
            var sm = new PanelStateMachine(RegWith(3, "A"));
            sm.OpenLesson("A");
            sm.NextPage(); sm.NextPage();
            sm.PrevPage(); Assert.Equal(1, sm.CurrentPage);
            sm.PrevPage(); Assert.Equal(0, sm.CurrentPage);
            sm.PrevPage(); Assert.Equal(0, sm.CurrentPage);
        }

        [Fact]
        public void OpenArchive_FromLesson_RemembersLesson()
        {
            var sm = new PanelStateMachine(RegWith(2, "A"));
            sm.OpenLesson("A");
            sm.OpenArchive();
            Assert.Equal(PanelView.Archive, sm.View);
            Assert.Equal("A", sm.CurrentLesson.Id);   // current lesson preserved
        }

        [Fact]
        public void NavigateToLesson_PushesBackStack()
        {
            var sm = new PanelStateMachine(RegWith(2, "A", "B"));
            sm.OpenLesson("A");
            sm.NavigateToLessonViaLink("B");
            Assert.Equal("B", sm.CurrentLesson.Id);
            Assert.True(sm.CanGoBack);
            sm.GoBack();
            Assert.Equal("A", sm.CurrentLesson.Id);
            Assert.False(sm.CanGoBack);
        }

        [Fact]
        public void OpenLessonDirect_ClearsBackStack()
        {
            var sm = new PanelStateMachine(RegWith(2, "A", "B", "C"));
            sm.OpenLesson("A");
            sm.NavigateToLessonViaLink("B");
            Assert.True(sm.CanGoBack);
            sm.OpenLesson("C");   // direct open clears stack
            Assert.False(sm.CanGoBack);
        }
    }
}
```

- [ ] **Step 2: Run tests, expect compile errors**

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Core/PanelStateMachine.cs`**

```csharp
using System.Collections.Generic;

namespace KerbalInstructionsKit.Core
{
    public enum PanelView { Lesson, Archive }

    public sealed class PanelStateMachine
    {
        private readonly LessonRegistry registry;
        private readonly Stack<string> backStack = new Stack<string>();

        public PanelView View { get; private set; } = PanelView.Archive;
        public Lesson CurrentLesson { get; private set; }
        public int CurrentPage { get; private set; }
        public bool CanGoBack => backStack.Count > 0;

        public PanelStateMachine(LessonRegistry registry)
        {
            this.registry = registry;
        }

        public void OpenLesson(string id)
        {
            var l = registry.Get(id);
            if (l == null) return;
            CurrentLesson = l;
            CurrentPage = 0;
            View = PanelView.Lesson;
            backStack.Clear();
        }

        public void NavigateToLessonViaLink(string id)
        {
            var l = registry.Get(id);
            if (l == null) return;
            if (CurrentLesson != null) backStack.Push(CurrentLesson.Id);
            CurrentLesson = l;
            CurrentPage = 0;
            View = PanelView.Lesson;
        }

        public void GoBack()
        {
            if (backStack.Count == 0) return;
            var prev = backStack.Pop();
            CurrentLesson = registry.Get(prev);
            CurrentPage = 0;
            View = PanelView.Lesson;
        }

        public void OpenArchive()
        {
            View = PanelView.Archive;
            // CurrentLesson preserved so "Back to Lesson" footer can return
        }

        public void BackToLesson()
        {
            if (CurrentLesson == null) return;
            View = PanelView.Lesson;
        }

        public void NextPage()
        {
            if (CurrentLesson == null) return;
            if (CurrentPage < CurrentLesson.Pages.Count - 1) CurrentPage++;
        }

        public void PrevPage()
        {
            if (CurrentLesson == null) return;
            if (CurrentPage > 0) CurrentPage--;
        }

        public void JumpToPage(int page)
        {
            if (CurrentLesson == null) return;
            if (page < 0 || page >= CurrentLesson.Pages.Count) return;
            CurrentPage = page;
        }

        public void Reset()
        {
            View = PanelView.Archive;
            CurrentLesson = null;
            CurrentPage = 0;
            backStack.Clear();
        }
    }
}
```

- [ ] **Step 4: Run tests, expect 8 pass**

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Core/PanelStateMachine.cs tests/KerbalInstructionsKit.Tests/PanelStateMachineTests.cs
git commit -m "feat: add PanelStateMachine with view/page/back-stack logic"
```

## Phase 2 — Public API and Scenario glue

### Task 15: InstructionsKit static API

The user-facing class. Mostly a thin facade that delegates to a singleton `KikRuntime` (created in Task 17 and held by the scenario). For now build the class with stub forwarding so other code can compile against the API.

**Files:**
- Create: `src/KerbalInstructionsKit/Core/InstructionsKit.cs`
- Create: `src/KerbalInstructionsKit/Core/IExpressionContextAdapter.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Core/IExpressionContextAdapter.cs`**

The adapter that lets `LessonState` answer `IExpressionContext` queries:

```csharp
using KerbalInstructionsKit.Util.Expression;

namespace KerbalInstructionsKit.Core
{
    public sealed class StateExpressionContext : IExpressionContext
    {
        private readonly LessonState state;
        public StateExpressionContext(LessonState state) { this.state = state; }
        public bool IsLessonUnlocked(string id) => state.IsUnlocked(id);
        public bool GetFlag(string name) => state.GetFlag(name);
    }
}
```

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Core/InstructionsKit.cs`**

```csharp
using System;
using KerbalInstructionsKit.Util.Expression;

namespace KerbalInstructionsKit.Core
{
    /// <summary>
    /// Public facade for KIK. All state lives behind the active scenario; this class
    /// forwards calls to whatever runtime is currently registered.
    /// </summary>
    public static class InstructionsKit
    {
        public static IKikRuntime Runtime { get; internal set; }

        public static LessonRegistry Lessons => Runtime?.Lessons;
        public static LessonState State => Runtime?.State;
        public static IExpressionContext ExpressionContext => Runtime?.ExpressionContext;

        public static event Action<string> LessonUnlocked
        {
            add    { if (Runtime != null) Runtime.LessonUnlocked += value; }
            remove { if (Runtime != null) Runtime.LessonUnlocked -= value; }
        }

        public static void OpenLesson(string lessonId) => Runtime?.OpenLesson(lessonId);
        public static void OpenArchive() => Runtime?.OpenArchive();
        public static void Close() => Runtime?.Close();

        public static void SetFlag(string name, bool value) => Runtime?.SetFlag(name, value);
        public static bool GetFlag(string name) => Runtime?.GetFlag(name) ?? false;
    }

    public interface IKikRuntime
    {
        LessonRegistry Lessons { get; }
        LessonState State { get; }
        IExpressionContext ExpressionContext { get; }
        event Action<string> LessonUnlocked;

        void OpenLesson(string lessonId);
        void OpenArchive();
        void Close();
        void SetFlag(string name, bool value);
        bool GetFlag(string name);
    }
}
```

- [ ] **Step 3: Build to verify compile**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

- [ ] **Step 4: Commit**

```bash
git add src/KerbalInstructionsKit/Core/InstructionsKit.cs src/KerbalInstructionsKit/Core/IExpressionContextAdapter.cs
git commit -m "feat: add InstructionsKit public facade and IKikRuntime interface"
```

### Task 16: KerbalInstructionsKitScenario

The `ScenarioModule` that owns per-save state. Loads lesson cfgs once at game-start (via a flag), wires up the trigger engine, and exposes itself as `IKikRuntime`.

**Files:**
- Create: `src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs`**

```csharp
using System;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Triggers;
using KerbalInstructionsKit.Util.Expression;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    [KSPScenario(
        ScenarioCreationOptions.AddToAllGames,
        GameScenes.SPACECENTER, GameScenes.EDITOR, GameScenes.FLIGHT,
        GameScenes.TRACKSTATION)]
    public sealed class KerbalInstructionsKitScenario : ScenarioModule, IKikRuntime
    {
        public LessonRegistry Lessons { get; private set; }
        public LessonState State { get; private set; }
        public LessonTriggerEngine TriggerEngine { get; private set; }
        public IExpressionContext ExpressionContext { get; private set; }

        public event Action<string> LessonUnlocked;

        // Panel UI hooks — set by the active KSPAddon for the current scene.
        public Action<string> PanelOpenLesson;
        public Action PanelOpenArchive;
        public Action PanelClose;

        public override void OnAwake()
        {
            Lessons = LessonContentLoader.LoadAll();
            State = new LessonState();
            ExpressionContext = new StateExpressionContext(State);
            TriggerEngine = new LessonTriggerEngine(State);

            TriggerEngine.LessonUnlocked += id =>
            {
                Debug.Log($"[KIK] lesson unlocked: {id}");
                LessonUnlocked?.Invoke(id);
            };

            LessonContentLoader.LoadTriggers(TriggerEngine);

            InstructionsKit.Runtime = this;
        }

        public override void OnLoad(ConfigNode node)
        {
            State.Load(node);

            // Run game-start triggers if this is a fresh save.
            if (!node.HasValue("KIK_GAME_START_FIRED"))
            {
                TriggerEngine.RunGameStartTriggers();
            }
        }

        public override void OnSave(ConfigNode node)
        {
            State.Save(node);
            node.AddValue("KIK_GAME_START_FIRED", "true");
        }

        public void OpenLesson(string id)
        {
            State.LastViewedLesson = id;
            State.LastViewedPage = 0;
            PanelOpenLesson?.Invoke(id);
        }

        public void OpenArchive() => PanelOpenArchive?.Invoke();
        public void Close() => PanelClose?.Invoke();

        public void SetFlag(string name, bool value)
        {
            State.SetFlag(name, value);
            TriggerEngine.OnFlagSet(name, value);
        }

        public bool GetFlag(string name) => State.GetFlag(name);

        public void OnDestroy()
        {
            if (InstructionsKit.Runtime == (IKikRuntime)this)
                InstructionsKit.Runtime = null;
        }
    }
}
```

- [ ] **Step 2: Add a placeholder `LessonContentLoader` so this compiles**

Create `src/KerbalInstructionsKit/Config/LessonContentLoader.cs`:

```csharp
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Triggers;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Config
{
    /// <summary>
    /// Walks GameDatabase for INSTRUCTION_LESSON and LESSON_TRIGGER nodes.
    /// </summary>
    public static class LessonContentLoader
    {
        public static LessonRegistry LoadAll()
        {
            var reg = new LessonRegistry();
            int loaded = 0, skipped = 0;
            foreach (var url in GameDatabase.Instance.GetConfigs("INSTRUCTION_LESSON"))
            {
                var lesson = LessonLoader.Load(new ConfigNodeAdapter(url.config));
                if (lesson != null && reg.Register(lesson)) loaded++;
                else skipped++;
            }
            Debug.Log($"[KIK] loaded {loaded} lessons (skipped {skipped})");
            return reg;
        }

        public static void LoadTriggers(LessonTriggerEngine engine)
        {
            int loaded = 0;
            foreach (var url in GameDatabase.Instance.GetConfigs("LESSON_TRIGGER"))
            {
                var t = TriggerLoader.Load(new ConfigNodeAdapter(url.config));
                if (t != null) { engine.Register(t); loaded++; }
            }
            Debug.Log($"[KIK] loaded {loaded} standalone LESSON_TRIGGER blocks");
        }
    }
}
```

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: success.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs src/KerbalInstructionsKit/Config/LessonContentLoader.cs
git commit -m "feat: add ScenarioModule and content loader"
```

---

## Phase 3 — Runtime: per-scene addons, toolbar, GameEvents

### Task 17: KikSceneAddonBase and per-scene KSPAddons

Each scene addon registers a toolbar button and hosts the IMGUI panel. The actual rendering (Task 23+) is delegated to a `LessonPanel` class shared across addons.

**Files:**
- Create: `src/KerbalInstructionsKit/Runtime/KikSceneAddonBase.cs`
- Create: `src/KerbalInstructionsKit/Runtime/KikKscAddon.cs`
- Create: `src/KerbalInstructionsKit/Runtime/KikEditorAddon.cs`
- Create: `src/KerbalInstructionsKit/Runtime/KikFlightAddon.cs`
- Create: `src/KerbalInstructionsKit/Runtime/KikTrackingStationAddon.cs`
- Create: `src/KerbalInstructionsKit/Util/IPauseController.cs`
- Create: `src/KerbalInstructionsKit/Util/FlightDriverPauseController.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Util/IPauseController.cs`**

```csharp
namespace KerbalInstructionsKit.Util
{
    public interface IPauseController
    {
        bool IsAvailable { get; }       // true only in Flight
        bool IsPaused { get; }
        void SetPaused(bool paused);
    }
}
```

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Util/FlightDriverPauseController.cs`**

```csharp
namespace KerbalInstructionsKit.Util
{
    public sealed class FlightDriverPauseController : IPauseController
    {
        public bool IsAvailable => HighLogic.LoadedSceneIsFlight;
        public bool IsPaused => FlightDriver.Pause;
        public void SetPaused(bool paused) => FlightDriver.SetPause(paused);
    }

    public sealed class NullPauseController : IPauseController
    {
        public bool IsAvailable => false;
        public bool IsPaused => false;
        public void SetPaused(bool paused) { }
    }
}
```

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Runtime/KikSceneAddonBase.cs`**

```csharp
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Rendering;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    public abstract class KikSceneAddonBase : MonoBehaviour
    {
        protected LessonPanel Panel;
        protected KikToolbarButton Toolbar;
        protected IPauseController PauseController;

        protected abstract IPauseController CreatePauseController();
        protected abstract string ToolbarSceneFlags();

        protected virtual void Start()
        {
            var scenario = HighLogic.CurrentGame?.scenarios?
                .Find(s => s.moduleName == nameof(KerbalInstructionsKitScenario))?
                .moduleRef as KerbalInstructionsKitScenario;
            if (scenario == null)
            {
                Debug.LogWarning("[KIK] scenario not yet attached; addon idle");
                return;
            }

            PauseController = CreatePauseController();
            Panel = new LessonPanel(scenario, PauseController);
            Toolbar = new KikToolbarButton(this, ToolbarSceneFlags(), () => Panel.Toggle());

            scenario.PanelOpenLesson = Panel.OpenLesson;
            scenario.PanelOpenArchive = Panel.OpenArchive;
            scenario.PanelClose = Panel.Close;
        }

        protected virtual void OnGUI() => Panel?.Draw();

        protected virtual void OnDestroy()
        {
            Toolbar?.Dispose();
            Panel?.Dispose();
        }
    }
}
```

- [ ] **Step 4: Write the four scene addons**

`src/KerbalInstructionsKit/Runtime/KikKscAddon.cs`:

```csharp
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.SpaceCentre, false)]
    public sealed class KikKscAddon : KikSceneAddonBase
    {
        protected override IPauseController CreatePauseController() => new NullPauseController();
        protected override string ToolbarSceneFlags() => "SpaceCentre";
    }
}
```

`src/KerbalInstructionsKit/Runtime/KikEditorAddon.cs`:

```csharp
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.EditorAny, false)]
    public sealed class KikEditorAddon : KikSceneAddonBase
    {
        protected override IPauseController CreatePauseController() => new NullPauseController();
        protected override string ToolbarSceneFlags() => "Editor";
    }
}
```

`src/KerbalInstructionsKit/Runtime/KikFlightAddon.cs`:

```csharp
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.Flight, false)]
    public sealed class KikFlightAddon : KikSceneAddonBase
    {
        protected override IPauseController CreatePauseController() => new FlightDriverPauseController();
        protected override string ToolbarSceneFlags() => "Flight,MapFlight";
    }
}
```

`src/KerbalInstructionsKit/Runtime/KikTrackingStationAddon.cs`:

```csharp
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.TrackingStation, false)]
    public sealed class KikTrackingStationAddon : KikSceneAddonBase
    {
        protected override IPauseController CreatePauseController() => new NullPauseController();
        protected override string ToolbarSceneFlags() => "TrackingStation";
    }
}
```

- [ ] **Step 5: Add stub `LessonPanel` and `KikToolbarButton` so the project compiles**

Create `src/KerbalInstructionsKit/Rendering/LessonPanel.cs` with a placeholder body:

```csharp
using System;
using KerbalInstructionsKit.Runtime;
using KerbalInstructionsKit.Util;

namespace KerbalInstructionsKit.Rendering
{
    public sealed class LessonPanel : IDisposable
    {
        private readonly KerbalInstructionsKitScenario scenario;
        private readonly IPauseController pause;

        public LessonPanel(KerbalInstructionsKitScenario scenario, IPauseController pause)
        {
            this.scenario = scenario;
            this.pause = pause;
        }

        public void Toggle() { /* implemented in Task 23 */ }
        public void OpenLesson(string id) { /* implemented in Task 23 */ }
        public void OpenArchive() { /* implemented in Task 23 */ }
        public void Close() { /* implemented in Task 23 */ }
        public void Draw() { /* implemented in Task 23 */ }
        public void Dispose() { }
    }
}
```

Create `src/KerbalInstructionsKit/Runtime/KikToolbarButton.cs` with a stub:

```csharp
using System;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    public sealed class KikToolbarButton : IDisposable
    {
        public KikToolbarButton(MonoBehaviour host, string sceneFlags, Action onClick)
        {
            // toolbar registration implemented in Task 18
        }
        public void Dispose() { }
    }
}
```

- [ ] **Step 6: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: success (warnings about unused fields are OK).

- [ ] **Step 7: Commit**

```bash
git add src/KerbalInstructionsKit/Runtime/ src/KerbalInstructionsKit/Util/IPauseController.cs src/KerbalInstructionsKit/Util/FlightDriverPauseController.cs src/KerbalInstructionsKit/Rendering/LessonPanel.cs
git commit -m "feat: add per-scene KSPAddons, pause controller, panel/toolbar stubs"
```

### Task 18: ToolbarController integration

Wire `KikToolbarButton` to `ToolbarController`. Registers stock + Blizzy, draws icon, fires onClick.

**Files:**
- Modify: `src/KerbalInstructionsKit/Runtime/KikToolbarButton.cs`
- Create: `Plugins/Icons/KIK_icon_38.png` (38×38 stock icon — placeholder OK for now)
- Create: `Plugins/Icons/KIK_icon_24.png` (24×24 Blizzy icon)

- [ ] **Step 1: Generate placeholder icons**

For now, use a solid color square. Manual / programmatic alternatives:
- Open any image editor, create 38×38 PNG with a recognizable color (e.g., RGB 100/180/220, blue), save as `Plugins/Icons/KIK_icon_38.png`. Make a 24×24 version.
- Or copy from `GameData/KerbalAdminKit/Plugins/Icons/` as a starting placeholder.

The icons can be replaced later with proper artwork.

- [ ] **Step 2: Replace `KikToolbarButton.cs` with the real implementation**

```csharp
using System;
using ToolbarControl_NS;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    public sealed class KikToolbarButton : IDisposable
    {
        private const string ModId = "KerbalInstructionsKit";
        private const string DisplayName = "Instructions";
        private const string LargeIconPath = "KerbalInstructionsKit/Plugins/Icons/KIK_icon_38";
        private const string SmallIconPath = "KerbalInstructionsKit/Plugins/Icons/KIK_icon_24";

        private readonly ToolbarControl control;

        public KikToolbarButton(MonoBehaviour host, string sceneFlags, Action onClick)
        {
            ApplicationLauncher.AppScenes scenes = ApplicationLauncher.AppScenes.NEVER;
            foreach (var s in sceneFlags.Split(','))
            {
                switch (s.Trim())
                {
                    case "SpaceCentre":     scenes |= ApplicationLauncher.AppScenes.SPACECENTER; break;
                    case "Editor":          scenes |= ApplicationLauncher.AppScenes.VAB | ApplicationLauncher.AppScenes.SPH; break;
                    case "Flight":          scenes |= ApplicationLauncher.AppScenes.FLIGHT; break;
                    case "MapFlight":       scenes |= ApplicationLauncher.AppScenes.MAPVIEW; break;
                    case "TrackingStation": scenes |= ApplicationLauncher.AppScenes.TRACKSTATION; break;
                }
            }

            control = host.gameObject.AddComponent<ToolbarControl>();
            control.AddToAllToolbars(
                onClick,
                onClick,
                scenes,
                ModId,
                "kikToolbarButton",
                LargeIconPath,
                SmallIconPath,
                DisplayName);
            control.UseBlizzy(true);
            control.UseStock(true);
        }

        public void Dispose()
        {
            if (control != null)
            {
                control.OnDestroy();
                UnityEngine.Object.Destroy(control);
            }
        }
    }
}
```

Note: ToolbarControl's API surface around scene visibility may need a small adjustment depending on the installed version. If `AppLauncher.VisibleInScenes` is not directly settable, fall back to passing `scenes` to `AddToAllToolbars`'s `appScenes` parameter (the third positional argument). Verify against `GameData/001_ToolbarControl/Plugins/ToolbarControl.dll` at implementation time.

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

If `AddToAllToolbars` overload doesn't match, inspect ToolbarControl.dll in dnSpy/ILSpy to confirm the parameter order — the canonical form takes (onTrueClick, onFalseClick, scenes, modId, buttonId, largeIconPath, smallIconPath, displayName).

- [ ] **Step 4: Commit**

```bash
git add src/KerbalInstructionsKit/Runtime/KikToolbarButton.cs Plugins/Icons/
git commit -m "feat: implement ToolbarControl registration for the KIK button"
```

### Task 19: GameEvents wiring

The scenario subscribes to KSP `GameEvents.Contracts.*` and forwards them to the trigger engine. Also subscribes to any `onGameEvent` triggers' specific events.

**Files:**
- Modify: `src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs`
- Create: `src/KerbalInstructionsKit/Triggers/GameEventBridge.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Triggers/GameEventBridge.cs`**

```csharp
using System;
using System.Collections.Generic;
using Contracts;
using UnityEngine;

namespace KerbalInstructionsKit.Triggers
{
    /// <summary>
    /// Bridges KSP GameEvents → LessonTriggerEngine. Subscribes only to events that have
    /// active triggers, so we don't pay for events nobody listens for.
    /// </summary>
    public sealed class GameEventBridge : IDisposable
    {
        private readonly LessonTriggerEngine engine;
        private readonly HashSet<string> subscribedEvents = new HashSet<string>();
        private readonly Dictionary<string, Action> handlers = new Dictionary<string, Action>();

        private bool contractsSubscribed;

        public GameEventBridge(LessonTriggerEngine engine)
        {
            this.engine = engine;
            ConfigureFromTriggers();
        }

        private void ConfigureFromTriggers()
        {
            foreach (var t in engine.All)
            {
                if (t.Kind == TriggerKind.GameEvent && !subscribedEvents.Contains(t.EventName))
                {
                    SubscribeGameEvent(t.EventName);
                    subscribedEvents.Add(t.EventName);
                }
                else if (t.Kind == TriggerKind.Contract && !contractsSubscribed)
                {
                    GameEvents.Contracts.OnOffered.Add(OnOffered);
                    GameEvents.Contracts.OnAccepted.Add(OnAccepted);
                    GameEvents.Contracts.OnCompleted.Add(OnCompleted);
                    GameEvents.Contracts.OnFailed.Add(OnFailed);
                    contractsSubscribed = true;
                }
            }
        }

        private void SubscribeGameEvent(string name)
        {
            // Reflect on the GameEvents class to find the static event with this name.
            // Most are EventVoid; we wrap with a relay to call engine.OnGameEvent(name).
            var field = typeof(GameEvents).GetField(name);
            if (field == null)
            {
                Debug.LogWarning($"[KIK] LESSON_TRIGGER onGameEvent='{name}' — not a GameEvents field. Skipping.");
                return;
            }
            var ev = field.GetValue(null);
            if (ev is EventVoid voidEv)
            {
                Action handler = () => engine.OnGameEvent(name);
                voidEv.Add(handler);
                handlers[name] = handler;
            }
            else
            {
                Debug.LogWarning($"[KIK] LESSON_TRIGGER onGameEvent='{name}' — only EventVoid is supported. Skipping.");
            }
        }

        private void OnOffered(Contract c) =>
            engine.OnContractEvent(c.GetType().Name, ContractState.Offered);
        private void OnAccepted(Contract c) =>
            engine.OnContractEvent(c.GetType().Name, ContractState.Accepted);
        private void OnCompleted(Contract c) =>
            engine.OnContractEvent(c.GetType().Name, ContractState.Completed);
        private void OnFailed(Contract c) =>
            engine.OnContractEvent(c.GetType().Name, ContractState.Failed);

        public void Dispose()
        {
            foreach (var kv in handlers)
            {
                var field = typeof(GameEvents).GetField(kv.Key);
                if (field?.GetValue(null) is EventVoid voidEv) voidEv.Remove(kv.Value);
            }
            handlers.Clear();
            if (contractsSubscribed)
            {
                GameEvents.Contracts.OnOffered.Remove(OnOffered);
                GameEvents.Contracts.OnAccepted.Remove(OnAccepted);
                GameEvents.Contracts.OnCompleted.Remove(OnCompleted);
                GameEvents.Contracts.OnFailed.Remove(OnFailed);
                contractsSubscribed = false;
            }
        }
    }
}
```

Important: `Contracts` namespace lives in CC's DLL when CC is installed AND in stock KSP via the Contracts namespace from `Assembly-CSharp`. This bridge uses stock `Contract` — no CC reference needed. (Note: `Contract` is a stock KSP class shared by both stock and CC contracts.)

- [ ] **Step 2: Modify `KerbalInstructionsKitScenario` to instantiate the bridge**

In `OnAwake`, after the `LessonContentLoader.LoadTriggers(...)` line, add:

```csharp
gameEventBridge = new GameEventBridge(TriggerEngine);
```

Add the field at class scope:

```csharp
private GameEventBridge gameEventBridge;
```

Add to `OnDestroy`:

```csharp
gameEventBridge?.Dispose();
gameEventBridge = null;
```

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

If `Contracts` namespace doesn't resolve, add `using Contracts;` and verify the `Contract` type comes from `Assembly-CSharp`. If still failing, the stock contract type lives in `Contracts.Contract` — adjust the using statement accordingly.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalInstructionsKit/Triggers/GameEventBridge.cs src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs
git commit -m "feat: wire KSP GameEvents to LessonTriggerEngine"
```

## Phase 4 — CC reflection and AttachLesson BEHAVIOUR

### Task 20: ContractConfigurator presence detection and AttachLesson registration

CC has its own `BEHAVIOUR` factory pattern. Each `BEHAVIOUR` type is a class derived from `ContractConfigurator.Behaviour.ContractBehaviour` plus a `BehaviourFactory` registration. Since KIK has a soft dep on CC, we do this entirely via reflection. If CC isn't loaded, we skip silently.

**Files:**
- Create: `src/KerbalInstructionsKit/Triggers/CcIntegration.cs`
- Create: `src/KerbalInstructionsKit/Triggers/AttachLessonBehaviour.cs`
- Create: `src/KerbalInstructionsKit/Triggers/AttachLessonFactory.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Triggers/CcIntegration.cs`**

```csharp
using System;
using System.Linq;
using System.Reflection;
using UnityEngine;

namespace KerbalInstructionsKit.Triggers
{
    public static class CcIntegration
    {
        public static bool IsAvailable { get; private set; }
        public static Assembly CcAssembly { get; private set; }

        public static void Detect()
        {
            CcAssembly = AssemblyLoader.loadedAssemblies
                .FirstOrDefault(a => a.name == "ContractConfigurator")
                ?.assembly;
            IsAvailable = CcAssembly != null;
            if (IsAvailable)
                Debug.Log("[KIK] ContractConfigurator detected; AttachLesson BEHAVIOUR enabled");
            else
                Debug.Log("[KIK] ContractConfigurator not present; AttachLesson BEHAVIOUR disabled. Use LESSON_TRIGGER for unlock control.");
        }

        public static Type GetCcType(string fullName) =>
            CcAssembly?.GetType(fullName);
    }
}
```

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Triggers/AttachLessonBehaviour.cs`**

This is the runtime BEHAVIOUR instance. Created per contract. Subscribes to its parent contract's state changes and unlocks the lesson at the configured state. Implemented as a generic-event handler so KIK doesn't need to compile against CC's base class — instead we register a factory that *constructs* an instance, but the actual BEHAVIOUR lifecycle is owned by CC. We use reflection to derive from CC's base class.

```csharp
using System;
using System.Reflection;
using KerbalInstructionsKit.Core;
using UnityEngine;

namespace KerbalInstructionsKit.Triggers
{
    /// <summary>
    /// Carrier for AttachLesson cfg parameters. Behaviour state for the linked contract
    /// is managed by CC; KIK only needs to know the lesson id and unlock state. The
    /// factory in AttachLessonFactory.cs subscribes to contract state events and unlocks
    /// when the right transition happens.
    /// </summary>
    public sealed class AttachLessonBinding
    {
        public string LessonId;
        public ContractState UnlockOn;
        public bool ShowButton;
    }

    public static class AttachLessonRegistry
    {
        // contractTypeName → list of attached lessons
        private static readonly System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<AttachLessonBinding>> byContract =
            new System.Collections.Generic.Dictionary<string, System.Collections.Generic.List<AttachLessonBinding>>();

        public static void Register(string contractTypeName, AttachLessonBinding b)
        {
            if (!byContract.TryGetValue(contractTypeName, out var list))
                byContract[contractTypeName] = list = new System.Collections.Generic.List<AttachLessonBinding>();
            list.Add(b);
        }

        public static System.Collections.Generic.IList<AttachLessonBinding> Get(string contractTypeName) =>
            byContract.TryGetValue(contractTypeName, out var list)
                ? (System.Collections.Generic.IList<AttachLessonBinding>)list
                : Array.Empty<AttachLessonBinding>();
    }
}
```

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Triggers/AttachLessonFactory.cs`**

Use reflection to register a custom CC `BehaviourFactory` for type name `AttachLesson`. The factory pulls cfg fields, populates an `AttachLessonBinding`, and registers it with `AttachLessonRegistry`. Then it subscribes via the `GameEventBridge` (which already listens to contract events) to drive unlocks through the trigger engine.

```csharp
using System;
using System.Reflection;
using KerbalInstructionsKit.Core;
using UnityEngine;

namespace KerbalInstructionsKit.Triggers
{
    public static class AttachLessonFactory
    {
        public static void TryRegister(LessonTriggerEngine engine)
        {
            if (!CcIntegration.IsAvailable) return;
            try
            {
                RegisterBehaviourFactoryViaReflection(engine);
            }
            catch (Exception e)
            {
                Debug.LogError($"[KIK] failed to register AttachLesson with CC: {e}");
            }
        }

        private static void RegisterBehaviourFactoryViaReflection(LessonTriggerEngine engine)
        {
            // CC's class is `ContractConfigurator.Behaviour.BehaviourFactory`, with
            // a static method `Register<T>(string name)` where T : BehaviourFactory.
            // Implementing a custom factory requires deriving from BehaviourFactory and
            // overriding Generate / Load methods.
            //
            // Cleanest approach: inject our own factory subclass at runtime via System.Reflection.Emit.
            // Simpler approach used here: register an existing CC factory (if any) is not feasible.
            //
            // Instead: use CC's "ConfigNodeUtil" + a contract-scope hook to scan all CONTRACT_TYPE
            // nodes for a BEHAVIOUR with type=AttachLesson, populate AttachLessonRegistry, and
            // emit equivalent LESSON_TRIGGER entries into the engine. CC's own loader will then
            // emit a warning for "unknown BEHAVIOUR type=AttachLesson", which we suppress by
            // registering a no-op factory class via reflection.
            //
            // The simplest path that works without compile-time CC reference is the
            // "synthetic LESSON_TRIGGER" path:
            ScanAndSynthesizeTriggers(engine);
            RegisterNoOpFactoryStub();
        }

        private static void ScanAndSynthesizeTriggers(LessonTriggerEngine engine)
        {
            int generated = 0;
            foreach (var url in GameDatabase.Instance.GetConfigs("CONTRACT_TYPE"))
            {
                var cfg = url.config;
                var contractName = cfg.GetValue("name");
                if (string.IsNullOrEmpty(contractName)) continue;

                foreach (ConfigNode behaviour in cfg.GetNodes("BEHAVIOUR"))
                {
                    if (behaviour.GetValue("type") != "AttachLesson") continue;

                    var lessonId = behaviour.GetValue("lesson");
                    if (string.IsNullOrEmpty(lessonId))
                    {
                        Debug.LogWarning($"[KIK] AttachLesson on contract '{contractName}' missing 'lesson', skipping");
                        continue;
                    }

                    var unlockOnStr = behaviour.GetValue("unlockOn") ?? "OFFERED";
                    var unlockOn = ParseState(unlockOnStr);

                    bool showButton = true;
                    if (behaviour.HasValue("showButton") &&
                        bool.TryParse(behaviour.GetValue("showButton"), out var sb))
                        showButton = sb;

                    var binding = new AttachLessonBinding
                    {
                        LessonId = lessonId,
                        UnlockOn = unlockOn,
                        ShowButton = showButton,
                    };
                    AttachLessonRegistry.Register(contractName, binding);

                    engine.Register(new LessonTrigger
                    {
                        Kind = TriggerKind.Contract,
                        ContractName = contractName,
                        ContractState = unlockOn,
                        LessonId = lessonId,
                    });
                    generated++;
                }
            }
            Debug.Log($"[KIK] generated {generated} contract-attached lesson triggers from AttachLesson BEHAVIOURs");
        }

        private static ContractState ParseState(string s)
        {
            switch ((s ?? "").ToUpperInvariant())
            {
                case "ACCEPTED":  return ContractState.Accepted;
                case "COMPLETED": return ContractState.Completed;
                case "FAILED":    return ContractState.Failed;
                default:          return ContractState.Offered;
            }
        }

        private static void RegisterNoOpFactoryStub()
        {
            // CC will warn about unknown BEHAVIOUR type=AttachLesson when validating
            // contract types. To suppress this we register a no-op factory using CC's
            // BehaviourFactory.Register API, located via reflection.
            try
            {
                var factoryType = CcIntegration.GetCcType("ContractConfigurator.Behaviour.BehaviourFactory");
                if (factoryType == null) return;

                var register = factoryType.GetMethod("Register", BindingFlags.Public | BindingFlags.Static);
                if (register == null || !register.IsGenericMethod)
                {
                    Debug.LogWarning("[KIK] could not locate CC BehaviourFactory.Register<T>");
                    return;
                }

                // NoOpAttachLessonBehaviour must derive from CC's BehaviourFactory at runtime.
                // Generating that subclass via reflection alone is not feasible without
                // System.Reflection.Emit machinery. As a simplification: we accept the CC
                // warning at load time. Players see one informational warning per contract
                // with AttachLesson, which is acceptable for v0.1.
                Debug.Log("[KIK] AttachLesson is consumed via cfg-scan; expect one CC warning per contract using AttachLesson (cosmetic, not a problem).");
            }
            catch (Exception e)
            {
                Debug.LogWarning($"[KIK] no-op factory stub failed: {e.Message}");
            }
        }
    }
}
```

Note: Producing a fully-CC-native factory class without compile-time CC reference would require `System.Reflection.Emit`. For v0.1, the cfg-scan approach gives identical behavior with a single cosmetic CC warning per contract. Accept this trade-off; revisit in a later release.

- [ ] **Step 4: Modify scenario `OnAwake` to detect CC and register AttachLesson**

In `KerbalInstructionsKitScenario.OnAwake`, after `LessonContentLoader.LoadTriggers(TriggerEngine);`, add:

```csharp
CcIntegration.Detect();
AttachLessonFactory.TryRegister(TriggerEngine);
```

- [ ] **Step 5: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

- [ ] **Step 6: Commit**

```bash
git add src/KerbalInstructionsKit/Triggers/CcIntegration.cs src/KerbalInstructionsKit/Triggers/AttachLessonBehaviour.cs src/KerbalInstructionsKit/Triggers/AttachLessonFactory.cs src/KerbalInstructionsKit/Runtime/KerbalInstructionsKitScenario.cs
git commit -m "feat: detect ContractConfigurator and synthesize triggers from AttachLesson BEHAVIOUR"
```

---

## Phase 5 — IMGUI rendering

### Task 21: LessonPanel — window scaffold and toggle/open/close

Build the IMGUI window using ClickThruBlocker. Implement Open/Close/Toggle, position persistence, and the view-state machine wiring. Drawing the actual content comes in Tasks 22-23.

**Files:**
- Modify: `src/KerbalInstructionsKit/Rendering/LessonPanel.cs`
- Create: `src/KerbalInstructionsKit/Rendering/PanelStyles.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Rendering/PanelStyles.cs`**

```csharp
using UnityEngine;

namespace KerbalInstructionsKit.Rendering
{
    public static class PanelStyles
    {
        public static GUIStyle RichLabel;
        public static GUIStyle Title;
        public static GUIStyle PageTitle;
        public static GUIStyle Caption;
        public static GUIStyle LinkButton;
        public static GUIStyle CategoryHeader;
        public static GUIStyle ChipOn;
        public static GUIStyle ChipOff;
        public static GUIStyle TocItem;
        public static GUIStyle TocItemActive;

        private static bool initialized;

        public static void Init()
        {
            if (initialized) return;
            RichLabel = new GUIStyle(GUI.skin.label) { richText = true, wordWrap = true };
            Title = new GUIStyle(GUI.skin.label) { richText = true, fontSize = 18, fontStyle = FontStyle.Bold };
            PageTitle = new GUIStyle(GUI.skin.label) { richText = true, fontStyle = FontStyle.Italic };
            Caption = new GUIStyle(GUI.skin.label) { richText = true, alignment = TextAnchor.MiddleCenter, fontStyle = FontStyle.Italic, wordWrap = true };
            LinkButton = new GUIStyle(GUI.skin.button) { richText = true };
            CategoryHeader = new GUIStyle(GUI.skin.label) { fontStyle = FontStyle.Bold, fontSize = 14 };
            ChipOn = new GUIStyle(GUI.skin.button) { fontSize = 11, padding = new RectOffset(8, 8, 2, 2) };
            ChipOff = new GUIStyle(ChipOn);
            TocItem = new GUIStyle(GUI.skin.label) { richText = true };
            TocItemActive = new GUIStyle(TocItem) { fontStyle = FontStyle.Bold };
            initialized = true;
        }
    }
}
```

- [ ] **Step 2: Replace `src/KerbalInstructionsKit/Rendering/LessonPanel.cs` with full window scaffold**

```csharp
using System;
using ClickThroughFix;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Runtime;
using KerbalInstructionsKit.Util;
using UnityEngine;

namespace KerbalInstructionsKit.Rendering
{
    public sealed class LessonPanel : IDisposable
    {
        private const int WindowId = 0x4B49_4B00;

        private readonly KerbalInstructionsKitScenario scenario;
        private readonly IPauseController pause;
        private readonly PanelStateMachine stateMachine;
        private readonly ArchiveQuery query = new ArchiveQuery();

        private bool visible;
        private Rect window = new Rect(60, 60, 900, 640);
        private Vector2 lessonScroll, archiveScroll;

        private bool wasPausedOnOpen;
        private bool kikPausedFlag;

        public LessonPanel(KerbalInstructionsKitScenario scenario, IPauseController pause)
        {
            this.scenario = scenario;
            this.pause = pause;
            stateMachine = new PanelStateMachine(scenario.Lessons);
        }

        public void Toggle()
        {
            if (visible) Close();
            else OpenDefault();
        }

        public void OpenDefault()
        {
            if (!string.IsNullOrEmpty(scenario.State.LastViewedLesson)
                && scenario.Lessons.Contains(scenario.State.LastViewedLesson))
            {
                stateMachine.OpenLesson(scenario.State.LastViewedLesson);
                stateMachine.JumpToPage(scenario.State.LastViewedPage);
            }
            else
            {
                stateMachine.OpenArchive();
            }
            Show();
        }

        public void OpenLesson(string id)
        {
            stateMachine.OpenLesson(id);
            Show();
        }

        public void OpenArchive()
        {
            stateMachine.OpenArchive();
            Show();
        }

        private void Show()
        {
            wasPausedOnOpen = pause.IsAvailable && pause.IsPaused;
            kikPausedFlag = false;
            visible = true;
        }

        public void Close()
        {
            if (kikPausedFlag && pause.IsAvailable)
                pause.SetPaused(wasPausedOnOpen);
            visible = false;

            // persist last viewed
            if (stateMachine.CurrentLesson != null)
            {
                scenario.State.LastViewedLesson = stateMachine.CurrentLesson.Id;
                scenario.State.LastViewedPage = stateMachine.CurrentPage;
            }
        }

        public void Draw()
        {
            if (!visible) return;
            PanelStyles.Init();
            window = ClickThruBlocker.GUILayoutWindow(WindowId, window, DrawWindow, "Kerbal Instructions Kit");
        }

        private void DrawWindow(int id)
        {
            if (stateMachine.View == PanelView.Lesson)
                DrawLessonView();
            else
                DrawArchiveView();
            GUI.DragWindow(new Rect(0, 0, window.width, 24));
        }

        private void DrawLessonView() { /* implemented in Task 22 */ }
        private void DrawArchiveView() { /* implemented in Task 23 */ }

        public void Dispose() { visible = false; }
    }
}
```

- [ ] **Step 3: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: success.

- [ ] **Step 4: Commit**

```bash
git add src/KerbalInstructionsKit/Rendering/
git commit -m "feat: add LessonPanel window scaffold with Open/Close/Toggle"
```

### Task 22: Lesson view rendering

**Files:**
- Modify: `src/KerbalInstructionsKit/Rendering/LessonPanel.cs`
- Create: `src/KerbalInstructionsKit/Rendering/LinkRouter.cs`
- Create: `src/KerbalInstructionsKit/Rendering/ImageCache.cs`

- [ ] **Step 1: Write `src/KerbalInstructionsKit/Rendering/ImageCache.cs`**

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace KerbalInstructionsKit.Rendering
{
    public static class ImageCache
    {
        private static readonly Dictionary<string, Texture2D> hits = new Dictionary<string, Texture2D>();
        private static readonly HashSet<string> misses = new HashSet<string>();

        public static Texture2D Get(string url)
        {
            if (string.IsNullOrEmpty(url)) return null;
            if (hits.TryGetValue(url, out var tex)) return tex;
            if (misses.Contains(url)) return null;
            tex = GameDatabase.Instance.GetTexture(url, false);
            if (tex == null)
            {
                Debug.LogWarning($"[KIK] image '{url}' not found in GameDatabase");
                misses.Add(url);
                return null;
            }
            hits[url] = tex;
            return tex;
        }
    }
}
```

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Rendering/LinkRouter.cs`**

```csharp
using System;
using KerbalInstructionsKit.Core;
using UnityEngine;

namespace KerbalInstructionsKit.Rendering
{
    public sealed class LinkRouter
    {
        private readonly Action<string> openLessonViaLink;

        public LinkRouter(Action<string> openLessonViaLink)
        {
            this.openLessonViaLink = openLessonViaLink;
        }

        public void Activate(Link link)
        {
            switch (link.Type)
            {
                case LinkType.Lesson:
                    openLessonViaLink?.Invoke(link.Target);
                    break;
                case LinkType.Kspedia:
                    OpenKspedia(link.Target);
                    break;
                case LinkType.Url:
                    OpenUrl(link.Target);
                    break;
            }
        }

        private void OpenKspedia(string page)
        {
            try { KSPedia.KSPediaSpawner.SpawnKSPedia(); }
            catch (Exception e) { Debug.LogWarning($"[KIK] failed to open KSPedia ('{page}'): {e.Message}"); }
        }

        private void OpenUrl(string url)
        {
            try { Application.OpenURL(url); }
            catch (Exception e) { Debug.LogWarning($"[KIK] failed to open URL '{url}': {e.Message}"); }
        }
    }
}
```

Note on `KSPedia.KSPediaSpawner.SpawnKSPedia()`: the exact API is `KSPedia.KSPediaSpawner.SpawnKSPedia()` (no parameters; opens KSPedia at last viewed page). KSP doesn't expose a "open at specific page" API — the `target` value is logged for future-proofing but isn't currently used. If your KSP version's class lives elsewhere, locate it via `Assembly-CSharp.dll` reflection.

- [ ] **Step 3: Replace `DrawLessonView` body in `LessonPanel.cs`**

Add fields:

```csharp
private LinkRouter linkRouter;
```

In the constructor, after `stateMachine = ...`:

```csharp
linkRouter = new LinkRouter(stateMachine.NavigateToLessonViaLink);
```

Replace `DrawLessonView`:

```csharp
private void DrawLessonView()
{
    var lesson = stateMachine.CurrentLesson;
    if (lesson == null)
    {
        stateMachine.OpenArchive();
        return;
    }

    GUILayout.BeginVertical();

    // Header row
    GUILayout.BeginHorizontal();
    if (stateMachine.CanGoBack)
    {
        if (GUILayout.Button("← Back", GUILayout.Width(80)))
            stateMachine.GoBack();
    }
    GUILayout.Label(lesson.Title, PanelStyles.Title);
    GUILayout.FlexibleSpace();
    if (pause.IsAvailable)
    {
        var label = pause.IsPaused && kikPausedFlag ? "▶ Resume" : "⏸ Pause";
        if (GUILayout.Button(label, GUILayout.Width(100)))
        {
            if (kikPausedFlag) { pause.SetPaused(false); kikPausedFlag = false; }
            else               { pause.SetPaused(true);  kikPausedFlag = true;  }
        }
    }
    if (GUILayout.Button("✕", GUILayout.Width(28))) Close();
    GUILayout.EndHorizontal();

    var page = lesson.Pages[stateMachine.CurrentPage];

    if (!string.IsNullOrEmpty(page.Title))
        GUILayout.Label(page.Title, PanelStyles.PageTitle);

    lessonScroll = GUILayout.BeginScrollView(lessonScroll);

    if (!string.IsNullOrEmpty(page.Image))
    {
        var tex = ImageCache.Get(page.Image);
        if (tex != null)
        {
            float w = Mathf.Min(600, tex.width);
            float h = w * tex.height / Mathf.Max(1, tex.width);
            var r = GUILayoutUtility.GetRect(w, h, GUILayout.Width(w), GUILayout.Height(h));
            // Center horizontally
            r.x = (window.width - w) / 2;
            GUI.DrawTexture(r, tex, ScaleMode.ScaleToFit);
            GUILayout.Space(h);
        }
        if (!string.IsNullOrEmpty(page.Caption))
            GUILayout.Label(page.Caption, PanelStyles.Caption);
    }

    if (!string.IsNullOrEmpty(page.Text))
        GUILayout.Label(page.Text.Replace("\\n", "\n"), PanelStyles.RichLabel);

    if (page.Links.Count > 0)
    {
        GUILayout.Space(8);
        GUILayout.BeginHorizontal();
        foreach (var link in page.Links)
        {
            string label = LabelFor(link);
            bool enabled = IsLinkEnabled(link);
            GUI.enabled = enabled;
            if (GUILayout.Button(label, PanelStyles.LinkButton))
                linkRouter.Activate(link);
            GUI.enabled = true;
        }
        GUILayout.EndHorizontal();
    }

    GUILayout.EndScrollView();

    // Page navigation
    GUILayout.BeginHorizontal();
    GUI.enabled = stateMachine.CurrentPage > 0;
    if (GUILayout.Button("◀ Prev", GUILayout.Width(100))) stateMachine.PrevPage();
    GUI.enabled = true;
    GUILayout.FlexibleSpace();
    GUILayout.Label($"Page {stateMachine.CurrentPage + 1} of {lesson.Pages.Count}");
    GUILayout.FlexibleSpace();
    GUI.enabled = stateMachine.CurrentPage < lesson.Pages.Count - 1;
    if (GUILayout.Button("Next ▶", GUILayout.Width(100))) stateMachine.NextPage();
    GUI.enabled = true;
    GUILayout.EndHorizontal();

    GUILayout.Space(6);
    if (GUILayout.Button("📚  All Lessons", GUILayout.Height(28)))
        stateMachine.OpenArchive();

    GUILayout.EndVertical();
}

private string LabelFor(Link link)
{
    var glyph = link.Type == LinkType.Lesson ? "→ "
              : link.Type == LinkType.Kspedia ? "↗ "
              : "↗ ";
    var arrow = link.Type == LinkType.Url ? " ↗" : "";
    return glyph + (link.Label ?? link.Target) + arrow;
}

private bool IsLinkEnabled(Link link)
{
    if (link.Type != LinkType.Lesson) return true;
    return scenario.Lessons.Contains(link.Target)
        && scenario.State.IsUnlocked(link.Target);
}
```

- [ ] **Step 4: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Expected: success. If `KSPedia.KSPediaSpawner` namespace differs, search `Assembly-CSharp.dll` for the actual type and adjust `LinkRouter.OpenKspedia`.

- [ ] **Step 5: Commit**

```bash
git add src/KerbalInstructionsKit/Rendering/
git commit -m "feat: render lesson view with image, text, links, page nav, pause"
```

### Task 23: Archive view rendering

**Files:**
- Modify: `src/KerbalInstructionsKit/Rendering/LessonPanel.cs`

- [ ] **Step 1: Replace `DrawArchiveView`**

```csharp
private void DrawArchiveView()
{
    GUILayout.BeginVertical();

    GUILayout.BeginHorizontal();
    GUILayout.Label("All Lessons", PanelStyles.Title);
    GUILayout.FlexibleSpace();
    if (pause.IsAvailable)
    {
        var label = pause.IsPaused && kikPausedFlag ? "▶ Resume" : "⏸ Pause";
        if (GUILayout.Button(label, GUILayout.Width(100)))
        {
            if (kikPausedFlag) { pause.SetPaused(false); kikPausedFlag = false; }
            else               { pause.SetPaused(true);  kikPausedFlag = true;  }
        }
    }
    if (GUILayout.Button("✕", GUILayout.Width(28))) Close();
    GUILayout.EndHorizontal();

    // Search
    GUILayout.BeginHorizontal();
    GUILayout.Label("🔎", GUILayout.Width(20));
    query.Search = GUILayout.TextField(query.Search ?? "");
    if (GUILayout.Button("Clear", GUILayout.Width(60))) query.Search = "";
    GUILayout.EndHorizontal();

    // Tag chips
    var allTags = ArchiveFilter.AllVisibleTags(scenario.Lessons, scenario.ExpressionContext);
    if (allTags.Count > 0)
    {
        GUILayout.BeginHorizontal();
        foreach (var tag in allTags)
        {
            bool selected = query.SelectedTags.Contains(tag);
            var label = selected ? "[x] " + tag : "[ ] " + tag;
            if (GUILayout.Button(label, selected ? PanelStyles.ChipOn : PanelStyles.ChipOff))
            {
                if (selected) query.SelectedTags.Remove(tag);
                else query.SelectedTags.Add(tag);
            }
        }
        GUILayout.EndHorizontal();
    }

    archiveScroll = GUILayout.BeginScrollView(archiveScroll);
    var groups = ArchiveFilter.Apply(scenario.Lessons, scenario.ExpressionContext, query);

    if (groups.Count == 0)
    {
        GUILayout.Label("No lessons unlocked yet — keep playing!", PanelStyles.RichLabel);
    }
    else
    {
        foreach (var group in groups)
        {
            GUILayout.Space(4);
            GUILayout.Label(group.Category, PanelStyles.CategoryHeader);
            foreach (var lesson in group.Lessons)
            {
                bool isLast = stateMachine.CurrentLesson?.Id == lesson.Id;
                var label = isLast ? "  ● " + lesson.Title + "  ◀ last viewed"
                                   : "  · " + lesson.Title;
                if (GUILayout.Button(label, isLast ? PanelStyles.TocItemActive : PanelStyles.TocItem,
                                     GUILayout.ExpandWidth(true)))
                {
                    stateMachine.OpenLesson(lesson.Id);
                }
            }
        }
    }
    GUILayout.EndScrollView();

    if (stateMachine.CurrentLesson != null)
    {
        if (GUILayout.Button("◀ Back to Lesson", GUILayout.Height(26)))
            stateMachine.BackToLesson();
    }

    GUILayout.EndVertical();
}
```

- [ ] **Step 2: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

- [ ] **Step 3: Commit**

```bash
git add src/KerbalInstructionsKit/Rendering/LessonPanel.cs
git commit -m "feat: render archive view with search, tag chips, grouped lessons"
```

## Phase 6 — Contract-window button injection

### Task 24: Harmony patch for contract details

KSP's contract details panel renders inside the Mission Control screen (which appears as a sub-screen of SpaceCenter). The button needs to appear next to the contract title when the selected contract has an `AttachLesson` BEHAVIOUR with `showButton = true`.

**Approach:** Harmony postfix on `MissionControl.OnContractsListChanged` (or the contract detail panel's render method). Walks the visible contract panel, finds the title row, injects a button. The exact target method must be discovered at implementation time by inspecting `Assembly-CSharp.dll` (use ILSpy or dnSpy on the user's KSP install).

**Files:**
- Create: `src/KerbalInstructionsKit/Runtime/ContractDetailsPatch.cs`
- Create: `src/KerbalInstructionsKit/Runtime/HarmonyBootstrap.cs`

- [ ] **Step 1: Inspect KSP's contract detail view**

```bash
# Install ILSpy or dnSpy. Open:
# C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp.dll
# Look in namespace KSP.UI.Screens (or similar) for: MissionControl, ContractDetail, GenericCascadingList, etc.
# Record the exact class and method name for the "render contract details" code path.
```

Document the discovered names in a comment at the top of `ContractDetailsPatch.cs`. Common candidates:
- `KSP.UI.Screens.MissionControl.UpdateInfoPanelContract`
- `KSP.UI.Screens.MissionControl.OnContractSelected`
- `KSP.UI.Screens.MissionControl.RebuildContractList`

Pick the one that runs after the contract title GameObject is built and exposes a reference we can attach to.

- [ ] **Step 2: Write `src/KerbalInstructionsKit/Runtime/HarmonyBootstrap.cs`**

```csharp
using HarmonyLib;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    [KSPAddon(KSPAddon.Startup.Instantly, true)]
    public sealed class HarmonyBootstrap : MonoBehaviour
    {
        private void Start()
        {
            try
            {
                var harmony = new Harmony("badgkat.kerbalinstructionskit");
                harmony.PatchAll(typeof(HarmonyBootstrap).Assembly);
                Debug.Log("[KIK] Harmony patches applied");
            }
            catch (System.Exception e)
            {
                Debug.LogError($"[KIK] Harmony bootstrap failed: {e}");
            }
        }
    }
}
```

- [ ] **Step 3: Write `src/KerbalInstructionsKit/Runtime/ContractDetailsPatch.cs`**

Skeleton (the target type name fills in once Step 1 is complete). Replace `KSP.UI.Screens.MissionControl` and `OnContractsListChanged` if you found different names.

```csharp
using System;
using Contracts;
using HarmonyLib;
using KerbalInstructionsKit.Core;
using KerbalInstructionsKit.Triggers;
using UnityEngine;

namespace KerbalInstructionsKit.Runtime
{
    [HarmonyPatch(typeof(KSP.UI.Screens.MissionControl), "OnContractsListChanged")]
    public static class ContractDetailsPatch
    {
        public static void Postfix(KSP.UI.Screens.MissionControl __instance)
        {
            try { TryInjectButton(__instance); }
            catch (Exception e) { Debug.LogWarning($"[KIK] contract-window patch error: {e.Message}"); }
        }

        private static void TryInjectButton(KSP.UI.Screens.MissionControl mc)
        {
            // Find currently-selected contract. The 'selectedContract' field name may
            // differ between KSP versions — verify in Step 1 inspection.
            var contractField = AccessTools.Field(typeof(KSP.UI.Screens.MissionControl), "selectedContract");
            var selected = contractField?.GetValue(mc) as Contract;
            if (selected == null) return;

            var typeName = selected.GetType().Name;
            var bindings = AttachLessonRegistry.Get(typeName);
            if (bindings == null || bindings.Count == 0) return;

            foreach (var binding in bindings)
            {
                if (!binding.ShowButton) continue;
                if (!InstructionsKit.State.IsUnlocked(binding.LessonId)) continue;

                // Locate the title row Transform on the contract details panel and
                // inject a Unity UI Button as a sibling. Exact path verified during
                // implementation.
                InjectButtonForLesson(mc, binding.LessonId);
                break; // only show one button even if multiple AttachLesson blocks exist
            }
        }

        private static void InjectButtonForLesson(KSP.UI.Screens.MissionControl mc, string lessonId)
        {
            // Pseudo-code — fill in once Step 1 inspection identifies the title row.
            // Find `mc.transform`/`mc.contractTitle` (or whatever the field is named).
            // Avoid double-injection: check if a button with name "kik_lesson_btn" already
            // exists as a child; if so, just update its onClick handler. Otherwise create.
            //
            // Simplest fallback: set a flag in KIK that the next OnGUI in KikKscAddon draws
            // an IMGUI button overlaying the contract details position. Less elegant but
            // independent of KSP UI internals.
            Debug.Log($"[KIK] would inject Instructions button for lesson '{lessonId}' on " +
                      $"contract '{mc.GetType().Name}' — implementation pending UI inspection");
        }
    }
}
```

Note: If the Unity UI injection path proves brittle (KSP version drift), fall back to the IMGUI overlay approach: draw a small `GUI.Button` from `KikKscAddon.OnGUI` that's positioned to overlap the contract title bar in Mission Control. Less elegant but more robust. Decide at implementation time based on inspection findings.

- [ ] **Step 4: Build**

```bash
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
```

Note: `KSP.UI.Screens.MissionControl` is in `Assembly-CSharp.dll`. If the namespace differs, adjust.

- [ ] **Step 5: Commit (the patch is a stub at this stage; a follow-on task wires it up after manual inspection)**

```bash
git add src/KerbalInstructionsKit/Runtime/HarmonyBootstrap.cs src/KerbalInstructionsKit/Runtime/ContractDetailsPatch.cs
git commit -m "feat: add Harmony bootstrap and contract-details patch skeleton"
```

### Task 25: Wire the contract-window button (after KSP UI inspection)

**Files:**
- Modify: `src/KerbalInstructionsKit/Runtime/ContractDetailsPatch.cs`

- [ ] **Step 1: Identify the KSP contract title GameObject path**

In ILSpy/dnSpy, find the prefab path or runtime hierarchy. Typical Unity UI pattern is:
- `MissionControl` MonoBehaviour holds a reference to a `selectedContract` UI panel.
- The panel has a `TextMeshProUGUI` or `Text` component for the contract title.
- We can attach a button as a sibling of the title.

Document the exact field/transform path in a code comment.

- [ ] **Step 2: Implement `InjectButtonForLesson`**

Replace the stub with concrete code that:
1. Walks `__instance.transform` to find the title row (use `transform.Find("path/to/title")` once you know the path).
2. Checks for an existing button named `"kik_lesson_btn"`; if present, just rebind its `onClick` to the new lesson ID.
3. Otherwise instantiates a new button (clone an existing button in the contract details panel, e.g., the "Accept" button, then strip its components down to the basics).
4. Sets the button text to `"📖 Instructions"`.
5. Adds an `onClick` handler: `() => InstructionsKit.OpenLesson(lessonId);`

If the prefab cloning path is too brittle, use the IMGUI fallback: have `KikKscAddon.OnGUI` draw a `GUI.Button` at fixed screen coordinates when Mission Control is the active screen and the selected contract has an unlocked lesson. The fallback adds a `MissionControl.Instance != null` gate.

- [ ] **Step 3: Manual KSP test**

1. Build, copy DLL to `Plugins/`.
2. Launch KSP with a save that has `BKEX_FirstScience` available + a matching `INSTRUCTION_LESSON` cfg attached via AttachLesson.
3. Open Mission Control, click the contract.
4. Verify the "📖 Instructions" button appears.
5. Click it; verify the lesson opens to page 1.

- [ ] **Step 4: Commit the working patch**

```bash
git add src/KerbalInstructionsKit/Runtime/ContractDetailsPatch.cs
git commit -m "feat: inject Instructions button on contract details panel"
```

---

## Phase 7 — Polish, packaging, manual smoke test

### Task 26: Toolbar icons (real artwork)

**Files:**
- Replace: `Plugins/Icons/KIK_icon_38.png` (38×38, transparent background, blue)
- Replace: `Plugins/Icons/KIK_icon_24.png` (24×24, same design)

- [ ] **Step 1: Create or commission icons**

Suggested motif: an open book with a Kerbal-green tint, or a "?" inside a circle. Solid color for visibility against KSP's stock toolbar. PNG, transparent background.

- [ ] **Step 2: Verify icons load by launching KSP and confirming the toolbar button appears**

- [ ] **Step 3: Commit**

```bash
git add Plugins/Icons/
git commit -m "art: add KIK toolbar icons (38px stock, 24px Blizzy)"
```

### Task 27: Dev test pack

A small cfg pack devs use to smoke-test KIK without needing a full campaign. Lives outside the loaded GameData path (`.cfg.example` extension).

**Files:**
- Create: `dev/test-pack/KIK_TestPack/lessons.cfg.example`
- Create: `dev/test-pack/KIK_TestPack/triggers.cfg.example`
- Create: `dev/test-pack/README.md`

- [ ] **Step 1: Write `dev/test-pack/KIK_TestPack/lessons.cfg.example`**

```cfg
INSTRUCTION_LESSON
{
    id = LSN_TestRocketBasics
    title = Rocket Basics
    category = Flight Basics
    sortOrder = 100
    tags = Beginner, Rockets
    PAGE
    {
        title = Stages
        text = Press <b>space</b> to fire your first stage. Watch your fuel gauge.\nHigher stages light when previous ones run dry.
    }
    PAGE
    {
        title = Throttle
        text = Use <b>shift</b> to throttle up, <b>ctrl</b> to throttle down.
    }
}

INSTRUCTION_LESSON
{
    id = LSN_TestManeuver
    title = Maneuver Nodes
    category = Flight Basics
    sortOrder = 200
    tags = Beginner, Maneuver
    PAGE
    {
        text = Click the gauge above the navball, then drag the prograde handle to plan a burn.
        LINK { type = lesson; target = LSN_TestRocketBasics; label = Rocket Basics }
        LINK { type = url; target = https://wiki.kerbalspaceprogram.com/wiki/Maneuver_node; label = KSP Wiki }
    }
}

INSTRUCTION_LESSON
{
    id = LSN_TestAdvanced
    title = Advanced Topic (visibleIf demo)
    category = Advanced
    tags = Advanced
    visibleIf = LSN_TestRocketBasics.unlocked
    PAGE { text = This appears only after Rocket Basics is unlocked. }
}
```

- [ ] **Step 2: Write `dev/test-pack/KIK_TestPack/triggers.cfg.example`**

```cfg
LESSON_TRIGGER
{
    lesson = LSN_TestRocketBasics
    onGameStart = true
}

LESSON_TRIGGER
{
    lesson = LSN_TestManeuver
    onGameEvent = onLaunch
}
```

- [ ] **Step 3: Write `dev/test-pack/README.md`**

```markdown
# KIK Test Pack

Smoke-test cfg for Kerbal Instructions Kit. Files use `.cfg.example` extension so KSP's
GameDatabase scan doesn't auto-load them from the dev directory.

## To use

```bash
# from the KerbalInstructionsKit repo root:
mkdir -p "/path/to/GameData/KIK_TestPack"
cp dev/test-pack/KIK_TestPack/lessons.cfg.example   "/path/to/GameData/KIK_TestPack/lessons.cfg"
cp dev/test-pack/KIK_TestPack/triggers.cfg.example  "/path/to/GameData/KIK_TestPack/triggers.cfg"
```

Launch KSP. The toolbar button should appear; clicking it shows three lessons (with the
third gated by `visibleIf`).
```

- [ ] **Step 4: Commit**

```bash
git add dev/
git commit -m "dev: add smoke-test cfg pack with .cfg.example extension"
```

### Task 28: README, .ckan metadata, version file

**Files:**
- Modify: `README.md`
- Create: `KerbalInstructionsKit.version`
- Create: `KerbalInstructionsKit.ckan` (optional — only if CKAN-publishing immediately)

- [ ] **Step 1: Expand `README.md`**

Replace the skeleton with full documentation:

```markdown
# Kerbal Instructions Kit (KIK)

**Content-driven instructional panel framework for KSP 1.12.x.** Lets campaign mods
ship multi-page lessons (text + image + cross-links) accessible from the toolbar in
every scene plus a contextual button on the contract details view. Inert without cfg
content.

## Status

v0.1.0

## Dependencies

| Mod | Tier |
|---|---|
| ToolbarController | Hard |
| ClickThroughBlocker | Hard |
| 000_Harmony | Hard |
| ContractConfigurator | Soft (optional — without it, use `LESSON_TRIGGER` for unlock control) |

## Authoring

See [the design spec](https://github.com/badgkat/BadgKatCareer/blob/master/docs/superpowers/specs/2026-04-29-instructions-kit-design.md)
for the full cfg grammar.

Quick example:

\`\`\`cfg
INSTRUCTION_LESSON
{
    id = MYMOD_LSN_Welcome
    title = Welcome
    category = Getting Started
    PAGE { text = Hello, world! }
}

LESSON_TRIGGER
{
    lesson = MYMOD_LSN_Welcome
    onGameStart = true
}
\`\`\`

For developer testing, see `dev/test-pack/README.md`.

## Public API (C#)

\`\`\`csharp
KerbalInstructionsKit.Core.InstructionsKit.OpenLesson("MYMOD_LSN_Welcome");
KerbalInstructionsKit.Core.InstructionsKit.OpenArchive();
KerbalInstructionsKit.Core.InstructionsKit.SetFlag("FlagName", true);
KerbalInstructionsKit.Core.InstructionsKit.LessonUnlocked += id => { /* ... */ };
\`\`\`

## License

MIT. See `LICENSE`.
```

- [ ] **Step 2: Write `KerbalInstructionsKit.version` (KSP-AVC format)**

```json
{
    "NAME": "Kerbal Instructions Kit",
    "URL": "https://raw.githubusercontent.com/badgkat/KerbalInstructionsKit/main/KerbalInstructionsKit.version",
    "DOWNLOAD": "https://github.com/badgkat/KerbalInstructionsKit/releases",
    "VERSION": { "MAJOR": 0, "MINOR": 1, "PATCH": 0 },
    "KSP_VERSION": { "MAJOR": 1, "MINOR": 12, "PATCH": 5 },
    "KSP_VERSION_MIN": { "MAJOR": 1, "MINOR": 12, "PATCH": 0 },
    "KSP_VERSION_MAX": { "MAJOR": 1, "MINOR": 12, "PATCH": 99 }
}
```

- [ ] **Step 3: Commit**

```bash
git add README.md KerbalInstructionsKit.version
git commit -m "docs: expand README and add KSP-AVC version file"
```

### Task 29: Manual smoke test in KSP

**Files:**
- None (manual verification)

This task verifies everything assembled actually works end-to-end. Take careful notes — bugs surfaced here go into a final fix-up task (Task 30).

- [ ] **Step 1: Copy the test pack to GameData**

```bash
cd "GameData/KerbalInstructionsKit"
mkdir -p "../KIK_TestPack"
cp dev/test-pack/KIK_TestPack/lessons.cfg.example  "../KIK_TestPack/lessons.cfg"
cp dev/test-pack/KIK_TestPack/triggers.cfg.example "../KIK_TestPack/triggers.cfg"
```

- [ ] **Step 2: Verify build artifacts are in place**

```bash
ls -la "GameData/KerbalInstructionsKit/Plugins/"
# Expect: KerbalInstructionsKit.dll
ls -la "GameData/KerbalInstructionsKit/Plugins/Icons/"
# Expect: KIK_icon_38.png, KIK_icon_24.png
```

- [ ] **Step 3: Launch KSP and run through the checklist**

Take screenshots / notes for each:

| Check | Pass? |
|---|---|
| KSP loads without errors related to KIK in `KSP.log` | |
| Toolbar button appears on the KSC scene | |
| Toolbar button appears in VAB | |
| Toolbar button appears in SPH | |
| Toolbar button appears in Flight | |
| Toolbar button appears in Map View | |
| Toolbar button appears in Tracking Station | |
| Click toolbar button → archive opens with 1 lesson visible (LSN_TestRocketBasics) | |
| Click LSN_TestRocketBasics → lesson view opens at page 1 | |
| Prev button is disabled on page 1 | |
| Next button advances to page 2 | |
| Page 2 displays correct title, text with **bold** rendering | |
| Next is disabled on last page | |
| "📚 All Lessons" footer button returns to archive | |
| Search box filters lessons by title substring | |
| Tag chips filter (toggle Beginner) | |
| Launch a vessel (any) → return to KSC → archive now shows LSN_TestManeuver | |
| visibleIf works: LSN_TestAdvanced now appears in archive (after Rocket Basics unlocked) | |
| Click external URL link → browser opens to wiki page | |
| Click KSPedia link → KSPedia opens | |
| Click cross-link to Rocket Basics → switches lesson, "← Back" button appears | |
| Click "← Back" → returns to previous lesson | |
| ⏸ Pause button visible only in Flight | |
| ⏸ Pause toggles game time correctly | |
| Close panel → time resumes if KIK paused it | |
| Open panel again → returns to last viewed lesson |
| Save/quit/reload → unlock state persists |

- [ ] **Step 4: Record issues**

For each failed check, file a one-line note in a temp file (`smoke-test-fails.txt`). Address them all in Task 30.

- [ ] **Step 5: Clean up the test pack from GameData**

```bash
rm -rf "GameData/KIK_TestPack"
```

- [ ] **Step 6: No commit (this task is verification, not code)**

### Task 30: Fix-ups from smoke test

**Files:**
- Per fix-up findings.

For each issue captured in Task 29 Step 4:

- [ ] **Step 1: Diagnose** — read logs, reproduce, narrow down.

- [ ] **Step 2: Fix and add a unit test** if the fix is in pure-logic. If KSP-integrated, fix and re-verify manually.

- [ ] **Step 3: Re-run the relevant smoke-test checklist row.**

- [ ] **Step 4: Commit** with a descriptive message:

```bash
git commit -m "fix: <what>"
```

Repeat until the smoke-test checklist is all-pass.

### Task 31: Tag v0.1.0 and update BadgKatCareer dependency chain

**Files:**
- Modify: `GameData/BadgKatCareer/README.md` (dependency list)
- Modify: `GameData/BadgKatCareer/BadgKatCareer.ckan` (depends list)

- [ ] **Step 1: Tag the release in the KIK repo**

```bash
cd "GameData/KerbalInstructionsKit"
git tag -a v0.1.0 -m "v0.1.0 — initial release"
```

- [ ] **Step 2: Build the release artifact**

```bash
cd "GameData/KerbalInstructionsKit"
dotnet build src/KerbalInstructionsKit/KerbalInstructionsKit.csproj -c Release
git add Plugins/KerbalInstructionsKit.dll Plugins/Icons/
git commit -m "build: v0.1.0 release artifact"
```

- [ ] **Step 3: Update BadgKatCareer's `.ckan` to depend on KIK**

```bash
cd "GameData/BadgKatCareer"
```

In `BadgKatCareer.ckan`, add to the `depends` list:

```json
{ "name": "KerbalInstructionsKit" }
```

In the README dependency chain section, add KerbalInstructionsKit to the list of hard deps and document the soft-CC behavior.

- [ ] **Step 4: Commit BadgKatCareer changes**

```bash
git add BadgKatCareer.ckan README.md
git commit -m "deps: add KerbalInstructionsKit to dependency chain"
```

- [ ] **Step 5: Archive this plan**

```bash
cd "GameData/BadgKatCareer"
mkdir -p docs/superpowers/plans/archive
git mv docs/superpowers/plans/2026-04-29-instructions-kit.md docs/superpowers/plans/archive/
git commit -m "docs: archive instructions-kit implementation plan (completed)"
```

---

## Self-review notes

- All spec sections covered: cfg grammar (Tasks 5-7, 11), expression evaluator (Task 8), registry (Task 9), state (Task 10), trigger engine (Tasks 11-12, 19), filtering (Task 13), state machine (Task 14), public API (Task 15), scenario (Tasks 16, 19), per-scene runtime (Task 17), toolbar (Task 18), CC integration (Task 20), rendering (Tasks 21-23), Harmony injection (Tasks 24-25), polish/test/release (Tasks 26-31).
- Variation points from spec resolved: soft CC (Task 20 reflection-only), BEHAVIOUR + LESSON_TRIGGER (Tasks 11, 20), per-scene addons (Task 17).
- Pause logic implemented per spec: `wasPausedOnOpen` + `kikPausedFlag` in Task 21.
- Error handling per spec: log/skip patterns implemented in each loader (Tasks 5-7, 11) and runtime (Tasks 19-20).
- Testing strategy: pure-logic xUnit (Tasks 3-14), manual KSP for runtime/UI (Tasks 25, 29).
- Assume default: lesson `unlockOn` defaults to `Offered` per spec; `LESSON_TRIGGER onContract` without `state` defaults to `Accepted` (Task 11). The asymmetry is intentional — `AttachLesson` is meant to be ergonomic ("show the lesson the moment the contract is offered"), `LESSON_TRIGGER` is more conservative.
- One spec gap: localization. The plan doesn't add explicit Localizer calls. A follow-on micro-task can wrap `lesson.Title`, `page.Text`, `page.Caption`, and link `label` in `Localizer.Format(...)` — defer until first user requests it.
