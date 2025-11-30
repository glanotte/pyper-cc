---
description: Apply systematic debugging methodology to investigate an issue
argument-hint: [issue description]
allowed-tools: [Read, Grep, Glob, Bash, WebSearch]
---

<objective>
Apply expert debugging methodology to investigate: $ARGUMENTS

This uses systematic investigation with evidence gathering, hypothesis testing, and rigorous verification.
</objective>

<debugging_mindset>
- Assume nothing. Verify everything.
- Gather evidence before forming hypotheses
- Test one variable at a time
- Document what you try and what you learn
- The bug is never where you first think it is
</debugging_mindset>

<process>
<phase_1_understand>
**Understand the Problem**
1. What is the expected behavior?
2. What is the actual behavior?
3. When did this start happening? (What changed?)
4. Can you reproduce it consistently?
5. What's the minimal reproduction case?
</phase_1_understand>

<phase_2_gather>
**Gather Evidence**
1. Read error messages carefully - every word matters
2. Check logs at the time of failure
3. Inspect the state of relevant data/variables
4. Trace the code path that leads to the failure
5. Look for recent changes in git history
</phase_2_gather>

<phase_3_hypothesize>
**Form Hypotheses**
Based on evidence, generate 2-4 hypotheses:
- Hypothesis A: [specific theory] - Evidence: [what supports this]
- Hypothesis B: [specific theory] - Evidence: [what supports this]
- Hypothesis C: [specific theory] - Evidence: [what supports this]

Rank by likelihood based on evidence strength.
</phase_3_hypothesize>

<phase_4_test>
**Test Hypotheses**
For each hypothesis (starting with most likely):
1. Design a test that would prove or disprove it
2. Predict the outcome if hypothesis is true
3. Run the test
4. Compare actual vs. predicted outcome
5. Update understanding based on results
</phase_4_test>

<phase_5_fix>
**Fix and Verify**
1. Implement the minimal fix
2. Verify the fix resolves the original issue
3. Check for regressions - did the fix break anything else?
4. Understand WHY the fix works (not just that it works)
</phase_5_fix>
</process>

<output_format>
**Issue:** [Clear statement of the problem]

**Evidence Gathered:**
- [Finding 1]
- [Finding 2]

**Hypotheses:**
1. [Most likely] - because [evidence]
2. [Less likely] - because [evidence]

**Tests Performed:**
- Test 1: [what was tested] → [result]
- Test 2: [what was tested] → [result]

**Root Cause:** [What was actually wrong]

**Fix:** [What was done to resolve it]

**Verification:** [How we confirmed it's fixed]
</output_format>

<success_criteria>
- Root cause identified (not just symptoms addressed)
- Fix is minimal and targeted
- Verification confirms the issue is resolved
- Understanding is gained (could explain why it failed)
</success_criteria>
