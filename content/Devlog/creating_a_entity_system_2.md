---
title: Creating an Entity System - Part 2
tags: ["c#", "monogame"]
draft: true
---

This is the second part of a series, I recommend you reading [the first one first](../Devlog/creating_a_entity_system_1.md).

# The issue(s)

If you remember from last time, `Component`s are struct that implement `IComponent`. The issue is that for evey component I have to implement the same autoproperties all the time. If I add a new one to the interface, I have modify all the components to add it, which is not acceptable.

Most of this data, is `Component Metadata`, which means that I can just have something like:

```csharp
public class ComponentStore<T>(int initialCapacity) : IComponentStore where T : struct, IComponent
{
    private readonly HandleMapGrowing<T> _components = new(initialCapacity);
    GrowableArray<ComponentMetadata> _componentMetadata = new(initialCapacity);
```

Then I can just use the Id of the component handle to index on that array. That way, I can make fields private in the struct so they can't be used from the game. As always, even if this library is only for my own usage, I like building it as if it were a public library.

Now there is a big issue. I might need to use this data in the component. There are two ways to do that.

---------------
 
The first one is passing it as a ref parameter in `Update`, `FixedUpdate` etc. But that leaks implementation details, and makes it harder to access the data from outside the component.

Another option is making extension methods like the last time and also adding a Handle<T> in the interface, but for that I need to make the interface generic. That's not really bad, I just prefer to keep things as simple as possible. Also, this would me that the access would be something like `World.GetMeta(handle).Property` under the hood. 

I've simplified the problem since I've already fixed it and I don't remember when I was thinking 8 hours ago.

There is a simple fix for all my problems: Source Generators and public/private API segregation.

First of all, I will keep the componentMetadata. But that will only store private data, the component won't be able to access it, the ComponentStore will manage it. One example of this is `Initialized` field. When an entitiy is added, is not initialized, componentes initialize at the end of the next frame. My components aren't deferred, when you call `AddComponent`, it gets added to the array. Why? Remember that it's an array of struct that returns a handle. You usually want to:

```csharp
var entityHandle = world.CreateEntity();
ref var entity = world.GetEntity(entityHandle);
var componentHandle = entity.AddComponent<PlayerController>()
ref var component = entity.GetComponent<PlayerController>()
component.speed = 10;
```

For this to work, I need the component to exist immediately. If this were an more OOP system, I could just store the class component instance and return it, given it's a reference, it would work even when I move it to the other array. This is harder with my system, so I have to add the component immediately. Given everything uses handles, even if the array resizes during the iteration, it will work properly.

The same problem applies when creating entities during any function callback.

That of course means that you can process a component that has just been added and not initialized. There are two ways to handle that:
- Initialize the component immediately
- Keep initialization deferred but flag the component as not initialize to not iterate it.

I could do the first and there won't be any issues thanks to the immediate mode. But the issue is that the component might or might not update depending on where we are in the iteration. Given I use a handle based array, I might be iterating component 40 of 50, but maybe component 27 is an empty position that wasn't being used and now the new component has been putted there.

And yes, this means that when you remove a component or an entity, the array size doesn't change, you still iterate the same amount of entities. And given I'm using handles, I can't reorder the entities, as that would invalidate the handles. This can be an issue under certain situations but not for the way I'm using this system, I accept this issue.

That settles me into the deferred options. I need a flag that must not be exposed in the `Component` interface