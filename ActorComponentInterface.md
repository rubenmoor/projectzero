# Use C++-interfaces for more flexible handling of components

In case you wondered what place do C++ interfaces have in Unreal,
here is one nice and simple use case.

Be sure to check out the [official documentation on C++ interface](https://docs.unrealengine.com/5.0/en-US/interfaces-in-unreal-engine/), too.

## Components without C++ interfaces

Let's say you have some component called `MyComponent`.
Typical examples for components are ways to interact with an actor, e.g.

* A treasure box that contains loot has a `ContainsItems` component
* A note or a way sign has the `CanBeRead` component

In order to check at runtime, whether or not a given actor has some component,
in principle you can call

```cpp
UMyComponent* MyComponent = Cast<UMyComponent>(MyActor->FindComponentByClass(MyComponent::StaticClass()));
```

and `MyComponent` will be `null` if the actor doesn't have `MyComponent`.

## Adding an interface to the mix

The code is not only a bit longish,
there is also the disadvantage that `FindComponentByClass` has to walk through the component array of the actor every time.
A both, more elegant and more efficient solution is the implementation of a C++ interface for you component, like this:

```cpp

// file: MyComponent.h

UINTERFACE(meta=(CannotImplementInterfaceInBlueprint))
class UHasMyComponent : public UInterface
{
    GENERATED_BODY()
};

class IHasMyComponent
{
    GENERATED_BODY()
    
public:
    // the `= 0` makes sure that any class that inherits from this interface
    // *has* to implement `GetMyComonent`
    virtual UMyComponent* GetMyComponent() = 0;
};
```

In case you wonder why I defined two classes to define the interface, read the [documentation on interfaces in Unreal](https://docs.unrealengine.com/5.0/en-US/interfaces-in-unreal-engine/).

Any actor that uses `MyComponent` has to inherit the interface and implement the *pure virtual function* `GetMyComponent`.
Even better: inherit the interface first and then the compiler will remind you to implement `GetMyComponent`, 
which in turn reminds you to actually put that component on the actor.

```cpp
UCLASS()
class MYGAME_API AMyActor : public AActor, public IHasMyComponent
{
  GENERATED_BODY()

protected:
  UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
  TObjectPtr<UMyComponent> MyComponent;
  
public:
  // putting `override` there makes sure that we override an inherited method
  // instead of accidentally declaring a new method, e.g. because of some typo
  virtual UMyComponent* GetMyComponent() override { return MyComponent; }
  // and so on..
};
```

## Using the interface

Now, at runtime, you can run

```cpp
Cast<IHasMyComponent>(MyActor)->GetMyComponent();
```

to get the component, efficient and clean.
Even better, you don't have to risk a null-pointer exception:

```cpp
if(MyActor->Implements<UHasMyComponent>()) // Note `U` instead of `I` when using `Implements` like this!
{
  // savely access the interface
  Cast<IHasMyComponent>(MyActor)->GetMyComponent();
}
```

*Even better still*, you can add more useful methods to the interface.
I find it useful to add a method `Construction`:

```cpp

// file: MyComponent.cpp

void IHasMyComponent::Construction(const FTransform& Transform)
{
  UMyComponent* MyComponent = GetMyComponent();
  // run code that is aware of the component and whatever you pass as parameter
  // e.g. to do some component-specific setup;
}
```

In Unreal, actor components don't have construction scripts like Actors do.
(You can use `UActorComponent::PostInitProperties`,  if you like)
But you can call the `Construction` method from the interface in `MyActor::OnConstruction` now.

## Blueprint compatibility

Note that the above code has a disadvantage:
`GetMyComponent` is not a `UFUNCTION` and can't be called from Blueprint.
The line `UINTERFACE(meta=(CannotImplementInterfaceInBlueprint))` makes that explicit.

Following the documentation of C++ interfaces in Unreal, this can be fixed.
We pay a price, though.
Staying within the world of C++ has the advantage of the code remaining concise.
And the conciseness was one of the selling points for the use of interfaces.
If you need Blueprint support, however, this is how you do it:

First, remove the `meta=(CannotImplementInterfaceInBlueprint)` and change the declation in the interface to  the following:

```cpp
UFUNCTION(BlueprintNativeEvent)
UMyComponent* GetComponent();
```

We don't use `virtual` and we can't make the function purely virtual, either.

Second, the implementation in `MyActor` changes:

```cpp
virtual UMyComponent* GetMyComponent_Implementation() override { return MyComponent; }
```

Any call to what was formerly `GetMyComponent` changes.
You see the change in the `Construction` method.
We are supposed to call Blueprint-compatible interface methods like this, everywhere:

```cpp
// casting doesn't always work, use `Implements<>` instead
if(MyActor->Implements<UHasMyComponent>())
{
  // automatically generated `Execute_` function
  IHasMyComponent::Execute_GetMyComponent(MyActor);
}
```

Only this way, it is ensured that an actor that implements the interface via Blueprint gets notified.
The C++ way isn't aware of Blueprint-interfaces.

Cf. the [article on C++ interfaces in the unreal community wiki](https://unrealcommunity.wiki/interfaces-in-cpp-tjd0j1kk).

Finally, the last inconvenience with Blueprint-compatible interfaces:
The signature of `Execute_GetMyComponent` differs from the C++-way: It requires to pass a `UObject` as parameter.
This means our `Construction` method needs a change: 

```cpp

// file: MyComponent.h

void IHasMyComponent::Construction(UObject* Object)
{
  UMyComponent* MyComponent = Execute_GetMyComponent(Object);
  // run a construction script for the component
}
```

Properly adapting to the Unreal API makes us Blueprint-compatible at the expense of code that is a bit noisy.
You might consider renaming `GetMyComponent` to `Get`.
`IHasMyComponent::Get(MyActor)` is a bit more elegant.
However, if you have some other component with an interface according to this recipe,
you get a naming clash when trying to implement `Get_Implementation()` in your Actor.
The two methods differ in their return type (and their origin), but that is not enough to distinguish them.
