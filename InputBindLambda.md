# Use a closure to bind actions to the Input Component

This is from [a comment of Neverender in the official forums](https://forums.unrealengine.com/t/can-i-use-bindaction-with-lambda/353633).

Closures in C++ are great.
They are called [Lambda Expressions](https://en.cppreference.com/w/cpp/language/lambda).
Unreal Engine has decent support for lambda expressions.
E.g., unfortunately, it isn't possible to bind anonymous functions to dynamic delegates.
But it is possible to bind anonymous functions to events and static delegates, using `BindLambda` and `AddLambda`.
In the case of the Input Component, there is a workaround that let's you define your own version of `BindLambda`.

Implement the following method `BindInputLambda` in the Player Controller, like this:

```cpp
// file: MyPlayerController.cpp
void AMyPlayerController::BindInputLambda(const FName ActionName, EInputEvent KeyEvent, TFunction<void()> Handler)
{
	FInputActionBinding Binding(ActionName, KeyEvent);
	Binding.ActionDelegate.GetDelegateForManualSet().BindLambda(Handler);
	InputComponent->AddActionBinding(Binding);
}
```

And you can use it in `SetupInputComponent` like this:

```cpp
// file: MyPlayerController.cpp
void AMyPlayerController::SetupInputComponent()
{
	Super::SetupInputComponent();

  // This is an example for an action that change player state stored in the Local Player
  // The associated `Player` of a Player Controller doesn't change, so it's save to set it once in the beginning
  // ... and include it in the closure
	UMyLocalPlayer* LocalPlayer = Cast<UMyLocalPlayer>(Player);
	
	BindInputLambda("Escape", IE_Pressed, [this, LocalPlayer] ()
	{
    // handle the escape key
    // e.g.
    if(LocalPlayer->GetIsInMainMenu())
    {
      GetHUD<AMyHUD>()->MainMenuHide();
      // ...
    }
  });
}
```

## A pitfall: passing existing variables into the closure

In C++ within the square brackets, you define what variables pass from the outer scope of the closure into its inner scope.
That is to say, in these square brackets you define the *capture* of the closure.
If you like to pass all available symbols, you can use

* `[=]` to pass all available symbols by value
* or `[&]` to pass all available symbols by reference
* *or* even make a mix like `[&, LocalPlayer]` to pass all available symbols by reference, except `LocalPlayer`

Note that using `[&]` or `[=]` is usually not a problem:
The compile will only include symbols that are *actually used* in the body of the lambda expression.
However, I like to explicitly list all elements of the capture.
This way, my IDE sometimes helps find errors and while boilerplate can get annoying at times, here I don't mind it.

Because:

BE AWARE of what you include in the capture.
In the above example, `LocalPlayer` is captured when `SetupInputComponent` executes.
Syntactially, it would be fine to write the following code instead:

```cpp
  bool bInMainMenu = LocalPlayer->GetIsInMainMenu();
  BindInputLambda("Escape", IE_Pressed, [this, bInMainMenu] ()
	{
    // You probably don't want to do this
    if(bInMainMenu)
    {
      GetHUD<AMyHUD>()->MainMenuHide();
      // ...
    }
  });
```

However, this is very likely not what you want.
In the above example, `GetIsInMainMenu()` is only evaluated once, while you need to evaluate it when reacting to the action, i.e. inside
the body of the lambda expression.
