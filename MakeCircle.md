# How to draw a circle in Unreal/C++

In case you create a UI using C++, which is *not* a bad idea, you might find yourself overriding the `NativePaint` method
of a `UserWidget`.
This gives you access to a couple of primitive drawing functions.
You might prefer not to use them, but rather create any drawing in some graphics editor and just add images to your Unreal project via UMG.
I do enjoy the flexibility that I get by drawing stuff myself using primitives.

## NativePaint to draw stuff using C++

The whole setup goes like this:

### Create a User Widget

* Create a C++ class that inherits from `UUserWidget` and call it `UW_MyWidget`.
* Create a Blueprint that inherits from `UW_MyWidget and call it `BP_UW_MyWidget`.
* In C++ implement the `NativePaint` method via override:

```cpp
// file: UW_MyWidget

UCLASS()
class MYGAME_API UUW_HUD : public UUserWidget
{
	GENERATED_BODY()
  
protected:
	virtual int32 NativePaint(const FPaintArgs& Args, const FGeometry& AllottedGeometry, const FSlateRect& MyCullingRect, FSlateWindowElementList& OutDrawElements, int32 LayerId, const FWidgetStyle& InWidgetStyle, bool bParentEnabled) const override;
};

// file: UW_MyWidget.cpp

int32 UUW_HUD::NativePaint(const FPaintArgs& Args, const FGeometry& AllottedGeometry,
                           const FSlateRect& MyCullingRect, FSlateWindowElementList& OutDrawElements, int32 LayerId,
                           const FWidgetStyle& InWidgetStyle, bool bParentEnabled) const
{
	const FPaintGeometry PG = AllottedGeometry.ToPaintGeometry();
	
	// draw stuff
	
  // don't forget this, otherwise some layer won't be drawn
	const int32 LayerIdNew = LayerId + 1;
  
	return Super::NativePaint(Args, AllottedGeometry, MyCullingRect, OutDrawElements, LayerIdNew, InWidgetStyle,
	                          bParentEnabled);
}
```

### Create a HUD

* Create a C++ class that inherits from `AHUD` and call it `MyHUD`
* Create a Blueprint that inherits from `MyHUD` and call it `BP_MyHUD`
* Set the the player HUD in the Game Mode to `BP_MyHUD`
* In C++, add a property for the user widget's class and for the user widget itself:

```cpp
// file: AMyHUD.h

UCLASS()
class MYGAME_API AMyHUD : public AMyHUDBase
{
	GENERATED_BODY()

protected:
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category="UMG Widget Classes")
	TSubclassOf<UUW_MyWidget> MyWidgetClass;
	
	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category="UMG Widgets")
	TObjectPtr<UUW_MYWidget> MyWidget;
  
  // see below for the implementation
  virtual void BeginPlay() override;
};
```

* In `BeginPlay` of the HUD, add the widget to the viewport

```cpp
// file: AMyHUD.cpp

void AMyHUD::BeginPlay()
{
	Super::BeginPlay();
	
	if(!IsValid(MyWidgetClass))
	{
		UE_LOG(LogSlate, Error, TEXT("%s: WidgetHUDClass null"), *GetFullName())
		return;
	}

	MyWidget = CreateWidget<UUW_MyWidget>(GI, MyWidgetClass, FName(TEXT("MyWidget")));
	MyWidget->SetVisibility(ESlateVisibility::HitTestInvisible);
	MyWidget->AddToViewport(0);
}
```

Now we are set up to draw stuff, using C++.

## Now to the circle

Going back to the implementation of `NativePaint` in `UW_MyWidget.cpp`, you can start to type `FSlateDrawElement::` to see what your editor suggests
(given you have some autocompletion feature in your editor).
I get a couple of methods that start with `Make`:

* MakeBox
* MakeCustom
* MakeGradient
* MakeLines
* MakeSpline
* MakeText
* MakeViewport
* MakeCustomVerts
* MakeDebugQuad
* MakeRotatedBox
* MakeShapedText
* MakeCubicBezierSpline
* MakeDrawSpaceSpline
* MakePostProcessPass
* MakeDrawSpaceGradientSpline
* MakeBoxInternal

Unfortunately, I don't know what all of those do, apart from the obvious once.
And unfortunately, too, there is nothing close to `MakeCircle` (or even make ellipse).
You can construct a circle from four splines, and this is what this recipe is about:

```cpp
FVector2D Center = FVector2D(400, 300);
float Radius = 32;
float Thickness = 0;
ESlateDrawEffect DrawEffect = ESlateDrawEffect::None;
FLinearColor Tint = FLinearColor::White;

constexpr float C = 1.65;
const FVector2D P1 = Center + FVector2D(-Radius, 0);
const FVector2D T1 = FVector2D(0, -Radius) * C;
const FVector2D P2 = Center + FVector2D(0, -Radius);
const FVector2D T2 = FVector2D(Radius, 0) * C;
const FVector2D P3 = Center + FVector2D(Radius, 0);
const FVector2D T3 = FVector2D(0, Radius) * C;
const FVector2D P4 = Center + FVector2D(0, Radius);
const FVector2D T4 = FVector2D(-Radius, 0) * C;

FSlateDrawElement::MakeSpline(OutDrawElements, LayerId, PG, P1, T1, P2, T2, Thickness, DrawEffect, Tint);
FSlateDrawElement::MakeSpline(OutDrawElements, LayerId, PG, P2, T2, P3, T3, Thickness, DrawEffect, Tint);
FSlateDrawElement::MakeSpline(OutDrawElements, LayerId, PG, P3, T3, P4, T4, Thickness, DrawEffect, Tint);
FSlateDrawElement::MakeSpline(OutDrawElements, LayerId, PG, P4, T4, P1, T1, Thickness, DrawEffect, Tint);
```

The magic number `C = 1.65` is a mystery to me, too.
But it creates tangents that work well for a circle that is perfect for all I can tell.
