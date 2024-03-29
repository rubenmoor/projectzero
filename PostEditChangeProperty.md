# The simple case: no nesting

Unreal offers to override `PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent)` for any `UObject`.
Typically this is overridden for Actors or Actor Components.
E.g. you have a couple of properties that are logically linked and only changing one of them, would result in an inconsistency.
In that case you could write something like the following:

```cpp
// MyActor.h
  // inside the the class declaration
  #if WITH_EDITOR
  virtual void PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent) override;
  #endif

// MyActor.cpp
#if WITH_EDITOR
void MyActor::PostEditChangeProperty(FPropertyChangedEvent& PropertyChangedEvent)
{
  Super::PostEditChangeProperty(PropertyChangedEvent);
  const FName Name = PropertyChangedEvent.GetPropertyName();
  
  // assuming the existence of the properties "Velocity" and "VecVelocity"
  static const FName FNameVelocity = GET_MEMBER_NAME_CHECKED(MyActor, Velocity)
  
  if(Name == FNameVelocity)
  {
    // we are in *Post*EditChangeProperty, i.e. `Velocity` already has its new value
    VecVelocity = VecVelocity.GetUnsafeNormal() * Velocity;
  }
}
#endif
```

The pragmas `#if WITH_EDITOR` and the closing `#endif` are quite important.
If you don't put them everywhere, your project won't package.

# The general case: nested values

*However*, one might think you can complete the above code for the case when you edit `VecVelocity`,
which has type `FVector` instead of `double`.
The problem is that `const FName Name = PropertyChangedEvent.GetPropertyName();` takes the values "X", "Y", and "Z",
depending on what component of `VecVelocity` was changed.
The name of the property `VecVelocity` is lost and this is true for changing values inside any `UStruct`, too.
The solution is the use of `PostEditChangeChainProperty`, like so:

```cpp
// MyActor.h
// inside the the class declaration
#if WITH_EDITOR
virtual void PostEditChangeChainProperty(FPropertyChangedChainEvent& PropertyChangedEvent) override;
#endif

// MyActor.cpp
#if WITH_EDITOR
void MyActor::PostEditChangeChainProperty(FPropertyChangedChainEvent& PropertyChangedChainEvent)
{
  Super::PostEditChangeChainProperty(PropertyChangedChainEvent);
  const FName Name = PropertyChangedChainEvent.PropertyChain.GetHead()->GetValue()->GetFName();
  
  // assuming the existence of the properties "Velocity" and "VecVelocity"
  static const FName FNameVelocity    = GET_MEMBER_NAME_CHECKED(MyActor, Velocity   );
  static const FName FNameVecVelocity = GET_MEMBER_NAME_CHECKED(MyActor, VecVelocity);

  if(Name == FNameVelocity)
  {
    // we are in *Post*EditChangeChainProperty, i.e. `Velocity` already has its new value
    VecVelocity = VecVelocity.GetUnsafeNormal() * Velocity;
  }
  else if(Name == FNameVecVelocity)
  {
    Velocity = VecVelocity.Length();
  }
}
#endif
```

The `GetHead()` in `const FName Name = PropertyChangedEvent.PropertyChain.GetHead()->GetValue()->GetFName();` digs up the name "VecVelocity".
If you want to go a level deeper, you would use `GetHead().GetNextNode()`.
For the normal struct that isn't nested deeply, you can use `GetTail()`, which reveals "X", "Y", and "Z" in our case.

Now you might think that you use `PostEditChangeChainProperty` for nested properties and `PostEditChangeProperty` for everything else?
Well, as both methods get called always, there isn't really a need to use the less powerful `PostEditChangeProperty` *ever*.

## Bonus: how does this work together with `OnConstruction`?

I encountered a not-so-far-fetched case in which I ensure consistency of some Actor not only using the above `PostEditChangeChainProperty`,
but also `OnConstruction`.
The reason is simple:
When I drag the actor in the editor (with "Run construction script on drag" in the *class settings* enabled), `OnConstruction` gets called.
And thus within `OnConstruction` I have to react to the changing of a property, too.

This turned out to be a problem.
This is the order, in which relevant methods get called:

1. `PreEditChange(FProperty PropertyAboutToChange)`, we haven't talked about this one, yet.
2. `OnConstruction`
3. `PostEditChangeChainProperty`

In my case, ensuring consistency in `OnConstruction` restored the value that was edited, before `PostEditChangeChainProperty` got executed.
There, consistency was restored with an unchanged value.
Unfortunately, moving code to `PreEditChange` is no use:
This method doesn't have access to the new value:
When it gets called, the properties still have their pre-edit values.
And the same is true for the parameter `PropertyAboutToChange`, rendering this whole method mostly useless.

So you have to decide to either use `OnConstruction` *or* `PostEditChangeChainProperty`?

Not quite.
I solved my problem by introducing a new property (not a `UProperty`, just a class member): `bSkipConstruction` and using it like this:

```cpp
#if WITH_EDITOR
void PreEditChange(FProperty)
{
  bSkipConstruction = true;
}
#endif

void OnConstruction(FTransform&)
{
  // `bSkipConstruction` should default to `false`
  if(bSkipConstruction)
  {
    return;
  }
  // fill in function body
  // ...
}

#if WITH_EDITOR
void PostEditChangeChainProperty(FPropertyChangedChainEvent&)
{
  // fil in function body
  //
  bSkipConstruction = false;
}
#endif
```

Quite the annoying workaround.
But I really like my actors well behaved and consistent at all times.
