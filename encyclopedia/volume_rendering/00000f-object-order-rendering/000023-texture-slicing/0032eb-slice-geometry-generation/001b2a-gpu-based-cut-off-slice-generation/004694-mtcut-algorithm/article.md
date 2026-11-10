# MTCut Algorithm

Also called: GPU-based Marching Tetra Cuts

## Overview
The MTCut algorithm generates cut geometry for volumetric data by intersecting a voxel grid with cutting planes, implemented entirely on the GPU via vertex shaders. It solves the problem of efficiently creating interactive slice boundaries for isosurfaces without CPU-side geometry generation. It differs from sibling methods by encoding voxel traversal state into 2D textures and using a vertex-shader-based Marching Tetrahedra (MT) variant to generate cut vertices, rather than incremental intersection calculations or proxy geometry alignment.

## How it works
The method consists of three rendering passes and an optional geometry recovery pass.

### Inputs and Outputs
**Inputs:**
*   Volumetric scalar field (3D texture).
*   Isosurface geometry (Vertex Buffer Object, VBO) representing the uncut model.
*   Cutting plane definitions (origin, normal, distance).
*   Voxel grid parameters (origin, dimensions, resolution).

**Outputs:**
*   Final image containing the uncut isosurface and the new cut geometry.
*   (Optional) Textures containing position and normal data for recovered geometry.

### Data Structures

**1. Cut Voxel Bitmap Texture (2D)**
*   **Format:** 2D texture, dimensions $W \times H$.
*   **Layout:** Texel at $(s, t)$ corresponds to voxel column $(i=s, j=t)$.
*   **Content:** A bitmap where the $k$-th bit is set to 1 if voxel $(i, j, k)$ is intersected by the cutting tool, else 0.
*   **Precision Limit:** Internal blending uses 16-bit precision. A $512 \times 512$ texture supports up to 64 depth values ($k$). A $1024 \times 1024$ texture supports up to 256 depth values [S1, unofficial].

**2. Vertex Attributes for Cut Geometry Shader**
To encode 17 values into 15 variables for the vertex shader processing tetrahedra in cut voxels:
*   `glVertex4f` (4 floats): Signed distances to the isosurface for the 4 vertices of the tetrahedron (classification).
*   `glMultiTexCoord4f` (4 floats): Signed distances to the first cutting plane for the 4 vertices.
*   `glNormal` (3 floats): Signed distances to the second cutting plane for the 4 vertices.
*   `glColor4f` (4 floats):
    *   $R, G$: Voxel indices $i, j$.
    *   $B$: Encoded integer $k \times 16 + \text{tetra} \times 4 + \text{vertex}$.
    *   $A$: Distance to the second cutting plane (remaining information).

### Steps

**Step 1: Texture Coding (Voxel Traversal)**
1.  Identify all voxels traversed by the cutting tool.
2.  Render a point for each cut voxel.
3.  Use `GL_ADD` blending.
4.  Encode the depth level $k$ into color channels such that the $k$-th bit of the texel at $(i, j)$ is set if voxel $(i, j, k)$ is cut [S1, unofficial].

**Step 2: Render Uncut Isosurface**
1.  Draw the original isosurface using the VBO.
2.  In the fragment shader, discard triangles lying inside cut voxels.
3.  **Pitfall:** Do not discard in the vertex shader. The isosurface representation does not duplicate shared vertices. Discarding a vertex would eliminate all adjacent triangles, including those outside the cut region [S1, unofficial].

**Step 3: Generate Cut Geometry**
Execute a vertex shader for quads belonging to cut voxels. The shader performs the following for each tetrahedron:
1.  **Calculate Coordinates:** Compute the 3D coordinates of the four tetrahedron vertices using the voxel index (decoded from `glColor`) and the voxel origin/dimensions.
2.  **Create Marching Pattern:** Determine the intersection pattern using the scalar data at each vertex (from `glVertex`) and the classification values relative to the cutting planes (from `glMultiTexCoord` and `glNormal`) [S1, unofficial].
3.  **Generate Vertices:** Compute the coordinates of new intersection vertices based on the marching pattern classification.
4.  **Compute Normals:** Calculate the normal for each new vertex. For triangles belonging to tetrahedra containing the isosurface, approximate the normal from the gradient at the point using volume info in the 3D texture. For triangles belonging to tetrahedra containing only cutting planes (interior to isosurface), calculate the average of the cutting plane normals [S1, unofficial].

**Optional Step: Geometry Recovery**
1.  Issue `GL_POINTS` for each generated vertex, passing the texel position.
2.  Vertex shader computes actual position and normal.
3.  Fragment shader writes position to a position texture and normal to a normal texture.
4.  Read back textures to CPU (approx. 100-120ms for $512 \times 512$).
5.  Fuse with isosurface and separate shells via a flooding algorithm [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Encode cut voxels
def encode_cut_voxels(cut_voxel_indices, texture_width, texture_height):
    # Initialize 2D texture with zeros
    cut_bitmap = zeros((texture_width, texture_height, 32)) # 32 bits for depth
    
    for (i, j, k) in cut_voxel_indices:
        # Set k-th bit for texel (i, j)
        cut_bitmap[i, j] |= (1 << k)
        
    # Render points for cut voxels using GL_ADD blending to populate texture
    # This assumes the GPU hardware handles the bit accumulation via blending
    render_points_to_texture(cut_bitmap)

# Step 2: Render uncut isosurface
def render_uncut_isosurface(isosurface_vbo, cut_bitmap_texture):
    for triangle in isosurface_vbo:
        # Check if triangle is inside any cut voxel
        if is_inside_cut_voxel(triangle, cut_bitmap_texture):
            discard_triangle()
        else:
            draw_triangle(triangle)

# Step 3: Vertex Shader for Cut Geometry
def vertex_shader_cut_geometry(vertex_index, voxel_origin, voxel_dims):
    # Decode voxel and tetrahedron info from color
    i = color.r
    j = color.g
    k = floor(color.b / 16)
    tetra_id = floor((color.b % 16) / 4)
    vert_id = color.b % 4
    
    # Decode distances
    iso_distances = vertex.xyzw
    plane1_distances = multiTexCoord.xyzw
    plane2_distances = normal.xyz
    plane2_remaining = color.a
    
    # 1. Calculate tetrahedron vertex coordinates
    tetra_vertices = calculate_tetra_coords(i, j, k, tetra_id, voxel_origin, voxel_dims)
    
    # 2. Create marching pattern
    # Classification based on iso_distances and plane distances
    pattern = create_marching_pattern(tetra_vertices, iso_distances, plane1_distances, plane2_distances)
    
    # 3. Generate new vertex coordinates
    new_vertices = generate_intersection_vertices(tetra_vertices, pattern)
    
    # 4. Compute normal
    # If tetra contains isosurface, use gradient from 3D texture
    # If tetra is interior (only planes), average plane normals
    normals = compute_normals(new_vertices, pattern, volume_texture, plane1_distances, plane2_distances)
    
    return new_vertices, normals

# Geometry Recovery (Optional)
def recover_geometry(cut_vertices):
    position_texture = create_texture()
    normal_texture = create_texture()
    
    for vertex in cut_vertices:
        # Issue point
        point = render_point(vertex)
        
        # Fragment shader writes to textures
        position_texture[point.texel] = point.position
        normal_texture[point.texel] = point.normal
        
    # Read back to CPU
    cpu_positions = read_texture_back(position_texture)
    cpu_normals = read_texture_back(normal_texture)
    
    return fuse_with_isosurface(cpu_positions, cpu_normals)
```

### Parameters
*   **Texture Resolution:** $512 \times 512$ (max 64 depth levels) or $1024 \times 1024$ (max 256 depth levels) [S1, unofficial].
*   **Blending Function:** `GL_ADD` [S1, unofficial].
*   **Illumination:** Phong fragment shader [S1, unofficial].

### Edge Cases and Pitfalls
*   **Shared Vertices:** Discarding vertices in the vertex shader during Step 2 is incorrect because it removes all adjacent triangles, even those outside the cut [S1, unofficial].
*   **Mass Conservation:** The algorithm may incur a small, visually undetectable violation of mass conservation in special slicing cases. The upper bound of lost mass is the volume of a tetrahedron [S1, unofficial].
*   **Performance:** 3D texture access in the vertex shader is extremely slow; normals for cut triangles must be computed in the fragment shader [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Geometry recovery (reading back textures) takes 100-120ms for a $512 \times 512$ resolution [S1, unofficial].
*   **Strengths:** Interactive performance via GPU offloading; no artifacts at the boundary between original and cut geometry because both use MT principles; supports progressive cutting and undo [S1, unofficial].
*   **Limitations:** Limited by 16-bit internal blending precision (max 64 depth values for $512^2$ texture); small mass conservation violation possible; 3D texture access is slow in vertex shaders [S1, unofficial].
*   **When to use:** When interactive cut-off slice generation is required and the volume depth is within the texture precision limits.
*   **When not to use:** When strict mass conservation is required or when volume depth exceeds 256 levels without using multiple render targets [S1, unofficial].

## Background
The MTCut algorithm was introduced by Pascucci et al. in 2007. It is based on the Marching Tetrahedra (MT) method, similar to Pascucci's earlier method [Pas04], but adapted for GPU execution with regular voxelization. It is used in medical imaging for interactive volume visualization and cutting [S1, unofficial].

## Sources
[S1] MTCut: GPU-based Marching Tetra Cuts, https://doi.org/10.2312/localchapterevents/tpcg/tpcg07/037-044, fetched 2026-10-26 (unofficial)
