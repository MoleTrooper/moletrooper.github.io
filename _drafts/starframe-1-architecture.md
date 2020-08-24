---
layout: post
title: "Starframe dev blog: Architecture"
#date:   2018-09-23 16:10:00 +0300
categories: engine
---

I've spent a large portion of the past two years writing and rewriting
the basic structures that describe a game and its content in [Starframe] (formerly known as MoleEngine).
This post is about what I've tried so far and what those attempts taught me.

<!--excerpt-->

I'll start with some rambling about the nature of game engines and game objects
before diving into the Entity-Component-System architecture and my efforts at implementing it.
Finally, I'll cover my latest approach, which represents objects as a more general graph.

Note: many details here are specific to the Rust language, as are all the code examples,
but a lot of the information can be applied to any environment.
Feel free to skip around if you're not interested in implementation details.

# Intro: Engines and objects

When I started this project I had this vague idea that it should follow an architecture of some kind.
After some thought and experience I realized there was probably no good definition of what that even meant.
Let me take a moment to ponder about it here.

First of all, it's difficult to define exactly what a game engine even is. The best-known ones sold as products of their own,
like Unity, Unreal and Godot, are huge suites of reusable tools for every aspect of game production.
Everything from physics libraries to graphical editing tools is considered part of the package that is the engine.
On the other hand, many games have tools and background libraries built specifically for them.
In these cases the separation between "engine" and "gameplay" code can get very blurry.

In general, though, the tools provided by an engine tend to be fairly independent from each other, especially if they're meant to be used in multiple games.
A physics library doesn't need to know anything about graphics.
A level editor doesn't need to know anything about physics, nor does an input manager or a sound system, and so on.
As such, it doesn't make much sense to talk about the overall architecture of an engine – most of these pieces hardly need to fit together at all.
However, there is one point where almost everything is connected, and that is the **game object**, also known as the **entity**.
I'll speak about game objects here to separate this more general concept from the entities found in the Entity-Component-System architecture.

Game objects are another thing whose definition is quite hard to nail down.
If you'll forgive the hand-waving, they're _things_ that exist in a game world in a similar,
but not quite the same, way to real objects in the real world. Things with a distinct identity (in the metaphysical sense, not the mathematical).

In order to have any purpose or behavior, game objects need to have some data associated with them.
These associations can be implemented in many ways, and any tool or system
that touches game objects (i.e. most of them) needs to be able to follow them.
Well, either that or you need additional compatibility code converting things to whatever formats each system wants,
which is the only option if you want your systems to be usable outside of the engine they were made for.
A large part of the [Amethyst] game engine's job, for instance, is to glue various libraries
(many of them also developed by the Amethyst team, but intentionally made independent of the engine)
into their entity system of choice.

Anyway, this is where architecture suddenly becomes crucial in a fully integrated engine like mine –
almost every higher-level system depends on how you compose game objects.
So in a sense, you could call the architecture of game objects the fundamental architecture of a game engine.
This is why I've spent so much time thinking about this and doing it over and over.
I don't want to build too many systems on top of my object model until I'm happy with it,
because every time I redo it I have to update large parts of everything else.

# Building up

Before we get to the things I've been building, let's establish a starting point that we can compare to.
I've made a thing that's considerably complicated, and there are reasons why I want to have that thing,
but you can boil the same concept down to a very simple thing indeed.

We already have a type system provided by our programming language, why not just use it to represent our objects?

```rust
struct Goblin {
  position: Vector2<f32>,
  attack_power: u32,
  health: u32,
  catchphrase: String,
  has_a_gun: bool,
}
```

In languages with support for sum types (a.k.a. tagged unions or fat enums)
you can even build a single all-encompassing object type (or multiple some-encompassing ones)
that can be queried for components in a similar fashion to the more complicated systems we're about to discuss.

```rust
enum GameObject {
  Player(Player),
  Goblin(Goblin),
  Projectile(Projectile),
  Decoration(Decoration),
  // ...
}
impl GameObject {
  pub fn get_rigid_body(&self) -> Option<&RigidBody> {
    match self {
      GameObject::Projectile(p) => Some(p.body),
      GameObject::Decoration(_) => None,
      // ... you get the idea
    }
  }
  pub fn get_position(&self) -> Option<&Vector2<f32>> { /* ... */ }
  pub fn get_sprite(&self) -> Option<&Sprite> { /* ... */ }
}
```

<small>(In real life I would probably implement these getters as traits
and use them to make my code generic to the concrete object type, but that's not relevant to the example)
</small>

There's a good chance something like this is already flexible enough to support your entire game without much trouble.
Check out [this post by Mason Remaley][way-of-rhea] for a more detailed description of such a system, as implemented in the game Way of Rhea.
This approach is dead simple and has some genuine advantages over other systems (simplicity in itself is a big one!),
but also some notable inefficiencies. We'll talk about them later when comparing back to this.

# Attempt 1: Generic ECS

Entity-Component-System, or ECS for short, seems (to my limited perspective)
to be a de-facto standard in most of the game dev world at the moment.
There are various Rust crates implementing it, such as [specs], [legion], and [hecs].
At the time I started, `specs` was the big ECS crate everyone was using, I saw [this popular talk by Catherine West][west-talk],
and was naturally led to look into the `specs` source as my primary reference.

**So what is it?**

ECS takes the struct from our initial example, gives it an identifier, rips all its fields out and distributes them
into what are essentially database tables where they can be looked up with the identifier.
The identifier is called the Entity, and the fields are Components.
Game logic can select entities that have a certain set of components and ignore the rest.
Functions that do this are called Systems.

This improves performance by grouping data in memory more efficiently than those big everything-structs from earlier.
To put it briefly, memory lookups are extremely slow, and processors do best when they get to go from one memory location
to the one right next to it and repeat. This optimization is accomplished using _caches_ of very fast memory close to the processor.
Most things in the big structs are irrelevant to most operations, but they still take up space in the cache all the time.
On the other hand, ECS "tables" only have one type of thing each, so we can pick and choose what to pull into the cache
by completely ignoring the tables we don't need.
This is quite important in rigid-body physics, where the number of objects tends to be very large and the only parts of them that matter
are bodies, colliders, positions, orientations, and velocities.

The other benefits of this pattern stem from its nature as a dynamic type system, as do many of its flaws.
When using structs we were defining types in Rust's static type system,
which lets us use the compiler to verify that our objects are well-formed,
but requires a programmer to write every object type the game supports.
On the other hand, in ECS, all our types are defined at runtime by inserting components into tables.
No programming is needed to create new types of object or to wire them into the game logic,
but we get no static analysis to help catch errors.
An interesting consequence is that an object can change its type at runtime —
a classic example is an RPG character getting status effects added and removed dynamically.

**My implementation**

At this point I was still wrapping my head around what an ECS even is, and I started by more or less just copying what `specs` did.
I had a top-level manager called `Space` that gave out entity IDs, kept track of alive entities with a `BitSet` from the `hibitset` crate,
and stored all the component tables in an `AnyMap` (a container that stores `Any` pointers indexed by the type,
essentially a `HashMap<TypeId, Box<Any>>`) from the `anymap` crate.

Tables were inside `RwLock`s to allow accessing many in parallel.
They also had their own `BitSet`s to keep track of entities using them,
and queries used binary operations on these to find the entities that had the components they were looking for.

I had different types of `Storage` for my component tables that would be selected by the user based on how common the component was —
`VecStorage` when almost every object has it, `DenseVecStorage` when it's common but not ubiquitous, `HashMapStorage` when it's rare,
and `NullStorage` when it's a zero-sized tag component.
Object IDs contained a generation index that allowed them to be deleted and overwritten with new things
in a way that invalidated any old components still hanging around.

Where things diverge a little from `specs` is in the queries that Systems use to select their desired component sets.
This was in some part because I didn't really care for the tuple-based interface in `specs`, but mostly because I didn't fully understand how it worked.
I wrote a rather long procedural macro that generated query implementations for structs of references (plus some special fields with special annotations)
like this one from my physics module:

```rust
// module paths modified for clarity, `sf` is `starframe`
#[derive(ComponentQuery)]
pub struct RigidBodyQuery<'a> {
    #[id]
    id: sf::ecs::IdType,
    tr: &'a mut sf::util::Transform,
    body: &'a mut sf::physics::RigidBody,
}
```

Running this query through a `Space` would return a slice `&mut [RigidBodyQuery]` where each item in the slice represented one entity.
I still think this is a neat interface, but the macro ended up so large it was quite hard to maintain.

Additional concepts I added from outside the realm of `specs` were `Recipe`s and `Pool`s.

A `Recipe` is very similar to the idea of a _prefab_ in most game engines.
It's a simple struct that knows how to turn itself into an entity, and with some macros around it,
also knows how to read itself from a file with `serde`.
[Here's what their definitions looked like.][ecs-impl-1-recipes]

<small>If you're wondering about the `recipes!` macro (which I don't use anymore because it made people wonder),
it simply creates an enum with the given types as its variants, plus a bit more magic that was later deemed unnecessary.</small>

With these I was able to read scenes from RON files like this:

```ron
[
    Player ((
        transform: ( position: (0, 0) ),
    )),
    DynamicBlock ((
        width: 1, height: 0.8, transform: ( position: (1, -1.5) ),
    )),
    StaticBlock ((
        width: 8, height: 0.2, transform: ( position: (0, -3) ),
    )),
]
```

I still use more or less the same recipe system today.
It's a concise but human-readable way to express scenes and helps a lot when a text file is the only level editor I have.

A `Pool` is an optimization for types of objects that are frequently spawned and deleted, such as bullets or other projectiles in an action game.
What it does is preallocate and take control of entity slots so that entities it manages are always placed in those slots.
The benefit of this is twofold — firstly, it prevents memory fragmentation from objects getting deleted and respawned far away in memory.
Secondly, it lets you limit the number of instances of an object type that can exist at one time,
preventing them from taking up more memory than your budget allows.
This was especially important in this implementation because my `Space`s had a maximum capacity and weren't growable at runtime.

The last revision of source code with this implementation can be found [here][ecs-impl-1].

# Attempt 2: Decentralized ECS

I kept most of the first implementation for this one. `Space`s, `Storage`s, `Pool`s, the whole gang was still around,
but the way I stored and queried my component tables was very different.

`AnyMap`s are cool and all, but they limit the compile-time checks you can do for things you put inside them.
What if I let the user define their own structs with the tables they want?
This way we could make better use of Rust's type system and, more importantly, borrow checker to hand out references instead
of locking things at runtime.
However, the central `Space` was still necessary, and if possible, I'd still like to force every table to be connected to it somehow.

What I came up with was to let the user put anything they wanted in one field of `Space`,
but make querying for components only available through an interface of `Space`.
While still technically possible to break things by mixing tables from different Spaces,
you'd need some pretty acrobatic moves to do it.
Here's what the types looked like from the perspective of a game:

```rust
pub type MainSpace = sf::core::Space<MainSpaceFeatures>;

pub struct MainSpaceFeatures {
    pub transform: sf::core::TransformFeature,
    pub shape: sf::graphics::ShapeFeature,
    pub physics: sf::physics::PhysicsFeature,
    pub player: player::PlayerController,
    pub camera: sf::graphics::camera::MouseDragCamera,
}
```

All these types ending in `Feature` contain component tables.
Many of them also have game logic, so you could call them Systems in the ECS sense while also being parts of the data structure itself.
This is why I'm calling this thing "decentralized" — the data structure is scattered across a variety of types.

A nice thing about the resulting interfaces is that they explicitly tell you what other Features/tables they depend on,
so you must have everything around to be able to call them at all (as opposed to earlier where systems
looked things up on their own using the macro-generated queries).
Running many of these in parallel would also be easy, safe and lock-free with Rust borrow-checking the references,
although I never actually did this.

```rust
// simplified from examples/testgame/main.rs
space.tick(|features, iter_seed, cmd_queue| {
    features.player.tick(
        iter_seed,
        &game.input,
        &mut features.transform,
        &mut features.physics,
        cmd_queue
    );

    let grav = forcefield::Gravity(Vec2::new(0.0, -9.81));
    features.physics.tick(
        iter_seed,
        &mut features.transform,
        dt,
        Some(&grav),
    );
});
```

You may notice the `iter_seed` here, which leads us to the new query system.
As mentioned, the macro-generated queries were gone; they couldn't exist without the ability to look up a table by its type.
I needed something a little less automagical, which I wanted for maintainability's sake anyway.

After some wrestling with generics and closures, I managed to produce a set of iterator combinators that looked like this:

```rust
// simplified from src/physics/mod.rs
let mut iter = iter_seed
  .overlay(self.bodies.iter_mut())
  .and(self.colliders.iter())
  .and(transforms.iter_mut())
  .with_ids()
  .into_iter();

for (((body, coll), tr), id) in iter {
  // do stuff...
}
```

The `IterSeed` is handed out by a `Space`, and produces a `()` for every entity that is alive.
You have to start with this. From there on you get various binary operations to add parts to the query,
filtering out entities that don't fit the mold.
`and` is by far the most useful one, but things like `not` and `or` are also possible.
For those interested, the code for these is [here][ecs-impl-2-iters].

Aside from the ugly nested tuples, which are fairly easily dealt with by one pattern match,
I was really happy with the expressiveness and lack of macros in this.
There was some awkwardness in the `Space` API, but overall I felt like this was an improvement over the original.

The last revision of source code with this implementation can be found [here][ecs-impl-2].

# Attempt 3: Graph

oien

Just for fun, let's look at what the previous architectures would look like if interpreted as a graph.

First, the big structs. The smallest unit we can operate on in this model is a whole object.
We can look at different parts inside them to produce different behaviors, but we can't pull those parts apart and put them in different places.
Therefore, the 'atomic' unit of this world and the node type of this graph would be the whole object.
Representing different behaviors with different colors, this graph would look something like this:

(graph here)

The nodes are entirely self-contained and don't need any edges to connect them to anything.

<!-- links -->

[starframe]: https://github.com/MoleTrooper/starframe
[amethyst]: https://amethyst.rs/
[way-of-rhea]: https://www.anthropicstudios.com/2019/06/05/entity-systems/
[specs]: https://github.com/amethyst/specs
[legion]: https://github.com/TomGillen/legion
[hecs]: https://github.com/Ralith/hecs
[west-talk]: https://www.youtube.com/watch?v=aKLntZcp27M
[ecs-impl-1]: https://github.com/MoleTrooper/starframe/tree/cec0dbec5bce8612ffb9dd82441e30eb9233ef60
[ecs-impl-1-recipes]: https://github.com/MoleTrooper/starframe/blob/cec0dbec5bce8612ffb9dd82441e30eb9233ef60/examples/testgame/recipes.rs
[ecs-impl-2]: https://github.com/MoleTrooper/starframe/tree/1518f8ee65b33427f580537639293360e7b9db35
[ecs-impl-2-iters]: https://github.com/MoleTrooper/starframe/blob/1518f8ee65b33427f580537639293360e7b9db35/src/core/container.rs#L100
