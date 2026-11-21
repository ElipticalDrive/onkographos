# Data Structures and Implementation


## Overview
This family covers the data structures, generation algorithms, and memory management strategies required to produce procedural noise and voxel data for volume rendering. It addresses the problem of efficiently generating high-frequency random fields (white noise) and structured procedural patterns (gradient-based, Worley, etc.) either on the CPU for baking or on the GPU for real-time evaluation. It differs from sibling articles by focusing on the *implementation mechanics* (data layouts, RNG seeding, texture formats, and streaming structures) rather than the mathematical definition of the noise functions themselves.

## Shared principles
The family shares the following mechanism, terms, and math:

1.  **Gradient Assignment**: Each integer lattice point (vertex) in the domain is assigned a pseudo-random gradient vector $\mathbf{g}$. In 3D, common sets include 12 face diagonals of a cube or 12 axis-aligned vectors [S1, unofficial][S2, unofficial].
2.  **Dot Product Contribution**: The noise value at a query point $\mathbf{P}$ is derived from the dot product between the gradient vector $\mathbf{g}_i$ at a lattice corner $i$ and the distance vector $\mathbf{d}_i = \mathbf{P} - \mathbf{V}_i$ from that corner to $\mathbf{P}$:
    $$ n_i = \mathbf{g}_i \cdot \mathbf{d}_i $$
3.  **Smoothing/Fade Function**: To ensure smooth transitions ($C^1$ continuity), the fractional coordinates of $\mathbf{P}$ within the cell are passed through a fade function $f(t)$ before interpolation. The quintic function $f(t) = 6t^5 - 15t^4 + 10t^3$ is standard for Perlin variants, while Simplex uses a different geometric approach [S1, unofficial][S2, unofficial].
4.  **Interpolation**: The contributions from the cell corners are combined using linear interpolation (`lerp`) along the axes, weighted by the fade values [S1, unofficial][S2, unofficial].
5.  **Hashing**: Integer coordinates are mapped to gradient indices using a permutation table to avoid modulo operations and ensure uniform distribution [S1, unofficial][S2, unofficial].

## Subtopics

### [2D White Noise Texture Generation](2485-2d-white-noise-texture-generation.md)
2D White Noise Texture Generation creates a 2D texture filled with random values, typically used for noise-based effects such as SSAO variation or dithering in volumetric rendering. It differs from gradient-based or value noise by providing uncorrelated, high-frequency randomness without spatial coherence or smoothness.

### GPU Procedural Noise & Normal Computation
This approach trades computation time for memory efficiency by computing normals on-the-fly rather than storing them [S1, unofficial]. Bricks are created on the GPU using procedural noise, which is more efficient than CPU evaluation and transfer [S1, unofficial].

### World Generation Data Structure
A `WorldGenerationData` struct is used to store generation parameters, including `octaves: u32`, `ceiling: u32`, `lacunarity: f32`, `gain: f32`, `frequency: f32`, `threshold: f32`, and `seed: f32` [S2, unofficial].

### Two-rate Field Generation
This method evaluates continent- and range-wide octaves at lattice nodes and blends them [S3, unofficial]. For each level, a compiled column program is run instead of walking the octave table [S3, unofficial]. Erosion octaves step their corners and share steering [S3, unofficial]. A key pitfall is ensuring results remain bit-exact between CPU and GPU implementations [S3, unofficial]. Performance improvements reduced generation speed from 41–43 µs per tile to 32–34 µs per tile [S3, unofficial].

### Built-In Noise Generation System
A system that procedurally creates Volumetric Noise, bakes it to a 3D Texture, and associates the 3D Texture to the corresponding Unity volume component asset [S4]. Supported noise types are Perlin, Worley, Perlin-Worley, Billow, and Curl [S4]. Users can provide a Seed value for consistent results, define Quality to limit texture size, Invert results, and adjust Frequency, Octaves, Lacunarity, and Gain [S4].

## Background
Procedural noise generation has evolved from CPU-based texture baking to GPU-accelerated real-time evaluation. Early implementations focused on memory efficiency, using compact data structures like node pools and brick pools to minimize VRAM usage [S1, unofficial]. Modern systems often integrate noise generation directly into rendering engines or game engines (e.g., Unity, custom WebGPU raymarchers) to support dynamic, destructible, or streaming volumes [S2, unofficial][S3, unofficial][S4].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S3] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
[S4] Volume Noise Sources, https://docs.occasoftware.com/buto/volume-noise-sources, fetched 2026-10-27
