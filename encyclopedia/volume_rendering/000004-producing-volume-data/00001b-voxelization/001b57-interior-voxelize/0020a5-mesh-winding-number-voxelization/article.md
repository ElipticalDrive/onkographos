# Mesh Winding Number Voxelization
Also called: Hierarchical Mesh Winding Number Voxelization

## Overview
Mesh Winding Number Voxelization determines the interior of a mesh by evaluating the Mesh Winding Number (MWN) at sample points within a voxel grid. Unlike [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) or [Front/Back Face Flagging](1506-front-back-face-flagging.md), which rely on ray intersection parity or face orientation, MWN provides a robust signed distance-like field that handles meshes with holes and non-manifold geometry gracefully. It sets itself apart by yielding an integer value for closed meshes and a smoothly diffusing value around open boundaries, allowing for a consistent threshold-based classification.

## How it works
**Inputs:**
- A closed or open triangular mesh.
- A target voxel grid resolution (e.g., `numcells` per axis).
- The spatial bounds of the mesh (min/max coordinates).

**Outputs:**
- A 3D boolean bitmap (`Bitmap3`) representing the interior voxels.

**Data Structures:**
- `Bitmap3`: A 3D array of bits. Internally, bits are packed into 32-bit integers. Due to the lack of atomic read-and-flip operations on individual bits, concurrent writes require synchronization (e.g., a `SpinLock`) via a method like `SafeSet`.
- `ShiftGridIndexer3`: Maps integer grid indices $(i, j, k)$ to 3D world coordinates $(x, y, z)$. It requires a minimum corner vector and a cell size.

**Parameters:**
- `numcells`: Base resolution per axis. The output grid dimensions are typically `numcells + 2` in each axis to account for boundary padding or specific indexing schemes [S1, unofficial].
- `cellsize`: Derived from the mesh bounds. $cellsize = \frac{\max(\text{bounds.MaxDim})}{numcells}$ [S1, unofficial].
- `threshold`: The value used to classify a voxel as inside. Typical value: $0.5$. Higher values (e.g., $0.8$) may result in missing chunks for meshes with holes [S1, unofficial].

**Steps:**
1.  **Initialize Grid:** Create a `Bitmap3` with dimensions $(numcells+2, numcells+2, numcells+2)$.
2.  **Setup Indexer:** Create a `ShiftGridIndexer3` using the mesh bounds' minimum corner and the calculated `cellsize`.
3.  **Seed Cache:** Call the hierarchical `WindingNumber` function on a point known to be outside the mesh (e.g., the origin or a point far from the bounds). This initializes the internal bounding-box tree cache for efficient hierarchical evaluation [S1, unofficial].
4.  **Parallel Evaluation:** Iterate over all grid indices in parallel.
    a.  Map the grid index to a 3D world coordinate using `indexer.FromGrid(idx)`.
    b.  Compute the MWN at this coordinate using the hierarchical evaluator.
    c.  If the MWN value is greater than the threshold (e.g., $> 0.5$), set the voxel to true.
5.  **Thread Safety:** Use `SafeSet` (or equivalent atomic lock) when writing to the `Bitmap3` to prevent race conditions caused by concurrent bit manipulation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_winding_number_voxelization(mesh, numcells):
    # 1. Calculate bounds and cell size
    bounds = mesh.get_bounds()
    max_dim = max(bounds.max - bounds.min)
    cellsize = max_dim / numcells
    
    # 2. Initialize grid and indexer
    # Output size is numcells + 2 in each axis [S1, unofficial]
    grid_dims = Vector3i(numcells + 2, numcells + 2, numcells + 2)
    bitmap = Bitmap3(grid_dims)
    
    # Map grid indices to world coordinates
    indexer = ShiftGridIndexer3(bounds.min, cellsize)
    
    # 3. Seed the hierarchical cache with an outside point [S1, unofficial]
    # Use a point guaranteed to be outside, e.g., origin or bounds.min - large_offset
    outside_point = Vector3d(0, 0, 0) 
    mesh.aabb_tree.winding_number(outside_point)
    
    # 4. Parallel evaluation
    def process_voxel(idx):
        # Map grid index to 3D world coordinate
        world_pos = indexer.from_grid(idx)
        
        # Compute MWN
        wn = mesh.aabb_tree.winding_number(world_pos)
        
        # Threshold check [S1, unofficial]
        if wn > 0.5:
            # Use SafeSet to handle race conditions on BitArray [S1, unofficial]
            bitmap.safe_set(idx, True)
            
    # Execute in parallel (e.g., using a thread pool or parallel loop)
    for idx in bitmap.indices():
        process_voxel(idx)
        
    return bitmap
```

**Edge Cases and Pitfalls:**
- **Race Conditions:** Writing to a `BitArray` from multiple threads without synchronization leads to data corruption because bits are packed into 32-bit integers and cannot be read-and-flipped atomically. `SafeSet` with a `SpinLock` is required [S1, unofficial].
- **Threshold Selection:** For meshes with holes, the MWN diffuses smoothly. A threshold of $0.5$ is robust, but higher thresholds (e.g., $0.8$) can cause significant parts of the interior to be missed [S1, unofficial].
- **Performance:** The hierarchical version is critical for large grids. Non-hierarchical evaluation is significantly slower (e.g., >1 minute vs. 1-2 seconds for a $64^3$ grid on a 6-core machine) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The hierarchical evaluation reduces the cost per query from $O(N)$ (where $N$ is the number of triangles) to $O(\log N)$ or better on average, depending on the tree structure. Total time is $O(V \log N)$, where $V$ is the number of voxels.
- **Memory Complexity:** $O(V)$ for the output bitmap. The hierarchical tree adds $O(N)$ overhead.
- **Measured Performance:** On a 6-core/12-thread machine in Release build, a $64^3$ voxelization takes 1-2 seconds using the hierarchical method. The non-hierarchical version takes well over a minute [S1, unofficial].
- **Strengths:** Handles meshes with holes and non-manifold geometry gracefully. Provides a smooth transition around boundaries.
- **Limitations:** More computationally expensive per voxel than simple ray casting. Requires a hierarchical bounding-box tree structure for efficiency.
- **When to use:** When the mesh may have holes, is non-manifold, or when a robust "inside" definition is needed that doesn't rely on watertightness.
- **When not to use:** For simple, watertight meshes where speed is paramount and ray-casting or face-flagging suffices.

## Background
The Mesh Winding Number was introduced by Alec Jacobson, Ladislav Kavan, and Olga Sorkine-Hornung in a SIGGRAPH paper in 2013 [S1, unofficial]. It extends the concept of the 2D polygon winding number to 3D meshes. For closed meshes, it yields an integer value (1 inside, 0 outside), while for meshes with holes, it diffuses smoothly around the boundaries. This property makes it particularly useful for voxelization tasks where mesh quality may vary or where open boundaries are present.

## Sources
[S1] 3D Bitmaps, Minecraft Cubes, and Mesh Winding Numbers — gradientspace, https://www.gradientspace.com/tutorials/2017/12/14/3d-bitmaps-and-minecraft-meshes, fetched 2026-10-29 (unofficial)
