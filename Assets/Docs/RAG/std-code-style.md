```yaml
doc_id: "std-code-style"
doc_type: "technical_standards"
title: "Project Code Style Rules - GameDev-Last-War"
owner: "team-unity"
tags:
  - "naming"
  - "xml_docs"
  - "comments"
  - "organization"
  - "modern_csharp"
  - "performance"
  - "unity"
  - "logging"
  - "network"
  - "testing"
  - "checklist"
scope: "GameDev-Last-War project, Unity C# development, Clean Architecture"
last_modified: "2026-01-22T09:34:05Z"
```

# **Project Code Style Rules - GameDev-Last-War**

## **1. Naming Conventions (Microsoft C#)**

### **1.1. Naming Cases**
- **PascalCase**: classes, interfaces, methods, properties, namespaces
- **camelCase**: local variables, method parameters
- **Private fields**: start with underscore `_fieldName`

### **1.2. Interface Prefixes**
- All interfaces start with `I`: `IView`, `IDatabaseProvider`, `IAzureSyncDTO`

### **1.3. SOLID Principles**
- Minimize dependencies between architectural layers
- Each class/module has single responsibility

## **2. XML Documentation**

### **2.1. Mandatory Documentation**
- All `public` classes, interfaces, methods and properties must have XML documentation
- Description should reflect **purpose and intent**, not implementation
- Avoid trivial/obvious comments

### **2.2. XML Documentation Format**
```csharp
/// <summary>
/// Description of class/method purpose
/// </summary>
/// <param name="parameterName">Parameter purpose</param>
/// <returns>What the method returns</returns>
```

## **3. Code Comments**

### **3.1. Commenting Principles**
- Comments only for **non-obvious decisions**, invariants, and intentions
- Avoid comments describing "what the code does" (code should be self-documenting)
- Explain **why** a particular solution was chosen, not how it works

### **3.2. Prohibited Comments**
- Commented-out code (delete or use version control)
- Redundant comments repeating method/variable names

## **4. Code Organization and Namespaces**

### **4.1. Folder Correspondence**
Namespaces must exactly match folder structure:
```
Assets/_VFXProject/Scripts/Domain/Gameplay/Models/Buildings/
→ namespace Domain.Gameplay.Models.Buildings

Assets/_VFXProject/Scripts/UseCases/Application/Authentication/
→ namespace UseCases.Application.Authentication

Assets/_VFXProject/Scripts/Presentation/Application/Views/
→ namespace Presentation.Application.Views

Assets/_VFXProject/Scripts/Infrastructure/Database/
→ namespace Infrastructure.Database

Assets/_VFXProject/Scripts/ContractsInterfaces/Application/Views/
→ namespace ContractsInterfaces.Application.Views
```

### **4.2. File Suffixes**
- `Model`: domain models in `Domain/`
- `UseCase`: business logic classes in `UseCases/`
- `View`: UI components in `Presentation/Views/`
- `Presenter`: presenters in `Presentation/Presenters/`
- `Repository`: ScriptableObject configs in `Repositories/`
- `DTO`: data transfer objects in `Domain/*/MessagesDTO/`
- `Tests`: test files

## **5. Modern C#**

### **5.1. Allowed Syntax Constructs**
- Target-typed new: `var list = new List<string>();`
- Expression-bodied members: `public int Count => _items.Count;`
- Nullable reference types: `string? nullableString = null;`

### **5.2. Async Programming (UniTask)**
- Use `UniTask`/`UniTaskVoid` instead of standard `Task`
- **Blocking operations prohibited**: `Thread.Sleep`, `Task.Wait`, `.Result`
- Fire-and-forget operations called with `.Forget()` with error handling

### **5.3. Reactivity (R3/UniRx)**
- All subscriptions added to `CompositeDisposable` and properly released via `Dispose()`
- "Lost" subscriptions without unsubscription mechanism prohibited
- When using R3 - API consistency, no mixing with UniRx without necessity

## **6. Performance and Memory Management**

### **6.1. Allocation Minimization**
- Reuse collections in "hot" paths (Update, loops)
- No unnecessary allocations in `Update()`, `FixedUpdate()` methods
- Use Object Pooling for frequent object creation/destruction

### **6.2. ECS and High-Performance Systems**
- Heavy graphics (map, runner, battle) - in ECS subsystems
- Use Burst compilation and Jobs system where applicable

### **6.3. Resource Management**
- Proper resource cleanup via `Dispose()`/`using`
- All lifecycle classes implement `IInitializable`/`IDisposable`

## **7. Unity-Specific**

### **7.1. UI Toolkit Patterns**
- View - passive, stores only UI element references
- Presenter communicates with View only through `IView` interface
- Base UI MonoBehaviour marked with `[RequireComponent(typeof(UIDocument))]`

### **7.2. ScriptableObject**
- Configurations designed as ScriptableObject in `Repositories/` folder
- **Business logic prohibited** in ScriptableObject

## **8. Logging and Error Handling**

### **8.1. Log Format (Serilog)**
```
logger.Debug("[ClassName.MethodName] Message {variableName}");
logger.Information("[ClassName.MethodName] Information {data}");
logger.Warning("[ClassName.MethodName] Warning: {issue}");
logger.Error("[ClassName.MethodName] Error: {exception}");
```
- `Debug.Log` - only for local debugging, not in production code

### **8.2. Exception Handling**
- Every `try` block must have corresponding `catch`
- "Swallowing" exceptions without logging or handling prohibited
- Use retry strategies or graceful degradation when necessary

## **9. Network Communication**

### **9.1. PlayFab SDK**
- All network operations through PlayFab SDK
- Direct HTTP requests prohibited without explicit approval

## **10. Testing**

### **10.1. Unit Tests (NUnit)**
- New logic must be covered by unit tests
- Critical components - target coverage ≥70%
- Test structure: Arrange/Act/Assert

### **10.2. Performance**
- Profiling when modifying heavy subsystems
- Monitoring metrics: FPS, garbage collections (GC), allocations

## **11. Quick Code Style Checklist**

- [ ] Naming per Microsoft C# (PascalCase/camelCase/_privateField)
- [ ] No cyrillic symbols used in executable code
- [ ] XML documentation on all public APIs
- [ ] Comments only for non-obvious decisions
- [ ] Namespaces correspond to folder structure
- [ ] File suffixes match their purpose
- [ ] Use of modern C# constructs (target-typed new, expression-bodied)
- [ ] UniTask instead of Task, no blocking operations
- [ ] All subscriptions properly Dispose()'d
- [ ] Log format: `[Class.Method] Message {vars}`
- [ ] Every try has catch, exceptions not "swallowed"
- [ ] No unnecessary allocations in Update() or Tick() methods, Object Pooling applied where applicable
- [ ] Critical logic test-covered (≥70%)
