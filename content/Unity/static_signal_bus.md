---
title: Simple static signal bus
draft: false
description: In this post, a simple and powerful type-safe signal bus system is presented. Usage and tips are included.
tags: ["tutorial", "unity"]
---

# Introduction

First of all, a little warning. This is not an original idea of mine. The code and ideas shown here are from [this repo.](https://github.com/Extrys/LightWeightSignalBuses) I've used this on all my Unity projects so it's worth showing it here.

> [!Note] Note
>
> The linked repo contains to systems: ``LightWeightSignalBus`` and `InterfaceCompatibleSignalBus`. In this post only the former one is explained.
> Also note that while I focus this to use in Unity, it uses vanilla C# features, it can be used in any C# project that supports Actions.


---------------------------------------

The [observer](https://www.gameprogrammingpatterns.com/observer.html) popularized by the Gang of Four eventually lead to similar patterns: [Signal Hubs](https://theduriel.github.io/Godot/Do-not-use---Signal-Hub), Command Queues, Event Systems... Despite them all being different, the code idea is the same: Allowing multiple endpoints to react to a data change.

# The Signal Bus

The following system is simple yet powerful, too much even. It should be used sparingly. For example, achievement tracking, data collecting... Let's say you want to track how many times your player died, how many times it jumped... There are many ways to implement that, this is one of them. Let's see the code.

```csharp
using System;
using System.Collections.Generic;
using UnityEngine;


/// <summary>Lightweight version of SignalBus</summary>
/// <typeparam name="T">Type of signal</typeparam>
public static class SignalBus<T>
{
	static Action<T> action;

	public static IDisposable Subscribe(Action<T> action)
	{
		SignalBus<T>.action += action;
		return new Subscription(action);
	}

	public static void Fire(T t) => action?.Invoke(t);

	public static void Unsusbscribe(Action<T> action) => SignalBus<T>.action -= action;

	//Subscription registered for Lightweight version
	public class Subscription : IDisposable
	{
		Action<T> boundAction;
		public Subscription(Action<T> boundAction) => this.boundAction = boundAction;
		public void Dispose() => Unsusbscribe(boundAction);
	}
}
```
The idea here is creating structs that will be our signals, subscribing to them and reacting to firing events. And even tho a `Unsubscribe` function is provided, I discourage its usage. It's recommended to use the IDisposable interface.

## The CompositeDisposable

The linked repository provides a [CompositeDisposable](https://github.com/Extrys/LightWeightSignalBuses/blob/main/Extra/CompositeDisposable.cs) object that comes really handy. The code is the following.

```csharp
using System;
using System.Collections.Generic;

/// <summary>
/// A simple list of disposables. Its made for adding disposables at the end of their build pattern, making the code cleaner and easier to read<br/>
/// mainly used for <see cref="DisposableExtensions.AddTo(IDisposable, CompositeDisposable)"/>
/// </summary>
public class CompositeDisposable : IDisposable
{
	List<IDisposable> disposables = new List<IDisposable>();

	public void Dispose()
	{
		int iterations = disposables.Count;
		for (int i = 0; i < iterations; i++)
			disposables[i].Dispose();
	}

	public void Add(IDisposable disposable) => disposables.Add(disposable);
}
```

## DisposableExtensions

In order to define Signals as one-liners, an [DisposableExtension](https://github.com/Extrys/LightWeightSignalBuses/blob/main/Extra/DisposableExtensions.cs) is provided too.

```csharp
using System;

public static class DisposableExtensions
{
	/// <summary> Adds disposables at the end of their build pattern, making the code cleaner and easier to read </summary>
	public static void AddTo(this IDisposable disposable, CompositeDisposable compositeDisposable) => compositeDisposable.Add(disposable);
}
```

# Usage example

## Signal Definition

```csharp
// Signals.cs

struct SignalCharacterDied { int id; }
struct SignalPlayerJumped {}
```

## Subscription and Disposal

```csharp
//StatisticsManager.cs

public class StatisticsManager : MonoBehaviour
{

    CompositeDisposables disposables = new();

    private void OnEnable() {
        SignalBus<SignalCharacterDied>.Subscribe(OnCharacterDied).AddTo(Disposables);
        SignalBus<SignalPlayerJumped>.Subscribe(OnPlayerJumped).AddTo(Disposables);        
    }

    private void OnDisable() => disposables.Dispose(); // Disposes both signals

    private void OnCharacterDied(SignalCharacterDied context) { /***/ }
}

```
## Firing

```csharp
//Player.cs

public class Player : MonoBehaviour
{
    private void TakeDamage(int damage) {
        health -= damage;

        if(health <= 0) {
            SignalBus<SignalCharacterDied>.Fire(new SignalCharacterDied() { id = 0 });
        }
    }
}

```
# End

And that's it! Now you can use a simple static type-safe signal bus in your Unity Projects

> [!tip] Tip
> 
> You can fire signals without data without having to create the object explicitly by doing this small modification in the library.
>```csharp
>//From
>	public static void Fire(T t) => action?.Invoke(t);
>
>//To
>	public static void Fire(T t = default) => action?.Invoke(t);
>```
>
> Example:
>
>```csharp
>// Signals.cs
>struct MySignal {}
>
>// OtherScript.cs
>SignalBus<MySignal>.Fire();
>```