# Harness Design Patterns

## Reusable Patterns for Orchestration

These patterns solve common harness orchestration problems.

### Pattern 1: Retry with Exponential Backoff

**Problem**: Transient failures (network, rate limits) should retry automatically

**Solution**:
```
attempt = 1
max_attempts = 5
delay = 1  # seconds

while attempt <= max_attempts:
    try:
        task()
        return success
    except TransientError:
        if attempt < max_attempts:
            wait(delay)
            delay *= 2  # exponential backoff
            attempt += 1
        else:
            raise
```

### Pattern 2: Request Batching

**Problem**: Multiple Claude API calls are expensive; batch them

**Solution**:
- Collect all related generation tasks
- Send in single API call with multiple prompts
- Parse responses, validate individually
- Cache batch result by input hash

**Example**: Generate code + tests + Terraform in one API call

### Pattern 3: Validation Loop

**Problem**: Generated code may not compile, validate, or pass security checks

**Solution**:
```
generated = claude.generate(spec)
errors = validate(generated)

while errors:
    generated = claude.fix(generated, errors)
    errors = validate(generated)
    attempt_count += 1
    if attempt_count > 3:
        raise ValidationError("Could not fix after 3 attempts")

return generated
```

### Pattern 4: Checkpointing

**Problem**: Long harness runs may fail mid-way; enable recovery without recomputing

**Solution**:
- Save state at each major stage (after analysis, after generation, after validation)
- Store state in database or file
- On restart, detect last checkpoint and resume from there

### Pattern 5: Prompt Engineering by Stage

**Problem**: Different stages need different Claude behaviors (code = deterministic, analysis = creative)

**Solution**:
- **Code generation**: temperature=0, max_tokens=4000, format-specific prompts
- **Analysis**: temperature=0.5, max_tokens=2000, open-ended prompts
- **Validation**: temperature=0, max_tokens=500, check-specific prompts

### Pattern 6: Error Context Propagation

**Problem**: Generic "generation failed" is not helpful; user needs to know why

**Solution**:
```
error = {
  stage: "code_generation",
  task: "generate_rest_api",
  error_type: "compilation_error",
  context: "TypeScript error on line 42: ...",
  attempt: 2,
  suggestion: "Review the function signature"
}
```

### Pattern 7: Cache Invalidation

**Problem**: When to use cached results vs. regenerate?

**Solution**:
- **Cache key**: hash(PRD content, architecture, language)
- **Invalidate on**: PRD change, explicit user request, cache TTL expiry
- **Keep** generation metadata: timestamp, validation status, Claude model used

### Pattern 8: Plugin Interface for Deployment

**Problem**: Support multiple deployment targets without coupling harness to each

**Solution**:
```
interface DeploymentPlugin {
    validate_config()
    provision_infrastructure()
    deploy_application()
    verify_deployment()
}

plugins = {
    'aws': AWSPlugin(),
    'terraform': TerraformPlugin(),
    'docker': DockerPlugin(),
}

harness.deploy(target='aws', config=config)
```

## To Be Developed

- Concurrency patterns (parallel generation, parallel validation)
- State machine patterns (workflow state transitions)
- User feedback loops (ask for clarification, offer alternatives)
- Cost optimization patterns (batch similar generations, cache aggressively)

