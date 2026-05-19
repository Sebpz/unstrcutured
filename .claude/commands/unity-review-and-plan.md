# Unity Test Plan Generator

You are a Unity QA engineer. Your job is to audit a Unity game codebase and produce a comprehensive test plan in Markdown.

## Steps

### 1. Discover the project layout

Search for Unity-specific structure:
- `Assets/Scripts/` — game logic (MonoBehaviours, ScriptableObjects, managers, utilities)
- `Assets/Tests/` — any existing EditMode or PlayMode tests
- `Packages/manifest.json` — installed packages (check for `com.unity.test-framework`, performance testing, etc.)
- `ProjectSettings/` — player settings, physics settings, quality settings
- Assembly definition files (`*.asmdef`) — module boundaries

Run searches like:
```
find . -name "*.cs" -not -path "*/Editor/*" | head -80
find . -name "*.asmdef"
find . -name "manifest.json" -path "*/Packages/*"
find . -path "*/Tests/*" -name "*.cs"
```

### 2. Classify every C# class found

For each `.cs` file, categorise it as one of:
- **MonoBehaviour** (game object component)
- **ScriptableObject** (data container)
- **Manager / Singleton** (global service)
- **Pure C# / POCO** (no Unity dependency — most testable)
- **Editor script** (skip for runtime tests)
- **Existing test** (note coverage gaps)

### 3. Identify testable systems and their risk level

Group classes into logical systems (e.g. Combat, Inventory, Movement, AI, Audio, UI, Save/Load, Networking).  
For each system rate: **High / Medium / Low** testability and **High / Medium / Low** criticality to gameplay.

### 4. Draft test scenarios

For every system produce at minimum:

**EditMode (unit) tests** — pure logic, no Play Mode needed:
- Happy-path scenario
- Boundary / edge-case scenario
- Error / invalid-input scenario

**PlayMode (integration) tests** — require the Unity runtime:
- Scene load and system initialisation
- Cross-system interaction (e.g. player takes damage → health UI updates)
- State-machine transitions

**Performance tests** (if `com.unity.test-framework.performance` is present):
- Frame-time budget for hot paths
- Memory allocation in tight loops

### 5. Define testing strategies

Include sections on:
- **Mocking strategy** — which dependencies need fakes/stubs and how (interface extraction, `[InjectDependency]`, or manual mock classes)
- **Test data strategy** — ScriptableObject fixtures vs hard-coded values vs random seeds
- **Automation entry points** — Unity Test Runner CLI flags, `batchmode` invocation, CI trigger conditions
- **Coverage targets** — recommended minimum line/branch coverage per system tier
- **Known gaps & risks** — systems that are hard to test (e.g. tight MonoBehaviour coupling) and mitigation ideas

### 6. Write the output file

Write everything to `UNITY_TEST_PLAN.md` at the repo root using this structure:

```markdown
# Unity Test Plan

## 1. Project Overview
...

## 2. Codebase Summary
| File | Class | Category | System | Testability | Criticality |
|------|-------|----------|--------|-------------|-------------|
...

## 3. Test Scenarios

### 3.1 [System Name]
#### EditMode Tests
- **Scenario**: ...  **Input**: ...  **Expected**: ...

#### PlayMode Tests
...

#### Performance Tests
...

## 4. Testing Strategies
### 4.1 Mocking Strategy
### 4.2 Test Data Strategy
### 4.3 Automation Entry Points
### 4.4 Coverage Targets

## 5. Existing Coverage Gaps

## 6. Priority Order
1. ...
```

Be specific — reference real class names, method names, and file paths discovered during the audit. Do not invent classes that do not exist in the codebase.

After writing the file, print a one-paragraph summary of the most critical findings.
