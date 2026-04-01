---
title: Separating Data & Display
date: 2026-04-01
categories: [GameDev, Crawler]
tags: [godot]
---
I am a very silly man. 

First of all, the dungeon crawl jam is a week not a month. It also finishes at the start of April - not starting then! 
So the dungeon crawler idea is a bit of a failure. 
But as a silver lining, I have learned some things, and can pivot my experiments to a genre I have a bit more interest in - traditional roguelikes.

I'm not talking the new definition, where everything really seems to be bullet hell with random upgrades. I mean [ToME](https://store.steampowered.com/app/259680/Tales_of_MajEyal/).

Last time I wrote about handling movement quicker with multiple enemies on screen, while waiting for their tween to finish. 
I'm sure anyone that read it was screaming, because it goes against every piece of gamedev advice that states that data and visuals should be separated! 
I figured for a game jam I could intermix the two, but there's a very good reason the advice is not to. So I've reworked the movement again, this time making the state update immediate - creating and starting the tween to slowly animate but immediately moving to the next action. 

This has vastly simplified the walk action - and actions in general, since they've all removed the Enter() method.
So it boils down to:
```csharp
public override ActionResult Perform(double delta)  
{
	var rotatedDirection = direction.Rotated(Vector3.Up, Actor.Rotation.Y);  
	var targetGridPosition = Actor.GridPosition + new Vector3I((int)rotatedDirection.X, (int)rotatedDirection.Y, (int)rotatedDirection.Z);
	
	// Update the world immediately  
	ActorManager.Instance.SetActorPosition(actor, targetGridPosition);
	
	// But animate slowly if roughly on screen  
	var targetGlobalPosition = ActorManager.Instance.GetGlobalPosition(targetGridPosition);
	
	Actor  
	    .CreateTween()  
	    .TweenProperty(Actor, "global_position", targetGlobalPosition,  
	        GlobalSettings.StepSpeed * (Actor is Mob ? GlobalSettings.MobSpeedModifier : 1))  
	    .SetEase(Tween.EaseType.Out)  
	    .SetTrans(Tween.TransitionType.Cubic);
}
```

I haven't spent nearly as much time on this project as I'd hoped the last few weeks - instead I was spending all my time painting up a new warhammer army for a doubles tournament. I am Alpharius.

So that's it for now, next update will be once I've shuffled over to a more traditional roguelike approach - and hopefully it'll include some screenshots!
