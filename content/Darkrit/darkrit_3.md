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