# Sphere Carving

Also called: Carve Sphere, Voxel Sphere Removal

## Overview
Sphere Carving removes solid voxels within a spherical radius from a center point in a voxel chunk, primarily used for destructible terrain scenarios [S1, unofficial]. It differs from its parent [3D Filled Circle Kernel Generation](1520-3d-filled-circle-kernel-generation.md) by operating directly on the chunk data to modify state (removing voxels) rather than generating a static binary kernel, and from its sibling [3D Binary Sphere Kernel Generation](4654-3d-binary-sphere-kernel-generation.md) by returning a count of removed elements rather than a mask [S1, unofficial].

## How it works
**Inputs:**
- `chunk`: The voxel chunk data structure to modify [S1, unofficial].
- `cx, cy, cz`: The center coordinates of the sphere [S1, unofficial].
- `radius`: The radius of the sphere to carve [S1, unofficial].

**Outputs:**
- The modified `chunk` with voxels set to 0 (air) inside the sphere [S1, unofficial].
- An integer count of the number of solid voxels removed [S1, unofficial].

**Data Structures:**
The `chunk` is assumed to be a 3D array or linear buffer indexed by `(x, y, z)` [G]. The specific internal layout (e.g., row-major vs column-major) is not specified in the sources, but access is via `chunk.get(x, y, z)` and `chunk.set(x, y, z, value)` [S1, unofficial]. A value of `0` represents air/empty space; any non-zero value represents solid material [S1, unofficial].

**Algorithm Steps:**
1. Calculate the squared radius: $r_2 = \text{radius} \times \text{radius}$ [S1, unofficial].
2. Determine the bounding box of the sphere within the chunk boundaries for each axis $v \in \{x, y, z\}$:
   - Lower bound: $\text{lo}(v) = \max(0, \lfloor v - \text{radius} \rfloor)$ [S1, unofficial].
   - Upper bound: $\text{hi}(v) = \min(\text{chunk.size} - 1, \lceil v + \text{radius} \rceil)$ [S1, unofficial].
3. Initialize a counter `removed_count` to 0 [G].
4. Iterate through the bounding box:
   - For $z$ from $\text{lo}(cz)$ to $\text{hi}(cz)$:
     - For $y$ from $\text{lo}(cy)$ to $\text{hi}(cy)$:
       - For $x$ from $\text{lo}(cx)$ to $\text{hi}(cx)$:
         - Calculate the squared distance from the center: $d_2 = (x - cx)^2 + (y - cy)^2 + (z - cz)^2$ [G].
         - If $d_2 > r_2$, continue to the next voxel [S1, unofficial].
         - If `chunk.get(x, y, z)` is not 0:
           - Increment `removed_count` [S1, unofficial].
           - Set `chunk.set(x, y, z, 0)` [S1, unofficial].
5. Return `removed_count` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sphere_carving(chunk, cx, cy, cz, radius):
    r2 = radius * radius
    removed_count = 0
    
    # Calculate bounds for each axis
    lo_x = max(0, math.floor(cx - radius))
    hi_x = min(chunk.size - 1, math.ceil(cx + radius))
    
    lo_y = max(0, math.floor(cy - radius))
    hi_y = min(chunk.size - 1, math.ceil(cy + radius))
    
    lo_z = max(0, math.floor(cz - radius))
    hi_z = min(chunk.size - 1, math.ceil(cz + radius))
    
    for z in range(lo_z, hi_z + 1):
        dz = z - cz
        dz2 = dz * dz
        for y in range(lo_y, hi_y + 1):
            dy = y - cy
            dy2 = dy * dy
            for x in range(lo_x, hi_x + 1):
                dx = x - cx
                d2 = dx*dx + dy2 + dz2
                
                if d2 > r2:
                    continue
                
                if chunk.get(x, y, z) != 0:
                    removed_count += 1
                    chunk.set(x, y, z, 0)
                    
    return removed_count
```

**Parameters:**
- `radius`: The radius of the carving sphere [S1, unofficial].
- `cx, cy, cz`: Center coordinates [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Clamping:** The bounds must be clamped to the chunk dimensions to prevent out-of-bounds access [S1, unofficial].
- **Degenerate Input:** A radius of 0 results in checking only the exact center voxel. If the center is solid, it is removed; otherwise, nothing changes [S1, unofficial].

## Subtopics

### [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md)
Voxel Deformation with Spherical Falloff modifies scalar voxel values within a spherical region around a 3D hit point to add or remove terrain mass, enabling real-time digging and building operations. It sets itself apart from its parent [Sphere Carving](2377-sphere-carving.md) by using a continuous soft falloff function to create smooth edges rather than binary inclusion/exclusion, and by supporting additive (build) as well as subtractive (dig) operations.

### Bounding Ellipsoids from SDFs
Generating a bounding volume as a union of ellipsoids by replacing the convex hull generation step with minimum-volume enclosing ellipsoid generation on the point set $P$ [S2]. The signed distance function for an ellipsoid $E$ is defined as $d(\mathbf{p}, \mathcal{E}) = a (\|\mathbf{A}(\mathbf{p} - \mathbf{c})\| - 1)$, where $\mathbf{A} = \mathbf{D}(1/a, 1/b, 1/c) \mathbf{R}^t$ [S2]. This function is 1-Lipschitz (conservative) [S2]. Evaluation requires approximately 4 scalar products and one square root [S2]. Ellipsoids are more compact in memory and computationally more efficient than convex polyhedra (which require $n$ scalar products for $n$ faces), though potentially more conservative depending on axis lengths [S2].

### Voxel Edit Brush Operations
Four brush operations (Place, Carve, Fill, Paint) are applied over a world-space spherical radius on an unbounded voxel grid [S3, unofficial]. Voxels are addressed using unbounded world-grid coordinates rather than chunk-local indices to ensure consistent behavior across chunk boundaries [S3, unofficial]. Edits are accumulated into a stroke across frames, and a single VoxelEditCommand is pushed on release [S3, unofficial]. The brush radius defines the spherical area of influence for the edit operations [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(r^3)$ where $r$ is the radius, as it iterates over the bounding box of the sphere [G].
- **Memory Complexity:** $O(1)$ auxiliary space, modifying the chunk in place [G].
- **Performance:** The use of squared distance comparisons avoids expensive square root operations [S1, unofficial].
- **Strengths:** Suitable for real-time destructible terrain scenarios [S1, unofficial].
- **Limitations:** Produces a hard spherical boundary due to binary voxel removal [S1, unofficial]. Performance degrades with very large radii due to the cubic growth of the bounding box volume [G].

## Background
Sphere Carving is a fundamental operation in voxel-based game engines and simulations, particularly for destructible environments [S1, unofficial]. It is often used in conjunction with voxelization and geometry processing pipelines [S1, unofficial].

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S2] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S3] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
