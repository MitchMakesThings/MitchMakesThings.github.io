---
title: Bump Attacks
date: 2026-04-21
categories: [GameDev, Crawler]
tags: [godot]
---
It's been a while, and truthfully I haven't really been working on this.
But I did make a little bit of progress (Ignore the jerkiness of a low framerate gif!)
![A progress update](/assets/gifs/ezgif-7f4bc4f613721159.gif)

As you can see, there's some bump attack actions added in - if an Actor tries to walk into another actor it'll perform a default attack.
I've also knocked together some very crude UI. It's come out pretty nicely decoupled.

# Decoupled UI
We've already got an ActorManager class - all our Actors register themselves from the `_Ready` method.
For the UI I've added a signal for the "Actor I care about" - ie the one we've hovered over.
```csharp
[Signal]
public delegate void OnActorSelectedEventHandler(Actor? actor);

public void SelectActor(Actor actor)
{
  EmitSignalOnActorSelected(actor);
}
```

Then, from my hero controller I can just call the `SelectActor` method whenever I want to announce that the UI should focus on an actor:
```csharp
ActorManager.Instance.SelectActor(actor);
```
Where `ActorManager.Instance` is a static variable set up from the `_EnterTree` method of the ActorManager, when it's instantiated as an autoload singleton.
I'm doing this from a raycast in `_PhysicsProcess`. If it collides with an Actor, we'll call `SelectActor`

Finally, I have a `InspectorUI` class that connects to the signal to update the display whenever needed.
The core parts are as follows:
```csharp
public partial class InspectorUi : Panel
{
	[Export] private Label _nameLabel;

	[Export] private Label _healthLabel;

	private Actor? _selectedActor;

	public override void _Ready()
	{
		base._Ready();
		ClearLabels();
		
		ActorManager.Instance.OnActorSelected += OnActorSelected;
	}

	private void OnActorSelected(Actor? actor)
	{
		if (_selectedActor != null)
		{
			_selectedActor.OnStateChanged -= OnSelectedActorStateChanged;
			_selectedActor.OnDeath -= OnSelectedCharacterDeath;
		}
		
		_selectedActor = actor;

		if (_selectedActor != null)
		{
			_selectedActor.OnStateChanged += OnSelectedActorStateChanged;
			_selectedActor.OnDeath += OnSelectedCharacterDeath;
		}
		
		UpdateLabels();
	}
}
```
Where we have a couple of labels that are exported and wired up in the editor.
Then in our `_Ready` we can connect to the actor manager to be notified whenever the user selects (or de-selects!) an Actor.
When that callback fires, we register for a few other signals on the Actor (so we can update the UI when they take damage or die).

So that's about it - some decoupled UI, some basic combat, and a waning interest in the project!

