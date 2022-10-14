WIP

## What this tutorial is about

Unreal Engine documentation is often incomplete.
With regards to multiplayer, this was/is especially annoying.
There is information on the power of [Actor Replication](https://docs.unrealengine.com/4.27/en-US/InteractiveExperiences/Networking/Actors/),
great, and I can learn about the role of different classes of [Unreal's Gameplay Framework](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Framework/),
but try to get multiplayer going for a fresh C++ project and you are up for a rough start.

The setup that I am going to present here will have you prepared for any of those:

* Singleplayer
* Split or shared screen
* LAN multiplayer
* Online multiplayer

The Generic LAN interface is important for me, as it is supposed to be an easy setup.
Also, I like the idea of offering multiplayer without any account registration required.
This is controversial though: It plays into the hands of software piracy, too.

For online multiplayer to work, I rely on the [EOS Online subsystem plugin](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Online/EOS/),
i.e. the online subsystem for the [Epic Online Services](https://dev.epicgames.com/en-US/home).

## Specifically, this tutorial will have answers for the following questions

(I might be wrong, but to me these questions are essential and online search results left me in the dark)

* What is the unique net id? Where do I get it? Where do I store it?
* How to have more than one local player and should I bother?
* What is `ULocalPlayer` good for and how does it relate to `GameInstance` and the usual suspects `PlayerState` and `PlayerController`?

And this [Local Player Context](https://docs.unrealengine.com/4.27/en-US/API/Runtime/Engine/Engine/FLocalPlayerContext/) looks useful.
Let's actually go ahead and use it.
