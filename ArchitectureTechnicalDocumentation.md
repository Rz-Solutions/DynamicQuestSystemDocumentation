# Dynamic Quest System - Technical Architecture Documentation

## 1. System Architecture Overview

The Dynamic Quest System is a modular, interface-driven framework for Unreal Engine 5.3+ that provides a complete solution for creating, managing, and tracking quests at runtime. The architecture follows core UE patterns while maintaining flexibility through smart abstractions.

## 2. Subsystems & Memory Management

The system is built upon UE's Subsystem framework for reliable lifecycle management and easy access:

### 2.1 UQuestManagerSubsystem

```cpp
class DYNAMICQUESTSYSTEM_API UQuestManagerSubsystem : public UGameInstanceSubsystem, public IQuestProvider
{
    // Quest definition repository - Global singleton
    UPROPERTY()
    TMap<FString, FQuestDefinition> QuestDataMap;
    
    // Player progress state repository
    UPROPERTY()
    TMap<FString, TMap<FString, FRuntimeQuestInstance>> PlayerQuestStates;
    
    // Owned data source instance
    UPROPERTY()
    TObjectPtr<UObject> DataSourceInstance;
};
```

**Memory Model**:
- Owned (`UPROPERTY`) data ensures garbage collection safety
- `TObjectPtr` for direct object references (UE5 memory optimization)
- `TMap` for O(1) quest lookup by ID

### 2.2 UPlayerQuestSubsystem

```cpp
class DYNAMICQUESTSYSTEM_API UPlayerQuestSubsystem : public ULocalPlayerSubsystem, public IQuestProgress
{
    // Active player quests
    UPROPERTY()
    TArray<FRuntimeQuestInstance> ActiveQuests;
    
    // Player tracking preferences
    UPROPERTY()
    TArray<FString> TrackedQuestIDs;
    
    // Async save handle for cancellation
    TWeakPtr<FAsyncSaveHandle> CurrentSaveOperation;
    
    // Timer handles for timed objectives
    TMap<FString, FTimerHandle> ObjectiveTimers;
};
```

**Memory Model**:
- `TWeakPtr` for async operation tracking without ownership
- `TTimerHandle` for lightweight timer references

## 3. Data Structures

### 3.1 Quest Definition

```cpp
USTRUCT(BlueprintType)
struct DYNAMICQUESTSYSTEM_API FQuestDefinition
{
    GENERATED_BODY()
    
    // Core identifiers
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest")
    FString QuestID;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest")
    FString Title;
    
    // Quest objectives
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objectives")
    TArray<FQuestObjectiveData> Objectives;
    
    // Rewards
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Rewards")
    FQuestRewardData Reward;
    
    // Requirements
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Requirements")
    int32 LevelRequirement = 1;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Requirements")
    TArray<FString> PrerequisiteQuestIDs;
    
    // Recurring quest settings
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Settings")
    bool bIsRecurring = false;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Settings", meta=(EditCondition="bIsRecurring"))
    float RecurringCooldownHours = 24.0f;
    
    // Serialization operator
    friend FArchive& operator<<(FArchive& Ar, FQuestDefinition& QuestDef);
};
```

### 3.2 Runtime Quest Instance

```cpp
USTRUCT(BlueprintType)
struct DYNAMICQUESTSYSTEM_API FRuntimeQuestInstance
{
    GENERATED_BODY()
    
    // Associated quest ID
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest")
    FString QuestID;
    
    // Current state
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="State")
    EQuestStateType CurrentState = EQuestStateType::NotStarted;
    
    // Objective progress tracking
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Progress")
    TMap<FString, int32> ObjectiveProgress;
    
    // Timestamps
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Timestamps")
    FDateTime StartTime;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Timestamps")
    FDateTime LastCompletionTime;
    
    // Time tracking for timed objectives
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Progress")
    TMap<FString, float> ObjectiveTimeRemaining;
    
    // Serialization operator
    friend FArchive& operator<<(FArchive& Ar, FRuntimeQuestInstance& Instance);
};
```

## 4. Interface Architecture

The system uses interface abstraction for flexibility and extension:

### 4.1 IQuestProvider

```cpp
UINTERFACE(MinimalAPI, Blueprintable)
class UQuestProvider : public UInterface
{
    GENERATED_BODY()
};

class DYNAMICQUESTSYSTEM_API IQuestProvider
{
    GENERATED_BODY()
    
public:
    // Retrieve a specific quest definition
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest")
    bool GetQuestData(const FString& QuestID, FQuestDefinition& OutQuestData) const;
    
    // Get all quest definitions
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest")
    bool GetAllQuests(TArray<FQuestDefinition>& OutQuests) const;
    
    // Check quest availability for a specific player
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest")
    bool IsQuestAvailable(const FString& QuestID, const FString& PlayerID, int32 PlayerLevel) const;
};
```

### 4.2 IQuestDataSource

```cpp
UINTERFACE(MinimalAPI)
class UQuestDataSource : public UInterface
{
    GENERATED_BODY()
};

class DYNAMICQUESTSYSTEM_API IQuestDataSource
{
    GENERATED_BODY()
    
public:
    // Load quest definitions from source
    virtual bool LoadQuests(TMap<FString, FQuestDefinition>& OutQuestMap) = 0;
    
    // Save quest definitions to source
    virtual bool SaveQuests(const TMap<FString, FQuestDefinition>& QuestMap) = 0;
    
    // Get source identifier
    virtual FString GetSourceName() const = 0;
};
```

### 4.3 IQuestProgress

```cpp
UINTERFACE(MinimalAPI, Blueprintable)
class UQuestProgress : public UInterface
{
    GENERATED_BODY()
};

class DYNAMICQUESTSYSTEM_API IQuestProgress
{
    GENERATED_BODY()
    
public:
    // Accept a new quest
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    bool AcceptQuest(const FString& QuestID);
    
    // Abandon an active quest
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    bool AbandonQuest(const FString& QuestID);
    
    // Complete a quest
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    bool CompleteQuest(const FString& QuestID);
    
    // Register progress for an objective
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    bool RegisterObjectiveProgress(const FString& ObjectiveTypeString, const FString& TargetID, int32 Count);
    
    // Check if player has an active quest
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    bool HasActiveQuest(const FString& QuestID) const;
    
    // Get all active quests
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Quest Progress")
    void GetActiveQuests(TArray<FRuntimeQuestInstance>& OutActiveQuests) const;
};
```

## 5. Asynchronous Operations

The system leverages UE's asynchronous capabilities for non-blocking operations:

### 5.1 Async Loading

```cpp
void UQuestManagerSubsystem::AsyncLoadQuests()
{
    // Avoid blocking the game thread during initialization
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [WeakThis = TWeakObjectPtr<UQuestManagerSubsystem>(this)]()
    {
        if (UQuestManagerSubsystem* StrongThis = WeakThis.Get())
        {
            // Perform the actual loading on background thread
            bool bSuccessful = StrongThis->ImportQuestsFromBinary();
            
            // Return to game thread to process results
            AsyncTask(ENamedThreads::GameThread, [WeakThis, bSuccessful]()
            {
                if (UQuestManagerSubsystem* StrongThis = WeakThis.Get())
                {
                    if (!bSuccessful)
                    {
                        // Fall back to data source
                        StrongThis->LoadQuestsAsync();
                    }
                    
                    StrongThis->OnQuestsLoaded.Broadcast();
                }
            });
        }
    });
}
```

### 5.2 Async Saving

```cpp
TSharedPtr<FAsyncSaveHandle> UQuestManagerSubsystem::SavePlayerProgressAsync(const FString& PlayerID)
{
    // Create a shared save handle for tracking/cancellation
    TSharedPtr<FAsyncSaveHandle> SaveHandle = MakeShared<FAsyncSaveHandle>();
    
    AsyncTask(ENamedThreads::AnyBackgroundThreadNormalTask, [WeakThis = TWeakObjectPtr<UQuestManagerSubsystem>(this), 
                                                           PlayerID, SaveHandle]()
    {
        if (SaveHandle->IsCancelled())
        {
            return;
        }
        
        if (UQuestManagerSubsystem* StrongThis = WeakThis.Get())
        {
            // Do serialization work on background thread
            bool bSuccess = StrongThis->SerializePlayerProgressToBinary(PlayerID);
            
            // Return to game thread for notifications
            AsyncTask(ENamedThreads::GameThread, [WeakThis, bSuccess, SaveHandle]()
            {
                if (UQuestManagerSubsystem* StrongThis = WeakThis.Get())
                {
                    if (bSuccess)
                    {
                        StrongThis->OnPlayerProgressSaved.Broadcast();
                    }
                    else
                    {
                        StrongThis->OnPlayerProgressSaveFailed.Broadcast();
                    }
                }
                
                // Mark operation as complete
                SaveHandle->SetComplete();
            });
        }
    });
    
    return SaveHandle;
}
```

### 5.3 Objective Timers

```cpp
void UPlayerQuestSubsystem::StartObjectiveTimer(const FString& QuestID, const FString& ObjectiveID, float TimeLimit)
{
    // Create unique timer key
    FString TimerKey = FString::Printf(TEXT("%s_%s"), *QuestID, *ObjectiveID);
    
    // Clear existing timer if present
    if (ObjectiveTimers.Contains(TimerKey))
    {
        GetWorld()->GetTimerManager().ClearTimer(ObjectiveTimers[TimerKey]);
    }
    
    // Create timer delegate
    FTimerDelegate TimerDelegate = FTimerDelegate::CreateUObject(
        this, &UPlayerQuestSubsystem::OnObjectiveTimerExpired, QuestID, ObjectiveID);
    
    // Start the timer
    FTimerHandle TimerHandle;
    GetWorld()->GetTimerManager().SetTimer(TimerHandle, TimerDelegate, TimeLimit, false);
    ObjectiveTimers.Add(TimerKey, TimerHandle);
    
    // Store remaining time for persistence
    for (FRuntimeQuestInstance& Instance : ActiveQuests)
    {
        if (Instance.QuestID == QuestID)
        {
            Instance.ObjectiveTimeRemaining.Add(ObjectiveID, TimeLimit);
            break;
        }
    }
}
```

## 6. Event System Architecture

The system uses a multi-layered event system to decouple components:

### 6.1 Direct Delegates (Subsystem-Level)

```cpp
class UPlayerQuestSubsystem : public ULocalPlayerSubsystem, public IQuestProgress
{
public:
    // Native C++ callbacks
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnPlayerQuestUpdatedNative, FString /*QuestID*/, FString /*ObjectiveID*/);
    DECLARE_MULTICAST_DELEGATE_OneParam(FOnPlayerQuestCompletedNative, FString /*QuestID*/);
    
    // Blueprint-accessible delegates
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FOnPlayerQuestUpdatedDynamic, FString, QuestID, FString, ObjectiveID);
    DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnPlayerQuestCompletedDynamic, FString, QuestID);
    
    // Native events for C++ binding
    FOnPlayerQuestUpdatedNative OnQuestUpdated;
    FOnPlayerQuestCompletedNative OnQuestCompleted;
    
    // Dynamic events for Blueprint binding
    UPROPERTY(BlueprintAssignable, Category = "Quest Events")
    FOnPlayerQuestUpdatedDynamic OnQuestUpdatedBP;
    
    UPROPERTY(BlueprintAssignable, Category = "Quest Events")
    FOnPlayerQuestCompletedDynamic OnQuestCompletedBP;
};
```

### 6.2 Global Event Manager (System-Wide)

```cpp
// Global event hub for quest-related events
class DYNAMICQUESTSYSTEM_API FQuestEventManager
{
private:
    static FQuestEventManager* Instance;

public:
    // Singleton access
    static FQuestEventManager& Get();
    
    // Global events
    DECLARE_MULTICAST_DELEGATE_OneParam(FOnQuestAvailable, FString /*QuestID*/);
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnQuestAccepted, FString /*QuestID*/, FString /*PlayerID*/);
    DECLARE_MULTICAST_DELEGATE_ThreeParams(FOnObjectiveProgress, FString /*QuestID*/, FString /*ObjectiveID*/, int32 /*NewProgress*/);
    DECLARE_MULTICAST_DELEGATE_TwoParams(FOnQuestCompleted, FString /*QuestID*/, FString /*PlayerID*/);
    
    // Event instances
    FOnQuestAvailable OnQuestAvailable;
    FOnQuestAccepted OnQuestAccepted;
    FOnObjectiveProgress OnObjectiveProgress;
    FOnQuestCompleted OnQuestCompleted;
};
```

## 7. Actor Components & World Integration

### 7.1 Quest Trigger Actor

```cpp
class DYNAMICQUESTSYSTEM_API AQuestTriggerActor : public AActor
{
    GENERATED_BODY()
    
public:
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<USceneComponent> RootComp;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UBoxComponent> TriggerBox;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UWidgetComponent> InteractionWidgetComponent;
    
    // NPC settings
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest NPC")
    FString NPCName;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest NPC")
    FText GreetingDialog;
    
    // Available quests
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Quest")
    TArray<FString> AvailableQuestIDs;
    
    // Trigger settings
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Trigger")
    EQuestTriggerType TriggerType = EQuestTriggerType::AcceptQuest;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Trigger")
    ETriggerActivationMethod ActivationMethod = ETriggerActivationMethod::Interact;
    
    // Objective spawning
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Spawning")
    TArray<FQuestObjectiveSpawnInfo> ObjectiveSpawnInfo;
    
protected:
    // Cached subsystem references
    TWeakObjectPtr<UQuestManagerSubsystem> QuestManagerSubsystem;
    TWeakObjectPtr<UPlayerQuestSubsystem> PlayerQuestSubsystem;
    
    // Spawned actors (for cleanup)
    UPROPERTY()
    TArray<TObjectPtr<AActor>> SpawnedActors;
    
public:
    // Public interface
    void InteractWithNPC(APlayerController* PlayerController);
    void ShowQuestMenu(APlayerController* PlayerController);
    void AcceptQuest(APlayerController* PlayerController, const FString& QuestID);
};
```

### 7.2 Quest Objective Actor

```cpp
class DYNAMICQUESTSYSTEM_API AQuestObjectiveActor : public AActor
{
    GENERATED_BODY()
    
public:
    // Components
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<USceneComponent> RootComp;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UStaticMeshComponent> MeshComponent;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<USphereComponent> InteractionSphere;
    
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category="Components")
    TObjectPtr<UWidgetComponent> ObjectiveMarkerWidget;
    
    // Objective settings
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objective")
    FString QuestID;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objective")
    FString ObjectiveID;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objective")
    ESpawnedObjectiveType ObjectiveType = ESpawnedObjectiveType::Collect;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objective")
    FString TargetID;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Objective")
    int32 RequiredAmount = 1;
    
    // Enemy/NPC spawning settings
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Spawning", meta=(EditCondition="ObjectiveType==ESpawnedObjectiveType::Kill"))
    TSubclassOf<AActor> ActorToSpawn;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Spawning", meta=(EditCondition="ObjectiveType==ESpawnedObjectiveType::Kill"))
    FString EnemyMeshPath;
    
protected:
    // Cached subsystem reference
    TWeakObjectPtr<UPlayerQuestSubsystem> PlayerQuestSubsystem;
    
    // Spawned enemy reference
    UPROPERTY()
    TObjectPtr<AActor> SpawnedEnemy;
    
public:
    // Public interface
    void InteractWithObjective(APlayerController* PlayerController);
    void CompleteObjective(APlayerController* PlayerController);
    void HandleEntityKilled(AActor* KilledEntity, AActor* Killer);
};
```

## 8. Memory Management & Smart Pointers

The system employs UE5's recommended memory management patterns:

### 8.1 Strong References (Ownership)

- `UPROPERTY()` with `TObjectPtr<T>` for owned UObjects
- Direct member variables for trivial data (POD types)
- `TArray`, `TMap` for collections of data

### 8.2 Weak References (Non-ownership)

- `TWeakObjectPtr<T>` for UObject references that shouldn't prevent garbage collection
- `TWeakPtr<T>` for shared non-UObject references to prevent ownership cycles

### 8.3 Shared References

- `TSharedPtr<T>` for shared ownership of non-UObject resources
  - Example: `TSharedPtr<FAsyncSaveHandle>` allows both subsystem and callers to control cancellation

### 8.4 Delegate Safety

```cpp
// Safe callback pattern using WeakThis
template<class UserClass>
void BindSafeCallback(UserClass* InUserObject)
{
    // Create weak pointer to avoid dangling references if object is destroyed
    TWeakObjectPtr<UserClass> WeakUserObject(InUserObject);
    
    // Bind using lambda that checks pointer validity
    SomeDelegate.AddLambda([WeakUserObject]() {
        if (UserClass* StrongUserObject = WeakUserObject.Get())
        {
            StrongUserObject->HandleCallback();
        }
    });
}
```

## 9. Serialization Architecture

### 9.1 Binary Format

```cpp
// Serialize FQuestDefinition to binary
FArchive& operator<<(FArchive& Ar, FQuestDefinition& QuestDef)
{
    // Version control for backward compatibility
    uint32 Version = 1;
    Ar << Version;
    
    // Core data
    Ar << QuestDef.QuestID;
    Ar << QuestDef.Title;
    Ar << QuestDef.Description;
    
    // Objectives
    int32 ObjectiveCount = QuestDef.Objectives.Num();
    Ar << ObjectiveCount;
    
    if (Ar.IsLoading())
    {
        QuestDef.Objectives.SetNum(ObjectiveCount);
    }
    
    for (int32 i = 0; i < ObjectiveCount; ++i)
    {
        Ar << QuestDef.Objectives[i];
    }
    
    // Rewards
    Ar << QuestDef.Reward;
    
    // Requirements
    Ar << QuestDef.LevelRequirement;
    
    int32 PrereqCount = QuestDef.PrerequisiteQuestIDs.Num();
    Ar << PrereqCount;
    
    if (Ar.IsLoading())
    {
        QuestDef.PrerequisiteQuestIDs.SetNum(PrereqCount);
    }
    
    for (int32 i = 0; i < PrereqCount; ++i)
    {
        Ar << QuestDef.PrerequisiteQuestIDs[i];
    }
    
    // Settings
    Ar << QuestDef.bIsRecurring;
    Ar << QuestDef.RecurringCooldownHours;
    
    // Add more fields as needed with version checks
    if (Version >= 2)
    {
        // Future additions
    }
    
    return Ar;
}
```

### 9.2 Data Source Implementation

```cpp
class DYNAMICQUESTSYSTEM_API UBinaryQuestDataSource : public UObject, public IQuestDataSource
{
    GENERATED_BODY()
    
public:
    // File path settings
    UPROPERTY(EditAnywhere, Category="Data Source")
    FString FilePath = TEXT("Quests/DefaultQuests.bin");
    
    UPROPERTY(EditAnywhere, Category="Data Source")
    bool bCreateBackups = true;
    
    UPROPERTY(EditAnywhere, Category="Data Source", meta=(EditCondition="bCreateBackups"))
    FString BackupExtension = TEXT(".bak");
    
    // IQuestDataSource implementation
    virtual bool LoadQuests(TMap<FString, FQuestDefinition>& OutQuestMap) override;
    virtual bool SaveQuests(const TMap<FString, FQuestDefinition>& QuestMap) override;
    virtual FString GetSourceName() const override;
    
protected:
    // Gets full path based on project Saved directory
    FString GetFullPath() const
    {
        return FPaths::Combine(FPaths::ProjectSavedDir(), FilePath);
    }
};
```

## 10. Network Replication Considerations

While the current implementation focuses on single-player scenarios, the architecture has been designed with network extensibility in mind:

### 10.1 Replication-Ready Structure

- **Subsystem Separation**: Having separate `UQuestManagerSubsystem` (server-side) and `UPlayerQuestSubsystem` (client-side) allows for a natural authority separation
- **Interface-Driven Design**: The interfaces provide clear contracts that can be extended for replication
- **Event System**: The event architecture provides hooks for network synchronization

### 10.2 Potential Network Implementation Path

```cpp
// Implementation strategy for networked quest system

// 1. Add replication flags to relevant structs
USTRUCT(BlueprintType, Replicated)
struct DYNAMICQUESTSYSTEM_API FRuntimeQuestInstance
{
    GENERATED_BODY()
    
    // Existing fields...
    
    // Add replication
    void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
    {
        DOREPLIFETIME(FRuntimeQuestInstance, QuestID);
        DOREPLIFETIME(FRuntimeQuestInstance, CurrentState);
        DOREPLIFETIME(FRuntimeQuestInstance, ObjectiveProgress);
        // etc.
    }
};

// 2. Create a server authority wrapper
class DYNAMICQUESTSYSTEM_API UServerQuestManager : public UActorComponent
{
    GENERATED_BODY()
    
    // Server-authoritative quest state
    UPROPERTY(Replicated)
    TArray<FRuntimeQuestInstance> PlayerQuestStates;
    
    // RPC for quest actions
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerAcceptQuest(const FString& QuestID, APlayerController* RequestingPlayer);
    
    UFUNCTION(Server, Reliable, WithValidation)
    void ServerRegisterObjectiveProgress(const FString& ObjectiveTypeString, 
                                       const FString& TargetID, 
                                       int32 Count,
                                       APlayerController* RequestingPlayer);
    
    // Multicast for quest events
    UFUNCTION(NetMulticast, Reliable)
    void MulticastOnQuestUpdated(const FString& QuestID, const FString& ObjectiveID, APlayerController* TargetPlayer);
    
    UFUNCTION(NetMulticast, Reliable)
    void MulticastOnQuestCompleted(const FString& QuestID, APlayerController* TargetPlayer);
};
```

## 11. Extension Points

The system offers multiple extension points for game-specific functionality:

### 11.1 Custom Data Sources

```cpp
// Example JSON data source
class MYGAME_API UJsonQuestDataSource : public UObject, public IQuestDataSource
{
    GENERATED_BODY()
    
public:
    // IQuestDataSource implementation
    virtual bool LoadQuests(TMap<FString, FQuestDefinition>& OutQuestMap) override
    {
        FString JsonString;
        FFileHelper::LoadFileToString(JsonString, *GetFullPath());
        
        TSharedPtr<FJsonObject> JsonObject;
        TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(JsonString);
        
        if (FJsonSerializer::Deserialize(Reader, JsonObject) && JsonObject.IsValid())
        {
            // Parse JSON into quest definitions
            // ...
            return true;
        }
        
        return false;
    }
    
    virtual bool SaveQuests(const TMap<FString, FQuestDefinition>& QuestMap) override
    {
        TSharedPtr<FJsonObject> RootObject = MakeShared<FJsonObject>();
        
        // Convert quest definitions to JSON
        // ...
        
        FString OutputString;
        TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&OutputString);
        FJsonSerializer::Serialize(RootObject.ToSharedRef(), Writer);
        
        return FFileHelper::SaveStringToFile(OutputString, *GetFullPath());
    }
    
    virtual FString GetSourceName() const override
    {
        return FString::Printf(TEXT("JSON File (%s)"), *FilePath);
    }
};
```

### 11.2 Custom Objective Handlers

```cpp
// Example for handling crafting objectives
class MYGAME_API UCraftingObjectiveHandler : public UActorComponent, public IQuestObjectiveHandler
{
    GENERATED_BODY()
    
public:
    // IQuestObjectiveHandler implementation
    virtual bool CanHandleObjectiveType(const FString& ObjectiveType) const override
    {
        return ObjectiveType.Equals(TEXT("Craft"), ESearchCase::IgnoreCase);
    }
    
    virtual bool ProcessEvent(const FString& EventType, const FString& TargetID, int32 EventValue) override
    {
        if (EventType.Equals(TEXT("ItemCrafted"), ESearchCase::IgnoreCase))
        {
            UPlayerQuestSubsystem* PlayerQS = GetOwner()->GetLocalPlayer()->GetSubsystem<UPlayerQuestSubsystem>();
            if (PlayerQS)
            {
                return IQuestProgress::Execute_RegisterObjectiveProgress(PlayerQS, TEXT("Craft"), TargetID, EventValue);
            }
        }
        
        return false;
    }
    
    virtual FString GetHandlerName() const override
    {
        return TEXT("Crafting Objective Handler");
    }
};
```

## 12. Best Practices for Integration

### 12.1 Initializing in Game Mode

```cpp
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();
    
    // Initialize quest UI
    if (QuestTrackerWidgetClass && QuestNotificationWidgetClass)
    {
        APlayerController* PC = UGameplayStatics::GetPlayerController(this, 0);
        if (PC)
        {
            // Create UI widgets
            QuestTrackerWidget = CreateWidget<UQuestTrackerWidget>(PC, QuestTrackerWidgetClass);
            QuestNotificationWidget = CreateWidget<UQuestNotificationWidget>(PC, QuestNotificationWidgetClass);
            
            // Add to viewport
            if (QuestTrackerWidget)
            {
                QuestTrackerWidget->AddToViewport(10);
            }
            
            if (QuestNotificationWidget)
            {
                QuestNotificationWidget->AddToViewport(100);
            }
            
            // Set up UI manager
            QuestUIManager = NewObject<UQuestUIManager>(this);
            if (QuestUIManager)
            {
                QuestUIManager->Initialize(QuestTrackerWidget, QuestNotificationWidget);
            }
        }
    }
}
```

### 12.2 Handling Player Interaction

```cpp
void AMyPlayerController::SetupInputComponent()
{
    Super::SetupInputComponent();
    
    // Bind interaction input
    InputComponent->BindAction("Interact", IE_Pressed, this, &AMyPlayerController::HandleInteractPressed);
}

void AMyPlayerController::HandleInteractPressed()
{
    // Trace for interactable objects
    FHitResult HitResult;
    if (TraceForInteractable(HitResult))
    {
        if (AQuestTriggerActor* QuestTrigger = Cast<AQuestTriggerActor>(HitResult.GetActor()))
        {
            QuestTrigger->InteractWithNPC(this);
            return;
        }
        
        if (AQuestObjectiveActor* QuestObjective = Cast<AQuestObjectiveActor>(HitResult.GetActor()))
        {
            QuestObjective->InteractWithObjective(this);
            return;
        }
        
        // Handle other interactables
    }
}

bool AMyPlayerController::TraceForInteractable(FHitResult& OutHit)
{
    FVector Location;
    FRotator Rotation;
    GetPlayerViewPoint(Location, Rotation);
    
    FVector Start = Location;
    FVector End = Start + (Rotation.Vector() * InteractionDistance);
    
    FCollisionQueryParams QueryParams;
    QueryParams.AddIgnoredActor(GetPawn());
    
    return GetWorld()->LineTraceSingleByChannel(OutHit, Start, End, ECC_Visibility, QueryParams);
}
```

### 12.3 Registering Quest Progress

```cpp
void AMyEnemyCharacter::Die(AActor* Killer)
{
    Super::Die(Killer);
    
    // Register kill for quest progress
    APlayerController* PC = Cast<APlayerController>(Killer->GetInstigatorController());
    if (PC)
    {
        ULocalPlayer* LP = PC->GetLocalPlayer();
        if (LP)
        {
            UPlayerQuestSubsystem* PlayerQS = LP->GetSubsystem<UPlayerQuestSubsystem>();
            if (PlayerQS)
            {
                // EnemyType matches TargetID in objective definitions
                FString EnemyType = GetEnemyType();
                IQuestProgress::Execute_RegisterObjectiveProgress(PlayerQS, TEXT("Kill"), EnemyType, 1);
            }
        }
    }
}
```

This document provides a comprehensive technical overview of the Dynamic Quest System architecture, highlighting its use of UE5's memory management, async operations, interfaces, and extensibility. The system's design allows for flexible integration into diverse game projects while providing a solid foundation for both single-player and potentially networked quest mechanics.