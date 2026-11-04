# Cell Retrieval via Neighbor-Finding

Also called: 8-voxel cell intersection, Cell-based ray tracing, Forward-neighbor retrieval, Neighbor-Finding on Octree, Octree Scalar Field Mapping via Forward Neighbors, voxel to cell mapping

## Overview
This method enables isosurface reconstruction from sparse octree volumes that store only a single scalar value per leaf voxel, avoiding the 8x storage expansion required to store full 8-voxel cells explicitly [S1, S2, S3]. It solves the problem of needing 8 corner values for a cell intersection primitive when only 1 scalar value is stored per leaf node by retrieving the 7 forward neighbors at intersection time [S2, S3]. Unlike [Cell Reconstruction via Octree Hashing](1595-cell-reconstruction-via-octree-hashing.md), which uses a bitwise arithmetic hash scheme for point location, this method relies on neighbor-finding techniques to retrieve forward-neighboring voxels, skewing heavily toward O(1) performance when neighbors share a parent node [S1, S2, S3].

## How it works
**Inputs:**
- An octree structure storing single scalar values at leaf nodes [S1, S2].
- A ray traversing the octree structure [S5, unofficial].
- The voxel width $w$ (assumed uniform) [S1, S2, S3].

**Outputs:**
- Intersection points and normals for the isosurface defined within 8-voxel cells [S1, S2].

**Data Structures:**
- **Octree Node:** Contains pointers to 8 children (octants) and a scalar value if it is a leaf [S1, S2][S5, unofficial].
- **Leaf Node:** Stores a single scalar value $v$ [S1, S2][S5, unofficial].
- **Cell Primitive:** Defined by 8 voxels: the target voxel and its 7 forward neighbors (positive X, Y, Z directions) [S1, S2, S3].

**Logical Mapping:**
The scalar field is mapped to cells by logically shifting the position of all scalars backward by half a unit of voxel width [S1, S2, S3][S4, unofficial]. This re-maps the scalar field such that each voxel is treated as the lower-left corner of a cell [S4, unofficial]. Values outside the octree data are defined to be zero [S4, unofficial].

**Algorithm Steps:**
1. **Ray Traversal:** Traverse the octree using the ray [S5, unofficial]. The traversal algorithm finds the order of intersection of the X, Y, and Z mid-planes to determine the exact order of traversal of child octants [S5, unofficial].
2. **Leaf Hit Detection:** Identify the leaf node intersected by the ray [S5, unofficial]. Let this be the target voxel $V_{target}$ with scalar value $v_{target}$ [S2][S5, unofficial].
3. **Neighbor Retrieval:** Perform neighbor-finding on the octree structure to retrieve the 7 forward-neighboring voxels [S2][S5, unofficial]. These neighbors are located in the positive X, Y, and Z directions relative to $V_{target}$ [S1, S2, S3]. If a neighbor lies outside the octree bounds, its value is 0 [S4, unofficial]. If a neighbor is not a leaf, the value is determined by its leaf descendants or treated as 0 [S4, S5, unofficial].
4. **Cell Construction:** Form an 8-voxel cell using $V_{target}$ and its 7 retrieved forward neighbors [S2][S5, unofficial].
5. **Intersection:** Use the retrieved cell of 8 voxels as the intersection primitive to reconstruct the isosurface [S5, unofficial]. This typically involves computing the min/max values of the cell for early rejection and then performing a detailed intersection test if the ray passes the bounding box [S2][S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_trace_octree_cell(ray_origin, ray_direction, octree_root, voxel_width, iso_value):
    """
    Ray trace an isosurface defined within 8-voxel cells using neighbor-finding.
    """
    intersections = []
    t_min = 0.0
    t_max = float('inf')
    stack = [(octree_root, t_min, t_max)]

    while stack:
        node, t_enter, t_exit = stack.pop()
        if node is None or t_enter > t_exit:
            continue

        if node.scalar is not None:
            # 1. Identify target voxel coordinates
            tx, ty, tz = node.coords

            # 2. Retrieve 7 forward neighbors
            neighbor_offsets = [
                (0, 0, 0), (1, 0, 0), (0, 1, 0), (0, 0, 1),
                (1, 1, 0), (1, 0, 1), (0, 1, 1), (1, 1, 1)
            ]

            cell_scalars = []
            for dx, dy, dz in neighbor_offsets:
                nx, ny, nz = tx + dx, ty + dy, tz + dz
                # Point location to find neighbor leaf [G]
                neighbor_node = octree_root
                while neighbor_node is not None and neighbor_node.scalar is None:
                    child_idx = compute_child_index(neighbor_node, nx, ny, nz)
                    neighbor_node = neighbor_node.children[child_idx]
                if neighbor_node is not None:
                    cell_scalars.append(neighbor_node.scalar)
                else:
                    cell_scalars.append(0.0) # Values outside octree data are zero [S4, unofficial]

            # 3. Check cell bounds for early rejection
            min_val = min(cell_scalars)
            max_val = max(cell_scalars)
            if not (min_val <= iso_value <= max_val):
                continue

            # 4. Map logical voxel positions to physical positions
            # Shift scalars backward by half a voxel width [S1, S2, S3]
            cell_origin_x = (tx - 0.5) * voxel_width
            cell_origin_y = (ty - 0.5) * voxel_width
            cell_origin_z = (tz - 0.5) * voxel_width
            cell_size = voxel_width

            # 5. Intersect ray with the 8-voxel cell primitive
            # Define the 8 corners of the cell in physical space
            corners = []
            for cx, cy, cz in [(0,0,0), (1,0,0), (0,1,0), (0,0,1), (1,1,0), (1,0,1), (0,1,1), (1,1,1)]:
                px = cell_origin_x + cx * cell_size
                py = cell_origin_y + cy * cell_size
                pz = cell_origin_z + cz * cell_size
                corners.append((px, py, pz))

            # Perform Marching Cubes intersection on the 8 corners and scalars [G]
            hit_point, hit_normal = intersect_cell_with_ray(
                ray_origin, ray_direction, corners, cell_scalars, iso_value
            )
            if hit_point is not None:
                intersections.append((hit_point, hit_normal))
        else:
            # Internal node: push children to stack in traversal order
            for i in range(8):
                child = node.children[i]
                if child is not None:
                    child_t_enter, child_t_exit = compute_child_bounds_and_t(child, t_enter, t_exit)
                    stack.append((child, child_t_enter, child_t_exit))

    return intersections

def compute_child_index(node, x, y, z):
    # Standard octree child index calculation based on relative position [G]
    return 0

def compute_child_bounds_and_t(child, t_enter, t_exit):
    # Standard ray-box intersection bounds calculation for octree children [G]
    return t_enter, t_exit

def intersect_cell_with_ray(origin, direction, corners, scalars, iso_value):
    # Standard Marching Cubes ray intersection algorithm [G]
    return None, None
```

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Values outside the octree data are defined to be zero [S4, unofficial]. Implementers must handle cases where forward neighbors lie outside the defined volume bounds by returning zero values [S4, unofficial].
- **Neighbor-Finding Cost:** Neighbor-finding on octrees is computationally intensive compared to array indexing [S1, S2, S3]. The worst-case complexity is O(log N), but it skews heavily toward O(1) when neighbors lie within the same parent [S1, S2, S3].
- **Point Location Overhead:** Querying neighbors in an octree is more intensive than in a 3D array, requiring full top-down point location from the root in the worst case [S1, S2, S3].

## Operating characteristics
- **Time Complexity:** Full top-down point location for neighbor retrieval is $O(\log N)$, where $N$ is the number of nodes [S1, S2, S3]. However, with neighbor-finding techniques, the complexity skews heavily toward $O(1)$ when neighboring voxels lie within the same parent node [S1, S2, S3].
- **Memory Complexity:** The method avoids the 8x storage expansion required to store full cells explicitly, keeping storage close to the raw octree size [S1, S2, S3][S4, unofficial].
- **Performance:** Neighbor-finding on octree data must perform competitively with the $O(1)$ complexity of lookup on uncompressed 3D arrays [S1, S2, S3]. In practice, the algorithm is efficient because most neighbors are found within the same parent node [S1, S2, S3].
- **Strengths:** Allows ray tracing of compressed data where leaf nodes store only a single scalar value [S1, S2, S3][S4, unofficial].
- **Limitations:** Neighbor-finding on octrees is computationally intensive compared to array indexing [S1, S2, S3]. The worst-case complexity is $O(\log N)$ [S1, S2, S3].

## Background
This method was introduced by Parker et al. [S1, S2, S3][S4, unofficial] and is a fundamental algorithmic component of interactive isosurface ray tracing of large octree volumes [S1, S2, S3]. It addresses the trade-off between storage efficiency and intersection speed in volume rendering [S1, S2, S3][S4, unofficial]. The approach is detailed in "Interactive Isosurface Ray Tracing of Large Octree Volumes" [S1, S2, S3][S4, S5, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://collections.lib.utah.edu/dl_files/1d/58/1d589a03a725c9d894a3866ad32f1386f0613f9d.pdf, fetched 2026-10-28
[S4] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S5] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
