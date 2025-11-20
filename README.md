# Zero Horizon
_A game by Horizon Studios_

## Overview
ZeroHorizonGame is an Unreal Engine sample focused on scalable multiplayer gameplay. It ships as a single primary module (`ZeroHorizonGame`) with dedicated Editor/Server targets. Runtime entry lives in `ZeroHorizonGameModule.cpp`, which extends `FDefaultGameModuleImpl` and hooks into Unreal’s startup/shutdown lifecycle before handing control to the engine-managed gameplay loop.

```
class FZeroHorizonGameModule : public FDefaultGameModuleImpl
{
	virtual void StartupModule() override
	{
	}

	virtual void ShutdownModule() override
	{
	}
};

IMPLEMENT_PRIMARY_GAME_MODULE(FZeroHorizonGameModule, ZeroHorizonGame, "ZeroHorizonGame");
```

## Build & Platform Targets
`ZeroHorizonGame.Build.cs` enumerates the module’s dependencies, exposing the systems the sample leans on (Gameplay Ability System, Modular Gameplay, Niagara, CommonUI, networking, etc.). This file is the authoritative source for anyone adding third-party code or toggling build features.

```
PublicDependencyModuleNames.AddRange(
	new string[] {
		"Core",
		"CoreOnline",
		"CoreUObject",
		"ApplicationCore",
		"Engine",
		"PhysicsCore",
		"GameplayTags",
		"GameplayTasks",
		"GameplayAbilities",
		"AIModule",
		"ModularGameplay",
		"ModularGameplayActors",
		"DataRegistry",
		"ReplicationGraph",
		"GameFeatures",
		"SignificanceManager",
		"Hotfix",
		"CommonLoadingScreen",
		"Niagara",
		"AsyncMixin",
		"ControlFlows",
		"PropertyPath"
	}
);

PrivateDependencyModuleNames.AddRange(
	new string[] {
		"InputCore",
		"Slate",
		"SlateCore",
		"RenderCore",
		"DeveloperSettings",
		"EnhancedInput",
		"NetCore",
		"RHI",
		"Projects",
		"Gauntlet",
		"UMG",
		"CommonUI",
		"CommonInput",
		"GameSettings",
		"CommonGame",
		"CommonUser",
		"GameSubtitles",
		"GameplayMessageRuntime",
		"AudioMixer",
		"NetworkReplayStreaming",
		"UIExtension",
		"ClientPilot",
		"AudioModulation",
		"EngineSettings",
		"DTLSHandlerComponent",
	}
);
```

## Gameplay Framework

### Character & Pawn Pipeline
`AZeroHorizonCharacter` owns capsule/mesh setup, injects custom movement (`UZeroHorizonCharacterMovementComponent`), wires the Pawn Extension, Health, and Camera components, and orchestrates replication-safe interactions with the Ability System and team interface. This class is the default pawn for both human and AI controllers.

```
AZeroHorizonCharacter::AZeroHorizonCharacter(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer.SetDefaultSubobjectClass<UZeroHorizonCharacterMovementComponent>(ACharacter::CharacterMovementComponentName))
{
	PrimaryActorTick.bCanEverTick = false;
	PrimaryActorTick.bStartWithTickEnabled = false;
	NetCullDistanceSquared = 900000000.0f;

	UCapsuleComponent* CapsuleComp = GetCapsuleComponent();
	check(CapsuleComp);
	CapsuleComp->InitCapsuleSize(40.0f, 90.0f);
	CapsuleComp->SetCollisionProfileName(NAME_ZeroHorizonCharacterCollisionProfile_Capsule);

	USkeletalMeshComponent* MeshComp = GetMesh();
	check(MeshComp);
	MeshComp->SetRelativeRotation(FRotator(0.0f, -90.0f, 0.0f));
	MeshComp->SetCollisionProfileName(NAME_ZeroHorizonCharacterCollisionProfile_Mesh);

	UZeroHorizonCharacterMovementComponent* ZeroHorizonMoveComp = CastChecked<UZeroHorizonCharacterMovementComponent>(GetCharacterMovement());
	...
	PawnExtComponent = CreateDefaultSubobject<UZeroHorizonPawnExtensionComponent>(TEXT("PawnExtensionComponent"));
	...
	HealthComponent = CreateDefaultSubobject<UZeroHorizonHealthComponent>(TEXT("HealthComponent"));
	...
	CameraComponent = CreateDefaultSubobject<UZeroHorizonCameraComponent>(TEXT("CameraComponent"));
	...
}
```

### Player Lifecycle
`AZeroHorizonPlayerController` manages autorun steering, replicated view state, replay capture, and pipes local input into the Ability System each frame, ensuring consistent activation on both server and client.

```
void AZeroHorizonPlayerController::PlayerTick(float DeltaTime)
{
	Super::PlayerTick(DeltaTime);

	if (GetIsAutoRunning())
	{
		if (APawn* CurrentPawn = GetPawn())
		{
			const FRotator MovementRotation(0.0f, GetControlRotation().Yaw, 0.0f);
			const FVector MovementDirection = MovementRotation.RotateVector(FVector::ForwardVector);
			CurrentPawn->AddMovementInput(MovementDirection, 1.0f);	
		}
	}

	AZeroHorizonPlayerState* ZeroHorizonPlayerState = GetZeroHorizonPlayerState();

	if (PlayerCameraManager && ZeroHorizonPlayerState)
	{
		...
	}
}

void AZeroHorizonPlayerController::PostProcessInput(const float DeltaTime, const bool bGamePaused)
{
	if (UZeroHorizonAbilitySystemComponent* ZeroHorizonASC = GetZeroHorizonAbilitySystemComponent())
	{
		ZeroHorizonASC->ProcessAbilityInput(DeltaTime, bGamePaused);
	}

	Super::PostProcessInput(DeltaTime, bGamePaused);
}
```

### Game Mode & Experiences
`AZeroHorizonGameMode` binds together the core gameplay types (state, session, controller, pawn) and dynamically selects a gameplay “experience” through configs, URL parameters, or defaults before spawning players. The experience determines pawn data, game features, and content to load.

```
AZeroHorizonGameMode::AZeroHorizonGameMode(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	GameStateClass = AZeroHorizonGameState::StaticClass();
	GameSessionClass = AZeroHorizonGameSession::StaticClass();
	PlayerControllerClass = AZeroHorizonPlayerController::StaticClass();
	ReplaySpectatorPlayerControllerClass = AZeroHorizonReplayPlayerController::StaticClass();
	PlayerStateClass = AZeroHorizonPlayerState::StaticClass();
	DefaultPawnClass = AZeroHorizonCharacter::StaticClass();
	HUDClass = AZeroHorizonHUD::StaticClass();
}

void AZeroHorizonGameMode::HandleMatchAssignmentIfNotExpectingOne()
{
	FPrimaryAssetId ExperienceId;
	FString ExperienceIdSource;

	// Precedence order (highest wins)
	//  - Matchmaking assignment (if present)
	//  - URL Options override
	//  - Developer Settings (PIE only)
	//  - Command Line override
	//  - World Settings
	//  - Dedicated server
	//  - Default experience

	UWorld* World = GetWorld();

	if (!ExperienceId.IsValid() && UGameplayStatics::HasOption(OptionsString, TEXT("Experience")))
	{
		...
	}

	...

	if (!ExperienceId.IsValid())
	{
		...
		ExperienceId = FPrimaryAssetId(FPrimaryAssetType("ZeroHorizonExperienceDefinition"), FName("B_ZeroHorizonDefaultExperience"));
		ExperienceIdSource = TEXT("Default");
	}

	OnMatchAssignmentGiven(ExperienceId, ExperienceIdSource);
}
```

## Gameplay Ability System
`UZeroHorizonAbilitySystemComponent` extends GAS with ZeroHorizon-specific input routing, activation groups, tag relationship mapping, and auto-activation on spawn. It brokers input presses/holds/releases, cancels mutually exclusive abilities, and syncs with the animation system.

```
void UZeroHorizonAbilitySystemComponent::TryActivateAbilitiesOnSpawn()
{
	ABILITYLIST_SCOPE_LOCK();
	for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
	{
		if (const UZeroHorizonGameplayAbility* ZeroHorizonAbilityCDO = Cast<UZeroHorizonGameplayAbility>(AbilitySpec.Ability))
		{
			ZeroHorizonAbilityCDO->TryActivateAbilityOnSpawn(AbilityActorInfo.Get(), AbilitySpec);
		}
	}
}
```

## Asset & Data Flow
`UZeroHorizonAssetManager` centralizes startup loading, ensuring gameplay cues and the primary `UZeroHorizonGameData` asset are ready before experiences run. Startup jobs are weighted so UI loading screens can report progress consistently.

```
void UZeroHorizonAssetManager::StartInitialLoading()
{
	SCOPED_BOOT_TIMING("UZeroHorizonAssetManager::StartInitialLoading");

	// This does all of the scanning, need to do this now even if loads are deferred
	Super::StartInitialLoading();

	STARTUP_JOB(InitializeGameplayCueManager());

	{
		// Load base game data asset
		STARTUP_JOB_WEIGHTED(GetGameData(), 25.f);
	}

	// Run all the queued up startup jobs
	DoAllStartupJobs();
}
```

