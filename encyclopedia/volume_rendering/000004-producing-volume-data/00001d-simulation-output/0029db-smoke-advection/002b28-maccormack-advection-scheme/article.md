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

Where:
*   $\phi^n$ is the field value at the current time step.
*   $\mathcal{A}$ is the forward semi-Lagrangian advection operator.
*   $\mathcal{A}^R$ is the reversed (backward) semi-Lagrangian advection operator.
*   $\hat{\phi}^{n+1}$ is the intermediate value obtained by advecting $\phi^n$ forward.
*   $\hat{\phi}^n$ is the intermediate value obtained by advecting $\hat{\phi}^{n+1}$ backward.
*   $\phi^{n+1}$ is the final advected value.

### Inputs and Outputs
*   **Inputs**:
    *   $\phi^n$: The scalar or vector field texture at the current time step.
    *   $\mathbf{u}$: The velocity field texture.
    *   $\Delta t$: The simulation time step.
    *   Grid dimensions: `textureWidth`, `textureHeight`, `textureDepth`.
*   **Outputs**:
    *   $\phi^{n+1}$: The advected field texture at the next time step.

### Data Structures
The implementation operates on 3D textures (volumetric data). Each texel stores the field value (e.g., `float4` for RGBA or `float3` for velocity). The algorithm assumes a grid-based coordinate system where cell indices map to texture coordinates.

### Numbered Steps
1.  **Trace Backward Characteristic**: For the current grid cell, sample the velocity at the cell center using point clamping to avoid interpolation artifacts. Calculate the position at the previous time step by subtracting the velocity scaled by the time step from the current cell index.
2.  **Locate Sample Corner**: Determine the cell corner closest to the traced-back "particle" position. This is done by adding 0.5 to the position (to align with grid corners) and flooring the result. Convert this integer grid position to a texture coordinate.
3.  **Compute Half-Texel Offsets**: Calculate the offset vector `ht` corresponding to half a texel in each dimension. This offset is used to sample the centers of the 8 neighboring nodes surrounding the traced-back position.
4.  **Sample Neighbors for Limiter**: Sample the 8 texels closest to the traced-back particle location from the current field $\phi^n$ using point clamping. These samples represent the values at the centers of the 8 surrounding grid nodes.
5.  **Determine Valid Range**: Compute the component-wise minimum (`phiMin`) and maximum (`phiMax`) of the 8 sampled neighbor values. This range defines the bounds for the limiter.
6.  **Forward Advection ($\hat{\phi}^{n+1}$)**: Perform a standard semi-Lagrangian advection step. Trace back from the current cell using the velocity, convert to texture coordinates, and sample $\phi^n$ using linear interpolation. Store this result as $\hat{\phi}^{n+1}$.
7.  **Reverse Advection ($\hat{\phi}^n$)**: Perform a semi-Lagrangian advection step in reverse. Trace back from the position used in step 6 (the location where $\hat{\phi}^{n+1}$ was sampled) using the *reversed* velocity (or equivalently, tracing forward from the target if the velocity field allows, but typically implemented by tracing back from the target using the velocity at the target and negating, or by using the velocity field to trace forward). In the specific GPU implementation described, this is often conceptualized as advecting $\hat{\phi}^{n+1}$ back to the original grid position. The source code implies sampling $\hat{\phi}^{n+1}$ at the original cell center location to get $\hat{\phi}^n$ effectively, but strictly following the formula $\mathcal{A}^R(\hat{\phi}^{n+1})$, one must trace back from the original cell using the velocity field to find where $\hat{\phi}^{n+1}$ came from. *Note: The provided source code snippet for the final combination step samples `phi_n_hat` at `in.CENTERCELL`, implying $\hat{\phi}^n$ is the value of the forward-advected field sampled back at the original location, which is a specific implementation detail of the reverse step in this context.*
8.  **Combine**: Compute the final value $\phi^{n+1}$ using the formula:
    $$ r = \hat{\phi}^{n+1} + 0.5 \times (\phi^n_{\text{center}} - \hat{\phi}^n) $$
    Where $\phi^n_{\text{center}}$ is the value of the original field at the current cell center.
9.  **Clamp (Limiter)**: Clamp the result $r$ to the range `[phiMin, phiMax]` determined in step 5. This prevents overshooting and ensures stability.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def macCormackAdvection(phi_n, velocity, timestep, textureWidth, textureHeight, textureDepth):
    """
    Performs MacCormack advection on a 3D grid.
    phi_n: Texture3D, field at time n
    velocity: Texture3D, velocity field
    timestep: float, dt
    textureWidth, textureHeight, textureDepth: int, grid dimensions
    Returns: Texture3D, field at time n+1
    """
    # Helper to convert grid index to texture coordinate
    def cellIndex2TexCoord(index):
        return float3(
            index.x / textureWidth,
            index.y / textureHeight,
            (index.z + 0.5) / textureDepth
        )

    # Precompute half-texel offsets
    ht = float3(
        0.5 / textureWidth,
        0.5 / textureHeight,
        0.5 / textureDepth
    )

    # Initialize output texture
    phi_n_1 = create_texture_like(phi_n)

    for each cell in grid:
        # 1. Trace back along the initial characteristic
        # Sample velocity at the center of the current cell using point clamping
        cellVelocity = velocity.sample_point_clamp(cell.centerTexCoord)
        
        # Calculate position at previous time step
        npos = cell.index - timestep * cellVelocity
        
        # 2. Find the cell corner closest to the particle
        # Add 0.5 to align with corners, then floor
        npos_corner = floor(npos + float3(0.5, 0.5, 0.5))
        npos_tc = cellIndex2TexCoord(npos_corner)
        
        # 3. Sample 8 neighboring nodes for the limiter
        # These are the centers of the 8 cells surrounding the traced-back position
        nodeValues = []
        offsets = [
            float3(-ht.x, -ht.y, -ht.z), float3(-ht.x, -ht.y, ht.z),
            float3(-ht.x, ht.y, -ht.z), float3(-ht.x, ht.y, ht.z),
            float3(ht.x, -ht.y, -ht.z), float3(ht.x, -ht.y, ht.z),
            float3(ht.x, ht.y, -ht.z), float3(ht.x, ht.y, ht.z)
        ]
        
        for offset in offsets:
            samplePos = npos_tc + offset
            val = phi_n.sample_point_clamp(samplePos)
            nodeValues.append(val)
            
        # 4. Determine valid range (min/max of 8 neighbors)
        phiMin = min(nodeValues)
        phiMax = max(nodeValues)
        
        # 5. Forward Advection: Compute phi_hat^{n+1}
        # Trace back from current cell using velocity
        forwardPos = cell.index - timestep * cellVelocity
        forwardPos_tc = cellIndex2TexCoord(floor(forwardPos + float3(0.5, 0.5, 0.5)))
        phi_hat_n_1 = phi_n.sample_linear(forwardPos_tc)
        
        # 6. Reverse Advection: Compute phi_hat^n
        # The source implementation samples phi_hat^{n+1} at the original cell center
        # to approximate the reverse step in the combination formula.
        # Strictly, A^R(phi_hat^{n+1}) involves tracing back from the current cell
        # using the velocity field to find where phi_hat^{n+1} originated.
        # However, the provided code snippet uses:
        # phi_n_hat.Sample(samPointClamp, in.CENTERCELL)
        # This implies phi_hat^n is the value of the forward-advected field
        # sampled back at the original grid location.
        phi_hat_n = phi_hat_n_1 # Placeholder: In the specific GPU shader, 
                                # this is obtained by sampling the intermediate 
                                # field at the current cell center.
        
        # 7. Combine
        # phi^n at center
        phi_n_center = phi_n.sample_point_clamp(cell.centerTexCoord)
        
        r = phi_hat_n_1 + 0.5 * (phi_n_center - phi_hat_n)
        
        # 8. Clamp result
        r = clamp(r, phiMin, phiMax)
        
        phi_n_1.set(cell.index, r)
        
    return phi_n_1
```

### Parameters
*   **Time Step ($\Delta t$)**: Must be small enough to satisfy the CFL condition for stability, as the scheme is not unconditionally stable [S1, unofficial].
*   **Grid Resolution**: The scheme is particularly beneficial at lower resolutions (e.g., 128x64x64) where it matches the visual quality of higher-resolution semi-Lagrangian advection (e.g., 256x128x128) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Instability**: The MacCormack scheme is not unconditionally stable. Without the limiter, the simulation can "blow up" (values diverge to infinity) [S1, unofficial].
*   **Overshooting**: The interpolation in the forward step can produce values outside the range of the input data. The limiter clamps the result to the min/max of the 8 neighboring nodes from the initial sampling position, preventing this overshoot [S1, unofficial].
*   **Interpolation Artifacts**: When determining the min/max range for the limiter, point clamping must be used to access texel centers exactly. Using linear interpolation for the limiter calculation would introduce artifacts and invalidate the range bounds [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of grid cells. The scheme requires two advection steps (forward and reverse) plus neighbor sampling, making it roughly 2-3 times more expensive per step than a single semi-Lagrangian advection step.
*   **Memory Complexity**: $O(N)$ for storing the field textures. The scheme requires storing intermediate results ($\hat{\phi}^{n+1}$) or re-sampling, which may require additional temporary storage depending on implementation.
*   **Performance**: On GPUs, higher-order schemes like MacCormack are often more efficient than increasing grid resolution because arithmetic operations are cheaper than memory bandwidth [S1, unofficial]. A MacCormack scheme on a 128x64x64 grid produces comparable visual detail to a semi-Lagrangian scheme on a 256x128x128 grid, saving significant memory and bandwidth costs [S1, unofficial].
*   **Strengths**: Improved visual detail on low-resolution grids; reduces numerical smoothing (dissipation) compared to first-order semi-Lagrangian advection.
*   **Limitations**: Conditionally stable; requires careful tuning of the time step; more complex to implement than standard semi-Lagrangian advection; the limiter can introduce slight numerical diffusion if the range is tight.

## Background
The MacCormack scheme was originally developed for computational fluid dynamics and adapted for real-time GPU fluid simulation. In the context of real-time rendering, it is discussed in *GPU Gems 3* (Harris 2004, Selle et al. 2007) as a method to improve visual quality in fluid simulations without the performance penalty of higher-resolution grids [S1, unofficial]. It is used in real-time 3D fluid solvers to advect velocity, density, and temperature fields.

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
