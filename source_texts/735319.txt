# Cub3D

Cub3D is a simple 3D game engine developed in C using raycasting, inspired by *Wolfenstein 3D*. The project was built as part of the École 42 curriculum and focuses on real-time rendering, texture mapping, and smooth player movement.

## Features

- Raycasting-based 3D rendering
- Texture mapping for walls and objects
- Basic player movement (forward, backward, left, right)
- Collision detection
- Configurable map loading from `.cub` files

## Requirements

- GCC Compiler (or any compatible C compiler)
- Make
- A Unix-based system (Linux, macOS)
- MiniLibX library for Linux (included in the repository)

## Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/Selmand42/Cub3D.git
   cd Cub3D
   ```

2. Build the project:

   ```bash
   make
   ```

3. Run the game with a valid map file:

   ```bash
   ./cub3D path/to/map.cub
   ```

## Controls

- `W`: Move forward  
- `S`: Move backward  
- `A`: Strafe left  
- `D`: Strafe right  
- `Left/Right Arrow`: Rotate view  
- `ESC`: Exit the game  

## Map Configuration

The map should be provided as a `.cub` file, which includes:

- Map layout using `1` for walls and `0` for empty spaces
- Textures for walls (North, South, East, West)
- Floor and ceiling colors (RGB values)

**Example:**

```
NO ./textures/wall_north.xpm
SO ./textures/wall_south.xpm
WE ./textures/wall_west.xpm
EA ./textures/wall_east.xpm
F 220,100,0
C 225,30,0

111111
100001
1000N1
100001
111111
```

## MiniLibX for Linux

This repository includes the Linux-compatible version of the **MiniLibX** library, required for rendering and handling graphical elements on Unix-based systems.

## Acknowledgements

- Inspired by *Wolfenstein 3D*  
- Built using MiniLibX (included for Linux systems)
