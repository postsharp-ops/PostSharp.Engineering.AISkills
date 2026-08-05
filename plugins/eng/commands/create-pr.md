---
description: Create a pull request with proper metadata, milestone, and issue linking
argument-hint: "[issue numbers to close]"
---

# Create Pull Request

Create a PR with proper metadata, milestone, and issue linking for Metalama/PostSharp repos.

## Arguments

$ARGUMENTS - Optional issue numbers to close (e.g., `1234 1235`)

## Workflow

1. **Commit and push** any remaining changes

2. **Check for breaking changes** compared to `develop/YYYY.N`:
   - If breaking: add `breaking` label to PR and linked issues
   - Add comment describing the breaking change

3. **Create PR** targeting `develop/YYYY.N` (NOT default branch):
   ```bash
   gh pr create --base develop/YYYY.N --title "<title>" --body "<body>"
   ```

   PR body format:
   ```
   ## Summary
   - Bullet points describing key changes

   ## Breaking Changes (if applicable)
   - List new interface members or behavioral changes

   ## Issues Fixed
   Fixes #1226 - Brief description
   Fixes #1232 - Brief description
   ```

   **Each issue MUST get a closing keyword** (`Fixes`/`Closes`/`Resolves` + `#NNNN`). A bare `- #1226` reference does not link the issue, and step 6 cannot link it later — GitHub only parses closing keywords from the body.

   **NO test plan section** - tests are verified through CI.

4. **Assign to milestone** - find latest open `YYYY.N.B-suffix`:
   ```bash
   gh api "repos/metalama/Metalama/milestones?state=all&per_page=100" --jq '.[] | "\(.number) \(.title) - \(.state)"' | grep "YYYY.N" | sort -V

   # Assign (use milestone NUMBER)
   gh api repos/metalama/Metalama/issues/<PR_NUMBER> -X PATCH -f milestone=<NUMBER>
   ```

   If no open milestone exists, propose creating one.

5. **Assign to current user**:
   ```bash
   gh api repos/metalama/Metalama/issues/<PR_NUMBER> -X PATCH -f assignees[]="<username>"
   ```

6. **Link issues** (workaround for non-default base branch):

   GitHub only parses closing keywords when the PR base is the repo's **default** branch (`release/YYYY.N`). PRs target `develop/YYYY.N`, so the keywords are ignored on creation. Toggling the base through the default branch makes GitHub parse the body; the links then persist when you toggle back.

   **This toggle only links issues that already have a closing keyword in the body (step 3).** With a bare `#NNNN` reference it is a silent no-op — it succeeds and links nothing.

   ```bash
   # Get PR node ID
   PR_ID=$(gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { pullRequest(number: <PR_NUMBER>) { id } } }' --jq '.data.repository.pullRequest.id')

   # Temporarily change base to release branch (GitHub default), then back - links persist!
   gh api graphql -f query="mutation { updatePullRequest(input: { pullRequestId: \"$PR_ID\" baseRefName: \"release/YYYY.N\" }) { pullRequest { id } } }"
   gh api graphql -f query="mutation { updatePullRequest(input: { pullRequestId: \"$PR_ID\" baseRefName: \"develop/YYYY.N\" }) { pullRequest { id } } }"
   ```

   **Verify - do not assume the toggle worked:**
   ```bash
   gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { pullRequest(number: <PR_NUMBER>) { baseRefName closingIssuesReferences(first: 10) { nodes { number } } } } }' \
     --jq '.data.repository.pullRequest | "base=\(.baseRefName) linked=\(.closingIssuesReferences.nodes | map(.number) | join(","))"'
   ```
   Expect `base=develop/YYYY.N` and every issue listed. If `linked=` is empty, the body is missing a closing keyword: fix the body with `gh pr edit`, then re-run the toggle.

7. **Set project status to "In Review"**:
   ```bash
   # Get project item ID
   ITEM_ID=$(gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { pullRequest(number: <PR_NUMBER>) { projectItems(first: 10) { nodes { id project { title } } } } } }' --jq '.data.repository.pullRequest.projectItems.nodes[] | select(.project.title == "Development") | .id')

   # Set status (In review: 4cc61d42)
   gh api graphql -f query="mutation { updateProjectV2ItemFieldValue(input: { projectId: \"PVT_kwDOC7gkgc4A030b\" itemId: \"$ITEM_ID\" fieldId: \"PVTSSF_lADOC7gkgc4A030bzgqb1vQ\" value: { singleSelectOptionId: \"4cc61d42\" } }) { projectV2Item { id } } }"
   ```

8. **Trigger TeamCity build**: Use `/eng:tc-build` or ask user.

## Error Handling

- **No open milestone**: Propose creating one with format `YYYY.N.B-rc`
- **PR not in Development project**: Skip project status update, note in output
- **`INSUFFICIENT_SCOPES` on step 7**: The `gh` token needs the `read:project` scope to read or set project fields. Skip the status update and tell the user to add the scope — the rest of the workflow is unaffected.
- **Issue linking fails**: Re-check that the body uses a closing keyword, not a bare `#NNNN`. If it still fails, note that issues will need manual linking after merge.
