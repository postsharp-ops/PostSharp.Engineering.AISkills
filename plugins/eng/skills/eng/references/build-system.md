# Build System (Build.ps1)

[PostSharp.Engineering](https://github.com/postsharp-ops/PostSharp.Engineering) is the build orchestration SDK — the in-house multi-repo continuous build and integration framework, usually checked out at `c:\src\PostSharp.Engineering`. Read its sources there when build behaviour needs explaining or changing. Each consuming repo is a **product** made of multiple **solutions**.

The build infrastructure it runs against — TeamCity Cloud, the build agents, the package feeds and the code-signing service — is documented in [Docs.Infrastructure](https://github.com/postsharp-ops/Docs.Infrastructure) (`c:\src\Docs.Infrastructure`), under its `build/` folder. Consult that before re-deriving CI facts from the TeamCity API.

## Key Concepts

- **Product** — a repo, configured in `eng/src/Program.cs`
- **Solution** — a first-level directory (e.g. `Metalama.Framework`, `Metalama.Patterns`)
- **Build.ps1** — the PowerShell front-end to `eng/src`, self-generated via `Build.ps1 generate-scripts`

## Reference Types

| Scope | Reference type | Notes |
|-------|----------------|-------|
| Within a solution | `ProjectReference` | Standard .NET references |
| Between solutions | `PackageReference` | Requires `Build.ps1 build` to update packages |
| Cross-repo | `PackageReference` | Resolves from TeamCity artifacts by default |

## Common Commands

```powershell
# Full build - produces the uniquely versioned packages that other solutions and repos consume
Build.ps1 build

# Kill locked processes after a failed build
Build.ps1 tools kill

# List cross-repo dependencies
Build.ps1 dependencies list

# Use a local repo instead of TeamCity artifacts
Build.ps1 dependencies set local <product>

# Revert to TeamCity artifacts
Build.ps1 dependencies reset --all
```

## Build Layers

`Product.Solutions` in `eng/src/Program.cs` is an **ordered** list, and that order is the layering: each solution consumes the ones before it through packages. `Build.ps1 list-solutions` prints the same list as a table of Id, solution path and type.

Metalama, for example, layers as: `Metalama.Backstage` → `Metalama.Framework` → `Metalama.Extensions` → `Metalama.Patterns` → `Metalama.Migration` → `Metalama.LinqPad`.

Locating the changed solution in that list answers which command is needed:

- **Nothing later in the list has to see the change** — it stays inside its layer, so `dotnet build` / `dotnet test` in that solution is enough.
- **A later layer has to see it** — crossing a layer means the consumer resolves it through a `PackageReference`, so new package versions must be produced, and that is `Build.ps1 build`.

Editing `Metalama.Framework` and running its own tests stays in-layer, so `dotnet test` suffices. Making that same edit visible to `Metalama.Patterns` crosses a layer and requires `Build.ps1 build`.

The Id from `list-solutions` is what `Build.ps1 build --solution <id>` takes; note that passing it implies `--no-dependencies`.

## Choosing a Build Command

| Scenario | Command |
|----------|---------|
| Changes within one solution | `dotnet build` / `dotnet test` |
| Cross-solution change, so consumers can pick it up | Ask the user to run `Build.ps1 build` |
| After pulling updates, to refresh inter-solution packages | Ask the user to run `Build.ps1 build` |
| Final validation before a PR | Ask the user to run `Build.ps1 build` |
| Locked processes after a failure | `Build.ps1 tools kill` |

`Build.ps1 build` must always be run by the user, never by Claude.

## Build Pitfalls

- Two `Build.ps1 build` runs cannot run in parallel.
- **Nothing else may touch the working tree while `Build.ps1 build` runs.** It starts with a clean that deletes every `bin` and `obj`, so a concurrent `dotnet build` or `dotnet test` fails on files removed underneath it. Run the build alone.
- **The exit code can be trusted.** Since PostSharp.Engineering 2023.2.408 (July 2026), `Build.ps1` captures `$LASTEXITCODE` and propagates it; before that it always exited 0 and every failure read as success. Interpret it via the `ExitCode` enum: `0` success, `1` reported failure, `2` no change made (**not** a failure — returned by `Build.ps1 dependencies update-eng`), `100` unhandled exception, `200` cancelled with Ctrl+C.
- A repo whose generated `Build.ps1` predates that release still always exits 0. If an obviously broken build reports success, grep the script for `LASTEXITCODE`; if it is absent, regenerate with `Build.ps1 generate-scripts`.
- The log is long. Redirect it to a file so it can be searched afterwards for `error` and `failed with exit code`.
- After `Build.ps1 build`, MSBuild binlogs are under `artifacts/logs`. Analyze them with `/eng:fix-binlog-warnings`.

## Local Cross-Repo Dependencies

By default, cross-repo `PackageReference` dependencies resolve from TeamCity (the last successful build). To consume local changes instead:

```powershell
# In the consuming repo, point to the local dependency
Build.ps1 dependencies set local Metalama.Premium
```

The `PackageReference` then resolves from the local repo's `Build.ps1 build` output, so `Build.ps1 build` must have run in the dependency repo first. Revert with `Build.ps1 dependencies reset --all`.

## MCP Approval Server (Docker)

When Claude Code runs inside a Docker container (the `RUNNING_IN_DOCKER` environment variable is set), operations needing host-level access — `git push`, GitHub CLI, and similar — are routed to the host through the MCP Approval Server, which provides a human-in-the-loop workflow.

Flow: the container's MCP client calls `execute_command` over HTTP/SSE; the host server receives the request, runs an AI risk analysis (Claude CLI), auto-approves/rejects or prompts the user, executes if approved, and returns the result.

Any PowerShell command is allowed, subject to that approval.

The MCP server starts automatically with `DockerBuild.ps1 -Claude`, and routing happens automatically via the `host-approval` MCP configuration. To disable it:

```powershell
.\DockerBuild.ps1 -Claude -NoMcp
```
