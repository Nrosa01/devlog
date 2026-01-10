---
title: Darkrit Engine
tags: ["showcase"]
---

A 2D game engine built on top of LÖVE (Love2D) framework with Lua. Heavily inspired by Unity but adding unique ideas. The engine is kinda rough, it has a lot of features, you can perfectly make a game with sounds, effects etc. But still lacks basics like material support that you have to handle on your own. [Source code here](https://github.com/Nrosa01/darkrit_engine)

> [!Warning] Warning!
> The engine hasn't been updated in a while, it might not receive support anymore. Use at your own risk!

In the next section, the engine features will be enumerated. Some might be missing since I haven't worked on this project for almost a year and only now I'm writting this after taking a quick glance to the code.

> [!Info] Info
> Be aware that while this engine draws strong inspiration from Unity, it diverges significantly in its core design philosophy. You’re encouraged to use the built-in scene system as a base and build yours on top. The engine relies a bit on it.

Within this structure, entities reside in the ``World``, but that doesn’t encompass the entire scene itself. Instead, you’ll need to define your world alongside a dedicated UI subsystem—an arrangement you can customize freely. The goal is enabling seamless communication: the World should emit signals detectable externally, allowing the UI to respond independently of the game logic. This decoupling ensures the UI remains a flexible, interchangeable component.

This system supports multiple worlds within a single scene, making it highly adaptable for complex simulations such as bullet trajectory tracking. Ultimately, it’s a modular architecture that grants you full creative control over how elements interact and connect. In short, think of this setup as a versatile foundation where every piece can be structured and positioned exactly as needed.

At the end of this post I'll put some short demo videos I recorded long ago.

## Features

### Entity System
- Custom Entity-Component System (not ECS). Attach components that has callbacks like Unity to your entities
- Entity groups for flexible organization. Similar to godot tags but more efficient
- Render order control per component. Support for y-sorting with a custom algorithm to sort sprites
- Execution order control per component. Similar to Unity script order execution
- Parent-child entity hierarchies
- Frame-based deferred entity creation and deletion for safe iteration
- Custom pivot for the sprite component

### Physics System
- Box2D integration through Love2D
- Slick physics integration
- Easy physics initialization with `entity: init_physics()`
- Unified transform API for both regular and physics-enabled entities
- `PhysicsTransform` wrapper that seamlessly integrates with Box2D bodies and slick bodies
- Physics utilities: 
  - `accelerate_to()` - Smart acceleration with max limits. Useful for platformer control
- Support for dynamic, static, and kinematic bodies
- Automatic shape scaling (recreates fixtures since Box2D doesn't support native scaling)

### Graphics System
- Flexible camera system with multiple scale modes: 
  - None
  - Letterbox
  - Pixel Perfect
  - Stretch
  - Stretch with aspect ratio (boundless)
- Camera boundary constraints with polygon collision detection
- Advanced coordinate conversion (screen ↔ world)
- Configurable pivot points (9 preset positions)
- Sorting layers system with Y-sorting support
- Transform system with position, rotation, and scale
- VSync control (disabled, enabled, adaptive, half refresh rate)
- Filter modes:  nearest (pixel art) or linear

### Asset Loading
- On-demand asset loading with **bjornbytes/cargo**
- Automatic Aseprite file processing (exports to PNG + JSON)
- Animated sprite support with **peachy** library (removed cron to improve performance)
- Script caching system
- Organized asset paths (sprites, scripts, components)
- Auto-detection and execution of Aseprite CLI

### Loading System
- Full-featured loading screen management
- Configurable transition phases: 
  - Start transition
  - Loading progress
  - End transition
- Custom draw callbacks for each phase
- Frame time limiting to prevent blocking
- Optional loading during transitions
- Progress tracking with custom data passing

### Input Management
- Built on **tesselode/baton** for robust input handling
- Multiple input maps support
- Unified API for keyboard, gamepad, and mouse
- Per-action input mapping
- Easy context switching between input schemes

### Rendering
- Automatic sprite batching
- Customizable sorting layers (BACKGROUND, ENVIRONMENT, CHARACTERS, OVERLAY, UI). Create your owns!
- Frame skip configuration for Y-sorting optimization
- Render order control per component
- Resolution-independent rendering

### Algorithm Library
- **Adaptive In-Place Sort** - Hybrid merge/insertion sort (threshold:  32)
- **Turbo Adaptive Sort** - Aggressive merge sort (threshold: 64)
- **Insertion Sort** - Classic algorithm for small/nearly-sorted arrays
- **Counting Sort** - Integer sorting with sparse hash optimization
- All algorithms optimized for LuaJIT

### Math & Utilities
- **nvec** vector library integration (`Darkrit.vec`)
- 2D vector operations (add, subtract, multiply, normalize, length)
- **bjornbytes/tick** for frame-independent timing with minor modifications
- Built-in OOP class system, lightweight with basic inheritance and mixing support
- Require utilities for organized module loading

### Configuration
- Central `darkrit_config. lua` for engine settings
- Override support for project-specific configs
- Customizable paths: 
  - Assets directory
  - Scripts directory
  - Components directory
- Graphics settings:
  - Game resolution (default: 320x180)
  - Fullscreen mode and type
  - Window resizing
  - VSync options
  - Filter mode
  - Y-sorting configuration

### Developer Tools
- Built-in test system
- Debugger integration (auto-loaded)
- Entity system unit tests included
- Component type flags for optimization (COMPONENT, UPDATEABLE, DRAWABLE)

## Dependencies

- **bjornbytes/cargo** - Asset management
- **bjornbytes/tick** - Time management
- **tesselode/baton** - Input handling
- **NPad93/nvec** - Vector math
- **josh-perry/peachy** - Aseprite animation

Most dependencies might have been modify to suit the engine needs.

> [!Info] Info
> Most dependencies might have been modify to suit the engine needs. For example I modified `peachy` to execute callbacks defined from aseprite. Useful for syncing steps sound with your animation, among other tasks.

## Demo videos

Layer System

![](LÖVE/assets/layer_system.mp4)

Collider resizing

![](LÖVE/assets/engine_collider_scale.mp4)