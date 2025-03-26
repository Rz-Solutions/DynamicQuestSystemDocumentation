# Dynamic Quest System: Usage Examples (C++)

Here are some common C++ code snippets demonstrating how to interact with the Dynamic Quest System.

## Getting Subsystem References

You typically get the subsystems within classes like `APlayerController`, `AGameModeBase`, `APawn`, or other gameplay classes.

```cpp
#include "Core/QuestManagerSubsystem.h"
#include "Core/PlayerQuestSubsystem.h"
#include "Engine/GameInstance.h"
#include "Engine/LocalPlayer.h"
#include "GameFramework/PlayerController.h" // Or other relevant header

// --- Getting QuestManagerSubsystem (Global) ---

UQuestManagerSubsystem* GetQuestManager()
{
    UGameInstance* GameInstance = GetWorld() ? GetWorld()->GetGameInstance() : nullptr;
    if (GameInstance)
    {
        return GameInstance->GetSubsystem<UQuestManagerSubsystem>();
    }
    return nullptr;
}

// --- Getting PlayerQuestSubsystem (Specific to a PlayerController) ---

UPlayerQuestSubsystem* GetPlayerQuestSystem(APlayerController* PlayerController)
{
    if (!PlayerController) return nullptr;

    ULocalPlayer* LocalPlayer = PlayerController->GetLocalPlayer();
    if (LocalPlayer)
    {
        return LocalPlayer->GetSubsystem<UPlayerQuestSubsystem>();
    }
    return nullptr;
}

// Example Usage in an Actor Component
void UMyComponent::DoSomethingWithQuests()
{
    APlayerController* MyOwnerPC = Cast<APlayerController>(GetOwner()); // Or however you get the relevant PC
    UPlayerQuestSubsystem* PlayerQS = GetPlayerQuestSystem(MyOwnerPC);
    UQuestManagerSubsystem* QuestManager = GetQuestManager();

    if (PlayerQS && QuestManager)
    {
        // ... Use the subsystems ...
    }
}
```

## Accepting a Quest

This is usually triggered by player input, like interacting with an NPC or trigger.

```cpp
#include "Core/PlayerQuestSubsystem.h"
#include "Interfaces/IQuestProgress.h" // For Execute_ functions

void AMyNPC::PlayerInteracted(APlayerController* PlayerController)
{
    UPlayerQuestSubsystem* PlayerQS = GetPlayerQuestSystem(PlayerController);
    if (!PlayerQS) return;

    const FString QuestIDToAccept = TEXT("Q001_KillSpiders"); // Example Quest ID

    // Optional: Check availability first (level, prerequisites etc.)
    // UQuestManagerSubsystem* QuestManager = GetQuestManager();
    // if (QuestManager && IQuestProvider::Execute_IsQuestAvailable(QuestManager, QuestIDToAccept, PlayerQS->GetPlayerID(), PlayerQS->GetPlayerLevel()))
    // { ... }

    // Attempt to accept
    bool bAccepted = IQuestProgress::Execute_AcceptQuest(PlayerQS, QuestIDToAccept);

    if (bAccepted)
    {
        UE_LOG(LogTemp, Log, TEXT("Player accepted quest: %s"), *QuestIDToAccept);
        // Play sound, update NPC dialog, etc.
    }
    else
    {
        UE_LOG(LogTemp, Warning, TEXT("Failed to accept quest: %s"), *QuestIDToAccept);
        // Maybe show a message "You already have this quest" or "Cannot accept this quest yet"
    }
}
```

## Updating Objective Progress (e.g., On Enemy Killed)

When a gameplay event relevant to an objective occurs.

```cpp
#include "Core/PlayerQuestSubsystem.h"
#include "Interfaces/IQuestProgress.h"

void AMyEnemyAIController::OnEnemyKilled(AActor* Killer) // Assume this is called when the enemy dies
{
    APlayerController* PlayerController = Cast<APlayerController>(Killer->GetController()); // Example: Get PC from the killer
    if (!PlayerController) return;

    UPlayerQuestSubsystem* PlayerQS = GetPlayerQuestSystem(PlayerController);
    if (!PlayerQS) return;

    FString EnemyTypeID = TEXT("Spider_Cave"); // ID matching FQuestObjectiveData::TargetID
    FString ObjectiveTypeString = TEXT("Kill"); // Must match the EObjectiveType enum string ("Kill", "Collect", etc.)

    // Update progress
    // Note: HandleKillObjective is a convenience function, RegisterObjectiveProgress is more generic
    // bool bProgressMade = PlayerQS->HandleKillObjective(EnemyTypeID, 1);
    bool bProgressMade = IQuestProgress::Execute_RegisterObjectiveProgress(PlayerQS, ObjectiveTypeString, EnemyTypeID, 1);


    if (bProgressMade)
    {
        UE_LOG(LogTemp, Log, TEXT("Quest progress updated for killing %s"), *EnemyTypeID);
    }
}
```

## Checking if a Quest is Active

Useful for conditional logic, like showing objective markers.

```cpp
#include "Core/PlayerQuestSubsystem.h"
#include "Interfaces/IQuestProgress.h"

bool IsQuestActiveForPlayer(APlayerController* PlayerController, const FString& QuestID)
{
    UPlayerQuestSubsystem* PlayerQS = GetPlayerQuestSystem(PlayerController);
    if (PlayerQS)
    {
        return IQuestProgress::Execute_HasActiveQuest(PlayerQS, QuestID);
    }
    return false;
}

// Example usage:
void AMyObjectiveMarker::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    APlayerController* PC = UGameplayStatics::GetPlayerController(GetWorld(), 0);
    bool bShouldBeVisible = IsQuestActiveForPlayer(PC, MyAssociatedQuestID);

    SetActorHiddenInGame(!bShouldBeVisible);
}
```

## Displaying Available Quests (e.g., for an NPC)

Used to populate the UQuestNPCMenuWidget or a custom UI.

```cpp
#include "Core/QuestManagerSubsystem.h"
#include "Core/PlayerQuestSubsystem.h"
#include "Interfaces/IQuestProvider.h"
#include "Core/QuestDataTypes.h" // For FQuestDefinition

TArray<FQuestDefinition> GetAvailableQuestsFromNPC(APlayerController* PlayerController, const TArray<FString>& NpcOffersQuestIDs)
{
    TArray<FQuestDefinition> AvailableQuests;
    UPlayerQuestSubsystem* PlayerQS = GetPlayerQuestSystem(PlayerController);
    UQuestManagerSubsystem* QuestManager = GetQuestManager();

    if (!PlayerQS || !QuestManager) return AvailableQuests;

    FString PlayerID = PlayerQS->GetPlayerID();
    int32 PlayerLevel = PlayerQS->GetPlayerLevel(); // Assuming PlayerQuestSubsystem tracks this

    for (const FString& QuestID : NpcOffersQuestIDs)
    {
        // Check if the quest definition exists and meets player requirements (level, prereqs)
        if (IQuestProvider::Execute_IsQuestAvailable(QuestManager, QuestID, PlayerID, PlayerLevel))
        {
            // Check if the player doesn't already have it (active or completed - depending on recurrence)
            bool bAlreadyHas = IQuestProgress::Execute_HasActiveQuest(PlayerQS, QuestID);
            // Add check for completed non-recurring quests if needed

            if (!bAlreadyHas)
            {
                FQuestDefinition QuestData;
                if (IQuestProvider::Execute_GetQuestData(QuestManager, QuestID, QuestData))
                {
                    AvailableQuests.Add(QuestData);
                }
            }
        }
    }
    return AvailableQuests;
}
```

## Subscribing to Events (UI Update Example)

In a UI Widget (like QuestTrackerWidget's C++ parent or another manager):

```cpp
#include "Core/PlayerQuestSubsystem.h"

void UMyQuestUIWidget::LinkToPlayerSubsystem(UPlayerQuestSubsystem* PlayerQS)
{
    if (LinkedPlayerQS.IsValid()) // Unbind from old one if necessary
    {
         LinkedPlayerQS->OnQuestUpdated.RemoveAll(this);
         LinkedPlayerQS->OnQuestCompleted.RemoveAll(this);
         // etc.
    }

    LinkedPlayerQS = PlayerQS;

    if (LinkedPlayerQS.IsValid())
    {
        LinkedPlayerQS->OnQuestUpdated.AddUObject(this, &UMyQuestUIWidget::HandleQuestUpdated);
        LinkedPlayerQS->OnQuestCompleted.AddUObject(this, &UMyQuestUIWidget::HandleQuestCompleted);
        // etc.

        // Initial UI refresh
        RefreshQuestDisplay();
    }
}

void UMyQuestUIWidget::HandleQuestUpdated(FString QuestID, FString ObjectiveID)
{
    UE_LOG(LogTemp, Verbose, TEXT("Quest UI received update for Quest %s, Objective %s"), *QuestID, *ObjectiveID);
    // Schedule a refresh or update specific parts of the UI
    RefreshQuestDisplay(); // Or a more targeted update
}

void UMyQuestUIWidget::HandleQuestCompleted(FString QuestID)
{
    UE_LOG(LogTemp, Verbose, TEXT("Quest UI received completion for Quest %s"), *QuestID);
    // Remove quest from active list, maybe show completion effect
    RefreshQuestDisplay();
}

// Remember to store LinkedPlayerQS as a UPROPERTY TWeakObjectPtr<UPlayerQuestSubsystem>
// and call LinkToPlayerSubsystem when the widget is initialized with a player controller.
```

These examples provide a starting point for integrating the Dynamic Quest System into your game's C++ logic. Remember to handle null checks and adapt the logic to your specific game structure.