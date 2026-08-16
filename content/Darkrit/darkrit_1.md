---
title: Darkrit devlog week 1
tags: ["c#", "darkrit", "monogame"]
---

I decided to split posts in week as daily progress will be slow and there won't be much to tell. That said, let's start:

# Day 1

I removed some tasks from the `Planned` list to the `Backlog`, among them: 
- FMOD Wrapper
- Asset packing
- Custom Sprite Batcher
- 2D shapes renderer
- Custom allocator
- Post processing system

Those are cool useful features... That I don't need now. I took time to think about the game I want to make. I really struggle with it since I'm not original in regards to game ideas. Given I never finished a platformer project, I decided to go with that.

That means that I need a collision system. I could use a Tile based system, but I prefer classic AABB collisions. Up until now, I've only used premade physics engine like Box2D. They are hard to control for "gameistics" physics, but it can be done. I'm already experienced with Box2D and have some cool unfinished prototypes with cool movement. But I won't do that. I decided that I want to keep dependencies as minimal as possible, so I will just write a simple AABB collision system.

Today was laying out the basics, I got the collision compute function and the responses, I also drafted the spatial grid class. Most of my work today was based on this [great article](https://gamedev.net/tutorials/programming/general-and-gameplay-programming/swept-aabb-collision-detection-and-response-r3084). What comes now will be purely mine. I have to admit that I feel a bit dumb for not figuring out the AABB swept formula alone, it's really simple and easy to understand but it took me a while to understand it, I'm rusty.

That said, I'm happy with how I'm planning this implementation. I will try a handle based approach so I don't need to move `RectangleF` structs here and there. This also allows me to keep everything as structs so iteration should be faster. I think this is what OOP haters call DOD, if not, please forgive me.

Next day I'll continue working on this system. I expect it to take me 10 days, which is a lot but it will be a nice collision system. And what I learn here will be useful for the `Tilemap` system.

# Day 2

I got stuck implementing AABB Swept, I ended up using [Falconers' implementation](https://github.com/Falconerd/engine-from-scratch/blob/rec/src/engine/physics/physics.c). The article I linked to yesterday was wrong on some aspects and its implementation was a bit more complex. I find this one easier to understand. Still, I had to admit that I spent hours just drawing on paper and manually executing cases to deeply understand how this works. It's really simple once you get but man I'm rusty...

I also build the handle map, but I haven't written tests and chances are there are bugs. Tomorrow I plan on building upon that and starting the spatial hashing

# Day 3

I started the implemenation of a GrowingArray for HandleBased array. I also started the World implementation. More on this other day

# Day 4

I got the basics of the physics system working. I wrote a dedicated post for it, [check here](../Devlog/creating_a_2d_collision_system_1.md)

# Day 5

I didn't do much. I changed the physics API to pass `Vector2 motion` instead of `Vector2 targetPosition` and I started to draft the SEC system. I won't do pure SEC, when I mean SEC I mean Scene-Entity-Component. A model really similar to the one Unity uses. Godot node system is interesting too, but it's harder to implement in a performant way.

I intend to build an OOP like system, that under the hood just runs ECS queries so it's efficient. I might not be able too. We'll know soon. This will likely take weeks.

# Day 6

Yesterday in bed, I had a revelation on how to approach the whole system. It didn't come out as nice as I imagined it, ergonomics aren't as good as I like but I'm happy I could get this done within 2 days. Same as with the physics system, I wrote a [dedicated post for it](../Devlog/creating_a_entity_system_1.md).

# Day 7

I continued working on the system and it's much better now. As usual, I wrote a [dedicated post for it](../Devlog/creating_a_entity_system_2.md).

I promise you that not all devlogs will just redirect to another post, but these last days have been wild.