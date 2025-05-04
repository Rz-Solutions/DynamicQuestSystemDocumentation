**Cartographie Technique du Système de Quêtes pour Extension Blueprint**

Ce document détaille l'architecture clé du système de quêtes pour permettre son extension via des classes Blueprint, en se basant sur l'analyse des headers C++.

**I. `UQuestObjectiveBase` (Classe Parente pour Objectifs BP)**

*   **Héritage BP :** `Parent Class: QuestObjectiveBase`.
*   **Propriétés Clés (Configurables via `EditAnywhere`, `SaveGame`) :**
    *   `ObjectiveID` (FName) : ID unique (dans la quête).
    *   `Description`, `Hint`, `CompletionText`, `FailureText` (FText).
    *   `RequiredAmount` (int32).
    *   `VisualizationType` (EObjectiveVisualizationType).
    *   `ImportanceLevel` (EObjectiveImportance).
    *   `bIsOptional`, `bFailsQuestOnFailure`, `bStartsActive` (bool).
    *   `TimeLimit` (float).
    *   `ObjectiveTags` (FGameplayTagContainer).
    *   `LocationData` (TArray<FObjectiveLocationData>).
    *   `PrerequisiteObjectiveIDs` (TArray<FName>).
    *   `ActivationCondition`, `FailureCondition`, `ProgressCondition` (Instanced `UQuestConditionBase*`).
    *   `ObjectiveDataTable` (UDataTable*), `ObjectiveDataRow` (FName).
*   **Propriété d'Accès (ReadOnly) :**
    *   `QuestOwner` (UObject*) : Le contexte (souvent l'instance de quête ou le `PlayerQuestComponent`). **Essentiel pour remonter au `PlayerQuestComponent`.**
*   **Fonctions Clés à Overrider (`BlueprintNativeEvent`) :**
    *   `InitializeObjective(Owner)` : Pour le setup initial. **Appeler Parent!**
    *   `ProcessQuestEvent(EventType, Parameters, &OutProgress)` : **Logique principale.** Filtrer les events entrants. Si pertinent, obtenir le `PlayerQuestComponent` (via `QuestOwner`) et appeler `UpdateQuestObjectiveProgress(...)`. **Appeler Parent!**
    *   `CanComplete(CurrentProgress)` : Retourner `CurrentProgress >= RequiredAmount`.
    *   `OnActivated()`, `OnCompleted()`, `OnFailed()`, `OnProgressUpdated(Current, Previous)`, `OnTimeUpdate(Remaining)` : Hooks pour logique/feedback. **Appeler Parent!**
    *   `GetFormattedProgressText(CurrentProgress)` : Pour l'affichage UI de la progression.

**II. `UQuestConditionBase` (Classe Parente pour Conditions BP)**

*   **Héritage BP :** `Parent Class: QuestConditionBase`. **(Note : Hérite de `UQuestActionBase`)**
*   **Propriétés Clés :** `bInvertResult` (bool). Ajouter les variables BP nécessaires.
*   **Fonctions Clés à Overrider (`BlueprintNativeEvent`) :**
    *   `InitializeCondition(Owner)` : Setup. **Appeler Parent!**
    *   `CheckCondition(Parameters, WorldContext)` : **Logique principale.** Retourner `true`/`false` basé sur la vérification custom. Utiliser `Owner` pour le contexte.
    *   `GetDisplayText()` : Pour affichage descriptif.

**III. `UQuestActionBase` (Classe Parente pour Actions/Récompenses BP)**

*   **Héritage BP :** `Parent Class: QuestActionBase`.
*   **Propriétés Clés :** `ActionID`, `QuestID`, `ObjectiveID` (FName), `Description` (FText), `ExecutionCondition` (Instanced `UQuestConditionBase*`), `ExecutionDelay` (float), `bUpdateObjectiveProgress` (bool), `ProgressAmount` (int32). Ajouter les variables BP nécessaires.
*   **Délégué Clé :** `OnActionCompleted` (Param: `FActionResultParams`).
*   **Fonctions Clés à Overrider (`BlueprintNativeEvent`) :**
    *   `InitializeAction(Owner)` : Setup. **Appeler Parent!**
    *   `ExecuteAction(Parameters, WorldContext)` : **Logique principale.** Implémenter l'effet. Peut remplir `ActionResults` (protégé). Mettre `bHasCompleted` (protégé) à `true`.
    *   `GetActionResult(&OutParameters)` : Fournir les résultats stockés.
    *   `CancelAction()`, `IsExecuting()`, `HasCompleted()` : Gestion d'état.

**IV. `UPlayerQuestComponent` (Point d'Interaction Principal pour BP)**

*   **Accès :** Obtenir via `GetComponentByClass` sur `PlayerState`/`Controller`.
*   **Fonctions d'Interaction Clés (`BlueprintCallable`/`Pure`) :**
    *   **Modification d'État :**
        *   `AcceptQuest(QuestID)`
        *   `AbandonQuest(QuestID)`
        *   `UpdateQuestObjectiveProgress(QuestID, ObjectiveID, Progress)` : **Crucial, à appeler depuis l'objectif BP.**
        *   `SetQuestTracked(QuestID, bTracked)`
    *   **Lecture d'État :**
        *   `GetQuestState(QuestID)` : Retourne `EQuestStateType`.
        *   `GetObjectiveProgress(QuestID, ObjectiveID)` : Retourne `int32`.
        *   `IsQuestTracked(QuestID)`
        *   `GetActiveQuestIDs()`, `GetCompletedQuestIDs()`, `GetFailedQuestIDs()`, `GetTrackedQuestIDs()` : Retournent `TArray<FName>`.
        *   `GetQuestInstanceData(QuestID, &OutQuestInstance)` : Pour état dynamique (`FRuntimeQuestInstance`).
        *   `GetQuestDefinitionData(QuestID, &OutQuestDefinition)` : Pour définition statique (`FQuestDefinition`).
        *   `GetQuestTitle(QuestID)`, `GetQuestDescription(QuestID)` : Retournent `FText`.
        *   `GetQuestModularObjectives(QuestID)` : Retourne `TArray<UQuestObjectiveBase*>`.
*   **Événements à Écouter (`BlueprintAssignable` Delegates pour UI/Logique Réactive) :**
    *   `OnQuestObjectiveUpdated(QuestID, ObjectiveID)`
    *   `OnQuestObjectiveCompleted(QuestID, ObjectiveID)`
    *   `OnQuestStateChanged(QuestID)` : **Utiliser pour détecter Completion/Failure en vérifiant `GetQuestState`.**
    *   `OnQuestCompleted(QuestID)`
    *   `OnQuestFailed(QuestID)`
    *   `OnQuestAbandoned(QuestID)`
    *   `OnQuestTrackingChanged(QuestID, bIsTracked)`

**V. Structs & Enums Clés (Exposés en BP via `QuestDataTypes.h`)**

*   **Enums :** `EQuestStateType`, `EObjectiveVisualizationType`, `EObjectiveImportance`, etc.
*   **Structs :** `FQuestDefinition` (Statique), `FRuntimeQuestInstance` (Dynamique, contient les TMap `ObjectiveProgress` et `ObjectiveTimeRemaining`), `FObjectiveLocationData`, `FGameplayTagContainer`.

---

**Principes Clés pour l'Extension BP :**

1.  **Hériter** des classes de base C++ (`UQuestObjectiveBase`, `UQuestConditionBase`, `UQuestActionBase`).
2.  **Overrider** les fonctions `_Implementation` pertinentes (surtout `ProcessQuestEvent`, `CheckCondition`, `ExecuteAction`) pour la logique custom. **Toujours considérer l'appel à `Parent:`**.
3.  **Interagir** via le `UPlayerQuestComponent` pour lire l'état et **notifier** les changements (principalement via `UpdateQuestObjectiveProgress`).
4.  **Utiliser** les `Delegates` du `UPlayerQuestComponent` pour la mise à jour réactive de l'UI et la logique de jeu découplée.
