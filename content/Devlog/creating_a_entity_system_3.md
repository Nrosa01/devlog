---
title: Creating an Entity System - Part 3
tags: ["c#", "monogame", "darkrit-entity-model"]
draft: false
date: 2026-08-17
---

This is the third part of a series, you check the rest of it [here](tags/darkrit-entity-model). I recommend reading those before this one as I assume concepts explained previously.

I have made some upgrades to the system, still a long way to go, but it's getting pretty interesting.

# Removing Components

This system has a problem. It can't easily delete entities. Why? Because `World.GetComponent<T>{:csharp}` is generic, it needs a type to resolve the index to the `ComponentStore<T>` array. But... We already know that index, the dictionary key is the component type index. If I expose a non generic method, it can work. Only issue will be the handle types, but that can be worked out easily.

```csharp
    internal bool RemoveComponent(int typeId, Handle iComponent) => _componentStores[typeId].TryRemove(iComponent);
```
And inside the store:

```csharp
    public bool TryRemove(Handle<IComponent> handle)
    {
        return TryRemove(new Handle<T>
        {
            Id = handle.Id,
            Generation = handle.Generation
        });
    }

    public bool TryRemove(Handle handle)
    {
        return TryRemove(new Handle<T>
        {
            Id = handle.Id,
            Generation = handle.Generation
        });
    }
```

Of course those have to be added to the `IComponentStore` interface. This way it's possible to remove an item even if we don't have the typed handle. Those will be made private in the future as from outside the library only typed handles will be allowed. Handles being typed are just to avoid misuse as you could accidentally use a `Handle<Entity>` in a function that expected a `Handle<Component>`.


# GetComponent Optimization

`GetComponentHandle` is a bit slow because it's going through a Dictionary. Sure they're constant time, but it's a high constant time. It would be much faster to build a dedicated data structure given the amount of component types is known at compile time. But I don't know how to do that so I'll just make a list wrapper.

```csharp
struct TypedHandle
{
    public Handle handle;
    public int type;

    public static TypedHandle Create<T>(Handle<T> handle) where T : struct, IComponent => new()
    {
        handle = new Handle
        {
            Id = handle.Id,
            Generation = handle.Generation
        },
        type = ComponentTypeId<T>.Id
    };
}

struct ComponentList
{
    List<TypedHandle> _handles = [];

    public readonly IReadOnlyList<TypedHandle> Components => _handles;
    public ComponentList()
    {
        _handles = [];
    }

    public readonly void Add<T>(Handle<T> handle) where T : struct, IComponent => _handles.Add(TypedHandle.Create(handle));

    public readonly Handle<T> Get<T>() where T : struct, IComponent
    {
        int id = ComponentTypeId<T>.Id;
        foreach (var item in _handles)
        {
            if (item.type == id)
                return new Handle<T>
                {
                    Id = item.handle.Id,
                    Generation = item.handle.Generation
                };
        }

        return Handle<T>.Default;
    }

    public readonly Handle Remove<T>() where T : struct, IComponent => Remove<T>(default, true);

    public readonly Handle Remove<T>(Handle<T> handle) where T : struct, IComponent => Remove<T>(handle, false);

    private readonly Handle Remove<T>(Handle<T> handle, bool onlyCheckType) where T : struct, IComponent
    {
        int id = ComponentTypeId<T>.Id;

        int toRemove = -1;

        for (int i = 0; i < _handles.Count; i++)
        {
            TypedHandle item = _handles[i];
            if (item.type == id && (onlyCheckType || (item.handle.Id == handle.Id && item.handle.Generation == handle.Generation)))
            {
                toRemove = i;
                break;
            }
        }

        // Swap and remove
        if (toRemove != -1)
        {
            var handleToRemove = _handles[toRemove];
            _handles[toRemove] = _handles[_handles.Count - 1];
            _handles.RemoveAt(_handles.Count - 1);

            return handleToRemove.handle;
        }

        return Handle.Default;
    }

    internal void Clear() => _handles.Clear();
}
```

As you see, it's just a list that defines an untyped handle and the type id as an int. That's the only information needed to do the GetHandle operation:

```csharp
    public Handle<T> GetComponentHandle<T>() where T : struct, IComponent => _componentList.Get<T>();
    
    public Handle<T> AddComponent<T>(T component) where T : struct, IComponent
    {
        Handle<T> componentHandle = World.AddComponent(Handle, component);

        _componentList.Add<T>(componentHandle);

        return componentHandle;
    }
    
    public bool RemoveComponent<T>(Handle<T> handle) where T : struct, IComponent
    {
        bool worldRemoves = World.RemoveComponent<T>(handle);
        var removedHandle = _componentList.Remove<T>(handle);
        bool entityRemoves = removedHandle.Id != 0;

        if (worldRemoves && !entityRemoves)
            Log.Warning($"Component of types {typeof(T)} with handle {handle} couldn't be removed from entity but it could be removed from World");

        if (!worldRemoves && entityRemoves)
            Log.Warning($"Component of types {typeof(T)} with handle {handle} couldn't be removed from World but it was removed from entityt");

        return worldRemoves && entityRemoves;
    }
```

My benchmarks show that this gives a nice speed boost when components are 10 or less, from there it might perform worse. Most of the time I have less than 10 components and anyways, Handles should be cached by the caller so this is nice to me, worth the extra code.

Only thing I don't fully like is the Entity calling a world method, the world shouldn't do those operations so O might just move those helpers to the Entity and directly access the store there so World doesn't even have those methods. I'm still thinking about that. I think World should just create entities and provide a component storage that entities can use. But that means entities have access to world internals, something that doesn't happen when I just call the World method

> [!Info] Info!
> I call the class `EntityRegistry` `World` often, they mean the same. But I don't feel that calling the class `World` is accurate. A World of what?

# System performance

I have measured the perfomance of 10_000 entities with 2 componentes:

```csharp

[Component]
public partial struct Mover
{
    static readonly int WindowsWidth = Core.GraphicsDevice.Viewport.Width;
    static readonly int WindowsHeight = Core.GraphicsDevice.Viewport.Height;

    Handle<SquareRenderer> squareHandle;
    ComponentStore<SquareRenderer> store;

    public Vector2 Velocity;

    public void Update(GameTime gameTime)
    {
        var size = 10; // We should get size from SquareRenderer

        Entity.Transform.Position.X += Velocity.X;
        Entity.Transform.Position.Y += Velocity.Y;

        if (Entity.Transform.Position.X < 0 || Entity.Transform.Position.X + size > WindowsWidth)
            Velocity.X *= -1;

        if (Entity.Transform.Position.Y < 0 || Entity.Transform.Position.Y + size > WindowsHeight)
            Velocity.Y *= -1;
    }
}

[Component]
public partial struct SquareRenderer
{
    public int Size;
    public void Draw(GameTime gameTime)
    {
        var pos = Entity.Transform.Position;
        float r = pos.X - MathF.Floor(pos.X);
        float g = pos.Y - MathF.Floor(pos.Y);

        var color = new Color(r, g, pos.X);
        Core.SpriteBatch.Draw(Core.Pixel, new Rectangle((int)pos.X, (int)pos.Y, Size, Size), null, color);
    }
}
```

As you see, the `SquareRenderer` just draws a Rectangle to screen. The interesting part is in `Mover`. It needs to know the size of the `SquareRenderer`, this kind of component dependency is usual, componentes by themselves are limited.

Now, this is a handle based system, unlike in Unity, you can't store a reference to a component. There are many ways to get a component reference. I'll walk you through the slowest to the fastest one.

First and slowest one:

```csharp
// This in the update function
Entity.GetComponent<SquareRenderer>()
```
As it doesn't receive any handle, it performs a search on the `ComponentList` we saw before, and returns the first component of type `SquareRenderer`. The search is fast, but it's still slow. My simulation runs at 0.20ms when doing this approach.

Second one:

```csharp {3,8}
    public void Start() 
    {
        squareHandle = Entity.GetComponentHandle<SquareRenderer>();
    }

    public void Update(GameTime gameTime)
    {
        var size = Entity.GetComponent(squareHandle).Size;
```

You get the ``Handle`` and cache it. Then you can get the reference via the ``Handle``. This is much faster, now it takes only 0.15ms to run. But it's still a lot. When I harcode the size into a variable, it runs at 0.10ms. Can I get closer to that?

When I profiled, I noticed where I was the overhead. Let's see what ``GetComponent(Handle)`` does again:

```csharp
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ref T GetComponent<T>(Handle<T> componentHandle) where T : struct, IComponent => ref World.GetComponent<T>(componentHandle);
```

Alright so let's see what the World one does:

```csharp
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal ref T GetComponent<T>(Handle<T> componentHandle) where T : struct, IComponent => ref GetStore<T>().Get(componentHandle);
    
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ComponentStore<T> GetStore<T>() where T : struct, IComponent => (ComponentStore<T>)(_componentStores[ComponentTypeId<T>.Id] ??= new ComponentStore<T>(initialCapacity));
```

So here's the issue. `GetStore<T>`. It seems that the casting adds a minimal overhead that is noticeable when it runs a lot of times per frame. I also tried returning just the component, without doing the `??=` part. And that didn't make a difference, so I left it there as that's the entry point to store, I could initialize the type in the `AddComponent<T>` but conceptually I find cleaner to have it in the `GetStore<T>` given it doesn't affect performance.

The `IComponentStore.Get` it's not an issue, under the hood it's an array access that gets inlined.

So... What now? Let's go to the third version of this

```csharp {1,6,11,12}
    ComponentStore<SquareRenderer> store;

    public Vector2 Velocity;
    public void Start() 
    {
        squareHandle = Entity.GetComponentHandle<SquareRenderer>();
    }

    public void Update(GameTime gameTime)
    {
        store ??= World.GetStore<SquareRenderer>();
        var size = store.Get(squareHandle).Size;
```

This is the fastest version, running at an average of 0.11-0.12ms. You might be thinking *"Shouldn't you cache the store in the ``Start`` instead of doing it on the `Update` every frame"*. And you're right, but I benchmarked and it made no difference so I chose this as it's easier to read, the less lines, the better (usually).

There is something else that can be done:

```csharp {1}
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Update(GameTime gameTime)
    {
        store ??= World.GetStore<SquareRenderer>();
        var size = store.Get(squareHandle).Size;
```

It seems like the method wasn't being inlined. And making it inline boosted it to 0.8-0.9 ms. Much better! The only issue is that it's not ergonomic to write. The price to pay for performance. Maybe I'm missing something something here, but after testing and benchmarking, this is the best I could get. Now, let's make it easy to use.

## Component Injection

I've been told to not abuse Source Generation, but I will allow myself to abuse it a bit more. All of the previous boiler plate is known at compile time, it can be generated, it's always the same pattern. So I did that. First, I need a way to provide the data to the generator. I'll borrow Unity's `RequireComponent(Type)` attribute idea and create a `InjectComponent(Type)` attribute. Its definition is:

```csharp
[AttributeUsage(AttributeTargets.Struct, AllowMultiple = true)]
public sealed class InjectComponentAttribute(Type componentType) : Attribute
{
    public Type ComponentType { get; } = componentType;
}
```

It's important to that `AllowMultiple` is true so you can inject different types of components. Then, in the source generator:

```csharp
    private static IEnumerable<INamedTypeSymbol> GetInjectedComponents(INamedTypeSymbol type)
    {
        foreach (var attribute in type.GetAttributes())
        {
            if (attribute.AttributeClass?.ToDisplayString() != "Darkrit.EntityModel.InjectComponentAttribute")
                continue;

            if (attribute.ConstructorArguments.Length == 0)
                continue;

            if (attribute.ConstructorArguments[0].Value is INamedTypeSymbol component)
                yield return component;
        }
    }

    private static string GenerateInjectedComponents(INamedTypeSymbol type)
    {
        StringBuilder builder = new();

        foreach (var component in GetInjectedComponents(type))
        {
            var componentName = component.ToDisplayString(SymbolDisplayFormat.MinimallyQualifiedFormat);
            var propertyName = component.Name;

            builder.AppendLine($$"""
            private ComponentStore<{{componentName}}> {{propertyName}}Store => field ??= World.GetStore<{{componentName}}>();

            private Handle<{{componentName}}> {{propertyName}}Handle
            {
                get
                {
                    if (field.Id == 0)
                        field = Entity.GetComponentHandle<{{componentName}}>();

                    return field;
                }
            }

            public ref {{componentName}} {{propertyName}} => ref {{propertyName}}Store.Get({{propertyName}}Handle);
            """);
        }

        return builder.ToString().TrimEnd().Replace("\n", "\n    ");
    }
```

What I generate is:
- The ``Handle<T>`` of the component
- The ``ComponentStore<T>`` of the component
- A ref T of the Component itself

And everything is implemented as properties that check themselves. This way I don't need to generate anything on the `Start` function, because I have no idea of how I would manage the situation where the user defined its own `Start`, I would probably need to parse that and append the fields at the beggining of the method and then... Just no, I want to keep things simple. My benchmark showed no performance penalty by having all of this checks, so this works for me.

Also, I use ``MinimallyQualifiedFormat`` because otherwise it would generate something like:

```csharp
    public void Draw(global::Microsoft.Xna.Framework.GameTime gameTime) { }
```

And that's ugly to read, I already generated the proper usings at the beggining of the file. I believe that the generated code should be nice to read to. It's not really needed, but I prefer it that way given it was fast and easy to do.

How does this look? Let me show you:

What I write:

```csharp {2,16}
[Component]
[InjectComponent(typeof(SquareRenderer))]
public partial struct Mover
{
    static readonly int WindowsWidth = Core.GraphicsDevice.Viewport.Width;
    static readonly int WindowsHeight = Core.GraphicsDevice.Viewport.Height;

    Handle<SquareRenderer> squareHandle;
    ComponentStore<SquareRenderer> store;

    public Vector2 Velocity;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Update(GameTime gameTime)
    {
        var size = SquareRenderer.Size;

        Entity.Transform.Position.X += Velocity.X;
        Entity.Transform.Position.Y += Velocity.Y;

        if (Entity.Transform.Position.X < 0 || Entity.Transform.Position.X + size > WindowsWidth)
            Velocity.X *= -1;

        if (Entity.Transform.Position.Y < 0 || Entity.Transform.Position.Y + size > WindowsHeight)
            Velocity.Y *= -1;
    }
}
```

And the generated code is:

```csharp {18-31}
using Microsoft.Xna.Framework;
using System.Runtime.InteropServices;
using global::Darkrit.EntityModel;
using global::Darkrit.Base;

namespace DarkritGame.Scenes;

[Updateable]
[StructLayout(LayoutKind.Auto)]
public partial struct Mover : IComponent
{
    public ref Entity Entity => ref World.GetEntity(EntityHandle); 

    public EntityRegistry World { get; set; }
    public Handle<Entity> EntityHandle { get; set; }
    public bool Enabled { get; set; }

    private ComponentStore<SquareRenderer> SquareRendererStore => field ??= World.GetStore<SquareRenderer>();
    
    private Handle<SquareRenderer> SquareRendererHandle
    {
        get
        {
            if (field.Id == 0)
                field = Entity.GetComponentHandle<SquareRenderer>();
    
            return field;
        }
    }
    
    public ref SquareRenderer SquareRenderer => ref SquareRendererStore.Get(SquareRendererHandle);

    public void Start() { }
    public void FixedUpdate(GameTime gameTime) { }
    public void Draw(GameTime gameTime) { }
}
```

Exactly what you'd expect, it generates the two autoproperties and the ref getter. Of course this only works to get component references from the entity it belongs to. If you want to get a component from another entity, you'll have to do the store caching and all of that manually. Often, caching is not needed if the access is one-off access.

Right now, the ergonomics of my system are not the best, Unity and Godot are much better than that. But I got a performance they can't rival, so I'm happy.

## Debug performance

There is one performance issue: It's REALLY slow on debug. Even with this optimized version, it runs at around 6ms in debug. That's quite a bit from the 0.08ms I was getting in Release. My `TinyECS` library that I used as ref, runs at 0.4ms in Debug and 0.6ms in Release. I guess the compiler can optimize the ECS one in debug given is more straightforward and has less indirections. That hurts me a bit, I couldn't get the same performance as ECS doing an OOP Unity like system. But that said, what matters is what players receive, and they will receive a fairly optimized game, while I get easy of development and tools I enjoy. I guess I can't have everything.

# End

I'm slowly getting close to having something usable. What's up for the next days?
- Finish writing Units Tests 
- Finish documenting code
- OnEnable/OnDisable callbacks
- Implementing Physics Interpolation for ``Entity.Position``
- Figure out how to implement `Boxy2D` without wiring up logic (although I will probably have to to sync the physics transform and the logical one...)
  - Might implement OnCollionEnter/Exit, also OnTriggerEnter/Exit, which would wire up physics with my entity system. But I can always take that out if I make a game without physics. The nice part of being your library and being able to bend it as you desire
- Add `Tag` to `Entity`
- Add `Name` field to `Entity` (via string interning)
- Implement the equivalent of Unity's [DefaultExecutionOrder](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/DefaultExecutionOrder.html)
- Implement component parallelization via `[Multithreaded]` (the user is responsible for correctly using it)
  - This is mainly to parallelize the `Update` and ``FixedUpdate` functions, `Draw` is always single-threaded

Until next time!

[> Part 2](../Devlog/creating_a_entity_system_1.md).
<!-- [> Part 4](../Devlog/creating_a_entity_system_4.md). -->