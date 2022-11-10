# Recipe: Tracking indicator for quest marks or important objects in the player's world

Imagine there is an important object in the world that you want to highlight for the player.
When the object is not visible on the screen you want to show an arrow at the edge of the screen, pointing in the direction of the objection.
In more technical terms:
If the projection of the objects world location to screen coordinates results in a position outside the viewporty, you display an indicator.

This mechanism makes sense for both, first-person view and third-person view.
The below example assumes a third-person view.

## The indicator

I use the [bindwidget approach](https://benui.ca/unreal/ui-bindwidget/) to access HUD elements in C++.
This means, I create a C++ class, `UW_HUD`, that inherits from `UUserWidget` and has the following member:

```cpp
UPROPERTY(meta=(BindWidget))
TObjectPtr<UCanvasPanel> CanvasIndicator;

UPROPERTY(meta=(BindWidget))
TObjectPtr<UImage> ImgPointer;
```

In the editor, I create a Blueprint that inherits from `UW_HUD` and call it "BP_UW_HUD".
Double-clicking it opens the UMG Widget designer where I have to add a Canvas Panel of the exact same name "CanvasIndicator".
Now I can put children into this Canvas Panel,
that is a small triangle that serves as arrow and an image that serves as symbol for the object that is being tracked.
I call that image `ImgPointer`.
Back to the C++ code, I can set the location of the same Canvas Panel (and the rotation of the image), but we will get to there in a bit.

## Setting up HUD and Widget

First, in `BeginPlay` of our HUD, we have to load the widget.

```cpp
// file MyHUD.h

/*
 * MyHUD: A C++ class that inherits from `AHUD`.
 * To use it, create a Blueprint "BP_MyHUD" that inherits from `MyHUD` and set the player HUD to "BP_MyHUD" in the Game Mode
 */
UCLASS()
class MYGAME_API AMyHUD : public AHUD
{
	GENERATED_BODY()

  // get access to private members of my Game Instance
	friend class UMyGameInstance;

protected:
  // `UUW_HUD` is the user widget that we created above;
  // `TSubClassOf<UUW_HUD>` holds any class that inherits from `UUW_HUD`;
  // In our case that will be the derived Blueprint "BP_UW_HUD";
  // After compiling this code, you will have to set this property in "BP_MyHUD" in the editor.
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="UMG Widget Classes")
	TSubclassOf<UUW_HUD> WidgetHUDClass;
	
	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category="UMG Widgets")
	TObjectPtr<UUW_HUD> WidgetHUD;
	
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	TObjectPtr<AMyCharacter> MyCharacter;

	virtual void BeginPlay() override;
	virtual void Tick(float DeltaSeconds) override;
	
};
```

```cpp
// file: MyHUD.cpp
void AMyHUD::BeginPlay()
{
	Super::BeginPlay();
	
  // when simulating in the editor, there is no Player Controller; no need to crash
	const AMyPlayerController* PC = Cast<AMyPlayerController>(GetOwningPlayerController());
	if(!PC)
	{
		UE_LOG(LogTemp, Warning, TEXT("%s: BeginPlay: no player controller, disabling tick"), *GetFullName())
		SetActorTickEnabled(false);
	}
	
	// get playing character
	MyCharacter = Cast<AMyCharacter>(GetOwningPawn());
	if(!IsValid(MyCharacter))
	{
		UE_LOG(LogTemp, Warning, TEXT("%s: BeginPlay: no pawn, disabling tick"), *GetFullName())
		SetActorTickEnabled(false);
	}

	UGameInstance* GI = GetGameInstance();
	
	// set up HUD

  // If we forgot to set the HUD Widget class in the Blueprint, we get a reminder instead of a crash
	if(!IsValid(WidgetHUDClass))
	{
		UE_LOG(LogSlate, Error, TEXT("%s: WidgetHUDClass null"), *GetFullName())
		return;
	}

  // Creating the widget in C++ and adding it to the viewport;
  // Note that the Blueprint is loaded, because we use `WidgetHUDClass` which we set in the editor
	WidgetHUD = CreateWidget<UUW_HUD>(GI, WidgetHUDClass, FName(TEXT("HUD")));
	WidgetHUD->SetVisibility(ESlateVisibility::HitTestInvisible);
	WidgetHUD->AddToViewport(0);
}
```

## Moving the indicator

```cpp
// file: MyHUD.cpp

void AMyHUD::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);
	
  const APlayerController* PC = GetOwningPlayerController();
  const FVector VecR = MyCharacter->GetActorLocation();
  
  // we need the viewport size for proper scaling
  const FVector2D Vec2DSize = UWidgetLayoutLibrary::GetViewportSize(GetWorld());
	FVector2D ScreenLocation;
	
  // `FVector VecF1` is the location of F1, the point in world space that I am tracking via screen indicator 
	// First, trying to project to screen coordinates, ...
	const bool bProjected = PC->ProjectWorldLocationToScreen(VecF1, ScreenLocation);
  // alternative method, don't know the advantages, but supposedly takes into account UI scaling
	//const bool bProjected = UWidgetLayoutLibrary::ProjectWorldLocationToWidgetPosition(GetOwningPlayerController(), VecF1, ScreenLocation, false);

  // I am using my own coordinates, X/YFromCenter, that go from 0 to 1
  // X = 0 is the left edge, X = 1 is the right edg
  // Y = 0 is the top, Y = 1 is the bottom
	float XFromCenter;
	float YFromCenter;
  
	// ... the first try doesn't always work, i.e. F1 must be in front of the camera, I believe, which isn't always the case
	if(!bProjected)
	{
		FVector CameraLocation;
		FRotator CameraRotation;
		PC->GetPlayerViewPoint(CameraLocation, CameraRotation);
    
    // ... but in that case we construct a new VecF1 that is right in front of the camera, in the same plane as the player
    // `VecR` is the world location of my player (third-person view)
		const FVector VecF1InViewportPlane =
			  VecF1
			+ CameraRotation.Vector()
			* (VecR - VecF1).Length();
		// ... and we try again
		if(!PC->ProjectWorldLocationToScreen(VecF1InViewportPlane, ScreenLocation))
		{
			// If the manual projection fails, too, we're out of options
			UE_LOG(LogActor, Error, TEXT("AMyHUD::Tick: couldn't project Center of mass to screen"))
		}
		// only we have to make sure that this manually conceived screen location `VecF1InViewportPlane` doesn't accidentally
		// end up on screen
    // ... we convert to the "FromCenter" coordinate system
		XFromCenter = ScreenLocation.X / Vec2DSize.X - .5;
		YFromCenter = ScreenLocation.Y / Vec2DSize.Y - .5;
    // ... and move the projected F1 out of the viewport, maintaining the direction of the indicator
		YFromCenter += copysign(0.5 * YFromCenter / XFromCenter, YFromCenter);
		XFromCenter += copysign(0.5, XFromCenter);
	}
	else
	{
    // convert to the "FromCenter" coordinate system and we are done projecting
		XFromCenter = ScreenLocation.X / Vec2DSize.X - .5;
		YFromCenter = ScreenLocation.Y / Vec2DSize.Y - .5;
	}
  
  // off-screen
  // thanks to the custom coordinate system, this is an easy check  
	if(abs(YFromCenter) > 0.5 || abs(XFromCenter) > 0.5)
	{
		WidgetHUD->CanvasIndicator->SetVisibility(ESlateVisibility::Visible);
		
    // this will be the position of `CanvasIndicator`
		float OverlayX, OverlayY;

		const auto Slot = UWidgetLayoutLibrary::SlotAsCanvasSlot(WidgetHUD->CanvasIndicator);

    // going case by case:
    // abs(YFromCenter) > abs(XFromCenter) implies that the indicator will be on the top or bottom edge,
    // i.e. the y position is fixed, while the x position moves
		if(abs(YFromCenter) > abs(XFromCenter))
		{
			// vertical case
      // the constant value of 0.5 is actually the distance from center to the to/bottom edge
			OverlayX = XFromCenter * 0.5 / abs(YFromCenter);
			if(YFromCenter < 0)
			{
				// above the viewport
				OverlayY = -0.5;
        // setting the alignment is important to make sure the indicator stays within the screen,
        // even when approaching the left and right edge
				Slot->SetAlignment(FVector2D(OverlayX + 0.5, 0.));
			}
			else
			{
				// below the viewport
				OverlayY = 0.5;
				Slot->SetAlignment(FVector2D(OverlayX + 0.5, 1.));
			}
		}
		else
		{
			// horizontal case
			OverlayY = YFromCenter * 0.5 / abs(XFromCenter);
			if(XFromCenter < 0)
			{
				// to the left of the viewport
				OverlayX = -0.5;
				Slot->SetAlignment(FVector2D(0., OverlayY + .5));
			}
			else
			{
				// to the right of the viewport
				OverlayX = 0.5;
				Slot->SetAlignment(FVector2D(1., OverlayY + .5));
			}
		}
		
    // the widget-way of setting the position of a Canvas Panel is a bit awkward via `SlotAsCanvasSlot`;
    // and we have to transform back to viewport coordinates
		UWidgetLayoutLibrary::SlotAsCanvasSlot(WidgetHUD->CanvasIndicator)->SetPosition(
			  FVector2D((OverlayX + .5) * Vec2DSize.X, (OverlayY + .5) * Vec2DSize.Y)
			/ ViewportScale);
		
    // The "ImgPointer" is a small triangle that serves as arrow and needs to rotate to show the right direction
		WidgetHUD->ImgPointer->SetRenderTransformAngle(atan2(YFromCenter, XFromCenter) * 180. / PI + 135);
	}
	// on-screen
	else
	{
    // don't show the indicator
    WidgetHUD->CanvasIndicator->SetVisibility(ESlateVisibility::Collapsed);
    // TODO: paint green circle around F1 when on screen
	}
}
```
