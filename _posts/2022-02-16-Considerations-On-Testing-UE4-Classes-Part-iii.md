---
layout: articles
title: Considerations On Testing UE4 Classes Part III - Replication
permalink: /articles/considerations-on-testing-ue4-classes-part-iii/
list_title: Considerations On Testing UE4 Classes Part III - Replication
date: 2022-02-16
excerpt: >
  A guide to test replication in Unreal Engine 4 using networked Play In Editor (PIE) sessions.
---
# Considerations On Testing UE4 Classes Part III: Replication

February 16th, 2022.

This is a series of consideration articles on testing C++ classes with UE4:

- [Part I: Worlds, Ticks and Forces](https://github.com/Floating-Island/articles/blob/main/Considerations%20On%20Testing%20UE4%20Classes%20Part%20I%20-%20Worlds%2C%20Ticks%20and%20Forces.md)
- [Part II: Editor Clicks and gamepad/keyboard button presses](https://github.com/Floating-Island/articles/blob/main/Considerations%20On%20Testing%20UE4%20Classes%20Part%20II%20-%20Editor%20clicks%20and%20Gamepad_Keyboard%20button%20presses.md)
- Part III: Replication

Hi, I’m going to show you how I’ve been doing simple replication when testing UE4 C++ classes!

# Simple replication via PIE

Ok, you want to test your game via LAN before jumping to online. You learned about clients and server, asynchronous calls, etc. Now what?

To test replication you normally have to spin up two instances of UE4 in order to have a minimum of a client and a server.
Another way of doing that (although a little buggy) is creating two instances in PIE via a networked PIE session. This later way of doing replication is the one I'll be talking in this article :).


# Configuring a PIE session

In order to make a networked PIE session, we have to set some play settings. 
For that, we use [ULevelEditorPlaySettings](https://docs.unrealengine.com/4.26/en-US/API/Editor/UnrealEd/Settings/ULevelEditorPlaySettings/) to configure the session.
It has a lot of options to personalize our session but we only need two for replication:
- SetPlayNetMode (which receives a [EPlayNetMode](https://docs.unrealengine.com/4.26/en-US/API/Editor/UnrealEd/Settings/EPlayNetMode/) to choose from a standalone, listen server or client session).
- SetPlayNumberOfClients (to set the number of instances, including server, in the case of a listen server).

In replication tests I always use the PIE_ListenServer net mode.

Then, to request the PIE session we need [FRequestPlaySessionParams](https://docs.unrealengine.com/4.26/en-US/API/Editor/UnrealEd/FRequestPlaySessionParams/).
It also has a bunch of options to set but again we only need to set two of them:
- EditorPlaySettings (where we put our ULevelEditorPlaySettings)
- DestinationSlateViewport (to know in which viewport to start the PIE session)

For DestinationSlateViewport we need to first get the [FLevelEditorModule](https://docs.unrealengine.com/4.27/en-US/API/Editor/LevelEditor/FLevelEditorModule/) and check with the [FModuleManager](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Core/Modules/FModuleManager/) that it's loaded.
When we finally have the FLevelEditorModule, we simply retrieve its first active viewport to use as slate viewport for the FRequestPlaySessionParams.

then, we only need to RequestPlaySession with GUnrealEd (UUnrealEdEngine).

This is how it looks like in code:


```cpp
#include "LevelEditor.h"
#include "Editor/UnrealEdEngine.h"
#include "UnrealEdGlobals.h"

...
int numParticipants = 2;// a client and a server

{
FLevelEditorModule& LevelEditorModule = FModuleManager::Get().GetModuleChecked<FLevelEditorModule>(TEXT("LevelEditor"));

ULevelEditorPlaySettings* playSettings =  GetMutableDefault<ULevelEditorPlaySettings>();
playSettings->SetPlayNumberOfClients(numParticipants);
playSettings->SetPlayNetMode(EPlayNetMode::PIE_ListenServer);

FRequestPlaySessionParams sessionParameters;
sessionParameters.DestinationSlateViewport = LevelEditorModule.GetFirstActiveViewport();//sets the server viewport in there. Otherwise, a window is created for the server.
sessionParameters.EditorPlaySettings = playSettings;	

GUnrealEd->RequestPlaySession(sessionParameters);
}
```
An example of using this code is found [here](https://github.com/Floating-Island/ProjectR/blob/d15f226ddbb9bd30ad9e14d5077d5f250f792738/Source/TestingModule/Testing/Commands/NetworkCommands.cpp) in line 12.


This way we can now create networked PIE sessions!

But how do we differentiate a server from a client?

# Differentiating Server from Clients

For this, you have to remember that the server always has the controllers of all the players, whereas the clients only have their own controller.
Then the problem is simpler and we only need the [FWorldContext](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Engine/Engine/FWorldContext/)(s):

```cpp
retrieveServerWorldContext(int expectedControllersInServer)
{
	FWorldContext serverWorldContext = FWorldContext();
	const TIndirectArray<FWorldContext>& worldContexts = GEditor->GetWorldContexts();
	for (auto& worldContext : worldContexts)
	{
		int numberOfControllers = worldContext.World()->GetNumPlayerControllers();
		if (numberOfControllers == expectedControllersInServer)
		{
			serverWorldContext = worldContext;
			return serverWorldContext;
		}
	}
	return serverWorldContext;
}
```

Then, the other world contexts will be clients.

This same function is used as a class method [here](https://github.com/Floating-Island/ProjectR/blob/d15f226ddbb9bd30ad9e14d5077d5f250f792738/Source/TestingModule/Testing/Utilities/NetworkedPIESessionUtilities.cpp) in line 14.

When checking something in a client world context, keep in mind that the server might not be ready for the client, given the asynchronous nature of networking. A good way of knowing if the server is ready is to check that it has all the expected controllers.

Also, if you want to spawn a replicated pawn and be controlled by a client, then you have to spawn it from the server and assign it to a controller with an index other than 0, because that one is the one that 'belongs' to the server.

As with other tests, creating a networked PIE session, retrieving server and clients and working with them has to be done with latent commands.

You can find many examples of this kind of replication in my [project](https://github.com/Floating-Island/ProjectR), for example [here](https://github.com/Floating-Island/ProjectR/blob/54771d8a6b97d62edc8c5e2ff237159ae746514b/Source/TestingModule/Testing/Tests/Jet/JetTest.cpp) in line 645.





## Conclusion

I hope that I made a little easier for you to test replication via PIE.
Once you know how it works, it's easy to do it yourself ;).

Keep in mind that networked PIE sessions are buggy but it's faster than spinning up two instances of the editor.

Bye!

Alberto Mikulan