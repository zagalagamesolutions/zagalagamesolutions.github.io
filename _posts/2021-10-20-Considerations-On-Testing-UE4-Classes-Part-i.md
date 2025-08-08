---
layout: post
title: Considerations On Testing UE4 Classes Part I - Worlds, Ticks and Forces
permalink: /articles/considerations-on-testing-ue4-classes-part-i/
list_title: Considerations On Testing UE4 Classes Part I - Worlds, Ticks and Forces
date: 2021-10-20
excerpt: >
  A guide to test C++ classes in Unreal Engine 4, focusing on worlds, ticks, and forces.
---
# Considerations On Testing UE4 Classes Part I: Worlds, Ticks and Forces

October 20th, 2021.

This is a series of consideration articles on testing C++ classes with UE4:

- Part I: Worlds, Ticks and Forces
- [Part II: Editor Clicks and gamepad/keyboard button presses](https://unrealcommunity.wiki/considerations-on-testing-ue4-classes:-part-ii-8b09e4)
- [Part III: Replication](https://unrealcommunity.wiki/considerations-on-testing-ue4-classes:-part-iii-replication-2d68d4)


Hi! I assume that you have read the [Automation Technical Guide](https://docs.unrealengine.com/4.26/en-US/TestingAndOptimization/Automation/TechnicalGuide/) for the creation of tests.

# Worlds, Ticks and Forces

Imagine that we want to accelerate a jet (***AJet***). To do that we should create first a test, inside it we create an instance of ***AJet***, try to accelerate it and verify that it was accelerated.

To simplify the implementation, let’s suppose that we want to accelerate our jet along the X axis and the acceleration is positive.

The test would look something like this:

```cpp
bool FAJetMovesForwardWhenAcceleratedTest::RunTest(const FString& Parameters)
{
    AJet* testJet = NewObject<AJet>();
    float initialLocation = testJet->GetActorLocation().X;
    testJet->accelerate();
    float finalLocation = testJet->GetActorLocation().X;
    TestTrue(FString("Jet X location should increase when accelerated."), finalLocation > initialLocation);
    return true;
}
```

*accelerate* implementation is this:

```cpp
AJet::accelerate()
{
    physicsMeshComponet->AddForce(FVector(5000, 0 , 0));
}
```

## Worlds

The previous implementation looks like it might work, but we would be forgetting one thing: we are creating an object instead of making it spawn in a world. No matter how much acceleration we apply to it, it wouldn’t be moving in a physical space.

Then, we must first load a map into the editor and spawn the jet into it:

```cpp
bool FAJetMovesForwardWhenAcceleratedTest::RunTest(const FString& Parameters)
{
    ADD_LATENT_AUTOMATION_COMMAND(FEditorLoadMap(FString("/Game/Tests/TestMaps/VoidWorld")));
    
    AJet* testJet = Cast<AJet, AActor>( GEditor->GetPIEWorldContext()->World()->SpawnActor(AJet::StaticClass()) );

    float initialLocation = testJet->GetActorLocation().X;
    testJet->accelerate();
    float finalLocation = testJet->GetActorLocation().X;
    TestTrue(FString("Jet X location should increase when accelerated."), finalLocation > initialLocation);
    return true;
}
```

FEditorLoadMap is a latent command present in Unreal Engine 4.

VoidWorld is the name of a map created in the editor.

## Worlds and Ticks

This implementation of the test is better, but there’re still two other problems: loading a map into the editor and spawn an actor inside a map are asynchronous actions (their completion could take many frames). That’s why we have to separate our code into smaller latent commands:

```cpp
bool FAJetMovesForwardWhenAcceleratedTest::RunTest(const FString& Parameters)
{
    ADD_LATENT_AUTOMATION_COMMAND(FEditorLoadMap(FString("/Game/Tests/TestMaps/VoidWorld")));
    ADD_LATENT_AUTOMATION_COMMAND(FSpawnJet);
    ADD_LATENT_AUTOMATION_COMMAND(FRetrieveAccelerateAndCheckJet(this)));
    return true;
}
```

```cpp
DEFINE_LATENT_AUTOMATION_COMMAND(FSpawnJet);
bool FSpawnJet::Update()
{
    //creates a jet at position: (X = 0, Y = 0, Z = 0).
    GEditor->GetPIEWorldContext()->World()->SpawnActor(AJet::StaticClass());
    return true;
}
```

```cpp
DEFINE_LATENT_AUTOMATION_COMMAND_ONE_PARAMETER(FRetrieveAccelerateAndCheckJet, FAutomationTestBase*, test);
bool FRetrieveAccelerateAndCheckJet::Update()
{
    UWorld* testWorld = GEditor->GetPIEWorldContext()->World();
    AJet* testJet = Cast<anActorDerivedClass, AActor>(UGameplayStatics::GetActorOfClass(testWorld, AJet::StaticClass()));
    if(testJet)
    {
        float initialLocation = testJet->GetActorLocation().X;
        testJet->accelerate();
        float finalLocation = testJet->GetActorLocation().X;
        test->TestTrue(FString("Jet X location should increase when accelerated."), finalLocation > initialLocation);
        return true;
    }
    return false;
}
```

A parameter is added to the latent command so we’re able to call the test instance when the verification takes place.

But there’s still another error! When you load a map into the editor, it’s only loaded statically (no forces will be applied).

The solution is to tell the editor to start a PIE session, run the necessary steps to fulfill the test and then close the PIE session:

```cpp
bool FAJetMovesForwardWhenAcceleratedTest::RunTest(const FString& Parameters)
{
    ADD_LATENT_AUTOMATION_COMMAND(FEditorLoadMap(FString("/Game/Tests/TestMaps/VoidWorld")));
    ADD_LATENT_AUTOMATION_COMMAND(FStartPIECommand(true));
    ADD_LATENT_AUTOMATION_COMMAND(FSpawnJet);
    ADD_LATENT_AUTOMATION_COMMAND(FRetrieveAccelerateAndCheckJet(this));
    ADD_LATENT_AUTOMATION_COMMAND(FEndPlayMapCommand);
    return true;
}
```

FStartPIECommand y FEndPlayMapCommand are commands provided by Unreal Engine 4.

We also have to update the latent commands to wait for the editor to start the PIE session:

```cpp
bool FSpawnJet::Update()
{
    if (GEditor->IsPlayingSessionInEditor())
    {
        //creates a jet at position: (X = 0, Y = 0, Z = 0).
        GEditor->GetPIEWorldContext()->World()->SpawnActor(AJet::StaticClass());
        return true;
    }
    return false;
}

bool FRetrieveAccelerateAndCheckJet::Update()
{
    if (GEditor->IsPlayingSessionInEditor())
    {
        UWorld* testWorld = GEditor->GetPIEWorldContext()->World();
        AJet* testJet = Cast<anActorDerivedClass, AActor>(UGameplayStatics::GetActorOfClass(testWorld, AJet::StaticClass()));
        if(testJet)
        {
            float initialLocation = testJet->GetActorLocation().X;
            testJet->accelerate();
            float finalLocation = testJet->GetActorLocation().X;
            test->TestTrue(FString("Jet X location should increase when accelerated."), finalLocation > initialLocation);
            return true;
        }
    }
    return false;
}
```

## Ticks and Forces

Now we would expect that everything works as expected, but there’s a last erroneous assumption: When we accelerate an actor, we only tell the physics system to add a force to the actor’s body. This force will be effectively applied in the frame that follows the frame where the AddForce method is called.

Because of that, we should have to rewrite FRetrieveAccelerateAndCheckJet.

The test verification will also have to be rewritten because it’ll be made between frames. For this, it’s convenient to have a latent command with a parameter that will act as the last frame’s X value of the jet:

```cpp
DEFINE_LATENT_AUTOMATION_COMMAND_TWO_PARAMETER(FRetrieveAccelerateAndCheckJet, float, previousXValue, FAutomationTestBase*, test);

bool FRetrieveAccelerateAndCheckJet::Update()
{
    if (GEditor->IsPlayingSessionInEditor())
    {
        UWorld* testWorld = GEditor->GetPIEWorldContext()->World();
        AJet* testJet = Cast<AJet, AActor>(UGameplayStatics::GetActorOfClass(testWorld, AJet::StaticClass()));
        if(testJet)
        {
            testJet->accelerate();
            float initialLocation = previousXValue;
            float finalLocation = testJet->GetActorLocation().X;
            bool hasMovedForward = finalLocation > initialLocation;
            if(hasMovedForward)
            {
                test->TestTrue(FString("Jet X location should increase when accelerated."), hasMovedForward);
                return true;
            }
            previousXValue = finalLocation;
        }
    }
    return false;
}
```

We should initially call this command with the jet’s X value set as zero, because in another command we’re setting the jet’s initial position at (0, 0, 0) (this could be improved to specify the desired initial position of the jet in a variable to access it later by another command):

```cpp
bool FAJetMovesForwardWhenAcceleratedTest::RunTest(const FString& Parameters)
{
    ADD_LATENT_AUTOMATION_COMMAND(FEditorLoadMap(FString("/Game/Tests/TestMaps/VoidWorld")));
    ADD_LATENT_AUTOMATION_COMMAND(FStartPIECommand(true));
    ADD_LATENT_AUTOMATION_COMMAND(FSpawnJet);
    ADD_LATENT_AUTOMATION_COMMAND(FRetrieveAccelerateAndCheckJet(0, this));
    ADD_LATENT_AUTOMATION_COMMAND(FEndPlayMapCommand);
    return true;
}
```

And now we’re finally able to test that our jet accelerates when calling accelerate.

We could also accelerate once in the test if we create another latent command just for it.

## Final Note

In this case, this test runs until the jet moves. We could add a counter in the verification command to fail the test if a number of frames have passed, in a similar way as when checking the current position of the jet against the old one.

A better way would be to create a custom test class that handles the frame counting.

Examples of these two approaches can be found in these links:

* [Verification in latent command](https://github.com/Floating-Island/UE4-TDD-CI_Testing/blob/4288f88013ef8e8e6968ec38ea4b67287f085318/Source/CITesting/Tests/AcceleratingPawnTest.cpp) (from line 186 to 193 and 212 to 214).

* [Verification in test class](https://github.com/Floating-Island/ProjectR/tree/55a29e6809bb24e977782f783796165b45f6caf2/Source/TestingModule/Testing/TestBaseClasses).

## Conclusion

As we can see, some tests in Unreal Engine require preparation and understanding of the system. We can create derived test classes and other utilities to parametrize this preparation and reduce the duplication of our code.

We have to pay attention to asynchronous methods and that is because their presence makes us modify tests that otherwise would be very simple.

I hope this article has been helpful to you :).

Bye!

Alberto Mikulan