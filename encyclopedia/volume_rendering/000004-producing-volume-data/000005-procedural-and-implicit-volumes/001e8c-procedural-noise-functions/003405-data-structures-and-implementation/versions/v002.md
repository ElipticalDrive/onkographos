# Data Structures and Implementation

Also called: Procedural Noise Data Structures, Noise Texture Generation, Voxel Data Layout

## Overview
This family covers the mechanisms for storing, generating, and accessing procedural noise data used in volume rendering, ranging from pre-computed 2D/3D textures to on-the-fly GPU evaluation. It differs from sibling articles by focusing on the **storage format** (e.g., `GL_R8` vs `GL_RGB32F`), **generation pipeline** (CPU bake vs. GPU procedural), and **data layout** (structs, bit-exact fields) rather than the mathematical definition of the noise functions themselves.

## Shared principles
1. **Data Representation**: Noise data is stored as discrete arrays (textures or buffers) or computed on-the-fly. The choice depends on the trade-off between memory bandwidth/compression and computational cost [S1, S2, S3, unofficial][S4].
2. **Precision and Format**: Data types range from 8-bit unsigned integers (`GL_R8`, 256 levels) to 16-bit floats (`R16`) and 32-bit floats (`GL_RGB32F`). Higher precision reduces banding but increases memory usage [S1, S2, S3, unofficial].
3. **Randomness Source**: Generation relies on Pseudo-Random Number Generators (PRNGs) seeded by integers or system time to ensure reproducibility [S1, S3, unofficial][S4].
4. **Coordinate Mapping**: Input coordinates are mapped to texture indices or lattice nodes. For textures, wrap modes (`GL_REPEAT`, `Mirror`) handle boundary conditions [S2, S3, unofficial].
5. **Reproducibility**: Critical for distributed or multi-pass systems (e.g., CPU/GPU sync), requiring bit-exact generation algorithms and deterministic seeds [S3, unofficial][S4].

## Subtopics

### [2D White Noise Texture Generation](2485-2d-white-noise-texture-generation.md)
Creates a 2D texture filled with uncorrelated random values, typically for SSAO variation or dithering. Unlike gradient-based noise, it lacks spatial coherence.
- **Data Structures**: Uses a PRNG (`std::mt19937` or `std::srand`) and a noise buffer ($W \times H$).
- **Formats**: Supports vector noise (normalized 3D vectors, `GL_RGB32F`), scalar noise (`R16`), or byte noise (`GL_R8`).
- **Implementation**: Iterates pixels, generates random values, normalizes vectors (with zero-vector guards), and uploads to GPU.
- **Pitfalls**: `GL_R8` causes banding; zero vectors cause division by zero during normalization.
- **Link**: [2D White Noise Texture Generation](2485-2d-white-noise-texture-generation.md)

### GPU Procedural Noise & Normal Computation
Generates noise and computes normals on the GPU to trade computation time for memory efficiency. This avoids CPU evaluation and data transfer overhead [S1, unofficial].
- **Mechanism**: Bricks (voxel clusters) are created procedurally on the GPU. Normals are derived from the procedural noise field on-the-fly rather than stored [S1, unofficial].
- **Benefit**: More efficient than CPU evaluation for large volumes, supporting interactive rendering of detailed structures [S1, unofficial].

### World Generation Data Structure
Defines the layout for procedural world generation parameters, ensuring consistent configuration across systems.
- **Layout**: A `WorldGenerationData` struct containing: `octaves: u32`, `ceiling: u32`, `lacunarity: f32`, `gain: f32`, `frequency: f32`, `threshold: f32`, `seed: f32` [S2, unofficial].
- **Usage**: These parameters control the fractal Brownian motion (FBM) layers and terrain thresholds [S2, unofficial].

### Two-rate Field Generation
A high-performance method for generating large-scale voxel fields (e.g., planets) by separating continent- and range-wide octaves.
- **Step**: Evaluates octaves at lattice nodes and blends them. Uses compiled column programs instead of walking octave tables for each level [S3, unofficial].
- **Erosion**: Erosion octaves step their corners and share steering to maintain consistency [S3, unofficial].
- **Pitfall**: Results must remain bit-exact between CPU and GPU implementations to ensure consistency [S3, unofficial].
- **Performance**: Improved generation speed from 41–43 µs to 32–34 µs per tile [S3, unofficial].

### Built-In Noise Generation System
A system that procedurally creates volumetric noise, bakes it to a 3D texture, and associates it with a volume component asset.
- **Definition**: Automates the creation of noise textures for engines like Unity/Buto [S4].
- **Types**: Supports Perlin, Worley, Perlin-Worley, Billow, and Curl noise [S4].
- **Parameters**: Uses Seed, Quality (texture size), Invert, Frequency, Octaves, Lacunarity, and Gain [S4].
- **Advantage**: Better visual results than pre-compressed single-channel textures, especially when remapping value ranges [S4].

## Background
Procedural noise data structures have evolved from simple CPU-generated lookup tables to complex GPU-driven streaming systems. Early implementations relied on pre-computed textures due to limited GPU compute power [S1, unofficial]. As GPU shaders became more powerful, on-the-fly generation (e.g., GPU procedural noise) emerged to reduce memory bandwidth constraints [S1, unofficial]. Modern systems, such as those for voxel planets, require bit-exact reproducibility across CPU/GPU boundaries, leading to specialized data layouts and two-rate generation algorithms [S3, unofficial]. Integrated engines now provide built-in noise generation systems to simplify asset creation [S4].

## Sources
[S1] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S2] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S3] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
[S4] Volume Noise Sources, https://docs.occasoftware.com/buto/volume-noise-sources, fetched 2026-10-27
