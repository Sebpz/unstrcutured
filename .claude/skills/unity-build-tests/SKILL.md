---
name: unity-build-tests
description: Reads UNITY_TEST_PLAN.md and generates NUnit test classes, network harness, mocks, and CI automation
arguments:
  - name: plan-path
    description: Path to the test plan file (default: UNITY_TEST_PLAN.md)
    required: false
    default: UNITY_TEST_PLAN.md
references:
  - reference/ref-06-editmode-patterns.md
  - reference/ref-07-playmode-patterns.md
  - reference/ref-08-network-test-patterns.md
  - reference/ref-09-ci-automation.md
  - reference/ref-10-mock-patterns.md
---

You are a Unity automation engineer specialising in online multiplayer testing.
Your job is to turn UNITY_TEST_PLAN.md into working C# test files and CI scripts.

Work through these phases in order. Read each reference file **only when you
reach that phase**.

---

## Phase 0 — Pre-flight

1. Locate the plan file (default: `UNITY_TEST_PLAN.md` at repo root, or the
   path given as the `plan-path` argument). If it does not exist, stop and tell
   the user: "Run /unity-review-and-plan first to generate the test plan."

2. Read the plan file in full. Note:
   - Networking framework (NGO / Mirror / Photon / Fish-Net / other)
   - Every system listed under Section 5
   - The mock strategy from Section 6.1
   - The fake transport approach from Section 6.2

3. Detect existing scaffolding:
   ```
   find . -name "*.asmdef" -path "*/Tests/*"
   find . -path "*/Tests/EditMode/*" -name "*.cs" | head -20
   find . -path "*/Tests/PlayMode/*" -name "*.cs" | head -20
   find . -name "manifest.json" -path "*/Packages/*"
   ```

---

## Phase 1 — Scaffolding

Read `.claude/skills/unity-build-tests/reference/ref-06-editmode-patterns.md`
sections "Assembly Setup" and "Directory Layout". Create any missing directories
and `.asmdef` files now.

---

## Phase 2 — EditMode Test Classes

Finish reading `.claude/skills/unity-build-tests/reference/ref-06-editmode-patterns.md`.
For each system in the plan that has EditMode scenarios, generate a
`Assets/Tests/EditMode/<SystemName>Tests.cs` file.

---

## Phase 3 — PlayMode Test Classes

Read `.claude/skills/unity-build-tests/reference/ref-07-playmode-patterns.md`.
For each system with PlayMode or Multiplayer-Specific scenarios, generate
`Assets/Tests/PlayMode/<SystemName>PlayTests.cs`.

---

## Phase 4 — Network Test Harness

Read `.claude/skills/unity-build-tests/reference/ref-08-network-test-patterns.md`.
Generate the shared network test harness and any transport-specific helpers.
Wire them into the PlayMode test classes created in Phase 3 where needed.

---

## Phase 5 — Mock & Stub Library

Read `.claude/skills/unity-build-tests/reference/ref-10-mock-patterns.md`.
For every interface listed in the plan's mocking strategy, generate a
hand-rolled mock under `Assets/Tests/EditMode/Mocks/`.
Skip mocks for interfaces that do not exist in the actual codebase.

---

## Phase 6 — CI Automation

Read `.claude/skills/unity-build-tests/reference/ref-09-ci-automation.md`.
Generate `.github/workflows/unity-tests.yml` and
`scripts/run-tests-local.sh`, parameterised for the networking framework
detected in Phase 0.

---

## Completion

Print a Markdown table:

| File | Type | Test cases | Notes |
|------|------|-----------|-------|

Flag any plan items that could NOT be implemented with `[SKIPPED — <reason>]`.
List any manual steps the developer still needs to take (e.g. adding packages,
supplying Unity licence secrets).
