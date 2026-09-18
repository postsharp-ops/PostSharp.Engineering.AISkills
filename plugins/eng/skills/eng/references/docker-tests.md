# Docker-Based Tests

A Docker-based test is a test that needs a container of its own. Each test builds its own image, runs one
container, and is judged by the exit code of that container. The product repositories that use this are
PostSharp, Metalama and SharpCrafters.Backstage.

Use this form only when the behaviour under test depends on the environment itself: a specific .NET SDK
version, a specific operating system or distribution, a specific processor architecture, or the fact of
running inside a container. An ordinary unit or integration test belongs in `dotnet test`, not here.

## The two layers

| Layer | Where it lives | Who writes it |
|-------|----------------|---------------|
| `RunDockerTests.ps1` — the launcher | Generated into the engineering directory (`eng/`) by `Build.ps1 generate-scripts`; the template is in PostSharp.Engineering | Generated. Never edit the copy in a product repository |
| `RunTest.ps1` — one per test | The product repository, next to the test's own files | By hand, per test |

The launcher discovers tests, decides which ones apply to the current platform, runs them one at a time, and
reports each one to TeamCity. The test script builds and runs its container and returns an exit code. The
split is what keeps a test runnable by hand: `RunTest.ps1` knows nothing about TeamCity.

## The host contract

A Docker test host needs exactly two things:

- PowerShell 7.5 or later (`pwsh`)
- A container engine whose operating system and processor architecture match the test's platform

It must not need the .NET SDK, MSBuild, Visual Studio, or any product tool chain. That is the rule the whole
design rests on. The tool chain under test lives inside the test's own image, so requiring one on the host
would add nothing and would force these configurations back inside the build container, where they would lose
the agent's image cache and gain the build container's secrets.

What the build produced reaches the agent through a TeamCity artifact dependency. Each test resolves it from
where the test itself lives; nothing tells it. Where several tests share a suite, the script they dot-source is
the place for that, so the repository layout is written once.

## Directory layout

One directory per test, flat, under a directory that the launcher is pointed at:

```
Tests/Docker/
  Issue15-AssetsFileV4/
    test.psd1
    Dockerfile
    RunTest.ps1
    TestProject/...
  ContainerDetection/
    test.psd1
    Dockerfile
    RunTest.ps1
```

The platforms a test runs on are declared in its manifest, not encoded in its position in the tree. Most
tests apply to more than one platform — the same Linux test should normally run on both `linux-x64` and
`linux-arm64` — and a directory per platform would mean duplicating the test to get that coverage.

## The manifest: `test.psd1`

```powershell
@{
    # Required. The platforms this test runs on.
    Platforms = @( 'linux-x64', 'linux-arm64' )

    # Optional. Default 900. The container is killed and the test fails when it is exceeded.
    TimeoutSeconds = 900

    # Optional. When present, the test is reported as ignored on every platform, with this reason.
    Skip = 'Blocked by #1234.'
}
```

`TimeoutSeconds` covers the whole run, including acquiring the image, not just the test's own work. That
distinction decides the number. A Windows suite on `mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2019`
was killed at 30 minutes on an agent meeting that image for the first time, having already built and passed in
two seconds and being on its final registry push -- several gigabytes of pull, and a failure that said nothing
about the product. Give a suite on one of the Windows Framework SDK images an hour; half of one is enough for
the rest. State in a comment why a suite differs from its neighbours, or the next reader will normalise them.
The four platform identifiers are `win-x64`, `win-arm64`, `linux-x64` and `linux-arm64`. They name the
container engine's operating system and architecture, which is also what selects the TeamCity agent.

A test whose manifest does not list the current platform is reported as ignored, not omitted. Every
configuration's log therefore shows the full inventory and says why each test did not run.

## What a test consumes: packages, not build output

A Docker test consumes the product the way a user does, through a `PackageReference` on the packages the
distribution build produced. It does not reference the compiler tree from `Build/bin`. Testing the packages is
the point: they are what ships, and a test that wove with the build output would pass while a packaging defect
went unnoticed.

Three things follow, and each has bitten:

- **The TeamCity configuration depends on the distribution stage**, not on the artifacts stage. Set
  `BuildSnapshotDependency` accordingly.

- **The packages must be where the NuGet source says they are.** The generated `CopyNuGetConfig` step puts
  `nuget.restored.config` at the repository root as `nuget.config`, and its source names the private artifacts
  directory. A local-folder source is not searched recursively, so the `.nupkg` files must sit directly in that
  directory. Where they arrive from is a product matter -- in PostSharp they are extracted into
  `Build/releases/v<version>` by the artifact rule and staged from there. Do not expand the published archives
  a second time: the artifact dependency has already extracted them, and expanding them again buries the
  packages one level down, where every restore fails `NU1101` against a source directory that does exist.

- **A missing `nuget.config` is fatal, not tolerable.** The launcher asserts it. Without it the restore falls
  back to nuget.org, and because the version under test has usually been released, it succeeds -- so the test
  passes having verified a published package rather than the one just built. A test that silently checks the
  wrong thing is worse than one that fails.

Use the product preparation hook for whatever staging a product needs: the launcher runs `eng/PrepareDockerTests.ps1`
once, before any test, when that file exists. Putting it there keeps the work in the product repository, where
the layout it knows about lives, and keeps it out of the generated launcher.
## The `RunTest.ps1` contract

```powershell
param(
    # The platform identifier the launcher selected, for example 'linux-x64'.
    [Parameter( Mandatory = $true )] [string] $Platform
)
```

The platform is all a test is told, because it is the only thing that varies per run. Everything else it
addresses by a path relative to the repository root: `DockerBuild.ps1` mounts the repository into the container
at its own path, and the command runs with that as its working directory.

Rules:

- **Exit code 0 means the test passed, 4 means it skipped, anything else means it failed.** Do not throw to
  report a failure of the code under test; reserve exceptions for the harness itself failing.
- **Use exit code 4 when the scenario cannot occur on this host** -- an SDK that ships a pack the scenario
  needs absent, a case-insensitive file system, a missing kernel facility. This is different from the
  manifest's `Skip`, which states what is known before the test runs. Write a line beginning with `SKIPPED:`
  explaining why; the launcher uses the last such line as the TeamCity `testIgnored` message.
- **Write to standard output and standard error normally.** The launcher captures both and attaches them to
  the test.
- **Never emit TeamCity service messages.** The launcher owns that protocol. A test that writes its own
  produces nested, malformed test reporting.
- **Build and run the container through `DockerBuild.ps1 -Test`**, never by calling `docker` directly. See
  the next section.
- **Splat a hashtable, never an array.** `& ./DockerBuild.ps1 @arguments` with an array does not bind the
  named parameters: they fall into `-BuildArgs`, and the script runs an ordinary product build, with the
  product secrets, instead of the test container. The script now refuses that, but write
  `@{ Test = $true; OS = $os }` rather than relying on the refusal.
- **Copy nothing into the image.** The Dockerfile is `FROM <base image>` and nothing else. The repository is
  mounted, so the test builds the real project at its real path and relative paths mean what they say. The
  context stays the directory holding the Dockerfile, which keeps the content hash small and stable.
- **Send build output outside the project directory.** Tests share projects, and the SDK globs `**/*.cs` under
  a project while excluding only the intermediate directory of the build in progress. Output left inside the
  project makes a sibling test's generated `AssemblyInfo.cs` part of this build, failing with `CS0579`. Give
  each test its own `BaseIntermediateOutputPath` and `BaseOutputPath` under a gitignored sibling directory.
- **Watch for an inherited `global.json`.** A project inside the mount inherits every `Directory.Build.props`
  and `global.json` above it. That is how `PostSharpBinDir` resolves correctly, but an SDK pin the image does
  not carry will fail the test; shadow it with a rolling-forward `global.json` in the suite directory.
- **Derive `-OS` from `$Platform`.** See
  [Windows and Linux on a development machine](#windows-and-linux-on-a-development-machine).
- **Pin the base image, and read the registry prefix from the environment.** See
  [Base images and the registry](#base-images-and-the-registry).

```powershell
$os = if ($Platform -like 'linux-*') { 'linux' } else { 'windows' }

& "$PSScriptRoot/../../../DockerBuild.ps1" -Test -OS $os `
    -Dockerfile "$PSScriptRoot/Dockerfile" `
    -Command ( 'cd Tests/Docker/TestProject && dotnet msbuild -t:Restore -t:Build ' +
               '/p:BaseIntermediateOutputPath=../.artifacts/Issue15/obj/ ' +
               '/p:BaseOutputPath=../.artifacts/Issue15/bin/' )

exit $LASTEXITCODE
```

## Running the container: `DockerBuild.ps1 -Test`

A test builds and runs its container through `DockerBuild.ps1 -Test`. The reason is the image and registry
logic that script already carries, and that a container-per-test suite depends on more than any other
consumer:

- A content-hash tag per image, folding the Dockerfile, its context, the parent's hash and an operating-system
  discriminator, so that identical inputs produce identical tags and hit the cache.
- Line-ending normalization before hashing, so a checkout with LF and one with CRLF share the same tag and
  therefore the same registry cache.
- Push to and pull from the configured registry, so an image built once on one agent is not rebuilt on the
  next.
- Removal of unused images, oldest first, when the image store exceeds its budget.

Reimplementing any of that in a test script would produce a slower and less correct copy. A suite that
builds one image per test is precisely the workload where a missed cache hit is expensive.

`-Test` is a distinct mode of the same script. It requires `-Dockerfile` and `-Command`, takes an optional
`-Context` that defaults to the directory containing the Dockerfile, and refuses to combine with `-Claude`,
`-Interactive`, `-BuildImage`, `-StartVsmon`, `-PostInit`, `-KeepInit` and `-Script`. Compared with a normal
build run it does not:

- build the product image chain from the repository's Dockerfiles, or the boot image over it
- invoke `Init.g.ps1`

What it does **not** change is the mounts. The repository, the NuGet cache, the source dependencies and the
sibling repositories are mounted as they are for any build, and the command runs with the repository as its
working directory. A test consuming a source dependency needs the same repositories the build needs, and the
repository is a volume mount rather than a build context: the Docker context is only what is needed to build
the image before the volume can be mounted.

`Init.g.ps1` is skipped for one reason only: a test image is chosen for the tool chain under test and is not
required to carry PowerShell 7, so a `.ps1` cannot be how it is configured. The environment that script would
have inlined is passed to `docker run` as `-e` arguments instead, so a test container receives what a build
container receives, including `NUGET_PACKAGES`, the licence variables and the git identity. A test container
is one the repository builds from a Dockerfile it owns and then runs, so it is trusted the way the build
container is.

An earlier version withheld that environment. It bought no isolation worth the cost: `NUGET_PACKAGES` is in
the set, and without it NuGet in the container fell back to `$HOME/.nuget/packages`, so the mounted host cache
was never read and every test restored over the network -- the opposite of what the mount exists for. `-Env`
is still honoured and is folded into the same set.

The test controls its own base image, because in most of these tests the base image is the subject: the point
of `mcr.microsoft.com/dotnet/framework/sdk:4.7.2` is that it is that SDK and not another one.

`-Mount` still works in test mode, but it takes a host directory and mounts it at the same absolute path in
the container, so a mounted path is a host path. Prefer the build context; use a mount only where a test
genuinely needs a live host directory.

## Windows and Linux on a development machine

A build agent runs one engine, the build is routed to the agent that has the right one, and nothing switches.
A development machine is different: Windows containers run on Docker Desktop, and Linux containers run on the
Docker engine inside WSL.

`DockerBuild.ps1 -OS windows|linux` says which is wanted and defaults to the host's own operating system.
Where the two disagree on a development machine, the script re-executes itself inside WSL, converting the
arguments that hold a path into `/mnt/<drive>/...` form. Everything below the hop then runs on a genuine
Linux host.

**It refuses rather than falling back** when `IS_TEAMCITY_AGENT` is set, when `wsl.exe` is missing, when the
distribution has no `pwsh`, when the engine inside WSL does not answer or is not a Linux engine, when that
engine reports another processor architecture, or when a Linux host is asked for `-OS windows`. Each refusal
names the condition it found.

`RunDockerTests.ps1` follows the same rule when it probes the engine, so `-Platform linux-x64` on a Windows
development machine asks the engine inside WSL rather than Docker Desktop. That is why each `RunTest.ps1`
must pass `-OS`: without it, `DockerBuild.ps1` would default to the host and use the wrong engine.

## What the launcher reports

`RunDockerTests.ps1` writes TeamCity service messages around each test. Three rules matter when reading or
changing it:

- **Values must be escaped.** In a service message, `|` becomes `||`, a single quote becomes `|'`, `[`
  becomes `|[`, `]` becomes `|]`, a newline becomes `|n` and a carriage return becomes `|r`. Docker output
  contains brackets constantly, so an unescaped message silently truncates or corrupts the report.
- **`testFailed` comes before `testFinished`**, never after. The launcher writes `testFinished` in a
  `finally`, which is what guarantees the order.
- **One test's failure must not end the run.** Each test is wrapped in its own `try`/`catch`. The launcher's
  own exit code is the aggregate of the results.

Tests are reported under a suite named `DockerTests.<platform>`, so a failure names the platform without the
reader having to open the configuration. The messages are written only when `TEAMCITY_VERSION` is set; a
local run prints plain text instead.

## The TeamCity configurations

Declare one `DockerTestsAdditionalCiBuildConfiguration` per platform in `eng/src/Program.cs`:

```csharp
new DockerTestsAdditionalCiBuildConfiguration(
    "DockerTestsLinuxX64",
    "Docker Tests (Linux x64)",
    DockerTestPlatform.LinuxX64 )
{
    BuildSnapshotDependency = BuildConfiguration.Debug,
    ProjectFolder = "Docker Tests",
    TimeoutInMinutes = 120
}
```

It passes **only** `-Platform`, and derives the agent requirements from it. Where the tests are is a fact about
the repository: the launcher is generated into its root, so `generate-scripts` writes it into the file, the way
it writes `$EngPath` into `DockerBuild.ps1`. `-Path` survives as a parameter defaulting to that value, for a run
by hand. Where the build output is, is a fact about the product, so the tests hold it and the SDK has none.

`generate-scripts` emits the launcher only for a product that declares at least one of these configurations.

Pass the configurations through `DockerTestsAdditionalCiBuildConfiguration.WithCompositeConfiguration`. They
are grouped into a `Docker Tests` sub-project, which the constructor sets, and the method adds a
`RunAllDockerTests` composite that starts every platform and reports their combined result — but only when
there is more than one platform, since a single one needs no second entry point to itself.

**Its requirements are a plain `BuildAgentRequirements`, deliberately not a `ContainerHostRequirements`.**
A `ContainerHostRequirements` makes the generator rewrite the step into `DockerBuild.ps1` with the real
script passed inside it, so the launcher would run in the product build container and every test would have
to nest one engine inside another to get a container of its own. The requirements name the operating system
and architecture the agents report rather than `env.BuildAgentType`, because an agent that has a container
engine but was not provisioned as a build container host does not publish that property, and requiring it
leaves the build queued indefinitely instead of failing.

Declaring such a configuration is also what makes `Build.ps1 generate-scripts` emit `RunDockerTests.ps1`
into the engineering directory, as `eng/RunDockerTests.ps1`. It is generated only for a product that declares
at least one such configuration. Add `eng/RunDockerTests.ps1 text eol=lf` to the repository's
`.gitattributes` alongside the other generated scripts, or a machine with `core.autocrlf=true` rewrites the
whole file on every regeneration.

## Base images and the registry

`env.DOCKER_REGISTRY` is a root project parameter, set to `docker-registry.home`, and that registry already
mirrors the operating-system base images. Prefer it: container-per-test multiplies image pulls, and the home
site reaches Docker Hub and `mcr.microsoft.com` over a single consumer link with rate limits.

It is not always reachable. The Azure and JetBrains-hosted agents have no route to `docker-registry.home`.
A test must therefore treat the registry as a prefix that may be absent and fall back to the public registry,
rather than hardcoding either one.

## Windows isolation

`DockerBuild.ps1` selects process isolation on Windows Server and Hyper-V isolation on Windows client.
Process isolation requires the image's Windows build to match the host's. The build agents run Windows
Server 2025, so an image based on an older Windows release — which includes the .NET Framework SDK images —
needs `-Isolation hyperv`. The agents are themselves Hyper-V guests, so that in turn requires nested
virtualization on the Hyper-V host.

A test that needs an older-kernel image must say so, by passing `-Isolation hyperv` explicitly.

## Running a test by hand

```powershell
# The whole suite for this host's own engine. Everything defaults: the directories to what generate-scripts
# wrote into the script, and the platform to what the container engine reports.
pwsh ./eng/RunDockerTests.ps1

# One test, through the launcher, so that it is reported the same way CI reports it.
pwsh ./eng/RunDockerTests.ps1 -Platform linux-x64 -Test Issue15-AssetsFileV4

# One test, directly. No TeamCity messages, plain exit code.
pwsh ./Tests/Docker/Issue15-AssetsFileV4/RunTest.ps1 -Platform linux-x64
```

The launcher asks the engine for its own platform and refuses to run when it disagrees with `-Platform`, so a
build routed to the wrong agent fails once with that reason rather than once per test with an unrelated one.

A test that cannot be run this way is a defect in the test, not an acceptable limitation. The launcher adds
discovery, platform filtering and reporting; it must never add anything the test needs in order to work.

## Related documentation

- The build agents, their operating systems, architectures and container engines:
  `build/build-agents.md` in Docs.Infrastructure.
- The base-image mirror: `services/docker-registry.md` in Docs.Infrastructure.
- `DockerBuild.ps1` itself, including the image chain, the content-hash tags and the mount model:
  `doc/dockerbuild.md` in PostSharp.Engineering.
- The mechanism behind this page, including the launcher template and the build configuration type:
  `doc/docker-tests.md` in PostSharp.Engineering.
