# Face Normal Computation

Also called: Face Normal Calculation, computeFaceNormal, getShadingFaceNormal

## Overview
Face normal computation determines the geometric orientation of a polygonal face (typically a triangle) to provide a surface normal for shading or ray intersection handling [S1, S4, unofficial]. It solves the problem of defining a consistent surface orientation when explicit vertex normals are unavailable or when a flat, geometric normal is required for primary-ray shading in voxel or mesh-based rendering [S1, unofficial]. It is distinguished from gradient-based SDF normal estimation by relying on discrete vertex positions rather than continuous distance field derivatives, and from vertex normal averaging by producing a single constant vector for the entire face rather than an interpolated value [S1, S2, S3, unofficial].

## How it works
The method computes the geometric normal of a triangle face from its three vertex positions. It involves extracting vertex indices, gathering positions, computing the cross product of two edge vectors, and normalizing the result.

**Inputs:**
- `f`: Array of face indices, where each face is defined by three vertex indices $[i_0, i_1, i_2]$.
- `v`: Array of vertex positions, where each vertex $i$ has a 3D position vector $\mathbf{p}_i$.

**Outputs:**
- `fn`: Array of face normals, where each face $j$ has a 3D normal vector $\mathbf{n}_j$.

**Data Structures:**
- `v`: `float[N][3]` or `vec3[N]`, where $N$ is the number of vertices.
- `f`: `int[M][3]` or `ivec3[M]`, where $M$ is the number of faces.
- `fn`: `float[M][3]` or `vec3[M]`, where $M$ is the number of faces.

**Steps:**
1. Iterate over each face $j$ from $0$ to $M-1$.
2. Extract the vertex indices for face $j$: $i_0 = f[j][0]$, $i_1 = f[j][1]$, $i_2 = f[j][2]$.
3. Gather the corresponding vertex positions: $\mathbf{p}_0 = v[i_0]$, $\mathbf{p}_1 = v[i_1]$, $\mathbf{p}_2 = v[i_2]$.
4. Compute two edge vectors: $\mathbf{e}_1 = \mathbf{p}_1 - \mathbf{p}_0$ and $\mathbf{e}_2 = \mathbf{p}_2 - \mathbf{p}_0$.
5. Compute the unnormalized face normal via the cross product: $\mathbf{n}_{raw} = \mathbf{e}_1 \times \mathbf{e}_2$.
6. Check for degenerate faces. If the squared magnitude $\|\mathbf{n}_{raw}\|^2 < \epsilon$ (where $\epsilon = 10^{-20}$), substitute $\mathbf{n}_{raw} = [0.0, 0.0, 1.0]$ [S4, unofficial].
7. Normalize the normal: $\mathbf{n}_j = \frac{\mathbf{n}_{raw}}{\|\mathbf{n}_{raw}\|}$.
8. Store $\mathbf{n}_j$ in `fn[j]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_face_normals(v, f):
    """
    Compute geometric face normals for a mesh.
    
    Args:
        v: List of vertex positions, each a tuple/list of 3 floats (x, y, z).
        f: List of face indices, each a tuple/list of 3 integers.
        
    Returns:
        List of face normals, each a tuple/list of 3 floats.
    """
    epsilon = 1e-20
    num_faces = len(f)
    face_normals = []
    
    for j in range(num_faces):
        # Step 2: Extract vertex indices
        i0, i1, i2 = f[j]
        
        # Step 3: Gather vertex positions
        p0 = v[i0]
        p1 = v[i1]
        p2 = v[i2]
        
        # Step 4: Compute edge vectors
        e1 = [p1[0] - p0[0], p1[1] - p0[1], p1[2] - p0[2]]
        e2 = [p2[0] - p0[0], p2[1] - p0[1], p2[2] - p0[2]]
        
        # Step 5: Cross product
        nx = e1[1] * e2[2] - e1[2] * e2[1]
        ny = e1[2] * e2[0] - e1[0] * e2[2]
        nz = e1[0] * e2[1] - e1[1] * e2[0]
        
        # Step 6: Handle degenerate faces
        mag_sq = nx*nx + ny*ny + nz*nz
        if mag_sq < epsilon:
            nx, ny, nz = 0.0, 0.0, 1.0
            mag_sq = 1.0
        
        # Step 7: Normalize
        mag = math.sqrt(mag_sq)
        face_normals.append([nx / mag, ny / mag, nz / mag])
        
    return face_normals
```

**Edge Cases and Pitfalls:**
- **Degenerate Faces:** Triangles with zero area (collinear or identical vertices) produce zero-length normals. The method handles this by checking the squared magnitude against a threshold ($10^{-20}$) and substituting a default up vector $[0, 0, 1]$ before normalization [S4, unofficial].
- **Winding Order:** The direction of the normal depends on the winding order of the vertices. Counter-clockwise (CCW) ordering typically yields a normal pointing outward, while clockwise (CW) yields an inward normal. Consistency in winding order is critical for correct shading [S2, S3, unofficial].
- **Precision:** Using squared magnitude for the degeneracy check avoids expensive square root operations and potential division by zero [S4, unofficial].

## Subtopics

### [Vertex Normal Averaging for Shading](3735-vertex-normal-averaging-for-shading.md)
Computes a smooth shading normal by averaging vertex normals. Chose this when smooth shading is desired over flat geometric shading.

## Operating characteristics
- **Time Complexity:** $O(M)$, where $M$ is the number of faces. Each face requires a constant number of arithmetic operations.
- **Memory Complexity:** $O(M)$ to store the resulting face normals.
- **Strengths:** Simple, fast, and geometrically accurate for flat shading. Does not require pre-computed vertex normals or gradient information.
- **Limitations:** Produces faceted shading on curved surfaces. Sensitive to vertex winding order. Degenerate faces must be handled explicitly.
- **When to use:** Flat shading, collision detection, or when vertex normals are not available.
- **When not to use:** Smooth shading is required (use [Vertex Normal Averaging](3735-vertex-normal-averaging-for-shading.md) or [Gradient-Based Surface Normal Estimation](4261-gradient-based-surface-normal-estimation.md)).

## Background
Face normal computation is a fundamental technique in computer graphics for determining surface orientation. It is widely used in rasterization and ray tracing for lighting calculations. The geometric approach using the cross product of edge vectors is a standard method described in introductory computer graphics literature. The specific handling of degenerate faces with a default up vector is noted in implementations such as DreamGaussian [S4, unofficial]. The term "face normal" is often used in contrast to "vertex normal" or "shading normal" [S1, S2, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
[S3] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
[S4] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
