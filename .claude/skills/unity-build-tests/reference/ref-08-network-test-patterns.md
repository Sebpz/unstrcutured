# Reference 08 — Network Test Patterns (Multiplayer)

Patterns for testing networked behaviour without live relay/relay infrastructure.

---

## 8.1 NGO — NetworkTestHarness (Instance Class)

`IEnumerator` methods cannot have `out` or `ref` parameters (CS1623). The
solution is an instance-based harness: create one per test class, run it in
`[UnitySetUp]`, and access results through properties.

Create `Assets/Tests/PlayMode/NetworkTestHarness.cs`:

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using Unity.Netcode;
using Unity.Netcode.Transports.InMemory;   // NGO 1.1+

public class NetworkTestHarness
{
    public NetworkManager  Host         { get; private set; }
    public NetworkManager  Client       { get; private set; }
    public NetworkManager  SecondClient { get; private set; }
    public GameObject      LastSpawned  { get; private set; }

    private readonly List<NetworkManager> _all = new();

    // ── Start ───────────────────────────────────────────────────────────────

    public IEnumerator StartHostAndClient()
    {
        Host   = CreateManager("Host");
        Client = CreateManager("Client");

        var hostTransport   = Host.GetComponent<InMemoryTransport>();
        var clientTransport = Client.GetComponent<InMemoryTransport>();
        clientTransport.ConnectToHost(hostTransport);

        Host.StartHost();
        Client.StartClient();

        yield return new WaitUntil(() => Client.IsConnectedClient);
    }

    public IEnumerator AddSecondClient()
    {
        SecondClient = CreateManager("Client2");
        var transport = SecondClient.GetComponent<InMemoryTransport>();
        transport.ConnectToHost(Host.GetComponent<InMemoryTransport>());
        SecondClient.StartClient();
        yield return new WaitUntil(() => SecondClient.IsConnectedClient);
    }

    // ── Spawn ────────────────────────────────────────────────────────────────

    public IEnumerator SpawnWithOwnership(string prefabName, ulong ownerClientId)
    {
        var prefab = Resources.Load<GameObject>(prefabName);
        LastSpawned = Object.Instantiate(prefab);
        LastSpawned.GetComponent<NetworkObject>().SpawnWithOwnership(ownerClientId);
        yield return null;
    }

    // ── Shutdown ─────────────────────────────────────────────────────────────

    public IEnumerator Shutdown()
    {
        foreach (var nm in _all)
        {
            if (nm != null && nm.IsListening)
                nm.Shutdown();
        }
        yield return null;
        foreach (var nm in _all)
        {
            if (nm != null)
                Object.Destroy(nm.gameObject);
        }
        _all.Clear();
    }

    // ── Helpers ──────────────────────────────────────────────────────────────

    private NetworkManager CreateManager(string name)
    {
        var go        = new GameObject(name + "_NetworkManager");
        var nm        = go.AddComponent<NetworkManager>();
        var transport = go.AddComponent<InMemoryTransport>();
        nm.NetworkConfig = new NetworkConfig { NetworkTransport = transport };
        _all.Add(nm);
        return nm;
    }
}
```

> **Note:** `InMemoryTransport` is built into NGO 1.1+.
> For NGO < 1.1, replace with the `MockTransport` from the NGO test utilities
> package and connect via its shared in-memory channel.

---

## 8.2 Mirror — MemoryTransport Harness

Mirror uses static `NetworkServer`/`NetworkClient` classes, not two separate
`NetworkManager` instances. A single `NetworkManager` drives both sides.

```csharp
using System.Collections;
using UnityEngine;
using Mirror;

public class MirrorTestHarness
{
    private NetworkManager _manager;

    public IEnumerator Start()
    {
        var go        = new GameObject("MirrorNetworkManager");
        _manager      = go.AddComponent<NetworkManager>();
        var transport = go.AddComponent<MemoryTransport>();
        Transport.active = transport;

        NetworkServer.Listen(1);
        NetworkClient.Connect("localhost");

        yield return new WaitUntil(() => NetworkClient.isConnected);
    }

    public IEnumerator Shutdown()
    {
        NetworkClient.Disconnect();
        NetworkServer.DisconnectAll();
        NetworkServer.Shutdown();
        yield return null;
        if (_manager != null) Object.Destroy(_manager.gameObject);
    }
}
```

> Mirror's `NetworkServer` and `NetworkClient` are separate static classes —
> access server state via `NetworkServer.*` and client state via
> `NetworkClient.*`. There is no separate "client manager" instance.

---

## 8.3 Forcing Disconnect (Host Migration / Reconnect Tests)

```csharp
[TestFixture, Category("PlayMode")]
public class DisconnectTests
{
    private NetworkTestHarness _harness;

    [UnitySetUp]
    public IEnumerator SetUp()
    {
        _harness = new NetworkTestHarness();
        yield return _harness.StartHostAndClient();
    }

    [UnityTearDown]
    public IEnumerator TearDown() => _harness.Shutdown();

    [UnityTest]
    public IEnumerator ClientDisconnect_ServerCleansUpPlayerObject()
    {
        ulong clientId = _harness.Client.LocalClientId;

        yield return _harness.SpawnWithOwnership("TestNetworkPlayer", clientId);
        var playerGo = _harness.LastSpawned;
        Assert.That(_harness.Host.ConnectedClients.ContainsKey(clientId), Is.True);

        // Force the client off — shut down only the client side
        _harness.Client.Shutdown();

        yield return CoroutineAssert.WaitUntil(
            () => !_harness.Host.ConnectedClients.ContainsKey(clientId),
            maxWait: 3f,
            "Host did not remove disconnected client within 3s");

        Assert.That(_harness.Host.ConnectedClients.ContainsKey(clientId), Is.False);
        Assert.That(playerGo == null || !playerGo.activeSelf, Is.True);

        // Harness.Shutdown() guards against double-shutdown via IsListening check
        yield return _harness.Shutdown();
    }
}
```

---

## 8.4 Late Joiner State Test

```csharp
[UnityTest]
public IEnumerator LateJoiner_ReceivesCurrentGameState()
{
    var harness = new NetworkTestHarness();
    yield return harness.StartHostAndClient();

    // Mutate state on host before the second client joins
    var gameState = Object.FindFirstObjectByType<NetworkGameState>();
    Assert.That(gameState, Is.Not.Null, "NetworkGameState not found — ensure it is spawned on start");
    gameState.RoundNumber.Value = 3;
    gameState.Score.Value      = 150;
    yield return null;

    // Add the late joiner — reuses the host's InMemoryTransport channel
    yield return harness.AddSecondClient();

    // NGO sends a full state snapshot on join; wait one tick for delivery
    yield return CoroutineAssert.WaitUntil(
        () =>
        {
            var gs = Object.FindFirstObjectByType<NetworkGameState>();
            return gs != null && gs.RoundNumber.Value == 3;
        },
        maxWait: 3f,
        "Late joiner did not receive RoundNumber=3 within 3s");

    var lateGs = Object.FindFirstObjectByType<NetworkGameState>();
    Assert.That(lateGs.RoundNumber.Value, Is.EqualTo(3));
    Assert.That(lateGs.Score.Value,       Is.EqualTo(150));

    yield return harness.Shutdown();
}
```

---

## 8.5 RPC Authority Rejection Test

```csharp
[UnityTest]
public IEnumerator ServerRpc_FromNonOwner_IsRejected()
{
    var harness = new NetworkTestHarness();
    yield return harness.StartHostAndClient();

    yield return harness.SpawnWithOwnership("TestNetworkPlayer",
                                            harness.Client.LocalClientId);
    var combatant = harness.LastSpawned.GetComponent<NetworkCombatant>();

    bool unauthorisedCallReceived = false;
    combatant.OnUnauthorisedRpcAttempt += () => unauthorisedCallReceived = true;

    // Host (clientId 0) is NOT the owner — call should be rejected
    combatant.RequestMoveServerRpc(new Vector3(999f, 0f, 0f));
    yield return null;

    Assert.That(unauthorisedCallReceived, Is.True);

    yield return harness.Shutdown();
}
```

> To simulate the call arriving from the host-as-non-owner, invoke the RPC
> directly on the server side. NGO checks `OwnerClientId` inside the generated
> RPC stub and invokes `OnUnauthorisedRpcAttempt` (an event you add to
> `NetworkCombatant`) when the sender is not the owner.

---

## 8.6 Two-Client Sync Test

```csharp
[UnityTest]
public IEnumerator TwoClients_BothReceiveDamageEvent()
{
    var harness = new NetworkTestHarness();
    yield return harness.StartHostAndClient();
    yield return harness.AddSecondClient();   // fully wires SecondClient

    var broadcaster = Object.FindFirstObjectByType<NetworkEventBroadcaster>();
    int callCount = 0;
    broadcaster.OnDamageEventReceived += () => callCount++;

    broadcaster.BroadcastDamageEventClientRpc(damage: 25f, position: Vector3.zero);

    yield return CoroutineAssert.WaitUntil(() => callCount >= 2,
        maxWait: 2f, "Both clients did not receive the ClientRpc within 2s");

    Assert.That(callCount, Is.EqualTo(2));

    yield return harness.Shutdown();
}
```

---

## 8.7 Simulating Latency (Advanced)

`InMemoryTransport` delivers messages immediately with no artificial delay.
For latency-sensitive tests install `com.unity.multiplayer.tools` (NGO 1.4+),
which ships `NetworkSimulator`, and attach it **before** calling `StartHost`.

Flag in the plan if `com.unity.multiplayer.tools` is absent; treat it as a
recommended package for any project with lag-compensated gameplay.

```csharp
public IEnumerator StartHostAndClientWithSimulator(int delayMs = 100,
                                                    int jitterMs = 20,
                                                    int dropPct  = 2)
{
    var harness = new NetworkTestHarness();

    // Attach NetworkSimulator before Start so it is active from the first tick
    var hostGo    = harness.Host.gameObject;     // host created in CreateManager
    var simulator = hostGo.AddComponent<NetworkSimulator>();
    simulator.ConnectionParameters = new SimulatorParameters
    {
        PacketDelayMs        = delayMs,
        PacketJitterMs       = jitterMs,
        PacketDropPercentage = dropPct
    };

    yield return harness.StartHostAndClient();
    // harness.Host and harness.Client are now available
}
```

> **Note:** Attach `NetworkSimulator` before calling `StartHost`. The component
> is on the `NetworkManager`'s `GameObject`; `GetComponent<NetworkSimulator>()`
> will return `null` if it was never added.
