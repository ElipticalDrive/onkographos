# Mesh Normalization


## Overview
Mesh Normalization is a preprocessing method that transforms a 3D mesh's vertex coordinates so that the geometry fits within a standardized canonical space, typically a unit cube or unit sphere centered at the origin [S1, S2, S3, unofficial]. This process solves the problem of inconsistent scale and position in input meshes, which is critical for downstream volume rendering pipelines such as Signed Distance Field (SDF) generation, where grid resolution and sampling bounds depend on predictable input dimensions [S1, S3, unofficial]. It differs from [SDF Grid Resampling from File](4878-sdf-grid-resampling-from-file.md) by operating on the source mesh geometry directly rather than on an existing voxel grid, ensuring the input data is normalized before discretization [S1, S3, unofficial].

## How it works
The method consists of three primary variants: Axis-Aligned Bounding Box (AABB) Normalization, Variance Normalization, and Mesh Loading with Normalization. All variants assume the input is a mesh defined by a set of 3D vertices $V = \{v_1, v_2, \dots, v_n\}$ where $v_i \in \mathbb{R}^3$.

### AABB Normalization (Standard)
This variant centers the mesh and scales it to fit within a unit cube $[-1, 1]^3$ [S1, unofficial].

**Inputs:**
- `vertices`: Array of $n$ 3D vectors.

**Outputs:**
- `vertices`: Transformed array of $n$ 3D vectors.

**Steps:**
1. Compute the bounding box of the mesh vertices.
2. Calculate the center of the bounding box.
3. Calculate the extents (width, height, depth) of the bounding box.
4. Determine the maximum extent.
5. Compute a scale factor.
6. Translate vertices to the origin.
7. Scale vertices by the scale factor.

**Formulas:**
Let $v_i = (x_i, y_i, z_i)$.
1. Bounds:
   $$ x_{min} = \min_i(x_i), \quad x_{max} = \max_i(x_i) $$
   $$ y_{min} = \min_i(y_i), \quad y_{max} = \max_i(y_i) $$
   $$ z_{min} = \min_i(z_i), \quad z_{max} = \max_i(z_i) $$
2. Center:
   $$ c = \left( \frac{x_{min} + x_{max}}{2}, \frac{y_{min} + y_{max}}{2}, \frac{z_{min} + z_{max}}{2} \right) $$
3. Extents:
   $$ e_x = x_{max} - x_{min}, \quad e_y = y_{max} - y_{min}, \quad e_z = z_{max} - z_{min} $$
4. Max Extent:
   $$ e_{max} = \max(e_x, e_y, e_z) $$
5. Scale Factor:
   $$ s = \frac{1.0}{e_{max}} $$
6. Translation:
   $$ v'_i = v_i - c $$
7. Scaling:
   $$ v''_i = v'_i \cdot s $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_mesh_aabb(vertices):
    """
    Normalizes mesh vertices to fit in a unit cube [-1, 1]^3.
    vertices: list of [x, y, z] floats.
    Returns: list of normalized [x, y, z] floats.
    """
    if not vertices:
        return []

    # 1. Compute bounds
    xs = [v[0] for v in vertices]
    ys = [v[1] for v in vertices]
    zs = [v[2] for v in vertices]

    x_min = min(xs)
    x_max = max(xs)
    y_min = min(ys)
    y_max = max(ys)
    z_min = min(zs)
    z_max = max(zs)

    # 2. Compute center
    center_x = (x_min + x_max) / 2.0
    center_y = (y_min + y_max) / 2.0
    center_z = (z_min + z_max) / 2.0
    center = (center_x, center_y, center_z)

    # 3. Compute extents
    extent_x = x_max - x_min
    extent_y = y_max - y_min
    extent_z = z_max - z_min

    # 4. Compute max extent
    max_extent = max(extent_x, extent_y, extent_z)

    # Handle degenerate case where max_extent is 0
    if max_extent == 0:
        return vertices

    # 5. Compute scale factor
    scale_factor = 1.0 / max_extent

    # 6 & 7. Translate and Scale
    normalized_vertices = []
    for v in vertices:
        tx = (v[0] - center_x) * scale_factor
        ty = (v[1] - center_y) * scale_factor
        tz = (v[2] - center_z) * scale_factor
        normalized_vertices.append([tx, ty, tz])

    return normalized_vertices
```

### Variance Normalization
This variant scales the mesh so that the mean squared distance from its center of mass is 1, effectively normalizing based on the covariance of the vertex distribution [S2, unofficial].

**Inputs:**
- `vertices`: Array of $n$ 3D vectors.
- `center_of_mass` (optional): If not provided, it is computed from vertices.

**Outputs:**
- `vertices`: Transformed array.
- `center_of_mass`: The computed center.

**Steps:**
1. Compute the center of mass (COM) of the vertices.
2. Translate the mesh so the COM is at the origin.
3. Compute the covariance matrix $C$ of the translated vertices.
4. Compute the scale factor $s$ using the trace of $C$.
5. Scale the vertices by $s$.
6. Translate the vertices back by the original COM.

**Formulas:**
1. Center of Mass:
   $$ \text{com} = \frac{1}{n} \sum_{i=1}^{n} v_i $$
2. Translation to Origin:
   $$ v'_i = v_i - \text{com} $$
3. Covariance Matrix $C$:
   $$ C = \frac{1}{n} \sum_{i=1}^{n} v'_i (v'_i)^T $$
   Where $C$ is a $3 \times 3$ matrix.
4. Scale Factor:
   $$ s = \frac{1}{\sqrt{\text{tr}(C)}} = \frac{1}{\sqrt{C_{00} + C_{11} + C_{22}}} $$
5. Scaling:
   $$ v''_i = v'_i \cdot s $$
6. Translation Back:
   $$ v'''_i = v''_i + \text{com} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_mesh_variance(vertices):
    """
    Normalizes mesh vertices so mean squared distance from center of mass is 1.
    vertices: list of [x, y, z] floats.
    Returns: list of normalized [x, y, z] floats.
    """
    if not vertices:
        return []

    n = len(vertices)
    
    # 1. Compute Center of Mass
    com_x = sum(v[0] for v in vertices) / n
    com_y = sum(v[1] for v in vertices) / n
    com_z = sum(v[2] for v in vertices) / n
    com = (com_x, com_y, com_z)

    # 2. Translate to origin
    translated = []
    for v in vertices:
        translated.append([v[0] - com_x, v[1] - com_y, v[2] - com_z])

    # 3. Compute Covariance Matrix C
    # C is 3x3. We only need the diagonal for the trace, but we compute full matrix for clarity.
    # C_ij = (1/n) * sum(v'_i * v'_j)
    c00 = sum(t[0]**2 for t in translated) / n
    c11 = sum(t[1]**2 for t in translated) / n
    c22 = sum(t[2]**2 for t in translated) / n
    
    # 4. Compute scale factor s = 1 / sqrt(tr(C))
    trace_c = c00 + c11 + c22
    if trace_c == 0:
        return vertices # Degenerate case
    
    s = 1.0 / (trace_c ** 0.5)

    # 5. Scale vertices
    scaled = []
    for t in translated:
        scaled.append([t[0] * s, t[1] * s, t[2] * s])

    # 6. Translate back
    final_vertices = []
    for t in scaled:
        final_vertices.append([t[0] + com_x, t[1] + com_y, t[2] + com_z])

    return final_vertices
```

### Mesh Loading and Normalization
This variant integrates normalization into the mesh loading process, often used in conjunction with ray tracing acceleration structures [S3, unofficial].

**Inputs:**
- File path to a 3D model.

**Outputs:**
- Normalized mesh data (vertices, indices) ready for acceleration structure construction.

**Steps:**
1. Load the 3D model file using a library (e.g., Assimp) with specific flags: `Triangulate`, `JoinIdenticalVertices`, `RemoveComponent`, `ImproveCacheLocality`, `OptimizeMeshes`, `OptimizeGraph`, `DropNormals`. Also load animations, bone weights, cameras, colors, lights, materials, normals, tangents, texcoords, and textures [S3, unofficial].
2. Extract vertices from the first mesh in the scene.
3. Calculate the bounding box centroid of the extracted vertices.
4. Calculate the radius $R$ as the maximum distance from the centroid to any vertex:
   $$ R = \max_i \| v_i - c \| $$
5. Scale vertices so that the radius is 1.0:
   $$ v'_i = \frac{v_i - c}{R} $$
6. Extract indices from faces and convert to 3-component indices (e.g., `glm::uvec3`) [S3, unofficial].
7. Build an acceleration structure (e.g., Embree) with the normalized vertices and indices [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_and_normalize_mesh(file_path):
    """
    Loads a mesh, normalizes it to a unit sphere, and prepares for acceleration structure.
    This is a high-level description of the C++ logic in [S3, unofficial].
    """
    # 1. Load model using Assimp (pseudo-code for library call)
    # scene = aiImportFile(file_path, aiProcess_Triangulate | aiProcess_JoinIdenticalVertices | ...)
    # Note: Specific flags from [S3, unofficial] are assumed to be handled by the loader.
    
    # 2. Extract vertices from the first mesh
    # vertices = [mesh.vertices[i] for i in range(mesh.num_vertices)]
    
    # 3. Calculate bounding box centroid
    if not vertices:
        return None, None
    
    n = len(vertices)
    c_x = sum(v[0] for v in vertices) / n
    c_y = sum(v[1] for v in vertices) / n
    c_z = sum(v[2] for v in vertices) / n
    centroid = (c_x, c_y, c_z)
    
    # 4. Calculate radius (max distance from centroid)
    max_dist = 0.0
    for v in vertices:
        dx = v[0] - c_x
        dy = v[1] - c_y
        dz = v[2] - c_z
        dist = (dx*dx + dy*dy + dz*dz) ** 0.5
        if dist > max_dist:
            max_dist = dist
            
    if max_dist == 0:
        return vertices, None # Degenerate
        
    # 5. Scale vertices so radius is 1.0
    normalized_vertices = []
    for v in vertices:
        nx = (v[0] - c_x) / max_dist
        ny = (v[1] - c_y) / max_dist
        nz = (v[2] - c_z) / max_dist
        normalized_vertices.append([nx, ny, nz])
        
    # 6. Extract indices and convert to uvec3 (pseudo-code)
    # indices = [face.indices for face in mesh.faces]
    # converted_indices = [glm::uvec3(i[0], i[1], i[2]) for i in indices]
    
    # 7. Build Accel structure (pseudo-code)
    # accel = Accel::build(normalized_vertices, converted_indices)
    
    return normalized_vertices, None # Indices omitted for brevity in Python pseudocode
```

## Subtopics

### Variance Normalization
Scales a mesh so that the mean squared distance from its center of mass is 1 [S2, unofficial]. It differs from AABB normalization by using statistical variance rather than geometric bounds, making it sensitive to the distribution of vertices rather than just the extremities. Choose this when the mesh has outliers that distort the bounding box but the core geometry is what matters for SDF generation. Link: [S2, unofficial].

### Mesh Loading and Normalization
Integrates normalization into the file loading pipeline, specifically targeting unit sphere normalization (radius 1.0) for use with Embree acceleration structures [S3, unofficial]. It differs from standalone normalization by including the loading step and specific Assimp processing flags. Choose this when building a renderer that requires immediate acceleration structure construction upon loading. Link: [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n)$ for all variants, where $n$ is the number of vertices. Computing bounds, mean, or covariance requires a single pass or two passes over the vertex data.
- **Memory Complexity:** $O(n)$ to store the transformed vertices. The covariance matrix for Variance Normalization is a fixed $3 \times 3$ size, so it is $O(1)$ auxiliary space.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with respect to vertex count.
- **Strengths:** Ensures consistent input scale for SDF grids; prevents numerical instability in ray marching or voxelization due to extreme coordinate values.
- **Limitations:** AABB normalization can be skewed by outliers (single distant vertices). Variance normalization can be skewed by sparse regions. Mesh Loading variant depends on the correctness of the Assimp parsing flags.
- **When to use:** Always before SDF generation or voxelization if the input mesh scale is unknown or variable.
- **When not:** If the mesh is already normalized or if the absolute scale is physically meaningful and must be preserved (though SDF generation usually requires normalization anyway).

## Background
Mesh normalization is a standard preprocessing step in computer graphics, particularly in pipelines involving Signed Distance Fields (SDFs) and voxelization. The AABB method is widely used in deep learning-based 3D generation (e.g., DCx) to standardize input shapes [S1, S2, S3, unofficial]. The Variance Normalization method appears in research on high-resolution sparse voxel DAGs [S2, unofficial]. The Mesh Loading variant is specific to implementations like SolidVoxelization, which integrates normalization with ray tracing acceleration structure construction [S3, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
[S3] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
