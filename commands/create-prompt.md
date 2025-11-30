---
description: Create a new prompt that another Claude can execute
argument-hint: [task description]
allowed-tools: [Read, Write, Glob, SlashCommand, AskUserQuestion]
---

<context>
Before generating prompts:
1. Use Glob to check `./prompts/*.md` to determine next sequence number
2. Auto-detect testing framework by checking for:
   - `package.json` with jest/vitest → JavaScript/TypeScript (Jest or Vitest)
   - `Gemfile` with rspec → Ruby (RSpec)
   - `spec/` or `test/` directories → infer from structure
3. TDD mode is **STRICT by default** for all coding tasks
</context>

<objective>
Act as an expert prompt engineer for Claude Code, specialized in crafting optimal prompts using XML tag structuring and best practices.

Create highly effective prompts for: $ARGUMENTS

Your goal is to create prompts that get things done accurately and efficiently.
</objective>

<process>

<step_0_intake_gate>
<title>Adaptive Requirements Gathering</title>

<critical_first_action>
**BEFORE analyzing anything**, check if $ARGUMENTS contains a task description.

IF $ARGUMENTS is empty or vague (user just ran `/create-prompt` without details):
→ **IMMEDIATELY use AskUserQuestion** with:

- header: "Task type"
- question: "What kind of prompt do you need?"
- options:
  - "Coding task" - Build, fix, or refactor code
  - "Analysis task" - Analyze code, data, or patterns
  - "Research task" - Gather information or explore options

After selection, ask: "Describe what you want to accomplish" (they select "Other" to provide free text).

IF $ARGUMENTS contains a task description:
→ Skip this handler. Proceed directly to adaptive_analysis.
</critical_first_action>

<adaptive_analysis>
Analyze the user's description to extract and infer:

- **Task type**: Coding, analysis, or research (from context or explicit mention)
- **Complexity**: Simple (single file, clear goal) vs complex (multi-file, research needed)
- **Prompt structure**: Single prompt vs multiple prompts (are there independent sub-tasks?)
- **Execution strategy**: Parallel (independent) vs sequential (dependencies)
- **Tech stack detection**: Ruby/Rails backend or TypeScript/React frontend (load domain expertise if applicable)
- **TDD mode**: Default STRICT for coding tasks (can be overridden to RELAXED)

Inference rules:
- Dashboard/feature with multiple components → likely multiple prompts
- Bug fix with clear location → single prompt, simple, TDD: write failing test first
- "Optimize" or "refactor" → needs specificity about what/where, TDD: ensure existing tests pass first
- Authentication, payments, complex features → complex, needs context
- Rails models/controllers/views → load ruby-rails expertise, use RSpec
- React components/hooks → load typescript-react expertise, use Jest/Vitest
</adaptive_analysis>

<tdd_framework_detection>
Before asking questions, auto-detect the testing framework:

1. **Check for JavaScript/TypeScript projects:**
   - Read `package.json` and look for:
     - `jest` in dependencies/devDependencies → Jest
     - `vitest` in dependencies/devDependencies → Vitest
     - Test scripts containing "jest" or "vitest"

2. **Check for Ruby/Rails projects:**
   - Read `Gemfile` and look for:
     - `rspec` or `rspec-rails` → RSpec
     - `minitest` → Minitest

3. **Fallback detection:**
   - `spec/` directory exists → likely RSpec
   - `test/` directory exists → likely Minitest or Jest
   - `__tests__/` directory exists → likely Jest

Store detected framework for use in prompt generation.
</tdd_framework_detection>

<contextual_questioning>
Generate 2-4 questions using AskUserQuestion based ONLY on genuine gaps.

<question_templates>

**For TDD mode** (ask for ALL coding tasks):
- header: "TDD Mode"
- question: "TDD is enabled by default. How strict should test enforcement be?"
- options:
  - "Strict (default)" - Tests must pass before AND after. Bug fixes require failing test first.
  - "Relaxed" - Tests recommended but not blocking. Good for prototyping.
  - "Skip TDD" - No test requirements. Only for non-production code.

**For ambiguous scope** (e.g., "build a dashboard"):
- header: "Dashboard type"
- question: "What kind of dashboard is this?"
- options:
  - "Admin dashboard" - Internal tools, user management, system metrics
  - "Analytics dashboard" - Data visualization, reports, business metrics
  - "User-facing dashboard" - End-user features, personal data, settings

**For unclear target** (e.g., "fix the bug"):
- header: "Bug location"
- question: "Where does this bug occur?"
- options:
  - "Frontend/UI" - Visual issues, user interactions, rendering
  - "Backend/API" - Server errors, data processing, endpoints
  - "Database" - Queries, migrations, data integrity

**For Rails tasks**:
- header: "Rails layer"
- question: "Which layer needs work?"
- options:
  - "Model" - ActiveRecord, validations, associations, scopes
  - "Controller" - Actions, filters, params handling
  - "Service/Job" - Business logic, background processing

**For React tasks**:
- header: "React layer"
- question: "Which part of the frontend?"
- options:
  - "Component" - UI elements, props, state
  - "State management" - Context, hooks, data flow
  - "API integration" - Fetching, mutations, caching

</question_templates>

<question_rules>
- Only ask about genuine gaps - don't ask what's already stated
- Each option needs a description explaining implications
- Prefer options over free-text when choices are knowable
- User can always select "Other" for custom input
- 2-4 questions max per round
</question_rules>
</contextual_questioning>

<decision_gate>
After receiving answers, present decision gate using AskUserQuestion:

- header: "Ready"
- question: "I have enough context to create your prompt. Ready to proceed?"
- options:
  - "Proceed" - Create the prompt with current context
  - "Ask more questions" - I have more details to clarify
  - "Let me add context" - I want to provide additional information

If "Ask more questions" → generate 2-4 NEW questions based on remaining gaps, then present gate again
If "Let me add context" → receive additional context via "Other" option, then re-evaluate
If "Proceed" → continue to generation step
</decision_gate>

<finalization>
After "Proceed" selected, state confirmation:

"Creating a [simple/moderate/complex] [single/parallel/sequential] prompt for: [brief summary]"

Then proceed to generation.
</finalization>
</step_0_intake_gate>

<step_1_generate_and_save>
<title>Generate and Save Prompts</title>

<pre_generation_analysis>
Before generating, determine:

1. **Single vs Multiple Prompts**
2. **Execution Strategy** (if multiple): Parallel or Sequential
3. **Domain expertise**: Load ruby-rails or typescript-react references if applicable
4. **Required tools**: File references, bash commands, MCP servers

</pre_generation_analysis>

Create the prompt(s) and save to the prompts folder.

**Prompt Construction Rules**

Always Include:
- XML tag structure with clear, semantic tags
- Contextual information: Why this task matters, what it's for
- Explicit, specific instructions
- Sequential steps with numbered lists
- File output instructions using relative paths
- Reference to reading the CLAUDE.md for project conventions
- Explicit success criteria

Conditionally Include:
- **Extended thinking triggers** for complex reasoning
- **Domain expertise references** for Rails or React tasks
- `<research>` tags when codebase exploration is needed
- `<validation>` tags for tasks requiring verification

Output Format:
1. Generate prompt content with XML structure
2. Save to: `./prompts/[number]-[descriptive-name].md`
3. File should contain ONLY the prompt, no explanations

<prompt_patterns>

For Coding Tasks:

```xml
<objective>
[Clear statement of what needs to be built/fixed/refactored]
</objective>

<context>
[Project type, tech stack, relevant constraints]
@[relevant files to examine]
</context>

<requirements>
[Specific functional requirements]
</requirements>

<implementation>
[Specific approaches or patterns to follow]
</implementation>

<output>
Create/modify files with relative paths:
- `./path/to/file.ext` - [what this file should contain]
</output>

<verification>
Before declaring complete, verify your work:
- [Specific test or check to perform]
</verification>
```

For Rails Tasks:

```xml
<objective>
[What needs to be built/fixed in Rails]
</objective>

<context>
Tech stack: Ruby on Rails
@Gemfile
@config/routes.rb
@[relevant models/controllers]
</context>

<rails_conventions>
- Follow Rails conventions (convention over configuration)
- Use ActiveRecord patterns appropriately
- Keep controllers thin, models fat (or use service objects)
- Write specs with RSpec
</rails_conventions>

<requirements>
[Specific requirements]
</requirements>

<output>
- `./app/models/[name].rb`
- `./app/controllers/[name]_controller.rb`
- `./spec/models/[name]_spec.rb`
</output>
```

For React Tasks:

```xml
<objective>
[What needs to be built in React]
</objective>

<context>
Tech stack: TypeScript + React
@package.json
@[relevant components]
</context>

<react_conventions>
- Use TypeScript with proper typing
- Functional components with hooks
- Follow existing patterns in codebase
</react_conventions>

<requirements>
[Specific requirements]
</requirements>

<output>
- `./src/components/[Name].tsx`
- `./src/components/[Name].test.tsx`
</output>
```

</prompt_patterns>

<tdd_workflow_template>
**CRITICAL: For ALL coding tasks, include this TDD workflow section based on selected mode:**

**STRICT TDD Mode (default):**
```xml
<tdd_workflow mode="strict">
<phase_1_baseline>
**BEFORE ANY CODE CHANGES:**
1. Run the full test suite: `[test_command]`
2. If tests fail: STOP. Fix existing failures first or confirm with user.
3. Document baseline: "[X] tests passing, [Y] pending, [Z] failing"
</phase_1_baseline>

<phase_2_test_first>
**For Bug Fixes:**
1. Write a failing test that reproduces the bug
2. Run test to confirm it FAILS (red)
3. Only then implement the fix
4. Run test to confirm it PASSES (green)

**For New Features:**
1. Write tests for the new functionality FIRST
2. Run tests to confirm they FAIL (red)
3. Implement minimum code to pass
4. Refactor if needed while keeping tests green
</phase_2_test_first>

<phase_3_verification>
**BEFORE DECLARING COMPLETE:**
1. Run full test suite: `[test_command]`
2. All tests must pass (green)
3. No new warnings or deprecations introduced
4. Test coverage should not decrease
</phase_3_verification>

<failure_protocol>
If tests fail after implementation:
1. DO NOT proceed to next task
2. Fix the failing tests
3. If stuck, ask user for guidance
4. Never skip or delete tests to make them pass
</failure_protocol>
</tdd_workflow>
```

**RELAXED TDD Mode:**
```xml
<tdd_workflow mode="relaxed">
<guidelines>
- Tests are strongly recommended but not blocking
- Run tests before and after changes when possible
- For bug fixes, try to add a regression test
- Prioritize working code, add tests as time permits
</guidelines>

<verification>
Before declaring complete:
1. Run tests if available: `[test_command]`
2. Note any failures in your summary
3. Suggest tests to add for future stability
</verification>
</tdd_workflow>
```

**Skip TDD Mode:**
```xml
<tdd_workflow mode="skip">
<note>TDD enforcement disabled for this task. Tests are optional.</note>
</tdd_workflow>
```

**Test Command Detection:**
- Jest/Vitest: `npm test` or `npm run test` or `npx vitest`
- RSpec: `bundle exec rspec` or `rspec`
- Minitest: `bundle exec rails test` or `ruby -Itest`
</tdd_workflow_template>
</prompt_patterns>
</step_1_generate_and_save>

<decision_tree>
After saving the prompt(s), present options:

**Prompt(s) created successfully!**

✓ Saved prompt to ./prompts/[filename]

What's next?

1. Run prompt now
2. Review/edit prompt first
3. Save for later

If user chooses #1, invoke via SlashCommand tool: `/run-prompt [number]`
</decision_tree>
</process>

<success_criteria>
- Intake gate completed (AskUserQuestion used for clarification if needed)
- User selected "Proceed" from decision gate
- Testing framework auto-detected (for coding tasks)
- TDD mode selected (default: STRICT)
- Prompt(s) generated with proper XML structure including TDD workflow
- Files saved to ./prompts/[number]-[name].md
- Decision tree presented to user
</success_criteria>
