# Zig 0.15.x Agent Skill

An [Agent Skill](https://agentskills.io/) for the Zig programming language, targeting version 0.15.x.

## Why This Exists

LLMs struggle with Zig more than languages like TypeScript or Swift. The language is still evolving, there's less training data available, and models frequently use deprecated or removed syntax. This skill provides up-to-date context to guide models toward correct, idiomatic Zig code.

## What's Included

- **SKILL.md**: 357 lines covering critical patterns and common mistakes
- **references/**: 51 files totaling ~21,500 lines of documentation
  - Standard library module documentation
  - Idiomatic patterns (from [Zig Code Patterns](https://ziggit.dev/t/code-patterns/1748))
  - Build system guidance
  - Language reference content

## How It Was Made

Generated using Claude Opus 4.5 with the Zig standard library source code as context. Each module was documented by feeding the model the actual source files and using the [skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md) to structure the output.

The methodology prioritizes grounding the model in real source code rather than hallucinated documentation.

## Usage

Install to `~/.claude/skills/` per [Claude Code skill documentation](https://docs.anthropic.com/en/docs/claude-code/skills).

Example prompt:
```
Using the zig skill please review @src/ and help me improve the codebase.
```

## Known Limitations

- Build system section may have rough edges
- Some explanatory text may have slight imprecision (code examples are generally correct)
- Patterns reference went through multiple iterations

## Specification

This skill follows the [Agent Skills Specification](https://agentskills.io/).

## License

MIT
