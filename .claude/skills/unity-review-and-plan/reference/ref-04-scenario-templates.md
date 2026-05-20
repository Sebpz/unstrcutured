# Reference 04 — Test Scenario Templates

Use these templates to draft every scenario in Section 5 of the plan.
Fill in real class names and real method names from the codebase.

---

## 4.1 EditMode (Unit) Scenario Template

```
**Scenario ID:** <System>-EM-<n>
**Class under test:** <ClassName> (<file path>)
**Method/Property:** <MethodName>
**Type:** [Happy Path | Edge Case | Error/Invalid Input | Security/Authority]
**Setup:** <what state the object is in before the call>
**Input:** <parameters or actions>
**Expected:** <observable outcome — return value, state change, exception>
**Why it matters:** <one sentence on the multiplayer risk if this breaks>
```

### Mandatory scenarios per system

For every system, write at minimum:
- One happy-path EditMode scenario per public method with business logic
- One boundary scenario (empty collection, zero value, max-int, null)
- One server-authority validation scenario (reject invalid client input)
- One error-path scenario (service unavailable, invalid state transition)

---

## 4.2 PlayMode (Integration) Scenario Template

```
**Scenario ID:** <System>-PM-<n>
**Scene:** <scene to load, or "no scene" if objects are spawned manually>
**Participants:** Host | 1 Client | 2 Clients | Server + Client
**Setup:** <NetworkManager state, which objects are spawned>
**Action:** <what the test does — call method, simulate input, force disconnect>
**Yield:** <frames to wait, or specific condition e.g. "until PlayerSpawned">
**Expected:** <observable outcome across host and/or client instances>
**Cleanup:** <what teardown is required>
```

---

## 4.3 Multiplayer-Specific Scenario Template

Use this for scenarios that specifically exercise networked behaviour:

```
**Scenario ID:** <System>-MP-<n>
**Network Topology:** [Host+Client | Server+Client | Host+2Clients]
**Trigger side:** [Server | Host | Client | OwningClient]
**Scenario:** <description>
**Setup:**
  - Host/Server: <state>
  - Client(s): <state>
**Action:** <which side performs the action>
**Expected on Server/Host:** <state or message>
**Expected on Client(s):** <state or message>
**Latency sensitivity:** [High | Medium | Low]
**Cheat-prevention concern:** [yes | no] — <describe if yes>
```

---

## 4.4 Performance Scenario Template

Only write these if `com.unity.test-framework.performance` is present.

```
**Scenario ID:** <System>-PERF-<n>
**Method/Path:** <hot path to measure>
**Iterations:** <number>
**Budget:** <target ms per frame or per call>
**Allocation budget:** <bytes per call, or "zero alloc">
**Measurement:** [Time | GC Alloc | Custom sampler]
```

---

## 4.5 Multiplayer-Specific Scenario Checklist

For EACH networked system, ensure the plan includes scenarios covering:

### Connection Lifecycle
- [ ] Client connects successfully (host already running)
- [ ] Client times out during connection
- [ ] Client disconnects mid-game; server cleans up state
- [ ] Server disconnects; clients receive callback
- [ ] Late joiner receives current game state (not stale)

### Ownership & Authority
- [ ] Only the owning client can invoke owner-only RPCs
- [ ] Non-owner RPC call is rejected / ignored
- [ ] Ownership transfer completes; new owner's RPCs are accepted
- [ ] Server rejects a ServerRpc with out-of-range / invalid parameters

### State Synchronisation
- [ ] `NetworkVariable` change on server propagates to all clients within one network tick (poll with `WaitUntil`, do not assert after a single `yield return null`)
- [ ] `NetworkVariable` value is correct for a late-joining client
- [ ] `OnValueChanged` callback fires exactly once per change on each client

### Disconnect & Reconnect
- [ ] Host disconnects; remaining clients enter correct fallback state (or migration)
- [ ] Client reconnect flow (if implemented) restores correct player state
- [ ] Player object is despawned when client leaves

### Anti-Cheat / Validation (server-authoritative games)
- [ ] ServerRpc rejects negative damage values
- [ ] ServerRpc rejects positions outside valid map bounds
- [ ] ServerRpc rejects actions from a client who does not own the object
- [ ] ServerRpc rejects actions that violate game rules (e.g. already dead player attacking)

---

## 4.6 Lobby & Matchmaking Scenarios

- [ ] Create lobby succeeds; lobby ID returned
- [ ] Create lobby fails (network error); UI shows error state
- [ ] Quick join finds an available lobby
- [ ] Quick join fails (no lobbies); falls back to create
- [ ] Host leaves lobby; lobby is deleted or ownership transferred
- [ ] Player count reaches max; new join is rejected
- [ ] Lobby data (map, mode) is visible to all members before starting

---

## 4.7 Relay Scenarios

- [ ] Host creates relay allocation; join code returned
- [ ] Client joins via join code; transport connects
- [ ] Relay allocation expires; error is surfaced to user
- [ ] Region selection propagates correct allocation region
