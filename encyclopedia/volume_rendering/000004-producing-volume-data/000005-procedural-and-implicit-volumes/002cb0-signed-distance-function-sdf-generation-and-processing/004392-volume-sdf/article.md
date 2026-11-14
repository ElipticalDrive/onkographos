# Volume SDF

Also called: Isocontour SDF Generation, SDF Rebuild, Volume to SDF Conversion

## Overview
The Volume SDF operation rebuilds a signed distance field (SDF) from a given isocontour within a volumetric dataset [S1]. It transforms a scalar volume (such as a fog density field) into a geometrically valid SDF where the zero-level set corresponds to the specified iso-surface, enabling fast distance queries and renormalization of distorted fields [S1]. Unlike [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md) which may focus on iterative propagation from explicit boundaries, this method operates directly on existing volumetric data to establish the distance metric relative to an internal iso-surface [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A standard volumetric primitive containing scalar values. The method does not support VDB volumes [S1].
*   **Output:** A new volume primitive of the same resolution as the input, where the boundary condition is set to SDF [S1].

**Data Structures:**
The input and output are dense 3D grids of scalar values [G]. Let $V_{in}$ be the input volume and $V_{out}$ be the output volume [G]. Both have dimensions $W \times H \times D$ [G]. Each voxel $v_{i,j,k}$ stores a single floating-point value representing the scalar field or distance [G].

**Parameters:**
1.  **Iso Surface ($I_{iso}$):** The scalar value in $V_{in}$ that defines the zero-level set of the output SDF [S1].
    *   Typical value: $0.5$ for fog volumes; $0$ for existing SDF sources [S1].
2.  **Invert:** A boolean flag. If true, the sign of the output SDF is flipped relative to the original volume values [S1].
    *   Default: False (higher input values yield higher output values) [S1].
    *   Use case: For fog volumes, set to True so high-density (white) areas become negative (inside) [S1].
3.  **Use Maximum Distance:** A boolean flag. If true, computation stops for voxels beyond a certain distance from the iso-surface [S1].
4.  **Maximum Distance ($D_{max}$):** The object-space distance limit. Voxels farther than $D_{max}$ from the iso-surface are clamped to $\pm D_{max}$ [S1].
5.  **Rebuild With Fast Iterative Method:** A boolean flag. If true, uses the Fast Iterative Method (FIM); otherwise, uses the Fast Marching Method (FMM) [S1].
    *   FIM is faster but less accurate [S1].
6.  **FIM Tolerance ($\tau_{frac}$):** A fraction of the smallest voxel size used as the convergence threshold for FIM [S1].
7.  **Max FIM Iterations ($N_{max}$):** The maximum number of iterations for FIM convergence [S1].

**Algorithm:**
The process identifies the zero-level set of the input volume at $I_{iso}$ and computes the Euclidean distance to this surface for all voxels [S1].

1.  **Identify Source Voxels:**
    Find all voxels $v_{i,j,k}$ in $V_{in}$ such that $|V_{in}[i,j,k] - I_{iso}| < \epsilon$, where $\epsilon$ is a small numerical tolerance [G]. These voxels form the initial set of known distances $d=0$ [S1].

2.  **Initialize Output:**
    Create $V_{out}$ initialized to infinity [G]. Set $V_{out}[i,j,k] = 0$ for all source voxels identified in Step 1 [S1].

3.  **Compute Distances:**
    Depending on the `Rebuild With Fast Iterative Method` parameter, execute either FMM or FIM [S1].

    **Option A: Fast Marching Method (Default)**
    *   Use a min-heap (priority queue) ordered by distance value [G].
    *   Insert all source voxels into the heap with distance 0 [G].
    *   While the heap is not empty:
        1.  Pop voxel $u$ with the smallest tentative distance $d_u$ [G].
        2.  For each neighbor $v$ of $u$ (6-connected or 26-connected, typically 6 for accuracy) [G]:
            1.  If $v$ is already finalized, skip [G].
            2.  Calculate the distance to $v$ based on the gradient of the input field between $u$ and $v$ [G].
            3.  Update $V_{out}[v]$ with the minimum of its current value and the calculated distance [G].
            4.  Push $v$ to the heap with the new distance [G].

    **Option B: Fast Iterative Method (FIM)**
    *   Initialize a set of active voxels containing all source voxels [G].
    *   Iterate $n$ from 1 to $N_{max}$:
        1.  Create a set of updated voxels $U = \emptyset$ [G].
        2.  For each voxel $v$ in the active set:
            1.  Compute the new distance $d_{new}$ for $v$ based on its neighbors' current distances in $V_{out}$. The update rule approximates the Eikonal equation $|\nabla d| = 1$ using the smallest neighbors [G].
            2.  If $|d_{new} - V_{out}[v]| > \tau_{frac} \cdot \Delta x_{min}$, where $\Delta x_{min}$ is the smallest voxel dimension in the volume [S1]:
                *   Update $V_{out}[v] = d_{new}$ [G].
                *   Add $v$ to $U$ [G].
        3.  If $U$ is empty, break (converged) [G].
        4.  Update the active set to $U$ [G].

4.  **Apply Inversion:**
    If `Invert` is true, multiply all values in $V_{out}$ by $-1$ [S1].

5.  **Apply Maximum Distance Clamping:**
    If `Use Maximum Distance` is true:
    *   For each voxel $v$ in $V_{out}$:
        *   If $V_{out}[v] > D_{max}$, set $V_{out}[v] = D_{max}$ [S1].
        *   If $V_{out}[v] < -D_{max}$, set $V_{out}[v] = -D_{max}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_sdf(input_grid, iso_surface, invert, use_max_dist, max_dist, 
               use_fim, fim_tolerance_frac, max_fim_iters):
    """
    Rebuilds a signed distance field from a volume isocontour.
    
    Args:
        input_grid: 3D list of scalars (D x H x W).
        iso_surface: Scalar value defining the zero-level set.
        invert: Boolean, flip signs if True.
        use_max_dist: Boolean, clamp distances.
        max_dist: Float, maximum distance to compute.
        use_fim: Boolean, use Fast Iterative Method.
        fim_tolerance_frac: Float, fraction of min voxel size for convergence.
        max_fim_iters: Int, max iterations for FIM.
        
    Returns:
        output_grid: 3D list of signed distances.
    """
    # Get dimensions
    D, H, W = len(input_grid), len(input_grid[0]), len(input_grid[0][0])
    
    # Initialize output with infinity
    output_grid = [[[float('inf')] * W for _ in range(H)] for _ in range(D)]
    
    # Find source voxels (iso-surface)
    epsilon = 1e-6  # [G] Small numerical tolerance for iso-surface detection
    source_voxels = []
    for d in range(D):
        for h in range(H):
            for w in range(W):
                if abs(input_grid[d][h][w] - iso_surface) < epsilon:
                    source_voxels.append((d, h, w))
                    output_grid[d][h][w] = 0.0
    
    # Voxel size assumption: uniform for simplicity; in practice read from volume primitive metadata [G]
    voxel_size = 1.0 
    fim_tolerance = fim_tolerance_frac * voxel_size
    
    if use_fim:
        # Fast Iterative Method
        active_voxels = set(source_voxels)
        
        for _ in range(max_fim_iters):
            updated_voxels = set()
            
            for (d, h, w) in active_voxels:
                # Get neighbors
                neighbors = get_6_neighbors(d, h, w, D, H, W)
                
                # Compute new distance based on neighbors
                # FIM update: d(x) = min_{n in neighbors} (d(n) + dist(x, n))
                min_dist = float('inf')
                for (nd, nh, nw) in neighbors:
                    d_n = output_grid[nd][nh][nw]
                    if d_n < min_dist:
                        min_dist = d_n
                
                new_dist = min_dist + voxel_size
                
                # Check convergence
                if abs(new_dist - output_grid[d][h][w]) > fim_tolerance:
                    output_grid[d][h][w] = new_dist
                    updated_voxels.add((d, h, w))
            
            active_voxels = updated_voxels
            if not active_voxels:
                break
    else:
        # Fast Marching Method (Default)
        import heapq
        heap = []
        
        # Add source voxels to heap
        for (d, h, w) in source_voxels:
            heapq.heappush(heap, (0.0, d, h, w))
            
        visited = set(source_voxels)
        
        while heap:
            dist, d, h, w = heapq.heappop(heap)
            
            # If already visited with a smaller distance, skip
            if (d, h, w) in visited:
                continue
            visited.add((d, h, w))
            
            # Update neighbors
            neighbors = get_6_neighbors(d, h, w, D, H, W)
            for (nd, nh, nw) in neighbors:
                if (nd, nh, nw) not in visited:
                    # Calculate distance to neighbor
                    new_dist = dist + voxel_size
                    
                    if new_dist < output_grid[nd][nh][nw]:
                        output_grid[nd][nh][nw] = new_dist
                        heapq.heappush(heap, (new_dist, nd, nh, nw))

    # Apply Inversion
    if invert:
        for d in range(D):
            for h in range(H):
                for w in range(W):
                    output_grid[d][h][w] = -output_grid[d][h][w]
        
    # Apply Maximum Distance Clamping
    if use_max_dist:
        for d in range(D):
            for h in range(H):
                for w in range(W):
                    val = output_grid[d][h][w]
                    if val > max_dist:
                        output_grid[d][h][w] = max_dist
                    elif val < -max_dist:
                        output_grid[d][h][w] = -max_dist
                        
    return output_grid

def get_6_neighbors(d, h, w, D, H, W):
    """Returns valid 6-connected neighbors."""
    neighbors = []
    for dd, hh, ww in [(-1,0,0), (1,0,0), (0,-1,0), (0,1,0), (0,0,-1), (0,0,1)]:
        nd, nh, nw = d+dd, h+hh, w+ww
        if 0 <= nd < D and 0 <= nh < H and 0 <= nw < W:
            neighbors.append((nd, nh, nw))
    return neighbors
```

## Subtopics

### SDF to Density Conversion
Converts an SDF back into a density volume, typically for rendering or fluid simulation. It is the inverse operation of creating an SDF from a density field. Choose this when you need to render an SDF as a volumetric cloud or fog. [SDF to Density Conversion](1282-sdf-to-density-conversion.md)

### Constrained Jacobi Smoothing for SDFs
Smooths an SDF while preserving the distance property. Use this when the SDF has noise or artifacts but must remain a valid distance field. [Constrained Jacobi Smoothing for SDFs](1583-constrained-jacobi-smoothing-for-sdfs.md)

### SDF Volume Generation & Ping-Pong Buffering
Generates SDFs using ping-pong buffering techniques, often for iterative or multi-pass generation. Choose this for high-performance GPU-based SDF generation or when integrating with specific rendering pipelines. [SDF Volume Generation & Ping-Pong Buffering](4570-sdf-volume-generation-ping-pong-buffering.md)

### Script-to-Shader SDF Preprocessor
Preprocesses SDF scripts for shader execution. Use this when deploying SDFs to real-time shaders. [Script-to-Shader SDF Preprocessor](6349-script-to-shader-sdf-preprocessor.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The Fast Iterative Method (FIM) is noted as generally faster than the Fast Marching Method (FMM) in practice, but convergence depends on the tolerance and bandwidth [S1].
*   **Memory Complexity:** Not given in the sources. The method stores the output volume and auxiliary data structures such as a heap or active set [G].
*   **Measured Performance:** FIM is noted as "generally faster" than FMM, but "not as accurate" [S1].
*   **Strengths:**
    *   Converts arbitrary scalar volumes (e.g., fog) into geometrically valid SDFs [S1].
    *   Renormalizes distorted SDFs [S1].
    *   Supports bandwidth limiting via `Maximum Distance` to save computation [S1].
*   **Limitations:**
    *   Only works with standard Houdini volumes; does not support VDBs [S1].
    *   FIM accuracy is lower than FMM [S1].
    *   Requires careful tuning of `FIM Tolerance` and `Max FIM Iterations` for convergence [S1].
*   **When to use:** When you have a scalar volume (like fog) and need a fast distance field for raymarching or collision detection [S1].
*   **When not to use:** When working with VDB volumes or when high geometric accuracy is required and performance is not a constraint (use FMM) [S1].

## Background
The Volume SDF operation was introduced in Houdini version 10.0 [S1]. It is part of the SOP (Surface Operator) network in SideFX Houdini, designed to manipulate volumetric geometry [S1]. The method leverages established algorithms for signed distance field generation, specifically the Fast Marching Method and the Fast Iterative Method, adapting them for volumetric data processing within a 3D animation and effects context [S1].

## Sources
[S1] Volume SDF - SideFX, https://www.sidefx.com/docs/houdini/nodes/sop/volumesdf.html, fetched 2026-10-27
