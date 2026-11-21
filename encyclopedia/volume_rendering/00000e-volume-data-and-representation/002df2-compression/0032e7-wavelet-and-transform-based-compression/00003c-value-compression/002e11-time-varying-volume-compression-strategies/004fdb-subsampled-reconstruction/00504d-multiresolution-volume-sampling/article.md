# Multiresolution Volume Sampling

Also called: Adaptive progressive spatial reconstruction, Progressive decoding, Repetitive cube scheme

## Overview
Multiresolution Volume Sampling is an adaptive spatial subsampling scheme that progressively reconstructs a 3D time-slice from a 4D base tensor to provide smooth rendering during fast time slider interactions, refining to full resolution when interaction slows [S1]. It differs from complex interpolation schemes by computing a subset of final data values and copying them to non-computed voxels, allowing previously computed values to be reused during refinement steps [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D base tensor $T_c$ (spatial dimensions $X, Y, Z$ and time dimension $W$) and a time column vector from the last TT core $C_4$ corresponding to the requested time step [S1].
*   **Output:** A 3D volume grid of values for the specified time step, initially at low resolution and progressively refined to full resolution [S1].

**Data Structures:**
The volume is conceptually divided into $2 \times 2 \times 2$ cubes [S1]. Each cube contains 8 voxels. The subsampling pattern defines which voxels are "computed" (blue in source figures) and which are "copied" (white in source figures) [S1].

**Parameters:**
*   **Initial Subsampling Ratio:** $1:8$ [S1].
*   **Intermediate Subsampling Ratio:** $1:4$ [S1].
*   **Target Resolution:** $1:1$ (full resolution) [S1].
*   **Time Budget:** $30$ ms per time step [S1].

**Steps:**
1.  **Initialize:** Start with the lowest subsampling ratio, typically $1:8$ [S1].
2.  **Define Subsampling Pattern:** Divide the 3D volume into disjoint $2 \times 2 \times 2$ cubes [S1].
3.  **Decode Subset:**
    *   Identify the subset of voxels to compute based on the current ratio. For $1:8$, this is one voxel per cube (e.g., the corner) [S1].
    *   Extract the corresponding spatial subtensor from $T_c$ for these voxels [S1].
    *   Perform tensor-vector multiplication: Multiply the partial subtensor with the time column vector of $C_4$ [S1]. This yields the values for the computed subset of voxels [S1].
4.  **Copy Values:** Copy the computed subset values to the remaining 7 voxels in each $2 \times 2 \times 2$ cube [S1]. This fills the volume at the current low resolution [S1].
5.  **Adaptive Refinement:**
    *   Measure the computation time for the current step [S1].
    *   If the time is within the budget (e.g., $< 30$ ms) and interaction is still active, increase the resolution [S1].
    *   **Refine to $1:4$:** Decode the cube's diagonal element (in addition to the corner) [S1]. Reuse the previously computed corner values [S1].
    *   **Refine to $1:1$:** Decode all remaining voxels in the cube [S1]. Reuse previously computed values [S1].
    *   Stop refining if the time budget is exceeded or if full resolution is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_volume_sampling(T_c, C4_col, time_budget_ms=30.0):
    """
    T_c: The 4D base tensor (spatial X, Y, Z, time W).
    C4_col: The time column vector from the last TT core C4.
    time_budget_ms: Maximum allowed computation time per step.
    
    Returns:
    volume: 3D array of shape (X, Y, Z) with values for the current time step.
    """
    # Initialize volume with zeros or previous frame data
    volume = create_empty_volume(T_c.spatial_shape)
    
    # Define cube size and subsampling patterns
    cube_size = 2
    # Pattern for 1:8: only (0,0,0) relative to cube start is computed
    pattern_1_8 = [(0, 0, 0)]
    # Pattern for 1:4: (0,0,0) and (1,1,1) relative to cube start are computed
    pattern_1_4 = [(0, 0, 0), (1, 1, 1)]
    # Pattern for 1:1: all 8 voxels
    pattern_1_1 = [(x, y, z) for x in range(2) for y in range(2) for z in range(2)]
    
    patterns = [pattern_1_8, pattern_1_4, pattern_1_1]
    
    # Start with lowest resolution
    current_pattern_idx = 0
    
    # Track which voxels have been computed to avoid recomputation
    # Using a boolean mask or set of coordinates
    computed_voxels = set()
    
    start_time = get_current_time_ms()
    
    while current_pattern_idx < len(patterns):
        current_pattern = patterns[current_pattern_idx]
        
        # Iterate over all cubes in the volume
        for cube_origin in get_cube_origins(T_c.spatial_shape, cube_size):
            # For each voxel in the current subsampling pattern relative to the cube
            for dx, dy, dz in current_pattern:
                voxel_coords = (
                    cube_origin.x + dx,
                    cube_origin.y + dy,
                    cube_origin.z + dz
                )
                
                # Skip if already computed in a previous refinement step
                if voxel_coords in computed_voxels:
                    continue
                
                # Extract the spatial subtensor for this single voxel
                # T_c is 4D: [X, Y, Z, W]. We fix X, Y, Z and keep W.
                # The subtensor is essentially a vector along the time dimension
                # for this specific spatial location.
                spatial_subtensor = T_c[voxel_coords.x, voxel_coords.y, voxel_coords.z, :]
                
                # Multiply subtensor with the time column vector C4
                # This is a dot product if C4_col is a vector, or tensor contraction
                # depending on the exact TT structure. Assuming standard TT matricization:
                # value = dot(spatial_subtensor, C4_col)
                value = dot_product(spatial_subtensor, C4_col)
                
                # Store the computed value
                volume[voxel_coords.x, voxel_coords.y, voxel_coords.z] = value
                computed_voxels.add(voxel_coords)
        
        # Check time budget
        elapsed_time = get_current_time_ms() - start_time
        if elapsed_time > time_budget_ms:
            break
            
        # Move to next refinement level
        current_pattern_idx += 1
        
    # Fill non-computed voxels by copying from computed ones in the same cube
    # This step ensures the volume is fully populated for rendering
    for cube_origin in get_cube_origins(T_c.spatial_shape, cube_size):
        # Find the first computed voxel in this cube to use as a source
        source_voxel = None
        for dx, dy, dz in current_pattern: # Use the pattern that got us this far
            voxel_coords = (
                cube_origin.x + dx,
                cube_origin.y + dy,
                cube_origin.z + dz
            )
            if voxel_coords in computed_voxels:
                source_voxel = voxel_coords
                break
        
        if source_voxel is None:
            # Fallback: if no voxel was computed (e.g., budget exceeded immediately),
            # copy from a default or previous frame. 
            # In practice, 1:8 always computes at least one voxel.
            continue
            
        source_value = volume[source_voxel.x, source_voxel.y, source_voxel.z]
        
        # Copy source value to all other voxels in the cube
        for dx in range(cube_size):
            for dy in range(cube_size):
                for dz in range(cube_size):
                    voxel_coords = (
                        cube_origin.x + dx,
                        cube_origin.y + dy,
                        cube_origin.z + dz
                    )
                    if voxel_coords not in computed_voxels:
                        volume[voxel_coords.x, voxel_coords.y, voxel_coords.z] = source_value
                        
    return volume

def get_cube_origins(shape, cube_size):
    """Generate origins for all 2x2x2 cubes in the volume."""
    origins = []
    for x in range(0, shape[0], cube_size):
        for y in range(0, shape[1], cube_size):
            for z in range(0, shape[2], cube_size):
                origins.append((x, y, z))
    return origins

def dot_product(a, b):
    """Standard dot product for vectors."""
    return sum(x * y for x, y in zip(a, b))

def get_current_time_ms():
    """Returns current time in milliseconds."""
    import time
    return time.time() * 1000
```

**Edge Cases and Pitfalls:**
*   **Fast Changing Time Steps:** If the time slider moves too quickly, the system may not reach full resolution reconstruction, resulting in lower-quality rendering [S1].
*   **Precision:** Copying values introduces discontinuities at cube boundaries if the underlying data is not smooth, though this is accepted for the sake of performance [S1].
*   **Boundary Conditions:** The cube division must handle volume dimensions that are not multiples of 2. The source does not specify padding or truncation strategies for non-aligned boundaries [G].

## Operating characteristics
*   **Time Complexity:** The complexity is proportional to the number of computed voxels. Initially, it is $1/8$ of the full volume, then $1/4$, then $1/1$. The adaptive nature ensures that computation stops as soon as the time budget (30 ms) is exceeded [S1].
*   **Memory Complexity:** Requires storage for the 4D base tensor $T_c$ and the TT cores. The output volume is stored at full resolution, but intermediate computations are sparse [S1].
*   **Measured Performance:** The reduced number of operations per reconstructed time step, combined with GPU acceleration, allows for interactive, flexible reconstruction of any requested time step [S1].
*   **Strengths:**
    *   Provides a dynamic overview during fast interactions [S1].
    *   Reuses computed values during refinement, avoiding redundant calculations [S1].
    *   Simpler than complex interpolation schemes [S1].
*   **Limitations:**
    *   May not achieve fully smooth and fluid rendering for quickly changing time steps [S1].
    *   Visual quality is lower at subsampled resolutions due to copying rather than interpolation [S1].
*   **When to Use:** During active sliding through time to give a dynamic overview, refining when interaction slows [S1].
*   **When Not to Use:** When high-fidelity, smooth rendering is required regardless of interaction speed, or when the time slider is static [S1].

## Background
*   **Origin:** Proposed in the context of Tensor-Train-based 4D Time-Dependent Volume Rendering (TT4D) [S1].
*   **History:** Developed to address the limitation of standard reconstruction strategies which, while achieving interactive rates for large data, may not reach fully smooth rendering for quickly changing time steps [S1].
*   **Usage:** Used in GPU-accelerated volume rendering systems where user interaction with the time selector is frequent [S1].

## Sources
[S1] TT4D: Tensor‐Train‐based 4D Time‐Dependent Volume Rendering, https://doi.org/10.1111/cgf.70471, fetched 2026-10-27
