# Reference 07 — PlayMode Test Patterns

---

## 7.1 PlayMode Test Class Structure

Use an instance `NetworkTestHarness` (defined in ref-08) as a class field.
`[UnitySetUp]` and `[UnityTearDown]` handle network lifecycle for every test.

```csharp
using System.Collections;
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;
using UnityEngine.SceneManagement;

[TestFixture, Category("PlayMode")]
public class PlayerSpawningPlayTests
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
}
```

For scene-based tests, load the scene in `[UnitySetUp]` before starting
the harness so `NetworkManager` prefabs in the scene are present when the
host starts:

```csharp
[UnitySetUp]
public IEnumerator SetUp()
{
    yield return SceneManager.LoadSceneAsync("Tests/MinimalNetworkScene",
                                             LoadSceneMode.Single);
    _harness = new NetworkTestHarness();
    yield return _harness.StartHostAndClient();
}
```

---

## 7.2 Timeout Guard Pattern (Inline)

`UnityEngine.WaitUntil` has no built-in timeout. For one-off waits, use the
`CoroutineAssert` helper (defined in section 7.8) rather than a raw
`WaitUntil`, which can hang the test runner indefinitely:

```csharp
// ✗ Can hang forever
yield return new WaitUntil(() => someCondition);

// ✓ Fails clearly after maxWait seconds
yield return CoroutineAssert.WaitUntil(() => someCondition, maxWait: 5f,
    "someCondition was not true within 5s");
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

    player.Die();
    Assert.That(player.IsAlive, Is.False);

    yield return CoroutineAssert.WaitUntil(() => player.IsAlive,
        maxWait: respawn.RespawnDelay + 1f,
        "Player did not respawn within the expected window");

    Assert.That(player.IsAlive, Is.True);
}
```

---

## 7.5 NetworkVariable Sync Test (NGO)

NetworkVariables are replicated on the NGO network tick (default 30 Hz), not
on every Unity frame. Use `CoroutineAssert.WaitUntil` to poll for the
replicated value rather than yielding a fixed number of frames.

```csharp
[UnityTest]
public IEnumerator NetworkVariable_HealthChange_PropagatestoClient()
{
    var prefab   = Resources.Load<GameObject>("TestNetworkPlayer");
    var instance = Object.Instantiate(prefab);
    instance.GetComponent<NetworkObject>().Spawn();
    yield return null; // let spawn propagate

    var hostHealth = instance.GetComponent<NetworkHealthComponent>();
    hostHealth.CurrentHealth.Value = 75f;

    // Poll until the client-side object reflects the change
    yield return CoroutineAssert.WaitUntil(
        () =>
        {
            var objs = Object.FindObjectsByType<NetworkHealthComponent>(
                FindObjectsSortMode.None);
            return objs.Length > 0 && objs[0].CurrentHealth.Value == 75f;
        },
        maxWait: 2f,
        "Health did not replicate within 2s (check NGO tick rate)");

    var clientObjects = Object.FindObjectsByType<NetworkHealthComponent>(
        FindObjectsSortMode.None);
    Assert.That(clientObjects, Has.Length.EqualTo(1));
    Assert.That(clientObjects[0].CurrentHealth.Value, Is.EqualTo(75f));
}
```

---

## 7.6 RPC Call Test (NGO)

```csharp
[UnityTest]
public IEnumerator ServerRpc_DealDamage_OnlyExecutesOnServer()
{
    yield return _harness.SpawnWithOwnership("TestNetworkPlayer",
                                             _harness.Client.LocalClientId);
    var combatant = _harness.LastSpawned.GetComponent<NetworkCombatant>();

    bool serverReceived = false;
    combatant.OnServerDamageReceived += () => serverReceived = true;

    combatant.RequestDealDamageServerRpc(targetId: 0, amount: 25f);

    yield return CoroutineAssert.WaitUntil(() => serverReceived,
        maxWait: 2f, "ServerRpc was not received within 2s");

    Assert.That(serverReceived, Is.True);
    Assert.That(combatant.CurrentHealth, Is.EqualTo(75f));
}
```

---

## 7.7 Scene Transition Test

```csharp
[UnityTest]
public IEnumerator Server_LoadsGameScene_AllClientsFollowTransition()
{
    _harness.Host.SceneManager.LoadScene("GameScene", LoadSceneMode.Single);

    yield return CoroutineAssert.WaitUntil(
        () => SceneManager.GetActiveScene().name == "GameScene",
        maxWait: 10f,
        "Clients did not follow server scene transition within 10s");

    Assert.That(SceneManager.GetActiveScene().name, Is.EqualTo("GameScene"));
}
```

---

## 7.8 CoroutineAssert Helper

Add `Assets/Tests/PlayMode/CoroutineAssert.cs`. All PlayMode tests should use
this instead of raw `WaitUntil` or polling loops to ensure failures surface
with a useful message rather than an infinite hang.

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

```csharp
[UnityTest]
public IEnumerator LobbyManager_CreateLobby_ShowsLobbyScreenOnSuccess()
{
    var mock = new MockLobbyService();
    mock.NextCreateResult = new Lobby
    {
        Id         = "room-99",
        Name       = "Test Room",
        MaxPlayers = 4
    };

    var go      = new GameObject("LobbyManager");
    var manager = go.AddComponent<LobbyManager>();
    manager.Inject(mock);

    manager.CreateLobby("Test Room", maxPlayers: 4);

    yield return CoroutineAssert.WaitUntil(() => manager.State == LobbyState.InLobby,
        maxWait: 3f, "Lobby did not reach InLobby state within 3s");

    Assert.That(manager.CurrentLobbyId, Is.EqualTo("room-99"));
    Object.Destroy(go);
}
```
