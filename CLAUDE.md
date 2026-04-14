# Hakver — Project Rules

## Language

- Code, file names, variable names, comments, commit messages: **English**
- User-facing content (UI, error messages, SEO, seed data): **Turkish**
- Inter-agent communication: **English**
- Responses to user: **Turkish**, short and clear

## Response Style

- Short, clear, no filler. Use bullet points.
- Report what was done, what's pending, what's blocked.
- Don't explain concepts the user already knows, don't repeat what the user said.

## Inter-Agent Communication Protocol

Structured English messages. No free-form chat.

**Task assignment:**

```
TASK: [description]
CONTEXT: [files, constraints]
ACCEPTANCE: [completion criteria]
```

**Completion:**

```
STATUS: DONE | BLOCKED | NEEDS_REVIEW
SUMMARY: [max 3 lines]
FILES_CHANGED: [list]
ISSUES: [if any, otherwise "none"]
```

**Review:**

```
VERDICT: APPROVE | REQUEST_CHANGES
CRITICAL: [n] IMPORTANT: [n] MINOR: [n]
DETAILS: [findings]
```

**Blocker:**

```
BLOCKED: [what]
REASON: [why]
NEEDS: [what's required to unblock]
```

## Hallucination Prevention

- Verify before claiming — check if file exists, if function works.
- Read before modifying.
- Don't assume — explore. Learn project structure and patterns from code.
- Don't create new schema/type/endpoint if one already exists.
- Run lint, build, test after changes. Report actual output, not expected.
- After 3 failed attempts, report BLOCKED. Subagents report to team-lead, team-lead reports to user. Don't enter infinite loops.

## Learning Project Context

Agents learn project-specific details from code — not from hardcoded instructions:

1. Read this file
2. Explore directory structure
3. Read existing code — learn patterns
4. Check shared package for existing schemas/types
5. Every change (bug fix, feature, edit) must follow existing project patterns — follow current structure before creating new patterns

## Compact Instructions

Preserve during compaction: list of changed files, active task status, architectural decisions made, failed approaches and their reasons.
