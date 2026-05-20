# Reference 10 — Mock & Stub Patterns

Hand-rolled mocks for Unity Gaming Services and networking boundaries.
Generate one file per interface under `Assets/Tests/EditMode/Mocks/`.
Only generate a mock if the interface actually exists in the codebase.

---

## 10.1 Mock File Convention

```csharp
// File: Assets/Tests/EditMode/Mocks/Mock<InterfaceName>.cs
// Namespace: Tests.EditMode.Mocks

using System;
using System.Threading.Tasks;

namespace Tests.EditMode.Mocks
{
    public class Mock<InterfaceName> : I<InterfaceName>
    {
        // Configurable results — set in test [SetUp]
        public SomeReturnType NextResult { get; set; }
        public Exception NextException { get; set; }

        // Call-tracking
        public int CallCount { get; private set; }
        public LastCallArgs LastCall { get; private set; }

        public async Task<SomeReturnType> SomeMethodAsync(Arg arg)
        {
            CallCount++;
            LastCall = new LastCallArgs { Arg = arg };
            if (NextException != null) throw NextException;
            return NextResult;
        }

        public record LastCallArgs { public Arg Arg { get; init; } }
    }
}
```

---

## 10.2 ILobbyService Mock

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Unity.Services.Lobbies;
using Unity.Services.Lobbies.Models;

namespace Tests.EditMode.Mocks
{
    public class MockLobbyService : ILobbyService
    {
        public Lobby NextCreateResult { get; set; }
        public Lobby NextJoinResult  { get; set; }
        public Exception NextCreateException { get; set; }
        public Exception NextJoinException   { get; set; }

        public int CreateCallCount   { get; private set; }
        public int QuickJoinCount    { get; private set; }
        public int DeleteCallCount   { get; private set; }
        public string LastDeletedId  { get; private set; }

        public Task<Lobby> CreateLobbyAsync(
            string lobbyName, int maxPlayers, CreateLobbyOptions options = null)
        {
            CreateCallCount++;
            if (NextCreateException != null) throw NextCreateException;
            return Task.FromResult(NextCreateResult
                ?? new Lobby { Id = "mock-lobby-id", Name = lobbyName,
                               MaxPlayers = maxPlayers });
        }

        public Task<Lobby> QuickJoinLobbyAsync(QuickJoinLobbyOptions options = null)
        {
            QuickJoinCount++;
            if (NextJoinException != null) throw NextJoinException;
            return Task.FromResult(NextJoinResult
                ?? new Lobby { Id = "mock-lobby-id" });
        }

        public Task DeleteLobbyAsync(string lobbyId)
        {
            DeleteCallCount++;
            LastDeletedId = lobbyId;
            return Task.CompletedTask;
        }

        public Task<Lobby> GetLobbyAsync(string lobbyId)
            => Task.FromResult(NextJoinResult ?? new Lobby { Id = lobbyId });

        public Task<QueryResponse> QueryLobbiesAsync(QueryLobbiesOptions options = null)
            => Task.FromResult(new QueryResponse
               { Results = new List<Lobby>() });

        // Add remaining interface members as needed — default to no-op / empty
    }
}
```

---

## 10.3 IRelayService Mock

```csharp
using System;
using System.Threading.Tasks;
using Unity.Services.Relay;
using Unity.Services.Relay.Models;

namespace Tests.EditMode.Mocks
{
    public class MockRelayService : IRelayService
    {
        public Allocation NextAllocation { get; set; }
        public JoinAllocation NextJoinAllocation { get; set; }
        public Exception NextException { get; set; }

        public int CreateCallCount { get; private set; }
        public int JoinCallCount   { get; private set; }

        public Task<Allocation> CreateAllocationAsync(
            int maxConnections, string region = null)
        {
            CreateCallCount++;
            if (NextException != null) throw NextException;
            return Task.FromResult(NextAllocation ?? new Allocation
            {
                AllocationId = Guid.NewGuid(),
                MaxConnections = maxConnections
            });
        }

        public Task<string> GetJoinCodeAsync(Guid allocationId)
            => Task.FromResult("MOCK-JOIN-CODE");

        public Task<JoinAllocation> JoinAllocationAsync(string joinCode)
        {
            JoinCallCount++;
            if (NextException != null) throw NextException;
            return Task.FromResult(NextJoinAllocation ?? new JoinAllocation
            {
                AllocationId = Guid.NewGuid()
            });
        }
    }
}
```

---

## 10.4 IAuthenticationService Mock

```csharp
using System;
using System.Threading.Tasks;
using Unity.Services.Authentication;

namespace Tests.EditMode.Mocks
{
    public class MockAuthenticationService : IAuthenticationService
    {
        public bool IsSignedIn { get; set; } = false;
        public string PlayerId { get; set; } = "mock-player-id";
        public string AccessToken { get; set; } = "mock-token";
        public Exception NextSignInException { get; set; }

        public int SignInCallCount  { get; private set; }
        public int SignOutCallCount { get; private set; }

        public Task SignInAnonymouslyAsync(SignInOptions options = null)
        {
            SignInCallCount++;
            if (NextSignInException != null) throw NextSignInException;
            IsSignedIn = true;
            return Task.CompletedTask;
        }

        public Task SignOutAsync()
        {
            SignOutCallCount++;
            IsSignedIn = false;
            AccessToken = null;
            return Task.CompletedTask;
        }

        public Task<string> GetAccessTokenAsync()
            => Task.FromResult(IsSignedIn ? AccessToken : null);

        // Satisfy remaining interface members as no-ops
        public event Action SignedIn;
        public event Action SignedOut;
        public event Action<RequestFailedException> SignInFailed;
    }
}
```

---

## 10.5 FakeLobby Helper

A plain C# record for building test lobby data without the SDK:

```csharp
namespace Tests.EditMode.Mocks
{
    public class FakeLobby
    {
        public string Id          { get; set; } = "test-lobby-id";
        public string Name        { get; set; } = "Test Lobby";
        public int    MaxPlayers  { get; set; } = 4;
        public int    PlayerCount { get; set; } = 1;
        public bool   IsPrivate   { get; set; } = false;

        public static FakeLobby Full(int capacity = 4) =>
            new FakeLobby { MaxPlayers = capacity, PlayerCount = capacity };

        public static FakeLobby Empty(int capacity = 4) =>
            new FakeLobby { MaxPlayers = capacity, PlayerCount = 0 };
    }
}
```

---

## 10.6 NetworkVariable Test Double

For EditMode tests that need a `NetworkVariable<T>`-like observable without
the actual networking stack:

```csharp
using System;

namespace Tests.EditMode.Mocks
{
    public class FakeNetworkVariable<T>
    {
        private T _value;
        public event Action<T, T> OnValueChanged;

        public T Value
        {
            get => _value;
            set
            {
                T old = _value;
                _value = value;
                if (!Equals(old, value))
                    OnValueChanged?.Invoke(old, value);
            }
        }

        public FakeNetworkVariable(T initialValue = default)
            => _value = initialValue;
    }
}
```

Usage: replace `NetworkVariable<float>` with `FakeNetworkVariable<float>` in
an extracted interface or testable base, then run logic in EditMode without
a `NetworkManager`.
