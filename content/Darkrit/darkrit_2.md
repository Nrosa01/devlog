---
title: Darkrit devlog week 2
tags: ["c#", "darkrit", "monogame"]
draft: false
---

# Day 1

I just documented the code and cleaned it a bit. What I have to do now is pretty straightforward and not interesting so I don't think I will be doing a dedicated post about the next steps, at least until I get to hierachies. I also started writing unit tests and benchmarks against my `TinyECS` library to make sure that what I'm doing performs correctly.

That said, yesterday I did so many improvements in the `Entity Model` I couldn't write about everything. So today I will cove that in (you guessed it) a [dedicated post for it](../Devlog/creating_a_entity_system_3.md).

# Day 2

I just did some unit tests and minor improvements like making the generated `ref Entity` a `readonly ref Entity`. I also realized that the component has the entity handle exposed, that means you can modify the handle which you mustn't. But I can't do anything about it, given I'm the user there will be no issues. Also I can't make the getters internal for Darkrit given some components are generated on the game assembly.

Next updates will be slow, as I will implement hierachies which is the hardest part. Doing a logical hierachy in itself is easy, the hard part is guaranteeing the update order. Say that Entity B is child of A. Entity A components should run before Entity B ones, regardless of the component types. That's impossible right now as componentes are executed through the stores. There might not be an anwer, but I will still add hiearchies, even if they don't guarantee update order, they can help me to organize scenes into group, disable group of entities and similar stuff. The following days we'll see.

# Day 3

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