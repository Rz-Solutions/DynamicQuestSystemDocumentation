# QAI Plugin Developer Guide

This document provides comprehensive guidance for understanding, extending, and working with the QAI plugin architecture. The QAI plugin is a high-performance, modular AI system that implements a behavior-archetype pattern with subsystem-driven runtime processing.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Components](#core-components)
3. [Behavior System](#behavior-system)
4. [Data Structures](#data-structures)
5. [Performance Features](#performance-features)
6. [Adding New AI Types](#adding-new-ai-types)
7. [File Organization](#file-organization)
8. [Best Practices](#best-practices)
9. [Debugging and Monitoring](#debugging-and-monitoring)
10. [Common Patterns](#common-patterns)

## Architecture Overview

The QAI plugin implements a **Behavior-Archetype System** with the following key principles:

- **Subsystem-driven**: `UQAI_SubSystem` manages the entire AI runtime
- **Structure-of-Arrays (SoA)**: Cache-friendly data layout via `UQAI_AgentRegistry`
- **Batch Processing**: Processors handle agents in batches for optimal performance
- **Behavior Delegation**: Type-specific logic is isolated in behavior classes

### Processing Pipeline

The system processes agents in three sequential phases every tick:

1. **State Processing** (`UQAI_StateProcessor`)
   - Computes state transitions and flags
   - Handles taser logic delegation

2. **Movement Processing** (`UQAI_MovementProcessor`) 
   - Applies per-agent movement intent
   - Handles rotation and speed scaling

3. **Combat Processing** (`UQAI_CombatProcessor`)
   - Executes combat actions (trace/taser for drones, melee/ranged/jump for autonomous)
   - Delegates to behavior-specific combat decisions

```cpp
// Typical tick flow in UQAI_SubSystem
void UQAI_SubSystem::TickSubsystem(float DeltaTime)
{
    if (!IsActive()) return;
    
    // Process in order - State -> Movement -> Combat
    StateProcessor->ProcessAgents(Registry, DeltaTime);
    MovementProcessor->ProcessAgents(Registry, DeltaTime);
    CombatProcessor->ProcessAgents(Registry, DeltaTime);
}
```

## Core Components

### UQAI_SubSystem

The central subsystem that orchestrates the entire AI system.

**Key Responsibilities:**
- Creates and manages all processors and registries
- Provides main tick loop coordination
- Exposes behavior registry access
- Handles system initialization and cleanup

**Location:** `Public/QAI_SubSystem.h`, `Private/QAI_SubSystem.cpp`

```cpp
// Access the subsystem from anywhere
UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
if (AISubsystem && AISubsystem->IsActive())
{
    UQAI_AgentRegistry* Registry = AISubsystem->GetAgentRegistry();
    // Use registry for agent operations
}
```

### UQAI_AgentRegistry (Structure-of-Arrays)

High-performance agent data storage using cache-friendly SoA layout.

**Key Features:**
- Separate arrays for hot/warm/cold data
- Sparse/dense index mapping for O(1) lookups
- Explicit prefetch helpers for batch operations
- Memory alignment optimizations

**Location:** `Public/Registry/QAI_AgentRegistry.h`, `Private/Registry/QAI_AgentRegistry.cpp`

```cpp
// Adding an agent to the registry
FQAI_PoliceAgent_Data AgentData;
AgentData.BaseAgent.bIsValid = true;
AgentData.Archetype = EQAI_AgentArchetype::Drone;
AgentData.DroneConfig.bIsCaptainDrone = true;

int32 AgentIndex = Registry->AddAgent(AgentData, OwnerActor);

// Batch processing with prefetch
TArray<int32> AgentIndices = Registry->GetActiveAgentIndices();
Registry->PrefetchAgentData(AgentIndices);
for (int32 Index : AgentIndices)
{
    // Process agent at Index - data is now in cache
}
```

### Processors

Specialized components that handle different aspects of agent behavior:

#### UQAI_StateProcessor
- Manages state transitions
- Handles taser logic via behavior delegation
- Updates agent state flags

#### UQAI_MovementProcessor  
- Computes movement inputs from behaviors
- Applies movement and rotation
- Handles speed scaling

#### UQAI_CombatProcessor
- Delegates combat decisions to behaviors
- Executes combat actions (tracing, taser, melee, ranged attacks)
- Manages combat state updates

**Common Pattern:**
```cpp
// Processor delegation to behavior
IQAI_AgentBehavior* Behavior = BehaviorRegistry->GetBehaviorForAgent(Registry, AgentIndex);
if (Behavior)
{
    FVector MovementInput;
    float SpeedScale;
    FVector TargetToFace;
    
    bool bSuccess = Behavior->ComputeMovementInput(
        Registry, AgentIndex, DeltaTime,
        MovementInput, SpeedScale, TargetToFace
    );
    
    if (bSuccess)
    {
        ApplyMovementInput(Registry, AgentIndex, MovementInput, SpeedScale, TargetToFace);
    }
}
```

## Behavior System

The behavior system provides clean separation of AI logic by agent type (archetype).

### Agent Archetypes

Defined in `Public/Behavior/QAI_AgentArchetype.h`:

```cpp
UENUM(BlueprintType)
enum class EQAI_AgentArchetype : uint8
{
    Unknown,        // Legacy - triggers inference
    Drone,         // Flying AI units
    Autonomous,    // Ground-based AI units
    // Add new types here
};
```

### Behavior Interface

All behaviors implement the `IQAI_AgentBehavior` interface:

**Location:** `Public/Behavior/QAI_AgentBehavior.h`

```cpp
class IQAI_AgentBehavior
{
public:
    // Movement computation
    virtual bool ComputeMovementInput(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime,
        FVector& OutMovementInput,
        float& OutSpeedScale,
        FVector& OutTargetToFace
    ) = 0;

    // Combat decision making
    virtual FQAI_CombatDecision ComputeCombatDecision(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime,
        const FQAI_CombatParams& Params
    ) = 0;

    // State assistance (optional hooks)
    virtual bool WantsTaser(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime,
        const FQAI_CombatParams& Params
    ) { return false; }
};
```

### Built-in Behaviors

#### Drone Behavior (`FQAI_DroneBehavior`)
- **Files:** `Public/Behavior/QAI_DroneBehavior.h/.cpp`
- **Capabilities:** Flying movement, line-of-sight combat, taser support
- **Captain Logic:** Enhanced behavior for captain drones

#### Autonomous Behavior (`FQAI_AutonomousBehavior`)  
- **Files:** `Public/Behavior/QAI_AutonomousBehavior.h/.cpp`
- **Capabilities:** Ground movement, melee/ranged combat, jump attacks

### Behavior Registry

The `UQAI_AgentBehaviorRegistry` manages behavior instances and routing.

**Location:** `Public/Behavior/QAI_AgentBehaviorRegistry.h`, `Private/Behavior/QAI_AgentBehaviorRegistry.cpp`

```cpp
// Register a new behavior
void UQAI_AgentBehaviorRegistry::Initialize()
{
    BehaviorByType.Add(EQAI_AgentArchetype::Drone, MakeUnique<FQAI_DroneBehavior>());
    BehaviorByType.Add(EQAI_AgentArchetype::Autonomous, MakeUnique<FQAI_AutonomousBehavior>());
    // Add new behaviors here
}

// Runtime behavior lookup
IQAI_AgentBehavior* GetBehaviorForAgent(UQAI_AgentRegistry* Registry, int32 AgentIndex)
{
    EQAI_AgentArchetype Archetype = Registry->GetAgentArchetype(AgentIndex);
    return BehaviorByType.FindRef(Archetype).Get();
}
```

## Data Structures

### Core Data Types

#### FQAI_PoliceAgent_Data
Main agent data structure containing:
- `FQAI_Agent_Data BaseAgent` - Basic agent info, movement, sensing
- `FQAI_Police_State PoliceState` - Police-specific state and wanted levels
- `FQAI_DroneConfig DroneConfig` - Drone-specific configuration
- `FQAI_AutonomousConfig AutonomousConfig` - Autonomous agent configuration
- `EQAI_AgentArchetype Archetype` - Explicit agent type

```cpp
FQAI_PoliceAgent_Data AgentData;
AgentData.BaseAgent.bIsValid = true;
AgentData.BaseAgent.Agent_Info.MaxHealth = 100.0f;
AgentData.BaseAgent.Agent_Movement.MaxSpeed = 600.0f;
AgentData.Archetype = EQAI_AgentArchetype::Drone;
AgentData.DroneConfig.bIsCaptainDrone = true;
AgentData.DroneConfig.bHasTaser = true;
```

#### Combat Data Structures

**FQAI_CombatParams** - Input parameters for combat decisions
**FQAI_CombatDecision** - Output of behavior combat computation

```cpp
struct FQAI_CombatDecision
{
    bool bShouldAttack = false;
    bool bUseRangedAttack = false;
    bool bShouldJump = false;
    FVector AttackTarget = FVector::ZeroVector;
    float AttackPower = 1.0f;
};
```

### Component Integration

#### QAI_AgentComponent
Bridges between UE5 actors and the QAI system.

**Location:** `Public/Agent/QAI_AgentComponent.h`, `Private/Agent/QAI_AgentComponent.cpp`

```cpp
// Add QAI capabilities to any actor
UCLASS(ClassGroup=(Custom), meta=(BlueprintSpawnableComponent))
class QAI_API UQAI_AgentComponent : public UActorComponent
{
    // Handles registration/unregistration with QAI subsystem
    // Provides data synchronization between actor and registry
    // Manages component lifecycle
};
```

## Performance Features

### Structure-of-Arrays (SoA) Layout

The registry uses SoA for optimal cache performance:

```cpp
// Instead of Array-of-Structures (AoS):
TArray<FAgentData> Agents; // Bad for cache when processing movement only

// QAI uses Structure-of-Arrays (SoA):
TArray<FVector> Positions;     // Hot data - accessed every frame
TArray<FVector> Velocities;    // Hot data
TArray<FAgentConfig> Configs;  // Cold data - rarely accessed
```

### SIMD Optimizations

**Location:** `Public/Performance/QAI_SIMD.h`

```cpp
// SIMD-optimized batch operations
namespace QAISIMD
{
    void BatchVectorAdd(TArray<FVector>& Results, const TArray<FVector>& A, const TArray<FVector>& B);
    void BatchDistanceCompute(TArray<float>& Distances, const TArray<FVector>& Positions, FVector Center);
}
```

### Spatial Partitioning

**Location:** `Public/Performance/QAI_SpatialHashGrid.h`, `Private/Performance/QAI_SpatialHashGrid.cpp`

```cpp
// Efficient spatial queries for large agent counts
class QAI_API FQAI_SpatialHashGrid
{
public:
    void UpdateAgentPosition(int32 AgentID, const FVector& Position);
    TArray<int32> QueryAgentsInRadius(const FVector& Center, float Radius);
    void BatchUpdate(const TArray<FVector>& Positions, const TArray<int32>& AgentIDs);
};
```

### Performance Constants

**Location:** `Public/Performance/QAI_PerformanceConstants.h`

```cpp
namespace QAI_Performance
{
    constexpr int32 BATCH_SIZE = 64;              // Optimal batch size for SIMD
    constexpr int32 CACHE_LINE_SIZE = 64;         // CPU cache line alignment
    constexpr int32 PREFETCH_DISTANCE = 8;       // Memory prefetch lookahead
    constexpr float SPATIAL_GRID_CELL_SIZE = 100.0f; // Spatial partitioning cell size
}
```

## Adding New AI Types

### Step-by-Step Process

#### 1. Define the Archetype
Add your new type to the enum in `Public/Behavior/QAI_AgentArchetype.h`:

```cpp
UENUM(BlueprintType)
enum class EQAI_AgentArchetype : uint8
{
    Unknown,
    Drone,
    Autonomous,
    Guardian,      // New archetype
    Specialist     // Another new archetype
};
```

#### 2. Create Behavior Class

**Header** (`Public/Behavior/QAI_GuardianBehavior.h`):
```cpp
#pragma once

#include "Behavior/QAI_AgentBehavior.h"

class QAI_API FQAI_GuardianBehavior : public IQAI_AgentBehavior
{
public:
    virtual bool ComputeMovementInput(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime,
        FVector& OutMovementInput,
        float& OutSpeedScale,
        FVector& OutTargetToFace
    ) override;

    virtual FQAI_CombatDecision ComputeCombatDecision(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime,
        const FQAI_CombatParams& Params
    ) override;

    // Optional: Add custom behavior hooks
    virtual bool WantsDefensivePosition(
        UQAI_AgentRegistry* Registry,
        int32 AgentIndex,
        float DeltaTime
    );

private:
    // Guardian-specific helper methods
    FVector ComputeDefensivePosition(const FQAI_PoliceAgent_Data& AgentData);
    bool ShouldActivateShield(const FQAI_CombatParams& Params);
};
```

**Implementation** (`Private/Behavior/QAI_GuardianBehavior.cpp`):
```cpp
#include "Behavior/QAI_GuardianBehavior.h"
#include "Registry/QAI_AgentRegistry.h"

bool FQAI_GuardianBehavior::ComputeMovementInput(
    UQAI_AgentRegistry* Registry,
    int32 AgentIndex,
    float DeltaTime,
    FVector& OutMovementInput,
    float& OutSpeedScale,
    FVector& OutTargetToFace)
{
    if (!Registry || !Registry->IsValidAgentIndex(AgentIndex))
    {
        return false;
    }

    const FQAI_PoliceAgent_Data& AgentData = Registry->GetAgentData(AgentIndex);
    
    // Guardian-specific movement logic
    if (WantsDefensivePosition(Registry, AgentIndex, DeltaTime))
    {
        FVector DefensivePos = ComputeDefensivePosition(AgentData);
        OutMovementInput = (DefensivePos - AgentData.BaseAgent.Transform.Position).GetSafeNormal();
        OutSpeedScale = 0.6f; // Move cautiously to defensive position
        OutTargetToFace = GetPrimaryThreatDirection(Registry, AgentIndex);
        return true;
    }
    
    // Fallback to standard patrol behavior
    return ComputePatrolMovement(Registry, AgentIndex, OutMovementInput, OutSpeedScale, OutTargetToFace);
}

FQAI_CombatDecision FQAI_GuardianBehavior::ComputeCombatDecision(
    UQAI_AgentRegistry* Registry,
    int32 AgentIndex,
    float DeltaTime,
    const FQAI_CombatParams& Params)
{
    FQAI_CombatDecision Decision;
    
    if (ShouldActivateShield(Params))
    {
        Decision.bShouldAttack = false;
        Decision.bActivateShield = true; // Custom decision field
        return Decision;
    }
    
    // Standard combat logic for threats
    Decision.bShouldAttack = Params.bHasLineOfSight && Params.DistanceToTarget < 1000.0f;
    Decision.bUseRangedAttack = Params.DistanceToTarget > 300.0f;
    Decision.AttackTarget = Params.TargetLocation;
    Decision.AttackPower = CalculateAttackPower(Params);
    
    return Decision;
}
```

#### 3. Register the Behavior

In `UQAI_AgentBehaviorRegistry::Initialize()`:
```cpp
void UQAI_AgentBehaviorRegistry::Initialize()
{
    BehaviorByType.Add(EQAI_AgentArchetype::Drone, MakeUnique<FQAI_DroneBehavior>());
    BehaviorByType.Add(EQAI_AgentArchetype::Autonomous, MakeUnique<FQAI_AutonomousBehavior>());
    BehaviorByType.Add(EQAI_AgentArchetype::Guardian, MakeUnique<FQAI_GuardianBehavior>());
    
    UE_LOG(LogQAI, Log, TEXT("Registered %d behavior types"), BehaviorByType.Num());
}
```

#### 4. Configure Agent Data

Extend the data structure if needed (`Public/Data/QAI_Struct.h`):
```cpp
USTRUCT(BlueprintType)
struct FQAI_GuardianConfig
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float ShieldStrength = 100.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float DefensiveRadius = 500.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bCanSummonBackup = false;
};

// Add to main agent data structure
USTRUCT(BlueprintType)
struct FQAI_PoliceAgent_Data
{
    GENERATED_BODY()
    
    // ... existing fields ...
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FQAI_GuardianConfig GuardianConfig = FQAI_GuardianConfig();
};
```

#### 5. Spawn Agents with New Type

```cpp
// Create a Guardian agent
void SpawnGuardianAgent(AActor* OwnerActor, const FVector& SpawnLocation)
{
    UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
    if (!AISubsystem) return;
    
    FQAI_PoliceAgent_Data AgentData;
    AgentData.BaseAgent.bIsValid = true;
    AgentData.BaseAgent.Agent_Info.MaxHealth = 150.0f; // Tankier than normal
    AgentData.BaseAgent.Agent_Movement.MaxSpeed = 400.0f; // Slower but stronger
    AgentData.Archetype = EQAI_AgentArchetype::Guardian;
    AgentData.GuardianConfig.ShieldStrength = 100.0f;
    AgentData.GuardianConfig.bCanSummonBackup = true;
    
    int32 AgentIndex = AISubsystem->GetAgentRegistry()->AddAgent(AgentData, OwnerActor);
    
    UE_LOG(LogQAI, Log, TEXT("Spawned Guardian agent at index %d"), AgentIndex);
}
```

### Extending Behavior Hooks

If your new AI type requires additional processor integration:

#### 1. Add New Interface Methods
```cpp
// In QAI_AgentBehavior.h
class IQAI_AgentBehavior
{
    // ... existing methods ...
    
    // New hook for Guardian-specific logic
    virtual bool WantsDefensivePosition(UQAI_AgentRegistry* Registry, int32 AgentIndex, float DeltaTime)
    {
        return false; // Default implementation
    }
};
```

#### 2. Use Hook in Processors
```cpp
// In UQAI_StateProcessor::ProcessAgent
IQAI_AgentBehavior* Behavior = BehaviorRegistry->GetBehaviorForAgent(Registry, AgentIndex);
if (Behavior && Behavior->WantsDefensivePosition(Registry, AgentIndex, DeltaTime))
{
    // Set defensive state flags
    Registry->SetAgentState(AgentIndex, EQAI_AgentState::Defensive);
}
```

## File Organization

### Directory Structure
```
Plugins/QAI/Source/QAI/
├── Public/
│   ├── Agent/                    # Agent component and actor integration
│   ├── Behavior/                 # Behavior system (archetypes, interface, built-ins)
│   ├── Component/                # Movement and utility components
│   ├── Data/                     # Data structures and configuration
│   ├── Performance/              # SIMD, spatial hashing, optimization
│   ├── Processor/                # State, movement, combat processors
│   ├── Registry/                 # SoA agent registry
│   ├── Sensing/                  # Sensing and perception components
│   ├── Subsystem/                # Coordination and management subsystems
│   ├── QAI.h                     # Main module header
│   ├── QAI_SubSystem.h           # Primary subsystem
│   └── QAI_FunctionLibrary.h     # Blueprint utility functions
└── Private/                      # Implementation files (mirror of Public structure)
```

### Key File Mapping

| Component | Header Location | Implementation |
|-----------|----------------|----------------|
| **Core System** | | |
| Main Subsystem | `Public/QAI_SubSystem.h` | `Private/QAI_SubSystem.cpp` |
| Agent Registry | `Public/Registry/QAI_AgentRegistry.h` | `Private/Registry/QAI_AgentRegistry.cpp` |
| **Processors** | | |
| State Processor | `Public/Processor/QAI_StateProcessor.h` | `Private/Processor/QAI_StateProcessor.cpp` |
| Movement Processor | `Public/Processor/QAI_MovementProcessor.h` | `Private/Processor/QAI_MovementProcessor.cpp` |
| Combat Processor | `Public/Processor/QAI_CombatProcessor.h` | `Private/Processor/QAI_CombatProcessor.cpp` |
| **Behavior System** | | |
| Behavior Interface | `Public/Behavior/QAI_AgentBehavior.h` | Header-only interface |
| Behavior Registry | `Public/Behavior/QAI_AgentBehaviorRegistry.h` | `Private/Behavior/QAI_AgentBehaviorRegistry.cpp` |
| Agent Archetypes | `Public/Behavior/QAI_AgentArchetype.h` | Header-only enum |
| Drone Behavior | `Public/Behavior/QAI_DroneBehavior.h` | `Private/Behavior/QAI_DroneBehavior.cpp` |
| Autonomous Behavior | `Public/Behavior/QAI_AutonomousBehavior.h` | `Private/Behavior/QAI_AutonomousBehavior.cpp` |
| **Data Structures** | | |
| Main Structures | `Public/Data/QAI_Struct.h` | Header-only structs |
| State Data | `Public/Data/QAI_State_Struct.h` | Header-only structs |
| Movement Data | `Public/Data/QAI_Movement_Struct.h` | Header-only structs |
| Sensing Data | `Public/Data/QAI_Sensing_Struct.h` | Header-only structs |
| **Performance** | | |
| SIMD Operations | `Public/Performance/QAI_SIMD.h` | Header-only optimizations |
| Spatial Hashing | `Public/Performance/QAI_SpatialHashGrid.h` | `Private/Performance/QAI_SpatialHashGrid.cpp` |
| **Components** | | |
| Agent Component | `Public/Agent/QAI_AgentComponent.h` | `Private/Agent/QAI_AgentComponent.cpp` |
| Movement Components | `Public/Component/QAI_*Movement.h` | `Private/Component/QAI_*Movement.cpp` |

## Best Practices

### Performance Guidelines

#### 1. Batch Operations
```cpp
// Bad - Individual agent processing
for (int32 AgentIndex : ActiveAgents)
{
    ProcessSingleAgent(AgentIndex);
}

// Good - Batch processing with prefetch
Registry->PrefetchAgentData(ActiveAgents);
for (int32 AgentIndex : ActiveAgents)
{
    ProcessSingleAgent(AgentIndex); // Data now in cache
}
```

#### 2. Minimize Memory Allocations
```cpp
// Bad - Allocations in hot path
void ComputeMovementInput(/* params */)
{
    TArray<FVector> Targets; // Allocation every call
    FindNearbyTargets(Targets);
}

// Good - Reuse containers
class FQAI_DroneBehavior
{
    TArray<FVector> CachedTargets; // Reused across calls
    
    void ComputeMovementInput(/* params */)
    {
        CachedTargets.Reset(); // Clear without deallocation
        FindNearbyTargets(CachedTargets);
    }
};
```

#### 3. Logging Performance
```cpp
// Bad - Excessive logging
UE_LOG(LogQAI, Log, TEXT("Processing agent %d"), AgentIndex); // Every frame!

// Good - Throttled logging
static float LastLogTime = 0.0f;
if (GetWorld()->GetTimeSeconds() - LastLogTime > 1.0f)
{
    UE_LOG(LogQAI, Log, TEXT("Processed %d agents this second"), ProcessedCount);
    LastLogTime = GetWorld()->GetTimeSeconds();
}
```

### Architecture Guidelines

#### 1. No Fallbacks Policy
```cpp
// Bad - Fallback logic
IQAI_AgentBehavior* Behavior = GetBehaviorForAgent(AgentIndex);
if (!Behavior)
{
    // Fallback to default behavior - FORBIDDEN!
    ApplyDefaultMovement(AgentIndex);
    return;
}

// Good - Explicit failure
IQAI_AgentBehavior* Behavior = GetBehaviorForAgent(AgentIndex);
if (!Behavior)
{
    UE_LOG(LogQAI, Error, TEXT("No behavior for agent %d archetype %s"), 
           AgentIndex, *UEnum::GetValueAsString(Archetype));
    return; // Fail explicitly
}
```

#### 2. Behavior Isolation
```cpp
// Bad - Type checking in processors
void ProcessMovement(int32 AgentIndex)
{
    if (IsDrone(AgentIndex))
    {
        ProcessDroneMovement(AgentIndex);
    }
    else if (IsAutonomous(AgentIndex))
    {
        ProcessAutonomousMovement(AgentIndex);
    }
    // ... more type checks
}

// Good - Behavior delegation
void ProcessMovement(int32 AgentIndex)
{
    IQAI_AgentBehavior* Behavior = BehaviorRegistry->GetBehaviorForAgent(Registry, AgentIndex);
    if (Behavior)
    {
        Behavior->ComputeMovementInput(Registry, AgentIndex, DeltaTime, /* outputs */);
    }
}
```

#### 3. Clean Data Flow
```cpp
// Good - Clear input/output separation
struct FQAI_MovementResult
{
    FVector MovementInput;
    float SpeedScale;
    FVector TargetToFace;
    bool bSuccess;
};

FQAI_MovementResult ComputeMovementInput(const FQAI_MovementParams& Params)
{
    FQAI_MovementResult Result;
    // ... computation
    Result.bSuccess = true;
    return Result;
}
```

## Debugging and Monitoring

### Built-in Debugging Tools

#### 1. Registry Validation
```cpp
// Validate registry integrity (non-shipping builds only)
if (!UE_BUILD_SHIPPING)
{
    bool bIsValid = Registry->ValidateIntegrity();
    if (!bIsValid)
    {
        UE_LOG(LogQAI, Error, TEXT("Registry integrity check failed!"));
    }
}
```

#### 2. Performance Metrics
```cpp
// Get processor statistics
UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
if (AISubsystem)
{
    FQAIProcessorStats Stats = AISubsystem->GetProcessorStats();
    UE_LOG(LogQAI, Log, TEXT("Avg processing time: %.2fms, Agents: %d"), 
           Stats.AverageProcessTimeMs, Stats.ActiveAgentCount);
}
```

#### 3. Console Commands
```cpp
// Available console commands (implement in module startup)
static FAutoConsoleCommand QAI_DumpStats(
    TEXT("QAI.DumpStats"),
    TEXT("Dump QAI subsystem statistics"),
    FConsoleCommandDelegate::CreateLambda([]()
    {
        if (GWorld)
        {
            UQAI_SubSystem* AISubsystem = GWorld->GetSubsystem<UQAI_SubSystem>();
            AISubsystem->DumpDebugStats();
        }
    })
);
```

### Common Debug Patterns

#### 1. Behavior Debug Visualization
```cpp
// In behavior implementations
void FQAI_DroneBehavior::ComputeMovementInput(/* params */)
{
    // ... logic ...
    
#if WITH_EDITOR
    // Debug visualization in editor
    if (GEngine && CVarShowQAIDebug.GetValueOnGameThread())
    {
        DrawDebugLine(GetWorld(), AgentPos, TargetPos, FColor::Green, false, 0.1f);
        DrawDebugString(GetWorld(), AgentPos + FVector(0, 0, 100), 
                       FString::Printf(TEXT("Speed: %.1f"), SpeedScale), 
                       nullptr, FColor::White, 0.1f);
    }
#endif
}
```

#### 2. State Transition Logging
```cpp
// Throttled state change logging
void LogStateTransition(int32 AgentIndex, EQAI_AgentState OldState, EQAI_AgentState NewState)
{
    static TMap<int32, float> LastLogTimes;
    float CurrentTime = GetWorld()->GetTimeSeconds();
    float* LastTime = LastLogTimes.Find(AgentIndex);
    
    if (!LastTime || (CurrentTime - *LastTime) > 1.0f)
    {
        UE_LOG(LogQAI, Warning, TEXT("Agent %d: %s -> %s"), 
               AgentIndex, 
               *UEnum::GetValueAsString(OldState),
               *UEnum::GetValueAsString(NewState));
        LastLogTimes.Add(AgentIndex, CurrentTime);
    }
}
```

## Common Patterns

### Agent Lifecycle Management

```cpp
// Complete agent lifecycle example
class UQAI_AgentManager : public UObject
{
public:
    int32 SpawnAgent(AActor* Owner, const FQAI_PoliceAgent_Data& AgentData)
    {
        UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
        if (!AISubsystem || !AISubsystem->IsActive())
        {
            UE_LOG(LogQAI, Error, TEXT("QAI Subsystem not available"));
            return INDEX_NONE;
        }
        
        int32 AgentIndex = AISubsystem->GetAgentRegistry()->AddAgent(AgentData, Owner);
        if (AgentIndex != INDEX_NONE)
        {
            RegisterAgentForCleanup(AgentIndex, Owner);
            NotifyAgentSpawned(AgentIndex);
        }
        
        return AgentIndex;
    }
    
    void DestroyAgent(int32 AgentIndex)
    {
        UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
        if (AISubsystem)
        {
            AISubsystem->GetAgentRegistry()->RemoveAgent(AgentIndex);
            UnregisterAgentFromCleanup(AgentIndex);
            NotifyAgentDestroyed(AgentIndex);
        }
    }
    
private:
    TMap<int32, TWeakObjectPtr<AActor>> AgentOwners;
    
    void RegisterAgentForCleanup(int32 AgentIndex, AActor* Owner)
    {
        AgentOwners.Add(AgentIndex, Owner);
    }
};
```

### Spatial Queries

```cpp
// Efficient spatial queries using the spatial hash grid
void FindNearbyEnemies(const FVector& Position, float Radius, TArray<int32>& OutEnemyIndices)
{
    UQAI_SubSystem* AISubsystem = GetWorld()->GetSubsystem<UQAI_SubSystem>();
    if (!AISubsystem) return;
    
    // Use spatial partitioning for initial filtering
    FQAI_SpatialHashGrid* SpatialGrid = AISubsystem->GetSpatialGrid();
    TArray<int32> CandidateAgents = SpatialGrid->QueryAgentsInRadius(Position, Radius);
    
    // Refine results with precise distance checks
    UQAI_AgentRegistry* Registry = AISubsystem->GetAgentRegistry();
    for (int32 AgentIndex : CandidateAgents)
    {
        if (IsEnemy(Registry, AgentIndex))
        {
            FVector AgentPos = Registry->GetAgentPosition(AgentIndex);
            if (FVector::DistSquared(Position, AgentPos) <= FMath::Square(Radius))
            {
                OutEnemyIndices.Add(AgentIndex);
            }
        }
    }
}
```

### Batch Processing Template

```cpp
// Template for efficient batch processing
template<typename ProcessFunc>
void ProcessAgentsBatch(UQAI_AgentRegistry* Registry, const TArray<int32>& AgentIndices, ProcessFunc Processor)
{
    if (AgentIndices.Num() == 0) return;
    
    // Prefetch agent data for better cache performance
    Registry->PrefetchAgentData(AgentIndices);
    
    // Process in batches for optimal cache usage
    constexpr int32 BatchSize = QAI_Performance::BATCH_SIZE;
    for (int32 StartIdx = 0; StartIdx < AgentIndices.Num(); StartIdx += BatchSize)
    {
        int32 EndIdx = FMath::Min(StartIdx + BatchSize, AgentIndices.Num());
        
        for (int32 i = StartIdx; i < EndIdx; ++i)
        {
            int32 AgentIndex = AgentIndices[i];
            if (Registry->IsValidAgentIndex(AgentIndex))
            {
                Processor(Registry, AgentIndex);
            }
        }
    }
}

// Usage example
void UpdateAgentHealthBatch(const TArray<int32>& DamagedAgents)
{
    ProcessAgentsBatch(Registry, DamagedAgents, 
        [](UQAI_AgentRegistry* Reg, int32 AgentIdx)
        {
            float& Health = Reg->GetAgentHealthRef(AgentIdx);
            Health = FMath::Max(0.0f, Health - 10.0f);
        });
}
```

---

## Summary

The QAI plugin provides a robust, high-performance AI system designed for scale and modularity. Key takeaways for developers:

1. **Embrace the Behavior Pattern**: Keep type-specific logic in behavior classes, not in processors
2. **Performance First**: Use SoA layout, batch processing, and SIMD where appropriate
3. **Explicit Architecture**: Set agent archetypes explicitly, don't rely on inference
4. **Clean Interfaces**: Keep behaviors pure and processors generic

The system is designed to be extended easily while maintaining performance and architectural integrity. When adding new AI types, focus on implementing clean behavior classes rather than adding complexity to the core processing pipeline.