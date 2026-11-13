# Nearest Neighbor Sampling

Also called: Nearest Neighbour Sampling, Vizinho mais próximo

## Overview
Nearest Neighbor Sampling is a direct volume ray casting interpolation technique that approximates the continuous volume function $V(\mathbf{p})$ by selecting the value of the single discrete sample point closest to the current ray position [S1, unofficial][S2, unofficial]. It serves as a computationally efficient alternative to trilinear interpolation, trading image quality for reduced execution time [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: The current 3D query position $\mathbf{p} = (x, y, z)$ along the ray, and the discrete volume dataset $V$ defined on a regular grid.
*   **Output**: The scalar value $V(\mathbf{p})$ retrieved from the volume texture or grid.

**Data Structures**
The volume data is stored as a 3D array or texture where each voxel $V_{i,j,k}$ is located at integer grid coordinates $(i, j, k)$. The coordinates are typically mapped to a normalized range (e.g., $[0, 1]$ or $[-1, 1]$) before sampling [S2430].

**Steps**
1.  **Coordinate Mapping**: Ensure the query position $\mathbf{p}$ is in the local coordinate system of the volume data. If the volume uses normalized coordinates, map the world/camera space position accordingly [S2430].
2.  **Integer Conversion**: Compute the integer grid coordinates $(i, j, k)$ corresponding to $\mathbf{p}$. This is done by truncating the floating-point coordinates to their nearest lower integer (floor) or rounding to the nearest integer, depending on the specific implementation's definition of "closest" [S2, unofficial]. In many fixed-function pipelines, this is implicitly handled by the texture sampling mode.
3.  **Lookup**: Retrieve the value $V_{i,j,k}$ stored at the computed integer indices.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nearest_neighbor_sample(volume, p_x, p_y, p_z, volume_dims):
    """
    Samples the volume at position (p_x, p_y, p_z) using nearest neighbor.
    volume: 3D array or texture object
    p_x, p_y, p_z: float coordinates in the volume's local space
    volume_dims: tuple (dim_x, dim_y, dim_z) defining the grid size
    """
    # 1. Map coordinates to grid indices
    # Assuming volume coordinates are in [0, 1] range mapped to [0, dims-1]
    # If coordinates are already in grid units, skip scaling.
    
    # Calculate raw indices
    idx_x = p_x * (volume_dims[0] - 1)
    idx_y = p_y * (volume_dims[1] - 1)
    idx_z = p_z * (volume_dims[2] - 1)
    
    # 2. Round to nearest integer (or floor, depending on specific "closest" definition)
    # Standard nearest neighbor usually implies rounding to the closest integer
    i = round(idx_x)
    j = round(idx_y)
    k = round(idx_z)
    
    # 3. Boundary clamping (Pitfall: out-of-bounds access)
    i = max(0, min(i, volume_dims[0] - 1))
    j = max(0, min(j, volume_dims[1] - 1))
    k = max(0, min(k, volume_dims[2] - 1))
    
    # 4. Retrieve value
    return volume[i, j, k]
```

**Parameters**
*   **Sampling Mode**: Often controlled by a flag (e.g., `INTER_MODE`). When not set to linear interpolation modes, nearest neighbor is the default fallback [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Aliasing**: Because it does not average neighboring values, nearest neighbor sampling is prone to aliasing artifacts, particularly at oblique viewing angles or high frequencies [S1, unofficial].
*   **Boundary Conditions**: If the ray position falls outside the volume bounds, the implementation must clamp the indices to the valid range $[0, \text{dim}-1]$ to prevent memory access violations [G].
*   **Precision**: Floating-point inaccuracies in ray marching can cause jittering artifacts as the ray crosses voxel boundaries, since the sampled value jumps discontinuously [G].

## Subtopics

### Trilinear Interpolation
Combines values from 8 neighboring voxels using linear weights. Chosen when higher image quality is required and computational cost is acceptable. [Volumetric Ray Casting with Trilinear Interpolation](13406-volumetric-ray-casting-with-trilinear-interpolation.md)

### Gradient-Based Interpolation
Uses gradient information for higher-order reconstruction. Chosen for smoothness and detail preservation. [Gradient-Based Interpolation by Sample Splatting](9072-gradient-based-interpolation-by-sample-splatting.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per sample. It requires only coordinate conversion, rounding, and a single memory lookup.
*   **Memory Complexity**: $O(1)$ auxiliary memory. No additional buffers for neighbor weights are needed.
*   **Performance**: Significantly faster than trilinear or higher-order interpolation methods because it avoids multiple texture fetches and arithmetic operations for weight calculation [S1, unofficial].
*   **Strengths**: Minimal computational overhead; suitable for real-time applications on hardware with limited processing power or high frame rate requirements.
*   **Limitations**: Produces lower quality images with visible blockiness and aliasing artifacts compared to smoothing interpolators [S1, unofficial]. Not suitable for high-fidelity visualization where smooth gradients are critical.

## Background
Nearest neighbor sampling is a fundamental technique in computer graphics, originating from early texture mapping and volume rendering research. It is widely used in hardware-accelerated rendering pipelines where the `GL_NEAREST` or `TEXTURE_MIN_FILTER` modes are explicitly selected. Its simplicity makes it a common baseline for comparing more complex interpolation schemes.

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
