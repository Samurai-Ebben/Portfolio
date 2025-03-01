![](/Sources/Ebben-Ring/Images/TitleScreen2.png)

![](/Sources/Ebben-Ring/Images/comboSystem.gif)    |  ![](/Sources/Ebben-Ring/Images/TogglingLocks.gif)
:-------------------------:|:-------------------------:
![](/Sources/Ebben-Ring/Images/DireRolls.gif)       |  ![](/Sources/Ebben-Ring/Images/DashAbility.gif)


## About the project
This project was made only for the purpose of learning and education, In Yrgo for 7 weeks.
EBBEN RING is a 3D Souls-like game developed in Unreal Engine 5 (UE5) using *AngelScript*. This project embodies the core principles of a challenging and immersive action RPG, drawing inspiration from the iconic Soulsborne genre. It features precise combat, deep player mechanics, and a dynamic environment that engages players in strategic and skill-based gameplay. One great limitation i faced during the development was animations. However, i animated some small sequances and made up for it with vfx and good game feel.


## Core Features
Souls-like Combat System:

- Responsive attacks with timing and strategy as key elements.
- Dynamic combo system that rewards skilled input and fluid transitions between moves.
- Damage mechanics for both the player and enemies, ensuring tactical gameplay.
- Player abilities designed to enhance combat variety and utility.

## Player Mechanics:

Designed and implemented entirely in AngelScript, delivering lightweight, efficient, and highly customizable functionality.

Features include:
- Attacking and damage dealing.
- Taking damage with stagger mechanics and recovery windows.
- A fluid combo system that adapts to player timing and enhances gameplay depth.
- Unique player abilities that can turn the tide of battle.
- Directional rolls and input buffering

### Switching players states using FSM
Switching between states was something i struggled with early on in the development of the combat system. The player could cancle any animation with another.
in order to control that in seamless way a State machine using the FSM (Finite State machine) was created with events called in every system to control how the player should exit and enter the previous and next animation respectively.

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
  
