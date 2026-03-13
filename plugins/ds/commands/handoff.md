---
description: Generate a structured handoff for workspace transitions. Creates .context/handoff.md with branch state, recent changes, and next steps so a new session can resume without context loss.
---

# Workspace Handoff

Generate a structured context file for transferring work between agent sessions.
Output goes to `.context/handoff.md` so the next session reads it automatically.

## Usage

```bash
/ds:handoff [resume-command]
```

**Examples:**

```bash
/ds:handoff                    # Generate handoff from current state
/ds:handoff "/ds:triage"       # Handoff with specific resume command
```

Arguments: $ARGUMENTS

## Step 1: Gather Workspace State

Collect in parallel:

```bash
# Branch and remote tracking
git branch --show-current
git log --oneline -10

# Uncommitted work
git status --short
git diff --stat

# Changed files vs default branch
DEFAULT_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
git diff origin/${DEFAULT_BRANCH:-main} --name-only 2>/dev/null | head -50

# Commits ahead of default branch
git rev-list --count origin/${DEFAULT_BRANCH:-main}..HEAD 2>/dev/null || echo 0

# Workspace identity
basename $(pwd)
```

## Step 2: Identify Session Context

Scan the current conversation for:

- What task was in progress (partner ticket, investigation, integration diagnosis)
- Which commands were run and their outcomes
- Any pending actions (drafts awaiting send, Front tags to apply, Jira updates)
- Blockers or open questions
- Partner context (name, integration type, ticket reference)

## Step 3: Generate Handoff

Create `.context/` directory if it does not exist.

Write to `.context/handoff.md` with this structure:

```markdown
# Workspace Handoff

**Generated:** [timestamp]
**Workspace:** [workspace_name]
**Branch:** [branch_name]

## Resume Command

[If $ARGUMENTS provided: the command to run]
[If no argument: description of what to do next]

## State

- **Uncommitted changes:** [yes/no + summary]
- **Commits ahead of ${DEFAULT_BRANCH}:** [count from rev-list]
- **Key files modified:** [list]

## Context

[2-4 sentences: what was happening, what is done, what remains]

## Partner Context

[If applicable: partner name, ticket ID, integration type, current stage]
[If not applicable: "N/A"]

## Pending Actions

[Bulleted list of gated actions not yet completed, or "None"]

## Commands Run This Session

[List of /ds: commands invoked and their outcome (success/partial/failed)]
```

## Step 4: Confirm

Output: "Handoff written to `.context/handoff.md`. Next session will read it automatically."

If the workspace has uncommitted changes, also output: "Warning: uncommitted changes detected. Consider committing before switching sessions."

## Rules

1. Create `.context/` directory if it does not exist: `mkdir -p .context`
2. If no uncommitted changes and no session context worth capturing:
   - Remove stale handoff if it exists: `rm -f .context/handoff.md`
   - Output: "Clean state. No handoff needed."
   - Stop.
3. Keep the handoff concise -- the receiving session needs enough to resume, not a full transcript.
4. Never include sensitive data (API keys, credentials, PII) in the handoff file.
5. `.context/` should already be in `.gitignore`. If it is not, warn the user.
