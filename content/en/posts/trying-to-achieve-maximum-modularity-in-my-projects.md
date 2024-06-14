+++
title = 'Making of Pixel Creator - A research on programmable automatas'
date = 2024-06-10T14:46:44+01:00
draft = false
toc = true
tocOpen = true
+++

<!-- Note: If you're reading this repo, don't take the post below seriously, I will actually do it better and with calm but give me time -->

# Introduction

This blog is mainly about [Pixel Creator](https://nrosa01.github.io/pixel-creator/), a little web game I built for my university thesis this year 2024. I encourage you to play it before readind this! And also try [Sandspiel Club](https://studio.sandspiel.club/), which is the game in which I based mine.

First of all I have to say both things. English is not my main language and my writting could be poor or contain grammatical errors. But I think I have more reach this way so I'm willing to pay that price, just be aware of that if you see something _unsound_ here. And the second one is that I didn't do all of this alone, I had a partner on this project. As I did most of the logic and research (he worked more on the GPU side) so I will be writing this alone. These last lines are just to make you aware that there was another person behind this who was important. Despite that, as this is my personal blog I will write it in first person from my own perspective.

# Falling Sand Games

First of all I should give a brief introduction to "falling sand" games. 

 | Powder Game                               | Powder Toy                          |
 | ----------------------------------------- | ----------------------------------- |
 | ![Powder Game](../2/images/powdertoy.png) | ![Powder Toy](../2/images/dust.png) |

These two games are [Powder Game](https://dan-ball.jp/en/javagame/dust/) and [Powder Toy](https://powdertoy.co.uk/). These games are little simulation sandbox where you have many particles that you can use to paint in the world. Each of this games is different but they are all make you wonder "What would happen if I mix this element and this other one?", "Can I do...?". Watching lots of little squares interacting between them is somehow pretty interesting and fun to play with.

But this doesn't mean all falling sand games must be simulation sandbox. My main motivation for this project was Noita.

![Noita Screenshot From Google](../2/images/Noita.png)

Noita is a roguelike where every pixel in the world is "simulated" (which is actually a lie but players won't notice it). It's a pretty amazing and hard game where you can break through the world, literally. You can destroy almost everything, you can mix elements... You are not a good like in the previous sandbox but you can enjoy a pretty rich, complex and dynamic world.

I want to do that too! When we started the project we aimed to make a Noita clone, but our priorities changed during development.

# Modularity

Given I've already written my thesis I don't want to spend many pages talking about "the state of art" so I will be kinda direct. I want to explain some specific things. If you, dear reader find this very poorly written or fast paced please let me know and I'll try to rework this little by little. Now let's go to more interesting and technical stuff.

We started our simulation in C++ and GLFW and we quickly hit a big obstacle. How should we define elements? Should we just hardcode everyone? But some elements share behaviour, there must be a way to abstract it... And that problem turned out so hard for us that, slowly, without realizing, we shifted our objective towards building a modular and easy to extend simulator rather than cloning Noita.

There is when I discovered Sandspiel:

 | Sandspiel                               | Sandspiel Studio                                   |
 | --------------------------------------- | -------------------------------------------------- |
 | ![Sandspiel](../2/images/sandspiel.png) | ![Sandspiel Studio](../2/images/sandspielclub.png) |

This was pretty cool! The first simulation was really performant and had an awesome fluid simulation. The second one was slower and didn't has that but it has a block based editor to **define your own elements**. And if this was not enough, there was a [blog](https://maxbittker.com/making-sandspiel) about its development.

I'm personally obssesed with modularity, up to the point where in the past I couldn't find my game projects because I always wanted to make everything fit perfectly with everything and thinking of "possible future needs". I'm aware of that issue of mine and I'm doing better now, but I still like to make everything as modular and easy to extend as possible, and this was my chance.

# Lua Simulation

My first step was adding Lua support to the C++ simulator. I've already worked with Lua previously so it was fairly easy. But... It was really slow. For that reason, I took the same decision as Sandspiel devs and I tried LÖVE, a 2D game framework for LuaJit. And that gave us a pretty neat performance improvement, but it wasn't enough, it was still slow. And apart from that, we realised that the simulation didn't look good.

Our particles has 2 properties: id and clock. Id allows us to know the type of element and clock allows us to not process the same particle twice or more in the same frame. The clock idea was taken from [the sandspiel blog](https://maxbittker.com/making-sandspiel).

![Lua Bias](../2/images/luabias.png)

# Rust Simulation