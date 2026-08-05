# Metalama Development Plugin

Specialized knowledge and tools for developing the Metalama framework itself. Provides comprehensive reference documentation, testing patterns, and TeamCity integration for Metalama contributors.

## Prerequisites

- Metalama source code repository
- .NET SDK (version matching Metalama requirements)
- Understanding of Roslyn and C# compilation
- TeamCity access (for CI/CD integration)
- Git authentication (via MCP authorization server when running in Docker)

## Features

### Aspect Testing
- **Snapshot-based testing** - Test aspects through the Metalama pipeline with expected output comparison
- **Test directives** - Use `@TestScenario`, `@IgnoredDiagnostic`, `@RemoveOutputCode`, and other directives
- **Pipeline validation** - Verify aspect behavior across different compilation stages
- **Output verification** - Compare transformed code against expected snapshots

### Test Directives Reference
- `@TestScenario(applyCodeFix)` - Apply code fixes during testing
- `@IgnoredDiagnostic(diagnosticId)` - Suppress expected diagnostics
- `@RemoveOutputCode` - Remove generated code from output
- `@Include(path)` - Include external files in test compilation
- `@DesignTime` - Test design-time behavior vs compile-time

### Code Model & Templates
- Metalama code model API reference
- Template syntax and best practices
- Compile-time vs run-time code patterns
- Diagnostic creation and reporting

### Debugging & Troubleshooting
- Debugging techniques for aspect development
- Pipeline inspection and diagnostic paths
- Common issues and resolutions
- Performance optimization strategies

## Skills

### metalama-reference
Comprehensive API reference and development patterns for Metalama framework development. Covers:
- Code model navigation and manipulation
- Template authoring and syntax
- Aspect lifecycle and pipeline
- Diagnostic and validation patterns
- Advanced scenarios and edge cases

**When to use**: When developing aspects, working with the code model, writing templates, or implementing Metalama framework features.

### metalama-teamcity
TeamCity CI/CD integration patterns specific to Metalama builds. Covers:
- Build configuration for Metalama projects
- Test execution and reporting
- Artifact management
- Build triggers and dependencies

**When to use**: When setting up or troubleshooting Metalama builds on TeamCity, or configuring CI/CD pipelines.

## Usage Examples

### Working with Aspects
Ask Claude:
- "How do I test an aspect that adds logging to methods?"
- "What's the correct way to use IMethod in the code model?"
- "How do I create a diagnostic for invalid aspect usage?"

The `metalama-reference` skill will automatically provide relevant API documentation and patterns.

### Test Development
Ask Claude:
- "Show me how to write a snapshot test for my aspect"
- "How do I use @TestScenario with code fixes?"
- "What test directives should I use for this scenario?"

### TeamCity Integration
Ask Claude:
- "How should I configure the Metalama build on TeamCity?"
- "What's the proper artifact structure for Metalama packages?"

The `metalama-teamcity` skill will provide CI/CD guidance.

## Installation

Add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "postsharp-engineering": {
      "source": {
        "source": "github",
        "repo": "postsharp-ops/PostSharp.Engineering.AISkills"
      }
    }
  },
  "enabledPlugins": ["metalama-dev@postsharp-engineering"]
}
```

Or install interactively:
```bash
/plugin marketplace add postsharp-ops/PostSharp.Engineering.AISkills
/plugin install metalama-dev@postsharp-engineering
```

## Troubleshooting

### Skills Not Loading
If the Metalama reference isn't being used:
1. Check that the plugin is enabled in settings
2. Verify you're in a Metalama project directory
3. Try asking explicitly: "Using the metalama-reference skill, how do I..."

### Missing API Documentation
The plugin includes extensive offline reference documentation. If information seems outdated:
1. Check the official Metalama documentation
2. Update the plugin to the latest version
3. Report missing or incorrect information

### Test Failures
When snapshot tests fail:
1. Review the diff between expected and actual output
2. Verify test directives are correctly applied
3. Check that the Metalama pipeline is running as expected
4. Regenerate snapshots if the behavior change is intentional

### Git Operations & Authentication

Check the `RUNNING_IN_DOCKER` environment variable to determine the execution context.

**When running in Docker** (`RUNNING_IN_DOCKER=true`):
- Git operations like `git push` use an **MCP authorization server**
- Claude will prompt for authorization when needed
- No manual authentication setup required

**When running on host** (no `RUNNING_IN_DOCKER` or `RUNNING_IN_DOCKER=false`):
- Uses local git credentials and SSH keys
- Manual `gh auth login` may be required

**General Git Workflow**:
- Most git operations (status, diff, commit) work without authentication
- Push operations require MCP authorization approval (Docker) or local credentials (host)
- Use the `eng` plugin commands for PR creation and release management

## Contributing

This plugin is part of the PostSharp Engineering AI Skills repository. Contributions, bug reports, and suggestions are welcome.

## Related Plugins

- **eng** - General PostSharp/Metalama engineering workflows (git, PRs, releases)

Use both plugins together for comprehensive Metalama development support.
