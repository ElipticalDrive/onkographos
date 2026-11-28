# Marching Cubes 3
## Overview
Marching Cubes 3 is a specific implementation variant of the Marching Cubes algorithm for extracting polygonal meshes from 3D scalar fields [S1, unofficial]. It provides a third implementation variant of the core algorithm, distinguished by its specific function signature and integration pattern [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field array `arr` of dimensions `numx` $\times$ `numy` $\times$ `numz` [S1, unofficial].
*   **Input:** Lower and upper bounds for the scalar field range, `lower` and `upper` [S1, unofficial].
*   **Input:** The `isovalue` scalar threshold [S1, unofficial].
*   **Output:** A list of `vertices` representing the 3D coordinates of the mesh vertices [S1, unofficial].
*   **Output:** A list of `polygons` representing the mesh faces [S1, unofficial].

**Data Structures:**
The implementation assumes a contiguous memory layout for the input array, typically accessed via a standard array indexing function or memory view [S1, unofficial]. The output structures are dynamic arrays or lists of vertices and indices [G].

**Steps:**
1.  **Validation and Initialization:**
    *   Validate that the input data is 3D [S1, unofficial].
    *   Extract the shape of the data to determine `numx`, `numy`, and `numz` [S1, unofficial].
    *   Compute bounds if required by the specific wrapper [S1, unofficial].
    *   Initialize empty vectors/lists for `vertices` and `polygons` [S1, unofficial].
2.  **Core Extraction:**
    *   Invoke the core algorithm function with the parameters `lower`, `upper`, `numx`, `numy`, `numz`, the scalar field accessor, `isovalue`, `vertices`, and `polygons` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_3(arr, isovalue, lower, upper):
    # Validate input is 3D
    if arr.ndim != 3:
        raise ValueError("Input array must be 3D")
    # Extract dimensions
    numx, numy, numz = arr.shape
    # Initialize output containers
    vertices = []
    polygons = []
    # Access scalar field data via standard array indexing
    def get_scalar(x, y, z):
        return arr[z, y, x]
    # Core extraction routine processes voxels and populates vertices/polygons
    # Implementation details for mc::marching_cubes3 are not provided in the sources [G]
    return vertices, polygons
```

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundary Conditions:** Not given in the sources.
*   **Degenerate Cases:** Not given in the sources.
## Subtopics

### [Marching Cubes with Lewiner 2003 Ambiguity Resolution](2292-marching-cubes-with-lewiner-2003-ambiguity-resolution.md)
This method resolves topological ambiguities (Ambiguous face and Internal ambiguity) present in the original 1987 algorithm to guarantee a manifold surface. It tests faces and interior points of the cube based on the Lewiner 2003 paper. Unlike the original MC which may produce cracks and non-manifold geometry, Lewiner's method guarantees manifold surfaces at the cost of larger lookup tables.

### [Ray-Unit Parallel Marching Cubes](5169-ray-unit-parallel-marching-cubes.md)
Ray-Unit Parallel Marching Cubes (rupMC) is an enhanced Marching Cubes algorithm designed for CPU/GPU heterogeneous architectures that reduces redundant edge intersection computations and preserves topological information of the isosurface. It replaces the standard cubic voxel with a ray unit to ensure each edge intersection is calculated and stored exactly once, while parallelizing I/O via OpenMPI and computation via CUDA to handle large-scale datasets efficiently. Unlike standard MC, which suffers from repeated calculations on shared edges and lacks explicit topological connectivity, rupMC embeds topological relationships directly into the triangle composition using unique intersection indices.

### [Isomesh](14236-isomesh.md)
Isomesh is a surface extraction technique that triangulates the level set of a 3D scalar field using the Marching Cubes algorithm and renders the result as a shaded, depth-tested mesh. It solves the visual artifacts of volume rendering approaches—specifically the striping inherent in ray-marched iso-shells and the lack of depth testing in MIP/absorption methods that forces the body to occlude the field. Unlike its siblings, it produces smooth, correctly occluded surfaces by leveraging GPU-accelerated rasterization of a pre-computed mesh rather than per-pixel ray marching.

### SurfaceExtractorMC
Marching Cubes extraction over an Image3, built on the MarchingCubes 256-entry table [S2, unofficial]. The upstream Extract overloads are replaced by `extractVoxel(level, perturb, F)` returning `{ valid, mesh }` and `extract(level, perturb)` [S2, unofficial]. Only the canonical query keeps the plain name [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
Not given in the sources.

**When to Use:**
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Port B46: surface extractors batch, https://github.com/gradientspaceai/gtengine-js/pull/131, fetched 2026-10-27 (unofficial)
