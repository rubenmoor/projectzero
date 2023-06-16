## This is buggy ...

Unfortunately, while using the `UComboBoxKey` like explained below, I encountered a very hard-to-reproduce bug.
The game would just crash eventually (access violation) without meaningful information in the stack trace.
The crash appeared whenever I spend time in the menus of the UI.
The stacktrace was somewhere deep in the slate widget hierarchy, no code of mine involved.
I switched to `UComboBoxString` which works similar, but without the events `OnGenerateContentWidget` and `OnGenerateItemWidget`.

## ... so go ahead at your own risk.

The combo box is a straightforward widget, or at least it should be.
It comes in two versions, `UComboBoxString` and `UComboBoxKey`.
If I get it right, `UComboBoxString` is simplified:
You provide options, or "items" of type `FString` and the user selects one of them.

The `UComboBoxKey`, in turn, allows you to put items of type `FName` as options.
Moreover, `UComboBoxKey` allows to put custom widgets to render:

* The combo box itself, typically with a selected item, and
* all the items in a vertical list, typically while the user selects an item.

This is how you set up sort of a minimal `UComboBoxKey`.
In the example it is used to select a screen resolution.
Read the comments to get a better idea of what the code does.

```cpp
// file: UW_Settings.h
#pragma once

#include "CoreMinimal.h"
#include "CommonTextBlock.h"
#include "Blueprint/UserWidget.h"

#include "UW_Settings.generated.h"

class UMyCommonButton;
class UCommonTextBlock;
class UComboBoxKey;

/**
 * To use this, create a blueprint in the editor that inherits from this class
 * Children of `UUserWidget` can then used just like any other widget.
 * You can also add them as parent widget to the viewport, using C++
 */
UCLASS()
class UUW_Settings : public UUserWidget
{
    GENERATED_BODY()

protected:
    // events
    virtual void NativeOnInitialized() override;

    // widgets
    
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UComboBoxKey> ComboResolution;
    
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UMyCommonButton> BtnApply;
    
    UPROPERTY(meta=(BindWidget))
    TObjectPtr<UMyCommonButton> BtnBack;

    // styling of the text that we put inside the combobox
    // with this in place, you can use a style asset that was created in the editor
    UPROPERTY(EditAnywhere)
    TSubclassOf<UCommonTextStyle> TextStyle;

private:
    // I prefer this to the `FString`s of `UComboBoxString`: any item among the combo box options is a key in the map
    TMap<FName, FIntPoint> Resolutions;
    
    // event handlers

    UFUNCTION()
    void HandleResolutionSelect(FName Item, ESelectInfo::Type _);
    
    // here, the widget is created that is shown in the combo box; we just put a `UCommonTextBlock`
    UFUNCTION()
    UWidget* HandleComboResolutionGenerate(FName Item);

    // here, the widget is created for every item in the dropdown menu when the user has clicked the combo
    // box to make their selection; again we just put a `UCommonTextBlock`
    UFUNCTION()
    UWidget* HandleComboResolutionItemGenerate(FName Item);
};
```

Unfortunately, `UComboBox`s rely on dynamic delegates and they don't allow the binding of lambdas.
The result is all those class methods starting with `Handle`.

```cpp
// file: UW_Settings.cpp

#include "UW_Settings.h"

#include "Blueprint/WidgetTree.h"
#include "Components/ComboBoxKey.h"

// user interface settings: unreal provides a lot of stuff out of the box
#include "Engine/UserInterfaceSettings.h"

// sorry, this is my custom `UCommonButton` child. When using `UCommonButton` you have to implement
// a child class
#include "HUD/MyCommonButton.h"

// again custom code here
// I am using two different HUDs, because the user can access the settings from the main menu
// **and** from the in-game menu
#include "HUD/MyHUD.h"
#include "HUD/MyHUDMenu.h"

// I am overriding `ULocalPlayer` and store in there whether the player is inside the game or inside the main menu
#include "Modes/MyLocalPlayer.h"

void UUW_Settings::NativeOnInitialized()
{
    Super::NativeOnInitialized();
    
    // cf. https://benui.ca/unreal/ui-resolution/
    // adding key-value pairs to the map
    Resolutions.Add("1280x720" , {1280, 720  });
    Resolutions.Add("1024x768" , {1024, 768  });
    Resolutions.Add("1360x768" , {1360, 768  });
    Resolutions.Add("1365x768" , {1365, 768  });
    Resolutions.Add("1280x800" , {1280, 800  });
    Resolutions.Add("1440x900" , {1440, 900  });
    Resolutions.Add("1600x900" , {1600, 900  });
    Resolutions.Add("1280x1024", {1280, 1024 });
    Resolutions.Add("1680x1050", {1680, 1050 });
    Resolutions.Add("1920x1080", {1920, 1080 });
    Resolutions.Add("2560x1080", {2560, 1080 });
    Resolutions.Add("1920x1200", {1920, 1200 });
    Resolutions.Add("2560x1440", {2560, 1440 });
    Resolutions.Add("3440x1440", {3440, 1440 });
    Resolutions.Add("3840x2160", {3840, 2160 });
        
    for(const auto Res : Resolutions)
    {
        ComboResolution->AddOption(Res.Key);
    }
    
    // this is how I prefer to bind events
    BtnBack->OnClicked().AddLambda([this] ()
    {
        if(Cast<UMyLocalPlayer>(GetPlayerContext().GetLocalPlayer())->GetIsInMainMenu())
        {
            GetPlayerContext().GetHUD<AMyHUDMenu>()->MenuMainShow();
        }
        else // in-game menu
        {
            GetPlayerContext().GetHUD<AMyHUD>()->InGameMenuShow();
        }
    });

    // using the game user settings utility as provided by unreal engine
    BtnApply->OnClicked().AddLambda([] ()
    {
        GEngine->GetGameUserSettings()->ApplyResolutionSettings(false);
    });

    // if you don't bind either one of the following two, the widget reminds you that something's missing
    ComboResolution->OnGenerateItemWidget.BindDynamic(this, &UUW_Settings::HandleComboResolutionItemGenerate);
    ComboResolution->OnGenerateContentWidget.BindDynamic(this, &UUW_Settings::HandleComboResolutionGenerate);
    
    ComboResolution->OnSelectionChanged.AddUniqueDynamic(this, &UUW_Settings::HandleResolutionSelect);
}

void UUW_Settings::HandleResolutionSelect(FName Item, ESelectInfo::Type)
{
    auto* Settings = GEngine->GetGameUserSettings();
    // getting the resolution of type `FIntPoint` from the map. so clean.
    Settings->SetScreenResolution(Resolutions[Item]);
    
    // when the selected settings differ from the actual settings, the "apply" button is enabled
    BtnApply->SetIsEnabled(Settings->IsDirty());
}

UWidget* UUW_Settings::HandleComboResolutionGenerate(FName Item)
{
    auto* GI = GetGameInstance();
    auto* TextItem = WidgetTree->ConstructWidget<UCommonTextBlock>();
    // the text style is a class member and must be set to an asset in the editor
    TextItem->SetStyle(TextStyle);
    
    // the lazy way: just make a string from the `Name`
    TextItem->SetText(FText::FromName(Item));
    
    // but you see, you can put any widget here
    return TextItem;
}

UWidget* UUW_Settings::HandleComboResolutionItemGenerate(FName Item)
{
    auto* GI = GetGameInstance();
    auto* TextItem = WidgetTree->ConstructWidget<UCommonTextBlock>();
    TextItem->SetStyle(TextStyle);
    TextItem->SetText(FText::FromName(Item));
    return TextItem;
}
```
