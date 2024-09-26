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

### Ghost kart
The ghost kart is the last best saved race of the player. It gives the player a challenge to break their own record or someone elses.

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
  
  	// Call the indicator update method each tick
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
  	//if (!bMiniGameActive) return;
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
