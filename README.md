# pyper-cc

A Claude Code plugin providing meta-prompting workflows, structured todo management, thinking models, and domain expertise for Ruby/Rails and TypeScript/React development.

## Features

### Meta-Prompting System

Create, manage, and execute prompts that Claude can run autonomously:

- **`/create-prompt`** - Generate well-structured prompts with XML formatting, TDD workflows, and domain-specific patterns
- **`/run-prompt`** - Execute prompts in fresh sub-task contexts (supports parallel execution)

### Todo Management

Persistent todo tracking that survives context switches:

- **`/add-to-todos`** - Add structured todos with problem descriptions, file references, and solution hints
- **`/check-todos`** - Review outstanding work, get context loaded, and pick what to work on next
- **`/whats-next`** - Generate comprehensive handoff documents for continuing work in fresh contexts

### Thinking Models

Mental frameworks for better problem-solving (in `commands/consider/`):

| Command | Purpose |
|---------|---------|
| `/consider:5-whys` | Drill to root cause by asking "why" repeatedly |
| `/consider:first-principles` | Break down to fundamentals, rebuild from base truths |
| `/consider:inversion` | Solve backwards - what guarantees failure? |
| `/consider:pareto` | Apply 80/20 rule to focus on what matters most |
| `/consider:second-order` | Think through consequences of consequences |

### Debugging

- **`/debug`** - Systematic debugging methodology with evidence gathering, hypothesis testing, and verification

### Domain Expertise Skills

Pre-loaded knowledge for specific tech stacks:

#### Ruby on Rails + Trailblazer
- Thin models (associations, scopes, readers only)
- Trailblazer operations for orchestration
- Reform contracts for validation
- Service layer (queries, commands, validators)
- RSpec testing patterns

#### TypeScript + React
- Component patterns (composition, compound components)
- TypeScript best practices (generics, discriminated unions)
- React hooks patterns
- React Testing Library approaches

## Installation

### Using Claude Code Plugin Manager

```bash
claude plugin add glanotte/pyper-cc
```

### Manual Installation

Clone this repository into your `.claude/plugins/` directory:

```bash
mkdir -p ~/.claude/plugins
cd ~/.claude/plugins
git clone https://github.com/glanotte/pyper-cc.git
```

## Usage

### Creating and Running Prompts

```
/create-prompt add user authentication with JWT tokens
```

Claude will:
1. Ask clarifying questions about the task
2. Detect your testing framework
3. Generate a structured prompt with TDD workflow
4. Save to `./prompts/001-user-authentication.md`

Then run it:

```
/run-prompt 001
```

Or run multiple prompts in parallel:

```
/run-prompt 001 002 003 --parallel
```

### Managing Todos

Add a todo from conversation context:

```
/add-to-todos
```

Or with a specific description:

```
/add-to-todos fix the login redirect bug
```

Review and work on todos:

```
/check-todos
```

### Using Thinking Models

Apply structured thinking to any problem:

```
/consider:first-principles should we use microservices?
```

```
/consider:pareto optimizing our CI pipeline
```

### Debugging

Start a systematic investigation:

```
/debug users can't complete checkout
```

### Creating Handoffs

Before ending a session, create a handoff document:

```
/whats-next
```

This generates `whats-next.md` with everything needed to continue in a fresh context.

## Project Structure

```
pyper-cc/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── commands/
│   ├── add-to-todos.md      # Todo management
│   ├── check-todos.md
│   ├── create-prompt.md     # Meta-prompting
│   ├── run-prompt.md
│   ├── debug.md             # Debugging workflow
│   ├── whats-next.md        # Context handoff
│   └── consider/            # Thinking models
│       ├── 5-whys.md
│       ├── first-principles.md
│       ├── inversion.md
│       ├── pareto.md
│       └── second-order.md
└── skills/
    ├── debug-like-expert/   # Debugging methodology
    └── expertise/
        ├── ruby-rails/      # Rails + Trailblazer patterns
        │   ├── SKILL.md
        │   └── references/
        └── typescript-react/ # React + TypeScript patterns
            ├── SKILL.md
            └── references/
```

## Prompt Format

Prompts created by `/create-prompt` follow this structure:

```xml
<objective>
What needs to be accomplished
</objective>

<context>
Tech stack, relevant files, constraints
</context>

<requirements>
Specific functional requirements
</requirements>

<tdd_workflow mode="strict">
Test-first development process
</tdd_workflow>

<output>
Expected files and their purposes
</output>

<verification>
How to confirm the work is complete
</verification>
```

## Todo Format

Todos in `TO-DOS.md` follow this structure:

```markdown
## Brief Context Title - YYYY-MM-DD HH:MM

- **[Action verb] [Component]** - [Brief description]. **Problem:** [What's wrong]. **Files:** [Paths with line numbers]. **Solution:** [Approach hints].
```

## Philosophy

This plugin is built around several core ideas:

1. **Fresh contexts are powerful** - Rather than accumulating stale context, create prompts that can run in fresh sub-tasks
2. **Structured thinking beats ad-hoc** - Mental models and frameworks improve problem-solving
3. **TDD by default** - Test-first development catches issues early
4. **Domain expertise matters** - Pre-loaded patterns for specific stacks reduce cognitive load
5. **Handoffs should be seamless** - Context switches shouldn't lose information

## Contributing

Contributions are welcome! Please feel free to submit issues and pull requests.

## License

MIT License - see [LICENSE](LICENSE) for details.

## Author

Created by [glanotte](https://github.com/glanotte)

## Acknowledgments

- Built for [Claude Code](https://claude.ai/claude-code) by Anthropic
- Inspired by the meta-prompting patterns from the AI engineering community
- Thinking models adapted from classical problem-solving frameworks
