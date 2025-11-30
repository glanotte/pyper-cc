# Ruby on Rails + Trailblazer Domain Expertise

This skill provides deep knowledge for building Rails applications using Trailblazer architecture with a clear separation of concerns: thin models, orchestrating operations, domain services, and validation contracts.

## When to Load

Load this skill when working on:
- Trailblazer operations and contracts
- Domain services for business/query logic
- Rails models (thin, associations and scopes only)
- Rails API controllers
- Pundit policies (co-located with domain concepts)
- RSpec testing for Trailblazer
- Background jobs

## Architecture Overview

```
app/
├── concepts/                    # Trailblazer domain organization
│   ├── {domain}/
│   │   ├── operations/         # Orchestrate business logic
│   │   ├── contracts/          # Validation + form helpers
│   │   └── default_policy.rb   # Authorization (Pundit)
├── services/                    # Domain logic, queries, mutations
│   └── {domain}/
│       ├── queries/            # Read operations (find, list, search)
│       └── commands/           # Write operations (create, update)
├── models/                      # Thin - associations, scopes, readers only
└── controllers/                 # HTTP concerns only
```

## Core Principles

### 1. Thin Models
Models contain ONLY:
- Associations (`belongs_to`, `has_many`, etc.)
- Scopes (query shortcuts)
- Simple reader methods (`enabled?`, `full_name`)

Models do NOT contain:
- Validations (use contracts)
- Business logic (use services)
- Mutator methods like `enable!(value)` (use services)
- Callbacks with side effects (use operations)

### 2. Operations Orchestrate
Operations are the entry point for business workflows. They:
- Are called from controllers, jobs, other operations
- Orchestrate by calling services
- Do NOT contain SQL or direct ActiveRecord queries
- Handle the "what" of a workflow, not the "how"

### 3. Services Do the Work
Services contain domain logic:
- Database queries (via query objects)
- Data mutations (via command objects)
- External API calls
- Complex business rules

### 4. Contracts Validate
Contracts handle:
- Input validation
- Can act as form helpers (`@form` in controllers/views)
- Property declarations
- Do NOT contain business logic

### 5. Composition Over Inheritance
Prefer composing behavior through services rather than:
- Adding methods to models
- Deep inheritance hierarchies
- Concerns that hide complexity

## References

See `references/` for detailed guidance on:
- `trailblazer-operations.md` - Operation patterns and orchestration
- `contracts-validation.md` - Contract patterns and form helpers
- `services-layer.md` - Service objects, queries, commands
- `thin-models.md` - Model patterns, scopes, associations
- `policies-authorization.md` - Pundit policies co-located with domains
- `testing-rspec.md` - Testing operations, services, contracts

## Workflows

See `workflows/` for step-by-step guidance on:
- `add-domain-concept.md` - Adding a new domain with full structure
- `add-operation.md` - Creating operations with proper orchestration
- `add-service.md` - Creating query/command services
- `add-api-endpoint.md` - Building JSON:API endpoints
