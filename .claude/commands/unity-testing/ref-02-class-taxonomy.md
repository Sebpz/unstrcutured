# Reference 02 — Class Taxonomy

Classify every `.cs` file found in Phase 1 using the rules below.
Build one row per file in the classification table.

---

## 2.1 Category Definitions

| Category | Rule | Testability |
|---|---|---|
| **NetworkBehaviour** | Inherits `NetworkBehaviour` (NGO/Mirror/Fish-Net) | Low — needs runtime & transport |
| **NetworkObject** | Has `[NetworkObject]` attribute or is registered as prefab | Low |
| **MonoBehaviour** | Inherits `MonoBehaviour`, not network-aware | Medium — needs GameObject |
| **ScriptableObject** | Inherits `ScriptableObject` | High — create with `ScriptableObject.CreateInstance<T>()` |
| **Pure C# / POCO** | No Unity base class, no `using UnityEngine` beyond math | Very High — `new T()` |
| **Interface / Abstract** | `interface` or `abstract class` | N/A — mockable boundary |
| **Manager / Singleton** | Holds global state, often `DontDestroyOnLoad` | Medium — must reset between tests |
| **Editor Script** | In `Editor/` folder or `#if UNITY_EDITOR` guard | Skip for runtime tests |
| **Generated / Third-Party** | In `ThirdParty/`, `Plugins/`, or auto-generated | Skip |

---

## 2.2 Multiplayer-Specific Sub-Categories

Within NetworkBehaviour classes, further label as:

| Sub-Category | Indicator |
|---|---|
| **Authority Logic** | Methods decorated `[ServerRpc]` or checking `IsServer`/`IsHost` |
| **Client Prediction** | Methods checking `IsOwner` or `IsLocalPlayer` |
| **State Sync** | Contains `NetworkVariable<T>` fields |
| **RPC Handler** | Methods decorated `[ClientRpc]` / `[ObserversRpc]` / `[TargetRpc]` |
| **Lobby/Matchmaking** | Calls `LobbyService`, `MatchmakerService`, or equivalent |
| **Relay/Transport** | Touches `UnityTransport`, `RelayService`, transport configuration |

---

## 2.3 Testability Score (1–5)

Score each class:

| Score | Meaning |
|---|---|
| 5 | Pure C# — `new T()`, zero Unity API surface |
| 4 | ScriptableObject — lightweight, no scene needed |
| 3 | MonoBehaviour — needs `new GameObject().AddComponent<T>()` |
| 2 | NetworkBehaviour with extracted logic interface — mockable boundary |
| 1 | NetworkBehaviour with tight coupling — requires full network stack |

---

## 2.4 Criticality Rating

Rate each system (not individual class) as:

| Rating | Meaning |
|---|---|
| **Critical** | Player progression, anti-cheat, server authority, connection lifecycle |
| **High** | Game state sync, combat/physics resolution, spawn logic |
| **Medium** | Inventory, UI, audio, settings |
| **Low** | Cosmetics, analytics, non-gameplay features |

---

## 2.5 Classification Table Format

Produce this table in the plan document (Section 3):

```markdown
| File | Class | Category | Sub-Category | System | Testability (1-5) | Criticality |
|------|-------|----------|--------------|--------|--------------------|-------------|
| Assets/Scripts/Network/PlayerSpawner.cs | PlayerSpawner | NetworkBehaviour | Authority Logic | Spawning | 2 | Critical |
| Assets/Scripts/Combat/DamageCalculator.cs | DamageCalculator | Pure C# / POCO | — | Combat | 5 | Critical |
| ... | | | | | | |
```

---

## 2.6 Heuristics for Finding Testable Seams

While classifying, flag any of these patterns — they are good extraction targets:

- A `NetworkBehaviour` that delegates business logic to a plain C# class → the
  delegate is already testable; the NetworkBehaviour just needs wiring tests.
- A `static` or singleton helper that could be injected via interface.
- `[SerializeField]` values that could be driven by a `ScriptableObject` fixture
  in tests.
- `Update()` loops that call a private method — the private method is the unit
  to test, exposed via `internal` + `InternalsVisibleTo`.
