# Pygame 3D Projection

A small personal experiment made out of curiosity about projecting 3D
points onto a 2D screen using Python and Pygame.

The goal was simply to try out the math and see how a basic projection
approach could be represented in code. This is a small prototype, not a
complete 3D renderer or game engine.

## What it explores

-   Representing 3D points with a simple `Vector3` class
-   Defining a basic camera with a field of view, range, and screen size
-   Using vector dot products and angles to reason about a point's
    position relative to the camera
-   Checking whether points fall within the camera's view and range
-   Converting projected screen aspects into pixel coordinates
-   Drawing the result with Pygame

## Project files

-   `utils.py` --- `Vector3` and `Camera` classes
-   `formulas.py` --- vector-angle, distance, and screen-coordinate
    helper functions
-   `projection.py` --- the point-to-screen projection experiment
-   `render.py` --- a small Pygame demo that draws a line between
    projected points

## Run

Requires Python 3.10+ and Pygame.

``` bash
python -m pip install pygame
python render.py
```

The current demo uses a fixed camera and a couple of fixed 3D points.
Press `Q` or close the window to exit.

## Status

This repository is a personal, curiosity-driven experiment. The
implementation is intentionally small and unfinished; it was published
to keep a record of trying the projection algorithm, not as a
production-ready graphics library.
