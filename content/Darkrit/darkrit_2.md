---
title: Darkrit devlog week 2
tags: ["c#", "darkrit", "monogame"]
draft: false
---

# Day 8

I just documented the code and cleaned it a bit. What I have to do now is pretty straightforward and not interesting so I don't think I will be doing a dedicated post about the next steps, at least until I get to hierachies. I also started writing unit tests and benchmarks against my `TinyECS` library to make sure that what I'm doing performs correctly.

That said, yesterday I did so many improvements in the `Entity Model` I couldn't write about everything. So today I will cove that in (you guessed it) a [dedicated post for it](../Devlog/creating_a_entity_system_3.md).

# Day 9

I just did some unit tests and minor improvements like making the generated `ref Entity` a `readonly ref Entity`. I also realized that the component has the entity handle exposed, that means you can modify the handle which you mustn't. But I can't do anything about it, given I'm the user there will be no issues. Also I can't make the getters internal for Darkrit given some components are generated on the game assembly.

Next updates will be slow, as I will implement hierachies which is the hardest part. Doing a logical hierachy in itself is easy, the hard part is guaranteeing the update order. Say that Entity B is child of A. Entity A components should run before Entity B ones, regardless of the component types. That's impossible right now as componentes are executed through the stores. There might not be an anwer, but I will still add hiearchies, even if they don't guarantee update order, they can help me to organize scenes into group, disable group of entities and similar stuff. The following days we'll see.

# Day 10

Due to medical issues I couldn't do much, probably this will extend to the rest of the week. I could only implement some tests and the basics of hiearchy, I added this to my entity struct:

```csharp
    internal Handle<Entity> _parent;
    internal Handle<Entity> _firstChild;
    internal Handle<Entity> _nextSibling;
    internal Handle<Entity> _previousSibling;
```

I also added `[assembly: InternalsVisibleTo("Dakrit.Tests")]` to get access to internal fields in the test project. It's surprising I could write +100 tests without that.

This way I don't need to create big allocations. Hiearchies are usually stable so even if updating the handles might be a bit expensive, I can afford it. I intent to propagate some calls to children to implement `OnEnable`, `OnDisable` and recursively destroy children.

Eventually I'll also want to add `Transform propagation` but that's a on its own league. Once I get the basics of relations I'll write a post about it. If I get better I might even get time to finally write some cool fancy ImGui windows.

Oh, I also found out that when you return a struct as out param, it doesn't return a reference, quite sad. So sadly I'm forced to

```csharp
var entityHandle = world.CreateEntity();
ref Entity entity = ref world.GetEntity(entityHandle);
```

I know I could make something like  `public ref Entity CreateEntity(out Handle<Entity> entityHandle){:csharp}`. But I don't like the inconsistency of the return type being that instead of the handle. Maybe in the future I end up allowing it if I get bored of writing those two lines. Or maybe, given the `Entity` has a handle to itself public, I can just return the Entity directly and you can access the handle if needed. I'll have to think about that as I don't want to abstract the handles, from a library perspective I want my hypothetic users to know about handles.

# Day 11

Alright so lots of things happened but I don't have time for longer posts. I managed to get the hierarchical transform update. It's working right, but the performance when there are structural changes (adding or removing components) is bad so I decided to not use it for now. I liked the idea because I'm used to Godot nodes updating in order, but I can also live with a Unity system that updates randomly but more efficiently. I left the implementation and I can toggle it with a single bool. I also had to add `internal Handle<Entity> _lastChild` to support inserting at the end without having to loop through all children.

I also had to deal with loops, that was a bit of a headache. When I say loops I say trying to do stuff like adding A as child of B and B as child of A. That's a simple loop but there are other ones. Nothing that crying and debugging on paper can't solve.

# Day 12

I implemented physics interpolation:

```csharp
    private ulong _lastWriteTick;
    private ulong _renderFrame;

    private Transform2D _previous;
    private Transform2D _current;
    private Transform2D _renderTransform;

    private void EnsureCurrentTick()
    {
        if (_lastWriteTick != World.Tick)
        {
            _previous = _current;
            _lastWriteTick = World.Tick;
        }
    }

    public Vector2 Position
    {
        get
        {
            if (World.IsDrawing)
                return RenderTransform.Position;

            return _current.Position;
        }

        set
        {
            EnsureCurrentTick();
            _current.Position = value;
        }
    }

    public void Teleport(Vector2 position)
    {
        _current.Position = position;
        _previous.Position = position;
        _renderTransform.Position = position;
    }

    public void ResetInterpolation()
    {
        _previous = _current;
        _renderFrame = ulong.MaxValue;
    }

    private Transform2D RenderTransform
    {
        get
        {
            if (_renderFrame != World.RenderFrame)
            {
                _renderTransform = Interpolate(_previous, _current, World.FixedUpdateAlpha);
                _renderFrame = World.RenderFrame;
            }

            return _renderTransform;
        }
    }
```

I just made created a property `Tick` that I increment at the beggining of the FixedUpdate. Then I use that to compute the interpolated Render Transform. I also track what part of the Game Loop I'm in so I can hide the RenderTransform and Drawable Componentes can just use Entity.Position or Entity.Transform. I love that the physics interpolation system is abstract to the component. I can just take it away if needed.

Why am I adding this? Well, part of premature optimization and part because it's fun. I want to be able to lower the physics tick while still having smooth render. The price to pay is slower runtime, and not because of the check, I profiled and the issue is cache thrashing caused by the Entity struct being bigger. I tried adding a couple of matrixes and it got x3 slower. So I need to be REALLY careful about how much data I add. I also added a `int flags` and `StringID name` that it's just an int. StringID is a pointer to a string, a simple string interning for entities.

I also changed the API. Now CreateEntity returns ref Entity. Why? Because often when you get an entity you later want its ref. Given the entity has a handle to itself you can still easily check it. Entities doesn't have a self handle yet, but they will probably have it. Let's say you want to destroy a component from within itself. You can't now, the component doesn't have a reference to itself. I tested performance when adding the Handle and it has a minimal but noticeable impact of around 0.005ms. So as long as I don't need that I won't add it. Perfomrnace right now is at 0.17-0.19ms at 10k entities. The issue is that this is only for entities with 2 components and little data. When I get more data it might be x10 worse. I will still be pretty good performance, just not crazy like in ECS.

Next day I will add OnEnable/OnDisable, start physics implemnentation and that will be the thing. Serialization is a monster that I'll tackle later, first I want to play with this system and see what I can do with it.

# Day 13

I added LateUpdate, OnEnable/Disable, changed the API again... One big change was that now, `HandleMapGrowing<T>` receives a IHandle, internally uses T instead of ``HandleItem<T>``. Why? Because I found out silly having the handle in the handle item and also in the entity, the component would eventually need its own handle. I realized that the design forces that, and given the handle is needed anyways, it doesn't incur into more space costs. The issue is encapsulation, now, you can modify the handle that's used to check against in the structure, and making it readonly is also troublesome. So to not waste more memory in handles, I decided to pay this price and go with it. That also made api more comfortable to use. I also added A TON of documentation to the system now that is more stable.

# Day 14

I didn't have much time today, I improved the editor debug render to display the hierachy using the ImGui clipper. I also added a basic component renderer for the inspector. Now I have a Unity-like inspector, including ``[SerializeField]`` and `[ShowInInspector]` and such. It's just fighting ImGui to make it look good and responsive. I'm quite happy with it, but these last days I'm running of time to write better posts. Once I get more things in place I'll try to write more.

I also ported Miisan ImGui gizmos to C#: https://github.com/Nrosa01/ImGizmo2D/

The next step is adding the my Boxy2D physics to the entity system. After that, I will try create some componentes like TilemapRenderer, Sprite and whatever I need to create a simple 2D platformer.

I have to be careful to not get crazy with the editor. I know I can't do like unity and having a Scene view and Game view, that's a nightmare to handle, mainly because of `static` and global state I would have to handle. [Domain Reloading](https://docs.unity3d.com/2022.3/Documentation/Manual/DomainReloading.html). I will still need to make some simple editor to auth my games, but first I will do harcoded scenes to test the system.