## The problem part 1: `Enhanced Input`

With Unreal Engine 5's Enhanced Input some questions arise on how to organize your code.
The [system is well documented](https://docs.unrealengine.com/5.2/en-US/enhanced-input-in-unreal-engine/)
and not that complex, I would argue.
Definitely study those official docs, if you haven't already done so.

Also, [this tutorial](https://dev.epicgames.com/community/learning/tutorials/aqrD/unreal-engine-enhanced-input-binding-with-gameplay-tags-c) is much more complete than mine in that it shows the relevant editor settings, too. You probably want to read and understand it first. I will mention just this tutorial again further down, first to adapt the use of Gameplay Tags in there, second to offer my critique and an improvement.

Implementing Enhanced Input thus isn't such a big challenge per se.
Remains the fact, that it's new and it's not easy to get things right, right from the start.
This is especially true when the new thing has to integrate somehow into all the other stuff

## The problem part 2: Gameplay Ability System

This **other stuff** might be the Gameplay Ability System.
This is another pretty well-documented system and, in my humble opinion, a great abstraction layer.
The high-quality documentation isn't the [offical one (even though you might want to start there)](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/),
but this one: [github.com/tranek/GASDocumentation](https://github.com/tranek/GASDocumentation).

Hence this title of this hopefully helpful article.

# How to have Enhanced Input work together with the Gameplay Ability System

I'll start with the most basic approach,
adding a little bit of opinion later on,
to then finally present my way of doing this stuff - which might or might not be something to copy for your own project.

## The basic intended workflow with Enhanced Input

Nothing out of the ordinary here.
I follow best practices and don't do anything novel.

It goes something like this:

1. Create Input Action assets in the Editor. Those have names like "IA_Accelerate", "IA_Jump", or "IA_Fire".
2. Create at least one `InputMappingContext` in the Editor. Their main feature is that you can have several ones, side-by-side or stacked with explicit priority.
3. Then bind some `UFUNCTION` to each Input Action as desired:

```cpp
AMyPlayerController::SetupInput()
{
    // ...
    Cast<UEnhancedInputComponent>(InputComponent)->BindAction(IA_Jump, ETriggerEvent::Triggered, this, &AMyPlayerController::HandleJump);
}

void AMyPlayerController::HandleJump(const FInputActionInstance& InputActionInstance)
{
    // this Value isn't terribly meaningful for key-based input, but now you know how to get it
    // if your input is the scroll wheel of the mouse, you get the scrolled distance like this
    float Value = InputActionInstance.GetValue().Get<FInputActionValue::Axis1D>();

    // alternative Value, indicating that a key is actually pressed now
    bool Value = InputActionInstance.GetValue().Get<bool>();

    GetPawn<AMyPawn>()->Jump();
}
```

## Bringing in Gameplay Abilities in the most simple way

If you set up a `AbilitySystemComponent` on your pawn, you might want to activate one or more Gameplay Abilities like this:

```cpp
void AMyPlayerController::HandleJump(const FInputActionInstance& InputActionInstance)
{
    TArray<FGameplayAbilitySpec*> AbilitiesToActivate;
    GetActivatableGameplayAbilitySpecsByAllMatchingTags(TagJump.GetSingletonContainer(), AbilitiesToActivate);

    for (auto GameplayAbilitySpec : AbilitiesToActivate)
    {
    	// fun fact: if you don't explicitly check if your ability is active already, your ability will just activate once more,
    	// which is something you probably want to control explicitly.
        // You can use tags to have an ability either block or cancel further activations (instances) of itself.
        // I use the code below to make this double activation impossible.
    	// Btw, stacking of gameplay effects is generally intended. Stacking abilities not so much.
        if(!GameplayAbilitySpec->IsActive()
        {
            TryActivateAbility(GameplayAbilitySpec->Handle);
        }
    }
}
```

This is a perfectly valid way to go about stuff.
There are shortcomings, but they depend heavily on the style of your projects. E.g. from here, there probably isn't any **generally recommended solution**.

A couple of comments on the code so far:

* The Input Actions are kind of wasteful. Each of them is a single asset and needs to be managed. Where do you put the reference? We will have an intermediate solution for that, shortly. Input Actions do not hold the actual Input (e.g. a key), those you find in the Input Mapping Context. Input Actions do not take care of executing the code that you bind to them, either. They really mostly clutter your content browser.
* You might want to check the `Value` in its boolean version, like above. If you configure an Input Action to have the triggers `Pressed` and `Released`, you can use the Value to distinguish between the two. When the `Released` trigger fires, it will be `false`. Use that, e.g. to end your Gameplay Ability and you have an ability that stays active as long the button is pressed.
* The function `BindAction` from the Enhanced Input Component has the parameter of type `ETriggerEvent`. I haven't seen a good reason why this should be anything else then `ETriggerEvent::Triggered`, ever. Note that the `ETriggerEvent` is an entirely different type then `Pressed`, `Released`, and `Down`, which have type `UInputTrigger`.
* While you can add even more triggers then just `Pressed` and `Released`, there are weird limits when you mix with `Tap`. In theory, the set of `Hold`, `HoldRelease`, and `Tap` could be a very useful combination of triggers in one singe Input Action. In practice you can't distinguish between `HoldRelease` and `Tap` in `HandleJump`. The value is `false` for both and thus more then two triggers don't usually make sense.

## Organize Input Actions in a Data Asset

An arguably elegant way to keep things organized is a new class that inherits from `UDataAsset`:

```cpp
// file: MyInputActions.h
UCLASS()  
class MYAWESOMEGAME_API UMyInputActions : public UDataAsset  
{  
    GENERATED_BODY()  
  
public:  
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Input Action Assets")  
    TMap<FName, UInputAction*> Map;
};
```

In the Editor, create an Editor asset that inherits  from `UMyInputActions`. Now you can either fill it in manually or you add this little goodie to have your computer do the work for you:

```cpp
// file: MyInputActions.h
// ...
#if WITH_EDITOR  
protected:  
    UFUNCTION(CallInEditor)  
    void RefreshMyInputActions();  
#endif
};

// file: MyInputActions.cpp

#include "AssetRegistry/AssetRegistryModule.h"  
#include "MyInputActions.h"  
  
#if WITH_EDITOR  
void UMyInputActions::RefreshMyInputActions()  
{  
    Map.Reset();  
    const auto& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry").Get();  
    TArray<FAssetData> AssetData;  
    AssetRegistryModule.GetAssetsByClass(UInputAction::StaticClass()->GetClassPathName(), AssetData);  
    for(auto AssetDatum : AssetData)  
    {
        auto* Asset = Cast<UInputAction>(AssetDatum.GetAsset());  
        Map.Add(Asset->GetFName(), Asset);  
    }
}  
#endif
```

The `CallInEditor` qualifier for UFUNCTION gives you a button in the editor to populate your data asset. By the way, the Data Asset doesn't really need a `TMap`,  a `TArray` would do the job just find, too. I prefer `TMap`s over `TArray`s though, because you can never accidentally add an item twice. And if you have the key, in this case: the FName, you can access any value efficiently with just `Map["IA_Jump"]`.

You add a property `UMyInputActions* MyInputActions` to the Player Controller to store a reference to the freshly created Data Asset class and then you can conveniently get any of your Input Actions in C++, like this:

```cpp
AMyPlayerController::SetupInput()
{
    Cast<UEnhancedInputComponent>(InputComponent)->BindAction
	    ( MyInputActions->Map["IA_Jump"]
        , ETriggerEvent::Triggered
        , this
        , &AMyPlayerController::HandleJump
        );
}
```

## Adding Gameplay Tags for actual organization

This FName-based map isn't really doing a lot. I don't want to rely on spelled out FNames, usually. There is potential for win-win. We need to access the references to the Input Action Assets somehow and we can activate Gameplay Abilities by Gameplay Tags. If your project has a lot of Input Actions that all trigger one or more Gameplay Abilities by Gameplay Tag, this is a very efficient way.

Gameplay Tags are a good-enough documented feature of the Gameplay Ability System. If you want to know how I set up my Native Gameplay Tags for easy use in C++, read the [recipe on the topic](NativeGameplayTags.md).

There is this [whole tutorial on how to wire Input Actions with Gameplay Tags](https://dev.epicgames.com/community/learning/tutorials/aqrD/unreal-engine-enhanced-input-binding-with-gameplay-tags-c).
I mentioned this tutorial right at the beginning for its great explanatory value.
What you can take away there to immediately improve our current solution:

Replace the `TMap<FName, UInputAction*>` with `TMap<FGameplayTag, UInputAction*>`. The way how to use this new map shouldn't come at a surprise. This is a Gameplay-Tag-centered approach. From within `AMyPlayerController::SetupInput`, you can - for any given Gameplay Tag - get the Input Action and bind a UFUNCTION:

```cpp
AMyPlayerController::SetupInput()
{
    for(auto Tag : MyAbilityTags)
    {
        Cast<UEnhancedInputComponent>(InputComponent)->BindAction
            ( MyInputActions->Map[Tag]
            , ETriggerEvent::Triggered
            , this
            , &AMyPlayerController::HandleInputAction
            );
    }
}
```

Now comes an awkward part though. From within `AMyPlayerController::HandleInputAction`, we don't have access to the Gameplay Tag anymore. Wouldn't it be nice to activate some Gameplay Ability based on the same Gameplay Tag that we linked to the corresponding Input Action in the map? There is a way to get this to work (ab)using the map for a reverse lookup:

```cpp
AMyPlayerController::HandleInputAction(const FInputActionInstance& InputActionInstance)
{
    TryActivateAbilityByTag(Map.FindKey(InputActionInstance.SourceAction));
}
```

You might as well turn around the map to `TMap<UInputAction*, FGameplayTag>` to avoid the reverse lookup. You probably would traverse through the map to bind every Input Action like this:

```cpp
AMyPlayerController::SetupInput()
{
    for(auto [InputAction, Tag] : MyInputActions.Map)
    {
        Cast<UEnhancedInputComponent>(InputComponent)->BindAction
            ( InputAction
            , ETriggerEvent::Triggered
            , this
            , &AMyPlayerController::HandleInputAction
            );
    }
}

AMyPlayerController::HandleInputAction(const FInputActionInstance& InputActionInstance)
{
    TryActivateAbilityByTag(MyInputActions.Map[InputActionInstance.SourceAction]);
}
```

### The limitations

I wasn't happy with this solution. In my game, I have a couple of Gameplay Abilities that I strictly bind to a certain key. And furthermore, I want to try out the **Hold**, **HoldRelase**, **Tap** pattern, where holding a key activates the ability and ends it on key release, whereas tapping a key toggle the ability. I.e. it ends when you tap the key again.

This presumably simple pattern, generically applied to a couple of Input Actions and Abilities results in a lot of code duplication. ... and asset duplication. For this pattern I will need two Input Actions for every ability. I still am no friend of those lightweight Input Action Assets littering my content browser. There is also the timing configuration: How long to hold a key down, to trigger **hold**? How short to still count as a **tap**? And there is the option `bIsOneShot` for hold-style triggers that is `false` by default but makes more sense set to `true`. Now those configuration values, I want to set globally (most likely) and I have to override `UInputAction` to set defaults. Or I use a INI configuration file. Still, if things seem off, I would have to check individual Input Action Assets to make sure that they don't deviate from the default. Messy, for my taste.

# A more complete solution for Input Actions with the Gameplay Ability system

This is my opinionated choice how to bind Input Actions to Gameplay abilities, more or less generically, just by means of tags.
**Generically** means that this code accomodates the general situation where one input (e.g. a key) is treated by means of any number of triggers (**hold**, **release**, **tap**, ...) to then start or end any number of Gameplay Abilities.
Configuring this chain happens in the editor, inside the `InputActionSet` asset.
Extra coding is only required to implement stuff that isn't a Gameplay Ability or a Gameplay Cue.

## An `InputActionSet`, to fit any combinations of triggers in one structure

An Input Action set combines a number of Input Actions. They will all be controlled by one input, e.g. a key on the keyboard. Then, for each trigger (pressed, down, release, tap, ...), a single InputAction is created and configured at runtime.
Thus the name `InputActionSet`.

I use template parameters to pass the trigger on to the handler. This allows to have one key control a Gameplay Ability (or Gameplay Cues or anything, really) by pressing, holding or tapping a key - or all of the above.
See [](BindInputToTemplatedFunction) for a short example on how templated functions can help when binding functions.

```cpp
// file: MyInputActionSet.h
#pragma once  
  
#include "EnhancedInputComponent.h"  
#include "GameplayTagContainer.h"  
#include "Modes/MyPlayerController.h"  
#include "InputTriggers.h"  
  
#include "MyInputActionSet.generated.h"  
  
class AMyPlayerController;  
class UInputTrigger;  
  
UCLASS()  
class MYAWESOMEGAME_API UMyInputActionSet : public UDataAsset  
{  
    GENERATED_BODY()  
  
public:  
    // just like with Input Actions, which used to be persisted assets and now are created ad-hoc in code,
    // we will dethrone the Input Mapping Context and define the key for an InputActionSet just once, here
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)  
    TArray<FKey> Keys;  

    //  an InputActionSet is defined by the set of triggers, each one will spawn a single InputAction
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)  
    TSet<EInputTrigger> InputTriggers;  

    // when you can have one tag, you can have many
    UPROPERTY(EditDefaultsOnly, BlueprintReadOnly)  
    FGameplayTagContainer InputActionTags;  
  
    // create and bind an input action for every trigger in `InputTriggers`
    // this function will be called by the Player Controller to start things off
    UFUNCTION(BlueprintCallable)  
    void BindActions(AMyPlayerController* InPlayercontroller, UInputMappingContext* IMC);  

    // configuration values, defined here and nowhere else
	
    UPROPERTY(EditDefaultsOnly, BlueprintReadWrite, Category="Input Action Settings")  
    float ActuationThreshold = 0.5f;  
  
    // Hold, HoldAndRelease, Tap, Pulse  
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Input Action Settings")  
    bool bAffectedByTimeDilation = false;  
  
    // Hold, HoldAndRelease  
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Input Action Settings")  
    float HoldTimeThreshold = 0.2f;  
    // Hold, HoldAndRelease  
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Input Action Settings")  
    bool bIsOneShot = true;  
  
    // Tap  
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Input Action Settings")  
    float TapReleaseTimeThreshold = 0.2f;  
  
protected:  
    // the template parameter effectively pass Input Trigger information down the line
    // Without this, the UFUNCTION that is bound to an Input Action only has access to FInputActionInstance, which sadly doesn't carry this information
    template<EInputTrigger InputTrigger>  
    void BindAction(UInputMappingContext* IMC);  

    // EInputTrigger is a custom enum defined in `AMyPlayerController` (see below)
    // here we translate from the enum to the `UInputTrigger` object
    // at this opportunity, we set the configuration values, too
    UInputTrigger* GetTriggerEvent(EInputTrigger InInputTrigger);  

    // this function doesn't do much more than translating the template parameter
    // to a regular function parameter and call a non-template function in the player controller
    // this is necessary, though:
    // try to implement a templated function in the player controller and you get 
    // frustrated by circular imports
    // (the implementations of templated functions go into the header file)
    template<EInputTrigger InputTrigger>  
    void HandleInput(const FInputActionInstance& InputActionInstance);  

    // this class is factored out of `AMyPlayerController`
    // leaving it in there, leaves the player controller more bloated
    // having it apart, we need to keep a reference to the player controller around
    UPROPERTY(BlueprintReadOnly)  
    TObjectPtr<AMyPlayerController> PC;  
};  
  
template <EInputTrigger InputTrigger>  
void UMyInputActionSet::BindAction(UInputMappingContext* IMC)  
{  
    // Input Actions are created at run-time now; no need to manage them, ever
    auto* InputAction = NewObject<UInputAction>(this);  
    InputAction->Triggers.Add(GetTriggerEvent(InputTrigger));  
    if(InputTriggers.Contains(InputTrigger))  
    {
        PC->GetInputComponent()->BindAction(InputAction, ETriggerEvent::Triggered, this, &UMyInputActionSet::HandleInput<InputTrigger>);  
    }  
    for(const auto& Key : Keys)  
    {
        IMC->MapKey(InputAction, Key);  
    }
}  
  
template <EInputTrigger InputTrigger>  
void UMyInputActionSet::HandleInput(const FInputActionInstance& InputActionInstance)  
{  
    // see the fruits of templated functios: we have now effectively bound to 
    // a regular function that knows whether a key was pressed, released, held, heldreleased, tapped, ...
    PC->RunInputAction(InputActionTags, InputTrigger, InputActionInstance);  
}

```

```cpp
// file: MyInputActionSet.cpp

#include "MyInputActionSet.h"  
#include "InputTriggers.h"  
  
void UMyInputActionSet::BindActions(AMyPlayerController* InPlayercontroller, UInputMappingContext* IMC)  
{  
    PC = InPlayercontroller;  
    using enum EInputTrigger;  

    // templates parameters are set at compile-time;
    // even if C++ rolls out short loops, we have to spell things out here
    BindAction<Down>          (IMC);  
    BindAction<Pressed>       (IMC);  
    BindAction<Released>      (IMC);  
    BindAction<Hold>          (IMC);  
    BindAction<HoldAndRelease>(IMC);  
    BindAction<Tap>           (IMC);  
    BindAction<Pulse>         (IMC);  
    BindAction<ChordAction>   (IMC);  
}  
  
UInputTrigger* UMyInputActionSet::GetTriggerEvent(EInputTrigger InInputTrigger)  
{  
    switch (InInputTrigger)  
    {
    case EInputTrigger::Down:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerDown>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            return InputTrigger;  
        }
    case EInputTrigger::Pressed:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerPressed>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            return InputTrigger;  
        }
    case EInputTrigger::Released:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerReleased>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            return InputTrigger;  
        }
    case EInputTrigger::Hold:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerHold>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            InputTrigger->bAffectedByTimeDilation = bAffectedByTimeDilation;  
            InputTrigger->HoldTimeThreshold = HoldTimeThreshold;  
            InputTrigger->bIsOneShot = bIsOneShot;  
            return InputTrigger;  
        }
    case EInputTrigger::HoldAndRelease:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerHoldAndRelease>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            InputTrigger->bAffectedByTimeDilation = bAffectedByTimeDilation;  
            InputTrigger->HoldTimeThreshold = HoldTimeThreshold;  
            return InputTrigger;  
        }
    case EInputTrigger::Tap:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerTap>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            InputTrigger->TapReleaseTimeThreshold = TapReleaseTimeThreshold;  
            return InputTrigger;  
        }
    case EInputTrigger::Pulse:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerPulse>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            InputTrigger->bAffectedByTimeDilation = bAffectedByTimeDilation;  
            return InputTrigger;  
        }
    case EInputTrigger::ChordAction:  
        {  
            const auto InputTrigger = NewObject<UInputTriggerChordAction>(this);  
            InputTrigger->ActuationThreshold = ActuationThreshold;  
            return InputTrigger;  
        }
    }
}
```

### A Data Asset for Input Action Sets

Nothing new here, just a `Map<FName, UMyInputActionSet>` inside of a Data Asset.

```cpp
// file: UMyInputActions.h

#pragma once  
  
#include "CoreMinimal.h"  
#include "Engine/DataAsset.h"  
  
#include "MyInputActions.generated.h"  
  
class UMyInputActionSet;  
  
/**  
 * store my newly defined InputActionSets in a Data Asset
 */  
UCLASS()  
class MURDERINSPACE_API UMyInputActions : public UDataAsset  
{  
    GENERATED_BODY()  
  
public:  
    UPROPERTY(EditAnywhere, BlueprintReadOnly, Category="Input Action Assets")  
    TMap<FName, UMyInputActionSet*> Map;  
  
#if WITH_EDITOR  
protected:  
    // auto-fill with all assets of appropiate type
    UFUNCTION(CallInEditor)  
    void RefreshMyInputActions();  
#endif  
};
```

```cpp
// file: UMyInputActions.cpp

#include "Input/MyInputActions.h"  
  
#include "AssetRegistry/AssetRegistryModule.h"  
#include "Input/MyInputActionSet.h"  
  
#if WITH_EDITOR  
void UMyInputActions::RefreshMyInputActions()  
{  
    Map.Reset();  
    const auto& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry").Get();  
    TArray<FAssetData> AssetData;  
    AssetRegistryModule.GetAssetsByClass(UMyInputActionSet::StaticClass()->GetClassPathName(), AssetData);  
    for(auto AssetDatum : AssetData)  
    {
        auto* Asset = Cast<UMyInputActionSet>(AssetDatum.GetAsset());  
        Map.Add(Asset->GetFName(), Asset);  
    }
}  
#endif
```

### Putting stuff together in the Player Controller

```cpp
// file: AMyPlayerController.h

#pragma once  
  
#include "CoreMinimal.h"  
#include "EnhancedInputSubsystems.h"  
#include "InputMappingContext.h"  
#include "GameFramework/PlayerController.h"  
  
#include "MyPlayerController.generated.h"  
  
class UMyInputActionSet;  
class UMyInputActions;  
class UMyAbilitySystemComponent;  
struct FGameplayTag;  
class UTaggedInputActionData;  
  
UENUM(BlueprintType)  
enum class EInputTrigger : uint8  
{  
      Down  
    , Pressed  
    , Released  
    , Hold  
    , HoldAndRelease  
    , Tap  
    , Pulse  
    , ChordAction  
};  
  
UCLASS()  
class MURDERINSPACE_API AMyPlayerController : public APlayerController  
{  
    GENERATED_BODY()  
  
    AMyPlayerController();  
  
    // setup defaults for my input actions:  
    // * bind them to gameplay abilities/cues by tag    // * or give special treatment    virtual void SetupInputComponent() override;  
  
public:  
    // a little bit of convenience to avoid the casting of the `InputComponent` property  
    UFUNCTION(BlueprintCallable)  
    UEnhancedInputComponent* GetInputComponent();  
  
    // here is where stuff happens  
    UFUNCTION(BlueprintCallable)  
    void RunInputAction(const FGameplayTagContainer& InputActionTags, EInputTrigger InputTrigger, const FInputActionInstance& InputActionInstance);  
protected:  
    // the Data Asset to store a map of `UMyInputActionSet`s  
    // formerly stored `UInputAction`s
    UPROPERTY(EditDefaultsOnly)  
    TObjectPtr<UMyInputActions> MyInputActions;
 
    // Thus we use `AcknowledgePossesion` to set up the camera and alike
    virtual void AcknowledgePossession(APawn* P) override;  
  
    virtual void BeginPlay() override;  
  
    // input events  
  
    /*
     * deal with input actions bindings apart from gameplay ability or cue
     * (an input action can have any combination of bindings to custom, ability, cue
     */  
    UFUNCTION()
    void RunCustomInputAction(FGameplayTag CustomBindingTag, EInputTrigger InputTrigger, const FInputActionInstance& InputActionInstance);  

    UPROPERTY(VisibleAnywhere)  
    TObjectPtr<UEnhancedInputLocalPlayerSubsystem> Input;  
  
    // convenience access to the AbilitySystemComponent of the possessed pawn  
    // only used by client to process inpu
    UPROPERTY(BlueprintReadOnly)  
    UMyAbilitySystemComponent* AbilitySystemComponent;  
};
```

```cpp
// file: AMyPlayerController.cpp

#include "Modes/MyPlayerController.h"  
  
#include "AbilitySystemComponent.h"  
#include "EnhancedInputSubsystems.h"  
#include "MyGameplayTags.h"  
#include "Actors/MyCharacter.h"  
#include "GameplayAbilitySystem/MyAbilitySystemComponent.h"  
#include "Input/MyInputAction.h"  
#include "Kismet/GameplayStatics.h"  
#include "Logging/StructuredLog.h"  
#include "Modes/MyGameInstance.h"  
#include "Modes/MyLocalPlayer.h"  
#include "EnhancedInputComponent.h"  
#include "GameplayAbilitySystem/MyGameplayAbility.h"  
#include "Input/MyInputActions.h"  
  
AMyPlayerController::AMyPlayerController()  
{  
    PrimaryActorTick.bCanEverTick = false;  
}  
  
void AMyPlayerController::SetupInputComponent()  
{  
    Super::SetupInputComponent();  
  
    Input = GetLocalPlayer()->GetSubsystem<UEnhancedInputLocalPlayerSubsystem>();  
    auto Tag = FMyGameplayTags::Get();  
  
    auto* IMC = NewObject<UInputMappingContext>(this, "InputMappingContext");  
    for(auto [Name, InputActionSet] : MyInputActions->Map)  
    {
        InputActionSet->BindActions(this, IMC);  
    }
    Input->AddMappingContext(IMC, 0);  
}  
  
UEnhancedInputComponent* AMyPlayerController::GetInputComponent()  
{  
     return Cast<UEnhancedInputComponent>(InputComponent);  
}  

// this is what we fought for
// generically deal with input and call gameplay actions and cues based on the tags
void AMyPlayerController::RunInputAction(const FGameplayTagContainer& InputActionTags, EInputTrigger InputTrigger,  
    const FInputActionInstance& InputActionInstance)  
{  
    auto Tag = FMyGameplayTags::Get();  
    const auto AbilityTags = InputActionTags.Filter(Tag.Ability.GetSingleTagContainer());  
    const auto CustomInputBindingTags = InputActionTags.Filter(Tag.InputBindingCustom.GetSingleTagContainer());  
  
    for(auto CustomInputBindingTag : CustomInputBindingTags)  
    {
        RunCustomInputAction(CustomInputBindingTag, InputTrigger, InputActionInstance);  
    }
    if(!AbilityTags.IsEmpty())  
    {
        TArray<FGameplayAbilitySpec*> Specs;  
        switch(InputTrigger)  
        {
        case EInputTrigger::Down:  
        case EInputTrigger::Pressed:  
        case EInputTrigger::Hold:  
            AbilitySystemComponent->GetActivatableGameplayAbilitySpecsByAllMatchingTags(AbilityTags, Specs, false);  
            for(auto Spec : Specs)  
            {
                // when you, like me, bind activation to the key pressed event and end the ability upon release,
                // you can be very restrictive regarding duplicate activations of one ability, e.g. with this assert:
                check(!Spec->IsActive())  
                AbilitySystemComponent->TryActivateAbility(Spec->Handle);  
            }
            break;  
        case EInputTrigger::Released:  
        case EInputTrigger::HoldAndRelease:  
            for(auto Spec : AbilitySystemComponent->GetActiveAbilities(&AbilityTags))  
            {
                // different options here,
                // you can cancel the ability or call some function on the ability instance like so:

                // InstancingPolicy: InstancedPerActor
                auto* AbilityInstance = Spec.GetPrimaryInstance();

                if(!AbilityInstance)
                    // InstancingPolicy: NonInstanced (Get the CDO)
                    AbilityInstance = Spec.Ability;

                // TODO: InstancedPerExecution doesn't make sense in this setup

                // In a multiplayer setup, calling a function on the ability instance must be a Server RPC
                // `UMyGameplayAbility` is my ability base clase to provide the interface
                Cast<UMyGameplayAbility>(AbilityInstance)->ServerRPC_SetReleased();
            }
            break;  
        case EInputTrigger::Tap:  
            AbilitySystemComponent->GetActivatableGameplayAbilitySpecsByAllMatchingTags(AbilityTags, Specs, false);  
            for(auto Spec : Specs)  
            {
                // different mechanism then the press/release logic above: tapping once do de-activate ...
                if(Spec->IsActive())  
                {
                    // like with the `HoldAndRelease` case above, cancel or call some function on the ability instance
                }
                // ... tapping once to activate.
                // Note that with this setup, you can control the trigger mechanisms that are actually in effect
                // by adding/removing any number of triggers to the InputActionSet
                else  
                {  
                    AbilitySystemComponent->TryActivateAbility(Spec->Handle);  
                }
            }
            break;  
        case EInputTrigger::Pulse:  
        case EInputTrigger::ChordAction:  
            UE_LOGFMT(LogMyGame, Error, "{THIS}: Input triggers Pulse and Chord not implemented.", GetFName());  
            break;  
        }
    }
}  
  
  
void AMyPlayerController::AcknowledgePossession(APawn* P)  
{  
    Super::AcknowledgePossession(P);  
  
    AMyCharacter* MyCharacter = Cast<AMyCharacter>(P);  
  
    // this `Get` is mere convenience: a static function that casts to `UMyAbilitySystemComponent`  
    AbilitySystemComponent = UMyAbilitySystemComponent::Get(MyCharacter);  
  
    // client-side initialization of the AbilitySystemComponent  
    AbilitySystemComponent->InitAbilityActorInfo(MyCharacter, MyCharacter);  
}  
  
void AMyPlayerController::BeginPlay()  
{  
    Super::BeginPlay();  
  
    const FInputModeGameAndUI InputModeGameAndUI;  
    SetInputMode(InputModeGameAndUI);  
}  
  
void AMyPlayerController::RunCustomInputAction(FGameplayTag CustomBindingTag, EInputTrigger InputTrigger, const FInputActionInstance& InputActionInstance)  
{  
    const auto Tag = FMyGameplayTags::Get();  
  
    // select  
    if(CustomBindingTag == Tag.InputBindingCustomSelect)  
    {
        // custom handling of `Select` action (where the player clicked and selected an object)  
        // this kind of simple mouse interaction doesn't really have a place in the Gameplay Ability System
    }  
    // zoom  
    else if(CustomBindingTag == Tag.InputBindingCustomZoom)  
    {
        // custom handling of zooming in and out using the mouse wheel  
        auto Delta = InputActionInstance.GetValue().Get<FInputActionValue::Axis1D>();  
        // apply Delta to zoom level  
    }  
}
```

## Consideration of alternatives

Some people might find that their Input Action assets together with one or more Input Mapping Contexts are quite managable in the editor without need for a grouping structure like the one I implement here.

There is also a whole machinery for so called modifiers. I.e. both a single input action and a mapping in the Input Mapping Context allow to manipulate the Input Action value before it reaches your Input Action binding. 
This doesn't even require coding. You can add a chain of modifiers by just clicking around in the editor.
Even if I do stuff mostly in C++, I appreciate the options to try out things in the editor quickly without need to recompile.
If needed, overriding `UInputModifier` isn't that complicated either for your own custom modifier.
The modifiers have been quite useless for my use case, though.

In theory, these modifiers help to have the Input Action value carry more information.
It is, however, not possible to put information on the input trigger (**hold**, **pressed**, **released**, **tap**, ...) into this value without creating one single Input Action asset for every trigger.
The resulting structure didn't appeal to me.
When I offer a configuration inside my game that allows the player to re-map keys, I prefer to have a structure that allows me to set the key in just one place rather than parsing the Input Mapping Context for occurrences of some key to be re-mapped.
