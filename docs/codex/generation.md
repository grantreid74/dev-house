# Claude Codex Integration

## Purpose

Generate production-ready code from PRD specifications using Claude's code generation capabilities.

## Responsibilities

- Convert structural requirements into compilable code
- Generate code in multiple languages (Node.js, Python, Go, etc.)
- Handle common patterns (REST APIs, databases, authentication)
- Validate generated code for correctness

## Supported Code Types

### Application Code
- REST APIs (Express, FastAPI, etc.)
- Batch processors (Node scripts, Python jobs)
- Web applications (React, Vue, etc.)

### Infrastructure Code
- Terraform modules
- Docker configurations
- Kubernetes manifests
- CloudFormation templates

### Test Code
- Unit tests
- Integration tests
- E2E test suites

## Prompt Patterns

*To be developed through exploration.*

### Example: REST API Generation

```
You are a senior developer. Generate a production-ready {language} REST API based on this specification:

[PRD section]

Requirements:
1. Follow {framework} conventions
2. Include error handling
3. Add input validation
4. Use environment variables for configuration
5. Include health check endpoint

Output format: Just the code, wrapped in code blocks.
```

## Code Quality Standards

- Linting passes (eslint, pylint, etc.)
- Type checking passes (TypeScript, mypy, etc.)
- Basic security checks (OWASP top 10)
- Test coverage > 70%

## Validation Pipeline

1. **Syntax validation** — Code compiles/parses
2. **Linting** — Style and convention checks
3. **Type checking** — Static type validation
4. **Security scanning** — Basic security patterns
5. **Execution** — Run tests, smoke tests

## Error Patterns

- Missing dependencies → Ask Claude to add imports
- Type errors → Ask Claude to fix types
- Test failures → Ask Claude to fix logic
- Security issues → Ask Claude to refactor

## Caching

- Cache generated code by (PRD + architecture + language)
- Invalidate on PRD change
- Keep cache metadata (generated timestamp, validation status)

## Next: Details

- Language-specific patterns (Node.js, Python, Go)
- Template system for common services
- Security scanning integration
- Performance optimization patterns

