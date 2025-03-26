# Dynamic Quest System: Gameplay Integration

This document explains how to integrate the quest system into your game world and player logic using the provided actors and subsystems.

## Quest Trigger Actor (`AQuestTriggerActor`)

This actor serves as the primary interaction point between the player/world and the quest system. It can represent NPCs, interactive objects, or trigger volumes.

*   **Purpose:** To initiate quest actions based on player interaction or proximity.
*   **Key Properties:**
    *   **`NPCName`, `GreetingDialog`, `FarewellDialog`:** Used when `ActivationMethod` is `Interact` to provide context for the NPC menu.
    *   **`AvailableQuestIDs` (`TArray<FString>`):** A list of `QuestID`s that this actor can offer to the player. Used primarily with the `Interact` activation method.
    *   **`QuestID` (`FString`):** The specific Quest ID this trigger acts upon *if* it's not offering quests (e.g., for completing an objective, abandoning, etc.).
    *   **`TriggerType` (`EQuestTriggerType`):** Defines the action performed when triggered (e.g., `AcceptQuest`, `CompleteObjective`, `CompleteQuest`).
    *   **`ActivationMethod` (`ETriggerActivationMethod`):** How the trigger is activated:
        *   `Overlap`: Action occurs when the player enters the `TriggerBox`.
        *   `Interact (NPC Dialog)`: Action (or showing the quest menu if `AvailableQuestIDs` is populated) occurs when the player is within range and presses the interact key (requires player controller logic).
        *   `Automatic`: Action occurs shortly after `BeginPlay`.
    *   **`ObjectiveID`, `ObjectiveType`, `TargetID`:** Used when `TriggerType` is `CompleteObjective` to specify *which* objective progress to register.
    *   **`ProgressAmount` (`int32`):** How much progress to add when completing an objective trigger.
    *   **`bCanBeUsedMultipleTimes`, `CooldownTime`, `bDeactivateAfterUse`:** Control trigger re-usability.
    *   **`InteractionWidgetComponent`, `InteractionWidgetClass`:** Displays a UI prompt (like "[F] Interact") when the player is in range and interaction is possible. Uses `UInteractionPromptWidget` by default.
    *   **`ObjectiveSpawnInfo` (`TArray<FQuestObjectiveSpawnInfo>`):** Defines actors (like enemies or collectibles) to spawn when a specific quest offered by this trigger is accepted.

*   **Workflow:**
    1.  Place `AQuestTriggerActor` in the level.
    2.  Configure its properties based on the desired function (Offer quests? Complete an objective? Trigger automatically?).
    3.  If offering quests (`Interact` + `AvailableQuestIDs`), the actor will show the `UQuestNPCMenuWidget` when interacted with.
    4.  If set to `CompleteObjective` (via `Overlap` or `Interact`), it will call `RegisterObjectiveProgress` on the `PlayerQuestSubsystem`.
    5.  Other `TriggerType`s call corresponding `PlayerQuestSubsystem` functions (`AcceptQuest`, `AbandonQuest`, etc.).

## Quest Objective Actor (`AQuestObjectiveActor`)

This actor is typically spawned *by* the system (often via `AQuestTriggerActor` or `UQuestObjectiveFactory`) to represent a physical objective in the world.

*   **Purpose:** To represent kill targets, collectible items, interaction points, or discovery zones related to a specific quest objective.
*   **Key Properties:**
    *   **`QuestID`, `ObjectiveID`:** Links this actor instance to a specific quest objective.
    *   **`DisplayName`:** Text used for UI markers.
    *   **`ObjectiveType` (`ESpawnedObjectiveType`):** The type of objective this actor represents (determines its behavior).
    *   **`TargetID`:** Identifier used for progress tracking (e.g., the type of enemy).
    *   **`RequiredAmount`:** How many of these *types* of objectives are needed (often 1 for a single spawned actor, but managed collectively for kill/collect counts).
    *   **`ActorToSpawn` (`TSubclassOf<AActor>`):** (Used mainly for `Kill` objectives) The class of actor (e.g., enemy Pawn) to spawn nearby.
    *   **`EnemyMeshPath`, `EnemyMeshVariations`:** Asset paths used to override the mesh of the spawned actor (especially for `Kill` objectives if `ActorToSpawn` doesn't have the desired visual).
    *   **`bCanInteract`, `bAutoCompleteOnOverlap`:** Control how the player completes the objective (Interact key vs. Overlap).
    *   **`MeshComponent`, `InteractionSphere`, `ObjectiveMarkerWidget`:** Visual and interaction components.

*   **Workflow:**
    1.  Typically spawned by `UQuestObjectiveFactory` or `AQuestTriggerActor` when a quest is accepted.
    2.  Configured based on the `FQuestObjectiveData` from the `FQuestDefinition`.
    3.  Handles its own interaction logic (overlap/interaction) or kill detection (via spawned actors).
    4.  Calls `UpdateObjectiveProgress` (which uses `PlayerQuestSubsystem` or `QuestManagerSubsystem`) when progress is made.
    5.  Calls `CompleteObjective` when its specific part is done (which might notify the owning `AQuestTriggerActor` or destroy itself).
    6.  For `Kill` objectives, it spawns the `ActorToSpawn`, potentially applies meshes, adds collision, and listens for interactions/destruction to call `HandleEntityKilled`.

## Quest Objective Factory (`UQuestObjectiveFactory`)

A helper class (currently static functions, could be an object) for creating `AQuestObjectiveActor` instances based on quest data.

*   **Purpose:** Centralizes the logic for spawning and configuring objective actors.
*   **Key Functions:**
    *   `CreateObjectiveActor`: Spawns a single `AQuestObjectiveActor` for a given `FQuestObjectiveData`.
    *   `CreateAllObjectivesForQuest`: Iterates through a `FQuestDefinition`'s objectives and spawns actors for each.
    *   `GetMeshForObjectiveType`, `GetMaterialForObjectiveType`, `GetActorClassForObjectiveType`: Helper functions to determine appropriate assets/classes based on the objective data (logic may need refinement, especially `GetActorClassForObjectiveType`).

## Player Interaction (`UPlayerQuestSubsystem`)

The player's interaction with the quest system happens primarily through the `UPlayerQuestSubsystem`.

*   **Getting the Subsystem:** In C++, use `PlayerController->GetLocalPlayer()->GetSubsystem<UPlayerQuestSubsystem>()`. Ensure the Local Player and Controller are valid.
*   **Common Tasks:**
    *   **Accepting:** Call `AcceptQuest(QuestID)` after the player confirms in a UI or interacts with a trigger.
    *   **Abandoning:** Call `AbandonQuest(QuestID)` from a quest log UI.
    *   **Updating Progress:** Call `RegisterObjectiveProgress(ObjectiveTypeString, TargetID, Count)` when a relevant gameplay event occurs (e.g., picking up an item, killing an enemy). `ObjectiveTypeString` should match the enum string (e.g., "Kill", "Collect").
    *   **Checking State:** Use `HasActiveQuest(QuestID)`, `GetActiveQuests()`, `GetCompletedQuestIDs()` for UI or gameplay logic.
    *   **Tracking:** Use `TrackQuest(QuestID)` and `UntrackQuest(QuestID)` to manage which quests appear in the `QuestTrackerWidget`.

See [Examples.md](./Examples.md) for code snippets.