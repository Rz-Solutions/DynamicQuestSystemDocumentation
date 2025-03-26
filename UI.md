# Dynamic Quest System: UI Components

The Dynamic Quest System includes several UMG widgets and a manager class to provide a basic user interface for quests.

## UI Manager (`UQuestUIManager`)

*   **Purpose:** A simple UObject class designed to coordinate the different UI widgets (Tracker, Notifications, potentially a Quest Log). It centralizes control over showing, hiding, and refreshing quest UI elements.
*   **Initialization:** Typically created and initialized in your `GameMode`'s `BeginPlay`. You pass instances of your created `QuestTrackerWidget` and `QuestNotificationWidget` to its `Initialize` function.
*   **Key Functions:**
    *   `Initialize(TrackerWidget, NotificationWidget)`: Sets up the manager with references to the widgets.
    *   `Shutdown()`: Cleans up references.
    *   `ToggleQuestLog()`: Shows/hides the Quest Tracker widget.
    *   `ShowQuestTracker()`, `HideQuestTracker()`: Explicit control over tracker visibility.
    *   `ShowNewQuestNotification(QuestTitle)`: Displays a notification for newly accepted quests.
    *   `ShowQuestUpdateNotification(QuestTitle, UpdateMessage)`: Displays a notification for objective progress.
    *   `ShowQuestCompletionNotification(QuestTitle)`: Displays a notification for completed quests.
    *   `RefreshAllUI()`: Forces an update of the managed UI elements.

## Quest Tracker Widget (`UQuestTrackerWidget`)

*   **Widget Blueprint:** Create a User Widget Blueprint inheriting from `UQuestTrackerWidget`. (e.g., `WBP_QuestTracker`).
*   **Purpose:** Displays a list of currently tracked quests and their objective progress on the HUD.
*   **Functionality:**
    *   Automatically listens for events from `UPlayerQuestSubsystem` (`OnQuestUpdated`, `OnQuestCompleted`, `OnQuestTrackingChanged`) to update its display.
    *   Pulls quest data from `UQuestManagerSubsystem` and progress data from `UPlayerQuestSubsystem`.
    *   Displays the title, description (optional), objectives with progress (e.g., "Kill Spiders 3/5"), and an overall completion progress bar for each tracked quest.
    *   Uses object pooling (`QuestWidgetPool`, `ObjectiveWidgetPool`) for potentially better performance when updating frequently.
    *   Manages which quests are displayed based on the `PlayerQuestSubsystem`'s tracked quests list, falling back to all active quests if none are explicitly tracked.
    *   Includes basic transition animations (fade/slide) when the list of displayed quests changes.
*   **Customization:** Style (colors, fonts) is driven by `UQuestUIStyles`. The visual layout is defined in the Widget Blueprint you create. Bind the required child widgets (`TrackerTitleText`, `MainScrollBox`, `QuestsContainer`, etc.) using the `meta=(BindWidget)` tag in your derived blueprint.

## Quest Notification Widget (`UQuestNotificationWidget`)

*   **Widget Blueprint:** Create a User Widget Blueprint inheriting from `UQuestNotificationWidget`. (e.g., `WBP_QuestNotification`).
*   **Purpose:** Displays temporary pop-up notifications for quest events (New, Updated, Completed).
*   **Functionality:**
    *   Provides functions (`ShowNewQuestNotification`, `ShowQuestUpdateNotification`, `ShowQuestCompletionNotification`) called by `UQuestUIManager` or directly.
    *   Displays a title and message.
    *   Automatically fades in, stays for a configurable duration (`DisplayTime`), and fades out.
    *   Uses different colors/icons (driven by `UQuestUIStyles`) for different notification types.
*   **Customization:** Layout defined in the Widget Blueprint. Bind child widgets (`NotificationTitleText`, `NotificationMessageText`, `NotificationBorder`, `IconImage`, `NotificationContainer`).

## Quest NPC Menu Widget (`UQuestNPCMenuWidget`)

*   **Widget Blueprint:** Create a User Widget Blueprint inheriting from `UQuestNPCMenuWidget`. (e.g., `WBP_QuestNPCMenu`).
*   **Purpose:** Provides a simple menu displayed when interacting with an `AQuestTriggerActor` that offers quests. Shows the NPC's name, dialog, and a list of available quests with "Accept" buttons.
*   **Functionality:**
    *   Initialized and shown by `AQuestTriggerActor::ShowQuestMenu`.
    *   Retrieves available quests for the player from the owning `AQuestTriggerActor`.
    *   Dynamically creates list items for each available quest, including title, description, and an "Accept" button.
    *   Handles button clicks to call `AQuestTriggerActor::AcceptQuest`.
    *   Provides a "Close" button.
    *   Automatically sets the input mode to `UIOnly` while open and restores `GameOnly` on close.
*   **Customization:** Layout defined in the Widget Blueprint. Bind child widgets (`NPCNameText`, `NPCDialogText`, `QuestListContainer`, `CloseButton`, etc.).

## Interaction Prompt Widget (`UInteractionPromptWidget`)

*   **Widget Blueprint:** Create a User Widget Blueprint inheriting from `UInteractionPromptWidget`. (e.g., `WBP_InteractionPrompt`).
*   **Purpose:** A very simple widget typically used by `AQuestTriggerActor`'s `InteractionWidgetComponent` to show an interaction key prompt (e.g., "[F]").
*   **Functionality:** Displays a key text (default 'F') inside a border.
*   **Customization:** Simple layout. Bind `KeyText` and `KeyBorder`.

## Styling (`UQuestUIStyles`)

*   **Purpose:** A `UObject` class containing static helper functions that define the default colors, fonts, margins, and other style properties used by the included UI widgets.
*   **Usage:** Widgets like `UQuestTrackerWidget` and `UQuestNotificationWidget` call functions like `UQuestUIStyles::GetBackgroundColor()`, `UQuestUIStyles::GetTitleFont()` to apply consistent styling.
*   **Customization:** You can modify this class directly or create your own styling solution and adapt the widgets to use it.

## Setup in GameMode

Typically, you'll create instances of your derived `QuestTracker` and `QuestNotification` widgets in your `AGameModeBase::BeginPlay` (or a dedicated HUD/UI manager class) and pass them to a `UQuestUIManager` instance.

```cpp
// Inside YourGameMode.cpp BeginPlay()

#include "UI/QuestTrackerWidget.h"
#include "UI/QuestNotificationWidget.h"
#include "UI/QuestUIManager.h"
#include "Blueprint/UserWidget.h"

// ...

void AYourGameMode::BeginPlay()
{
    Super::BeginPlay();

    APlayerController* PC = UGameplayStatics::GetPlayerController(this, 0);
    if (!PC) return;

    // Load widget classes (replace with your actual paths)
    TSubclassOf<UUserWidget> TrackerClass = LoadClass<UUserWidget>(nullptr, TEXT("/Game/UI/WBP_QuestTracker.WBP_QuestTracker_C"));
    TSubclassOf<UUserWidget> NotifyClass = LoadClass<UUserWidget>(nullptr, TEXT("/Game/UI/WBP_QuestNotification.WBP_QuestNotification_C"));

    if (TrackerClass)
    {
        QuestTrackerWidget = CreateWidget<UQuestTrackerWidget>(PC, TrackerClass);
        if (QuestTrackerWidget)
        {
            QuestTrackerWidget->AddToViewport(5); // Add with appropriate Z-order
        }
    }

    if (NotifyClass)
    {
        QuestNotificationWidget = CreateWidget<UQuestNotificationWidget>(PC, NotifyClass);
        if (QuestNotificationWidget)
        {
            QuestNotificationWidget->AddToViewport(20); // Add with appropriate Z-order
        }
    }

    // Create and initialize the UI Manager
    QuestUIManager = NewObject<UQuestUIManager>(this);
    if (QuestUIManager)
    {
        QuestUIManager->Initialize(QuestTrackerWidget, QuestNotificationWidget);
    }
}

// Make sure to store QuestTrackerWidget, QuestNotificationWidget, and QuestUIManager as member variables (UPROPERTY() TObjectPtr<...> or UPROPERTY()).