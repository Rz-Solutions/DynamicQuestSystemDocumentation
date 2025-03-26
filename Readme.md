# Dynamic Quest System for Unreal Engine

A flexible, modular quest system for Unreal Engine 5.3+ designed for runtime creation, management, and persistence of quests using Subsystems, Interfaces, and multiple data storage options.

## Features

*   **Modular Design:** Uses Game Instance and Local Player Subsystems for clear separation of concerns.
*   **Interface-Driven:** Leverages interfaces (`IQuestProvider`, `IQuestDataSource`, `IQuestProgress`) for easy extension and modification.
*   **Data-Driven:** Define quests using flexible `FQuestDefinition` structs.
*   **Multiple Data Sources:** Comes with a `BinaryQuestDataSource` for saving/loading quests to binary files. Easily extensible for JSON, DataTables, databases, etc.
*   **Runtime Quest Management:** Create, update, delete, accept, abandon, complete, and fail quests during gameplay.
*   **Player Progress Tracking:** Manages individual player quest states, including objective progress and completion status.
*   **Objective System:** Supports various objective types (`Kill`, `Collect`, `Interact`, `Discover`, etc.) and custom objectives. Includes optional time limits per objective.
*   **Reward System:** Define currency, experience, item, reputation, and custom rewards.
*   **Gameplay Integration:** Includes `AQuestTriggerActor` for easy world interaction (NPCs, triggers) and `AQuestObjectiveActor` for representing objectives in the level.
*   **UI Widgets:** Provides basic widgets for a Quest Tracker (`UQuestTrackerWidget`), Notifications (`UQuestNotificationWidget`), NPC interaction menu (`UQuestNPCMenuWidget`), and interaction prompts (`UInteractionPromptWidget`).
*   **Editor Tool:** A dedicated Slate-based Quest Editor window for creating, editing, and managing quests directly within the Unreal Editor.
*   **Persistence:** Saves/loads quest definitions and player progress automatically or manually. Supports async operations for smoother performance.
*   **Configuration:** Settings exposed via `Project Settings -> Plugins -> Dynamic Quest System`.

## Installation

1.  **Marketplace:** Add the plugin to your engine/project via the Epic Games Launcher.
2.  **Source:** Clone or download the plugin source code into your project's `Plugins` directory (e.g., `MyProject/Plugins/DynamicQuestSystem`). Regenerate project files.
3.  **Enable:** Open your project in Unreal Engine, go to `Edit -> Plugins`, find "Dynamic Quest System" under the "Gameplay" category, and ensure it is enabled. Restart the editor if prompted.

See [Setup.md](./Setup.md) for detailed configuration.

## Quick Start

1.  **Enable the Plugin:** (See Installation)
2.  **Open Quest Editor:** Go to `Window -> Quest Editor` in the Unreal Editor menu bar or click the toolbar icon.
3.  **Create a Quest:**
    *   Click "New Quest".
    *   Fill in the `Title`, `Description`.
    *   Add an objective (e.g., Type: `Kill`, TargetID: `Spider`, Required: `5`).
    *   Define rewards (e.g., Currency: `100`, Experience: `50`).
    *   Click "Update Quest".
4.  **Export Quests:** Click "Export to Runtime" in the Quest Editor to save the quest definitions for gameplay.
5.  **Add Quest Giver:**
    *   Place an `AQuestTriggerActor` in your level.
    *   In its Details panel:
        *   Set `NPC Name`.
        *   Add the `QuestID` you created (e.g., "Q001") to the `Available Quest IDs` array.
        *   Ensure `Activation Method` is set to `Interact (NPC Dialog)`.
6.  **Player Interaction:**
    *   Ensure your `PlayerController` class calls `TraceForInteractable` (or similar logic) and `InteractWithNPC` on an input action (like 'F'). See `QuestTestPlayerController.h/.cpp` for an example.
    *   Make sure your `GameMode` sets up the UI widgets (`QuestTrackerWidget`, `QuestNotificationWidget`) and the `UQuestUIManager`. See `QuestTestGameMode.h/.cpp` for an example.
7.  **Play:** Walk up to the `AQuestTriggerActor` and press the interact key. Accept the quest. The quest tracker should appear.

## Core Components

*   **`UQuestManagerSubsystem`:** Manages all quest definitions and global quest data. Handles loading/saving quest definitions.
*   **`UPlayerQuestSubsystem`:** Manages the quest progress for a specific local player. Handles accepting, completing, abandoning quests, and tracking objective progress.
*   **`AQuestTriggerActor`:** An actor placed in the level to allow players to interact with the quest system (e.g., accept quests from an NPC, complete objectives by entering an area).
*   **Quest Editor Window:** A dedicated editor tool for managing quest definitions.

## Documentation Files

*   [README.md](./README.md) - This file: Overview and features.
*   [Setup.md](./Setup.md) - Installation and configuration details.
*   [CoreConcepts.md](./CoreConcepts.md) - Explanation of subsystems, data types, interfaces, and events.
*   [DataManagement.md](./DataManagement.md) - How quest data is stored, loaded, and serialized.
*   [GameplayIntegration.md](./GameplayIntegration.md) - Using Triggers, Objectives, and the Player Subsystem.
*   [UI.md](./UI.md) - Details about the included UI widgets and manager.
*   [EditorTool.md](./EditorTool.md) - Documentation for the Quest Editor window.
*   [Examples.md](./Examples.md) - Code snippets for common use cases.

## License

No license. Code in entirely private.

## Contributing

My brain