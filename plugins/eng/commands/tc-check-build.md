---
description: Check the status of a TeamCity build
argument-hint: "<buildId> [continuous]"
allowed-tools:
  - Bash
  - Read
---

# Check TeamCity Build Status

Check the status of a TeamCity build and display results.

## Arguments

$ARGUMENTS - Build ID or "latest" to check the most recent build for current branch (default: latest)

## Instructions

### 1. Determine Build to Check

If argument is a build ID (numeric), use it directly. Otherwise:
1. Get current branch: `git rev-parse --abbrev-ref HEAD`
2. Read `.teamcity/settings.kts` to find:
   - VCS root ID: `AbsoluteId("...")` in vcs block
   - Build type prefix from VCS root (e.g., `Metalama_Metalama20260_Metalama`)
3. Construct build type ID: `<Prefix>_DebugBuild` (e.g., `Metalama_Metalama20260_Metalama_DebugBuild`)
4. Query latest build using the API

### 2. Authentication Context

Check `RUNNING_IN_DOCKER` environment variable to determine authentication method:

**When running in Docker** (`RUNNING_IN_DOCKER=true`):
- TeamCity API calls may require MCP authorization
- Claude will prompt for authorization when needed
- Use PowerShell `Invoke-RestMethod` for API calls

**When running on host** (`RUNNING_IN_DOCKER` not set or false):
- Use `TEAMCITY_CLOUD_TOKEN` environment variable for authentication
- Use PowerShell `Invoke-RestMethod` for API calls

### 3. Query Build Status Using PowerShell

**Get Specific Build by ID:**
```powershell
$build = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/builds/id:$buildId" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }
```

**Get Latest Build for Branch/BuildType:**
```powershell
$builds = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/builds?locator=buildType:$buildTypeId,branch:$branchName,count:1" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }

$build = $builds.build[0]
```

### 4. Parse and Display Results

**Key Fields in Response:**

| Field | Description | Values |
|-------|-------------|--------|
| `id` | Build ID | Numeric |
| `status` | Build result | `SUCCESS`, `FAILURE`, `ERROR` |
| `state` | Build state | `queued`, `running`, `finished` |
| `statusText` | Detailed status message | Text description of error/success |
| `webUrl` | Link to build in TC UI | URL |
| `branchName` | Git branch | Branch name |
| `startDate` | When build started | ISO 8601 timestamp |
| `finishDate` | When build finished | ISO 8601 timestamp (null if running) |
| `queuedDate` | When build was queued | ISO 8601 timestamp |
| `buildType.name` | Build configuration name | e.g., "Build [Debug]" |
| `agent.name` | Build agent | Agent name |
| `problemOccurrences.count` | Number of problems | Numeric |

### 5. Display Build Status

**Format output as:**
```
Build #<number> (<buildType.name>)
Status: <status> (<state>)
Branch: <branchName>
<statusText>

Started:  <startDate>
Finished: <finishDate> (or "Still running" if state=running)
Duration: <calculated from dates>
Agent: <agent.name>

Problems: <problemOccurrences.count>
URL: <webUrl>
```

### 6. Handle Running Builds

If `state == "running"`:
- Display "Build is still running"
- Show elapsed time since `startDate`
- Use `finishDate == null` to detect running state

### 6b. Monitoring a Build Until It Completes

When asked to *monitor* a build rather than check it once, poll `/app/rest/builds/id:<id>` in a `Monitor`, not in a backgrounded shell command: a full build takes far longer than the ten-minute ceiling of a foreground command, whereas a `Monitor` runs up to an hour.

Four rules, each of which has gone wrong in practice:

- **Key the change detection on `state`, and on `status` only once the build is finished. Never on `statusText`.** While a build runs, `statusText` holds the name of the step currently executing, so a watch keyed on it emits an event for every step of the build. That floods the conversation and gets the watch stopped automatically for excessive output.
- **Report every terminal state, not only success, and report API errors.** A watch that matches only the success marker stays silent when the build fails, and silence is indistinguishable from a build that is still running.
- **Tolerate transient request failures.** One failed HTTP call must not kill the watch. Count consecutive failures and give up only after several, reporting that you did.
- **Poll every 60 seconds** and stop the loop as soon as `state` is `finished`.

```bash
prev=""; fails=0
while true; do
  out=$(pwsh -NoProfile -Command '$ErrorActionPreference="Stop"; try { $r = Invoke-RestMethod -Uri "https://postsharp.teamcity.com/app/rest/builds/id:<ID>" -Headers @{ Authorization = "Bearer $env:TEAMCITY_CLOUD_TOKEN"; Accept = "application/json" }; if ($r.state -eq "finished") { "finished|$($r.status)|$($r.statusText)" } else { "$($r.state)|$($r.status)" } } catch { "APIERROR|$($_.Exception.Message)" }' 2>/dev/null)
  if [ -z "$out" ]; then fails=$((fails+1)); if [ "$fails" -ge 5 ]; then echo "MONITOR: giving up after 5 empty responses"; break; fi; sleep 60; continue; fi
  fails=0
  if [ "$out" != "$prev" ]; then echo "build <ID>: $out"; prev="$out"; fi
  case "$out" in finished*) break;; esac
  sleep 60
done
```

When the build finishes with a failure, follow up with the problem and failed-test queries in the sections below rather than reporting only the status line.

### 7. Get Problem Details (Optional)

If `problemOccurrences.count > 0`, fetch details using PowerShell:
```powershell
$problems = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/problemOccurrences?locator=build:(id:$buildId)" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }
```

**Problem fields:**
- `type`: Problem type (e.g., `TC_EXIT_CODE`, `TC_COMPILATION_ERROR`, `TC_FAILED_TESTS`)
- `identity`: Step that failed
- `details`: Additional information

### 8. Get Test Results

**Test Summary in Build Response:**

The build response includes a `testOccurrences` field with summary:
```json
{
  "count": 6553,
  "failed": 2,
  "passed": 6368,
  "ignored": 183,
  "href": "/app/rest/testOccurrences?locator=build:(id:311629)"
}
```

**Get All Test Results:**
```powershell
$tests = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/testOccurrences?locator=build:(id:$buildId)" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }
```

**Get Failed Tests Only:**
```powershell
$failedTests = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/testOccurrences?locator=build:(id:$buildId),status:FAILURE" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }
```

**Get Detailed Test Information:**
```powershell
$testDetail = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/testOccurrences/build:(id:$buildId),id:$testId" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }
```

**Test Occurrence Fields:**

| Field | Description | Example |
|-------|-------------|---------|
| `id` | Test occurrence ID | `2000007320` |
| `name` | Full test name | `TargetCode.Method()` |
| `status` | Test result | `SUCCESS`, `FAILURE`, `UNKNOWN` |
| `duration` | Test duration in ms | `45` |
| `details` | Full failure details including stack trace | Multi-line string with exception and stack |
| `test.name` | Short test name | Method name only |
| `muted` | Whether test is muted | `true`/`false` |
| `currentlyMuted` | Whether currently muted | `true`/`false` |
| `href` | Link to full test info | API URL |

**Display Test Results:**

When displaying failed tests:
1. Show test count summary from `testOccurrences` in build
2. If `failed > 0`, fetch failed test details
3. For each failed test, display:
   - Test name
   - Status
   - Duration
   - First few lines of error message (from `details` field)
   - Option to show full stack trace

## PowerShell Implementation Example

```powershell
# Get build info
$build = Invoke-RestMethod -Method GET `
    -Uri "https://postsharp.teamcity.com/app/rest/builds/id:$buildId" `
    -Headers @{
        "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
        "Accept" = "application/json"
    }

# Display status
Write-Host "Build #$($build.number) - $($build.buildType.name)" -ForegroundColor Cyan
Write-Host "Status: $($build.status) ($($build.state))" -ForegroundColor $(
    if ($build.status -eq "SUCCESS") { "Green" } else { "Red" }
)
Write-Host "Branch: $($build.branchName)"
Write-Host $build.statusText
Write-Host ""
Write-Host "Started:  $($build.startDate)"
if ($build.state -eq "finished") {
    Write-Host "Finished: $($build.finishDate)"
    $duration = [DateTime]::Parse($build.finishDate) - [DateTime]::Parse($build.startDate)
    Write-Host "Duration: $($duration.ToString('hh\:mm\:ss'))"
} else {
    Write-Host "Still running..."
    $elapsed = [DateTime]::UtcNow - [DateTime]::Parse($build.startDate)
    Write-Host "Elapsed:  $($elapsed.ToString('hh\:mm\:ss'))"
}
Write-Host ""
Write-Host "Agent: $($build.agent.name)"
if ($build.problemOccurrences.count -gt 0) {
    Write-Host "Problems: $($build.problemOccurrences.count)" -ForegroundColor Yellow
}

# Display test results summary
if ($build.testOccurrences) {
    Write-Host ""
    Write-Host "Tests: $($build.testOccurrences.count) total, " -NoNewline
    Write-Host "$($build.testOccurrences.passed) passed, " -NoNewline -ForegroundColor Green
    if ($build.testOccurrences.failed -gt 0) {
        Write-Host "$($build.testOccurrences.failed) failed, " -NoNewline -ForegroundColor Red
    }
    Write-Host "$($build.testOccurrences.ignored) ignored" -ForegroundColor Gray

    # Show failed tests if any
    if ($build.testOccurrences.failed -gt 0) {
        Write-Host ""
        Write-Host "Failed Tests:" -ForegroundColor Red

        $failedTests = Invoke-RestMethod -Method GET `
            -Uri "https://postsharp.teamcity.com/app/rest/testOccurrences?locator=build:(id:$buildId),status:FAILURE" `
            -Headers @{
                "Authorization" = "Bearer $env:TEAMCITY_CLOUD_TOKEN"
                "Accept" = "application/json"
            }

        foreach ($test in $failedTests.testOccurrence) {
            Write-Host "  - $($test.name)" -ForegroundColor Yellow
            if ($test.details) {
                # Show first line of error
                $errorLine = ($test.details -split "`n")[0]
                Write-Host "    $errorLine" -ForegroundColor DarkGray
            }
        }
    }
}

Write-Host ""
Write-Host "URL: $($build.webUrl)" -ForegroundColor Cyan
```

## API Reference

**Base URL:** `https://postsharp.teamcity.com/app/rest`

**Authentication:** Bearer token via `TEAMCITY_CLOUD_TOKEN` environment variable

**Common Locators:**
- By ID: `id:<buildId>`
- Latest by type: `buildType:<buildTypeId>,count:1`
- Latest by type+branch: `buildType:<buildTypeId>,branch:<branchName>,count:1`
- Running builds: `buildType:<buildTypeId>,running:true`
- By status: `buildType:<buildTypeId>,status:FAILURE,count:10`

**Useful Endpoints:**
- `/app/rest/builds/id:<id>` - Get specific build
- `/app/rest/builds?locator=...` - Query builds
- `/app/rest/problemOccurrences?locator=build:(id:<id>)` - Get problems
- `/app/rest/testOccurrences?locator=build:(id:<id>)` - Get test results
- `/app/rest/builds/id:<id>/statistics` - Get build statistics

## Error Handling

**Authentication Errors:**
- When running on host: If `TEAMCITY_CLOUD_TOKEN` not set, display error message
- When running in Docker: If MCP authorization fails or is denied, display error message
- Check `RUNNING_IN_DOCKER` environment variable to provide appropriate error context

**API Errors:**
- If build ID not found, API returns 404 - display "Build not found"
- If branch has no builds, API returns empty array - display "No builds found for this branch"
- Invalid locator syntax returns 400 with error details - display the error message
- Network errors should be caught and displayed clearly
