# Debug Like Expert

A systematic debugging skill for investigating complex issues with evidence gathering, hypothesis testing, and rigorous verification.

## When to Load

Load this skill when:
- Standard troubleshooting has failed
- The bug is elusive or intermittent
- You need systematic root cause analysis
- Multiple hypotheses need testing

## Debugging Mindset

1. **Assume nothing** - Verify every assumption
2. **Evidence first** - Gather data before forming hypotheses
3. **One variable at a time** - Isolate changes
4. **Document everything** - What you tried, what you learned
5. **The bug is never where you first think** - Stay open

## Investigation Protocol

### Phase 1: Understand

Before touching code, answer:
1. What is the expected behavior?
2. What is the actual behavior?
3. When did this start? What changed?
4. Can you reproduce it consistently?
5. What's the minimal reproduction case?

### Phase 2: Gather Evidence

1. Read error messages carefully - every word
2. Check logs at time of failure
3. Inspect relevant state/data
4. Trace the code path
5. Check git history for recent changes

### Phase 3: Hypothesize

Form 2-4 specific hypotheses:
- Based on evidence, not guessing
- Each must be testable
- Ranked by likelihood

### Phase 4: Test

For each hypothesis:
1. Design a test that proves/disproves it
2. Predict the outcome if hypothesis is true
3. Run the test
4. Compare actual vs. predicted
5. Update understanding

### Phase 5: Fix and Verify

1. Implement minimal fix
2. Verify it resolves the issue
3. Check for regressions
4. Understand WHY the fix works

## Common Debugging Techniques

### Binary Search
When you don't know where the bug is:
- Comment out half the code
- Does bug persist? Bug is in remaining half
- Repeat until isolated

### Print Debugging
Strategic logging to trace execution:
```ruby
Rails.logger.debug ">>> Entering method with: #{params.inspect}"
Rails.logger.debug ">>> State after operation: #{@object.inspect}"
```

### Git Bisect
Find the commit that introduced the bug:
```bash
git bisect start
git bisect bad  # current commit is bad
git bisect good <known-good-commit>
# Git will checkout commits, you test and mark good/bad
```

### Rubber Duck Debugging
Explain the problem out loud, step by step.
Often reveals assumptions you didn't know you had.

## References

See `references/` for specific techniques:
- `debugging-mindset.md` - Mental approach
- `hypothesis-testing.md` - Scientific method for bugs
- `investigation-techniques.md` - Specific tools and methods
- `verification-patterns.md` - Confirming fixes work
