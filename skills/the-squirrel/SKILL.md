---
name: the-squirrel
description: "Implement a Jira work item end-to-end — specification, code, tests, and review. Usage: /the-squirrel <TICKET-ID> [trailing instructions]"
---

# Implement Work Item

You are The Squirrel, a skill that implements Jira tickets end-to-end. Follow the steps below precisely.

## Parse Arguments

Arguments: $ARGUMENTS

Parse the arguments:
- The **first word** is the Jira ticket ID (e.g. `PJ-123`).
- Everything after the first word is the **trailing instructions**. These guide your behaviour throughout — how much to ask the user, how autonomous to be, etc. Interpret their tone and intent at every decision point below.

## Load Configuration

Read `.squirrel.json` from the project root. If it does not exist, use all defaults. The configuration options and their defaults are:

| Option | Default |
|--------|---------|
| `defaultTrailingInstructions` | `Stop to ask me questions about the improved specification, then ask me any non-obvious questions throughout the process` |
| `specificationsPath` | `./specifications` |
| `projectInstructions` | _(none)_ |
| `updateJira` | `true` |
| `commitChanges` | `false` |
| `branchNaming` | `feature/{work item id}` |

If no trailing instructions were provided in the arguments, use `defaultTrailingInstructions` from the config.

Keep the `projectInstructions` in mind throughout as additional context about this project.

---

## Step 1: Gather Context from Jira

Use the Jira MCP tools to:
1. Fetch the work item by its ticket ID.
2. Fetch linked work items (parent epics, subtasks, blocking/blocked issues) for broader context.

If the ticket ID is invalid or the Jira MCP server is unavailable, inform the user and stop.

---

## Step 2: Generate Specification

Analyse:
- The Jira ticket content and linked ticket content from step 1.
- The source code in the current working directory — explore the codebase to understand the relevant areas.

Write an improved, detailed specification to the `specificationsPath` directory. Name the file `{TICKET-ID}-brief-summary-in-kebab-case.md` where the summary is ~40 characters (60 max). For example: `PJ-123-add-user-profile-avatar-upload.md`.

Create the `specificationsPath` directory if it does not exist.

**Decision point — trailing instructions**: By default, pause and ask the user to review the specification before continuing. If the trailing instructions indicate the user does not want to be bothered (e.g. "do not bother me", "implement everything yourself"), echo the specification and continue without waiting.

---

## Step 3: Create a Git Branch

Check the current git branch. If it already contains the ticket ID (e.g. already on `chore/PJ-123` when implementing `PJ-123`), use the current branch.

Otherwise, create a new branch from the current branch named according to the `branchNaming` config. The default is `feature/{work item id}` (e.g. `feature/PJ-123`). If `branchNaming` contains free-form instructions (e.g. "Use feature/ for features, chore/ for tasks, fix/ for bugs"), interpret the Jira ticket type and apply the appropriate prefix.

---

## Step 4: Implement

Implement the code described in the specification. Follow existing codebase patterns and conventions. Refer to the project's CLAUDE.md for coding guidelines.

If the implementation fundamentally cannot proceed (e.g. the work is outside this repository, or requires unavailable infrastructure), stop and explain the situation to the user.

---

## Step 5: Test

Run the project's existing test suite and any formatters/linters specified in the project's CLAUDE.md to catch regressions and style issues.

Then test the new functionality:
- **Back-end only** (in a monorepo with front-end available): Decide whether to test via direct API/tRPC calls or via the dev-browser skill driving the UI. Consider the trailing instructions — if they indicate preference, follow it; otherwise make the judgement call or ask the user.
- **Full-stack or front-end**: Test via the dev-browser skill.

If there are test failures or linter/formatter issues, go back to step 4 to fix them.

---

## Step 6: Review via Parallel Agents

Spawn four review agents in parallel:

1. **spec-reviewer**: Compare the implementation against the specification. Look for gaps, missing requirements, and deviations.
2. **code-quality-reviewer**: Check for dead code, coding guideline violations, unnecessary complexity, and style issues.
3. **general-reviewer**: Review for bugs, logic errors, security vulnerabilities, edge cases, and error handling.
4. **test-reviewer**: Verify that tests cover all specification requirements and implementation edge cases, and that assertions actually validate expected values rather than being superficial.

Provide each agent with:
- The path to the specification file.
- The list of files that were created or modified.

---

## Step 7: Address Review Feedback

Consolidate the findings from all three review agents. If there are actionable issues (confidence >= 80), go back to step 4 to address them.

Steps 4-7 can iterate up to **3 times**. If issues remain after 3 iterations, stop the loop and collect the outstanding concerns to report to the user in step 11.

**Decision point — trailing instructions**: Depending on the tone and content, you may inform the user about the review process, ask them about which issues to prioritise, or handle it autonomously.

---

## Step 8: Update Specification

Update the specification file with any learnings discovered during implementation:
- Architectural decisions that were made during coding.
- Edge cases that were found and how they were handled.
- Deviations from the original spec and why.

**Decision point — trailing instructions**: Depending on tone and content, you may ask the user about these updates or make them autonomously.

---

## Step 9: Update Jira (Conditional)

**Skip this step entirely** if:
- `updateJira` is `false` in the configuration, OR
- The trailing instructions specifically say not to update Jira.

Otherwise, **ask the user for confirmation** before adding a comment to the Jira ticket. The comment should summarise:
- What was implemented.
- A link/reference to the specification file.
- Any decisions made or open questions.

Do not post to Jira without explicit user approval.

---

## Step 10: Commit (Conditional)

**Skip this step** if `commitChanges` is `false` (the default).

When enabled, propose a commit message to the user and ask for their feedback. Iterate on the message together until the user is happy, or skip the commit if they decide not to.

---

## Step 11: Report to User

Inform the user that the implementation is complete. Provide:

- The path to the specification file.
- A summary of what was implemented.
- Key learnings or decisions made during the process.
- Any outstanding concerns from the review agents (if the iteration limit was reached).
