---
title: Starting a Dungeon Crawler
date: 2026-03-02
categories: [GameDev, Crawler]
tags: [godot]
---
# Time for a game jam!
It's been a while since I've done anything gamedev related, and for some reason I've recently been obsessing over old-school dungeon crawlers. So, with [Dungeon Crawler Jam 2026](https://itch.io/jam/dcjam2026) coming up next month, I thought I should make a dungeon crawler!

Until the Jam theme is announced I don't have any solid ideas on setting or theme, but I've been thinking through some of the stuff that I would like to have (in addition to the Jam requirements):
* Turn Based (No Grimrock here!)
* Party-based - though I'm still undecided on the party size.
* Fully 3D - I want to make myself get some Blender experience in.

To this end I've started prototyping some systems in [Godot](https://godotengine.org/) so that I have a starting point - just simple stuff like movement, interaction, turn scheduling (Don't worry, it all falls within the Jam rules!)

Since I'm aiming for 3D, [Grid Maps](https://docs.godotengine.org/en/stable/tutorials/3d/using_gridmaps.html) seem like a very natural starting point - though I'm not sure that I'll wind up using any of the collision/navigation support. I guess I'll be inheriting from it and adding additional support to help with movement (validating whether tiles can be moved into etc)

I was also recently reading [Bob Nystrom's A turn-based game loop blogpost](https://journal.stuffwithstuff.com/2014/07/15/a-turn-based-game-loop/) which has given me some inspiration on a starting point for turn scheduling. I think the Command Pattern makes a lot of sense. I'm less convinced by the energy system, though I've seen many roguebasin articles with similar ideas.
I think I would prefer to have a very obvious turn queue on screen, particularly so that the player could see exactly how long multi-turn actions (like channelling a spell) will do to the queue. Without diving deeper it's hard to plan how this would feel best.

Anyway, this is just a quick intro devlog to throw on the internet - and to give me some sort of accountability. I should post more of these as I noodle away on this project. Who knows, maybe by the end of April I'll have a full-fledged if very simple game, and a cool devlog documenting the making of.
