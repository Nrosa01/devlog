---
title: Creating an Entity System - Part 1
tags: ["c#", "monogame", "darkrit-entity-model"]
draft: false
date: 2026-08-15
---

# Motivation

Fat structs start falling apart quicky and I'm used to Godot and Unity. I just want more control about the implementation so I will be doing that. Both Godot and Unity systems have things I like and things I don't. 

Things I like: 
- They are easy to use
- I'm experienced at using them
- They're easy to build

Things I don't like:
- The performance could be better
- You're forced to use them

My system won't be enforced. I can use the Entity System, but I can also mix it with other system, or not using at all. My scenes aren't constrained to anything

# Requeriments

- Struct and handle based. No reference issues
- Entities are contiguous in memory
- Components are contiguous structs in memory, one array per component type. No sparse sets
- Optional component update paralellization
- Components that don't implement Update, shouldn't need to update
- Familiar API, GetComponent, AddComponent, QueueFree... Inspired by Unity and Godot
- Simple implementation, easy to maintain or change

I don't intend for this to be a rigid system. I will write a basic generic system, but I will adapt it for every game I make. Sure my library won't be compatible for all my projects, every project will use a specific custom version of `Darkrit`. But that's fine because my goal ultimately is doing games the way I want, not doing a public library so it's fine.

# Implementation

Before doing the usual code dump and explaining, I'd recommend reading [this post](./creating_a_2d_collision_system_1.md) first as some of the concepts I explain there are heavily used here, especially the `Handles`.

> [!Warning] Warning!
> This implementation is still WIP, some features are missing, some others my change in the future.

## Components

So let's start. I will first part about the smallest part: Components. They are structs that implement `IComponent` interface:

```csharp
public interface IComponent
{
    public EntityRegistry World { get; set; }

    public Handle<Entity> EntityHandle { get; set; }

    public ref Entity Entity => ref World.GetEntity(EntityHandle);

    /// <summary>
    /// Whether this Component is enabled
    /// </summary>
    public bool Enabled { get; set; }

    /// <summary>
    /// Whether this component's <see cref="EntityHandle"/> is active in the hierarchy
    /// </summary>
    public bool ActiveSelf { get => World.GetEntity(EntityHandle).ActiveSelf; }

    /// <summary>
    /// Whether this component's <see cref="EntityHandle"/> is active in the scene
    /// Say, Entity A has a child Entity B. B could be active but A 
    /// be inactive, which would result in B <see cref="ActiveInHierachy"/> be false
    /// while <see cref="ActiveSelf"/> is true
    /// </summary>
    public bool ActiveInHierachy { get => World.GetEntity(EntityHandle).ActiveInHierachy; }

    void Start() { }

    void Update(GameTime gameTime) { }
    void FixedUpdate(GameTime gameTime) { }
    void Draw(GameTime gameTime) { }
}
```

It just defines the basic functions it needs, the handle of the entity it belongs too and a reference to the World as everything depends on that. Unlike in Unity, here there isn't a World singleton, you can't GameObject.Instantiate or things like that, you need a reference to do that. The system is self contained that way and can work with others without having to care about global state.

Entity Handle and World are set upon component creation, as components can only be created through the world.

The current implementation doesn't have hiearchies, that's a placeholder for the future.

You might have noticed something suspicious, all components will have Update, FixedUpdate and Draw. But not all components do all of that. That means, I will be calling empty functions for some components type. This is already addressed, more about that soon.

## Component Store

Where are those componentes stored? In a `HandleMapGrowing`, everything is a handle here. A Component store is mainly a thin wrapper over it:

```csharp showLineNumbers{17}
public class ComponentStore<T>(int initialCapacity) : IComponentStore where T : struct, IComponent
{
    public readonly HandleMapGrowing<T> Components = new(initialCapacity);
    private Stack<Handle<T>> nonInitializedComponents = new();
```
The class provided an ``Add``, ``TryRemove``, ``Contains``, and ``ref T Get`` methods. Those are straightforward so I won't show them. In this class no `Handle<Entity>` is used, this class only cares about components.

Now, it also has `Draw`, `Update` and `FixedUpdate` methods:

```csharp showLineNumbers{61}
    public void FixedUpdate(GameTime gameTime)
    {
        if (!IsUpdateable) return;

        foreach (ref var handleItem in Components)
            handleItem.Item.FixedUpdate(gameTime);
    }

    public void Draw(GameTime gameTime)
    {
        if (!IsRenderable) return;

        foreach (ref var handleItem in Components)
            handleItem.Item.Draw(gameTime);
    }
```

Nothing fancy, it just iterates the `IComponent`s and calls the appropiate method. Given they are contiguous in memory the updates are fast, or so I think, I haven't benchmarked yet but it should be better than having a list of pointer reference to class components.

You might have noticed to flags there `IsRenderable` and `IsUpdateable`. Components of type T run those methods based on that flags. Where does that come from? 

```csharp showLineNumbers{17}
[AttributeUsage(AttributeTargets.Struct)]
public sealed class UpdateableAttribute : Attribute { }

[AttributeUsage(AttributeTargets.Struct)]
public sealed class RenderableAttribute : Attribute { }

public class ComponentStore<T>(int initialCapacity) : IComponentStore where T : struct, IComponent
{
    private static readonly bool IsUpdateable =
        typeof(T).IsDefined(typeof(UpdateableAttribute), inherit: false);

    private static readonly bool IsRenderable =
        typeof(T).IsDefined(typeof(RenderableAttribute), inherit: false);
```

Component structs might implement those attributes optionally, they might not even implement then! This allows to save some resources quite easily. An example of defining a component would be:

```csharp
[Updateable]
public struct PlayerController : IComponent
{
```

And it works with Native AOT compilation!

## EntityRegistry

Here things get a bit more interesting.

```csharp showLineNumbers{25}
public class EntityRegistry(int initialCapacity)
{
    private readonly IComponentStore[] _componentStores = new IComponentStore[ComponentTypeId.Count];
    private readonly HandleMapGrowing<Entity> _entities = new(initialCapacity);
```

Here I just initialize the Entity array with an initial capacity and the array of IComponentStores. the size of that array is based on the amount. 

```csharp showLineNumbers{13}
public static class ComponentTypeId
{
    private static int _nextId;

    public static int Next()
    {
        return Interlocked.Increment(ref _nextId) - 1;
    }

    public static readonly int Count = ReflectionUtils.CountDerivedTypes<IComponent>();
}

public static class ComponentTypeId<T> where T : struct, IComponent
{
    public static readonly int Id = ComponentTypeId.Next();
}
```

I use a NativeAOT compatible Reflection util to count the amount of structs that implement IComponent and store that into that class for convenience. Then, I have a generic version of that class to generate a sequential id that is used here:

```csharp showLineNumbers{40}
    public ComponentStore<T> GetStore<T>() where T : struct, IComponent => (ComponentStore<T>)(_componentStores[ComponentTypeId<T>.Id] ??= new ComponentStore<T>(initialCapacity));
```
That way I have fast lookups without needing dictionaries. I have to thank `PaperClip` on `MonoGame` discord for that tip. 
And there isn't much here, EntityRegistry is mainly a wrapper over the Entity array that also manages the component stores.

```csharp showLineNumbers{36}
    public Handle<Entity> CreateEntity()
    {
        return _entities.Add(new Entity
        {
            World = this,
            Handle = _entities.PeekNextHandle()
        });
    }

    //****//

    public void Update(GameTime gameTime)
    {
        foreach (var store in _componentStores)
        {
            store.InitializePendingComponents();
            store.Update(gameTime);
        }
    }

    public void FixedUpdate(GameTime gameTime)
    {
        foreach (var store in _componentStores)
            store.FixedUpdate(gameTime);
    }

    public void Draw(GameTime gameTime)
    {
        foreach (var store in _componentStores)
            store.Draw(gameTime);
    }
```

Component.Start is deferred, it's called the next frame after it's added to the entity, before the update. Now here there is something interesting. As you'll soon see, `Entity` requires to store its own handle. To keep it more readable, I added `PeekNextHandle` to `HandleMapGrowing`, so I could assign the handle in the constructor before creating the entity. I could have done something like `var handle = _entities.Add(...); _entities[handle].Handle = handle` but that was awful to read. And also I wanted `Entity.Handle` to be readonly so I had to add that helper. Usually HandleMaps don't expose that, but it's my library so I can do it :D

`EntityRegistry` has more stuff, but before going to the Entity struct, you just need to know this:

```csharp showLineNumbers{56}
    public Handle<T> AddComponent<T>(Handle<Entity> entityHandle, T component) where T : struct, IComponent
    {
        component.World = this;
        component.EntityHandle = entityHandle;
        return GetStore<T>().Add(component);
    }

    public ref T GetComponent<T>(Handle<T> componentHandle) where T : struct, IComponent => ref GetStore<T>().Get(componentHandle);
```
All operations are done through this class. Entity will just use these

## Entities

This was the hardest part. And the one where the implementation fells apart a bit. But as a first attempt works fine.   

```csharp
public struct Entity
{
    public EntityRegistry World { get; init; }

    readonly Dictionary<int, Handle<IComponent>> componentIds = [];

    internal readonly Handle<Entity> Handle { get; init; }

    /// <summary>
    /// Whether this Entity is active in the hierarchy
    /// </summary>
    public bool ActiveSelf { get; set; }

    /// <summary>
    /// Whether this Entity is active in the scene
    /// Say, Entity A has a child Entity B. B could be active but A 
    /// be inactive, which would result in B <see cref="ActiveInHierachy"/> be false
    /// while <see cref="ActiveSelf"/> is true
    /// </summary>
    public bool ActiveInHierachy { get; internal set; }

    public Transform Transform;

    public Vector2 Position
    {
        get => Transform.Position;
        set => Transform.Position = value;
    }
}
```

It just has a reference to the world and a Handle to itself. I'd like to get rid of it, but you'll see why it's needed soon, in fact, now:

```csharp showLineNumbers{54}
    public Handle<T> AddComponent<T>(T component) where T : struct, IComponent
    {
        Handle<T> componentHandle = World.AddComponent(Handle, component);

        if (componentHandle is Handle<IComponent> icomponentHandle)
            componentIds[ComponentTypeId<T>.Id] = icomponentHandle;

        return componentHandle;
    }

    public Handle<T> GetComponentHandle<T>() where T : struct, IComponent
    {
        if (componentIds[ComponentTypeId<T>.Id] is Handle<T> handle)
            return handle;

        return Handle<T>.Default;
    }

    public ref T GetComponent<T>() where T : struct, IComponent => ref World.GetComponent<T>(GetComponentHandle<T>());
```

`Components` require a handle to its entity. That handle has to be stored somewhere. And as you see here, I'm using again `ComponentTypeId<T>.Id` so access the `HandleComponent`. Having a reference type in the `Entity` is not cool. I could avoid it by using Sparse Sets, the thing is that you need some data structure to map `Handle<Entity>` to `Handle<Component>`. I chose to instead have a list in the `Entity`. This structure only do allocations when adding new components or removing components, so it's not that bad. GetComponent is a bit slow, but you shouldn't do them anyways, you have to cache the `Handle<Component>` and check if it's valid before using.

Having this Dictionary is also needed as I need to clear the handles when destroying an Entity.

I'd like to get rid of the Handle, it's redundant, but I'm willing to pay that price so the rest of the system works.

I haven't benchmarked yet, so I can't say that this system performs better than the typical naive implementation where Components are classed store in a List in the entity, then you iterate each entity and from each entity, each component. But I'd expect that it performs better than that.

Finally. Accessing the Entity for stuff like transform it's common, so I did a extension for it. Agains, thanks to `PaperClip` for the idea

```csharp
public static class ComponentExtensions
{
    extension(IComponent component)
    {
        public ref Entity Entity => ref component.World.GetEntity(component.EntityHandle);
    }
}
```

# Example

This system lacks a lot of features, like physics implementation. But it's actually usable. Here's a demo code:

```csharp
using Darkrit;
using Darkrit.Base;
using Darkrit.EntityModel;
using Darkrit.Graphics;
using Darkrit.InputSystem;
using Darkrit.InputSystem.Bindings;
using Darkrit.Scenes;
using Darkrit.Utilities;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using GamepadButton = Microsoft.Xna.Framework.Input.Buttons;
using Key = Microsoft.Xna.Framework.Input.Keys;

namespace DarkritGame.Scenes;

[Renderable, Updateable]
public struct SpriteComponent : IComponent
{
    public EntityRegistry World { get; set; }
    public Handle<Entity> EntityHandle { get; set; }

    public bool Enabled { get; set; }

    public AnimatedSprite Sprite;

    public void Start()
    {
        Sprite.Scale = Vector2.One * 4;
    }

    public void FixedUpdate(GameTime gameTime) => Sprite.Update(gameTime);

    public void Draw(GameTime gameTime) => Sprite.Draw(Core.SpriteBatch, this.Entity.Position);
}

[Updateable]
public struct PlayerController : IComponent
{
    public EntityRegistry World { get; set; }
    public Handle<Entity> EntityHandle { get; set; }
    public bool Enabled { get; set; }

    public static bool Updateable { get; set; } = true;

    InputAction moveUp;
    InputAction moveDown;
    InputAction moveLeft;
    InputAction moveRight;

    Vector2 direction;
    private readonly float speed = 500f;

    public PlayerController()
    {
    }

    public void Start()
    {
        moveUp = Core.Input.CreateAction("Move Up").AddBindings([
            new KeyboardBinding(Key.Up),
            new KeyboardBinding(Key.W),
            new GamepadBinding(GamepadButton.DPadUp),
            new GamepadBinding(GamepadButton.LeftThumbstickUp),
        ]);

        moveDown = Core.Input.CreateAction("Move Down").AddBindings([
            new KeyboardBinding(Key.Down),
            new KeyboardBinding(Key.S),
            new GamepadBinding(GamepadButton.DPadDown),
            new GamepadBinding(GamepadButton.LeftThumbstickDown),
        ]);

        moveLeft = Core.Input.CreateAction("Move Left").AddBindings([
            new KeyboardBinding(Key.Left),
            new KeyboardBinding(Key.A),
            new GamepadBinding(GamepadButton.DPadLeft),
            new GamepadBinding(GamepadButton.LeftThumbstickLeft),
        ]);

        moveRight = Core.Input.CreateAction("Move Right").AddBindings([
            new KeyboardBinding(Key.Right),
            new KeyboardBinding(Key.D),
            new GamepadBinding(GamepadButton.DPadRight),
            new GamepadBinding(GamepadButton.LeftThumbstickRight),
        ]);
    }

    public void FixedUpdate(GameTime gameTime)
    {
        if (moveUp.IsPressed)
        {
            direction.Y = -1;
        }
        else if (moveDown.IsPressed)
        {
            direction.Y = 1;
        }
        else
            direction.Y = 0;

        if (moveLeft.IsPressed)
        {
            direction.X = -1;
        }
        else if (moveRight.IsPressed)
        {
            direction.X = 1;
        }
        else
            direction.X = 0;

        this.Entity.Position += direction.Normalized * speed * gameTime.Delta;
    }
}

public class TestSceneEntityModel : Scene
{
    EntityRegistry entityWorld;
    Handle<Entity> player;
    Camera camera = new();

    public override void Initialize()
    {
        TextureAtlas atlas = TextureAtlas.FromFile(Core.Content, "images/atlas-definition.xml");

        entityWorld = new(10);
        player = entityWorld.CreateEntity();
        ref Entity playerRef = ref entityWorld.GetEntity(player);
        playerRef.AddComponent<SpriteComponent>(new SpriteComponent
        {
            Sprite = atlas.CreateAnimatedSprite("slime-animation")
        });
        playerRef.AddComponent<PlayerController>(new PlayerController());
    }

    public override void Update(GameTime gameTime) => entityWorld.Update(gameTime);

    public override void FixedUpdate(GameTime gameTime) => entityWorld.FixedUpdate(gameTime);

    public override void Draw(GameTime gameTime)
    {
        Core.GraphicsDevice.Clear(new Color(32, 40, 78, 255));

        Core.SpriteBatch.Begin(samplerState: SamplerState.PointClamp, transformMatrix: camera.GetViewMatrix(Core.Viewport), rasterizerState: RasterizerState.CullNone);
        entityWorld.Draw(gameTime);
        Core.SpriteBatch.End();

        base.Draw(gameTime);
    }

    public override void EditorDraw(GameTime gameTime)
    {
        base.EditorDraw(gameTime);
        camera.EditorDraw();
    }

    public override void Deinitialize() { }
}
```

As you see, the system works and it's easey to use. Unlike using handles, here you have to explicitly resolve the handle to the type it points to, it's like you're doing the pointer jumping explicit, which I find interesting.

# End

I still have many things to add and polish in this system. I also have to test performance, but for now I'm happy. I haven't seen an implementation like this yet, and that can only be that I'm doing something terrible or that anyone tried this before. Or maybe there is a third option in which this is something common and I just don't know about it.

Whatever the case may be I'm pretty happy and I can continue to build upon it.

You might have wondered... How are you planning on building entity relations and update order dependency? Like, making an entity child of another, so parent components execute first than the children so the children gets the transform updated that frame. Answer is I don't know. I don't even know if I need it. I'd like to have hiearchies as they're useful for certain stuff, but for most 2D games I can live without it. I'll try to figure that out on the next version. Unlike this one that only took me a one day and a half, relations might take a lot more. I might not be able to implement them. If you're curious about that, be sure to check the IRS feed to not miss any post! 👀

Until next time!


[> Part 2](../Devlog/creating_a_entity_system_2.md).