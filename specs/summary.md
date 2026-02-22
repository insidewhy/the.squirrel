# The Squirrel

The Squirrel is a Claude Code custom slash command (skill) for implementing tasks from Jira tickets.

It will be invoked like this:

```
/the-squirrel PJ-123 Stop to ask me questions about the improved specification, then ask me any non-obvious questions throughout the process
```

The first word would be the ID of a Jira ticket and the rest of the text are instructions I will call "the trailing instructions" throughout the rest of this document. If no trailing instructions are provided, the `defaultTrailingInstructions` from the configuration is used (see below).

## Configuration

The Squirrel is optionally configured via a `.squirrel.json` file at the root of the project. If this file does not exist, all defaults are used. It has the following options:

- `defaultTrailingInstructions`: the trailing instructions to use when none are provided. Defaults to `Stop to ask me questions about the improved specification, then ask me any non-obvious questions throughout the process`.
- `specificationsPath`: a string representing a directory path where generated specification files will be stored. Defaults to `./specifications`. Each specification file is named `{TICKET-ID}-brief-summary-in-kebab-case.md`, where the summary portion should be ~40 characters (60 max in exceptional cases). For example: `PJ-123-add-user-profile-avatar-upload.md`.
- `projectInstructions`: additional domain context about the current project that supplements the project's `CLAUDE.md`. This is intended for information the skill specifically needs, such as tech stack details, architectural conventions, or domain-specific terminology — as opposed to general coding instructions which belong in `CLAUDE.md`.
- `updateJira`: boolean, defaults to `true`. When `false`, step 9 (updating Jira) is skipped entirely.
- `commitChanges`: boolean, defaults to `false`. When `true`, the skill will offer to commit the changes at the end (see step 10).
- `branchNaming`: free-form text describing how git branches should be named for work items. The skill will interpret this and apply it based on the Jira ticket type and ID. Default behaviour if not specified is `feature/{work item id}` (e.g. `feature/PJ-123`). Example value: `"Use feature/{work item id} for a feature, chore/{work item id} for a task, or fix/{work item id} for a bug fix"`.

## What the skill will do

### Step 1: Gather context from Jira

Query for the Jira work item via the Jira MCP server, and also query work items linked to that work item (parent epics, subtasks, blocking/blocked issues) to gather broader context.

### Step 2: Generate specification

Analyse the work item content (and linked work item content) and the source code in the current working directory, before writing an improved specification to `specificationsPath`. After this it may stop to ask the user about the generated specification depending on the tone and content of the trailing instructions. If the instructions indicate not to talk to the user much (e.g. "Do not bother me at all, implement everything by yourself") it will just echo the specification to the user via Claude Code before continuing to the next step. By default (no trailing instructions, or trailing instructions that don't say otherwise), the skill should pause for user review of the specification before proceeding.

### Step 3: Create a git branch

If the current branch already contains the work item ID (e.g. already on `chore/PJ-123` when implementing `PJ-123`), use the current branch. Otherwise, create a new branch from the current branch, named according to the `branchNaming` configuration (defaulting to `feature/{work item id}`).

### Step 4: Implement

Implement the code described in the generated specification.

### Step 5: Test

Test the code. Run the project's existing test suite and any formatters/linters (as specified in the project's `CLAUDE.md`) first to catch regressions and style issues, then test the new functionality:

- 5.1: If the code relates to a back-end service in a monorepo where the front-end service is also available, it will decide whether to test the back-end directly via REST/tRPC calls etc. or whether to use the dev-browser skill to drive a browser to test the backend via the interface. Depending on the tone and content of the trailing instructions it may make this choice itself or question the user about the choice.
- 5.2: If the code is full-stack or front-end only it will likely test it via the dev-browser skill.

If there are test failures, go back to step 4.

### Step 6: Review via parallel subagents

Spawn parallel subagents (as many as needed for the checks, typically 3-8) to investigate the implementation:

- 6.1: Look for issues like dead code that was introduced or where coding guidelines were not followed.
- 6.2: Compare the code to the specification and look for gaps or bugs.
- 6.3: Assess the code for quality in general.

### Step 7: Address review feedback

Implement the feedback of the parallel agents by going back to step 4. Steps 4-7 can run iteratively until the parallel agents are satisfied with the code quality, up to a maximum of 3 iterations. If issues remain after 3 iterations, stop and report the outstanding concerns to the user. Claude Code may stop to inform/ask users about the parallel agent process depending on the content and tone of the trailing instructions.

### Step 8: Update specification

Update the generated specification with any new learnings discovered during implementation (architectural decisions made, edge cases found, etc.), again potentially asking the user depending on the content and tone of the trailing instructions.

### Step 9: Update Jira (conditional)

Skip this step if `updateJira` is set to `false` in the configuration, or if the trailing instructions specifically say not to update Jira. Otherwise, ask the user for confirmation before adding a comment to the ticket. The comment should summarise the work done, link to the specification file, and note any decisions or open questions.

### Step 10: Commit (conditional)

Skip this step if `commitChanges` is `false` (the default). When enabled, propose a commit message to the user and ask for their feedback before committing. The user may approve, edit, or reject the commit message — iterate on it together until the user is happy, or skip the commit if they decide not to.

### Step 11: Report to user

Inform the user that the implementation is complete. Provide:

- A link to the specification file.
- A summary of what was implemented.
- Any learnings or decisions made during the process.
- Any outstanding concerns from the review subagents (if iterations were exhausted).

## Error handling

- If the Jira ticket ID is invalid or the MCP server is unavailable, inform the user immediately and stop.
- If the implementation fundamentally cannot proceed (e.g. the ticket describes work outside the current repository, or requires infrastructure not available), stop and explain the situation to the user rather than attempting to force a solution.
