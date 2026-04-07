# Claude Code XrmToolBox Testing Skill

A [Claude Code](https://claude.ai/claude-code) skill for testing XrmToolBox plugins for Dynamics 365 / Dataverse.

## What It Does

Provides Claude with deep knowledge of XrmToolBox plugin testing patterns so it can:

- Scaffold test projects for existing XrmToolBox plugins
- Create mock `IOrganizationService` implementations for unit testing
- Write smoke tests for services, data loading, and business logic
- Test concurrent and async patterns (thread safety, `SemaphoreSlim`, `BlockingCollection`)
- Validate NuGet packaging and deployment without manual XrmToolBox testing
- Guide integration testing with real Dataverse connections

## Installation

### Via HurleySk Marketplace

If you have the marketplace installed:

```bash
claude plugin install xrmtoolbox-testing@hurleysk-marketplace
```

### As a Claude Code Plugin

```bash
claude plugin add HurleySk/claude-xrmtoolbox-testing-skill
```

### As a Global Skill (manual)

Copy `skills/xrmtoolbox-testing/SKILL.md` to `~/.claude/skills/xrmtoolbox-testing/SKILL.md`.

## Usage

Once installed, use the slash command in any Claude Code session:

```
/xrmtoolbox-testing scaffold
/xrmtoolbox-testing mock
/xrmtoolbox-testing smoke
/xrmtoolbox-testing help
```

Or just mention testing an XrmToolBox plugin in conversation and Claude will use this skill's knowledge automatically.

## Related

- [XrmToolBox Plugin Dev Skill](https://github.com/HurleySk/claude-xrmtoolbox-skill) - Skill for building XrmToolBox plugins
- [XrmToolBox Plugin Template](https://github.com/HurleySk/XrmToolBox-Plugin-Template) - GitHub template for scaffolding new plugins

## License

MIT
