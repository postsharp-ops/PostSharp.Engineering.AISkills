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

7. **Proposed release notes**:

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

8. **Proposed Metalama.Compiler release notes** (when creating):
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

1. **Create Metalama.Compiler release** (if compiler version changed and release doesn't exist):
   ```bash
   gh release create release/<COMPILER_VERSION> --repo metalama/Metalama.Compiler --target <COMPILER_COMMIT> --title "<COMPILER_VERSION>" --notes "<NOTES>" [--prerelease]
   ```
   Add `--prerelease` if version contains `-preview` or `-rc`.

2. **Create Metalama release** (with updated notes including compiler reference and Premium issues/commits):
   ```bash
   gh release create release/<VERSION> --target <COMMIT> --title "Metalama <VERSION>" --notes "<NOTES>"
   ```

3. **Close Metalama milestone**:
   ```bash
   gh api repos/metalama/Metalama/milestones/<NUMBER> -X PATCH -f state=closed
   ```

4. **Close Metalama.Compiler milestone** (if one exists for this compiler version):
   ```bash
   gh api repos/metalama/Metalama.Compiler/milestones/<COMPILER_MILESTONE_NUMBER> -X PATCH -f state=closed
   ```

5. **Close Metalama.Premium milestone** (if one exists for this version):
   ```bash
   gh api repos/metalama/Metalama.Premium/milestones/<NUMBER> -X PATCH -f state=closed
   ```

6. **Update project status to "Done"** for each issue:
   ```bash
   # Done option: 98236657
   gh api graphql -f query='mutation { updateProjectV2ItemFieldValue(input: { projectId: "PVT_kwDOC7gkgc4A030b" itemId: "<ITEM_ID>" fieldId: "PVTSSF_lADOC7gkgc4A030bzgqb1vQ" value: { singleSelectOptionId: "98236657" } }) { projectV2Item { id } } }'
   ```

7. **Add release comment** to each issue:
   ```bash
   gh issue comment <NUMBER> --repo metalama/Metalama --body "Released in [<VERSION>](https://github.com/metalama/Metalama/releases/tag/release/<VERSION>).

   — Claude"
   ```

## Release Notes Guidelines

- **Do NOT mention PRs** if PR implements a listed issue
- **Categorize by labels**: `breaking` → Breaking, `enhancement` → New/Enhancements, `bug` → Fixes
- **Use full issue links**: `[#1247](https://github.com/metalama/Metalama/issues/1247)`
- **Sign comments**: `— Claude`
- **Metalama.Compiler release notes**: Built from commit history, excluding `<<VERSION_BUMP>>`, `<<AUTO_UPDATED_VERSIONS>>`, merge commits, and `Update eng` commits
- **Compiler reference in Metalama notes**: Only include when compiler version changed between releases
- **Metalama.Premium issues**: Include in Metalama release notes under normal categories. Use Metalama.Premium issue links for Premium-only issues; use Metalama issue links when commits reference Metalama issues
- **Metalama.Premium commits**: Meaningful commits (not eng updates, version bumps, or merges) that don't reference any issue should be mentioned as bullet points in the Metalama release notes
