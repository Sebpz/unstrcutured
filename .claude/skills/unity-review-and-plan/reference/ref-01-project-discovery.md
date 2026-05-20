# Reference 01 — Project Discovery

Run the commands below and record every result. Later phases depend on this data.

---

## 1.1 Directory Layout

```bash
# Top-level structure
ls -1

# Scripts tree (first 100 files)
find . -name "*.cs" \
  -not -path "*/.git/*" \
  -not -path "*/Library/*" \
  -not -path "*/Temp/*" \
  -not -path "*/obj/*" | sort | head -100

# Existing tests
find . -path "*/Tests/*" -name "*.cs" | sort

# Assembly definitions
find . -name "*.asmdef" -not -path "*/Library/*" | sort

# Scenes
find . -name "*.unity" -not -path "*/Library/*" | sort
```

Record:
- Does `Assets/Tests/` exist?
- How many `.cs` files total?
- Which scenes exist (note any "Bootstrap", "Lobby", "Game", "Menu" scenes)

---

## 1.2 Networking Framework Detection

```bash
# Packages manifest
cat Packages/manifest.json 2>/dev/null || echo "manifest not found"
```

Identify the networking framework from the manifest:

| Package ID | Framework |
|---|---|
| `com.unity.netcode.gameobjects` | **Unity Netcode for GameObjects (NGO)** |
| `com.vis2k.mirror` | **Mirror Networking** |
| `com.exitgames.photonunity` or Photon PUN folder | **Photon PUN 2** |
| `com.photon.fusion` | **Photon Fusion** |
| `com.firstgeargames.fishnet` | **Fish-Net** |
| `com.darkriftnetworking.*` | **DarkRift 2** |

Also check for Unity Gaming Services:

| Package ID | Service |
|---|---|
| `com.unity.services.lobby` | Unity Lobby |
| `com.unity.services.relay` | Unity Relay |
| `com.unity.services.authentication` | Unity Authentication |
| `com.unity.services.vivox` | Vivox (voice) |
| `com.unity.services.matchmaker` | Unity Matchmaker |
| `com.unity.services.multiplay` | Unity Multiplay (dedicated server) |
| `com.unity.multiplayer.tools` | Network Simulator + runtime stats (NGO) |

```bash
# Fallback: grep for framework-specific types in source
grep -rl "NetworkManager\|NetworkBehaviour\|NetworkObject" \
  --include="*.cs" Assets/ | head -20

grep -rl "PhotonNetwork\|PunRPC" \
  --include="*.cs" Assets/ | head -10

grep -rl "NetworkRunner\|INetworkRunnerCallbacks" \
  --include="*.cs" Assets/ | head -10

grep -rl "NetworkManager\b" \
  --include="*.cs" Assets/ | head -10
```

---

## 1.3 Test Framework Packages

Check for:
- `com.unity.test-framework` — required (EditMode + PlayMode)
- `com.unity.test-framework.performance` — optional performance tests
- `com.unity.testtools.codecoverage` — optional coverage reports

Note which are present. If `com.unity.test-framework` is absent, append a
manual step to the plan's Risk Register:
> "Add `com.unity.test-framework` via Package Manager before running tests."

---

## 1.4 Server vs Client Build Targets

```bash
# Dedicated server indicator
grep -rl "Application.isBatchMode\|IsServer\|IsHost\|NetworkManager.Singleton.IsServer" \
  --include="*.cs" Assets/ | head -10

# Check for Multiplay / server build symbols
grep -r "UNITY_SERVER\|DEDICATED_SERVER" \
  --include="*.cs" Assets/ | head -10
```

Note whether the project has a dedicated server build or is host-client only.
This affects PlayMode test topology in Phase 3.

---

## 1.5 Assembly Map

For each `.asmdef` found, record:
- Name
- References (other asmdefs it depends on)
- Whether it is Editor-only (`"includePlatforms": ["Editor"]`)
- Whether it already has `"optionalUnityReferences": ["TestAssemblies"]`

This determines which assembly the generated test code must reference.

---

## Output to carry forward

After running these commands, record the following facts for later phases:

```
NETWORKING_FRAMEWORK: <NGO | Mirror | Photon PUN | Photon Fusion | Fish-Net | Unknown>
HAS_DEDICATED_SERVER: <yes | no | unknown>
UNITY_SERVICES: <list>
TEST_FRAMEWORK_PRESENT: <yes | no>
PERFORMANCE_PACKAGE_PRESENT: <yes | no>
RUNTIME_ASMDEF_NAMES: <list>
SCENE_LIST: <list>
TOTAL_CS_FILES: <n>
EXISTING_TEST_COUNT: <n>
```
