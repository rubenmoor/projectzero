# Kepler Orbits

How to implement a simplified solution of Newtonian celestial mechanics, i.e. the Kepler orbits in Unreal Engine/C++

## What are Kepler Orbits?

Newton discovered that one and the same force gives rise to too seemingly different phenomena:
Things falling on the ground on earth *and* planetary motion.
Einstein later swapped out Newton's gravitation with his theory of general relativity, thus technically the Newtonian motion isn't accurate.
Even more technical, Einstein's general relativity can't be quite right, either:
It is irreconilable with Quantum Mechanics and it doesn't account for what is called [Dark Matter](https://en.wikipedia.org/wiki/Dark_matter).
We don't care: as we will see, Newtonian physics are good enough for our purpose.
... and we are still free to add relativistic effects on top, whenever we want.

If, and only if, we allow us to reduce the planetary motion to a two-body problem, Newton's insights provide us with a general, analytical solution.
This means, there exist formulas that, together, are a complete recipe to model planetary motion in a game engine.
And it goes like this:

* Pick an initial mass for the central body, usually that would be the sun.
* Pick an initial mass for the planet; only one planet is allowed at a time.
* Put the sun and the planet somewhere in 3D space, Unreal engine (UE) users know what to do.
* Choose an initial velocity for your planet. In theory, the sun has an initial velocity, too, but you will see.
* Use the formulae from the [Wikipedia article on elliptic orbits](https://en.wikipedia.org/wiki/Elliptic_orbit) to calculate the *orbital parameters*.
* In the UE editor, create a spline that matches the calculated orbit.
* Make the planet advance along the spline; respecting its velocity, for which there is a different formula.

The splines that you create this way, are called the "Kepler orbits", because Kepler discovered them before Newton invented physics.
You might think of the Kepler orbits as ellipses, which isn't wrong, however:

First, Kepler orbits include several cases, of which wikipedia mentions only a couple:

* A closed line
* An infinite line
* A parabola
* A circle
* An ellipse
* A hyperbola

All of these case are physically possible.
E.g. the [object that passed through our solar system](https://en.wikipedia.org/wiki/%CA%BBOumuamua), followed a hyperbola.
For actual planets, only the ellipse-case is interesting.
The linear orbits (closed and infinite line) imply an object passing right through the sun ignoring collision.
And while this isn't strictly physical, we will later see that these cases need to be accounted for, too.

## The limits of this approach

There are a couple of things we rushed through.
Let's clarify now.

### Maybe you have to translate from UE world space to "math space"

First, in order to solve the orbital equations from the Wikipedia article, we will move stuff around.
The recipe in Wikipedia requires the center-of-mass be at the origin, i.e. at `(0, 0, 0)`.
Furthermore, all the movement is assumed to be in the X-Y-plane.
This simplifies the equations quite a bit, but note:
If you place your sun and planet in an actual 3D world you will have to make a decision:
Either you follow the recipe literally, *or* you translate your coordinates from UE world space to math space before you plug them into any equation from Wikipedia.

### Having more than one planet introduces errors

"Only one planet allowed at a time" means:
The Kepler orbit is accurate as long as you describe a two-body system.
There's no problem repeating the recipe for a second planet, however:
The planets will be ignorant towards their respective presence.
In reality, planet Jupiter is so heavy that it has a measurable impact on the orbits of any other planet (or asteroid, or Oumuamua).

You gotta stay strong though:
Once you accurately model the gravitational pull between planets, you are not doing Kepler orbits anymore.
Worse: planetary motion realistic to that degree doesn't generally result in any sort of periodic orbit.
Forecasting the motion of actual planets is only accurate for 60 Million years, both forwards and backwards in time.
Before or after that, we can't tell where the planets are, because at its core, their movement is chaotic.

For what I want to build, the "measurable impact" of another planet is still something I don't care about.
I favor the elegant math of Kepler.

### Optionally: Define the sun as center of mass

According to the recipe, the center of mass is at the origin.
We can simplify our job even further by treating the position of the sun as the origin, too.
In that case, the sun doesn't move.
If you want more accuracy, you can model the orbit of the sun too:
Yes, it really orbits around the center of mass of our solar system.
The real position of this center of mass again depends on the position of all the planets, but it's expected to be well within the confines of the sun itself ...
maybe you understand why I didn't bother.

This simplification is justified whenever your central body is way more massive than the other body.
And along with this comes another simplification which is called the "reduced mass" α = G (m1 + m2) ≈ Gm1, which allows to ignore the mass of any planet entirely for now.

### The perihelion precession of Mercury isn't included

Einstein's general relativity doesn't have a lot of impact on our everyday lives (contrary to his theory of special relativity).
A notable exception is the [perihelion precession of Mercury](https://en.wikipedia.org/wiki/Tests_of_general_relativity#Perihelion_precession_of_Mercury), which is caused by both, other planets and relativistic effects.
"Perihelion precession" can be thought of like this:
In an elliptic orbit, there is the point closest to the central body.
This is the perihelion.
Perihelion precession means that every time Mercury finishes an orbit, its perihelion has moved a bit.
Expert readers will note right away that this isn't a Kepler orbit anymore and thus not part of our way of simulating planets. 


## An alternative approach, both simpler and arguably more powerful: Numerical Newtonian orbits
