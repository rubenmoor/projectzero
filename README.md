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
* [Draw a circle in NativePaint](DrawCircle.md)
