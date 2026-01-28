# Command: Make Work

Read VISION.md, ARCHITECTURE.md, and claude/AGENTS.md.

First, check the current state of work:

1. Run `bd list` to see existing epics and issues
2. Check what's in progress, what's completed, what's pending

Then, summarize:

1. What problem this project solves
2. The high-level architecture (major components and how they fit together)
3. The current state of implementation (what's done, what's in progress)

Based on this, propose next steps:

1. If epics exist: suggest new issues to add to existing epics, or identify what to work on next
2. If no epics exist: suggest epics to create and initial issues for each
3. Identify dependencies - what should be built first and why?

When proposing issues, structure each with:

- Requirements: What needs to be built (functional requirements)
- Design Decisions: Key technical choices or constraints to follow
- Acceptance Criteria: How we know it's done (tests, behaviors)

Don't create any issues yet - just propose the breakdown so we can discuss it.

After we agree on the plan and you create epics/issues:

- Commit the beads changes after each epic is created with a clear message
- Run `bd sync` to sync with git

After you implement work:

- Commit your changes with a clear message
- Update the relevant Beads issue status
- Note any new issues discovered during implementation