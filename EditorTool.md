# Dynamic Quest System: Quest Editor Tool

The plugin includes a dedicated editor window built with Slate for creating, editing, and managing quest definitions (`FQuestDefinition`).

## Accessing the Editor

You can open the Quest Editor window in two ways:

1.  **Toolbar:** Click the "Quests" icon added to the main editor toolbar (usually near the Play button).
    ![Toolbar Icon](placeholder_toolbar.png) <!-- Replace with actual image -->
2.  **Menu Bar:** Go to `Window -> Quest Editor`.

## Editor Subsystem (`UDynamicQuestSystemEditorSubsystem`)

The editor window doesn't directly modify the runtime `UQuestManagerSubsystem`. Instead, it interacts with its own backend, the `UDynamicQuestSystemEditorSubsystem`.

*   **Purpose:** Manages a separate copy of the quest data (`QuestDataMap`) specifically for the editor session. This prevents accidental modification of runtime data during editing.
*   **Data Loading:** On editor startup, it attempts to load quest data, prioritizing the *runtime* binary file (`Engine/Binaries/.../Quests/RuntimeQuests.bin`) to provide a relevant starting point. If that fails, it might load from an editor-specific binary (`Saved/Quests/EditorQuests.bin`).
*   **Data Saving:**
    *   `SaveQuestsToBinary()`: Saves the editor's current quest data to a binary file chosen by the user via a "Save As" dialog (defaults to `EditorQuests.bin`). This is for saving the editor's working state.
    *   `LoadQuestsFromBinary()`: Loads quest data into the editor from a binary file chosen by the user via an "Open" dialog.
    *   `ExportQuestsToRuntime()`: **This is the crucial function to get editor changes into the game.** It performs a *safe merge* operation:
        1.  Loads the existing quests from the runtime binary file (`RuntimeQuests.bin`).
        2.  Compares them with the quests currently held by the editor subsystem.
        3.  Adds new quests created in the editor.
        4.  Updates existing quests, preferring the editor's version if changes are detected (it logs conflicts but currently overwrites with the editor version).
        5.  Saves the resulting merged quest list back to the `RuntimeQuests.bin` file, creating a backup (`.previous`) of the old file.

## Editor Window Layout (`FQuestEditorWindow`)

The editor window is composed of several key sections:

1.  **Toolbar:** Buttons for common actions:
    *   `New Quest`: Creates a blank quest definition.
    *   `Save Binary`: Saves the editor's current state to an `EditorQuests.bin` (or user-chosen file).
    *   `Load Binary`: Loads quests into the editor from an `EditorQuests.bin` (or user-chosen file).
    *   `Export to Runtime`: Saves the current editor quests to the runtime binary file used by the game, merging with existing data.

2.  **Quest List Panel (Left):**
    *   **Search Bar:** Filters the quest list by ID, Title, Location, NPC, or Tags.
    *   **Quest List (`SQuestList`):** Displays all loaded quests. Clicking a quest selects it for editing.

3.  **Details Panel (Right - Scrollable):** Contains sections for editing the currently selected quest:
    *   **Basic Information:** Edit `QuestID`, `Title`, `Description`, `QuestGiver`, `LocationName`.
    *   **Requirements & Settings:** Edit `LevelRequirement`, `Difficulty`, `EstimatedTime`, `bIsMainQuest`, `bIsRecurring`/`Cooldown`, `bIsTimeLimited`/`Start`/`End`, `Prerequisites`, `Tags`.
    *   **Dialog & Feedback:** Edit `AcceptDialog`, `CompleteDialog`, `AcceptSoundPath`, `CompleteSoundPath`, `CustomData`.
    *   **Objectives (`SObjectivesList`):**
        *   Lists all objectives for the current quest.
        *   Allows adding/removing objectives.
        *   Provides fields to edit each objective's `Description`, `Type`, `TargetID`, `RequiredAmount`, `bIsOptional`, `Weight`, `Location`/`Radius`, `TimeLimit`, `Hint`.
        *   Includes special controls for `Kill` objectives: `EnemyMeshPath` and `EnemyMeshVariations`.
        *   Buttons to set objective location based on editor camera.
    *   **Rewards (`SRewardsList`):**
        *   Allows editing the quest's `FQuestRewardData`.
        *   Fields for `RewardID`, `Description`, `CurrencyAmount`, `ExperienceAmount`.
        *   Sections to add/remove/edit `ItemRewards` (with quantities) and `ReputationRewards` (Faction ID + Value).
        *   Section for `CustomRewardType` and `CustomRewardData`.
    *   **Update/Delete Buttons:**
        *   `Update Quest`: Saves the changes made in the Details Panel to the selected quest *within the editor subsystem's memory*. (Does not save to file).
        *   `Delete Quest`: Removes the selected quest from the editor subsystem's memory.

## Workflow Summary

1.  Open the Quest Editor. It loads quests from the runtime binary (or editor binary if runtime fails/is empty).
2.  Select a quest from the list on the left, or click "New Quest".
3.  Edit the quest details, objectives, and rewards in the panel on the right.
4.  Click "Update Quest" frequently to save changes *to the editor's internal representation* of the selected quest.
5.  **(Optional)** Click "Save Binary" to save the entire editor state to a file (e.g., `EditorQuests.bin`) for backup or later editing sessions. Use "Load Binary" to restore from such a file.
6.  When ready to make the quests available in the game, click **"Export to Runtime"**. This merges the editor's quests with the existing runtime file and saves the result to `Engine/Binaries/.../Quests/RuntimeQuests.bin`.
7.  Close the editor. You may be prompted to save changes via "Export to Runtime" if you haven't already.

**Important:** Changes made in the editor are *not* reflected in the game until you use **"Export to Runtime"**. Simply clicking "Update Quest" only modifies the data held by the editor window/subsystem for the current session.