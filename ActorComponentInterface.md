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

UINTERFACE()
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
Cast<IHasOrbit>(MyActor)->GetMyComponent();
```

to get the component, efficient and clean.
Even better, you don't have to risk a null-pointer exception:

```cpp
if(MyActor->Implements<IHasMyComponent>())
{
  // savely access the interface
  Cast<IHasOrbit>(MyActor)->GetMyComponent();
}
```

*Even better still*, you can add more useful methods to the interface.
I find it useful to add a method `Construction`:

```cpp

// file: MyComponent.cpp

void IHasMyComponent::Construction(const Transform& Transform)
{
  // run a construction script for the component
}
```

In Unreal, actor components don't have construction scripts like Actors do.
(You can use `UActorComponent::PostInitProperties`,  if you like)
But you can call the `Construction` method from the interface in `MyActor::OnConstruction` now.

## Blueprint compatibility

Note that the above code has a disadvantage:
`GetMyComponent` is not a `UFUNCTION` and can't be called from Blueprint.

Following the documentation of C++ interfaces in Unreal, this can be fixed.
Just change the declation in the interface to  the following:

```cpp
UFUNCTION(BlueprintNativeEvent)
UMyComponent* GetComponent();
```

We don't use `virtual` and we can't make the function purely virtual, either.
The implementation in `MyActor` changes, too:

```cpp
virtual UMyComponent* GetMyComponent_Implementation() override { return MyComponent; }
```
