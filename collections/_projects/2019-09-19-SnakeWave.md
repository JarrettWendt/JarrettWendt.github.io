---
title: SnakeWave
author: Jarrett Wendt
excerpt: My first game made at the Florida Interactive Entertainment Academy.
thumb: assets/images/SnakeWave.png
sourceRepo: https://github.com/JarrettWendt/SnakeWave
category: projects
layout: post
cwd: '../'
---

Or play it right now on <a href="https://jarrettwendt.itch.io/snakewave" target="_blank">itch</a>!

<img src="{{ page.cwd }}{{ page.thumb }}" alt="SnakeWave" width="500" style="display: block; margin-left: auto; margin-right: auto;">

This was the first game I made at the Florida Interactive Entertainment Academy. It was also my first time working with an interdisciplinary team. I had carried out plenty of projects with other programmers, but never had I worked with artists and producers directly.

The team members were:
- Jarrett Wendt (myself): programmer
- Jonathan Baldessari: level design
- Ben Hsiao: helped with scripting some power-ups
- Rachel Morton: particle effects
- Michael Marte: background shaders

There were a lot of cool parts of this project, so let me get into them.

## Particle System
This was both myself and Rachel's first time working with Unity's Particle System, and boy did we learn a mouthful because this whole game basically runs on Unity's Particle System.

As you can probably tell, particles were used for the missile and laser power up as well as the player death explosion. A really handy thing about Unity's particle system is that it can detect collisions with regular RigidBodies. So I was able to set up death collisions between the players and any particle.

What might not be obvious that uses particles is the tail. We experimented with using a LineRenderer for the tail, but couldn't get the desired effect. With particles, it turned out beautiful and luckily we already had death collisions with all particles, so we didn't have to set up any extra logic for tail collisions.

If I could do this project over again, I wouldn't have used particles for the tail. When the players have lived a long time and they have really long tails on-screen there can be upwards of 50K particles on screen at a time, at which performance starts to dip. I'm very impressed with how well Unity is able to handle so many particles though

## Shaders
Michael did a great job with making the shader for the background. It was either of our first time working with Unity's shader graph system. The end result turned out just as dynamic yet subtle as we wanted.

Michael exposed as many variables into his shader as he could, giving me control to manipulate them in C#. At idle, the background simply rotates and gently cycles through colors. When a player performs an action such as picking up a power up, the background pulses. The idea here is to generate a sense of increased intensity and to give a visual queue to the other player who might not have been paying attention that their adversary has a power-up.

## Power-Ups
This was the most interesting part of the game and where I spent 80% of my time. Our game has 5 power ups:
- Boost: increases the player's speed for a short duration
- DropTail: detaches the player's tail, leaving it on the map as an obstacle which disappears after a while
- Laser: shoots a laser-gatling gun who's bullets ricochet everywhere
- Missile: fires a bomb which explodes with anything on contact
- Phase: makes the player invincible for a short duration

For all of these except Missile, I employed Unity's coroutines in order to deactivate them after a time.

Where these power-ups become complex is how they interact with each other and other elements of the game. Problems arose when a player would die while a power-up was still active, or picking up a new power-up while still receiving the effects of an old one. These cases were tricky to solve since they were often specific to certain power-ups and situations. They were also reminiscent of race conditions. Coroutines in Unity aren't _truly_ asynchronous (they all run in serial along with `Update()`) but even though Coroutines won't actually be running _at the same moment in time_ as other code, their order of execution is still not well defined.

This entire experience has brought me a whole new level of respect for games with many different unique types of interactions, such as Overwatch. At some point in that game's development, someone had to as: "What happens if Tracer plants her sticky bomb on Reaper just as he's teleporting?" and then I'll bet some developer had to put an if-statement somewhere to account for that specific scenario.

## Color Picker
<img src="{{ page.cwd }}assets/images/SnakeWaveColorPicker.png" alt="SnakeWave" width="800" style="display: block; margin-left: auto; margin-right: auto;">

We really wanted our players to be able to pick the color of their Snake. This would influence the color of their tail, particles, and UI. However, Unity doesn't have a built-in color-picker UI element and for this assignment we weren't allowed to use any asset packs. It might seem like overkill to make a whole color-picker UI just for a little 2-week game, but not for me.

I happen to work with colors _a lot_. Whether it be LEDs controlled through an Arduino, or parsing the pixel data of bitmap images, I seem to write the same RGB/HSV code over and over again. It's unfortunate that I can't ever re-use the code since the applications are usually wildly different or they're in entirely separate languages.

It turned out pretty simple. Just have a `Texture2D` hooked up to a UI image and on start iterate over all the pixels and set their color. The trick was figuring out what color to use based on the X/Y coordinate:

```c#
private Color ColorFromXY(Vector2 v)
{
	float halfHeight = texture.height / 2f;
	Color color = Color.HSVToRGB(v.x / texture.width, 1f, 1f);
	if (v.y > halfHeight)
	{
		color.SetSaturation(2f - v.y / halfHeight);
	}
	else
	{
		color.SetValue(v.y / halfHeight);
	}
	return color;
}
```

`SetSaturation()` and `SetValue()` are extension methods I wrote which call Unity's `HSVToRGB()` and `RGBToHSV()` in order to achieve the method's namesake.

The hue, as a float from 0 to 1, can be derived from only the x coordinate in relation to the total width. For the top half of the image, we increase the saturation as we go up. For the bottom half, we decrease the value as we go down. This creates a full HSV color picker in one UI element, rather than separate elements for each or separating one of saturation or value out to a slider.
