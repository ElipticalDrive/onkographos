# Ray Tracer

A recursive ray tracer implementing physically-based light simulation, written in C++. Built as part of CENG 477 — Introduction to Computer Graphics at METU.

## Overview

Given a 3D scene described in XML, the program simulates light propagation and produces a 2D raster image in PPM format. No GPU or graphics API is used — the entire ray tracing pipeline runs on the CPU.

The project was built on a course-provided scaffold (XML parser, PPM writer). The ray tracing logic is implemented in `raytracer.cpp`.

## Features

- **Ray-object intersections** — Sphere, triangle, mesh, plane, and cylinder
- **Shading** — Blinn-Phong model with ambient, diffuse, and specular components
- **Shadow rays** — Point light occlusion with epsilon offset to avoid self-intersection
- **Mirror reflection** — Recursive ray casting up to a configurable maximum depth
- **Multiple cameras** — Each camera produces a separate output image
- **Multiple light sources** — Ambient light and multiple point lights with inverse-square falloff

## Build & Run

```bash
make raytracer
./raytracer <scene.xml>
```

Output is written as `.ppm` files, one per camera defined in the scene. To convert to PNG:

```bash
convert output.ppm output.png  # requires ImageMagick
```

## Input Format

Scene files are in XML and define background color, shadow epsilon, recursion depth, cameras, lights, materials, vertices, and objects (meshes, triangles, spheres, planes, cylinders).

## Tech Stack

- **Language:** C++
- **Build:** Make
- **XML Parsing:** tinyxml2
- **Output Format:** PPM

## Author

**Doğu Erbaş**  
doguerbass@gmail.com  
METU Computer Engineering — CENG 477
