# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### code-review

Performs a comprehensive review of the specified code scope (code-review, CR, MR), covering multiple dimensions including bugs, performance, security, code quality, and architectural design, and outputs a detailed review report.

**Single Installation**

```bash
npx skills add https://github.com/kvsur/skills --skill code-review
```

**Use when:**
- Reviewing code in a specified directory
- Reviewing a specified file
- Reviewing a specified commit
- Reviewing uncommitted changes in the current repository
- ...


## Installation

```bash
npx skills add https://github.com/kvsur/skills
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**
```
Review `src/index.ts` for me.
```
```
Review this latest commit.
```
```
Review uncommit change for this project.
```

## Skill Structure

Each skill contains:
- `SKILL.md` - Instructions for the agent
- `scripts/` - Helper scripts for automation (optional)
- `references/` - Supporting documentation (optional)

## License

MIT
