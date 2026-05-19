# /unity-review-and-plan
# Generates UNITY_TEST_PLAN.md for an online multiplayer Unity project.
# Usage: /unity-review-and-plan [optional: subfolder path]

You are a senior Unity QA engineer specialising in online multiplayer games.
Your goal is to produce a comprehensive, multiplayer-aware test plan.

Work through these phases in order. Read each reference file **only when you
reach that phase** — do not front-load all reads.

---

## Phase 1 — Project Discovery

Read `.claude/commands/unity-testing/ref-01-project-discovery.md` and follow
every step in it. Record the outputs (framework, packages, scenes, assembly
layout) before moving on.

---

## Phase 2 — Class Classification

Read `.claude/commands/unity-testing/ref-02-class-taxonomy.md`.
Apply the taxonomy to every `.cs` file found in Phase 1.
Build the classification table described there.

---

## Phase 3 — Multiplayer System Mapping

Read `.claude/commands/unity-testing/ref-03-multiplayer-systems.md`.
Map the classes from Phase 2 onto the multiplayer system catalogue defined
there. Identify which networking framework is in use.

---

## Phase 4 — Scenario Writing

Read `.claude/commands/unity-testing/ref-04-scenario-templates.md`.
For every system identified in Phase 3, draft the full set of test scenarios
using the templates in that file (EditMode, PlayMode, Performance,
Multiplayer-specific).

---

## Phase 5 — Strategy & Gaps

Read `.claude/commands/unity-testing/ref-05-testing-strategies.md`.
Produce the strategy sections: mocking approach, test-data approach,
coverage targets, CI entry points, and known gaps.

---

## Phase 6 — Write UNITY_TEST_PLAN.md

Assemble everything into `UNITY_TEST_PLAN.md` at the repo root.

Use this exact top-level structure:

```
# Unity Multiplayer Test Plan

## 1. Project Overview
## 2. Networking Framework & Packages
## 3. Codebase Classification Table
## 4. Multiplayer System Map
## 5. Test Scenarios
   ### 5.x  <SystemName>
       #### EditMode (Unit) Tests
       #### PlayMode (Integration) Tests
       #### Multiplayer-Specific Tests
       #### Performance Tests          (if applicable)
## 6. Testing Strategies
   ### 6.1 Mocking & Dependency Isolation
   ### 6.2 Fake Network Transport Setup
   ### 6.3 Test Data & Fixtures
   ### 6.4 Coverage Targets
   ### 6.5 CI Entry Points
## 7. Risk Register & Known Gaps
## 8. Priority Order
```

Reference only class names, method names, and file paths that were actually
found in the codebase. Do not invent anything.

---

## Completion

Print a two-paragraph summary: (1) the networking framework detected and the
highest-risk systems found; (2) the three most urgent test gaps.
