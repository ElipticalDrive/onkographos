# HKTex Chunking and Tiling

Also called: HKTex Rendering Efficiency

## Overview
This method optimizes the evaluation of Heat Kernel Textures (HKTex) during the rendering pipeline by processing surface queries in fixed-size batches and optionally tiling the image plane. It addresses GPU memory constraints and computational bottlenecks inherent in evaluating geodesic-based texture fields for high-resolution meshes. It differs from standard dense texture lookups by batching queries to improve cache locality and reduce peak memory usage [S1].

## How it works
The method operates on the output of the HKTex generation phase, where texture parameters are defined by a set of kernel centers and eigenmodes on a mesh surface.

**Inputs:**
- A set of $B$ surface points (or ray-surface intersections) requiring texture evaluation.
- HKTex parameters: Kernel centers, eigenmode coefficients, and learnable scale $\tau$ [S1].

**Outputs:**
- Rendered pixel colors for the current batch or tile.

**Data Structures:**
- **Chunk Buffer:** A contiguous memory block holding $N_{chunk}$ surface points. The standard size is $N_{chunk} = 1024$ points [S1].
- **Tile Grid:** An optional subdivision of the image plane into crops of size $W_{tile} \times H_{tile}$. The standard size is $32 \times 32$ pixels [S1].

**Steps:**
1. **Chunking:** Divide the total set of $B$ surface points into chunks of size $N_{chunk}$. If $B$ is not divisible by $N_{chunk}$, the final chunk contains the remaining points [S1].
2. **Evaluation:** For each chunk, evaluate the heat-kernel texture queries. This involves computing the weighted sum of eigenmodes based on geodesic distances from the query points to the kernel centers [S1].
3. **Tiling (Optional):** For final high-quality renderings, subdivide the image plane into tiles of $32 \times 32$ pixels. Process each tile independently to limit GPU memory consumption during the rasterization or ray-marching phase [S1].
4. **Denoising:** Apply the OptiX denoiser to the displayed renderings. This step is excluded from the ray-based training loss calculation [S1].

**Parameters:**
- Chunk size: 1024 surface points [S1].
- Tile size: $32 \times 32$ pixels [S1].
- Eigenmodes $K$: Empirically sufficient value is 256 [S1].

**Pitfalls:**
- **Lighting Baking:** When training against real photographs, ensure the PBR pipeline disentangles lighting, shadows, and view-dependent reflections. Optimizing only the albedo component against real images causes the model to incorrectly bake scene lighting into the diffuse color [S1].
- **Vertex Count Misconception:** The method is not constrained by the mesh vertex count. It can represent textures at resolutions higher than the mesh geometry, as demonstrated by fitting an image on a two-triangle rectangle with $K=3$ [S1].

## Subtopics

### HKTex Kernel Distribution and Scalability
HKTex kernels are distributed adaptively based on texture complexity rather than uniform mesh density. In test meshes with an average of $N \approx (9.6 \pm 19)$k vertices, the method uses an average of $S \approx (4.3 \pm 0.6)$k kernels. Approximately 49.8% of faces have no kernels, and 13.2% have only one. The average number of kernel centers per face is $4.8 \pm 7.1$, with a maximum of $70 \pm 123$. This distribution proves that kernels move to locations that benefit texture quality most, allowing the method to outperform alternatives despite using fewer kernels than vertices [S1].

## Operating characteristics
- **Memory:** Chunking limits peak memory usage for texture queries to the size of 1024 points. Tiling limits GPU memory for the final rendering buffer to $32 \times 32$ pixel tiles [S1].
- **Performance:** The method is efficient enough to handle meshes with high vertex counts using a sparse set of kernels. The use of $K=256$ eigenmodes provides sufficient control without excessive computational overhead [S1].
- **Limitations:** Direct editing of HKTex parameters is non-trivial, similar to Gaussian Splatting, due to the complexity of the geodesic basis. Modifications require baking the target region to a dense point cloud, painting, and re-optimizing [S1].

## Background
The chunking and tiling strategies are part of the Heat Kernel Textures (HKTex) method, which represents textures using geodesic Gaussians. The method was introduced in the paper "Heat Kernel Textures the Geodesic Gaussians That Do Not Splat" [S1]. The approach leverages the spectral decomposition of the Laplace-Beltrami operator to define texture fields on surfaces, offering an alternative to traditional UV mapping or splatting-based texture representations.

## Sources
[S1] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
