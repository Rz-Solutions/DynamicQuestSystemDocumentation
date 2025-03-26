# Dynamic Quest System: Core Concepts

This document explains the fundamental ideas and components that make up the Dynamic Quest System.

## Overview

The system is built around a few key concepts:

1.  **Central Management:** A global subsystem (`QuestManagerSubsystem`) manages all quest *definitions*.
2.  **Player-Specific Progress:** A per-player subsystem (`PlayerQuestSubsystem`) tracks the *state* of quests for each player.
3.  **Data Abstraction:** Interfaces (`IQuestDataSource`, `IQuestProvider`) decouple the core logic from how quests are stored and accessed.
4.  **Structured Data:** Quests, objectives, and rewards are defined using specific C++ structs (`FQuestDefinition`, etc.).
5.  **Event-Driven Updates:** Delegates are used to broadcast quest-related events for UI updates or other gameplay logic.

## Subsystems

Subsystems are automatically managed instances that provide easy access to functionality.

### `UQuestManagerSubsystem` (Game Instance Subsystem)

*   **Purpose:** Acts as the central database for all available quest definitions (`FQuestDefinition`). It is responsible for loading these definitions from a data source at startup and potentially saving them back (primarily in the editor context).
*   **Scope:** Exists once per game instance. Accessible globally.
*   **Key Responsibilities:**
    *   Loading quest definitions from the configured `IQuestDataSource`.
    *   Providing access to quest definitions (`GetQuestData`, `GetAllQuests`).
    *   Validating quest availability based on prerequisites, level, cooldowns (`IsQuestAvailable`).
    *   Creating, updating, and deleting quest *definitions* (primarily used by the editor tool).
    *   Saving quest definitions back to a data source.
    *   Managing player progress save/load operations (delegates actual file I/O).
    *   Generating unique Quest IDs.
    *   Exporting/Importing quest data to/from binary formats (for runtime use or editor backup).
*   **Access:** `GetGameInstance()->GetSubsystem<UQuestManagerSubsystem>()`

### `UPlayerQuestSubsystem` (Local Player Subsystem)

*   **Purpose:** Tracks the progress and state (`FRuntimeQuestInstance`) of all quests for a *specific* local player. Handles the gameplay interactions related to quests for that player.
*   **Scope:** Exists once per local player.
*   **Key Responsibilities:**
    *   Accepting new quests (`AcceptQuest`).
    *   Abandoning active quests (`AbandonQuest`).
    *   Completing quests (`CompleteQuest`) and triggering reward granting (via `QuestManagerSubsystem`).
    *   Failing quests (`FailQuest`).
    *   Registering progress towards objectives (`RegisterObjectiveProgress`, `HandleKillObjective`).
    *   Providing the player's current quest states (`GetActiveQuests`, `GetCompletedQuests`, `HasActiveQuest`).
    *   Tracking/Untracking quests for UI display (`TrackQuest`, `UntrackQuest`).
    *   Loading and saving its own state via `QuestManagerSubsystem`.
    *   Broadcasting events when the player's quest state changes (`OnQuestUpdated`, `OnQuestCompleted`, etc.).
*   **Access:** `GetLocalPlayer()->GetSubsystem<UPlayerQuestSubsystem>()`

## Data Structures (`QuestDataTypes.h`)

These structs define the information that makes up quests.

*   **`FQuestDefinition`:** The static definition of a quest. Contains:
    *   `QuestID`: Unique identifier (e.g., "Q001", "MainStory_FindKey").
    *   `Title`, `Description`: Display text.
    *   `Objectives`: An array of `FQuestObjectiveData`.
    *   `Reward`: An `FQuestRewardData` struct.
    *   `LevelRequirement`: Minimum player level.
    *   `PrerequisiteQuestIDs`: List of other `QuestID`s that must be completed first.
    *   `bIsRecurring`, `RecurringCooldownHours`: Settings for repeatable quests.
    *   `QuestTags`: `FName` tags for categorization/filtering.
    *   `QuestGiver`, `LocationName`, `QuestStartLocation`: Contextual info.
    *   `Difficulty`, `EstimatedTimeMinutes`, `bIsMainQuest`: Metadata.
    *   `bIsTimeLimited`, `AvailabilityStartTime`, `AvailabilityEndTime`: For event quests.
    *   `AcceptDialog`, `CompleteDialog`: Text for interactions.
    *   `AcceptSoundPath`, `CompleteSoundPath`: Asset paths for audio feedback.
    *   `CustomData`: A string field for arbitrary extra data.

*   **`FQuestObjectiveData`:** Defines a single task within a quest. Contains:
    *   `ObjectiveID`: Unique identifier within the quest (e.g., "KillSpiders", "CollectHerbs").
    *   `Description`: Display text for the objective.
    *   `RequiredAmount`: How many times the objective must be completed (e.g., kill 5 spiders).
    *   `ObjectiveType`: An `EObjectiveType` enum value (`Kill`, `Collect`, `Interact`, etc.).
    *   `TargetID`: Specific identifier related to the type (e.g., "Spider_Cave", "Herb_BlueFlower", "Lever_MainGate"). Used for matching progress events.
    *   `bIsOptional`: Whether this objective is required for quest completion.
    *   `WeightInCompletion`: How much this objective contributes to the overall quest percentage (0.0 to 1.0).
    *   `Location`, `Radius`: Optional world location/area for the objective.
    *   `TimeLimit`: Optional time limit in seconds for this specific objective.
    *   `Hint`: Optional help text.
    *   `EnemyMeshPath`, `EnemyMeshVariations`: Used by `QuestObjectiveActor` for spawning specific enemies.

*   **`FQuestRewardData`:** Defines the rewards for completing a quest. Contains:
    *   `CurrencyAmount`, `ExperienceAmount`.
    *   `ItemRewards`: Array of item IDs/paths.
    *   `ItemQuantities`: Array corresponding to `ItemRewards`.
    *   `ReputationRewards`: Map of Faction ID (`FString`) to reputation change (`int32`).
    *   `CustomRewardType`, `CustomRewardData`: For game-specific reward logic.

*   **`FRuntimeQuestInstance`:** Represents the dynamic state of a specific quest for a specific player. Contains:
    *   `QuestID`: Links to the `FQuestDefinition`.
    *   `CurrentState`: `EQuestStateType` (`NotStarted`, `InProgress`, `Completed`, `Failed`).
    *   `ObjectiveProgress`: Map of `ObjectiveID` (`FString`) to current progress count (`int32`).
    *   `StartTime`, `LastCompletionTime`, `FailTime`, `CompletionDuration`: Timestamps for tracking.
    *   `AttemptCount`, `CompletionCount`: For tracking retries/repeats.
    *   `ObjectiveTimeRemaining`: Map of `ObjectiveID` to remaining time for timed objectives.
    *   `CustomRuntimeData`: String for arbitrary dynamic data specific to this player's instance.

*   **Enums:** Define discrete states and types:
    *   `EQuestStateType`: The current status of a player's quest instance.
    *   `EObjectiveType`: The category of an objective task.
    *   `ERewardType`: (Seems unused in `FQuestRewardData`, which uses specific fields instead).
    *   `EQuestDifficulty`: Difficulty rating for metadata/UI.

## Interfaces

Interfaces define contracts for different parts of the system, allowing for customization and extension.

*   **`IQuestProvider`:** Defines how to *retrieve* quest definition data.
    *   Implemented by: `UQuestManagerSubsystem` (main implementation), `AQuestEditor` (deprecated).
    *   Key Methods: `GetQuestData`, `GetAllQuests`, `IsQuestAvailable`.
*   **`IQuestDataSource`:** Defines how to *persist* (load and save) quest definitions.
    *   Implemented by: `UBinaryQuestDataSource`. You can create your own (e.g., `UJsonQuestDataSource`, `UDataTableQuestDataSource`).
    *   Key Methods: `LoadQuests`, `SaveQuests`, `GetSourceName`.
*   **`IQuestProgress`:** Defines the contract for managing a player's quest progression.
    *   Implemented by: `UPlayerQuestSubsystem`.
    *   Key Methods: `AcceptQuest`, `AbandonQuest`, `CompleteQuest`, `FailQuest`, `RegisterObjectiveProgress`, `GetActiveQuests`, `HasActiveQuest`.
*   **`IQuestObjectiveHandler`:**.
    *   **Purpose:** Designed to allow different objects or systems to process gameplay events and determine if they contribute to specific objective types. A theoretical `KillObjectiveHandler` might subscribe to enemy death events and call `RegisterObjectiveProgress` on the `PlayerQuestSubsystem`.
    *   Key Methods: `CanHandleObjectiveType`, `ProcessEvent`, `GetHandlerName`.

## Event System (`QuestEvents.h` & Delegates)

The system uses `DECLARE_MULTICAST_DELEGATE` and `DECLARE_DYNAMIC_MULTICAST_DELEGATE` to notify other parts of the engine/game about significant quest events.

*   **`UQuestManagerSubsystem` Delegates:**
    *   `FOnQuestDataAdded`: Broadcast when a new quest *definition* is loaded/created.
    *   `FOnQuestStateChanged`: Broadcast by the manager (often triggered by player actions relayed from `PlayerQuestSubsystem`) when a player's quest instance changes state (e.g., `InProgress` -> `Completed`).
*   **`UPlayerQuestSubsystem` Delegates:**
    *   `FOnPlayerQuestUpdatedNative`/`FOnPlayerQuestUpdatedDynamic`: Broadcast when an objective's progress updates for the player. (Provides QuestID, ObjectiveID).
    *   `FOnPlayerQuestCompletedNative`/`FOnPlayerQuestCompletedDynamic`: Broadcast when the player completes a quest. (Provides QuestID).
    *   `FOnPlayerQuestFailed`: Broadcast when a quest fails for the player.
    *   `FOnPlayerQuestAbandoned`: Broadcast when the player abandons a quest.
    *   `FOnPlayerQuestTrackingChanged`: Broadcast when the player starts or stops tracking a quest in the UI.
*   **`FQuestEventManager` (Global Event Hub - `QuestEvents.h`):**
    *   Provides static access to global delegates mirroring player actions. This allows systems *outside* the `PlayerQuestSubsystem` to react easily without needing direct references.
    *   `OnQuestAvailable`, `OnQuestAccepted`, `OnObjectiveProgress`, `OnObjectiveCompleted`, `OnQuestCompleted`, `OnQuestFailed`, `OnQuestAbandoned`, `OnQuestRewardsGranted`.

This combination of subsystems, data structures, interfaces, and events creates a decoupled and extensible quest system foundation.
