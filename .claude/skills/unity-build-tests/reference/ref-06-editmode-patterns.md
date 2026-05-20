# Reference 06 — EditMode Test Patterns

---

## 6.1 Assembly Setup

### Directory Layout

```
Assets/Tests/
  EditMode/
    Mocks/
    <SystemName>Tests.cs
    ...
  PlayMode/
    Scenes/          ← small test scenes (optional)
    <SystemName>PlayTests.cs
    ...
```

### EditMode asmdef — `Assets/Tests/EditMode/Tests.EditMode.asmdef`

```json
{
  "name": "Tests.EditMode",
  "rootNamespace": "Tests.EditMode",
  "references": [
    "UnityEngine.TestRunner",
    "UnityEditor.TestRunner",
    "<YOUR_RUNTIME_ASMDEF_NAME>"
  ],
  "includePlatforms": ["Editor"],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": true,
  "precompiledReferences": ["nunit.framework.dll"],
  "autoReferenced": false,
  "defineConstraints": [],
  "versionDefines": [],
  "noEngineReferences": false
}
```

Replace `<YOUR_RUNTIME_ASMDEF_NAME>` with the name from `ref-01` Phase 1.5.

### PlayMode asmdef — `Assets/Tests/PlayMode/Tests.PlayMode.asmdef`

```json
{
  "name": "Tests.PlayMode",
  "rootNamespace": "Tests.PlayMode",
  "references": [
    "UnityEngine.TestRunner",
    "<YOUR_RUNTIME_ASMDEF_NAME>"
  ],
  "includePlatforms": [],
  "excludePlatforms": [],
  "allowUnsafeCode": false,
  "overrideReferences": true,
  "precompiledReferences": ["nunit.framework.dll"],
  "autoReferenced": false,
  "optionalUnityReferences": ["TestAssemblies"]
}
```

---

## 6.2 Pure C# / POCO Tests

```csharp
using NUnit.Framework;

[TestFixture, Category("EditMode")]
public class DamageCalculatorTests
{
    private DamageCalculator _sut;

    [SetUp]
    public void SetUp() => _sut = new DamageCalculator();

    [Test]
    public void Calculate_BaseAttackVsNoArmour_ReturnsFullDamage()
    {
        float result = _sut.Calculate(attack: 50f, armour: 0f);
        Assert.That(result, Is.EqualTo(50f).Within(0.001f));
    }

    [TestCase(50f, 25f, 37.5f, TestName = "50% armour halves damage")]
    [TestCase(50f, 100f, 0f,   TestName = "Full armour blocks all damage")]
    [TestCase(0f,  0f,  0f,   TestName = "Zero attack deals zero damage")]
    public void Calculate_VariousArmourValues(float attack, float armour, float expected)
    {
        Assert.That(_sut.Calculate(attack, armour), Is.EqualTo(expected).Within(0.001f));
    }

    [Test]
    public void Calculate_NegativeAttack_ThrowsArgumentOutOfRange()
    {
        Assert.Throws<System.ArgumentOutOfRangeException>(
            () => _sut.Calculate(attack: -1f, armour: 0f));
    }
}
```

---

## 6.3 MonoBehaviour Tests (EditMode)

```csharp
using NUnit.Framework;
using UnityEngine;
using UnityEngine.TestTools;

[TestFixture, Category("EditMode")]
public class HealthComponentTests
{
    private GameObject _go;
    private HealthComponent _health;

    [SetUp]
    public void SetUp()
    {
        _go = new GameObject("TestPlayer");
        _health = _go.AddComponent<HealthComponent>();
        _health.Initialise(maxHealth: 100f);
    }

    [TearDown]
    public void TearDown() => Object.DestroyImmediate(_go);

    [Test]
    public void TakeDamage_ReducesCurrentHealth()
    {
        _health.TakeDamage(30f);
        Assert.That(_health.Current, Is.EqualTo(70f).Within(0.001f));
    }

    [Test]
    public void TakeDamage_BeyondMax_ClampsToZero()
    {
        _health.TakeDamage(999f);
        Assert.That(_health.Current, Is.EqualTo(0f));
        Assert.That(_health.IsDead, Is.True);
    }

    [Test]
    public void Heal_AboveMax_ClampsToMaxHealth()
    {
        _health.TakeDamage(50f);
        _health.Heal(999f);
        Assert.That(_health.Current, Is.EqualTo(100f));
    }
}
```

---

## 6.4 Server-Authority Validation Tests

These test the POCO logic extracted from `[ServerRpc]` handlers.
They do NOT require the network stack.

```csharp
using NUnit.Framework;

[TestFixture, Category("EditMode")]
public class ServerInputValidatorTests
{
    private ServerInputValidator _validator;

    [SetUp]
    public void SetUp() => _validator = new ServerInputValidator(
        mapBounds: new UnityEngine.Bounds(UnityEngine.Vector3.zero,
                                          UnityEngine.Vector3.one * 200f));

    [Test]
    public void ValidateMove_InsideBounds_ReturnsTrue()
    {
        bool valid = _validator.ValidateMove(clientId: 1, position: UnityEngine.Vector3.zero);
        Assert.That(valid, Is.True);
    }

    [Test]
    public void ValidateMove_OutsideBounds_ReturnsFalse()
    {
        bool valid = _validator.ValidateMove(clientId: 1,
            position: new UnityEngine.Vector3(9999f, 0, 0));
        Assert.That(valid, Is.False);
    }

    [Test]
    public void ValidateAttack_DeadTarget_ReturnsFalse()
    {
        bool valid = _validator.ValidateAttack(attackerId: 1, targetId: 2,
            targetIsDead: true);
        Assert.That(valid, Is.False);
    }

    [Test]
    public void ValidateAttack_NegativeDamage_ReturnsFalse()
    {
        bool valid = _validator.ValidateAttack(attackerId: 1, targetId: 2,
            damage: -10f);
        Assert.That(valid, Is.False);
    }
}
```

---

## 6.5 ScriptableObject Tests

```csharp
using NUnit.Framework;
using UnityEngine;

[TestFixture, Category("EditMode")]
public class WeaponDataTests
{
    private WeaponData _weapon;

    [SetUp]
    public void SetUp()
    {
        _weapon = ScriptableObject.CreateInstance<WeaponData>();
        _weapon.BaseDamage = 40f;
        _weapon.FireRate = 0.5f;
    }

    [TearDown]
    public void TearDown() => Object.DestroyImmediate(_weapon);

    [Test]
    public void DPS_CalculatedCorrectly()
    {
        Assert.That(_weapon.DPS, Is.EqualTo(80f).Within(0.001f));
    }
}
```

---

## 6.6 Service Wrapper Tests (Mocked UGS)

```csharp
using NUnit.Framework;
using System.Threading.Tasks;

[TestFixture, Category("EditMode")]
public class LobbyManagerTests
{
    private MockLobbyService _mockLobby;
    private LobbyManager _sut;

    [SetUp]
    public void SetUp()
    {
        _mockLobby = new MockLobbyService();
        _sut = new LobbyManager(_mockLobby);
    }

    [Test]
    public async Task CreateLobby_OnSuccess_StoresLobbyId()
    {
        _mockLobby.NextCreateResult = new FakeLobby { Id = "abc123" };
        await _sut.CreateAndJoinLobbyAsync("My Game", maxPlayers: 4);
        Assert.That(_sut.CurrentLobbyId, Is.EqualTo("abc123"));
    }

    [Test]
    public async Task CreateLobby_OnServiceException_SetsErrorState()
    {
        _mockLobby.NextCreateException = new System.Exception("Rate limited");
        // ThrowsAsync must be awaited; without await the task is fire-and-forget
        // and the subsequent assertion runs before the exception is observed.
        await Assert.ThrowsAsync<System.Exception>(
            () => _sut.CreateAndJoinLobbyAsync("My Game", maxPlayers: 4));
        Assert.That(_sut.State, Is.EqualTo(LobbyState.Error));
    }
}
```

---

## 6.7 Performance Tests (EditMode)

Only generate if `com.unity.test-framework.performance` is in `manifest.json`.

```csharp
using NUnit.Framework;
using Unity.PerformanceTesting;

[TestFixture, Category("Performance")]
public class PathfindingPerfTests
{
    [Test, Performance]
    public void FindPath_100Calls_UnderBudget()
    {
        var pathfinder = new AStarPathfinder(gridSize: 64);
        Measure.Method(() =>
        {
            pathfinder.FindPath(UnityEngine.Vector2Int.zero,
                                new UnityEngine.Vector2Int(63, 63));
        })
        .WarmupCount(3)
        .IterationsPerMeasurement(100)
        .MeasurementCount(10)
        .GC()
        .Run();
    }
}
```
