---
name: eng
description: This skill should be used when working in a PostSharp or Metalama repository and the user asks to "commit", "create a branch", "open a PR", "merge", "prepare a release", "run Build.ps1", "trigger a TeamCity build", "check the build", "monitor the build", "watch the build", "tell me when the build is done", "fix binlog warnings", "start work on issue NNNN", "write XML documentation", "write a comment", "write an issue", "write release notes", or mentions topic/develop/release branches, milestones, breaking changes, `<see>` tags, cross-repo or local dependencies, or the MCP approval server in Docker. Covers git conventions, the GitHub issue workflow, TeamCity CI/CD (triggering, checking and monitoring builds to completion), the PostSharp.Engineering build system, documentation conventions, and the copywriting rules for all prose (XML documentation, code comments, Markdown documents, commit messages, pull request descriptions, GitHub issues and comments).
---

# PostSharp Engineering Workflows

Conventions and commands for PostSharp/Metalama repositories: git, GitHub, TeamCity, and the `Build.ps1` build system.

## Commands

Invoke these commands, or read their instructions on demand:

- `/eng:create-pr` — create (prepare) a pull request
- `/eng:fix-binlog-warnings` — analyze warnings from binlog output of `Build.ps1 build`
- `/eng:prepare-release` — prepare a GitHub release, release notes; typically run after deployment
- `/eng:reflect` — capture learnings (mistakes, patterns, knowledge gaps, user corrections) into `CLAUDE.md` or the plugin files. Run it automatically after solving a problem that took several failed attempts
- `/eng:tc-build` — schedule a TeamCity (TC, CI) build. **Pushing a branch does not trigger a build.** CI runs only when a build is explicitly scheduled, so after pushing, trigger one with this command when CI results are wanted — and never wait on, or report, a build that was never scheduled
- `/eng:tc-check-build` — check the status of a TC build, or monitor one until it finishes (use it for any "watch the build", "poll until done", "notify me when it completes" request; it polls in a `Monitor`, since a build outlasts a foreground command)

## Related Repositories

Three GitHub organizations are in use at once — none of them is a rename of another:

- **`metalama`** — the current product repositories
- **`postsharp`** — the older products; still active, and with its own Development project whose IDs differ from `metalama`'s
- **`postsharp-ops`** — engineering and infrastructure, created June 2026; several repos were moved here from `postsharp`, so old `postsharp/…` URLs still redirect

By convention every repo is checked out at **`c:\src\<RepoName>`** on developer machines, so a sibling repo can be read straight from disk instead of fetched — in particular:

- **[PostSharp.Engineering](https://github.com/postsharp-ops/PostSharp.Engineering)** (`c:\src\PostSharp.Engineering`) — the in-house multi-repo continuous build and integration framework that `Build.ps1` and `eng/src` are built on. Read it to explain or change build behaviour.
- **[Docs.Infrastructure](https://github.com/postsharp-ops/Docs.Infrastructure)** (`c:\src\Docs.Infrastructure`) — the authoritative documentation of build and IT infrastructure: TeamCity Cloud and build agents, package feeds, the code-signing service, Azure, networking, and the credential inventory (names and expiry only — the repo holds no secret values, by policy). Consult it before re-deriving infrastructure facts from `az`, TeamCity or Cloudflare; start at its `README.md`.
- **[PostSharp.Engineering.AISkills](https://github.com/postsharp-ops/PostSharp.Engineering.AISkills)** — the source of this plugin.

## Branches

Each major version has two long-lived branches:

- **`release/YYYY.N`** — updated only during deployment; the GitHub default branch for that version line
- **`develop/YYYY.N`** — the continuous CI/CD development branch

Flow: `topic/YYYY.N/XXXX-description` → `develop/YYYY.N` (PR) → `release/YYYY.N` (after successful deployment).

## Branch Naming

Pattern: `topic/YYYY.N/XXXX-short-description`

- `YYYY.N` — version/milestone (e.g. `2026.0`)
- `XXXX` — issue number (required). With no issue, use the date instead: `YY-MM-DD`
- `short-description` — brief and hyphenated
- If the branch already exists, append a numeric suffix: `-2`

**Merge target for `topic/YYYY.N/*` is ALWAYS `develop/YYYY.N`. Never target the release branch directly.**

## Commits

**Never commit autonomously.** Unless the user explicitly asks for a commit:

1. Ask the user to review the changes (they review in their IDE)
2. Wait for explicit approval
3. Only then commit

Message rules:

1. Keep the subject short (50–72 chars)
2. Include the issue number: `(#XXXX)`
3. Use imperative mood — "Fix bug", not "Fixed bug"
4. **Never sign commits** — no "Generated with Claude Code" trailer. Sign only PRs, issues and comments, modestly, with "— Claude for \<user\>"

Examples:

- `Fix cache invalidation on timeout (#1234)`
- `Add retry logic for API calls (#5678)`

## Building

A product is made of several solutions. Inside a solution, projects reference each other directly, so `dotnet build` / `dotnet test` picks up a change immediately — that is the tool for ordinary work. Across solutions, consumption goes through NuGet packages instead, and only `Build.ps1 build` produces those packages.

So use `Build.ps1 build` for exactly two things:

- **Final validation** of the whole product, typically before opening a PR
- **Producing the packages** a cross-solution or cross-repo change needs before the consuming solution can see it

**The decisive question is whether the change crosses a solution boundary — not how large it is.** Any rebuild that stays inside one solution belongs in `dotnet build` / `dotnet test`, whether that means one test or the entire solution.

To answer that question rather than guess it, read the build layers: the solutions are an **ordered** list, declared in `eng/src/Program.cs` and printed by `Build.ps1 list-solutions`. Each solution consumes the ones before it through packages, so a change that a *later* layer has to see requires new package versions — and only `Build.ps1 build` produces those.

**Critical build rules:**

- **NEVER** run `Build.ps1 build` — ask the user to run it (the tool timeout cuts it off and triggers retries)
- **NEVER** run `Build.ps1 prepare` — it deletes all built artifacts and forces a subsequent `Build.ps1 build`
- **NEVER** clear the global NuGet package cache — it is never needed
- **Nothing else may touch the working tree while `Build.ps1 build` runs** — it deletes every `bin` and `obj` at the start, so a concurrent `dotnet build` or `dotnet test` fails on files removed underneath it
- **Never** use `Build.ps1 build` for an intra-solution rebuild — it earns its place only when the change crosses a solution boundary, or as final validation

## Coding Rules

- **Prefer `pwsh`** (PowerShell 7); never use old `cmd`
- **No hardcoded delays in tests** — use barriers, `TaskCompletionSource`, or sync points
- **Never await without a cancellation token**
- Add a `PackageVersion` entry to `Directory.Packages.props` whenever adding a package reference
- Focus on green tests first; leave cosmetic warnings (redundant usings, etc.) until the finalizing stage

## Copywriting Rules

These rules apply to every piece of prose, including XML documentation, code comments, Markdown documents, commit messages, pull request descriptions, and GitHub issues and comments.

- **Be accurate.** The statement must be true of the code as written. A summary that says the class stores nothing, on a class that declares a field, is a defect, not a style problem.
- **Use accurate software engineering language. Do not use analogies or slang.** Name the mechanism: "resolves the identifier through the symbol table", not "asks the symbol table for the identifier"; "costs an allocation and a dictionary lookup", not "buys nothing".
- **Do not lead with a mystery or with a rhetorical construct.** State the subject in the first clause. Write "A durable reference stores an identifier at design time and the original reference during a batch compilation", not "What a durable reference stores depends on the scope". Avoid openings such as "The one thing this must never do", "Two consequences are worth knowing", and "This is a cache and nothing else".
- **No uncommon acronyms or abbreviations.** Expand anything that is not already standard in this codebase.
- **Assume the reader is not a native English speaker.** Prefer short sentences and a plain vocabulary. Avoid inversion, ellipsis, and idiom. One idea per sentence.
- **Do not use bold text for emphasis inside a paragraph**, and do not use italics to stress a word. Structure the text instead. Agent instruction files such as `CLAUDE.md` and skills are exempt: there, bold marks critical rules.

## Writing Documentation

- Use `<see>` tags for type/member references
- Maintain a consistent lexicon within a class family (same suffix)
- Keep code examples short
- Cross-reference conceptual docs via `<seealso href="@..."/>` tags
- Use the api-docs-reviewer subagent when writing XML doc or API doc
- **NEVER** replace `<see>` tags with `<c>` to fix XML doc errors — fix the reference or add a `using` instead

## Updating This Skill

To make permanent changes, edit the source files under `plugins/` in the [PostSharp.Engineering.AISkills](https://github.com/postsharp-ops/PostSharp.Engineering.AISkills) repository (usually at `c:\src\PostSharp.Engineering.AISkills`), and bump the plugin version as required by that repository's `CLAUDE.md`. Claude Code manages the plugin cache; update the installed copy with `/plugin`.

## Reference Files

- **`references/build-system.md`** — solution layout, reference types, the full `Build.ps1` command set, local cross-repo dependencies, build pitfalls and exit codes, and the Docker MCP approval server
- **`references/github-workflow.md`** — starting work on an issue, milestone format, the per-org Development project and status field IDs (with the query to resolve them), GraphQL node-ID queries, breaking-change policy
