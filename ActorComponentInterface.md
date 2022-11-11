# Use C++-interfaces for more flexible handling of components

## Establishing basic knowledge on Actors and Components first

Whenever you put a thing into your Unreal game world, you usually place an actor.
I.e. actors are quite intuitively the basic elements of your world.
There is not much confusion there.

When creating something new, you can usually create a new C++ class that inherits from `AActor`.
When your new creation can be possessed by a player, you typically inherit from `APawn` (which inherits from `AActor`).

E.g. my game plays in space where stuff is always orbiting some gravitational body in the center of the map.
I can go ahead and implement orbital mechanics (movement along a spline that represents a Kepler orbit) in some Actor.
However, I realize that I have Actors and Pawns that both have 


When in doubt, use a component.

Components are 
