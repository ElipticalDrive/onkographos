# Volumetric Distance Function Computation

Also called: Distance Field Voxelization via Sweeping, Sweeping SDF

## Overview
Volumetric Distance Function Computation populates a sparse voxel grid (containing only surface-intersecting and empty voxels) with signed distance values, transforming it into a dense Signed Distance Field (SDF) where every voxel stores the distance to the nearest surface point [S1, unofficial]. It is distinguished from brute-force nearest-neighbor searches by using efficient multi-pass "sweeping" algorithms that propagate distance information across the grid, offering a significant performance advantage for large volumes [S1, unofficial].

## How it works
**Inputs:** A 3D voxel grid where each voxel is marked as either "empty" or "surface-intersecting" (derived from an initial voxelization step).
**Outputs:** A dense 3D grid where every voxel contains a signed distance value.

**Data Structures:**
*   `Grid`: A 3D array of floats, dimensions $W \times H \times D$.
*   `Mask`: A 3D array of booleans or integers, same dimensions, indicating if a voxel is surface-intersecting (1) or empty (0).

**Steps:**
1.  **Initialization:** Iterate through the grid. For every voxel $v$ marked as surface-intersecting, set $SDF[v] = 0$. For all empty voxels, initialize $SDF[v] = \infty$ [S1, unofficial].
2.  **Sweeping Passes:** Perform multiple directional sweeps over the volume to propagate distance values from surface voxels to empty voxels. A standard implementation uses two passes (forward and backward) along each axis (X, Y, Z) [S1, unofficial].
    *   **Forward Sweep:** Iterate through the grid in increasing index order. For each voxel $v$, update its distance based on its neighbors that have already been processed.
    *   **Backward Sweep:** Iterate through the grid in decreasing index order. For each voxel $v$, update its distance based on its neighbors that have already been processed.
    *   *Note:* The exact update formula depends on the specific sweeping variant (e.g., Chamfer distance approximation vs. exact Euclidean). The source notes describe the general concept of "sweepings" without providing the specific recurrence relation [S1, unofficial].
3.  **Sign Assignment:** Assign signs to the computed distances. Voxels outside the surface receive positive distances; voxels inside receive negative distances. This requires a separate occupancy test or connectivity analysis to distinguish inside from outside [S2, unofficial].

**Parameters:**
*   Number of sweeps: Typically 2-6 passes (forward/backward per axis) are sufficient for good approximations [S1, unofficial].
*   Default distance: If a region is completely isolated or all surrounding voxels are empty, a default value (e.g., 1.0) may be returned [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Sweeping algorithms often compute approximate Euclidean distances (Chamfer distances) rather than exact Euclidean distances. This can introduce artifacts in gradient-based applications [S1, unofficial].
*   **Inside/Outside Ambiguity:** Determining the sign of the distance requires knowing whether a voxel is inside or outside the mesh. Simple voxelization does not provide this; a separate ray-casting or flood-fill step is needed [S2, unofficial].
*   **Boundary Conditions:** Voxels at the grid boundary must be handled carefully to avoid accessing out-of-bounds memory during neighbor lookups.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_sdf(grid_width, grid_height, grid_depth, surface_mask):
    """
    Computes a volumetric SDF using sweeping passes.
    surface_mask: 3D array where 1 indicates surface-intersecting, 0 indicates empty.
    Returns: 3D array of signed distances.
    """
    # Initialize SDF grid
    sdf = [[[[float('inf')] for _ in range(grid_depth)] for _ in range(grid_height)] for _ in range(grid_width)]
    
    # Initialize surface voxels to 0
    for x in range(grid_width):
        for y in range(grid_height):
            for z in range(grid_depth):
                if surface_mask[x][y][z] == 1:
                    sdf[x][y][z] = 0.0
    
    # Define sweep directions (forward and backward for each axis)
    # Note: The exact update rule for sweeping is not specified in the sources [S1, unofficial].
    # A common approximation uses Chamfer distances.
    # This pseudocode outlines the structure of the sweeps.
    
    def sweep_forward():
        for x in range(grid_width):
            for y in range(grid_height):
                for z in range(grid_depth):
                    if sdf[x][y][z] == float('inf'):
                        # Update based on previous neighbors (x-1, y-1, z-1)
                        # The specific formula is not given in the sources [S1, unofficial]
                        pass

    def sweep_backward():
        for x in range(grid_width - 1, -1, -1):
            for y in range(grid_height - 1, -1, -1):
                for z in range(grid_depth - 1, -1, -1):
                    if sdf[x][y][z] == float('inf'):
                        # Update based on next neighbors (x+1, y+1, z+1)
                        # The specific formula is not given in the sources [S1, unofficial]
                        pass

    # Perform multiple sweeps
    for _ in range(2): # Typical number of passes
        sweep_forward()
        sweep_backward()
    
    # Assign signs (inside/outside)
    # This step is not detailed in the sources [S1, S2, unofficial]
    # A common approach is to ray-cast from each voxel to determine occupancy
    for x in range(grid_width):
        for y in range(grid_height):
            for z in range(grid_depth):
                if sdf[x][y][z] != float('inf'):
                    # Determine if voxel is inside or outside
                    # If inside, negate the distance
                    # If outside, keep positive
                    # Implementation detail not provided in sources [S2, unofficial]
                    pass
                    
    return sdf
```

## Subtopics

### Euclidean Distance Voxel SDF
A method that computes the exact Euclidean distance to the nearest occupied voxel (for outside points) or empty voxel (for inside points) by sampling the 27-voxel neighborhood (the voxel itself and its 26 immediate neighbors) [S2, unofficial]. It is distinguished from sweeping methods by its accuracy (exact Euclidean) versus its high computational cost (sampling 27 voxels per point) [S2, unofficial]. Choose this method when exact distance values are critical and performance is less of a concern, or for small grids. If all 27 voxels are empty, return a default distance (e.g., 1.0) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Sweeping methods are generally $O(N)$ where $N$ is the number of voxels, as each voxel is visited a constant number of times (2-6 passes) [S1, unofficial]. Euclidean neighborhood sampling is $O(27N)$, which is also linear but with a much higher constant factor [S2, unofficial].
*   **Memory Complexity:** $O(N)$ to store the dense SDF grid [S1, unofficial].
*   **Performance:** Sweeping is "very efficient" and easy to implement, making it suitable for large volumes [S1, unofficial]. Euclidean sampling is "expensive" due to the 27-voxel lookups and distance calculations [S2, unofficial].
*   **Strengths:** Sweeping provides a good balance of speed and accuracy for many applications. Euclidean sampling provides exact distances.
*   **Limitations:** Sweeping may produce approximate distances (Chamfer distances). Euclidean sampling is computationally intensive. Both methods require a separate step to determine the sign (inside/outside) of the distance [S2, unofficial].
*   **When to use:** Use sweeping for large-scale volume rendering or simulation where approximate distances are acceptable. Use Euclidean sampling for high-precision applications or small grids.

## Background
The method originates from techniques for projecting 0-genus mesh models onto a sphere, where a signed distance field is computed from an initial voxelization to guide mesh evolution towards a convex configuration [S1, unofficial]. The sweeping algorithm is described as a well-known efficient technique for computing distance functions on voxel grids, with references to surveys on the topic [S1, unofficial]. The Euclidean neighborhood sampling approach is discussed in the context of ray-marching voxel grids with proper normals, highlighting the limitations of simple voxelization (which does not produce a true signed distance field) [S2, unofficial].

## Sources
[S1] How to project a 0 genus mesh model on a sphere?, https://scicomp.stackexchange.com/questions/21048, fetched 2026-10-30 (unofficial)
[S2] Ray-Marching: Rendering a Voxel Grid with Proper Normals, https://computergraphics.stackexchange.com/questions/12143/ray-marching-rendering-a-voxel-grid-with-proper-normals, fetched 2026-10-25 (unofficial)
