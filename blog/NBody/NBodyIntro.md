---
title:  "N-Body Simulation (Introduction)"
permalink: "blog/NBodyIntroduction"
layout: post
---

Hello! This will be my first blog post on a project called N-Body Simulation. I would like to share what I have learned so far while working on this project. 

## N-Body Problem

According to [Wikipedia](https://en.wikipedia.org/wiki/N-body_problem), N-Body Problem is "the problem of predicting the individual motions of a group of celestial objects interacting with each other gravitationally". The objective of this project is to create a particle simulation where each particle will be representing as a "celestial object" with properties of location, velocity, and attraction force. I will be creating the simulation using different methods like WebGPU, Web Workers, and QuadTree and compare their performance value. (FPS) 

Just to talk little bit more on N-Body Problem, let's first see the Two-body problem, where `n = 2`. 

![Alt Text](https://media.giphy.com/media/lMFyRfKDAcEdwA8VBw/giphy.gif)

[Source](https://en.wikipedia.org/wiki/Two-body_problem)

As you can see from the above GIF, there is a consistent pattern of how two objects orbit around. This means that if we want to know the location of these two objects at certain `t` value, the location will be consistent throughout series of simulation as long as the intial value of properties are consistent.

However, when `n >= 3` (where it starts to get called as N-Body Problem), we will actually be able observe a chaotic 

![Alt Text](https://media.giphy.com/media/dq8BsHPX7qYsGEolW0/giphy.gif)

[Source](https://vimeo.com/11993047)

When there are more than 3 objects, there is no longer a consistent pattern of how these objects move. This means that the location of each object at certain `t` value will be different among the series of simulation, resulting in a chaotic motion. Just a note: I am not an expert in astrodynamics, so if any of the information that I present is incorrect, please feel free to contact me and let me know! 

So now, let's see how we can create a simulation of N-body problem. Here is the general structure of the code: 
1. First store the properties of particles with initial value of location and velocity. (This will be random)
2. Traverse through each particle and calculate its next location and velocity based on other particles' location.
3. After going through all the particles, render all the particles with their new location.
4. Repeat 2 and 3.

The main issue here is that Step 2 requires a for-loop (to iterate all the particles) and a nested for-loop (to go through all particles excluding the current one), resulting in `O(n^2)` time complexity. Since I'm aiming to create a simulation with hundreds, or even thousands of particles, coming up with a more efficient method on Step 2 will be the main task of this project. 
