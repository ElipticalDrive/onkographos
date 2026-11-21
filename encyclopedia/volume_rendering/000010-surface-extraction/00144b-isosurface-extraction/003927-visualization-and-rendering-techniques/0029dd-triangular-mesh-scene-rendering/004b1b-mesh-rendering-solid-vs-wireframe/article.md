# Mesh Rendering (Solid vs Wireframe)
Also called: Face rendering, Triangle mesh draw

## Overview
Mesh rendering draws a triangular mesh either as solid filled faces or as wireframe edges [S1, unofficial]. Solid mode renders each face as a thick triangle, while wireframe mode renders each face's edges as capsules [S1, unofficial]. It differs from [Water Surface Rendering](5636-water-surface-rendering.md) and [Stream Surface Rendering](7716-stream-surface-rendering.md) by treating the geometry as opaque or skeletal primitives rather than fluid or flow-based surfaces.

## How it works
**Inputs:**
- `vertices`: Array of 3D points.
- `faces`: Array of triplets of vertex indices.
- `tMatrix`: Transformation matrix.
- `mode`: Enum `SOLID` or `WIREFRAME`.
- `maxScale`: Maximum scale factor from transformation [S1, unofficial].

**Outputs:**
- Rendered image or primitive list.

**Data Structures:**
- `CuboidGeometry`: Struct containing `v1` (start point), `v2` (end point or second vertex), and `isWireframe` (boolean) [S1, unofficial].

**Steps:**
1. Transform vertices using `matColMul(tMatrix, vertices[face[i]])` [S1, unofficial].
2. If `mode` is `WIREFRAME`:
   - Iterate all unique vertex pairs `(i, j)` per face where `i` ranges from 0 to 1 and `j` ranges from `i` to 2 [S1, unofficial].
   - Draw a capsule between the transformed vertices `i` and `j` [S1, unofficial].
3. If `mode` is `SOLID`:
   - Draw a triangle for each face using the transformed vertices [S1, unofficial].

**Parameters:**
- `maxScale`: Maximum scale factor from transformation. Used for potential culling or LOD [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mesh(vertices, faces, t_matrix, mode, max_scale):
    """
    Render a triangle mesh as solid or wireframe.
    
    Args:
        vertices: List of 3D vectors.
        faces: List of tuples (i, j, k) of vertex indices.
        t_matrix: 4x4 transformation matrix.
        mode: 'SOLID' or 'WIREFRAME'.
        max_scale: Maximum scale factor for culling/LOD.
    """
    # Transform vertices
    transformed_vertices = []
    for v in vertices:
        # matColMul applies matrix multiplication column-wise
        transformed_v = mat_col_mul(t_matrix, v)
        transformed_vertices.append(transformed_v)
        
    if mode == 'WIREFRAME':
        # Iterate all unique vertex pairs per face
        for face in faces:
            i, j, k = face
            # Unique pairs: (0,1), (0,2), (1,2)
            pairs = [(i, j), (i, k), (j, k)]
            for v_idx1, v_idx2 in pairs:
                v1 = transformed_vertices[v_idx1]
                v2 = transformed_vertices[v_idx2]
                draw_capsule(v1, v2)
    elif mode == 'SOLID':
        # Draw triangle for each face
        for face in faces:
            i, j, k = face
            v1 = transformed_vertices[i]
            v2 = transformed_vertices[j]
            v3 = transformed_vertices[k]
            draw_triangle(v1, v2, v3)
```

## Subtopics

### Wireframe Cube Rendering
Creates a mesh cube with a wireframe modifier and a solid color material [S2, unofficial]. Steps: Add primitive cube with `bpy.ops.mesh.primitive_cube_add(size=1, enter_editmode=False, align="WORLD", location=position)` [S2, unofficial]. Set scale and rotation: `cube.scale = scale; cube.rotation_euler = rotation` [S2, unofficial]. Apply transforms: `bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)` [S2, unofficial]. Add wireframe modifier: `bpy.ops.object.modifier_add(type="WIREFRAME"); set thickness=wireframe_scale` [S2, unofficial]. Create material: `bpy.data.materials.new(name="CubeMaterial"); material.use_nodes = False; material.diffuse_color = color` [S2, unofficial]. Assign material: `if len(cube.data.materials) > 0: cube.data.materials[0] = material else: cube.data.materials.append(material)` [S2, unofficial].

## Operating characteristics
Time and memory complexity are linear with respect to the number of faces and vertices. Solid mode is generally faster than wireframe mode as it involves fewer draw calls (one per face vs. three per face for edges). Wireframe mode provides better visibility of internal structure but may suffer from aliasing on thin edges. Solid mode is suitable for opaque objects, while wireframe is useful for debugging or transparent overlays.

## Background
Originated in early computer graphics for visualizing 3D models. The specific implementation details described here are from unofficial sources [S1, unofficial][S2, unofficial].

## Sources
[S1] quertzuiop/Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
[S2] risteon/blender_kitti, https://github.com/risteon/blender_kitti, fetched 2026-10-27 (unofficial)
