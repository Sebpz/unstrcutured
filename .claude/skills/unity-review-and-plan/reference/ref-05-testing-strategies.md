# Reference 05 — Testing Strategies

Produce Section 6 of the plan using the guidance below.
Tailor each subsection to the actual framework and services detected in Phase 1.

---

## 5.1 Mocking & Dependency Isolation (Plan Section 6.1)

### Principle
Extract dependencies behind interfaces so tests can substitute fakes.
For Unity Gaming Services, wrap the SDK singletons:

```csharp
// Wrap this:  LobbyService.Instance.CreateLobbyAsync(...)
// With this:
public interface ILobbyService
{
    Task<Lobby> CreateLobbyAsync(string name, int maxPlayers, CreateLobbyOptions opts);
    Task<Lobby> QuickJoinLobbyAsync(QuickJoinLobbyOptions opts);
    Task DeleteLobbyAsync(string lobbyId);
    // ... other methods actually used in the codebase
}
```

Repeat for: `IRelayService`, `IAuthenticationService`, `IMatchmakerService`.

### Unity Netcode for GameObjects
- Use `NetworkManager`'s built-in `MockTransport` / `InMemoryTransport` rather
  than mocking `NetworkManager` itself (it is not interface-backed).
- Extract server-side validation into plain C# classes that take primitive
  inputs — mock nothing, just `new T()` in tests.

### Mirror
- `MemoryTransport` is Mirror's built-in fake transport for integration tests.
- Wrap `NetworkServer`/`NetworkClient` calls behind a thin interface if direct
  calls are in business logic classes.

### Photon Fusion
- `NetworkRunner` supports `SimulationConfig` overrides; use `GameMode.Shared`
  or `GameMode.Single` for isolated tests.
- Lobby/relay: wrap `FusionAppSettings` access behind an interface.

---

## 5.2 Fake Network Transport Setup (Plan Section 6.2)

### NGO — InMemoryTransport

```
Transport to add: com.unity.netcode.gameobjects (built-in since 1.x)
Class: Unity.Netcode.Transports.InMemoryTransport (or MockTransport in older versions)
```

Test setup pattern (see ref-08 for full code):
1. Create a `GameObject` with `NetworkManager` + `InMemoryTransport`.
2. Call `NetworkManager.StartHost()` on the "host" manager.
3. Create a second `NetworkManager` + `InMemoryTransport` pointing at the same
   in-memory channel.
4. Call `StartClient()` on the second manager.
5. `yield return new WaitUntil(() => client.IsConnectedClient)` to confirm.
6. In `[UnityTearDown]`, call `Shutdown()` on both managers and destroy the
   GameObjects.

### Mirror — MemoryTransport

Add `Mirror.MemoryTransport` to both server and client `NetworkManager`
instances. Call `NetworkServer.Listen(1)` then `NetworkClient.Connect("localhost")`.

### Photon Fusion

Use `GameMode.Single` for offline simulation; use Photon's `SimulationBehaviour`
stubs for unit-level tests.

---

## 5.3 Test Data & Fixtures (Plan Section 6.3)

| Scenario | Strategy |
|---|---|
| Player stats | `ScriptableObject` fixture created with `ScriptableObject.CreateInstance<PlayerStatsSO>()` |
| Level / map data | Small dedicated test scenes checked into `Assets/Tests/Scenes/` |
| Network messages / RPCs | Plain C# structs — no serialisation fixture needed |
| Lobby options | Builder helper: `LobbyOptionsBuilder.Default()` returning safe defaults |
| Random seeds | Fixed seed `new System.Random(42)` in `[SetUp]` — never `UnityEngine.Random` |

Never write tests that depend on `Resources.Load` or `AssetDatabase` for test
data; prefer `ScriptableObject.CreateInstance<T>()` or inline construction.

---

## 5.4 Coverage Targets (Plan Section 6.4)

| System Tier | Line Coverage Target | Branch Coverage Target |
|---|---|---|
| Server authority / anti-cheat | 90% | 85% |
| Core network lifecycle | 80% | 75% |
| Lobby / matchmaking wrappers | 80% | 70% |
| Game state sync | 75% | 70% |
| Gameplay logic (POCO) | 85% | 80% |
| MonoBehaviour UI / audio | 50% | 40% |
| Editor-only code | excluded | excluded |

These are recommended minimums. Adjust down only for classes with testability
score ≤ 2 where the cost of refactoring outweighs the benefit.

---

## 5.5 CI Entry Points (Plan Section 6.5)

```
# EditMode tests only (fast — no Play Mode overhead)
Unity -batchmode -nographics -quit \
  -runTests -testPlatform editmode \
  -testResults results/editmode.xml

# PlayMode tests (requires display or virtual framebuffer)
Unity -batchmode -nographics -quit \
  -runTests -testPlatform playmode \
  -testResults results/playmode.xml

# Full suite
Unity -batchmode -nographics -quit \
  -runTests -testPlatform all \
  -testResults results/all.xml
```

Recommend running EditMode tests on every PR push (fast, < 2 min).
Run PlayMode tests on merge to `develop`/`main` or nightly (slower, 5–15 min).

---

## 5.6 Risk Register & Known Gaps (Plan Section 7)

Include at least one entry per category:

| Gap | Category | Risk | Mitigation |
|---|---|---|---|
| `NetworkBehaviour.Update` logic not extracted | Architecture | High — untestable without full network stack | Refactor to delegate POCO |
| Relay allocation tested manually only | Integration | Medium — CI cannot call live Unity Relay | Mock `IRelayService` in CI; test live in staging |
| Host migration path has no tests | Feature | High — rare but impactful | Add PlayMode test with forced host disconnect |
| No performance baseline for RPC volume | Performance | Medium | Add Measure.Method test for 100 RPCs/frame |

---

## 5.7 Priority Order (Plan Section 8)

Map Criticality to a number first: Critical = 4, High = 3, Medium = 2, Low = 1.
Then rank by: `CriticalityScore × (5 − Testability) / 2`
(high criticality + low current testability = highest priority).

Break ties by: connection lifecycle first, anti-cheat second, state sync third.
