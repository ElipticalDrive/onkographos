# MacCormack Advection Scheme

Also called: MacCormack routine

## Overview
The MacCormack advection scheme is a higher-order numerical method used to advect scalar fields (such as smoke density or temperature) and vector fields (velocity) in fluid simulations, designed to reduce the numerical smoothing inherent in standard semi-Lagrangian advection [S1, unofficial]. It achieves this by combining a forward semi-Lagrangian step with a backward correction step, allowing low-resolution grids to retain visual detail comparable to significantly higher-resolution grids using standard first-order methods [S1, unofficial]. Unlike the unconditionally stable semi-Lagrangian scheme, the MacCormack scheme is conditionally stable and requires a limiter to prevent numerical blow-up [S1, unofficial].

## How it works
The method advects a quantity $\phi$ from time step $n$ to $n+1$ using two intermediate semi-Lagrangian advection operations. The core update formula is:

$$
\begin{aligned}
\hat{\phi}^{n+1} &= \mathcal{A}(\phi^n) \\
\hat{\phi}^n &= \mathcal{A}^R(\hat{\phi}^{n+1}) \\
\phi^{n+1} &= \hat{\phi}^{n+1} + \frac{1}{2}(\phi^n - \hat{\phi}^n)
\end{aligned}
$$

Where $\mathcal{A}$ is the forward semi-Lagrangian advection operator, $\mathcal{A}^R$ is the reversed (backward) semi-Lagrangian advection operator, and $\phi^n$ is the field value at the current time step [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   $\phi^n$: The scalar or vector field texture at the current time step [S1, unofficial].
    *   $\mathbf{u}$: The velocity field texture [S1, unofficial].
    *   $\Delta t$: The simulation time step [S1, unofficial].
    *   Grid dimensions: `textureWidth`, `textureHeight`, `textureDepth` [S1, unofficial].
*   **Outputs**:
    *   $\phi^{n+1}$: The advected field texture at the next time step [S1, unofficial].

### Data Structures
The implementation operates on 3D textures (volumetric data) where each texel stores the field value (e.g., `float4` or `float3`) [S1, unofficial]. The algorithm assumes a grid-based coordinate system where cell indices map to texture coordinates via `cellIndex2TexCoord` [S1, unofficial].

### Numbered Steps
1.  **Trace Backward Characteristic**: For the current grid cell, sample the velocity at the cell center using point clamping to avoid interpolation artifacts [S1, unofficial]. Calculate the position at the previous time step by subtracting the velocity scaled by the time step from the current cell index: `npos = in.cellIndex – timestep * cellVelocity` [S1, unofficial].
2.  **Locate Sample Corner**: Determine the cell corner closest to the traced-back "particle" position by adding 0.5 to the position (to align with grid corners) and flooring the result, then convert this integer grid position to a texture coordinate: `npos = floor(npos + float3(0.5f, 0.5f, 0.5f)); npos = cellIndex2TexCoord(npos);` [S1, unofficial].
3.  **Compute Half-Texel Offsets**: Calculate the offset vector `ht` corresponding to half a texel in each dimension: `ht = float3(0.5f / textureWidth, 0.5f / textureHeight, 0.5f / textureDepth)` [S1, unofficial].
4.  **Sample Neighbors for Limiter**: Sample the 8 texels closest to the traced-back particle location from the current field $\phi^n$ using point clamping at offsets relative to the cell corner: `nodeValues[k] = phi_n.Sample(samPointClamp, npos + offset)` for k=0..7 [S1, unofficial].
5.  **Determine Valid Range**: Compute the component-wise minimum (`phiMin`) and maximum (`phiMax`) of the 8 sampled neighbor values to establish the clamping bounds [S1, unofficial].
6.  **Forward Advection ($\hat{\phi}^{n+1}$)**: Perform a standard semi-Lagrangian advection step by sampling $\phi^n$ at the traced-back position using linear interpolation to obtain $\hat{\phi}^{n+1}$ [S1, unofficial].
7.  **Reverse Advection & Combine**: Compute $\hat{\phi}^n$ by sampling the forward-advected field $\hat{\phi}^{n+1}$ at the original cell center using point clamping [S1, unofficial]. Combine the intermediate values using the update formula: `r = phi_n_1_hat.Sample(samLinear, nposTC) + 0.5 * (phi_n.Sample(samPointClamp, in.CENTERCELL) - phi_n_hat.Sample(samPointClamp, in.CENTERCELL))` [S1, unofficial].
8.  **Clamp (Limiter)**: Clamp the result `r` to the range `[phiMin, phiMax]` to prevent overshooting and ensure stability: `r = max(min(r, phiMax), phiMin)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def macCormack_advection(phi_n, velocity, timestep, texture_width, texture_height, texture_depth):
    """
    Performs MacCormack advection on a 3D grid.
    phi_n: dict mapping (x,y,z) to float, field at time n
    velocity: dict mapping (x,y,z) to (vx,vy,vz), velocity field
    timestep: float, dt
    texture_width, texture_height, texture_depth: int, grid dimensions
    Returns: dict mapping (x,y,z) to float, field at time n+1
    """
    def cell_index_to_tex_coord(index):
        return (index[0] / texture_width, index[1] / texture_height, (index[2] + 0.5) / texture_depth)

    ht = (0.5 / texture_width, 0.5 / texture_height, 0.5 / texture_depth)
    offsets = [
        (-ht[0], -ht[1], -ht[2]), (-ht[0], -ht[1], ht[2]),
        (-ht[0], ht[1], -ht[2]), (-ht[0], ht[1], ht[2]),
        (ht[0], -ht[1], -ht[2]), (ht[0], -ht[1], ht[2]),
        (ht[0], ht[1], -ht[2]), (ht[0], ht[1], ht[2])
    ]

    phi_n_1 = {}

    for cell_index in sorted(phi_n.keys()):
        # 1. Trace back along initial characteristic [S1, unofficial]
        cell_center_tc = cell_index_to_tex_coord((cell_index[0] + 0.5, cell_index[1] + 0.5, cell_index[2] + 0.5))
        cell_velocity = velocity[cell_index]
        npos = (cell_index[0] - timestep * cell_velocity[0],
                cell_index[1] - timestep * cell_velocity[1],
                cell_index[2] - timestep * cell_velocity[2])

        # 2. Locate sample corner [S1, unofficial]
        npos_corner = (math.floor(npos[0] + 0.5), math.floor(npos[1] + 0.5), math.floor(npos[2] + 0.5))
        npos_tc = cell_index_to_tex_coord(npos_corner)

        # 3. Sample 8 neighbors for limiter [S1, unofficial]
        node_values = [sample_point_clamp(phi_n, (npos_tc[0] + off[0], npos_tc[1] + off[1], npos_tc[2] + off[2])) for off in offsets]
        phi_min = min(node_values)
        phi_max = max(node_values)

        # 4. Forward advection: phi_hat^{n+1} [S1, unofficial]
        phi_hat_n_1 = sample_linear(phi_n, npos_tc)

        # 5. Reverse advection & Combine [S1, unofficial]
        phi_hat_n = sample_point_clamp(phi_hat_n_1, cell_center_tc)
        phi_n_center = sample_point_clamp(phi_n, cell_center_tc)
        r = phi_hat_n_1 + 0.5 * (phi_n_center - phi_hat_n)

        # 6. Clamp (Limiter) [S1, unofficial]
        r = max(min(r, phi_max), phi_min)

        phi_n_1[cell_index] = r

    return phi_n_1

def sample_linear(texture, coord):
    # [G] Standard trilinear interpolation of texture at coord
    return texture.get(coord, 0.0)

def sample_point_clamp(texture, coord):
    # [G] Point sampling with clamping to texture bounds
    return texture.get(coord, 0.0)
```

### Parameters
*   **Grid Resolution**: The scheme is particularly beneficial at lower resolutions (e.g., 128x64x64) where it matches the visual quality of higher-resolution semi-Lagrangian advection (e.g., 256x128x128) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Instability**: The MacCormack scheme is not unconditionally stable. Without the limiter, the simulation can "blow up" (values diverge to infinity) [S1, unofficial].
*   **Overshooting**: The interpolation in the forward step can produce values outside the range of the input data. The limiter clamps the result to the min/max of the 8 neighboring nodes from the initial sampling position, preventing this overshoot [S1, unofficial].
*   **Interpolation Artifacts**: When determining the min/max range for the limiter, point clamping must be used to access texel centers exactly to avoid getting interpolated values that would invalidate the range bounds [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Performance**: On GPUs, higher-order schemes are often a better way to get improved visual detail than simply increasing the grid resolution, because arithmetic operations are cheaper than memory bandwidth [S1, unofficial]. A MacCormack scheme on a 128x64x64 grid produces comparable visual detail to a semi-Lagrangian scheme on a 256x128x128 grid, saving significant memory and bandwidth costs [S1, unofficial].
*   **Strengths**: Not given in the sources.
*   **Limitations**: Not given in the sources.

## Background
The scheme is discussed in *GPU Gems 3* (Selle et al. 2007) as a method to improve visual quality in real-time GPU fluid simulations without the performance penalty of higher-resolution grids [S1, unofficial]. It is applied to both velocity and smoke density fields in the referenced GPU-based fluid solver [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
