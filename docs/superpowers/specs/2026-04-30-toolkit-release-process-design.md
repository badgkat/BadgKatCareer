# Toolkit Release Process — Design

**Date:** 2026-04-30
**Status:** Draft — pending implementation plan
**Scope:** KerbalDialogueKit (KDK), KerbalCampaignKit (KCK), KerbalAdminKit (KAK), KerbalInstructionsKit (KIK)

## Motivation

The four toolkit mods are out of the "is the code right?" phase and into the "how does it reach players?" phase. ChatterBox PR #1 has merged upstream, which unblocks publishing KDK; KCK and KAK depend on KDK; KIK is in dev and will inherit whatever standard we adopt now.

Today each repo has a `v0.1.0` git tag but no GitHub release, no SpaceDock listing, no CKAN registration, and no forum thread. There is also no CI, no CHANGELOG, and no scripted packaging. The four mods need to ship publicly with a repeatable process that does not depend on remembering ten fiddly steps each time.

This spec defines a single release standard applied identically across all four toolkit repos, plus a once-and-done forum + SpaceDock + CKAN setup per mod. The campaign layer (BadgKatCareer) is deliberately out of scope here — it is cfg-only, has different distribution needs (CKAN metapackage, no DLL), and gets its own release thread for a different audience (players, not modders).

## Scope

**In scope:**

- Repo skeleton (files, locations, naming) shared across the four toolkit mods.
- Build + package contract — what the release zip contains and what gets excluded.
- Tag-driven GitHub Actions release pipeline, identical across mods (only the mod name parameterized).
- One-time CKAN netkan setup per mod, after which CKAN auto-indexes from SpaceDock.
- SpaceDock page conventions (fields, descriptions, cross-links).
- Single umbrella forum thread covering all four toolkit mods, with per-mod sections.
- KAK `.version` file normalization to match KDK/KCK schema.
- Centralized `badgkat/ksp-libs` reference-DLL repo to support CI builds across the four toolkits.

**Out of scope:**

- BadgKatCareer release process, CKAN metapackage shape, or campaign forum thread content. Covered separately when the campaign reaches public-release readiness.
- KIK's source design (covered in `2026-04-29-instructions-kit-design.md`). This spec only ensures KIK *inherits* the toolkit release standard on day one.
- Any per-release marketing / changelog *content* — this spec defines structure and process; release content is hand-written per release.
- Forum maintenance automation. The umbrella thread is hand-edited.
- ChatterBox upstream releases. We do not release ChatterBox; we depend on it.

## Five-Layer Architecture

The standard has five independently usable layers:

1. **Repo skeleton** — identical file set across all four toolkit repos.
2. **Build + package contract** — deterministic rules for what the release zip contains.
3. **CI release pipeline** — tag-driven GitHub Actions workflow, identical across mods.
4. **CKAN distribution** — one-time netkan setup; SpaceDock-as-primary, CKAN bot indexes automatically.
5. **Forum + SpaceDock presence** — single umbrella forum thread, per-mod SpaceDock pages.

Each section below specifies one layer.

## Layer 1: Repo Skeleton

Every toolkit repo carries this exact tree of release-relevant files. KDK and KCK already match most of it; KAK needs a small fixup; KIK inherits it on day one.

```
<RepoRoot>/
├── .github/
│   └── workflows/
│       └── release.yml              ← identical across mods, only mod name templated
├── netkan/
│   └── <ModName>.netkan             ← submitted once to KSP-CKAN/NetKAN, kept in repo as source of truth
├── src/<ModName>/                   ← unchanged
├── tests/<ModName>.Tests/           ← unchanged
├── Plugins/                         ← committed DLL+PDB (status quo)
├── CHANGELOG.md                     ← NEW; Keep a Changelog format
├── README.md                        ← unchanged; gains a one-paragraph "Releasing" pointer
├── LICENSE                          ← unchanged
├── <ModName>.version                ← KSP-AVC; KAK normalized to KDK/KCK schema
├── <ModName>.sln                    ← unchanged
└── Directory.Build.props            ← unchanged
```

**New files this spec introduces:**

- `CHANGELOG.md` (one per repo). Keep a Changelog format with `## [Unreleased]` / `## [0.1.0] — YYYY-MM-DD` sections, each containing `Added` / `Changed` / `Fixed` / `Removed` subsections. Initial entry is `0.1.0` describing the shipped state at time of first GitHub release.
- `.github/workflows/release.yml` (one per repo, generated from a single template).
- `netkan/<ModName>.netkan` (one per repo). Mirrors what is upstream in `KSP-CKAN/NetKAN`; in-repo copy is the source of truth for future metadata edits.

**Existing-state cleanup:**

- KAK's `KerbalAdminKit.version` currently contains a redundant top-level `KSP_VERSION` field (a remnant of an older KSP-AVC schema). Drop it; keep `KSP_VERSION_MIN` and `KSP_VERSION_MAX` only, matching KDK/KCK.

**Explicitly NOT in the skeleton:**

- No `RELEASING.md` per repo. This spec is the authoritative process document; each repo's `README.md` gets a one-paragraph "Releasing" section linking back here.
- No `.editorconfig`, build-tooling churn, or test-framework changes. Out of scope.

## Layer 2: Build + Package Contract

What `dotnet build -c Release` produces and what the release zip contains. Identical rules across the four mods.

### Build Output

- `dotnet build src/<ModName>/<ModName>.csproj -c Release` writes `<ModName>.dll` + `<ModName>.pdb` into `Plugins/`. Already configured this way via `Directory.Build.props` in each repo.
- Tests: `dotnet test tests/<ModName>.Tests/<ModName>.Tests.csproj`. CI runs both build and test; any test failure aborts the release before any side effects.

### Zip Contents (Strict GameData-Rooted)

```
<ModName>-<MAJOR>.<MINOR>.<PATCH>.zip
└── GameData/
    └── <ModName>/
        ├── Plugins/
        │   ├── <ModName>.dll
        │   └── <ModName>.pdb
        ├── <ModName>.version
        ├── README.md
        ├── LICENSE
        ├── CHANGELOG.md
        └── [mod-specific extras — see below]
```

### Mod-Specific Extras Inside `GameData/<ModName>/`

| Mod | Extra payload | Reason |
|---|---|---|
| KDK | _none_ | No runtime content; the toolkit is API + cfg schema only. `Examples/`, `demo/`, `docs/` are dev-time files. |
| KCK | _none_ | Same as KDK. `Examples/` and `demo-campaign.cfg` are repo-only. |
| KAK | `Art/` | Textures and icons loaded at runtime by the IMGUI panels. |
| KIK | TBD on ship | Default to "ship only what runtime loads"; runtime images for lessons live in the content pack, not the framework. |

### Always Excluded

Regardless of mod:
- `src/`, `tests/`, `demo/`, `Examples/`, `docs/`
- `*.sln`, `Directory.Build.props`, `*.csproj`
- `bin/`, `obj/`
- `.git*`, `.github/`
- `netkan/` (only the upstream NetKAN-meta repo cares about this directory)

### Filename and Tag Conventions

- Zip filename: `<ModName>-<MAJOR>.<MINOR>.<PATCH>.zip` (no `v` prefix in the filename version).
- Git tag: `v<MAJOR>.<MINOR>.<PATCH>` (with `v` prefix).
- GitHub release name: `v<MAJOR>.<MINOR>.<PATCH>` (matches tag).

This split (no `v` in filename, `v` in tag) is the most common convention across KSP mods on SpaceDock.

### Version-Tag Consistency

The first step of the workflow asserts that the pushed tag (with `v` stripped) exactly matches the `MAJOR.MINOR.PATCH` triple in `<ModName>.version`. Mismatch hard-fails the workflow before any destructive step. This is the safety rail against tagging without bumping the version file.

## Layer 3: CI Release Pipeline

The `.github/workflows/release.yml` lives identically in each repo (only the mod name differs). Triggered on push of any tag matching `v*.*.*`.

### Pipeline Steps

1. **Checkout** at the tag.
2. **Verify tag matches `.version` JSON.** Read `<ModName>.version`, compare `v${MAJOR}.${MINOR}.${PATCH}` against `${{ github.ref_name }}`. Hard-fail on mismatch.
3. **Verify `CHANGELOG.md` has a section for this version.** Grep for `## [<version>]` heading. Hard-fail if missing — forces release notes to exist before tagging.
4. **Setup .NET build environment.** `actions/setup-dotnet` plus a checkout of the centralized `badgkat/ksp-libs` repo (see Layer 3.1).
5. **`dotnet build -c Release`** with warnings-as-errors.
6. **`dotnet test`** — release fails on any test failure.
7. **Stage zip contents** — script copies the allowlisted files into a temp `GameData/<ModName>/` tree per Layer 2's rules; everything not on the allowlist is silently dropped.
8. **Create zip** — `<ModName>-${version}.zip` (no `v` prefix in filename).
9. **Extract release notes from CHANGELOG.** Pull the section between `## [<version>]` and the next `## [` heading. Pass as the GitHub release body.
10. **Create GitHub release** via `softprops/action-gh-release` (or `gh release create`). Mark as pre-release if version `< 1.0.0`.
11. **Upload zip to SpaceDock** via SpaceDock's API. Pass version string, KSP version range from `.version`, and the changelog body. Requires `SPACEDOCK_API_TOKEN` repo secret.
12. **Job summary.** Print release URL, SpaceDock URL, zip size, and a one-liner "CKAN's bot will index this within ~30 minutes."

### Layer 3.1: KSP Reference Assemblies

The build needs `Assembly-CSharp.dll`, `UnityEngine.*.dll`, and the ContractConfigurator / ChatterBox / ToolbarController / ClickThroughBlocker DLLs that each toolkit references. CI cannot rely on a local KSP install.

**Solution:** a centralized `badgkat/ksp-libs` repo that holds the reference DLLs needed across all four toolkit mods. CI in each toolkit repo checks out `ksp-libs` at a pinned ref, sets a build property pointing the `Reference` items there, then proceeds.

- Repo can stay private; CI uses a deploy key (or fine-grained PAT) to clone.
- Single source of truth across all four toolkit repos — KSP version bumps and dependency updates happen once.
- Each toolkit's `Directory.Build.props` keeps using local paths for developer-machine builds; CI overrides the reference root via an MSBuild property passed on the command line.
- Initial `ksp-libs` content: the same set of DLLs the four mods currently reference locally, captured from the developer's KSP 1.12.5 install plus matching mod versions.

### Failure Modes and Rollback

- **Tag points to wrong commit.** Delete the tag (`git push --delete origin v0.1.1`) and re-tag. The GitHub Action re-runs.
- **Zip uploads to GitHub but SpaceDock upload fails.** GitHub release exists; manually upload to SpaceDock from the Actions artifact. CKAN bot still indexes once SpaceDock has the file.
- **CHANGELOG missing for the tagged version.** Workflow hard-fails at step 3, before the GitHub release is created. Add the changelog section, push as a new commit, retag.

## Layer 4: CKAN Distribution

CKAN uses SpaceDock as the canonical download source. CKAN's NetKAN bot polls SpaceDock for new versions and auto-indexes them. After a one-time netkan PR per mod, ongoing releases require zero CKAN action.

### `.netkan` File Per Mod

Lives at `netkan/<ModName>.netkan` in the toolkit repo and is mirrored upstream in `KSP-CKAN/NetKAN`. Skeleton:

```yaml
spec_version: v1.34
identifier: KerbalDialogueKit
name: Kerbal Dialogue Kit
abstract: Code-triggered dialogue, branching, and choice cards on top of ChatterBox.
author: badgkat
license: <fill in from LICENSE file>
resources:
  homepage: https://github.com/badgkat/KerbalDialogueKit
  repository: https://github.com/badgkat/KerbalDialogueKit
  bugtracker: https://github.com/badgkat/KerbalDialogueKit/issues
$kref: "#/ckan/spacedock/<spacedock-id>"
$vref: "#/ckan/ksp-avc"
depends:
  - name: ChatterBox
ksp_version_min: "1.12.0"
ksp_version_max: "1.12.99"
```

- `$kref` tells CKAN's indexer "watch SpaceDock listing N for new versions."
- `$vref` tells it "use the KSP-AVC `.version` file inside the zip for compat metadata."

### Per-Mod Dependency Wiring

| Mod | `depends` |
|---|---|
| KDK | `ChatterBox` |
| KCK | `KerbalDialogueKit` (min_version: matching), `ChatterBox` |
| KAK | `KerbalCampaignKit` (min_version: matching), `KerbalDialogueKit` (min_version: matching), `ChatterBox` |
| KIK | `KerbalDialogueKit` (min_version: matching) at minimum; full deps determined when KIK ships |

**`min_version` policy:** floor at the version where the API or cfg-schema feature your mod uses landed. Do not add `max_version` unless you actively know a downstream version breaks compatibility — over-pinning is the most common netkan mistake and produces dependency-resolution deadlocks for end users.

### One-Time Setup Per Mod

1. Create SpaceDock entry for the mod under the `badgkat` account. Record the numeric SpaceDock ID.
2. Fill the `<spacedock-id>` placeholder in the in-repo `netkan/<ModName>.netkan`.
3. PR `<ModName>.netkan` to `KSP-CKAN/NetKAN` repo.
4. CKAN maintainers review and merge (typically <48 h). Bot starts indexing.
5. Sanity check: `ckan show <ModName>` from a CKAN client should resolve and offer install.

After this setup, each new GitHub release that uploads to SpaceDock automatically appears in CKAN within ~30 minutes via the bot. **CI does not touch CKAN directly.**

## Layer 5: Forum and SpaceDock Presence

### SpaceDock Pages (Per Mod)

One page per toolkit mod under the `badgkat` account. Identical structure across the four:

| Field | Source |
|---|---|
| Name | Full mod name (e.g. "Kerbal Dialogue Kit") |
| Short description | The `abstract` field from the `.netkan` |
| Author | badgkat |
| License | Matches the `LICENSE` file in the repo |
| GitHub link | Repo URL |
| Forum link | The umbrella toolkit thread URL (same on all four mod pages) |
| Long description | Lifted from the repo's `README.md`, re-rendered into SpaceDock's markdown/BBCode |
| KSP version compat | `1.12.x` (matches `.version` file) |
| Tags | `Plugin`, `Library` |

Per-release changelog churn lives in the upload's release notes (which CI passes through). The long-description page is updated only when the README's headline sections change.

### Umbrella Forum Thread (One Total)

Single thread on the KSP forums covering all four toolkit mods. The toolkit family shares an audience (modders building career and story content); forking it into four threads would split that audience across mostly-redundant posts.

**Title format:**

> **[1.12.x] Kerbal *Kit Toolkit Family — Dialogue, Campaigns, Admin, Instructions frameworks**

**OP structure:**

1. **One-paragraph intro.** What the toolkit family is and who it is for (modders building career or story content; not players directly).
2. **Family overview block.** A row per mod with: one-line purpose, status badge (Released / In dev), download link (SpaceDock), latest version. KIK starts as "🚧 in development" with no link until ship.
3. **Dependency chain diagram.** ASCII or simple list — `ChatterBox → KDK → KCK → KAK ← KIK(?)` — so new readers immediately understand install order.
4. **Per-mod sections** (collapsible / spoiler tags). For each mod: feature list, public API one-liner (e.g. `DialogueKit.Enqueue(scene)`), link to the repo's `README.md` for full docs.
5. **License + source links** per mod.
6. **Changelog block.** For each mod, the most recent CHANGELOG section quoted; older versions link to the repo's `CHANGELOG.md`.
7. **Acknowledgements.** ChatterBox upstream and any contributors.

**Maintenance pattern:** on each toolkit-mod release, edit the OP to bump the affected mod's row in the overview block and paste in the new CHANGELOG section. Reply to the thread with a short "v0.1.1 of KCK is out — [summary]" — that reply bumps the thread for forum lurkers and creates a stable URL for that release announcement.

### Campaign Forum Thread (Out of Scope, Noted Here)

BadgKatCareer gets its own thread for a different audience: players, not modders. Skip the dependency-chain diagram (CKAN's metapackage handles install order automatically); include screenshots and the contract-progression chart from `CLAUDE.md` instead. Content for that thread is BadgKatCareer-specific design work and is not part of this standard.

## Versioning and Changelog Policy

### SemVer Surface

Each toolkit mod has two version-bearing surfaces:

1. **Public C# API** — `DialogueKit.*`, `CampaignKit.*`, KAK's exposed types, KIK's exposed types when it ships.
2. **Cfg schema** — node names, required fields, expression grammar, condition combinators that downstream cfg authors write against.

Breaking either surface bumps **major**. Adding a new API method or new optional cfg field is **minor**. Internal refactor with same API + same cfg schema is **patch**.

**Content does not version the toolkit.** The four toolkit mods ship zero runtime content; content lives in BadgKatCareer or future content packs. Updating a dialogue scene cfg in BadgKatCareer is a BadgKatCareer release — KDK stays at whatever version.

### CHANGELOG Format

Keep a Changelog (https://keepachangelog.com) format. One file per repo at `CHANGELOG.md`. Sections:

- `## [Unreleased]` — accumulates entries during development.
- `## [<version>] — YYYY-MM-DD` — created at release time by moving the `Unreleased` content under a versioned heading.
- Within each version, subsections in this order: `### Added`, `### Changed`, `### Fixed`, `### Removed`. Omit subsections that have no entries.

The CI pipeline reads the section matching the tagged version verbatim and uses it as both the GitHub release body and the SpaceDock changelog. Write the `[<version>]` section before tagging, or the workflow hard-fails.

### Release Trigger

Tag-driven, hand-bumped:

1. Edit `<ModName>.version` JSON to the new MAJOR.MINOR.PATCH.
2. Move `Unreleased` content under a new `## [<version>] — YYYY-MM-DD` heading in `CHANGELOG.md`.
3. Commit both changes (e.g. `chore: release v0.1.1`).
4. `git tag v0.1.1 && git push --tags`.
5. CI runs, builds, tests, packages, publishes to GitHub release + SpaceDock.

The tag is the source of truth. To roll back: delete the tag and the GitHub release; SpaceDock allows deleting an upload from the mod admin page; CKAN's bot will skip the deleted version on next index.

## Release Order and Cross-Mod Coordination

The toolkit dependency chain is `ChatterBox → KDK → KCK → KAK ← KIK`. When a release of one mod requires a new version of a dependency, the dependency must be released first, in order.

**Example:** KCK adds a feature that requires a new public method on `DialogueKit.Flags`. The release order is:

1. Release KDK first with the new method (e.g. `KDK 0.2.0`).
2. Wait until KDK's GitHub release is live (CI complete) — SpaceDock + CKAN indexing can lag, that's fine.
3. Update KCK's `netkan/KerbalCampaignKit.netkan` `min_version` for KDK to `0.2.0`.
4. Release KCK with the dependency bump (e.g. `KCK 0.2.0`).

**The CI pipeline does not enforce this ordering.** Each repo is independent; cross-repo coordination is a human decision. CKAN's resolver enforces the dependency at install time — if a player tries to install `KCK 0.2.0` and only `KDK 0.1.0` is available, CKAN refuses the install with a clear error.

## Implementation Phasing

The standard rolls out in four phases, in this order:

### Phase 1 — `ksp-libs` repo and CI template
- Create `badgkat/ksp-libs` repo with the KSP and dependency reference DLLs all four toolkits need.
- Author the parameterized `release.yml` template covering the full pipeline from Layer 3.
- Document the `SPACEDOCK_API_TOKEN` and `ksp-libs` deploy-key secrets that each toolkit repo must set.

### Phase 2 — SpaceDock prerequisites
For each of KDK, KCK, KAK:
1. Create the SpaceDock entry under the `badgkat` account. Record the numeric SpaceDock ID. The entry can ship with placeholder content; the long description is updated in Phase 4.
2. Configure repo secrets in the GitHub repo: `SPACEDOCK_API_TOKEN`, plus the deploy key (or PAT) for `ksp-libs` checkout.

This phase must complete before Phase 3 because Phase 3 smoke-tests the upload pipeline.

### Phase 3 — Per-mod skeleton rollout and first release
For each of KDK, KCK, KAK, in dependency order (KDK → KCK → KAK):
1. Add `CHANGELOG.md` with a `[0.1.0]` section describing the current shipped state.
2. Add `.github/workflows/release.yml` from the Phase 1 template.
3. Add `netkan/<ModName>.netkan` with the SpaceDock ID from Phase 2.
4. (KAK only) normalize `.version` schema.
5. Smoke-test the workflow against a throwaway pre-release tag (`v0.0.99-rc<N>`): push the tag, watch the workflow run end-to-end, verify the GitHub release artifact and SpaceDock upload, then delete the test release and retract the tag.
6. Promote the existing `v0.1.0` tag to a real release: delete the local tag, retag the same commit, push. Workflow runs, GitHub release + SpaceDock upload happen.
7. Submit `<ModName>.netkan` PR to `KSP-CKAN/NetKAN`.

### Phase 4 — Public surface
1. Update each SpaceDock entry's long description from each repo's README.
2. Once all three netkan PRs are merged and CKAN is indexing, create the umbrella forum thread per the OP structure in Layer 5.
3. Update each repo's `README.md` to link to its SpaceDock page and the umbrella forum thread.

### Future — KIK at first ship
KIK is the first mod to go through the standard from day one rather than retrofitting. When KIK reaches v0.1.0 release readiness, repeat Phase 2 + Phase 3 + Phase 4 for KIK alone. The umbrella forum thread already exists; the KIK row is updated from "🚧 in development" to released, and the OP gains a KIK section.

## Open Decisions

None remaining at design time. All branching choices were resolved during brainstorming:

- CI-driven release (vs manual or workflow_dispatch).
- Strict GameData-rooted zip including PDBs.
- SpaceDock as primary distribution, GitHub release as mirror.
- Single umbrella forum thread for the four toolkit mods.
- Tag-driven, hand-bumped versioning with Keep a Changelog.
- Centralized `badgkat/ksp-libs` repo for CI references.
- `.netkan` files committed in-repo as source of truth alongside upstream NetKAN-meta entries.

## References

- KSP-CKAN NetKAN repository: https://github.com/KSP-CKAN/NetKAN
- KSP-AVC `.version` file format documented at: https://github.com/linuxgurugamer/KSP-AVC
- Keep a Changelog: https://keepachangelog.com/en/1.1.0/
- SpaceDock API documentation: https://spacedock.info/about
- Existing in-repo specs: `2026-04-29-instructions-kit-design.md` (KIK source design); `2026-04-23-admin-kit-design.md`, `2026-04-23-campaign-kit-design.md`, `2026-04-18-dialogue-kit-design.md` (per-mod source designs)
