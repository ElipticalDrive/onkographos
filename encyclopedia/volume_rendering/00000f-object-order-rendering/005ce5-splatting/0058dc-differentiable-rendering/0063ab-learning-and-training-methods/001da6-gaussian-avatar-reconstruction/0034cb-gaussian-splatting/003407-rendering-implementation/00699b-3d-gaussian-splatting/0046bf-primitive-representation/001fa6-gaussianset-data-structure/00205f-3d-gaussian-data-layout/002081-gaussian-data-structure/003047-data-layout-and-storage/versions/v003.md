# Data Layout and Storage


## Overview
Data Layout and Storage defines the memory organization, serialization formats, and buffer management strategies for 3D Gaussian primitives, ensuring efficient GPU access, dynamic scene updates, and cross-platform compatibility. It differs from sibling methods by focusing on the fundamental memory layout (SoA vs. AoS), coefficient indexing, and dynamic buffer growth rather than rendering algorithms [G].

## Shared principles
The family adheres to a Structure-of-Arrays (SoA) paradigm for GPU rendering, where each attribute (position, scale, rotation, opacity, SH coefficients) occupies a separate contiguous linear buffer. This layout maximizes cache coherency during batched rendering by ensuring shaders iterate over single attributes without skipping irrelevant data [5568-splat-data-buffer-layout.md].

Key mathematical and structural conventions include:
- **Indexing:** For a buffer of $N$ splats, the $i$-th splat's attribute starts at index $i \times \text{stride}$.
- **Spherical Harmonics (SH):** Coefficients are stored in a flat buffer of size $N \times D \times 3$, where $D$ is the number of coefficients per channel. The index for the $c$-th channel of the $d$-th coefficient of the $i$-th splat is:
  $$ idx = ((i \times D) + d) \times 3 + c $$
  [5570-spherical-harmonics-coefficient-layout.md].
- **Log-Scale Storage:** Scales are often stored in log-space to ensure positivity and improve optimization stability. Linear scales are recovered via $s_{linear} = \exp(s_{log})$ [S1, unofficial].
- **Quaternion Ordering:** Rotations are stored as 4-component quaternions. JSplat uses scalar-last order $(x, y, z, w)$ [S1, unofficial].

## Subtopics

### [Splat Data Buffer Layout](5568-splat-data-buffer-layout.md)
Defines the fundamental SoA memory layout for all Gaussian attributes. Choose this for standard GPU rendering pipelines requiring cache-efficient batched access.

### [Spherical Harmonics Coefficient Layout](5570-spherical-harmonics-coefficient-layout.md)
Specializes in the contiguous flat buffer organization for SH coefficients. Choose this when optimizing color/SH data transfer or storage.

### [Covariance Splat Reading](6840-covariance-splat-reading.md)
Handles decoding Gaussian properties from compressed texture arrays (`usampler2DArray`) into a structured `CovSplat` representation. Choose this for texture-based storage or compressed scene formats.

### [Dynamic Gaussian Splatting Scene Management](6907-dynamic-gaussian-splatting-scene-management.md)
Provides thread-safe, GPU-resident SoA structures with automatic buffer growth and pending batch queues for real-time incremental updates. Choose this for dynamic scenes or SLAM applications.

### [GS-W Network Architecture](5927-gs-w-network-architecture.md)
Not given in the sources.

### [VastGaussian](6000-vastgaussian.md)
Not given in the sources.

### [SLAM and Tracking](6236-slam-and-tracking.md)
Not given in the sources.

### [Deformation and Dynamic Modeling](11474-deformation-and-dynamic-modeling.md)
Not given in the sources.

### [WASM Data Packing](7761-wasm-data-packing.md)
Manages WebAssembly heap allocation and data packing for CPU-side processing or GPU upload in browser environments. Choose this for Web-based Gaussian Splatting implementations.

## Background
The SoA layout for Gaussian Splatting was popularized by the original 3D Gaussian Splatting paper [G]. Subsequent implementations like JSplat [S1, unofficial] and dynamic scene managers [S1, unofficial] refined these layouts for specific performance constraints. The WASM packing strategy [S1, unofficial] addresses the unique memory model of WebAssembly. The HLOD format v2 [S2, unofficial] extends these concepts for large-scale world storage with strict bounds validation.

## Sources

[S1] JSplat, https://github.com/javagl/JSplat, fetched 2026-10-30 (unofficial)
[S2] gaussian_splatting: HLOD S1a: .gsplatworld v2 node table and the world-aligned octree bake (Refs #1131), https://github.com/klausi3D/godotGS/pull/1149, fetched 2026-10-27 (unofficial)
