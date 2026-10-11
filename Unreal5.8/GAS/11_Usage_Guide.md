# GAS Usage Guide (Unreal Engine 5.8)

This guide documents the end-to-end integration, project configuration, actor architecture, ability/effect runtime flow, cheat tooling, and common pitfalls for the Gameplay Ability System (GAS) in Unreal Engine 5.8. All code examples, architectural descriptions, and configuration properties are strictly grounded in and cited from the Unreal Engine 5.8 source tree under `Engine/Plugins/Runtime/GameplayAbilities/` and `Engine/Source/Runtime/`.

---

## 1. Architecture

The Gameplay Ability System is structured around a centralized actor component (`UAbilitySystemComponent`) that interacts with attribute containers (`UAttributeSet`), runtime-invoked state machines (`UGameplayAbility`), data-driven rulesets (`UGameplayEffect`), decoupled visual/audio notifications (`UGameplayCueSet` / `GameplayCueNotify`), and engine-level global managers (`UAbilitySystemGlobals`, `UGameplayAbilitiesDeveloperSettings`, and `UGameplayCueManager`).

### 1.1 Core Modules and Plugin Dependencies
Defined in `Engine/Plugins/Runtime/GameplayAbilities/GameplayAbilities.uplugin`:
- **Runtime Module**: `GameplayAbilities` (Loading Phase: `PreDefault`, Type: `Runtime`, defined in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesModule.h`).
- **Editor Module**: `GameplayAbilitiesEditor` (Loading Phase: `PreDefault`, Type: `UncookedOnly`, defined in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilitiesEditor/Public/GameplayAbilitiesEditorModule.h`).
- **Required Plugin Dependencies**:
  - `EngineAssetDefinitions` (`Engine/Plugins/Runtime/EngineAssetDefinitions/EngineAssetDefinitions.uplugin`)
  - `GameplayTagsEditor` (`Engine/Plugins/Editor/GameplayTagsEditor/GameplayTagsEditor.uplugin`)
  - `Niagara` (`Engine/Plugins/FX/Niagara/Niagara.uplugin`)
  - `DataRegistry` (`Engine/Plugins/Runtime/DataRegistry/DataRegistry.uplugin`)

In project build configurations (`YourProject.Build.cs`), the module dependencies are declared as:
```csharp
PublicDependencyModuleNames.AddRange(new string[] {
    "Core",
    "CoreUObject",
    "Engine",
    "GameplayTags",      // Engine/Source/Runtime/GameplayTags/GameplayTags.Build.cs
    "GameplayTasks",     // Engine/Source/Runtime/GameplayTasks/GameplayTasks.Build.cs
    "GameplayAbilities"  // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/GameplayAbilities.Build.cs
});
```

### 1.2 Key Types and Responsibilities
- **`UAbilitySystemComponent` (ASC)**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h`
  - Acts as the central hub for granting, executing, replicating, and cancelling abilities, managing active gameplay effects, and storing attribute set references.
- **`IAbilitySystemInterface`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h`
  - Outer interface exposing `GetAbilitySystemComponent()` so external gameplay systems can query an actor's ASC without hard cast coupling.
- **`UAttributeSet` and `FGameplayAttributeData`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h`
  - Encapsulates float gameplay attributes (`BaseValue` and `CurrentValue`). Handles modifier aggregation, clamping (`PreAttributeChange`, `PreAttributeBaseChange`), and gameplay effect execution callbacks (`PreGameplayEffectExecute`, `PostGameplayEffectExecute`).
- **`UGameplayAbility` and `FGameplayAbilitySpec`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpec.h`
  - Implements game mechanics. Wrapped at runtime by `FGameplayAbilitySpec` on the server and replicated via handles (`FGameplayAbilitySpecHandle` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpecHandle.h`).
- **`UGameplayEffect` and `FGameplayEffectSpec`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h`
  - Immutable definitions (`UGameplayEffect`) instantiated into transient context-bearing runtime instances (`FGameplayEffectSpec`) that apply instant, duration, or infinite attribute modifications and tag operations.
- **`UAbilitySystemGlobals` & `UGameplayAbilitiesDeveloperSettings`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h`
  - Global singletons and project configuration classes governing global curves, attribute initters, cue managers, target prediction, and activation failure tags.
- **`UAbilitySystemCheatManagerExtension`**:
  - Source: `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.h`
  - Provides console commands and `UCheatManagerExtension` bindings to grant/activate/cancel abilities and inspect active effects.

---

## 2. Project Setup & Configuration

### 2.1 Enabling the Plugin
Add `GameplayAbilities` to your `.uproject` file:
```json
{
  "Plugins": [
    {
      "Name": "GameplayAbilities",
      "Enabled": true
    }
  ]
}
```

### 2.2 Global Settings (`Config/DefaultGame.ini`)
In UE 5.8, global ability settings are configured via `UGameplayAbilitiesDeveloperSettings` (exposed under Project Settings -> Gameplay Abilities Settings) which backs configuration into `Config/DefaultGame.ini` (see `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:29-120` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:56-120`):

```ini
[/Script/GameplayAbilities.GameplayAbilitiesDeveloperSettings]
; Custom subclass of UAbilitySystemGlobals if overridden by project (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h)
AbilitySystemGlobalsClassName=/Script/GameplayAbilities.AbilitySystemGlobals

; Prediction and tag replication policies (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:69-76)
PredictTargetGameplayEffects=True
ReplicateActivationOwnedTags=True

; Global Cue Manager paths and settings (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:52-61)
GlobalGameplayCueManagerClass=/Script/GameplayAbilities.GameplayCueManager
+GameplayCueNotifyPaths="/Game/GameplayCues"
+GameplayCueNotifyPaths="/Game/Effects/Cues"

; Attribute default curves (CurveTables keyed by AttributeName/Level) (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:45)
+GlobalAttributeSetDefaultsTableNames="/Game/Data/Curves/CT_AttributeDefaults.CT_AttributeDefaults"

; Evaluation channels for Attribute Modifiers (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:107-116)
bAllowGameplayModEvaluationChannels=False
DefaultGameplayModEvaluationChannel=Channel0

; Minimal replication bits for tag count serialization (default: 5) (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:120)
MinimalReplicationTagCountBits=5

; Global Tags for activation failures (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:78-101)
ActivateFailCooldownTag="Ability.ActivationFail.Cooldown"
ActivateFailCostTag="Ability.ActivationFail.Cost"
ActivateFailTagsBlockedTag="Ability.ActivationFail.TagsBlocked"
ActivateFailTagsMissingTag="Ability.ActivationFail.TagsMissing"
ActivateFailNetworkingTag="Ability.ActivationFail.Networking"
```

### 2.3 Global Initialization Requirement (`InitGlobalData`)
`UAbilitySystemGlobals::InitGlobalData()` must be invoked once early during engine/game initialization (e.g. in `UGameInstance::Init()` or `UAssetManager::StartInitialLoading()`), as defined in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:68-72` and implemented in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemGlobals.cpp:168`:
```cpp
#include "AbilitySystemGlobals.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h

void USkyGameInstance::Init()
{
    Super::Init();
    
    // Loads global tag tables, curve tables, and cue caches
    UAbilitySystemGlobals::Get().InitGlobalData();
}
```

### 2.4 Declaring Native Gameplay Tags
In UE 5.8, native tags should be declared using `NativeGameplayTags.h` macros (`UE_DECLARE_GAMEPLAY_TAG_EXTERN` in headers and `UE_DEFINE_GAMEPLAY_TAG` in source files, defined in `Engine/Source/Runtime/GameplayTags/Public/NativeGameplayTags.h:31-41`) rather than raw string lookups:

**In `SkyGameplayTags.h`**:
```cpp
#pragma once

#include "NativeGameplayTags.h" // Engine/Source/Runtime/GameplayTags/Public/NativeGameplayTags.h

namespace SkyGameplayTags
{
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(Ability_Action_Fire);
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(State_Debuff_Stun);
    UE_DECLARE_GAMEPLAY_TAG_EXTERN(GameplayCue_Combat_FireBurst);
}
```

**In `SkyGameplayTags.cpp`**:
```cpp
#include "SkyGameplayTags.h"

namespace SkyGameplayTags
{
    UE_DEFINE_GAMEPLAY_TAG(Ability_Action_Fire, "Ability.Action.Fire");
    UE_DEFINE_GAMEPLAY_TAG(State_Debuff_Stun, "State.Debuff.Stun");
    UE_DEFINE_GAMEPLAY_TAG(GameplayCue_Combat_FireBurst, "GameplayCue.Combat.FireBurst");
}
```

---

## 3. Actor Wiring Architecture

### 3.1 OwnerActor vs. AvatarActor
GAS cleanly separates the conceptual *Owner* of ability data from the physical *Avatar* in the world (`FGameplayAbilityActorInfo` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1523` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityTypes.h:105-188`):
1. **Player Characters**:
   - **Owner**: `APlayerState` (`Engine/Source/Runtime/Engine/Classes/GameFramework/PlayerState.h`). Persists across pawn respawns, deaths, and class switches; ensures scores, cooldowns, and permanent attributes are retained.
   - **Avatar**: `APawn` / `ACharacter` (`Engine/Source/Runtime/Engine/Classes/GameFramework/Character.h`). The physical actor representing the player in the 3D world (receives transforms, skeletal meshes, animations).
2. **AI / Minions / NPCs**:
   - **Owner & Avatar**: `APawn` (`Engine/Source/Runtime/Engine/Classes/GameFramework/Pawn.h`). Suitable when the AI pawn does not respawn or when state does not outlive pawn destruction (as demonstrated in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestPawn.h`).

### 3.2 Where and When to Call `InitAbilityActorInfo`
- **On Pawn / Character (AI / Self-Owned ASC)**:
  - Call `AbilitySystemComponent->InitAbilityActorInfo(this, this)` in `PossessedBy()` on authority (`Engine/Source/Runtime/Engine/Classes/GameFramework/Pawn.h`), and in `OnRep_PlayerState()` or `OnRep_Controller()` on autonomous client proxies.
- **On PlayerState-Owned ASC**:
  - **Server**: In `APawn::PossessedBy(AController* NewController)`:
    ```cpp
    APlayerState* PS = GetPlayerState();
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(PS))
    {
        ASC->InitAbilityActorInfo(PS, this); // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1523
    }
    ```
  - **Client**: In `APawn::OnRep_PlayerState()` or `APlayerController::AcknowledgePossession(APawn* P)` (`Engine/Source/Runtime/Engine/Classes/GameFramework/PlayerController.h`):
    ```cpp
    APlayerState* PS = GetPlayerState();
    if (UAbilitySystemComponent* ASC = UAbilitySystemGlobals::GetAbilitySystemComponentFromActor(PS))
    {
        ASC->InitAbilityActorInfo(PS, this); // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1523
    }
    ```

### 3.3 Modular Gameplay & Game Features Integration
In modular game architectures (such as Epic's GameFeatures plugin and `AbilitySystemGameFeatureActions`), abilities and attribute defaults can be modularly bound without hardcoding:
- **`UGameFeatureAction_AddAttributeDefaults`**:
  - Source: `Engine/Plugins/Experimental/AbilitySystemGameFeatureActions/Source/AbilitySystemGameFeatureActions/Public/GameFeatureAction_AddAttributeDefaults.h`
  - Attaches curve tables containing attribute defaults dynamically during feature registration/activation (`OnGameFeatureActivating`, `OnGameFeatureDeactivating`).
- **`UGameFeatureAction_AddComponents`**:
  - Source: `Engine/Plugins/Runtime/GameFeatures/Source/GameFeatures/Public/GameFeatureAction_AddComponents.h`
  - Allows injecting ability-granting receiver components or attribute sets onto target receiver actor classes when a feature is activated.

---

## 4. Canonical C++ Implementation

The following complete, runnable-style C++ example demonstrates a canonical Pawn wiring grounded in `AbilitySystemTestPawn` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestPawn.h` and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp`), `AbilitySystemTestAttributeSet` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h`), and standard UE 5.8 attribute patterns (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h`).

### 4.1 Header: `SkyCharacterPawn.h`
```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Pawn.h" // Engine/Source/Runtime/Engine/Classes/GameFramework/Pawn.h
#include "AbilitySystemInterface.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h
#include "GameplayCueInterface.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueInterface.h
#include "AttributeSet.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h
#include "AbilitySystemComponent.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h
#include "SkyCharacterPawn.generated.h"

// Standard UE 5.8 accessor macros from Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:420-423
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

class UAbilitySystemComponent;
class UGameplayAbility;

/**
 * Example AttributeSet defining Health and MaxHealth.
 * Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestAttributeSet.h
 */
UCLASS()
class USkyAttributeSet : public UAttributeSet
{
	GENERATED_BODY()

public:
	USkyAttributeSet();

	virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
	virtual void PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue) override;
	virtual void PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data) override;

	UPROPERTY(BlueprintReadOnly, Category = "Attributes|Health", ReplicatedUsing = OnRep_Health)
	FGameplayAttributeData Health;
	ATTRIBUTE_ACCESSORS(USkyAttributeSet, Health)

	UPROPERTY(BlueprintReadOnly, Category = "Attributes|Health", ReplicatedUsing = OnRep_MaxHealth)
	FGameplayAttributeData MaxHealth;
	ATTRIBUTE_ACCESSORS(USkyAttributeSet, MaxHealth)

protected:
	UFUNCTION()
	virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);

	UFUNCTION()
	virtual void OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth);
};

/**
 * Example Pawn implementing IAbilitySystemInterface and IGameplayCueInterface.
 * Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemTestPawn.h
 */
UCLASS()
class ASkyCharacterPawn : public APawn, public IAbilitySystemInterface, public IGameplayCueInterface
{
	GENERATED_BODY()

public:
	ASkyCharacterPawn(const FObjectInitializer& ObjectInitializer);

	// IAbilitySystemInterface (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemInterface.h:23)
	virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;

	virtual void PossessedBy(AController* NewController) override;
	virtual void OnRep_PlayerState() override;

protected:
	virtual void BeginPlay() override;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
	TObjectPtr<UAbilitySystemComponent> AbilitySystemComponent;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Abilities")
	TObjectPtr<USkyAttributeSet> AttributeSet;

	UPROPERTY(EditDefaultsOnly, Category = "Abilities")
	TArray<TSubclassOf<UGameplayAbility>> DefaultAbilities;
};
```

### 4.2 Source: `SkyCharacterPawn.cpp`
```cpp
// Copyright Epic Games, Inc. All Rights Reserved.

#include "SkyCharacterPawn.h"
#include "Net/UnrealNetwork.h" // Engine/Source/Runtime/Net/Core/Public/Net/UnrealNetwork.h
#include "GameplayEffectExtension.h" // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectExtension.h

// ---------------------------------------------------------------------------
// USkyAttributeSet Implementation
// ---------------------------------------------------------------------------

USkyAttributeSet::USkyAttributeSet()
	: Health(100.0f)
	, MaxHealth(100.0f)
{
}

void USkyAttributeSet::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	// Replicate base values to clients with condition None and repnotify ALWAYS
	// Grounded in Engine/Source/Runtime/Engine/Classes/Net/UnrealNetwork.h:47
	DOREPLIFETIME_CONDITION_NOTIFY(USkyAttributeSet, Health, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(USkyAttributeSet, MaxHealth, COND_None, REPNOTIFY_Always);
}

void USkyAttributeSet::PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)
{
	Super::PreAttributeChange(Attribute, NewValue);

	// Clamping current value adjustments (e.g. from buffs/debuffs)
	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:207
	if (Attribute == GetHealthAttribute())
	{
		NewValue = FMath::Clamp(NewValue, 0.0f, GetMaxHealth());
	}
}

void USkyAttributeSet::PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)
{
	Super::PostGameplayEffectExecute(Data);

	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:218
	if (Data.EvaluatedData.Attribute == GetHealthAttribute())
	{
		// Clamp base value after instant damage or healing
		SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
	}
}

void USkyAttributeSet::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:402-405
	GAMEPLAYATTRIBUTE_REPNOTIFY(USkyAttributeSet, Health, OldHealth);
}

void USkyAttributeSet::OnRep_MaxHealth(const FGameplayAttributeData& OldMaxHealth)
{
	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:402-405
	GAMEPLAYATTRIBUTE_REPNOTIFY(USkyAttributeSet, MaxHealth, OldMaxHealth);
}

// ---------------------------------------------------------------------------
// ASkyCharacterPawn Implementation
// ---------------------------------------------------------------------------

ASkyCharacterPawn::ASkyCharacterPawn(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	bReplicates = true;

	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp:14-15
	AbilitySystemComponent = CreateDefaultSubobject<UAbilitySystemComponent>(TEXT("AbilitySystemComponent0"));
	AbilitySystemComponent->SetIsReplicated(true);
	AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Mixed);

	// Construct AttributeSet subobject
	AttributeSet = CreateDefaultSubobject<USkyAttributeSet>(TEXT("AttributeSet0"));
}

UAbilitySystemComponent* ASkyCharacterPawn::GetAbilitySystemComponent() const
{
	// Grounded in Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemTestPawn.cpp:35-38
	return AbilitySystemComponent;
}

void ASkyCharacterPawn::PossessedBy(AController* NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		// Server initializes actor info (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1523)
		AbilitySystemComponent->InitAbilityActorInfo(this, this);

		// Grant default abilities on authority (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:268)
		if (HasAuthority())
		{
			for (const TSubclassOf<UGameplayAbility>& AbilityClass : DefaultAbilities)
			{
				if (AbilityClass)
				{
					FGameplayAbilitySpec Spec(AbilityClass, 1, INDEX_NONE, this);
					AbilitySystemComponent->GiveAbility(Spec);
				}
			}
		}
	}
}

void ASkyCharacterPawn::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	if (AbilitySystemComponent)
	{
		// Client initializes actor info when player state replicates
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}
}

void ASkyCharacterPawn::BeginPlay()
{
	Super::BeginPlay();

	// Fallback for autonomous client in single-player or non-possessed scenarios
	if (AbilitySystemComponent && !HasAuthority() && !AbilitySystemComponent->AbilityActorInfo.IsValid())
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}
}
```

---

## 5. Runtime Ability, Effect, and Cue Flow

### 5.1 Granting Abilities (`GiveAbility`)
Abilities are granted on the authority via `UAbilitySystemComponent::GiveAbility` or `GiveAbilityAndActivateOnce` (defined in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:268-278`):
```cpp
// Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitySpec.h:98-132
FGameplayAbilitySpec AbilitySpec(UGameplayAbility_FireBolt::StaticClass(), /*Level=*/ 1, /*InputID=*/ 0, /*SourceObject=*/ this);
FGameplayAbilitySpecHandle SpecHandle = ASC->GiveAbility(AbilitySpec);
```

### 5.2 Ability Activation Pipeline
When activating an ability locally or predictively:
1. `UAbilitySystemComponent::TryActivateAbility(SpecHandle)` is called (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:352`).
2. The ASC creates or queries the `FGameplayAbilityActorInfo` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityTypes.h:105`).
3. Calls `UGameplayAbility::CanActivateAbility(...)` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:169`):
   - Checks ASC tags against `ActivationBlockedTags` and `ActivationRequiredTags`.
   - Checks `CheckCost()` and `CheckCooldown()` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:207-215`).
4. Enters prediction scope (`FPredictionKey` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:199`).
5. Calls `UGameplayAbility::ActivateAbility(...)` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:191`).
6. Inside `ActivateAbility`, calling `CommitAbility()` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:203`) consumes the cost and applies the cooldown effect.

### 5.3 Applying Gameplay Effects (`MakeEffectContext` & `ApplyGameplayEffectSpecToSelf`)
```cpp
// 1. Create Effect Context (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:879)
FGameplayEffectContextHandle ContextHandle = ASC->MakeEffectContext();
ContextHandle.AddSourceObject(this);
ContextHandle.AddInstigator(GetInstigator(), this);

// 2. Create Spec Handle (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:887)
FGameplayEffectSpecHandle SpecHandle = ASC->MakeOutgoingSpec(DamageEffectClass, /*Level=*/ 1.0f, ContextHandle);
if (SpecHandle.IsValid())
{
    // Assign dynamic SetByCaller values if needed (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:1225)
    SpecHandle.Data->SetSetByCallerMagnitude(SkyGameplayTags::Data_Damage, 25.0f);

    // 3. Apply to Target ASC (Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:894)
    FActiveGameplayEffectHandle ActiveGEHandle = ASC->ApplyGameplayEffectSpecToTarget(*SpecHandle.Data.Get(), TargetASC);
}
```

### 5.4 Dispatching Gameplay Cues
Gameplay Cues represent pure audio and visual effects decoupled from mechanics:
- **Instant (Burst/Static)**:
  Dispatched via `UAbilitySystemComponent::ExecuteGameplayCue` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1260`):
  ```cpp
  FGameplayCueParameters CueParams; // Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectTypes.h:698
  CueParams.Location = HitResult.ImpactPoint;
  CueParams.Normal = HitResult.ImpactNormal;
  CueParams.Instigator = this;
  ASC->ExecuteGameplayCue(SkyGameplayTags::GameplayCue_Combat_FireBurst, CueParams);
  ```
- **Looping (State-Driven / Actor Notifies)**:
  Added via Gameplay Effect components (`UGameplayCueGameplayEffectComponent` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/GameplayCueGameplayEffectComponent.h`) or directly via ASC (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1263-1266`):
  ```cpp
  ASC->AddGameplayCue(SkyGameplayTags::GameplayCue_State_Burning, CueParams);
  // Later:
  ASC->RemoveGameplayCue(SkyGameplayTags::GameplayCue_State_Burning);
  ```

---

## 6. Blueprint Workflow

While core attributes and system hooks are best implemented in C++, GAS is optimized for Blueprint designers:

1. **Creating Gameplay Abilities (`UGameplayAbility`)**:
   - Create Blueprint deriving from `GameplayAbility` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h`).
   - In **Class Defaults**:
     - Set **Instancing Policy**: Prefer `InstancedPerActor` for abilities with state or latent tasks (`EGameplayAbilityInstancingPolicy` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:58`).
     - Set **Net Execution Policy**: `LocalPredicted` for player combat actions; `ServerInitiated` for server-authoritative world mechanics (`EGameplayAbilityNetExecutionPolicy` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:40`).
     - Set **Asset Tags**, **Cancel Abilities with Tag**, and **Block Abilities with Tag**.
   - Implement `Event ActivateAbility`:
     - Place `Commit Ability` node immediately after activation to consume resources (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:203`).
     - Use Latent Ability Tasks (e.g., `Play Montage And Wait` via `UAbilityTask_PlayMontageAndWait` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_PlayMontageAndWait.h`, `Wait Gameplay Event` via `UAbilityTask_WaitGameplayEvent` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/Abilities/Tasks/AbilityTask_WaitGameplayEvent.h`).
     - Always route execution into `End Ability` when the action terminates (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:245`).
2. **Authoring Gameplay Effects (`UGameplayEffect`)**:
   - Create Data-Only Blueprint deriving from `GameplayEffect` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h`).
   - Configure **Duration Policy**: `Instant` (permanent base value delta), `Has Duration` (temporary buff/debuff), or `Infinite` (`EGameplayEffectDurationType` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffect.h:47`).
   - In UE 5.3+, behaviors are organized via **Gameplay Effect Components** (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponent.h`):
     - `TargetTagsGameplayEffectComponent` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/TargetTagsGameplayEffectComponent.h`): Grants or removes tags while active.
     - `UGameplayCueGameplayEffectComponent` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/GameplayCueGameplayEffectComponent.h`): Triggers audio/Niagara cues on application.
     - `UChanceToApplyGameplayEffectComponent` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayEffectComponents/ChanceToApplyGameplayEffectComponent.h`): Probability rolls.
3. **Implementing Gameplay Cue Notifies**:
   - **One-Shot**: Derive from `UGameplayCueNotify_Static` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Static.h`) or `AGameplayCueNotify_BurstLatent` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_BurstLatent.h`). Implements `OnExecute`.
   - **Looping**: Derive from `AGameplayCueNotify_Looping` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueNotify_Looping.h`). Implements `OnBecomeRelevant` (spawns looping audio/Niagara particles) and `OnCeaseRelevant` (cleans up FX upon removal).

---

## 7. Developer Tooling & Cheat Commands

UE 5.8 exposes a rich suite of built-in console commands defined in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:580-680`, `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.h:10-49`, and `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h`:

### 7.1 Built-in Console Commands
- `AbilitySystem.Ability.Grant <ClassName/AssetName>`:
  Grants an ability to the local player on the server (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:581`).
- `AbilitySystem.Ability.Activate [-Server] <TagName/ClassName/AssetName>`:
  Activates an ability matching the specified tag or asset name (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:613`).
- `AbilitySystem.Ability.Cancel [-Server] <PartialName>`:
  Cancels a running ability (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:597`).
- `AbilitySystem.Ability.ListGranted`:
  Lists all abilities currently granted to the local player ASC (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:632`).
- `AbilitySystem.Effect.Apply [-Server] <Class/AssetName> [Level]`:
  Applies a gameplay effect to the player (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:675`).
- `AbilitySystem.Effect.ListActive [-Server]`:
  Dumps active gameplay effects and durations (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:642`).
- `AbilitySystem.Effect.Remove [-Server] <Handle/Name>`:
  Removes an active gameplay effect by name or handle (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AbilitySystemCheatManagerExtension.cpp:648`).

### 7.2 Gameplay Cue Debugging Commands
- `AbilitySystem.DisplayGameplayCues <1/0>`: Renders 3D in-world debug text for cue events (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:115`).
- `AbilitySystem.DisableGameplayCues <1/0>`: Disables all cue events globally for performance/isolation testing (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:125`).
- `AbilitySystem.LogGameplayCueActorSpawning <1/0>`: Logs instantiation of `GameplayCueNotify_Actor` instances (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:110`).
- `AbilitySystem.GameplayCueCheckForTooManyRPCs <1/0>`: Warns when cues exceed RPC limits (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayCueManager.cpp:140`).

### 7.3 Gameplay Debugger & Visual Logger
- Press `'` (Apostrophe) or `Shift + '` to open the **Gameplay Debugger** category for Ability System (`FGameplayDebuggerCategory_Abilities` in `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayDebuggerCategory_Abilities.h`). Visualizes active tags, granted abilities, active effects, and attribute values with server/client color coding.
- Open **Visual Logger** (`vislog` console command, `Engine/Source/Runtime/Engine/Classes/VisualLogger/VisualLogger.h`) to inspect frame-by-frame activation histories and replicated ASC state dumps.

---

## 8. Common Pitfalls & Solutions

### 8.1 Prediction Key Misuse and Dependency Chains
- **Pitfall**: Attempting to predict actions outside of a valid client prediction scope, or misconfiguring prediction dependency chains.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayPrediction.cpp:32-37` (`CVarDependentChainBehaviorValue`)
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/Tests/PredictionKeyTests.cpp:18-22`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayPrediction.h:199-280` (`FPredictionKey`)
- **Details**:
  - The client can only predict actions initiated by direct user input (where `ASC->ScopedPredictionKey` is active).
  - Calling `ApplyGameplayEffectSpecToTarget` predictively on a foreign target (another player) fails unless `PredictTargetGameplayEffects` is explicitly enabled in `UGameplayAbilitiesDeveloperSettings:69` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:69`).
  - In UE 5.8, `AbilitySystem.PredictionKey.DepChainBehavior` defaults to `1` (bit 0x1: newer accepted keys imply older accepted keys; bit 0x2: older accepted does not imply newer accepted). Failing to handle dependent keys properly causes rejected predictions to fail rollbacks or leak unacknowledged keys.

### 8.2 Attribute Initialization Timing and Replicated Base Values
- **Pitfall**: Manually modifying attribute values in constructors or bypassing `InitAbilityActorInfo` before initializing default values.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:402-405` (`GAMEPLAYATTRIBUTE_REPNOTIFY`)
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1523` (`InitAbilityActorInfo`)
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/AttributeSet.cpp:45-80`
- **Details**:
  - Direct assignment to `Health = 100.f` skips aggregator initialization. Use `InitHealth(100.f)` via `ATTRIBUTE_ACCESSORS` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AttributeSet.h:451-454`).
  - On clients, `OnRep_Health` MUST invoke `GAMEPLAYATTRIBUTE_REPNOTIFY(UMySet, Health, OldValue)` so that the ASC's internal aggregator updates its base value via `UAbilitySystemComponent::SetBaseAttributeValueFromReplication` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1609`).
  - Applying initial effects before calling `InitAbilityActorInfo` results in crashes or unowned effects because the ASC has no valid avatar or owner pointer.

### 8.3 Gameplay Cue Relevancy and Proxy Culling
- **Pitfall**: Triggering looping cues via unreliable burst RPCs, or failing to configure replication proxies for player states.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/README.md:193-204`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemReplicationProxyInterface.h:14-30`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayCueInterface.h:199` (`FMinimalGameplayCueReplicationProxy`)
- **Details**:
  - Burst cues (`OnExecute`) replicate via unreliable multicast RPCs. If a client is outside the network relevancy bubble when the RPC fires, it will never receive it.
  - When the ASC lives on `APlayerState`, the player state is always relevant, but the pawn has a spatial relevancy distance. Without implementing `IAbilitySystemReplicationProxyInterface` on the Pawn and enabling `bReplicationProxyEnabled` on the ASC (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:2075`), unreliable cue RPCs can be dispatched to remote clients regardless of distance or culled incorrectly.
  - In looping notifies, never pair `OnBurst` (unreliable) with `OnCeaseRelevant` without implementing `OnBecomeRelevant`, as clients becoming relevant midway will not run the spawn FX but will attempt cleanup.

### 8.4 InstancedPerActor Variable Retention Across Activations
- **Pitfall**: Assuming member variables in a `UGameplayAbility` reset between consecutive activations.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/README.md:86-90`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:58-65` (`InstancedPerActor`)
- **Details**:
  - With `InstancingPolicy = InstancedPerActor`, the ability object is allocated once and persists for the lifetime of the grant.
  - Variables modified during execution (e.g., target arrays, combo counts, timers) retain their values when the ability finishes.
  - If they are not explicitly reset in `EndAbility(const FGameplayAbilitySpecHandle Handle, ...)` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:245`) or at the beginning of `ActivateAbility(...)`, the next activation starts with corrupt state.

### 8.5 Replicated Properties Deprecation in Gameplay Abilities
- **Pitfall**: Using `UPROPERTY(Replicated)` inside `UGameplayAbility` subclasses.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/README.md:110-124`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h`
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Private/GameplayAbility.cpp:112` (`AbilitySystem.DeprecateReplicatedProperties`)
- **Details**:
  - Replicating member variables directly on `UGameplayAbility` instances is deprecated starting in UE 5.5 (`AbilitySystem.DeprecateReplicatedProperties`).
  - Object property replication does not guarantee packet ordering relative to RPC activation flows (`ServerTryActivateAbility`). Reading a replicated property inside `ActivateAbility` often yields stale data.
  - **Solution**: Use Reliable RPCs, `FGameplayEventData` payloads, or target data structs (`FGameplayAbilityTargetDataHandle`) with `FScopedServerAbilityRPCBatcher` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilityTypes.h:740`) to guarantee deterministic arrival.

### 8.6 Tag Lifecycle: `ActivationOwnedTags` vs `LooseGameplayTags`
- **Pitfall**: Manually modifying tags on the ASC using `AddLooseGameplayTag` inside an ability without removing them on cancellation.
- **Source Grounding**:
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemGlobals.h:107` (`ShouldReplicateActivationOwnedTags`)
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbilitiesDeveloperSettings.h:75-77` (`ReplicateActivationOwnedTags`)
  - `Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/AbilitySystemComponent.h:1450-1470` (`AddLooseGameplayTag`, `RemoveLooseGameplayTag`)
- **Details**:
  - `LooseGameplayTags` on the ASC are not automatically tied to ability lifecycles and do not replicate unless explicitly configured via minimal replication containers.
  - Tags that should represent an ability being active must be placed in `ActivationOwnedTags` on the `UGameplayAbility` (`Engine/Plugins/Runtime/GameplayAbilities/Source/GameplayAbilities/Public/GameplayAbility.h:345`). These tags are automatically applied when the ability activates, removed when the ability ends (even on abnormal abortion or death), and replicated to clients according to `ReplicateActivationOwnedTags = true`.
