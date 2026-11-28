# Dust Simulation

An interactive 2D dust particle simulation built with **Python** and **Pygame**.

The project creates hundreds of glowing dust particles that drift through a dark environment, react to the mouse, and move using procedural turbulence and randomized motion. Multiple depth layers create a simple parallax effect to give the scene more visual depth.

## Features

* 800 simulated dust particles
* 3 depth layers for a parallax-style depth effect
* Procedural turbulence using layered value noise
* Mouse-influenced particle movement
* Random angular drift
* Particle fading and respawning
* Pulsing particle glow
* Additive blending for atmospheric lighting
* Seamless screen-edge wrapping
* Configurable particle size and speed
* Dark atmospheric background
* Runs at up to 60 FPS

## Technologies

* **Python**
* **Pygame**
* Standard Python libraries:

  * `random`
  * `math`
  * `sys`

## How It Works

Each dust particle is represented by a `DustParticle` object containing its position, depth layer, size, speed, color, lifetime, and procedural noise seed.

During each frame, particles are updated using several movement influences:

1. **Mouse attraction**
   Particles are influenced toward the current mouse position.

2. **Procedural turbulence**
   Layered noise generates irregular movement that gives the dust a swirling, wind-like appearance.

3. **Random drift**
   A small amount of random angular movement prevents the particles from following completely predictable paths.

4. **Depth scaling**
   Particles in different layers have different sizes and movement speeds, creating the illusion of depth.

5. **Screen wrapping**
   Particles that leave the screen reappear on the opposite side.

6. **Glow and pulsing**
   Particle brightness changes over time, while multiple translucent circles create a soft glowing effect.

## Configuration

The main simulation parameters can be changed near the top of the Python file.

```python
NUM_DUST_PARTICLES = 800
MAX_RADIUS = 2.5
MIN_RADIUS = 0.5
MAX_SPEED = 0.3
MIN_SPEED = 0.05
FADE_RATE = 0.001
WIND_INTENSITY = 0.002
NOISE_SCALE = 0.002
DEPTH_LAYERS = 3
GLOW_INTENSITY = 50
MOUSE_SENSITIVITY = 0.1
```

These values control the particle count, size, movement, turbulence, depth layers, glow, and mouse interaction.

## Controls

* **Move the mouse:** Influence the movement of nearby dust particles
* **Close the window:** Exit the simulation

There are no complicated controls because sometimes a project is allowed to simply have particles floating around looking nice. Humanity has suffered enough configuration menus.

## Running the Project

Make sure Python and Pygame are installed.

Install Pygame:

```bash
pip install pygame
```

Then run:

```bash
python dust_sim.py
```

## Project Type

This is primarily a **visual and interactive simulation project** rather than a physically accurate model of real dust.

The turbulence, glow, mouse attraction, depth scaling, and particle behavior are designed to create an atmospheric visual effect.

## Purpose

The project was created to experiment with:

* Particle systems
* Procedural noise
* Interactive motion
* Layered 2D rendering
* Pygame surfaces and blending
* Simple depth illusions
* Real-time animation

## License

Educational and personal project.
