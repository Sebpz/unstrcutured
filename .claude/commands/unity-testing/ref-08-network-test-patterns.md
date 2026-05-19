# Reference 08 — Network Test Patterns (Multiplayer)

Patterns for testing networked behaviour without live relay/relay infrastructure.

---

## 8.1 NGO — NetworkTestHarness (InMemoryTransport)

Create `Assets/Tests/PlayMode/NetworkTestHarness.cs`:

```csharp
using System.Collections;
using UnityEngine;
using Unity.Netcode;
using Unity.Netcode.Transports.InMemory;   // NGO 1.x+

public static class NetworkTestHarness
{
    public static IEnumerator StartHostAndClient(
        out NetworkManager host, out NetworkManager client)
    {
        // --- Host ---
        var hostGo = new GameObject("Host_NetworkManager");
        host = hostGo.AddComponent<NetworkManager>();
        var hostTransport = hostGo.AddComponent<InMemoryTransport>();
        host.NetworkConfig = new NetworkConfig
        {
            NetworkTransport = hostTransport
        };
        host.StartHost();

        // --- Client ---
        var clientGo = new GameObject("Client_NetworkManager");
        client = clientGo.AddComponent<NetworkManager>();
        var clientTransport = clientGo.AddComponent<InMemoryTransport>();
        client.NetworkConfig = new NetworkConfig
        {
            NetworkTransport = clientTransport
        };

        // Point client at the same in-memory channel
        clientTransport.ConnectToHost(hostTransport);
        client.StartClient();

        yield return new WaitUntil(() => client.IsConnectedClient);
    }

    // Convenience overload for out-params in IEnumerator (use tuple version below)
    public static IEnumerator Shutdown(NetworkManager host, NetworkManager client)
    {
        client.Shutdown();
        yield return null;
        host.Shutdown();
        yield return null;
        Object.Destroy(client.gameObject);
        Object.Destroy(host.gameObject);
    }

    public static IEnumerator SpawnWithOwnership(
        string prefabName, ulong ownerClientId, out GameObject spawned)
    {
        var prefab = Resources.Load<GameObject>(prefabName);
        spawned = Object.Instantiate(prefab);
        spawned.GetComponent<NetworkObject>().SpawnWithOwnership(ownerClientId);
        yield return null;
    }
}
```

> **Note:** NGO's `InMemoryTransport` is available since NGO 1.1.0.  
> For NGO < 1.1, use the `MockTransport` from the NGO test package or mirror
> the in-memory approach with a `NetworkTransport` subclass that queues messages
> in a shared `ConcurrentQueue<byte[]>`.

---

## 8.2 Mirror — MemoryTransport Harness

Mirror ships `Mirror.Tests` with `MemoryTransport`. Use it directly:

```csharp
using System.Collections;
using UnityEngine;
using Mirror;

public static class MirrorTestHarness
{
    public static IEnumerator StartServerAndClient(
        out NetworkManager server, out NetworkManager client)
    {
        // Replace NetworkManager transport with MemoryTransport
        var serverGo = new GameObject("Server");
        server = serverGo.AddComponent<NetworkManager>();
        var transport = serverGo.AddComponent<MemoryTransport>();
        Transport.active = transport;

        NetworkServer.Listen(1);
        NetworkClient.Connect("localhost");

        yield return new WaitUntil(() => NetworkClient.isConnected);
        client = server; // Mirror is single-manager; alias for API symmetry
    }

    public static IEnumerator Shutdown()
    {
        NetworkClient.Disconnect();
        NetworkServer.DisconnectAll();
        NetworkServer.Shutdown();
        yield return null;
    }
}
```

---

## 8.3 Forcing Disconnect (Host Migration / Reconnect Tests)

```csharp
[UnityTest]
public IEnumerator ClientDisconnect_ServerCleansUpPlayerObject()
{
    NetworkManager host = null, client = null;
    yield return NetworkTestHarness.StartHostAndClient(out host, out client);

    ulong clientId = client.LocalClientId;

    // Spawn a player for the client
    GameObject playerGo = null;
    yield return NetworkTestHarness.SpawnWithOwnership(
        "TestNetworkPlayer", clientId, out playerGo);
    yield return null;

    Assert.That(host.ConnectedClients.ContainsKey(clientId), Is.True);

    // Force disconnect
    client.Shutdown();
    yield return new WaitUntil(
        () => !host.ConnectedClients.ContainsKey(clientId), maxWait: 3f);

    Assert.That(host.ConnectedClients.ContainsKey(clientId), Is.False);
    // Confirm player object was despawned
    Assert.That(playerGo == null || !playerGo.activeSelf, Is.True);

    yield return NetworkTestHarness.Shutdown(host, client);
}
```

---

## 8.4 Late Joiner State Test

```csharp
[UnityTest]
public IEnumerator LateJoiner_ReceivesCurrentGameState()
{
    NetworkManager host = null, client1 = null;
    yield return NetworkTestHarness.StartHostAndClient(out host, out client1);

    // Mutate state on host
    var gameState = Object.FindFirstObjectByType<NetworkGameState>();
    gameState.RoundNumber.Value = 3;
    gameState.Score.Value = 150;
    yield return null;

    // Late joiner connects
    var lateClientGo = new GameObject("LateClient_NetworkManager");
    var lateClient = lateClientGo.AddComponent<NetworkManager>();
    // ... configure transport same as above ...
    lateClient.StartClient();
    yield return new WaitUntil(() => lateClient.IsConnectedClient);

    // Late joiner should receive current state immediately
    var lateGameState = Object.FindFirstObjectByType<NetworkGameState>();
    Assert.That(lateGameState.RoundNumber.Value, Is.EqualTo(3));
    Assert.That(lateGameState.Score.Value, Is.EqualTo(150));

    yield return NetworkTestHarness.Shutdown(host, client1);
    lateClient.Shutdown();
    Object.Destroy(lateClientGo);
}
```

---

## 8.5 RPC Authority Rejection Test

```csharp
[UnityTest]
public IEnumerator ServerRpc_FromNonOwner_IsRejected()
{
    NetworkManager host = null, client1 = null;
    yield return NetworkTestHarness.StartHostAndClient(out host, out client1);

    // Spawn player owned by client1
    GameObject playerGo = null;
    yield return NetworkTestHarness.SpawnWithOwnership(
        "TestNetworkPlayer", client1.LocalClientId, out playerGo);

    var combatant = playerGo.GetComponent<NetworkCombatant>();
    bool unauthorisedCallReceived = false;
    combatant.OnUnauthorisedRpcAttempt += () => unauthorisedCallReceived = true;

    // Attempt RPC from HOST (not the owner) — should be rejected
    combatant.RequestMoveServerRpc(
        new Vector3(999, 0, 0),
        new ServerRpcParams
        {
            Receive = new ServerRpcReceiveParams { SenderClientId = 0 }
        });

    yield return null;

    Assert.That(unauthorisedCallReceived, Is.True);

    yield return NetworkTestHarness.Shutdown(host, client1);
}
```

---

## 8.6 Two-Client Sync Test

```csharp
[UnityTest]
public IEnumerator TwoClients_BothReceiveDamageEvent()
{
    NetworkManager host = null, clientA = null;
    yield return NetworkTestHarness.StartHostAndClient(out host, out clientA);

    NetworkManager clientB = null;
    // Start second client (same pattern as harness)
    // ... omitted for brevity — copy harness pattern ...

    // Host fires a ClientRpc
    var broadcaster = Object.FindFirstObjectByType<NetworkEventBroadcaster>();
    int callCount = 0;
    broadcaster.OnDamageEventReceived += () => callCount++;

    broadcaster.BroadcastDamageEventClientRpc(damage: 25f, position: Vector3.zero);

    yield return new WaitUntil(() => callCount >= 2, maxWait: 2f);
    Assert.That(callCount, Is.EqualTo(2)); // both clients received it

    yield return NetworkTestHarness.Shutdown(host, clientA);
}
```

---

## 8.7 Simulating Latency (Advanced)

NGO's `InMemoryTransport` does not simulate latency natively.
For latency-sensitive tests, use the `NetworkSimulator` package:

```csharp
// In test setup, configure the NetworkSimulator (NGO 1.4+)
var simulator = host.GetComponent<NetworkSimulator>();
if (simulator != null)
{
    simulator.ConnectionParameters = new SimulatorParameters
    {
        PacketDelayMs = 100,
        PacketJitterMs = 20,
        PacketDropPercentage = 2
    };
}
```

Flag in the plan if `com.unity.multiplayer.tools` (which includes
`NetworkSimulator`) is not present; add it as a recommended package.
