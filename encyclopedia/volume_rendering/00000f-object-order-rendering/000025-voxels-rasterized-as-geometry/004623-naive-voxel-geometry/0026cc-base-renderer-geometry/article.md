# Base Renderer Geometry

Also called: Cube Indices Layout, DrawVoxel, Square Face Indices Layout, Square Line Indices Layout, VoxelPass

## Overview
Base Renderer Geometry is a method for rendering voxel data by converting scalar field values into geometric primitives (lines or quads) that are submitted directly to the graphics pipeline. It solves the problem of visualizing volumetric data by treating each voxel as a simple 2D face or 3D cube edge, relying on the GPU's rasterizer for depth testing and blending rather than complex ray-marching or ray-casting algorithms [S1, S2, unofficial]. It differs from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by explicitly managing vertex buffers and index layouts for geometric primitives, and from [Cube Geometry for Volume Rendering](10885-cube-geometry-for-volume-rendering.md) by supporting both full cube edge definitions and simplified square face definitions depending on the orientation and culling requirements [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A list of voxel positions (`Vertices`), associated colors (`Colors`), and optional index buffers (`Indices`). Transformation matrices (`LocalToWorld`, `Orientation`) and rendering parameters (`transparency`, `CullMode`, `DefaultColor`, `skinning`) control the output.
*   **Outputs:** Rasterized pixels on the screen, with depth buffer updates and optional alpha blending based on the `transparency` flag [S1, S2, unofficial].

**Data Structures:**
The method relies on specific index layouts to define geometry from vertex data [S1, unofficial]:

1.  **`CUBE_INDICES`**: An integer array of 24 elements defining the 12 edges of a cube. The layout is:
    ```text
    [0, 1, 1, 2, 2, 3, 3, 0, 4, 5, 5, 6, 6, 7, 7, 4, 0, 4, 1, 5, 2, 6, 3, 7]
    ```
    This defines lines connecting vertices 0-1, 1-2, 2-3, 3-0 (bottom face), 4-5, 5-6, 6-7, 7-4 (top face), and verticals 0-4, 1-5, 2-6, 3-7 [S1, unofficial].

2.  **`SQUARE_LINE_INDICES`**: An integer array of 8 elements defining the perimeter of a square (quad):
    ```text
    [0, 1, 1, 2, 2, 3, 3, 0]
    ```
    This defines lines connecting vertices 0-1, 1-2, 2-3, 3-0 [S1, unofficial].

3.  **`SQUARE_FACE_INDICES`**: An integer array of 6 elements defining two triangles for a filled square face:
    ```text
    [0, 1, 2, 0, 2, 3]
    ```
    This defines triangles (0,1,2) and (0,2,3) [S1, unofficial].

4.  **Vertex Attributes**:
    *   **Base Format**: Comprises `Position` (Vector4), `Normal`, `Tangent`, and `Surface` attributes. The `Surface` attribute packs texture coordinates and ambient occlusion into a single 3-component float array (RGB 32-bit float) [S2, unofficial].
    *   **Skinned Format**: Extends the base format with `JointCombined` for skeletal animation support [S2, unofficial].

5.  **Renderer State (`DrawVoxel` struct)**:
    *   `pipeline_basic`: Graphics pipeline for standard rendering.
    *   `pipeline_skinned`: Graphics pipeline for skinned rendering.
    *   `static_batches`: A two-level batch structure mapping `MaterialId` to a list of vertex arguments.
    *   `skinned_batches`: Similar to `static_batches` but for skinned vertex arguments.
    *   `models`: Dynamic vertex buffer for static mesh data.
    *   `skinned_models`: Dynamic vertex buffer for skinned mesh data.
    *   `transparency`: Boolean flag controlling alpha filtering and blend state [S2, unofficial].

**Numbered Steps:**

1.  **Pipeline Construction**:
    *   Create a layout from descriptor set layouts.
    *   Build a basic graphics pipeline with:
        *   Back-face culling enabled.
        *   Depth test function set to `Less`.
        *   Depth write enabled if `!transparent`.
        *   Optional premultiplied alpha blending if `transparent` is true [S2, unofficial].
    *   If `skinning` is enabled, build a child pipeline with a different vertex descriptor and skinned vertex shader [S2, unofficial].

2.  **Preparation (Batching)**:
    *   Fetch `VoxelMesh`, `Atlas`, `Handle<VoxelMesh>`, `Transform`, `JointTransforms`, and `Tint` from the world entities [S2, unofficial].
    *   Filter meshes by transparency match: only process meshes if their alpha state matches the pass configuration (`transparency`). This prevents incorrect blending or culling due to alpha mismatch [S2, unofficial].
    *   Group filtered meshes by material ID.
    *   Insert grouped meshes into `static_batches` or `skinned_batches` depending on the `skinning` flag.
    *   Write batch data to `models` (static) and `skinned_models` (skinned) dynamic vertex buffers [S2, unofficial].

3.  **Drawing**:
    *   Bind the basic graphics pipeline (`pipeline_basic` or `pipeline_skinned`).
    *   Bind the environment descriptor set.
    *   Iterate over `static_batches` (or `skinned_batches`).
    *   For each material, bind the material descriptor set.
    *   Iterate over mesh batches within the material.
    *   Call `mesh.bind_and_draw` with instance ranges to issue draw calls [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxels(voxel_data, camera, renderer_config):
    """
    Renders voxel data using Base Renderer Geometry.
    
    Args:
        voxel_data: List of voxel positions and colors.
        camera: Camera object with worldToCameraMatrix and projectionMatrix.
        renderer_config: Configuration including LocalToWorld, Orientation, transparency, etc.
    """
    # Step 1: Pipeline Setup (One-time or when config changes)
    if renderer_config.pipeline_not_built:
        pipeline = create_pipeline(
            cull_mode=renderer_config.CullMode,
            depth_test="Less",
            depth_write=not renderer_config.transparency,
            blend=renderer_config.transparency # Premultiplied alpha
        )
        if renderer_config.skinning:
            skinned_pipeline = create_skinned_pipeline()
        renderer_config.pipeline_not_built = False

    # Step 2: Preparation and Batching
    static_batches = {}
    skinned_batches = {}
    
    for voxel in voxel_data:
        # Filter by transparency
        if voxel.alpha != renderer_config.transparency:
            continue
            
        material_id = voxel.material_id
        if material_id not in static_batches:
            static_batches[material_id] = []
            
        # Transform vertex positions
        local_pos = voxel.position
        world_pos = transform(local_pos, renderer_config.LocalToWorld)
        
        # Determine geometry type based on Orientation
        if renderer_config.Orientation == "XY":
            geometry = generate_xy_geometry(world_pos, voxel.color, renderer_config)
        elif renderer_config.Orientation == "XZ":
            geometry = generate_xz_geometry(world_pos, voxel.color, renderer_config)
        else:
            geometry = generate_cube_geometry(world_pos, voxel.color, renderer_config)
            
        static_batches[material_id].append(geometry)

    # Upload to dynamic vertex buffers
    upload_to_vbo(static_batches, renderer_config.models)
    if renderer_config.skinning:
        upload_to_vbo(skinned_batches, renderer_config.skinned_models)

    # Step 3: Drawing
    bind_pipeline(renderer_config.pipeline_basic)
    bind_descriptor_set(renderer_config.env_descriptor)
    
    for material_id, batches in static_batches.items():
        bind_material_descriptor(material_id, renderer_config.materials)
        for batch in batches:
            bind_vertex_buffer(batch.vbo)
            bind_index_buffer(batch.ibo)
            draw_elements(batch.index_count, batch.instance_count)
```

**Formulas:**

*   **Vertex Position Transformation**:
    $$ P_{world} = M_{local\_to\_world} \times P_{local} $$
    Where $M_{local\_to\_world}$ is the `LocalToWorld` matrix and $P_{local}$ is the voxel's local position [S1, unofficial].

*   **Quad Generation (XY Orientation)**:
    Given a voxel center $(x, y, z)$ and size $s$:
    $$
    \begin{aligned}
    v_0 &= (x + s/2, y + s/2, z) \\
    v_1 &= (x + s/2, y - s/2, z) \\
    v_2 &= (x - s/2, y - s/2, z) \\
    v_3 &= (x - s/2, y + s/2, z)
    \end{aligned}
    $$
    These vertices are connected using `SQUARE_LINE_INDICES` or `SQUARE_FACE_INDICES` [S1, unofficial].

*   **Orthographic Scale Adjustment**:
    If `ScaleOnZoom` is enabled and the camera is orthographic:
    $$ s_{effective} = s_{base} \times \frac{camera.orthographicSize}{10.0} $$
    This ensures voxels maintain visual consistency when zooming [S1, unofficial].

**Parameters:**
*   `LocalToWorld`: `Matrix4x4` transformation from local to world space. Default: Identity [S1, unofficial].
*   `Orientation`: Enum `DRAW_ORIENTATION` with values `XY` or `XZ`. Default: `XY` [S1, unofficial].
*   `DefaultColor`: `Color` for voxels without explicit color. Default: White [S1, unofficial].
*   `CullMode`: Rendering cull mode. Default: Off [S1, unofficial].
*   `transparency`: `bool` determines whether the pass handles transparent or opaque voxels. Controls alpha filtering and blend state. Default: False [S2, unofficial].
*   `skinning`: `bool` enables the construction and use of a secondary pipeline with skinning support. Default: False [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Alpha Mismatch**: Meshes are only processed if their alpha state matches the pass configuration. If a mesh has alpha 0.5 but the pass is configured for opaque (`transparency=False`), it will be excluded, potentially causing visual artifacts or missing data [S2, unofficial].
*   **Precision**: Using `float` for vertex positions and transformations may lead to jittering in large worlds. Consider using double precision or chunk-based coordinate systems for very large scenes [G].
*   **Degenerate Input**: Empty voxel lists or zero-size voxels should be handled gracefully to avoid division by zero or empty draw calls [G].

## Subtopics

### [OpenGL Client State Array Rendering](11008-opengl-client-state-array-rendering.md)
Renders pre-generated voxel mesh data using OpenGL client state arrays for vertices, normals, and colors, supporting both indexed and non-indexed drawing. It differs from modern core-profile rendering by relying on per-draw-call state enablement and pointer specification rather than encapsulated Vertex Array Objects (VAOs).

### [Dynamic VBO with Octree Rendering](13670-dynamic-vbo-with-octree-rendering.md)
A rendering approach for voxel terrain using an octree data structure where a single dynamic Vertex Buffer Object (VBO) is rebuilt every frame to contain geometry for visible octree nodes, combined with an index buffer. Unlike fixed-chunk approaches, this method rebuilds the VBO every frame based on visibility, allowing for dynamic LOD and culling. It builds on [Base Renderer Geometry](3902-base-renderer-geometry.md) by replacing static geometry uploads with a per-frame visibility-driven reconstruction.

### [Geometry Shader Voxel Cube Generation with Face Culling](16232-geometry-shader-voxel-cube-generation-with-face-culling.md)
This method renders voxel cubes by generating triangle geometry from point primitives within a geometry shader. It solves the problem of rendering large voxel scenes efficiently by using the alpha component of the input vertex color as a bitmask to determine which of the six cube faces to generate, thereby performing occlusion culling of faces adjacent to other cubes. It differs from sibling methods by generating geometry on the GPU per-voxel rather than batching static indices, and from [Dynamic VBO with Octree Rendering](13670-dynamic-vbo-with-octree-rendering.md) by not requiring hierarchical data structures or dynamic buffer updates.

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels, as each voxel is processed independently to generate geometry. Batching reduces draw call overhead to $O(M)$ where $M$ is the number of unique materials [S2, unofficial].
*   **Memory Complexity**: $O(N)$ for vertex and index buffers. The size depends on the chosen geometry (lines vs. quads vs. cubes) [S1, S2, unofficial].
*   **Measured Performance**: Performance is highly dependent on the number of draw calls. Batching by material is crucial for performance [S2, unofficial].
*   **Scaling**: Scales linearly with the number of voxels. For very large scenes, consider chunking or LOD techniques [G].
*   **Strengths**: Simple to implement, leverages standard graphics pipeline features (depth testing, blending), supports transparency [S1, S2, unofficial].
*   **Limitations**: High draw call count if not batched, geometry overhead for large cubes, no inherent occlusion culling (unless using geometry shaders or octrees) [S1, S2, unofficial].
*   **When to Use**: Small to medium voxel scenes, debugging, educational purposes, or when transparency is required and ray-marching is too expensive [S1, S2, unofficial].
*   **When Not to Use**: Very large voxel worlds without chunking, scenes requiring high-performance occlusion culling, or when memory bandwidth is a bottleneck [G].

## Background
Base Renderer Geometry originates from early voxel rendering techniques that utilized the graphics pipeline's rasterization capabilities. It is commonly found in voxel engines and tools that prioritize simplicity and compatibility with standard rendering APIs [S1, S2, unofficial]. The method has been adapted in various forms, including OpenGL client state arrays and modern dynamic VBO approaches, to optimize performance and flexibility [S1, S2, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] Kurble/amethyst_voxel, https://github.com/Kurble/amethyst_voxel, fetched 2026-10-27 (unofficial)
