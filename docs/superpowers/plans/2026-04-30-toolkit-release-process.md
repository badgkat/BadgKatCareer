# Toolkit Release Process — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a single, repeatable release standard across the four Kerbal\*Kit toolkit mods (KDK / KCK / KAK / KIK), then perform first public releases of KDK, KCK, KAK on GitHub + SpaceDock + CKAN, plus a single umbrella forum thread.

**Architecture:** A new `badgkat/ksp-libs` repo holds KSP reference DLLs that all toolkit CI builds pull. A parameterized GitHub Actions workflow lives identically in each toolkit repo (only the mod name differs) and publishes on `v*.*.*` tag push. CKAN distribution routes through SpaceDock — one-time netkan PR per mod, then automatic indexing. See `docs/superpowers/specs/2026-04-30-toolkit-release-process-design.md` for the full design.

**Tech Stack:** GitHub Actions (CI), .NET Framework 4.7.2 (`dotnet build`), xUnit, KSP-AVC `.version` files, CKAN netkan YAML, SpaceDock REST API, Keep a Changelog markdown.

**Repos touched:**
- `GameData/BadgKatCareer/` (branch: `master`) — plan + spec home, this file lives here
- `badgkat/ksp-libs` (NEW repo, will be created in Task 2)
- `GameData/KerbalDialogueKit/` (branch: `main`) — release skeleton + first release
- `GameData/KerbalCampaignKit/` (branch: `main`) — release skeleton + first release
- `GameData/KerbalAdminKit/` (branch: `main`) — `.version` cleanup + release skeleton + first release
- `KSP-CKAN/NetKAN` (upstream, fork-and-PR) — one PR per toolkit mod

**Four separate local git repos for the toolkits.** Always check the repo before running `cd` and git commands. Cross-repo work means committing in each repo separately. Working directory in PowerShell persists across commands; commands below use absolute paths or set the directory explicitly.

**Manual external steps:** Tasks 4, 11, 17, 23, 25 require human action on external services (SpaceDock account UI, forum posting, NetKAN-meta PR). Plan steps describe exactly what to do but cannot automate them.

---

## Phase 1 — Foundation: ksp-libs and CI template

### Task 1: Inventory the KSP reference DLLs the four toolkits need

**Files:**
- Read: `GameData/KerbalDialogueKit/Directory.Build.props`
- Read: `GameData/KerbalCampaignKit/Directory.Build.props`
- Read: `GameData/KerbalAdminKit/Directory.Build.props`

- [ ] **Step 1: Capture the full reference list across all three toolkits**

Open each `Directory.Build.props` and list every `<Reference>` and `<HintPath>`. Compile a deduplicated set. Expect entries like:
- `Assembly-CSharp.dll` (KSP root)
- `Assembly-CSharp-firstpass.dll`
- `UnityEngine.dll`, `UnityEngine.CoreModule.dll`, `UnityEngine.IMGUIModule.dll`, `UnityEngine.UIModule.dll` (etc.)
- `0Harmony.dll` (HarmonyKSP)
- `ContractConfigurator.dll`
- `ChatterBox.dll`
- `ToolbarController.dll`
- `ClickThroughBlocker.dll` (note: actual class is `ClickThruBlocker` in `ClickThroughFix` namespace)

- [ ] **Step 2: Build a manifest**

Write the deduplicated list to a scratch file you'll reference in Task 2. Group by source (KSP managed/, KSP managed/UnityEngine.*/, GameData/<ModName>/Plugins/). Note the file size of each so you can sanity-check the upload.

- [ ] **Step 3: No commit. This is research output for Task 2.**

### Task 2: Create `badgkat/ksp-libs` repo and populate it

**Files:**
- Create: new GitHub repo `badgkat/ksp-libs`
- Create: `lib/<DllName>.dll` for every reference from Task 1
- Create: `README.md`

- [ ] **Step 1: Create the repo on GitHub**

Run:

```powershell
gh repo create badgkat/ksp-libs --private --description "KSP 1.12.x reference DLLs for badgkat toolkit CI builds" --clone
cd ksp-libs
```

Expected: empty repo cloned to current directory.

- [ ] **Step 2: Copy reference DLLs into `lib/`**

```powershell
mkdir lib
$kspRoot = "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program"
# KSP managed:
Copy-Item "$kspRoot\KSP_x64_Data\Managed\Assembly-CSharp.dll" lib\
Copy-Item "$kspRoot\KSP_x64_Data\Managed\Assembly-CSharp-firstpass.dll" lib\
Copy-Item "$kspRoot\KSP_x64_Data\Managed\UnityEngine*.dll" lib\
# Modded refs from Task 1's manifest:
Copy-Item "$kspRoot\GameData\HarmonyKSP\0Harmony.dll" lib\
Copy-Item "$kspRoot\GameData\ContractConfigurator\Plugins\ContractConfigurator.dll" lib\
Copy-Item "$kspRoot\GameData\ChatterBox\Plugins\ChatterBox.dll" lib\
Copy-Item "$kspRoot\GameData\000_ClickThroughBlocker\Plugins\ClickThroughBlocker.dll" lib\
Copy-Item "$kspRoot\GameData\001_ToolbarControl\Plugins\ToolbarControl.dll" lib\
# (Adjust paths if any of the above moved.)
```

Verify:

```powershell
ls lib | Measure-Object | Select-Object Count
```

Expected: count matches your manifest from Task 1.

- [ ] **Step 3: Write `README.md`**

```markdown
# ksp-libs

KSP 1.12.x reference DLLs used by `badgkat` toolkit CI builds (KDK / KCK / KAK / KIK).

## Layout

`lib/` contains reference assemblies for KSP, Unity, and modded dependencies.

## Usage from a toolkit repo's CI

```yaml
- name: Checkout ksp-libs
  uses: actions/checkout@v4
  with:
    repository: badgkat/ksp-libs
    path: ksp-libs
    ssh-key: ${{ secrets.KSP_LIBS_DEPLOY_KEY }}

- name: Build with ksp-libs reference root
  run: dotnet build src/<ModName>/<ModName>.csproj -c Release /p:KspLibsPath=$GITHUB_WORKSPACE/ksp-libs/lib
```

## Updating

Bump KSP, Unity, or mod dependency DLLs by replacing files in `lib/` and committing. Toolkit CI checks out at `main` by default; pin to a SHA in workflow if needed.
```

- [ ] **Step 4: Initial commit and push**

```powershell
git add lib README.md
git commit -m "init: ksp-libs reference DLLs for KSP 1.12.x toolkit CI"
git push -u origin main
```

- [ ] **Step 5: Generate a deploy key for read-only checkout from CI**

```powershell
ssh-keygen -t ed25519 -f ksp-libs-deploy-key -N '""' -C "ksp-libs CI deploy key"
gh repo deploy-key add ksp-libs-deploy-key.pub --repo badgkat/ksp-libs --title "Toolkit CI"
# Show private key for adding to toolkit repo secrets:
cat ksp-libs-deploy-key
```

Save the private key contents — you'll add them as `KSP_LIBS_DEPLOY_KEY` secret in each toolkit repo (Tasks 5, 12, 18). Delete the local key files after secrets are configured everywhere.

- [ ] **Step 6: No further commit; the repo is ready.**

### Task 3: Author the parameterized release workflow template

**Files:**
- Create: `GameData/BadgKatCareer/docs/superpowers/templates/release.yml.template`

This template lives in BadgKatCareer's docs so a single source of truth exists. Per-toolkit copies are made by Tasks 8, 14, 20.

- [ ] **Step 1: Create the templates directory**

```powershell
mkdir "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\docs\superpowers\templates" -ErrorAction SilentlyContinue
```

- [ ] **Step 2: Write the template**

Create `release.yml.template` with this exact content. The string `{{MOD_NAME}}` is the only template variable; per-toolkit instantiation is a simple find-and-replace.

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Verify tag matches .version JSON
        shell: bash
        run: |
          TAG="${GITHUB_REF_NAME#v}"
          VERSION_JSON="{{MOD_NAME}}.version"
          MAJOR=$(jq -r '.VERSION.MAJOR' "$VERSION_JSON")
          MINOR=$(jq -r '.VERSION.MINOR' "$VERSION_JSON")
          PATCH=$(jq -r '.VERSION.PATCH' "$VERSION_JSON")
          VFILE="${MAJOR}.${MINOR}.${PATCH}"
          if [ "$TAG" != "$VFILE" ]; then
            echo "Tag $GITHUB_REF_NAME does not match .version $VFILE"
            exit 1
          fi
          echo "VERSION=$VFILE" >> "$GITHUB_ENV"

      - name: Verify CHANGELOG.md has section for this version
        shell: bash
        run: |
          if ! grep -q "^## \[$VERSION\]" CHANGELOG.md; then
            echo "CHANGELOG.md is missing a [$VERSION] section"
            exit 1
          fi

      - name: Checkout ksp-libs
        uses: actions/checkout@v4
        with:
          repository: badgkat/ksp-libs
          path: ksp-libs
          ssh-key: ${{ secrets.KSP_LIBS_DEPLOY_KEY }}

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '6.0.x'

      - name: Restore
        run: dotnet restore src/{{MOD_NAME}}/{{MOD_NAME}}.csproj

      - name: Build
        run: dotnet build src/{{MOD_NAME}}/{{MOD_NAME}}.csproj -c Release --no-restore /p:KspLibsPath=${{ github.workspace }}/ksp-libs/lib /p:TreatWarningsAsErrors=true

      - name: Test
        run: dotnet test tests/{{MOD_NAME}}.Tests/{{MOD_NAME}}.Tests.csproj -c Release --no-build

      - name: Stage GameData payload
        shell: bash
        run: |
          STAGE="stage/GameData/{{MOD_NAME}}"
          mkdir -p "$STAGE/Plugins"
          cp Plugins/{{MOD_NAME}}.dll "$STAGE/Plugins/"
          cp Plugins/{{MOD_NAME}}.pdb "$STAGE/Plugins/"
          cp {{MOD_NAME}}.version "$STAGE/"
          cp README.md LICENSE CHANGELOG.md "$STAGE/"
          # Mod-specific extras: copy Art/ if present
          if [ -d Art ]; then cp -r Art "$STAGE/"; fi

      - name: Create zip
        shell: bash
        run: |
          cd stage
          zip -r "../{{MOD_NAME}}-${VERSION}.zip" GameData

      - name: Extract release notes from CHANGELOG
        shell: bash
        run: |
          awk -v ver="$VERSION" '
            $0 ~ "^## \\["ver"\\]" { capture=1; next }
            capture && /^## \[/ { exit }
            capture { print }
          ' CHANGELOG.md > release-notes.md
          echo "RELEASE_NOTES_FILE=release-notes.md" >> "$GITHUB_ENV"

      - name: Determine pre-release flag
        shell: bash
        run: |
          MAJOR=$(echo "$VERSION" | cut -d. -f1)
          if [ "$MAJOR" -lt 1 ]; then
            echo "PRERELEASE=true" >> "$GITHUB_ENV"
          else
            echo "PRERELEASE=false" >> "$GITHUB_ENV"
          fi

      - name: Create GitHub release
        uses: softprops/action-gh-release@v2
        with:
          name: ${{ github.ref_name }}
          body_path: release-notes.md
          prerelease: ${{ env.PRERELEASE }}
          files: '{{MOD_NAME}}-*.zip'

      - name: Upload to SpaceDock
        shell: bash
        env:
          SPACEDOCK_API_TOKEN: ${{ secrets.SPACEDOCK_API_TOKEN }}
          SPACEDOCK_MOD_ID: ${{ secrets.SPACEDOCK_MOD_ID }}
        run: |
          jq -r '"\(.KSP_VERSION_MIN.MAJOR).\(.KSP_VERSION_MIN.MINOR).\(.KSP_VERSION_MIN.PATCH // 0)"' {{MOD_NAME}}.version > /tmp/ksp_min
          KSP_MIN=$(cat /tmp/ksp_min)
          curl -fsSL -X POST "https://spacedock.info/api/mod/${SPACEDOCK_MOD_ID}/update" \
            -H "Authorization: Bearer ${SPACEDOCK_API_TOKEN}" \
            -F "version=${VERSION}" \
            -F "game-version=${KSP_MIN}" \
            -F "notify-followers=yes" \
            -F "changelog=$(cat release-notes.md)" \
            -F "zipball=@{{MOD_NAME}}-${VERSION}.zip"

      - name: Job summary
        shell: bash
        run: |
          {
            echo "## Release {{MOD_NAME}} ${VERSION}"
            echo "- GitHub release: https://github.com/${{ github.repository }}/releases/tag/${{ github.ref_name }}"
            echo "- SpaceDock: https://spacedock.info/mod/${SPACEDOCK_MOD_ID:-unknown}"
            echo "- Zip size: $(stat -c%s {{MOD_NAME}}-${VERSION}.zip) bytes"
            echo ""
            echo "CKAN's NetKAN bot will index this within ~30 minutes."
          } >> "$GITHUB_STEP_SUMMARY"
```

- [ ] **Step 3: Verify the template is well-formed YAML**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\docs\superpowers\templates"
# Substitute KerbalDialogueKit and confirm yamllint accepts it (use any YAML linter):
(Get-Content release.yml.template) -replace '\{\{MOD_NAME\}\}', 'KerbalDialogueKit' | Out-File -Encoding utf8 /tmp/test-release.yml
# If yamllint or python is available:
python -c "import yaml; yaml.safe_load(open('/tmp/test-release.yml'))"
```

Expected: no exception thrown. If you don't have a YAML parser handy, eyeball the structure for indent consistency.

- [ ] **Step 4: Commit the template to BadgKatCareer**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer"
git add docs/superpowers/templates/release.yml.template
git commit -m "build: add toolkit release.yml template (parameterized by mod name)"
```

---

## Phase 2 — SpaceDock prerequisites

### Task 4: Create SpaceDock entries for KDK, KCK, KAK (manual)

**External:** https://spacedock.info — log in as `badgkat`.

This is a one-time manual step per mod. SpaceDock requires an initial mod entry before the API will accept version uploads.

- [ ] **Step 1: Create KDK entry**

On SpaceDock: "Add a Mod". Fill:
- **Name:** Kerbal Dialogue Kit
- **Short description:** Code-triggered dialogue, branching, and choice cards on top of ChatterBox.
- **License:** match `GameData/KerbalDialogueKit/LICENSE`
- **GitHub URL:** https://github.com/badgkat/KerbalDialogueKit
- **Source:** https://github.com/badgkat/KerbalDialogueKit
- **KSP version:** 1.12.5 (or your install's exact patch version)
- **Tags:** Plugin, Library

For the initial upload, use a placeholder zip (or skip the upload requirement if SpaceDock allows draft entries — current behavior may vary). The first real upload comes from CI in Task 10.

After creating: record the numeric mod ID from the URL (`https://spacedock.info/mod/<NNNN>/...`). You'll need it three times — for the GitHub repo secret in Task 5, the netkan in Task 7, and the netkan PR in Task 11.

- [ ] **Step 2: Create KCK entry**

Same form, with:
- **Name:** Kerbal Campaign Kit
- **Short description:** Chapter state, reputation economy, hierarchical notifications, and event-driven actions for campaign mods.
- All other fields per pattern. Record the mod ID.

- [ ] **Step 3: Create KAK entry**

- **Name:** Kerbal Admin Kit
- **Short description:** Customizes the Administration Building and surfaces character-authored program info across KSC.
- All other fields per pattern. Record the mod ID.

- [ ] **Step 4: Generate a SpaceDock API token**

In SpaceDock account settings, generate a personal access token. Copy it once (it won't be shown again). You'll add this as `SPACEDOCK_API_TOKEN` to each toolkit repo's secrets in Task 5.

- [ ] **Step 5: No commit. Record three numeric mod IDs and one API token in your password manager / scratch notes.**

### Task 5: Configure GitHub repo secrets for the three toolkit repos

**External:** GitHub repo settings → Secrets and variables → Actions.

For each of `badgkat/KerbalDialogueKit`, `badgkat/KerbalCampaignKit`, `badgkat/KerbalAdminKit`:

- [ ] **Step 1: Add `SPACEDOCK_API_TOKEN` secret**

Value: the token from Task 4 step 4.

- [ ] **Step 2: Add `SPACEDOCK_MOD_ID` secret**

Value: the per-mod numeric ID from Task 4. (KDK ID for KDK repo, etc.)

- [ ] **Step 3: Add `KSP_LIBS_DEPLOY_KEY` secret**

Value: the private key contents from Task 2 step 5.

- [ ] **Step 4: Verify each repo has all three secrets**

```powershell
gh secret list -R badgkat/KerbalDialogueKit
gh secret list -R badgkat/KerbalCampaignKit
gh secret list -R badgkat/KerbalAdminKit
```

Expected: each repo lists `SPACEDOCK_API_TOKEN`, `SPACEDOCK_MOD_ID`, `KSP_LIBS_DEPLOY_KEY`.

- [ ] **Step 5: Delete the local copy of the deploy key private key file**

```powershell
Remove-Item ksp-libs-deploy-key, ksp-libs-deploy-key.pub -ErrorAction SilentlyContinue
```

(Or wherever you generated them in Task 2 step 5.)

- [ ] **Step 6: No commit. Secrets are server-side.**

---

## Phase 3 — Per-mod skeleton rollout (KDK, then KCK, then KAK)

### Task 6: Add `CHANGELOG.md` to KDK

**Files:**
- Create: `GameData/KerbalDialogueKit/CHANGELOG.md`

- [ ] **Step 1: Write the changelog**

```markdown
# Changelog

All notable changes to this project will be documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.1.0] — 2026-04-30

Initial public release.

### Added

- Public API in `KerbalDialogueKit.Core` namespace: `DialogueKit.Enqueue(scene)` and `DialogueKit.EnqueueById(string)` for code- and cfg-triggered scenes.
- Per-save flag store at `DialogueKit.Flags`, backing `visibleIf` expressions.
- ConfigNode loaders for `SCENE`, `LINE`, `CHOICE` nodes.
- Branching dialogue and choice cards rendered via ChatterBox under the hood.
- Tokenizer support for `:` and `.` in flag identifiers (enables namespaced flag queries like `chapter.prologue.complete`).
```

- [ ] **Step 2: Commit**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
git add CHANGELOG.md
git commit -m "docs: add CHANGELOG.md for v0.1.0 release"
```

### Task 7: Add the netkan file for KDK

**Files:**
- Create: `GameData/KerbalDialogueKit/netkan/KerbalDialogueKit.netkan`

- [ ] **Step 1: Read the LICENSE to determine the SPDX identifier**

```powershell
Get-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\LICENSE" | Select-Object -First 3
```

Map the license to its CKAN SPDX identifier (e.g. `MIT`, `GPL-3.0`, `Apache-2.0`). If the LICENSE file's first line says "MIT License", the SPDX identifier is `MIT`. Reference: https://github.com/KSP-CKAN/CKAN/blob/master/Spec.md#license

- [ ] **Step 2: Write the netkan**

Create `netkan/KerbalDialogueKit.netkan`. Substitute `<KDK_SPACEDOCK_ID>` with the numeric ID from Task 4 step 1, and `<LICENSE_SPDX>` with the SPDX identifier from step 1 above.

```yaml
spec_version: v1.34
identifier: KerbalDialogueKit
name: Kerbal Dialogue Kit
abstract: Code-triggered dialogue, branching, and choice cards on top of ChatterBox.
author: badgkat
license: <LICENSE_SPDX>
resources:
  homepage: https://github.com/badgkat/KerbalDialogueKit
  repository: https://github.com/badgkat/KerbalDialogueKit
  bugtracker: https://github.com/badgkat/KerbalDialogueKit/issues
$kref: "#/ckan/spacedock/<KDK_SPACEDOCK_ID>"
$vref: "#/ckan/ksp-avc"
depends:
  - name: ChatterBox
ksp_version_min: "1.12.0"
ksp_version_max: "1.12.99"
```

- [ ] **Step 3: Validate as YAML**

```powershell
python -c "import yaml; print(yaml.safe_load(open('netkan/KerbalDialogueKit.netkan')))"
```

Expected: a dictionary printed, no exception.

- [ ] **Step 4: Commit**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
git add netkan/KerbalDialogueKit.netkan
git commit -m "build: add CKAN netkan source-of-truth file"
```

### Task 8: Instantiate the release workflow for KDK

**Files:**
- Create: `GameData/KerbalDialogueKit/.github/workflows/release.yml`

- [ ] **Step 1: Copy the template, substituting the mod name**

```powershell
$template = Get-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\docs\superpowers\templates\release.yml.template" -Raw
$rendered = $template -replace '\{\{MOD_NAME\}\}', 'KerbalDialogueKit'
mkdir "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\.github\workflows" -Force
Set-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\.github\workflows\release.yml" -Value $rendered -NoNewline
```

- [ ] **Step 2: Confirm no `{{MOD_NAME}}` placeholders remain**

```powershell
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\.github\workflows\release.yml" -Pattern '\{\{'
```

Expected: no output (no matches).

- [ ] **Step 3: Confirm `Directory.Build.props` accepts the `KspLibsPath` property**

Open `GameData/KerbalDialogueKit/Directory.Build.props`. Find the `<HintPath>` entries for KSP/Unity references. Update them to use `$(KspLibsPath)` when set, falling back to the local KSP install path when not. Pattern (existing line on top, replacement on bottom):

```xml
<!-- BEFORE -->
<HintPath>C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp.dll</HintPath>

<!-- AFTER -->
<HintPath Condition=" '$(KspLibsPath)' != '' ">$(KspLibsPath)\Assembly-CSharp.dll</HintPath>
<HintPath Condition=" '$(KspLibsPath)' == '' ">C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\KSP_x64_Data\Managed\Assembly-CSharp.dll</HintPath>
```

Apply the same dual-HintPath pattern to every reference in the file.

- [ ] **Step 4: Verify the local build still works after the props change**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
dotnet build src\KerbalDialogueKit\KerbalDialogueKit.csproj -c Release
```

Expected: build succeeds. (No `KspLibsPath` set → falls back to local KSP path.)

- [ ] **Step 5: Run tests to confirm nothing broke**

```powershell
dotnet test tests\KerbalDialogueKit.Tests\KerbalDialogueKit.Tests.csproj -c Release --no-build
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```powershell
git add .github\workflows\release.yml Directory.Build.props
git commit -m "build: add tag-driven release workflow and KspLibsPath override"
```

### Task 9: Smoke-test the KDK release workflow against a throwaway pre-release tag

**Files:**
- (none — this is a tag-and-watch test)

- [ ] **Step 1: Push the current state to GitHub**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
git push
```

- [ ] **Step 2: Bump `.version` JSON to a smoke-test version**

Edit `KerbalDialogueKit.version`. Change the version block:

```json
"VERSION": {
    "MAJOR": 0,
    "MINOR": 0,
    "PATCH": 99
}
```

- [ ] **Step 3: Add a CHANGELOG entry for the smoke-test version**

In `CHANGELOG.md`, immediately after `## [Unreleased]`, insert:

```markdown
## [0.0.99] — 2026-04-30

Smoke-test release. Will be deleted.
```

- [ ] **Step 4: Commit and tag**

```powershell
git add KerbalDialogueKit.version CHANGELOG.md
git commit -m "chore: smoke-test release v0.0.99-rc1"
git tag v0.0.99
git push origin main v0.0.99
```

- [ ] **Step 5: Watch the workflow**

```powershell
gh run watch -R badgkat/KerbalDialogueKit
```

Expected: workflow finishes successfully. GitHub release `v0.0.99` exists with attached zip. SpaceDock has a v0.0.99 upload visible.

- [ ] **Step 6: Inspect the zip artifact**

```powershell
gh release download v0.0.99 -R badgkat/KerbalDialogueKit -p '*.zip' -D /tmp
Expand-Archive /tmp\KerbalDialogueKit-0.0.99.zip /tmp\smoke -Force
Get-ChildItem -Recurse /tmp\smoke
```

Expected tree:
```
/tmp/smoke/GameData/KerbalDialogueKit/Plugins/KerbalDialogueKit.dll
/tmp/smoke/GameData/KerbalDialogueKit/Plugins/KerbalDialogueKit.pdb
/tmp/smoke/GameData/KerbalDialogueKit/KerbalDialogueKit.version
/tmp/smoke/GameData/KerbalDialogueKit/README.md
/tmp/smoke/GameData/KerbalDialogueKit/LICENSE
/tmp/smoke/GameData/KerbalDialogueKit/CHANGELOG.md
```

No `src/`, `tests/`, `demo/`, `docs/`, `.github/`, `netkan/`, `*.sln`, or `Directory.Build.props`.

- [ ] **Step 7: Tear down the smoke release**

```powershell
gh release delete v0.0.99 -R badgkat/KerbalDialogueKit --yes
git push --delete origin v0.0.99
git tag -d v0.0.99
git revert HEAD --no-edit
git push
# On SpaceDock: manually delete the 0.0.99 upload from the mod admin page (no API for delete-only).
```

If anything failed: **stop here and fix.** Do not proceed to Task 10 until the smoke test cleanly produces a valid GameData-rooted zip and a SpaceDock upload.

### Task 10: Promote KDK's `v0.1.0` tag to a real release

**Files:**
- (none)

- [ ] **Step 1: Confirm `.version`, `CHANGELOG.md`, and tag are aligned**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
Get-Content KerbalDialogueKit.version | Select-String '"MAJOR"|"MINOR"|"PATCH"'
Select-String -Path CHANGELOG.md -Pattern '^## \[0\.1\.0\]'
git tag -l v0.1.0
```

Expected: version reads `0.1.0`, CHANGELOG has `## [0.1.0]` heading, tag exists locally.

- [ ] **Step 2: Move the v0.1.0 tag to the current HEAD**

The existing local `v0.1.0` tag (created when the .dll was first built) points at a commit that predates the release skeleton. We need it at the current HEAD, which contains the CHANGELOG, workflow, and KspLibsPath plumbing.

```powershell
# Delete the old local tag:
git tag -d v0.1.0
# If it was somehow pushed remotely (unlikely — `gh release list` returned empty earlier), delete remote:
git push --delete origin v0.1.0 2>$null
# Tag the current HEAD:
git tag v0.1.0
# Push the tag — the workflow fires from this push:
git push origin v0.1.0
```

- [ ] **Step 3: Watch the workflow**

```powershell
gh run watch -R badgkat/KerbalDialogueKit
```

Expected: success. Real GitHub release `v0.1.0` published, SpaceDock has v0.1.0 upload.

- [ ] **Step 4: Visual confirmation**

Visit https://github.com/badgkat/KerbalDialogueKit/releases/tag/v0.1.0 and https://spacedock.info/mod/\<KDK_ID\>. Both should show v0.1.0 with the changelog body matching `CHANGELOG.md`'s `[0.1.0]` section.

- [ ] **Step 5: No commit. The release is the artifact.**

### Task 11: Submit netkan PR for KDK to KSP-CKAN/NetKAN (manual)

**External:** https://github.com/KSP-CKAN/NetKAN

- [ ] **Step 1: Fork KSP-CKAN/NetKAN to your account**

```powershell
gh repo fork KSP-CKAN/NetKAN --clone --remote
cd NetKAN
```

- [ ] **Step 2: Add the KDK netkan upstream**

```powershell
Copy-Item "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit\netkan\KerbalDialogueKit.netkan" NetKAN\
git checkout -b add-kerbaldialoguekit
git add NetKAN/KerbalDialogueKit.netkan
git commit -m "Add KerbalDialogueKit"
git push -u origin add-kerbaldialoguekit
```

- [ ] **Step 3: Open the PR**

```powershell
gh pr create --repo KSP-CKAN/NetKAN --title "Add KerbalDialogueKit" --body "Adds the KerbalDialogueKit netkan. SpaceDock-distributed, KSP-AVC compat metadata. Repo: https://github.com/badgkat/KerbalDialogueKit"
```

- [ ] **Step 4: Wait for review and merge**

CKAN maintainers typically review within 48 h. They may request small metadata fixes; address those by force-pushing the same branch.

- [ ] **Step 5: After merge, sanity-check from a CKAN client**

Wait ~30 minutes after merge for the indexer to run, then:

```powershell
ckan show KerbalDialogueKit
```

Expected: shows v0.1.0 as installable. (If you don't have a CKAN client locally, check https://ksp-ckan.space/?searchString=KerbalDialogueKit instead.)

- [ ] **Step 6: No commit in the toolkit repo. The netkan in `netkan/` is already the source of truth.**

### Task 12: Repeat Tasks 6-11 for KCK

**Files:**
- Create: `GameData/KerbalCampaignKit/CHANGELOG.md`
- Create: `GameData/KerbalCampaignKit/netkan/KerbalCampaignKit.netkan`
- Create: `GameData/KerbalCampaignKit/.github/workflows/release.yml`
- Modify: `GameData/KerbalCampaignKit/Directory.Build.props` (KspLibsPath dual HintPath)

- [ ] **Step 1: CHANGELOG.md for KCK 0.1.0**

```markdown
# Changelog

All notable changes to this project will be documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.1.0] — 2026-04-30

Initial public release.

### Added

- Public API in `KerbalCampaignKit.Core` namespace: `CampaignKit.Chapters` (chapter state), `CampaignKit.Reputation` (passive income / decay / gates), `CampaignKit.Notifications` (hierarchical notification store).
- Trigger engine wiring KSP `GameEvents` to cfg-defined `ACTION` nodes.
- ConfigNode loaders for `CAMPAIGN_CHAPTER`, `ACTION`, `REQUIREMENT` blocks.
- Pure-logic abstraction `ISceneNode` so xUnit tests run without KSP loaded.
- Contract event source publishes `contract:<name>.<state>` flags into the KDK flag store.
- ContractConfigurator `REQUIREMENT` types for chapter and reputation gating.
```

Commit:

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit"
git add CHANGELOG.md
git commit -m "docs: add CHANGELOG.md for v0.1.0 release"
```

- [ ] **Step 2: netkan/KerbalCampaignKit.netkan**

Substitute `<KCK_SPACEDOCK_ID>` and `<LICENSE_SPDX>`. The depends list adds KDK at the version released in Task 10:

```yaml
spec_version: v1.34
identifier: KerbalCampaignKit
name: Kerbal Campaign Kit
abstract: Chapter state, reputation economy, hierarchical notifications, and event-driven actions for campaign mods.
author: badgkat
license: <LICENSE_SPDX>
resources:
  homepage: https://github.com/badgkat/KerbalCampaignKit
  repository: https://github.com/badgkat/KerbalCampaignKit
  bugtracker: https://github.com/badgkat/KerbalCampaignKit/issues
$kref: "#/ckan/spacedock/<KCK_SPACEDOCK_ID>"
$vref: "#/ckan/ksp-avc"
depends:
  - name: ChatterBox
  - name: KerbalDialogueKit
    min_version: "0.1.0"
ksp_version_min: "1.12.0"
ksp_version_max: "1.12.99"
```

Validate and commit:

```powershell
python -c "import yaml; print(yaml.safe_load(open('netkan/KerbalCampaignKit.netkan')))"
git add netkan/KerbalCampaignKit.netkan
git commit -m "build: add CKAN netkan source-of-truth file"
```

- [ ] **Step 3: .github/workflows/release.yml**

```powershell
$template = Get-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\docs\superpowers\templates\release.yml.template" -Raw
$rendered = $template -replace '\{\{MOD_NAME\}\}', 'KerbalCampaignKit'
mkdir "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\.github\workflows" -Force
Set-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\.github\workflows\release.yml" -Value $rendered -NoNewline
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\.github\workflows\release.yml" -Pattern '\{\{'
```

Expected: no `{{...}}` matches.

- [ ] **Step 4: Update Directory.Build.props with KspLibsPath dual HintPath**

Apply the same Task 8 step 3 dual-HintPath pattern to KCK's `Directory.Build.props`. Verify local build:

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit"
dotnet build src\KerbalCampaignKit\KerbalCampaignKit.csproj -c Release
dotnet test tests\KerbalCampaignKit.Tests\KerbalCampaignKit.Tests.csproj -c Release --no-build
```

Expected: build and tests pass.

- [ ] **Step 5: Commit**

```powershell
git add .github\workflows\release.yml Directory.Build.props
git commit -m "build: add tag-driven release workflow and KspLibsPath override"
```

- [ ] **Step 6: Smoke-test the workflow** (mirrors Task 9)

Bump `.version` to `0.0.99`, add `[0.0.99]` CHANGELOG section, commit, tag `v0.0.99`, push, watch. Inspect zip. Tear down on success.

- [ ] **Step 7: Promote v0.1.0** (mirrors Task 10)

Confirm version/CHANGELOG align. Move the local `v0.1.0` tag to current HEAD per Task 10 step 2 (it currently points at a pre-skeleton commit):

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit"
git tag -d v0.1.0
git push --delete origin v0.1.0 2>$null
git tag v0.1.0
git push origin v0.1.0
```

Watch the workflow with `gh run watch -R badgkat/KerbalCampaignKit`. Confirm GitHub release and SpaceDock upload.

- [ ] **Step 8: Submit netkan PR to KSP-CKAN/NetKAN** (mirrors Task 11)

```powershell
cd <wherever-you-cloned-NetKAN-fork>
git checkout main
git pull upstream main
git checkout -b add-kerbalcampaignkit
Copy-Item "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit\netkan\KerbalCampaignKit.netkan" NetKAN/
git add NetKAN/KerbalCampaignKit.netkan
git commit -m "Add KerbalCampaignKit"
git push -u origin add-kerbalcampaignkit
gh pr create --repo KSP-CKAN/NetKAN --title "Add KerbalCampaignKit" --body "Depends on KerbalDialogueKit (already submitted in #<KDK_PR_NUMBER>). SpaceDock-distributed, KSP-AVC compat. Repo: https://github.com/badgkat/KerbalCampaignKit"
```

Wait for merge. Sanity-check `ckan show KerbalCampaignKit`.

### Task 13: Repeat Tasks 6-11 for KAK with `.version` cleanup

**Files:**
- Modify: `GameData/KerbalAdminKit/KerbalAdminKit.version` (drop redundant `KSP_VERSION` field)
- Create: `GameData/KerbalAdminKit/CHANGELOG.md`
- Create: `GameData/KerbalAdminKit/netkan/KerbalAdminKit.netkan`
- Create: `GameData/KerbalAdminKit/.github/workflows/release.yml`
- Modify: `GameData/KerbalAdminKit/Directory.Build.props`

- [ ] **Step 1: Normalize KerbalAdminKit.version**

Current state contains a redundant top-level `KSP_VERSION` field. Edit to match KDK/KCK schema:

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

(Note: KDK/KCK currently lack `PATCH` on the KSP_VERSION_MIN/MAX. Add `PATCH: 0` and `PATCH: 99` to KDK and KCK as well in this task to make the schema fully uniform — three-line edits each. Commit those alongside KAK's normalization.)

- [ ] **Step 2: Verify KAK still loads at game start (manual)**

Launch KSP. Confirm the `[KSP-AVC]` log lines do not show parse errors for KerbalAdminKit. (KSP-AVC is forgiving about extra fields, but the schema mismatch is a worth-confirming change.) If you don't want to launch KSP for this, skip — the field is non-load-bearing.

- [ ] **Step 3: Commit the .version normalization (KAK + KDK + KCK)**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit"
git add KerbalAdminKit.version
git commit -m "fix: normalize .version schema to match KDK/KCK"

cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
git add KerbalDialogueKit.version
git commit -m "fix: add explicit PATCH on KSP_VERSION_MIN/MAX"

cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit"
git add KerbalCampaignKit.version
git commit -m "fix: add explicit PATCH on KSP_VERSION_MIN/MAX"
```

(KDK and KCK changes are tiny and don't merit a separate task — they're a uniformity sweep paired with KAK's bigger normalization.)

- [ ] **Step 4: CHANGELOG.md for KAK 0.1.0**

```markdown
# Changelog

All notable changes to this project will be documented in this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.1.0] — 2026-04-30

Initial public release.

### Added

- Replacement Administration Building UI when `replaceAdminBuilding = true` in cfg: three-panel IMGUI screen (CharacterPanel / DashboardPanel / DeskPanel) intercepted via Harmony patch on `SpaceCenterBuilding.OnClicked`.
- Per-building character chips on Mission Control and Tracking Station scenes.
- KSC notification markers and condition-driven memos with edge-triggered rearm dismissal.
- Disposition decay system.
- PR Campaign action.
- Memo composite ANY/ALL conditions, ChapterAtLeast condition, character-chip orange "!" badge for unselected focus.
- 52 xUnit tests covering loaders, condition evaluation, decay compilation. KSP-integrated code (IMGUI / Harmony / MonoBehaviours) verified manually in-game.

### Fixed

- `.version` JSON now matches KDK/KCK schema (removed redundant top-level `KSP_VERSION`).
```

Commit:

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit"
git add CHANGELOG.md
git commit -m "docs: add CHANGELOG.md for v0.1.0 release"
```

- [ ] **Step 5: netkan/KerbalAdminKit.netkan**

Substitute `<KAK_SPACEDOCK_ID>` and `<LICENSE_SPDX>`:

```yaml
spec_version: v1.34
identifier: KerbalAdminKit
name: Kerbal Admin Kit
abstract: Customizes the Administration Building and surfaces character-authored program info across KSC.
author: badgkat
license: <LICENSE_SPDX>
resources:
  homepage: https://github.com/badgkat/KerbalAdminKit
  repository: https://github.com/badgkat/KerbalAdminKit
  bugtracker: https://github.com/badgkat/KerbalAdminKit/issues
$kref: "#/ckan/spacedock/<KAK_SPACEDOCK_ID>"
$vref: "#/ckan/ksp-avc"
depends:
  - name: ChatterBox
  - name: KerbalDialogueKit
    min_version: "0.1.0"
  - name: KerbalCampaignKit
    min_version: "0.1.0"
ksp_version_min: "1.12.0"
ksp_version_max: "1.12.99"
```

Validate and commit:

```powershell
python -c "import yaml; print(yaml.safe_load(open('netkan/KerbalAdminKit.netkan')))"
git add netkan/KerbalAdminKit.netkan
git commit -m "build: add CKAN netkan source-of-truth file"
```

- [ ] **Step 6: .github/workflows/release.yml from template**

```powershell
$template = Get-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\BadgKatCareer\docs\superpowers\templates\release.yml.template" -Raw
$rendered = $template -replace '\{\{MOD_NAME\}\}', 'KerbalAdminKit'
mkdir "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit\.github\workflows" -Force
Set-Content "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit\.github\workflows\release.yml" -Value $rendered -NoNewline
Select-String -Path "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit\.github\workflows\release.yml" -Pattern '\{\{'
```

Expected: no matches.

**Note:** The template's stage step copies `Art/` if it exists. KAK has `Art/` — confirm the staging step in the rendered YAML still references `if [ -d Art ]; then cp -r Art "$STAGE/"; fi`.

- [ ] **Step 7: Update Directory.Build.props with KspLibsPath dual HintPath**

Same pattern as Task 8 step 3. Verify local build:

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit"
dotnet build src\KerbalAdminKit\KerbalAdminKit.csproj -c Release
dotnet test tests\KerbalAdminKit.Tests\KerbalAdminKit.Tests.csproj -c Release --no-build
```

Expected: build and 52 tests pass.

- [ ] **Step 8: Commit**

```powershell
git add .github\workflows\release.yml Directory.Build.props
git commit -m "build: add tag-driven release workflow and KspLibsPath override"
```

- [ ] **Step 9: Smoke-test the workflow** (mirrors Task 9)

Bump `.version` to `0.0.99`, add CHANGELOG entry, commit, tag `v0.0.99`, push, watch. **Critical:** inspect the zip and confirm `Art/` is included under `GameData/KerbalAdminKit/Art/`. Tear down on success.

- [ ] **Step 10: Promote v0.1.0** (mirrors Task 10)

Move the local `v0.1.0` tag to current HEAD per Task 10 step 2:

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit"
git tag -d v0.1.0
git push --delete origin v0.1.0 2>$null
git tag v0.1.0
git push origin v0.1.0
```

Watch the workflow with `gh run watch -R badgkat/KerbalAdminKit`. Confirm GitHub release and SpaceDock upload, paying particular attention to the `Art/` directory inside the staged zip.

- [ ] **Step 11: Submit netkan PR to KSP-CKAN/NetKAN** (mirrors Task 11)

```powershell
cd <NetKAN-fork>
git checkout main
git pull upstream main
git checkout -b add-kerbaladminkit
Copy-Item "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit\netkan\KerbalAdminKit.netkan" NetKAN/
git add NetKAN/KerbalAdminKit.netkan
git commit -m "Add KerbalAdminKit"
git push -u origin add-kerbaladminkit
gh pr create --repo KSP-CKAN/NetKAN --title "Add KerbalAdminKit" --body "Depends on KerbalDialogueKit and KerbalCampaignKit (already submitted in #<KDK_PR> and #<KCK_PR>). SpaceDock-distributed, KSP-AVC compat. Repo: https://github.com/badgkat/KerbalAdminKit"
```

Wait for merge. Sanity-check `ckan show KerbalAdminKit`.

---

## Phase 4 — Public surface

### Task 14: Fill SpaceDock long descriptions from each repo's README

**External:** SpaceDock mod admin pages.

- [ ] **Step 1: KDK long description**

Open https://spacedock.info/mod/\<KDK_ID\>/admin. Paste the contents of `GameData/KerbalDialogueKit/README.md` into the long-description field, formatted as SpaceDock's markdown/BBCode flavor expects (most markdown renders directly; check the live preview).

Add at the top: link to the umbrella forum thread (placeholder text "📌 Forum thread: <to be filled in Task 15>" — return and update after Task 15).

- [ ] **Step 2: KCK long description**

Same pattern with `GameData/KerbalCampaignKit/README.md`.

- [ ] **Step 3: KAK long description**

Same pattern with `GameData/KerbalAdminKit/README.md`.

- [ ] **Step 4: No commit. SpaceDock content lives upstream.**

### Task 15: Create the umbrella forum thread (manual)

**External:** https://forum.kerbalspaceprogram.com/

- [ ] **Step 1: Draft the OP locally**

Write the post text in a scratch file. Sections in order:

1. **One-paragraph intro.** What the toolkit family is, who it's for (modders building career and story content).
2. **Family overview block** — table or list with one row per mod:

   | Mod | Purpose | Status | SpaceDock | Latest |
   |---|---|---|---|---|
   | Kerbal Dialogue Kit (KDK) | Code-triggered dialogue and choice cards | Released | [link] | v0.1.0 |
   | Kerbal Campaign Kit (KCK) | Chapter state, reputation, notifications | Released | [link] | v0.1.0 |
   | Kerbal Admin Kit (KAK) | Admin Building UI + KSC chips/memos | Released | [link] | v0.1.0 |
   | Kerbal Instructions Kit (KIK) | Instructional lessons and lesson archive | 🚧 In development | — | — |

3. **Dependency chain.**

   ```
   ChatterBox → KerbalDialogueKit → KerbalCampaignKit → KerbalAdminKit ← KerbalInstructionsKit (future)
   ```

4. **Per-mod sections** (use forum spoiler tags for collapse). For each released mod: feature list (3-5 bullets), one-liner public-API summary (e.g. `DialogueKit.Enqueue(scene)`), link to the GitHub repo's README for full docs.

5. **License + source links** per mod.

6. **Changelog block.** Quote each mod's current `## [0.1.0]` section. Link to the full `CHANGELOG.md` in each repo for older versions (none yet, but the structure is set up).

7. **Acknowledgements.** ChatterBox upstream (link to its forum thread / repo).

- [ ] **Step 2: Post the thread**

Forum section: "Add-on Releases" (KSP 1.12.x). Title:

> **[1.12.x] Kerbal *Kit Toolkit Family — Dialogue, Campaigns, Admin, Instructions frameworks**

- [ ] **Step 3: Record the forum thread URL**

Save the URL — you'll need it in Tasks 14 (back-fill SpaceDock long descriptions) and 16 (back-fill repo READMEs and netkan resources).

- [ ] **Step 4: No commit. The forum thread content lives upstream.**

### Task 16: Back-fill forum-thread URL in repo READMEs and netkan resources

**Files:**
- Modify: `GameData/KerbalDialogueKit/README.md`
- Modify: `GameData/KerbalCampaignKit/README.md`
- Modify: `GameData/KerbalAdminKit/README.md`
- Modify: `GameData/KerbalDialogueKit/netkan/KerbalDialogueKit.netkan`
- Modify: `GameData/KerbalCampaignKit/netkan/KerbalCampaignKit.netkan`
- Modify: `GameData/KerbalAdminKit/netkan/KerbalAdminKit.netkan`

- [ ] **Step 1: Add a "Releasing" pointer + forum/SpaceDock links to each README**

For each toolkit repo's `README.md`, append (or update an existing intro section to include) a links block:

```markdown
## Releases

- **SpaceDock:** https://spacedock.info/mod/<MOD_ID>
- **CKAN:** `ckan install <ModName>`
- **Forum thread:** <UMBRELLA_THREAD_URL>

See [the toolkit release process](https://github.com/badgkat/BadgKatCareer/blob/master/docs/superpowers/specs/2026-04-30-toolkit-release-process-design.md) for how releases are produced.
```

- [ ] **Step 2: Add `resources.ksp_forum_thread` (and `resources.spacedock`) to each in-repo netkan**

For each `netkan/<ModName>.netkan`, expand the `resources:` block:

```yaml
resources:
  homepage: https://github.com/badgkat/<ModName>
  repository: https://github.com/badgkat/<ModName>
  bugtracker: https://github.com/badgkat/<ModName>/issues
  spacedock: https://spacedock.info/mod/<MOD_ID>
  ksp_forum_thread: <UMBRELLA_THREAD_URL>
```

(Order within the block doesn't matter for CKAN; pick a stable convention.)

- [ ] **Step 3: Update upstream NetKAN-meta with the new resources**

For each toolkit, open a small PR to KSP-CKAN/NetKAN updating its netkan with the new `spacedock` and `ksp_forum_thread` resources. (CKAN bot will not pick these up from your in-repo copy — upstream is the source of truth for indexer behavior.)

```powershell
cd <NetKAN-fork>
git checkout main
git pull upstream main
git checkout -b add-thread-urls
# Copy each updated netkan from your repos into NetKAN/
git add NetKAN/KerbalDialogueKit.netkan NetKAN/KerbalCampaignKit.netkan NetKAN/KerbalAdminKit.netkan
git commit -m "Add forum thread and SpaceDock resources to KDK/KCK/KAK"
git push -u origin add-thread-urls
gh pr create --repo KSP-CKAN/NetKAN --title "Add thread/SpaceDock resources to KDK/KCK/KAK" --body "Adds forum thread and SpaceDock resource URLs to the existing netkans."
```

- [ ] **Step 4: Commit each toolkit's README and netkan changes**

```powershell
cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalDialogueKit"
git add README.md netkan/KerbalDialogueKit.netkan
git commit -m "docs: link to SpaceDock, CKAN, and umbrella forum thread"

cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalCampaignKit"
git add README.md netkan/KerbalCampaignKit.netkan
git commit -m "docs: link to SpaceDock, CKAN, and umbrella forum thread"

cd "C:\Program Files (x86)\Steam\steamapps\common\Kerbal Space Program\GameData\KerbalAdminKit"
git add README.md netkan/KerbalAdminKit.netkan
git commit -m "docs: link to SpaceDock, CKAN, and umbrella forum thread"
```

- [ ] **Step 5: Update SpaceDock long descriptions with the now-known forum URL**

Return to Task 14's SpaceDock admin pages and replace the placeholder forum-thread text with the real URL.

---

## Done condition

- KDK, KCK, KAK each have v0.1.0 GitHub releases with strict-GameData zip artifacts.
- Each mod has a SpaceDock page with the v0.1.0 upload, full long description, and umbrella forum thread link.
- Each mod is `ckan install`-able, dependencies resolve in order ChatterBox → KDK → KCK → KAK.
- Single umbrella forum thread linked from all three SpaceDock pages and all three repo READMEs.
- `badgkat/ksp-libs` repo holds the reference DLLs; CI in all three toolkit repos checks it out at build time.
- The release.yml template at `BadgKatCareer/docs/superpowers/templates/release.yml.template` is the source of truth for future toolkit CI changes.

When KIK reaches v0.1.0 readiness, the Phase 2 + Phase 3 + Phase 4 steps repeat for KIK alone — the template, ksp-libs repo, and forum thread are already in place. The KIK row in the umbrella forum OP changes from "🚧 in development" to released, and the netkan / SpaceDock entry / first release follow the same pattern.
