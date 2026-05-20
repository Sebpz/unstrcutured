# Reference 09 — CI Automation

Generate these two files as-is, then substitute the runtime assembly name and
networking framework detected in Phase 0 of `/unity-build-tests`.

---

## 9.1 GitHub Actions Workflow

**File:** `.github/workflows/unity-tests.yml`

```yaml
name: Unity Tests

on:
  push:
    branches: [main, develop, "feature/**"]
  pull_request:
    branches: [main, develop]

concurrency:
  group: unity-tests-${{ github.ref }}
  cancel-in-progress: true

jobs:
  editmode-tests:
    name: EditMode Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Cache Unity Library
        uses: actions/cache@v4
        with:
          path: Library
          key: Library-EditMode-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-

      - name: Run EditMode Tests
        uses: game-ci/unity-test-runner@v4
        id: editmode
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          projectPath: .
          testMode: editmode
          artifactsPath: TestResults/EditMode
          coverageOptions: >
            generateAdditionalMetrics;
            generateHtmlReport;
            generateBadgeReport;
            assemblyFilters:+<RUNTIME_ASMDEF_NAME>

      - name: Upload EditMode Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: editmode-results
          path: ${{ steps.editmode.outputs.artifactsPath }}

  playmode-tests:
    name: PlayMode Tests
    runs-on: ubuntu-latest
    needs: editmode-tests      # only run if EditMode passes
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Cache Unity Library
        uses: actions/cache@v4
        with:
          path: Library
          key: Library-PlayMode-${{ hashFiles('Assets/**', 'Packages/**', 'ProjectSettings/**') }}
          restore-keys: Library-

      - name: Run PlayMode Tests
        uses: game-ci/unity-test-runner@v4
        id: playmode
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          projectPath: .
          testMode: playmode
          artifactsPath: TestResults/PlayMode

      - name: Upload PlayMode Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playmode-results
          path: ${{ steps.playmode.outputs.artifactsPath }}

  coverage-report:
    name: Coverage Gate
    runs-on: ubuntu-latest
    needs: [editmode-tests, playmode-tests]
    if: github.event_name == 'pull_request'
    steps:
      - name: Download EditMode Results
        uses: actions/download-artifact@v4
        with:
          name: editmode-results
          path: TestResults/EditMode

      - name: Check Coverage Threshold
        run: |
          # Extract line-rate from OpenCover XML using python3 (portable; avoids
          # grep -P which is GNU-only and unavailable on macOS BSD grep)
          python3 - <<'EOF'
import sys, re
xml = open("TestResults/EditMode/coverage.xml").read()
m = re.search(r'line-rate="([^"]+)"', xml)
if not m:
    print("ERROR: line-rate not found in coverage.xml"); sys.exit(1)
rate = float(m.group(1))
print(f"Line coverage: {rate:.1%}")
sys.exit(0 if rate >= 0.75 else 1)
EOF
```

> **Required secrets** in GitHub repo settings:
> - `UNITY_LICENSE` — contents of `.ulf` licence file
> - `UNITY_EMAIL` — Unity account email
> - `UNITY_PASSWORD` — Unity account password
>
> See https://game.ci/docs/github/activation for licence activation.

---

## 9.2 Local Headless Test Runner

**File:** `scripts/run-tests-local.sh`

```bash
#!/usr/bin/env bash
# Run Unity tests headlessly. Set UNITY_PATH before calling.
# Usage: ./scripts/run-tests-local.sh [editmode|playmode|all]
set -euo pipefail

UNITY_PATH="${UNITY_PATH:-}"
MODE="${1:-all}"
PROJECT_PATH="$(cd "$(dirname "$0")/.." && pwd)"
RESULTS_DIR="$PROJECT_PATH/TestResults"

if [[ -z "$UNITY_PATH" ]]; then
  # Common install locations
  for candidate in \
    "/Applications/Unity/Hub/Editor/$(ls /Applications/Unity/Hub/Editor/ 2>/dev/null | sort -t. -k1,1n -k2,2n -k3,3n | tail -1)/Unity.app/Contents/MacOS/Unity" \
    "/opt/unity/Editor/Unity" \
    "C:/Program Files/Unity/Hub/Editor/*/Editor/Unity.exe"
  do
    if [[ -f "$candidate" ]]; then
      UNITY_PATH="$candidate"
      break
    fi
  done
fi

if [[ -z "$UNITY_PATH" || ! -f "$UNITY_PATH" ]]; then
  echo "ERROR: Unity executable not found. Set UNITY_PATH env var."
  exit 1
fi

mkdir -p "$RESULTS_DIR"

run_mode() {
  local mode="$1"
  echo "--- Running $mode tests ---"
  "$UNITY_PATH" \
    -batchmode \
    -nographics \
    -quit \
    -projectPath "$PROJECT_PATH" \
    -runTests \
    -testPlatform "$mode" \
    -testResults "$RESULTS_DIR/${mode}-results.xml" \
    -logFile "$RESULTS_DIR/${mode}.log" \
    -enableCodeCoverage \
    -coverageResultsPath "$RESULTS_DIR/Coverage/${mode}"
  echo "Results: $RESULTS_DIR/${mode}-results.xml"
}

case "$MODE" in
  editmode) run_mode editmode ;;
  playmode) run_mode playmode ;;
  all)
    run_mode editmode
    run_mode playmode
    ;;
  *)
    echo "Usage: $0 [editmode|playmode|all]"
    exit 1
    ;;
esac

echo "Done. Test results in $RESULTS_DIR"
```

Make it executable: `chmod +x scripts/run-tests-local.sh`

---

## 9.3 Server Build Test Job (Optional)

If the project has a dedicated server build target, add this job to the workflow:

```yaml
  server-build:
    name: Dedicated Server Build
    runs-on: ubuntu-latest
    needs: editmode-tests
    steps:
      - uses: actions/checkout@v4
        with:
          lfs: true
      - uses: game-ci/unity-builder@v4
        env:
          UNITY_LICENSE: ${{ secrets.UNITY_LICENSE }}
          UNITY_EMAIL: ${{ secrets.UNITY_EMAIL }}
          UNITY_PASSWORD: ${{ secrets.UNITY_PASSWORD }}
        with:
          targetPlatform: StandaloneLinux64
          buildName: DedicatedServer
          customParameters: -standaloneBuildSubtarget Server
      - uses: actions/upload-artifact@v4
        with:
          name: server-build
          path: build/StandaloneLinux64
```

---

## 9.4 Test Result Badge (Optional)

In `README.md`, add the GameCI badge after generating your first run:

```markdown
[![Unity Tests](https://github.com/<ORG>/<REPO>/actions/workflows/unity-tests.yml/badge.svg)](https://github.com/<ORG>/<REPO>/actions/workflows/unity-tests.yml)
```
