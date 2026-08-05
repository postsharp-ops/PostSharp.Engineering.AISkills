# GitHub Workflow

## Starting Work on an Issue

1. **Read the issue** — fetch the full issue content from GitHub
2. **Check documentation** — look for related conceptual docs
3. **Create the branch** — `topic/YYYY.N/XXXX-short-description`
4. **Assign the issue** to the user in GitHub
5. **Set the milestone** — the latest open milestone for the current `YYYY.N` version, in the `YYYY.N.B-suffix` format below
6. **Set the issue status** to the org's "In progress" option in its Development project — resolve the IDs for that org first, since they differ per org
7. **Track progress** in an `<issue-number>-TODO.md` file (do not commit it)
8. **File bugs promptly** — create issues as soon as bugs are discovered during development

## Milestones

Always use the `YYYY.N.B-suffix` format (e.g. `2026.0.8-rc`), never `YYYY.N` alone.

Suffix conventions:

- `-preview` — early preview releases
- `-rc` — release candidates
- no suffix — stable releases

Rules:

- Never assign anything to a closed milestone
- Never reopen a closed milestone
- Instead, propose that the user create a new milestone with an incremented version number

## Development Project IDs

**Every one of these IDs is per-organization.** Never reuse one org's project ID, status field ID or option IDs against another org — they are unrelated values, and the status *option names* differ between orgs as well. Resolve the IDs for the org of the repository being worked on.

| Org | Project | Project ID | Status field ID |
|---|---|---|---|
| `metalama` | Development (#1) | `PVT_kwDOC7gkgc4A030b` | `PVTSSF_lADOC7gkgc4A030bzgqb1vQ` |
| `postsharp` | Development (#1) | `PVT_kwDOABWZoc4AWAr1` | `PVTSSF_lADOABWZoc4AWAr1zgOEQdY` |

Status options in `metalama`: Backlog `f75ad846`, Planned `08afe404`, In progress `47fc9ee4`, In review `4cc61d42`, Merged `9d4ab054`, Done `98236657`.

Status options in `postsharp` — note the different set and the emoji in the names: 🆕 New `86721d1e`, 📋 Backlog `288dd1c8`, 🔖 Ready `08326279`, 🏗 In progress `ea11b64f`, 👀 In review `7efe817f`, ✅ Done `9801334b`.

`postsharp-ops` and `postsharp-contrib` have no ProjectV2 board, so there is no status to set for issues in those orgs.

To resolve the IDs for any org — do this rather than guessing, and to confirm the table above is still current:

```bash
ORG=metalama

# Projects in the org
gh api graphql -f query="{ organization(login: \"$ORG\") { projectsV2(first: 10) { nodes { id title number closed } } } }" \
  --jq '.data.organization.projectsV2.nodes[] | "\(.number)\t\(.id)\t\(.title)"'

# Status field and its options, for project number 1
gh api graphql -f query="{ organization(login: \"$ORG\") { projectV2(number: 1) { field(name: \"Status\") { ... on ProjectV2SingleSelectField { id options { id name } } } } } }" \
  --jq '.data.organization.projectV2.field | "field=\(.id)", (.options[] | "  \(.name)\t\(.id)")'
```

## Getting Node IDs

```bash
# PR node ID
gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { pullRequest(number: 1228) { id } } }' --jq '.data.repository.pullRequest.id'

# Issue node ID
gh api graphql -f query='{ repository(owner: "metalama", name: "Metalama") { issue(number: 1226) { id } } }' --jq '.data.repository.issue.id'
```

## Breaking Changes

The `breaking` label is for **user-facing** breaking changes only — a change that breaks code our users have written against our public API.

- **Do not label a break in an internal API.** Internal APIs are ours to change; those breaks are not tracked and the label is noise on them. This is the common mistake — when in doubt, ask whether a user outside the team could have depended on the thing that changed.
- Adding members to interfaces marked `[InternalImplement]` (including inherited members) is **not** a breaking change.
- For a genuine user-facing break: add a comment to the issue describing the change, and add the `breaking` label to the issue and the PR.
