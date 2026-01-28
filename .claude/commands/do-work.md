# Command: Do Work

Read VISION.md, ARCHITECTURE.md, and claude/AGENTS.md for context.

Check the current state:

1. Run `bd ready` to see available work
2. Pick the highest priority ready issue
3. Run `bd update <issue-id> --status in_progress` to claim it

Implement that issue:

1. Read the issue description carefully
2. Implement according to the Requirements and Design Decisions
3. Verify the Acceptance Criteria are met
4. Write tests if specified

After implementation:

1. Run any tests to verify your work
2. Calculate tokens used (starting tokens - current tokens from context)
3. Log token usage and close issue:
   ```bash
   bd comments add <issue-id> "tokens: <count>"
   bd close <issue-id>
   ```
4. Commit your changes AND the .beads/issues.jsonl file together:

   ```bash
   git add -A
   git commit -m "Your commit message"
   ```

5. If you discovered new work or issues, file them in Beads

IMPORTANT: 
- Always commit .beads/issues.jsonl along with your code changes
- Track token usage for every issue closed
- Issue status changes need to be committed too!

Show me what you completed and what's next.