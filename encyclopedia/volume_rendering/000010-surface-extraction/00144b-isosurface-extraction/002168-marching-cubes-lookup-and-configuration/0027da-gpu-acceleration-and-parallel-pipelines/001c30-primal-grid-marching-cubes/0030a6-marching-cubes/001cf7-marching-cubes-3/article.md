# Marching Cubes 3


## Overview
Marching Cubes 3 is a specific implementation variant of the Marching Cubes algorithm for extracting polygonal meshes from 3D scalar fields [S1, unofficial]. It builds on the primal grid processing of [Marching Cubes](774-marching-cubes.md) but follows a distinct function signature and internal configuration for the lookup and vertex generation steps [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field array `arr` of dimensions `numx` $\times$ `numy` $\times$ `numz`.
- **Input:** Lower and upper bounds for the scalar field values (`lower`, `upper`).
- **Input:** The target `isovalue` for surface extraction.
- **Output:** A list of `vertices` (3D coordinates).
- **Output:** A list of `polygons` (indices into the vertices list).

**Data Structures:**
The scalar field is accessed via a callback function that maps grid coordinates $(i, j, k)$ to scalar values. The output structures are dynamic lists that grow as triangles are generated.

**Steps:**
1. **Validation:** Verify the input data is 3-dimensional [S1, unofficial].
2. **Preparation:** Extract the shape of the input array to determine `numx`, `numy`, and `numz` [S1, unofficial].
3. **Bounds:** Compute or validate the `lower` and `upper` bounds of the scalar field values [S1, unofficial].
4. **Initialization:** Initialize empty containers for `vertices` and `polygons` [S1, unofficial].
5. **Execution:** Iterate over each voxel, compute the case index from the 8 corner values, look up the corresponding triangle configuration, interpolate edge vertices, and append the resulting triangles to the output lists [S1, unofficial].

**Formulas:**
Linear interpolation for a vertex on an edge between corners with values $v_1$ and $v_2$:
$$v = v_1 + \frac{(iso - v_1)}{(v_2 - v_1)} (v_2 - v_1)$$
The case index is computed as $\sum_{b=0}^{7} b_b \cdot 2^b$, where $b_b = 1$ if the $b$-th corner value is $\ge iso$, else $0$ [G].

**Parameters:**
- `isovalue`: The scalar threshold. Not given in the sources.
- `lower`, `upper`: Scalar bounds. Not given in the sources.

**Edge Cases:**
- If the array is not 3D, the method fails validation [S1, unofficial].
- Precision issues may arise if `isovalue` is exactly equal to a vertex value; standard implementations handle this via epsilon comparisons or specific edge cases in the lookup table [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_3(arr, isovalue):
    if arr.ndim != 3:
        raise ValueError("Input must be a 3D array")
    numx, numy, numz = arr.shape
    lower = float(arr.min())
    upper = float(arr.max())
    vertices = []
    polygons = []

    # Standard Marching Cubes lookup tables [G]
    edge_table = [...] # 256 entries
    tri_table = [...]  # 256 entries
    edge_vertices = [...] # Maps edge index to corner indices

    for i in range(numx - 1):
        for j in range(numy - 1):
            for k in range(numz - 1):
                v = [arr[i, j, k], arr[i+1, j, k], arr[i+1, j+1, k], arr[i, j+1, k],
                     arr[i, j, k+1], arr[i+1, j, k+1], arr[i+1, j+1, k+1], arr[i, j+1, k+1]]
                case_idx = sum((1 << b) for b in range(8) if v[b] >= isovalue)
                
                for tri_idx in range(0, 15, 3):
                    e1, e2, e3 = tri_table[case_idx][tri_idx:tri_idx+3]
                    if e1 == -1: break
                    
                    def interp(e_idx):
                        c1, c2 = edge_vertices[e_idx]
                        v1, v2 = v[c1], v[c2]
                        t = (isovalue - v1) / (v2 - v1) if abs(v2 - v1) > 1e-8 else 0.0
                        # Standard edge-to-3D coordinate mapping [G]
                        base = [i + (c1 & 1), j + ((c1 >> 1) & 1), k + ((c1 >> 2) & 1)]
                        dir_vec = [0, 0, 0]
                        if c1 != c2:
                            for d in range(3):
                                if ((c1 >> d) & 1) != ((c2 >> d) & 1):
                                    dir_vec[d] = 1.0
                        return [base[d] + t * dir_vec[d] for d in range(3)]
                    
                    p1 = interp(e1)
                    p2 = interp(e2)
                    p3 = interp(e3)
                    vertices.extend([p1, p2, p3])
                    base_idx = len(vertices) - 3
                    polygons.append([base_idx, base_idx+1, base_idx+2])
    return vertices, polygons
```

## Subtopics

### [Marching Cubes with Lewiner 2003 Ambiguity Resolution](2292-marching-cubes-with-lewiner-2003-ambiguity-resolution.md)
Resolves topological ambiguities in standard MC to guarantee manifold surfaces. Choose this when topological correctness is critical.

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Strengths and Limitations:** Not given in the sources.
**When to use:** Not given in the sources.

## Background
Marching Cubes 3 is identified as a variant within the `differentiable_volumetric_rendering` repository [S1, unofficial]. It is part of the broader family of primal grid marching cubes methods [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
