![](/Sources/Ebben-Ring/Images/TitleScreen2.png)

![](/Sources/Ebben-Ring/Images/comboSystem.gif)    |  ![](/Sources/Ebben-Ring/Images/TogglingLocks.gif)
:-------------------------:|:-------------------------:
![](/Sources/Ebben-Ring/Images/DireRolls.gif)       |  ![](/Sources/Ebben-Ring/Images/DashAbility.gif)


## About the project
This project was developed solely for learning and educational purposes at Yrgo over a span of 7 weeks. EBBEN RING is a 3D Souls-like game built in Unreal Engine 5 (UE5) using AngelScript. It embodies the core principles of a challenging and immersive action RPG, drawing inspiration from the iconic Soulsborne genre. The game features precise combat, robust player mechanics, and a dynamic environment that rewards strategic, skill-based gameplay. One significant challenge I encountered was with animations; however, I compensated by animating a few short sequences and enhancing the overall experience with compelling visual effects and excellent game feel.


## Core Features
Souls-like Combat System
Responsive Attacks: Emphasizes timing and strategy.
Dynamic Combo System: Rewards skilled input and seamless transitions between moves.
Tactical Damage Mechanics: Balances damage for both the player and enemies.
Unique Player Abilities: Enhances combat variety and utility.

## Player Mechanics
Designed and implemented entirely in AngelScript, the player mechanics deliver lightweight, efficient, and highly customizable functionality. Key features include:

Attacking & Damage Dealing
Stagger Mechanics & Recovery Windows: Provides realistic damage feedback.
Fluid Combo System: Adapts to player timing for enhanced gameplay depth.
Special Abilities: Can turn the tide of battle.
Directional Rolls & Input Buffering
Player State Switching using an FSM: A Finite State Machine (FSM) was implemented to ensure seamless transitions between animations.

### Switching players states using FSM

Switching between states was challenging early on in the combat system development because animations could be canceled arbitrarily. To address this, I created an FSM that triggers events for a controlled exit from the current animation and entry into the next. Below is an excerpt of the FSM player state controller:

<details>
  <Summary>FSM Player state controller</Summary>

  ```cpp
  event void FStateEnterDelegate(EPlayerStates NewState);
event void FStateExitDelegate(EPlayerStates NewState);

class UPlayerStateManager: UActorComponent{
//=======Private Data=======\\
    APlayerCharcter Player;

//=======Private Data ======\\

//=======Public Data=========\\


    UPROPERTY()
    EPlayerStates CurrentState = EPlayerStates::Idle;

    UPROPERTY()
    EPlayerStates PreviousState = EPlayerStates::Idle;

    UPROPERTY()
    FStateEnterDelegate OnEnterState;

    UPROPERTY()
    FStateExitDelegate OnExitState;
//=======Public Data=========\\

    UFUNCTION(BlueprintOverride)
    void BeginPlay()
    {
        Player = Cast<APlayerCharcter>(GetOwner());
    }
    
    UFUNCTION()
    bool RequestStateChange(EPlayerStates NewState)
    {
        if (CanTransition(NewState))
        {
            OnStateExit(CurrentState); // Exit the current state
            PreviousState = CurrentState;
            CurrentState = NewState;
            OnStateEnter(CurrentState); // Enter the new state
            return true;
        }
        return false;
    }

    bool CanTransition(EPlayerStates NewState){
        switch(CurrentState){
            case EPlayerStates::Idle:
                return true;
        
            case EPlayerStates::Interacting:
                return NewState == EPlayerStates::Idle;

            case EPlayerStates::Sprint:
                return NewState != EPlayerStates::Block 
                && NewState != EPlayerStates::Consume
                && NewState != EPlayerStates::Interacting;

            case EPlayerStates::Roll:
                return
                    NewState != EPlayerStates::Block &&
                    NewState != EPlayerStates::Ability && 
                    NewState != EPlayerStates::Consume&&
                    NewState != EPlayerStates::Interacting;

            case EPlayerStates::Jump:
                return NewState == EPlayerStates::Attack 
                // || NewState != EPlayerStates::Roll
                || NewState != EPlayerStates::Block
                || NewState != EPlayerStates::Jump
                || NewState != EPlayerStates::Interacting;

            case EPlayerStates::Block:
                return
                NewState != EPlayerStates::Block&&
                NewState != EPlayerStates::Sprint&&
                NewState != EPlayerStates::Consume;

            case EPlayerStates::Attack:
                return NewState != EPlayerStates::Block &&
                    NewState != EPlayerStates::Roll &&
                    NewState != EPlayerStates::Jump &&
                    NewState != EPlayerStates::Ability&&
                    NewState != EPlayerStates::HitReact&&
                    NewState != EPlayerStates::Consume&&
                    NewState != EPlayerStates::Interacting;

            case EPlayerStates::HitReact:
                return NewState != EPlayerStates::Attack &&
                    NewState != EPlayerStates::Jump &&
                    NewState != EPlayerStates::Ability&&
                    NewState != EPlayerStates::Consume &&
                    NewState != EPlayerStates::Interacting;

            case EPlayerStates::Ability:
                return NewState == EPlayerStates::Idle
                || NewState == EPlayerStates::Ability;

            case EPlayerStates::Dead:
                return false;

            default:
                return true;
        }
    }

    void OnStateEnter(EPlayerStates NewState){
        OnEnterState.Broadcast(NewState);
    }

    void OnStateExit(EPlayerStates NewState){
        OnExitState.Broadcast(NewState);
    }
}

  ```
</details>


### The Directional Roll

The directional roll is one of my favorite mechanics to develop in a combat game. In a Souls-like title, a roll is considered valid only when the player provides directional input; otherwise, it functions as a simple hop back to dodge attacks. Additionally, the player becomes briefly invincible at the start of the roll animation. The following code snippet demonstrates how I calculate the ideal roll direction to help the player dodge and avoid deadly attacks:

![](/Sources/Ebben-Ring/Images/DireRolls.gif) 

<details>
  <summary>
    Directional Roll Script
  </summary>

  ```cpp
event void FOnRollUpdated(float Cost);

class URollComponent: UActorComponent{
//================REFS=====================\\
    APlayerCharcter Player;
    UMotionWarpingComponent MotionWarping;
//================REFS====================\\

//===============PUBLIC DATA===========\\
    UPROPERTY()
    FOnRollUpdated OnRoll;

    UPROPERTY()
    UAnimMontage RollAnim;
    UPROPERTY()
    UAnimMontage HopBackAnim;

    UPROPERTY()
    FOnAttackPerformed OnAttackPerformed;

    UPROPERTY()
    float RotOnX = 120;

    UPROPERTY()
    bool bRollBuffer = false;

    UPROPERTY()
    float RollBufferTime = 0.3;
//================PUBLIC DATA============\\

    UFUNCTION(BlueprintOverride)
    void BeginPlay()
    {
        Player = Cast<APlayerCharcter>(GetOwner());
    }

    UFUNCTION()
    void UpdateRoll(bool bHasDire = false){
        float StaminaCost = 25;
        if(!Player.HasEnoughStamina(StaminaCost)) return;

        if(!Player.GetCombatComp().bCanAttack || !Player.CharacterMovement.IsMovingOnGround()){

            if(Player.PlayerStateManager.RequestStateChange(EPlayerStates::Roll)) {
                bRollBuffer = true;
                System::SetTimer(this, n"ClearRollBuffer", RollBufferTime, false);
                return;
            }
        }
        if(Player.PlayerStateManager.RequestStateChange(EPlayerStates::Roll)){
            
            Player.TraceComponent.bInvincible = true;
            Player.GetCombatComp().bCanAttack = false;
            Player.GetBlockComp().bCanBlock = false;
            if(bHasDire){
                Player.PlayAnimMontage(RollAnim);
            }
            else{
                Player.PlayAnimMontage(HopBackAnim);
            }
            bRollBuffer = false;
            OnRoll.Broadcast(StaminaCost);
        }
    }

    UFUNCTION()
    bool HasRollDirection(FVector Direction){
        return Direction.IsNearlyZero() ? false : true;
    }

    UFUNCTION()
    void ClearRollBuffer()
    {
        bRollBuffer = false;
    }

    UFUNCTION()
    void ExecuteBufferedRoll()
    {
        if (bRollBuffer)
        {
            if(Player.PlayerStateManager.RequestStateChange(EPlayerStates::Roll)){

                UpdateRoll(HasRollDirection(Player.GetLastMovementInputVector()));
                bRollBuffer = false;
            }
            else{
                ClearRollBuffer();
            }
        }
    }

    UFUNCTION()
    FVector CalculateRollDirection(){  
        Player.GetBlockComp().bCanBlock = false;
        FVector LastInput = Player.GetLastMovementInputVector().GetSafeNormal();
        if(!HasRollDirection(LastInput))
        {
            return -Player.ActorForwardVector;
        }    
        return LastInput.GetSafeNormal();
    }


    UFUNCTION()
    FVector CalculateHopBack(){
        FRotator CurrentRotation = Player.GetActorRotation();
        FVector Direction = CurrentRotation.RotateVector(FVector(-250, 0,0));
        return Direction+Player.ActorRelativeLocation;
    }
}
  ```
</details>


### The Lock-On

Locking on to an enemy is a vital ability that allows the player to focus all attacks on a single target. While the basic concept of aligning the player’s direction to the target is straightforward, the challenge lies in seamlessly toggling between multiple enemies. The following snippet highlights the core calculations for the lock-on system:

![](/Sources/Ebben-Ring/Images/TogglingLocks.gif)

the script i wrote is a bit long and defines more details, but the following script will only feature the important calculations.

<details>
  <summary>Lock-On component</summary>

  ```cpp
  UFUNCTION(BlueprintOverride)
    void Tick(float DeltaSeconds)
    {
        if (IsValid(CurrentTargetActor))
        {
            FVector CurrentLocation = OwnerRef.GetActorLocation();
            FVector TargetLocation = CurrentTargetActor.GetActorLocation();
            auto curreDist = Distance(CurrentLocation,TargetLocation);
            if(curreDist >= BreakDistance){
                EndLockOn();
                return;
            }
            TargetRotation = FindLookAtRotation(CurrentLocation, TargetLocation);
            TargetRotation.Pitch = Math::Clamp(TargetRotation.Pitch, -5, 10);

            // Smoothly update camera position for lock-on
            FVector SmoothedPosition = Math::VInterpTo(CameraComp.GetRelativeLocation(), DesiredCameraPosition, DeltaSeconds, 5.0f);
            CameraComp.SetRelativeLocation(SmoothedPosition);

            // Smoothly update rotation to look at target
            FRotator SmoothedRotation = Math::RInterpTo(Controller.GetControlRotation(), TargetRotation, DeltaSeconds, 5.0f);
            Controller.SetControlRotation(SmoothedRotation);
            // If switching targets, gradually move rotation towards the new target
            if (bIsSwitchingTarget && SmoothedRotation.Equals(TargetRotation, 0.1f))
            {
                bIsSwitchingTarget = false;  // Stop switching once target rotation is reached
            }
        }
        else
        {
            FVector SmoothedPosition = Math::VInterpTo(CameraComp.GetRelativeLocation(), OriginalCameraPosition, DeltaSeconds, 5.0f);
            CameraComp.SetRelativeLocation(SmoothedPosition);
            EndLockOn();
        }
    }

    void BEnableCameraLag(bool bEnable){
        SpringArmComp.bEnableCameraLag = bEnable;
        SpringArmComp.bEnableCameraRotationLag = bEnable;
    }

    UFUNCTION()
    AActor GetNearestTargetAvailable(const TArray<AActor>& TargetAvailable){
        FVector CurrentLocation = OwnerRef.GetActorLocation();
        float32 ClosestDist = 0;
        return Gameplay::FindNearestActor(CurrentLocation, TargetAvailable, ClosestDist);
    }

    void GetAvailableTargets(float32 Radius = 1860){
        AvailableTargets.Empty();

        FVector CurrentLocation = GetOwner().GetActorLocation();
        FVector CameraForward = GetOwner().GetComponent(UCameraComponent).ForwardVector;

        IgnoredActors.Add(OwnerRef);

        bHasFoundTarget = System::SphereTraceMulti(
        CurrentLocation,
        CurrentLocation,
        Radius,
        TraceType,
        false,
        IgnoredActors,
        LockDrawDebug,
        OutHits,
        true,
        FLinearColor(1,0,0,1), 
        FLinearColor(0,1,0,1),5
        );

        if(!bHasFoundTarget){return;}

        for(const FHitResult& Hit: OutHits){
            if(!IsValid(Hit.GetActor().GetComponent(UUEnemy))) {return;}

            auto Target = Hit.GetActor();
            FVector TargetDir = (Target.GetActorLocation() - CurrentLocation).GetSafeNormal();
            float DotProduct = CameraForward.DotProduct(TargetDir);

            // Only add targets within a 90-degree field of view (front of the camera)
            if (DotProduct > 0)  
            {
                AvailableTargets.AddUnique(Target);
            }
        }
    }

    UFUNCTION()
    void StartLockOnWithLineTrace(float32 Radius= 1860){
        if(!bHasFoundTarget){return;}

        CurrentTargetActor = GetNearestTargetAvailable(AvailableTargets);

        if(!IsValid(CurrentTargetActor)) {return;}

        SettingMovementValues(true);

        DesiredCameraPosition = OriginalCameraPosition + LockOnCameraOffset;

        FVector CurrentLocation = OwnerRef.GetActorLocation();
        FVector TargetLocation = CurrentTargetActor.GetActorLocation();
        TargetRotation = FindLookAtRotation(CurrentLocation, TargetLocation);

        CurrentTargetActor.GetComponent(UUEnemy).OnSelectTarget.Broadcast(nullptr);
        UpdateTarget.Broadcast(this, CurrentTargetActor);
    }

    UFUNCTION()
    void ToggleLockon(float32 Radius = 720){
        GetAvailableTargets(Radius);
        if(IsValid(CurrentTargetActor)){
            EndLockOn();
        }
        else{
            StartLockOnWithLineTrace(Radius);
        }
    }

    void GetTargetsInView(TArray<AActor>& OutActors)
    {
        if (AvailableTargets.IsEmpty()) return;

        FVector PlayerLocation = OwnerRef.GetActorLocation();
        FVector CameraForward = Controller.GetControlRotation().Vector();  // Get camera forward direction

        // Filter and add only targets in front of the camera
        for (AActor AvailableTarget : AvailableTargets)
        {
            if (!IsValid(AvailableTarget)) continue;

            FVector TargetDirection = (AvailableTarget.GetActorLocation() - PlayerLocation).GetSafeNormal();
            float DotProduct = CameraForward.DotProduct(TargetDirection);

            if (DotProduct > 0)  // Target is in front of the camera
            {
                OutActors.AddUnique(AvailableTarget);
            }
        }
    }

    UFUNCTION()
    void SwitchTarget(float Direction)
    {
        GetAvailableTargets();
        TArray<AActor> TargetsInView;
        GetTargetsInView(TargetsInView);

        if (TargetsInView.IsEmpty() || !IsValid(CurrentTargetActor) || !bHasFoundTarget) return;

        int CurrentIndex = TargetsInView.FindIndex(CurrentTargetActor);

        // Calculate the next index based on Direction
        int NewIndex = CurrentIndex + (Direction > 0 ? 1 : -1);

        if (NewIndex >= TargetsInView.Num()) NewIndex = 0;
        else if (NewIndex < 0) NewIndex = TargetsInView.Num() - 1;

        AActor NewTarget = TargetsInView[NewIndex];

        if (IsValid(NewTarget) && NewTarget != CurrentTargetActor)
        {
            // Deselect the current target and select the new one
            CurrentTargetActor.GetComponent(UUEnemy).OnDeselectTarget.Broadcast(nullptr);
            CurrentTargetActor = NewTarget;

            // Smooth rotation transition
            TargetRotation  = FindLookAtRotation(OwnerRef.GetActorLocation(), NewTarget.GetActorLocation());
            bIsSwitchingTarget = true;

            // Broadcast events for the new target
            UpdateTarget.Broadcast(this, CurrentTargetActor);
            CurrentTargetActor.GetComponent(UUEnemy).OnSelectTarget.Broadcast(nullptr);
        }
    }
  ```
</details>





