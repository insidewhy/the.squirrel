# The Squirrel

A Claude Code skill that takes a Jira ticket and turns it into a tested, reviewed implementation — generating a specification, writing the code, running tests, and reviewing via parallel agents.

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- [Atlassian MCP server](https://mcp.atlassian.com) configured in Claude Code
- [dev-browser](https://github.com/SawyerHood/dev-browser) skill (optional, for front-end/full-stack testing)

## Usage

```
/the-squirrel PJ-123
```

Or with custom instructions:

```
/the-squirrel PJ-123 Do not bother me at all, implement everything by yourself
```

When no trailing instructions are provided, the configured default is used (see below).

## Configuration

Optionally create a `.squirrel.json` at the root of your project. If this file does not exist, all defaults are used.

```json
{
  "specificationsPath": "./specifications",
  "defaultTrailingInstructions": "Stop to ask me questions about the improved specification, then ask me any non-obvious questions throughout the process",
  "branchNaming": "Use feature/{work item id} for a feature, chore/{work item id} for a task, or fix/{work item id} for a bug fix",
  "updateJira": true,
  "projectInstructions": "Next.js app with tRPC and Prisma. PostgreSQL database."
}
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `specificationsPath` | string | `./specifications` | Directory where generated spec files are stored |
| `defaultTrailingInstructions` | string | `"Stop to ask me questions about the improved specification, then ask me any non-obvious questions throughout the process"` | Trailing instructions used when none are provided |
| `branchNaming` | string | `feature/{work item id}` | Free-form text describing branch naming conventions |
| `commitChanges` | boolean | `false` | Whether to offer committing the changes (asks for user feedback on the message first) |
| `updateJira` | boolean | `true` | Whether to offer updating the Jira ticket with a summary comment |
| `projectInstructions` | string | — | Domain context that supplements the project's `CLAUDE.md` |

## How it works

1. **Gather context** — Fetches the Jira ticket and linked work items for context.
2. **Generate specification** — Analyses the ticket and codebase, writes a spec to `specificationsPath`.
3. **Branch** — Creates a git branch (or reuses the current one if it already matches the ticket).
4. **Implement** — Writes the code described in the specification.
5. **Test** — Runs existing tests, formatters, and linters for regressions and style issues, then tests new functionality (via CLI or dev-browser).
6. **Review** — Spawns parallel subagents to check for dead code, spec gaps, and quality issues.
7. **Iterate** — Addresses review feedback (steps 4-7 repeat up to 3 times).
8. **Update spec** — Records any learnings discovered during implementation.
9. **Update Jira** — Optionally comments on the ticket (asks for confirmation first).
10. **Commit** — Optionally commits changes (proposes a message and iterates with the user first).
11. **Report** — Provides a summary with a link to the spec.

See [specs/summary.md](specs/summary.md) for the full specification.
