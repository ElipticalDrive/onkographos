# Layer-Based Cube Traversal

Also called: Ping-pong layer traversal, Two-layer marching traversal

## Overview
Layer-Based Cube Traversal is a memory-optimized implementation of the Marching Cubes algorithm that processes a 3D scalar field by iterating through it one $z$-slice at a time. It solves the problem of high memory consumption in naive voxel traversal by maintaining only two layers of voxel data in memory at any given time, rather than the entire volume. This approach sets it apart from standard Marching Cubes implementations by significantly reducing the spatial complexity from $O(N^3)$ to $O(N^2)$ for a grid of size $N$, making it suitable for large datasets or constrained memory environments.

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (volume data) of dimensions $S \times S \times S$ (or $W \times H \times D$), a threshold value for isosurface extraction, and a callback function to process extracted triangles.
- **Output:** A mesh (list of vertices and indices) generated via the callback.

**Data Structures:**
The core data structure is a two-layer buffer `self.layers`.
- **Layout:** A 2D array of size $[2][S \times S]$.
- **Element Type:** Each element stores a tuple `(corner_position, sampled_value)`.
- **Access Pattern:** Elements are accessed linearly within the layer buffer: `self.layers[layer_index][y * S + x]`.
- **Corner Offsets:** A static 8x3 integer array `CORNERS` defines the relative offsets of the 8 corners of a cube.

**Algorithm Steps:**
1.  **Initialization:** Allocate `self.layers` with two layers, each containing $S \times S$ elements. Initialize the first layer with sampled scalar values and positions for $z=0$.
2.  **Layer Iteration:** Iterate through the $z$-axis from $0$ to $S-1$ (or $D-1$). Let the current layer index be $z$.
3.  **Layer Preparation:**
    -   For the current $z$, compute the scalar values and positions for the current layer ($z$) and the next layer ($z+1$).
    -   Store these in `self.layers[0]` and `self.layers[1]` respectively. Note: The source implies a specific ordering where the "current" processing layer and the "next" layer are managed via swapping.
4.  **Cube Traversal:**
    -   Iterate $y$ from $0$ to $S-2$ (assuming `size_minus_one` implies $S-1$ is the exclusive bound for cube start indices to avoid boundary issues, or $S-1$ is the max index and cubes are defined up to $S-2$).
    -   Iterate $x$ from $0$ to $S-2$.
    -   For each cube at $(x, y, z)$:
        a.  **Corner Key Computation:** Compute the 8 corner keys using the `CORNERS` lookup table. The position of corner $i$ is given by:
            $$ \text{corner}_{i} = (x + \text{CORNERS}[i][0], y + \text{CORNERS}[i][1], z + \text{CORNERS}[i][2]) $$
            where $\text{CORNERS}[i]$ provides the relative offset $(dx, dy, dz)$ for the $i$-th corner.
        b.  **Data Fetching:** Retrieve the scalar values and positions for these 8 corners from `self.layers`. The $z$-offset in `CORNERS` determines whether the corner is fetched from the current layer buffer or the adjacent layer buffer.
        c.  **Marching Cubes:** Pass the 8 corner values and positions to the standard Marching Cubes lookup table logic to determine the edge intersections and topology.
        d.  **Callback:** Pass the resulting triangle data to the provided callback function.
5.  **Layer Swap:** After processing all cubes in the current $z$-slice, swap the references or indices of `self.layers[0]` and `self.layers[1]` to reuse the buffers for the next $z$-slice. This ensures that the "next" layer becomes the "current" layer for the subsequent iteration.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def layer_based_marching_cubes(volume, size, threshold, callback):
    # CORNERS is an 8x3 array of relative offsets (0 or 1)
    # e.g., [[0,0,0], [1,0,0], [0,1,0], [1,1,0], [0,0,1], [1,0,1], [0,1,1], [1,1,1]]
    
    # Initialize two layers: [2][size * size]
    # Each element is (position, value)
    layers = [[None] * (size * size) for _ in range(2)]
    
    # Helper to get index in layer buffer
    def get_idx(x, y):
        return y * size + x

    # Helper to sample volume
    def sample_volume(x, y, z):
        # Clamp or handle boundaries as needed
        if 0 <= x < size and 0 <= y < size and 0 <= z < size:
            return (Vector(x, y, z), volume[x][y][z])
        return None

    # Pre-load first two layers if necessary, or load on demand
    # The algorithm processes z from 0 to size-1
    # It needs layer z and z+1. 
    # To handle the boundary at z=size-1, we assume the loop range or boundary checks handle it.
    
    for z in range(size - 1): # Process cubes up to z=size-2, requiring z and z+1
        # Load current layer z into layers[0]
        for y in range(size):
            for x in range(size):
                layers[0][get_idx(x, y)] = sample_volume(x, y, z)
        
        # Load next layer z+1 into layers[1]
        for y in range(size):
            for x in range(size):
                layers[1][get_idx(x, y)] = sample_volume(x, y, z + 1)
        
        # Traverse cubes in this slice
        for y in range(size - 1):
            for x in range(size - 1):
                # Fetch 8 corners
                corners_data = []
                for i in range(8):
                    dx, dy, dz = CORNERS[i]
                    # Determine which layer buffer to fetch from
                    # If dz == 0, fetch from layers[0], else layers[1]
                    layer_idx = 0 if dz == 0 else 1
                    abs_x = x + dx
                    abs_y = y + dy
                    # Note: The source uses self.layers[layer_index][y * self.size + x]
                    # Here, the local x,y are relative to the cube start.
                    # The buffer stores the whole slice.
                    idx = get_idx(abs_x, abs_y)
                    corners_data.append(layers[layer_idx][idx])
                
                # Extract triangles using standard Marching Cubes logic
                # Pass corners_data to callback
                triangles = marching_cubes_step(corners_data, threshold)
                callback(triangles)
        
        # Swap layers for next iteration
        # In Python, swapping references is efficient
        layers[0], layers[1] = layers[1], layers[0]
```

**Edge Cases and Pitfalls:**
-   **Boundary Conditions:** The loops for $x$ and $y$ run up to `size_minus_one` (i.e., $S-2$). This implies that cubes at the very edge of the grid (where $x=S-1$ or $y=S-1$) are not processed, or the grid is padded. If the volume is exactly $S \times S \times S$, the last row and column of cubes are skipped to avoid out-of-bounds access when fetching adjacent corners. Implementers must ensure this margin is intentional or pad the volume.
-   **Precision:** Floating-point precision in position storage may affect mesh continuity if not handled consistently.
-   **Memory Layout:** The linear indexing `y * size + x` assumes row-major order.

## Subtopics

### [Smart Vertex Interpolation with Auxiliary Arrays](1137-smart-vertex-interpolation-with-auxiliary-arrays.md)
Uses auxiliary arrays to optimize vertex interpolation. Choose when interpolation speed is critical.

### [Marching Cubes Edge Intersection and Topology Test](2251-marching-cubes-edge-intersection-and-topology-test.md)
Focuses on robust edge intersection calculation. Choose when topological correctness is paramount.

## Operating characteristics
-   **Time Complexity:** $O(N^3)$, where $N$ is the grid dimension. Each voxel is processed a constant number of times.
-   **Memory Complexity:** $O(N^2)$. Only two layers of size $N \times N$ are stored, reducing memory usage from $O(N^3)$ to $O(N^2)$.
-   **Strengths:** Significantly reduced memory footprint, allowing processing of larger volumes on limited hardware.
-   **Limitations:** Requires careful handling of boundary conditions. The layer-swap mechanism adds slight overhead compared to direct array access.
-   **When to Use:** Large volumes where memory is a constraint.
-   **When Not to Use:** Small volumes where memory is not an issue, or when random access to arbitrary slices is required.

## Background
The method is described in the context of "Feature Sensitive Surface Extraction from Volume Data" [S1, unofficial]. It is an optimization technique for the Marching Cubes algorithm, focusing on memory efficiency rather than geometric fidelity improvements.

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
