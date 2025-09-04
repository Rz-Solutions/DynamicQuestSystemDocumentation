# C++20 Features in UE5 by RzZz, Implementation Guide

This document provides comprehensive guidance for implementing C++20 features in Unreal Engine 5 projects, based on extensive testing and debugging made on the QPolice plugin. It covers successfully implemented features, known limitations, and critical patterns for avoiding common pitfalls.

## Successfully Implemented Features

### C++20 Concepts

Concepts provide compile-time interface validation with zero runtime overhead and full UE5 compatibility.

```cpp
// Interface validation with compile-time checking
template<typename T>
concept GameActor = requires(T* actor, const T* constActor) {
    { actor->GetActorType() } -> std::same_as<EActorType>;
    { actor->SetActorType(EActorType{}) } -> std::same_as<void>;
    { constActor->IsActive() } -> std::same_as<bool>;
    { actor->Initialize() } -> std::same_as<void>;
    { actor->Cleanup() } -> std::same_as<void>;
    { constActor->IsValid() } -> std::same_as<bool>;
};

// Usage with automatic interface validation
template<GameActor T>
void ConfigureActor(T* Actor) {
    if (Actor->IsValid()) {
        Actor->Initialize();
    }
}
```

Benefits:
- Compile-time interface validation
- Superior error messages compared to SFINAE
- Zero runtime performance impact
- Full compatibility with UE5 reflection system

### C++20 Coroutines

Coroutines enable asynchronous operations with clean syntax, but require careful lifetime management in UE5.

#### Core Implementation Pattern

```cpp
// Coroutine task implementation
namespace ProjectCoroutines {
    template<typename T = void>
    struct Task {
        struct promise_type {
            Task get_return_object() { 
                return Task{std::coroutine_handle<promise_type>::from_promise(*this)}; 
            }
            std::suspend_never initial_suspend() { return {}; }
            std::suspend_never final_suspend() noexcept { return {}; }
            void unhandled_exception() { std::terminate(); }
            
            // For void return types
            void return_void() requires std::is_void_v<T> {}
            
            // For non-void return types
            void return_value(const T& value) requires (!std::is_void_v<T>) {
                result = value;
            }
            
            T result{};
        };
        
        std::coroutine_handle<promise_type> coro;
        
        explicit Task(std::coroutine_handle<promise_type> h) : coro(h) {}
        ~Task() { if (coro) coro.destroy(); }
        
        // Move-only semantics
        Task(const Task&) = delete;
        Task& operator=(const Task&) = delete;
        Task(Task&& other) noexcept : coro(std::exchange(other.coro, {})) {}
        Task& operator=(Task&& other) noexcept {
            if (this != &other) {
                if (coro) coro.destroy();
                coro = std::exchange(other.coro, {});
            }
            return *this;
        }
        
        bool done() const { return coro && coro.done(); }
        T get_result() requires (!std::is_void_v<T>) { return coro.promise().result; }
    };

    // Timer awaitable for UE5 integration
    struct TimerAwaitable {
        float DelaySeconds;
        
        bool await_ready() { return DelaySeconds <= 0.0f; }
        void await_suspend(std::coroutine_handle<> handle) { 
            // For simple delays during development, resume immediately
            // Production implementations should integrate with UE5 timer system
            handle.resume(); 
        }
        void await_resume() {}
    };
    
    TimerAwaitable DelayFor(float Seconds) { return {Seconds}; }
}
```

#### Usage Example

```cpp
// Coroutine implementation
ProjectCoroutines::Task<bool> UAsyncHelper::ProcessDataAsync() {
    UWorld* World = GetWorld();
    if (!World) {
        UE_LOG(LogTemp, Error, TEXT("Invalid world context"));
        co_return false;
    }
    
    // Load asset asynchronously  
    UClass* AssetClass = co_await LoadClassAsync(AssetPath);
    if (!AssetClass) {
        UE_LOG(LogTemp, Error, TEXT("Failed to load asset class"));
        co_return false;
    }
    
    // Small delay for processing
    co_await DelayFor(0.1f);
    
    // Process the loaded data
    if (UGameSubsystem* Subsystem = UGameSubsystem::Get(World)) {
        Subsystem->ProcessUpdate();
        co_return true;
    }
    
    co_return false;
}
```

#### Critical Lifetime Management

The most common coroutine issue is premature Task destruction causing apparent hanging.

```cpp
// INCORRECT - Task destroyed immediately, killing coroutine
void SomeMethod() {
    auto UpdateTask = Helper->ProcessDataAsync(); // Destroyed at end of scope
    // Coroutine dies here, appears to hang
}

// CORRECT - Store task to prevent destruction
class UGameSubsystem : public UWorldSubsystem {
    // Store the task to keep coroutine alive
    TOptional<ProjectCoroutines::Task<bool>> ActiveProcessingTask;
    
public:
    void StartProcessing() {
        // Store task to prevent immediate destruction
        ActiveProcessingTask = Helper->ProcessDataAsync();
    }
};
```

### std::span Integration

std::span provides memory-safe array views with cross-platform compatibility considerations.

```cpp
#include <algorithm>  // Required for Linux compatibility

class UComponentManager {
    void ProcessComponents(std::span<FComponentData> components) {
        // AVOID - Linux libcxx incompatible
        // auto count = std::ranges::count_if(components, [](const auto& comp) { return comp.IsActive(); });
        
        // RECOMMENDED - Cross-platform compatible  
        auto count = std::count_if(components.begin(), components.end(), [](const auto& comp) { 
            return comp.IsActive(); 
        });
        
        // Safe array access without bounds checking overhead
        for (const auto& component : components) {
            if (component.IsActive()) {
                component.Update();
            }
        }
    }
};
```

## Failed Features and Limitations

### Designated Initializers

UE5's reflection system is incompatible with designated initializers in USTRUCT declarations.

```cpp
// COMPILATION FAILURE - USTRUCT incompatible
USTRUCT(BlueprintType)
struct FGameConfig {
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxPlayers = 10;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite) 
    float SessionTimeout = 5.0f;
};

// This will not compile with UPROPERTY
FGameConfig Config = {
    .MaxPlayers = 20,        // ERROR: USTRUCT doesn't support designated initializers
    .SessionTimeout = 3.0f
};

// WORKAROUND - Use factory functions
static FGameConfig CreateCustomConfig() {
    FGameConfig Config;
    Config.MaxPlayers = 20;
    Config.SessionTimeout = 3.0f;
    return Config;
}
```

### std::format

Linux libcxx has incomplete std::format support, causing compilation failures.

```cpp
// LINUX COMPILATION FAILURE
#include <format>  // Missing in Linux libcxx
auto message = std::format("Entity {} has {} health points", EntityName, Health);

// CROSS-PLATFORM SOLUTION - Use UE5 FString
#define PROJECT_LOG_FORMAT(CategoryName, Verbosity, Format, ...) \
    UE_LOG(CategoryName, Verbosity, Format, ##__VA_ARGS__)

// Usage
PROJECT_LOG_FORMAT(LogGame, Log, TEXT("Entity %s has %d health points"), 
                   *EntityName, Health);
```

### std::ranges

Linux libcxx has incomplete std::ranges support.

```cpp
// LINUX COMPILATION FAILURE
#include <ranges>
auto filtered = entities | std::views::filter([](const auto& entity) { 
    return entity.IsAlive(); 
});

// CROSS-PLATFORM SOLUTION
#include <algorithm>
std::vector<FEntity> filteredEntities;
std::copy_if(entities.begin(), entities.end(), 
             std::back_inserter(filteredEntities),
             [](const auto& entity) { return entity.IsAlive(); });
```

## Cross-Platform Compatibility

### Library Support Differences

| Feature | Windows MSVC | Linux libcxx | Recommendation |
|---------|--------------|---------------|----------------|
| `std::format` | Available | Missing | Use UE5 `FString::Printf` |
| `std::ranges` | Available | Missing | Use standard `<algorithm>` |
| Concepts | Full support | Full support | Safe to use |
| Coroutines | Full support | Full support | Safe to use |
| `std::span` | Full support | Full support | Safe to use |

### Build Configuration

```cs
// Module.Build.cs
public ProjectModule(ReadOnlyTargetRules Target) : base(Target)
{
    PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
    CppStandard = CppStandardVersion.Cpp20;  // Enable C++20
    
    // Linux-specific settings
    if (Target.Platform == UnrealTargetPlatform.Linux)
    {
        // Ensure proper C++20 support
        bUseRTTI = true;
        bEnableExceptions = false;  // UE5 default
    }
}
```

## UE5-Specific Considerations

### Exception Handling

UE5 disables exceptions by default. Use UE5-compatible error handling patterns.

```cpp
// COMPILATION FAILURE - UE5 disables exceptions
ProjectCoroutines::Task<bool> BadExample() {
    try {
        co_await SomeOperation();
        co_return true;
    } catch (const std::exception& e) {  // ERROR: Exceptions disabled
        co_return false;
    }
}

// CORRECT - Use UE5-compatible error handling
#define PROJECT_CORO_CHECK(condition, returnValue, message) \
    if (!(condition)) { \
        UE_LOG(LogTemp, Error, TEXT("Coroutine failed: %s"), message); \
        co_return returnValue; \
    }

ProjectCoroutines::Task<bool> GoodExample() {
    UWorld* World = GetWorld();
    PROJECT_CORO_CHECK(World, false, TEXT("Invalid world"));
    
    UClass* AssetClass = co_await LoadClassAsync();
    PROJECT_CORO_CHECK(AssetClass, false, TEXT("Failed to load asset"));
    
    co_return true;
}
```

### Template Argument Deduction

Explicit template arguments are required for coroutine return types.

```cpp
// COMPILATION ERROR - Missing template argument
ProjectCoroutines::Task ProcessAsync() {  // ERROR: Task<> needs argument
    co_return;
}

// CORRECT - Explicitly specify void for no return value
ProjectCoroutines::Task<void> ProcessAsync() {
    co_await DelayFor(0.1f);
    co_return;  // void return
}

ProjectCoroutines::Task<bool> ValidateAsync() {
    co_await DelayFor(0.1f);
    co_return true;  // bool return
}
```

### UObject Integration

Proper UObject-based implementation for coroutine helpers.

```cpp
// Coroutine helper integrated with UObject system
UCLASS()
class UAsyncOperationHelper : public UObject {
    // Store world reference safely
    TWeakObjectPtr<UWorld> CachedWorld;
    
    // Factory method for proper UObject creation
    static UAsyncOperationHelper* Get(const UObject* WorldContext) {
        UWorld* World = WorldContext->GetWorld();
        if (!World) return nullptr;
        
        UAsyncOperationHelper* Helper = NewObject<UAsyncOperationHelper>();
        Helper->Initialize(World);
        return Helper;
    }
};
```

## Common Pitfalls and Solutions

### Coroutine Hanging

**Problem:** Coroutines appear to hang at co_await calls  
**Root Cause:** Task<> objects destroyed before coroutine completes  
**Solution:** Store Task objects in member variables

```cpp
// CAUSES HANGING
void StartAsyncOperation() {
    auto task = SomeCoroutine();  // Destroyed at end of scope
    // Coroutine appears to hang here
}

// PREVENTS HANGING
TOptional<ProjectCoroutines::Task<bool>> ActiveTask;

void StartAsyncOperation() {
    ActiveTask = SomeCoroutine();  // Task stays alive
}
```

### Timer Callback Crashes

**Problem:** EXCEPTION_ACCESS_VIOLATION in timer callbacks  
**Root Cause:** Objects destroyed while timers still reference them  
**Solution:** Use TWeakObjectPtr and validate before access

```cpp
// CRASH PRONE
GetWorld()->GetTimerManager().SetTimer(TimerHandle, [this]() {
    this->ProcessUpdate();  // 'this' might be destroyed
}, 1.0f, true);

// CRASH SAFE
TWeakObjectPtr<UGameSubsystem> WeakThis = this;
GetWorld()->GetTimerManager().SetTimer(TimerHandle, [WeakThis]() {
    if (WeakThis.IsValid()) {
        WeakThis->ProcessUpdate();  // Safe
    }
}, 1.0f, true);
```

### ProcessEvent Safety

**Problem:** ProcessEvent crashes with invalid function pointers  
**Root Cause:** Function not found or object destroyed  
**Solution:** Always validate functions and prefer C++ APIs

```cpp
// CRASH PRONE
UFunction* Func = Component->FindFunction("SomeFunction");
Component->ProcessEvent(Func, &Params);  // Func might be nullptr

// SAFE
UFunction* Func = Component->FindFunction("SomeFunction");
if (Func && IsValid(Component)) {
    Component->ProcessEvent(Func, &Params);
} else {
    UE_LOG(LogTemp, Error, TEXT("Function not found or component invalid"));
}

// PREFERRED - Use C++ API when available
if (auto* GameComponent = Component->GetGameComponent()) {
    GameComponent->ExecuteAction();  // Direct C++ call
}
```

## Build System Integration

### Module Dependencies

```cs
// ProjectModule.Build.cs
PublicDependencyModuleNames.AddRange(new string[] {
    "Core",
    "CoreUObject", 
    "Engine",
    "AIModule",           // For AI controllers
    "GameplayTasks",      // For async operations
    "NavigationSystem",   // For pathfinding
    "UMG"                // For UI widgets
});

PrivateDependencyModuleNames.AddRange(new string[] {
    "NetCore",            // For networking
    "ReplicationGraph"    // For replication
});
```

### Compiler Settings

```cs
// Enable C++20 while maintaining UE5 compatibility
CppStandard = CppStandardVersion.Cpp20;
bUseRTTI = true;           // Required for concepts
bEnableExceptions = false;  // UE5 standard (maintain default)
```

## Implementation Checklist

### Before Starting C++20 Integration

- [ ] Test cross-platform compilation (Windows + Linux)
- [ ] Plan coroutine lifetime management strategy

### During Implementation

- [ ] Use concepts for interface validation
- [ ] Don't use std::format and std::ranges because of Linux
- [ ] Store Task<> objects to prevent hanging
- [ ] Use TWeakObjectPtr for timer callbacks
- [ ] Validate UFunction pointers before ProcessEvent
- [ ] Prefer C++ APIs over Blueprint ProcessEvent

### After Implementation

- [ ] Test coroutine execution from start to finish
- [ ] Verify no hanging at co_await points  
- [ ] Check for memory leaks in Task destruction
- [ ] Validate cross-platform builds
- [ ] Performance test with multiple concurrent coroutines

## Best Practices

1. **Start Simple:** Implement concepts first (easiest integration)
2. **Coroutines Last:** Most complex feature, requires careful lifetime management
3. **Linux First:** If code compiles on Linux libcxx, it works on all platforms
4. **Store Tasks:** Always store coroutine Task objects in member variables
5. **Weak References:** Use TWeakObjectPtr for any timer or callback references
6. **Validate Everything:** Check function pointers, object validity, world context