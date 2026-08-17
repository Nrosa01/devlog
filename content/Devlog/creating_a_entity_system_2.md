---
title: Creating an Entity System - Part 2
tags: ["c#", "monogame", "darkrit-entity-model"]
draft: false
date: 2026-08-16
---

This is the second part of a series, I recommend you reading [the first one first](../Devlog/creating_a_entity_system_1.md).

# The issue(s)

If you remember from last time, `Component`s are struct that implement `IComponent`. The issue is that for evey component I have to implement the same autoproperties all the time. If I add a new one to the interface, I have modify all the components to add it, which is not acceptable.

Most of this data, is `Component Metadata`, which means that I can just have something like:

```csharp {4}
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

There is a simple fix for all my problems: Source Generators and public/private API segregation. But before going into that, I will show an example and another problem that arises when doing something basic.

First of all, I will keep the componentMetadata. But that will only store private data, the component won't be able to access it, the ComponentStore will manage it. One example of this is `Initialized` field. When an entitiy is added, is not initialized, components initialize at the end of the next frame. My components aren't deferred, when you call `AddComponent`, it gets added to the array. Why? Remember that it's an array of struct that returns a handle. You usually want to:

```csharp
var entityHandle = world.CreateEntity();
ref var entity = world.GetEntity(entityHandle);
var componentHandle = entity.AddComponent<PlayerController>()
ref var component = entity.GetComponent<PlayerController>()
component.speed = 10;
// Note, there will be an api that returns the handle as out parameter Handle<> and a ref to the object, but I haven't done that yet
```

For this to work, I need the component to exist immediately. If this were an more OOP system, I could just store the class component instance and return it, given it's a reference, it would work even when I move it to the other array. This is harder with my system, so I have to add the component immediately. Given everything uses handles, even if the array resizes during the iteration, it will work properly.

The same problem applies when creating entities during any function callback.

That of course means that you can process a component that has just been added and not initialized. There are two ways to handle that:
- Initialize the component immediately
- Keep initialization deferred but flag the component as not initialize to not iterate it.

I could do the first and there won't be any issues thanks to the immediate mode. But the issue is that the component might or might not update depending on where we are in the iteration. Given I use a handle based array, I might be iterating component 40 of 50, but maybe component 27 is an empty position that wasn't being used and now the new component has been putted there.

And yes, this means that when you remove a component or an entity, the array size doesn't change, you still iterate the same amount of entities. And given I'm using handles, I can't reorder the entities, as that would invalidate the handles. This can be an issue under certain situations but not for the way I'm using this system, I accept this issue.

That settles me into the deferred options. I need a flag that must not be exposed in the `Component` interface

-------------------------------

Back to where we were, there are 2 main problems:
- I don't want to implement the interface properties for every component
- I need some component metadata that is not visible outside the library

For the second one, the answer is easy. Private data goes into the `ComponentMetadata` array. The `ComponentStorage<T>` uses it as needed. The `Component` struct only has public data.

For that last problem and the first one in our list, the answer is: Code generation.

# Source generation

I can make a source generator that implements the auto properties, that way I don't have to do so. And what is more, do you remember the `Updateable` attribute and the gang?

```csharp
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
```

With source code generation, I can also automatize that. I can even add fields that are ``[assembly: InternalsVisibleTo("Darkrit")]`` so the struct have them, allowing me to not need that `ComponentMetadata` array. But to keep things simple I'm still going with it.

This is not a tutorial about code generation, so I will just log here what I did and why. If you need to know about source generation, the discord C# server is a nice play to ask questions. Or reading [roslyn doc](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md)

Now, I will show the generated code, then I'll explain it:

```csharp showLineNumbers{23}
        context.RegisterSourceOutput(input, static (spc, data) =>
        {
            var component = data.Left;
            var iComponent = data.Right;

            var namespaceName = component.ContainingNamespace.ToDisplayString();
            var typeName = component.Name;

            var source = $$"""
                using Microsoft.Xna.Framework;
                using System.Runtime.InteropServices;
                using global::Darkrit.EntityModel;
                using global::Darkrit.Base;

                namespace {{namespaceName}};

                {{GenerateAttributes(component, iComponent)}}
                [StructLayout(LayoutKind.Auto)]
                public partial struct {{typeName}} : IComponent
                {
                    public ref Entity Entity => ref World.GetEntity(EntityHandle); 

                    public EntityRegistry World { get; set; }
                    public Handle<Entity> EntityHandle { get; set; }
                    public bool Enabled { get; set; }

                    {{GenerateMethods(component, iComponent)}}
                }
                """;

            spc.AddSource($"{typeName}.g.cs", source);
        });
```

I generate a partial struct that already implement IComponent. I use `[StructLayout(LayoutKind.Auto)]` so the compiler optimizes the layout, as using partial structs results in not having control about the padding and offset of the fields.

Now, `{{GenerateAttributes(component, iComponent)}}` generates `[Updateable]` and the rest of attributes automatically based on whether I implemented those callbacks. Let's say I added the `Update` function to my component, then it gets `[Updateable]`. This makes the system more robuts as previously you could easily forget one attribute. And they are opt-in by design, nothing executes unless the attribute is defined.

Then `{{GenerateMethods(component, iComponent)}}`. This is similar to the previous one, but it generates method implementations. I have been told and proved that doing

```csharp
public interface MyInterface
{
  public void MyFunction() {}
}
```

Boxes the struct, as `this` would be `MyInterface` and not the type that implements it.

To avoid the boxing, there are no default implementations in my interface now. And I didn't want to write empty functions in my components, so I just used the generator for that too. Totally needless, but it's the first time I'm using a Source Generator in C# so I wanted to push ergonomics to the limit. If I add more functions to the interface in the future, like OnEnabled, I won't need to modify my components nor the generator, it automatically picks unimplemented methods from the interfaces and implements them. Otherwise, modifying my API would require me to implement methods on all of struct. Sure you don't usually change a system API mid development, but this system itself is developing so I have to care about these details.

# Example

With all of that in place, let me show you how this looks now.

```csharp
// What I write
[Component]
public partial struct SpriteComponent
{
    public AnimatedSprite Sprite;

    public void Start()
    {
        Sprite.Scale = Vector2.One * 4;
    }

    public void FixedUpdate(GameTime gameTime) => Sprite.Update(gameTime);

    public void Draw(GameTime gameTime) => Sprite.Draw(Core.SpriteBatch, Entity.Position);
}

// What it gets generated
using Microsoft.Xna.Framework;
using System.Runtime.InteropServices;
using global::Darkrit.EntityModel;
using global::Darkrit.Base;

namespace DarkritGame.Scenes;

[FixedUpdateable]
[Drawable]
[StructLayout(LayoutKind.Auto)]
public partial struct SpriteComponent : IComponent
{
    public ref Entity Entity => ref World.GetEntity(EntityHandle); 

    public EntityRegistry World { get; set; }
    public Handle<Entity> EntityHandle { get; set; }
    public bool Enabled { get; set; }

    public void Update(GameTime gameTime) { }
}
```

Now I don't have to bother with properties, I just write what I need and it will just work.

# End

I still have to figure out relations, but I'm happy with this system, I can already build games with it and I know that I can just write components and everything will work really well under the hood thanks to the component storage.

There is something else I haven't written today, I've optimized the `Entity.GetComponent` and some other stuff, but I will have to write about that tomorrow as I'm running of awake time today.

Until next time!

[> Part 1](../Devlog/creating_a_entity_system_1.md).

[> Part 3](../Devlog/creating_a_entity_system_2.md).