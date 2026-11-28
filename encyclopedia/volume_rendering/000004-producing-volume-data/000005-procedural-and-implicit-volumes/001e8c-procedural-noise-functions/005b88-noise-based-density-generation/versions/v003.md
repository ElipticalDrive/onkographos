# Noise-based Density Generation
## Overview
Noise-based Density Generation creates volumetric density fields by evaluating procedural noise functions at query points [G]. It solves the problem of defining complex, organic material distributions (such as fire, clouds, or smoke) without the memory overhead of storing explicit voxel grids [G]. It differs from its siblings by serving as the foundational mathematical primitive for procedural volumes, whereas specific implementations like [Perlin-Worley Noise](7139-perlin-worley-noise-for-cloud-density.md) or [Cartoon Cloud Density Generation](8309-cartoon-cloud-density-generation.md) apply specific compositional strategies to these primitives [G].

## Shared principles
This family shares the mechanism, terms, and math defined in [Procedural Noise Functions](1799-procedural-noise-functions.md).

## Subtopics

### [Procedural Volume Rendering with Noise Perturbation](1370-procedural-volume-rendering-with-noise-perturbation.md)
Enhances coarse volumetric data with high-frequency details using a tiled noise volume, solving missing microstructure without increasing voxel resolution. Sets itself apart by applying noise as a displacement field or optical modifier rather than a direct density scalar. When to choose it: Not given in the sources.

### [Noise-based 2D Fire](2697-noise-based-2d-fire.md)
Generates base density and color texture for volumetric fire using combined Perlin and Voronoi noise mapped to a 2D profile. Distinguishes itself by providing continuous, artifact-free textures supporting dynamic scaling without stretching. When to choose it: Not given in the sources.

### [Perlin-Worley Noise for Cloud Density](7139-perlin-worley-noise-for-cloud-density.md)
Generates cloud density by combining Perlin noise with Worley noise, storing multiple frequency levels of Worley noise in separate texture channels for high-frequency detail reconstruction. When to choose it: Not given in the sources.

### [Cartoon Cloud Density Generation](8309-cartoon-cloud-density-generation.md)
Generates stylized cloud densities using independent noise sets for macro structure, body puffs, edge details, and flow, combined with geometric envelopes. Differs by relying on multi-layered noise composition and explicit geometric shapes for a blocky/stylized appearance. When to choose it: Not given in the sources.

### [Line-Based Pipe Volume Generation](16974-line-based-pipe-volume-generation.md)
Generates 3D volume density along predefined line paths using OpenSimplex noise to create pipe-like structures, restricting the noise evaluation domain to the vicinity of parametric line segments. When to choose it: Not given in the sources.

### [Voxel Noise/Checkerboard Application](17946-voxel-noise-checkerboard-application.md)
Adds procedural noise to voxel materials, switching between a sampled noise texture and a deterministic checkerboard pattern based on a cell ID. When to choose it: Not given in the sources.

### [Noise Application](18254-noise-application.md)
Adds statistical noise to projection or volume data using either Poisson or Normal distributions, scaled by a norm factor or standard deviation. When to choose it: Not given in the sources.

### Geometry-Aware Noise Injection
A noise injection method that ensures multi-view consistency by assigning Gaussian noise to mesh vertices and rasterizing it, avoiding the view-inconsistency of 2D pixel-space noise [S1, unofficial]. Requires marching cubes extraction and rasterization, slower but captures stochastic details [S1, unofficial].

### Chunk Explosion Density Update
A parallel job that updates noise density values within a spherical region (explosion) in a chunk's noise map [S2, unofficial].

## Background
Not given in the sources.

## Sources
[S1] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
[S2] Fobri/Fast-Unity-Marching-Cubes, https://github.com/Fobri/Fast-Unity-Marching-Cubes, fetched 2026-10-26 (unofficial)
