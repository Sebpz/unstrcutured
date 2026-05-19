# /unity-test-pipeline
#
# Iterative planning + build pipeline for online multiplayer Unity projects.
#
# Usage: /unity-test-pipeline [max-plan-passes] [max-build-passes]
#   Defaults: 3 plan passes, 3 build passes
#   Example:  /unity-test-pipeline 4 2
#
# Arguments are the first two space-separated integers in $ARGUMENTS.
# If omitted, use 3 for each.

Parse $ARGUMENTS: set MAX_PLAN_PASSES and MAX_BUILD_PASSES from the first two
integers found. Default both to 3 if absent.

---

## PHASE A — Planning Loop

### A.0  Initialise tracking

Read `.claude/commands/unity-testing/ref-11-coverage-evaluation.md` now and
keep it in context for the rest of this skill.

Create `UNITY_TEST_STATUS.md` at the repo root with this skeleton:

```markdown
# Unity Test Pipeline Status

## Config
- Max plan passes: <MAX_PLAN_PASSES>
- Max build passes: <MAX_BUILD_PASSES>
- Plan coverage threshold: 85%
- Build coverage threshold: 85%

## Planning Phase
<!-- populated by the pipeline -->

## Building Phase
<!-- populated by the pipeline -->
```

Set PLAN_PASS = 0.

### A.1  Planning pass loop

Repeat until PLAN_PASS >= MAX_PLAN_PASSES OR last recorded plan score >= 85%:

  **Increment:** PLAN_PASS += 1

  **Execute the pass:**
  - If PLAN_PASS == 1:
    Follow every phase in `.claude/commands/unity-review-and-plan.md` in full.
    This produces or overwrites `UNITY_TEST_PLAN.md`.
  - If PLAN_PASS >= 2:
    Read `UNITY_TEST_PLAN.md`. Identify all gaps using the gap checklist in
    ref-11 Section 11.2. For each gap, fill it in directly in the plan file
    using the reference files for guidance:
    - Missing system scenarios → ref-03 + ref-04
    - Incomplete strategy sections → ref-05
    - Template placeholder text → replace with real class names from codebase
    Do NOT re-run discovery or re-classify files already classified.

  **Score the plan:** Apply the Plan Coverage Rubric from ref-11 Section 11.1.
  Record the score as PLAN_SCORE_<N>.

  **Append to UNITY_TEST_STATUS.md** under "## Planning Phase":
  ```markdown
  ### Pass <N> — Plan
  - Score: <PLAN_SCORE_N>%
  - Systems with complete scenarios: <x>/<total>
  - Gaps found: <list>
  - Gaps filled this pass: <list>
  - Decision: <CONTINUE | STOP — score met | STOP — max passes reached>
  ```

  **Decide:** if score >= 85% or PLAN_PASS >= MAX_PLAN_PASSES → exit loop.

### A.2  Plan review and fix pass

Read `UNITY_TEST_PLAN.md` from top to bottom. Check every item in the Plan
Quality Checklist from ref-11 Section 11.3. Apply fixes directly in the file.

Append to UNITY_TEST_STATUS.md:
```markdown
### Plan Review Pass
- Issues found: <list>
- Fixes applied: <list>
- Final plan score: <score>%
```

---

## PHASE B — Build Loop

Set BUILD_PASS = 0.

### B.1  Build pass loop

Repeat until BUILD_PASS >= MAX_BUILD_PASSES OR last recorded build score >= 85%:

  **Increment:** BUILD_PASS += 1

  **Execute the pass:**
  - If BUILD_PASS == 1:
    Follow every phase in `.claude/commands/unity-build-tests.md` in full.
  - If BUILD_PASS >= 2:
    Read `UNITY_TEST_PLAN.md`. Apply the Build Coverage Rubric from ref-11
    Section 11.4 to find untranslated plan scenarios. For each gap:
    - Missing test method → add it to the appropriate existing `.cs` file
    - Missing mock file → generate it (ref-10)
    - Missing CI file → generate it (ref-09)
    Do NOT regenerate files that already pass the rubric for their system.

  **Score the build:** Apply the Build Coverage Rubric from ref-11 Section 11.4.
  Record as BUILD_SCORE_<N>.

  **Append to UNITY_TEST_STATUS.md** under "## Building Phase":
  ```markdown
  ### Pass <N> — Build
  - Score: <BUILD_SCORE_N>%
  - Plan scenarios translated: <x>/<total>
  - Test files touched: <list>
  - Gaps filled this pass: <list>
  - Decision: <CONTINUE | STOP — score met | STOP — max passes reached>
  ```

  **Decide:** if score >= 85% or BUILD_PASS >= MAX_BUILD_PASSES → exit loop.

### B.2  Final script review and fix pass

For every `.cs` file under `Assets/Tests/`:
1. Verify each `[Test]` / `[UnityTest]` method has a real assertion (not just
   `Assert.Pass()` or `// TODO`).
2. Verify `[SetUp]` / `[TearDown]` / `[UnitySetUp]` / `[UnityTearDown]` clean
   up all GameObjects and NetworkManagers they create.
3. Verify test class names and file names match the system in the plan.
4. Verify no test reads from `Resources.Load` for test data when a
   `ScriptableObject.CreateInstance<T>()` approach is feasible.
5. Apply fixes inline.

Append to UNITY_TEST_STATUS.md:
```markdown
### Final Script Review
- Files reviewed: <list>
- Issues found: <list>
- Fixes applied: <list>
- Final build score: <score>%
```

---

## Completion

Print a final summary:
- Total plan passes run and final plan score
- Total build passes run and final build score
- Total test methods generated (EditMode / PlayMode / Performance)
- Any items that remained below threshold after max passes, with the reason
- Next recommended manual steps (e.g. "Add Unity licence secrets to GitHub")
