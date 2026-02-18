```yaml
doc_id: "std-messaging-and-dto"
doc_type: "technical_standards"
title: "Messaging & DTO rules - GameDev-Last-War"
owner: "team-unity"
tags:
  - "messagepipe"
  - "lifetime_disposal"
  - "dto_organization"
  - "integration_checklist"
  - "testing"
scope: "GameDev-Last-War project, Unity C# development, Clean Architecture"
last_modified: "2026-01-22T09:34:05Z"
```

# **Messaging & DTO Communication Rules - GameDev-Last-War**

## **1. MessagePipe Publishing/Subscribing Rules**

### **1.1. Core Communication Pattern**
- **Primary Mechanism**: All inter-layer and inter-module communication MUST use `IPublisher`/`ISubscriber` from MessagePipe
- **Architectural Purpose**: Enforce loose coupling between layers while maintaining strong typing and compile-time safety
- **Prohibited Pattern**: Directly accessing foreign models or services across layer boundaries

### **1.2. Publishing Rules**
```csharp
// CORRECT: Publisher injected through DI, event data as DTO
[Inject] private readonly IPublisher<PlayerDataUpdatedDTO> _publisher;

public async UniTask UpdatePlayerData(PlayerDataDTO data)
{
    // Business logic here
    var updateEvent = new PlayerDataUpdatedDTO 
    { 
        PlayerId = data.PlayerId,
        NewScore = data.Score 
    };
    
    await _publisher.PublishAsync(updateEvent);
}

// FIRE-AND-FORGET PATTERN (When necessary)
public void SendAnalyticsEvent(AnalyticsEventDTO data)
{
    _publisher.PublishAsync(data).Forget(e => 
        _logger.Error($"[{nameof(MyClass)}.SendAnalyticsEvent] Failed: {e}"));
}
```

- **Publisher Location**: UseCases layer primarily, but can be used in Presentation/Infrastructure for cross-cutting concerns
- **Event Data**: MUST be DTOs from `Domain/(Application|Gameplay)/MessagesDTO/`
- **Async Pattern**: Use `PublishAsync()` for asynchronous operations
- **Fire-and-Forget**: Operations not requiring await MUST use `.Forget()` with proper error handling

### **1.3. Subscribing Rules**
```csharp
// CORRECT: Subscription managed with CompositeDisposable
private readonly CompositeDisposable _disposables = new();

[Inject]
public void Initialize(ISubscriber<PlayerDataUpdatedDTO> subscriber)
{
    subscriber.Subscribe(async (dto, ct) =>
    {
        await UpdateUI(dto);
    }).AddTo(_disposables);
}

public void Dispose()
{
    _disposables.Dispose();
}
```

- **Subscription Ownership**: Each subscribing class MUST own and manage its subscriptions
- **Injection Pattern**: `ISubscriber<T>` injected via constructor or `[Inject]` method
- **Handler Pattern**: Subscribe with async lambda or dedicated handler method

### **1.4. Context Separation (CRITICAL)**
```csharp
// APPLICATION CONTEXT (Login, Settings, Meta-progression)
namespace Domain.Application.MessagesDTO;
public class UserLoginDTO { /* ... */ }

// GAMEPLAY CONTEXT (Battle, Inventory, Quests)
namespace Domain.Gameplay.MessagesDTO;
public class BattleStartedDTO { /* ... */ }

// VIOLATION: Mixing contexts
namespace Domain.Application.MessagesDTO;
public class BattleEndedDTO { /* ... */ } // WRONG! Battle is Gameplay context
```

- **Strict Context Isolation**: Application and Gameplay contexts MUST NOT share DTO namespaces
- **AssemblyDefinition Enforcement**: Each context has isolated .asmdef files preventing cross-context references
- **Validation Rule**: DTO namespace MUST match folder structure AND context

## **2. Message Lifetime and Disposal Management**

### **2.1. Subscription Lifetime Rules**
- **Mandatory Container**: All subscriptions MUST be added to a `CompositeDisposable`
- **Cleanup Responsibility**: The owning class MUST implement `IDisposable` and dispose its subscriptions
- **"Lost" Subscription Prevention**: Prohibited to create subscriptions without tracking mechanism

### **2.2. Class Lifecycle Integration**
```csharp
// CORRECT: Full lifecycle implementation
public class PlayerPresenter : IInitializable, IDisposable
{
    private readonly CompositeDisposable _disposables = new();
    
    [Inject]
    private ISubscriber<PlayerDataUpdatedDTO> _playerDataSubscriber;
    
    public void Initialize()
    {
        _playerDataSubscriber
            .Subscribe(OnPlayerDataUpdated)
            .AddTo(_disposables);
    }
    
    private async UniTask OnPlayerDataUpdated(
        PlayerDataUpdatedDTO dto, 
        CancellationToken ct)
    {
        // Handle update
    }
    
    public void Dispose()
    {
        _disposables?.Dispose();
    }
}
```

- **Lifecycle Interfaces**: All classes with subscriptions MUST implement `IInitializable`/`IDisposable`
- **Initialization Separation**: Subscriptions created in `Initialize()` method, not constructor
- **Context-Aware Disposal**: Presenters in Presentation layer dispose on scene/context change

### **2.3. Module-Level Lifetime Management**
```csharp
// Module's LifetimeScope configuration
public class BattleModuleLifetimeScope : LifetimeScope
{
    protected override void Configure(IContainerBuilder builder)
    {
        // Register MessagePipe handlers with scoped lifetime
        builder.RegisterMessagePipe();
        builder.RegisterMessageBroker<BattleStartedDTO>(Lifetime.Scoped);
        builder.RegisterMessageBroker<BattleEndedDTO>(Lifetime.Scoped);
        
        // Register presenters with proper lifecycle
        builder.RegisterEntryPoint<BattlePresenter>(Lifetime.Scoped);
    }
}
```

- **Scoped Registration**: Message handlers registered with appropriate lifetime (Scoped/Transient/Singleton)
- **Module Isolation**: Each module's LifetimeScope manages its own message pipeline
- **Context Boundaries**: Application vs Gameplay contexts have separate message pipelines

## **3. DTO Registration and Organization Rules**

### **3.1. DTO Definition and Location**
- **Directory Structure**: 
  - `Assets/_VFXProject/Scripts/Domain/Application/MessagesDTO/` (for Application context)
  - `Assets/_VFXProject/Scripts/Domain/Gameplay/MessagesDTO/` (for Gameplay context)
- **File Naming**: `[Purpose]DTO.cs` (e.g., `PlayerDataUpdatedDTO.cs`, `InventoryItemAddedDTO.cs`)
- **Namespace Pattern**: `namespace Domain.[Context].MessagesDTO;`

### **3.2. DTO Design Principles with Performance Considerations**
```csharp
// OPTIMIZED DTO: Struct for high-frequency messages (if small, immutable)
public readonly struct PlayerPositionUpdateDTO
{
    public readonly int PlayerId;
    public readonly Vector3 Position;
    public readonly float Timestamp;
    
    public PlayerPositionUpdateDTO(int playerId, Vector3 position, float timestamp)
    {
        PlayerId = playerId;
        Position = position;
        Timestamp = timestamp;
    }
}

// CLASS DTO for complex data
public class PlayerDataUpdatedDTO
{
    /// <summary>
    /// Unique identifier of the player whose data was updated
    /// </summary>
    [Required]
    public string PlayerId { get; set; }
    
    /// <summary>
    /// New score value after update
    /// </summary>
    [Range(0, int.MaxValue)]
    public int NewScore { get; set; }
    
    // ... additional properties
}
```

- **Struct Consideration**: For high-frequency messages (>1000/sec), consider readonly struct DTOs to avoid GC allocations
- **Object Pooling**: If creating DTOs in Update loops, implement object pooling (reference section 6.1 of code style rules)
- **Size Awareness**: Large DTOs (>1KB) should be avoided in hot paths; consider chunking or incremental updates

### **3.3. Network Integration with PlayFab SDK**
```csharp
// Infrastructure Layer handles PlayFab communication
public class PlayFabInventoryService : IInventoryService
{
    [Inject] private IPublisher<InventoryUpdateDTO> _publisher;
    
    public async UniTask<PlayFabResult<AddUserVirtualCurrencyResult>> AddCurrency(int amount)
    {
        var request = new AddUserVirtualCurrencyRequest 
        { 
            VirtualCurrency = "GD", 
            Amount = amount 
        };
        
        var result = await PlayFabClientAPI.AddUserVirtualCurrencyAsync(request);
        
        if (result.Error == null)
        {
            // Convert PlayFab result to internal DTO
            var dto = new CurrencyUpdatedDTO
            {
                CurrencyType = "GD",
                NewBalance = result.Result.Balance,
                Source = "QuestReward"
            };
            
            await _publisher.PublishAsync(dto);
        }
        
        return result;
    }
}

// PROHIBITED: Direct HTTP calls
public async UniTask UpdatePlayerData()
{
    using var client = new HttpClient(); // VIOLATION!
    var response = await client.GetAsync("https://api.game.com/player");
    // ... 
}
```

- **PlayFab Exclusive**: All network communication MUST go through PlayFab SDK wrappers in Infrastructure layer
- **DTO Conversion**: PlayFab responses MUST be converted to internal DTOs before publishing to other layers
- **Direct HTTP Prohibition**: No `HttpClient`, `UnityWebRequest`, or direct REST calls without explicit architectural approval

### **3.4. AssemblyDefinition and Dependency Rules**
```
// Presentation.asmdef references:
- Domain
- ContractsInterfaces
- MessagePipe.Unity
- VContainer

// UseCases.asmdef references:
- Domain  
- ContractsInterfaces
- MessagePipe.Unity
- VContainer

// Infrastructure.asmdef references:
- ContractsInterfaces
- MessagePipe.Unity
- VContainer
- PlayFab SDK assemblies

// VIOLATION: Infrastructure referencing Domain directly
```

- **Strict Dependency Flow**: 
  - Presentation → UseCases → Domain (through messages only)
  - Infrastructure → ContractsInterfaces only (never Domain directly)
- **MessagePipe Assembly**: All layers using messaging MUST reference `MessagePipe.Unity`
- **Module Isolation**: Each SubEpic module with own .asmdef can only reference ContractsInterfaces and MessagePipe, not other modules

### **3.5. Reactive Extensions Compatibility (R3/UniRx)**
```csharp
// CONSISTENT APPROACH: Choose one reactive framework per module
// Option A: Pure MessagePipe
_subscriber.Subscribe(Handler).AddTo(_disposables);

// Option B: MessagePipe + R3 (if project uses R3)
_messageBroker.GetAsyncSubscriber<T>()
    .Subscribe(async (message, ct) => { /* handler */ })
    .AddTo(_disposables);

// PROHIBITED: Mixing patterns without necessity
_subject.OnNext(data); // UniRx pattern
_publisher.PublishAsync(data); // MessagePipe pattern
// ^ Choose one, don't mix in same class/module
```

- **Framework Consistency**: Within a module, use either MessagePipe patterns OR R3/UniRx patterns, not both
- **Migration Strategy**: When migrating from UniRx to R3, update entire module consistently
- **Performance Consideration**: R3 may offer performance benefits for specific high-frequency scenarios

## **4. MessagePipe-DTO Integration Checklist**

### **4.1. Mandatory Checks**
- [ ] DTOs placed in correct `Domain/[Context]/MessagesDTO/` folder
- [ ] DTO namespace matches folder structure: `Domain.[Context].MessagesDTO`
- [ ] All DTOs have XML documentation for class and each property
- [ ] DTOs contain only data, no business logic
- [ ] Publishers use DTOs exclusively for event data
- [ ] Subscribers manage subscriptions with `CompositeDisposable`
- [ ] All subscribing classes implement `IInitializable`/`IDisposable`
- [ ] Subscriptions are created in `Initialize()` method
- [ ] `Dispose()` method properly cleans up all subscriptions
- [ ] No direct model access across layers—only through DTOs
- [ ] MessagePipe used for all inter-module communication
- [ ] Each module has own MessagePipe configuration in LifetimeScope
- [ ] Fire-and-forget operations use `.Forget()` with error handling
- [ ] No mixing of Application and Gameplay context DTOs

### **4.2. Critical Architecture Violations (Blockers)**
- ❌ Creating subscriptions without adding to `CompositeDisposable`
- ❌ DTOs placed outside `Domain/.../MessagesDTO/` folders
- ❌ DTOs containing methods or business logic
- ❌ Missing `IDisposable` implementation in classes with subscriptions
- ❌ Direct layer communication bypassing MessagePipe/DTO
- ❌ Missing XML documentation on DTO classes or properties
- ❌ Mixing Application and Gameplay context DTOs in same namespace
- ❌ **Direct HTTP calls** instead of PlayFab SDK
- ❌ Infrastructure layer referencing Domain layer directly
- ❌ Missing error handling on fire-and-forget operations
- ❌ Creating DTOs in Update() without pooling for high-frequency messages

### **4.3. Package Requirements (from Architecture Rules)**
```
Packages/manifest.json must include:
- "jp.hadashikick.vcontainer": "1.15.0" (or latest)
- "com.cysharp.messagepipe": "2.4.1" (or latest)
- "com.cysharp.unitask": "2.5.0" (or latest)
- PlayFab SDK via Unity Package Manager
- R3 OR UniRx (choose one per module, not both)
```

### **4.4. Performance Audit Points**
- **Hot Path Analysis**: Profile message frequency in Update/FixedUpdate methods
- **Allocation Tracking**: Monitor GC allocations from DTO creation in performance-critical systems
- **Subscription Leaks**: Use memory profiler to verify all subscriptions are properly disposed
- **Network Efficiency**: Ensure PlayFan DTO conversions don't create unnecessary intermediate objects

## **5. Testing Message Flows**

```csharp
[Test]
public async Task PlayerLevelUp_PublishesCorrectDTO()
{
    // Arrange
    var mockPublisher = new Mock<IPublisher<PlayerLevelUpDTO>>();
    var useCase = new PlayerUseCase(mockPublisher.Object);
    
    // Act
    await useCase.LevelUpPlayer("player123");
    
    // Assert
    mockPublisher.Verify(p => p.PublishAsync(
        It.Is<PlayerLevelUpDTO>(dto => 
            dto.PlayerId == "player123" && 
            dto.NewLevel > 0),
        default), 
        Times.Once);
}
```
