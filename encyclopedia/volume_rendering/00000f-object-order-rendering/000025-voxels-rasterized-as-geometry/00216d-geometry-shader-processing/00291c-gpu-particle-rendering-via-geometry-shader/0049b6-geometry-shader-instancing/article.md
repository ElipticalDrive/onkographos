# Geometry Shader Instancing

Also called: GS Instancing, Programmatic Geometry Amplification, Programmatic Geometry Instancing

## Overview
Geometry Shader Instancing is a real-time rendering technique that amplifies point primitives into surfels (oriented quadrilaterals) or voxels (cubes) and instances them on the GPU using the programmable Geometry Shader unit [S1, unofficial]. It solves the problem of efficiently rendering large point clouds without CPU-side mesh generation, offering an alternative to hardware-controlled instancing pipelines by keeping the geometry description in a user-defined shader program [S1, unofficial]. Unlike hardware instancing, which relies on fixed-function repetition, this method performs object, world, view, and projection transformations programmatically within the geometry stage [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A buffer containing point attributes (position, normal vector, color, splat size) representing a point cloud [S1, unofficial].
*   **Output:** Triangle strips representing amplified and transformed surfels or voxels [S1, unofficial].

**Data Structures:**
*   **Point Attribute Buffer:** A contiguous memory buffer acting as a mirrored copy of the point cloud data array [S1, unofficial].
    *   **Fields:**
        *   `position`: 3D vector (x, y, z) [S1, unofficial].
        *   `normal`: 3D vector (nx, ny, nz) [S1, unofficial].
        *   `color`: 4D vector (r, g, b, a) [S1, unofficial].
        *   `splat_size`: Scalar value defining the size of the primitive [S1, unofficial].
    *   **Layout:** The buffer is immutable and packed sequentially for each point [S1, unofficial].

**Numbered Steps:**
1.  **CPU Preparation:** The CPU packs point attributes (position, normal, color, splat size) into the Point Attribute Buffer [S1, unofficial].
2.  **Draw Call:** The CPU issues a single draw call on the buffer to the GPU [S1, unofficial].
3.  **Vertex Shader Execution:** The Vertex Shader receives the point attributes and forwards them untouched to the Geometry Shader [S1, unofficial].
4.  **Geometry Shader Input:** The Geometry Shader receives the point primitives [S1, unofficial].
5.  **Amplification:** The Geometry Shader promotes the point primitive into a canonical template geometry (a quadrilateral for surfels or a cube for voxels) [S1, unofficial].
6.  **Transformation:** The Geometry Shader transforms the coordinates of the canonical template based on the point attributes [S1, unofficial]. This includes applying object, world, view, and projection transformations [S1, unofficial].
7.  **Instancing:** The Geometry Shader replicates the transformed geometry for each input point [S1, unofficial].
8.  **Rasterization:** The resulting triangle strips are passed to the Fragment Shader for rasterization [S1, unofficial].

**Formulas:**
Let $P_i$ be the $i$-th point in the buffer with attributes:
*   $\mathbf{p}_i = (x_i, y_i, z_i)$: Position
*   $\mathbf{n}_i = (n_x, n_y, n_z)$: Normal vector
*   $\mathbf{c}_i = (r_i, g_i, b_i, a_i)$: Color
*   $s_i$: Splat size

The Geometry Shader generates vertices $\mathbf{v}_{i,j}$ for the $j$-th vertex of the amplified primitive for point $i$. The transformation is defined as:
$$ \mathbf{v}_{i,j} = M_{proj} \cdot M_{view} \cdot M_{world} \cdot (\mathbf{p}_i + \mathbf{t}_{i,j} \cdot s_i) $$
Where:
*   $M_{proj}$, $M_{view}$, $M_{world}$ are the projection, view, and world transformation matrices [S1, unofficial].
*   $\mathbf{t}_{i,j}$ is the local offset of the $j$-th vertex of the canonical template relative to the point center [S1, unofficial].
*   $s_i$ scales the template according to the splat size [S1, unofficial].

**Parameters:**
*   **Amplification Level:** Restricted to moderate levels; only constant-size amplifications (quadrilaterals or cubes) are supported [S1, unofficial].
*   **Point Attributes:** Position, normal vector, color, and splat size [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Performance:** Heavy tessellation is ill-suited for the Geometry Shader unit [S1, unofficial].
*   **Branching:** Alternative methods using fragment shader branching and discard instructions can negatively impact performance [S1, unofficial].
*   **Bandwidth:** Preprocessing surfels on the CPU increases memory and CPU-GPU bandwidth usage [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
def vertex_shader(point_attributes):
    """
    Forwards point attributes untouched to the Geometry Shader.
    """
    return point_attributes

# Geometry Shader
def geometry_shader(point_attributes):
    """
    Amplifies a point primitive into a surfel (quadrilateral) or voxel (cube)
    and transforms it according to the point attributes.
    """
    # Extract attributes
    position = point_attributes.position
    normal = point_attributes.normal
    color = point_attributes.color
    splat_size = point_attributes.splat_size

    # Define canonical template vertices (e.g., for a quadrilateral)
    # Assuming a unit square centered at origin in local space
    template_vertices = [
        vec3(-0.5, -0.5, 0.0),
        vec3( 0.5, -0.5, 0.0),
        vec3( 0.5,  0.5, 0.0),
        vec3(-0.5,  0.5, 0.0)
    ]

    # Compute basis vectors for the surfel orientation
    # Assuming 'normal' is the surface normal and we need an up vector
    # For simplicity, assume a fixed up vector or compute from normal
    up = vec3(0.0, 1.0, 0.0)
    if abs(dot(normal, up)) > 0.999:
        up = vec3(1.0, 0.0, 0.0)
    right = normalize(cross(normal, up))
    up = cross(right, normal)

    # Transform template vertices
    output_vertices = []
    for tv in template_vertices:
        # Scale by splat size
        scaled_tv = tv * splat_size
        # Rotate to align with normal
        rotated_tv = right * scaled_tv.x + up * scaled_tv.y
        # Translate to position
        world_pos = position + rotated_tv
        # Apply view and projection transformations
        # Note: In a real shader, these matrices are passed as uniforms
        clip_pos = projection_matrix * view_matrix * world_matrix * vec4(world_pos, 1.0)
        output_vertices.append(clip_pos)

    # Emit triangle strip
    # For a quad, we need two triangles: (0,1,2) and (0,2,3)
    # Geometry Shader emits vertices in order
    emit_vertex(output_vertices[0], color)
    emit_vertex(output_vertices[1], color)
    emit_vertex(output_vertices[2], color)
    emit_vertex(output_vertices[3], color)

    # End primitive
    end_primitive()
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points, $O(N)$, where $N$ is the number of point attributes in the buffer [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of points, $O(N)$, for the Point Attribute Buffer [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with the number of points as long as the amplification level is moderate [S1, unofficial].
*   **Strengths:**
    *   Real-time amplification and instancing of surfels and voxels [S1, unofficial].
    *   Optimal streaming efficiency and draw batching [S1, unofficial].
    *   Guarantees immutability of the point cloud data [S1, unofficial].
*   **Limitations:**
    *   The Geometry Shader unit is ill-suited for heavy tessellation [S1, unofficial].
    *   Preprocessing on the CPU (if not using this method) increases bandwidth usage [S1, unofficial].
*   **When to use:** When rendering large point clouds with surfels or voxels in real-time and hardware instancing is not available or suitable [S1, unofficial].
*   **When not to use:** When heavy tessellation is required or when the overhead of the Geometry Shader is prohibitive compared to hardware instancing [S1, unofficial].

## Background
Geometry Shader Instancing was investigated as a contribution to real-time surfel and voxel rendering of scanned 4D media [S1, unofficial]. It leverages the programmable Geometry Shader unit, which sits between the Vertex Shader and Fragment Shader, to perform programmatic modifications on geometry and topology [S1, unofficial]. The technique was proposed as an alternative to CPU-side preprocessing and hardware-controlled geometry instancing pipelines [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
