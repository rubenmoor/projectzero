# Unreal Engine's Gameplay Ability System

There is a good chance you want to use the Gameplay Ability System, even if you are not working on some Action RPG (which is the most obvious use case). 
It takes time to learn it and know your way around the concept that it introduces, it also isn't perfect and introduces new, confusing concepts and design choices, **and** it is lacking in good, official documentation, but:

If you think of an *ability* as a gameplay interaction of the player with the game world, it becomes obvious that the Gameplay Ability System provides a useful layer of abstraction.
Reasons not to use it:

* You feel confident that you can build this kind of layer on your own.
* Even in the abstract definition, *abilities* are not an important characteristic of your game. E.g. there are not many distinct of those gameplay interactions.
* You found an alternative that is a better match for your needs.

### The Gameplay Ability System and ...

#### ... Multiplayer

If your game supports multiplayer, you have yet another reason why you want to use the Gameplay Ability System for its capacity of local prediction at little to no cost to the programmer.

#### ... User Input

The Gameplay Ability System has decent integration with Unreal Engine's Input System. If you use the [Enhanced Input](https://docs.unrealengine.com/5.2/en-US/enhanced-input-in-unreal-engine/), you might be interested in [the recipe "Enhanced Input with Abilities"](EnhancedInputWithAbilities.md).

#### ... Unreal Engine Visual Scripting "Blueprint"

The Gameplay Ability System is meant to be set up in C++.
In this sense, programming in C++ is required.
There are a lot of design choices that then enable the game designers to work with the Gameplay Ability System **without** coding in C++.
E.g. you can, in principle, create new Gameplay Abilities completely in Blueprint, including their impact on the game in terms of numbers ("Gameplay Attributes") and visual/audio effects via Gameplay Cues.

If you want to work exclusively in C++, check out [ue5coro](https://github.com/landelare/ue5coro), which introduces latent actions as C++ coroutines. 
Those are important as they allow you to write stuff like

```cpp
co_await Latent::UntilDelegate(UAnimInstance->OnMontageEnded);
```

to have a non-blocking way of waiting for some animation montage to finish before you continue with whatever follows the animation.

### Documentation

While the [official documentation](https://docs.unrealengine.com/5.0/en-US/gameplay-ability-system-for-unreal-engine/) is well worth checking out, you will find yourself frequently returning to [https://github.com/tranek/GASDocumentation](), especially because of a number of quirks and critical stuff that the official documentation never mentions, anywhere.

However, Tranek's unofficial documentation raised a couple of question and thus my comments on the topic.

# My comments on Tranek's GAS documentation

And some extras.
## Initializing attributes

Tranek mentions:

> For hardcoded maximum and minimum values, there is a way to define a `DataTable` with `FAttributeMetaData` that can set maximum and minimum values, but Epic's comment above the struct calls it a "work in progress".

If you check out the code in `AttributeSet.h`, you see that the "work in progress" comment is still there. You can use the Epic code to actually create a version of `UAttributeSet::InitFromMetaDataTable` that does something meaningful. If you don't, better ignore it or risk getting confused.

## Ability instancing policies

All the information on ability instancing policies in Tranek's documentation are accurate and helpful. Trying to implement those myself, I still ran into trouble.

> Instanced per Actor | This will probably be the `Instancing Policy` that you use the most.

That's absolutely right. The "Non-instanced" policy being the "most restrictive" is correct, but I ran into a lot of troubles naively trying to get a non-instanced policy to work.
A plain recommendation against the non-instanced policy is also adequate.
It turns out that you can't reliably add and remove gameplay effects from inside a non-instanced gameplay ability. I don't know why. 
Moving to the "instanced per actor" policy solved my problems.
## Creating gameplay effects

> Typically designers will create many Blueprint child classes of `UGameplayEffect`.

That's right. If you are fine with C++ only, there is no reason to ever create Blueprint assets from your C++ classes that implement `UGameplayEffect`, keeping everything lean. You can still add those Blueprint children at any moment later on.
Here's an example of how to set up a gameplay effect in C++:

The example makes use of [native Gameplay Tags in a way I documented here](NativeGameplayTags.md).

```cpp
// a gameplay effect that applies a counter-clockwise torque to an actor
UGE_TorqueCCW::UGE_TorqueCCW()  
{  
    DurationPolicy = EGameplayEffectDurationType::Infinite;  

	// my way of working with Gameplay Tags as native references
    const auto& Tag = FMyGameplayTags::Get();  
	// effectively multicasting the tag "GameplayCue.ThrustersFire" to trigger a visual effect
    GameplayCues.Add(FGameplayEffectCue(Tag.CueThrustersFire, 0., 1.));  

	// set the attribute `Torque` to `TorqueMax`
	FAttributeBasedFloat AttributeBasedFloat;  
	AttributeBasedFloat.BackingAttribute =  
	    FGameplayEffectAttributeCaptureDefinition  
	        ( UAttrSetAcceleration::GetTorqueMaxAttribute()  
	        , EGameplayEffectAttributeCaptureSource::Source  
	        , false  
	        );  
	AttributeBasedFloat.Coefficient = 1.0;  
	// for clockwise torque:
	//AttributeBasedFloat.Coefficient = -1.0;  
	Modifiers.Add(
	    { .Attribute         = UAttrSetAcceleration::GetTorqueAttribute()
	    , .ModifierOp        = EGameplayModOp::Override  
	    , .ModifierMagnitude = AttributeBasedFloat  
	    });
}
```

## Passing payload to gameplay abilities

> Activating a `GameplayAbility` by event allows you to [pass in a payload of data with the event](https://github.com/tranek/GASDocumentation#concepts-ga-data).

In a multiplayer setup, passing a payload isn't trivial. See the next section for an example where I pass information on the current mouse position to a LookAt-ability.

The information of the mouse position is local to the controlling client. If the server shall know about it, you have to pass it somehow over there: via a replicated property, a Server RPC, as `TargetData` (see link above) or as event payload as described here.

```cpp
UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)
```

This is maybe the most straight-forward way to activate an ability with a payload. Now note that the `FgameplayEventData` struct contains two `OptionalObject` UObjects. If you just create them on the client, the server will never see their contents. So you are probably good simply ignoring them. they don't help with the payload problem, at all.

## "Cancel abilities with Tag" and `bRetriggerInstancedAbility`

Trying to have a gameplay ability cancel other instances of itself using the `Cancel abilities with Tag` property, doesn't work. There is the `bRetriggerInstancedAbility` option.
This implies the use of the `Instanced per actor` policy and is probably what you want.

If you use [ue5coroGAS](https://github.com/landelare/ue5coro/blob/master/Docs/GAS.md), the `bRetriggerInstancedAbility` option again doesn't work. And in that case, the manual re-triggering of an ability after cancelling it, doesn't work either.
According to the author of ue5coro, there is probably a bug in Epic's latent action manager.
This is my workaround for a activating `UUE5CoroGameplayAbility` instances back to back (policy `InstancedPerActor`):

```cpp
void AMyPlayerController::Tick(float DeltaSeconds)
{
	if(/* some mouse movement triggers the gameplay action*/)  
	{  
		auto& Tag = FMyGameplayTags::Get();  

		// the gameplay ability makes the character *look at* wherever points the mouse
		// it is identified by the tag `AbilityLookAt` ("Ability.LookAt")
		auto Specs = AbilitySystemComponent->GetActiveAbilities(&Tag.AbilityLookAt.GetSingleTagContainer());  
		if(!Specs.IsEmpty())  
		{
			// an instance is currently active
			check(Specs.Num() == 1)  
			AbilitySystemComponent->CancelAbilitySpec(Specs[0], nullptr);  

			// schedule the activation of the ability for after the next tick
			// and cancel any schedule activation that happened in between
			Handle.Cancel();  
			Handle = ActivateLookAtAfterNextTick();  
		}
		else  
		{  
			// no instance currently active;
			// activate immediately
			ActivateLookAt();  
		}  
	}
}

// ue5coro-style coroutine
TCoroutine<> AMyPlayerController::ActivateLookAtAfterNextTick()  
{  
	// non-blocking wait
    co_await Latent::Ticks(2);  
    ActivateLookAt();  
}  
  
void AMyPlayerController::ActivateLookAt()  
{  
    auto& Tag = FMyGameplayTags::Get();  
    FGameplayEventData EventData;  
    EventData.EventMagnitude = MouseAngle;  

	// activate the ability with the mouse angle as payload
	UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(this, Tag.AbilityLookAt, EventData)
}
```

## Gameplay Cues

I was missing some top-level information on the **why** of Gameplay Cues. Let's try to fill the gap:

When activating a Gameplay Ability, you implicitly do a Server RPC. This is why Gameplay Abilities generally work fine in a multiplayer setup. They even facilitate the client-side prediction.
The effects of any Gameplay Ability, typically a Gameplay Effect, in turn changes some Gameplay Attributes. Those are replicated, too. So far, the networking has been taking care of in the background.

Note, however that the Gameplay Ability is **not run** on simulated proxies. E.g. client1 with some player1 triggers a Gameplay Action by some button push and runs the very same action as part of local prediction. The server runs the Gameplay Action authoritatively and replicates the resulting changes in Gameplay Attributes of player1 to client1 and client2 (and any other connected client), such that client2 is aware of the changed attributes of player1.

But what if some explosion is part of the gameplay action? If the explosion is triggered as part of the Gameplay Action, it will only happen on the server and client1. Client2 won't see any of it. ... that is unless you trigger the explosion by means of a Gameplay Cue.
Those are replicated by means of a multicast RPC. So any visual/audio effect can be considered as an implementation of a Gameplay Cue.

### Sending Gameplay Cues is easy ...

Either add Gameplay Cues to some Gameplay Effect, or just add/remove them explicitly during the execution of a Gameplay action, e.g. using `UAbilitySystemComponent::AddGameplayCue`. I found it useful to implement those two little adaptions:

```cpp
bool UMyAbilitySystemComponent::AddGameplayCueUnlessExists(FGameplayTag Cue)  
{  
    if(!HasMatchingGameplayTag(Cue))  
    {
	    AddGameplayCue(Cue);  
        return true;  
    }
    return false;  
}  
  
bool UMyAbilitySystemComponent::RemoveGameplayCueIfExists(FGameplayTag Cue)  
{  
    if(HasMatchingGameplayTag(Cue))  
    {
	    RemoveGameplayCue(Cue);  
        return true;  
    }
    return false;  
}
```

This way, I avoid stacking Gameplay Cues - which is rarely useful.

### ... but how to implement the visual/audio effect?

Tranek mentions that 

> `GameplayCueNotify` objects and other `Actors` that implement the `IGameplayCueInterface` can subscribe to these events

... "these events" meaning the `Add/Remove/Execute` of a Gameplay Cue. The second part is the important one. You don't have to implement neither `GameplayCueNotify_Static`  nor `GameplayCueNotify_Actor`-usually you can just have your Pawn-class, e.g. `AMyPawn`, implement `IGameplayCueInterface`. And now the magic:

The default implementation of `IGameplayCueInterface::HandleGameplayCue` looks for `UFUNCTION`s inside `AMyPawn` automatically. E.g. 

```cpp
void AMyPawn::GameplayCue_ShowThrusters(EGameplayCueEvent::Type Event, const FGameplayCueParameters& Parameters)
{
    // ...
}
```

will automagically be bound to `Add`/`Remove`/`Execute` of the Gameplay Cue `GameplayCue.ShowThrusters` with the corresponding `EGameplayCueEvent` type as the first parameter.

This `UFUNCTION` consumes the Gameplay Cue event. If you want to pass the event to the next subscribed handlers, call `IGameplayCueInterface::ForwardGameplayCueToParent()` in the first handler.

### `Local Gameplay Cues` don't make sense to me

Tranek suggests to define helper functions to `Add`/`Remove`/`Execute` Gameplay Cues locally. 
I don't see when that would be useful. Those local Gameplay Cues will affect some client1 and the server, but not any other connected client.

When I want a truly **local effect**, I have to poll `IsLocallyControlled`, e.g. run

```cpp
if(ActorInfo->IsLocallyControlled())  
{  
    Cast<AMyCharacter>(ActorInfo->AvatarActor)->DoSomethingTrulyLocal();  
}
```

If you like little abstractions, you can define

```cpp
void UMyGameplayAbility::LocallyControlledDo(const FGameplayAbilityActorInfo* ActorInfo, std::function<void(AMyCharacter*)> Func)  
{  
    if(ActorInfo->IsLocallyControlled())  
    {
	    Func(Cast<AMyCharacter>(ActorInfo->AvatarActor));  
    }
}
```

and run it like this:

```cpp
LocallyControlledDo(ActorInfo, [] (AMyCharacter* MyCharacter)  
{  
	MyCharacter->DoSomethingTrulyLocal();
});
```

You are not saving a lot of lines of code with this, but it's an expressive way to run actions local to the controlling client (which might be the server in a listen-server setup) and it captures the fact that casting the `AvatarActor` to `AMyCharacter` is save in that case.