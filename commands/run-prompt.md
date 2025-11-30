---
name: run-prompt
description: Delegate one or more prompts to fresh sub-task contexts
argument-hint: <prompt-number(s)-or-name> [--parallel|--sequential]
allowed-tools: [Read, Task, Bash(ls:*), Bash(mv:*), Bash(git:*)]
---

<context>
Git status: !`git status --short`
Recent prompts: !`ls -t ./prompts/*.md 2>/dev/null | head -5`
</context>

<objective>
Execute one or more prompts from `./prompts/` as delegated sub-tasks with fresh context.
</objective>

<input>
The user will specify which prompt(s) to run via $ARGUMENTS:

**Single prompt:**
- Empty: Run the most recently created prompt
- A prompt number (e.g., "001", "5")
- A partial filename (e.g., "user-auth")

**Multiple prompts:**
- Multiple numbers (e.g., "005 006 007")
- With flag: "005 006 007 --parallel" or "005 006 007 --sequential"
- Default: --sequential for safety
</input>

<process>
<step1_parse_arguments>
Parse $ARGUMENTS to extract:
- Prompt numbers/names
- Execution strategy flag (--parallel or --sequential)
</step1_parse_arguments>

<step2_resolve_files>
For each prompt number/name:
- If empty or "last": Find with `ls -t ./prompts/*.md | head -1`
- If a number: Find file matching that zero-padded number
- If text: Find files containing that string

<matching_rules>
- If exactly one match: Use that file
- If multiple matches: List them and ask user to choose
- If no matches: Report error and list available prompts
</matching_rules>
</step2_resolve_files>

<step3_execute>
<single_prompt>
1. Read the complete contents of the prompt file
2. Delegate as sub-task using Task tool with subagent_type="general-purpose"
3. Wait for completion
4. Archive prompt to `./prompts/completed/`
5. Return results
</single_prompt>

<parallel_execution>
1. Read all prompt files
2. **Spawn all Task tools in a SINGLE MESSAGE** (critical for parallel execution)
3. Wait for ALL to complete
4. Archive all prompts
5. Return consolidated results
</parallel_execution>

<sequential_execution>
1. For each prompt in order:
   - Read prompt file
   - Spawn Task tool
   - Wait for completion
   - Archive prompt
2. Return consolidated results
</sequential_execution>
</step3_execute>
</process>

<output>
<single_prompt_output>
✓ Executed: ./prompts/005-implement-feature.md
✓ Archived to: ./prompts/completed/

<results>
[Summary of what the sub-task accomplished]
</results>
</single_prompt_output>

<parallel_output>
✓ Executed in PARALLEL:
- ./prompts/005-auth.md
- ./prompts/006-api.md

✓ All archived to ./prompts/completed/

<results>
[Consolidated summary]
</results>
</parallel_output>
</output>

<critical_notes>
- For parallel execution: ALL Task tool calls MUST be in a single message
- Archive prompts only after successful completion
- If any prompt fails in sequential mode, stop and report error
</critical_notes>
