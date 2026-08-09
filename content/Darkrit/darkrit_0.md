---
title: Darkrit devlog - Why another engine
tags: ["c#", "darkrit", "monogame"]
---

# Motivation

The past month I decided to embark on a new engine adventure. I had been thinking about it for a long while. I've wanted to avoid it because I know myself. I love working on tooling and even if I use agile, I will find excuses to build tools to *aid* in the game development. It would be a fun journey, but also a long one

So what happened now? I finished a game in Godot. I finally did what I failed to do after many years. It's not as good as I wanted, but it's done. At the time of writing this it hasn't released, that will be the next month. But the hard part is done.

There is another reason for this, it's not just that. But I found myself that, although Godot is a great engine, it's not fully for me. Over years I have struggled to do some things on a certain way and not being able to do so **easily**. Both Godot and Unity are incredibly flexible and mature. But often they impose paths I don't like. They also allow me to procrastinate on learning linear algebra since they already handle Transforms, Matrices, Quaternions and all of that fancy math.

I have been asked many times: *Is it your goal to make games or to make an engine?*. And my answer is always the same: **Both**. I want to make games, but I want them to be made on a certain ways, they have to meet certain criteria. I can't stand slow or bloated software. At the timing of writting, Godot default export is 70MB. Unity is around 50MB. Not too much, but more than it should. I one of those people obssesed with optimization. That's why my previous attempt was with [Love2D](../LÖVE/darkrit_engine.md). But that's the reason I left it too. Binary was tiny and performance was great but not perfect for handling big amount of entities, which I will probably do at some point. All of these criteria is not really that important to make a game, but I just feel bad if I don't try to squeeze the CPU.

The hard truth is that even if I want, I can't realistically. I have a 8 hours job, a boyfriend, responsabilities, other hobbies... I just don't have the time to build something that big, or at least, **not now**. More on that later.

# MonoGame

The language and framework was a tough decision. I wanted to go as low level as I my time and patience allowed. There were only two sensible options: C# and Odin. I'm already experienced with C#, which adds a lot of points. I'm not with Odin, but I already build an [Interpreter](https://github.com/Nrosa01/OdinLox) with it. It's a simple, fun and powerful language, data oriented, with nice vendored libs for gamedev. And it supports reflection too!

But after trying SDL_GPU, Sokol, BGFX and more I ended up going with C# and MonoGame. I think it's the best environment I can use: MonoGame gives enough control over the game loop and rendering, even if it doesn't support bleeding edge features. I first need to master the basics. C# allows me to iterate FAST. Sure the code won't be as performant, but since it has [Native AOT](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/?tabs=windows%2Cnet8) that's not really an issue. Being able to use `Interfaces` is quite nice, in Odin I would have to create my own vtables which takes more time. Apart from that, msbuild is pretty nice to me, I can even do runtime code generation like this

```xml
	<PropertyGroup>
		<EditorGeneratedPath>$(MSBuildProjectDirectory)\Editor\EditorProjectPath.g.cs</EditorGeneratedPath>
	</PropertyGroup>

	<Target Name="GenerateEditorProjectPath" BeforeTargets="CoreCompile">
		<MakeDir Directories="$(MSBuildProjectDirectory)\Editor" />

		<WriteLinesToFile
			File="$(EditorGeneratedPath)"
			Overwrite="true"
			Lines="namespace Darkrit.Editor%3B; internal static class EditorProjectPath { public const string Root = @&quot;$(MSBuildProjectDirectory)&quot;%3B }" />

		<ItemGroup>
			<Compile Include="$(EditorGeneratedPath)" />
		</ItemGroup>
	</Target>
```

I already have some cool features in the engine:
- InputSystem with Recordable/Replayable Actions
- Hot Reloading of assets, including shaders
- Simple Editor system based on ImGui
  - Profiling windows
  - Scene switcher windows
  - Layout save and load
  - Editor data export and import
  - Toggable Viewport
- Physics Interpolation
- Camera
- StringIDs
- Texture, Sprite and other basics classes
- Test Suite
- Custom Sprite Batcher (WIP)
- Simple Instanced Renderer
- Tiny ECS (that I won't use)
- Probably something more that I haven't documented yet

And all the code is properly documented. The more time it passes the more I appreaciate how important is doing so.

# Goals

My current plan is building a small 2D engine and grow it slowly by making little games. From my past experience, I'd like to think that I've improved at game scoping and planning. Enforcing agile works well for me because it doesn't allow me to divert too much. I will build only what I need for a given game. Some of that code will be common for the library and some other game specific. I will also limit the amount of times I can invest in tooling each sprint so I actually make progress in the game while the tools are getting done as to not get stuck as always.

MonoGame for 2D exposes all I need from the ``RenderingDevice``. But I know myself, once I manage to get an environment in which I can make 2D games comfortably and fast, I will want to try 3D. And it can perfectly be done in MonoGame, but I will want to have a bit more of control, access features like `Structured buffers` and so.

I have a plan for that, and it's just migrating the renderer code to `SDL_GPU`. From all the APIs I've tried, it is the one I like the most, along with `BGFX`. I like to keep dependencies small and as a minimal as possible. Since MonoGame already uses `SDL` for some of it's backend, I find nice using it too. And I can use it in C# so I wouldn't lost all that I have already build.

I only worry that I might regret using C# and not Odin in the future. If it were for me, I'd do an Odin implementation in parallel but if I want to release games I can't do that.

# Wrapping up

I want to do daily devlogs, or maybe weekly on the progress I do. I want to document everything mainly for future reference for myself and because it might help someone. Most of what I know is thanks to people who share they knowledge and code online.

I would have liked to start this devlog earlier but I forgot I had this site until now. Hopefully I can keep up with it.

Until the next post!