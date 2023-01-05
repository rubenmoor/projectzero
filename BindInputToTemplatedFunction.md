Binding input actions to some code usually looks like this:

```cpp
void AMyPlayerController::SetupInputComponent()
{
	Super::SetupInputComponent();
	
  InputComponent->BindAction(TEXT("Forward") , IE_Pressed, this, &AMyPlayerController::HandleForward );
  InputComponent->BindAction(TEXT("Backward"), IE_Pressed, this, &AMyPlayerController::HandleBackward);
  InputComponent->BindAction(TEXT("Left")    , IE_Pressed, this, &AMyPlayerController::HandleLeft    );
  InputComponent->BindAction(TEXT("Right")   , IE_Pressed, this, &AMyPlayerController::HandleRight   );
  // ...
}
```

Where you have to implement every single of these methods:

```cpp
AMyPlayerController::HandleForward()
{
  // ...
}

AMyPlayerController::HandleBackward()
{
  // ...
}

AMyPlayerController::HandleLeft()
{
  // ...
}

AMyPlayerController::HandleRight()
{
  // ...
}
```

This becomes tedious in cases where each of these methods do similar stuff and you would like to deal with the movement with something like this:

```cpp
AMyPlayerController::HandleMovement(EMovementType MovementType)
{
  // just an example:
  switch(MovementType)
  {
  case EMovementType::Forward:
    // ...
    break;
  // ...
  }
}
```

Unfortunately, you can't pass arguments to the function that you bind to.
Except you can, with [template parameters]([url](https://en.cppreference.com/w/cpp/language/template_parameters)).

This is how the above code gets simplified:

```cpp
void AMyPlayerController::SetupInputComponent()
{
	Super::SetupInputComponent();
	
  InputComponent->BindAction(TEXT("Forward") , IE_Pressed, this, &AMyPlayerController::HandleMovement<EMovementType::Forward> );
  InputComponent->BindAction(TEXT("Backward"), IE_Pressed, this, &AMyPlayerController::HandleMovement<EMovementType::Backward>);
  InputComponent->BindAction(TEXT("Left")    , IE_Pressed, this, &AMyPlayerController::HandleMovement<EMovementType::Left>    );
  InputComponent->BindAction(TEXT("Right")   , IE_Pressed, this, &AMyPlayerController::HandleMovement<EMovementType::Right>   );
  // ...
}

template<EMovementType MovementType>
AMyPlayerController::HandleForward()
{
  // again, just an exmaple. You don't have to use an enum as the parameter
  switch(MovementType)
  {
  case EMovementType::Forward:
    // ...
    break;
  // ...
  }
}
```

Note that this templated code is run at compile-time and can't rely on value that are determined at runtime.
I.e. when you call a templated function like `HandleMovement<SomeTemplateParameter>()`, the template parameter must be a contant expression.
It can't be any variable or the result of some function (static casts are allowed, though).
