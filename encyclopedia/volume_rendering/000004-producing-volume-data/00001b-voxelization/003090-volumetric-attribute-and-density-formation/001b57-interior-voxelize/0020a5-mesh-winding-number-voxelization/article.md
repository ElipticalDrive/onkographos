# Mesh Winding Number Voxelization
Also called: Hierarchical Mesh Winding Number Voxelization

## Overview
Mesh Winding Number Voxelization determines the interior of a 3D mesh by evaluating the Mesh Winding Number (MWN) at sample points within a voxel grid. Unlike [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) or [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md), which rely on ray-mesh intersections, this method computes a scalar field based on the solid angle subtended by the mesh triangles. It is distinct from [Front/Back Face Flagging](1506-front-back-face-flagging.md) and [Tight Voxelization (Point-in-Mesh Test)](1709-tight-voxelization-point-in-mesh-test.md) because it yields an integer value for closed meshes and smoothly diffuses around open boundaries, allowing it to correctly handle non-manifold geometry and meshes with holes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A triangular mesh (typically represented with an AABB tree for acceleration) and a target grid resolution (`numcells`).
- Output: A 3D boolean bitmap (`Bitmap3`) where `true` indicates the voxel center is inside the mesh.

**Data Structures:**
- `Bitmap3`: A 3D array of bits. Internally, bits are packed into 32-bit integers. Due to the lack of atomic read-and-flip operations on individual bits within a 32-bit integer, concurrent writes require synchronization (e.g., a `SpinLock`) to prevent race conditions [S1, unofficial].
- `ShiftGridIndexer3`: Maps grid indices $(i, j, k)$ to world coordinates $(x, y, z)$. It requires a bounding box (`bounds`) and a cell size (`cellsize`).

**Parameters:**
- `numcells`: The resolution of the voxel grid along one axis.
- `bounds`: The axis-aligned bounding box of the mesh.
- `cellsize`: Calculated as `bounds.MaxDim / numcells`.
- `threshold`: The MWN value above which a point is considered inside. Typical value: $0.5$. Values like $0.8$ may result in missing chunks for meshes with holes [S1, unofficial].

**Steps:**
1. **Calculate Grid Parameters:** Determine `cellsize` from the mesh bounding box and `numcells`. Create a `ShiftGridIndexer3` using `bounds.Min` and `cellsize`.
2. **Seed Hierarchical Cache:** Call `WindingNumber` on a point known to be outside the mesh (e.g., the origin or a point far from the mesh). This initializes the hierarchical acceleration structure [S1, unofficial].
3. **Parallel Evaluation:** Iterate over all grid indices $(i, j, k)$ in parallel.
4. **Coordinate Mapping:** For each index, compute the world coordinate $v$ using `indexer.FromGrid(idx)`.
5. **Winding Number Computation:** Compute the MWN at $v$ using the hierarchical tree structure.
6. **Voxel Assignment:** Set the voxel to `true` if `WindingNumber(v) > threshold`, otherwise `false`. Use a thread-safe set operation (e.g., `SafeSet`) to handle concurrent writes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_winding_number_voxelization(mesh, numcells):
    # 1. Setup grid parameters
    bounds = mesh.get_bounding_box()
    cellsize = bounds.max_dim / numcells
    indexer = ShiftGridIndexer3(bounds.min, cellsize)
    
    # Output bitmap dimensions are numcells + 2 in each axis [S1, unofficial]
    # Note: The source code example uses numcells+2, but standard voxelization 
    # often uses numcells. We follow the source's specific implementation detail.
    bmp = Bitmap3(Vector3i(numcells + 2, numcells + 2, numcells + 2))
    
    # 2. Seed the hierarchical cache with a point outside the mesh
    # Using origin as an example; ensure it is outside the mesh bounds
    spatial = DMeshAABBTree3(mesh)
    spatial.winding_number(Vector3d(0, 0, 0))
    
    # 3. Parallel evaluation
    def process_voxel(idx):
        # 4. Map grid index to world coordinate
        v = indexer.from_grid(idx)
        
        # 5. Compute winding number
        w = spatial.winding_number(v)
        
        # 6. Set voxel if inside (threshold 0.5)
        # Use SafeSet to avoid race conditions on BitArray [S1, unofficial]
        bmp.safe_set(idx, w > 0.5)
    
    # Execute in parallel
    g_parallel_for_each(bmp.indices(), process_voxel)
    
    return bmp
```

**Edge Cases and Pitfalls:**
- **Race Conditions:** Writing to the `Bitmap3` from multiple threads without synchronization causes data corruption because bits are packed into 32-bit integers. Use `SafeSet` or equivalent atomic operations [S1, unofficial].
- **Threshold Selection:** Using a high threshold (e.g., $0.8$) for meshes with holes can lead to incomplete voxelization ("missing chunks"). A threshold of $0.5$ is recommended [S1, unofficial].
- **Non-Hierarchical Performance:** Computing the winding number without the hierarchical AABB tree acceleration is extremely slow for large grids (e.g., >1 minute for $64^3$) [S1, unofficial].

## Subtopics

### [Generalized Winding Number Voxelization](7623-generalized-winding-number-voxelization.md)
Generalized Winding Number Voxelization constructs a binary image (voxelization) from a set of oriented point clouds in $\mathbb{R}^3$ by evaluating an implicit surface defined by the generalized winding number framework. It differs from standard mesh voxelization by operating directly on oriented point clouds without requiring a manifold mesh, using the winding number to handle non-manifold geometry and holes via smooth diffusion. It builds on [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md) by providing the specific implementation for point clouds using the libIGL backend.

## Operating characteristics
- **Time Complexity:** The hierarchical evaluation reduces the cost per query from $O(N)$ to $O(\log N)$ or better, where $N$ is the number of triangles.
- **Measured Performance:** On a 6-core/12-thread machine in Release build, computing a $64^3$ voxelization takes 1–2 seconds using the hierarchical version. The non-hierarchical version takes well over a minute [S1, unofficial].
- **Memory Complexity:** Proportional to the number of voxels ($O(V)$) plus the memory for the AABB tree ($O(N)$).
- **Strengths:** Handles non-manifold geometry, open boundaries, and self-intersections robustly. It does not require the mesh to be watertight.
- **Limitations:** Computationally more expensive per voxel than simple ray casting for closed, manifold meshes. Requires careful handling of thread safety for the output bitmap.
- **When to Use:** When the mesh has holes, is non-manifold, or when a smooth transition around boundaries is desired.
- **When Not to Use:** For simple, closed, manifold meshes where performance is critical and ray-casting methods are sufficient.

## Background
The Mesh Winding Number was introduced by Alec Jacobson, Ladislav Kavan, and Olga Sorkine-Hornung in a SIGGRAPH paper in 2013 [S1, unofficial]. It extends the concept of the 2D polygon winding number to 3D meshes, providing a robust way to determine point containment. The hierarchical implementation leverages bounding volume hierarchies to accelerate the computation for large-scale voxelization tasks.

## Sources
[S1] 3D Bitmaps, Minecraft Cubes, and Mesh Winding Numbers — gradientspace, https://www.gradientspace.com/tutorials/2017/12/14/3d-bitmaps-and-minecraft-meshes, fetched 2026-10-29 (unofficial)
