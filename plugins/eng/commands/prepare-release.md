---
description: Prepare a GitHub release for a milestone with proper release notes
argument-hint: "<milestone URL or number>"
---

# Prepare GitHub Release

Prepare a GitHub release for a milestone. Requires user approval before execution.

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
   - Check if a matching milestone exists and list its issues:
     ```bash
     gh issue list --repo metalama/Metalama.Premium --milestone "<VERSION>" --state all --json number,title,state,labels
     ```
   - Issues from Metalama.Premium will be included in the Metalama release notes

8. **Review what each issue actually shipped**. An issue title is written when the defect is *reported*, so it usually describes the symptom or the first suspected cause. The real defect, and the user-visible effect of the fix, are often known only once the work is done — which means the closing PR, not the issue title, is the accurate source for a release note.

   For every issue that will appear in the notes, fetch the issue body and its closing PR:
   ```bash
   gh issue view <NUMBER> --repo metalama/Metalama --json title,body,labels,closedByPullRequestsReferences
   gh pr view <PR_NUMBER> --repo metalama/Metalama --json title,body
   ```

   From the PR description, establish: what the user observed, under what conditions, and what changed. Write the release note item from that — not by paraphrasing the issue title.

   Flag for renaming any issue whose title no longer matches what shipped, or that is unintelligible to a user (see "Writing Release Note Items" below). Collect these as rename proposals for Phase 2; never rename silently.

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

6. **Premium status**: Whether there are meaningful commits or issues
   - List any issues/commits to be included in Metalama release notes

7. **Proposed issue renames**: every issue whose title should change, as a table of issue number, current title, proposed title, and the reason (stale after implementation, or not intelligible to a user). Renaming is part of the release, not an afterthought: the title is what the reader sees when following the link from the notes, so a note that reads well next to a title that contradicts it is still a failure. **Wait for approval on each rename** — never rename an issue that the user has not approved.

8. **Proposed release notes**:

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

9. **Proposed Metalama.Compiler release notes** (when creating):
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
   gh issue edit <NUMBER> --repo metalama/Metalama --title "<APPROVED TITLE>"
   ```
   Apply only the renames the user approved, with the exact wording approved.

2. **Create Metalama.Compiler release** (if compiler version changed and release doesn't exist):
   ```bash
   gh release create release/<COMPILER_VERSION> --repo metalama/Metalama.Compiler --target <COMPILER_COMMIT> --title "<COMPILER_VERSION>" --notes "<NOTES>" [--prerelease]
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

7. **Update project status to "Done"** for each issue:
   ```bash
   # Done option: 98236657
   gh api graphql -f query='mutation { updateProjectV2ItemFieldValue(input: { projectId: "PVT_kwDOC7gkgc4A030b" itemId: "<ITEM_ID>" fieldId: "PVTSSF_lADOC7gkgc4A030bzgqb1vQ" value: { singleSelectOptionId: "98236657" } }) { projectV2Item { id } } }'
   ```

8. **Add release comment** to each issue:
   ```bash
   gh issue comment <NUMBER> --repo metalama/Metalama --body "Released in [<VERSION>](https://github.com/metalama/Metalama/releases/tag/release/<VERSION>).

   — Claude"
   ```

## Writing Release Note Items

Release notes are read by users, not by the team. Every item must let a reader decide **whether the change affects them**, without knowing our internal types, methods, or file layout. An item that only makes sense to whoever fixed it has failed, however accurate it is.

### Opening verb

Start each item with a past-tense verb naming what changed for the user:

| Category | Opening verb | Example |
|---|---|---|
| Fixes | **Fixed …** | Fixed a design-time crash when a project contains duplicate project references. |
| Enhancements | **Improved …**, **Extended …** | Improved crash reports, which now capture the loader exceptions behind a type-load failure. |
| New | **Added …** | Added a public `IDurableRef` interface family. |
| Breaking Changes | **Removed …**, **Renamed …**, **Changed …** | Renamed the test synchronization hooks into the `Metalama.Testing.Hooks` package. |

Stating the new behaviour in the present tense is also acceptable where it reads better — "Crash reports now capture …", "The compile-time compilation no longer force-defines `NETSTANDARD_2_0`." What is never acceptable is a bare symptom or a bare cause: "`KeyNotFoundException` in `SplitResultsByTree`" tells a user nothing.

### The reader test

Before accepting an item, check it against a user who does not work on Metalama:

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

When the shipped fix does not match the title — the usual case when the reported symptom turned out to have a different cause — propose a new title alongside the release note item. Rules:

- Rewrite the title to describe **the defect that was actually fixed**, in the user's terms, not the symptom first reported.
- Keep the title a statement of the problem (the issue records a defect); the release note states the resolution. Title: "Design-time crash in projects with linked files." Note: "Fixed a design-time crash affecting projects with linked or generated files."
- Do not rename an issue merely because its wording is awkward; rename when it is **wrong**, **stale**, or **unintelligible to a user**.
- Never rename without approval, and never edit the issue body.

### Exclusions

Issues with no user-visible effect do not belong in the notes at all, whatever their milestone: those labelled `test-only`, flaky-test fixes, and pure build or engineering changes. Report them in Phase 2 as deliberately excluded so the omission is a decision rather than an oversight.

## Release Notes Guidelines

- **Do NOT mention PRs** if PR implements a listed issue
- **Categorize by labels**: `breaking` → Breaking, `enhancement` → New/Enhancements, `bug` → Fixes
- **Use full issue links**: `[#1247](https://github.com/metalama/Metalama/issues/1247)`
- **Sign comments**: `— Claude`
- **Metalama.Compiler release notes**: Built from commit history, excluding `<<VERSION_BUMP>>`, `<<AUTO_UPDATED_VERSIONS>>`, merge commits, and `Update eng` commits
- **Compiler reference in Metalama notes**: Only include when compiler version changed between releases
- **Metalama.Premium issues**: Include in Metalama release notes under normal categories. Use Metalama.Premium issue links for Premium-only issues; use Metalama issue links when commits reference Metalama issues
- **Metalama.Premium commits**: Meaningful commits (not eng updates, version bumps, or merges) that don't reference any issue should be mentioned as bullet points in the Metalama release notes
