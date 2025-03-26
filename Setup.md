# Dynamic Quest System: Setup and Configuration

This document details the steps required to install and configure the Dynamic Quest System plugin.

## Installation

1.  **Marketplace:**
    *   Find the "Dynamic Quest System" plugin on the Unreal Engine Marketplace.
    *   Add it to your engine version or directly to your project via the Epic Games Launcher.
    *   Open your project.

2.  **From Source:**
    *   Clone or download the plugin's source code.
    *   Create a `Plugins` directory in your project's root folder if it doesn't exist.
    *   Copy the entire plugin folder (e.g., `DynamicQuestSystem`) into the `Plugins` directory. The structure should look like `MyProject/Plugins/DynamicQuestSystem/DynamicQuestSystem.uplugin`.
    *   Right-click your project's `.uproject` file and select "Generate Visual Studio project files".
    *   Open your project's solution (`.sln`) and build it.
    *   Open your project in the Unreal Editor.

## Enabling the Plugin

1.  In the Unreal Editor, go to `Edit -> Plugins`.
2.  Search for "Dynamic Quest System".
3.  Ensure the checkbox next to the plugin name is checked (Enabled).
4.  If prompted, restart the Unreal Editor.

Your project's `.uproject` file should now include the plugin under the `Plugins` section:

```json
{
  // ... other project settings
  "Plugins": [
    {
      "Name": "DynamicQuestSystem",
      "Enabled": true
    }
    // ... other plugins
  ]
}
```

## Configuration (DefaultDynamicQuestSystem.ini)

You can customize the plugin's behavior by creating or modifying the `Config/DefaultDynamicQuestSystem.ini` file in your project directory.

Here are the available settings, defined in `UDynamicQuestSystemSettings`:

```ini
[/Script/DynamicQuestSystem.DynamicQuestSystemSettings]

; Default path within the Saved directory for the binary quest data file used by BinaryQuestDataSource.
; Relative to FPaths::ProjectSavedDir().
DefaultQuestsPath=/Game/Quests ; Note: This seems incorrect in the provided .ini, likely should be relative to Saved dir, e.g., "Quests/RuntimeQuests.bin" or similar.

; Maximum number of quests a player can have active (InProgress) simultaneously.
MaxActiveQuestsPerPlayer=10

; Interval in seconds for automatically saving player progress. Set to 0.0 to disable autosave.
AutosaveIntervalSeconds=300.0

; If true, the BinaryQuestDataSource and QuestManagerSubsystem will create backup files (.bak) before overwriting save data.
bCreateBackups=true

; Specifies the default IQuestDataSource class to use for loading/saving quest definitions if none is explicitly set.
; Must be a subclass of UObject implementing IQuestDataSource.
DefaultDataSourceClass=/Script/DynamicQuestSystem.BinaryQuestDataSource

; Enables the Quest Tracker UI widget. Requires a valid widget class to be set in the GameMode or elsewhere.
bEnableQuestTracker=true

; Enables the Quest Notification UI widget. Requires a valid widget class to be set in the GameMode or elsewhere.
bEnableQuestNotifications=true

; Maximum number of quests displayed in the Quest Tracker UI at one time.
MaxTrackedQuests=5

; Duration in seconds for which quest notifications are displayed on screen.
NotificationDisplayTime=3.0

; If true, enables various debug visualizations and logging (e.g., QuestTriggerActor debug info).
bShowDebugInfo=false

; If true, enables detailed logging of quest events (Accept, Complete, Progress, etc.).
bLogQuestEvents=false
```

Example `Config/DefaultDynamicQuestSystem.ini`:

```ini
[/Script/DynamicQuestSystem.DynamicQuestSystemSettings]
DefaultQuestsPath=Quests/MyGameQuests.bin ; Save quests to Saved/Quests/MyGameQuests.bin
MaxActiveQuestsPerPlayer=20
AutosaveIntervalSeconds=120.0
DefaultDataSourceClass=/Script/MyGame.MyCustomJsonDataSource ; Using a custom data source
bEnableQuestTracker=true
MaxTrackedQuests=3
NotificationDisplayTime=5.0
bLogQuestEvents=true
```

## Build Dependencies (DynamicQuestSystem.Build.cs)

If building the plugin or your project from source, ensure the necessary modules are included. The plugin itself depends on:

**Public**: Core, CoreUObject, Engine, InputCore, UMG, Slate, SlateCore, DeveloperSettings, ApplicationCore

**Private**: AssetRegistry, Settings, Projects

**Editor-Only**: UnrealEd, AssetTools, EditorStyle, EditorSubsystem, LevelEditor, Json, JsonUtilities, PropertyEditor, ToolMenus, EditorFramework, DesktopPlatform

Your game module might need to add DynamicQuestSystem to its dependencies in its Build.cs file if you are accessing plugin classes directly:

```cpp
// In YourGame.Build.cs
PublicDependencyModuleNames.AddRange(new string[] {
    "Core",
    "CoreUObject",
    "Engine",
    "InputCore",
    "EnhancedInput", // Or your input system
    "UMG",
    "DynamicQuestSystem" // Add this line
});
```

## Next Steps

- Understand the Core Concepts.
- Learn about Data Management.
- Integrate the system into your gameplay using Gameplay Integration.
- Set up the UI.
- Use the Editor Tool to create quests.