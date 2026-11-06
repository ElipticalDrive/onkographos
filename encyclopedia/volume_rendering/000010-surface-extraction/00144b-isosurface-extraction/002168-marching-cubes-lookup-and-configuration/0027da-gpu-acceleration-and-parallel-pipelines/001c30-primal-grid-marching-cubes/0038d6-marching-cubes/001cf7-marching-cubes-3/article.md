# Marching Cubes 3


## Overview
Marching Cubes 3 is a specific implementation variant of the Marching Cubes algorithm for extracting polygonal meshes from 3D scalar fields [S1, unofficial]. It builds on the standard Marching Cubes topology lookup but is distinguished by its integration into differentiable volumetric rendering pipelines, where it serves as the surface extraction layer for gradient-based optimization [S1, unofficial]. Unlike generic implementations, it is designed to interface directly with tensor-based data structures (e.g., PyTorch tensors) and C-style function pointers for efficient memory access [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field array (`arr`) of dimensions `numx`, `numy`, `numz`; a lower bound (`lower`) and upper bound (`upper`) for the data range; an `isovalue` threshold; and a function pointer `PyArrayToCFunc` that maps array indices to scalar values [S1, unofficial].
*   **Outputs:** A list of `vertices` (3D coordinates) and a list of `polygons` (indices into the vertices list) forming the mesh [S1, unofficial].

**Data Structures:**
The method assumes a contiguous 3D array layout. The `PyArrayToCFunc` abstraction hides the underlying memory layout (row-major vs. column-major) and data type conversion, providing a uniform `double get_value(int x, int y, int z)` interface [S1, unofficial].

**Steps:**
1.  **Validation:** Verify the input data is 3-dimensional. Extract the shape to determine `numx`, `numy`, and `numz` [S1, unofficial].
2.  **Bounds Initialization:** Compute or accept the `lower` and `upper` bounds of the scalar field [S1, unofficial].
3.  **Vector Initialization:** Initialize empty dynamic arrays (vectors) for `vertices` and `polygons` [S1, unofficial].
4.  **Core Extraction:** Invoke the core algorithm `mc::marching_cubes3<double>` with the parameters: `lower`, `upper`, `numx`, `numy`, `numz`, the value accessor `PyArrayToCFunc(arr)`, `isovalue`, and the output vectors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_3(arr, isovalue, lower, upper):
    """
    Wrapper for the Marching Cubes 3 implementation.
    arr: 3D numpy array or tensor.
    isovalue: float, the threshold for surface extraction.
    lower, upper: float, bounds of the scalar field.
    Returns: vertices (N, 3), polygons (M, 3)
    """
    # 1. Validate 3D and extract dimensions
    if arr.ndim != 3:
        raise ValueError("Input must be a 3D array")
    
    numx, numy, numz = arr.shape
    
    # 2. Initialize output containers
    vertices = []
    polygons = []
    
    # 3. Define value accessor
    # In a real C++/Python binding, this maps to PyArrayToCFunc
    def get_value(x, y, z):
        return arr[x, y, z]
        
    # 4. Call core algorithm
    # The core algorithm iterates over all (numx-1) x (numy-1) x (numz-1) cubes
    # and populates vertices and polygons lists based on the 256-entry lookup table.
    core_marching_cubes_3(
        lower=lower,
        upper=upper,
        numx=numx,
        numy=numy,
        numz=numz,
        value_func=get_value,
        isovalue=isovalue,
        vertices=vertices,
        polygons=polygons
    )
    
    return vertices, polygons
```

**Edge Cases and Pitfalls:**
*   **Precision:** The implementation uses `double` precision for the core algorithm to maintain numerical stability during interpolation [S1, unofficial].
*   **Boundaries:** The algorithm processes cubes defined by vertices $(0,0,0)$ to $(numx-1, numy-1, numz-1)$, effectively ignoring the outermost layer of voxels as cube corners [S1, unofficial].

## Subtopics

### [Marching Cubes with Lewiner 2003 Ambiguity Resolution](2292-marching-cubes-with-lewiner-2003-ambiguity-resolution.md)
This variant resolves topological ambiguities (Ambiguous face and Internal ambiguity) present in the original 1987 algorithm to guarantee a manifold surface. It tests faces and interior points of the cube to disambiguate cases where the standard 256-entry table yields non-manifold geometry or cracks. It is chosen when topological correctness is critical, at the cost of larger lookup tables and increased computational overhead.

### [Ray-Unit Parallel Marching Cubes](5169-ray-unit-parallel-marching-cubes.md)
Ray-Unit Parallel Marching Cubes (rupMC) is an enhanced version designed for CPU/GPU heterogeneous architectures. It replaces the standard cubic voxel with a "ray unit" to ensure each edge intersection is calculated exactly once, eliminating redundant computations on shared edges. It embeds topological relationships directly into triangle composition using unique intersection indices and parallelizes I/O via OpenMPI and computation via CUDA. It is chosen for large-scale datasets requiring efficient parallel processing and explicit topological connectivity.

### SurfaceExtractorMC
SurfaceExtractorMC is a Marching Cubes extraction class built over an `Image3` structure, utilizing the standard Marching Cubes 256-entry table [S2, unofficial]. It replaces the upstream pure virtual `Extract` method with `extractVoxel(level, perturb, F)` which returns a tuple `{valid, mesh}` and `extract(level, perturb)` [S2, unofficial]. Only the canonical query keeps the plain name `extract` [S2, unofficial]. It is used within the `gtengine-js` ported surface extractor framework [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is processed independently.
*   **Memory Complexity:** $O(V + F)$ where $V$ is the number of vertices and $F$ is the number of faces in the output mesh.
*   **Performance:** Performance is heavily dependent on the efficiency of the `PyArrayToCFunc` accessor and the underlying C++ implementation. The use of `double` precision ensures accuracy but may be slower than single-precision variants [S1, unofficial].
*   **Strengths:** Simple implementation, widely supported, differentiable (in specific variants like the one in [S1, unofficial]).
*   **Limitations:** Standard Marching Cubes can produce non-manifold geometry in ambiguous cases; requires post-processing for topological correctness in those cases [S1, unofficial].

## Background
The Marching Cubes algorithm was originally introduced by Lorensen and Cline in 1987 [S1, unofficial]. Marching Cubes 3 refers to a specific implementation variant found in the `autonomousvision/differentiable_volumetric_rendering` repository, designed to integrate surface extraction into differentiable rendering pipelines [S1, unofficial]. The `SurfaceExtractorMC` variant is part of the `gtengine-js` porting efforts (Port B46) to bring C++ surface extractors to JavaScript/TypeScript environments [S2, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Port B46: surface extractors batch, https://github.com/gradientspaceai/gtengine-js/pull/131, fetched 2026-10-27 (unofficial)
