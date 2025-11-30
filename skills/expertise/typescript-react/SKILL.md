# TypeScript React Domain Expertise

This skill provides deep TypeScript and React knowledge for building robust, type-safe frontend applications.

## When to Load

Load this skill when working on:
- React components (functional, with hooks)
- TypeScript types and interfaces
- State management (Context, hooks, Zustand, etc.)
- React Query / data fetching
- Form handling (React Hook Form, etc.)
- Component testing

## Core Principles

### TypeScript First
- Define types before implementation
- Avoid `any` - use `unknown` and narrow
- Use generics for reusable components
- Let TypeScript infer when possible

### Functional Components Only
Class components are legacy. Always use functional components with hooks.

### Composition Over Inheritance
Build complex UIs by composing simple components.

### Colocation
Keep related code together:
- Component + its types + its styles + its tests in same directory
- State close to where it's used

## References

See `references/` for detailed guidance on:
- `component-patterns.md` - Component structure and patterns
- `typescript-patterns.md` - TypeScript best practices
- `hooks-patterns.md` - Custom hooks and hook rules
- `state-management.md` - State patterns and tools
- `testing-patterns.md` - Testing with React Testing Library

## Workflows

See `workflows/` for step-by-step guidance on:
- `create-component.md` - Creating new components
- `add-api-integration.md` - Connecting to APIs
- `add-form.md` - Building forms with validation
- `optimize-performance.md` - Performance optimization
