---
title: Animated Movement Problem
date: 2026-03-10
categories: [GameDev, Crawler]
tags: [godot]
---
I've started working on a very-old-school style dungeon crawler. My intent is to participate in the [2026 Dungeon Crawler Jam](https://itch.io/jam/dcjam2026) this year, so I'm experimenting with how the core systems could work.

As with most gamedev stuff these days, Godot is my engine of choice. And it provides some super nice [Tween](https://docs.godotengine.org/en/stable/classes/class_tween.html) behaviour. I'm using the Command Pattern for actions in game - so there's an overall turn manager that asks each Actor what action it would like to perform, and they return something like:
```csharp
using Crawler.Scripts.Actors;  
using Godot;  
  
namespace Crawler.Scripts.Actions;  
  
public class WalkAction(Actor actor, Vector3 direction) : BaseAction(actor)  
{  
    private bool _isFinished;
    
    public override void Enter()  
    {        base.Enter();  
  
        var rotatedDirection = direction.Rotated(Vector3.Up, Actor.Rotation.Y);  
        var targetGridDirection = Actor.GetGridPosition() + new Vector3I((int)rotatedDirection.X, (int)rotatedDirection.Y, (int)rotatedDirection.Z);  
        var gridCellItem = ActorManager.Instance.GridMap.GetCellItem(targetGridDirection);  
  
        // Empty space we can walk into  
        if (gridCellItem == GridMap.InvalidCellItem)  
        {            var targetGlobalPosition = ActorManager.Instance.GetGlobalPosition(targetGridDirection);  
  
            Actor  
                .CreateTween()  
                .TweenProperty(Actor, "global_position", targetGlobalPosition, GlobalSettings.StepSpeed)  
                .SetEase(Tween.EaseType.Out)  
                .SetTrans(Tween.TransitionType.Cubic)  
                .Finished += () => _isFinished = true;  
        }        else  
        {  
            // Anything else just abandon immediately.  
            _isFinished = true;  
        }    }  
    public override ActionResult Perform(double delta)  
    {        if (!_isFinished)  
        {            return new ActionResult(ReturnValue.Ongoing);  
        }  
        return new ActionResult(ReturnValue.Success);  
    }}
```

So in this case, we simply use a tween to move one square in the specified direction. While the tween is executing we return telling the turn manager that we're still in progress.
Note that the tween speed is controlled by a global value, which I intend to expose in the game settings - that way people can tune the animation time to what best works for them.

However, this raises a significant problem - because the player character works like any other Actor, we have to wait for our turn to come around before we can issue our next command.
That means we have to wait for every single enemy in the turn queue to make their action (lets assume just movement for now)  - so every player turn then requires waiting (0.3 * # of enemies) seconds before the next action.

Unsurprisingly, this sucks.

For now, I've been looking at the [VisibleOnScreenNotifier3D](https://docs.godotengine.org/en/stable/classes/class_visibleonscreennotifier3d.html) as a way of simplifying things - we can skip animations if the enemy isn't visible. Note that it uses a bit of simplified positional logic, so it doesn't really track whether something is visible - our enemies could be behind a wall. But this is good enough for now.

So, as a simple implementation we can extend our Actor class so that it tracks whether it's visible or not:
```csharp
using Godot;  
  
namespace Crawler.Scripts.Actors;  
  
public partial class Mob : Actor  
{  
    private VisibleOnScreenNotifier3D _visibleOnScreenNotifier = null!;  
    // Whether the mob is _in front of_ the camera - this doesn't account for walls etc being in the way.  
    public bool IsVisibleOnScreen { get; private set; }  
      
    public override void _EnterTree()  
    {        
	    base._EnterTree();  
  
        _visibleOnScreenNotifier = new VisibleOnScreenNotifier3D();  
        _visibleOnScreenNotifier.ScreenEntered += OnBecomingVisible;  
        _visibleOnScreenNotifier.ScreenExited += OnBecomingInvisible;  
        AddChild(_visibleOnScreenNotifier);  
    }  
    
    public override void _ExitTree()  
    {
	    _visibleOnScreenNotifier.ScreenEntered -=  OnBecomingVisible;  
        _visibleOnScreenNotifier.ScreenExited -=  OnBecomingInvisible;  
        
        base._ExitTree();  
    }  
    
    private void OnBecomingVisible()  
    {
		IsVisibleOnScreen = true;  
    }  
    
    private void OnBecomingInvisible()  
    {
	    IsVisibleOnScreen = false;  
    }
}
```

Then we can add update our walk behaviour to exit early if the mob isn't visible:
```csharp
// Empty space we can walk into  
if (gridCellItem == GridMap.InvalidCellItem)  
{  
    var targetGlobalPosition = ActorManager.Instance.GetGlobalPosition(targetGridDirection);  
  
    if (Actor is Mob mob && !mob.IsVisibleOnScreen)  
    {        // Immediately teleport off-screen mobs, so the user doesn't have to wait.  
        Actor.GlobalPosition = targetGlobalPosition;  
        _isFinished = true;  
        return;  
    }  
    Actor  
        .CreateTween()  
        .TweenProperty(Actor, "global_position", targetGlobalPosition, GlobalSettings.StepSpeed * (Actor is Mob ? GlobalSettings.MobSpeedModifier : 1))  
        .SetEase(Tween.EaseType.Out)  
        .SetTrans(Tween.TransitionType.Cubic)  
        .Finished += () => _isFinished = true;  
}
```
This way any mobs out of sight can immediately teleport to their destination, while the ones on-screen show their movement animation.

I've been testing it out, and this feels significantly snappier, though we still have to wait for a while if there are several Mobs on screen.

For now this will work, but I might have to revisit this and look at whether I should completely decouple the logic & animations - that way all animations could play simultaneously, so we'd have a relatively fixed delay per 'round' of initiative rather than scaling based on the number of onscreen actors.

That's it for this week, but hopefully I'll keep making progress so there's something to share next week!
