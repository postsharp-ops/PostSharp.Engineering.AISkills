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

## Issue Types

Classify every issue with its GitHub **issue type**, never with a label. The `bug` and `enhancement` labels were deleted from all `postsharp` and `metalama` repositories on 2026-10-10; do not recreate them.

Set the type when creating or triaging an issue. The release notes are categorized by it, so an untyped issue in a milestone holds up the release.

### Which type to use

| Type | Orgs | Use when | Examples | Release notes |
|---|---|---|---|---|
| `Bug` | both | The product does not behave as documented, as designed, or as it did before | Crash, wrong output, wrong or missing diagnostic, regression (including a performance regression), incorrect documentation | Fixes |
| `Enhancement` | both | An existing feature works as designed, and the change makes it better without adding a concept the user must learn | A new option on an existing aspect, a clearer error message, a performance improvement, support for a new .NET SDK or IDE version, a new documentation article | Enhancements |
| `Feature` | both | Users gain a capability they did not have — a new API, aspect, pattern or integration they must learn to use | A new pattern library, a new aspect kind, a new public API surface | New |
| `User Story` | both | One user goal that spans several issues or PRs. It is the parent; each child issue gets its own type | "Make advice parameter binding a public feature", with its sub-issues | New, summarized as one item |
| `Task` | `metalama` | Internal work with no user-visible effect | Refactoring, build or CI changes, test infrastructure | Omitted |
| `Request` | `metalama` | An incoming report that cannot be acted on yet: no reproduction, or unclear whether it is a defect | A customer report awaiting a repro | Never shipped as-is |

Deciding between neighbors:

- **`Bug` or `Enhancement`** — was the current behavior intended? If not, it is a `Bug`, even when the fix needs a new API or option. If it was intended and documented, it is an `Enhancement`, even if users report it as a bug.
- **`Enhancement` or `Feature`** — does the user need to learn something new to benefit? A new concept or API surface is a `Feature`; making an existing one work better or in more places is an `Enhancement`. Size does not decide it.
- **`Feature` or `User Story`** — a `User Story` exists to group other issues. A single issue implemented by a single PR is a `Feature`, however large.
- **`Request`** is a triage state, not a final classification. Retype it as `Bug`, `Enhancement` or `Feature` once it is understood, or close it — before it is assigned to a milestone.
- `postsharp` has no `Task` type. Purely internal work there is usually not tracked by an issue; if it is, leave it untyped, and `prepare-release` reports it in Phase 2 for exclusion.

The `breaking` label stays a label, because it applies on top of any type. When an issue already has a type, keep it unless it is clearly wrong under the rules above.

### Setting and reading the type

`gh` (2.101 or later) handles issue types by name:

```bash
gh issue create --type Bug --title "..." --body "..."
gh issue edit 1234 --type Enhancement
gh issue list --type Bug --milestone 2026.0.8
gh issue view 1234 --json issueType --jq .issueType.name
```

Type names are the same in both orgs, but their IDs differ. Use names rather than IDs.

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

The `breaking` label is for **user-facing** breaking changes only — a change whose effect a user can observe: code written against our public API stops compiling or behaves differently, or the observable behavior of the build or of the product changes (MSBuild properties and integration, configuration, diagnostics, generated code, run-time behavior).

- **Do not label a break in an internal API.** Internal APIs are ours to change; those breaks are not tracked and the label is noise on them. This is the common mistake — when in doubt, ask whether a user outside the team could have depended on the thing that changed.

### Which APIs are public

The decision is made **per package**, not per type: a package's API is public if and only if its assembly is in the API reference of the product documentation — the `metadata` section of `docfx.json` in `postsharp/PostSharp.Documentation` or `metalama/Metalama.Documentation`. Every other package is internal, **even when it is published on nuget.org**. Within a public package, the public C# surface is public API; `[EditorBrowsable(Never)]`, `[InternalImplement]` interfaces (see below) and members of internal packages are not.

When the lists below and `docfx.json` disagree, `docfx.json` wins — update this file.

Public packages, as of 2026-10-10 (Metalama.Documentation `develop/2026.1`, PostSharp.Documentation `develop/2026.0`):

| Repository | Public packages |
|---|---|
| `metalama/Metalama` | `Metalama.Framework` (assembly shipped in `Metalama.Framework.Redist`), `Metalama.Framework.Sdk`, `Metalama.Framework.Introspection`, `Metalama.Framework.Workspaces`, `Metalama.Testing.AspectTesting`, `Metalama.Testing.UnitTesting`, `Metalama.LinqPad`, `Metalama.Migration`, `Metalama.Extensions.DependencyInjection`, `Metalama.Extensions.DependencyInjection.ServiceLocator`, `Metalama.Extensions.Multicast`, `Metalama.Extensions.Metrics`, `Metalama.Patterns.Contracts`, `Metalama.Patterns.Memoization`, `Metalama.Patterns.Immutability`, `Metalama.Patterns.Observability`, `Metalama.Patterns.Wpf`, `Metalama.Patterns.Caching`, `Metalama.Patterns.Caching.Backend`, `Metalama.Patterns.Caching.Aspects`, `Flashtrace`, `Flashtrace.Formatters` |
| `metalama/Metalama.Premium` | `Metalama.Extensions.Architecture`, `Metalama.Extensions.CodeFixes`, `Metalama.Extensions.Validation`, `Metalama.Patterns.Caching.Backends.Azure`, `Metalama.Patterns.Caching.Backends.Redis` |
| `metalama/Metalama.Compiler` | `Metalama.Compiler.Sdk` (assembly `Metalama.Compiler.Interface`) |
| `postsharp/PostSharp` | `PostSharp` (assembly shipped in `PostSharp.Redist`); `PostSharp.Patterns.Common`, `.Aggregation`, `.Model`, `.Threading`, `.Xaml`, `.Caching`, `.Diagnostics` (assemblies shipped in the matching `.Redist` packages); `PostSharp.Patterns.Caching.Azure`, `.Caching.IMemoryCache`, `.Caching.Redis`; `PostSharp.Patterns.Diagnostics.AspNetCore`, `.AspNetFramework`, `.DiagnosticSource`, `.HttpClient`, `.ApplicationInsights`, `.CommonLogging`, `.Log4Net`, `.Microsoft`, `.NLog`, `.Serilog`, `.Tracing`, `.Configuration` |

Internal packages that are nevertheless published on nuget.org — a break in them is **not** labeled:

- Metalama: `Metalama.Backstage*`, `Metalama.Testing.Hooks`, `Metalama.Framework.Implementation.*`, `Metalama.Tool`, `Metalama.Extensions.HtmlWriter`, `Metalama.Extensions.DiffEngine`, `Metalama.Patterns.TestHelpers`, `Metalama.Patterns.Caching.TestHelpers`, `Metalama.Licensing`.
- PostSharp: `PostSharp.Compiler.Common`, `PostSharp.Compiler.Engine`, `PostSharp.Compiler.Platforms`, `PostSharp.Compiler.Settings`, and the weaver assemblies inside public packages (`PostSharp.Patterns.*.Weaver`).

An internal package can still be the channel of an observable break — for example, a `Metalama.Tool` command-line change or a `PostSharp.Compiler.*` change that alters the woven code. Label it then, for the observable effect, not for the API change.

### Rules
- Adding members to interfaces marked `[InternalImplement]` (including inherited members) is **not** a breaking change.
- For a genuine user-facing break: add a comment to the issue describing the change, and add the `breaking` label to the issue and the PR.
