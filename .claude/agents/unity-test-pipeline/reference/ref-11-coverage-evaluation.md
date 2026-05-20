# Reference 11 — Coverage Evaluation & Gap Analysis

Used by `/unity-test-pipeline` to score each pass and decide when to stop.

---

## 11.1  Plan Coverage Rubric

Score the plan out of 100 using the weighted checklist below.
Tally the points earned, divide by the total possible, multiply by 100.

### A — System Completeness  (max 40 pts)

For each system listed in Section 4 (System Map), check that Section 5
contains a matching subsection. Award points per system:

| Criterion | Points per system |
|---|---|
| Section 5.x exists for this system | 1 |
| Has ≥ 2 EditMode scenarios with real class names (no `<ClassName>`) | 2 |
| Has ≥ 1 PlayMode or Multiplayer-Specific scenario (if any NetworkBehaviour in system) | 2 |
| No scenario contains placeholder text like `[TODO]`, `<method>`, or `...` | 1 |

Scale the raw score to 40 pts: `(earned / max_possible) * 40`.

### B — Multiplayer Coverage  (max 25 pts)

| Criterion | Points |
|---|---|
| Connection lifecycle scenarios present (connect, disconnect, timeout) | 4 |
| Server-authority / anti-cheat scenarios present | 5 |
| NetworkVariable sync scenario present | 4 |
| RPC routing scenario (ServerRpc rejected from non-owner) | 4 |
| Late-joiner state scenario present | 4 |
| Host migration scenario present (or explicitly noted as not applicable) | 4 |

### C — Strategy Completeness  (max 20 pts)

| Criterion | Points |
|---|---|
| Section 6.1 names ≥ 1 specific interface to mock (not generic placeholder) | 4 |
| Section 6.2 names the detected transport and describes the test setup | 4 |
| Section 6.4 has coverage targets for ≥ 3 system tiers | 4 |
| Section 7 (Risk Register) has ≥ 1 entry per Critical system | 4 |
| Section 8 (Priority Order) is non-empty | 4 |

### D — Quality  (max 15 pts)

| Criterion | Points |
|---|---|
| No section header is empty or contains only a template subtitle | 3 |
| Class references in scenarios match actual file paths found in Phase 1 | 6 |
| Scenario IDs follow the `<System>-EM-<n>` / `PM` / `MP` / `PERF` convention | 3 |
| Performance scenarios present (or explicitly noted as N/A — no perf package) | 3 |

**Plan Score = (Total earned / Total possible) × 100**

---

## 11.2  Gap Checklist (Planning Passes 2+)

Run this after reading `UNITY_TEST_PLAN.md`. Each `[ ]` that remains unfilled
is a gap to address in the current pass.

### Systems
- [ ] Every row in the Section 4 System Map has a Section 5.x entry
- [ ] No Section 5.x heading contains only template lines (e.g. "TBD")
- [ ] Every Section 5.x with a NetworkBehaviour class has a Multiplayer-Specific subsection
- [ ] Systems rated Critical have ≥ 3 distinct EditMode scenarios

### Scenarios
- [ ] No scenario `Expected:` field says "something happens" or "as expected"
- [ ] Every scenario `Class under test:` names a file path that exists
- [ ] Every `[ServerRpc]` method in the codebase appears in at least one scenario
- [ ] Every `NetworkVariable<T>` field appears in at least one sync scenario

### Strategy
- [ ] Section 6.1 has one interface entry per Unity Gaming Service actually used
- [ ] Section 6.2 matches the networking framework detected (not a generic placeholder)
- [ ] Section 7 has an entry for every system rated Critical with no test scenarios yet
- [ ] Section 8 lists systems in highest-priority-first order (not alphabetical)

### Completeness signals
When re-reading the plan, flag any line that contains:
- `<ClassName>`, `<MethodName>`, `<system>`, `<n>`, `...`, `TBD`, `TODO`, `[placeholder]`

Each flagged line is a gap. Replace with real content from the codebase.

---

## 11.3  Plan Quality Checklist (Review Pass)

Apply after the planning loop exits. These are harder to detect automatically
but strongly affect plan usefulness.

- [ ] Scenario descriptions are specific enough that a developer who didn't
      write the code could implement the test without reading the source.
- [ ] Every mock interface in Section 6.1 is an interface that actually exists
      OR is recommended for extraction (with the source class noted).
- [ ] No two scenarios under the same system test the exact same condition.
- [ ] The Priority Order (Section 8) is consistent with the Criticality ratings
      in the System Map (Section 4).
- [ ] The Risk Register (Section 7) includes at least one entry covering a
      multiplayer-specific risk (e.g. desync, RPC flooding, late-join corruption).
- [ ] Each scenario's "Why it matters" field connects to a multiplayer failure mode,
      not just a generic description.

For each failed check: edit `UNITY_TEST_PLAN.md` directly to fix it.

---

## 11.4  Build Coverage Rubric

Score the generated test code out of 100.

### A — Plan Translation  (max 50 pts)

For every scenario in UNITY_TEST_PLAN.md, check whether a corresponding
test method exists in `Assets/Tests/`:

```bash
# Count total scenarios
grep -c "Scenario ID:" UNITY_TEST_PLAN.md

# Count test methods
grep -Erc "\[Test\]|\[UnityTest\]" Assets/Tests/ --include="*.cs"
```

- Award 1 pt per EditMode scenario that has a `[Test]` method with a real assertion
- Award 1.5 pts per PlayMode / MP scenario that has a `[UnityTest]` with yield + assert
- Award 0.5 pts per Performance scenario that has `[Test, Performance]`

Scale to 50 pts: `(earned / max_possible) * 50`.

### B — Infrastructure  (max 20 pts)

| Criterion | Points |
|---|---|
| `Assets/Tests/EditMode/Tests.EditMode.asmdef` exists and references runtime asmdef | 5 |
| `Assets/Tests/PlayMode/Tests.PlayMode.asmdef` exists with `TestAssemblies` reference | 5 |
| `.github/workflows/unity-tests.yml` exists with editmode + playmode jobs | 5 |
| `scripts/run-tests-local.sh` exists and is executable | 5 |

### C — Mock Coverage  (max 15 pts)

For every interface listed in Section 6.1 of the plan:

| Criterion | Points per interface |
|---|---|
| Mock file exists at `Assets/Tests/EditMode/Mocks/Mock<Name>.cs` | 2 |
| Mock tracks call count | 1 |
| Mock supports configurable return value AND configurable exception | 2 |

Scale to 15 pts.

### D — Test Quality  (max 15 pts)

Scan all generated `.cs` test files:

| Criterion | Points |
|---|---|
| No `[Test]` method body contains only `Assert.Pass()` or is empty | 5 |
| No `// TODO` comment exists inside a test method body | 4 |
| Every `[UnitySetUp]` that creates a `NetworkManager` has a matching `[UnityTearDown]` that calls `Shutdown()` | 3 |
| No test creates `new GameObject()` without a matching `Object.DestroyImmediate()` in `[TearDown]` | 3 |

**Build Score = (Total earned / Total possible) × 100**

---

## 11.5  Build Gap Checklist (Build Passes 2+)

Run this after reading the plan and scanning `Assets/Tests/`:

### Missing test methods
- For each `Scenario ID` in the plan, grep for it as a comment or for the
  scenario's class + method in the test files.
  If absent → add the test method.

```bash
# Find scenario IDs in plan
grep "Scenario ID:" UNITY_TEST_PLAN.md

# Check which appear in test code (as comments or method names)
grep -r "Combat-EM-1\|Spawn-PM-2\|..." Assets/Tests/ --include="*.cs"
```

### Missing mocks
- For each interface in plan Section 6.1, check:
  `find Assets/Tests/EditMode/Mocks -name "Mock*.cs"`
  If the interface has no matching file → generate it from ref-10 templates.

### Incomplete assertions
Scan for these anti-patterns and fix each one:
- `Assert.IsTrue(true)` — meaningless assertion
- `Assert.Pass()` — no assertion at all
- `yield return null; // TODO` — incomplete PlayMode test
- Empty `[Test]` method body

### NetworkManager teardown gaps
```bash
grep -n "NetworkManager" Assets/Tests/PlayMode/ -r --include="*.cs" |
  grep -v "Shutdown\|TearDown"
```
Any NetworkManager reference not inside a TearDown or Shutdown call is a leak.
Add the cleanup.

---

## 11.6  Final Script Review Checklist

Applied once after the build loop exits. Fix every issue found.

- [ ] All test class names end in `Tests` or `PlayTests`
- [ ] All test method names describe what they test: `<Subject>_<Condition>_<Expected>`
- [ ] No test file imports a namespace that doesn't exist in the asmdef references
- [ ] No test uses `Thread.Sleep` — replace with `yield return new WaitForSeconds`
      or `yield return new WaitUntil(...)`
- [ ] Performance tests use `Measure.Method`, not `Stopwatch` + custom assertion
- [ ] Every mock file is in namespace `Tests.EditMode.Mocks`
- [ ] `Tests.EditMode.asmdef` and `Tests.PlayMode.asmdef` reference the correct
      runtime assembly name (not a placeholder string)
- [ ] The CI workflow `RUNTIME_ASMDEF_NAME` placeholder has been replaced with
      the actual assembly name
