# Raymarching Renderer

This is a renderer that uses Signed Distance Functions and Raymarching (not to
be confused with raytracing). SDFs can represent smooth surfaces, and they can
be blended smoothly together; this allows raymarching to render scenes that
would not be possible with the usual rasterization methods used in most
applications of computer graphics.

For example, raymarching can easily render smooth surfaces that blend together,
like liquids or soft solids such as ice cream.

![screenshot](raymarched_icecream.png)

this project uses the SDL2 library for opening a window and for handling input,
GLAD for loading OpenGL functions, and GLM for math.

Unfortunately, in its current state there are a few graphical artifacts and
visual bugs that occur in the scene at certain camera angles; these are
difficult to trouble shoot, but they will be fixed in the future.

## Building:

This project uses a simple makefile. All you need to do is clone this repo, and
in the main folder with the makefile, run:

```bash
make
```
and that should compile the executable. To run it, use:

```bash
./Raymarcher
```

## Controls:

The controls are very simple:

move - WASD keys

move faster - hold left shift

show the mouse cursor - escape

quit - q
