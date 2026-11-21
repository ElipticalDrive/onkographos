# Voxel Data Interpolation and Sampling

Also called: Sample Grid, Sample Grid Index, getNextIntersectColor, getNextIntersectionGradient, getNextIntersectionScalar

## Overview
Voxel Data Interpolation and Sampling is a method for retrieving scalar values, gradients, and transformed colors at arbitrary 3D positions within a dense volume grid [S1, unofficial]. It solves the problem of mapping continuous ray-casting coordinates to discrete grid data, supporting both nearest-neighbor and trilinear interpolation modes [S1, unofficial]. It differs from its siblings by providing the low-level access primitives (scalar and gradient retrieval) that higher-level transfer functions and shading models consume, rather than defining the basis functions themselves [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Continuous query position $\mathbf{p} = (x, y, z)$; Grid dimensions $X_{DIM}, Y_{DIM}, Z_{DIM}$; Interpolation mode $INTER\_MODE$; Transfer function mode $TRN\_MODE$.
*   **Output:** Interpolated scalar value $v$, gradient vector $\mathbf{g}$, and optionally transformed color components.

**Data Structures:**
The volume data is stored in two vectors [S1, unofficial]:
1.  `scalars`: `vector<float>` storing scalar values (e.g., density).
2.  `gradients`: `vector<glm::vec3>` storing pre-computed gradient vectors.

The index for a voxel at integer coordinates $(x, y, z)$ is calculated as [S1, unofficial]:
$$ index = x + (y \cdot X_{DIM}) + (z \cdot X_{DIM} \cdot Y_{DIM}) $$

**Parameters:**
*   $INTER\_MODE$: Controls interpolation. $1$ for trilinear interpolation, $2$ for nearest neighbor [S1, unofficial].
*   $TRN\_MODE$: Controls transfer function input. $1$ for $(Voxel Value, Voxel Value)$, $2$ for $(Voxel Value, Gradient Magnitude)$ [S1, unofficial].

**Steps:**
1.  **Boundary Check:** Determine the integer coordinates $(ix, iy, iz)$ of the query point. If any coordinate is outside the grid bounds $[0, X_{DIM}-1]$, $[0, Y_{DIM}-1]$, or $[0, Z_{DIM}-1]$, return a default value (e.g., $(0,0,0,0)$) [S1, unofficial].
2.  **Value Retrieval:**
    *   If $INTER\_MODE = 2$ (Nearest Neighbor): Retrieve the scalar $v$ and gradient $\mathbf{g}$ at $(ix, iy, iz)$ using the index formula [S1, unofficial].
    *   If $INTER\_MODE = 1$ (Trilinear): Retrieve the 8 corner values and gradients, then compute the weighted sum based on fractional coordinates (see Pseudocode) [S1, unofficial]. *Note: Some implementations default to nearest neighbor if voxel intersection logic is not fully implemented [S1, unofficial].*
3.  **Transfer Function Preparation:**
    *   If $TRN\_MODE = 1$: Pass $(v, v)$ to the transfer function transformer.
    *   If $TRN\_MODE = 2$: Compute gradient magnitude $m = \|\mathbf{g}\|$. Clamp $m$ to $[0, 1]$. Scale by $256$. Pass $(v, m \cdot 256)$ to the transformer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_voxel(p_x, p_y, p_z, scalars, gradients, x_dim, y_dim, z_dim, inter_mode, trn_mode):
    # 1. Boundary Check
    ix = int(p_x)
    iy = int(p_y)
    iz = int(p_z)
    
    if ix < 0 or ix >= x_dim or iy < 0 or iy >= y_dim or iz < 0 or iz >= z_dim:
        return 0.0, (0.0, 0.0, 0.0)

    # 2. Value Retrieval
    if inter_mode == 2: # Nearest Neighbor
        idx = ix + (iy * x_dim) + (iz * x_dim * y_dim)
        v = scalars[idx]
        g = gradients[idx]
    elif inter_mode == 1: # Trilinear
        # Fractional coordinates
        fx = p_x - ix
        fy = p_y - iy
        fz = p_z - iz
        
        # Indices of the 8 corners
        # Note: Implementation details for trilinear weighting are marked as needing implementation in source [S1, unofficial]
        # Standard trilinear interpolation would involve:
        # v = sum(B_i * corner_v) for i in 0..7
        # g = sum(B_i * corner_g) for i in 0..7
        # For this implementation, we fall back to nearest neighbor if not implemented [S1, unofficial]
        idx = ix + (iy * x_dim) + (iz * x_dim * y_dim)
        v = scalars[idx]
        g = gradients[idx]
    else:
        raise ValueError("Unknown interpolation mode")

    # 3. Transfer Function Preparation
    if trn_mode == 2:
        mag = sqrt(g.x**2 + g.y**2 + g.z**2)
        mag = max(0.0, min(1.0, mag)) # Clamp to [0,1]
        scaled_mag = mag * 256.0
        # Transformer would receive (v, scaled_mag)
        # Here we just return the raw values as per the sampling method definition
        pass
        
    return v, g
```

## Subtopics

### Sample Grid Index
Provides low-level access to actual values stored in the grid at integer coordinates without interpolation [S2, unofficial].

### Sample Grid
Computes the interpolated value at a continuous position in the grid, typically using the underlying interpolation logic [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for nearest neighbor. $O(1)$ for trilinear (constant number of memory accesses and arithmetic operations).
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, storing both scalar and gradient fields.
*   **Strengths:** Fast access; supports gradient-based shading which improves surface appearance [S1, unofficial].
*   **Limitations:** Pre-computed gradients increase memory usage by 3x (assuming float scalars and vec3 gradients) [S1, unofficial]. Trilinear interpolation may be computationally expensive if not optimized or if hardware acceleration is unavailable [S1, unofficial].
*   **When to use:** When ray-casting requires smooth density transitions or gradient-based lighting. When memory bandwidth allows storing auxiliary gradient data.

## Background
This method originates from early volume ray-casting implementations, specifically those using CT scan data formats (`.ctscan`) [S1, unofficial]. It was developed to support 1D transfer functions and later extended to support gradient-based opacity control [S1, unofficial]. The concepts are also reflected in modern node-based geometry systems like Blender's Geometry Nodes, where "Sample Grid" and "Sample Grid Index" nodes provide similar functionality for field evaluation [S2, unofficial].

## Sources
[S1] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S2] Volume Grids in Geometry Nodes — Blender Developers Blog, https://code.blender.org/2025/10/volume-grids-in-geometry-nodes/, fetched 2026-10-27 (unofficial)
