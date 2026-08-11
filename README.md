# PostSharp Engineering AI Skills

Claude Code plugin marketplace for PostSharp and Metalama development workflows.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [eng](./plugins/eng/) | Git conventions, PRs, releases, TeamCity CI/CD, build diagnostics |
| [metalama-dev](./plugins/metalama-dev/) | Metalama framework development: aspect testing, test directives |

## Installation

### Add Marketplace

```bash
/plugin marketplace add postsharp-ops/PostSharp.Engineering.AISkills
```

### Install Plugin

```bash
/plugin install eng@postsharp-engineering
```

### Team Configuration

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
  "enabledPlugins": ["eng@postsharp-engineering"]
}
```

## Plugin: eng

PostSharp/Metalama engineering workflows:

### Commands

| Command | Description |
|---------|-------------|
| `/eng:create-pr` | Create PR with metadata, milestone, issue linking |
| `/eng:prepare-release` | Prepare GitHub release for a milestone |
| `/eng:tc-build` | Trigger TeamCity build |
| `/eng:tc-check-build` | Check TeamCity build status |
| `/eng:fix-binlog-warnings` | Fix compiler warnings from MSBuild binlog files |

### Skills

The `eng` skill provides git workflow conventions:
- Branch naming: `topic/YYYY.N/XXXX-description`
- Commit messages: Short, imperative, with issue numbers
- Merge targets: Always `develop/YYYY.N`, never default branch
- GitHub API patterns for milestones, project status, issue linking

## Recommended Directory Structure

Organize your source code with product families at the first level, repos underneath:

```
C:\src\
├── Metalama-2025.1\           # Product family (specific major version)
│   ├── Metalama\              # Main repo
│   ├── Metalama.Premium\
│   ├── Metalama.Documentation\
│   └── Metalama.Samples\
├── Metalama-2026.0\           # Another major version
│   ├── Metalama\
│   ├── Metalama.Premium\
│   └── ...
├── PostSharp.Engineering\     # Cross-product tooling (not version-specific)
│   ├── PostSharp.Engineering\
│   └── PostSharp.Engineering.AISkills\
└── ...
```

This structure allows:
- Working on multiple major versions simultaneously
- Shared tooling repos outside product families
- Clear separation between version-specific and cross-cutting concerns

## Contributing

**Developers are encouraged to clone this repo locally and improve it.**

For local development, reference the local path in your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "postsharp-engineering": {
      "source": "C:/src/PostSharp.Engineering/PostSharp.Engineering.AISkills"
    }
  },
  "enabledPlugins": ["eng@postsharp-engineering"]
}
```

When you've made improvements:
1. Test your changes locally
2. Submit a PR back to this repository
3. Once merged, teams using the GitHub source will get the updates automatically

## License

MIT
