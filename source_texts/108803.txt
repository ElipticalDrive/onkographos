# Voxel-Space-Rendering
[![build for linux](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/linux-build.yml/badge.svg)](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/linux-build.yml)
[![build for mac](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/build-mac.yml/badge.svg)](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/build-mac.yml)
[![build for windows](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/build-windows.yml/badge.svg)](https://github.com/artegful/Voxel-Space-Rendering/actions/workflows/build-windows.yml)

This is inspired by the original rendering idea of Voxel Space engine.

## Voxel Space
[Voxel Space](https://wikipedia.org/wiki/Voxel_Space) is voxel raster graphics rendering engine developed by Kyle Freeman.

## Showcase
![Gameplay (sort of)](https://github.com/artegful/Voxel-Space-Rendering/blob/gif/showcase.gif)

## How it works
Engine casts a ray for each pixels column on the screen. Ray makes small steps through the world reading mountain height data from color values of height_map.png and color from color_map.png. After calculating mountain fragment's height projection, engine draws a vertical line on the screen, taking occlusion into account. 
<p float="left">
  <img src="https://github.com/artegful/Voxel-Space-Rendering/blob/main/assets/color_map.png" width="400" alt="color map">
  <img src="https://github.com/artegful/Voxel-Space-Rendering/blob/main/assets/height_map.png" width="400" alt="height map">
</p>

## What has been used
- [SFML library](https://www.sfml-dev.org) for rendering
- [entt](https://github.com/skypjack/entt) for ECS and reactive system

## How to run
It is better to run a release optimized build:
```
make release && make run
```

To run with debug info:
```
make debug && make run
```
