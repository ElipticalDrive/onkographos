# Flying Edges

Also called: vtkFlyingEdges3D

## Overview
Flying Edges is a parallel, threaded algorithm for volumetric isocontouring of continuous scalar fields that addresses the performance limitations of Marching Cubes by processing volume edges rather than voxel cells [S1, unofficial]. It eliminates point merging bottlenecks by using an edge-based iterator to generate output points and triangle cells, avoiding the need for hashing duplicate points [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A structured grid (volume) with continuous scalar values at vertices, and an isovalue $q$.
- Output: A polygonal surface (polydata) consisting of points and triangle cells representing the isocontour.

**Data structures:**
The algorithm uses a multi-pass approach.
1.  **Pass 1 (Gathering):** The algorithm iterates through the volume to gather information about the output size (number of points and triangles) without generating geometry. This enables precise, one-time memory allocation [S1, unofficial].
2.  **Pass 2 (Generation):** Multiple threads simultaneously produce output using the pre-allocated memory, processing data along each edge independently [S1, unofficial].

**Algorithm Steps:**
1.  Initialize memory allocation for points and triangles based on the gathered information from Pass 1.
2.  Divide the volume edges among available threads.
3.  For each thread, iterate through its assigned edges:
    a.  Retrieve the scalar values at the two endpoints of the edge.
    b.  Check if the isovalue $q$ lies between the endpoint values.
    c.  If so, compute the intersection point using linear interpolation.
    d.  Determine the triangle connectivity based on the surrounding cell topology.
    e.  Write the point and triangle data directly into the pre-allocated buffers.
4.  Finalize the output polydata structure.

**Formulas:**
For an edge with endpoints having scalar values $v_1$ and $v_2$, and spatial coordinates $\mathbf{p}_1$ and $\mathbf{p}_2$, the intersection point $\mathbf{p}_{int}$ at isovalue $q$ is calculated as:
$$
\mathbf{p}_{int} = \mathbf{p}_1 + \frac{q - v_1}{v_2 - v_1} (\mathbf{p}_2 - \mathbf{p}_1)
$$
This formula assumes $v_1 \neq v_2$. If $v_1 = v_2$, the edge is parallel to the isosurface and no intersection is generated [G].

**Parameters:**
- Isovalue $q$: The threshold value for continuous scalar fields.
    - Example: $q=100$ for CT-Angio (16-bit) [S1, unofficial].
    - Example: $q=0.07$ for SuperNova (32-bit float) [S1, unofficial].
    - Example: $q=1800$ for Nano (16-bit unsigned) [S1, unofficial].

**Edge cases and pitfalls:**
- Parallel efficiency declines as the number of threads increases, typically showing up when the number of threads approaches 20-40 CPU threads due to memory bus saturation, cache locality issues, and thread overhead [S1, unofficial].
- The algorithm assumes continuous scalar fields. Discrete or integer fields may require preprocessing or different handling [G].

## Operating characteristics
- **Time Complexity:** The algorithm accesses voxel values only one time and performs edge intersections only once, compared to Marching Cubes which accesses each voxel value up to eight times and processes edges up to four times [S1, unofficial].
- **Memory Complexity:** Requires two passes: one for gathering output size and one for generation. Memory is allocated once based on the gathered size [S1, unofficial].
- **Performance:** Flying Edges is much faster than Marching Cubes even when run sequentially (with one thread) [S1, unofficial]. Performance improvements are typically one to two orders of magnitude greater than MC-based methods [S1, unofficial].
- **Strengths:** High performance, parallel efficiency (up to a point), no point merging bottleneck, precise memory allocation.
- **Limitations:** Parallel efficiency declines with high thread counts (20-40+ threads) due to memory bus saturation and cache issues [S1, unofficial].
- **When to use:** When high-performance isocontouring of continuous scalar fields is required, especially in parallel environments.
- **When not to use:** When the number of threads is very high (20-40+) and memory bandwidth is a bottleneck, or when discrete scalar fields are used.

## Background
Flying Edges was developed by the VTK community to improve system performance for applications like ParaView, 3D Slicer, and trame [S1, unofficial]. It was introduced in a blog post titled "Really Fast Isocontouring" by Will Schroeder and Spiros Tsalikis in June 2023 [S1, unofficial]. The algorithm is implemented in VTK as `vtkFlyingEdges3D` [S1, unofficial]. Robert Maynard has also implemented Flying Edges in `vtk-m` [S1, unofficial].

## Sources
[S1] Really Fast Isocontouring, https://www.kitware.com/really-fast-isocontouring/, fetched 2026-10-30 (unofficial)
