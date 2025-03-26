# Dynamic Quest System: Data Management & Persistence

This document explains how the Dynamic Quest System stores, loads, and manages quest data and player progress.

## Data Abstraction: `IQuestDataSource`

The core system doesn't dictate *how* quest definitions are stored. Instead, it relies on the `IQuestDataSource` interface. This allows you to implement different storage backends (Binary, JSON, DataTable, Database, Web Service) without modifying the core `UQuestManagerSubsystem`.

The interface defines three key functions:

*   `LoadQuests(TMap<FString, FQuestDefinition>& OutQuestMap)`: Reads quest definitions from the source and populates the provided map. Returns `true` on success.
*   `SaveQuests(const TMap<FString, FQuestDefinition>& QuestMap)`: Writes the given quest definitions map to the source. Returns `true` on success.
*   `GetSourceName() const`: Returns a descriptive name for the data source (e.g., "Binary File (Quests.bin)").

The active data source is determined by the `DefaultDataSourceClass` setting in `DefaultDynamicQuestSystem.ini` or can be set explicitly via `UQuestManagerSubsystem::SetDataSource`.

## Included Implementation: `UBinaryQuestDataSource`

The plugin provides one concrete implementation: `UBinaryQuestDataSource`.

*   **Functionality:** Saves and loads the entire `TMap<FString, FQuestDefinition>` to/from a single binary file using Unreal's serialization (`FArchive`).
*   **Configuration:**
    *   `FilePath`: The path to the binary file, relative to `FPaths::ProjectSavedDir()`. Default seems to be `Quests/DefaultQuests.bin` (though the `.ini` shows `/Game/Quests`, which is unusual for a save location).
    *   `bCreateBackups`: If true, creates a `.bak` file before overwriting the main save file.
    *   `BackupExtension`: The extension used for backup files (default: `.bak`).
*   **Usage:** This is the default data source if none other is specified in the project settings.

## Binary Serialization Format

Both the quest definitions (`UBinaryQuestDataSource` and internal manager functions) and player progress (`UQuestManagerSubsystem`) use a similar custom binary format:

1.  **Version (`uint32`):** A version number (`CurrentBinaryVersion`) to handle future format changes. The system checks this on load to ensure compatibility.
2.  **Count (`int32`):** The number of items (quests or player quest instances) being serialized.
3.  **Items (Loop `Count` times):**
    *   **Quest Definitions:**
        *   `QuestID` (`FString`): The key for the quest map.
        *   `QuestDefinition` (`FQuestDefinition`): The entire struct, serialized using its `operator<<`.
    *   **Player Progress:**
        *   `PlayerID` (`FString`): Stored once after the version number.
        *   `QuestID` (`FString`): The ID of the player's quest instance.
        *   `RuntimeQuestInstance` (`FRuntimeQuestInstance`): The entire struct, serialized using its `operator<<`.

This format is relatively compact and efficient for loading/saving large amounts of structured data compared to text-based formats like JSON, especially at runtime.

## Quest Definition Persistence

*   **Loading:** At startup, `UQuestManagerSubsystem::Initialize` calls `AsyncLoadQuests`. This first attempts to load from the *runtime binary* (`Engine/Binaries/.../Quests/RuntimeQuests.bin`) via `ImportQuestsFromBinaryAsync`. If that fails (e.g., file doesn't exist), it falls back to loading using the configured `IQuestDataSource` via `LoadQuestsAsync`.
*   **Saving (Editor):** The `UDynamicQuestSystemEditorSubsystem` uses `SaveQuestsToBinary` / `LoadQuestsFromBinary` to manage an editor-specific binary file (e.g., `Saved/Quests/EditorQuests.bin`). This allows editing quests without immediately affecting the runtime version.
*   **Exporting (Editor to Runtime):** The `ExportQuestsToRuntime` function in the editor subsystem is crucial. It performs a *safe merge*:
    1.  Loads the current runtime quests (`RuntimeQuests.bin`).
    2.  Compares them with the quests currently loaded in the editor subsystem (`QuestDataMap`).
    3.  If a quest exists in both and has changed in the editor, it attempts to resolve conflicts (currently just logs them and prefers the editor version).
    4.  Adds any new quests from the editor.
    5.  Saves the merged set back to `RuntimeQuests.bin`, creating a backup (`.previous`) of the old runtime file.

## Player Progress Persistence

*   **Loading:** `UPlayerQuestSubsystem::Initialize` calls `LoadProgress_Implementation`, which in turn calls `UQuestManagerSubsystem::LoadPlayerProgress`. This loads the player's `TMap<FString, FRuntimeQuestInstance>` from a dedicated binary file (e.g., `Saved/Quests/Progress/Player_0.bin`).
*   **Saving:**
    *   **Autosave:** If `AutosaveIntervalSeconds` > 0, `UPlayerQuestSubsystem` periodically calls `SaveProgress_Implementation`.
    *   **Manual/Event-Driven:** `SaveProgress_Implementation` is also called after significant actions like accepting, abandoning, completing, or failing a quest, and updating objective progress.
    *   **Process:** `SaveProgress_Implementation` calls `UQuestManagerSubsystem::SavePlayerProgress` (or `SavePlayerProgressAsync`), which handles the serialization and file writing to the player-specific file, including backup creation if enabled.

## File Locations Summary

*   **Editor Quest Data (Default):** `[Project]/Saved/Quests/EditorQuests.bin` (Used by the Quest Editor window for its working copy). File location can be changed via Save As/Load As dialogs.
*   **Runtime Quest Data (Shipped):** `[Engine]/Binaries/[Platform]/Quests/RuntimeQuests.bin` (This is the file the game actually loads definitions from at runtime). Exported via the "Export to Runtime" button in the editor.
*   **Player Progress Data:** `[Project]/Saved/Quests/Progress/[PlayerID].bin` (e.g., `Player_0.bin`).

**Note:** The `DefaultQuestsPath` in the `.ini` seems misleadingly named/defaulted in the provided code (`/Game/Quests`). It's typically used by `UBinaryQuestDataSource` relative to the `Saved` directory, unless `GetFullPath` is overridden or a different data source is used. The editor subsystem explicitly manages `EditorQuests.bin` and the runtime export targets the Engine binaries directory.