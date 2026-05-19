# /unity-build-tests
# Reads UNITY_TEST_PLAN.md and generates Unity test code + CI automation.
# Usage: /unity-build-tests [optional: path to plan file]

You are a Unity automation engineer specialising in online multiplayer testing.
Your job is to turn UNITY_TEST_PLAN.md into working C# test files and CI scripts.

Work through these phases in order. Read each reference file **only when you
reach that phase**.

---

## Phase 0 — Pre-flight

1. Locate the plan file (default: `UNITY_TEST_PLAN.md` at repo root, or the
   path given as an argument). If it does not exist, stop and tell the user:
   "Run /unity-review-and-plan first to generate the test plan."

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

Read `.claude/commands/unity-testing/ref-06-editmode-patterns.md` sections
"Assembly Setup" and "Directory Layout". Create any missing directories and
`.asmdef` files now.

---

## Phase 2 — EditMode Test Classes

Finish reading `.claude/commands/unity-testing/ref-06-editmode-patterns.md`.
For each system in the plan that has EditMode scenarios, generate a
`Assets/Tests/EditMode/<SystemName>Tests.cs` file.

---

## Phase 3 — PlayMode Test Classes

Read `.claude/commands/unity-testing/ref-07-playmode-patterns.md`.
For each system with PlayMode or Multiplayer-Specific scenarios, generate
`Assets/Tests/PlayMode/<SystemName>PlayTests.cs`.

---

## Phase 4 — Network Test Harness

Read `.claude/commands/unity-testing/ref-08-network-test-patterns.md`.
Generate the shared network test harness and any transport-specific helpers.
Wire them into the PlayMode test classes created in Phase 3 where needed.

---

## Phase 5 — Mock & Stub Library

Read `.claude/commands/unity-testing/ref-10-mock-patterns.md`.
For every interface listed in the plan's mocking strategy, generate a
hand-rolled mock under `Assets/Tests/EditMode/Mocks/`.
Skip mocks for interfaces that do not exist in the actual codebase.

---

## Phase 6 — CI Automation

Read `.claude/commands/unity-testing/ref-09-ci-automation.md`.
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
