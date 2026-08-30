---
title: Darkrit devlog week 3
tags: ["c#", "darkrit", "monogame"]
draft: false
---

# Day 15 - Entity struct optimization

Today I had a revelation. My code was slower that at the beggining, I were at 0.19ms. I remember when I was at like 0.11ms. What did it change? The answer is **the entity size**. That struct grow bigger, the `StringID`, the `flags`, the `_lastWriteTick` and `_renderFrame` for physics interpolation, the 4 handles to create the hierarchy, the `_childCount`... All of that added little by little and made the struct big enough to cause cache misses.

But when I thought about it, I found out it was rather silly. Components mainly access the Entity to use the `Transform2D` or get some component handle, just that. *Why that incurs in bringing more data I don't need?* I thought. And the thing, that I don't need to bring that data.

I did a little experiment. I created a new array in World of ``EntityMetadata``

```csharp
internal struct EntityMetadata
{
    public Handle<Entity> _parent;
    public Handle<Entity> _firstChild;
    public Handle<Entity> _lastChild;
    public Handle<Entity> _nextSibling;
    public Handle<Entity> _previousSibling;
    public int _childCount;
}
```

And I changed my Entity fields to be getters to that structure.

```csharp
    internal readonly Handle<Entity> _parent
    {
        get => World.MetadataOf(Handle)._parent;
        set => World.MetadataOf(Handle)._parent = value;
    }
    internal readonly Handle<Entity> _firstChild
    {
        get => World.MetadataOf(Handle)._firstChild;
        set => World.MetadataOf(Handle)._firstChild = value;
    }
    // Same for the other fields
```


This was pretty cool, I didn't have to refactor anything else, just replace fields by properties.

---------------------------

And this worked surprisingly well, now I got back to 0.15ms, which is not the 0.11ms I had before but it's quite a lot. Unlike back then, I have some extra checks to perform between processing components

```csharp
    public void Update(GameTime gameTime)
    {
        InitializePendingComponents();
        
        if (!IsUpdateable) return;

        foreach (ref var handleItem in _components)
        {
            if (handleItem.Enabled && _componentMetadata[handleItem.Handle.Id].CanExecute)
                handleItem.Update(gameTime);
        }
    }
```

So there is the little overhead that adds to 0.15ms

But even so, I'm pretty happy with it. I did another experiment, I filled the entity with

```csharp
Matrix4x4 t1
Matrix4x4 t2
Matrix4x4 t3
Matrix4x4 t4
```

That made the simulation slow down to 0.5ms! Okay so then, will that happen too if I put that into the `EntityMetadata`? And the answer is no! Of course, if you access the parent or call `AddChild`, it will still be accessed and a penalty will be paid. But as I said, you don't do that on every single frame.

I also took advantage of this and took `ComponentList` as a separate array too, but for a different reason. Until now, when an entity was created or destroyed, a new ``List<TypedHandle>`` was created. Now I just reuse then, when the entity is destroyed, the list is cleared, when it's created again, it just access it through its handle:

```csharp
    private readonly ref ComponentList _componentList => ref World.ComponentsOf(Handle);
```

This call is inlined so there isn't any penalty or extra indirection from before. And know, I make less allocations. Really useful for a future bullet hell I want to make after the platformer. I don't need to do object pooling, my system is already so efficient you can just destroy and create entities on the fly.

-------------------------------

Now, you might wonder: Did you put that in `EntityMetadata`? And the answer is **no**. Even tho components handles can be cached and I can also cache the component store to not even use the entity, components are a bit more common to use, and I don't want to make them bring the hierarchy data too, so they live in its own array.

At the end, the entity is mainly just an ID to data that are now in other arrays. This accidentally looks a bit more like pure ECS, but it still has the Unity-ish API I like.

Due to work, I'm a bit more busy, but I hope that I can get the physics integration within this week. This sprint has been purely working on the Entity system, I don't want to spend much more on it. Once I add physics, I'll start working on the core components.

Until next time!

# Day 16 - Basic physics

I did a basics physic integration, but it's a bit clunky. I added a physics world inside the ``EntityRegistry`` to make this easier to use. Still I want a immediate mode api that doesn't rely on sync steps.

# Day 17 - Better physics

Thanks to Godot source code, I could fix some of my issues and I also centered the sprite and the colliders and I gave them an offset property. I also implemented some attributes for the editor: HideInInspector, Button, ReadOnly

This is taking form but it will take time. Given I'm making a platformer, I'm making the PhysicsBody be what CharacterBody is for Godot. Given my API is immediate mode, not calling `MoveAndSlide` means no movement will happen at all. I like this because even when PhysicsBody is specialized, it still works well for my use cases. Now, I've realized that using structs means I can't do derived components, but if I were using ECS I wouldn't be able neither, I guess I have to learn to live without inheritance.

I stil have to expose masks and layers, expose collisions... I will also create a small fixed buffers of 3 handles per PhysicsBody. It's a bit of space, but some operations require many calls to the physics world, and every Move request clears the PhysicsWorld collision list, so I have to save it. I have been avoiding it and I will still try, but I'll eventually need to store collision pairs for more complex processing.

# Day 18 - Moving platforms

I had clear in my mind how to do this one but, I have to confess that I looked into Godot source code again because I was insecure. Funnily enough I didn't end up following it.

So, Godot's way of dealing with moving platforms is just storing the platforms you're in, and adding their velocities to the body's. My system isn't physic, Boxy2D doesn't store velocities. And even if my `PhysicsBody` does, that's not reliable without hacks. Let's say I have this script:

```csharp
[Component]
[InjectComponent(typeof(PhysicsBody))]
public partial struct MovingPlatform
{
    [SerializeField] float amplitude = 100f;
    [SerializeField] float speed = 1f;

    Vector2 startPosition;

    /// <inheritdoc/>
    public void OnAdd()
    {
        startPosition = Entity.Position;
    }

    public void FixedUpdate(GameTime gameTime)
    {
        var timer = (float)gameTime.TotalGameTime.TotalSeconds;

        Entity.Position = startPosition with
        {
            X = startPosition.X + MathF.Sin(timer * speed) * amplitude
        };

        PhysicsBody.Teleport(Entity.Position);
    }
}
```

You see, I want to be able to teleport the physics body, or even just changing Entity.Position without manually syncing with PhysicsBody.Teleport.

Well, I did that:

```csharp
    public void FixedUpdate(GameTime gameTime)
    {
        if (_platformHandle.Id == 0)
            return;

        Vector2 platformPosition = World.Physics.Get(_platformHandle).Bounds.Location;
        Vector2 platformDelta = platformPosition - _platformPreviousPosition;

        if (platformDelta != Vector2.Zero)
        {
            Entity.Position += platformDelta;
            SyncPosition();
        }

        _platformPreviousPosition = platformPosition;
    }
```

I just store the platform handle when ``isOnFloor`` is true and I add the delta to the entity movement. This is not fully correct because if the platform moves to a wall, I will go through it, but as a first implementation works fine. This will get more complex in the future when I move the body physically, because there might be collisions that must be handled, and remember, my API is immediate, but this is not. I will probablly just store the collisions and save them for the next time the player calls MoveAndSlide, unless many frames passed since that in which case I would invalidate the handles. Idk, I'll have to see, for now I'm happy with this, it's enough to prototype.

Also, I added the `[Header("Title")]` attribute so have nice separators in the inspector. Since I learnt how to make attributes I can't stop.

Next day I want to expose layers in the inspector and think about how to implement triggers. I need that, a simple tween system, and particles before doing the game itself.

# Day 19 - Triggers

I just added a trigger area. Due to the nature of the system, triggers are not physcis objects. This is because I need them to detect others even whey they don't move, it doesn't make sense to use a AABB Swept for this. I also added more attributes, this time it was this:

```csharp
    [OnEditorChange(nameof(ApplyCollisionFilter))]
    [SerializeField] uint _layer = 1;
```

Given I can serialize auto properties but not properties, I had to make them explicit. This `_layer` field is only used in the editor, it does't exist in  release. It's just a proxy that allows me to change the inner body layer and mask during runtime inspection.

# Day 20 - Camera

I have a simple Gizmo system that didn't work when I used a camera in my scene. What I did for that was making Camera a static property of Core. Other scenes can build virtual cameras around it. This adds some coupling but given I'm making a specialized framework I guess I can get away with it.

# Day 21 - Doubts

I'm pretty happy with how my system works at a technical level. And even when I can tweak it, I'm not sure if it's the best for the game I want to make. Which makes sense, I kinda made this just because I wanted to build it, not because I needed it. The followings day I will be focusing on game design. I will think about what mechanics I need, how easy is implementing them in this system and how easy it would be if I code it differently. Probably the next week devlogs will be just "I'm thinking, I didn't do much" for a while.