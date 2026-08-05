---
description: Prepare a GitHub release for a milestone with proper release notes
argument-hint: "<milestone URL or number>"
---

# Prepare GitHub Release

Prepare a GitHub release for a milestone. Requires user approval before execution.

## Products and Repositories

Four products are versioned and released independently, each with its own release notes. Tag and title conventions differ per product — check this table before creating any release:

| Product | Release + issues repo | Tag | Release title |
|---|---|---|---|
| Metalama | `metalama/Metalama` | `release/2026.1.21` | `Metalama 2026.1.21` |
| Metalama.Compiler | `metalama/Metalama.Compiler` | `release/2026.1.13` | `Metalama.Compiler 2026.1.13` |
| PostSharp | `postsharp/PostSharp.Public` | `v2026.0.15` | `PostSharp 2026.0.15` |
| Metalama.Vsx | `metalama/Metalama.Vsx.Public` | `release/2026.1.5` | `PostSharp and Metalama VSX 2026.1.5` |

**The title is always `<product prefix> <VERSION>`, regardless of what earlier releases used.** Several published releases predate this rule — Metalama.Compiler titled `2026.1.13`, PostSharp titled `v2026.0.15`. Do not copy the previous release's title. The version in the title carries no `v` prefix and no `release/` prefix even where the tag does.

PostSharp and Metalama.Vsx are built from private repos (`postsharp/PostSharp`, `metalama/Metalama.Vsx`); the `.Public` repo is where their issues, milestones and releases live. Work in the public repo for both.

The default target of this command is the **Metalama** release, described in Phases 1–3, plus the Metalama.Compiler release when the compiler version changed. For the other two, follow those phases with the deltas in "PostSharp Release" and "Metalama.Vsx Release" below.

Metalama is not one repository. It is several repositories sharing one version, released together, covered by one set of release notes in `metalama/Metalama`. Inspect the issues and milestones of all three:

| Repository | Visibility | Notes |
|---|---|---|
| `metalama/Metalama` | public | Own issues; the milestone the release is named for |
| `metalama/Metalama.Premium` | private | Usually has no issues — `metalama/Metalama` is its issue tracker, so its commit log is the reliable signal |
| `metalama/Metalama.Community` | public | Own milestone per version (e.g. `2026.1.22`), with issues |

Metalama.Compiler is a separate product, not a Metalama repo: it gets its own release and notes, which the Metalama notes reference when the version changed.

## Arguments

$ARGUMENTS - Milestone URL or number (e.g., `https://github.com/metalama/Metalama/milestone/31` or `31`)

## Phase 1: Analysis

Gather all information first:

1. **Fetch milestone details**:
   ```bash
   gh api repos/metalama/Metalama/milestones/<NUMBER> --jq '{title, state, open_issues, closed_issues}'
   ```

2. **List all issues** (open and closed):
   ```bash
   gh issue list --repo metalama/Metalama --milestone "<TITLE>" --state all --json number,title,state,labels
   ```

3. **Check project status** for each issue:
   ```bash
   gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { issue(number: <NUMBER>) { projectItems(first: 10) { nodes { id project { title } fieldValues(first: 10) { nodes { ... on ProjectV2ItemFieldSingleSelectValue { name field { ... on ProjectV2SingleSelectField { name } } } } } } } } } }'
   ```

4. **Check branch status**:
   - Compare `develop/YYYY.N` with `release/YYYY.N`
   - Check version bump in MainVersion.props
   - Identify previous release tag

5. **Check upstream version lines** (e.g., 2025.1 merged into 2026.0):
   ```bash
   # Latest releases per version line
   git tag -l "release/2025.1.*" --sort=-v:refname | head -1
   git tag -l "release/2026.0.*" --sort=-v:refname | head -1

   # Check ancestry
   git merge-base --is-ancestor release/<UPSTREAM> <TARGET_COMMIT>
   git merge-base --is-ancestor release/<UPSTREAM> release/<PREV_VERSION>
   ```

6. **Check Metalama.Compiler version change**:
   - Fetch `eng/AutoUpdatedVersions.props` at the current and previous Metalama release tags:
     ```bash
     gh api repos/metalama/Metalama/contents/eng/AutoUpdatedVersions.props?ref=release/<VERSION> --jq '.content' | base64 -d
     gh api repos/metalama/Metalama/contents/eng/AutoUpdatedVersions.props?ref=release/<PREV_VERSION> --jq '.content' | base64 -d
     ```
   - Extract `MetalamaCompilerVersion` from each
   - If they differ:
     - Fetch commit log between the two compiler tags:
       ```bash
       gh api repos/metalama/Metalama.Compiler/compare/release/<PREV_COMPILER>...release/<CURRENT_COMPILER> --jq '.commits[] | {sha: .sha[:8], message: .commit.message}'
       ```
     - Summarize meaningful commits (exclude `<<VERSION_BUMP>>`, `<<AUTO_UPDATED_VERSIONS>>`, merge commits, `Update eng` commits)
     - Check for issues referenced in commits (from either repo)
     - Check if a release already exists:
       ```bash
       gh release view --repo metalama/Metalama.Compiler release/<CURRENT_COMPILER>
       ```
     - Find previous compiler release tag for the "Based on" link
     - Check if a Metalama.Compiler milestone exists for this compiler version:
       ```bash
       gh api repos/metalama/Metalama.Compiler/milestones --jq '.[] | select(.title == "<CURRENT_COMPILER>") | {number, title, state, open_issues, closed_issues}'
       ```

7. **Check Metalama.Premium changes**:
   - Fetch commit log between matching Metalama.Premium release tags:
     ```bash
     gh api repos/metalama/Metalama.Premium/compare/release/<PREV_VERSION>...release/<VERSION> --jq '.commits[] | {sha: .sha[:8], message: .commit.message}'
     ```
   - Summarize meaningful commits (same exclusion rules: `<<VERSION_BUMP>>`, `<<AUTO_UPDATED_VERSIONS>>`, merge commits, `Update eng` commits)
   - Check whether a matching milestone exists and list its issues:
     ```bash
     gh issue list --repo metalama/Metalama.Premium --milestone "<VERSION>" --state all --json number,title,state,labels
     ```
   - Expect no issues: Premium work is normally tracked in `metalama/Metalama`. An empty milestone is not evidence that Premium is unchanged — use the commit log for that
   - Any Premium issues that do exist go in the Metalama release notes, linked to the Premium repo

8. **Check Metalama.Community changes**. Community shares Metalama's version and has its own milestone per version:
   ```bash
   gh api repos/metalama/Metalama.Community/milestones --jq '.[] | {number, title, state, open_issues, closed_issues}'
   gh issue list --repo metalama/Metalama.Community --milestone "<VERSION>" --state all --json number,title,state,labels
   ```
   - The Community milestone may carry a different patch number; match on the version being released, and ask if ambiguous
   - Its issues go in the Metalama release notes under the normal categories, linked to the Community repo: `[Metalama.Community#98](https://github.com/metalama/Metalama.Community/issues/98)`

9. **Review what each issue actually shipped**. A title is written when the defect is reported, so it describes the symptom or the first suspected cause; the real defect is often known only once the work is done. Use the closing PR, not the title, as the source for the note.

   For every issue that will appear in the notes — from any of the three repos — fetch the issue body and its closing PR. **`--repo` must be the issue's own repo**, since the same number exists in all of them:
   ```bash
   gh issue view <NUMBER> --repo <ISSUE_REPO> --json title,body,labels,closedByPullRequestsReferences
   gh pr view <PR_NUMBER> --repo <PR_REPO> --json title,body
   ```

   From the PR description, establish what the user observed, under what conditions, and what changed. Write the item from that, not by paraphrasing the title.

   Flag for renaming any issue whose title no longer matches what shipped, or is unintelligible to a user (see "Writing Release Note Items"). Collect these for Phase 2; never rename silently.

## Phase 2: Present Plan

Present to user for approval:

1. **Milestone status**: Open/closed issues, blockers
2. **Issues**: Categorized as Breaking/New/Enhancements/Fixes
3. **Branch status**: Sync state, version bump
4. **Project status**: Flag issues not in "Merged" or "Done"

5. **Compiler status**: Whether compiler version changed, from/to versions
   - If changed and no release exists: show proposed Metalama.Compiler release notes
   - If changed and release already exists: note that it will be referenced in Metalama notes
   - If unchanged: note that compiler reference will not be included

6. **Premium and Community status**: report all three repos, including those with nothing to report, so an empty result is distinguishable from an unchecked repo
   - Premium: meaningful commits, and any issues
   - Community: milestone and its issues

7. **Excluded issues**: issues in the milestones that will not appear in the notes, with the reason (see "Exclusions")

8. **Proposed issue renames**: a table of issue number, current title, proposed title, and reason (stale after implementation, or unintelligible to a user). The title is what a reader sees on following the link, so it has to match the note. **Wait for approval on each rename.**

9. **Proposed release notes**:

   Single base:
   ```
   Metalama <VERSION> is based on [<PREV>](https://github.com/metalama/Metalama/releases/tag/release/<PREV>), plus the following changes.
   ```

   Multiple bases (upstream merged):
   ```
   Metalama <VERSION> is based on [<PREV_SAME_LINE>](...) and [<UPSTREAM>](...), plus the following changes.
   ```

   When compiler version changed, add after the "based on" paragraph:
   ```
   This release updates Metalama.Compiler to [<COMPILER_VERSION>](https://github.com/metalama/Metalama.Compiler/releases/tag/release/<COMPILER_VERSION>).
   ```

   Then:
   ```
   ### Breaking Changes
   - [#XXXX](https://github.com/metalama/Metalama/issues/XXXX) Description

   ### New
   - [#XXXX](https://github.com/metalama/Metalama/issues/XXXX) Description

   ### Enhancements
   - [#XXXX](https://github.com/metalama/Metalama/issues/XXXX) Description

   ### Fixes
   - [#XXXX](https://github.com/metalama/Metalama/issues/XXXX) Description

   ### Resources
   - [Milestone](https://github.com/metalama/Metalama/milestone/<NUMBER>?closed=1)
   - **Full Changelog**: [release/<PREV>...release/<VERSION>](https://github.com/metalama/Metalama/compare/release/<PREV>...release/<VERSION>)
   ```

   Metalama.Premium issues are included in the normal categories above with links to the Premium repo: `[#XX](https://github.com/metalama/Metalama.Premium/issues/XX)`. Premium commits that reference Metalama issues should use Metalama issue links instead.

10. **Proposed Metalama.Compiler release notes** (when creating):
   ```
   **Release date:** YYYY-MM-DD

   Based on [<PREV_COMPILER>](https://github.com/metalama/Metalama.Compiler/releases/tag/release/<PREV_COMPILER>).

   - <bullet points from commit messages>

   ### Resources
   - [Milestone](https://github.com/metalama/Metalama.Compiler/milestone/<COMPILER_MILESTONE_NUMBER>?closed=1)
   ```

**STOP and wait for user approval.**

## Phase 3: Execute

After approval:

1. **Apply approved issue renames** (before creating the release, so the links in the notes resolve to the corrected titles):
   ```bash
   gh issue edit <NUMBER> --repo <ISSUE_REPO> --title "<APPROVED TITLE>"
   ```
   Apply only the renames the user approved, with the exact wording approved. Pass the issue's own repo — a number from Community or Premium addresses a different issue in `metalama/Metalama`.

2. **Create Metalama.Compiler release** (if compiler version changed and release doesn't exist):
   ```bash
   gh release create release/<COMPILER_VERSION> --repo metalama/Metalama.Compiler --target <COMPILER_COMMIT> --title "Metalama.Compiler <COMPILER_VERSION>" --notes "<NOTES>" [--prerelease]
   ```
   Add `--prerelease` if version contains `-preview` or `-rc`.

3. **Create Metalama release** (with updated notes including compiler reference and Premium issues/commits):
   ```bash
   gh release create release/<VERSION> --target <COMMIT> --title "Metalama <VERSION>" --notes "<NOTES>"
   ```

4. **Close Metalama milestone**:
   ```bash
   gh api repos/metalama/Metalama/milestones/<NUMBER> -X PATCH -f state=closed
   ```

5. **Close Metalama.Compiler milestone** (if one exists for this compiler version):
   ```bash
   gh api repos/metalama/Metalama.Compiler/milestones/<COMPILER_MILESTONE_NUMBER> -X PATCH -f state=closed
   ```

6. **Close Metalama.Premium milestone** (if one exists for this version):
   ```bash
   gh api repos/metalama/Metalama.Premium/milestones/<NUMBER> -X PATCH -f state=closed
   ```

7. **Close Metalama.Community milestone** (if one exists for this version):
   ```bash
   gh api repos/metalama/Metalama.Community/milestones/<NUMBER> -X PATCH -f state=closed
   ```

8. **Update project status to "Done"** for each issue:
   ```bash
   # Done option: 98236657
   gh api graphql -f query='mutation { updateProjectV2ItemFieldValue(input: { projectId: "PVT_kwDOC7gkgc4A030b" itemId: "<ITEM_ID>" fieldId: "PVTSSF_lADOC7gkgc4A030bzgqb1vQ" value: { singleSelectOptionId: "98236657" } }) { projectV2Item { id } } }'
   ```

9. **Add release comment** to each issue:
   ```bash
   gh issue comment <NUMBER> --repo <ISSUE_REPO> --body "Released in [<VERSION>](https://github.com/metalama/Metalama/releases/tag/release/<VERSION>).

   — Claude"
   ```

## PostSharp Release

Prepared in `postsharp/PostSharp.Public`, which holds the issues, the milestones and the releases for the private `postsharp/PostSharp`.

Deltas from the Metalama workflow:

- **Pass `--repo postsharp/PostSharp.Public` to every `gh` command**, including `gh release create` — the Phase 3 commands omit `--repo` and would otherwise act on the current directory's repo.
- **Tag is `v<VERSION>`**, not `release/<VERSION>`. The title is `PostSharp <VERSION>` — not the tag, and without the `v`, even though every published PostSharp release so far is titled after its tag.
- **Two version lines are live at once** (`2024.0.x` and `2026.0.x`), and fixes are forward-ported from the older to the newer. When a release forward-ports, cite both bases and link the upstream milestone as such.
- Single base:
  ```
  PostSharp <VERSION> is based on [v<PREV>](https://github.com/postsharp/PostSharp.Public/releases/tag/v<PREV>), plus the following changes.
  ```
- Forward-port, with a paragraph naming what was carried over:
  ```
  PostSharp <VERSION> is based on [v<PREV>](…/v<PREV>) and [v<UPSTREAM>](…/v<UPSTREAM>).

  This release forward-ports the upstream [v<UPSTREAM>](…/v<UPSTREAM>) fixes to the <LINE> line: <summary>.

  ### Resources
  - [Milestone <UPSTREAM>](https://github.com/postsharp/PostSharp.Public/milestone/<N>?closed=1) (upstream)
  ```
- Categories, item wording, and the `### Resources` milestone link are as for Metalama.

## Metalama.Vsx Release

Prepared in `metalama/Metalama.Vsx.Public`, which holds the issues, the milestones and the releases for the private `metalama/Metalama.Vsx`. One VSIX ships the Visual Studio tooling for **both** PostSharp and Metalama, which is why the title names both.

Deltas from the Metalama workflow:

- **Pass `--repo metalama/Metalama.Vsx.Public` to every `gh` command**, including `gh release create`.
- Tag is `release/<VERSION>`; the title is `PostSharp and Metalama VSX <VERSION>`.
- Header sentence:
  ```
  PostSharp and Metalama VSX <VERSION> is based on [<PREV>](https://github.com/metalama/Metalama.Vsx.Public/releases/tag/release/<PREV>), plus the following changes.
  ```
- **Add a lead paragraph** stating what the release is about when it has a theme (a Metalama version adoption, a security pass), before the categories.
- **Custom category headings are allowed** where they group the work better than Breaking/New/Enhancements/Fixes — for example `### Privacy & telemetry`, `### Security hardening`, `### Other changes`. Item wording rules are unchanged.
- **Add an upstream-versions section** listing the versions this VSIX consumes, one line per product and per live Metalama version line:
  ```
  ## Upstream versions
  - PostSharp: [<VERSION>](https://github.com/postsharp/PostSharp.Public/releases/tag/v<VERSION>)
  - Metalama <LINE>: [<VERSION>](https://github.com/metalama/Metalama/releases/tag/release/<VERSION>)
  ```
- **Resources include the VSIX download**, alongside the milestone:
  ```
  - [Download PostSharpMetalama.<VERSION>.vsix](https://www.postsharp.net/downloads/vsx/vsx-<MAJOR.MINOR>/v<VERSION>/PostSharpMetalama.<VERSION>.vsix)
  ```
- Prereleases are tagged `release/<VERSION>-rc` and `release/<VERSION>-preview`; pass `--prerelease`.

## Writing Release Note Items

Release notes are read by users, not by the team. Every item must let a reader decide whether the change affects them, without knowing our internal types, methods or file layout.

### Opening verb

Start each item with a past-tense verb naming what changed for the user:

| Category | Opening verb | Example |
|---|---|---|
| Fixes | **Fixed …** | Fixed a design-time crash when a project contains duplicate project references. |
| Enhancements | **Improved …**, **Extended …** | Improved crash reports, which now capture the loader exceptions behind a type-load failure. |
| New | **Added …** | Added a public `IDurableRef` interface family. |
| Breaking Changes | **Removed …**, **Renamed …**, **Changed …** | Renamed the test synchronization hooks into the `Metalama.Testing.Hooks` package. |

Present tense is fine where it reads better — "Crash reports now capture …", "The compile-time compilation no longer force-defines `NETSTANDARD_2_0`". A bare symptom or cause is not: "`KeyNotFoundException` in `SplitResultsByTree`" tells a user nothing.

### The reader test

Check each item against a user who does not work on Metalama:

1. **Can they tell whether it affects them?** Name the user-visible situation — design time, a specific target framework, a configuration file, a particular API.
2. **Is every term one they can encounter?** Public API names, MSBuild properties, configuration keys and diagnostic IDs are fine. Internal class and method names are not.
3. **Is the effect stated, not just the mechanism?** "A race in the disposal logic" is a cause; "cached values could be lost when the Redis connection was disposed" is an effect.

### Examples

Real issue titles, and what belongs in the notes:

| Issue title (as reported) | Release note item | Why the title fails |
|---|---|---|
| Design-time pipeline crashes: `KeyNotFoundException` on a syntax-tree path key in `SplitResultsByTree` | Fixed a design-time crash affecting projects with linked or generated files. | Names an internal method and an exception type. A user cannot tell whether their project is affected. |
| A `diagnostics.json` without a `logging` member is silently discarded (`NullReferenceException` in `DiagnosticsConfiguration.Validate`) | Fixed the silent loss of a `diagnostics.json` file that has no `logging` section. | Leads with the internal cause; the user-visible effect (settings ignored) is buried in parentheses. |
| Redis: potential race in the disposal logic | Fixed a race condition that could cause cached values to be lost when a Redis connection was disposed. | States a mechanism, not a consequence. "Potential" leaves the reader unable to judge the risk. |
| Excessive RAM usage by `devhub.exe` process is triggered by use of Metalama | Fixed excessive memory usage of the `devhub.exe` process when a solution uses Metalama. | Acceptable as-is — the symptom, the process and the trigger are all user-observable. |

Good items already published, for reference in tone:

- Fixed a permanent hang of the design-time RPC server when a client disconnected while connecting.
- Fixed a design-time `InvalidCastException` when contract options were merged across two TFM-specific compile-time assemblies.
- Telemetry is now correctly disabled for a project located outside any git repository.

### Renaming issues

When the shipped fix does not match the title, propose a new title alongside the note.

- Describe the defect that was actually fixed, in the user's terms, not the symptom first reported.
- Keep the title a statement of the problem; the note states the resolution. Title: "Design-time crash in projects with linked files." Note: "Fixed a design-time crash affecting projects with linked or generated files."
- Rename when a title is wrong, stale or unintelligible to a user — not merely awkward.
- Never rename without approval, and never edit the issue body.

### Exclusions

Keep out of the notes, whatever the milestone says:

- **`unpublished`** — the defect only ever affected a build or commit that was never published, so no user can have hit it. Reporting it would describe a bug that never reached anyone.
- **`test-only`**, flaky-test fixes, and pure build or engineering changes — no user-visible effect.

List the exclusions in Phase 2, so the omission is visible.

## Release Notes Guidelines

- **Do NOT mention PRs** if PR implements a listed issue
- **Categorize by labels**: `breaking` → Breaking, `enhancement` → New/Enhancements, `bug` → Fixes
- **Use full issue links**: `[#1247](https://github.com/metalama/Metalama/issues/1247)`
- **Sign comments**: `— Claude`
- **Metalama.Compiler release notes**: Built from commit history, excluding `<<VERSION_BUMP>>`, `<<AUTO_UPDATED_VERSIONS>>`, merge commits, and `Update eng` commits
- **Compiler reference in Metalama notes**: Only include when compiler version changed between releases
- **Metalama.Premium issues**: Include in Metalama release notes under normal categories. Use Metalama.Premium issue links for Premium-only issues; use Metalama issue links when commits reference Metalama issues (the usual case, since Metalama is Premium's issue tracker)
- **Metalama.Premium commits**: Meaningful commits (not eng updates, version bumps, or merges) that don't reference any issue should be mentioned as bullet points in the Metalama release notes
- **Metalama.Community issues**: Include under the normal categories, linked to the Community repo — `[Metalama.Community#98](https://github.com/metalama/Metalama.Community/issues/98)`
