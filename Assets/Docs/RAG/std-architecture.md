```yaml
doc_id: "std-architecture"
doc_type: "technical_standards"
title: "Project Architecture Rules - GameDev-Last-War"
owner: "team-unity"
tags:
  - "clean_architecture"
  - "context_separation"
  - "modular_architecture"
  - "dependency_injection"
  - "architecture_messaging"
  - "assembly_dependencies"
  - "unity_architecture"
  - "architecture_network"
  - "meta_mechanics"
  - "checklist"
  - "module_communication"
  - "lifecycle_management"
  - "architecture_packages"
scope: "GameDev-Last-War project, Unity C# development with Clean Architecture and modular design"
last_modified: "2026-01-22T09:34:05Z"
```

# **Project Architecture Rules - GameDev-Last-War**

## **1. Clean Architecture Layers and Boundaries**

### **1.1. Layer Structure and Rules**

#### **Domain Layer (Application/Gameplay)**
- **Content**: Contains only data models and DTOs (MessagesDTO)
- **Allowed**: Validation, add/remove operations
- **Prohibited**: 
  - Business logic
  - Dependencies on other layers
  - Service calls
  - File system operations
  - Network calls
  - External SDK calls
  - Unity-specific logic
- **Paths**:
  - `Assets/_VFXProject/Scripts/Domain/Application/(Models|MessagesDTO)`
  - `Assets/_VFXProject/Scripts/Domain/Gameplay/(Models|MessagesDTO)`

#### **UseCases Layer (Application/Gameplay)**
- **Purpose**: High-level business logic
- **Unique Responsibility**: ONLY layer that modifies Domain models
- **Communication**: Through DTOs and MessagePipe (IPublisher/ISubscriber), or through contracts (ContractsInterfaces)
- **Prohibited**: Direct access to Infrastructure implementations
- **Paths**: `Assets/_VFXProject/Scripts/UseCases/(Application|Gameplay)/...`

#### **Presentation Layer (Application/Gameplay)**
- **Structure**: Separated into Presenters and Views
- **View**: Passive (UIToolkit)
- **Presenter-View Communication**: Through IView interface
- **Model Modification**: Presenter does NOT modify models directly - only through messages to UseCases (MessagePipe/DTO)
- **ECS Subsystem**: For heavy graphics (map, runner, battle)
- **Paths**: `Assets/_VFXProject/Scripts/Presentation/(Application|Gameplay)/(Presenters|Views|ECS)`

#### **Infrastructure Layer**
- **Purpose**: Integration of external services (PlayFab, Firebase), contract implementations, adapters
- **Prohibited**: Business logic and direct dependencies on Domain models
- **Paths**: `Assets/_VFXProject/Scripts/Infrastructure/...`

#### **Repositories Layer**
- **Content**: Configurations/parameters, ScriptableObject
- **Prohibited**: Business logic
- **Paths**: `Assets/_VFXProject/Scripts/Repositories/...`

#### **ContractsInterfaces Layer**
- **Content**: ONLY interfaces (contracts) for DI and exchange between layers/modules
- **Organization**: Separated by contexts (Application/Gameplay) and layers
- **Paths**: `Assets/_VFXProject/Scripts/ContractsInterfaces/...`

## **2. Contextual Separation (Application/Gameplay)**

### **2.1. Context Organization**
- Each layer is divided into Application and Gameplay contexts
- Files must be placed in correct contexts
- Contexts are limited by their own AssemblyDefinition

### **2.2. Assembly Dependencies**
- Eliminate invalid references between contexts/layers (see section 9.3)

## **3. Modular Architecture (SubEpic-Features)**

### **3.1. Module Definition**
Each SubEpic is a separate module with:
- Own AssemblyDefinition (.asmdef)
- Own LifetimeScope (VContainer dependency registration)
- Communication with other modules ONLY through MessagePipe or contracts in ContractsInterfaces
- **Prohibited**: Direct module-to-module references

## **4. Dependency Injection (VContainer)**

### **4.1. DI Principles**
- **Injection Methods**: `[Inject]` or constructor injection
- **Prohibited**: Static dependencies/singletons
- **Prohibited**: `new Service()`/`new Logger()` inside business code - ONLY through DI
- **Lifecycle Classes**: Must implement `IInitializable`/`IDisposable`

### **4.2. Registration Locations**
- **Application**: `Assets/_VFXProject/Scripts/Installers/Application/CommonLifetimeScope.cs`
- **Gameplay**: `Assets/_VFXProject/Scripts/Installers/Gameplay/GameplayLifetimeScope.cs`
- **Modules**: Own LifetimeScope within the module

### **4.3. Examples**
**Correct**:
```csharp
[Inject] private readonly ILogger _logger;
[Inject] private AuthTokenModel _authTokenModel;
public class AuthUseCase : IInitializable, IDisposable {
    public void Initialize() { }
    public void Dispose() { }
}
```

**Incorrect**:
```csharp
public static ILogger Logger;
private ILogger logger = new Logger();
```

## **5. Communication (MessagePipe and DTO)**

### **5.1. MessagePipe Usage**
- Events and messages through `IPublisher`/`ISubscriber`
- **Prohibited**: Bypassing messages and directly accessing foreign models/services

### **5.2. DTO Organization**
- DTOs placed in `Domain/(Application|Gameplay)/MessagesDTO`

## **6. Assembly Dependencies and Namespaces**

### **6.1. Allowed Dependencies (AssemblyDefinition)**
- **Presentation** ↔ Domain, ContractsInterfaces (CANNOT directly reference Infrastructure)
- **UseCases** ↔ Domain, ContractsInterfaces (implementations supplied through DI)
- **Infrastructure** ↔ ContractsInterfaces (CANNOT directly depend on Domain)
- **Modules/SubEpic** - own .asmdef; direct module→module references prohibited
- **Contexts Application/Gameplay** - isolated; no incorrect cross-references

### **6.2. Namespace Organization**
Namespaces must correspond to folder structure:
- `namespace Domain.Gameplay.Models.Buildings`
- `namespace UseCases.Application.Authentication`
- `namespace Presentation.Application.Views`
- `namespace Infrastructure.Database`
- `namespace ContractsInterfaces.Application.Views`

### **6.3. File Suffixes**
- Model, UseCase, View, Presenter, Repository, DTO, Tests

## **7. Unity-Specific Architecture Patterns**

### **7.1. MVP and UI Toolkit**
- **View**: Passive, stores references to UI elements (UIDocument/VisualElement), animations
- **Presenter ↔ View**: ONLY through IView interface
- **Presenter Prohibitions**: 
  - Contains direct Domain logic
  - Does NOT modify models directly - only through UseCases/messages
- **UI MonoBehaviours**: Marked with `[RequireComponent(typeof(UIDocument))]`

### **7.2. ScriptableObject and Configs**
- Configurations designed as ScriptableObject in Repositories
- **Prohibited**: Logic in ScriptableObjects

### **7.3. ECS and Performance Graphics**
- Heavy subsystems (map/runner/battle) - in ECS subsystems `Presentation/Gameplay/ECS`
- Frequently created objects - through Object Pool

## **8. Network Communication and External Services**

### **8.1. Network Operations**
- **Protocol**: All network operations through PlayFab SDK
- **Prohibited**: Direct HTTP requests without explicit approval

### **8.2. External Integrations**
- **Location**: In Infrastructure layer
- **Access**: Through ContractsInterfaces and DI
- **Async Pattern**: Through UniTask

## **9. Global/Meta Mechanics Requirements**

### **9.1. When Feature Affects Global Map/Core/Meta**
- All layers and contexts must be respected
- Configs as ScriptableObject in Repositories (without logic)
- Interaction through MessagePipe/contracts
- Documentation and test scenarios must be present

## **10. Architecture Validation Checklist**

### **10.1. Critical Architecture Checks**
- [ ] Files in correct Application/Gameplay folders; namespaces correspond
- [ ] Domain - without business logic; UseCases are the only ones modifying models
- [ ] VContainer: DI without static; registrations in correct LifetimeScope (+ modules)
- [ ] Communication: MessagePipe/DTO and contracts; no direct inter-module references
- [ ] AssemblyDefinitions: Dependencies correct; SubEpic module with own .asmdef/LifetimeScope
- [ ] Context Separation: No mixing of Application/Gameplay contexts

### **10.2. Critical Architecture Violations (Blockers)**
- Business logic in Domain
- Presenter modifying models directly
- Direct HTTP calls
- Missing DI
- Missing Dispose of subscriptions
- Mixing Application/Gameplay contexts
- Missing XML documentation on public APIs

## **11. Module Communication Standards**

### **11.1. Inter-Module Communication**
- **Allowed**: MessagePipe events, contracts through ContractsInterfaces
- **Prohibited**: Direct references between modules
- **Pattern**: Each module isolated with own assembly and DI scope

### **11.2. Dependency Flow**
```
Presentation → UseCases → Domain
       ↓          ↓
ContractsInterfaces ← Infrastructure
```

## **12. Lifecycle Management**

### **12.1. Resource Management**
- Correct `Dispose()`/`using` patterns
- Release of subscriptions and resources
- All classes with lifecycle implement `IInitializable`/`IDisposable`

### **12.2. Performance Architecture**
- ECS/Jobs/Burst for high-load logic
- Object Pooling where frequent object creation/destruction
- Reuse collections in hot paths

## **13. Package and Dependency Management**

### **13.1. Package Requirements**
- VContainer: `jp.hadashikick.vcontainer`
- MessagePipe, UniTask, R3/UniRx (when compatible)
- External SDKs: ONLY in Infrastructure, through contracts and DI

### **13.2. Migration Notes**
- When migrating to R3: Cannot rely on UniRx types without their availability
- Prefer complete transition to R3 or ensure compatibility at dependency level
