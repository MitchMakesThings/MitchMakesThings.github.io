---
layout: post
title:  "Networking Game Example"
date:   2023-02-19 12:00:00 +0000
categories: godot networking
---
With Godot 4 ready for prime time, I've cleaned up the networking project to show off a few approaches to the new networking nodes.

# Intro
Here's a quick preview of what the project looks like. This is running a spectating server (top left) with two clients.
![demo video of the project](/assets/2023-02-19/example.gif)

> You can get the [Whale Game project on Github](https://github.com/MitchMakesThings/Godot-Things/tree/main/Networking/Game)

# Building on an old project
This builds on an [older project](https://github.com/MitchMakesThings/Godot-Things/tree/main/Networking/Explained) that has an accompanying [youtube tutorial](https://www.youtube.com/watch?v=nQ4P3ogXp2Q) which explains the Jetpack and player networking. The rest of this blogpost will make more sense if you've seen that.

# Adding Mobs
I'm not sure where the phrase "Mobs" comes from, but it seems to be the going name for enemies in a game. I've made a very simple example to show off some of the networking techniques.
![Mob scene](/assets/2023-02-19/mob_scene.png)

## Setget variables
One cool feature we're using here are `setget` variables. These are special functions that are called when a variable is read from / written to. We use this to show a damage flashing animation whenever the mobs are hurt. The relevant code (stripped of comments!) is:
```GDScript
var health_sync := 3:
	set (value):
		var has_changed = value != health_sync
		health_sync = value
		if has_changed:
			flash_damage()
	get:
		return health_sync
```
We're using a `MultiplayerSynchronizer` node to send the health value from the server to all connected clients. It writes the `health_sync` variable for us, which calls the `set` method. Because `health_sync` is written every time the `MultiplayerSync` receives a packet, we need to check whether the value has actually changed. If it has changed we call the `flash_damage` method which uses Sprite Modulation to flash to indicate the mob was hurt. In a real project we _should_ check that health has been decremented, but in this project health never goes up!

## Node groups
Another handy feature is to put a node into a *Group*. Godot provides a handy method to fetch all the nodes in a group. We use this to randomly select a Mob to be killed when the in-game blue button is pressed.

To add a node to a group, you can use the Node editor.
![Node group](/assets/2023-02-19/node_group.png)

You can get all the nodes from a group with a simple call to:
```GDScript
get_tree().get_nodes_in_group("mobs")
```

And you could use that to randomly select and damage a mob:
```GDScript
var mobs = get_tree().get_nodes_in_group("mobs")
if not mobs.is_empty():
    mobs.pick_random().take_damage(100)
```

# Adding a gun
The other major feature added from the previous project was a weapon. This consists of a few parts, namely aiming and shooting.

## Aiming
We support both mouse and controller input. Every frame (_not_ physics frame!) we adjust the aim based on user input. This is stored in a `sync_weapon_rotation` variable.

For networked clients they simply update their local copy of a gun to match the synced rotation. This ensures all players see where other players are aiming.

Calculating the rotation for a controller makes use of the built-in `Input.get_vector` method:
```GDScript
var weapon_angle : float = 0
var controller_input : Vector2 = Input.get_vector("aim_left", "aim_right", "aim_up", "aim_down")
weapon_angle = controller_input.angle()
$WeaponParent.rotation = weapon_angle
```

The mouse rotation is even simpler:
```GDScript
var weapon_angle : float = 0
weapon_angle = $WeaponParent.get_angle_to(get_global_mouse_position())
$WeaponParent.rotate(weapon_angle)
```

Once the angle has been set, we need to remember to update the sync variable so that other network clients can update their display:
```GDScript
$Networking.sync_weapon_rotation = $WeaponParent.rotation
```

And that's pretty much all there is for aiming. The whole method can be viewed [on Github](https://github.com/MitchMakesThings/Godot-Things/blob/main/Networking/Game/Project/Characters/Aliens/Character.gd#L46)

## Shooting
Shooting a weapon has been pushed into the `Weapon` class. A weapon exposes a `Shoot` method which the `Character` calls when you press the shoot button.

This means that individual weapons can be written to handle their own shooting logic (Firing projectiles, swinging a sword, using raycasting etc) without needing to change our character script.

### Remote Procedure Calls
Unlike most of the project, which uses the NetworkSynchronizer node to achieve our networked behaviour, the weapons use Remote Procedure Calls (RPCs).

Shooting is handled via RPCs in a manner similar to what I remember of Unreal Engine.

1. The `Character` script calls the `Weapon`s `shoot` method.
1. An RPC triggers the `shoot_server` method on the server.
1. The server triggers an RPC to call `shoot_client` on all connected client.
1. A `shoot_impl` method implements the graphical and audio effects.
    - This is called on all systems, from different places:
        - The shooting client calls it immediately after the `shoot_server` RPC
        - The server calls it during `shoot_server`
        - All other clients call it during `shoot_client`

This is divided up so that the server can handle authoritative behaviour (Checking the weapon has ammunition etc). It also means clients can determine whether to show graphics (ie, don't bother with projectile graphics for weapons halfway across the map). It also makes it easier to follow what logic is running on which system - just look for the server/client postfix.

By breaking `shoot_impl` into its own method we've isolated the effects, and ensured that they behave consistently between all systems.

The full implementation can [be viewed on Github](https://github.com/MitchMakesThings/Godot-Things/blob/main/Networking/Game/Project/Items/Weapons/weapon.gd)

The key parts are the RPC annotations:
```GDScript
@rpc("any_peer")
@rpc
```
The best explanation I've seen is currently on [this blog post](https://godotengine.org/article/multiplayer-changes-godot-4-0-report-2/) - though note that the keywords passed to the RPC annotation now (at least from Godot 4 RC1) need to be strings.

In short, `@rpc` marks a method as callable via the network. It can be invoked in two ways.
1. Using the `rpc` method, the _network authority_ can call a corresponding method on connected clients.
    - The server uses this to call `shoot_client` on all connected clients:
    ```GDScript
    rpc("shoot_client", pos, rot)
    ```
1. Using the `rpc_id` method to call an RPC on a specific machine.
    - The server _always_ has ID #1
    - Our shooting client uses this approach to call `shoot_server`:
    ```GDScript
    rpc_id(1, "shoot_server", global_position, get_parent().global_rotation)
    ```

Note that the Godot only allows `@rpc` method calls from the _network authority_ - this will be the server by default. This is where the `any_peer` keyword comes in. Marking an RPC as `@rpc("any_peer")` means that _any_ machine can cause the method to trigger on _any_ other remote machine.
> Security warning: This means any connected client can trigger the method!

Since the `shoot_server` method can be called by any client, the first thing it does is checks that it's being called by the right client. If not it'll return early.
The `shoot_client` method doesn't have the `"any_peer"` keyword - so it can only be called by the Server!

# Outro
Hopefully this little blog post has taught you something! Thank you for reading.

If you want to see some more Godot stuff [I have a youtube channel](https://www.youtube.com/@MitchMakesThings)

I have a variety of Godot example projects [available on Github](https://github.com/MitchMakesThings/Godot-Things)