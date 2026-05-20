# Reference 07 — PlayMode Test Patterns

---

## 7.1 PlayMode Test Class Structure

```csharp
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;
using UnityEngine.SceneManagement;

[TestFixture, Category("PlayMode")]
public class PlayerSpawningPlayTests
{
    // Shared state created in SetUp, torn down after each test
    private GameObject _networkManagerGo;

    [UnitySetUp]
    public IEnumerator SetUp()
    {
        // Load a minimal test scene OR build the scene in code
        yield return SceneManager.LoadSceneAsync("Tests/MinimalNetworkScene",
                                                 LoadSceneMode.Single);
    }

    [UnityTearDown]
    public IEnumerator TearDown()
    {
        // Always shut down networking before destroying objects
        if (_networkManagerGo != null)
            Object.Destroy(_networkManagerGo);
        yield return null;
    }
}
```

---

## 7.2 Waiting for Conditions

Prefer `WaitUntil` over fixed frame counts — tests are less brittle:

```csharp
// Wait up to 5 seconds for a condition
yield return new WaitUntil(() => condition);

// With a timeout guard (custom helper)
IEnumerator WaitForConditionOrTimeout(System.Func<bool> condition,
                                      float timeoutSec = 5f)
{
    float elapsed = 0f;
    while (!condition() && elapsed < timeoutSec)
    {
        elapsed += Time.deltaTime;
        yield return null;
    }
    Assert.That(condition(), Is.True,
        $"Condition not met within {timeoutSec}s");
}
```

---

## 7.3 Scene-Based Test

```csharp
[UnityTest]
public IEnumerator GameScene_Loads_NetworkManagerPresent()
{
    yield return SceneManager.LoadSceneAsync("GameScene", LoadSceneMode.Single);
    yield return null; // let Awake/Start run

    var nm = Object.FindFirstObjectByType<NetworkManager>();
    Assert.That(nm, Is.Not.Null);
}
```

---

## 7.4 Cross-System Interaction Test

```csharp
[UnityTest]
public IEnumerator PlayerDeath_TriggersRespawnSystem_AfterDelay()
{
    yield return SceneManager.LoadSceneAsync("Tests/MinimalGameScene",
                                             LoadSceneMode.Single);
    yield return null;

    var player = Object.FindFirstObjectByType<PlayerController>();
    var respawn = Object.FindFirstObjectByType<RespawnSystem>();
    Assert.That(player, Is.Not.Null);

    player.Die(); // force death

    // Respawn should be queued but not immediate
    Assert.That(player.IsAlive, Is.False);

    yield return new WaitForSeconds(respawn.RespawnDelay + 0.1f);

    Assert.That(player.IsAlive, Is.True);
}
```

---

## 7.5 NetworkVariable Sync Test (NGO)

Requires the fake transport harness from `ref-08`.

```csharp
[UnityTest]
public IEnumerator NetworkVariable_HealthChange_PropagatestoClient()
{
    var (host, client) = yield return NetworkTestHarness.StartHostAndClient();

    // Spawn a player-like object on the host
    var prefab = Resources.Load<GameObject>("TestNetworkPlayer");
    var instance = Object.Instantiate(prefab);
    instance.GetComponent<NetworkObject>().Spawn();

    yield return null; // let spawn propagate

    // Mutate on host
    var hostHealth = instance.GetComponent<NetworkHealthComponent>();
    hostHealth.CurrentHealth.Value = 75f;

    yield return null; // one frame for sync

    // Observe on client
    var clientObjects = Object.FindObjectsByType<NetworkHealthComponent>(
        FindObjectsSortMode.None);
    Assert.That(clientObjects, Has.Length.EqualTo(1));
    Assert.That(clientObjects[0].CurrentHealth.Value, Is.EqualTo(75f));

    yield return NetworkTestHarness.Shutdown(host, client);
}
```

---

## 7.6 RPC Call Test (NGO)

```csharp
[UnityTest]
public IEnumerator ServerRpc_DealDamage_OnlyExecutesOnServer()
{
    var (host, client) = yield return NetworkTestHarness.StartHostAndClient();

    // Spawn a network object owned by the client
    var playerGo = yield return NetworkTestHarness.SpawnWithOwnership(
        prefabName: "TestNetworkPlayer",
        ownerClientId: client.LocalClientId);

    var combatant = playerGo.GetComponent<NetworkCombatant>();
    bool serverReceived = false;
    combatant.OnServerDamageReceived += () => serverReceived = true;

    // Call the ServerRpc from the owning client side
    combatant.RequestDealDamageServerRpc(targetId: 0, amount: 25f);

    yield return new WaitUntil(() => serverReceived, maxWait: 2f);

    Assert.That(serverReceived, Is.True);
    Assert.That(combatant.CurrentHealth, Is.EqualTo(75f));

    yield return NetworkTestHarness.Shutdown(host, client);
}
```

---

## 7.7 Scene Transition Test

```csharp
[UnityTest]
public IEnumerator Server_LoadsGameScene_AllClientsFollowTransition()
{
    var (host, client) = yield return NetworkTestHarness.StartHostAndClient();

    // Host triggers networked scene load
    host.SceneManager.LoadScene("GameScene", LoadSceneMode.Single);

    // Both host and client should end up in GameScene
    yield return new WaitUntil(
        () => SceneManager.GetActiveScene().name == "GameScene",
        maxWait: 10f);

    Assert.That(SceneManager.GetActiveScene().name, Is.EqualTo("GameScene"));

    yield return NetworkTestHarness.Shutdown(host, client);
}
```

---

## 7.8 Coroutine Test Helper

Avoid duplicating timeout logic — add this helper to `Assets/Tests/PlayMode/`:

```csharp
using System;
using System.Collections;
using NUnit.Framework;
using UnityEngine;

public static class CoroutineAssert
{
    public static IEnumerator WaitUntil(Func<bool> condition,
                                        float maxWait = 5f,
                                        string message = null)
    {
        float elapsed = 0f;
        while (!condition() && elapsed < maxWait)
        {
            elapsed += Time.deltaTime;
            yield return null;
        }
        Assert.That(condition(), Is.True,
            message ?? $"Condition not satisfied within {maxWait}s");
    }
}
```

---

## 7.9 Lobby Integration Test (PlayMode, mocked service)

If `LobbyManager` uses a constructor-injected `ILobbyService`:

```csharp
[UnityTest]
public IEnumerator LobbyManager_CreateLobby_ShowsLobbyScreenOnSuccess()
{
    var mock = new MockLobbyService();
    mock.NextCreateResult = new FakeLobby { Id = "room-99", PlayerCount = 1 };

    var go = new GameObject("LobbyManager");
    var manager = go.AddComponent<LobbyManager>();
    manager.Inject(mock);  // or however DI is wired

    manager.CreateLobby("Test Room", maxPlayers: 4);
    yield return new WaitUntil(() => manager.State == LobbyState.InLobby);

    Assert.That(manager.CurrentLobbyId, Is.EqualTo("room-99"));
    Object.Destroy(go);
}
```
