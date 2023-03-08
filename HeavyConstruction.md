WIP

This is about `AActor::OnConstruction`, `UObject::PostEditChangeChainPropery`, what you can do inside there and how.

## The basics

Skip to the next part if you know already about the limitations of the C++ constructor and `OnConstruction` in Unreal.

### To get things out of the way: The C++-Constructor

The constructor is a feature of C++ classes (and structs) and gets confused with `AActor::OnConstruction`.
This happens when people know about the BluePrint `Construction Script` and try to find the C++ version.
While the construction script (BluePrint) and `AActor::OnConstruction` aren't identical, it's fair to treat them more or less equal.

This is not the case for the C++ constructor.

According to official documentation, the C++ constructor of any C++ object (class or struct) is used to set defaults for class properties.
This is very true and the constructor can't to much more.

#### The C++-Constructor doesn't have access to Unreal systems

In Unreal Engine, objects get constructed all the time.
When you implement something like `class MyCharacter : public ACharacter`, a `MyCharacter` object certainly gets created when--some day--the player starts your game.
Or when you hit simulate or play to start play-in-editor.
But a `MyCharacter` object also will be created when you just start up the Unreal Editor.
... and when you edit a BluePrint class object `BP_MyCharacter` that inherits from `MyCharacter`.

Just put a break point into the constructor and run debug and start counting how many times your break point is hit while you edit instance defaults in Blueprint.

The reason for this:
Unreal Engine has to build instances of your classes so you can work with them in the editor
(including, but not limited to, the
[Class Default Object](https://docs.unrealengine.com/4.26/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Objects/).
)
And for this reason, from inside a C++-constructor you can't rely on the presence of any other objects.
No `GameInstance` (loads only once you it play), no `PlayerController`, no `GameState`.
There is no `GEngine` either.

#### What the C++-Constructor is good for

This is what the C++-constructor might look like:

```cpp
AOrbit::AOrbit()
{
    // setting defaulft values is fine
    PrimaryActorTick.bCanEverTick = true;
    PrimaryActorTick.bStartWithTickEnabled = false;
    
    bNetLoadOnClient = false;
    bReplicates = true;
    bAlwaysRelevant = true;
    AActor::SetReplicateMovement(false);
    
    // initializing components is fine
    // `CreateDefaultSubobject` is meant to be called in the constructor and nowhere else
    Root = CreateDefaultSubobject<USceneComponent>("Root");
    Root->SetMobility(EComponentMobility::Stationary);
    SetRootComponent(Root);

    Spline = CreateDefaultSubobject<USplineComponent>("Orbit");
    Spline->SetMobility(EComponentMobility::Stationary);
    
    // setup attachment is meant to be called here
    Spline->SetupAttachment(Root);
    Spline->SetIsReplicated(true);

    SplineMeshParent = CreateDefaultSubobject<USceneComponent>("SplineMeshes");
    SplineMeshParent->SetupAttachment(Root);
    SplineMeshParent->SetMobility(EComponentMobility::Stationary);

    TemporarySplineMeshParent = CreateDefaultSubobject<USceneComponent>("TemporarySplineMeshes");
    TemporarySplineMeshParent->SetupAttachment(Root);
    TemporarySplineMeshParent->SetMobility(EComponentMobility::Stationary);
}
```

#### You can't pass arguments to your C++-constructor

Well you can, technically, but it's no use.
You are free to overload the constructor with one that take arguments but it won't get called.

In the code sample above you see calls to `CreateDefaultSubobject` to initialize components,
which internally only ever calls the constructor without arguments.
If you drag a Blueprint actor into a scene, there is no way to pass arguments either.
If you use `UObject::SpawnActor` to instantiate an actor again the same.

### OnConstruction: quite different, yet similar limitations

`AActor::OnConstruction` is quite different from the C++-constructor in that it is a Unreal Engine characteristic and not a built-in feature of C++.
One consequence of this:
Inside your `OnConstruction` you usually see a call to `Super::OnConstruction`, to call the `OnConstruction` method of the parent class.
This call is optional and you can omit it (or even call to `SomeParentFurtherUp::OnConstruction) as you wish.
However, it is entirely impossible to have an object contructed *without running the C++constructors of its parents*.

Like with the constructor, it is worthwhile putting a breakpoint into your `OnConstruction` method to see how often it gets called and at what times.
Like with the constructor, you can't rely on Engine systems to be present, in general.
And like with the constructor, there are no arugments passed to `OnConstruction` except for `FTransform`.

#### What OnConstruction is good for

What you can and should do:
Use the `OnConstruction` method to set up an actor based on its properties.
All the properties, including components, have been intialized by the C++-constructor already.
Inside `OnConstruction` you can run code that depends on its properties and the properties of its components.
And this is how you can effectively pass arguments to the creation of an actor:

* Create your actor in C++: `AMyActor`
* Create a Blueprint that inherits from it `BP_MyActor`
* Set meaningful defaults for the actor properties in C++ (inside the class definition and the constructor)
* Play around with the actor properties inside the editor: no need to recompile anything!
* Anytime you change a property, `OnConstruction` is called
* Drag the Blueprint actor into the scene to create an instance

## Pushing the limits of OnConstruction

At some point, I decided that I want to spawn an actor from inside the `OnConstruction` method of another actor.
The reason for this:
I have asteroid actors that I want to drag into the editor
and whenever I do that, the asteroid actor shall spawn an orbit actor, which has a spline and determines the movement of the asteroid.

It is totally legit to spawn actors from inside `OnConstruction`,
but given that the `OnConstruction` method gets called a lot, we need to be smart about it.

After long hours of trying, crying, and questioning my existence I arrived at this code:

```cpp
AAsteroid::OnConstruction(FTransform& Transform)
{
    Super::OnConstruction(Transform);
    
    // necessary to avoid crashes 
    if(GetWorld()->WorldType != EWorldType::EditorPreview)
        && !HasAnyFlags(RF_Transient)
        )
    {
#if WITH_EDITOR
        if(IsValid(GetOrbit()) && GetOrbit()->Body != Actor)
        {
            // this actor has been copied in the editor (ALT + drag), to the effect that a new orbit is spawned in a first step
            // but in a second step, the Orbit is overwritten when the old values are copied over
            // therefore we need to find the correctly spawned orbit and set it
            FString OrbitLabel = AOrbit::MakeOrbitLabel(Actor);
            for(TObjectIterator<AOrbit> IOrbit; IOrbit; ++IOrbit)
            {
                if(    *IOrbit->GetWorld() == GetWorld()
                    && *IOrbit->GetActorLabel() == OrbitLabel
                  )
                {
                    SetOrbit(*IOrbit);
                }
            }
        }
#endif

        if(IsValid(GetOrbit()))
        {
            // this only ever gets executed when creating objects in the editor (and dragging them around)
            // the orbit is recalculated based on the asteroids new position
            GetOrbit()->UpdateByInitialParams();
        }
        else
        {
            // here we actually spawn a new orbit
            check(IsValid(GetOrbitClass()))

            FActorSpawnParameters Params;
            Params.CustomPreSpawnInitalization = [Actor] (AActor* ActorOrbit)
            {
                auto* Orbit = Cast<AOrbit>(ActorOrbit);
                Orbit->Body = Actor;
            };
            AOrbit* NewOrbit = Actor->GetWorld()->SpawnActor<AOrbit>(GetOrbitClass(), Params);
            SetOrbit(NewOrbit);
        }
    }
}    
```

The most important part happens right at the beginning.
By checking the `WorldType` of our current world, we make sure that we are not in `EWorldType::EditorPreview`.
The editor preview world is ephemeral and spawning actors there results in null pointers.

By ruling out the flag `RF_Transient` we do a little trick, which is necessary with Unreal Engine but not really great:
When you initially drag a Blueprint actor into a scene, right before you let go of the mouse button,
Unreal creates an instance that you drag around in front of you but it doesn't have a proper location.

This is different to moving an already placed actor via dragging later on.

This initial drag-instance is ephemeral and shouldn't spawn an orbit.
Once you let go of your mouse button, the actual instance is spawned and `OnConstrucion` gets called again.
We would end up with two orbits, only that the initial drag-instance doesn't have a location--the orbit is non-sensical.
By ruling out transient asteroid actors, we avoid these problems.

Note however, the ugliness:
In my project there are legitimate cases of transient actors spawning orbits later on.
E.g. the player controlled pawn that auto-spawns in my game at `BeginPlay` deserves an orbit, too
I have to create that one manually (I use `OnPossess`), because of the trick above.

## Some anti-patterns to avoid

Anti-patterns refer to a kind of solution that is popular but might cause pain and suffering later on.

The problem that the C++-constructor and the `OnConstruction` method do not generally have access to other objects,
is sometimes solved by checking for null pointers.

I.e. you try to get `GEngine` or the Game Instance from inside a constructor and if the result is a null pointer, you do nothing.
If the result is valid, you execute the code that now can rely on `GEngine` or the Game Instance being present.
Note how you basically silence a null-pointer exception this way.

There are cases when this approach is merited, but quite often it's not.
The first obvious recommendation is to limit your C++-constructor to the initialization of components and the setting of default values.
The second recommendation is: Fix the null-pointer exception, rather then silence it!

There can be different reasons for a null pointer.
The Game Instance might not be available, when

* Your editor is starting up and your calling `GetGameInstance` from inside a constructor
* The game is not running and your calling `GetGameInstance` from inside `OnConstruction`
* A cast failed: `GetGameInstance<UMyGameInstance>()` when the game instance isn't set to `UMyGameInstance`
* I cast to the wrong type or forget to cast: `GetGameInstance()->SomethingSpecificToUMyGameInstance`

This gets even worse for other Systems that might be available only after `BeginPlay`.
When multiplayer gets involved, certain systems will be available on the server only, and so on.

And in those cases, you probably want to fix your code rather then to silence the error.

In the above example, I check `WorldType` and the object flags, but after that I assume that my pointers are valid.





