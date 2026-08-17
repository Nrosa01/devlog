---
title: Creating a 2D Collision System - Part 1
tags: ["c#", "physics", "monogame"]
draft: false
date: 2026-08-13
---

# Motivation

Box2D is awesome and you can perfectly do 2D games with it, in fact, you might want to do that. But it's a complex library that solves many issues I don't have, which in turns, creates other ones. Also I just wanted to try doing my own.

# Requeriments

There is an existing implementation for C# [here](https://github.com/dotnet-ad/Humper/), and that was is inspired this [Love2D](https://github.com/kikito/bump.lua) one. Both are good and they inspired mine, but I had some issues with them:ç

- Bump.lua is written in Lua
- Humper wraps everything into references that you can't know if they're valid or not

I want my implementation to be safe to use, performant and flexible enough to cover most of my usecases:
- Support for implementing Triggers
- Layermasks
- Ray and Cast queries against the World
- Immediate mode API

By immediate mode API, I mean that you query the world on demand, there is no a simulation step. You move your body then you check if there were any collisions. My current approach it's not thread safe, but I intend to avoid the need for multithreading in my framework. Most 2D games don't require it and the complexity it's not worth for me.

# Implementation

## World and Body

My implementation is inspired by the [handle based map](https://zylinski.se/posts/handle-based-arrays/) post of Karl Zylinski. I recommend reading that to get a better idea of why I'm going this path.

Before going to the collision code itself, I will first show the high level structures and usage.

I have a `World<T>` that stores objects of type `Body<T>`. The type is this:

```csharp
/// <summary>
/// Item of <see cref="World{T}"/>
/// </summary>
/// <typeparam name="T">The type of the custom <see cref="UserData"/>. If none is wanted use an empty struct</typeparam>
public struct Body<T>
{
    /// <summary>
    /// Actual AABB collider, readonly. To modify use <see cref="World{T}"/> methods
    /// </summary>
    public RectangleF Bounds { get; internal set; } // This must not be directly modified EVER
    
    /// <summary>
    /// Layer this Body is in
    /// </summary>
    public uint Layer;

    /// <summary>
    /// Layer this Body scans for
    /// </summary>
    public uint Mask;

    /// <summary>
    /// Optional userData for more personalized behaviour
    /// </summary>
    public T UserData;
}
```

This is the physics item I use, just the bounds, layermask and the `userdata` so I can store handles to my entities.
What I just told you is kind of a lie, what is stored really is a `HandleItem<T>`, but that's not exposed to the user. The type is this

```csharp
/// <summary>
/// Agrupation of an item and its handle that represents it in a container
/// </summary>
/// <typeparam name="T"></typeparam>
public struct HandleItem<T> : IHandle<T> where T : new()
{
    public readonly static HandleItem<T> Default = new() { Handle = Handle<T>.Default, Item = default };

    public Handle<T> Handle { get; set; }

    public T Item;
}
```

You will understand why I need to store both together later.

What the API exposes is the `Handle<T>`:

```csharp
public readonly struct Handle<T> : IEquatable<Handle<T>>
{
    public int Id { get; init; }
    public int Generation { get; init; }

    public readonly static Handle<T> Default = new() { Id = 0, Generation = 0 };
}
```

This is what the user actually sees. I like approaching my engine as if I were writing a public library but allowing myself some leniency.

## Basic usage example

I will stop here for a moment and show you how I'm currently using this library.

> [!Warning] Warning!
> This is sample of real code from my project, but I omitted many parts to keep it simply, this won't just run on its own

```csharp
    enum ItemType
    {
        Player,
        Wall1,
        Wall2
    };

    Vector2 position = new Vector2(-200, 0);
    Handle<Body<ItemType>> playerHandle;
    Darkrit.Physics.Boxy2D.World<ItemType> world;

    public override void Initialize()
    {
        world = new();
        world.Create(new Vector2(0, 0), new Vector2(600, 20), (uint)CollisionLayer.World, userData: ItemType.Wall1);
        world.Create(new Vector2(0, 0), new Vector2(30, 200), (uint)CollisionLayer.World, userData: ItemType.Wall2);

        playerHandle = world.Create(position, slimeAnimation.Size, (uint)CollisionLayer.None, (uint)CollisionLayer.World, userData: ItemType.Player);
    }

    public override void FixedUpdate()
    {
        Vector2 goalPosition = position + velocity.Normalized * speed * (float)gameTime.ElapsedGameTime.TotalSeconds;

        var hasCollision = world.Move(playerHandle, goalPosition, CollisionFilters<ItemType>.Slide);
        
        // Sync with physics
        position = world.Get(playerHandle).Bounds.Location;
        
        if (hasCollision)
        {
            var latestCollisions = world.LastCollsions;
            foreach (var item in latestCollisions)
            {
                // UserData is the ItemType enum
                Log.Info($"Collision with {world.Get(item.Handle).UserData}");
            }
        }
    }
```

You can see the result here:

![](Darkrit/assets/14-08-2026%20AABB%20Collisions%201.mp4)


As you can see, right now my usage of the API is pretty barebones as I have to manually sync the logical position with the physics one. I don't have yet an entity system. But as you can see, the user facing API is not that hard. Just that instead of having a reference, you have a Handle that has to go through the world to be resolved.

## Collision Responses

One thing I wanted is the ability to have triggers. But I didn't want to harcode them into the collision system. The system doesn't know about triggers, it just needs the least amount of data to handle collisions, that's all it does. 

Now I can finally show the most important piece of code of `World`

```csharp
    /// <summary>
    /// Attemps to move the body associated with <paramref name="handle"/> to <paramref name="targetPosition"/>
    /// </summary>
    /// <param name="handle">Handle to the item to move</param>
    /// <param name="targetPosition">Desired position the body at <paramref name="handle"/> should move</param>
    /// <param name="collisionFilter">Filter function that decides how each collision is handled</param>
    /// <param name="maxCollisions">Some <see cref="CollisionResponseFunction"/> need many iterations to be solved.
    /// This parameter limits the amount of iterations that can be done</param>
    /// <param name="testOnly">If <paramref name="testOnly"/> is true, the body does not move but the would-be collision information is given.</param>
    /// <returns>True if there was a collision</returns>
    public bool Move(Handle<Body<T>> handle, Vector2 targetPosition, CollisionFilterFunction<T> collisionFilter, int maxCollisions = 5, bool testOnly = false)
```

This is a big chunky function where the magic happens. But before going into that, I think it's more interesting to check what happens when a collision happens.

```csharp
public delegate void CollisionResponseFunction(ref RectangleF body, ref Vector2 velocity, CollisionInfo collisionResponse);

public delegate CollisionResponseFunction CollisionFilterFunction<T>(ref Body<T> self, ref Body<T> other);
```

These two delegates are the most important part. The first one answer the questions "How do I resolve a collision"? By this point, the collision has happened, it passed the layer masks and filters and something has to be done. I provide a couple of built-in responses:

```csharp
    public static void Slide(ref RectangleF body, ref Vector2 velocity, CollisionInfo response)
    {
        body.X += velocity.X * response.CollisionTime;
        body.Y += velocity.Y * response.CollisionTime;

        velocity *= response.RemaininTime;

        float normalVelocity = Vector2.Dot(velocity, response.Normal);

        if (normalVelocity < 0.0f)
            velocity -= response.Normal * normalVelocity;
    }

    public static void Stop(ref RectangleF r1, ref Vector2 velocity, CollisionInfo collisionResponde)
    {
        r1.X += velocity.X * collisionResponde.CollisionTime;
        r1.Y += velocity.Y * collisionResponde.CollisionTime;

        velocity = Vector2.Zero;
    }

    public static void Cross(ref RectangleF r1, ref Vector2 velocity, CollisionInfo collisionResponde)
    {
        velocity = Vector2.Zero;
    }
```

Stops is the default behaviour. When there is a collision, it puts the object at the border of it.
Slide uses the remaining velocity to "slide" over the normal akin to Godot's move_and_slide. That's why many iterations are needed, the first part of the slide just moves the body to the collision point, then adjusts the velocity. While the velocity magnitude is bigger than 0, it will continue iterating

But now imagine that you want to implement triggers. How do you do that? For that, you implement a `CollisionFilterFunction<T>` where T is the type of your userdata. You can then do something like:

```csharp
(ref Body<T> body, ref Body<T> other) => {
    if (other.isTrigger) return CollisionResponses.Cross;
    else return CollisionResponses.Slide;
}
```

That allows you to detect a collision while not being stopped by it based on your custom rules, without having to tinker with layers and masks.

## Move

With all of that, I can already show you the big function

```csharp

    /// <summary>
    /// Attemps to move the body associated with <paramref name="handle"/> to <paramref name="targetPosition"/>
    /// </summary>
    /// <param name="handle">Handle to the item to move</param>
    /// <param name="targetPosition">Desired position the body at <paramref name="handle"/> should move</param>
    /// <param name="collisionFilter">Filter function that decides how each collision is handled</param>
    /// <param name="maxCollisions">Some <see cref="CollisionResponseFunction"/> need many iterations to be solved.
    /// This parameter limits the amount of iterations that can be done</param>
    /// <param name="testOnly">If <paramref name="testOnly"/> is true, the body does not move but the would-be collision information is given.</param>
    /// <returns>True if there was a collision</returns>
    public bool Move(Handle<Body<T>> handle, Vector2 targetPosition, CollisionFilterFunction<T> collisionFilter, int maxCollisions = 5, bool testOnly = false)
    {
        Debug.Assert(collisionFilter != null);
        Debug.Assert(_bodies.IsValid(handle));

        _lastCollisions.Clear();

        ref Body<T> body = ref _bodies[handle];

        RectangleF bounds = body.Bounds;
        Vector2 velocity = targetPosition - body.Bounds.Location;
        CollisionInfo lastCollision = CollisionInfo.NoCollision;

        for (int iteration = 0; iteration < maxCollisions; iteration++)
        {
            if (velocity.LengthSquared() <= float.Epsilon)
                break;

            CollisionInfo closestCollision = CollisionInfo.ValidFurthestCollision;
            Handle<Body<T>> lastCollisionHandle = Handle<Body<T>>.Default;

            foreach (HandleItem<Body<T>> item in _bodies)
            {
                if (item.Handle == handle)
                    continue;

                // Assymetric check, allows A to collide with B but B might not collide with A
                if ((body.Mask & item.Item.Layer) == 0)
                    continue;

                var response = CollisionFunctions.SweptAABB(bounds, item.Item.Bounds, velocity);

                if (response.HasCollision && response.CollisionTime < closestCollision.CollisionTime)
                {
                    closestCollision = response;
                    lastCollisionHandle = item.Handle;
                }
            }

            if (!closestCollision.HasCollision)
            {
                bounds.X += velocity.X;
                bounds.Y += velocity.Y;
                break;
            }

            lastCollision = closestCollision;
            _lastCollisions.Add(new(closestCollision, lastCollisionHandle));

            collisionFilter(ref Get(handle), ref Get(lastCollisionHandle))(ref bounds, ref velocity, closestCollision);
        }

        if (!testOnly)
            body.Bounds = bounds;

        return lastCollision.HasCollision;
    }
```

I think the code is self explanatory, but it shorts:
- It iterates N times
- In each iteration, it checks for collisions against every body, no spatial hashing whatsoever
- I store the closest collision, if any
- If there was a collision, add it to the array and then executes the response based on the filter
- Do it again until the for is over
- If testOnly is true, it doesn't move the item, but still reports the collisions
- Returns the last collision

Now that I'm writing this, I think there is a flaw, I might need to get the modified velocity, I should pass the motion instead of the target position so certain behaviours can be implemented, that will be for other day.

Inspired by Godot, I store the collisions in an array that can later be retreived with:

```csharp
    public ReadOnlySpan<CollisionHit<Body<T>>> LastCollsions => _lastCollisions.AsReadOnlySpan();
```

Only valid between `Move` calls, ideally should be called only once after `Move`. I could return it directly in Move and the user could check if it's empty to know whether there was a collision, but I didn't like that.

The implementation of CollisionHit is this:

```csharp
/// <summary>
/// Struct that contains collision info and a handle to the items the collision happened with
/// </summary>
/// <typeparam name="T"></typeparam>
/// <param name="collisionInfo"></param>
/// <param name="handle"></param>
public readonly struct CollisionHit<T>(CollisionInfo collisionInfo, Handle<T> handle)
{
    /// <summary>
    /// Normalized time in which the collision happened in the frame
    /// This value is invalid is <see cref="HasCollision"/> is false
    /// </summary>
    public readonly float CollisionTime { get; init; } = collisionInfo.CollisionTime;

    /// <summary>
    /// Remaining normalized time in the frame after the collision
    /// This value is invalid is <see cref="HasCollision"/> is false
    /// </summary>
    public readonly float RemaininTime => 1.0f - CollisionTime;

    /// <summary>
    /// Normal of the collision
    /// This value is invalid is <see cref="HasCollision"/> is false
    /// </summary>
    public readonly Vector2 Normal { get; init; } = collisionInfo.Normal;

    /// <summary>
    /// Handle to the object
    /// This value is invalid is <see cref="HasCollision"/> is false
    /// </summary>
    public readonly Handle<T> Handle = handle;

    /// <summary>
    /// Whether there was a collision
    /// </summary>
    public bool HasCollision { get; init; } = collisionInfo.HasCollision;
}
```

You don't get the `Body<T>` because it might be an struct, you have to resolve the handle as always. It might be cumbersome, but you can build your layer upon it and make it easier to use while keeping the performance and control.

# End

Seeing it now it looks like a code dump, but what I showed here it's just half or less of the whole implemenation. But anyways...

I hope you liked it! I'm finishing writing this at 1:41 but my personal contract forces me to document my progress daily so here we are. Some other day I'll rewrite this post a bit better. I know I haven't explained some decisions properly, but at the very least the information provided here is enough for meyself to remember in case I forget.