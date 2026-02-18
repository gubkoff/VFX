```yaml
doc_id: "std-logging"
doc_type: "technical_standards"
title: "Logging & Error Handling Rules - GameDev-Last-War"
owner: "team-unity"
tags:
  - "logging_configuration"
  - "log_format"
  - "exception_handling"
  - "metrics"
  - "environments"
  - "checklist"
scope: "GameDev-Last-War project, Unity C# development, Clean Architecture"
last_modified: "2026-01-22T09:34:05Z"
```

# **Logging & Error Handling Rules - GameDev-Last-War**

## **1. Logging Framework & Configuration**

### **1.1. Serilog as Primary Framework**
- **Mandatory Framework**: Use Serilog for all structured logging
- **Unity Compatibility**: Configure Serilog to work with Unity's console and file systems
- **Production Ready**: Ensure Serilog configuration supports both development and production environments

### **1.2. Structured Logging Pattern**
```csharp
// CORRECT: Structured logging with Serilog
logger.Debug("[{ClassName}.{MethodName}] Processing request for player {PlayerId}", 
    nameof(PlayerService), nameof(ProcessPlayerRequest), playerId);

logger.Information("[{ClassName}.{MethodName}] Player {PlayerId} completed level {LevelNumber} in {Duration}ms",
    nameof(GameplayManager), nameof(CompleteLevel), playerId, levelNumber, duration);

logger.Warning("[{ClassName}.{MethodName}] Low inventory for item {ItemId}. Current stock: {StockLevel}",
    nameof(InventoryManager), nameof(CheckInventory), itemId, stockLevel);

logger.Error(exception, "[{ClassName}.{MethodName}] Failed to save player data for {PlayerId}", 
    nameof(PlayerRepository), nameof(SavePlayerData), playerId);
```

- **Template Pattern**: Always use structured logging templates with property placeholders `{PropertyName}`
- **Class/Method Context**: Include `{ClassName}.{MethodName}` in every log message for traceability
- **Property Names**: Use descriptive property names that appear in log aggregation tools

## **2. Log Format Standards**

### **2.1. Standardized Log Message Structure**
```
[ClassName.MethodName] Descriptive message {keyProperty1} {keyProperty2}
```

- **Mandatory Brackets**: Class and method names must be enclosed in `[ ]`
- **Method Scope**: Always log at method level, not class level only
- **Descriptive Messages**: Log messages should explain what's happening in business terms

### **2.2. Log Level Usage Guidelines**

#### **Debug Level**
```csharp
// For development and troubleshooting
logger.Debug("[{ClassName}.{MethodName}] Starting processing of {ItemCount} items", 
    nameof(BatchProcessor), nameof(ProcessBatch), items.Count);
```

- **Usage**: Detailed information for diagnosing problems during development
- **Production**: Typically filtered out in production unless explicitly enabled
- **Scope**: Fine-grained events that are useful for debugging specific flows

#### **Information Level**
```csharp
// For normal operational events
logger.Information("[{ClassName}.{MethodName}] User {UserId} successfully authenticated", 
    nameof(AuthService), nameof(AuthenticateUser), userId);
```

- **Usage**: Track normal application flow and business events
- **Frequency**: Moderate - not every method call, but key milestones
- **Value**: Should provide audit trail of user actions and system state changes

#### **Warning Level**
```csharp
// For unexpected but recoverable conditions
logger.Warning("[{ClassName}.{MethodName}] Cache miss for key {CacheKey}. Loading from database...",
    nameof(CacheService), nameof(GetCachedItem), cacheKey);
```

- **Usage**: Situations that are unusual but don't prevent operation
- **Response**: System can continue, but condition should be investigated
- **Examples**: Retry attempts, fallback mechanisms, performance degradation

#### **Error Level**
```csharp
// For failures that affect a single operation
logger.Error(exception, "[{ClassName}.{MethodName}] Database connection failed for query {QueryType}",
    nameof(DatabaseService), nameof(ExecuteQuery), queryType);
```

- **Usage**: Failures in specific operations that don't crash the application
- **Exception Inclusion**: Always include the exception object when available
- **Context**: Provide enough context to diagnose without debugging

#### **Fatal/Critical Level**
```csharp
// For failures that cause application crash or data corruption
logger.Fatal(exception, "[{ClassName}.{MethodName}] Critical system failure. Application shutting down.",
    nameof(CoreSystem), nameof(Initialize));
```

- **Usage**: Catastrophic failures requiring immediate attention
- **Impact**: Application cannot continue normal operation
- **Action**: Should trigger alerts and immediate intervention

### **2.3. Unity Debug.Log Restrictions**
```csharp
// ALLOWED ONLY for temporary debugging
Debug.Log("TEMP: Checking value: " + someValue); // Must remove before commit

// PROHIBITED in production code
Debug.Log("Player moved to position: " + position); // VIOLATION!
Debug.LogError("Network error: " + errorMessage); // VIOLATION!

// CORRECT: Use Serilog instead
logger.Error("[PlayerController.Update] Network error: {ErrorMessage}", errorMessage);
```

- **Temporary Only**: `Debug.Log` may be used for quick local debugging
- **Commit Requirement**: All `Debug.Log` calls must be removed before committing code
- **Serilog Replacement**: Production logging must use Serilog exclusively
- **Build Impact**: `Debug.Log` statements affect build performance and should be avoided

## **3. Exception Handling Standards**

### **3.1. Try-Catch Requirements**
```csharp
// CORRECT: Every try block has proper catch with logging
public async UniTask<PlayerData> LoadPlayerData(string playerId)
{
    try
    {
        var data = await _playFabService.GetPlayerDataAsync(playerId);
        logger.Information("[{ClassName}.{MethodName}] Successfully loaded data for {PlayerId}",
            nameof(PlayerService), nameof(LoadPlayerData), playerId);
        return data;
    }
    catch (PlayFabException ex)
    {
        // Log with context and handle appropriately
        logger.Error(ex, "[{ClassName}.{MethodName}] PlayFab error loading data for {PlayerId}",
            nameof(PlayerService), nameof(LoadPlayerData), playerId);
        
        // Graceful degradation: return cached or default data
        return await LoadCachedPlayerData(playerId);
    }
    catch (Exception ex)
    {
        // Catch-all with proper logging
        logger.Error(ex, "[{ClassName}.{MethodName}] Unexpected error loading data for {PlayerId}",
            nameof(PlayerService), nameof(LoadPlayerData), playerId);
        throw; // Re-throw for upper layers
    }
}

// INCORRECT: Swallowing exceptions
try
{
    RiskyOperation();
}
catch (Exception ex)
{
    // VIOLATION: No logging, exception swallowed
}
```

- **Mandatory Catch**: Every `try` block must have at least one `catch` block
- **No Empty Catches**: Never catch exceptions without handling or logging
- **Specific Exceptions**: Catch specific exception types before general `Exception`
- **Contextual Logging**: Include business context in error messages

### **3.2. Exception Swallowing Prohibition**
```csharp
// PROHIBITED: Swallowing exceptions
try
{
    await _networkService.SendData(data);
}
catch (Exception)
{
    // Empty catch - VIOLATION!
    // Exception completely lost, no indication of failure
}

// PROHIBITED: Logging without action
try
{
    await _networkService.SendData(data);
}
catch (Exception ex)
{
    logger.Error(ex, "Error sending data");
    // VIOLATION: Logged but no recovery or notification
}

// CORRECT: Proper handling with recovery strategy
try
{
    await _networkService.SendData(data);
}
catch (NetworkTimeoutException ex)
{
    logger.Warning(ex, "[{ClassName}.{MethodName}] Network timeout. Retrying...",
        nameof(DataService), nameof(SendData));
    
    // Implement retry logic
    await RetryWithBackoff(() => _networkService.SendData(data), maxRetries: 3);
}
catch (Exception ex)
{
    logger.Error(ex, "[{ClassName}.{MethodName}] Failed to send data after all retries",
        nameof(DataService), nameof(SendData));
    
    // Graceful degradation: queue for later or notify user
    await _offlineQueue.StoreForLater(data);
    _uiPresenter.ShowWarning("Data will be sent when connection is restored");
}
```

- **Swallowing Definition**: Catching an exception without logging, re-throwing, or handling
- **Business Recovery**: Always implement appropriate recovery or fallback strategies
- **User Notification**: Inform users of failures when appropriate
- **Data Preservation**: Ensure data isn't lost due to exceptions

### **3.3. Retry Strategies & Graceful Degradation**
```csharp
public async UniTask<T> ExecuteWithRetry<T>(Func<UniTask<T>> operation, int maxRetries = 3)
{
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (TransientException ex) when (attempt < maxRetries)
        {
            var delay = CalculateBackoff(attempt);
            logger.Warning(ex, 
                "[{ClassName}.{MethodName}] Attempt {Attempt} failed. Retrying in {Delay}ms",
                nameof(ResilientService), nameof(ExecuteWithRetry), attempt, delay);
            
            await UniTask.Delay(delay);
        }
        catch (TransientException ex)
        {
            logger.Error(ex, 
                "[{ClassName}.{MethodName}] All {MaxRetries} attempts failed",
                nameof(ResilientService), nameof(ExecuteWithRetry), maxRetries);
            throw new OperationFailedException("Service unavailable after retries", ex);
        }
    }
    
    // Should never reach here due to throw in final catch
    throw new InvalidOperationException("Unexpected flow in retry logic");
}
```

- **Transient Failures**: Identify which exceptions are transient (network, timeouts)
- **Exponential Backoff**: Implement increasing delays between retries
- **Circuit Breaker**: Consider circuit breaker pattern for repeated failures
- **Fallback Mechanisms**: Provide alternative functionality when primary fails

## **4. Performance Monitoring & Metrics**

### **4.1. Performance Logging**
```csharp
public async UniTask<Result> ProcessHeavyOperation()
{
    var stopwatch = Stopwatch.StartNew();
    logger.Debug("[{ClassName}.{MethodName}] Starting heavy operation",
        nameof(PerformanceService), nameof(ProcessHeavyOperation));
    
    try
    {
        // Perform operation
        var result = await _heavyService.Process();
        
        var elapsed = stopwatch.ElapsedMilliseconds;
        if (elapsed > 1000) // Log warning for operations over 1 second
        {
            logger.Warning("[{ClassName}.{MethodName}] Heavy operation took {ElapsedMs}ms",
                nameof(PerformanceService), nameof(ProcessHeavyOperation), elapsed);
        }
        else
        {
            logger.Debug("[{ClassName}.{MethodName}] Operation completed in {ElapsedMs}ms",
                nameof(PerformanceService), nameof(ProcessHeavyOperation), elapsed);
        }
        
        return result;
    }
    finally
    {
        stopwatch.Stop();
    }
}
```

- **Timing Measurement**: Log execution times for critical operations
- **Threshold Alerts**: Define performance thresholds that trigger warnings
- **Resource Usage**: Monitor memory allocations, GC pressure, Unity frame times

### **4.2. GC Allocation Monitoring**
```csharp
// Log GC allocations in performance-critical code
private long _lastGCCount;

public void Update()
{
    // Avoid logging every frame, sample periodically
    if (Time.frameCount % 60 == 0) // Log once per second at 60fps
    {
        var currentGC = GC.CollectionCount(0);
        if (currentGC != _lastGCCount)
        {
            logger.Warning("[{ClassName}.Update] GC occurred. Count: {GCCount}. Frame: {Frame}",
                nameof(PerformanceMonitor), currentGC, Time.frameCount);
            _lastGCCount = currentGC;
        }
    }
}
```

- **GC Awareness**: Monitor garbage collection frequency
- **Allocation Hotspots**: Identify and log areas causing excessive allocations
- **Frame Timing**: Track frame time spikes and correlate with logging events

## **5. Production vs Development Logging**

### **5.1. Environment-Specific Configuration**
```json
// Development appsettings.json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Debug",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  }
}

// Production appsettings.json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning",
        "GameDevLastWar.Gameplay": "Warning"
      }
    }
  }
}
```

- **Development**: Verbose logging including Debug level
- **Production**: Reduced logging to Information and above
- **Performance Critical**: Further restrict logging in hot paths for production

### **5.2. Conditional Compilation**
```csharp
public void Update()
{
#if DEVELOPMENT_BUILD || UNITY_EDITOR
    // Detailed performance logging for development
    var startTime = Time.realtimeSinceStartup;
#endif
    
    // Game logic here
    
#if DEVELOPMENT_BUILD || UNITY_EDITOR
    var elapsed = Time.realtimeSinceStartup - startTime;
    if (elapsed > 0.016f) // More than 16ms at 60fps
    {
        logger.Warning("[{ClassName}.Update] Frame took {ElapsedMs}ms",
            nameof(PerformanceSystem), elapsed * 1000);
    }
#endif
}
```

- **Development Builds**: Include detailed diagnostics
- **Editor Mode**: Full logging for debugging
- **Release Builds**: Minimize logging overhead

## **6. Logging Checklist**

### **6.1. Mandatory Checks**
- [ ] All `try` blocks have corresponding `catch` blocks
- [ ] No exceptions are "swallowed" without logging or handling
- [ ] All log messages include `[ClassName.MethodName]` format
- [ ] Structured logging used with property placeholders `{PropertyName}`
- [ ] Appropriate log levels used (Debug/Info/Warning/Error)
- [ ] `Debug.Log` only used for temporary debugging and removed before commit
- [ ] Error logs include exception objects when available
- [ ] Performance-critical paths have minimal logging in production
- [ ] Retry strategies implemented for transient failures
- [ ] Graceful degradation paths exist for critical failures

### **6.2. Critical Violations (Blockers)**
- ❌ Empty catch blocks (exception swallowing)
- ❌ Using `Debug.Log` in production code
- ❌ Log messages without class/method context
- ❌ String concatenation in logs instead of structured templates
- ❌ Missing exception parameter in Error/Fatal logs
- ❌ Logging sensitive data (passwords, tokens, PII)
- ❌ Excessive logging in Update() methods causing performance issues
- ❌ No recovery strategy for expected exceptions

### **6.3. Security Considerations**
- **No Sensitive Data**: Never log passwords, authentication tokens, or personal data
- **Data Masking**: Mask partial data when needed (e.g., "Card ending in ****1234")
- **Compliance**: Ensure logging complies with data protection regulations
- **Log Retention**: Define and implement log retention policies
- **Access Control**: Secure log files from unauthorized access