# Lightning attack library for GZDoom

This is a ZScript library that offers a system for lightning attacks which can be used in weapons.

### Features

* A function that attachs a controller to a specific actor (meant to be the target of an attack)

* The initial victim can create extra arcs that link to other valid victims around it

* Customizable number of arcs (splits)

* Customizable number of total links (how many times it can link from one victim to the next)

* Customizable damage per tic

* Visuals: a function that drawns a particle-based lightning between two points (can be replaced with entirely different visuals, if desired)

* The same victims can't be hit multiple times from the same source, meaning the attack will actually spread through crows instead of looping back to random victims

* When splitting, lightning arcs will always hit pick the nearest valid victims first

### License

MIT. Do whatever you want with it, just keep credit and copy my license text to your project. See [LICENSE.txt](https://github.com/jekyllgrim/ZSLightningGun/blob/main/LICENSE.txt).

## Classes

`ArcSplitController` is the sole class that controls the behavior. It's an `Inventory`-based class that is placed in the victim's inventory and handles searching and arcing behavior. It features a number of static functions.

## Functions

### StartChain

```csharp
static ArcSplitController StartChain(Actor damageSource, Actor victim, int damage, double range, int duration = 1, int delay = 0, int maxSplits = 1, int maxLinks = 0, ArcSplitController parent = null)
```

Static function that begins the attack. Call from anywhere as `ArcSplitController.StartChain`.

##### Parameters

* `damageSource`
  
  Pointer to the actor that should be responsible for the damage dealt by this cycle of lightning. If fired from a monster or a weapon function, pass `self` here so it sets the player as the source of damage.

* `Actor victim`
  
  Pointer to the first victim to be hit. If you're going to use this function from a weapon, you will need to obtain a pointer to the victim first, for example, using [LineTrace](https://zdoom.org/wiki/LineTrace). You can see an example in the `LightningGun` class attached as the example.

* `damage`
  
  Integer number. Amount of damage that the lightning will deal to the monster it hits per tic.

* `range`
  
  Float-point number. The maximum range of the lightning. Further lightning arcs created from the first victim and subsequent victims will only extend to this range. If a monster is being hit by the lightning does not have any valid targets within this range, the lightning will not extend.

* `duration`
  
  Integer number (default: 1). The duration of the lightning effect per tic. If you're designing a lightning attack that is supposed to be sustained (like Quake's Lightning Gun), then leave this at 1. If you're making a lightning effect that is supposed to linger, you can use a higher value. In this case, the lightning will stick to the victim for the specified number of tics, damaging them in a loop.

* `delay`
  
  Integer number (default: 0). If positive, the lightning will not split to further victims immediately but will instead wait for a while, damaging the initial victim, and only then split.

* `maxSplits`
  
  Integer number (default: 1). How many extra arcs can be created from a single victim at once. At 1 the lightning will split only once, to the nearest victim. At higher values, multiple links can be created (provided there are enough valid victims within the valid range).

* `maxLinks`
  
  Integer number (default: 0). How "deep" the lightning goes, i.e. how many times one arc can split from one victim to the next. 
  
  Note, this is individual for every split, so, for example, if `maxsplits` is 1 and `maxLinks` is 5, it means the lightning will jump once from each victim to the next, up to 5 times. If `maxSplits` is, say, 5, that means that 5 arcs will split from the first victim to 5 valid victims around, then from *each* of those victims 5 more will be created, and so up to however many times you specified with `maxLinks`. 

* `parent`
  
  You don't need to set it directly. This is only set internally by `ExtendChain()`, which is a separate function that creates a new arc from one victim to the next.

### DrawLightning

```csharp
static void DrawLightning(Vector3 from, Vector3 to, bool spawnSpark = true, PlayerInfo playersource = null)
```

This draws a lightning between two coordinates, `from` and `to`. Doesn't deal any damage, this is visual only. You can call it manually (for example, from an attack function), but lightning controllers will also call this automatically whenever lightning is split and hits another victim.

##### Parameters

* `from`
  
  Vector3 coordinates for the start point.

* `to`
  
  Vector3 coordinates of the end point.

* `spawnSpark`
  
  Boolean (default: true). If true, some sparks will be spawned at the end position.

* `playersource`
  
  A `PlayerInfo` pointer (default: null). If you're drawing a lightning from the player, pass its `player` field here. If this is done, the particles of the lightning will receive velocity from the player, which makes them less jittery.

This function can be completely replaced with your own custom behavior if you want different visuals. You could draw a scalable 3D model or anything else you like.

```csharp
static void DrawLightningSegment(Vector3 from, Vector3 to, double density = 8, double size = 10, double posOfs = 2, bool spawnSpark = true, PlayerInfo playerSource = null)
```

This is used by `DrawLightning` internally: `DrawLightningSegment` does exactly what it says on the tin—it drawns a straight line of particles between 2 coordinates. `DrawLightning` calls this function multiple times in order to draw segments of the lightning between multiple coordinates along a vector, to give it a zig-zagged look.

##### Parameters

- `from`
  
  Vector3 coordinates for the start point.

- `to`
  
  Vector3 coordinates of the end point.

- `density`
  
  Float-point (default: 8). Per how many map units to spawn a particle.

- `size`
  
  Float-point (default: 10). The size of each individual particle in map units.

- `posOfs`
  
  Float-point (default: 2, but `DrawLightning` sets this to 0). If positive, each particle's position will be randomized around its initial calculated position, giving the beam a more noisy look.

- `spawnSpark`
  
  Boolean (default: true). If true, some sparks will be spawned at the end position. When called from `DrawLightning`, this will only be set to true if it was set to true in `DrawLightning` *and* only when `to` matches the final position in `DrawLightning` (i.e. sparks are not created for each segment).

- `playersource`
  
  A `PlayerInfo` pointer (default: null). If you're drawing a lightning from the player, pass its `player` field here. If this is done, the particles of the lightning will receive velocity from the player, which makes them less jittery. `DrawLightning` automatically passes its `playerSource` pointer here.

### GetBeamAttachPos

```csharp
static Vector3 GetBeamAttachPos(Actor source, double horOfs = 0, double vertOfs = 0)
```

This function takes an actor pointer `source` and obtains the position to which the lightning beam must be attached.

For monsters it simply returns their exact midpoint (half of their height).

For players, it offsets relative to the camera and takes parameters into account.

##### Parameters

* `horOfs`
  
  Float-point (default: 0). Only works if `source` is a PlayerPawn. How much to offset the origin of the lightning horizontally relative to the player's screen. (Positive is right, negative is left). 0 means horizontal center of the screen.

* `vertOfs`
  
  Float-point (default: 0). Only works if `source` is a PlayerPawn. How much to offset the origin of the lightning vertically relative to the player's screen. (Positive is up, negative is down). 0 means vertical center of the screen.

Note, this is a purely visual function and it has no bearing on the lightning's accuracy.

When extending lightning arcs between monsters, this function will be used by the controller automatically.

### GetAttackHeight

```csharp
static double GetAttackHeight(PlayerPawn source)
```

Gets the exact point relative to the player's vertical position from which its attacks should come out. Note, this is *not* for visual effects, this is for the actual attacks (like if you're using `LineTrace` to find a valid first victim). Pass this to `LineTrace`'s `offsetz` argument to make the trace hit exactly where a hitscan attack would.

## Example weapon

The library comes with an example weapon, `LightningGun`, which is pased on the Plasma Rifle and uses its sprites (not included in the library; they will be picked from doom.wad/doom2.wad if you run it with Doom). This weapon features an example of how you could use the above library in a weapon, with this function:

```csharp
action void A_FireLightningGun(int damage, bool useammo = true, double horOfs = 0, double spawnheight = 0, double range = 512, int duration = 1, int delay = 0, int maxsplits = 1, int maxlinks = 5)
```

This is an example attack function that lets you create attacks similar to `A_FireBullets`.

##### Parameters

* `damage`
  
  Integer. Damage the lightning will deal per tic to its victim.

* `useammo`
  
  Boolean. If true, the function will consume ammo. Default: `true`.

* `horOfs`
  
  Float-point. Horizontal offset of the lightning's origin point relative to the horizontal center of the player's screen. Default: 0. Positive numbers will shift it to the right.

* `spawnheight`
  
  Float-point. Vertical offset relative of the lightning's origin point relative to the vertical center of the player's screen. Default: 0 (which means it spawns at the center). Negative numbers will shift it lower.

* `range`
  
  Float-point. The maximum range of the lightning. Default: 512.

* `duration `
  
  Integer. The duration in tics of how long the lightning will linger on the victim. Gets passed to [`ArcSplitController.StartChain()`](#StartChain).

* `delay`
  
  Integer. How many tics the lightning will linger on the victim before splitting. Gets passed to [`ArcSplitController.StartChain()`](#StartChain).

* `maxsplits`
  
  Integer. How many new arcs can be created from each victim at once. Gets passed to [`ArcSplitController.StartChain()`](#StartChain).

* `maxlinks`
  
  Integer. How many times each arc can split to the next victim. Gets passed to [`ArcSplitController.StartChain()`](#StartChain).
