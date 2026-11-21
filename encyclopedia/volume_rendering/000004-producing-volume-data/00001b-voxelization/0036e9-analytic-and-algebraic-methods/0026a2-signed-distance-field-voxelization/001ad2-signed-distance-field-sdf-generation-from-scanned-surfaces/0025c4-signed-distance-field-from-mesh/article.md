# Signed Distance Field from Mesh

Also called: Mesh SDF Rendering, MeshSDF, Signed Distance Field (SDF) for static meshes, Signed Mesh SDF, Triangle Mesh SDF

## Overview
This method converts a triangulated surface mesh into a volumetric Signed Distance Field (SDF) texture, enabling the mesh to be rendered as an implicit surface via ray marching [S1, unofficial][S5]. It solves the problem of rendering complex static geometry with robust shading features like dynamic ambient occlusion and soft shadows without runtime generation overhead [S5]. Unlike [Slice-based SDF Voxelization](1736-slice-based-sdf-voxelization.md), which processes the mesh slice-by-slice, this method typically computes the distance for every voxel in a volume texture by querying the mesh geometry directly, often using GPU-accelerated ray tracing or brute-force triangle iteration [S2, unofficial][S5].

## How it works
The generation process is performed offline to create a static volume texture that stores the signed distance from any point in space to the nearest point on the mesh surface [S5]. The sign is negative inside the mesh and positive outside [S5].

### Inputs and Outputs
*   **Input:** A triangulated mesh (e.g., STL, OBJ) [S1, unofficial]. The mesh is assumed to be closed for correct sign determination, though open meshes can be processed with specific handling [S3, unofficial][S4].
*   **Output:** A 3D volume texture (e.g., `res x res x res`) where each texel stores the signed distance [S5].

### Data Structures
*   **Volume Texture:** Stores the SDF values. The maximum size for a single mesh's volume texture is 8 megabytes, corresponding to a resolution of $128 \times 128 \times 128$ [S5].
*   **Mesh Representation:** The mesh is loaded into a structure supporting fast distance queries, such as an AABB tree or a raw vertex buffer [S1, unofficial][S4]. In GPU implementations, vertices may be stored in a texture (`meshtex`) with a layout of 3 floats (RGB) per texel and a stride of 16384 texels per row [S2, unofficial].

### Parameters
*   **Resolution (`res`):** The dimensions of the volume texture. Default is often 50, but higher resolutions capture more detail at the cost of memory [S2, unofficial][S5].
*   **Distance Field Resolution Scale:** A multiplier controlling the quality of the mesh distance field expression [S5].
*   **Is Closed:** A boolean flag indicating if the input mesh is closed, used to determine the sign of the distance field [S4].
*   **Truncation Distance (`trunc_dist`):** For open meshes, the distance past the rim where a "skirt" of signed distance is generated [S3, unofficial].

### Steps
1.  **Load Mesh:** Load the mesh file into a `MeshSDF` object or equivalent structure. Compute the bounding box and build acceleration structures (e.g., AABB tree) if necessary [S1, unofficial][S4].
2.  **Initialize Volume:** Allocate a 3D texture of size `res` x `res` x `res` [S5].
3.  **Iterate Voxels:** For each voxel center `p` in the volume:
    a. Map voxel coordinates to the mesh's local space (typically normalized to $[-1, 1]^3$) [S2, unofficial].
    b. Compute the unsigned distance $d$ from `p` to the closest point on the mesh surface [S4].
    c. Determine the sign of the distance based on the mesh topology and orientation [S3, unofficial][S4].
    d. Store the signed distance in the volume texture [S5].

### Formulas and Distance Computation
The core of the method is computing the distance from a point $p$ to the mesh.

**Triangle Distance (GPU/Shader approach):**
For a triangle with vertices $v_0, v_1, v_2$, let $\mathbf{e1} = v_1 - v_0$ and $\mathbf{e2} = v_2 - v_0$. The distance $d$ is computed by finding the closest point on the triangle [S2, unofficial]:
$$ \mathbf{proj} = \mathbf{e1} \cdot \text{clamp}\left(\frac{\mathbf{e1} \cdot (p - v_0)}{\mathbf{e1} \cdot \mathbf{e1}}, 0, 1\right) - (p - v_0) $$
If the point is outside the triangle's projection, $d = \|\mathbf{proj}\|$. If inside, $d$ is the height:
$$ d = \frac{|\mathbf{n} \cdot (p - v_0)|}{\|\mathbf{n}\|} $$
where $\mathbf{n} = \mathbf{e1} \times \mathbf{e2}$ is the face normal [S2, unofficial].

**Ray-Triangle Intersection (Möller-Trumbore):**
Used to determine if a point is inside or outside, or for ray marching [S2, unofficial]:
$$ \mathbf{pvec} = \mathbf{dir} \times \mathbf{e2}, \quad det = \mathbf{e1} \cdot \mathbf{pvec} $$
If $|det| < 10^{-8}$, the ray is parallel. Otherwise:
$$ u = \frac{(\mathbf{tvec} \cdot \mathbf{pvec})}{det}, \quad v = \frac{(\mathbf{dir} \cdot \mathbf{qvec})}{det} $$
Hit if $0 \le u, v$ and $u + v \le 1$ [S2, unofficial].

**Sign Determination:**
*   **Normal-based:** For closed meshes with counter-clockwise winding (viewed from outside), the sign is determined by the closest triangle's face normal. If the normal points towards $p$, $p$ is outside (positive); otherwise, inside (negative) [S3, unofficial].
*   **Side-of-Triangle-Mesh:** Using a functor like `Side_of_triangle_mesh`, if the mesh is closed and $p$ is on the bounded side, negate the distance: $d = -d$ [S4].
*   **Ray Casting:** Cast a ray from $p$ along an axis. If the number of intersections with the mesh is odd, $p$ is inside [S2, S3, unofficial]. Note: Some implementations ignore this and output absolute distance, losing the sign [S2, unofficial].

### Pseudocode
The following pseudocode implements the brute-force GPU approach for generating an SDF from a mesh stored in a texture [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh_sdf(mesh_vertices, res):
    """
    Generates a Signed Distance Field volume from a mesh.
    mesh_vertices: List of triangle vertex triples (v0, v1, v2).
    res: Resolution of the output volume.
    """
    volume = empty_3d_array(res, res, res)
    
    # Precompute mesh data for fast access
    triangles = []
    for i in range(0, len(mesh_vertices), 3):
        v0 = mesh_vertices[i]
        v1 = mesh_vertices[i+1]
        v2 = mesh_vertices[i+2]
        triangles.append((v0, v1, v2))
        
    # Determine bounding box for normalization
    min_pt = min(mesh_vertices)
    max_pt = max(mesh_vertices)
    center = (min_pt + max_pt) / 2.0
    scale = 2.0 / (max_pt - min_pt) # Assuming uniform scaling for simplicity
    
    for z in range(res):
        for y in range(res):
            for x in range(res):
                # 1. Calculate voxel center in [-1, 1]^3
                pos = vec3(x, y, z) / res * 2.0 - 1.0
                
                # 2. Transform to mesh space (inverse of normalization)
                # Note: This assumes the mesh was normalized to [-1, 1]^3 during load
                # If not, transform pos by mesh's inverse transform
                p = pos # Placeholder for actual transformation
                
                # 3. Initialize distance to infinity
                dist = 10000.0
                
                # 4. Loop through all triangles
                for (v0, v1, v2) in triangles:
                    # Compute distance to triangle
                    d = distance_to_triangle(p, v0, v1, v2)
                    if d < dist:
                        dist = d
                        
                # 5. Determine sign (simplified: assumes closed mesh, normal check omitted for brevity)
                # In a full implementation, check if p is inside using ray casting or normal dot product
                sign = 1.0
                if is_inside(p, triangles):
                    sign = -1.0
                    
                volume[x, y, z] = sign * dist
                
    return volume

def distance_to_triangle(p, v0, v1, v2):
    """
    Computes the squared distance from point p to triangle v0-v1-v2.
    Returns unsigned distance.
    """
    e1 = v1 - v0
    e2 = v2 - v0
    pa = p - v0
    
    # Project p onto the plane of the triangle
    n = cross(e1, e2)
    n_len_sq = dot(n, n)
    
    if n_len_sq == 0:
        return 10000.0 # Degenerate triangle
        
    # Distance to plane
    dist_plane = abs(dot(n, pa)) / sqrt(n_len_sq)
    
    # Check if projection is inside the triangle
    # Using barycentric coordinates or edge functions
    # Simplified: check if p is within the circumcircle or use edge tests
    # For robustness, use the udTriangle logic from [S2, unofficial]
    
    # Edge tests
    ba = v0 - v1
    cb = v2 - v1
    ac = v1 - v0
    
    # Check if p is outside any edge
    # This is a simplified check; full implementation requires precise barycentric logic
    # If outside, distance to closest edge/vertex
    # If inside, distance to plane
    
    # For the purpose of this article, we assume a robust library function `udTriangle` is used [S2, unofficial]
    return udTriangle(p, v0, v1, v2)

def is_inside(p, triangles):
    """
    Determines if point p is inside the mesh by ray casting.
    """
    hits = 0
    dir = vec3(1.0, 0.0, 0.0) # Cast along X-axis
    for (v0, v1, v2) in triangles:
        if ray_triangle_intersect(p, dir, v0, v1, v2):
            hits += 1
    return hits % 2 == 1
```

### Edge Cases and Pitfalls
*   **Open Meshes:** Open meshes grow a "skirt" of signed distance up to `trunc_dist` past their rim [S3, unofficial]. The sign may be incorrect near the rim.
*   **Winding Order:** The mesh must have consistent counter-clockwise winding (viewed from outside) for the normal-based sign to work correctly [S3, unofficial]. Inside-out meshes remain inside-out.
*   **Sharp Corners:** Edges/corners with faces meeting at under 90 degrees may take the wrong side due to nearest-face tie-breaking by lowest index [S3, unofficial]. A 90-degree edge off-grid axes also fails: past a corner on the line continuing an edge, a tied face's normal is perpendicular to the offset, and rounding picks the wrong side [S3, unofficial].
*   **Thin Surfaces:** Thin surfaces can only be expressed using negative texels inside the mesh to find the root. Increasing resolution captures more detail, but for Distance Field Ambient Occlusion (DFAO) alone, surfaces may not be represented correctly [S5].
*   **Grazing Angles:** Rays at grazing angles require more steps to intersect with a plane, resulting in less accurate visualization (whiter regions) [S5].
*   **Precision:** The code may calculate intersection counts but fail to use them for inside/outside determination, outputting absolute distance instead [S2, unofficial].

## Subtopics

### [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md)
This method converts explicit geometry, such as triangle meshes or oriented point clouds, into a Truncated Signed Distance Field (TSDF) voxel grid by computing the closest-surface distance for each voxel center and assigning a sign based on surface orientation. It differs from [Projective TSDF Fusion](2658-projective-tsdf-fusion.md) by operating directly on explicit geometric primitives rather than raw depth buffers, and from [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md) by focusing on the initial conversion of a single explicit surface representation into a dense or sparse voxel volume.

### [SDF Dataset Normalization](3787-sdf-dataset-normalization.md)
SDF Dataset Normalization is a preprocessing step that transforms the vertex coordinates of an input 3D mesh so that the geometry fits within a standardized canonical space, typically the unit cube $[-1, 1]^3$ centered at the origin.

### [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md)
A method that converts a triangle mesh into a sparse, tree-structured signed or unsigned distance grid using OpenVDB, storing only active voxels near the surface for memory efficiency. It differs from dense voxelization approaches by allocating memory only for the narrow-band region around the mesh, making it suitable for large or complex models where ray-marched rendering or fast implicit queries are required.

### [Mesh to SDF Volume Conversion](7167-mesh-to-sdf-volume-conversion.md)
This method converts a watertight 3D mesh into a voxelized Signed Distance Field (SDF) volume. It solves the problem of creating a continuous implicit representation from discrete triangle data for use in volume rendering or differentiable rendering pipelines. It is distinguished from naive CPU-based generation by its use of GPU-accelerated ray tracing for initial occupancy and fast sweeping for redistancing, often followed by a surface refinement pass to reduce discretization error. It differs from TSDF fusion methods by relying on a single static mesh instance rather than multi-view sensor fusion.

### [Naive SDF Generation (CPU)](10688-naive-sdf-generation-cpu.md)
Naive SDF Generation (CPU) computes the Signed Distance Field (SDF) for a triangular mesh by explicitly evaluating the Euclidean distance from every voxel center to the mesh surface on the CPU. It solves the problem of creating a volumetric distance representation for ray marching or volume rendering by iterating through all triangles for every sample point. Unlike [Explicit Surface to TSDF Conversion](2615-explicit-surface-to-tsdf-conversion.md), which relies on volume integration or ray casting from virtual cameras, or [Sparse VDB SDF Generation](4498-sparse-vdb-sdf-generation.md), which uses hierarchical data structures for efficiency, this method performs a direct geometric query. It differs from [GPU-based SDF Generation](10690-gpu-based-sdf-generation.md) by executing the distance calculations sequentially or with limited CPU threading rather than leveraging massive parallelism in shaders.

### [SDF BSP Tree Construction](15293-sdf-bsp-tree-construction.md)
A method to generate a Signed Distance Field (SDF) from a triangle mesh by recursively subdividing space into two sets using dividing hyperplanes, allowing ray traversal to calculate the distance to the nearest point on the surface. It offers perfect accuracy compared to voxel grids by using original triangles, but suffers from poor generation and runtime performance due to sequential CPU construction and GPU-unfriendly branching.

### Mesh Distance Fields (MDF)
Mesh Distance Fields (MDF) are a specific application of SDFs in game engines like Unreal Engine, used for dynamic ambient occlusion and shadows on static mesh actors [S5]. They are generated offline and stored in volume textures, allowing for GPU particle collision and dynamic flow maps [S5].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of triangles. This is computationally expensive for high-resolution volumes and complex meshes [S2, unofficial].
*   **Memory Complexity:** $O(V)$, where $V = res^3$. The maximum size for a single mesh's volume texture is 8 megabytes, corresponding to a resolution of $128 \times 128 \times 128$ [S5].
*   **Performance:** Generation is offline and cannot be done at runtime [S5]. Higher resolution captures important features but increases memory usage [S5].
*   **Strengths:** Enables robust rendering of static meshes with dynamic lighting effects (AO, shadows) without runtime generation overhead [S5]. Allows safe skipping of empty space during ray marching (sphere tracing) [S5].
*   **Limitations:** High memory usage for high-resolution meshes. Thin surfaces and grazing angles can lead to inaccuracies [S5]. Open meshes and non-manifold geometry require special handling [S3, unofficial].
*   **When to Use:** Static meshes in game engines or offline rendering where dynamic lighting effects are desired and the mesh does not change [S5].
*   **When Not to Use:** Dynamic meshes, real-time generation, or scenes with very large meshes where memory constraints are tight [S5].

## Background
The concept of rendering triangle meshes as Signed Distance Fields has been explored in various contexts, from academic research on implicit surface rendering to practical implementations in game engines [S1, unofficial][S5]. The `MeshSDF` class in the `isoray` library provides an implicit function interface for rendering STL files using interval-arithmetic bisection [S1, unofficial]. CGAL's Isosurfacing_3 module provides tools for generating signed offset fields from closed surface meshes using AABB trees [S4]. In the game industry, Unreal Engine popularized the use of Mesh Distance Fields for dynamic ambient occlusion and shadows on static mesh actors [S5].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S3] taojin-6/volumetric_kit_grid, https://github.com/taojin-6/volumetric_kit_grid, fetched 2026-10-28 (unofficial)
[S4] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S5] 虚幻引擎网格体距离场, https://dev.epicgames.com/documentation/unreal-engine/mesh-distance-fields-in-unreal-engine?lang=zh-CN, fetched 2026-10-30
