# Cube SDF Sampling and Sign Configuration

Also called: Cube SDF Sampling, Sign Configuration

## Overview
This method constructs the topological state of a single voxel cube for Marching Cubes by sampling a Signed Distance Field (SDF) at its eight vertices and determining which vertices lie inside or outside a specified isosurface threshold [S1, unofficial]. It differs from raw grid lookups by explicitly handling the degenerate case where an SDF value equals the threshold, ensuring a strict sign configuration to prevent ambiguity in edge traversal [S1, unofficial]. It builds on [Cube Edge Vertex Retrieval](3217-cube-edge-vertex-retrieval.md) by providing the vertex data and sign flags required to identify which edges intersect the surface [S1, unofficial].

## How it works
**Inputs:**
- `sdf`: A function evaluating the SDF value at any 3D point [S1, unofficial].
- `isoLevel`: The scalar threshold defining the isosurface (typically 0.0 for SDFs) [S1, unofficial].
- `cube_min`: The 3D coordinates of the cube's minimum corner $(x_{min}, y_{min}, z_{min})$ [S1, unofficial].
- `cube_size`: The dimensions of the cube $(s_x, s_y, s_z)$ [S1, unofficial].

**Outputs:**
- `edgeIndex`: A 12-bit integer representing the sign configuration of the cube [S1, unofficial].
- `pos[8]`: An array of 8 vertex structures containing the spatial coordinates of the cube's vertices [S1, unofficial].

**Data Structures:**
1. **Vertex Positions (`pos[8]`)**: The vertices are ordered according to a specific bit-indexing scheme used for the sign configuration integer [S1, unofficial]. The layout is:
   - `pos[0]`: $(x_{min}, y_{min}, z_{min})$
   - `pos[1]`: $(x_{min}+s_x, y_{min}, z_{min})$
   - `pos[2]`: $(x_{min}+s_x, y_{min}, z_{min}+s_z)$
   - `pos[3]`: $(x_{min}, y_{min}, z_{min}+s_z)$
   - `pos[4]`: $(x_{min}, y_{min}+s_y, z_{min})$
   - `pos[5]`: $(x_{min}+s_x, y_{min}+s_y, z_{min})$
   - `pos[6]`: $(x_{min}+s_x, y_{min}+s_y, z_{min}+s_z)$
   - `pos[7]`: $(x_{min}, y_{min}+s_y, z_{min}+s_z)$ [S1, unofficial].

2. **Edge Definitions (`edges[12]`)**: Each edge is defined by a 12-bit flag (used for bitwise operations) and two vertex indices (0-7) [S1, unofficial]. The mapping is:
   - Edge 0: Flag $1$, Vertices 0-1
   - Edge 1: Flag $2$, Vertices 1-2
   - Edge 2: Flag $4$, Vertices 2-3
   - Edge 3: Flag $8$, Vertices 3-0
   - Edge 4: Flag $16$, Vertices 4-5
   - Edge 5: Flag $32$, Vertices 5-6
   - Edge 6: Flag $64$, Vertices 6-7
   - Edge 7: Flag $128$, Vertices 7-4
   - Edge 8: Flag $256$, Vertices 0-4
   - Edge 9: Flag $512$, Vertices 1-5
   - Edge 10: Flag $1024$, Vertices 2-6
   - Edge 11: Flag $2048$, Vertices 3-7 [S1, unofficial].

**Algorithm Steps:**
1. Initialize `edgeIndex` to 0 [S1, unofficial].
2. Iterate $i$ from 0 to 7 [S1, unofficial]:
   a. Retrieve the vertex position `p` from `pos[i]` [S1, unofficial].
   b. Evaluate the SDF value: $v = \text{sdf}(p)$ [S1, unofficial].
   c. **Perturbation for Zero Values**: If $v$ is exactly equal to 0, perturb the value by adding a small epsilon $\epsilon = 10^{-6}$ to ensure a strict inequality [S1, unofficial].
   d. **Sign Check**: If the (potentially perturbed) $v < \text{isoLevel}$, set the $i$-th bit of `edgeIndex`:
      $$ \text{edgeIndex} \ |= \ (1 \ll i) $$ [S1, unofficial].
3. Return `edgeIndex` and `pos` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def cube_sdf_sampling_and_sign_configuration(sdf, iso_level, cube_min, cube_size):
    sx, sy, sz = cube_size
    xmin, ymin, zmin = cube_min
    
    # Define vertex positions according to the specific bit-ordering
    pos = [
        Vec3(xmin, ymin, zmin),
        Vec3(xmin + sx, ymin, zmin),
        Vec3(xmin + sx, ymin, zmin + sz),
        Vec3(xmin, ymin, zmin + sz),
        Vec3(xmin, ymin + sy, zmin),
        Vec3(xmin + sx, ymin + sy, zmin),
        Vec3(xmin + sx, ymin + sy, zmin + sz),
        Vec3(xmin, ymin + sy, zmin + sz)
    ]
    
    edge_index = 0
    
    for i in range(8):
        p = pos[i]
        v = sdf(p)
        
        # Perturb exact zero crossings to ensure strict sign
        if v == 0:
            v += 1e-6
            
        # Set bit if vertex is inside (value < threshold)
        if v < iso_level:
            edge_index |= (1 << i)
            
    return edge_index, pos
```

**Edge Cases and Pitfalls:**
- **Exact Zero Crossing**: If an SDF value is exactly equal to the isosurface level, the sign is ambiguous [S1, unofficial]. The method mandates perturbing the value by $10^{-6}$ [S1, unofficial]. Failure to do so may result in incorrect topology or missing geometry [S1, unofficial].
- **Vertex Ordering**: The vertex indices must strictly follow the defined layout (0-7) to ensure the `edgeIndex` bits correspond correctly to the physical edges defined in `edges[12]` [S1, unofficial]. Mismatched ordering will produce twisted or invalid triangles [S1, unofficial].

## Subtopics

### [Normalized Mesh SDF Sampling](3716-normalized-mesh-sdf-sampling.md)
Extends this method by mapping normalized coordinates $[-1, 1]$ to grid indices and handling out-of-bounds queries by approximating distance to the bounding box, preventing interpolation artifacts near volume edges.

## Operating characteristics
- **Time Complexity**: $O(1)$ per cube, as it performs a fixed number of 8 SDF evaluations and bitwise operations [G].
- **Memory Complexity**: $O(1)$, storing only 8 vertex positions and one integer flag [G].
- **Strengths**: Deterministic topology generation; handles degenerate zero-crossings explicitly [G].
- **Limitations**: Does not handle interpolation itself; relies on downstream logic (like [Cube Edge Vertex Retrieval](3217-cube-edge-vertex-retrieval.md)) to find intersection points [G]. SDF evaluation cost depends on the underlying data structure (e.g., trilinear interpolation vs. nearest neighbor) [G].

## Background
This method is a standard component of the Marching Cubes algorithm, specifically adapted for Signed Distance Fields [G]. The vertex ordering and edge flag definitions are derived from the classic implementation described by Paul Bourke [S1, unofficial]. The specific perturbation of zero values is a practical implementation detail found in the Magnus2/MeshReconstruction library [S1, unofficial].

## Sources
[S1] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
