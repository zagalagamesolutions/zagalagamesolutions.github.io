---
layout: post
title: Considerations On Testing UE4 Classes Part II - Editor Clicks and Gamepad/Keyboard Button Presses
permalink: /articles/considerations-on-testing-ue4-classes-part-ii/
list_title: Considerations On Testing UE4 Classes Part II - Editor Clicks and Gamepad/Keyboard Button Presses
date: 2021-11-24
excerpt: >
  A guide to test editor clicks and gamepad/keyboard button presses in Unreal Engine 4.
---
# Considerations On Testing UE4 Classes Part II: Editor Clicks and Gamepad/Keyboard Button Presses

November 24th, 2021.

This is a series of consideration articles on testing C++ classes with UE4:

- [Part I: Worlds, Ticks and Forces](https://github.com/Floating-Island/articles/blob/main/Considerations%20On%20Testing%20UE4%20Classes%20Part%20I%20-%20Worlds%2C%20Ticks%20and%20Forces.md)
- Part II: Editor Clicks and gamepad/keyboard button presses
- [Part III: Replication](https://unrealcommunity.wiki/considerations-on-testing-ue4-classes:-part-iii-replication-2d68d4)

Hi, I’m going to show you how I’ve been doing editor clicks and gamepad/keyboard button pressing when testing UE4 C++ classes!

# Editor Clicks

Although testing UI is discouraged, one can test the logic parts of UI as User Acceptance Testing. In that case, we test against the UI logic layer of the application.

It’s a nice approach to create a layer on top of that UI logic interface. This new layer will be the only point of communication between our UI tests and the UI interface.

In Unreal Engine, this UI interface is [FSlateApplication](https://docs.unrealengine.com/en-US/API/Runtime/Slate/Framework/Application/FSlateApplication/index.html) , which handles the UI Events and editor windows.

A click is an event so there must be a method that processes it. The method that handles a click is: ProcessMouseButtonDoubleClickEvent:


```cpp
bool ProcessMouseButtonDoubleClickEvent( const TSharedPtr< FGenericWindow >& PlatformWindow,
 const FPointerEvent& InMouseEvent );
```

It needs a window in which the click was made and the click event that occurred.

You get a window creating a generic window, FGenericWindow, that communicates with the OS to get the space where the platform window resides.

FPointerEvent constructor parameters look like this:


```cpp
FPointerEvent(
		uint32 InPointerIndex,
		const FVector2D& InScreenSpacePosition,
		const FVector2D& InLastScreenSpacePosition,
		const TSet<FKey>& InPressedButtons,
		FKey InEffectingButton,
		float InWheelDelta,
		const FModifierKeysState& InModifierKeys
	);
```
Generally, you can instance it like this:

```cpp
FSlateApplication& slateApplication = FSlateApplication::Get();
const TSet<FKey> pressedButtons = TSet<FKey>({ EKeys::LeftMouseButton });
FPointerEvent mouseMoveAndClickEvent(
	0,
	slateApplication.CursorPointerIndex,
	atCoordinates,
	FVector2D(0, 0),
	pressedButtons,
	EKeys::LeftMouseButton,
	0,
	slateApplication.GetPlatformApplication()->GetModifierKeys()
);
```

atCoordinates are the (X, Y) coordinates where you want to click. 
You also need a method to retrieve the absolute coordinates of a button to be able to click it (FSlateApplication controls the main window of the application).

If you have a UI class with a button like this:

```cpp
UPROPERTY(meta = (BindWidget))
		UButton* textButton;
```

Then you could write a method that retrieves the button coordinates:

```cpp
buttonCoordinates()
{
	FVector2D buttonCenter = FVector2D(0.5f, 0.5f);
	return textButton->GetTickSpaceGeometry().GetAbsolutePositionAtCoordinates(buttonCenter);
}
```

I use the button center so we are sure that we’re clicking the button’s center and nothing else.

_I know that maybe adding code to a runtime class and only using it for tests might be a code smell but I haven’t found a better way to do it yet._

To process an editor click I created this method:

```cpp
void processEditorClick(FVector2D atCoordinates)
{
	FSlateApplication& slateApplication = FSlateApplication::Get();
	const TSet<FKey> pressedButtons = TSet<FKey>({ EKeys::LeftMouseButton });
	FPointerEvent mouseMoveAndClickEvent(
		0,
		slateApplication.CursorPointerIndex,
		atCoordinates,
		FVector2D(0, 0),
		pressedButtons,
		EKeys::LeftMouseButton,
		0,
		slateApplication.GetPlatformApplication()->GetModifierKeys()
	);
	TSharedPtr<FGenericWindow> genericWindow;
bool mouseClick = slateApplication.ProcessMouseButtonDoubleClickEvent(genericWindow, mouseMoveAndClickEvent);
}
```

You could add it to your UI test helper class and use it in one frame like this:

```cpp
FVector2D aButtonAbsoluteCoordinates = aUIObjectToTest->aButtonAbsoluteCenterPosition();
UE_LOG(LogTemp, Log, TEXT("resume button coordinates in viewport: %s"), *aClassButtonAbsoluteCoordinates.ToString());
UE_LOG(LogTemp, Log, TEXT("attempting click"));
aUITestSessionUtilities.processEditorClick(aClassButtonAbsoluteCoordinates);
```

And on the next frame you check if the logic intended in that button was accomplished by the test.


## Keyboard/Gamepad Button Presses

Suppose that you want to test that when pressing the button associated with the action mapping “accelerate” our jet would accelerate.

It’s logical to think that you would only need to retrieve a player controller, make it possess the jet to test and trigger the action mapping.

The problem is that it won’t work because to handle input, you need to spawn a player which owns that controller.

If we want to spawn a player, then we need to use something like this:


```cpp
void spawnLocalPlayer()
{
	AGameModeBase* testGameMode = testWorld->GetAuthGameMode();
	testGameMode->SpawnPlayerFromSimulate(FVector(0), FRotator(0));//spawns a player with controller and the default pawn set in the world game mode.
}
```

Now the problem is that the world attempts to summon a player with a player controller and pawn set in the world properties. So, we need to ensure that the world we’re testing has the jet as the default pawn (and the player controller class needed for that jet).

Then, we could call a method like this to trigger a button press in our test:


```cpp
//to be able to process inputs:
#include "GameFramework/PlayerInput.h"

//…

processLocalPlayerActionInputFrom(FName anActionMappingName)
{
	AGameModeBase* testGameMode = testWorld->GetAuthGameMode();
	APlayerController* controller = Cast<APlayerController, AActor>(testGameMode->GetGameInstance()->GetFirstLocalPlayerController(testWorld));
	processActionKeyPressFrom(anActionMappingName, controller);
}

processActionKeyPressFrom(FName anActionMappingName, APlayerController* aController)
{
	FName const actionName = anActionMappingName;
	TArray<FInputActionKeyMapping> actionMappings = aController->PlayerInput->GetKeysForAction(actionName);
	FKey actionKey = actionMappings[0].Key;//only the first key/button associated with that action mapping

	aController->InputKey(actionKey, EInputEvent::IE_Pressed, 5.0f, false);
}
```

In the frame following the call to processLocalPlayerActionInputFrom the button press is already done (and it’s kept pressed for 5 seconds).

You can also test axis mappings if you want to (filtering the axis scale to what you need):

```cpp
processKeyPressFrom(FName anAxisMappingName, APlayerController* aController)
{
	FName const actionName = anAxisMappingName;
	TArray<FInputAxisKeyMapping> axisMappings = aController->PlayerInput->GetKeysForAxis(actionName);
	FKey actionKey;
	for (auto axisMap : axisMappings)
	{
		if (axisMap.Scale > 0)
		{
			actionKey = axisMap.Key;
			break;
		}
	}
	aController->InputKey(actionKey, EInputEvent::IE_Repeat, 5.0f, false);
}
```

## Conclusion

I hope that I made a little easier for you to test clicking UI buttons and pressing keyboard/gamepad buttons.
Once you know how it works, it's easy to do it yourself ;).

Bye!

Alberto Mikulan