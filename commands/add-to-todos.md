---
description: Add todo item to TO-DOS.md with context from conversation
argument-hint: <todo-description> (optional - infers from conversation if omitted)
allowed-tools:
  - Read
  - Edit
  - Write
---

# Add Todo Item

## Context

- Current timestamp: !`date "+%Y-%m-%d %H:%M"`

## Instructions

1. Read TO-DOS.md in the working directory (create with Write tool if it doesn't exist)

2. Check for duplicates:
   - Extract key concept/action from the new todo
   - Search existing todos for similar titles or overlapping scope
   - If found, ask user: "A similar todo already exists: [title]. Skip, replace, or add anyway?"
   - Wait for user response before proceeding

3. Extract todo content:
   - **With $ARGUMENTS**: Use as the focus/title for the todo
   - **Without $ARGUMENTS**: Analyze recent conversation to extract:
     - Specific problem or task discussed
     - Relevant file paths that need attention
     - Technical details (line numbers, error messages)
     - Root cause if identified

4. Append new section to bottom of file:
   - **Heading**: `## Brief Context Title - YYYY-MM-DD HH:MM` (3-8 word title)
   - **Todo format**: `- **[Action verb] [Component]** - [Brief description]. **Problem:** [What's wrong/why needed]. **Files:** [Paths with line numbers]. **Solution:** [Approach hints, if applicable].`
   - **Required fields**: Problem and Files (with line numbers like `path/to/file.ts:123-145`)
   - **Optional field**: Solution
   - Make each section self-contained for future Claude to understand weeks later

5. Confirm and offer to continue:
   - Confirm: "✓ Saved to todos."
   - Ask: "Continue with [original task]?"

## Format Example

```markdown
## Add Todo Command Improvements - 2025-11-15 14:23

- **Add structured format to add-to-todos** - Standardize todo entries. **Problem:** Current todos lack consistent structure. **Files:** `commands/add-to-todos.md:22-29`. **Solution:** Use inline bold labels with required Problem and Files fields.
```
