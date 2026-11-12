# Data Layout and Storage

Also called: Splat Buffer Layout, SoA Splat Storage

## Overview
Data Layout and Storage defines the memory organization for 3D Gaussian splats, separating attributes into contiguous buffers to optimize memory bandwidth and SIMD vectorization during rendering and optimization. It distinguishes itself from interleaved (Structure-of-Arrays vs. Array-of-Structures) alternatives by grouping data by parameter type rather than by splat instance, ensuring that GPU kernels access homogeneous data streams [S1, unofficial].

## Shared principles
The family operates on a Structure-of-Arrays (SoA) paradigm where each attribute of the Gaussian primitive occupies a separate contiguous linear buffer. This layout minimizes cache misses during batched rendering passes, as shaders or compute kernels iterate over a single attribute (e.g., positions) without skipping irrelevant data (e.g., rotations or opacities).

The core data structures for the base layout are:
1.  **Positions:** A `FloatBuffer` of size $N \times 3$, storing $(x, y, z)$ coordinates.
2.  **Scales:** A `FloatBuffer` of size $N \times 3$, storing $(s_x, s_y, s_z)$.
3.  **Rotations:** A `FloatBuffer` of size $N \times 4$, storing quaternion components.
4.  **Opacities:** A `FloatBuffer` of size $N$, storing a single scalar opacity value per splat.
5.  **Spherical Harmonics (SH):** A `FloatBuffer` of size $N \times D \times 3$, where $D$ is the number of SH coefficients per channel and 3 represents the RGB channels.

The indexing formula for accessing the $c$-th component (where $c$ is the channel index, e.g., 0 for R, 1 for G, 2 for B, or spatial dimension) of the $d$-th coefficient of the $i$-th splat in a flat buffer is:
$$ idx = ((i \times D) + d) \times 3 + c $$
For non-SH attributes, the stride is fixed by the dimensionality of the attribute (3 for position/scale, 4 for rotation, 1 for opacity).

## Subtopics

### [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md)
This method defines the primary SoA memory layout for storing 3D Gaussian splat attributes. It separates parameters into contiguous buffers to maximize memory bandwidth utilization. It is chosen when rendering performance and efficient GPU transfer are prioritized over cache locality for individual splat updates. See [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md).

### [Spherical Harmonics Coefficient Layout](5570-spherical-harmonics-coefficient-layout.md)
This method specializes the storage of SH coefficients, solving the problem of efficiently packing per-splat color data into contiguous buffers for GPU transfer. It groups data by splat first, then by coefficient, then by spatial dimension (RGB). This differs from layouts that might group globally by dimension or coefficient. See [Spherical Harmonics Coefficient Layout](5570-spherical-harmonics-coefficient-layout.md).

### [Covariance Splat Reading](6840-covariance-splat-reading.md)
A method to read a single 3D Gaussian Splat in covariance representation from two texture arrays, decoding its properties into a `CovSplat` struct. This is used when the storage format relies on texture-based sampling rather than linear buffers for specific rendering paths. See [Covariance Splat Reading](6840-covariance-splat-reading.md).

### [Dynamic Gaussian Splatting Scene Management](6907-dynamic-gaussian-splatting-scene-management.md)
Manages the lifecycle of splats in dynamic scenes, handling insertion, deletion, and buffer resizing. See [Dynamic Gaussian Splatting Scene Management](6907-dynamic-gaussian-splatting-scene-management.md).

### [WASM Data Packing](7761-wasm-data-packing.md)
Manages WebAssembly memory allocation and data packing for 3D Gaussian splat attributes (positions, rotations, scales, colors, selection) to prepare them for GPU upload or WASM-side processing. See [WASM Data Packing](7761-wasm-data-packing.md).

### Log-Scale Storage
Scales are stored internally in log-space to ensure positivity and improve optimization stability. When writing to buffers, linear values are converted to log-space via $\log(s)$. When reading for rendering or optimization, log-space values are converted to linear via $s_{linear} = \exp(s_{log})$ [S1, unofficial].

### Rotation Quaternion Ordering
Rotations are stored as 4-component quaternions in scalar-last order $(x, y, z, w)$ within a `FloatBuffer` of size $N \times 4$. For each splat $i$, the components are written sequentially to indices $i \times 4 + 0$ through $i \times 4 + 3$ [S1, unofficial].

## Background
The SoA layout for Gaussian splats was formalized in the JSplat library, which provides a reference implementation for efficient CPU-side manipulation of splat data structures [S1, unofficial]. This approach aligns with general graphics programming best practices for SIMD processing, where contiguous memory access patterns reduce cache thrashing during the massive parallelism required by differentiable rendering pipelines.

## Sources
[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
