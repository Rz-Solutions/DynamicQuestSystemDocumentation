# Dynamic Quest System: Blueprint Examples

This document provides practical examples of how to interact with the Dynamic Quest System using Blueprints. For C++ examples, see [Examples.md](./Examples.md).

## Accessing Quest Subsystems in Blueprint

The quest system is primarily driven by two subsystems that can be accessed from any Blueprint:

### Getting the Quest Manager Subsystem

The global Quest Manager Subsystem handles all quest definitions and is accessible from any Blueprint.

```
/* Description: This sequence gets a reference to the global Quest Manager Subsystem */
1. Get Game Instance
2. Get Subsystem (Class: QuestManagerSubsystem)
3. Store reference or use directly
```

### Getting the Player Quest Subsystem

The Player Quest Subsystem tracks an individual player's progress and is specific to each local player.

```
/* Description: This sequence gets a reference to the Player Quest Subsystem for a specific player */
1. Get Player Controller (or Get Owning Player from a Character/Pawn)
2. Get Local Player
3. Get Subsystem (Class: PlayerQuestSubsystem)
4. Store reference or use directly
```

## Creating a Character Blueprint with Quest Interaction

Here's how to set up a player character that can interact with quest triggers:

```
/* Description: Set up character interaction with quest triggers */

// Input Action Setup
1. Create Input Action "Interact" (Type: Button)
2. Add to Input Mapping Context

// Blueprint Event Graph
1. Input Action Interact (Triggered)
2. Line Trace for Interactable
   - Start: Camera Location
   - End: Camera Location + (Camera Forward Vector * 300)
   - Trace Complex: True
   - Actor to Ignore: Self
   - Break Hit Result
3. Branch (IsValid(Hit Actor))
4. True Path:
   - Cast To QuestTriggerActor
   - Branch (Cast Succeeded)
   - True Path: 
     - Call InteractWithNPC on QuestTriggerActor
```

## Creating a Quest Giver NPC

Use a Quest Trigger Actor to create an NPC that offers quests to the player:

```
/* Description: Configure a Quest Trigger Actor as an NPC Quest Giver */

// In the Level Editor:
1. Drag AQuestTriggerActor into the level
2. In Details Panel:
   - Set NPCName = "Village Elder"
   - Set GreetingDialog = "Greetings, traveler. I have tasks that need attention."
   - Set FarewellDialog = "Return to me when your task is complete."
   - Set TriggerType = "Accept Quest"
   - Set ActivationMethod = "Interact (NPC Dialog)"
   - Add to Available Quest IDs: [
     "Q001_KillSpiders",
     "Q002_CollectHerbs"
   ]
   - Set Interaction Distance = 200
```

## Listening for Quest Events in Blueprint

You can create Blueprint actors that respond to quest events through the event system:

```
/* Description: Blueprint set up to respond to quest events */

// In a Blueprint Actor's Event Graph:

// 1. Subscribe to Events:
Event Begin Play
|
+-- Get Player Controller (Index: 0)
|   |
|   +-- Get Local Player
|       |
|       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |
|           +-- Assign to variable "PlayerQuestSystem"
|           |
|           +-- Bind Event to On Player Quest Updated Dynamic
|           |
|           +-- Bind Event to On Player Quest Completed Dynamic

// 2. Quest Updated Event Handler:
Event On Player Quest Updated (Quest ID, Objective ID)
|
+-- Branch (Quest ID == "Q001_KillSpiders")
    |
    +-- True: Play Sound/Effect for Spider Quest Progress

// 3. Quest Completed Event Handler:
Event On Player Quest Completed (Quest ID)
|
+-- Branch (Quest ID == "Q001_KillSpiders")
    |
    +-- True: Play Completion Effect/Reward Animation
```

## Tracking Quest Progress with Blueprints

Here's how to register progress for different objective types:

### Registering Kill Objective Progress

```
/* Description: Register progress when an enemy is killed */

// In Enemy Character's Event Graph:
Event On Death (Killer)
|
+-- Cast To PlayerCharacter (Killer)
|   |
|   +-- Branch (Cast Succeeded)
|       |
|       +-- True Path:
|           |
|           +-- Get Player Controller
|           |   |
|           |   +-- Get Local Player
|           |       |
|           |       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |           |
|           |           +-- Cast To IQuestProgress
|           |               |
|           |               +-- Execute Register Objective Progress
|           |                   - Objective Type String: "Kill"
|           |                   - Target ID: "Spider_Cave" (or variable from the enemy)
|           |                   - Progress Amount: 1
```

### Registering Collect Objective Progress

```
/* Description: Register progress when an item is collected */

// In Collectible Actor's Event Graph:
Event On Collected (Collector)
|
+-- Get Player Controller from Collector
|   |
|   +-- Get Local Player
|       |
|       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |
|           +-- Cast To IQuestProgress
|               |
|               +-- Execute Register Objective Progress
|                   - Objective Type String: "Collect"
|                   - Target ID: "Herb_BlueFlower" (or variable from collectible)
|                   - Progress Amount: 1

// After registering progress:
+-- Destroy Actor (Self)
```

## Creating a Custom Objective Handler

You can create custom Blueprint handlers for special objective types:

```
/* Description: A custom objective handler component for weather-related objectives */

// Create a Blueprint Component class that implements IQuestObjectiveHandler:

// In Component Event Graph:
Event Begin Play
|
+-- Get Owner
|   |
|   +-- Get Player Controller
|       |
|       +-- Get Local Player
|           |
|           +-- Get Subsystem (Class: PlayerQuestSubsystem)
|               |
|               +-- Register Custom Objective Handler (Self)

// Interface Functions:
// 1. Can Handle Objective Type:
Implementation of Can Handle Objective Type (Objective Type)
|
+-- Return (Objective Type == "WeatherSurvival")

// 2. Process Event:
Implementation of Process Event (Event Type, Target ID, Event Value)
|
+-- Branch (Event Type == "WeatherChange")
    |
    +-- True Path:
        |
        +-- Get Player Controller 
        |   |
        |   +-- Get Local Player
        |       |
        |       +-- Get Subsystem (Class: PlayerQuestSubsystem)
        |           |
        |           +-- Cast To IQuestProgress
        |               |
        |               +-- Execute Register Objective Progress
        |                   - Objective Type String: "WeatherSurvival"
        |                   - Target ID: Target ID
        |                   - Progress Amount: Event Value

// 3. Get Handler Name:
Implementation of Get Handler Name
|
+-- Return "Weather System Objective Handler"
```

## Setting Up UI Widgets in Blueprint

Here's how to initialize the UI system in your Game Mode Blueprint:

```
/* Description: Initialize quest UI widgets in Game Mode Blueprint */

// In GameMode Blueprint Event Graph:
Event Begin Play
|
+-- Get Player Controller (Index: 0)
|   |
|   +-- Create Widget (Class: WBP_QuestTracker)
|   |   |
|   |   +-- Add to Viewport (Z-Order: 5)
|   |   |
|   |   +-- Assign to variable "QuestTrackerWidget"
|   |
|   +-- Create Widget (Class: WBP_QuestNotification)
|       |
|       +-- Add to Viewport (Z-Order: 20)
|       |
|       +-- Assign to variable "QuestNotificationWidget"
|
+-- Create Object (Class: QuestUIManager)
    |
    +-- Call Initialize
        - Tracker Widget: QuestTrackerWidget
        - Notification Widget: QuestNotificationWidget
        
// Store references to these widgets as variables
```

## Creating a Custom Quest Reward Handler

You can create custom reward processors for specialized quest rewards:

```
/* Description: A Blueprint function library for handling custom rewards */

// Create a Blueprint Function Library:

// Function: Process Custom Reward
Inputs:
- Custom Reward Type (String)
- Custom Reward Data (String)
- Player Controller (Player Controller Reference)

// Implementation:
Process Custom Reward(Custom Reward Type, Custom Reward Data, Player Controller)
|
+-- Switch on String (Custom Reward Type)
    |
    +-- Case "UnlockAbility":
    |   |
    |   +-- Parse JSON (Custom Reward Data)
    |   |   |
    |   |   +-- Extract "AbilityID"
    |   |
    |   +-- Cast To MyCharacter (Player Controller→Get Pawn)
    |       |
    |       +-- Call Unlock Ability (AbilityID)
    |
    +-- Case "UnlockArea":
        |
        +-- Parse JSON (Custom Reward Data)
        |   |
        |   +-- Extract "AreaTag"
        |
        +-- Get Game Instance
            |
            +-- Cast To MyGameInstance
                |
                +-- Call Unlock Map Area (AreaTag)
```

## Checking Quest Availability in Blueprint

Use this pattern to check if a quest is available before offering it:

```
/* Description: Check if a quest is available to a player */

// In an NPC Blueprint or other logic:
Function: Is Quest Available To Player
Inputs:
- Quest ID (String)
- Player Controller (Player Controller Reference)
Returns: Boolean

// Implementation:
Is Quest Available To Player(Quest ID, Player Controller)
|
+-- Get Game Instance
|   |
|   +-- Get Subsystem (Class: QuestManagerSubsystem)
|       |
|       +-- Cast To IQuestProvider
|           |
|           +-- Get Player Controller → Get Local Player
|           |   |
|           |   +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |       |
|           |       +-- Get Player ID
|           |
|           +-- Get Player Level (from your level system or default to 1)
|           |
|           +-- Execute Is Quest Available
|               - Quest ID: Quest ID
|               - Player ID: Player ID
|               - Player Level: Player Level
|
+-- Return Result
```

## Creating a Simple Quest Log Widget

Here's a simplified approach to creating a quest log widget:

```
/* Description: Create a simple quest log widget */

// In a Widget Blueprint:

// 1. Widget Components Setup:
- Add a ScrollBox named "QuestListScrollBox"
- Create a Quest Entry widget with:
  - Text blocks for Title, Description
  - Progress Bar for completion
  - Button for "Track/Untrack"
  - Button for "Abandon"

// 2. Widget Blueprint Functions:
Function: Refresh Quest Log
|
+-- Get Owning Player
|   |
|   +-- Get Local Player
|       |
|       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |
|           +-- Get Active Quests
|               |
|               +-- Clear QuestListScrollBox
|               |
|               +-- For Each Active Quest:
|                   |
|                   +-- Get Game Instance
|                   |   |
|                   |   +-- Get Subsystem (Class: QuestManagerSubsystem)
|                   |       |
|                   |       +-- Cast To IQuestProvider
|                   |           |
|                   |           +-- Execute Get Quest Data
|                   |               |
|                   |               +-- Create Quest Entry Widget
|                   |                   |
|                   |                   +-- Set Quest Data
|                   |                   |
|                   |                   +-- Set On Track Button Clicked
|                   |                   |
|                   |                   +-- Set On Abandon Button Clicked
|                   |                   |
|                   |                   +-- Add to QuestListScrollBox

// 3. Event Graph:
Event Construct
|
+-- Call Refresh Quest Log

// 4. Track Button Handler:
On Track Button Clicked(Quest ID)
|
+-- Get Owning Player
    |
    +-- Get Local Player
        |
        +-- Get Subsystem (Class: PlayerQuestSubsystem)
            |
            +-- Cast To IQuestProgress
                |
                +-- Execute Track Quest (Quest ID)

// 5. Abandon Button Handler:
On Abandon Button Clicked(Quest ID)
|
+-- Get Owning Player
    |
    +-- Get Local Player
        |
        +-- Get Subsystem (Class: PlayerQuestSubsystem)
            |
            +-- Cast To IQuestProgress
                |
                +-- Execute Abandon Quest (Quest ID)
                |
                +-- Refresh Quest Log
```

## Creating a Custom Objective Mini-Game

Here's how to create a custom objective that requires a mini-game to complete:

```
/* Description: A custom mini-game objective */

// 1. Create a Mini-Game Actor Blueprint:
- Variables:
  - QuestID (String)
  - ObjectiveID (String)

// 2. Event Graph:
Event Begin Play
|
+-- Set Up Mini-Game

// 3. Mini-Game Success Function:
On Mini-Game Completed Successfully(Score)
|
+-- Get Player Controller (Player using the mini-game)
|   |
|   +-- Get Local Player
|       |
|       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |
|           +-- Cast To IQuestProgress
|               |
|               +-- Execute Register Objective Progress
|                   - Objective Type String: "MiniGame"
|                   - Target ID: Mini-Game ID (Variable)
|                   - Progress Amount: 1

// 4. To use this in a quest:
// In the Quest Editor, create an objective with:
// - ObjectiveType: Custom
// - TargetID: Match the Mini-Game ID
// - CustomData: Include any configuration needed
```

## Setting Up Level Streaming Based on Quest Completion

Control level streaming and access based on quest progression:

```
/* Description: Control level streaming based on quest completion */

// In Game Mode or Level Blueprint:

// Function: Update Available Levels
Update Available Levels()
|
+-- Get Player Controller (Index: 0)
|   |
|   +-- Get Local Player
|       |
|       +-- Get Subsystem (Class: PlayerQuestSubsystem)
|           |
|           +-- Get Completed Quest IDs
|               |
|               +-- For Each Completed Quest:
|                   |
|                   +-- Branch (Quest ID == "Q003_UnlockCave")
|                       |
|                       +-- True Path:
|                           |
|                           +-- Get Stream Level Name from Quest ID
|                           |
|                           +-- Load Stream Level
|                               - Level Name: "DungeonCave"
|                               - Make Visible: True

// Call this function:
// 1. At Begin Play
// 2. After any quest completion via event binding
```

## Weather System Integration with Quests

Example of integrating an environmental system with quests:

```
/* Description: Weather system that can trigger quest progress */

// In Weather Manager Blueprint:

// Function: Weather State Changed
Weather State Changed(New Weather Type)
|
+-- Broadcast to all relevant systems
|
+-- For Each Player Controller:
    |
    +-- Get Local Player
        |
        +-- Get Subsystem (Class: PlayerQuestSubsystem)
            |
            +-- Get Active Quests
                |
                +-- For Each Active Quest:
                    |
                    +-- Branch (Has Weather Objective)
                        |
                        +-- True Path:
                            |
                            +-- Cast To IQuestProgress
                                |
                                +-- Execute Register Objective Progress
                                    - Objective Type String: "WeatherEvent"
                                    - Target ID: New Weather Type
                                    - Progress Amount: 1
```

These examples demonstrate how to integrate the Dynamic Quest System into your Blueprint-based gameplay systems. You can extend these patterns for more complex game mechanics and interactions.