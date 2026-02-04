# CLAUDE.md

## Development Principles

### TDD (Test-Driven Development)
- Write tests before writing implementation code
- Follow the Red-Green-Refactor cycle: write a failing test, make it pass, then refactor
- Every new feature or bug fix must start with a test that demonstrates the expected behavior
- Run the full test suite before considering any task complete

### DRY (Don't Repeat Yourself)
- Extract shared logic into reusable functions or modules when the same code appears in more than two places
- Prefer single sources of truth for configuration, constants, and business logic
- When modifying duplicated code, refactor it into a shared abstraction first

### YAGNI (You Aren't Gonna Need It)
- Only implement what is explicitly requested — no speculative features
- Do not add abstractions, configuration options, or extensibility points for hypothetical future use
- Prefer the simplest solution that satisfies the current requirement
- Remove dead code rather than commenting it out "just in case"
