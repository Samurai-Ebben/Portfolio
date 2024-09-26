![Hell_Racer_logo.png](/HellRacer/Images/Hell_Racer_logo.png)

![](/HellRacer/Images/Ghost.png)    |  ![](/HellRacer/Images/Counter.gif)
:-------------------------:|:-------------------------:
 ![](/HellRacer/Images/Drifting.gif) | ![](/HellRacer/Images/Driving.png)
## Summary of the project
Hell Racer was an 8 weeks game project in Yrgo developed by 8 members. The 4 programmers wanted to experince in making a game in Unreal Engine 5 using only C++.
Hence, Hell Racer.

[Hell Racer ITCH.IO](https://yrgo-game-creator.itch.io/hellracer)

## About
Hell Racer is a single player kart game set in hell. You play as an imp and will race the ghost of your own racing lap records.

## Responsibilities
A kart game involves numerous mechanics, but to ensure the gameplay is fun and exciting, the most crucial aspect is movement. Among these, the drifting mechanic stands out as particularly important. I was primarily responsible for developing this key drifting mechanic, along with creating the ghost system and a significant portion of the user interface (UI).

### Drift
The drift mechanic was the toughest to implement. Not for the fact that i never actually drifted before. But because applying the *Centripetal force* and/or *centrifugal force* to a fake physics system for the acceleration of the car is tricky. However, after iterating and taking many feedback from testers, The drift became better and granted a great experince for players.

In *HellRacer*, we wanted to reward players for drifting, so I added a feature where drifting builds up a charge for a boost.


![](/HellRacer/Images/Drifting.gif)

<details>
 <summary>Drift</summary>

 ```CPP
void ACharacterInput::StartDrift(const FInputActionValue& Value)
{
	
	if (!bDrifting && fabs(SteeringInput) > .96f) {
		bDrifting = true;
		MovementComponent->bIsDrifting = bDrifting;
		DriftLockCounter = DriftLockTimer;

		CurrentDrivingState = EDrivingState::S_Drifting;
		InitialDriftDirection = GetActorForwardVector();

		//Lerping from the current rotation to the desired one.
		MovementComponent->WorldRotateSpeed = FMath::Lerp(MovementComponent->SetWorldRotationHighSpeed, 170, .5f);
		InitialSteeringInput = SteeringInput;
		MovementComponent->CharacterMovementComponent->GroundFriction = 3.8f;
		DriftAudio->Play();

		DriftSpark1Effect = UNiagaraFunctionLibrary::SpawnSystemAttached(DriftParticles, CarMesh, NAME_None,
			FVector(-75.f, -50.f, 5.f), FRotator(0.F), EAttachLocation::SnapToTarget, true, true);

		DriftSpark2Effect = UNiagaraFunctionLibrary::SpawnSystemAttached(DriftParticles, CarMesh, NAME_None,
			FVector(-75.f, 50.f, 5.f), FRotator(0.F), EAttachLocation::SnapToTarget, true, true);
	}
}

void ACharacterInput::StopDrift(const FInputActionValue& Value)
{
	if (CarIsMovingBackWard) { return; }

	if (bDrifting)
	{
		bDrifting = false;
		MovementComponent->WorldRotateSpeed = FMath::Lerp(150, MovementComponent->SetWorldRotationHighSpeed, .3f);
		MovementComponent->CharacterMovementComponent->GroundFriction = 8.f;
		MovementComponent->bIsDrifting = false;
		DriftLockCounter = 0;
		CurrentDrivingState = EDrivingState::S_Driving;
		DriftComp->StopDrift();
		DriftAudio->Stop();

		DriftSpark2Effect->Deactivate();
		DriftSpark1Effect->Deactivate();

		if (bDriftBoostCharged) {
			MovementComponent->AddBoostToVelocity(1830, 1830);
			DriftBoostCounter = DriftBoostTimer;
			bDriftBoosted = true;
		}

		DriftTimer = 0;
		bDriftBoostCharged = false;
	}
}
 ```
</details>

------ 

### Ghost kart
The ghost kart is the last best saved race of the player. It gives the player a challenge to break their own record or someone elses.

![](/HellRacer/Images/Ghost.png)

<details>
 
 <summary> GhostRecorder script</summary>
 
 ```CPP
       void AGhostRecorder::StartRecording()
       {
        	RecordedData.Empty();
        	StartTime = GetWorld()->GetTimeSeconds();
        	bIsRecording = true;
        	GetWorld()->GetTimerManager().SetTimer(TimerHandle, this, &AGhostRecorder::CaptureDataPoint, .40f, true);
       }
       
       void AGhostRecorder::StopRecording(bool IsHighScoreBrocken)
       {
       
       	GetWorld()->GetTimerManager().ClearTimer(TimerHandle);
       
       	bIsRecording = false;
       
       	//Check if it is higher(less) than the highest score and then save it.
       	if (IsHighScoreBrocken) {
       		SaveToFile();
       	}
       }
       
       void AGhostRecorder::CaptureDataPoint()
       {
       	if (!bIsRecording)
       		return;
       
       	ACharacterInput* player = Cast<ACharacterInput>(UGameplayStatics::GetPlayerCharacter(GetWorld(), 0));
       	FGhostDataPoint DataPoint;
       	DataPoint.Timestamp = GetWorld()->GetTimeSeconds() - StartTime;
       	DataPoint.Position = player->GetActorLocation();
       	DataPoint.Rotation = player->GetActorRotation();
       	DataPoint.Velocity = player->VelocityFloat;
       
       	RecordedData.Add(DataPoint);
       }
       
       void AGhostRecorder::SaveToFile()
       {
       	FString FileName = FString(TEXT("GhostData.json")); //The file where the data will be saved. Json cuz easier!
       	// Saving the data where the current project is located plus adding the data file
       	FString SavePath = FPaths::ProjectSavedDir() + FileName;
       	FString OutputString;
       	TSharedRef<TJsonWriter<>> DataWriter = TJsonWriterFactory<>::Create(&OutputString);
       	DataWriter->WriteObjectStart();
       	DataWriter->WriteArrayStart(TEXT("GhostPoints"));
       
       	for (const FGhostDataPoint& DataPoint : RecordedData)
       	{
       		DataWriter->WriteObjectStart();  // Start of a GhostDataPoint object
       		DataWriter->WriteValue(TEXT("Timestamp"), DataPoint.Timestamp);
       		DataWriter->WriteValue(TEXT("PositionX"), DataPoint.Position.X);
       		DataWriter->WriteValue(TEXT("PositionY"), DataPoint.Position.Y);
       		DataWriter->WriteValue(TEXT("PositionZ"), DataPoint.Position.Z);
       		DataWriter->WriteValue(TEXT("RotationYaw"), DataPoint.Rotation.Yaw);
       		DataWriter->WriteValue(TEXT("RotationPitch"), DataPoint.Rotation.Pitch);
       		DataWriter->WriteValue(TEXT("RotationRoll"), DataPoint.Rotation.Roll);
       		DataWriter->WriteValue(TEXT("Velocity"), DataPoint.Velocity);
       		DataWriter->WriteObjectEnd();  // MUST: End of a GhostDataPoint object
       	}
       
       	DataWriter->WriteArrayEnd();
       	DataWriter->WriteObjectEnd();
       	DataWriter->Close();
       
       	FFileHelper::SaveStringToFile(OutputString, *SavePath);
       }
       
       void AGhostRecorder::LoadFromFile()
       {
       	FString FileName = FString(TEXT("GhostData.json"));
       	FString LoadPath = FPaths::ProjectSavedDir() + FileName;
       	FString ResString;
       
       	if (FFileHelper::LoadFileToString(ResString, *LoadPath))
       	{
       		TSharedPtr<FJsonObject> JsonObject;
       		// A reader this time. Everything read will be saved to the ResString
       		TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(ResString);
       
       		if (FJsonSerializer::Deserialize(Reader, JsonObject))
       		{
       			TArray<TSharedPtr<FJsonValue>> Points = JsonObject->GetArrayField(TEXT("GhostPoints"));
       
       			for (int32 i = 0; i < Points.Num(); i++)
       			{
       				TSharedPtr<FJsonObject> Point = Points[i]->AsObject();
       				FGhostDataPoint DataPoint;
       				DataPoint.Timestamp = Point->GetNumberField(TEXT("Timestamp"));
       				DataPoint.Position.X = Point->GetNumberField(TEXT("PositionX"));
       				DataPoint.Position.Y = Point->GetNumberField(TEXT("PositionY"));
       				DataPoint.Position.Z = Point->GetNumberField(TEXT("PositionZ"));
       				DataPoint.Rotation.Yaw = Point->GetNumberField(TEXT("RotationYaw"));
       				DataPoint.Rotation.Pitch = Point->GetNumberField(TEXT("RotationPitch"));
       				DataPoint.Rotation.Roll = Point->GetNumberField(TEXT("RotationRoll"));
       				DataPoint.Velocity = Point->GetNumberField(TEXT("Velocity"));
       
       				RecordedData.Add(DataPoint);
       			}
       		}
       	}
       }
 ```

</details>

--------------------------

### The Launcher
The launcher serves as a crucial component in any racing game, playing an integral role in enhancing gameplay dynamics. In this particular kart racing game, the launcher functions as a mini-game, where players must strategically accelerate, aligning the arrow with the green zone to achieve an optimal start advantage.

Oh, and in case you haven’t noticed, there’s a giant eyeball in the background. I synced the leds animations—blinking and changing pupil colors—with the launcher sequence. After the launch, the eye flies to the starting line and just stares at the players when they get close, adding a bit of a creepy vibe!

![](/HellRacer/Images/Counter.gif)

<details>
 <summary>Launcher script</summary>
 
 ```CPP
   void ULauncherUID::NativeConstruct()
  {
  	Super::NativeConstruct();
  	bMiniGameActive = false;
  	IndicatorRotation = -90.f;
  	IndicatorSpeed = 30;
  
  	if (Indicator)
  	{
  		Indicator->SetRenderTransformAngle(IndicatorRotation);
  	}
  	FindEyeCounterActor();
  	
  
  }
  
  void ULauncherUID::NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
  {
  	Super::NativeTick(MyGeometry, InDeltaTime);
  
  	UpdateIndicatorPos();
  }
  
  void ULauncherUID::StartCountdown()
  {
  	bMiniGameActive = true;
  	GetWorld()->GetTimerManager().SetTimer(TimerHandle, this, &ULauncherUID::UpdateCountdown, 1.0f, true);	
  }
  
  void ULauncherUID::UpdateCountdown()
  {
  	AEyeCounter* EyeCounter = Cast<AEyeCounter>(EyeCounterActor);
  	if (EyeCounter)
  	{
  		EyeCounter->ChangeEyeTexture(Countdown - 1);
  	}
  
  	if (Countdown > 0) {
  		TxtBlockTimer->SetText(FText::AsNumber(Countdown));
  		Countdown--;
  	}
  	else {
  		Countdown = 0;
  		EyeCounter->StopAnim();
  		FinishCountdown();
  	}
  }
  
  void ULauncherUID::StopMiniGame()
  {
  	TargetZoneCenter = 70.0f;
  	TargetZoneTolerance = 37.0f;
  
  	if (IndicatorRotation >= TargetZoneTolerance && IndicatorRotation <= TargetZoneCenter) {
  		//Player gets A BOOOOOOOST!
  		bTargetHit = true;
  	}
  }
  
  void ULauncherUID::FinishCountdown()
  {
  	bMiniGameActive = false;
  	GetWorld()->GetTimerManager().ClearTimer(TimerHandle);
  	TxtBlockTimer->SetText(FText::FromString("GO!"));
  	bCountdownFinish = true;
  	StopMiniGame();
  }
  
  bool ULauncherUID::GetCountdownFinished()
  {
  	return ULauncherUID::bCountdownFinish;
  }
  
  void ULauncherUID::UpdateIndicatorPos()
  {
  	if (IndicatorRotation >= TargetZoneTolerance && IndicatorRotation <= TargetZoneCenter) {
  		APlayerController* PlayerController = GetWorld()->GetFirstPlayerController();
  		if (PlayerController) {
  			PlayerController->PlayDynamicForceFeedback(.5f, .3f, false, true, false, true);
  		}
  	}
  	else {
  		APlayerController* PlayerController = GetWorld()->GetFirstPlayerController();
  		if (PlayerController) {
  			PlayerController->PlayDynamicForceFeedback(.0f, -1, false, true, false, true);
  		}
  	}

  	IndicatorRotation = FMath::Clamp(IndicatorRotation, -90, 90);
  	if (bPressingAcceleration) {
  		IndicatorRotation = FMath::Fmod(IndicatorRotation + IndicatorSpeed * 0.1f, 180.0f);
  	}
  	else {
  		IndicatorRotation = FMath::Fmod(IndicatorRotation - IndicatorSpeed * 0.05f, 180.0f);
  	}
  
  	if (Indicator)
  	{
  		Indicator->SetRenderTransformAngle(IndicatorRotation);
  	}
  }
  
  void ULauncherUID::RiseBar()
  {
  	bPressingAcceleration = true;
  }
  
  bool ULauncherUID::TargetHit()
  {
  	return bTargetHit;
  }
  
  void ULauncherUID::SetTargetHit(bool newVal) {
  	bTargetHit = newVal;
  }
  
  void ULauncherUID::LowerBar()
  {
  	bPressingAcceleration = false;
  }
  
  void ULauncherUID::FindEyeCounterActor()
  {
  	EyeCounterActor = UGameplayStatics::GetActorOfClass(GetWorld(), AEyeCounter::StaticClass());
  	if (!EyeCounterActor)
  	{
  		UE_LOG(LogTemp, Warning, TEXT("EyeCounter not found in the world."));
  	}
  }
 ```

</details>

<details>
	<summary>Eye Counter</summary>
	
```CPP
 		AEyeCounter::AEyeCounter()
		{
			// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
			PrimaryActorTick.bCanEverTick = true;
			bShouldMove = false;
			EyeMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("EyeMesh"));
			EyeLock = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("EyeLock"));
			DirectionIndicator = CreateDefaultSubobject<UArrowComponent>(TEXT("DirectionIndicator"));
		
			EyeLock->SetupAttachment(EyeMesh);
			RootComponent = EyeMesh;
			MoveSpeed = 200.0f;
			DetectionRadius = 5000.0f;
			TargetLocation = GetActorLocation();
			CountdownAudio = CreateDefaultSubobject<UAudioComponent>("Countdown Audio Component");
		
		}
		
		// Called when the game starts or when spawned
		void AEyeCounter::BeginPlay()
		{
			Super::BeginPlay();
			if (EyeLock)
			{
				EyeLockAnimInstance = Cast<UEyeLockAnimInstance>(EyeLock->GetAnimInstance());
				if (!EyeLockAnimInstance)
				{
					UE_LOG(LogTemp, Warning, TEXT("AnimInstance is null"));
				}
			}
			PlayerActor = UGameplayStatics::GetPlayerPawn(GetWorld(), 0);
			EyeLockAnimInstance->EyeLockAnimationSpeed = 0.f;
		
		}
		
		// Called every frame
		void AEyeCounter::Tick(float DeltaTime)
		{
			Super::Tick(DeltaTime);
		
			//if(bAnimate)
				
			if (bShouldMove)
			{
				FVector CurrentLocation = GetActorLocation();
				FVector NewLocation = FMath::VInterpConstantTo(CurrentLocation, DirectionIndicator->GetComponentLocation(), DeltaTime, MoveSpeed);
				SetActorLocation(NewLocation);
		
				// Stop moving if close enough to the target location
				if (FVector::Dist(NewLocation, TargetLocation) < 1.0f)
				{
					bShouldMove = false;
				}
			}
		
			if (IsPlayerWithinRadius())
			{
				RotateToPlayer();
			}
		}
		
		void AEyeCounter::ChangeEyeTexture(int32 Index)
		{
			if (EyeLockAnimInstance)
			{
				EyeLockAnimInstance->EyeLockAnimationSpeed = 100.f;
			}
		
			CountdownAudio->SetSound(CountdownSound);
			CountdownAudio->Play();
			if(Index >= 0 && Index < Materials.Num())
				EyeMesh->SetMaterial(0, Materials[Index]);
		}
		
		void AEyeCounter::StopAnim()
		{
			bAnimate = false;
			bShouldMove = true;
			EyeLockAnimInstance->EyeLockAnimationSpeed = 2;
			EyeMesh->SetMaterial(0, Materials[2]);
			CountdownAudio->Stop();
		
		}
		
		void AEyeCounter::MoveToLocation(FVector NewTargetLocation)
		{
			TargetLocation = NewTargetLocation;
			bShouldMove = true;
		}
		
		bool AEyeCounter::IsPlayerWithinRadius()
		{
			if (!PlayerActor) return false;
		
			float DistanceToPlayer = FVector::Dist(PlayerActor->GetActorLocation(), GetActorLocation());
			return DistanceToPlayer <= DetectionRadius;
		}
		
		void AEyeCounter::RotateToPlayer()
		{
			if (!PlayerActor) return;
		
			FVector DirectionToPlayer = PlayerActor->GetActorLocation() - GetActorLocation();
			FRotator LookAtRotation = FRotationMatrix::MakeFromX(DirectionToPlayer).Rotator() - FRotator(0,90,0);
			LookAtRotation.Pitch = 0.0f;
			//LookAtRotation.Roll = 0.0f;
		
			SetActorRotation(FMath::RInterpTo(GetActorRotation(), LookAtRotation, GetWorld()->GetDeltaSeconds(), 5.0f));
		}
```
 
</details>
