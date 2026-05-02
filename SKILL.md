---
name: snowmerak-skills
description: A collection of reusable AI agent skills for snowmerak. Provides various tools and utilities for development, code analysis, documentation, and automation tasks.
license: MIT
metadata:
  author: snowmerak
  version: "1.0"
  type: skill-collection
---

# Snowmerak Skills Collection

This repository contains a curated collection of reusable AI agent skills for the snowmerak ecosystem. Each skill is designed to provide specific capabilities that agents can leverage to perform tasks more effectively.

## Overview

Snowmerak Skills is a centralized collection of Agent Skills that provide standardized, reusable capabilities for AI agents. This collection includes skills for:

- **Development**: Code generation, refactoring, debugging
- **Documentation**: Writing, reviewing, and maintaining documentation
- **Analysis**: Code analysis, performance optimization
- **Automation**: Task automation, workflow management
- **Testing**: Test generation, test execution, coverage analysis

## Directory Structure

Each skill in this collection follows the Agent Skills specification:

```
snowmerak-skills/
├── SKILL.md           # This file - collection metadata
├── scripts/           # Executable utilities
├── references/        # Documentation and references
└── assets/            # Templates and resources
```

## Available Skills

Skills are organized in subdirectories. Each skill directory contains its own `SKILL.md` file with metadata and instructions.

### Coming Soon

More skills will be added to this collection. To contribute a skill:

1. Create a new directory under the repository root
2. Add a `SKILL.md` file with proper frontmatter
3. Include any necessary scripts, references, or assets
4. Ensure the directory name matches the `name` field in frontmatter

## Quickstart

To use a skill from this collection:

1. Navigate to the skill directory
2. Read the `SKILL.md` file to understand its capabilities
3. Follow the instructions provided in the skill

## Agent Skills Specification

This collection adheres to the [Agent Skills Specification](references/AGENT-SKILLS-SPEC.md). Key requirements:

- Each skill must have a `SKILL.md` file with YAML frontmatter
- The `name` field must match the directory name
- The `description` field should clearly explain when and how to use the skill
- Keep `SKILL.md` under 500 lines for optimal performance

## Progressive Disclosure

Skills in this collection are designed for progressive disclosure:

- **Metadata** (~100 tokens): Loaded at startup for skill discovery
- **Instructions** (< 5000 tokens): Loaded when skill is activated
- **Resources** (as needed): Loaded on demand for detailed tasks

## Contributing

We welcome contributions to the Snowmerak Skills collection!

### Adding a New Skill

1. Create a new directory with a lowercase, hyphenated name (e.g., `code-review`)
2. Add a `SKILL.md` file with required frontmatter:
   ```yaml
   ---
   name: your-skill-name
   description: Clear description of what the skill does and when to use it.
   ---
   ```
3. Add any optional directories (`scripts/`, `references/`, `assets/`)
4. Test the skill to ensure it works correctly

### Skill Requirements

- `name`: 1-64 characters, lowercase letters, numbers, and hyphens only
- `description`: 1-1024 characters, must describe functionality and use cases
- Directory name must match the `name` field exactly

## Validation

All skills in this collection are validated against the Agent Skills specification:

1. `SKILL.md` file exists
2. Required fields (`name`, `description`) are present
3. `name` follows naming constraints
4. Directory name matches `name` field

## License

This collection is released under the MIT License. Each individual skill may have its own license specified in its `SKILL.md` frontmatter.

## Resources

- [Agent Skills Specification](references/AGENT-SKILLS-SPEC.md) - Complete format specification
- [Agent Skills Documentation](https://agentskills.io) - Official documentation
- [Agent Skills Discord](https://discord.gg/MKPE9g8aUy) - Community support
