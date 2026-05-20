# Reference 03 — Multiplayer System Catalogue

Map every class from the taxonomy onto the systems below. Add unlisted systems
if the codebase contains them. Each system becomes a Section 5.x in the plan.

---

## 3.1 Core Network Lifecycle

**What to look for:** Classes managing `NetworkManager`, connection state,
disconnect/reconnect, scene transitions triggered by the server.

**Key classes (NGO):** `NetworkManager`, custom `INetworkPrefabInstanceHandler`
implementations, scene-management wrappers.

**Key classes (Mirror):** `NetworkManager`, `NetworkRoomManager` subclasses.

**Key classes (Photon Fusion):** `NetworkRunner`, `INetworkRunnerCallbacks`
implementations.

**Criticality:** Critical  
**Primary test type:** PlayMode + Fake Transport

---

## 3.2 Player Spawning & Ownership

**What to look for:** Classes that instantiate player prefabs, assign ownership,
handle late-joiner spawning, and despawn on disconnect.

**Indicators:** `NetworkObject.SpawnAsPlayerObject`, `SpawnWithOwnership`,
`OnPlayerConnected`, `OnClientConnectedCallback`, `NetworkObject.Despawn`.

**Criticality:** Critical  
**Primary test type:** PlayMode + Fake Transport

---

## 3.3 Server-Authoritative Game State

**What to look for:** Classes that mutate game state only on the server and
replicate results to clients. Anti-cheat / validation lives here.

**Indicators:** `if (IsServer)` guards, `[ServerRpc]`-decorated methods that
validate inputs, `NetworkVariable<T>` writes.

**Criticality:** Critical  
**Primary test type:** EditMode (for validation logic extracted to POCO) +
PlayMode (for end-to-end sync verification)

---

## 3.4 State Synchronisation (NetworkVariables / Dirty Flags)

**What to look for:** `NetworkVariable<T>` field declarations, `OnValueChanged`
callbacks, `INetworkSerializable` implementations, custom `NetworkBehaviour`
subclasses that sync custom structs.

**Criticality:** High  
**Primary test type:** PlayMode (observe sync across fake host+client pair)

---

## 3.5 RPC Routing

**What to look for:** `[ServerRpc]`, `[ClientRpc]`, `[ObserversRpc]`,
`[TargetRpc]` (Mirror) decorated methods, and callers of those methods.

**Test focus:** Verify the RPC is *called* on the correct side, with the
correct parameters, and that server-side RPCs validate caller authority.

**Criticality:** High  
**Primary test type:** PlayMode + Fake Transport

---

## 3.6 Lobby & Matchmaking

**What to look for:** Wrappers around `LobbyService.Instance`, `MatchmakerService`,
`PhotonNetwork.CreateRoom`, `NetworkRunner.StartGame`, or similar entry points.

**Indicators:** `CreateLobbyAsync`, `QuickJoinLobbyAsync`, `JoinLobbyByIdAsync`,
`RequestMatchmakerTicketAsync`.

**Criticality:** High (user-facing connection funnel)  
**Primary test type:** EditMode with mocked service interfaces

---

## 3.7 Relay & Transport Configuration

**What to look for:** Code that calls `RelayService.Instance.CreateAllocationAsync`,
sets `UnityTransport.SetRelayServerData`, or configures transport parameters.

**Criticality:** High (required for NAT traversal in production)  
**Primary test type:** EditMode with mocked `IRelayService` + `IUnityTransport`

---

## 3.8 Authentication & Session Management

**What to look for:** Wrappers around `AuthenticationService.Instance.SignInAnonymouslyAsync`,
token refresh logic, player ID propagation into game state.

**Criticality:** High  
**Primary test type:** EditMode with mocked `IAuthenticationService`

---

## 3.9 Interest Management / Visibility

**What to look for:** Custom `NetworkVisibility` components, distance-based
relevancy checks, team-visibility logic.

**Criticality:** Medium  
**Primary test type:** PlayMode (spawn objects, move out of range, assert observer list)

---

## 3.10 Host Migration (if applicable)

**What to look for:** Logic that detects host disconnect, nominates a new host,
and resumes game state.

**Indicators:** `OnHostMigration`, `INetworkRunnerCallbacks.OnHostMigration`
(Photon Fusion), custom disconnect handlers that re-host.

**Criticality:** High  
**Primary test type:** PlayMode + Fake Transport (force-disconnect the host)

---

## 3.11 Gameplay Systems (non-network)

For each gameplay domain found (Combat, Movement, Inventory, AI, etc.):

- Identify which parts run server-side only vs all clients.
- Flag any `NetworkVariable` or RPC coupling.
- If core logic is extracted to POCO classes → high-coverage EditMode tests.
- If logic is in `NetworkBehaviour.Update` → flag as test gap in plan.

---

## 3.12 System Map Table Format

Produce this table in Section 4 of the plan:

```markdown
| System | Primary Classes | Networking Role | Test Types | Criticality |
|--------|----------------|-----------------|------------|-------------|
| Core Network Lifecycle | NetworkBootstrapper, GameNetworkManager | Host/Client/Server | PlayMode, FakeTransport | Critical |
| Player Spawning | PlayerSpawnManager, NetworkPlayerController | Authority Logic | PlayMode, FakeTransport | Critical |
| Server Game State | GameStateManager, RoundController | Server Authority | EditMode, PlayMode | Critical |
| Lobby | LobbyManager, LobbyUI | Client-only (service calls) | EditMode (mocked) | High |
| ... | | | | |
```
