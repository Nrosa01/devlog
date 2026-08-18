---
title: Darkrit devlog week 2
tags: ["c#", "darkrit", "monogame"]
draft: false
---

# Day 2

I just documented the code and cleaned it a bit. What I have to do now is pretty straightforward and not interesting so I don't think I will be doing a dedicated post about the next steps, at least until I get to hierachies. I also started writing unit tests and benchmarks against my `TinyECS` library to make sure that what I'm doing performs correctly.

That said, yesterday I did so many improvements in the `Entity Model` I couldn't write about everything. So today I will cove that in (you guessed it) a [dedicated post for it](../Devlog/creating_a_entity_system_3.md).

# Day 3

I just did some unit tests and minor improvements like making the generated `ref Entity` a `readonly ref Entity`. I also realized that the component has the entity handle exposed, that means you can modify the handle which you mustn't. But I can't do anything about it, given I'm the user there will be no issues. Also I can't make the getters internal for Darkrit given some components are generated on the game assembly.

Next updates will be slow, as I will implement hierachies which is the hardest part. Doing a logical hierachy in itself is easy, the hard part is guaranteeing the update order. Say that Entity B is child of A. Entity A components should run before Entity B ones, regardless of the component types. That's impossible right now as componentes are executed through the stores. There might not be an anwer, but I will still add hiearchies, even if they don't guarantee update order, they can help me to organize scenes into group, disable group of entities and similar stuff. The following days we'll see.