You can find 80% of the information here [jambax.co.uk](https://jambax.co.uk/using-gameplay-tags-in-cpp/), already.
There is a part missing though: I want my Gameplay Tags available and functioning from within my C++ constructors,
which is totally possible - following e.g. [this forum post by SVasilev](https://forums.unrealengine.com/t/loading-native-gameplay-tags/265060/2?u=rubm123) or this very article.

# About Gameplay Tags

Gameplay Tags have been described as "FNames on steroids", because they support hierarchy (and corresponding matching)
together with performance optimizations for replication (the "fast replication" option needs to be switched on for that) and,
being tags rather than names, you can attach arbitrary numbers of Gameplay Tags to anything.
One typical use of a Gameplay Tag:
Some pawn has the tag "Movement.Jumping", which might replace a boolean property `bIsJumping`.
(In practice, pawns don't have Gameplay Tags, but rather their `AbilitySystemComponent` has them)
Gameplay Tags accomany the Gameplay Ability System, but they can be used and make sense on their own.
The [official documentation](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Tags/) provides sort of an overview, however ...

# Gameplay Tags and C++

... the way Gameplay Tags are introduced in the official documentation,
you declare them either from within the editor or by some .ini file.
And in order to get a reference to a Gameplay Tag in C++ you use

```cpp
const FGameplayTag MyTag = FGameplayTag::RequestGameplayTag(FName(“MyTag.MyTagChild”));
```

And you can use Gameplay Tags as references inside C++ now, right?
Unless you don't care about basic type-safety you cannot.

## Native Gameplay Tags

Here come to the rescue the "native gameplay tags" and a couple of articles and forum posts on how to implement them.
The idea is to create the Gameplay Tag from within C++ (native).

```cpp
#include "GameplayTagsManager" // module 'GameplayTags'

// somewhere
auto& GTM = UGameplayTagsManager::Get();
TagJumping = GTM.AddNativeGameplayTag("Movement.Jumping"); // register native tag
```

You are supposed to bind a function that registers your native Gameplay Tags to `UGameplayTagsManager::OnLastChanceToAddNativeTags()`, 
which just exists for this purpose.
If you so wish, you can even use the [FGameplayTagNativeAdder struct](https://docs.unrealengine.com/5.1/en-US/API/Runtime/GameplayTags/FGameplayTagNativeAdder/)
and override its `AddTags` with the exact same effect - don't expect any benefit though.

## Why this still isn't great

The Native Gameplay Tags are available in C++ **and** in the editor. Great ... or not?
If you want to use them in the constructor of some object, they are still not registered.
Your C++ code doesn't care about that and pretends everything's fine.
Any tags assigned in some constructor just won't be there when you run the editor or the game.

(I don't know why this has to be this way.
In theory, it should be possible to assign a Native Gameplay Tag before registration and have it properly displayed in the editor after its registration.
In practice this just doesn't seem to be the case)

There isn't a hook to register your Gameplay Tags that would be early enough to make sure you can use your Native Gameplay Tags safely, **everywhere**.
That includes the editor.

# A module for Native Gameplay Tags

The solution is actually quite simple.
Define and register your Native Gameplay Tags in a seperate module.
This module will get loaded as a dependency to your primary game module.
This way, inside your primary game module, all Native Gameplay Tags are ready to use anywhere.

There is nothing special about that module. 
This is how I organized everything in the end:

```cpp
// file: Games/Source/MyGameplayTags/Public/MyGameplayTags.h
#include "GameplayTagContainer.h"
#include "Modules/ModuleManager.h"

struct MYGAMEPLAYTAGS_API FMyGameplayTags
{
    FMyGameplayTags();
    static FMyGameplayTags& Get() { return MyGameplayTags; }
    
    FGameplayTag Movement;
    FGameplayTag MovementJumping;
    // Just put all your tags here

private:
    // static object to access this struct as a singleton
    static FMyGameplayTags MyGameplayTags;
};

class GameplayTagsModule : public IModuleInterface
{
    // literally nothing else to do inside the module
};
```

```cpp
// file: Games/Source/MyGameplayTags/Private/MyGameplayTags.cpp

#include "MyGameplayTags.h"

#include "GameplayTagsManager.h"

FMyGameplayTags::FMyGameplayTags()
{
	auto& GTM = UGameplayTagsManager::Get();

  // an explicit parent tag makes sense:
  // later you can use one of the many `match` functions with this parent tag and it will match all children, too
  Movement = GTM.AddNativeGameplayTag("Movement");

  // call it `Jumping` if you want to keep it short
	MovementJumping = GTM.AddNativeGameplayTag("Movement.Jumping");
  // ...
}

// initialize the global singleton
FMyGameplayTags FMyGameplayTags::MyGameplayTags;

IMPLEMENT_MODULE(GameplayTagsModule, MyGameplayTags)
```

```cs
// file: Games/Source/MyGameplayTagsMyGameplayTags.Build.cs

using UnrealBuildTool;

public class MyGameplayTags : ModuleRules
{
    public MyGameplayTags(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = ModuleRules.PCHUsageMode.UseExplicitOrSharedPCHs;

        PublicDependencyModuleNames.AddRange(
            new string[]
            {
                "Core",
            }
        );

        PrivateDependencyModuleNames.AddRange(
            new string[]
            {
                  "CoreUObject"
                , "Engine"
                , "GameplayTags"
            }
        );
    }
}
```

To use the module, you need two more things.
In your primary game module, add it to the `PrivateDependencyModuleNames`, like you would do with any other module.
In your .uproject file, list the module under `Modules`, like this:

```json
"Modules": [
		{ // any other module
    },
		{
			"Name": "MyGameplayTags",
			"Type": "RuntimeAndProgram",
			"LoadingPhase": "Default"
		}
```

For the loading phase, "Default" is sufficient.

## How to use Native Gameplay Tags

Probably you figured it out by yourself by now but `FGameplayTag::RequestGameplayTag("Foo.Bar")` is not something you will ever use.
Instead it goes like this now:

```cpp
#include "MyGameplayTags.h"

// somewhere else:
auto Tag = FMyGameplayTags::Get();
Tag.MovementJumping // use it
```

I.e. the use of Native Gameplay Tags happens exclusively by there reference objects in C++ and is thus completely type-safe.
In case you make changes to the string registered along with the native tag and call it "IsJumping" from now on,
the editor will have the now invalid "Jumping" tag **only** where you put that tag manually via the editor.
