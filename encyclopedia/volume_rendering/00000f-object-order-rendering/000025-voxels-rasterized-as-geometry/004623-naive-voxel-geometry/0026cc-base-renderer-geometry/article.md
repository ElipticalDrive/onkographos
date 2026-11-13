# Base Renderer Geometry

Also called: Cube Indices Layout, DrawVoxel, Square Face Indices Layout, Square Line Indices Layout, VoxelPass

## Overview
Base Renderer Geometry is a method for rendering voxel data by converting occupied voxel cells into explicit geometric primitives (cubes, squares, or lines) and submitting them to the graphics pipeline for rasterization [S1, S2, unofficial]. It solves the problem of visualizing scalar field data by treating voxels as discrete mesh objects rather than ray-marching a volume [G]. It differs from [Naive Voxel Rendering](1103-naive-voxel-rendering.md) by explicitly generating vertex/index buffers for geometry submission, and from [Cube Geometry for Volume Rendering](10885-cube-geometry-for-volume-rendering.md) by supporting multiple primitive topologies (faces, lines, full cubes) and dynamic batching strategies [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of occupied voxel coordinates, a transformation matrix ($LocalToWorld$), a color/texture atlas, and rendering parameters (orientation, culling mode, transparency flag) [S1, S2, unofficial].
*   **Output:** Rasterized pixels on the screen, with depth buffer updates and optional alpha blending [S2, unofficial].

**Data Structures:**
The method relies on specific index layouts to define primitives from a set of vertices [S1, unofficial]:
*   **Vertices:** A list of 3D positions ($Vector4$ or $Vector3$) and colors ($Color$) [S1, unofficial].
*   **Indices:**
    *   `CUBE_INDICES`: An integer array of 24 elements defining the 12 edges of a cube for line rendering: `[0, 1, 1, 2, 2, 3, 3, 0, 4, 5, 5, 6, 6, 7, 7, 4, 0, 4, 1, 5, 2, 6, 3, 7]` [S1, unofficial].
    *   `SQUARE_LINE_INDICES`: An integer array of 8 elements defining the perimeter of a square: `[0, 1, 1, 2, 2, 3, 3, 0]` [S1, unofficial].
    *   `SQUARE_FACE_INDICES`: An integer array of 6 elements defining two triangles for a square face: `[0, 1, 2, 0, 2, 3]` [S1, unofficial].
*   **Batching Structures (Advanced):** In dynamic implementations, a `DrawVoxel` struct manages `static_batches` and `skinned_batches` (organized by Material ID and mesh type), along with dynamic vertex buffers (`models`, `skinned_models`) and pipeline handles [S2, unofficial]. Vertex formats may include Position, Normal, Tangent, and Surface attributes (packing texture coordinates and ambient occlusion into RGB 32-bit floats) [S2, unofficial].

**Numbered Steps:**

1.  **Pipeline Construction:**
    *   Create a graphics pipeline layout from descriptor set layouts [S2, unofficial].
    *   Configure the basic pipeline state: enable back-face culling, set depth test function to `Less`, and set depth write to `true` if the material is not transparent [S2, unofficial].
    *   If transparency is enabled, configure alpha blending (e.g., premultiplied alpha) [S2, unofficial].
    *   If skinning is enabled, construct a child pipeline with a different vertex descriptor and skinned vertex shader [S2, unofficial].

2.  **Preparation (Data Fetching and Batching):**
    *   Fetch voxel mesh data, atlas handles, transform matrices, joint transforms (if skinned), and tint colors from the world/entity storage [S2, unofficial].
    *   Filter meshes based on the pass's transparency setting. Meshes are excluded if their alpha state does not match the pass configuration (e.g., opaque meshes in a transparent pass) [S2, unofficial].
    *   Group filtered meshes by Material ID.
    *   Insert grouped meshes into `static_batches` or `skinned_batches` depending on the skinning flag [S2, unofficial].
    *   Write batch data into dynamic vertex buffers (`models` or `skinned_models`) [S2, unofficial].

3.  **Coordinate Transformation:**
    *   For each voxel, calculate its world position. If using a local-to-world matrix ($LocalToWorld$), apply it to the voxel's local coordinates [S1, unofficial].
    *   For unit voxels centered at integer indices $(i, j, k)$ with size $V_{size}$, the vertices are generated relative to the center:
        $$
        P_{vertex} = P_{center} + \text{offset} \times V_{size}
        $$
        where $\text{offset} \in \{-0.5, 0.5\}$ for each axis [S1, unofficial].

4.  **Primitive Generation:**
    *   Based on the `Orientation` parameter (XY or XZ) and the chosen index layout (`SQUARE_FACE_INDICES`, `SQUARE_LINE_INDICES`, or `CUBE_INDICES`), generate the vertex and index data for each voxel [S1, unofficial].
    *   For XY orientation, vertices are generated in the XY plane with Z constant [S1, unofficial].
    *   For XZ orientation, vertices are generated in the XZ plane with Y constant [S1, unofficial].

5.  **Drawing:**
    *   Bind the basic graphics pipeline and environment descriptor set [S2, unofficial].
    *   Iterate over `static_batches` (or `skinned_batches` if skinning is active) [S2, unofficial].
    *   For each material, bind the material descriptor set [S2, unofficial].
    *   Iterate over mesh batches and issue draw calls using the appropriate index buffer and instance ranges [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxels(voxel_data, local_to_world_matrix, orientation, cull_mode, is_transparent):
    """
    Renders voxels using Base Renderer Geometry.
    
    Args:
        voxel_data: List of occupied voxel centers (x, y, z) and colors.
        local_to_world_matrix: 4x4 transformation matrix.
        orientation: Enum (XY or XZ).
        cull_mode: Enum (Off, CW, CCW).
        is_transparent: Boolean for alpha blending.
    """
    
    # Step 1: Pipeline Setup
    pipeline = create_pipeline(
        cull_mode=cull_mode,
        depth_test="Less",
        depth_write=not is_transparent,
        blend_mode="PremultipliedAlpha" if is_transparent else "None"
    )
    
    # Step 2: Preparation & Batching
    static_batches = {} # Map: MaterialID -> List[VoxelBatch]
    
    for voxel in voxel_data:
        # Filter by transparency
        if is_transparent != voxel.is_transparent:
            continue
            
        material_id = voxel.material_id
        if material_id not in static_batches:
            static_batches[material_id] = []
            
        # Generate geometry for this voxel
        vertices, indices = generate_voxel_geometry(
            center=voxel.center,
            color=voxel.color,
            orientation=orientation,
            index_layout="SQUARE_FACE_INDICES" # Example layout
        )
        
        batch = VoxelBatch(vertices=vertices, indices=indices, material_id=material_id)
        static_batches[material_id].append(batch)
        
    # Step 3: Drawing
    bind_pipeline(pipeline)
    bind_descriptor_set(pipeline, "env", get_environment_data())
    
    for material_id, batches in static_batches.items():
        bind_descriptor_set(pipeline, "material", get_material_data(material_id))
        
        for batch in batches:
            bind_vertex_buffer(batch.vertices)
            bind_index_buffer(batch.indices)
            draw_elements(mode="TRIANGLES", count=len(batch.indices)) # Or LINES/QUADS

def generate_voxel_geometry(center, color, orientation, index_layout):
    """
    Generates vertex and index data for a single voxel.
    """
    half_size = 0.5
    vertices = []
    
    # Define local cube corners relative to center
    # 0: (-1,-1,-1), 1: (1,-1,-1), 2: (1,1,-1), 3: (-1,1,-1)
    # 4: (-1,-1,1), 5: (1,-1,1), 6: (1,1,1), 7: (-1,1,1)
    local_offsets = [
        [-1, -1, -1], [1, -1, -1], [1, 1, -1], [-1, 1, -1],
        [-1, -1, 1], [1, -1, 1], [1, 1, 1], [-1, 1, 1]
    ]
    
    for offset in local_offsets:
        # Transform to world space
        local_pos = Vector4(offset[0]*half_size, offset[1]*half_size, offset[2]*half_size, 1.0)
        world_pos = local_to_world_matrix * local_pos
        vertices.append(Vertex(pos=world_pos.xyz, color=color))
        
    if index_layout == "SQUARE_FACE_INDICES":
        # Simplified for XY plane example
        # Assuming we only render front/back or specific faces based on orientation
        # This is a placeholder for specific face selection logic
        indices = [0, 1, 2, 0, 2, 3] # Example quad
    elif index_layout == "CUBE_INDICES":
        indices = [0, 1, 1, 2, 2, 3, 3, 0, 4, 5, 5, 6, 6, 7, 7, 4, 0, 4, 1, 5, 2, 6, 3, 7]
    else:
        indices = []
        
    return vertices, indices
```

**Parameters:**
*   `LocalToWorld`: Matrix4x4 transformation matrix from local to world space. Default: Identity [S1, unofficial].
*   `Orientation`: Enum `DRAW_ORIENTATION` with values `XY` or `XZ`. Default: `XY` [S1, unofficial].
*   `DefaultColor`: Color used for default rendering. Default: White [S1, unofficial].
*   `CullMode`: Rendering cull mode. Default: Off [S1, unofficial].
*   `transparency`: Boolean determining whether the pass handles transparent or opaque voxels; controls alpha filtering and blend state [S2, unofficial].
*   `skinning`: Boolean enabling the construction and use of a secondary pipeline with skinning support [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Alpha Mismatch:** Meshes are only processed if their alpha state matches the pass configuration. Mismatched meshes are excluded to prevent incorrect blending or culling [S2, unofficial].
*   **Precision:** Floating-point precision in vertex generation may cause gaps between adjacent voxels if not handled with slight overlaps or proper depth biasing [G].
*   **Boundary Conditions:** Frustum culling must be applied to exclude voxels outside the camera view to manage performance [G].

## Subtopics

### [OpenGL Client State Array Rendering](11008-opengl-client-state-array-rendering.md)
Renders pre-generated voxel mesh data by uploading vertex, normal, and color attributes to OpenGL client state arrays. It differs from modern core-profile rendering by relying on immediate-mode-style state enablement and pointer specification per draw call rather than encapsulated Vertex Array Objects (VAOs).

### [Dynamic VBO with Octree Rendering](13670-dynamic-vbo-with-octree-rendering.md)
A rendering approach for voxel terrain using an octree data structure where a single dynamic Vertex Buffer Object (VBO) is rebuilt every frame to contain geometry for visible octree nodes. Unlike fixed-chunk approaches, this method rebuilds the VBO every frame based on visibility, allowing for dynamic LOD and culling.

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the number of visible voxels, as each voxel requires geometry generation and a draw call (or batched draw call). Batching reduces the number of draw calls to $O(M)$ where $M$ is the number of unique materials [S2, unofficial].
*   **Memory Complexity:** $O(V)$ for storing vertex and index data in dynamic buffers [S2, unofficial].
*   **Performance:** Performance is heavily dependent on the number of draw calls. Batching by material is critical for performance [S2, unofficial].
*   **Strengths:** Simple to implement; leverages standard rasterization pipeline; supports complex materials via texture atlases [S2, unofficial].
*   **Limitations:** High vertex count for large volumes; draw call overhead if not batched; transparency sorting issues if not handled carefully [G].
*   **When to use:** Small to medium voxel volumes; when material complexity (textures, lighting) is high; when dynamic updates are frequent but volume size is manageable [S2, unofficial].
*   **When not to use:** Very large volumes where ray marching or tile-based rendering is more efficient; when memory bandwidth is a bottleneck [G].

## Background
Base Renderer Geometry is a fundamental technique in voxel rendering, often implemented in game engines and voxel editors. It is closely associated with the Amethyst voxel rendering system and various OpenGL-based voxel implementations [S1, S2, unofficial]. The method evolved from simple immediate-mode rendering of cubes to more optimized batched and dynamic VBO-based approaches [S1, S2, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S2] Kurble/amethyst_voxel, https://github.com/Kurble/amethyst_voxel, fetched 2026-10-27 (unofficial)
