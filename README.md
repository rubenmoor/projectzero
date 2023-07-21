# Tutorials for Unreal Engine/C++

* [The most basic, yet complete setup for multiplayer](https://github.com/rubenmoor/TutorialMPBasics) Getting your project setup
  right, to eventually implement single player, split/shared screen, LAN and online multiplayer - no need
  to pay for a plugin (the link leads to a different repo, which contains a whole sample project)
* WIP [Simulation of Kepler orbits for planets and alike](KeplerOrbits.md) Orbits are determined analytically based on the initial
  conditions, i.e. distance from center and speed, and implemented as splines along which the object then moves
* WIP [Physical gyration of a static mesh with torque-free precession](PhysicalGyration.md) Implementing
  continuous rotation for a mesh without relying on Unreal's  `simulate physics`, featuring [torque-free
  precession (Wikipedia)](https://en.wikipedia.org/wiki/Precession) for free
  
## Recipes

* [Reacting to property changes in the editor](PostEditChangeProperty.md)
* [Accessing a property value](AccessPropertyValue.md)
* [Tracking a world location via indicator in the HUD](TrackingIndicator.md)
* [Draw a circle in NativePaint](MakeCircle.md)
* [Use non-type template parameters when binding actions to functions](BindInputToTemplatedFunction.md)
  * or instead: [Bind a lambda expression to the Input Component](InputBindLambda.md)
* [Add flexibility to your components with C++ interfaces](ActorComponentInterface.md)
* [Heavy construction done right: How to spawn actors from inside OnConstruction and what to do about Components](HeavyConstruction.md)
* [Example code for UMG widget ComboBoxKey](ComboBoxKey.md)
* [The proper way to create and use Native Gameplay Tags](NativeGameplayTags.md)
