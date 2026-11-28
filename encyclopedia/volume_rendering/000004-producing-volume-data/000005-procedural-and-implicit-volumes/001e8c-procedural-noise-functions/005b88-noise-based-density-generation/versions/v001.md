# Noise-based Density Generation

Also called: Procedural Noise Density, Noise Field Generation

## Overview
Noise-based Density Generation creates volumetric density fields by evaluating procedural noise functions at query points. It solves the problem of defining complex, organic material distributions (such as fire, clouds, or smoke) without the memory overhead of storing explicit voxel grids. It differs from its siblings by serving as the foundational mathematical primitive for procedural volumes, whereas specific implementations like [Perlin-Worley Noise](7139-perlin-worley-noise-for-cloud-density.md) or [Cartoon Cloud Density Generation](8309-cartoon-cloud-density-generation.md) apply specific compositional strategies to these primitives.

## Shared principles
The family shares the following mechanism, terms, and math:

1.  **Gradient Assignment**: Each integer lattice point (vertex) in the domain is assigned a pseudo-random gradient vector $\mathbf{g}$. In 3D, common sets include 12 face diagonals of a cube or 12 axis-aligned vectors .
2.  **Dot Product Contribution**: The noise value at a query point $\mathbf{P}$ is derived from the dot product between the gradient vector $\mathbf{g}_i$ at a lattice corner $i$ and the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ from that corner to $\mathbf{P}$:
    $$ n_i = \mathbf{g}_i \cdot \mathbf{d}_i $$
3.  **Smoothing/Fade Function**: To ensure smooth transitions ($C^1$ continuity), the fractional coordinates of $\mathbf{P}$ within the cell are passed through a fade function $f(t)$ before interpolation. The quintic function $f(t) = 6t^5 - 15t^4 + 10t^3$ is standard for Perlin variants, while Simplex uses a different geometric approach .
4.  **Interpolation**: The contributions from the cell corners are combined using linear interpolation (`lerp`) along the axes, weighted by the fade values .
5.  **Hashing**: Integer coordinates are mapped to gradient indices using a permutation table to avoid modulo operations and ensure uniform distribution .

## Subtopics

### [Noise-based 2D Fire](2697-noise-based-2d-fire.md)
Generates real-time procedural volumetric fire by combining a 2D fire profile texture with procedural noise functions (specifically Improved Perlin Noise and Voronoi Noise) to create micro-level flame details. It sets itself apart by using a lattice-based volume representation driven by noise for micro-interactivity, distinct from pure volumetric ray-marching.

### [Perlin-Worley Noise for Cloud Density](7139-perlin-worley-noise-for-cloud-density.md)
Generates base density for volumetric clouds by combining the structural continuity of Perlin noise with the billowy, cellular detail of Worley noise. It differs by using Worley noise to modulate the amplitude of Perlin noise, creating organic shapes with high-frequency billowiness at the edges.

### [Cartoon Cloud Density Generation](8309-cartoon-cloud-density-generation.md)
Generates cartoon-style cloud densities by combining independent multi-octave noise sets with geometric envelopes specific to cloud types. It differs from [Blocky Volumetric Clouds](12303-blocky-volumetric-clouds.md) by using independent noise sets for the base structure rather than a static 2D density map.

## Background
The underlying gradient noise algorithms (Perlin and Simplex) were introduced by Ken Perlin in 1985  and 2001  respectively to provide coherent random fields for computer graphics. These methods replaced earlier value noise techniques to eliminate grid-aligned artifacts. The application of these noise functions to procedural density generation for volumes like fire and clouds became standard in real-time rendering pipelines in the 2000s and 2010s, allowing for dynamic, infinite-resolution textures without storage costs.

## Sources
[S1] Ken Perlin, "An Image Synthesizer," SIGGRAPH 1985.
[S2] Ken Perlin, "An Image Synthesizer," SIGGRAPH 1985.
